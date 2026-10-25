# Phase 4 Architecture: Coverage-Guided Mutation

**Date:** 2026-07-21  
**Status:** Design phase

---

## Overview

Phase 4 implements **coverage-guided mutation** with **corpus management**, transforming the Phase 3 random fuzzer into a smart fuzzer that learns from coverage feedback and prioritizes inputs that discover new code paths.

---

## Goals

### Primary Goals

1. **Corpus management** - Store and retrieve test cases that increase coverage
2. **Coverage-guided mutation** - Prioritize inputs that hit new edges
3. **Seed corpus** - Hand-crafted base cases for bootstrapping
4. **Mutator strategies** - Multiple mutation techniques
5. **Energy scheduling** - Prioritize high-value inputs

### Success Criteria

- Corpus grows from seed inputs over fuzzing runs
- Coverage increases beyond Phase 3 baseline (>25 edges)
- Mutation strategies generate valid inputs
- Energy scheduling improves efficiency
- Baseline coverage documented and exceeded

---

## Architecture

### Component Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Coverage-Guided Fuzzer                     │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────┐      ┌──────────────┐      ┌────────────┐  │
│  │   Corpus    │─────▶│   Scheduler  │─────▶│  Executor  │  │
│  │  Manager    │      │   (Energy)   │      │   (KCOV)   │  │
│  └─────────────┘      └──────────────┘      └────────────┘  │
│         │                     │                     │         │
│         │                     │                     ▼         │
│         │              ┌──────────────┐      ┌────────────┐  │
│         └─────────────▶│   Mutator    │      │  Coverage  │  │
│                        │   Engine     │      │  Feedback  │  │
│                        └──────────────┘      └────────────┘  │
│                               │                     │         │
│                               └─────────────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow

1. **Scheduler** selects input from corpus based on energy
2. **Mutator** generates variant of selected input
3. **Executor** runs mutated input, collects coverage via KCOV
4. **Feedback** compares coverage to known edges
5. **Corpus** saves input if new coverage discovered

---

## Component Design

### 1. Corpus Manager

**Purpose:** Store and retrieve test cases that increase coverage

**Data Structure:**
```rust
pub struct CorpusEntry {
    /// Syscall sequence
    pub sequence: Vec<Syscall>,
    
    /// Coverage edges hit by this input
    pub edges: Vec<u32>,
    
    /// Number of unique edges
    pub unique_edges: usize,
    
    /// Execution time (for energy calculation)
    pub exec_time_us: u64,
    
    /// Number of times selected for mutation
    pub selection_count: u32,
    
    /// Total descendants (mutants) generated
    pub descendant_count: u32,
    
    /// Number of descendants that found new coverage
    pub productive_descendants: u32,
    
    /// Energy (priority for selection)
    pub energy: f32,
}

pub struct Corpus {
    /// All test cases
    entries: Vec<CorpusEntry>,
    
    /// Global coverage bitmap (union of all entries)
    global_edges: HashSet<u32>,
    
    /// Total unique edges discovered
    total_edges: usize,
}
```

**Operations:**
- `add(entry) -> bool` - Add if new coverage, return true if added
- `select() -> &CorpusEntry` - Select entry based on energy
- `update_energy()` - Recalculate energy for all entries
- `get_coverage() -> &HashSet<u32>` - Get global edge set

**Storage Strategy:**
- In-memory corpus for Phase 4
- Persistent corpus (Phase 5) will save to disk
- Size limit: 1000 entries (cull oldest/least-productive)

### 2. Energy Scheduler

**Purpose:** Prioritize high-value inputs for mutation

**Energy Calculation:**
```rust
fn calculate_energy(entry: &CorpusEntry) -> f32 {
    let mut energy = 1.0;
    
    // Boost: fewer selections = more energy
    energy *= 1.0 / (1.0 + entry.selection_count as f32).sqrt();
    
    // Boost: more unique edges = more energy
    energy *= entry.unique_edges as f32 / 10.0;
    
    // Boost: higher productivity = more energy
    let productivity = if entry.descendant_count > 0 {
        entry.productive_descendants as f32 / entry.descendant_count as f32
    } else {
        1.0  // New entries get full boost
    };
    energy *= productivity;
    
    // Penalty: longer execution = less energy
    energy /= (entry.exec_time_us as f32 / 1000.0).max(1.0);
    
    energy
}
```

**Selection Strategy:**
- Weighted random selection based on energy
- Update energy after every 100 mutations
- Adaptive: shifts focus as corpus evolves

### 3. Mutator Engine

**Purpose:** Generate variants of corpus entries

**Mutation Strategies:**

#### A. Bit Flip Mutations
```rust
// Flip single bit in argument
fn mutate_bit_flip(syscall: &mut Syscall) {
    let arg_idx = rand() % syscall.args.len();
    let bit_idx = rand() % 64;
    syscall.args[arg_idx] ^= 1u64 << bit_idx;
}
```

#### B. Arithmetic Mutations
```rust
// Add/subtract small value
fn mutate_arithmetic(syscall: &mut Syscall) {
    let arg_idx = rand() % syscall.args.len();
    let delta = (rand() % 35) - 17;  // [-17, +17]
    syscall.args[arg_idx] = syscall.args[arg_idx].wrapping_add(delta);
}
```

#### C. Interesting Value Substitution
```rust
// Replace with known interesting value
const INTERESTING_VALUES: &[u64] = &[
    0, 1, u64::MAX,  // Boundary values
    0x7fffffff, 0x80000000,  // i32 boundaries
    0xffffffff, 0x100000000,  // u32 boundaries
    4096, 8192, 16384,  // Common sizes
];

fn mutate_interesting_value(syscall: &mut Syscall) {
    let arg_idx = rand() % syscall.args.len();
    syscall.args[arg_idx] = INTERESTING_VALUES[rand() % INTERESTING_VALUES.len()];
}
```

#### D. Dictionary-Based Substitution
```rust
// Use values from grammar corpus
fn mutate_dictionary(syscall: &mut Syscall, grammar: &Grammar) {
    let arg_idx = rand() % syscall.args.len();
    let constraint = &grammar.constraints[syscall.args[arg_idx].constraint_id];
    syscall.args[arg_idx] = sample_from_constraint(constraint);
}
```

#### E. Syscall Insertion
```rust
// Insert new syscall into sequence
fn mutate_insert_syscall(sequence: &mut Vec<Syscall>, grammar: &Grammar) {
    let pos = rand() % (sequence.len() + 1);
    let new_syscall = generate_random_syscall(grammar);
    sequence.insert(pos, new_syscall);
}
```

#### F. Syscall Deletion
```rust
// Remove syscall from sequence
fn mutate_delete_syscall(sequence: &mut Vec<Syscall>) {
    if sequence.len() > 1 {
        let pos = rand() % sequence.len();
        sequence.remove(pos);
    }
}
```

#### G. Syscall Duplication
```rust
// Duplicate existing syscall
fn mutate_duplicate_syscall(sequence: &mut Vec<Syscall>) {
    let src_pos = rand() % sequence.len();
    let dst_pos = rand() % (sequence.len() + 1);
    let syscall = sequence[src_pos].clone();
    sequence.insert(dst_pos, syscall);
}
```

#### H. Syscall Reordering
```rust
// Swap two syscalls
fn mutate_reorder(sequence: &mut Vec<Syscall>) {
    if sequence.len() >= 2 {
        let pos1 = rand() % sequence.len();
        let pos2 = rand() % sequence.len();
        sequence.swap(pos1, pos2);
    }
}
```

**Mutation Selection:**
- 40% argument mutations (bit flip, arithmetic, interesting values)
- 30% dictionary-based (grammar-guided)
- 20% sequence mutations (insert, delete, duplicate)
- 10% reordering

**Mutation Budget:**
- 1-3 mutations per input
- Deterministic stage: all bit flips first
- Havoc stage: random mutations after

### 4. Executor

**Purpose:** Run mutated input and collect coverage

**Integration with KCOV:**
```rust
fn execute_input(sequence: &[Syscall]) -> ExecutionResult {
    // Initialize KCOV
    syscall(SYS_KCOV_INIT, 4096);
    
    // Enable coverage
    syscall(SYS_KCOV_ENABLE);
    
    // Execute sequence
    let start = read_timestamp();
    for syscall in sequence {
        execute_syscall(syscall);
    }
    let end = read_timestamp();
    
    // Disable and dump
    syscall(SYS_KCOV_DISABLE);
    let mut edges = [0u32; 256];
    let edge_count = syscall(SYS_KCOV_DUMP, edges.as_mut_ptr(), edges.len() * 4);
    
    // Reset for next iteration
    syscall(SYS_KCOV_RESET);
    
    ExecutionResult {
        edges: edges[..edge_count].to_vec(),
        exec_time_us: end - start,
    }
}
```

**Error Handling:**
- Syscall failures are not fatal (fuzzer continues)
- Timeout after 100ms per syscall (detect hangs)
- Crash detection (Phase 5)

### 5. Coverage Feedback

**Purpose:** Detect new coverage and guide mutation

**Edge Comparison:**
```rust
fn has_new_coverage(edges: &[u32], global_edges: &HashSet<u32>) -> (bool, Vec<u32>) {
    let mut new_edges = Vec::new();
    
    for &edge in edges {
        if !global_edges.contains(&edge) {
            new_edges.push(edge);
        }
    }
    
    (!new_edges.is_empty(), new_edges)
}
```

**Coverage Metrics:**
- **Absolute coverage:** Total unique edges discovered
- **Coverage rate:** % of 37 instrumented edges
- **Coverage gain:** New edges per N mutations
- **Saturation:** Iterations since last new edge

---

## Seed Corpus

### Purpose

Bootstrap fuzzer with hand-crafted test cases that:
- Hit different syscall categories
- Exercise common patterns
- Cover expected success paths

### Seed Test Cases

**Seed 1: Simple I/O**
```rust
[
    Syscall::write(1, "hello", 5),
    Syscall::read(0, buf, 16),
]
// Expected: 4 edges (write entry/stdout, read entry/stdin)
```

**Seed 2: Query Syscalls**
```rust
[
    Syscall::getpid(),
    Syscall::getuid(),
    Syscall::getgid(),
]
// Expected: 3 edges (one per syscall)
```

**Seed 3: Memory Operations**
```rust
[
    Syscall::brk(0),  // Query
    Syscall::mmap(0, 4096, PROT_READ, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0),
]
// Expected: 3 edges (brk entry/query, mmap entry/anonymous)
```

**Seed 4: File Operations**
```rust
[
    Syscall::open("/test", O_RDONLY, 0),
    Syscall::close(3),  // Assume fd=3
]
// Expected: 4 edges (open entry/internal, close entry/callback)
```

**Seed 5: Mixed Sequence**
```rust
[
    Syscall::getpid(),
    Syscall::write(1, "test", 4),
    Syscall::brk(0),
    Syscall::getuid(),
]
// Expected: 5 edges
```

**Total seed corpus:** 5 entries, ~19 expected edges (≈Phase 3 baseline)

---

## Implementation Plan

### File Structure

```
userspace/
├── fuzzer/
│   ├── corpus.rs          Corpus management
│   ├── scheduler.rs       Energy scheduling
│   ├── mutator.rs         Mutation strategies
│   ├── executor.rs        KCOV integration
│   ├── feedback.rs        Coverage analysis
│   ├── seeds.rs           Seed corpus
│   └── main.rs            Fuzzer main loop
│
docs/fuzzing/
├── phase4-architecture.md This document
├── phase4-implementation.md (TBD)
└── phase4-results.md      (TBD)
```

### Main Fuzzer Loop

```rust
fn main() {
    // Initialize
    let mut corpus = Corpus::new();
    let mut mutator = Mutator::new();
    let grammar = load_grammar();
    
    // Load seed corpus
    for seed in SEEDS {
        let result = execute_input(&seed);
        corpus.add(CorpusEntry {
            sequence: seed,
            edges: result.edges,
            unique_edges: result.edges.len(),
            exec_time_us: result.exec_time_us,
            ...
        });
    }
    
    println!("Seed corpus: {} entries, {} edges", 
        corpus.entries.len(), corpus.total_edges);
    
    // Fuzzing loop
    for iteration in 0..MAX_ITERATIONS {
        // Update energy every 100 iterations
        if iteration % 100 == 0 {
            corpus.update_energy();
        }
        
        // Select input
        let entry = corpus.select();
        
        // Mutate
        let mutant = mutator.mutate(&entry.sequence, &grammar);
        
        // Execute
        let result = execute_input(&mutant);
        
        // Check for new coverage
        let (has_new, new_edges) = has_new_coverage(&result.edges, corpus.get_coverage());
        
        if has_new {
            corpus.add(CorpusEntry {
                sequence: mutant,
                edges: result.edges.clone(),
                unique_edges: result.edges.len(),
                exec_time_us: result.exec_time_us,
                ...
            });
            
            println!("[+] Iteration {}: {} new edges (total: {})", 
                iteration, new_edges.len(), corpus.total_edges);
        }
        
        // Stats every 1000 iterations
        if iteration % 1000 == 0 {
            print_stats(&corpus, iteration);
        }
    }
    
    // Final report
    print_final_report(&corpus);
}
```

---

## Performance Targets

### Phase 4 Goals vs Phase 3 Baseline

| Metric | Phase 3 Baseline | Phase 4 Target |
|--------|------------------|----------------|
| **Unique edges** | 15-20 (random) | >25 (>125% of baseline) |
| **Coverage rate** | 40-54% | >67% |
| **Corpus size** | N/A | 50-100 entries |
| **Iterations** | 5 | 10,000+ |
| **Time to 90% coverage** | N/A | <1000 iterations |

### Success Criteria

- ✅ Coverage exceeds Phase 3 baseline by ≥25%
- ✅ Corpus grows from 5 seeds to 50+ entries
- ✅ At least 5 new edges discovered beyond random baseline
- ✅ Energy scheduling demonstrably improves efficiency
- ✅ Multiple mutation strategies contribute new coverage

---

## Testing & Validation

### Unit Tests

- Mutator produces valid syscalls
- Corpus correctly tracks global coverage
- Energy calculation is deterministic
- Feedback correctly identifies new edges

### Integration Tests

- Fuzzer runs for 1000 iterations without crash
- Coverage increases over first 100 iterations
- Corpus size grows appropriately
- Energy scheduling affects selection distribution

### Performance Tests

- Execution rate: >100 iterations/second
- Memory usage: <100 MB for 1000 entry corpus
- Coverage saturation: reach 90% in <10 minutes

---

## Comparison to AFL/libFuzzer

| Feature | AFL | libFuzzer | Nilix Phase 4 |
|---------|-----|-----------|---------------|
| Coverage feedback | ✓ | ✓ | ✓ (KCOV) |
| Corpus management | ✓ | ✓ | ✓ |
| Energy scheduling | ✓ | ✓ | ✓ |
| Bit flip mutations | ✓ | ✓ | ✓ |
| Dictionary | ✓ | ✓ | ✓ (grammar) |
| Deterministic stage | ✓ | ✗ | ✓ |
| Havoc stage | ✓ | ✓ | ✓ |
| Splicing | ✓ | ✓ | Phase 5 |
| Crash detection | ✓ | ✓ | Phase 5 |
| Persistent mode | ✗ | ✓ | ✓ (always) |

**Verdict:** Phase 4 matches AFL/libFuzzer core features.

---

## Next Steps

1. Implement corpus management
2. Implement mutation strategies
3. Implement energy scheduler
4. Integrate with KCOV executor
5. Create seed corpus
6. Run baseline comparison
7. Document results

**Timeline:** 1-2 weeks

---

**Phase 4 Architecture:** ✅ Complete  
**Next:** Implementation
