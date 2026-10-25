# Extended Test Suite Documentation

This document describes the comprehensive test infrastructure for Nilix, including stress tests, performance regression gates, security mitigation tests, melting tests, and extended SMP validation.

## Overview

The test suite is organized into the following categories:

1. **Stress Tests** - Resource leak and stability validation
2. **Performance Regression Gate** - Prevents accidental slowdowns
3. **Security Mitigation Tests** - Hardware/compiler-dependent security validation
4. **Melting Tests** - Sustained maximum load (real hardware only)
5. **Extended SMP Tests** - High core count validation (8-core, 16-core)

## 1. Stress Tests

**Location:** `scripts/gates/stress/stress_test.sh`

**Purpose:** Catches resource exhaustion, memory leaks, and stability issues under sustained load that fuzzing won't expose in short runs.

**Test Scenarios:**
- Memory Pressure - approach heap limits, verify recovery
- CPU Saturation - all cores 100% for extended duration
- SMP Contention - cross-CPU lock pressure
- I/O Sustained - VFS/block layer endurance
- Process Churn - rapid fork/exit cycles
- Combined Load - all stressors simultaneously

**Usage:**
```bash
# Standard stress test (60 seconds per test)
make stress-test

# Extended stress test (300 seconds per test)
make stress-test-extended

# Custom duration and CPU count
STRESS_DURATION=120 STRESS_CPUS=8 bash scripts/gates/stress/stress_test.sh esp
```

**Environment Variables:**
- `STRESS_DURATION` - Duration in seconds per test (default: 60)
- `STRESS_CPUS` - Number of CPU cores for SMP tests (default: 4)
- `STRESS_MEM` - Memory size for tests (default: 256M)

**Exit Codes:**
- 0 = STABLE - all stress tests passed
- 1 = UNSTABLE - crash, hang, or panic detected
- 2 = DEGRADED - completed but with performance degradation
- 3 = BLOCKED - missing prerequisites

**What It Catches:**
- Memory leaks (heap usage grows unbounded)
- Deadlocks under contention
- Resource exhaustion recovery failures
- SMP race conditions under heavy load
- Scheduler starvation issues

---

## 2. Performance Regression Gate

**Location:** `scripts/gates/performance/perf_regression_test.sh`

**Purpose:** Validates that performance metrics stay within acceptable bounds (±10% threshold by default). Prevents accidental slowdowns from reaching production.

**Benchmarks Tracked:**
- Syscall latency baseline (< 500ns)
- Context switch overhead (< 5μs)
- Page fault handling (< 2μs)
- Memory allocation speed (> 1M ops/sec)
- VFS operations throughput (> 50K ops/sec)
- Network loopback (> 1Gbps)

**Usage:**
```bash
# Run performance regression gate
make test-perf

# Custom threshold
REGRESSION_THRESHOLD=15 bash scripts/gates/performance/perf_regression_test.sh esp
```

**Environment Variables:**
- `PERF_TIMEOUT` - Performance test timeout (default: 120s)
- `REGRESSION_THRESHOLD` - Percentage slowdown limit (default: 10)

**Exit Codes:**
- 0 = PASS - all metrics within bounds
- 1 = REGRESSION - >threshold% slowdown detected
- 2 = BLOCKED - missing prerequisites

**Status:**
⚠️ **Current Implementation:** Framework in place with placeholder benchmarks. Full implementation requires dedicated performance benchmark code in kernel/userspace.

**Future Work:**
- Implement dedicated benchmark userspace programs
- Add kernel performance counters
- Integrate with TSC-based precise timing
- Add historical performance tracking

---

## 3. Security Mitigation Tests

**Location:** `kernel/security/tests.rs`

**Purpose:** Validates hardware and compiler-dependent security mitigations are properly enabled.

**Tests Included:**
- `spectre_v1_bounds_check` - Array bounds check enforcement
- `spectre_v2_retpoline` - Indirect call mitigation via retpoline
- `smap_enforcement` - Supervisor Mode Access Prevention (CR4 bit validation)
- `smep_enforcement` - Supervisor Mode Execution Prevention (CR4 bit validation)
- `quick_wxorx_check` - W^X policy validation (existing)
- `full_wxorx_validation` - Full page table walk (existing)
- `rng_entropy_health` - CSPRNG health check (existing)
- `kptr_guard_obfuscation` - Kernel pointer obfuscation (existing)
- `spectre_mitigation_status` - Overall Spectre/Meltdown status (existing)

**Usage:**
```bash
# Security tests are integrated into runtime test suite
make test

# Run only security subsystem tests
# (Access via runtime_tests.rs SecuritySubsystemTest)
```

**What It Validates:**
- **SMAP:** Kernel cannot dereference user pointers without explicit access
- **SMEP:** Kernel cannot execute user code
- **Spectre V1:** Bounds checks prevent speculative execution attacks
- **Spectre V2:** Retpoline or IBRS mitigates indirect branch prediction
- **W^X:** No pages are both writable and executable
- **KASLR:** Kernel addresses are randomized (boot_check.sh)

**Implementation Details:**
The security tests are now part of the runtime test framework and execute during `make test`. They validate:
1. CPU feature bits (CPUID checks)
2. Control register settings (CR4 for SMAP/SMEP)
3. Compiler mitigation flags (retpoline)
4. Runtime security policy enforcement

---

## 4. Melting Tests

**Location:** `scripts/gates/performance/melting_test.sh`

**Purpose:** Tests kernel stability under sustained maximum load for extended periods (10+ minutes). **Designed for REAL HARDWARE ONLY** - QEMU results are unrealistic.

**Test Scenarios:**
- CPU thermal stress (all cores 100% for 10+ minutes)
- Memory pressure sustained (near capacity for 10+ minutes)
- I/O continuous (disk/VFS saturation)
- Network packet flood (sustained high packet rate)
- Combined maximum load (all stressors for 30+ minutes)

**Usage:**
```bash
# Standard melting test (10 minutes minimum)
make test-melting

# Extended melting test (custom duration)
MELT_DURATION=1800 bash scripts/gates/performance/melting_test.sh
```

**Environment Variables:**
- `MELT_DURATION` - Duration in seconds (default: 600, minimum: 600)

**Exit Codes:**
- 0 = STABLE - survived sustained load
- 1 = UNSTABLE - crash, panic, or severe degradation
- 2 = BLOCKED - not on real hardware or missing prerequisites

**Why Real Hardware Only:**
- QEMU doesn't model thermal behavior
- Artificial CPU scheduling in VMs
- Unrealistic memory pressure behavior
- Network timing doesn't match real hardware

**Prerequisites:**
- Physical x86_64 machine
- Adequate cooling
- Monitoring tools (sensors, vmstat, perf)
- Remote logging infrastructure

**Status:**
⚠️ **Current Implementation:** Framework in place. Full implementation requires:
1. Bare metal boot infrastructure (bootable USB/PXE)
2. Stress workload userspace programs
3. Remote monitoring and logging
4. Automated pass/fail criteria

**Monitors:**
- No crashes or panics
- No memory leaks (heap usage stable)
- No performance cliff (throughput degradation)
- Thermal stability (if sensors available)

---

## 5. Extended SMP Tests

**Location:** `scripts/gates/boot/extended_smp_test.sh`

**Purpose:** Validates SMP functionality at higher core counts (8-core, 16-core) that standard 2/4-core tests don't expose.

**Test Configurations:**
- 8-core stress test
- 16-core stress test

**Usage:**
```bash
# Run extended SMP tests
make test-smp-extended

# Custom timeout
EXTENDED_SMP_TIMEOUT=120 bash scripts/gates/boot/extended_smp_test.sh esp
```

**Environment Variables:**
- `EXTENDED_SMP_TIMEOUT` - Test timeout in seconds (default: 90)

**Exit Codes:**
- 0 = PASS - all cores online, tests passed
- 1 = FAILED - panic, hang, or core initialization failure
- 2 = BLOCKED - missing prerequisites

**What It Validates:**
- All CPUs come online properly
- Scheduler handles high core count
- Lock contention doesn't cause deadlocks
- IPI/TLB shootdown scales
- No race conditions under high concurrency

**Acceptance Criteria:**
- At least 75% of CPUs must come online
- `smp_online` test must PASS
- No NX violations or CPU resets
- No supervisor faults
- SMP-specific tests pass (R175, RF178-33, IPI, TLB)

---

## Comprehensive Test Suite

**Purpose:** Runs all test categories in sequence for complete validation.

**Usage:**
```bash
# Run all tests (boot, runtime, SMP, stress, perf)
make test-comprehensive

# Quick smoke test (essential gates only)
make test-quick
```

**Test Sequence:**
1. Boot health check (`boot_check.sh`)
2. Runtime test suite (`kernel_test.sh`)
3. Musl conformance (`musl_check.sh`)
4. SMP 2-core validation (`smp_test.sh`)
5. SMP 4-core validation (`smp_test_4core.sh`)
6. Extended SMP validation (8-core, 16-core)
7. Ext3/JBD2 production gate
8. Stress test suite
9. Performance regression gate

**Total Duration:** ~15-20 minutes (depending on STRESS_DURATION)

---

## Integration with Existing Tests

The extended test suite complements the existing test infrastructure:

### Existing Tests (Already Implemented)
- ✅ `boot-check.sh` - Boot health validation
- ✅ `kernel_test.sh` - Runtime test suite gate
- ✅ `musl_check.sh` - Musl libc conformance
- ✅ `smp_test.sh` - 2-core SMP validation
- ✅ `smp_test_4core.sh` - 4-core SMP validation
- ✅ Fuzzing integration - syzkaller + KCOV

### New Tests (This Implementation)
- ✅ `stress_test.sh` - Stress testing suite
- ✅ `perf_regression_test.sh` - Performance regression gate
- ✅ `melting_test.sh` - Melting test framework
- ✅ `extended_smp_test.sh` - 8/16-core SMP validation
- ✅ Security mitigation tests (in `kernel/security/tests.rs`)

---

## CI/CD Integration

### Recommended CI Pipeline

```yaml
# Fast feedback (run on every commit)
quick_check:
  - make boot-check
  - make test
  - make lint

# Full validation (run on PR)
comprehensive:
  - make test-comprehensive

# Nightly validation (run on schedule)
extended:
  - make stress-test-extended
  - make test-smp-extended
  - make test-perf

# Release validation (manual trigger)
release_gate:
  - make test-comprehensive
  - make stress-test-extended
  - MELT_DURATION=1800 make test-melting  # Real hardware only
```

---

## Test Coverage Summary

| Category | Current Status | What It Catches |
|----------|---------------|-----------------|
| **Boot Tests** | ✅ Implemented | Boot failures, NX violations, early panics |
| **Self Tests** | ✅ 50+ tests | Structural invariants, API contracts |
| **Runtime Tests** | ✅ 40+ tests | Functional correctness, integration |
| **SMP Tests** | ✅ 2/4/8/16-core | Multi-core races, scheduler issues |
| **Stress Tests** | ✅ Implemented | Memory leaks, resource exhaustion |
| **Performance** | ⚠️ Framework | Regression detection (needs benchmarks) |
| **Security** | ✅ Enhanced | SMAP/SMEP/Spectre/W^X validation |
| **Melting** | ⚠️ Framework | Thermal stability (needs real HW) |
| **Fuzzing** | ✅ syzkaller+KCOV | Edge cases, input validation |

**Legend:**
- ✅ Fully implemented and operational
- ⚠️ Framework in place, full implementation pending

---

## Known Limitations

1. **Performance benchmarks** are placeholder - require dedicated benchmark implementation
2. **Melting tests** need real hardware infrastructure (bootable USB/PXE, remote logging)
3. **Security tests** depend on CPU features (SMAP/SMEP not available on all CPUs)
4. **Extended SMP** tests are most effective on real hardware with actual multi-core CPUs

---

## Future Enhancements

1. **Performance benchmarks:** Implement dedicated userspace benchmark programs
2. **Melting infrastructure:** Set up bare metal boot and remote logging
3. **Historical tracking:** Store performance metrics over time
4. **Coverage reporting:** Integrate with code coverage tools
5. **Real hardware CI:** Add physical test machines to CI pipeline

---

## Troubleshooting

### Stress tests fail with "KERNEL PANIC"
- Check serial log for panic location
- May indicate resource exhaustion bug or deadlock
- Run with `STRESS_DURATION=30` for faster iteration

### Performance tests always skip
- Benchmarks are placeholder - this is expected
- Implement dedicated benchmark code to enable

### Security tests fail with "not supported by CPU"
- SMAP/SMEP require modern CPUs (Intel Haswell+, AMD Zen+)
- Tests will WARN but not FAIL on unsupported hardware

### Extended SMP tests timeout
- Increase `EXTENDED_SMP_TIMEOUT` for slow hosts
- Check that QEMU supports high core counts
- Verify AP initialization completes in serial log

---

## Contact and Support

For issues or questions about the test suite:
- Check logs in serial output and QEMU interrupt log
- Review test script comments for detailed behavior
- Consult existing gate scripts (`boot_check.sh`, `smp_test.sh`) for patterns

---

**Last Updated:** 2026-07-21
