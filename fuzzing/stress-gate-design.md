# Stress Gate Enhancement Design

**Date:** 2026-08-04  
**Status:** Design Phase  
**Target:** Replace placeholder tests with real multi-workload scenarios

---

## Overview

The current stress gate (`kernel/src/runtime_tests/heavy_stress.rs`) contains three placeholder tests waiting for Ring-3 infrastructure. This document specifies seven concrete multi-workload scenarios that stress different kernel subsystems under sustained contention.

**Current State:**
- 3 placeholder tests returning `TestResult::Warning`
- No actual stress workloads executing
- Waiting for Ring-3 syscalls (mmap, fork, signal, etc.)

**Target State:**
- 7 real stress workloads
- Each runs for 30-60 seconds under multi-core contention
- Validates specific correctness properties (no data corruption, no deadlocks)
- CI integration via `make stress-test` and `make stress-test-extended`

---

## Workload Specifications

### Workload 1: TLB Shootdown Storm

**Target:** R175 D0-CROSS-2 TLB shootdown fence correctness

**Scenario:**
```
4 processes, each on a dedicated CPU core
Each process loop (10,000 iterations):
  1. mmap(NULL, 4KB, PROT_READ|PROT_WRITE, MAP_ANONYMOUS|MAP_PRIVATE)
  2. Write unique pattern (process_id ^ iteration_count)
  3. Read back and validate pattern
  4. munmap(addr, 4KB)
```

**Correctness Properties:**
- No pattern corruption (UAF would read wrong pattern)
- No segfaults (TLB not flushed would access unmapped page)
- All 40,000 allocations succeed
- Memory usage returns to baseline after test

**Ring-3 Syscalls Required:**
- `mmap(2)` - MAP_ANONYMOUS allocation
- `munmap(11)` - Deallocation triggering TLB shootdown
- `sched_setaffinity(203)` - Pin process to CPU

**Success Criteria:**
- 0 pattern mismatches
- 0 unexpected page faults
- Runtime < 120 seconds on 4-core system

---

### Workload 2: Concurrent Fork/Exec Storm

**Target:** Process lifecycle, PT accounting, exec-migration atomicity

**Scenario:**
```
Parent process loop (100 iterations):
  1. fork() - creates child
  2. Child execs tiny ELF (/bin/stress_exec_target.elf)
  3. Child writes marker to /tmp/fork-{pid}.done
  4. Child exits with code 42
  5. Parent wait4(child_pid) and validates exit code
  6. Parent reads marker file and deletes it
```

**Correctness Properties:**
- All 100 children exit with code 42
- All marker files written and readable
- No zombie processes after test
- No PT frame leaks (check memory.current before/after)

**Ring-3 Syscalls Required:**
- `fork(57)` or `clone(56)` with CLONE_VM=0
- `execve(59)` - Load and jump to new ELF
- `wait4(61)` - Reap child
- `open(2)`, `write(1)`, `close(3)` - Marker file

**Stress Exec Target ELF:**
```c
// userspace/stress_exec_target.c
int main(int argc, char **argv) {
    char path[64];
    snprintf(path, sizeof(path), "/tmp/fork-%d.done", getpid());
    int fd = open(path, O_CREAT | O_WRONLY, 0644);
    write(fd, "OK", 2);
    close(fd);
    return 42;
}
```

**Success Criteria:**
- All 100 children reaped with exit code 42
- All marker files present and contain "OK"
- No PT frame accounting drift

---

### Workload 3: Futex Contention Matrix

**Target:** R172-08 bucket TOCTOU, M0-5 SLICE 1b-1b LOCK_PI EINTR, lost wakeups

**Scenario:**
```
8 threads, 4 shared futexes (futex_addrs[4])
Each thread loop (5,000 iterations):
  1. Select random futex index (0-3)
  2. Select random operation:
     - 40%: FUTEX_WAIT (timeout 1ms)
     - 40%: FUTEX_WAKE (wake 1)
     - 10%: FUTEX_LOCK_PI (timeout 2ms)
     - 10%: FUTEX_UNLOCK_PI
  3. Track operation counts per thread
```

**Correctness Properties:**
- No deadlocks (all threads complete within 60 seconds)
- Wake count == Wait count ± 100 (some spurious wakes tolerated)
- No LOCK_PI without matching UNLOCK_PI
- No panics in futex bucket management

**Ring-3 Syscalls Required:**
- `futex(202)` - All operations (WAIT/WAKE/LOCK_PI/UNLOCK_PI)
- `clone(56)` with CLONE_VM=1 - Shared address space threads

**Success Criteria:**
- All threads complete within 60 seconds
- Wake/Wait balance within 2% tolerance
- No kernel panics

---

### Workload 4: VFS Metadata Storm

**Target:** VFS locking, inode cache, dcache correctness under contention

**Scenario:**
```
4 threads, shared directory /tmp/stress-vfs/
Each thread loop (1,000 iterations):
  1. Create file /tmp/stress-vfs/thread-{tid}-{iter}.tmp
  2. Write 128 bytes
  3. fsync()
  4. stat() to verify size
  5. Rename to /tmp/stress-vfs/thread-{tid}-{iter}.final
  6. unlink()
```

**Correctness Properties:**
- All 4,000 files created successfully
- All stat() calls return size=128
- All unlink() calls succeed
- Directory empty at end of test
- No inode leaks (check ramfs inode count)

**Ring-3 Syscalls Required:**
- `open(2)` with O_CREAT
- `write(1)` - 128 bytes
- `fsync(74)` - Force write
- `stat(4)` - Verify metadata
- `rename(82)` - Metadata update
- `unlink(87)` - Delete

**Success Criteria:**
- All files created, renamed, deleted
- Final directory empty
- No inode reference leaks

---

### Workload 5: Signal Delivery Flood Under Migration

**Target:** R175 D0-CROSS-1 cross-CPU signal frame corruption, M0-5 SLICE 1b-2 same-return delivery

**Scenario:**
```
2 processes:
  Process A: Install SIGUSR1 handler, loop forever incrementing counter
  Process B: Rapid signal sender + migrator

Thread B1 (signal sender):
  Loop (10,000 iterations):
    kill(process_a_pid, SIGUSR1)
    usleep(100)

Thread B2 (migrator):
  Loop (10,000 iterations):
    sched_setaffinity(process_a_pid, cpu=(iter % num_cpus))
    usleep(150)
```

**Correctness Properties:**
- Process A counter reaches ~10,000 (some signals may coalesce)
- No signal frame corruption (handler always sees valid RIP/RSP/RFLAGS)
- Process A never crashes with SIGSEGV
- All migrations succeed

**Ring-3 Syscalls Required:**
- `rt_sigaction(13)` - Install handler
- `kill(62)` - Send signal
- `sched_setaffinity(203)` - Migrate process
- `nanosleep(35)` - Delay

**Signal Handler (Process A):**
```c
static volatile sig_atomic_t signal_count = 0;

void sigusr1_handler(int sig, siginfo_t *info, void *ucontext) {
    signal_count++;
    
    // Validate context sanity
    ucontext_t *ctx = (ucontext_t *)ucontext;
    if (ctx->uc_mcontext.rip == 0 || ctx->uc_mcontext.rip >= 0xffff800000000000) {
        // Corrupted frame - would crash on return
        _exit(1);
    }
}
```

**Success Criteria:**
- signal_count >= 9,500 (allow up to 5% loss due to coalescing)
- Process A exits cleanly (not killed by SIGSEGV)
- No signal frame corruption detected

---

### Workload 6: Network Packet Flood with Interface Toggle

**Target:** Network namespace isolation, ARP cache races, packet queue consistency

**Scenario:**
```
2 threads:
  Thread 1 (packet sender):
    Loop (5,000 iterations):
      Send UDP packet to 127.0.0.1:9999
      usleep(200)

  Thread 2 (interface toggler):
    Loop (100 iterations):
      ip link set lo down
      usleep(500)
      ip link set lo up
      usleep(500)
```

**Correctness Properties:**
- No kernel panics during interface state changes
- No packet queue corruption (no UAF on packet buffers)
- Loopback interface returns to UP state at end
- No ARP cache deadlocks

**Ring-3 Syscalls Required:**
- `socket(41)` - Create UDP socket
- `sendto(44)` - Send packet
- `ioctl(16)` - SIOCSIFFLAGS for interface up/down

**Success Criteria:**
- At least 4,000 packets sent successfully (allow some failures during down state)
- Interface UP at end of test
- No kernel panics

---

### Workload 7: Cgroup Migration Under Memory Pressure

**Target:** R149-4 migration EBUSY gate, R171 mem_pinned accounting, charge consistency

**Scenario:**
```
2 cgroups (cg-A, cg-B), each with memory.max = 16MB
4 processes, initially in cg-A

Each process loop (1,000 iterations):
  1. Allocate 1MB via mmap()
  2. Write to all pages (force charge)
  3. Try to migrate to cg-B (may fail with EBUSY if limit hit)
  4. If migration succeeded, free 1MB
  5. If migration failed, retry after 10ms
```

**Correctness Properties:**
- All processes eventually migrate at least once
- No memory accounting underflows (memory.current never negative)
- No charge leaks (final memory.current matches allocated pages)
- No migration-induced deadlocks

**Ring-3 Syscalls Required:**
- `cgroup_attach(600)` - Migrate process to cgroup
- `mmap(9)` - Allocate memory
- `munmap(11)` - Free memory

**Success Criteria:**
- Each process migrates successfully at least 100 times
- Final memory.current matches expected (within 1 page tolerance)
- No charge accounting errors in kernel log

---

## Implementation Phases

### Phase 1: Ring-3 Infrastructure (Prerequisites)

**Status:** Mostly complete (M0 items 1-7 DONE per memory)

- [x] mmap/munmap/mprotect syscalls
- [x] fork/clone/execve syscalls
- [x] futex syscalls (WAIT/WAKE/LOCK_PI/UNLOCK_PI)
- [x] Signal delivery (kill/rt_sigaction/rt_sigreturn)
- [x] VFS operations (open/write/fsync/stat/rename/unlink)
- [x] Network syscalls (socket/sendto/recvfrom)
- [ ] Cgroup syscalls (cgroup_attach - needs implementation)

### Phase 2: Stress Test Harness

**File:** `userspace/stress_runner/src/main.rs`

```rust
// Stress test coordinator that runs all 7 workloads
fn main() {
    let config = StressConfig::from_env();
    
    let workloads = vec![
        Box::new(TlbShootdownWorkload::new()),
        Box::new(ForkExecWorkload::new()),
        Box::new(FutexContentionWorkload::new()),
        Box::new(VfsMetadataWorkload::new()),
        Box::new(SignalFloodWorkload::new()),
        Box::new(NetworkFloodWorkload::new()),
        Box::new(CgroupMigrationWorkload::new()),
    ];
    
    for workload in workloads {
        println!("=== Running: {} ===", workload.name());
        let result = workload.run(&config);
        match result {
            Ok(stats) => {
                println!("PASS: {}", workload.name());
                println!("Stats: {:?}", stats);
            }
            Err(e) => {
                println!("FAIL: {} - {}", workload.name(), e);
                std::process::exit(1);
            }
        }
    }
    
    println!("All stress workloads PASSED");
}
```

### Phase 3: Per-Workload Implementation

Each workload is a separate module:

```
userspace/stress_runner/src/
  main.rs
  workloads/
    mod.rs
    tlb_shootdown.rs
    fork_exec.rs
    futex_contention.rs
    vfs_metadata.rs
    signal_flood.rs
    network_flood.rs
    cgroup_migration.rs
```

### Phase 4: CI Integration

**Makefile target:**

```makefile
stress-test: build-stress ensure-ext3-image
	@echo "=== Running Stress Test Suite ==="
	@STRESS_DURATION=60 STRESS_CPUS=4 bash scripts/gates/stress/stress_test.sh "$(STRESS_ESP)"

stress-test-extended: build-stress ensure-ext3-image
	@echo "=== Running Extended Stress Test Suite ==="
	@STRESS_DURATION=300 STRESS_CPUS=4 bash scripts/gates/stress/stress_test.sh "$(STRESS_ESP)"
```

**GitHub Actions workflow:**

```yaml
- name: Run stress gate
  run: make stress-test
  timeout-minutes: 10
```

---

## Resource Requirements

| Workload | CPUs | Memory | Duration | Iterations |
|----------|------|--------|----------|------------|
| TLB Shootdown | 4 | 512MB | 60s | 10K per core |
| Fork/Exec | 1 | 256MB | 60s | 100 |
| Futex Contention | 4 | 128MB | 60s | 5K per thread |
| VFS Metadata | 4 | 256MB | 60s | 1K per thread |
| Signal Flood | 2 | 128MB | 60s | 10K signals |
| Network Flood | 2 | 256MB | 60s | 5K packets |
| Cgroup Migration | 4 | 512MB | 60s | 1K per process |

**Total runtime:** ~7 minutes for standard suite, ~35 minutes for extended

---

## Success Metrics

### Per-Workload
- Completion within timeout (no deadlocks)
- All correctness assertions pass
- No kernel panics or unexpected exceptions
- Memory accounting consistent

### Suite-Level
- All 7 workloads pass on single run
- Repeatable across 10 runs (no flakiness)
- Performance regression detection (runtime ±10% baseline)

---

## Failure Investigation

When a workload fails:

1. **Check kernel serial log:**
   - Panics, assertions, deadlock warnings
   - Memory accounting errors

2. **Check workload-specific correctness:**
   - Pattern mismatches (TLB workload)
   - Exit code mismatches (Fork/Exec workload)
   - Balance violations (Futex workload)

3. **Reproduce with extended duration:**
   ```bash
   STRESS_DURATION=600 make stress-test-extended
   ```

4. **Run workload in isolation:**
   ```bash
   ./stress_runner --workload tlb_shootdown --duration 300
   ```

5. **Enable debug tracing:**
   ```bash
   RUST_LOG=debug ./stress_runner
   ```

---

## References

- **Audit Findings:** R172 (futex bucket), R175 (TLB/signal), R171 (mem_pinned)
- **M0 Conformance:** Items 1-7 (Ring-3 syscalls)
- **Current Placeholders:** `kernel/src/runtime_tests/heavy_stress.rs`
- **Implementation Plan:** `docs/fuzzing/implementation-plan.md`
