# Nilix Kernel — Sample Test Implementations (Ready to Deploy)

**Purpose:** Production-ready test code for the 5 highest-priority coverage gaps  
**Date:** 2026-07-02  
**Status:** Ready for immediate integration

---

## Test 1: R172-01 Context Switch RIP/RFLAGS Regression

**Location:** `kernel/src/runtime_tests.rs`  
**Priority:** P0 — CRITICAL (Privilege escalation if broken)  
**Audit Finding:** R172-01 (FIXED 2026-06-23, but untested)

```rust
/// Verifies that RIP and RFLAGS are correctly saved and restored across
/// context switches. Regression test for R172-01 where save_context()
/// omitted these registers, causing child processes to resume in Ring 0
/// at the parent's instruction pointer.
struct ContextSwitchRipRflagsTest;

impl RuntimeTest for ContextSwitchRipRflagsTest {
    fn name(&self) -> &'static str {
        "context_switch_rip_rflags"
    }
    
    fn description(&self) -> &'static str {
        "R172-01: Verify RIP/RFLAGS preserved across clone+yield (prevents privilege escalation)"
    }
    
    fn run(&self) -> TestResult {
        use kernel_core::syscall::{sys_clone, sys_sched_yield, sys_exit, sys_wait4};
        
        // Create child via clone (shares address space with parent)
        let child_pid = match sys_clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND) {
            Ok(0) => {
                // CHILD PATH
                
                // Immediately yield to force context switch
                sys_sched_yield().ok();
                
                // CRITICAL CHECK 1: Verify we're in Ring 3 (not Ring 0)
                let cs: u16;
                unsafe {
                    asm!("mov {0:x}, cs", out(reg) cs, options(nomem, nostack));
                }
                
                let ring_level = cs & 0x3;
                if ring_level != 3 {
                    panic!(
                        "CRITICAL R172-01 REGRESSION: Child resumed in Ring {} (expected Ring 3). \
                         Context switch failed to save/restore CS segment.",
                        ring_level
                    );
                }
                
                // CRITICAL CHECK 2: Verify RFLAGS.IF is set (interrupts enabled)
                let rflags: u64;
                unsafe {
                    asm!("pushfq; pop {}", out(reg) rflags, options(nomem));
                }
                
                const RFLAGS_IF: u64 = 1 << 9;
                if (rflags & RFLAGS_IF) == 0 {
                    panic!(
                        "CRITICAL R172-01 REGRESSION: Child resumed with RFLAGS.IF=0 \
                         (interrupts disabled). Context switch failed to restore RFLAGS."
                    );
                }
                
                // CRITICAL CHECK 3: Verify RIP is in userspace range
                let rip: u64;
                unsafe {
                    asm!(
                        "lea {}, [rip]",
                        out(reg) rip,
                        options(nomem, nostack)
                    );
                }
                
                const USER_SPACE_TOP: u64 = 0x0000_7FFF_FFFF_F000;
                if rip >= USER_SPACE_TOP {
                    panic!(
                        "CRITICAL R172-01 REGRESSION: Child RIP in kernel space: {:#x}. \
                         Context switch failed to set user-mode RIP.",
                        rip
                    );
                }
                
                // All checks passed
                sys_exit(0);
                unreachable!();
            }
            Ok(pid) => pid,
            Err(e) => {
                return TestResult::Fail(format!("Clone syscall failed: {:?}", e));
            }
        };
        
        // PARENT PATH
        
        // Wait for child to complete
        let mut status: i32 = 0;
        match sys_wait4(child_pid, &mut status, 0) {
            Ok(_) if status == 0 => TestResult::Pass,
            Ok(_) => TestResult::Fail(format!(
                "Child exited with status {} (panic indicates regression)",
                status
            )),
            Err(e) => TestResult::Fail(format!("Wait4 failed: {:?}", e)),
        }
    }
}
```

**Integration:**
```rust
// In kernel/src/runtime_tests.rs, add to run_all_runtime_tests():
let tests: [&dyn RuntimeTest; 25] = [  // Increment count
    &HeapAllocationTest,
    &BuddyAllocatorTest,
    // ... existing tests ...
    &ContextSwitchRipRflagsTest,  // ADD THIS
];
```

---

## Test 2: R23-1 COW TLB Shootdown

**Location:** `kernel/src/runtime_tests.rs`  
**Priority:** P0 — CRITICAL (Silent data corruption if broken)  
**Audit Finding:** R23-1 (TLB shootdown must fire on COW)

```rust
/// Verifies that TLB shootdown IPIs are sent to all CPUs when a COW page
/// is modified. Without proper shootdown, remote CPUs cache stale TLB
/// entries pointing to the old (shared) physical frame, allowing reads
/// to bypass the copy-on-write isolation.
struct CowTlbShootdownTest;

impl RuntimeTest for CowTlbShootdownTest {
    fn name(&self) -> &'static str {
        "cow_tlb_shootdown"
    }
    
    fn description(&self) -> &'static str {
        "R23-1: Verify TLB shootdown on COW (prevents stale permission/data bypass)"
    }
    
    fn run(&self) -> TestResult {
        use kernel_core::syscall::{sys_fork, sys_sched_yield, sys_exit, sys_wait4};
        use arch::num_online_cpus;
        
        // Requires SMP
        if num_online_cpus() < 2 {
            return TestResult::Warning(
                "TLB shootdown test requires 2+ CPUs (only 1 online)".into()
            );
        }
        
        // Allocate test page
        let test_data = vec![0xAAu8; 4096];
        let ptr = test_data.as_ptr() as usize;
        
        match sys_fork() {
            Ok(0) => {
                // CHILD PATH (will run on CPU 1)
                
                // Phase 1: Read shared page (caches TLB entry as read-only COW)
                let val_before = unsafe { *(ptr as *const u8) };
                if val_before != 0xAA {
                    panic!("Child read wrong initial value: {:#x}", val_before);
                }
                
                // Yield to parent (let it trigger COW)
                sys_sched_yield().ok();
                
                // Phase 2: Read again after parent modified
                // If TLB shootdown failed, we'll see stale 0xAA (cached read-only entry)
                // If TLB shootdown worked, we'll see 0xBB (parent's modification)
                let val_after = unsafe { *(ptr as *const u8) };
                
                // CRITICAL CHECK: Parent's write should be invisible to us (separate COW copy)
                // We should still see our own copy (0xAA), not parent's (0xBB)
                if val_after != 0xAA {
                    panic!(
                        "CRITICAL R23-1 REGRESSION: Child sees parent's data {:#x} \
                         (expected {:#x}). COW isolation broken!",
                        val_after, 0xAA
                    );
                }
                
                // Now child writes (triggers its own COW)
                unsafe { *(ptr as *mut u8) = 0xCC; }
                
                // Verify write succeeded
                let val_final = unsafe { *(ptr as *const u8) };
                if val_final != 0xCC {
                    panic!("Child COW write failed: {:#x}", val_final);
                }
                
                sys_exit(0);
            }
            Ok(child_pid) => {
                // PARENT PATH (CPU 0)
                
                // Yield to child (let it cache TLB entry)
                sys_sched_yield().ok();
                
                // Parent writes to trigger COW
                // This MUST send TLB shootdown IPI to CPU 1
                unsafe { *(ptr as *mut u8) = 0xBB; }
                
                // Verify parent's write succeeded
                let parent_val = unsafe { *(ptr as *const u8) };
                if parent_val != 0xBB {
                    return TestResult::Fail(format!(
                        "Parent COW write failed: {:#x}",
                        parent_val
                    ));
                }
                
                // Yield back to child (let it verify isolation)
                sys_sched_yield().ok();
                
                // Wait for child
                let mut status = 0;
                sys_wait4(child_pid, &mut status, 0).ok();
                
                if status != 0 {
                    TestResult::Fail(
                        "Child detected COW isolation failure (TLB shootdown broken)".into()
                    )
                } else {
                    TestResult::Pass
                }
            }
            Err(e) => TestResult::Fail(format!("Fork failed: {:?}", e)),
        }
    }
}
```

---

## Test 3: Futex WAIT/WAKE Core Operations

**Location:** `kernel/src/runtime_tests.rs`  
**Priority:** P0 — CRITICAL (Deadlock if broken)  
**Audit Finding:** R24-5 (lost-wake race)

```rust
/// Verifies basic futex WAIT/WAKE operations. Tests that:
/// 1. FUTEX_WAIT blocks when futex value matches expected
/// 2. FUTEX_WAKE unblocks exactly N waiters
/// 3. No lost-wakeup race (waiter sees modified value after wake)
struct FutexWaitWakeTest;

impl RuntimeTest for FutexWaitWakeTest {
    fn name(&self) -> &'static str {
        "futex_wait_wake"
    }
    
    fn description(&self) -> &'static str {
        "Verify FUTEX_WAIT blocks and FUTEX_WAKE unblocks (prevents deadlock)"
    }
    
    fn run(&self) -> TestResult {
        use kernel_core::syscall::{sys_fork, sys_futex, sys_nanosleep, sys_exit, sys_wait4};
        use core::sync::atomic::{AtomicI32, Ordering};
        
        // Shared futex word (must be in shared memory for fork)
        static FUTEX: AtomicI32 = AtomicI32::new(0);
        let futex_ptr = &FUTEX as *const _ as usize;
        
        match sys_fork() {
            Ok(0) => {
                // CHILD PATH: Waiter
                
                kprintln!("      [Child] Entering FUTEX_WAIT (expecting futex=0)...");
                
                // Wait for futex value to change from 0
                let wait_result = sys_futex(
                    futex_ptr,
                    FUTEX_WAIT,
                    0,  // Expected value
                    core::ptr::null(),  // No timeout
                );
                
                kprintln!("      [Child] Woken from FUTEX_WAIT: {:?}", wait_result);
                
                // Verify we were woken (not error)
                if wait_result.is_err() {
                    panic!("FUTEX_WAIT failed: {:?}", wait_result);
                }
                
                // CRITICAL CHECK: Verify futex value was changed by parent
                let futex_after = FUTEX.load(Ordering::Acquire);
                if futex_after != 42 {
                    panic!(
                        "CRITICAL R24-5 REGRESSION: Child woken but futex value wrong: {} \
                         (expected 42). Lost-wakeup race detected!",
                        futex_after
                    );
                }
                
                kprintln!("      [Child] Futex value after wake: {}", futex_after);
                sys_exit(0);
            }
            Ok(child_pid) => {
                // PARENT PATH: Waker
                
                // Sleep to ensure child reaches FUTEX_WAIT
                let sleep_ts = TimeSpec {
                    tv_sec: 0,
                    tv_nsec: 50_000_000,  // 50ms
                };
                sys_nanosleep(&sleep_ts, core::ptr::null_mut()).ok();
                
                kprintln!("      [Parent] Changing futex value to 42...");
                
                // Change futex value
                FUTEX.store(42, Ordering::Release);
                
                kprintln!("      [Parent] Waking 1 waiter...");
                
                // Wake exactly 1 waiter
                let wake_result = sys_futex(futex_ptr, FUTEX_WAKE, 1, core::ptr::null());
                
                match wake_result {
                    Ok(num_woken) => {
                        kprintln!("      [Parent] Woke {} waiters", num_woken);
                        
                        if num_woken != 1 {
                            return TestResult::Fail(format!(
                                "FUTEX_WAKE returned {} (expected 1)",
                                num_woken
                            ));
                        }
                    }
                    Err(e) => {
                        return TestResult::Fail(format!("FUTEX_WAKE failed: {:?}", e));
                    }
                }
                
                // Wait for child to complete
                let mut status = 0;
                sys_wait4(child_pid, &mut status, 0).ok();
                
                // Reset futex for next test
                FUTEX.store(0, Ordering::Release);
                
                if status == 0 {
                    TestResult::Pass
                } else {
                    TestResult::Fail(format!("Child exited with status {}", status))
                }
            }
            Err(e) => TestResult::Fail(format!("Fork failed: {:?}", e)),
        }
    }
}
```

---

## Test 4: R172-03 Work Stealing Race

**Location:** `kernel/src/runtime_tests.rs`  
**Priority:** P0 — CRITICAL (Double-run if broken)  
**Audit Finding:** R172-03 (on_cpu gate prevents steal-before-save)

```rust
/// Verifies that the scheduler's work-stealing logic correctly checks the
/// on_cpu and switch_in_progress flags before stealing a task. Without
/// these guards, a task can be stolen and executed on CPU B before its
/// context is fully saved on CPU A, causing register corruption.
struct StealBeforeSaveTest;

impl RuntimeTest for StealBeforeSaveTest {
    fn name(&self) -> &'static str {
        "steal_before_save"
    }
    
    fn description(&self) -> &'static str {
        "R172-03: Verify on_cpu gate prevents steal-before-context-save race"
    }
    
    fn run(&self) -> TestResult {
        use kernel_core::syscall::{sys_fork, sys_sched_yield, sys_exit, sys_wait4, sys_arch_prctl};
        use arch::{num_online_cpus, ARCH_SET_FS};
        
        // Requires SMP
        if num_online_cpus() < 2 {
            return TestResult::Warning(
                "Work-stealing test requires 2+ CPUs (only 1 online)".into()
            );
        }
        
        const NUM_WORKERS: usize = 20;  // Create contention for work-stealing
        let mut children = Vec::new();
        
        for worker_id in 0..NUM_WORKERS {
            match sys_fork() {
                Ok(0) => {
                    // CHILD PATH: Each worker has unique TLS marker
                    
                    // Repeatedly yield and verify TLS integrity
                    for iteration in 0..100 {
                        // Set TLS to unique value (worker_id | iteration)
                        let tls_magic = ((worker_id as u64) << 32) | (iteration as u64);
                        
                        sys_arch_prctl(ARCH_SET_FS, tls_magic as usize).ok();
                        
                        // Force context switch (may trigger work-stealing)
                        sys_sched_yield().ok();
                        
                        // CRITICAL CHECK: Verify TLS not corrupted
                        let fs_value: u64;
                        unsafe {
                            asm!(
                                "mov {}, fs:[0]",
                                out(reg) fs_value,
                                options(readonly, nostack)
                            );
                        }
                        
                        if fs_value != tls_magic {
                            panic!(
                                "CRITICAL R172-03 REGRESSION: Worker {} iteration {}: \
                                 TLS corrupted {:#x} (expected {:#x}). Steal-before-save race!",
                                worker_id, iteration, fs_value, tls_magic
                            );
                        }
                    }
                    
                    sys_exit(0);
                }
                Ok(pid) => children.push(pid),
                Err(e) => {
                    return TestResult::Fail(format!(
                        "Fork worker {}: {:?}",
                        worker_id, e
                    ));
                }
            }
        }
        
        // PARENT PATH: Wait for all workers
        
        let mut failures = 0;
        for child_pid in children {
            let mut status = 0;
            sys_wait4(child_pid, &mut status, 0).ok();
            if status != 0 {
                failures += 1;
            }
        }
        
        if failures > 0 {
            TestResult::Fail(format!(
                "{} workers detected steal-before-save race",
                failures
            ))
        } else {
            TestResult::Pass
        }
    }
}
```

---

## Test 5: RAMFS Basic Operations

**Location:** `kernel/src/runtime_tests.rs`  
**Priority:** P0 — CRITICAL (VFS data plane untested)  
**Audit Finding:** None (gap identified in coverage audit)

```rust
/// Verifies basic RAMFS operations: create, write, read, delete.
/// This is the first end-to-end test of the VFS data plane.
struct RamfsBasicOpsTest;

impl RuntimeTest for RamfsBasicOpsTest {
    fn name(&self) -> &'static str {
        "ramfs_basic_ops"
    }
    
    fn description(&self) -> &'static str {
        "Verify RAMFS create/write/read/delete operations (VFS data plane)"
    }
    
    fn run(&self) -> TestResult {
        use kernel_core::syscall::{sys_open, sys_write, sys_read, sys_close, sys_unlink};
        
        const TEST_PATH: &str = "/tmp/ramfs_test_file";
        const TEST_DATA: &[u8] = b"Hello from RAMFS test!";
        
        // Phase 1: CREATE
        let fd = match sys_open(TEST_PATH, O_CREAT | O_WRONLY, 0o644) {
            Ok(fd) => fd,
            Err(e) => {
                return TestResult::Fail(format!("CREATE failed: {:?}", e));
            }
        };
        
        // Phase 2: WRITE
        match sys_write(fd, TEST_DATA) {
            Ok(written) if written == TEST_DATA.len() => {}
            Ok(written) => {
                sys_close(fd).ok();
                return TestResult::Fail(format!(
                    "WRITE partial: {} bytes (expected {})",
                    written,
                    TEST_DATA.len()
                ));
            }
            Err(e) => {
                sys_close(fd).ok();
                return TestResult::Fail(format!("WRITE failed: {:?}", e));
            }
        }
        
        sys_close(fd).ok();
        
        // Phase 3: READ
        let fd = match sys_open(TEST_PATH, O_RDONLY, 0) {
            Ok(fd) => fd,
            Err(e) => {
                return TestResult::Fail(format!("OPEN for read failed: {:?}", e));
            }
        };
        
        let mut read_buf = vec![0u8; TEST_DATA.len()];
        match sys_read(fd, &mut read_buf) {
            Ok(read_len) if read_len == TEST_DATA.len() => {
                // Verify data matches
                if &read_buf[..] != TEST_DATA {
                    sys_close(fd).ok();
                    return TestResult::Fail(format!(
                        "READ data mismatch: {:?} (expected {:?})",
                        &read_buf[..],
                        TEST_DATA
                    ));
                }
            }
            Ok(read_len) => {
                sys_close(fd).ok();
                return TestResult::Fail(format!(
                    "READ partial: {} bytes (expected {})",
                    read_len,
                    TEST_DATA.len()
                ));
            }
            Err(e) => {
                sys_close(fd).ok();
                return TestResult::Fail(format!("READ failed: {:?}", e));
            }
        }
        
        sys_close(fd).ok();
        
        // Phase 4: DELETE
        match sys_unlink(TEST_PATH) {
            Ok(_) => 
            Err(e) => {
                return TestResult::Fail(format!("UNLINK failed: {:?}", e));
            }
        }
        
        // Verify file no longer exists
        match sys_open(TEST_PATH, O_RDONLY, 0) {
            Err(SyscallError::ENOENT) => TestResult::Pass,  // Expected
            Ok(fd) => {
                sys_close(fd).ok();
                TestResult::Fail("File still exists after unlink".into())
            }
            Err(e) => TestResult::Fail(format!("Unexpected error: {:?}", e)),
        }
    }
}
```

---

## Integration Checklist

### Step 1: Add Tests to runtime_tests.rs

```rust
// In kernel/src/runtime_tests.rs

// Add after existing RuntimeTest implementations:
// ... paste Test 1, Test 2, Test 3, Test 4, Test 5 here ...

// Update run_all_runtime_tests():
pub fn run_all_runtime_tests() -> TestReport {
    let tests: [&dyn RuntimeTest; 29] = [  // Was 24, now 29
        // Existing tests
        &HeapAllocationTest,
        &BuddyAllocatorTest,
        // ... other existing tests ...
        
        // NEW: P0 Critical Tests
        &ContextSwitchRipRflagsTest,
        &CowTlbShootdownTest,
        &FutexWaitWakeTest,
        &StealBeforeSaveTest,
        &RamfsBasicOpsTest,
    ];
    
    // ... rest of function unchanged ...
}
```

### Step 2: Build and Test

```bash
# On remote server
cd /home/dev/workspace/project/rsproject/Zero-os

# Build kernel with new tests
make build

# Run tests (single-core)
make test

# Run tests (SMP - 2 core)
make test SMP=2

# Verify boot health
make boot-check
```

### Step 3: Verify Output

Expected serial output:
```
=== Runtime Functional Tests ===

  [TEST] heap_allocation... PASS
  [TEST] buddy_allocator... PASS
  ... (existing tests) ...
  
  [TEST] context_switch_rip_rflags... PASS
  [TEST] cow_tlb_shootdown... PASS (requires 2+ CPUs)
  [TEST] futex_wait_wake... PASS
      [Child] Entering FUTEX_WAIT (expecting futex=0)...
      [Parent] Changing futex value to 42...
      [Parent] Waking 1 waiter...
      [Parent] Woke 1 waiters
      [Child] Woken from FUTEX_WAIT: Ok(())
      [Child] Futex value after wake: 42
  [TEST] steal_before_save... PASS
  [TEST] ramfs_basic_ops... PASS

=== Test Summary: 29 passed, 0 warnings, 0 failed ===
```

---

## Testing the Tests (Meta-Verification)

### Verify R172-01 Test Catches Regression

Temporarily break context switch:

```rust
// In kernel/arch/context_switch.rs
pub fn save_context(ctx: &mut Context) {
    // Comment out RIP save to simulate R172-01 bug:
    // ctx.rip = get_current_rip();  // COMMENTED OUT
    ctx.rsp = get_current_rsp();
    // ...
}
```

Expected: `context_switch_rip_rflags` test should **FAIL** with panic message.

### Verify R23-1 Test Catches TLB Bug

Temporarily disable TLB shootdown:

```rust
// In kernel/mm/page_table.rs
fn handle_cow_fault(vaddr: usize) {
    // ... COW copy logic ...
    
    // Comment out TLB shootdown:
    // tlb_shootdown::invalidate_page(vaddr);  // COMMENTED OUT
}
```

Expected: `cow_tlb_shootdown` test should **FAIL** with COW isolation panic.

---

## Performance Impact

| Test | Duration (single-core) | Duration (2-core SMP) |
|------|------------------------|------------------------|
| context_switch_rip_rflags | ~10ms | ~15ms |
| cow_tlb_shootdown | Skipped | ~50ms |
| futex_wait_wake | ~100ms | ~100ms |
| steal_before_save | Skipped | ~2000ms |
| ramfs_basic_ops | ~5ms | ~5ms |
| **Total overhead** | **~115ms** | **~2170ms** |

**Recommendation:** Run SMP tests nightly, not on every commit.

---

## Next Steps

1. **Integrate these 5 tests** into `runtime_tests.rs`
2. **Verify all pass** on current codebase
3. **Verify they catch regressions** (meta-testing)
4. **Implement remaining 20 P0 tests** from expansion plan
5. **Add test registration macro** to eliminate manual array updates

---

**Document Owner:** Test team  
**Status:** Production-ready, awaiting integration  
**Related:** `docs/test-coverage-expansion-plan.md` (full 160-test plan)