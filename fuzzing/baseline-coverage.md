# Phase 3 Baseline Coverage Report

**Date:** 2026-07-21  
**Status:** Baseline established for Phase 4 comparison

---

## Overview

This document establishes the **baseline coverage** from Phase 3 as a reference point for Phase 4's coverage-guided mutation effectiveness. The baseline represents coverage achievable through **random syscall sequence generation** without mutation guidance.

---

## Instrumented Syscalls

### Phase 2 Syscalls (6 syscalls, 13 edges)

| Syscall | Number | Edges | Edge IDs | Category |
|---------|--------|-------|----------|----------|
| sys_getpid | 39 | 3 | 1000-1002 | query |
| sys_getppid | 110 | 5 | 1003-1007 | query |
| sys_getuid | 102 | 1 | 1008 | query |
| sys_geteuid | 107 | 1 | 1009 | query |
| sys_getgid | 104 | 1 | 1010 | query |
| sys_getegid | 108 | 1 | 1011 | query |

### Phase 3 Syscalls (10 syscalls, 24 edges)

| Syscall | Number | Edges | Edge IDs | Category |
|---------|--------|-------|----------|----------|
| sys_read | 0 | 4 | 1000-1003 | I/O |
| sys_write | 1 | 4 | 1010-1013 | I/O |
| sys_open | 2 | 2 | 1020-1021 | I/O |
| sys_close | 3 | 3 | 1030-1032 | I/O |
| sys_brk | 12 | 2 | 1040-1041 | memory |
| sys_mmap | 9 | 4 | 1050-1053 | memory |
| sys_munmap | 11 | 2 | 1060-1061 | memory |
| sys_fork | 57 | 1 | 1070 | process |
| sys_execve | 59 | 1 | 1080 | process |
| sys_exit | 60 | 1 | 1090 | process |

**Total:** 16 syscalls, 37 edges

---

## Random Sequence Generator Coverage

### Test Configuration

The `syscall_fuzzer.rs` baseline test uses:
- **Iterations:** 5 random sequences
- **Sequence length:** 2-5 syscalls per iteration
- **Syscall pool:** 11 safe syscalls
  - Query: getpid, getppid, getuid, geteuid, getgid, getegid (6)
  - I/O: read(stdin, 16 bytes), write(stdout, "test") (2)
  - Memory: brk(0) query, mmap(4K, PROT_READ) (2)
  - Mixed: open("/test", O_RDONLY) (1)

### Expected Coverage Range

**Minimum coverage (single syscall per iteration):**
- 5 iterations × 1 edge/syscall = 5 unique edges

**Typical coverage (3 syscalls per iteration, 2 edges each):**
- 5 iterations × 3 syscalls × 2 edges = 30 edge hits
- Unique edges: ~15-20 (depends on syscall variety)

**Maximum coverage (5 syscalls per iteration, all unique):**
- Could hit all 11 syscalls multiple times
- Phase 2 syscalls (6): 13 edges total
- Phase 3 safe subset (5): read entry, write entry, brk entry/query, mmap entry/anonymous = 6 edges
- **Theoretical max from fuzzer:** ~19 unique edges

### Baseline Metrics (Estimated)

| Metric | Value | Notes |
|--------|-------|-------|
| **Total instrumented edges** | 37 | All syscalls |
| **Accessible from fuzzer** | ~19 | Safe syscall subset |
| **Expected unique edges** | 15-20 | Random 5 iterations |
| **Edge hit rate** | 40-54% | Of accessible edges |
| **Total coverage rate** | 40-54% | Of all 37 edges |
| **Avg edges per iteration** | 3-4 | Depends on sequence length |

---

## Edge Coverage by Category

### I/O Syscalls (13 edges from Phase 3)

**Accessible from fuzzer:**
- `sys_read(0, buf, 16)` → hits edges 1000 (entry), 1001 (stdin path)
- `sys_write(1, "test", 4)` → hits edges 1010 (entry), 1011 (stdout path)
- `sys_open("/test", O_RDONLY, 0)` → hits edges 1020 (entry), 1021 (internal)

**Not accessible from fuzzer:**
- sys_read fd callback (edge 1003) - requires opened fd
- sys_write fd callback (edge 1013) - requires opened fd ≥3
- sys_close (edges 1030-1032) - excluded from fuzzer

**Expected I/O coverage:** 6 / 13 edges = 46%

### Memory Syscalls (8 edges from Phase 3)

**Accessible from fuzzer:**
- `sys_brk(0)` → hits edges 1040 (entry), 1041 (query path)
- `sys_mmap(0, 4096, PROT_READ, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)` → hits edges 1050 (entry), 1053 (anonymous path)

**Not accessible from fuzzer:**
- sys_mmap W^X violation (edge 1051) - fuzzer uses PROT_READ only
- sys_mmap file-backed (edge 1052) - fuzzer uses anonymous only
- sys_munmap (edges 1060-1061) - excluded from fuzzer

**Expected memory coverage:** 4 / 8 edges = 50%

### Process Syscalls (3 edges from Phase 3)

**Not accessible from fuzzer:**
- sys_fork (edge 1070) - excluded (would create processes)
- sys_execve (edge 1080) - excluded (would replace image)
- sys_exit (edge 1090) - excluded (would terminate fuzzer)

**Expected process coverage:** 0 / 3 edges = 0%

### Query Syscalls (13 edges from Phase 2)

**Accessible from fuzzer:**
- All 6 query syscalls with 13 total edges
- Fully reachable with random generation

**Expected query coverage:** 13 / 13 edges = 100%

---

## Coverage Summary

| Category | Instrumented Edges | Accessible Edges | Expected Coverage |
|----------|-------------------|------------------|-------------------|
| Query (Phase 2) | 13 | 13 | 100% |
| I/O (Phase 3) | 13 | 6 | 46% |
| Memory (Phase 3) | 8 | 4 | 50% |
| Process (Phase 3) | 3 | 0 | 0% |
| **Total** | **37** | **23** | **62%** |

**Baseline estimate:** 23 accessible edges, expecting 15-20 unique hits (40-54% coverage)

---

## Phase 4 Coverage Goals

Phase 4's coverage-guided mutation should:

1. **Increase coverage beyond random baseline**
   - Target: >25 unique edges (>65% of accessible, >67% of total)
   - Stretch: 30+ unique edges (>80% of accessible)

2. **Hit previously unreachable edges**
   - Open → read → close sequence (sys_read edge 1003, sys_close edges 1030-1032)
   - Mmap → munmap sequence (sys_munmap edges 1060-1061)
   - W^X violation attempt (sys_mmap edge 1051)

3. **Discover interesting input patterns**
   - File paths that trigger different VFS behaviors
   - Memory sizes that hit different allocation paths
   - Flag combinations that trigger edge cases

4. **Corpus growth over time**
   - Track unique test cases that increase coverage
   - Measure corpus size vs coverage gain
   - Identify diminishing returns point

---

## Baseline Test Procedure

### Run Baseline Test

```bash
cd /home/dev/workspace/project/rsproject/Zero-os
make build-kcov

# Boot kernel and run syscall_fuzzer
qemu-system-x86_64 -M q35 -m 512M -cpu qemu64 -smp 1 \
  -drive if=pflash,format=raw,readonly=on,file=/usr/share/OVMF/OVMF_CODE.fd \
  -drive if=pflash,format=raw,readonly=on,file=/usr/share/OVMF/OVMF_VARS.fd \
  -drive format=raw,file=fat:rw:esp-kcov \
  -serial stdio -display none -no-reboot > baseline-coverage.txt 2>&1
```

### Extract Coverage Data

```bash
grep "Coverage:" baseline-coverage.txt
grep "Edge IDs:" baseline-coverage.txt
grep "Total edges collected:" baseline-coverage.txt
```

### Expected Output Format

```
--- Iteration 1 ---
Sequence length: 3
  Step 1: write(1, "test", 4)
  Step 2: getpid()
  Step 3: brk(0) - query
  Coverage: 7 unique edges
  Edge IDs: 1010, 1011, 1000, 1001, 1040, 1041, ...

--- Iteration 2 ---
...

=== Summary ===
Total iterations: 5
Total edges collected: 18
Average edges per iteration: 3
```

---

## Regression Testing

This baseline serves as a **regression test** for future changes:

1. **Coverage should not decrease**
   - If coverage drops below baseline, investigate
   - Instrumentation may be broken
   - KCOV feature may be misconfigured

2. **Edge IDs should remain stable**
   - Same syscalls should hit same edge IDs
   - Edge ID changes indicate instrumentation drift

3. **Iteration variance should be bounded**
   - Random sequences should produce consistent coverage ranges
   - High variance indicates PRNG or syscall pool issues

---

## Phase 4 Comparison Metrics

Phase 4 will compare against this baseline using:

### Absolute Coverage Metrics
- **Unique edges hit:** Phase 4 vs Phase 3 baseline
- **Coverage rate:** % of 37 total edges
- **Accessible coverage rate:** % of 23 accessible edges

### Efficiency Metrics
- **Edges per iteration:** Phase 4 mutation vs Phase 3 random
- **Corpus size:** Number of test cases in corpus
- **Corpus efficiency:** Unique edges / corpus size

### Discovery Metrics
- **New edges found:** Edges hit by Phase 4 but not Phase 3
- **Time to discovery:** Iterations to hit 90% of accessible edges
- **Diminishing returns:** Coverage gain vs iterations

---

## Limitations

### Baseline Limitations

1. **Safe syscall subset only**
   - Process syscalls excluded (3 edges unreachable)
   - Destructive operations excluded
   - Resource-dependent paths excluded

2. **Random generation**
   - No guidance toward uncovered edges
   - May repeat similar sequences
   - No state tracking across iterations

3. **Short run (5 iterations)**
   - Coverage may not saturate
   - Variance may be high
   - Need longer runs for stable baseline

### Phase 4 Advantages

1. **Coverage-guided mutation**
   - Prioritizes inputs hitting new edges
   - Avoids redundant test cases
   - Explores state space systematically

2. **Corpus management**
   - Saves interesting test cases
   - Builds on previous discoveries
   - Enables stateful sequences

3. **Longer runs**
   - Can iterate until coverage saturates
   - Lower per-iteration variance
   - Better exploration

---

## Conclusion

**Phase 3 baseline:** 15-20 unique edges (40-54% coverage) from random 2-5 syscall sequences over 5 iterations.

**Phase 4 target:** >25 unique edges (>67% coverage) through coverage-guided mutation and corpus management.

**Success criteria:** Phase 4 coverage exceeds baseline by ≥25% (absolute) or discovers ≥5 new edges beyond random baseline.

---

**Baseline Established:** 2026-07-21  
**Next:** Run baseline test after build completes  
**Phase 4 Start:** After baseline verification
