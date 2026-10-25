# Fuzzing Schedule — 2026-07-22

## Overview

Nilix fuzzing runs on three different schedules optimized for cost and coverage:

| Trigger | Duration | Workers | Frequency | Purpose |
|---------|----------|---------|-----------|---------|
| **Push** | 5 minutes | 4 | Every push to main | Quick smoke test for fast feedback |
| **Weekly** | 1 hour | 4 | Sunday 2 AM UTC | Comprehensive coverage testing |
| **Daily cargo-fuzz** | 10 minutes | 10 targets | Daily 2 AM UTC | LibFuzzer target testing |

---

## Schedule Details

### 1. Quick Push Fuzzing (5 minutes)

**Trigger:** Any push to `main` that modifies:
- `kernel/**`
- `userspace/fuzzer/**`
- `fuzz/**`
- `.github/workflows/fuzz.yml`

**Configuration:**
```yaml
on:
  push:
    branches:
      - main
    paths:
      - 'kernel/**'
      - 'userspace/fuzzer/**'
      - 'fuzz/**'
      - '.github/workflows/fuzz.yml'
```

**Runtime:**
- Duration: 300 seconds (5 minutes)
- Workers: 4 parallel workers
- Report interval: 60 seconds
- Timeout: 30 minutes (includes build time)

**Purpose:**
- Fast feedback loop for developers
- Catch obvious crashes immediately
- Low CI cost per push
- Does not block merge (runs asynchronously)

**Expected behavior:**
- Each worker runs ~5 minutes of fuzzing
- Reports every 60 seconds
- Minimal corpus growth (quick iterations)
- Catches only the most obvious bugs

---

### 2. Weekly Comprehensive Fuzzing (1 hour)

**Trigger:** Scheduled cron job

```yaml
schedule:
  - cron: '0 2 * * 0'  # Sunday 2 AM UTC
```

**Runtime:**
- Duration: 3600 seconds (1 hour)
- Workers: 4 parallel workers
- Report interval: 60 seconds
- Timeout: 400 minutes (6h + overhead)

**Purpose:**
- Deep coverage exploration
- Find edge cases and rare bugs
- Build substantial corpus
- Weekly regression testing

**Expected behavior:**
- Each worker runs full 1 hour
- Substantial corpus growth
- Higher chance of finding rare bugs
- Full crash triage and GitHub issue creation

---

### 3. Daily Cargo-Fuzz Targets (10 minutes)

**Trigger:** Scheduled cron job

```yaml
schedule:
  - cron: '0 2 * * *'  # Daily 2 AM UTC
```

**Runtime:**
- Duration: 600 seconds (10 minutes per target)
- Targets: 10 parallel libFuzzer targets
- Timeout: 30 minutes per target

**Targets:**
- `fuzz_syscall`
- `fuzz_vfs_path`
- `fuzz_signal_delivery`
- `fuzz_memory_ops`
- `fuzz_ipc_message`
- `fuzz_scheduler`
- `fuzz_network_packet`
- `fuzz_cgroup_ops`
- `fuzz_elf_loader`
- `fuzz_futex_ops`

**Purpose:**
- Test specific kernel subsystems
- Structured input fuzzing with libFuzzer
- Complement KCOV-based fuzzing
- Daily regression check

---

## Dynamic Duration Logic

The workflow automatically adjusts duration based on trigger:

```bash
if [ "${{ github.event_name }}" == "push" ]; then
  duration_secs=300  # 5 minutes
elif [ "${{ github.event.schedule }}" == "0 2 * * 0" ]; then
  duration_secs=3600  # 1 hour
else
  duration_secs=${{ github.event.inputs.duration || '6' }}  # Manual input
fi
```

---

## Manual Workflow Dispatch

The workflow can be manually triggered with custom parameters:

```yaml
workflow_dispatch:
  inputs:
    mode:
      - continuous  # KCOV-based fuzzing
      - targets     # Cargo-fuzz libFuzzer
      - both        # Run both modes
    duration:
      default: '6'  # Hours for continuous mode
    timeout:
      default: '600'  # Seconds per cargo-fuzz target
    workers:
      default: '4'  # Parallel workers for continuous mode
```

**Use cases:**
- Test specific subsystem before merge
- Extended fuzzing session (6+ hours)
- Reproduce crash conditions
- Benchmark fuzzing performance

---

## Cost Optimization

### Before (every 6 hours)

- 4 runs/day × 6 hours × 4 workers = 96 worker-hours/day
- High GitHub Actions cost
- Overkill for most pushes

### After (split schedule)

| Schedule | Frequency | Duration | Workers | Daily cost |
|----------|-----------|----------|---------|-----------|
| Push | ~5-10/day | 5 min | 4 | 0.33-0.67 hours |
| Weekly | 1/week | 1 hour | 4 | 0.57 hours/day |
| Daily cargo-fuzz | 1/day | 10 min | 10 | 1.67 hours |
| **Total** | - | - | - | **~2.5 hours/day** |

**Savings:** 96 hours/day → 2.5 hours/day = **97.4% cost reduction**

While maintaining:
- ✅ Fast push feedback (5 min)
- ✅ Weekly deep testing (1 hour)
- ✅ Daily subsystem coverage (10 min)

---

## Artifacts and Retention

| Artifact | Retention | Trigger |
|----------|-----------|---------|
| Crashes | 90 days | All runs |
| Corpus | 7 days | All runs |
| Fuzzer logs | 7 days | All runs |
| Dashboards | 90 days | All runs |
| Triage reports | 90 days | All runs |
| Aggregate report | 90 days | All runs |

---

## GitHub Issue Creation

Crashes found by the fuzzer automatically create GitHub issues:

**Conditions:**
1. Crash triage job completes
2. Unique crashes > 0 (after deduplication)
3. Each unique crash gets one issue

**Issue format:**
```
Title: [Fuzzing] <crash_type>: <crash_message>
Labels: fuzzing, bug, needs-triage
Body: Full crash report with signature, stack trace, input
```

**Example:**
```
Title: [Fuzzing] Panic: assertion failed in vfs::lookup
Labels: fuzzing, bug, needs-triage
Body:
Type: Panic
Message: assertion failed: inode.is_valid()
Signature: vfs_lookup_panic_line_342
Original: /path/to/crash-file
...
```

---

## Monitoring

### Step Summary

Every run generates a step summary visible in the Actions UI:

```markdown
# Comprehensive Fuzzing Report

**Run:** 1234567890
**Date:** 2026-07-22 02:00:00 UTC
**Trigger:** schedule

## Continuous Fuzzing (KCOV-based)

- **Worker 1:** 50000 iterations, 1200 corpus, 450 edges, 0 crashes
- **Worker 2:** 49800 iterations, 1180 corpus, 445 edges, 0 crashes
- **Worker 3:** 50200 iterations, 1230 corpus, 460 edges, 1 crashes
- **Worker 4:** 50100 iterations, 1210 corpus, 455 edges, 0 crashes

**Totals:**
- Iterations: 200100
- Corpus size: 4820
- Max coverage: 460 edges
- Crashes found: 1

## Cargo-Fuzz Targets

✓ No crashes found across all cargo-fuzz targets

## Crash Triage

- **Total crashes:** 1
- **Unique crashes:** 1
- **Deduplication rate:** 0.0%
- **GitHub issues created:** 1
```

### Artifacts

Download from Actions UI → Artifacts:
- `continuous-crashes-worker-N` (90 days)
- `continuous-dashboard-worker-N` (90 days)
- `continuous-corpus-worker-N` (7 days)
- `cargo-fuzz-crashes-<target>` (30 days)
- `triaged-crashes` (90 days)
- `aggregate-report` (90 days)

---

## Future Improvements

### Phase 3+
- Syscall descriptions
- Coverage-guided mutation
- Resource tracking
- Stateful fuzzing
- Longer weekly runs (6-12 hours)

### Infrastructure
- Distributed fuzzing across multiple runners
- Corpus synchronization between workers
- Coverage diffing between runs
- Crash bisection for root cause

---

## Rationale

### Why 5 minutes for push?

- Fast feedback (results in ~10 minutes including build)
- Catches obvious regressions
- Low cost per push
- Developers see results quickly

### Why 1 hour for weekly?

- Balance between coverage and cost
- Enough time to explore edge cases
- Fits within free GitHub Actions quota
- Can be extended manually if needed

### Why Sunday 2 AM UTC?

- Low traffic time (weekend)
- Before Monday work week starts
- Consistent with daily cargo-fuzz schedule
- Easy to remember and monitor

---

## Configuration Files

| File | Purpose |
|------|---------|
| `.github/workflows/fuzz.yml` | Main workflow definition |
| `userspace/fuzzer/.cargo/config.toml` | Force Linux target for fuzzer binaries |
| `userspace/fuzzer/Cargo.toml` | Fuzzer dependencies |
| `userspace/fuzzer/src/bin/continuous-fuzzer.rs` | KCOV-based fuzzer |
| `userspace/fuzzer/src/bin/crash-triage.rs` | Crash deduplication tool |

---

**Last updated:** 2026-07-22  
**Related:** Phase 2 KCOV implementation  
**Status:** Active and running  
**Next review:** After Phase 3 implementation
