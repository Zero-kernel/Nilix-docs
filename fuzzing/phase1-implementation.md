# Nilix Fuzzing Implementation Plan - Phase 1

**Date:** 2026-07-21
**Phase:** Phase 1 - Minimal KCOV
**Status:** In Progress
**Duration:** Week 1

---

## Overview

This document tracks the implementation of Phase 1: proving that LLVM SanitizerCoverage instrumentation works in Nilix kernel.

## Completed Work

### 1. Architecture Design ✓
- **File:** `docs/fuzzing/syzkaller-architecture.md`
- **Content:** Complete architecture spec covering all 7 phases
- **Status:** Written and uploaded to both local and remote

### 2. Coverage Module ✓
- **Files:** 
  - `kernel/coverage/lib.rs` - Core KCOV implementation
  - `kernel/coverage/Cargo.toml` - Module manifest
- **Features:**
  - Per-task coverage buffer (4KB bitmap)
  - LLVM callbacks: `__sanitizer_cov_trace_pc_guard` and `_init`
  - IRQ-safe recording (try_lock, no allocations)
  - Conditional compilation via `--features kcov`
- **Status:** Written and uploaded to both local and remote

## Remaining Phase 1 Tasks

### 3. Kernel Integration (IN PROGRESS)
**Goal:** Wire coverage module into kernel build and add syscall interface

**Subtasks:**
- [ ] Add `coverage` crate to kernel workspace dependencies
- [ ] Reserve syscall numbers for KCOV (520-524 range available)
- [ ] Implement syscall handlers:
  - `sys_kcov_init(buf_size: usize) -> Result<(), SyscallError>`
  - `sys_kcov_enable() -> Result<(), SyscallError>`
  - `sys_kcov_disable() -> Result<(), SyscallError>`
  - `sys_kcov_dump(user_buf: *mut u8, len: usize) -> Result<isize, SyscallError>`
  - `sys_kcov_reset() -> Result<(), SyscallError>`
- [ ] Add coverage buffer field to `Process` struct
- [ ] Initialize coverage subsystem in `main.rs`

**Files to modify:**
- `kernel/Cargo.toml` - Add coverage dependency
- `kernel/kernel_core/syscall.rs` - Add KCOV syscalls (lines ~3655+ after cgroup)
- `kernel/kernel_core/process.rs` - Add coverage_buffer field to Process
- `kernel/src/main.rs` - Call `coverage::init_coverage()` during boot

### 4. Build System Integration
**Goal:** Enable KCOV builds via Makefile target

**Subtasks:**
- [ ] Add `build-kcov` Makefile target
- [ ] Configure RUSTFLAGS for instrumentation:
  ```makefile
  build-kcov:
      cd kernel && \
      RUSTFLAGS="-Zinstrument-coverage -C llvm-args=--sanitizer-coverage-level=3" \
      cargo build --release --features kcov --target x86_64-unknown-none
  ```
- [ ] Add conditional compilation guards in coverage crate

**Files to modify:**
- `Makefile` - Add build-kcov target

### 5. Boot Test
**Goal:** Verify KCOV initializes correctly during kernel boot

**Subtasks:**
- [ ] Build kernel with `make build-kcov`
- [ ] Boot in QEMU: `make run`
- [ ] Verify serial output contains: `[KCOV] Initialized N edge guards`
- [ ] Ensure kernel boots successfully (no crashes from instrumentation)

**Acceptance criteria:**
- Kernel boots with KCOV enabled
- Edge guard count printed to serial (expected: 1000-10000 edges)
- No performance regression in non-KCOV builds

### 6. Manual Syscall Test
**Goal:** Prove coverage data collection works end-to-end

**Subtasks:**
- [ ] Write userspace test program: `userspace/kcov_test.rs`
  ```rust
  // Pseudo-code:
  // 1. syscall(SYS_KCOV_INIT, 4096)
  // 2. syscall(SYS_KCOV_ENABLE)
  // 3. for _ in 0..100 { syscall(SYS_GETPID) }
  // 4. let edges = syscall(SYS_KCOV_DUMP, buf, len)
  // 5. assert!(edges > 0)
  ```
- [ ] Build and run test: `make build-syscall-test && make run`
- [ ] Verify edge count increases (getpid path should hit ~10-50 edges)
- [ ] Verify repeated calls don't increase count (same edges)

**Files to create:**
- `userspace/kcov_test.rs` - Userspace test program

---

## Current Blocker

Need to determine available syscall numbers in 500+ range. Found:
- 500 = sys_cgroup_create
- Need to survey 501-519 to find available slots

**Next action:** Survey syscall dispatch in `kernel/kernel_core/syscall.rs` to find free numbers.

---

## Phase 1 Success Criteria

✓ = Done, ◯ = TODO

- [✓] Architecture document written
- [✓] Coverage module implemented
- [◯] KCOV syscalls added to kernel
- [◯] Kernel builds with `--features kcov`
- [◯] Kernel boots and prints edge count
- [◯] Userspace test collects coverage data
- [◯] Coverage increases on first syscall, stable on repeat

**Target completion:** 2026-07-22 (tomorrow)

---

## Known Risks

1. **LLVM instrumentation breaks kernel:** Mitigated by conditional compilation
2. **Performance overhead:** Measure and document (expect <5% with KCOV)
3. **SMAP violations:** Coverage buffer access must use copy_to_user

---

## Next Phase Preview

**Phase 2:** Syscall Executor (Week 2)
- Add userspace executor: reads syscall sequences from stdin
- Execute multi-syscall sequences
- Dump coverage after each sequence
- Target: 5-10 core syscalls (open, read, write, close, getpid)
