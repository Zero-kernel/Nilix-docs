# sync_safe::Mutex Migration Guide

**Target Audience:** Nilix kernel developers  
**Purpose:** Step-by-step guide for adopting IRQ-safe locking primitives  
**Related:** kernel/sync_safe/lib.rs, IRQ_SAFETY_AUDIT.md

---

## Table of Contents

1. [Overview](#overview)
2. [When to Use sync_safe::Mutex](#when-to-use)
3. [Migration Steps](#migration-steps)
4. [Testing Checklist](#testing-checklist)
5. [Common Pitfalls](#common-pitfalls)
6. [Examples](#examples)

---

## Overview

### What is sync_safe::Mutex?

`sync_safe::Mutex` is a drop-in replacement for `spin::Mutex` that adds **runtime IRQ-safety checks** in debug mode:

```rust
impl<T> Mutex<T> {
    pub fn lock(&self) -> MutexGuard<'_, T> {
        #[cfg(debug_assertions)]
        {
            // Panic if interrupts are disabled (RFLAGS.IF=0)
            self.assert_irq_safe("Mutex::lock()");
        }
        // ... acquire lock ...
    }

    pub fn try_lock(&self) -> Option<MutexGuard<'_, T>> {
        // Safe from any context - returns None if contended
    }
}
```

**Key Features:**
- ✅ Drop-in replacement for `spin::Mutex`
- ✅ Zero overhead in release builds
- ✅ Catches R173-class deadlocks at runtime
- ✅ Same API: `lock()`, `try_lock()`, `MutexGuard`

---

## When to Use sync_safe::Mutex

### ✅ USE for These Locks

**1. Global Tables Accessed from IRQ Context**
- PROCESS_TABLE (timer IRQ, #NM)
- SOCKET_WAITERS (timer IRQ check_timeouts)
- Any lock acquired in deferred drains

**2. Locks Held Across Context Switches**
- Scheduler data structures
- Process control blocks
- Waitqueues

**3. Hot-Path Locks**
- Locks acquired frequently (>1000/sec)
- Locks in timer IRQ path
- Locks in syscall fast paths

### ❌ DO NOT USE for These Locks

**1. IRQ-Only Locks**
Locks acquired ONLY in IRQ handlers with interrupts already disabled:
```rust
// This is safe - IRQs are already off in handler entry
extern "x86-interrupt" fn some_irq_handler(...) {
    let guard = LOCK.lock(); // OK - IRQs off by x86-interrupt
}
```

**2. Init-Only Locks**
Locks acquired only during single-threaded boot:
```rust
pub fn init_subsystem() {
    static INIT_LOCK: Mutex<()> = Mutex::new(());
    let _guard = INIT_LOCK.lock(); // OK - no IRQs during init
}
```

**3. Per-CPU Locks with Explicit IRQ Disable**
```rust
x86_64::instructions::interrupts::without_interrupts(|| {
    let guard = CPU_LOCAL_LOCK.lock(); // OK - explicit IRQ disable
});
```

### ⚠️ MAYBE USE (Case-by-Case)

**Process-Context-Only Locks**
If a lock is ONLY acquired from syscalls (never from IRQs), `spin::Mutex` is technically safe. But `sync_safe::Mutex` adds defense-in-depth:

```rust
// fd_table is only accessed from syscalls
// But sync_safe::Mutex catches accidental IRQ use
pub struct Process {
    pub fd_table: sync_safe::Mutex<FileDescriptorTable>,
}
```

---

## Migration Steps

### Step 1: Identify Candidate Locks

**Automated Search:**
```bash
# Find all spin::Mutex declarations
rg "Mutex<" kernel/ | grep -v "sync_safe::"

# Find IRQ-reachable functions
rg "interrupt.*fn|extern \"x86-interrupt\"" kernel/arch/
```

**Manual Analysis:**
For each lock, ask:
1. Is this function called from an IRQ handler?
2. Is this lock acquired in a deferred drain?
3. Could this lock be held when an IRQ fires?

**Use call graph tracing:**
```bash
# Trace all callers of a function
rg "function_name\(" kernel/
```

### Step 2: Replace Import

**Before:**
```rust
use spin::Mutex;
```

**After:**
```rust
use sync_safe::Mutex;
```

**For RwLock (no sync_safe version yet):**
```rust
use spin::RwLock; // Keep as-is
use sync_safe::Mutex; // Only for Mutex
```

### Step 3: Update Lock Type (If Static)

**Before:**
```rust
static GLOBAL_LOCK: spin::Mutex<Data> = spin::Mutex::new(...);
```

**After:**
```rust
static GLOBAL_LOCK: sync_safe::Mutex<Data> = sync_safe::Mutex::new(...);
```

### Step 4: Verify All Acquisition Sites

For each `.lock()` call, verify one of:
- ✅ Called only from process context (syscalls, kthreads)
- ✅ Wrapped in `without_interrupts(|| ...)`
- ✅ Converted to `try_lock()` with defer logic

**Anti-pattern:**
```rust
pub fn some_function_called_from_irq() {
    let guard = LOCK.lock(); // BAD - will panic in debug if IRQ-reachable
}
```

**Correct patterns:**
```rust
// Pattern A: try_lock with defer
pub fn irq_drain_function() {
    let Some(guard) = LOCK.try_lock() else {
        return; // Defer to next opportunity
    };
}

// Pattern B: without_interrupts guard
pub fn registration_function() {
    x86_64::instructions::interrupts::without_interrupts(|| {
        let guard = LOCK.lock(); // SAFE - IRQs explicitly off
    });
}
```

### Step 5: Add IRQ-Safety Documentation

```rust
/// Drain deferred work items.
///
/// # IRQ Safety
///
/// This function is called from `reschedule_if_needed()` which runs in
/// process context with interrupts ENABLED (enforced by debug_assert).
/// All lock acquisitions use try_lock pattern to prevent same-CPU deadlock.
///
/// # Panics
///
/// In debug builds, panics if called with interrupts disabled.
pub fn drain_deferred_work() {
    // ...
}
```

---

## Testing Checklist

### Pre-Deployment Testing

- [ ] **Build with debug assertions enabled**
  ```bash
  cd /path/to/kernel
  make build  # Should have debug_assertions in dev profile
  ```

- [ ] **Boot kernel in QEMU**
  ```bash
  make boot-check
  ```
  Watch for panic messages like:
  ```
  FATAL: Mutex::lock() called with interrupts disabled (RFLAGS.IF=0)
  This is a R173-01/02 class deadlock!
  Use try_lock() instead in IRQ context.
  ```

- [ ] **Run test suite**
  ```bash
  make test
  ```
  All tests should pass without IRQ-safety panics.

- [ ] **Trigger IRQ handlers**
  - **Timer IRQ:** Happens naturally every ~1ms
  - **Keyboard IRQ:** Type characters in console
  - **#NM:** Run FPU-using process
  - **Page Fault:** Access unmapped memory (e.g., COW)

### Stress Testing

- [ ] **High timer frequency**
  Reduce APIC timer divisor to increase IRQ rate:
  ```rust
  // In kernel/arch/apic.rs
  const TIMER_DIVISOR: u32 = 1; // Increase IRQ frequency 16x
  ```

- [ ] **Fork bomb**
  ```bash
  # In userspace shell
  :(){ :|:& };:
  ```
  Should not deadlock or panic.

- [ ] **Socket I/O stress**
  Run network throughput test to trigger socket waiters + timer IRQ interaction.

### Negative Testing

Add deliberate violation to verify detection works:

```rust
#[cfg(test)]
mod tests {
    #[test]
    #[should_panic(expected = "called with interrupts disabled")]
    fn test_irq_safety_violation() {
        use sync_safe::Mutex;
        static TEST_LOCK: Mutex<u32> = Mutex::new(0);
        
        x86_64::instructions::interrupts::without_interrupts(|| {
            let _guard = TEST_LOCK.lock(); // Should panic
        });
    }
}
```

---

## Common Pitfalls

### Pitfall #1: Nested Locks with IRQ Disable

**Problem:**
```rust
let guard1 = LOCK_A.lock(); // Disables IRQs
let guard2 = LOCK_B.lock(); // Panics: IRQs already off!
```

**Solution:**
Disable IRQs once around both:
```rust
x86_64::instructions::interrupts::without_interrupts(|| {
    let guard1 = LOCK_A.lock();
    let guard2 = LOCK_B.lock();
});
```

### Pitfall #2: Long-Held Locks with IRQs Off

**Problem:**
```rust
x86_64::instructions::interrupts::without_interrupts(|| {
    let guard = LOCK.lock();
    do_expensive_work(); // IRQs off for milliseconds!
});
```

**Solution:**
Minimize critical section:
```rust
let data = x86_64::instructions::interrupts::without_interrupts(|| {
    LOCK.lock().clone() // Copy while IRQs off
});
do_expensive_work_on_copy(data); // IRQs enabled
```

### Pitfall #3: Forgetting to Check IRQ Context

**Problem:**
```rust
pub fn helper_function() {
    let guard = LOCK.lock(); // Might be called from IRQ!
}
```

**Solution:**
Document and verify:
```rust
/// # IRQ Safety
/// This function MUST NOT be called from IRQ context.
pub fn helper_function() {
    debug_assert!(
        x86_64::instructions::interrupts::are_enabled(),
        "helper_function called from IRQ context"
    );
    let guard = LOCK.lock();
}
```

### Pitfall #4: False Sense of Security

**Problem:**
```rust
// Developer thinks this is safe because it's in a syscall
pub fn sys_some_operation() {
    let guard = LOCK.lock(); // But what if IRQ fires here?
}
```

**Solution:**
Use sync_safe::Mutex everywhere, not just "obviously IRQ-adjacent" code:
```rust
// Even syscall-only locks benefit from sync_safe
static SYSCALL_LOCK: sync_safe::Mutex<Data> = ...;
```

---

## Examples

### Example 1: Converting PROCESS_TABLE

**Before (spin::Mutex):**
```rust
use spin::Mutex;

static PROCESS_TABLE: Lazy<Mutex<HashMap<ProcessId, Arc<Mutex<Process>>>>> = 
    Lazy::new(|| Mutex::new(HashMap::new()));

pub fn get_process(pid: ProcessId) -> Option<Arc<Mutex<Process>>> {
    PROCESS_TABLE.lock().get(&pid).cloned()
}
```

**After (sync_safe::Mutex):**
```rust
use sync_safe::Mutex;

static PROCESS_TABLE: Lazy<Mutex<HashMap<ProcessId, Arc<Mutex<Process>>>>> = 
    Lazy::new(|| Mutex::new(HashMap::new()));

// Syscall-only version (blocking)
pub fn get_process(pid: ProcessId) -> Option<Arc<Mutex<Process>>> {
    PROCESS_TABLE.lock().get(&pid).cloned()
}

// IRQ-safe version (non-blocking)
pub fn try_get_process(pid: ProcessId) -> Option<Option<Arc<Mutex<Process>>>> {
    PROCESS_TABLE.try_lock().map(|table| table.get(&pid).cloned())
}
```

### Example 2: Converting Deferred Drain

**Before (blocking locks):**
```rust
pub fn drain_deferred_work() {
    x86_64::instructions::interrupts::without_interrupts(|| {
        let mut queue = WORK_QUEUE.lock();
        while let Some(work) = queue.pop() {
            if let Some(proc) = get_process(work.pid) {
                let mut guard = proc.lock();
                guard.state = ProcessState::Ready;
            }
        }
    });
}
```

**After (try_lock pattern):**
```rust
pub fn drain_deferred_work() {
    let mut queue = match WORK_QUEUE.try_lock() {
        Some(q) => q,
        None => return, // Defer to next reschedule
    };

    while let Some(work) = queue.pop() {
        match try_get_process(work.pid) {
            Some(Some(proc)) => {
                if let Some(mut guard) = proc.try_lock() {
                    guard.state = ProcessState::Ready;
                }
            }
            _ => continue,
        }
    }
}
```

### Example 3: Registration with IRQ Disable

**Before:**
```rust
pub fn register_callback(cb: Callback) {
    let mut cbs = CALLBACKS.lock(); // Risky if IRQ fires
    cbs.push(cb);
}
```

**After:**
```rust
pub fn register_callback(cb: Callback) {
    x86_64::instructions::interrupts::without_interrupts(|| {
        let mut cbs = CALLBACKS.lock(); // SAFE - IRQs off
        cbs.push(cb);
    });
}

// IRQ handler can now safely acquire lock
extern "x86-interrupt" fn irq_handler(...) {
    let cbs = CALLBACKS.lock(); // Won't deadlock
    for cb in cbs.iter() {
        cb();
    }
}
```

---

## FAQ

### Q: What's the performance impact?

**A:** In **release builds**: Zero overhead. The IRQ check is compiled out.

In **debug builds**: ~2-3 CPU cycles per lock acquisition (one RFLAGS read + test).

### Q: Can I mix spin::Mutex and sync_safe::Mutex?

**A:** Yes. They have identical ABIs and can coexist:
```rust
use spin::Mutex as SpinMutex;
use sync_safe::Mutex as SafeMutex;

static COLD_PATH: SpinMutex<Data> = SpinMutex::new(...);
static HOT_PATH: SafeMutex<Data> = SafeMutex::new(...);
```

### Q: What if I need RwLock?

**A:** sync_safe currently only provides `Mutex`. For `RwLock`, use:
- `spin::RwLock` with `try_read()` / `try_write()` in IRQ paths
- Manual `debug_assert!(interrupts::are_enabled())` before `.read()` / `.write()`

### Q: How do I debug a sync_safe panic?

**A:** The panic message includes:
1. **Lock site:** Which `.lock()` call failed
2. **RFLAGS state:** "interrupts disabled (RFLAGS.IF=0)"
3. **Stack trace:** (via Rust panic handler)

Use GDB to inspect call stack:
```
(gdb) bt
#0  sync_safe::Mutex::assert_irq_safe (...)
#1  sync_safe::Mutex::lock (...)
#2  problematic_function (...)
```

Trace backward to find where IRQs were disabled or which IRQ handler called the function.

### Q: Should I convert all locks at once?

**A:** No. Incremental migration is safer:
1. **Phase 1:** Convert IRQ-adjacent locks (PROCESS_TABLE, waitqueues)
2. **Phase 2:** Convert hot-path locks (scheduler, PCBs)
3. **Phase 3:** Convert remaining global locks (defense-in-depth)

Test thoroughly after each phase.

---

## Rollback Procedure

If sync_safe::Mutex causes issues:

1. **Revert import:**
   ```rust
   -use sync_safe::Mutex;
   +use spin::Mutex;
   ```

2. **Keep try_lock fixes:**
   All try_lock conversions are universally safe and should be kept.

3. **File bug report with:**
   - Panic message and stack trace
   - Reproduction steps
   - QEMU serial log
   - Output of `git diff` showing changes

---

## References

- [kernel/sync_safe/lib.rs](../sync_safe/lib.rs) - Implementation
- [kernel/sync_safe/README.md](../sync_safe/README.md) - Design rationale
- [IRQ_SAFETY_AUDIT.md](./IRQ_SAFETY_AUDIT.md) - Audit findings
- [IRQ_LOCK_SITE_INVENTORY.md](./IRQ_LOCK_SITE_INVENTORY.md) - Complete lock catalog
- R173 bug report - Historical context

---

**Document Version:** 1.0  
**Last Updated:** 2026-07-02  
**Next Update:** After Phase 2 sync_safe rollout
