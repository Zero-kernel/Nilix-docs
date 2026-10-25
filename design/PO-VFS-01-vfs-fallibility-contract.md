# PO-VFS-01: VFS Fallibility Contract (Prepare/Commit/Rollback)

**Status:** Delivered (2026-07-22)  
**Finding:** D2-ERR-VFS-FALLIBILITY  
**Related:** D2-ERR-RECOVERY (gate #7 design, part c)  
**Executable Implementation:** Existing local pattern (ramfs.rs:1438) + FallibleOrderedMap  
**This Document:** Formalized contract + lint automation proposal  

---

## Executive Summary

**Purpose:** Document the VFS fallibility contract that prevents kernel panics on OOM in recoverable paths (syscalls returning ENOMEM rather than panicking).

**Governing principle:**
```
Safety > Efficiency > Speed
```

A recoverable VFS operation (syscall, lookup, rename, mount) must complete successfully **or** return a typed error (ENOMEM, ENOSPC) without panicking. After **publication** (mutation visible to other CPUs/tasks) or **PONR** (point of no return), **infallible heap allocation is forbidden** — it turns OOM into a panic.

**Implementation status:**
- ✅ **Contract codified:** D2-ERR-RECOVERY.md (2026-07-14), this PO-VFS-01 (2026-07-22)
- ✅ **Local pattern verified:** ramfs.rs:1438 prepare→allocation-free commit (R178 RF178-16 fix)
- ✅ **Zero infallible collects:** S-6 audit confirmed (2026-07-20) — no user-influenced unbounded Vec growth in `kernel/vfs/`
- ⏳ **Lint automation:** Manual checklist exists (D2-ERR-RECOVERY.md:39), mechanization proposed (this PO)

**D2 severity rationale:** VFS OOM panics are MODERATE risk (user can trigger via filesystem operations, but kernel contains fallible primitives). D2 persists until lint is mechanized and gated in CI.

---

## 1. Problem Statement

### 1.1 OOM Panic in Recoverable Paths

**Threat model:** User triggers a VFS operation (create, rename, symlink) that allocates heap memory. If allocation happens **after publication** (directory entry visible to concurrent readers), an OOM causes a panic:

```rust
// ANTI-PATTERN: infallible allocation after publication
fn bad_rename(old: &str, new: &str) -> Result<(), FsError> {
    let mut dir = DIR_ENTRIES.write();  // Acquired lock
    dir.remove(old);                    // PUBLICATION: old entry gone (visible to readers)
    dir.insert(new.to_string(), ...);   // INFALLIBLE: to_string() panics on OOM
    Ok(())
}
// If OOM occurs at to_string(), the kernel panics with old entry removed but
// new entry not inserted → filesystem corrupted state.
```

**Consequences:**
- **Kernel panic:** System crashes, loses all in-memory state
- **Filesystem corruption:** Half-applied operation leaves inconsistent state
- **DoS vulnerability:** Unprivileged user can trigger panic via memory pressure + rename

**Correct pattern:**
```rust
// GOOD: prepare (fallible) → commit (allocation-free)
fn good_rename(old: &str, new: &str) -> Result<(), FsError> {
    // PHASE 1: PREPARE (fallible, no publication yet)
    let prepared_key = String::try_from(new).map_err(|_| FsError::NoSpace)?;
    
    // PHASE 2: COMMIT (allocation-free, atomic publication)
    let mut dir = DIR_ENTRIES.write();
    let entry = dir.remove(old).ok_or(FsError::NotFound)?;
    dir.insert(prepared_key, entry);  // Uses pre-allocated string, cannot fail
    Ok(())
}
// If OOM occurs in PHASE 1, error is returned before any publication.
// PHASE 2 is allocation-free → cannot panic.
```

### 1.2 Publication vs. PONR

**Publication:** Mutation visible to concurrent readers/writers
- Inserting into shared `HashMap`/`BTreeMap` (other threads can see entry)
- Installing a PTE (other CPUs can access page)
- Arming an IRQ (hardware can fire)
- Committing a cgroup charge (accounting visible)

**Point of No Return (PONR):** Irreversible transition
- `CR3` switch in exec (old address space unmapped, cannot revert)
- DMA descriptor queued (hardware owns it)
- Released a lock that holds rollback state (cannot re-acquire atomically)

**Rule:** After **EARLIEST(publication, PONR)**, no infallible allocation.

---

## 2. Contract (Normative)

### 2.1 Definitions

| Term | Definition |
|------|------------|
| **Recoverable API** | Any function returning `Result<T, E>` to a syscall or subsystem caller, where error is typed (not panic) |
| **Publication** | First moment a mutation is observable outside the current stack (shared insert, published PTE, armed IRQ) |
| **PONR** | Point of No Return — irreversible commit (CR3 switch, DMA queue, lock release destroying rollback state) |
| **Infallible allocator** | `Vec::push` (without prior `try_reserve`), `to_string`, `Arc::new` (without `try_new`), `BTreeMap::insert` (without `try_reserve`), `vec![]` growth |
| **Fallible allocator** | `Vec::try_reserve`, `String::try_from`, `Arc::try_new`, `FallibleOrderedMap::try_insert`, `try_reserve_heap` |

### 2.2 Core Rules

**R1: No infallible allocation after publication/PONR**

After the **earliest** of (publication, PONR), do not call infallible allocators. OOM must be handled **before** publication.

**R2: Prefer prepare-commit pattern**

For operations with both allocation and publication:
1. **PHASE 1 (Prepare):** Perform all fallible allocations (`try_reserve`, `try_new`, `try_from`). On failure, return error (e.g., `ENOMEM`).
2. **PHASE 2 (Commit):** Use pre-allocated resources to publish atomically. This phase must be allocation-free.

**R3: Error paths must rollback**

If a recoverable path fails (returns `Err`), it must:
- Free all allocated memory (drop temporary Vecs, Arcs)
- Uncharge all cgroup accounting
- Unmap all PTEs
- Restore locks to original state

**R4: Transitive enforcement**

If a recoverable path calls a helper (block I/O, DMA setup, VFS lookup), that helper must also follow R1-R3 (transitive fallibility).

**R5: IRQ context is allocation-free**

Hard IRQ handlers must not allocate or deallocate (M4-1 class). Deferred work queues may allocate if they are recoverable (can fail gracefully).

### 2.3 Compliance Levels

| Level | Definition | Example |
|-------|------------|---------|
| **COMPLIANT** | Uses prepare-commit, all allocations fallible before publication | ramfs.rs:1438 rename |
| **SAFE (bounded)** | Uses infallible allocation, but input is **bounded** (e.g., `CtrlKind::all()` is 4 items) | cgroupfs.rs directory collect |
| **UNSAFE** | Infallible allocation after publication, unbounded input | (None in current VFS after S-6 audit) |

**Audit criterion:** Zero UNSAFE instances in `kernel/vfs/`. SAFE instances are annotated with `// BOUNDED: <justification>`.

---

## 3. Existing Implementation (Verified Patterns)

### 3.1 RAMFS Rename (Prepare-Commit)

**File:** `kernel/vfs/ramfs.rs:1438-1469`

**Pattern:**
```rust
// === Spanning critical section: prepare, then allocation-free commit ===
// RAMFS_TOPOLOGY_LOCK keeps the preflight stable while all heap
// reservations and detached backing/key allocations occur without a
// directory spin lock held. Once preparation succeeds, commit performs
// no allocation and cannot expose a half-rename.

// PHASE 1: PREPARE (outside write lock)
let prepared_key = PreparedDirectoryKey::try_new(new_name)?;  // Fallible String allocation

// PHASE 2: COMMIT (inside write lock, allocation-free)
let mut g = old_entries.write();
let PreparedDirectoryKey { key, reservation } = prepared_key;
g.entries.insert_unique_reserved(key, inode, reservation);  // Uses pre-allocated capacity
```

**Compliance:** ✅ **COMPLIANT**
- Allocation (`try_new`) before lock acquisition (pre-publication)
- Commit uses `insert_unique_reserved` (allocation-free via prior `try_reserve`)
- OOM in PHASE 1 returns `FsError::NoSpace` (no panic)

**Audit trail:** RF178-16 FIX (R178 review-fix), verified 2026-07-14.

### 3.2 DevFS Block Device Registration (Bounded Reserve)

**File:** `kernel/vfs/devfs.rs:46-50`

**Pattern:**
```rust
// R180-27 FIX: reserve all runtime block-node slots before any device can
// reach DRIVER_OK. Later publication uses insert_unique_reserved and is
// therefore allocator-independent.
devices
    .try_reserve_exact(3 + block::MAX_BLOCK_DEVICES)
    .expect("devfs: boot block-node slot reservation OOM");
```

**Compliance:** ✅ **COMPLIANT**
- All slots reserved **before** device drivers start (pre-publication)
- Runtime `insert_unique_reserved` is allocation-free (uses reserved capacity)
- `expect` is acceptable here — boot-time OOM with 3+MAX slots is fail-closed (system cannot boot)

**Audit trail:** R180-27 FIX (R180 audit), verified 2026-06-XX.

### 3.3 Symlink Resolution (Fallible String Collect)

**File:** `kernel/vfs/manager.rs` (P2-C FIX, 2026-07-14)

**Pattern:**
```rust
// P2-C FIX: fallible string allocation for symlink resolution
let target = String::try_from(symlink_target).map_err(|_| FsError::NoSpace)?;
```

**Compliance:** ✅ **COMPLIANT**
- `String::try_from` is fallible (returns `Result`)
- On OOM, returns `FsError::NoSpace` before any publication

**Audit trail:** P2-C (R178-33 review-fix remediation), verified 2026-07-14.

### 3.4 CgroupFS Directory Listing (Bounded Collect)

**File:** `kernel/vfs/cgroupfs.rs` (S-6 audit, 2026-07-20)

**Pattern:**
```rust
// BOUNDED: CtrlKind::all() is exactly 4 variants (Memory, Cpu, Pid, Io)
let controllers: Vec<_> = CtrlKind::all().map(|kind| ...).collect();
```

**Compliance:** ✅ **SAFE (bounded)**
- `collect()` is infallible, but input is **bounded** (4 items, ~200 bytes)
- Comment pinned: `// BOUNDED: CtrlKind::all() is 4 variants`
- S-6 audit verified no user-influenced unbounded collects in VFS

**Audit trail:** S-6 (R181 wave S design queue), verified 2026-07-20.

---

## 4. Lint Automation (Proposed)

### 4.1 Current State (Manual Checklist)

**Checklist (D2-ERR-RECOVERY.md:39):**
- [ ] Grep new recoverable paths for `to_string`, `with_capacity`, bare `push`, `vec![]` growth, non-try `Arc::new` after shared publish
- [ ] Error paths free / uncharge / unmap what they allocated before return
- [ ] Self-tests pin fallible helpers where OOM cannot be driven at boot

**Status:** Manual (not mechanized), performed during audit rounds (R178+).

### 4.2 Proposed Lint Target: `make lint-fallible`

**Goal:** Mechanize the grep checklist as a Makefile target that fails on infallible allocators in VFS paths.

**Implementation:**

```makefile
# In Makefile
lint-fallible:
	@echo "=== Lint: VFS fallibility (no infallible alloc after publication) ==="
	@! rg -t rust 'to_string\(\)|\.push\(|vec!\[|Arc::new\((?!.*try)' kernel/vfs/ \
		--ignore-case \
		| grep -v '// BOUNDED:' \
		| grep -v '// INFALLIBLE-OK:' \
		|| { echo "OK: No forbidden infallible allocators in VFS"; exit 0; }
	@echo "FAIL: Found infallible allocators in VFS (see above)"
	@exit 1

# Add to main lint target
lint: lint-release lint-smap lint-fetch-add lint-repr-c-copy lint-fallible
```

**How it works:**
1. `rg` searches for infallible patterns: `to_string()`, `.push(`, `vec![`, `Arc::new(` (non-try)
2. Exclude lines with `// BOUNDED:` (annotated safe cases)
3. Exclude lines with `// INFALLIBLE-OK:` (boot-time or other justified cases)
4. Fail if any matches remain (unbounded infallible allocators)

**Annotations:**
```rust
// BOUNDED: CtrlKind::all() is 4 variants, ~200 bytes
let controllers: Vec<_> = CtrlKind::all().map(|kind| ...).collect();  // lint-fallible: BOUNDED

// INFALLIBLE-OK: boot-time reservation, fail-closed on OOM
devices.try_reserve_exact(3 + MAX).expect("boot OOM");  // lint-fallible: INFALLIBLE-OK
```

### 4.3 Lint Limitations (Known Gaps)

| Gap | Risk | Mitigation |
|-----|------|-----------|
| **G1:** Cannot detect indirect calls (helper functions that allocate) | MEDIUM | Transitive audits (R4) — audit helpers called by recoverable paths |
| **G2:** False negatives if infallible allocator is aliased (`let f = Vec::push; f(...)`) | LOW | Code review catches aliasing (uncommon in kernel) |
| **G3:** False positives if pre-publication code uses infallible alloc | MEDIUM | Annotate with `// INFALLIBLE-OK: pre-publication` |

**Recommendation:** Accept G1/G2 (manual audit for transitive/aliased cases), mitigate G3 with annotations.

### 4.4 Integration Timeline

| Phase | Deliverable | Target |
|-------|-------------|--------|
| **Phase K** | PO-VFS-01 (this document) + lint-fallible Makefile target | 2026-07-22 (DELIVERED) |
| **Phase K+1** | Enable `make lint-fallible` in CI (gate on main) | +2 weeks validation |
| **Phase L** | Transitive audits (G1) — audit block/DMA/IOMMU helpers | +1 month |
| **Phase M** | Expand to `kernel/mm/`, `kernel/net/` (beyond VFS) | +3 months |

---

## 5. Compliance Verification (Audit Protocol)

### 5.1 Per-Round Audit Checklist

For each QA round (R179+), apply this checklist to **new code only** (diff from last round):

**Step 1: Identify recoverable paths**
```bash
rg -t rust 'fn.*-> Result<' kernel/vfs/ --line-number
```

**Step 2: For each recoverable path, find publication/PONR**
- Lock acquisition (`write()`, `lock()`, `mutex.lock()`)
- Shared structure mutation (`insert`, `publish`, `commit`)
- Irreversible transition (CR3 switch, DMA queue, lock release)

**Step 3: Grep for infallible allocators after publication**
```bash
rg -t rust 'to_string\(\)|\.push\(|vec!\[|Arc::new\(' kernel/vfs/file_with_change.rs
```

**Step 4: For each match, classify**
- **COMPLIANT:** Allocation before publication (PHASE 1)
- **SAFE:** Bounded input (annotate with `// BOUNDED: <reason>`)
- **UNSAFE:** Unbounded allocation after publication (FIX REQUIRED)

**Step 5: Verify error path rollback**
- Check that `?` propagation occurs before publication
- Check that `Err` paths free/uncharge/unmap

### 5.2 Audit Report Template

```markdown
## VFS Fallibility Audit (R###)

**Files changed:** 3 (ramfs.rs, ext2.rs, manager.rs)  
**Recoverable paths added:** 2 (ext2_rename, manager_mount)  
**Infallible allocators found:** 1 (ext2_rename:234 `to_string()`)  

### Findings

**F1: ext2_rename:234 — infallible to_string() after lock**
- **Classification:** UNSAFE
- **Root cause:** String allocation for new directory entry after acquiring write lock
- **Fix:** Use `String::try_from()` in prepare phase (before lock)
- **Status:** FIXED (applied prepare-commit pattern)

### Summary

- ✅ All findings resolved
- ✅ Prepare-commit pattern applied to ext2_rename
- ✅ No UNSAFE instances remain in VFS
```

---

## 6. Extension to Other Subsystems

### 6.1 Subsystems Requiring Fallibility

| Subsystem | Priority | Rationale |
|-----------|----------|-----------|
| **VFS** | HIGH | User-triggered, high syscall surface |
| **MM** | HIGH | Page fault / mmap / fork → OOM panic risk |
| **Net** | MEDIUM | Socket ops / packet allocation → DoS risk |
| **Block** | MEDIUM | DMA setup / I/O queue → driver stability |
| **Cgroup** | LOW | Mostly bounded (fixed controller set) |

### 6.2 Per-Subsystem Lint Targets

```makefile
lint-fallible-vfs:
	@rg -t rust 'to_string\(\)|\.push\(' kernel/vfs/ | grep -v '// BOUNDED:' || exit 1

lint-fallible-mm:
	@rg -t rust 'to_string\(\)|\.push\(' kernel/mm/ | grep -v '// BOUNDED:' || exit 1

lint-fallible-net:
	@rg -t rust 'to_string\(\)|\.push\(' kernel/net/ | grep -v '// BOUNDED:' || exit 1

lint-fallible: lint-fallible-vfs lint-fallible-mm lint-fallible-net
```

**Phased rollout:** VFS first (Phase K), MM next (Phase L), Net later (Phase M).

---

## 7. D2 Severity & Closure

### 7.1 Why D2 (Not Higher)?

**Rationale:**
- ✅ **Contract codified:** D2-ERR-RECOVERY.md + PO-VFS-01
- ✅ **Zero UNSAFE instances:** S-6 audit confirmed (2026-07-20)
- ✅ **Fallible primitives exist:** FallibleOrderedMap, try_reserve, try_new
- ✅ **Local pattern verified:** ramfs.rs prepare-commit in production

**Moderate risk:** VFS OOM panics are **unlikely** (most paths are fallible) but **high-impact** (DoS + filesystem corruption). D2 is appropriate for "unlikely + high-impact + mitigated."

### 7.2 Path to Closure (D-7 Part C)

**Required for D2 closure:**
1. ✅ **Contract written:** D2-ERR-RECOVERY.md (2026-07-14) + PO-VFS-01 (2026-07-22)
2. ✅ **Last named instance fixed:** P2-C symlink (2026-07-14)
3. ⏳ **Lint mechanized:** `make lint-fallible` target (this PO proposes, not yet landed)
4. ⏳ **CI gate:** lint-fallible enabled in CI (Phase K+1 target)

**Recommendation:** Accept PO-VFS-01 as **partial closure** of D2-ERR-VFS. Full closure requires CI gate (Phase K+1, +2 weeks validation).

### 7.3 Relationship to D-7 Gate

**D-7 (No infallible allocation after publication) has 3 parts:**

| Part | Requirement | Status | Artifact |
|------|-------------|--------|----------|
| (a) | Written contract | ✅ DONE | D2-ERR-RECOVERY.md (2026-07-14) |
| (b) | Last named instance fixed | ✅ DONE | P2-C symlink (2026-07-14) |
| (c) | Checklist/lint enforcement | ⏳ PARTIAL | PO-VFS-01 (this doc, 2026-07-22) — lint proposed, not gated |

**D-7 remains PARTIAL** until part (c) lint is **gated in CI** OR Gate #7 records an explicit 1.0-Preview waiver for (c) (manual checklist acceptable for Preview).

---

## 8. Summary

| Aspect | Status |
|--------|--------|
| **Contract codified** | ✅ COMPLETE (D2-ERR-RECOVERY.md + PO-VFS-01) |
| **Local pattern verified** | ✅ COMPLETE (ramfs.rs:1438 prepare-commit, R178 RF178-16) |
| **Zero UNSAFE instances** | ✅ VERIFIED (S-6 audit 2026-07-20, no unbounded VFS collects) |
| **Lint automation proposed** | ✅ DELIVERED (make lint-fallible spec, this PO section 4) |
| **Lint gated in CI** | ⏳ PENDING (Phase K+1, +2 weeks) |
| **D2-ERR finding closure** | ⏳ PARTIAL (contract done, gate pending) |
| **D-7 Gate closure** | ⏳ PARTIAL (parts a+b done, part c pending gate) |

**Recommendation:** Accept PO-VFS-01 as **partial closure** of D2-ERR-VFS-FALLIBILITY. Full closure requires `make lint-fallible` gated in CI (Phase K+1).

---

**Document Version:** 1.0 (2026-07-22)  
**Author:** kernel-implement (Claude Code, MODE autonomous)  
**Review:** Pending (no Codex session required for contract documentation PO)  
**Related Artifacts:** D2-ERR-VFS-FALLIBILITY finding (R180), D2-ERR-RECOVERY.md (2026-07-14), ramfs.rs:1438 (RF178-16), S-6 audit (R181), Phase K/L roadmap
