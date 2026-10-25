# PO-ARC-01: Cross-Registry Lock Ordering and Transaction API

**Status:** IMPLEMENTED → **D2-ARC-CLONE-LOCK downgraded D2 → D3** (2026-07-24). Superseded design notes below.
**Finding:** D2-ARC-CLONE-LOCK
**Related:** PO-ID-01 (identity/membership token design, bundled here)
**Executable Implementation:** `ProcessRegistryTxn` fail-closed try-lock API landed (`process.rs`, D2-ARC slice 2026-07-24)
**This Document:** Formal API design + token machinery + upgrade path

---

## As-Built Amendment (2026-07-24, D2-ARC slice — supersedes §3.2/§4.2/§6.3/§7 where they conflict)

**Landed:** a closure-scoped `with_process_registry_txn(|txn| ...) -> Result<T, RegistryTxnError>` in `kernel/kernel_core/process.rs`. The transaction owns ONLY a reborrow of the entry-frame-held `PROCESS_TABLE` guard plus a fixed `Cell<[Option<ProcessId>; 4]>` held-pid registry; process guards (`TxnProcessGuard<'a,'t>`) are returned to caller locals with region lifetimes and can never escape (HRTB closure signature). Alloc-free, fail-closed (any contention → `Contention`), Drop-before-unlock, self-tested (10-leg boot self-test). Codex-converged SAFE; all remote gates green.

**§3.2 sketch REJECTED** (recorded for the audit trail): the guard-accumulating `Vec` design in §3.2 is unsound and was NOT implemented — (a) `Vec::new()`/`push` under the held `PROCESS_TABLE` guard violates RF180-44 (no allocation while the table is held); (b) `push`-then-`last_mut` is a borrow/realloc hazard; (c) a struct owning the table guard AND storing process guards that borrow through it is self-referential. The `abort → rollback` bullet is also stricken: the API is **validate-everything-then-mutate lock-scope safety, NOT transactional memory** — there is no mutation journal and no rollback.

**Second-caller re-disposition (HONEST):** §4.2's "refactor cgroup migration" premise had drifted — BOTH cgroup front doors (`sys_cgroup_attach` and `cgroupfs.rs` `cgroup.procs`) already consume the R181-3 primitive, which is now reimplemented on the combinator, so both inherit it with zero call-site churn. The genuinely new second consumer is the seccomp STRICT/FILTER install recount (`try_seccomp_isolation_recount_holding`). **Both seccomp legs are DEFENSE-IN-DEPTH, not race fixes.** In particular the FILTER+TSYNC leg **NARROWS but does NOT close** the R37-1 TOCTOU: a residual recount→commit window remains (the caller's held `Process::inner` never blocks a sibling's clone, and the clone gate checks only the cloning task's own `seccomp_installing`), and post-install an unfiltered TSYNC-admitted sibling can still clone a pure-VM task at any time (TSYNC does not fan the filter out). **The R37-1 TSYNC residual stays OPEN** (D3 backlog: a clone-side space-member seccomp gate — itself a future txn consumer — or TSYNC filter fan-out).

**§6.3 verdict:** D2-ARC-CLONE-LOCK → **D3** (bounded adoption backlog). The general primitive exists, is alloc-free/fail-closed/self-tested/documented in `lock_ordering.rs`; first consumer adopted with provably identical semantics; an honest second consumer adopted. Remaining D3-shaped backlog: compile-time token machinery (§3.1, Phase M), optional SLICE-5 cgroup front-door unification, and the R37-1 TSYNC clone-side residual. The downgrade does NOT rest on "the second consumer closes a real race."

---

**Status (original):** Delivered (2026-07-22)  
**Finding:** D2-ARC-CLONE-LOCK  
**Related:** PO-ID-01 (identity/membership token design, bundled here)  
**Executable Implementation:** Point primitive exists (try_address_space_share_count_holding), general API proposed  
**This Document:** Formal API design + token machinery + upgrade path  

---

## Executive Summary

**Purpose:** Design an enforceable cross-registry lock ordering and transaction API that prevents ABBA deadlocks when operating on multiple process/cgroup/namespace registries simultaneously.

**Governing principle:**
```
Safety > Efficiency > Speed
```

A multi-registry operation (clone/fork, cgroup migration, namespace attach) must acquire locks in **global level order** (never reverse) to prevent deadlock. When a caller already holds a lock (e.g., Process::inner at Level 5) and needs to query/modify another registry (e.g., PROCESS_TABLE also at Level 5), they must either:
1. **Release-and-reacquire** (drop Process::inner, take PROCESS_TABLE, re-take Process::inner, verify state)
2. **Try-lock with fail-closed semantics** (try_lock on PROCESS_TABLE, abort on contention)

**Current state:**
- ✅ **Point primitive exists:** `try_address_space_share_count_holding` (process.rs:4636; reimplemented on the ProcessRegistryTxn combinator 2026-07-24) — lock-order-safe CLONE_VM re-check for callers holding Process::inner
- ✅ **Lock levels documented:** `sched/lock_ordering.rs` — 9-level hierarchy with acquisition rules
- ❌ **No token machinery:** Locks don't carry compile-time proof of ordering compliance
- ❌ **No transaction API:** Multi-step operations (lookup → modify → commit) lack atomic primitives

**D2 severity rationale:** Lock ordering violations are MODERATE risk (SMP only, rare caller patterns) but HIGH impact (deadlock = system hang). Existing point primitive works for CLONE_VM; general API needed for future multi-registry ops.

---

## 1. Problem Statement

### 1.1 ABBA Deadlock Risk

**Scenario:** Two threads operating on shared registries in opposite order:

```
Thread A (CPU 0):              Thread B (CPU 1):
lock(PROCESS_TABLE)            lock(Process::inner[PID 42])
  lookup process 42              ...
  lock(Process::inner[42])  <--- WAIT on B
                                 query PROCESS_TABLE  <--- WAIT on A
                                 DEADLOCK
```

**Root cause:** Both locks are at **Level 5** (same hierarchy level), but one operation takes PROCESS_TABLE → Process::inner while another takes Process::inner → PROCESS_TABLE. This violates the "acquire in order, release in reverse" rule.

**Real example (R181-3):** Cgroup migration (cgroupfs attach) holds Process::inner while needing to check CLONE_VM share count (requires PROCESS_TABLE lock). The blocking `address_space_share_count` would ABBA deadlock.

### 1.2 Why Compile-Time Tokens Are Needed

**Current approach (manual discipline):**
```rust
// Correct: PROCESS_TABLE before Process::inner
let table = PROCESS_TABLE.lock();
let process = table.get(pid).unwrap();
let mut proc = process.lock();  // OK: Level 5 → Level 5 (same object)
```

**Problem:** Nothing prevents a developer from writing:
```rust
// WRONG: Process::inner before PROCESS_TABLE (reverse order)
let process = get_current_process();
let mut proc = process.lock();
let table = PROCESS_TABLE.lock();  // ABBA risk if another thread takes opposite order
```

**Desired:** Compile-time tokens that enforce ordering:
```rust
// With tokens (hypothetical):
let table_guard = PROCESS_TABLE.lock();  // Returns ProcessTableGuard<'a>
let process = table_guard.get(pid);      // Returns ProcessWithTableToken<'a>
let proc = process.lock(&table_guard);   // Requires token, proves ordering
```

---

## 2. Existing Point Primitive (R181-3)

### 2.1 try_address_space_share_count_holding

**Function:** `process::try_address_space_share_count_holding(memory_space: usize, held_pid: ProcessId) -> Option<usize>`

**Contract:**
- Caller MUST hold `Process::inner` lock for `held_pid`
- Uses `try_lock` on PROCESS_TABLE (fails closed on contention)
- Skips locking `held_pid` (caller holds it, safe to read without re-lock)
- Uses `try_lock` on all other Process::inner (fails closed on ANY contention)
- Returns `Some(count)` on complete observation, `None` on contention

**Why it's correct:**
1. **PROCESS_TABLE acquired first:** try_lock respects level order (no blocking)
2. **Held lock bypassed:** Caller proves they hold `held_pid` via contract (no re-lock needed)
3. **Fail-closed:** Contention → `None` → EBUSY (user retries), never waits

**Why it's LIMITED:**
- **Single-use:** Only counts CLONE_VM shares, not a general cross-registry primitive
- **No token:** Caller must MANUALLY pass `held_pid` (no compile-time proof they hold it)
- **No write operations:** Read-only query, not a transaction

**Audit trail:** R181-3 FIX (cgroup migration CLONE_VM bypass), verified 2026-07-20.

---

## 3. Proposed API Design

### 3.1 Lock Level Token (Compile-Time Ordering Proof)

**Goal:** Make lock ordering violations a **compile-time error**.

**Design:**
```rust
/// Marker trait: This token proves a lock at Level N has been acquired.
/// Higher-level locks require lower-level tokens to be acquired.
pub trait LockLevelToken {
    const LEVEL: u8;
}

/// Token for PROCESS_TABLE (Level 5)
pub struct ProcessTableToken<'a> {
    _phantom: PhantomData<&'a ()>,
}

impl LockLevelToken for ProcessTableToken<'_> {
    const LEVEL: u8 = 5;
}

/// PROCESS_TABLE lock returns a token
pub fn lock_process_table() -> (ProcessTableGuard, ProcessTableToken<'_>) {
    let guard = PROCESS_TABLE.lock();
    let token = ProcessTableToken { _phantom: PhantomData };
    (guard, token)
}

/// Process::inner lock requires a token IF caller also needs PROCESS_TABLE
impl Process {
    /// Lock without token (standalone access)
    pub fn lock(&self) -> ProcessGuard<'_> { ... }
    
    /// Lock WITH token (caller holds PROCESS_TABLE, proves ordering)
    pub fn lock_with_table_token<'t>(&self, _token: &ProcessTableToken<'t>) -> ProcessGuard<'_> {
        self.inner.lock()  // Safe: table acquired first
    }
}
```

**Usage:**
```rust
// CORRECT: Enforced by token
let (table, token) = lock_process_table();
let process = table.get(pid).unwrap();
let proc = process.lock_with_table_token(&token);  // Requires token

// INCORRECT: Won't compile (no token)
let process = get_current_process();
let proc = process.lock();
let (table, _) = lock_process_table();  // ERROR: no lock_with_table_token method available
```

**Benefits:**
- ✅ Compile-time deadlock prevention (wrong order = type error)
- ✅ Self-documenting (token proves caller acquired table first)
- ✅ Zero runtime cost (PhantomData optimizes away)

**Limitations:**
- ❌ Requires API changes (every cross-registry caller must adopt tokens)
- ❌ Doesn't solve try-lock patterns (tokens can't express contention)

### 3.2 Try-Lock Transaction (Fail-Closed Multi-Step Ops)

**Goal:** Atomic multi-step operations (lookup → check → modify → commit) that fail closed on contention.

**Design:**
```rust
/// Transaction builder for cross-registry operations
pub struct ProcessRegistryTransaction<'a> {
    table_guard: Option<ProcessTableGuard<'a>>,
    held_locks: Vec<(ProcessId, ProcessGuard<'a>)>,
    aborted: bool,
}

impl<'a> ProcessRegistryTransaction<'a> {
    /// Start a transaction (acquires PROCESS_TABLE with try_lock)
    pub fn begin() -> Result<Self, TransactionError> {
        let table_guard = PROCESS_TABLE.try_lock().ok_or(TransactionError::Contention)?;
        Ok(Self {
            table_guard: Some(table_guard),
            held_locks: Vec::new(),
            aborted: false,
        })
    }
    
    /// Try to lock a process (fail-closed on contention)
    pub fn try_lock_process(&mut self, pid: ProcessId) -> Result<&mut ProcessGuard<'a>, TransactionError> {
        if self.aborted {
            return Err(TransactionError::Aborted);
        }
        
        let table = self.table_guard.as_ref().unwrap();
        let process = table.get(pid).ok_or(TransactionError::NotFound)?;
        
        let guard = process.try_lock().ok_or_else(|| {
            self.aborted = true;  // Contention aborts transaction
            TransactionError::Contention
        })?;
        
        self.held_locks.push((pid, guard));
        Ok(self.held_locks.last_mut().unwrap().1)
    }
    
    /// Commit transaction (releases all locks in reverse order)
    pub fn commit(mut self) {
        self.held_locks.clear();
        self.table_guard = None;
    }
    
    /// Abort transaction (releases all locks, no changes)
    pub fn abort(mut self) {
        self.aborted = true;
        self.commit();  // Just release locks
    }
}
```

**Usage:**
```rust
// Example: Cgroup migration with CLONE_VM check
pub fn migrate_process_to_cgroup(pid: ProcessId, new_cgroup: Arc<CgroupNode>) -> Result<(), FsError> {
    let mut tx = ProcessRegistryTransaction::begin().map_err(|_| FsError::Busy)?;
    
    // Step 1: Lock process (fail-closed on contention)
    let proc = tx.try_lock_process(pid).map_err(|_| FsError::Busy)?;
    
    // Step 2: Check CLONE_VM share count (fail-closed, within transaction)
    let share_count = tx.count_address_space_shares(proc.memory_space, pid)
        .map_err(|_| FsError::Busy)?;
    
    if share_count > 1 {
        tx.abort();
        return Err(FsError::PermissionDenied);  // CLONE_VM blocks migration
    }
    
    // Step 3: Modify cgroup membership (atomic)
    proc.cgroup_memberships.clear();
    proc.cgroup_memberships.push(new_cgroup);
    
    // Step 4: Commit (releases all locks)
    tx.commit();
    Ok(())
}
```

**Benefits:**
- ✅ Atomic multi-step operations (all-or-nothing)
- ✅ Fail-closed on contention (never blocks)
- ✅ Automatic lock release (RAII)
- ✅ Clear abort semantics (partial changes rolled back)

**Limitations:**
- ❌ Runtime cost (Vec allocation for held_locks — mitigate with SmallVec<[(ProcessId, Guard); 4]>)
- ❌ Complex lifetimes (guards must outlive transaction)

### 3.3 Identity Token (Membership Proof)

**Goal:** Prove a process is a member of a registry without holding the registry lock.

**Design:**
```rust
/// Token proving a process is in PROCESS_TABLE at generation N
pub struct ProcessIdentityToken {
    pid: ProcessId,
    generation: u64,  // Increments on every process creation/reap
}

impl ProcessIdentityToken {
    /// Validate token (cheap, no lock)
    pub fn is_valid(&self) -> bool {
        // Compare against global generation counter (atomic read)
        PROCESS_GENERATION.load(Ordering::Acquire) == self.generation
    }
    
    /// Re-validate under lock (authoritative)
    pub fn revalidate_holding_table(&self, table: &ProcessTableGuard) -> bool {
        table.get(self.pid).map(|p| p.generation == self.generation).unwrap_or(false)
    }
}

/// PROCESS_TABLE issues tokens on lookup
impl ProcessTableGuard<'_> {
    pub fn get_with_token(&self, pid: ProcessId) -> Option<(Arc<Process>, ProcessIdentityToken)> {
        let process = self.get(pid)?;
        let token = ProcessIdentityToken {
            pid,
            generation: PROCESS_GENERATION.load(Ordering::Acquire),
        };
        Some((process, token))
    }
}
```

**Usage:**
```rust
// Step 1: Lookup with token (under PROCESS_TABLE lock)
let (process, token) = {
    let table = PROCESS_TABLE.lock();
    table.get_with_token(pid).ok_or(FsError::NotFound)?
};

// Step 2: Release PROCESS_TABLE lock, do work
drop(table);
let expensive_result = compute_something(&process);

// Step 3: Re-acquire PROCESS_TABLE, validate token
let table = PROCESS_TABLE.lock();
if !token.revalidate_holding_table(&table) {
    return Err(FsError::Stale);  // Process was reaped/replaced
}

// Step 4: Commit changes (token proves process still valid)
let mut proc = process.lock();
proc.state = expensive_result;
```

**Benefits:**
- ✅ Allows long-running work outside lock (reduces contention)
- ✅ Detects stale references (process reaped during computation)
- ✅ Cheap validation (atomic load)

**Limitations:**
- ❌ Generation counter must wrap safely (u64 = 2^64 processes, effectively infinite)
- ❌ False negatives possible (generation wraps, new process reuses PID+generation — mitigate with 128-bit UUID)

---

## 4. Upgrade Path (Incremental Adoption)

### 4.1 Phase 1: Document + Point Primitives (CURRENT)

**Status:** ✅ COMPLETE (2026-07-22)
- Lock levels documented (`lock_ordering.rs`)
- Point primitive for CLONE_VM check (`try_address_space_share_count_holding`)
- Manual discipline (code review enforces ordering)

**Remaining gaps:**
- No compile-time enforcement
- No transaction API
- No identity tokens

### 4.2 Phase 2: Transaction API (Next)

**Target:** Phase K or L

**Deliverables:**
1. `ProcessRegistryTransaction` implementation (section 3.2)
2. Refactor cgroup migration to use transaction API
3. Add tests: contention abort, multi-process lock, RAII cleanup

**Benefits:**
- Reduces boilerplate (try_lock + abort logic reusable)
- Fail-closed semantics enforced (no silent skip-and-continue)

**Risks:**
- API surface expansion (more for developers to learn)
- Performance regression if Vec allocation is hot (mitigate with SmallVec)

### 4.3 Phase 3: Lock Level Tokens (Later)

**Target:** Phase M or 1.0-Post

**Deliverables:**
1. `LockLevelToken` trait + `ProcessTableToken` (section 3.1)
2. Refactor PROCESS_TABLE lock API to return token
3. Refactor Process::lock to require token when table is held
4. Migrate all callers (grep for `PROCESS_TABLE.lock()` → `lock_process_table()`)

**Benefits:**
- Compile-time deadlock prevention (wrong order = type error)
- Self-documenting (token in signature proves ordering)

**Risks:**
- Large refactor (every PROCESS_TABLE caller must adopt new API)
- Lifetime complexity (tokens tied to guard lifetimes)

**Mitigation:**
- Incremental migration (add token API alongside old API, deprecate old API over 2-3 releases)
- Macro for common patterns (`with_process_table_lock! { ... }`)

### 4.4 Phase 4: Identity Tokens (Optional)

**Target:** Phase M or later (if long-running cross-registry ops become common)

**Deliverables:**
1. `ProcessIdentityToken` implementation (section 3.3)
2. Generation counter in `Process` struct
3. `get_with_token` API on ProcessTableGuard
4. Tests: revalidation, stale detection, wrap-around

**Benefits:**
- Enables long-running work outside lock (reduces contention)
- Detects stale references (ABA prevention)

**Risks:**
- Generation counter overhead (extra u64 per process)
- False negative on wrap (mitigate with 128-bit UUID)

---

## 5. Current Lock Levels (Reference)

**From `sched/lock_ordering.rs` (curated subset):**

| Level | Lock | Module | Type | Notes |
|-------|------|--------|------|-------|
| **3** | READY_QUEUE | sched/enhanced_scheduler.rs | Mutex<BTreeMap> | Per-CPU in SMP |
| **4a** | COW_FAULT_LOCK | kernel_core/fork.rs | Mutex<()> | Serialize COW faults |
| **4b** | PT_LOCK | mm/page_table.rs | Mutex<()> | Global page table ops |
| **4** | BUDDY_ALLOCATOR | mm/buddy_allocator.rs | Mutex | Global frame allocator |
| **5** | PROCESS_TABLE | kernel_core/process.rs | Array<Option<Arc<Mutex>>> | Global process registry |
| **5** | Process::inner | kernel_core/process.rs | Mutex<ProcessInner> | Per-process state |
| **5** | CGROUP_REGISTRY | kernel_core/cgroup.rs | spin::RwLock<BTreeMap> | Global, NON-reentrant |
| **5** | CgroupNode.limits | kernel_core/cgroup.rs | Mutex<CgroupLimits> | Per-node leaf |
| **6** | VFS_ROOT | vfs/lib.rs | Arc<dyn Fs> | Global VFS root |
| **8** | SERIAL_PORT | drivers/serial.rs | Mutex | Debug output |

**Key ordering constraints:**
1. **PROCESS_TABLE → Process::inner** (NEVER reverse)
2. **COW_FAULT_LOCK before PT_LOCK** (memory subsystem)
3. **CGROUP_REGISTRY → CgroupNode.limits** (cgroup hierarchy)
4. **Process::inner → cgroup charge primitives** (allowed: charge outside process lock)

**R181-3 exception:** Caller holding Process::inner can query PROCESS_TABLE via `try_lock` (fail-closed, never blocks).

---

## 6. D2 Severity & Closure

### 6.1 Why D2 (Not Higher)?

**Rationale:**
- ✅ **Point primitive exists:** R181-3's try-lock pattern works for CLONE_VM
- ✅ **Lock levels documented:** Developers have a reference (`lock_ordering.rs`)
- ✅ **Low surface area:** Only 2-3 callers need cross-registry ops today
- ✅ **SMP-only risk:** Single-CPU systems cannot deadlock (scheduler is non-preemptive)

**Moderate risk:** ABBA deadlocks are **unlikely** (rare caller patterns, mostly fixed) but **high-impact** (system hang, requires reboot). D2 is appropriate for "unlikely + high-impact + partially mitigated."

### 6.2 Path to Closure

**Required for D2 closure:**
1. ✅ **Point primitive delivered:** R181-3 `try_address_space_share_count_holding` (2026-07-20)
2. ✅ **PO-ARC-01/PO-ID-01 delivered:** This document (2026-07-22)
3. ⏳ **Transaction API implemented:** Phase K/L (section 4.2)
4. ⏳ **Second caller migrated:** Refactor cgroup migration to use transaction API (Phase K)

**Recommendation:** Accept PO-ARC-01 as **partial closure** of D2-ARC-CLONE-LOCK. Full closure requires transaction API implementation + second caller migration (Phase K, +1 month).

### 6.3 Downgrade to D3?

**Argument for downgrade:**
- Point primitive works (R181-3 in production, tested)
- Low caller count (2-3 sites total)
- Manual code review catches new violations

**Argument against:**
- Transaction API is straightforward (section 3.2 is ~100 LOC)
- General API prevents future violations (self-service for new callers)
- Compile-time tokens (Phase 3) justify staying D2 until delivered

**Verdict:** Keep D2 until transaction API lands (Phase K). After that, consider D3 (well-mitigated).

---

## 7. Summary

| Aspect | Status |
|--------|--------|
| **Point primitive (R181-3)** | ✅ COMPLETE (try_address_space_share_count_holding) |
| **Lock levels documented** | ✅ COMPLETE (sched/lock_ordering.rs) |
| **PO-ARC-01 formal design** | ✅ DELIVERED (this document, 2026-07-22) |
| **Transaction API proposed** | ✅ DELIVERED (section 3.2, not implemented) |
| **Lock level tokens proposed** | ✅ DELIVERED (section 3.1, not implemented) |
| **Identity tokens proposed** | ✅ DELIVERED (section 3.3, not implemented) |
| **Transaction API implemented** | ⏳ PENDING (Phase K, +1 month) |
| **Second caller migrated** | ⏳ PENDING (cgroup migration refactor, Phase K) |
| **D2-ARC finding closure** | ⏳ PARTIAL (design done, implementation pending) |

**Recommendation:** Accept PO-ARC-01/PO-ID-01 as **partial closure** of D2-ARC-CLONE-LOCK. Full closure requires transaction API implementation (Phase K, section 4.2).

---

**Document Version:** 1.0 (2026-07-22)  
**Author:** kernel-implement (Claude Code, MODE autonomous)  
**Review:** Pending (no Codex session required for API design PO)  
**Related Artifacts:** D2-ARC-CLONE-LOCK finding (R181), `try_address_space_share_count_holding` (process.rs:4510), `sched/lock_ordering.rs`, R181-3 FIX, Phase K/L/M roadmap
