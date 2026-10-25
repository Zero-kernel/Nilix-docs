# Nilix Fuzz Infrastructure Implementation Plan

**Date:** 2026-08-04  
**Status:** Phase 1 - Foundation  
**Goal:** Achieve comprehensive fuzz coverage across all kernel subsystems

---

## Overview

This plan implements the roadmap from `fuzz-coverage-audit.md` in four phases:

1. **Foundation** (This session) - Critical P0 targets + syzkaller foundation
2. **Breadth** (Week 2) - Convert stubs + add P1 targets
3. **Depth** (Week 3-4) - P2 targets + feedback loop + stress gate
4. **Hardening** (Ongoing) - CI integration + corpus expansion

---

## Phase 1: Foundation (This Session)

### Objectives

✅ Complete systematic audit document  
🔄 Convert 3 critical stub targets to real implementations  
🔄 Add 2 new P0 host-harness targets  
🔄 Design syzkaller executor architecture  
🔄 Document stress gate enhancement plan

### Task 1.1: Convert fuzz_buddy_allocator (New P0 Target)

**Why:** Buddy allocator has no coverage despite R178-31, R180 heap budget fixes

**Attack Surface:**
- Allocation size overflow (e.g., size > available memory)
- Fragmentation edge cases (repeated alloc/free creating unusable gaps)
- Double-free detection
- Use-after-free detection
- Cross-order allocation corruption

**Implementation:**

```rust
// fuzz/fuzz_targets/fuzz_buddy_allocator.rs
#![no_main]

use arbitrary::Arbitrary;
use libfuzzer_sys::fuzz_target;

#[derive(Arbitrary, Debug, Clone)]
enum BuddyOperation {
    Allocate { order: u8 },
    Free { handle: u16 },
    AllocateContiguous { count: u16 },
    QueryFree,
}

#[derive(Arbitrary, Debug)]
struct BuddyFuzzInput {
    operations: Vec<BuddyOperation>,
}

fuzz_target!(|input: BuddyFuzzInput| {
    // Host harness simulating buddy allocator operations
    let mut allocator = mm::test_harness::BuddyAllocatorHarness::new();
    let mut handles = Vec::new();

    for op in input.operations.iter().take(1000) {
        match op {
            BuddyOperation::Allocate { order } => {
                let order = *order % 12; // Limit to reasonable orders (0-11 = 4KB to 8MB)
                if let Ok(frame) = allocator.allocate_frames(1 << order) {
                    handles.push(frame);
                }
            }
            BuddyOperation::Free { handle } => {
                if !handles.is_empty() {
                    let idx = (*handle as usize) % handles.len();
                    let frame = handles.swap_remove(idx);
                    let _ = allocator.free_frames(frame);
                }
            }
            BuddyOperation::AllocateContiguous { count } => {
                let count = (*count as usize).min(256); // Limit to 1MB
                if let Ok(frames) = allocator.allocate_contiguous(count) {
                    handles.extend(frames);
                }
            }
            BuddyOperation::QueryFree => {
                let _ = allocator.query_free_memory();
            }
        }
    }

    // Cleanup: free all remaining allocations
    for frame in handles {
        let _ = allocator.free_frames(frame);
    }

    // Verify: no leaked frames, no corruption
    allocator.verify_integrity();
});
```

**Host Harness Requirements:**

Add to `kernel/mm/src/lib.rs`:

```rust
#[cfg(feature = "host_harness")]
pub mod test_harness {
    use super::*;
    
    pub struct BuddyAllocatorHarness {
        // Simplified buddy allocator for host testing
        // Uses Vec instead of real physical frames
    }
    
    impl BuddyAllocatorHarness {
        pub fn new() -> Self { /* ... */ }
        pub fn allocate_frames(&mut self, count: usize) -> Result<Frame, AllocError> { /* ... */ }
        pub fn free_frames(&mut self, frame: Frame) -> Result<(), AllocError> { /* ... */ }
        pub fn verify_integrity(&self) { /* ... */ }
    }
}
```

**Success Criteria:**
- Compiles with `cargo fuzz build fuzz_buddy_allocator`
- Runs without panics for 60 seconds
- Detects known double-free patterns (add regression test)

---

### Task 1.2: Convert fuzz_page_table_ops (New P0 Target)

**Why:** Page table manipulation has no coverage despite R174-B3, R168-1

**Attack Surface:**
- Invalid PTE flags combinations (W+NX, U+S conflicts)
- Non-canonical addresses (holes in 48-bit space)
- TLB consistency violations
- PT-charge accounting bugs

**Implementation:**

```rust
// fuzz/fuzz_targets/fuzz_page_table_ops.rs
#![no_main]

use arbitrary::Arbitrary;
use libfuzzer_sys::fuzz_target;

#[derive(Arbitrary, Debug, Clone)]
enum PageTableOp {
    Map { va: u64, flags: u8 },
    Unmap { va: u64 },
    ChangeProtection { va: u64, new_flags: u8 },
    Lookup { va: u64 },
}

#[derive(Arbitrary, Debug)]
struct PageTableFuzzInput {
    operations: Vec<PageTableOp>,
}

fuzz_target!(|input: PageTableFuzzInput| {
    let mut pt_harness = mm::test_harness::PageTableHarness::new();

    for op in input.operations.iter().take(500) {
        match op {
            PageTableOp::Map { va, flags } => {
                // Canonicalize address to user space
                let va_canonical = *va & 0x0000_7FFF_FFFF_FFFF;
                let va_aligned = va_canonical & !0xFFF;
                
                // Parse flags (R/W/X/U)
                let readable = flags & 0x01 != 0;
                let writable = flags & 0x02 != 0;
                let executable = flags & 0x04 != 0;
                let user = flags & 0x08 != 0;
                
                let _ = pt_harness.map_page(va_aligned, readable, writable, executable, user);
            }
            PageTableOp::Unmap { va } => {
                let va_canonical = *va & 0x0000_7FFF_FFFF_FFFF;
                let va_aligned = va_canonical & !0xFFF;
                let _ = pt_harness.unmap_page(va_aligned);
            }
            PageTableOp::ChangeProtection { va, new_flags } => {
                let va_canonical = *va & 0x0000_7FFF_FFFF_FFFF;
                let va_aligned = va_canonical & !0xFFF;
                
                let readable = new_flags & 0x01 != 0;
                let writable = new_flags & 0x02 != 0;
                let executable = new_flags & 0x04 != 0;
                let user = new_flags & 0x08 != 0;
                
                let _ = pt_harness.change_protection(va_aligned, readable, writable, executable, user);
            }
            PageTableOp::Lookup { va } => {
                let va_canonical = *va & 0x0000_7FFF_FFFF_FFFF;
                let _ = pt_harness.lookup(va_canonical);
            }
        }
    }

    // Verify: no dangling PTEs, accounting correct
    pt_harness.verify_integrity();
});
```

**Host Harness Requirements:**

Add to `kernel/mm/src/lib.rs`:

```rust
#[cfg(feature = "host_harness")]
pub mod test_harness {
    pub struct PageTableHarness {
        // Simplified 4-level page table for host testing
        // Uses HashMap instead of real physical frames
    }
    
    impl PageTableHarness {
        pub fn new() -> Self { /* ... */ }
        pub fn map_page(&mut self, va: u64, r: bool, w: bool, x: bool, u: bool) -> Result<(), PTError> { /* ... */ }
        pub fn unmap_page(&mut self, va: u64) -> Result<(), PTError> { /* ... */ }
        pub fn change_protection(&mut self, va: u64, r: bool, w: bool, x: bool, u: bool) -> Result<(), PTError> { /* ... */ }
        pub fn lookup(&self, va: u64) -> Option<PageTableEntry> { /* ... */ }
        pub fn verify_integrity(&self) { /* ... */ }
    }
}
```

**Success Criteria:**
- Compiles with `cargo fuzz build fuzz_page_table_ops`
- Runs without panics for 60 seconds
- Detects W^X violations, canonicality errors

---

### Task 1.3: Convert fuzz_syscall from Mock to QEMU Executor

**Why:** Current fuzz_syscall uses MockKernelContext, not real kernel

**Current State:**
```rust
// fuzz/fuzz_targets/fuzz_syscall.rs (current)
fuzz_target!(|data: &[u8]| {
    let mut ctx = MockKernelContext::new();
    let _ = ctx.syscall(nr, args);  // NOT REAL KERNEL
});
```

**Target State:**
```rust
// fuzz/fuzz_targets/fuzz_syscall.rs (new)
fuzz_target!(|data: &[u8]| {
    // Parse syscall sequence from fuzzer input
    let program = parse_syscall_program(data);
    
    // Send to QEMU guest via virtio-serial
    let result = qemu_executor::execute_syscall_program(program);
    
    // Collect KCOV coverage
    let coverage = result.coverage_bitmap;
});
```

**Architecture:**

```
┌─────────────────┐
│ cargo fuzz run  │  (Host process)
└────────┬────────┘
         │
         ├─> Parse fuzzer input into syscall program
         │
         ├─> Launch QEMU with KCOV kernel
         │
         ├─> Send program via virtio-serial
         │        │
         │        v
         │   ┌────────────────────────┐
         │   │  Nilix Kernel (QEMU)   │
         │   │  + syz-executor.elf    │
         │   └────────────────────────┘
         │        │
         │        ├─> Execute syscalls with KCOV
         │        ├─> Collect coverage bitmap
         │        └─> Report crashes/hangs
         │
         └─< Receive coverage + result
```

**Implementation Steps:**

1. **Add QEMU executor module** (`fuzz/src/qemu_executor.rs`):

```rust
pub struct QemuExecutor {
    qemu_process: Child,
    virtio_serial: VirtioSerialChannel,
}

impl QemuExecutor {
    pub fn new(kernel_path: &Path) -> Result<Self> {
        // Launch QEMU with virtio-serial
        // Connect to serial channel
    }
    
    pub fn execute_program(&mut self, program: &[u8]) -> ExecutionResult {
        // Send program to guest
        // Wait for result (with timeout)
        // Parse coverage bitmap
    }
}

pub struct ExecutionResult {
    pub exit_code: i32,
    pub coverage: Vec<u64>,
    pub crashed: bool,
    pub timeout: bool,
}
```

2. **Add syz-executor userspace program** (`userspace/syz_executor/src/main.rs`):

```rust
// Ring-3 program that receives syscall programs and executes them
fn main() {
    // Open virtio-serial device
    let serial = open("/dev/vport0p1").unwrap();
    
    loop {
        // Read syscall program
        let program = read_program(&serial);
        
        // Enable KCOV
        syscall(SYS_KCOV_INIT_TRACE, ...);
        
        // Execute syscalls
        for call in program.calls {
            let _ = syscall(call.nr, call.args[0], ...);
        }
        
        // Collect coverage
        let coverage = collect_kcov_coverage();
        
        // Send result back
        write_result(&serial, &coverage);
    }
}
```

3. **Update fuzz_syscall target**:

```rust
#![no_main]
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    // Parse syscall program from fuzzer input
    if data.len() < 8 { return; }
    
    let mut offset = 0;
    let mut program = Vec::new();
    
    while offset + 56 <= data.len() {
        let nr = u64::from_le_bytes(data[offset..offset+8].try_into().unwrap());
        let args = [/* parse 6 args */];
        program.push(SyscallEntry { nr, args });
        offset += 56;
        
        if program.len() >= 10 { break; }  // Limit to 10 syscalls
    }
    
    // Execute via QEMU (reuse existing executor instance)
    static EXECUTOR: Lazy<Mutex<QemuExecutor>> = Lazy::new(|| {
        Mutex::new(QemuExecutor::new(Path::new("esp-kcov/kernel.elf")).unwrap())
    });
    
    let mut executor = EXECUTOR.lock().unwrap();
    let result = executor.execute_program(&program);
    
    // Coverage feedback happens automatically via libfuzzer
});
```

**Success Criteria:**
- QEMU executor launches and connects
- syz-executor receives and executes syscalls
- Coverage data flows back to fuzzer
- Runs for 5 minutes without hanging

---

### Task 1.4: Syzkaller Executor Design Document

**Goal:** Document the full syzkaller integration architecture

**Deliverable:** `docs/fuzzing/syzkaller-integration.md`

**Contents:**
1. Architecture diagram (syz-manager → syz-fuzzer → QEMU → syz-executor)
2. Syscall grammar specification (Nilix syscall definitions)
3. KCOV integration details
4. Build instructions (make build-syz-executor, make run-syz-fuzz)
5. Corpus seeding strategy
6. Crash triage workflow

---

### Task 1.5: Stress Gate Enhancement Design

**Goal:** Document multi-workload stress test scenarios

**Deliverable:** `docs/fuzzing/stress-gate-design.md`

**Contents:**
1. Current state analysis (placeholders in heavy_stress.rs)
2. Seven concrete workload specifications
3. Resource requirements (Ring-3 syscalls needed)
4. Success criteria per workload
5. Implementation roadmap

---

## Phase 2: Breadth (Week 2)

### Task 2.1: Convert Remaining Stubs

- fuzz_futex_ops (P0) - Real futex operations via QEMU
- fuzz_memory_ops (P1) - Real mmap/munmap/mprotect via QEMU
- fuzz_scheduler (P1) - Real sched_yield/setpriority via QEMU
- fuzz_signal_delivery (P1) - Real kill/sigaction via QEMU
- fuzz_cgroup_ops (P1) - Real cgroup operations via QEMU
- fuzz_ipc_message (P3) - Message queue operations

### Task 2.2: Add New P1 Targets

- fuzz_cow_fault - Multi-process COW trigger
- fuzz_fork_clone - Process lifecycle stress
- fuzz_pipe_ops - Pipe read/write/close races
- fuzz_file_io - VFS read/write operations
- fuzz_tcp_state - TCP state machine transitions

---

## Phase 3: Depth (Week 3-4)

### Task 3.1: Add P2 Targets

- fuzz_tlb_shootdown - TLB consistency under contention
- fuzz_page_cache - Page cache operations
- fuzz_seccomp - Seccomp filter evaluation
- fuzz_mount_ns - Mount namespace operations
- fuzz_socket_ops - Socket lifecycle

### Task 3.2: Syzkaller Feedback Loop

- Implement coverage-guided corpus selection
- Add crash minimization (syz-repro)
- Implement crash deduplication (stack hash)
- Wire up continuous fuzzing daemon

### Task 3.3: Stress Gate Implementation

- Implement 7 multi-workload scenarios in heavy_stress.rs
- Add CI integration (make stress-test-extended)
- Set success criteria (no crashes, no data corruption)

---

## Phase 4: Hardening (Ongoing)

### Task 4.1: CI Integration

- Add nightly fuzzing job (GitHub Actions)
- Run each target for 4 hours
- Upload crash reports to artifacts
- Alert on new crashes

### Task 4.2: Corpus Expansion

- Seed from boot test traces
- Seed from musl_check execution
- Seed from past audit round reproducers
- Auto-minimize corpus (deduplicate by coverage)

### Task 4.3: Crash Triage Automation

- Parse crash logs into structured findings
- Map crashes to source locations
- Deduplicate by stack hash
- Generate reproducer scripts

---

## Success Metrics (Revisited)

### Coverage (Measured via KCOV)

- **Baseline (now):** 40% line coverage from boot tests
- **Target (Phase 2):** 50% line coverage
- **Target (Phase 3):** 60% line coverage
- **Target (Phase 4):** 65% line coverage

### Bug Discovery Rate

- **Phase 1:** 2-3 findings (buddy allocator, PT ops, syscall validation)
- **Phase 2:** 5-8 findings (memory ops, futex, scheduler)
- **Phase 3:** 8-12 findings (broader subsystem coverage)
- **Phase 4:** 10-15 findings/month (continuous fuzzing)

### Performance

- **Cargo-fuzz executions:** 1000-5000 exec/sec per target (host harness)
- **Syzkaller executions:** 50-100 exec/sec per worker (QEMU overhead)
- **Parallel workers:** 4-8 workers on CI, 16+ on dedicated fuzz cluster

---

## File Tree After Phase 1

```
fuzz/
  fuzz_targets/
    fuzz_buddy_allocator.rs       [NEW]
    fuzz_page_table_ops.rs        [NEW]
    fuzz_syscall.rs                [CONVERTED to QEMU]
    fuzz_elf_loader.rs             [existing, real]
    fuzz_network_packet.rs         [existing, real]
    fuzz_vfs_path.rs               [existing, real]
    fuzz_futex_ops.rs              [stub, convert in Phase 2]
    fuzz_memory_ops.rs             [stub, convert in Phase 2]
    fuzz_scheduler.rs              [stub, convert in Phase 2]
    fuzz_signal_delivery.rs        [stub, convert in Phase 2]
    fuzz_cgroup_ops.rs             [stub, convert in Phase 2]
    fuzz_ipc_message.rs            [stub]
  src/
    lib.rs
    mock_kernel.rs                 [deprecate after QEMU executor]
    qemu_executor.rs               [NEW]
  corpus/
    syscall/                       [NEW]
    memory/                        [NEW]
    buddy_allocator/               [NEW]
    page_table/                    [NEW]

userspace/
  syz_executor/                    [NEW]
    src/main.rs
    Cargo.toml

docs/
  fuzzing/
    fuzz-coverage-audit.md         [DONE]
    implementation-plan.md         [THIS FILE]
    syzkaller-integration.md       [Phase 1 Task 1.4]
    stress-gate-design.md          [Phase 1 Task 1.5]
```

---

## Dependencies & Tooling

### Cargo-fuzz
- `cargo install cargo-fuzz`
- Already present in CI

### Syzkaller
- Clone from https://github.com/google/syzkaller
- Build syz-manager, syz-fuzzer (no kernel patching needed)
- Adapt syscall descriptions to Nilix ABI

### QEMU
- Already used for boot tests
- Add virtio-serial device for executor communication

### KCOV
- Already implemented in kernel (`--features kcov`)
- Expose via syscall(548, KCOV_INIT_TRACE)

---

## Next Steps (This Session)

1. ✅ Complete audit document
2. ✅ Complete implementation plan (this file)
3. 🔄 Implement fuzz_buddy_allocator + mm/test_harness
4. 🔄 Implement fuzz_page_table_ops + mm/test_harness
5. 🔄 Design QEMU executor architecture (partial - document skeleton)
6. 🔄 Create syzkaller-integration.md (skeleton)
7. 🔄 Create stress-gate-design.md (skeleton)

**End of Phase 1 Gate:**
- All 5 new/converted targets compile
- Run each for 60 seconds without panic
- Dual-write all files to remote
- Push to Git (if user requests)

---

## References

- **Audit:** `docs/fuzzing/fuzz-coverage-audit.md`
- **Memory Lessons:** `C:\Users\Admin\.claude\projects\D--project-Zero-os\memory\diagnosing-layout-dependent-heap-corruption.md`
- **KCOV Implementation:** `kernel/trace/kcov.rs`
- **Current Fuzz Targets:** `fuzz/fuzz_targets/`
- **Syzkaller Docs:** https://github.com/google/syzkaller/tree/master/docs
