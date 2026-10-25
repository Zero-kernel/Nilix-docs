# Nilix Kernel Fuzz Coverage Audit

**Date:** 2026-08-04  
**Purpose:** Systematic assessment of fuzzing coverage across all kernel subsystems

## Executive Summary

**Current State:**
- 10 cargo-fuzz targets exist, but only 3 call real kernel code
- Stress gate contains placeholders waiting for Ring-3 infrastructure
- KCOV gate is a deterministic regression check, not active fuzzing
- Syzkaller integration referenced but not implemented

**Coverage Score:** 15% (3 of 20 major subsystems have active fuzz harnesses)

**Priority Actions:**
1. Convert 7 stub fuzz targets to real implementations
2. Add 12 new fuzz targets for uncovered subsystems
3. Implement syzkaller-style feedback-driven fuzzing
4. Upgrade stress gate to multi-workload scenarios

---

## Subsystem Coverage Matrix

| Subsystem | Component | Current Coverage | Real Code? | Priority | Target |
|-----------|-----------|------------------|------------|----------|--------|
| **Memory Management** | Buddy allocator | ❌ None | - | P0 | fuzz_buddy_allocator |
| | Page tables (PT ops) | ❌ None | - | P0 | fuzz_page_table_ops |
| | COW page fault | ❌ None | - | P1 | fuzz_cow_fault |
| | Demand paging | ❌ None | - | P1 | fuzz_demand_paging |
| | mmap/munmap/mprotect | 🟡 Stub only | No | P1 | fuzz_memory_ops (fix) |
| | brk heap | 🟡 Stub only | No | P1 | fuzz_memory_ops (fix) |
| | TLB shootdown | ❌ None | - | P2 | fuzz_tlb_shootdown |
| | Page cache | ❌ None | - | P2 | fuzz_page_cache |
| | DMA allocator | ❌ None | - | P3 | fuzz_dma |
| **Process/Scheduler** | Context switch | 🟡 Stub only | No | P0 | fuzz_scheduler (fix) |
| | Priority/affinity | 🟡 Stub only | No | P1 | fuzz_scheduler (fix) |
| | Fork/clone | ❌ None | - | P1 | fuzz_fork_clone |
| | Exec/load_elf | ✅ **REAL** | **Yes** | ✅ | fuzz_elf_loader |
| | Process teardown | ❌ None | - | P2 | fuzz_process_lifecycle |
| **IPC** | Futex operations | 🟡 Stub only | No | P0 | fuzz_futex_ops (fix) |
| | Pipes | ❌ None | - | P1 | fuzz_pipe_ops |
| | Signals (delivery) | 🟡 Stub only | No | P1 | fuzz_signal_delivery (fix) |
| | Signals (frame/SROP) | ❌ None | - | P1 | fuzz_signal_frame |
| | Shared memory | ❌ None | - | P2 | fuzz_shm |
| | Message queues | 🟡 Stub only | No | P3 | fuzz_ipc_message (fix) |
| **VFS** | Path normalization | ✅ **REAL** | **Yes** | ✅ | fuzz_vfs_path |
| | File I/O (read/write) | ❌ None | - | P1 | fuzz_file_io |
| | Directory ops | ❌ None | - | P1 | fuzz_dir_ops |
| | ramfs operations | ❌ None | - | P1 | fuzz_ramfs |
| | ext2 operations | ❌ None | - | P2 | fuzz_ext2 |
| | Mount namespace | ❌ None | - | P2 | fuzz_mount_ns |
| | Symlinks/hardlinks | ❌ None | - | P2 | fuzz_link_ops |
| **Network** | Packet parsing | ✅ **REAL** | **Yes** | ✅ | fuzz_network_packet |
| | TCP state machine | ❌ None | - | P1 | fuzz_tcp_state |
| | Socket operations | ❌ None | - | P1 | fuzz_socket_ops |
| | ARP cache | ❌ None | - | P2 | fuzz_arp_cache |
| | Routing table | ❌ None | - | P2 | fuzz_routing |
| | Network namespace | ❌ None | - | P2 | fuzz_net_ns |
| **Cgroups** | Resource limits | 🟡 Stub only | No | P1 | fuzz_cgroup_ops (fix) |
| | Process migration | 🟡 Stub only | No | P2 | fuzz_cgroup_ops (fix) |
| | Accounting | ❌ None | - | P2 | fuzz_cgroup_accounting |
| **Security** | Seccomp filters | ❌ None | - | P1 | fuzz_seccomp |
| | LSM policy | ❌ None | - | P2 | fuzz_lsm_policy |
| | Capability checks | ❌ None | - | P2 | fuzz_capabilities |
| | KCOV itself | ❌ None | - | P3 | fuzz_kcov |
| **Syscall** | Syscall dispatch | 🟡 Mock only | No | P0 | fuzz_syscall (fix) |
| | Argument validation | ❌ None | - | P1 | fuzz_syscall_args |
| | Usercopy | ❌ None | - | P1 | fuzz_usercopy |
| **Arch** | Interrupt handling | ❌ None | - | P2 | fuzz_interrupts |
| | Exception handling | ❌ None | - | P2 | fuzz_exceptions |
| | IOMMU operations | ❌ None | - | P3 | fuzz_iommu |

**Legend:**
- ✅ REAL: Active fuzz target calling production kernel code
- 🟡 Stub: Target file exists but contains empty test functions
- ❌ None: No fuzz target exists

---

## Priority 0 (Critical - Block 1.0)

These subsystems have the highest bug density from audit rounds and zero active fuzz coverage:

### 1. fuzz_buddy_allocator
**Gap:** Buddy allocator has no fuzz coverage despite R178-31 (boot-reserved COW), R180 heap budget fixes
**Attack surface:** Allocation size overflow, fragmentation, double-free, UAF
**Implementation:** Host-harness calling mm::allocate_frames, mm::free_frames with arbitrary sizes/counts

### 2. fuzz_page_table_ops
**Gap:** Page table manipulation has no coverage despite R174-B3 (PT-charge asymmetry), R168-1 (mprotect double-uncharge)
**Attack surface:** PTE flags, canonicality, NX violations, TLB consistency
**Implementation:** Host-harness simulating PT operations (map/unmap/change_prot) with fuzzed VAs/flags

### 3. fuzz_syscall (convert from mock)
**Gap:** Current fuzz_syscall uses MockKernelContext, not real syscall dispatch
**Attack surface:** Invalid syscall numbers, argument validation, privilege escalation
**Implementation:** QEMU-driven executor calling real syscall interface via Ring-3 harness

### 4. fuzz_futex_ops (convert from stub)
**Gap:** Empty test functions despite R172-08 (bucket TOCTOU), M0-5 SLICE 1b-1b (LOCK_PI EINTR)
**Attack surface:** Bucket corruption, lost wakeups, deadlock, priority inversion
**Implementation:** Multi-threaded harness with QEMU executor

---

## Priority 1 (High - Target 1.0)

### 5. fuzz_cow_fault
**Gap:** COW page fault path has no coverage despite R174-A4 (blocking lock)
**Attack surface:** Race conditions, charge accounting, shared PT corruption
**Implementation:** Multi-process harness triggering COW faults on shared pages

### 6. fuzz_memory_ops (convert from stub)
**Gap:** Current stub has validation logic but no real kernel calls
**Attack surface:** VMA overlaps, MAP_FIXED collisions, stack guard bypass
**Implementation:** QEMU executor calling real mmap/munmap/mprotect/brk syscalls

### 7. fuzz_scheduler (convert from stub)
**Gap:** Empty test functions despite R172-03 (steal-before-save), R172-05 (syscall_active leak)
**Attack surface:** Context switch corruption, priority inversion, runqueue corruption
**Implementation:** Multi-core QEMU executor with rapid task operations

### 8-14. [Additional P1 targets listed in matrix]

---

## Priority 2 (Medium - Post-1.0)

### 15-22. [P2 targets from matrix]

---

## Priority 3 (Low - Future)

### 23-30. [P3 targets from matrix]

---

## Implementation Phases

### Phase 1: Foundation (Week 1)
- ✅ **Audit document** (this file)
- Convert 3 critical stubs (fuzz_syscall, fuzz_futex_ops, fuzz_memory_ops)
- Add 2 new P0 targets (fuzz_buddy_allocator, fuzz_page_table_ops)
- Wire up basic syzkaller executor

### Phase 2: Breadth (Week 2)
- Convert remaining 4 stubs (scheduler, signal_delivery, cgroup_ops, ipc_message)
- Add 5 new P1 targets (COW, fork, pipe, file_io, TCP state)
- Enhance stress gate with multi-workload scenarios

### Phase 3: Depth (Week 3-4)
- Add 10 P2 targets (TLB, page cache, security subsystems)
- Implement syzkaller feedback loop
- Add corpus seeding and crash triage

### Phase 4: Hardening (Ongoing)
- Continuous fuzzing CI integration
- Coverage-guided corpus expansion
- Regression test generation from crashes

---

## Syzkaller Integration Design

### Architecture
```
┌─────────────┐
│ syz-manager │  (Host Linux, manages workers)
└──────┬──────┘
       │
       ├─────> syz-fuzzer (worker 1) ──> QEMU(Nilix) ──> syz-executor
       ├─────> syz-fuzzer (worker 2) ──> QEMU(Nilix) ──> syz-executor
       └─────> syz-fuzzer (worker N) ──> QEMU(Nilix) ──> syz-executor
```

### Components

**1. syz-executor (Ring-3 Nilix binary)**
- Embedded userspace program (similar to hello_musl.elf)
- Receives syscall programs via virtio-serial
- Executes syscalls with KCOV instrumentation enabled
- Reports coverage back to syz-fuzzer

**2. syz-fuzzer (Host process)**
- Generates syscall programs using grammar-based generation
- Launches QEMU instances with Nilix kernel
- Sends programs to syz-executor via virtio-serial
- Collects KCOV coverage data
- Mutates inputs based on coverage feedback

**3. syz-manager (Host coordinator)**
- Manages pool of syz-fuzzer workers
- Deduplicates crashes using stack hash
- Maintains corpus of interesting inputs
- Generates HTML reports

### Syscall Descriptions

Subset of Linux syscall grammar adapted for Nilix ABI:

```
# Memory management
mmap(addr vma, len len[addr], prot flags[mmap_prot], flags flags[mmap_flags], fd fd, offset fileoff)
munmap(addr vma, len len[addr])
mprotect(addr vma, len len[addr], prot flags[mmap_prot])
brk(brk ptr)

# Process management  
clone(flags flags[clone_flags], stack ptr, parent_tid ptr, child_tid ptr, tls ptr)
execve(file ptr[in, filename], argv ptr[in, array[ptr[in, string]]], envp ptr[in, array[ptr[in, string]]])
wait4(pid pid, status ptr[out, int32], options flags[wait_options], rusage ptr[out, rusage])

# IPC
futex(uaddr ptr[in, int32], op flags[futex_op], val int32, timeout ptr[in, timespec], uaddr2 ptr[in, int32], val3 int32)
pipe(fds ptr[out, pipe_fds])
kill(pid pid, sig signal)

# VFS
open(file ptr[in, filename], flags flags[open_flags], mode flags[open_mode])
read(fd fd, buf buffer[out], count len[buf])
write(fd fd, buf buffer[in], count len[buf])
close(fd fd)

# Network
socket(domain flags[socket_domain], type flags[socket_type], protocol int32)
bind(fd sock, addr ptr[in, sockaddr_storage], addrlen len[addr])
connect(fd sock, addr ptr[in, sockaddr_storage], addrlen len[addr])
sendto(fd sock, buf buffer[in], len len[buf], flags flags[send_flags], addr ptr[in, sockaddr_storage], addrlen len[addr])
recvfrom(fd sock, buf buffer[out], len len[buf], flags flags[recv_flags], addr ptr[out, sockaddr_storage], addrlen ptr[inout, len[addr, int32]])
```

### Coverage Collection

**KCOV Integration:**
- Kernel compiled with `--features kcov`
- Per-task coverage buffer (64KB ring)
- syz-executor enables KCOV via syscall(548, KCOV_INIT_TRACE)
- Coverage sent back as edge bitmap

**Instrumentation Points:**
- All kernel functions (rustc `-C instrument-coverage`)
- Syscall entry/exit
- Page fault handlers
- IRQ handlers (sampled, not exhaustive)

---

## Stress Gate Enhancement

### Current State
`heavy_stress.rs` contains 3 placeholder tests:
- HeavyTlbShootdownContentionTest - requires mmap/munmap
- RapidSignalMigrationContentionTest - requires fork/signal/migration
- NamespaceTeardownStormTest - requires unshare/kill

### Proposed Workloads

#### 1. Multi-Process Memory Churn (TLB Stress)
```rust
// 4 processes, 10K iterations each
// Each: mmap 4KB → write pattern → munmap → validate no corruption
// Target: R175 D0-CROSS-2 TLB shootdown fence
```

#### 2. Concurrent Fork/Exec Storm
```rust
// 100 forks, each execs a small ELF
// Target: Process lifecycle, PT accounting, exec-migration atomicity
```

#### 3. Futex Contention Matrix
```rust
// 8 threads, 4 futexes
// Mixed WAIT/WAKE/LOCK_PI operations
// Target: Bucket TOCTOU, lost wakeups, PI inversion
```

#### 4. VFS Metadata Storm
```rust
// Concurrent create/unlink/rename/stat on 1000 files
// Target: VFS locking, inode cache, dcache correctness
```

#### 5. Signal Delivery Flood
```rust
// 2 processes, rapidly send 10K signals while migrating
// Target: R175 D0-CROSS-1 cross-CPU signal frame corruption
```

#### 6. Network Packet Flood
```rust
// Send 10K packets to localhost while toggling interface up/down
// Target: Network namespace isolation, ARP cache races
```

#### 7. Cgroup Migration Storm
```rust
// Migrate processes between cgroups under memory pressure
// Target: Charge accounting, migration EBUSY gate
```

---

## Corpus Seeding Strategy

### Seed Sources

**1. Regression Tests**
- Extract syscall sequences from boot tests (kernel/src/runtime_tests/)
- Convert to syzkaller programs

**2. Known-Good Workloads**
- musl_check hello_musl execution trace
- Shell interactive session
- ext3 mount/read/write sequence

**3. Past Crashes**
- R-round findings converted to syzkaller reproducers
- Store in `fuzz/corpus/regressions/`

**4. Edge Cases**
- Boundary values (0, -1, MAX, NULL pointers)
- Alignment violations (non-page-aligned addresses)
- Race-inducing sequences (clone → yield)

### Corpus Format
```
fuzz/
  corpus/
    syscall/               # Generic syscall sequences
    memory/                # Memory-focused workloads
    ipc/                   # IPC-focused workloads
    network/               # Network-focused workloads
    regressions/           # Known crashes
      R172-01-clone-yield.syz
      R174-B3-pt-charge.syz
      R178-31-boot-reserved.syz
```

---

## Success Metrics

### Coverage Targets
- **Line coverage:** 60% of kernel code (currently ~40% via boot tests)
- **Branch coverage:** 50% (currently ~25%)
- **Subsystem coverage:** 80% of major subsystems (currently 15%)

### Bug Discovery
- **Goal:** Find 5+ HIGH or CRITICAL issues per fuzzing sprint
- **False positive rate:** <10% (verified crashes only)
- **Crash deduplication:** Stack hash + reproducer minimization

### Performance
- **Executions/sec:** 100+ per worker (syzkaller target)
- **Workers:** 4-8 parallel (scalable to 16+)
- **Corpus growth:** 10-20 new interesting inputs per hour

---

## Next Steps

1. **Immediate (This Session):**
   - ✅ Complete this audit document
   - Convert fuzz_syscall from mock to real QEMU executor
   - Add fuzz_buddy_allocator host harness
   - Add fuzz_page_table_ops host harness

2. **This Week:**
   - Convert remaining 6 stubs to real implementations
   - Implement basic syzkaller executor (Ring-3 binary)
   - Add 5 new P1 fuzz targets
   - Upgrade stress gate with 3 real multi-workload tests

3. **Next Week:**
   - Wire up syzkaller-manager → fuzzer → QEMU pipeline
   - Implement KCOV feedback loop
   - Seed initial corpus from boot tests
   - Run 24-hour continuous fuzzing campaign

4. **Ongoing:**
   - Integrate fuzzing into CI (nightly runs)
   - Triage and fix discovered crashes
   - Expand syscall grammar coverage
   - Add P2/P3 fuzz targets incrementally

---

## References

- **Audit Rounds:** R169-R180 (security/correctness findings)
- **Memory Lessons:** diagnosing-layout-dependent-heap-corruption.md
- **Fuzz Infrastructure:** kernel/fuzz/, userspace/fuzzer/
- **KCOV Gate:** make test-kcov, scripts/gates/qemu/fuzz_runner_test.sh
- **Syzkaller Docs:** https://github.com/google/syzkaller/tree/master/docs
