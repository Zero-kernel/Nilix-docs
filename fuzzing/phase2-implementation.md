# Phase 2: Multi-Syscall Executor Implementation

**Date Started:** 2026-07-21  
**Status:** 🚧 IN PROGRESS

---

## Overview

Phase 2 builds the foundational execution engine that can:
1. Run sequences of syscalls
2. Track results (success/failure/crash)
3. Provide basic repeatability (fixed seed)
4. Integrate with KCOV to measure coverage

This phase does NOT include:
- Sophisticated argument generation (that's Phase 3)
- Coverage-guided mutation (that's Phase 4)
- Resource tracking (that's Phase 5)

---

## Goals

### Primary Goals
1. ✅ Complete KCOV syscall implementations (already done in Phase 1)
2. 🚧 Build a simple executor that can run syscall sequences
3. 🚧 Integrate with KCOV to measure coverage per run
4. 🚧 Basic result verification (did it crash? did it succeed?)
5. 🚧 Repeatability (same sequence = same result with fixed seed)

### Stretch Goals
- Simple random argument generation (integers, pointers, flags)
- Basic corpus: save interesting sequences
- CLI tool to run executor

---

## Architecture

### Executor Design

```
┌─────────────────────────────────────────────┐
│           Fuzzing Executor (Rust)           │
│  • Reads syscall sequences from stdin/file  │
│  • Forks child processes to isolate crashes │
│  • Measures coverage via KCOV               │
│  • Reports results: OK/FAIL/CRASH/TIMEOUT   │
└─────────────────────────────────────────────┘
                    │
                    ▼
         ┌──────────────────────┐
         │   Child Process      │
         │  1. kcov_init        │
         │  2. kcov_enable      │
         │  3. run syscalls     │
         │  4. kcov_dump        │
         │  5. exit             │
         └──────────────────────┘
```

### Syscall Sequence Format (Simple)

**Phase 2 uses a dead-simple format:**

```
# Lines starting with # are comments
# Format: syscall_num arg0 arg1 arg2 arg3 arg4 arg5
# All arguments are integers (pointers are just addresses)

# Example: open("/tmp/foo", O_RDWR|O_CREAT, 0644)
2 0x1000 66 420 0 0 0

# Example: write(fd=3, buf=0x2000, len=100)
1 3 0x2000 100 0 0 0

# Example: close(fd=3)
3 3 0 0 0 0 0
```

**Why so simple?**
- Phase 2 is about execution mechanics, not smart generation
- Phase 3 will add proper syscall descriptions
- This format is easy to parse and debug

---

## Implementation Plan

### Step 1: Executor Framework (Rust) ✅ NEXT

**File:** `tools/fuzz_executor/src/main.rs`

Core functionality:
- Parse syscall sequences from file
- Fork child process
- Execute syscalls in child
- Collect coverage from child
- Report results to parent

**Key operations:**
```rust
struct SyscallSpec {
    num: u64,
    args: [u64; 6],
}

fn execute_sequence(seq: &[SyscallSpec]) -> ExecutionResult {
    // 1. kcov_init(4096)
    // 2. kcov_enable()
    // 3. Execute each syscall
    // 4. kcov_dump()
    // 5. Return coverage + status
}
```

### Step 2: Basic Result Verification

**Outcomes:**
- `OK`: All syscalls completed, no crash
- `CRASH`: Process died (SIGSEGV, SIGABRT, etc.)
- `TIMEOUT`: Exceeded time limit
- `KCOV_FAIL`: KCOV syscalls failed

### Step 3: Integration with KCOV

**Flow:**
```
Parent                  Child
  |                      |
  fork() --------------> |
  |                      kcov_init(4096)
  |                      kcov_enable()
  |                      execute syscall 1
  |                      execute syscall 2
  |                      ...
  |                      kcov_disable()
  |                      kcov_dump(buf, 4096)
  |                      exit(edge_count)
  wait() <-------------- |
  read coverage          |
  report results         |
```

### Step 4: CLI Tool

```bash
# Run a single sequence
./fuzz_executor run sequence.txt

# Run in loop (10000 iterations, random mutations)
./fuzz_executor loop --iterations 10000

# Verify a crashing sequence
./fuzz_executor verify crash-001.txt
```

---

## Acceptance Criteria

### Phase 2 Complete When:

1. ✅ **KCOV syscalls work** (verified in Phase 1)
2. ⏳ **Executor can run syscall sequences from file**
3. ⏳ **Executor reports: OK/CRASH/TIMEOUT**
4. ⏳ **Executor integrates with KCOV and reports edge counts**
5. ⏳ **At least one example sequence runs successfully**
6. ⏳ **Documentation updated with executor usage**

---

## Current Status

### Completed
- ✅ KCOV module implementation (Phase 1)
- ✅ Syscall interface 520-524 (Phase 1)
- ✅ Test program `kcov_test.c` (Phase 1)
- ✅ Boot verification shows KCOV init (Phase 1)

### In Progress
- 🚧 Executor framework design
- 🚧 Syscall sequence parser

### Blocked
- None

---

## Next Actions

1. Create `tools/fuzz_executor/` directory structure
2. Implement syscall sequence parser
3. Implement basic executor (no fork yet)
4. Add fork isolation
5. Integrate KCOV
6. Test with example sequences

---

## Example Sequences for Testing

### Sequence 1: Simple KCOV Test
```
# Initialize KCOV
520 4096 0 0 0 0 0

# Enable coverage
521 0 0 0 0 0 0

# Do some syscalls
# getpid()
39 0 0 0 0 0 0

# Disable coverage
522 0 0 0 0 0 0
```

### Sequence 2: File Operations
```
# open("/tmp/test", O_RDWR|O_CREAT, 0644)
2 0x10000000 66 420 0 0 0

# write(fd, "hello", 5)
1 3 0x10000100 5 0 0 0

# read(fd, buf, 5)
0 3 0x10000200 5 0 0 0

# close(fd)
3 3 0 0 0 0 0
```

---

## Design Decisions

### Why Rust for the Executor?

**Pros:**
- Type safety prevents many bugs
- Great for parsing and error handling
- Can easily call libc syscalls
- Integrates well with the kernel

**Cons:**
- Slightly more complex than C
- Requires `unsafe` for raw syscalls

**Decision:** Use Rust. The safety benefits outweigh the complexity.

### Why Fork Isolation?

Each sequence runs in a fresh child process to:
- Isolate crashes (parent survives)
- Reset state (no leftover fds, memory, etc.)
- Enable parallel execution later

### Why Simple Format in Phase 2?

Phase 3 will add a proper syscall description language. Phase 2's simple format:
- Easy to parse
- Easy to hand-write test cases
- Easy to debug
- Sufficient for basic execution mechanics

---

## Timeline Estimate

- **Step 1 (Executor framework):** 4-6 hours
- **Step 2 (Result verification):** 2-3 hours
- **Step 3 (KCOV integration):** 3-4 hours
- **Step 4 (CLI tool):** 2-3 hours
- **Testing & Documentation:** 2-3 hours

**Total:** 13-19 hours (1.5-2.5 days)

---

## Notes

- Keep it simple: Phase 2 is about execution mechanics, not smart fuzzing
- Phase 3 will add proper syscall descriptions
- Phase 4 will add coverage-guided mutation
- Don't overengineer: the executor will evolve significantly in later phases
