# Nilix Syzkaller-Style Fuzzer Architecture

**Date:** 2026-07-21
**Status:** Design Document
**Goal:** Implement Linux syzkaller-equivalent coverage-guided kernel fuzzing for Nilix

---

## Executive Summary

This document specifies the architecture for a syzkaller-inspired fuzzer and KCOV-like coverage infrastructure for Nilix. The design prioritizes **incremental value delivery**, **CI integration**, and **security preservation** while achieving deep kernel testing.

**Key Innovation:** Hybrid approach combining compile-time LLVM coverage with runtime kernel buffers, enabling both fast userspace fuzzing (cargo-fuzz) and deep whole-kernel fuzzing (QEMU execution).

---

## 1. Architecture Overview

### 1.1 Components

```
┌─────────────────────────────────────────────────────────────┐
│                    Fuzzing Orchestrator                     │
│  (Rust userspace: corpus management, mutation, scheduling)  │
└────────┬────────────────────────────────┬───────────────────┘
         │                                │
         ├─ Fast Path: cargo-fuzz        │ Deep Path: QEMU
         │  (component fuzzing)           │ (whole-kernel)
         │                                │
┌────────▼────────┐              ┌────────▼──────────────────┐
│  libFuzzer      │              │  Nilix Kernel (QEMU)      │
│  Harness        │              │  + KCOV instrumentation   │
│  (host process) │              │  + syscall executor       │
└─────────────────┘              └────────┬──────────────────┘
                                          │
                                  ┌───────▼──────────┐
                                  │ Coverage Buffer  │
                                  │ (per-task kernel)│
                                  └──────────────────┘
```

### 1.2 Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Coverage Method** | LLVM SanitizerCoverage (`-Zinstrument-coverage`) | Native Rust support, edge coverage, minimal runtime overhead |
| **Syscall Descriptions** | Declarative TOML + Rust proc-macro codegen | Type-safe, maintainable, supports both generation and validation |
| **Execution Model** | Hybrid: cargo-fuzz (fast) + QEMU (deep) | Incremental complexity, reuses existing infrastructure |
| **Corpus Storage** | Flat files + SQLite metadata | Simple, CI-friendly, supports minimization |
| **Coverage Feedback** | Per-task kernel buffer (4KB) + shmem export | IRQ-safe, low overhead, battle-tested (Linux KCOV) |

---

## 2. KCOV Implementation

### 2.1 Compile-Time Instrumentation

Use LLVM's SanitizerCoverage via Rust's `-Zinstrument-coverage`:

```rust
// .cargo/config.toml (kernel crate)
[build]
rustflags = [
    "-Zinstrument-coverage",
    "-C llvm-args=--sanitizer-coverage-level=3",  # Edge coverage
    "-C llvm-args=--sanitizer-coverage-trace-pc-guard",
]
```

**Generated code:** LLVM inserts calls to `__sanitizer_cov_trace_pc_guard(u32* guard)` at every basic block edge.

### 2.2 Kernel-Side Coverage Buffer

```rust
// kernel/coverage/lib.rs

/// Per-task coverage buffer (4 KB = 4096 edges)
pub struct CoverageBuffer {
    edges: [u32; 1024],      // Bitmap: each u32 covers 32 edges
    edge_count: usize,       // Total edges hit this task
    enabled: bool,
}

/// Global coverage state
static COVERAGE: CpuLocal<Option<Arc<Mutex<CoverageBuffer>>>> = CpuLocal::new(|| None);

#[no_mangle]
pub extern "C" fn __sanitizer_cov_trace_pc_guard(guard: *mut u32) {
    if guard.is_null() {
        return;
    }
    
    // Safety: called from instrumented code, guard is valid
    let edge_id = unsafe { *guard };
    if edge_id == 0 {
        return; // Uninitialized guard
    }
    
    // Record edge hit (IRQ-safe: no allocations, atomic bit set)
    if let Some(buf) = COVERAGE.get() {
        if let Ok(mut buf) = buf.try_lock() {
            let idx = (edge_id / 32) as usize;
            let bit = edge_id % 32;
            if idx < buf.edges.len() {
                let prev = buf.edges[idx];
                buf.edges[idx] |= 1 << bit;
                if prev != buf.edges[idx] {
                    buf.edge_count += 1;
                }
            }
        }
    }
}

#[no_mangle]
pub extern "C" fn __sanitizer_cov_trace_pc_guard_init(start: *mut u32, stop: *mut u32) {
    // LLVM calls this once at module init to assign edge IDs
    let count = unsafe { stop.offset_from(start) } as usize;
    kprintln!("[KCOV] Initialized {} edge guards", count);
    
    // Assign sequential IDs
    let mut edge_id = 1u32;
    let mut ptr = start;
    while ptr < stop {
        unsafe {
            *ptr = edge_id;
            ptr = ptr.add(1);
        }
        edge_id += 1;
    }
}
```

### 2.3 Syscall Integration

Add coverage control syscalls:

```rust
// kernel/kernel_core/syscall.rs

/// Enable coverage collection for current task
/// syscall(SYS_kcov_enable, buf_size: usize) -> i32
pub fn sys_kcov_enable(buf_size: usize) -> Result<(), SyscallError> {
    if buf_size > 1024 * 1024 {
        return Err(SyscallError::InvalidArgument);
    }
    
    let buf = Arc::new(Mutex::new(CoverageBuffer::new()));
    COVERAGE.set(Some(buf.clone()));
    
    // Store in current task for later retrieval
    with_current_process(|proc| {
        proc.coverage_buffer = Some(buf);
    })?;
    
    Ok(())
}

/// Retrieve coverage data
/// syscall(SYS_kcov_dump, user_buf: *mut u32, len: usize) -> isize
pub fn sys_kcov_dump(user_buf: usize, len: usize) -> Result<isize, SyscallError> {
    let buf = with_current_process(|proc| {
        proc.coverage_buffer.clone()
    })?.ok_or(SyscallError::InvalidArgument)?;
    
    let buf_lock = buf.lock();
    let to_copy = core::cmp::min(len, buf_lock.edges.len() * 4);
    
    // Copy to userspace
    usercopy::copy_to_user(
        user_buf,
        buf_lock.edges.as_ptr() as usize,
        to_copy,
    )?;
    
    Ok(buf_lock.edge_count as isize)
}
```

---

## 3. Syscall Description Language

### 3.1 TOML Schema

```toml
# fuzz/syscall_descriptions/open.toml

[[syscall]]
name = "open"
number = 2
abi = "x86_64"

[[syscall.args]]
name = "pathname"
type = "ptr"
pointee = "string"
constraints = [
    { type = "length", min = 1, max = 4096 },
    { type = "charset", allowed = "printable" },
]
resources = ["creates:fd"]

[[syscall.args]]
name = "flags"
type = "i32"
constraints = [
    { type = "bitmask", values = ["O_RDONLY", "O_WRONLY", "O_RDWR", "O_CREAT", "O_TRUNC", "O_APPEND"] },
]

[[syscall.args]]
name = "mode"
type = "u32"
constraints = [
    { type = "range", min = 0, max = 0o777 },
]
conditional = { depends_on = "flags", has_flag = "O_CREAT" }

[[syscall.returns]]
success = { type = "fd", range = [0, 65535] }
error = { type = "errno", values = ["ENOENT", "EACCES", "ENOMEM", "EMFILE"] }
```

### 3.2 Codegen (Proc Macro)

```rust
// fuzz/syscall_gen/src/lib.rs

use proc_macro::TokenStream;
use syn::parse_macro_input;

#[proc_macro]
pub fn syscall_descriptions(_input: TokenStream) -> TokenStream {
    let descriptions = load_toml_files("fuzz/syscall_descriptions/*.toml");
    
    let generators = descriptions.iter().map(|desc| {
        quote! {
            pub struct #desc.name_Generator {
                rng: StdRng,
            }
            
            impl SyscallGenerator for #desc.name_Generator {
                fn generate(&mut self) -> SyscallArgs {
                    SyscallArgs {
                        number: #desc.number,
                        args: vec![
                            #(generate_arg(&desc.args[i], &mut self.rng)),*
                        ],
                    }
                }
            }
        }
    });
    
    quote! {
        #(#generators)*
    }.into()
}
```

---

## 4. Implementation Phases

### Phase 1: Minimal KCOV (Week 1)
**Goal:** Prove coverage instrumentation works

**Deliverables:**
- `kernel/coverage/` crate with `__sanitizer_cov_trace_pc_guard` implementation
- Conditional compilation: `--features kcov` enables instrumentation
- Boot test: print "KCOV: N edges initialized" to serial
- Single-syscall manual test: call `getpid()` 100x, verify edge count increases

**Acceptance:** Kernel boots with KCOV, prints edge count, no performance regression in normal builds.

---

### Phase 2: Syscall Executor (Week 2)
**Goal:** Execute multi-syscall sequences from userspace

**Deliverables:**
- `sys_kcov_enable/dump` syscalls (numbers TBD)
- Userspace harness: `userspace/fuzz_executor.rs`
  - Reads serialized syscall sequences from stdin
  - Executes each syscall via inline assembly
  - Dumps coverage to stdout
- End-to-end test: execute `[open("/tmp/test"), write(fd, "hi"), close(fd)]`, retrieve coverage

**Acceptance:** Userspace program executes syscall sequences, coverage data flows back.

---

### Phase 3: Basic Syscall Descriptions (Week 3)
**Goal:** Generate valid syscall arguments

**Deliverables:**
- TOML schema for 10 core syscalls: `open, read, write, close, fork, execve, mmap, munmap, getpid, exit`
- Proc macro: `syscall_descriptions!()` generates Rust structs
- Random generator: produce valid arguments per constraints
- Validation: ensure generated args satisfy constraints before execution

**Acceptance:** Generator produces 1000 syscall sequences, 90%+ execute without instant EFAULT.

---

### Phase 4: Coverage-Guided Mutation (Week 4)
**Goal:** Use coverage feedback to guide fuzzing

**Deliverables:**
- Corpus storage: `fuzz/corpus/{syscall_name}/{hash}.bin`
- Mutation engine:
  - Bit flips, arithmetic mutations, splice existing corpus
  - Coverage-guided selection: prioritize inputs that hit new edges
- Coverage tracking: SQLite DB with `(input_hash, edge_bitmap, crash_status)`
- Minimization: reduce crashing inputs to minimal reproducer

**Acceptance:** Fuzzer discovers new code paths over time, corpus grows, coverage increases monotonically.

---

### Phase 5: Resource Tracking (Week 5-6)
**Goal:** Multi-syscall sequences with dependencies

**Deliverables:**
- Resource tracker: map `fd` returned by `open` → consumed by `read/write/close`
- Dependency resolver: ensure `read(fd)` only generated after `open() → fd`
- Cleanup: add `close(fd)` before sequence end to prevent leak
- Handle types: `fd`, `pid`, `tid`, `timer_id`, `mmap_addr`

**Acceptance:** Generate valid sequences: `open→read→write→close`, `fork→waitpid`, `mmap→munmap`.

---

### Phase 6: Stateful Fuzzing (Week 7-8)
**Goal:** Build complex kernel state across calls

**Deliverables:**
- State model: track kernel objects (processes, fds, memory mappings, futexes)
- Multi-threaded sequences: `clone(CLONE_VM) → futex_wait/wake`
- Namespace operations: `unshare(CLONE_NEWNS) → mount → umount`
- Error injection: occasionally pass invalid args to test error paths

**Acceptance:** Fuzzer generates 100-syscall sequences building realistic state (e.g., multi-process IPC).

---

### Phase 7: CI Integration & Continuous Fuzzing (Week 9)
**Goal:** Production-ready fuzzing infrastructure

**Deliverables:**
- GitHub Actions workflow: `.github/workflows/fuzz-continuous.yml`
  - Runs 24h fuzzing jobs on dedicated runners
  - Uploads crashes/coverage to artifacts
- Crash triage: deduplicate by stack trace, prioritize by severity
- Dashboard: HTML report showing coverage trends, unique crashes, corpus size
- Incremental fuzzing: resume from previous corpus

**Acceptance:** CI runs fuzzing daily, surfaces crashes as GitHub issues, coverage dashboard accessible.

---

## 5. Integration with Existing Infrastructure

### 5.1 Preserve Security Hardening

- **SMAP:** Coverage buffer access uses `copy_to_user`, respects AC flag
- **Stack guards:** No recursion in coverage hooks, fixed-size buffer
- **IRQ safety:** Coverage hooks use `try_lock()`, never block
- **No allocations:** Coverage path is allocation-free (bitmap only)

### 5.2 Build System

```makefile
# Makefile

.PHONY: build-kcov fuzz-run fuzz-corpus

build-kcov:
	cd kernel && \
	RUSTFLAGS="-Zinstrument-coverage" \
	cargo build --release --features kcov --target x86_64-unknown-none

fuzz-run: build-kcov
	cd fuzz && \
	cargo run --bin orchestrator -- \
	  --kernel ../esp/kernel.elf \
	  --qemu qemu-system-x86_64 \
	  --duration 3600 \
	  --corpus corpus/

fuzz-corpus:
	cd fuzz && cargo run --bin corpus-minimize
```

### 5.3 Coexist with cargo-fuzz

Keep existing `fuzz/fuzz_targets/*` for fast component testing:

```rust
// fuzz/fuzz_targets/fuzz_vfs_path.rs

#![no_main]
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    // Fast: pure function fuzzing, no kernel boot
    vfs::normalize_path(data);
});
```

New whole-kernel fuzzing lives in `fuzz/orchestrator/`:

```rust
// fuzz/orchestrator/src/main.rs

fn main() {
    let mut fuzzer = Fuzzer::new();
    fuzzer.load_corpus("corpus/");
    
    loop {
        let input = fuzzer.mutate();
        let (coverage, crash) = execute_in_qemu(&input);
        fuzzer.feedback(input, coverage, crash);
    }
}
```

---

## 6. Syscall Coverage Priority

Focus on high-value, complex subsystems first:

**Tier 1 (Phase 3):** `open, read, write, close, mmap, munmap, fork, clone, waitpid, execve`
**Tier 2 (Phase 4):** `socket, bind, connect, sendto, recvfrom, poll, epoll_*`
**Tier 3 (Phase 5):** `futex, shm_open, mq_open, semop, msgctl`
**Tier 4 (Phase 6):** `unshare, setns, mount, umount, pivot_root, cgroup_*`
**Tier 5 (Phase 7):** `seccomp, prctl, ptrace, bpf, perf_event_open`

---

## 7. Expected Outcomes

### 7.1 Quantitative

- **Coverage:** 70%+ edge coverage of syscall paths by Phase 7
- **Bug discovery:** 5-10 new bugs per week during active development
- **Performance:** <5% overhead with KCOV enabled, 0% overhead without
- **CI runtime:** 1-2 hours/day fuzzing, fits in GitHub Actions free tier

### 7.2 Qualitative

- **Regression prevention:** Every fixed bug gets minimized reproducer in corpus
- **Confidence:** Continuous fuzzing provides signal before releases
- **Documentation:** Syscall descriptions serve as machine-readable spec
- **Upstream:** Architecture reusable for other Rust OS projects

---

## 8. Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **LLVM instrumentation breaks kernel** | Critical | Phase 1 proves it works; conditional compilation allows fallback |
| **Coverage overhead too high** | Medium | Use edge coverage (not full path), lazy initialization |
| **QEMU snapshot/restore complexity** | Medium | Start with simple execute-and-reset, defer snapshotting to Phase 6 |
| **Syscall description maintenance burden** | Low | Codegen reduces boilerplate; focus on high-value syscalls first |
| **CI resource exhaustion** | Medium | Cap fuzzing duration, use artifact pruning, self-hosted runners if needed |

---

## 9. References

- **Linux KCOV:** https://www.kernel.org/doc/html/latest/dev-tools/kcov.html
- **syzkaller:** https://github.com/google/syzkaller
- **LLVM SanitizerCoverage:** https://clang.llvm.org/docs/SanitizerCoverage.html
- **Rust `-Zinstrument-coverage`:** https://doc.rust-lang.org/rustc/instrument-coverage.html

---

## Appendix A: Syscall Number Allocation

Reserve syscall numbers for KCOV control:

```rust
// kernel/kernel_core/syscall.rs

const SYS_KCOV_INIT: usize = 500;      // Initialize coverage buffer
const SYS_KCOV_ENABLE: usize = 501;    // Enable for current task
const SYS_KCOV_DISABLE: usize = 502;   // Disable
const SYS_KCOV_DUMP: usize = 503;      // Dump coverage bitmap
const SYS_KCOV_RESET: usize = 504;     // Clear coverage data
```

---

**End of Architecture Document**
