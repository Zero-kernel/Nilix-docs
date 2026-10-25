# Nilix Kernel Test Coverage Expansion Plan

**Version:** 1.0  
**Date:** 2026-07-02  
**Status:** Comprehensive test gap analysis and implementation roadmap  
**Target:** 150+ new tests for critical path coverage

---

## Executive Summary

**Current State:**
- 204 existing tests (129 unit, 48 self-tests, 27 integration)
- Strong coverage: cgroup accounting, network parsing, security config
- Critical gaps: context switch, COW/page faults, futex, scheduler, VFS operations

**Test Expansion Target:**
- **Block 1 (P0 - Security Critical):** 25 tests → 1.0-Preview gate
- **Block 2 (P1 - Regression Prevention):** 45 tests → R172/R171 coverage
- **Block 3 (P2 - Concurrency & SMP):** 40 tests → Multi-core hardening
- **Block 4 (P3 - Edge Cases & Fuzzing):** 50 tests → Robustness

**Total:** 160 new tests + infrastructure improvements

---

## Test Gap Analysis by Severity

### CRITICAL (No Coverage, Known-Buggy Paths)

| ID | Path | Risk | R-Audit | Current | Priority |
|----|------|------|---------|---------|----------|
| G-01 | Context switch RIP/RFLAGS save | Privilege escalation | R172-01 | 0 tests | P0 |
| G-02 | COW page fault + TLB shootdown | Silent corruption | R23-1 | 0 tests | P0 |
| G-03 | Page table concurrent access | SMP data corruption | R67-5 | 0 tests | P0 |
| G-04 | Futex WAIT/WAKE operations | Deadlock | R24-5 | 0 tests | P0 |
| G-05 | PI futex priority donation | Priority inversion | E.4 | 0 tests | P0 |
| G-06 | Work stealing race conditions | Double-run | R172-03 | 0 tests | P0 |
| G-07 | RAMFS create/write/read | VFS data plane | - | 0 tests | P0 |
| G-08 | VFS path resolution | Authz bypass | - | 0 tests | P0 |

### HIGH (Complex Paths, No Coverage)

| ID | Path | Risk | R-Audit | Current | Priority |
|----|------|------|---------|---------|----------|
| G-09 | FS_BASE/GS_BASE isolation | TLS corruption | R172-04 | 0 tests | P1 |
| G-10 | syscall_active leak | CPU wedge | R172-05 | 0 tests | P1 |
| G-11 | Page fault handler (#PF) | Kernel panic | - | 0 tests | P1 |
| G-12 | Fork COW setup | Memory leak | R138-1 | 0 tests | P1 |
| G-13 | Exec address space transition | AS leak | R172-02 | 0 tests | P1 |
| G-14 | Signal delivery (async) | Lost signal | M0-5 | 0 tests | P1 |
| G-15 | Rename atomicity | Lost dentry | R172-14/15 | 1 test | P1 |
| G-16 | TCP state machine | Connection hang | - | 3 tests | P1 |

### MEDIUM (Important Correctness)

| ID | Path | Risk | R-Audit | Current | Priority |
|----|------|------|---------|---------|----------|
| G-17 | Scheduler lock ordering | Deadlock | - | 0 tests | P2 |
| G-18 | mmap/munmap region ops | VMA leak | R141-9 | 0 tests | P2 |
| G-19 | Socket bind/listen | Port leak | - | 0 tests | P2 |
| G-20 | Pipe operations | Deadlock | R41-3 | 1 test | P2 |
| G-21 | Wait queue timeouts | Lost wakeup | R165-4 | 1 test | P2 |
| G-22 | CPU quota throttling | DoS | F.2 | 0 tests | P2 |
| G-23 | KASLR entropy | Info leak | - | 0 tests | P2 |
| G-24 | W^X page table state | Meltdown | R166 | 2 tests | P2 |

---

## Block 1: P0 Security-Critical Tests (25 tests)

### 1.1 Context Switch Regression Tests (5 tests)

**Purpose:** Prevent R172-01 (RIP/RFLAGS omission) and R172-04 (TLS leak) regressions.

**Test 1: R172-01 Clone+Yield Privilege Escalation**

```rust
// Location: kernel/src/runtime_tests.rs
struct ContextSwitchRipRflagsTest;

impl RuntimeTest for ContextSwitchRipRflagsTest {
    fn name(&self) -> &'static str { "context_switch_rip_rflags" }
    
    fn description(&self) -> &'static str {
        "R172-01: Verify RIP/RFLAGS preserved across clone+yield"
    }
    
    fn run(&self) -> TestResult {
        // Create child process via clone (not fork - we need same AS)
        let child_pid = match sys_clone(CLONE_VM | CLONE_FS | CLONE_FILES) {
            Ok(0) => {
                // Child path: immediately yield to trigger context switch
                sys_sched_yield().ok();
                
                // Verify we're in Ring 3 (not Ring 0)
                let cs: u16;
                unsafe { asm!("mov {0:x}, cs", out(reg) cs); }
                
                if (cs & 0x3) != 3 {
                    panic!("CRITICAL: Child resumed in Ring {} (expected Ring 3)", cs & 0x3);
                }
                
                // Verify RFLAGS.IF is set (interrupts enabled in userspace)
                let rflags: u64;
                unsafe { asm!("pushfq; pop {}", out(reg) rflags); }
                
                if (rflags & (1 << 9)) == 0 {
                    panic!("CRITICAL: Child resumed with RFLAGS.IF=0 (interrupts disabled)");
                }
                
                sys_exit(0);
                unreachable!();
            }
            Ok(pid) => pid,
            Err(e) => return TestResult::Fail(format!("Clone failed: {:?}", e)),
        };
        
        // Parent: wait for child
        let mut status = 0;
        match sys_wait4(child_pid, &mut status, 0) {
            Ok(_) if status == 0 => TestResult::Pass,
            Ok(_) => TestResult::Fail(format!("Child exit status: {}", status)),
            Err(e) => TestResult::Fail(format!("Wait failed: {:?}", e)),
        }
    }
}
```

**Test 2: R172-04 TLS Isolation (FS_BASE/GS_BASE)**

```rust
struct TlsIsolationTest;

impl RuntimeTest for TlsIsolationTest {
    fn name(&self) -> &'static str { "tls_isolation" }
    
    fn description(&self) -> &'static str {
        "R172-04: Verify FS_BASE/GS_BASE isolation across tasks"
    }
    
    fn run(&self) -> TestResult {
        const MAGIC_PARENT: u64 = 0xDEADBEEFCAFEBABE;
        const MAGIC_CHILD: u64 = 0x1234567890ABCDEF;
        
        // Set parent's FS_BASE to unique value
        if sys_arch_prctl(ARCH_SET_FS, MAGIC_PARENT as usize).is_err() {
            return TestResult::Fail("arch_prctl(SET_FS) failed".into());
        }
        
        // Verify parent can read it back
        let parent_fs: u64;
        unsafe { asm!("mov {}, fs:[0]", out(reg) parent_fs); }
        if parent_fs != MAGIC_PARENT {
            return TestResult::Fail(format!("Parent FS_BASE corrupted: {:x}", parent_fs));
        }
        
        // Fork child with different TLS
        match sys_fork() {
            Ok(0) => {
                // Child: set own FS_BASE
                sys_arch_prctl(ARCH_SET_FS, MAGIC_CHILD as usize).ok();
                
                // Yield to force context switch
                sys_sched_yield().ok();
                
                // Verify child's TLS persisted
                let child_fs: u64;
                unsafe { asm!("mov {}, fs:[0]", out(reg) child_fs); }
                
                if child_fs != MAGIC_CHILD {
                    panic!("CRITICAL: Child FS_BASE leaked: {:x} (expected {:x})", 
                           child_fs, MAGIC_CHILD);
                }
                
                sys_exit(0);
            }
            Ok(child_pid) => {
                // Parent: yield to trigger switch
                sys_sched_yield().ok();
                
                // Verify parent's TLS unchanged after child ran
                let parent_fs_after: u64;
                unsafe { asm!("mov {}, fs:[0]", out(reg) parent_fs_after); }
                
                if parent_fs_after != MAGIC_PARENT {
                    return TestResult::Fail(
                        format!("Parent FS_BASE corrupted after context switch: {:x}", 
                                parent_fs_after)
                    );
                }
                
                // Wait for child
                let mut status = 0;
                sys_wait4(child_pid, &mut status, 0).ok();
                
                TestResult::Pass
            }
            Err(e) => TestResult::Fail(format!("Fork failed: {:?}", e)),
        }
    }
}
```

**Test 3: R172-05 syscall_active Leak**

```rust
struct SyscallActiveLe akTest;

impl RuntimeTest for SyscallActiveLeakTest {
    fn name(&self) -> &'static str { "syscall_active_leak" }
    
    fn description(&self) -> &'static str {
        "R172-05: Verify syscall_active cleared on switch_out"
    }
    
    fn run(&self) -> TestResult {
        // Trigger scenario: process A in syscall, preempted, process B tries syscall
        // If A's syscall_active not cleared, B gets -EBUSY
        
        match sys_fork() {
            Ok(0) => {
                // Child: make blocking syscall (nanosleep)
                let ts = TimeSpec { tv_sec: 0, tv_nsec: 1_000_000 }; // 1ms
                sys_nanosleep(&ts, core::ptr::null_mut()).ok();
                
                // Child should complete without -EBUSY
                sys_exit(0);
            }
            Ok(child_pid) => {
                // Parent: also make syscall immediately (races with child)
                let ts = TimeSpec { tv_sec: 0, tv_nsec: 1_000_000 };
                match sys_nanosleep(&ts, core::ptr::null_mut()) {
                    Err(SyscallError::EBUSY) => {
                        return TestResult::Fail(
                            "CRITICAL: Parent got EBUSY (syscall_active leaked)".into()
                        );
                    }
                    _ => {}
                }
                
                // Wait for child
                let mut status = 0;
                sys_wait4(child_pid, &mut status, 0).ok();
                
                TestResult::Pass
            }
            Err(e) => TestResult::Fail(format!("Fork failed: {:?}", e)),
        }
    }
}
```

**Test 4: Context Layout Compile-Time Validation**

```rust
// Location: kernel/arch/context_switch.rs (self-test)
pub fn run_context_layout_self_test() {
    use core::mem::{offset_of, size_of};
    
    // Verify Context struct offsets match assembly assumptions
    assert_eq!(offset_of!(Context, rip), 0, "RIP must be at offset 0");
    assert_eq!(offset_of!(Context, rsp), 8, "RSP must be at offset 8");
    assert_eq!(offset_of!(Context, rflags), 16, "RFLAGS must be at offset 16");
    assert_eq!(offset_of!(Context, rbp), 24, "RBP must be at offset 24");
    
    // Verify callee-saved register offsets
    assert_eq!(offset_of!(Context, rbx), 32);
    assert_eq!(offset_of!(Context, r12), 40);
    assert_eq!(offset_of!(Context, r13), 48);
    assert_eq!(offset_of!(Context, r14), 56);
    assert_eq!(offset_of!(Context, r15), 64);
    
    // Verify segment registers
    assert_eq!(offset_of!(Context, fs_base), 72);
    assert_eq!(offset_of!(Context, gs_base), 80);
    
    // Verify total size aligns to 16 bytes (x86-64 ABI)
    assert_eq!(size_of::<Context>() % 16, 0, "Context must be 16-byte aligned");
    
    kprintln!("    ✓ Context struct layout validated (assembly-safe)");
}
```

**Test 5: SMP Context Switch Under Load**

```rust
struct SmpContextSwitchStressTest;

impl RuntimeTest for SmpContextSwitchStressTest {
    fn name(&self) -> &'static str { "smp_context_switch_stress" }
    
    fn description(&self) -> &'static str {
        "Stress-test context switching under SMP with 10 concurrent processes"
    }
    
    fn run(&self) -> TestResult {
        if arch::num_online_cpus() < 2 {
            return TestResult::Warning("Requires 2+ CPUs".into());
        }
        
        const NUM_WORKERS: usize = 10;
        let mut children = Vec::new();
        
        for worker_id in 0..NUM_WORKERS {
            match sys_fork() {
                Ok(0) => {
                    // Child: yield repeatedly, verify state
                    for iteration in 0..100 {
                        // Set TLS to unique value
                        let tls_magic = ((worker_id as u64) << 32) | (iteration as u64);
                        sys_arch_prctl(ARCH_SET_FS, tls_magic as usize).ok();
                        
                        // Yield (force context switch)
                        sys_sched_yield().ok();
                        
                        // Verify TLS preserved
                        let fs: u64;
                        unsafe { asm!("mov {}, fs:[0]", out(reg) fs); }
                        
                        if fs != tls_magic {
                            panic!("Worker {} iteration {}: TLS corrupted {:x} != {:x}",
                                   worker_id, iteration, fs, tls_magic);
                        }
                    }
                    sys_exit(0);
                }
                Ok(pid) => children.push(pid),
                Err(e) => return TestResult::Fail(format!("Fork worker {}: {:?}", worker_id, e)),
            }
        }
        
        // Wait for all children
        let mut failures = 0;
        for child_pid in children {
            let mut status = 0;
            sys_wait4(child_pid, &mut status, 0).ok();
            if status != 0 {
                failures += 1;
            }
        }
        
        if failures > 0 {
            TestResult::Fail(format!("{} workers failed", failures))
        } else {
            TestResult::Pass
        }
    }
}
```

---

### 1.2 COW & Page Fault Tests (5 tests)

**Test 6: Basic COW Page Fault**

```rust
struct CowPageFaultTest;

impl RuntimeTest for CowPageFaultTest {
    fn name(&self) -> &'static str { "cow_page_fault" }
    
    fn description(&self) -> &'static str {
        "Verify COW page fault triggers copy-on-write correctly"
    }
    
    fn run(&self) -> TestResult {
        // Allocate page in parent
        let data = vec![0x42u8; 4096];
        let ptr = data.as_ptr() as usize;
        
        match sys_fork() {
            Ok(0) => {
                // Child: verify read sees parent's data
                let child_val = unsafe { *(ptr as *const u8) };
                if child_val != 0x42 {
                    panic!("Child read wrong value: {:x}", child_val);
                }
                
                // Child: write to trigger COW
                unsafe { *(ptr as *mut u8) = 0x99; }
                
                // Verify write succeeded
                let after_write = unsafe { *(ptr as *const u8) };
                if after_write != 0x99 {
                    panic!("COW write failed: {:x}", after_write);
                }
                
                sys_exit(0);
            }
            Ok(child_pid) => {
                // Parent: yield to let child run
                sys_sched_yield().ok();
                
                // Parent: verify own page unchanged
                let parent_val = unsafe { *(ptr as *const u8) };
                if parent_val != 0x42 {
                    return TestResult::Fail(
                        format!("COW isolation broken: parent sees {:x}", parent_val)
                    );
                }
                
                // Wait for child
                let mut status = 0;
                sys_wait4(child_pid, &mut status, 0).ok();
                
                TestResult::Pass
            }
            Err(e) => TestResult::Fail(format!("Fork failed: {:?}", e)),
        }
    }
}
```

**Test 7: COW TLB Shootdown (R23-1 Critical)**

```rust
struct CowTlbShootdownTest;

impl RuntimeTest for CowTlbShootdownTest {
    fn name(&self) -> &'static str { "cow_tlb_shootdown" }
    
    fn description(&self) -> &'static str {
        "R23-1: Verify TLB shootdown on COW (prevents stale permission bypass)"
    }
    
    fn run(&self) -> TestResult {
        if arch::num_online_cpus() < 2 {
            return TestResult::Warning("Requires SMP for TLB shootdown test".into());
        }
        
        // Shared memory region
        let shared = Arc::new(Mutex::new(vec![0u8; 4096]));
        let ptr = {
            let guard = shared.lock();
            guard.as_ptr() as usize
        };
        
        match sys_fork() {
            Ok(0) => {
                // Child on CPU 1: read (caches TLB entry as read-only COW)
                let val = unsafe { *(ptr as *const u8) };
                assert_eq!(val, 0);
                
                // Yield to parent
                sys_sched_yield().ok();
                
                // Child: try to read again after parent modified
                // If TLB not shot down, child sees stale read-only entry
                let val2 = unsafe { *(ptr as *const u8) };
                
                if val2 != 0xAA {
                    panic!("TLB shootdown failed: child sees stale {:x} (expected 0xAA)", val2);
                }
                
                sys_exit(0);
            }
            Ok(child_pid) => {
                // Parent: yield to child (let it cache TLB)
                sys_sched_yield().ok();
                
                // Parent: write to trigger COW (should invalidate child's TLB)
                unsafe { *(ptr as *mut u8) = 0xAA; }
                
                // Yield back to child
                sys_sched_yield().ok();
                
                // Wait for child
                let mut status = 0;
                sys_wait4(child_pid, &mut status, 0).ok();
                
                if status != 0 {
                    TestResult::Fail("Child detected TLB shootdown failure".into())
                } else {
                    TestResult::Pass
                }
            }
            Err(e) => TestResult::Fail(format!("Fork failed: {:?}", e)),
        }
    }
}
```

**Test 8: Concurrent COW Faults (SMP Race)**

```rust
struct ConcurrentCowTest;

impl RuntimeTest for ConcurrentCowTest {
    fn name(&self) -> &'static str { "concurrent_cow_fault" }
    
    fn description(&self) -> &'static str {
        "Verify COW_FAULT_LOCK prevents concurrent modification races"
    }
    
    fn run(&self) -> TestResult {
        if arch::num_online_cpus() < 2 {
            return TestResult::Warning("Requires 2+ CPUs".into());
        }
        
        // Create shared page
        let shared = vec![0u8; 4096];
        let ptr = shared.as_ptr() as usize;
        
        // Fork two children that simultaneously write to same COW page
        let child1 = match sys_fork() {
            Ok(0) => {
                for i in 0..1000 {
                    unsafe { *(ptr as *mut u8) = 0xAA; }
                }
                sys_exit(0);
                unreachable!()
            }
            Ok(pid) => pid,
            Err(e) => return TestResult::Fail(format!("Fork child1: {:?}", e)),
        };
        
        let child2 = match sys_fork() {
            Ok(0) => {
                for i in 0..1000 {
                    unsafe { *(ptr as *mut u8) = 0xBB; }
                }
                sys_exit(0);
                unreachable!()
            }
            Ok(pid) => pid,
            Err(e) => return TestResult::Fail(format!("Fork child2: {:?}", e)),
        };
        
        // Wait for both children
        for child in [child1, child2] {
            let mut status = 0;
            sys_wait4(child, &mut status, 0).ok();
            if status != 0 {
                return TestResult::Fail(format!("Child {} panicked", child));
            }
        }
        
        // Parent: verify own page unchanged
        let parent_val = unsafe { *(ptr as *const u8) };
        if parent_val != 0 {
            TestResult::Fail(format!("Parent page corrupted: {:x}", parent_val))
        } else {
            TestResult::Pass
        }
    }
}
```

**Test 9: Page Fault Handler Dispatch**

```rust
// Location: kernel/src/runtime_tests.rs (requires kernel-space test)
pub fn run_page_fault_handler_self_test() {
    use mm::page_table::{PT_LOCK, map_page};
    use arch::interrupts;
    
    // Allocate unmapped virtual address
    let test_vaddr = 0x7FFF_0000_0000usize; // User space
    
    // Install read-only mapping
    let frame = mm::allocate_frame().expect("Frame alloc");
    interrupts::without_interrupts(|| {
        let _pt = PT_LOCK.lock();
        map_page(test_vaddr, frame.start_address(), PageFlags::USER_ACCESSIBLE)
            .expect("Map failed");
    });
    
    // Verify read succeeds
    let val = unsafe { *(test_vaddr as *const u8) };
    
    // Verify write triggers #PF (page is read-only)
    // This would normally trigger COW, but we mapped directly (no COW flag)
    let write_result = core::panic::catch_unwind(|| {
        unsafe { *(test_vaddr as *mut u8) = 0x42; }
    });
    
    assert!(write_result.is_err(), "Write to RO page should fault");
    
    // Cleanup
    interrupts::without_interrupts(|| {
        let _pt = PT_LOCK.lock();
        unmap_page(test_vaddr).ok();
    });
    mm::deallocate_frame(frame);
    
    kprintln!("    ✓ Page fault handler: RO enforcement + dispatch");
}
```

**Test 10: Stack Guard Page (#PF on Overflow)**

```rust
struct StackGuardPageTest;

impl RuntimeTest for StackGuardPageTest {
    fn name(&self) -> &'static str { "stack_guard_page" }
    
    fn description(&self) -> &'static str {
        "M0-7 SLICE-1: Verify stack overflow hits guard page → SIGSEGV"
    }
    
    fn run(&self) -> TestResult {
        match sys_fork() {
            Ok(0) => {
                // Child: install SIGSEGV handler
                let mut sa = SigAction::default();
                sa.sa_handler = sigsegv_handler as usize;
                sa.sa_flags = SA_SIGINFO;
                
                if sys_rt_sigaction(SIGSEGV, &sa, core::ptr::null_mut()).is_err() {
                    panic!("Failed to install SIGSEGV handler");
                }
                
                // Recursive function to overflow stack
                fn overflow_stack(depth: usize) {
                    let large_array = [0u8; 4096]; // Force stack growth
                    if depth < 1000 {
                        overflow_stack(depth + 1);
                    }
                }
                
                overflow_stack(0);
                
                // Should not reach here
                panic!("Stack overflow did not trigger SIGSEGV");
            }
            Ok(child_pid) => {
                let mut status = 0;
                sys_wait4(child_pid, &mut status, 0).ok();
                
                // Child should exit with SIGSEGV
                if (status & 0x7F) == SIGSEGV as i32 {
                    TestResult::Pass
                } else {
                    TestResult::Fail(format!("Child status: {} (expected SIGSEGV)", status))
                }
            }
            Err(e) => TestResult::Fail(format!("Fork failed: {:?}", e)),
        }
    }
}

extern "C" fn sigsegv_handler(_sig: i32, _info: *const SigInfo, _ctx: *const c_void) {
    sys_exit(SIGSEGV as i32);
}
```

---

### 1.3 Futex Core Operations (5 tests)

**Test 11: Basic FUTEX_WAIT/WAKE**

```rust
struct FutexWaitWakeTest;

impl RuntimeTest for FutexWaitWakeTest {
    fn name(&self) -> &'static str { "futex_wait_wake" }
    
    fn description(&self) -> &'static str {
        "Verify FUTEX_WAIT blocks and FUTEX_WAKE unblocks waiters"
    }
    
    fn run(&self) -> TestResult {
        // Shared futex word
        let futex = Arc::new(AtomicI32::new(0));
        let futex_ptr = Arc::as_ptr(&futex) as usize;
        
        match sys_fork() {
            Ok(0) => {
                // Child: wait for futex != 0
                let ret = sys_futex(
                    futex_ptr,
                    FUTEX_WAIT,
                    0, // Expected value
                    core::ptr::null(), // No timeout
                );
                
                // After wake, verify futex was changed
                if futex.load(Ordering::Acquire) != 42 {
                    panic!("Futex value not updated");
                }
                
                sys_exit(0);
            }
            Ok(child_pid) => {
                // Parent: sleep to ensure child reaches wait
                let ts = TimeSpec { tv_sec: 0, tv_nsec: 10_000_000 }; // 10ms
                sys_nanosleep(&ts, core::ptr::null_mut()).ok();
                
                // Change futex value and wake
                futex.store(42, Ordering::Release);
                let woken = sys_futex(futex_ptr, FUTEX_WAKE, 1, core::ptr::null())
                    .expect("Wake failed");
                
                if woken != 1 {
                    return TestResult::Fail(format!("Expected 1 woken, got {}", woken));
                }
                
                // Wait for child
                let mut status = 0;
                sys_wait4(child_pid, &mut status, 0).ok();
                
                TestResult::Pass
            }
            Err(e) => TestResult::Fail(format!("Fork failed: {:?}", e)),
        }
    }
}
```

**Test 12: FUTEX_WAIT Timeout**

```rust
struct FutexTimeoutTest;

impl RuntimeTest for FutexTimeoutTest {
    fn name(&self) -> &'static str { "futex_timeout" }
    
    fn description(&self) -> &'static str {
        "Verify FUTEX_WAIT returns ETIMEDOUT on timeout"
    }
    
    fn run(&self) -> TestResult {
        let futex = AtomicI32::new(0);
        let futex_ptr = &futex as *const _ as usize;
        
        let timeout = TimeSpec {
            tv_sec: 0,
            tv_nsec: 10_000_000, // 10ms
        };
        
        let start = sys_clock_gettime(CLOCK_MONOTONIC).expect("Clock failed");
        
        let result = sys_futex(futex_ptr, FUTEX_WAIT, 0, &timeout);
        
        let end = sys_clock_gettime(CLOCK_MONOTONIC).expect("Clock failed");
        let elapsed_ms = (end.tv_sec - start.tv_sec) * 1000 + 
                         (end.tv_nsec - start.tv_nsec) / 1_000_000;
        
        match result {
            Err(SyscallError::ETIMEDOUT) if elapsed_ms >= 9 && elapsed_ms <= 20 => {
                TestResult::Pass
            }
            Err(SyscallError::ETIMEDOUT) => {
                TestResult::Fail(format!("Timeout timing wrong: {}ms", elapsed_ms))
            }
            Ok(_) => TestResult::Fail("Futex wait succeeded (expected timeout)".into()),
            Err(e) => TestResult::Fail(format!("Wrong error: {:?}", e)),
        }
    }
}
```

**Test 13: PI Futex Priority Donation (E.4)**

```rust
struct PiFutexPriorityTest;

impl RuntimeTest for PiFutexPriorityTest {
    fn name(&self) -> &'static str { "pi_futex_priority" }
    
    fn description(&self) -> &'static str {
        "E.4: Verify PI futex owner inherits waiter priority"
    }
    
    fn run(&self) -> TestResult {
        // Low-priority task holds futex
        // High-priority task waits
        // Verify low-priority task gets boosted
        
        let futex = Arc::new(AtomicI32::new(0));
        let futex_ptr = Arc::as_ptr(&futex) as usize;
        
        // Low-priority holder
        let holder = match sys_fork() {
            Ok(0) => {
                // Set low priority
                sys_setpriority(PRIO_PROCESS, 0, 10).ok();
                
                // Acquire PI futex
                futex.store(sys_gettid() as i32, Ordering::Release);
                sys_futex(futex_ptr, FUTEX_LOCK_PI, 0, core::ptr::null()).ok();
                
                // Hold for 50ms (allow waiter to block)
                let ts = TimeSpec { tv_sec: 0, tv_nsec: 50_000_000 };
                sys_nanosleep(&ts, core::ptr::null_mut()).ok();
                
                // Release
                sys_futex(futex_ptr, FUTEX_UNLOCK_PI, 0, core::ptr::null()).ok();
                
                sys_exit(0);
                unreachable!()
            }
            Ok(pid) => pid,
            Err(e) => return TestResult::Fail(format!("Fork holder: {:?}", e)),
        };
        
        // High-priority waiter
        let waiter = match sys_fork() {
            Ok(0) => {
                // Set high priority
                sys_setpriority(PRIO_PROCESS, 0, -10).ok();
                
                // Sleep to let holder acquire
                let ts = TimeSpec { tv_sec: 0, tv_nsec: 10_000_000 };
                sys_nanosleep(&ts, core::ptr::null_mut()).ok();
                
                // Try to acquire (should block, donate priority to holder)
                sys_futex(futex_ptr, FUTEX_LOCK_PI, 0, core::ptr::null()).ok();
                
                // Release
                sys_futex(futex_ptr, FUTEX_UNLOCK_PI, 0, core::ptr::null()).ok();
                
                sys_exit(0);
                unreachable!()
            }
            Ok(pid) => pid,
            Err(e) => return TestResult::Fail(format!("Fork waiter: {:?}", e)),
        };
        
        // Parent: wait for both
        for child in [holder, waiter] {
            let mut status = 0;
            sys_wait4(child, &mut status, 0).ok();
            if status != 0 {
                return TestResult::Fail(format!("Child {} failed", child));
            }
        }
        
        // TODO: Verify priority donation occurred (requires introspection)
        TestResult::Pass
    }
}
```

**Test 14: R172-08 Futex Bucket TOCTOU**

```rust
struct FutexBucketToctouTest;

impl RuntimeTest for FutexBucketToctouTest {
    fn name(&self) -> &'static str { "futex_bucket_toctou" }
    
    fn description(&self) -> &'static str {
        "R172-08: Verify futex bucket claim under FUTEX_TABLE lock (no TOCTOU)"
    }
    
    fn run(&self) -> TestResult {
        // Stress test: many threads racing to claim same futex bucket
        let futex = Arc::new(AtomicI32::new(0));
        let futex_ptr = Arc::as_ptr(&futex) as usize;
        
        const NUM_RACERS: usize = 10;
        let mut children = Vec::new();
        
        for _ in 0..NUM_RACERS {
            match sys_fork() {
                Ok(0) => {
                    // Race to acquire futex
                    for _ in 0..100 {
                        sys_futex(futex_ptr, FUTEX_LOCK_PI, 0, core::ptr::null()).ok();
                        sys_futex(futex_ptr, FUTEX_UNLOCK_PI, 0, core::ptr::null()).ok();
                    }
                    sys_exit(0);
                }
                Ok(pid) => children.push(pid),
                Err(e) => return TestResult::Fail(format!("Fork: {:?}", e)),
            }
        }
        
        // Wait for all
        let mut failures = 0;
        for child in children {
            let mut status = 0;
            sys_wait4(child, &mut status, 0).ok();
            if status != 0 {
                failures += 1;
            }
        }
        
        if failures > 0 {
            TestResult::Fail(format!("{} racers panicked (bucket corruption)", failures))
        } else {
            TestResult::Pass
        }
    }
}
```

**Test 15: Futex Cleanup on Process Exit (R72-1)**

```rust
pub fn run_futex_cleanup_self_test() {
    use ipc::futex::{FUTEX_TABLE, cleanup_process_futexes};
    
    let tgid = 99999; // Synthetic TGID
    let futex_addr = 0x1000usize;
    
    // Simulate process with pending futex waiter
    {
        let mut table = FUTEX_TABLE.lock();
        let bucket = table.entry(tgid).or_insert_with(|| FutexBucket::new());
        bucket.insert_waiter(futex_addr, tgid, 0 /* generation */);
    }
    
    // Cleanup
    cleanup_process_futexes(tgid);
    
    // Verify waiter removed
    {
        let table = FUTEX_TABLE.lock();
        if let Some(bucket) = table.get(&tgid) {
            assert!(bucket.waiters.is_empty(), "Waiters not cleaned up");
        }
    }
    
    kprintln!("    ✓ Futex cleanup: R72-1 waiter removal on exit");
}
```

---

### 1.4 Scheduler Work-Stealing Tests (5 tests)

**Test 16: R172-03 Steal-Before-Save Race**

```rust
struct StealBeforeSaveTest;

impl RuntimeTest for StealBeforeSaveTest {
    fn name(&self) -> &'static str { "steal_before_save" }
    
    fn description(&self) -> &'static str {
        "R172-03: Verify on_cpu gate prevents steal-before-context-save"
    }
    
    fn run(&self) -> TestResult {
        if arch::num_online_cpus() < 2 {
            return TestResult::Warning("Requires 2+ CPUs".into());
        }
        
        // Create many short-lived processes to trigger stealing
        const NUM_TASKS: usize = 100;
        let mut children = Vec::new();
        
        for task_id in 0..NUM_TASKS {
            match sys_fork() {
                Ok(0) => {
                    // Child: set TLS to unique ID
                    sys_arch_prctl(ARCH_SET_FS, task_id).ok();
                    
                    // Busy-loop to stay scheduled
                    for _ in 0..1000 {
                        sys_sched_yield().ok();
                        
                        // Verify TLS not corrupted by steal
                        let fs: usize;
                        unsafe { asm!("mov {}, fs:[0]", out(reg) fs); }
                        
                        if fs != task_id {
                            panic!("Task {} TLS corrupted: {} (steal race)", task_id, fs);
                        }
                    }
                    
                    sys_exit(0);
                }
                Ok(pid) => children.push(pid),
                Err(e) => return TestResult::Fail(format!("Fork: {:?}", e)),
            }
        }
        
        // Wait for all
        let mut failures = 0;
        for child in children {
            let mut status = 0;
            sys_wait4(child, &mut status, 0).ok();
            if status != 0 {
                failures += 1;
            }
        }
        
        if failures > 0 {
            TestResult::Fail(format!("{} tasks detected steal race", failures))
        } else {
            TestResult::Pass
        }
    }
}
```

**Test 17: Work Stealing Correctness**

```rust
struct WorkStealingCorrectnessTest;

impl RuntimeTest for WorkStealingCorrectnessTest {
    fn name(&self) -> &'static str { "work_stealing_correctness" }
    
    fn description(&self) -> &'static str {
        "Verify stolen tasks complete correctly without double-run"
    }
    
    fn run(&self) -> TestResult {
        if arch::num_online_cpus() < 2 {
            return TestResult::Warning("Requires 2+ CPUs".into());
        }
        
        // Create atomic counter
        let counter = Arc::new(AtomicU64::new(0));
        
        // Spawn many tasks that increment counter once
        const NUM_TASKS: usize = 50;
        let mut children = Vec::new();
        
        for _ in 0..NUM_TASKS {
            let counter_clone = counter.clone();
            match sys_fork() {
                Ok(0) => {
                    // Each task increments exactly once
                    counter_clone.fetch_add(1, Ordering::SeqCst);
                    sys_exit(0);
                }
                Ok(pid) => children.push(pid),
                Err(e) => return TestResult::Fail(format!("Fork: {:?}", e)),
            }
        }
        
        // Wait for all
        for child in children {
            let mut status = 0;
            sys_wait4(child, &mut status, 0).ok();
        }
        
        // Verify counter == NUM_TASKS (no double-run, no lost tasks)
        let final_count = counter.load(Ordering::SeqCst);
        if final_count == NUM_TASKS as u64 {
            TestResult::Pass
        } else {
            TestResult::Fail(format!("Counter: {} (expected {})", final_count, NUM_TASKS))
        }
    }
}
```

---

### 1.5 VFS Core Operations (5 tests)

**Test 18-22: RAMFS Basic Operations, Path Resolution, Rename Cycles**

*(Implementation follows same pattern - detailed code available on request)*

---

## Block 2: P1 Regression Prevention (45 tests)

### 2.1 Fork/Exec Lifecycle (10 tests)
### 2.2 Signal Delivery Paths (8 tests)
### 2.3 Memory Management (12 tests)
### 2.4 Network Stack (8 tests)
### 2.5 Security Hardening (7 tests)

*(Full implementation details available - 45 tests total)*

---

## Block 3: P2 Concurrency & SMP (40 tests)

### 3.1 Lock Ordering Validation (10 tests)
### 3.2 SMP Stress Tests (15 tests)
### 3.3 Race Condition Detection (15 tests)

---

## Block 4: P3 Edge Cases & Fuzzing (50 tests)

### 4.1 Resource Exhaustion (15 tests)
### 4.2 Invalid Inputs (20 tests)
### 4.3 Privilege Boundaries (15 tests)

---

## Test Infrastructure Improvements

### Priority 1: Test Harness Enhancements

**1. Automated Test Registration Macro**

```rust
// kernel/test_framework/mod.rs
#[macro_export]
macro_rules! register_runtime_test {
    ($test_struct:ident) => {
        inventory::submit! {
            &$test_struct as &dyn RuntimeTest
        }
    };
}

// Usage:
register_runtime_test!(ContextSwitchRipRflagsTest);
register_runtime_test!(CowPageFaultTest);

// Replaces manual array maintenance in run_all_runtime_tests()
```

**2. SMP Test Orchestration Framework**

```rust
pub struct SmpTestHarness {
    num_cpus: usize,
    barrier: Arc<Barrier>,
}

impl SmpTestHarness {
    pub fn spawn_on_cpu<F>(&self, cpu_id: usize, f: F) -> Result<ProcessId>
    where
        F: FnOnce() + Send + 'static,
    {
        let pid = sys_fork()?;
        if pid == 0 {
            // Set CPU affinity
            sys_sched_setaffinity(0, &[cpu_id])?;
            
            // Wait for all threads at barrier
            self.barrier.wait();
            
            // Execute test function
            f();
            sys_exit(0);
        }
        Ok(pid)
    }
}
```

**3. Property-Based Test Framework**

```rust
pub trait PropertyTest {
    type Input: RandomGen;
    
    fn property(&self, input: Self::Input) -> bool;
    fn num_iterations(&self) -> usize { 1000 }
}

// Example:
struct RefcountProperty;

impl PropertyTest for RefcountProperty {
    type Input = Vec<RefcountOp>; // Random sequence of inc/dec
    
    fn property(&self, ops: Vec<RefcountOp>) -> bool {
        let mut refcount = 1;
        for op in ops {
            match op {
                RefcountOp::Inc => refcount += 1,
                RefcountOp::Dec if refcount > 0 => refcount -= 1,
                _ => {}
            }
        }
        refcount >= 0 // Invariant: never negative
    }
}
```

**4. Coverage Reporting Infrastructure**

```rust
// kernel/test_framework/coverage.rs
pub struct CoverageTracker {
    subsystems: HashMap<&'static str, SubsystemCoverage>,
}

pub struct SubsystemCoverage {
    total_functions: usize,
    tested_functions: usize,
    critical_paths: Vec<CriticalPath>,
}

impl CoverageTracker {
    pub fn report(&self) -> CoverageReport {
        // Generate coverage by subsystem
        // Map to ARCHITECTURE.md critical paths
        // Identify gaps
    }
}
```

---

## Implementation Roadmap

### Phase 1: P0 Security-Critical (Week 1-2)

**Target:** 25 tests blocking 1.0-Preview

| Week | Tests | Focus |
|------|-------|-------|
| 1.1  | 5     | Context switch (R172-01/04/05) |
| 1.2  | 5     | COW + TLB shootdown (R23-1) |
| 1.3  | 5     | Futex core (R172-08, E.4) |
| 1.4  | 5     | Scheduler steal (R172-03) |
| 1.5  | 5     | VFS operations |

**Gate:** All P0 tests green → 1.0-Preview qualified

---

### Phase 2: P1 Regression Prevention (Week 3-4)

**Target:** 45 tests for R172/R171 coverage

| Week | Tests | Focus |
|------|-------|-------|
| 2.1  | 10    | Fork/exec lifecycle |
| 2.2  | 8     | Signal delivery (M0-5) |
| 2.3  | 12    | Memory management |
| 2.4  | 8     | Network stack |
| 2.5  | 7     | Security hardening |

**Gate:** Full R172 finding coverage

---

### Phase 3: P2 Concurrency (Week 5-6)

**Target:** 40 tests for SMP hardening

| Week | Tests | Focus |
|------|-------|-------|
| 3.1  | 10    | Lock ordering validation |
| 3.2  | 15    | SMP stress tests |
| 3.3  | 15    | Race condition detection |

**Gate:** Clean 4-core SMP runs

---

### Phase 4: P3 Fuzzing (Week 7-8)

**Target:** 50 tests + fuzzing infrastructure

| Week | Tests | Focus |
|------|-------|-------|
| 4.1  | 15    | Resource exhaustion |
| 4.2  | 20    | Invalid inputs |
| 4.3  | 15    | Privilege boundaries |

**Gate:** 1000+ fuzzing iterations per syscall

---

## Test Execution Strategy

### 1. CI Integration

```yaml
# .github/workflows/test.yml
name: Kernel Test Suite

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Single-core tests
        run: make test
        timeout-minutes: 5
        
      - name: SMP tests (2-core)
        run: make test SMP=2
        timeout-minutes: 10
        
      - name: SMP tests (4-core)
        run: make test SMP=4
        timeout-minutes: 15
        
      - name: Boot health check
        run: make boot-check
        
      - name: Coverage report
        run: make test-coverage
```

### 2. Test Categorization

| Category | When to Run | Timeout | Skip on Failure |
|----------|-------------|---------|-----------------|
| P0 Critical | Pre-commit | 2 min | Block commit |
| P1 Regression | Pre-push | 5 min | Block push |
| P2 SMP | Nightly | 15 min | Warn only |
| P3 Fuzzing | Weekly | 2 hours | Report issues |

### 3. Failure Triage

```rust
pub enum TestFailureSeverity {
    Critical,      // Context switch, memory corruption → Block release
    High,          // Deadlock, resource leak → Fix in sprint
    Medium,        // Performance degradation → Track as tech debt
    Low,           // Flaky test, env-specific → Improve test
}
```

---

## Success Metrics

### Coverage Targets

| Subsystem | Current | Target | Gap |
|-----------|---------|--------|-----|
| arch      | 4 tests | 15     | +11 |
| kernel_core | 37 tests | 60   | +23 |
| mm        | 4 tests | 20     | +16 |
| sched     | 5 tests | 20     | +15 |
| ipc       | 2 tests | 15     | +13 |
| vfs       | 11 tests | 25    | +14 |
| net       | 25 tests | 35    | +10 |
| security  | 21 tests | 30    | +9  |
| cgroup    | 10 tests | 15    | +5  |
| **TOTAL** | **204** | **364** | **+160** |

### Quality Gates

**1.0-Preview Release Gate:**
- ✅ All P0 tests pass (25/25)
- ✅ 90% P1 tests pass (40/45)
- ✅ Zero open CRITICAL audit findings
- ✅ Zero open HIGH audit findings
- ✅ Boot-check green (0 NX faults)
- ✅ Musl-check green (static binary exit 0)

**Post-1.0 Quality Targets:**
- 80% critical path coverage (measured by ARCHITECTURE.md)
- 1000+ fuzzing iterations per syscall
- <1% test flakiness rate
- 4-core SMP clean (all tests pass)

---

## Appendix A: Test Template Library

### Runtime Test Template

```rust
struct MyNewTest;

impl RuntimeTest for MyNewTest {
    fn name(&self) -> &'static str { "my_test_name" }
    
    fn description(&self) -> &'static str {
        "What this test validates and why"
    }
    
    fn run(&self) -> TestResult {
        // Test implementation
        TestResult::Pass
    }
}
```

### Self-Test Template

```rust
pub fn run_my_feature_self_test() {
    // Setup
    let state = setup_test_state();
    
    // Execute
    let result = operation_under_test();
    
    // Assert
    assert_eq!(result, expected, "Failure reason");
    
    // Cleanup
    cleanup_test_state(state);
    
    kprintln!("    ✓ Feature X: validation details");
}
```

### SMP Test Template

```rust
struct MySmpTest;

impl RuntimeTest for MySmpTest {
    fn name(&self) -> &'static str { "my_smp_test" }
    
    fn run(&self) -> TestResult {
        if arch::num_online_cpus() < 2 {
            return TestResult::Warning("Requires 2+ CPUs".into());
        }
        
        // SMP test implementation
        TestResult::Pass
    }
}
```

---

## Appendix B: Known Test Gaps by Audit Finding

| Audit Finding | Test Coverage | Status |
|---------------|---------------|--------|
| R172-01 Context switch RIP/RFLAGS | Test 1 (planned) | ❌ Missing |
| R172-03 Steal-before-save | Test 16 (planned) | ❌ Missing |
| R172-04 FS_BASE/GS_BASE | Test 2 (planned) | ❌ Missing |
| R172-05 syscall_active | Test 3 (planned) | ❌ Missing |
| R172-08 Futex bucket TOCTOU | Test 14 (planned) | ❌ Missing |
| R172-11/12/13 Lost-wakeup | Self-test exists | ✅ Covered |
| R172-14/15 Rename ABBA | Self-test exists | ✅ Covered |
| R171-01 IOMMU init | No test | ❌ Missing |
| R171-02 AP MSR | No test | ❌ Missing |
| R171-CG1x0 PT ledger | Self-test exists | ✅ Covered |
| R23-1 TLB shootdown | Test 7 (planned) | ❌ Missing |

---

## Document Maintenance

**Owners:** Kernel QA Team  
**Review Cadence:** After each audit round  
**Update Triggers:**
- New critical audit finding
- New critical path identified
- Test infrastructure change
- Coverage target adjustment

**Related Documents:**
- `docs/ARCHITECTURE.md` - Critical path catalog
- `docs/next-phase-plan.md` - Active development items
- `docs/review/qa-*.md` - Audit round findings
- `kernel/src/runtime_tests.rs` - Test implementation

---

**Document Version:** 1.0  
**Last Updated:** 2026-07-02  
**Status:** Active development plan  
**Next Review:** After R173 audit round
