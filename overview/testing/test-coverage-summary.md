# Nilix Kernel Test Coverage — Executive Summary

**Date:** 2026-07-02  
**Status:** Coverage audit complete, expansion plan ready  
**Recommendation:** Implement Block 1 (P0) tests before 1.0-Preview release

---

## Key Findings

### Current Coverage: 204 Tests

| Test Type | Count | Quality |
|-----------|-------|---------|
| Unit tests | 129 | Good (metadata, parsers, config) |
| Self-tests | 48 | Excellent (cgroup, quota, budgets) |
| Integration | 27 | Fair (boot, SMP, namespaces) |

**Strengths:**
- ✅ Cgroup accounting (M2-1, J.2) — 10 comprehensive self-tests
- ✅ Network parsing — 22 unit tests (TCP, UDP, ARP, firewall)
- ✅ Security config — 21 tests (KASLR, RNG, W^X logic)
- ✅ Namespace isolation — 3 integration tests

**Critical Gaps:**
- ❌ **Context switch** — 0 tests (R172-01 privilege escalation untested)
- ❌ **COW page faults** — 0 tests (R23-1 TLB shootdown untested)
- ❌ **Futex operations** — 0 tests (core IPC primitive completely untested)
- ❌ **VFS data plane** — 0 tests (RAMFS create/write/read untested)
- ❌ **Scheduler work-stealing** — 0 tests (R172-03 steal race untested)
- ❌ **Fork/exec lifecycle** — 0 tests (COW setup, AS transition untested)
- ❌ **Page fault handler** — 0 tests (#PF dispatch completely untested)

---

## Risk Assessment

### CRITICAL Untested Paths (8 gaps)

| ID | Path | Known Bug | Impact if Broken |
|----|------|-----------|------------------|
| 1 | Context switch RIP/RFLAGS | R172-01 | Privilege escalation to Ring 0 |
| 2 | COW + TLB shootdown | R23-1 | Silent data corruption, permission bypass |
| 3 | Page table SMP access | R67-5 | Memory corruption under load |
| 4 | Futex WAIT/WAKE | R24-5 | Deadlock, lost-wakeup |
| 5 | PI futex priority | E.4 | Priority inversion DoS |
| 6 | Work stealing race | R172-03 | Double-run, task corruption |
| 7 | RAMFS operations | - | VFS data loss |
| 8 | VFS path resolution | - | Authorization bypass |

**Verdict:** These paths have **zero test coverage** and are **known to have been buggy** (R172/R171 fixes). One regression could block 1.0-Preview.

---

## Recommended Action: 4-Block Test Expansion

### Block 1: P0 Security-Critical (25 tests) — **REQUIRED for 1.0-Preview**

**Timeline:** 2 weeks  
**Effort:** 40 engineer-hours  
**Risk Mitigation:** Prevents regression of R172-01, R172-03, R172-04, R172-05, R23-1

| Week | Tests | Subsystem | Critical Path |
|------|-------|-----------|---------------|
| 1 | 5 | arch | Context switch RIP/RFLAGS/TLS |
| 1 | 5 | mm | COW page faults + TLB shootdown |
| 2 | 5 | ipc | Futex WAIT/WAKE/PI |
| 2 | 5 | sched | Work stealing correctness |
| 2 | 5 | vfs | RAMFS operations + path resolution |

**Gate:** All 25 tests green → 1.0-Preview qualified

---

### Block 2: P1 Regression Prevention (45 tests)

**Timeline:** 2 weeks  
**Effort:** 70 engineer-hours  
**Coverage:** Full R172/R171 audit finding coverage

- Fork/exec lifecycle (10 tests)
- Signal delivery paths (8 tests)
- Memory management (12 tests)
- Network stack (8 tests)
- Security hardening (7 tests)

---

### Block 3: P2 Concurrency & SMP (40 tests)

**Timeline:** 2 weeks  
**Effort:** 60 engineer-hours  
**Target:** Clean 4-core SMP runs

- Lock ordering validation (10 tests)
- SMP stress tests (15 tests)
- Race condition detection (15 tests)

---

### Block 4: P3 Edge Cases & Fuzzing (50 tests)

**Timeline:** 2 weeks  
**Effort:** 80 engineer-hours  
**Goal:** Robustness hardening

- Resource exhaustion (15 tests)
- Invalid inputs (20 tests)
- Privilege boundaries (15 tests)

---

## Test Infrastructure Improvements

### Priority 1: Remove Manual Test Registration

**Problem:** Tests must be manually added to hardcoded arrays  
**Solution:** Inventory-based automatic discovery

```rust
// Current (manual):
let tests: [&dyn RuntimeTest; 24] = [
    &Test1, &Test2, // ... must update count
];

// Proposed (automatic):
register_runtime_test!(Test1);
register_runtime_test!(Test2);
// Count auto-discovered at compile time
```

### Priority 2: SMP Test Orchestration

**Problem:** No framework for multi-CPU synchronization  
**Solution:** SMP test harness with barriers, CPU affinity

```rust
let harness = SmpTestHarness::new();
harness.spawn_on_cpu(0, || { /* CPU 0 work */ });
harness.spawn_on_cpu(1, || { /* CPU 1 work */ });
harness.barrier(); // Synchronize
```

### Priority 3: Coverage Reporting

**Problem:** No visibility into coverage by subsystem  
**Solution:** Automated coverage tracker mapped to ARCHITECTURE.md

```rust
CoverageTracker::report()
// Output:
//   arch: 15/15 critical paths covered (100%)
//   mm: 8/20 critical paths covered (40%)
//   ...
```

---

## Success Metrics

### 1.0-Preview Release Gate

- ✅ All 25 P0 tests pass
- ✅ 90% P1 tests pass (40/45)
- ✅ 0 open CRITICAL findings
- ✅ 0 open HIGH findings
- ✅ Boot-check green
- ✅ Musl-check green

### Post-1.0 Quality Targets

- 80% critical path coverage (measured against ARCHITECTURE.md)
- 1000+ fuzzing iterations per syscall
- <1% test flakiness rate
- 4-core SMP clean

---

## Implementation Roadmap

### Week 1-2: P0 Critical (Block 1)

**Deliverables:**
- 5 context switch tests (R172-01/04/05)
- 5 COW/TLB tests (R23-1)
- 5 futex tests (R172-08, E.4)
- 5 scheduler tests (R172-03)
- 5 VFS tests (RAMFS ops)

**Gate:** `make test` includes all P0 tests, all green

---

### Week 3-4: P1 Regression (Block 2)

**Deliverables:**
- 45 tests covering all R172/R171 findings
- Full fork/exec lifecycle coverage
- Signal delivery path coverage

**Gate:** No R172/R171 finding lacks test

---

### Week 5-6: P2 SMP Hardening (Block 3)

**Deliverables:**
- 40 SMP stress tests
- Lock ordering validation
- 4-core test runs

**Gate:** `make test SMP=4` green

---

### Week 7-8: P3 Fuzzing (Block 4)

**Deliverables:**
- 50 edge case tests
- Syscall fuzzing infrastructure
- Resource exhaustion coverage

**Gate:** 1000+ fuzzing iterations per syscall

---

## Cost-Benefit Analysis

### Investment

- **Engineer time:** 250 hours (160 tests + infrastructure)
- **CI time:** +10 minutes per commit (P0 tests)
- **Maintenance:** ~5 hours/month (test updates)

### Return

- **Prevent 1 critical regression:** Saves 80+ hours (audit + fix + verification)
- **Catch bugs pre-production:** 10x cheaper than post-release fix
- **Enable confident refactoring:** Test coverage = safety net
- **Reduce audit rounds:** Fewer findings = faster audits

**ROI:** 1 critical bug prevented pays for entire test expansion

---

## Recommendations

### Immediate Actions (This Sprint)

1. **Implement 5 P0 arch tests** (context switch) — 8 hours
2. **Implement 5 P0 mm tests** (COW/TLB) — 8 hours
3. **Add test registration macro** — 4 hours

### Next Sprint

4. **Complete Block 1** (remaining 15 P0 tests) — 24 hours
5. **Add SMP test harness** — 8 hours
6. **Start Block 2** (P1 tests) — 30 hours

### Before 1.0-Preview

7. **All 25 P0 tests green**
8. **40/45 P1 tests green**
9. **Coverage report showing gap closure**

---

## Open Questions

1. **Should P0 tests block commits or just PRs?**
   - Recommendation: Block PRs (too slow for pre-commit)

2. **Run SMP tests on every commit or nightly?**
   - Recommendation: 2-core on commit, 4-core nightly

3. **Fuzzing: continuous or scheduled?**
   - Recommendation: Weekly 2-hour fuzzing runs

4. **Test flakiness: retry or fail-fast?**
   - Recommendation: 1 retry for SMP tests, fail-fast for unit tests

---

## Appendix: Test Count by Subsystem

| Subsystem | Current | Target | Gap | Priority |
|-----------|---------|--------|-----|----------|
| arch | 4 | 15 | +11 | **P0** |
| kernel_core | 37 | 60 | +23 | P1 |
| mm | 4 | 20 | +16 | **P0** |
| sched | 5 | 20 | +15 | **P0** |
| ipc | 2 | 15 | +13 | **P0** |
| vfs | 11 | 25 | +14 | **P0** |
| net | 25 | 35 | +10 | P2 |
| security | 21 | 30 | +9 | P2 |
| cgroup | 10 | 15 | +5 | P3 |
| **TOTAL** | **204** | **364** | **+160** | |

---

## Appendix: Test Infrastructure Status

| Component | Status | Priority |
|-----------|--------|----------|
| Test registration | Manual | P1 (automate) |
| SMP orchestration | None | P1 (needed for P0 tests) |
| Coverage tracking | None | P2 |
| Fuzzing framework | None | P3 |
| Property testing | None | P3 |
| Performance benchmarks | None | P4 |

---

**Document Owner:** QA Team  
**Stakeholders:** Kernel team, Release management  
**Next Review:** After implementing Block 1 (2 weeks)

**Full Details:** See `docs/test-coverage-expansion-plan.md` for complete test implementations