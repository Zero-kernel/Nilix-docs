# Phase 7 Architecture: CI Integration & Continuous Fuzzing

**Date:** 2026-07-21  
**Status:** 🚧 IN PROGRESS

---

## Overview

Phase 7 delivers **continuous fuzzing infrastructure** that runs 24/7 in CI, automatically triages crashes, synchronizes corpus across runs, and provides real-time dashboards.

This is the final phase of the syzkaller-inspired fuzzing infrastructure for Nilix.

---

## Goals

### Primary Objectives
1. **Continuous fuzzing** - Fuzzer runs 24/7 in CI without manual intervention
2. **Automatic crash triage** - Crashes are deduplicated, minimized, and filed automatically
3. **Corpus synchronization** - Corpus shared across multiple fuzzing instances
4. **Performance dashboards** - Real-time visibility into fuzzing progress
5. **CI/CD integration** - Seamless integration with GitHub Actions

### Success Criteria
- [ ] Fuzzer runs continuously in GitHub Actions
- [ ] Crashes automatically deduplicated by stack trace
- [ ] Corpus persisted and synchronized across runs
- [ ] Dashboard shows coverage, crashes, corpus size
- [ ] Integration with existing CI pipeline (no conflicts)
- [ ] Documentation for maintainers and contributors

---

## Architecture

### System Components

```
┌─────────────────────────────────────────────────────────────┐
│                     GitHub Actions CI                        │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Fuzzer 1   │  │   Fuzzer 2   │  │   Fuzzer N   │      │
│  │  (Worker)    │  │  (Worker)    │  │  (Worker)    │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
│         │                  │                  │               │
│         └──────────────────┴──────────────────┘               │
│                            │                                  │
│                    ┌───────▼────────┐                        │
│                    │  Corpus Store  │                        │
│                    │  (Git Repo /   │                        │
│                    │   Artifacts)   │                        │
│                    └───────┬────────┘                        │
│                            │                                  │
│         ┌──────────────────┴──────────────────┐              │
│         │                                      │              │
│  ┌──────▼────────┐                   ┌────────▼────────┐    │
│  │ Crash Triage  │                   │   Dashboard     │    │
│  │ (Dedup +      │                   │   (Metrics +    │    │
│  │  Minimize)    │                   │    Reports)     │    │
│  └───────────────┘                   └─────────────────┘    │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## Component Specifications

### 1. Continuous Fuzzing Worker

**Purpose:** Run fuzzing loop continuously in CI

**Implementation:**
```rust
// userspace/fuzzer/continuous.rs

pub struct ContinuousFuzzer {
    corpus: Corpus,
    coverage: StatefulCoverage,
    crash_dir: PathBuf,
    stats: FuzzingStats,
    config: FuzzingConfig,
}

pub struct FuzzingConfig {
    max_iterations: Option<usize>,  // None = infinite
    timeout_per_input: Duration,    // Per-input timeout
    corpus_dir: PathBuf,
    crash_dir: PathBuf,
    stats_interval: Duration,       // Stats reporting interval
}

impl ContinuousFuzzer {
    pub fn run(&mut self) -> FuzzingResult {
        loop {
            // 1. Select input from corpus (coverage-guided)
            let input = self.corpus.select_interesting();
            
            // 2. Mutate input
            let mutated = self.mutate(input);
            
            // 3. Execute on kernel
            let result = self.execute(mutated);
            
            // 4. Classify result
            match result {
                ExecutionResult::Crash(crash) => {
                    self.handle_crash(crash);
                }
                ExecutionResult::NewCoverage(cov) => {
                    self.corpus.add(mutated, cov);
                }
                ExecutionResult::Normal => {
                    // Discard
                }
            }
            
            // 5. Update stats
            self.stats.iteration += 1;
            if self.stats.should_report() {
                self.report_stats();
            }
            
            // 6. Check exit condition
            if self.should_stop() {
                break;
            }
        }
        
        FuzzingResult {
            iterations: self.stats.iteration,
            crashes: self.stats.crashes,
            coverage: self.coverage.clone(),
        }
    }
}
```

**Key features:**
- Coverage-guided input selection
- Crash detection and persistence
- Periodic stats reporting
- Configurable exit conditions

---

### 2. Crash Triage System

**Purpose:** Deduplicate crashes and minimize reproducer

**Implementation:**
```rust
// userspace/fuzzer/crash_triage.rs

pub struct CrashTriageSystem {
    seen_crashes: BTreeMap<CrashSignature, CrashInfo>,
    minimizer: Minimizer,
}

#[derive(Clone, PartialEq, Eq, PartialOrd, Ord)]
pub struct CrashSignature {
    crash_type: CrashType,      // Panic, PageFault, TripleFault, etc.
    panic_message: Option<String>,
    stack_trace_hash: u64,      // Hash of first 5 stack frames
}

pub struct CrashInfo {
    signature: CrashSignature,
    first_seen: SystemTime,
    count: usize,
    reproducer: Vec<Syscall>,   // Minimized reproducer
    reproducer_minimized: bool,
}

impl CrashTriageSystem {
    pub fn process_crash(&mut self, crash: Crash) -> TriageResult {
        // 1. Extract signature
        let signature = self.extract_signature(&crash);
        
        // 2. Check if duplicate
        if let Some(existing) = self.seen_crashes.get_mut(&signature) {
            existing.count += 1;
            return TriageResult::Duplicate {
                signature: signature.clone(),
                first_seen: existing.first_seen,
                count: existing.count,
            };
        }
        
        // 3. New crash - minimize reproducer
        let minimized = self.minimizer.minimize(
            crash.input.clone(),
            FailCondition::CrashSignature(signature.clone())
        );
        
        // 4. Store crash info
        let info = CrashInfo {
            signature: signature.clone(),
            first_seen: SystemTime::now(),
            count: 1,
            reproducer: minimized,
            reproducer_minimized: true,
        };
        
        self.seen_crashes.insert(signature.clone(), info);
        
        // 5. Return new crash
        TriageResult::NewCrash {
            signature,
            reproducer: minimized,
        }
    }
    
    fn extract_signature(&self, crash: &Crash) -> CrashSignature {
        // Extract crash type from serial output
        let crash_type = if crash.output.contains("PANIC") {
            CrashType::Panic
        } else if crash.output.contains("Page Fault") {
            CrashType::PageFault
        } else if crash.output.contains("Triple Fault") {
            CrashType::TripleFault
        } else {
            CrashType::Unknown
        };
        
        // Extract panic message (if any)
        let panic_message = self.extract_panic_message(&crash.output);
        
        // Hash stack trace (first 5 frames)
        let stack_trace_hash = self.hash_stack_trace(&crash.output);
        
        CrashSignature {
            crash_type,
            panic_message,
            stack_trace_hash,
        }
    }
}
```

**Deduplication strategy:**
- Crash type (panic, page fault, triple fault)
- Panic message (if present)
- Stack trace hash (first 5 frames)

**Minimization:**
- Use Phase 6 minimizer to reduce reproducer
- Delta debugging ensures minimal test case

---

### 3. Corpus Synchronization

**Purpose:** Share corpus across multiple fuzzing instances

**Implementation:**
```rust
// userspace/fuzzer/corpus_sync.rs

pub struct CorpusSyncManager {
    local_corpus: Corpus,
    remote_corpus_dir: PathBuf,
    sync_interval: Duration,
    last_sync: SystemTime,
}

impl CorpusSyncManager {
    pub fn sync(&mut self) -> Result<SyncStats, SyncError> {
        let now = SystemTime::now();
        if now.duration_since(self.last_sync).unwrap() < self.sync_interval {
            return Ok(SyncStats::default());
        }
        
        let mut stats = SyncStats::default();
        
        // 1. Pull new inputs from remote
        let remote_inputs = self.list_remote_inputs()?;
        for input in remote_inputs {
            if !self.local_corpus.contains(&input) {
                self.local_corpus.add_from_remote(input)?;
                stats.pulled += 1;
            }
        }
        
        // 2. Push new local inputs to remote
        let local_new = self.local_corpus.get_new_since(self.last_sync);
        for input in local_new {
            self.push_to_remote(input)?;
            stats.pushed += 1;
        }
        
        self.last_sync = now;
        Ok(stats)
    }
}

pub struct SyncStats {
    pub pulled: usize,  // Inputs downloaded from remote
    pub pushed: usize,  // Inputs uploaded to remote
}
```

**Storage options:**
1. **Git repository** - Corpus as text files in separate repo
2. **GitHub Artifacts** - Upload/download via Actions artifacts API
3. **S3-compatible storage** - If available

**Synchronization strategy:**
- Every N minutes (configurable)
- Before worker shutdown
- After finding new crash

---

### 4. Performance Dashboard

**Purpose:** Real-time visibility into fuzzing progress

**Implementation:**
```rust
// userspace/fuzzer/dashboard.rs

pub struct FuzzingDashboard {
    stats: FuzzingStats,
    start_time: SystemTime,
}

pub struct FuzzingStats {
    pub iterations: usize,
    pub crashes: usize,
    pub unique_crashes: usize,
    pub corpus_size: usize,
    pub edge_coverage: usize,
    pub state_coverage: usize,
    pub transaction_coverage: usize,
    pub exec_per_sec: f64,
}

impl FuzzingDashboard {
    pub fn generate_report(&self) -> String {
        let runtime = SystemTime::now()
            .duration_since(self.start_time)
            .unwrap()
            .as_secs();
        
        format!(
            "=== Fuzzing Dashboard ===\n\
             Runtime: {}h {}m {}s\n\
             Iterations: {}\n\
             Exec/sec: {:.2}\n\
             \n\
             Corpus: {} inputs\n\
             Coverage: {} edges, {} states, {} transactions\n\
             \n\
             Crashes: {} total ({} unique)\n\
             ",
            runtime / 3600,
            (runtime % 3600) / 60,
            runtime % 60,
            self.stats.iterations,
            self.stats.exec_per_sec,
            self.stats.corpus_size,
            self.stats.edge_coverage,
            self.stats.state_coverage,
            self.stats.transaction_coverage,
            self.stats.crashes,
            self.stats.unique_crashes,
        )
    }
    
    pub fn generate_html_report(&self) -> String {
        // HTML dashboard with charts
        // (Can use simple HTML + CSS, no JS needed for CI)
        todo!()
    }
}
```

**Metrics tracked:**
- Runtime
- Iterations (total syscall sequences executed)
- Exec/sec (executions per second)
- Corpus size
- Coverage (edges, states, transactions)
- Crashes (total and unique)

**Output formats:**
- Text (for CI logs)
- HTML (for GitHub Pages or artifacts)
- JSON (for programmatic access)

---

### 5. GitHub Actions Integration

**Purpose:** Run fuzzer in CI continuously

**Implementation:**
```yaml
# .github/workflows/fuzzing.yml

name: Continuous Fuzzing

on:
  # Trigger options
  schedule:
    - cron: '0 */6 * * *'  # Every 6 hours
  workflow_dispatch:       # Manual trigger
  push:
    branches:
      - main
    paths:
      - 'kernel/**'
      - 'userspace/fuzzer/**'

jobs:
  fuzz:
    runs-on: ubuntu-latest
    timeout-minutes: 360  # 6 hours max
    
    strategy:
      matrix:
        worker: [1, 2, 3, 4]  # 4 parallel workers
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Rust
        uses: actions-rs/toolchain@v1
        with:
          toolchain: nightly-2025-12-08
          components: rust-src
      
      - name: Install QEMU
        run: |
          sudo apt-get update
          sudo apt-get install -y qemu-system-x86 ovmf
      
      - name: Restore corpus
        uses: actions/cache@v3
        with:
          path: corpus/
          key: fuzzing-corpus-${{ github.sha }}
          restore-keys: |
            fuzzing-corpus-
      
      - name: Build KCOV kernel
        run: make build-kcov
      
      - name: Run fuzzer
        run: |
          cargo run --release --bin continuous-fuzzer -- \
            --corpus-dir corpus/ \
            --crash-dir crashes/ \
            --max-time 21600 \
            --worker-id ${{ matrix.worker }}
      
      - name: Generate report
        if: always()
        run: |
          cargo run --release --bin fuzzing-dashboard -- \
            --stats-dir stats/ \
            --output-html dashboard.html
      
      - name: Upload crashes
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: crashes-${{ matrix.worker }}
          path: crashes/
      
      - name: Upload dashboard
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: dashboard-${{ matrix.worker }}
          path: dashboard.html
      
      - name: Save corpus
        if: always()
        uses: actions/cache@v3
        with:
          path: corpus/
          key: fuzzing-corpus-${{ github.sha }}
      
      - name: Create issue for new crashes
        if: always()
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            const crashFiles = fs.readdirSync('crashes/new/');
            
            for (const file of crashFiles) {
              const crash = JSON.parse(fs.readFileSync(`crashes/new/${file}`));
              
              await github.rest.issues.create({
                owner: context.repo.owner,
                repo: context.repo.repo,
                title: `[Fuzzing] ${crash.signature.crash_type}: ${crash.signature.panic_message}`,
                body: `## Crash Details\n\n` +
                      `- **Type:** ${crash.signature.crash_type}\n` +
                      `- **Message:** ${crash.signature.panic_message}\n` +
                      `- **Stack trace hash:** ${crash.signature.stack_trace_hash}\n\n` +
                      `## Reproducer\n\n` +
                      `\`\`\`rust\n${crash.reproducer}\n\`\`\`\n\n` +
                      `Found by continuous fuzzing worker ${matrix.worker}.`,
                labels: ['fuzzing', 'bug', 'needs-triage']
              });
            }
```

**Key features:**
- Scheduled runs (every 6 hours)
- Manual trigger support
- Parallel workers (4 instances)
- Corpus caching
- Crash artifact upload
- Automatic issue creation

---

## Crash Workflow

### 1. Detection
```
Fuzzer executes input → Kernel crashes → QEMU exits abnormally
                                       ↓
                              Fuzzer detects crash
                                       ↓
                              Save crash info
```

### 2. Triage
```
Extract crash signature:
  - Crash type (panic, page fault, triple fault)
  - Panic message (if present)
  - Stack trace hash (first 5 frames)
                     ↓
Check if duplicate → YES → Increment count, skip
                     ↓
                    NO
                     ↓
Minimize reproducer (Phase 6 minimizer)
                     ↓
Save to crashes/new/ directory
```

### 3. Reporting
```
CI job finishes → Upload crashes/ as artifact
                          ↓
GitHub Action creates issue for each new crash
                          ↓
Issue contains:
  - Crash signature
  - Minimized reproducer
  - Stack trace
  - Labels: fuzzing, bug, needs-triage
```

---

## Corpus Management

### Corpus Structure
```
corpus/
├── inputs/
│   ├── 0001-baseline.toml        # From existing tests
│   ├── 0002-coverage-A5F3.toml   # Found by fuzzer (coverage A5F3)
│   ├── 0003-coverage-B7C2.toml
│   └── ...
├── metadata/
│   ├── 0001.meta                  # Coverage info
│   ├── 0002.meta
│   └── ...
└── stats.json                     # Corpus statistics
```

### Input Format (TOML)
```toml
# corpus/inputs/0042-coverage-D3E1.toml

[[syscall]]
id = 2  # sys_open
args = ["/tmp/test", 0x42, 0o644]

[[syscall]]
id = 1  # sys_write
args = [3, "data", 4]

[[syscall]]
id = 3  # sys_close
args = [3]
```

### Metadata Format (JSON)
```json
{
  "input_id": "0042-coverage-D3E1",
  "discovered": "2026-07-21T10:30:45Z",
  "worker_id": 2,
  "coverage": {
    "edges": 127,
    "states": 15,
    "transactions": 3
  },
  "execution_time_us": 450
}
```

---

## Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| **Exec/sec** | 100+ | Per worker |
| **Corpus growth** | +50 inputs/hour | Initially, slows over time |
| **Coverage growth** | +5% every 24h | Initially, plateaus |
| **Crash dedup rate** | >95% | Avoid duplicate reports |
| **Minimization ratio** | 70%+ | Reduce reproducer size |
| **Worker utilization** | >90% | Minimize idle time |

---

## Implementation Plan

### Step 1: Continuous Fuzzer Loop
- [ ] Implement `ContinuousFuzzer` struct
- [ ] Integration with Phase 4-6 modules
- [ ] Configurable exit conditions
- [ ] Stats tracking and reporting

### Step 2: Crash Triage
- [ ] Crash signature extraction from serial output
- [ ] Deduplication by signature
- [ ] Integration with Phase 6 minimizer
- [ ] Crash persistence (JSON format)

### Step 3: Corpus Sync
- [ ] Corpus directory structure
- [ ] Input/metadata serialization (TOML + JSON)
- [ ] Sync manager with pull/push
- [ ] GitHub Artifacts integration

### Step 4: Dashboard
- [ ] Stats aggregation
- [ ] Text report generation
- [ ] HTML report with charts
- [ ] JSON export for programmatic access

### Step 5: CI Integration
- [ ] GitHub Actions workflow
- [ ] Parallel workers
- [ ] Corpus caching
- [ ] Crash artifact upload
- [ ] Automatic issue creation

### Step 6: Documentation
- [ ] Maintainer guide
- [ ] Contributor guide
- [ ] Dashboard interpretation
- [ ] Troubleshooting

---

## Security Considerations

### Corpus Security
- **Sanitization:** Inputs are syscall sequences, not arbitrary data
- **Size limits:** Prevent corpus growth attack (max 10,000 inputs)
- **Validation:** All inputs validated before execution

### CI Security
- **Sandboxing:** Fuzzer runs in QEMU (isolated from host)
- **Resource limits:** Timeout per worker (6 hours max)
- **Rate limiting:** Max 4 workers per run
- **Secrets:** No secrets needed (public project)

### Crash Handling
- **No data leaks:** Crashes don't contain sensitive data
- **Public issues:** All crashes filed as public issues (research project)
- **Reproducibility:** Minimized reproducers included in issues

---

## Monitoring & Alerting

### Health Checks
- [ ] Worker heartbeat (stats every 5 minutes)
- [ ] Coverage growth (alert if stalled >24h)
- [ ] Crash rate (alert if >10 crashes/hour)
- [ ] Corpus size (alert if >10,000 inputs)

### Alerts
- **Email:** On first unique crash
- **GitHub issue:** On new crash (automatic)
- **Status badge:** Show fuzzing health in README

---

## Future Enhancements (Post-Phase 7)

### Distributed Fuzzing
- Run fuzzer on multiple machines
- Central corpus server
- Load balancing

### Smart Scheduling
- Prioritize recent code changes
- Focus on low-coverage areas
- A/B testing of mutation strategies

### Integration with syzkaller
- Import syzkaller syscall descriptions
- Export Nilix findings to syzkaller format
- Cross-OS comparison

### Machine Learning
- Learn effective mutation strategies
- Predict interesting inputs
- Crash clustering

---

## Success Metrics

Phase 7 is **complete** when:
- [x] Fuzzer runs continuously in GitHub Actions
- [x] Crashes deduplicated automatically
- [x] Corpus synchronized across runs
- [x] Dashboard shows real-time metrics
- [x] Documentation complete
- [x] At least one real bug found and filed

---

## Timeline

**Estimated:** 1-2 weeks

| Task | Duration | Dependencies |
|------|----------|--------------|
| Continuous fuzzer loop | 2 days | Phase 4-6 |
| Crash triage system | 2 days | Phase 6 minimizer |
| Corpus sync | 1 day | - |
| Dashboard | 1 day | Stats tracking |
| CI integration | 2 days | All above |
| Documentation | 1 day | All above |
| Testing & polish | 2 days | All above |

---

## Conclusion

Phase 7 completes the syzkaller-inspired fuzzing infrastructure for Nilix. After this phase:

- **Automated bug finding:** Fuzzer runs 24/7 without manual intervention
- **Efficient triage:** Crashes deduplicated and minimized automatically
- **Continuous improvement:** Corpus grows, coverage increases over time
- **Developer-friendly:** Clear dashboards, automatic issue creation
- **Research-grade:** Comparable to Linux syzkaller

The Nilix kernel will have a **production-ready, continuously running fuzzing system** that finds bugs before they reach users.

---

**Next:** Implementation of continuous fuzzer loop (Step 1)
