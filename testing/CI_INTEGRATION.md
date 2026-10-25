# CI/CD Integration Guide

This document describes how the extended test suite integrates with CI/CD pipelines.

## Overview

The Nilix test infrastructure supports multiple testing tiers:

1. **Fast Feedback** - Every commit (~5 minutes)
2. **Pull Request Gate** - Before merge (~20 minutes)
3. **Monthly Stress Test** - Last day of each month (automated)
4. **Manual Extended Tests** - On-demand validation

---

## 1. Fast Feedback (Every Commit)

**Triggered:** On every push to any branch

**Duration:** ~5 minutes

**Tests:**
```bash
make boot-check  # Boot health validation
make test        # Runtime + security tests
make lint        # Code quality checks
```

**Configuration:** `.github/workflows/ci.yml` (existing)

**Purpose:** Catch regressions immediately during development

---

## 2. Pull Request Gate (Before Merge)

**Triggered:** On pull request creation/update

**Duration:** ~20 minutes

**Tests:**
```bash
make test-comprehensive
```

**Includes:**
- Boot health check
- Runtime test suite (40+ tests)
- Musl conformance
- SMP validation (2-core, 4-core)
- Extended SMP (8-core, 16-core)
- Ext3/JBD2 production gate
- Stress test suite (60s duration)
- Performance regression gate

**Configuration:** Add to existing PR workflow or create `.github/workflows/pr-gate.yml`

**Purpose:** Ensure comprehensive validation before merging changes

---

## 3. Monthly Stress Test (Automated)

**Triggered:** Last day of each month at 2:00 AM UTC

**Duration:** ~60-120 minutes

**Tests:**
- Standard stress test (60s per scenario, 6 scenarios)
- Extended stress test (300s per scenario, 6 scenarios)

**Configuration:** `.github/workflows/monthly-stress-test.yml` ✅ IMPLEMENTED

**Features:**
- ✅ Automatic execution on the last day of every month
- ✅ Manual trigger option (`workflow_dispatch`)
- ✅ Handles months with 28/29/30/31 days correctly
- ✅ Generates stress test report artifact (90-day retention)
- ✅ Posts commit comment on failure
- ✅ 120-minute timeout for extended tests

**Purpose:** Detect memory leaks, stability issues, and resource exhaustion over time

**Schedule Logic:**
```yaml
schedule:
  # Runs at 2:00 AM UTC on days 28-31
  - cron: '0 2 28-31 * *'

# Then checks if tomorrow is day 1 (= today is last day)
```

**Manual Trigger:**
```bash
# Via GitHub UI: Actions → Monthly Stress Test → Run workflow
# Or via CLI:
gh workflow run monthly-stress-test.yml
```

---

## 4. Manual Extended Tests

**Triggered:** Manual execution for deep validation

**Tests Available:**

### Extended Stress Test
```bash
make stress-test-extended
```
- 300 seconds per scenario
- 6 scenarios: memory, CPU, SMP, I/O, process churn, combined
- Total duration: ~30 minutes

### Extended SMP Test
```bash
make test-smp-extended
```
- 8-core and 16-core validation
- Tests scheduler scaling and high-contention scenarios
- Duration: ~10 minutes

### Performance Regression Gate
```bash
make test-perf
```
- Validates performance metrics within bounds
- Framework operational (benchmarks pending)
- Duration: ~2 minutes

**Purpose:** Deep validation before releases or after major changes

---

## Test Matrix by Environment

| Test Category | Fast CI | PR Gate | Monthly | Manual |
|---------------|---------|---------|---------|--------|
| Boot health | ✅ | ✅ | - | ✅ |
| Runtime tests | ✅ | ✅ | - | ✅ |
| Lint checks | ✅ | ✅ | - | ✅ |
| SMP 2/4-core | - | ✅ | - | ✅ |
| SMP 8/16-core | - | ✅ | - | ✅ |
| Stress (60s) | - | ✅ | ✅ | ✅ |
| Stress (300s) | - | - | ✅ | ✅ |
| Performance gate | - | ✅ | - | ✅ |
| Security tests | ✅ | ✅ | - | ✅ |

---

## Monthly Stress Test Details

### Workflow File: `.github/workflows/monthly-stress-test.yml`

**Key Features:**

1. **Smart Scheduling:**
   - Cron runs on days 28-31 at 2:00 AM UTC
   - Job checks if tomorrow is day 1 (= today is last day)
   - Only executes on actual last day of month
   - Handles all month lengths: 28, 29, 30, 31 days

2. **Two-Tier Testing:**
   - **Standard (60s):** Quick stress validation
   - **Extended (300s):** Deep sustained-load testing
   - Extended only runs if standard passes

3. **Reporting:**
   - Generates markdown report with timestamps and results
   - Uploads as artifact (90-day retention)
   - Posts commit comment on failure
   - Links to workflow run for detailed logs

4. **Error Handling:**
   - `continue-on-error: true` allows both tests to run
   - Workflow fails if either test fails
   - Preserves logs and reports even on failure

5. **Manual Execution:**
   - `workflow_dispatch` trigger allows on-demand runs
   - Bypasses last-day check when manually triggered
   - Useful for testing before expected stress scenarios

### Example Report

```markdown
# Monthly Stress Test Report

**Date:** 2026-07-31 02:00:00 UTC
**Commit:** da72874...
**Branch:** main

## Results

### Standard Stress Test (60s per scenario)
Status: success

### Extended Stress Test (300s per scenario)
Status: success

## Summary

- ✅ Standard Test: PASS
- ✅ Extended Test: PASS

See workflow logs for detailed output.
```

---

## Recommended CI Pipeline

### Minimal (Small Teams)
```yaml
# Every commit
- boot-check
- test
- lint

# Monthly (automated)
- stress-test (standard + extended)
```

### Standard (Medium Teams)
```yaml
# Every commit
- boot-check
- test
- lint

# Pull Request
- test-comprehensive

# Monthly (automated)
- stress-test (standard + extended)
```

### Comprehensive (Large Teams/Production)
```yaml
# Every commit
- boot-check
- test
- lint

# Pull Request
- test-comprehensive
- code-review

# Nightly
- stress-test-extended
- test-smp-extended
- fuzz (syzkaller)

# Monthly (automated)
- stress-test (standard + extended)
- performance-tracking

# Release
- test-comprehensive
- stress-test-extended
- security-audit
```

---

## Environment Variables

### Stress Tests
```bash
STRESS_DURATION=60        # Duration in seconds per test (default: 60)
STRESS_CPUS=4             # CPU cores for SMP tests (default: 4)
STRESS_MEM=256M           # Memory size (default: 256M)
```

### Performance Tests
```bash
PERF_TIMEOUT=120          # Performance test timeout (default: 120)
REGRESSION_THRESHOLD=10   # Percentage slowdown limit (default: 10)
```

### SMP Tests
```bash
EXTENDED_SMP_TIMEOUT=90   # Extended SMP timeout (default: 90)
SMP_TEST_TIMEOUT=60       # Standard SMP timeout (default: 60)
```

---

## Troubleshooting CI Failures

### Stress Test Failures

**Symptom:** Monthly stress test fails with "UNSTABLE"

**Common Causes:**
- Memory leak in kernel code
- Deadlock under high contention
- Resource exhaustion not handled properly

**Action:**
1. Check workflow logs for panic location
2. Review serial output in artifacts
3. Run locally with same parameters: `STRESS_DURATION=300 make stress-test`
4. Use shorter duration for faster iteration: `STRESS_DURATION=30`

### Performance Test Issues

**Symptom:** Performance regression detected

**Common Causes:**
- Algorithm change increased complexity
- Lock contention introduced
- Cache behavior changed

**Action:**
1. Compare performance metrics in logs
2. Run `make test-perf` locally
3. Profile the changed code path
4. Consider if performance tradeoff is acceptable

### SMP Test Timeouts

**Symptom:** Extended SMP test exceeds timeout

**Common Causes:**
- Slow CI runners
- Scheduler issue under high core count
- AP initialization delay

**Action:**
1. Increase `EXTENDED_SMP_TIMEOUT` in workflow
2. Check if CPUs came online in serial log
3. Verify no deadlocks in scheduler code

---

## Cost Optimization

### GitHub Actions Minutes

**Estimated Usage:**
- Fast feedback: ~5 min × commits/day
- PR gate: ~20 min × PRs/month
- Monthly stress: ~120 min × 1/month

**Example (10 commits/day, 20 PRs/month):**
- Daily: 50 min (fast feedback)
- Monthly: 400 min (PR gates) + 120 min (stress) = 520 min/month
- Total: ~1,520 min/month (~25 hours)

**Optimization Tips:**
1. Cache Rust dependencies (`rust-cache@v2`)
2. Run stress tests only monthly (not nightly)
3. Use `workflow_dispatch` for manual runs instead of scheduled
4. Consider self-hosted runners for stress tests

---

## Self-Hosted Runners

For teams with heavy CI usage, consider self-hosted runners:

**Benefits:**
- No GitHub Actions minute limits
- Faster build times (persistent caches)
- Real hardware for melting tests
- Custom CPU configurations

**Setup:**
```bash
# On your build server
./config.sh --url https://github.com/yourorg/nilix --token YOUR_TOKEN
./run.sh

# In workflow YAML
runs-on: self-hosted
```

---

## Future Enhancements

### Planned CI Improvements

1. **Performance Tracking Database**
   - Store performance metrics over time
   - Visualize trends
   - Detect gradual regressions

2. **Nightly Extended Tests**
   - Run stress-test-extended nightly
   - Run fuzzing campaigns (8+ hours)
   - Test on multiple CPU configurations

3. **Real Hardware Integration**
   - Self-hosted runner with physical machine
   - Full melting test validation
   - IOMMU/hardware-specific tests

4. **Coverage Reporting**
   - Integrate KCOV coverage data
   - Track coverage trends
   - Identify untested code paths

---

## Summary

✅ **Implemented:**
- Monthly stress test workflow (automated)
- Manual trigger support
- Comprehensive reporting
- Smart scheduling for last day of month

✅ **Integration Points:**
- Fast feedback (existing CI)
- PR gate (test-comprehensive)
- Monthly validation (stress tests)
- Manual execution (all test categories)

✅ **Cost Efficient:**
- Stress tests run only once per month
- Can be manually triggered when needed
- Caching reduces build times
- Self-hosted option available

---

**Last Updated:** 2026-07-21  
**Workflow File:** `.github/workflows/monthly-stress-test.yml`  
**Status:** ✅ Operational and ready for deployment
