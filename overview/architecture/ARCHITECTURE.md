# Nilix Kernel Architecture Map

> **See first:** [`docs/architecture.md`](../../architecture.md) for the readable
> composition overview, the Mermaid crate dependency DAG, the boot flow, the syscall path,
> and the core component summary. This document is the deep reference — lock ordering,
> critical code paths, and the historical R166–R172 audit-finding record.
> **Note:** the status block and "Last 5 Audit Rounds" below were last refreshed 2026-07-30;
> for current audit numbers (R187 KCOV 2026-08-08; R188 standalone audit remediated
> 2026-08-27) see [`docs/security-audit-status.md`](../../security-audit-status.md).

**Version:** 1.2  
**Date:** 2026-07-30  
**Status:** Authoritative architecture documentation for reconstruction  
**Based on:** 146 Rust files across 25 kernel crates, 187 R-series audit rounds, ~1,184 findings remediated (plus the 2026-08-07 standalone full-codebase audit)

---

## Executive Summary

Nilix (formerly Zero-OS) is a capability-first, safety-hardened x86-64 microkernel implementing a three-tier security model:

1. **IRQ-context safety** (lock discipline and IRQ-masked locking; the `sync_safe` runtime detection is dormant since R192-8)
2. **Memory safety** (Rust + W^X enforcement + KPTI)
3. **Privilege isolation** (SMEP/SMAP + namespaces + cgroups)

**Current State (2026-07-30):**
- **Development Phase:** Phase U user-mode ABI plus post-R186 hardening
- **Security Posture:** 0 CRITICAL open, 1 HIGH open (`R186-4`), 0 open MEDIUM; `R186-18` fixed and review-verified
- **Audit Coverage:** 187 R-series rounds, ~1,340 filed IDs, ~1,184 fixed/resolved (current numbers in [`docs/security-audit-status.md`](../../security-audit-status.md))
- **0-HIGH Streak:** 0/3 (R186 reset the streak)
- **Gate Status:** 1.0-Preview implementation/design blockers are `R186-4` and its
  aggregate-admission design parent; the independent zero-HIGH streak is also unsatisfied at 0/3
- **Environment Status:** final eight-file ReviewFix payload uploaded with local/remote SHA-256
  parity; focused capability/IPC/kernel/net checks, net default-parallel 110/110, fmt-check,
  clippy, build, lint, make test, boot-check, and musl-check all passed

**Design Principles:**
- Safety > Efficiency > Speed (ZeroLang hierarchy)
- Fail-closed on resource exhaustion
- Lock ordering hierarchical (10 documented levels)
- Atomic critical sections under minimal locks
- Defense-in-depth across hardware/software layers

---

## Table of Contents

1. [Subsystem Catalog](#subsystem-catalog)
2. [Lock Ordering Hierarchy](#lock-ordering-hierarchy)
3. [Security Architecture](#security-architecture)
4. [Memory Safety Boundaries](#memory-safety-boundaries)
5. [Concurrency Model](#concurrency-model)
6. [IRQ-Context Constraints](#irq-context-constraints)
7. [Critical Code Paths](#critical-code-paths)
8. [Dependency Graph](#dependency-graph)
9. [High-Risk Areas for Audit](#high-risk-areas-for-audit)
10. [Known Audit Findings](#known-audit-findings)

---

## Subsystem Catalog

### Layer 0: Hardware Abstraction (arch)

**Primary Responsibilities:**
- CPU feature detection (SMAP, SMEP, NX, IBRS/STIBP)
- GDT/IDT/TSS per-CPU initialization
- Context switching (save/restore RIP/RFLAGS/segments/FPU)
- KPTI trampoline setup (dual-root CR3 isolation)
- Syscall MSR configuration (LSTAR/CSTAR for Ring 3→0 entry)
- SMP bring-up (ACPI MADT parsing, AP initialization)
- IPI dispatch (reschedule, TLB shootdown)
- INVPCID TLB operations (single-context, all-nonglobal)

**Files:** 11 files
- `arch/lib.rs` - Module root, CPU feature detection
- `arch/context_switch.rs` - **CRITICAL** save_context/switch_context (R172-01 RIP/RFLAGS fix)
- `arch/syscall.rs` - Syscall entry/exit, CR3 switch, FS/GS restore (R172-04 fix)
- `arch/interrupts.rs` - IDT setup, #PF handler, timer IRQ
- `arch/gdt.rs` - GDT/TSS per-CPU setup
- `arch/smp.rs` - AP boot, ACPI MADT parsing
- `arch/ipi.rs` - Inter-processor interrupts
- `arch/invpcid.rs` - TLB shootdown primitives
- `arch/apic.rs` - LAPIC/IOAPIC initialization (R171-1 IOMMU critical fix)
- `arch/hpet.rs` - High-precision timer
- `arch/cpu_protection.rs` - SMAP/SMEP/NX detection

**Lock Types:** None (per-CPU data, no synchronization)

**IRQ Context:** Mixed (entry/exit run with IRQs off; some operations safe in IRQ)

**Security Boundaries:**
- KPTI enforces kernel/user page-table separation
- SMEP blocks kernel execution of user pages
- SMAP blocks kernel access to user memory (requires explicit STAC/CLAC)
- Syscall entry validates user RIP/RSP canonical + low-half

**Known Issues:**
- R172-01 [CRITICAL, FIXED]: `save_context` omitted RIP/RFLAGS on first switch
- R172-04 [HIGH, FIXED]: FS_BASE/GS_BASE leak across switch_context branch
- R171-2 [CRITICAL, FIXED]: AP syscall MSRs never programmed

---

### Layer 1: Core Services (kernel_core)

**Primary Responsibilities:**
- Process/thread lifecycle (fork, exec, exit, clone)
- Namespace management (PID, IPC, mount, net, user)
- Signal delivery (rt_sigaction, rt_sigprocmask, rt_sigreturn)
- Syscall dispatch (300+ syscalls, fail-closed macros)
- User copy primitives (copy_from_user, copy_to_user with SMAP)
- File descriptor table management
- Capability table integration (Phase U preview)
- Cgroup resource accounting
- LSM context provider callbacks

**Files:** 15 files
- `kernel_core/lib.rs` - Module root, public API
- `kernel_core/process.rs` - **CRITICAL** Process struct, lifecycle, ~4,500 lines
- `kernel_core/syscall.rs` - **CRITICAL** Syscall dispatcher, ~6,800 lines
- `kernel_core/fork.rs` - fork_inner, COW setup, cgroup charging
- `kernel_core/cgroup.rs` - **CRITICAL** Cgroup v2 implementation, ~2,100 lines
- `kernel_core/signal.rs` - Signal handling (M0-5 SLICE 1a/1b/3)
- `kernel_core/signal_frame.rs` - rt_sigframe layout, SROP defense
- `kernel_core/elf_loader.rs` - ELF parsing, PT_LOAD mapping
- `kernel_core/user_stack.rs` - Auxv builder (M0-1)
- `kernel_core/usercopy.rs` - SMAP-aware copy primitives
- `kernel_core/{pid,ipc,net,mount,user}_namespace.rs` - Namespace isolation
- `kernel_core/rcu.rs` - Read-Copy-Update synchronization
- `kernel_core/time.rs` - Timestamp tracking
- `kernel_core/exception_table.rs` - Kernel fault handling
- `kernel_core/syscall_macros.rs` - Fail-closed dispatch helpers

**Lock Types:**
- `Arc<Mutex<Process>>` - Per-process lock (Level 4)
- `Mutex<HashMap<ProcessId, Arc<Mutex<Process>>>>` - Global process table (Level 4)
- `RwLock<CapTable>` - Per-process capability table (Level 4)

**IRQ Context:** Mostly IRQ-unsafe (syscalls block, memory allocates)

**Security Boundaries:**
- Usercopy validates user pointers (USER_SPACE_TOP, page-accessible)
- KPTI enforces kernel/user separation
- Namespace isolation prevents cross-namespace PID/IPC access
- Cgroup limits enforce resource quotas (CPU, memory, PIDs, files, ports)

**Critical Invariants:**
- Process lock NEVER held across page-table ops, memory allocs, or I/O
- Fork charging: worst-case COW footprint pre-charged (R138-1)
- Exec: EBUSY multithread/CLONE_VM check FIRST (unbypassable)
- Signal delivery: SROP-defended (canonical RIP/RSP, RFLAGS sanitize)

**Known Issues:**
- R172-01 [CRITICAL, FIXED]: Context-switch omitted RIP/RFLAGS
- R172-03 [HIGH, FIXED]: Schedule marked Ready before save (steal race)
- R172-18 [MEDIUM, DEFERRED]: Futex PI/robust ABI (COW deadlock hazard)

---

### Layer 1: Memory Management (mm)

**Primary Responsibilities:**
- Physical frame allocation (buddy allocator)
- Page table management (map/unmap, COW, TLB shootdown)
- Page cache with reclaim/writeback
- DMA buffer management
- OOM killer (process selection & termination)
- MMIO mapping for device regions

**Files:** 9 files
- `mm/lib.rs` - Module root, public API
- `mm/buddy_allocator.rs` - **CRITICAL** Physical frame allocator, ~800 lines
- `mm/page_table.rs` - **CRITICAL** Page table operations, COW, ~1,200 lines
- `mm/memory.rs` - Address space management
- `mm/page_cache.rs` - Page cache, writeback
- `mm/dma.rs` - DMA buffer pools
- `mm/oom_killer.rs` - Out-of-memory process selection
- `mm/tlb_shootdown.rs` - SMP TLB invalidation
- `mm/fallible_map.rs` - Fallible BTreeMap wrapper (R172-22 follow-on)

**Lock Types:**
- `Mutex<FrameAllocator>` - Frame allocation (Level 3)
- `Mutex<()> PT_LOCK` - Page table operations (Level 3)
- `RwLock<PageTable>` - Per-process page tables

**Lock Ordering (CRITICAL):**
```
Page-fault entry (IF=0)
  └── try_lock(PT_LOCK)
        ├── acquired: resolve COW atomically
        └── contended: return Busy, IRETQ, then retry the faulting instruction
```
R186-10 removed the redundant `COW_FAULT_LOCK`. Blocking on `PT_LOCK` from the interrupt-gate
page-fault path is forbidden because it can prevent the current CPU from servicing the lock
holder's TLB-shootdown IPI.

**IRQ Context:** UNSAFE (page faults block, allocations may sleep)

**Security Boundaries:**
- Page table flags validation (PRESENT, USER_ACCESSIBLE, NX)
- Physical address bounds checking
- W^X enforcement (D1-BOOT-NX-KASLR-LAYOUT, R166)
- Kernel/user address separation (PHYSICAL_MEMORY_OFFSET)

**Critical Invariants:**
- PT_LOCK always paired with `interrupts::without_interrupts()`
- COW refcount decrement + frame removal atomic (R69-1 ABA prevention)
- Buddy merge checks full extent + order match (R104-1, R74-4)
- Permanent reservations never freed (R167-B: kernel, framebuffer, UEFI)

**Known Issues:**
- R166 [CRITICAL, FIXED]: Transient NX on live `.text` during boot
- R172-16 [MEDIUM, FIXED]: brk/mmap VMA TOCTOU (scalar reservation guards)
- R171-CG1x0 [HIGH, FIXED]: PT frame leak on exec migration (pt_charged_frames ledger)

---

### Layer 1: Scheduler (sched)

**Primary Responsibilities:**
- Per-CPU run queue management (priority buckets)
- Work stealing and load balancing
- Context switching (shadow buffer for safe state transfer)
- Priority inheritance for futex (E.4 PI)
- Cpuset enforcement (per-process CPU affinity)
- Lock dependency checking (lockdep)
- Time-slice accounting (100ms default, deprioritize CPU-hogs)

**Files:** 6 files
- `sched/lib.rs` - Module root
- `sched/scheduler.rs` - **CRITICAL** Scheduler core, ~1,100 lines
- `sched/enhanced_scheduler.rs` - **CRITICAL** SMP scheduler, ~1,800 lines
- `sched/process.rs` - Process state transitions
- `sched/cpuset.rs` - CPU affinity management
- `sched/lock_ordering.rs` - Lockdep validation

**Lock Types:**
- `Mutex<ReadyQueues>` - Per-CPU run queues (Level 3)
- `Mutex<Option<ProcessId>>` - Current task per-CPU (Level 3)
- `AtomicBool need_resched` - Per-CPU reschedule flag (lockless)

**IRQ Context:** Mixed (timer tick safe, schedule() IRQ-unsafe)

**Security Boundaries:**
- Cpuset isolation prevents unauthorized CPU access
- Priority inheritance prevents priority inversion attacks

**Critical Invariants:**
- Context shadow buffer (per-CPU UnsafeCell) prevents use-after-unlock (R70-4)
- `on_cpu` + `switch_in_progress` guard prevents steal-before-save (R172-03)
- `syscall_active` + `frame_ptr` cleared on enter_usermode branch (R172-05)
- Process state + `stopped` flag checked atomically

**Known Issues:**
- R172-03 [HIGH, FIXED]: Steal-before-save race (marked Ready before save complete)
- R172-05 [HIGH, FIXED]: `syscall_active` leak → -EBUSY CPU wedge

---

### Layer 2: Security & Access Control

#### security (Hardening)

**Primary Responsibilities:**
- W^X enforcement validation
- NX bit enforcement on data pages
- Identity map cleanup/hardening
- KASLR (Kernel Address Space Layout Randomization)
- kptr guard (kernel pointer obfuscation)
- Spectre/Meltdown mitigations (IBRS/STIBP/IBPB)
- Hardware RNG + ChaCha20 CSPRNG

**Files:** 9 files
- `security/lib.rs` - Security initialization
- `security/memory_hardening.rs` - **CRITICAL** W^X enforcement
- `security/wxorx.rs` - W^X validation
- `security/kaslr.rs` - Address randomization
- `security/kptr.rs` - Kernel pointer obfuscation
- `security/spectre.rs` - Spectre mitigations
- `security/rng.rs` - CSPRNG
- `security/fips.rs` - FIPS compliance mode
- `security/tests.rs` - Security self-tests

**Lock Types:**
- `Mutex<RngState>` - CSPRNG state (Level 9)
- `AtomicPtr<KptrXorKey>` - kptr guard seed (Level 9)

**IRQ Context:** Mixed (RNG try_lock from IRQ safe)

**Security Boundaries:**
- W^X policy: no page both writable and executable
- NX policy: data pages have NX bit set
- Identity map removed or marked read-only (cleanup_strategy)
- KASLR randomizes kernel text/stack/heap addresses
- kptr guard XORs kernel pointers before logging (KASLR bypass prevention)

**Critical Invariants:**
- Single-pass W^X enforcement (D1-BOOT-NX-KASLR-LAYOUT, R166)
- AT_RANDOM 16 bytes from hardware RNG (HARD-fail on EAGAIN, never weak)

---

#### lsm (Linux Security Module Hooks)

**Primary Responsibilities:**
- Per-file open/create/unlink/rename hooks
- Pluggable policy engine
- DAC + MAC integration
- Capability-based access control integration
- Audit event generation

**Files:** 2 files
- `lsm/lib.rs` - LSM hook dispatcher
- `lsm/policy.rs` - Policy engine

**Lock Types:** None (callback-based, uses capability table locks)

**Security Boundaries:**
- Per-operation hooks (open, create, unlink, rename)
- Inode identity binding (R172-X-F4: LSM decision bound to inode under lock)
- Single-resolution enforcement (R172-02: VFS read feeds SAME bytes to LSM + ELF loader)

---

#### cap (Capability System)

**Primary Responsibilities:**
- Non-forgeable capability handles (generation counters)
- Monotonic rights restriction (delegate reduced rights only)
- Per-process capability table
- Fork/exec inheritance (CLOEXEC/CLOFORK flags)
- Capability revocation (generation counter increment)

**Files:** 2 files
- `cap/lib.rs` - Capability table
- `cap/types.rs` - Capability types

**Lock Types:**
- `RwLock<CapTable>` - Per-process capability table (Level 4)

**Security Boundaries:**
- Generation counter prevents use-after-revoke
- Rights cannot be escalated (monotonic restriction)
- Per-process isolation (table not shared across fork by default)

**Current Status:** Phase U preview (M0: `cap_table` field exists, syscall wiring pending)

---

#### seccomp (Syscall Filtering)

**Primary Responsibilities:**
- BPF-based syscall filtering
- Pledge.rs-style promise allow-lists
- Per-process sandbox enforcement

**Files:** 2 files
- `seccomp/lib.rs` - BPF filter evaluation
- `seccomp/types.rs` - **CRITICAL** Syscall number constants

**Lock Types:** None (per-process state, no synchronization)

**Security Boundaries:**
- Opt-in only (no default filter)
- R150-3: Pledge + Seccomp filters lock-step (single truth)
- M0-6-SLICE1: Parity self-test prevents allowed-but-ENOSYS divergence

**Known Issues:**
- R172-6 follow-on [LOW, FIXED]: FATTR syscall number mismatch (91/93 vs 93/94)

---

#### audit (Security Audit Log)

**Primary Responsibilities:**
- SHA-256 hash-chained audit log
- Tamper-evidence detection
- Security event logging

**Files:** 1 file
- `audit/lib.rs` - Audit log

**Lock Types:**
- `Mutex<AuditLog>` - Global audit log (Level 9)

---

### Layer 3: IPC & Synchronization (ipc)

**Primary Responsibilities:**
- Futex with priority inheritance (FUTEX_LOCK_PI/UNLOCK_PI)
- Pipes (blocking I/O, per-TGID futex tables)
- Message queues (capability-based)
- Wait queues, condition variables, semaphores
- Per-namespace IPC endpoint registry

**Files:** 5 files
- `ipc/lib.rs` - Module root
- `ipc/futex.rs` - **CRITICAL** Futex implementation, ~900 lines
- `ipc/pipe.rs` - Pipe implementation
- `ipc/ipc.rs` - Message queue
- `ipc/sync.rs` - Wait queue primitives

**Lock Types:**
- `Mutex<FutexTable>` - Per-TGID futex hash table (Level 7)
- `Mutex<BTreeMap<EndpointId, Endpoint>>` - Per-namespace endpoint registry (Level 7)
- `Mutex<Pipe>` - Per-pipe state (Level 7)
- `Mutex<WaitQueue>` - Wait list serialization (Level 7)

**IRQ Context:** UNSAFE (all blocking operations)

**Security Boundaries:**
- Futex address validation: 4-byte aligned, user-accessible, present (R32-IPC-2)
- Per-namespace endpoint isolation (R75-1, R75-2)
- TGID-based futex isolation for pthread semantics (R37-2)

**Critical Invariants:**
- Futex claim/bump under FUTEX_TABLE lock (R172-8 TOCTOU fix)
- Wait queue timeout re-check under lock (R172-11/12/13 lost-wakeup fix)
- Pipe I/O releases process lock before blocking (R41-3)

**Known Issues:**
- R172-08 [HIGH, FIXED]: Futex bucket TOCTOU (claim after lock drop)
- R172-11/12/13 [HIGH/MED, FIXED]: Lost-wakeup cluster (missing under-lock re-check)
- R172-18 [MEDIUM, DEFERRED]: Futex PI/robust ABI (COW deadlock hazard)

---

### Layer 4: File Systems & Networking

#### vfs (Virtual File System)

**Primary Responsibilities:**
- Multi-filesystem abstraction (devfs, ramfs, procfs, cgroupfs, ext2, initramfs)
- Path resolution with symlink loop detection
- Inode cache with reference counting
- File descriptor table (per-process)
- Mount namespace implementation
- Directory entry caching
- Atomic rename/unlink (R172-20/21/22)

**Files:** 11 files
- `vfs/lib.rs` - Module root
- `vfs/manager.rs` - **CRITICAL** VFS manager, path resolution
- `vfs/ramfs.rs` - **CRITICAL** RAM filesystem, ~1,100 lines
- `vfs/cgroupfs.rs` - Cgroup filesystem
- `vfs/procfs.rs` - /proc filesystem
- `vfs/devfs.rs` - Device filesystem
- `vfs/initramfs.rs` - Initial RAM disk
- `vfs/ext2.rs` - EXT2 filesystem
- `vfs/mount_namespace.rs` - Mount namespace isolation
- `vfs/traits.rs` - FileSystem trait
- `vfs/types.rs` - VFS types

**Lock Types:**
- `RwLock<MountTable>` - Mount points per namespace (Level 6)
- `Mutex<Inode>` - Per-inode state (Level 6)
- `RwLock<InodeCache>` - Global inode cache (Level 6)
- `Mutex<FileMeta>` - File metadata (Level 6)
- `Mutex<()> RAMFS_RENAME_LOCK` - Global rename serialization (Level 8, R172-14)

**IRQ Context:** UNSAFE (file I/O blocks)

**Security Boundaries:**
- LSM hooks on open/create/unlink/rename
- DAC checks via credentials
- Mount namespace isolation
- Capability-based FD passing
- Atomic rename with inode identity binding (R172-X-F4)

**Critical Invariants:**
- Path string validation (length limits, no null bytes)
- Inode reference counting (Arc prevents use-after-free)
- Symlink loop detection (MAX_SYMLINK_DEPTH)
- Atomic insert-first/remove-after rename (R172-14: single lock span)

**Known Issues:**
- R172-14/15 [HIGH, FIXED]: ramfs rename half-mutation + ancestry TOCTOU
- R172-22 follow-on [LOW, FIXED]: FallibleOrderedMap migration (devfs/initramfs)

---

#### net (Networking Stack)

**Primary Responsibilities:**
- Full TCP/IP stack (Ethernet, ARP, IPv4, ICMP, TCP, UDP)
- VirtIO-net device driver with DMA
- Packet buffer pools with headroom/tailroom
- TCP connection tracking with SYN cookies
- UDP datagram delivery
- Firewall with connection tracking (R175-2, R176)
- Rate limiting (ARP, ICMP via token buckets)
- Per-namespace network device isolation
- Per-cgroup ephemeral port budgets (J2-8)

**Files:** 11 files
- `net/src/lib.rs` - Module root
- `net/src/socket.rs` - **CRITICAL** Socket table, ~1,000 lines
- `net/src/tcp.rs` - TCP implementation
- `net/src/udp.rs` - UDP implementation
- `net/src/ipv4.rs` - IPv4 routing
- `net/src/ethernet.rs` - Ethernet framing
- `net/src/arp.rs` - ARP cache
- `net/src/icmp.rs` - ICMP handling
- `net/src/firewall.rs` - Firewall rules
- `net/src/conntrack.rs` - Connection tracking
- `net/src/fragment.rs` - IP fragmentation
- `net/src/virtio_net.rs` - VirtIO-net driver
- `net/src/device.rs` - Network device abstraction
- `net/src/stack.rs` - Network stack
- `net/src/buffer.rs` - Packet buffers
- `net/src/pci.rs` - PCI device enumeration

**Lock Types:**
- `Mutex<SocketTable>` - Global socket registry (Level 8)
- `Mutex<TcpConnTable>` - TCP connection tracking (Level 8)
- `RwLock<FirewallTable>` - Firewall rule set (Level 8)
- `Mutex<ArpCache>` - ARP entry cache (Level 8)
- `Mutex<FragmentCache>` - IP fragment reassembly (Level 8)

**IRQ Context:** Mixed (RX/TX from IRQ safe with try_lock)

**Security Boundaries:**
- Firewall rule evaluation (R175-2, R176)
- Connection tracking state validation
- SYN flood mitigation via SYN cookies
- Per-namespace port binding
- Per-cgroup ephemeral port budget (J2-8, R170-2)

**Known Issues:**
- R170-2 [HIGH, FIXED]: Port budget tracking (origin-keyed, delete-gate sampling)

---

## Lock Ordering Hierarchy

**CRITICAL:** Violations cause deadlock. Always acquire locks in ascending level order.

```
Level 0: IRQ Atomic (no locks, atomics only)
Level 1: Per-CPU Local (cpu_local, no blocking)
Level 2: Cgroup Registry Read (try_read from IRQ)
Level 3: Page Table Global Lock (PT_LOCK)
Level 4: Process Lock (per-PCB Mutex<Process>)
Level 5: Cgroup Limits Lock (per-node Mutex<CgroupLimits>)
Level 6: Cgroup Registry Write (CGROUP_REGISTRY.write)
Level 7: MmState Lock (per-process Mutex<MmState>)
Level 8: VFS Locks (RAMFS_RENAME_LOCK, per-inode RwLock)
Level 9: IPC & Net Locks (FUTEX_TABLE, SOCKET_TABLE)
Level 10: Allocator/Heap (Buddy, page cache)
```

**Key Rules:**
1. **Process → MmState** — never reverse
2. **#PF → try_lock(PT_LOCK)** — never block with IF=0 (R186-10)
3. **Global rename lock → per-inode** (R172-14)
4. **Never hold Process across blocking I/O** (R41-3)
5. **Cgroup snapshot BEFORE charging** (R110-2)

---

## Security Architecture

### 1. Hardware Security Features

**SMEP (Supervisor Mode Execution Prevention)**
- Prevents kernel from executing user-space code
- Enabled via CR4.SMEP bit
- Enforced on all CPUs (boot + APs)

**SMAP (Supervisor Mode Access Prevention)**
- Prevents kernel from accessing user-space memory
- Requires explicit STAC (set AC) before user access
- CLAC (clear AC) after access
- Enforced in copy_from_user/copy_to_user primitives

**NX (No-Execute)**
- Data pages have NX bit set
- Code pages are executable but not writable (W^X)
- D1-BOOT-NX-KASLR-LAYOUT (R166): Single-pass enforcement

**KPTI (Kernel Page Table Isolation)**
- Dual-root CR3: kernel CR3 (full map) vs user CR3 (user + entry-island)
- Mitigates Meltdown (speculative execution leaks)
- Syscall entry switches CR3 atomically

**Spectre Mitigations**
- IBRS (Indirect Branch Restricted Speculation)
- STIBP (Single Thread Indirect Branch Predictors)
- IBPB (Indirect Branch Predictor Barrier)

---

### 2. Software Security Boundaries

**Namespace Isolation (5 types)**
- PID namespace: Process ID isolation
- IPC namespace: Futex/message queue isolation
- Mount namespace: Filesystem view isolation
- Net namespace: Network device isolation
- User namespace: UID/GID mapping

**Cgroup Resource Limits**
- CPU quota (cpu.max): throttle CPU-bound processes
- Memory limit (memory.max): OOM killer on breach
- PID limit (pids.max): prevent fork bombs
- FD limit (J2-7): per-cgroup open file budget
- Port limit (J2-8): per-cgroup ephemeral port budget

**Capability System (Phase U preview)**
- Non-forgeable handles with generation counters
- Monotonic rights restriction (delegate reduced rights only)
- Per-process capability table
- M0: `cap_table` field present, syscall wiring pending

---

### 3. Memory Safety Mechanisms

**W^X Enforcement**
- No page both writable and executable
- Enforced at page-table level
- Violations → #PF → SIGSEGV (user) or panic (kernel)

**Stack Protection**
- M0-7 SLICE-1: Guard page unmapped below user stack
- Downward overflow → not-present → SIGSEGV
- Kernel stack guard (red zone)

**COW (Copy-On-Write)**
- Fork shares read-only pages (refcount tracking)
- First write → page fault → copy → private page
- Serialized by `PT_LOCK`; page-fault contention returns typed `Busy` and retries after IRETQ (R186-10)

**User Pointer Validation**
- SMAP enforces explicit STAC/CLAC
- Range check: [USER_BASE, USER_SPACE_TOP)
- Page-accessible check before deref
- Canonical address validation (RIP/RSP)

---

## Memory Safety Boundaries

### Physical Memory Management

**Buddy Allocator Safety**
- R74-4: Allocation order recorded per-page; free rejects order mismatch
- R104-1: Buddy merge checks full extent (prevents overrun)
- R152-15: Merge aborts if buddy marked free but not in free_list
- R167-B: Permanent reservations (kernel, framebuffer, UEFI) never freed

**Reservation-Aware Allocation (R167)**
- Conventional-only admission (no allocation from reserved regions)
- Per-page reservation tracking (heap, kernel, framebuffer, UEFI)
- Fail-closed overflow checks
- BootInfo ABI v1 for memory map handoff

---

### Virtual Memory Management

**Page Table Safety**
- PT_LOCK serializes all page-table modifications
- Always paired with `interrupts::without_interrupts()`
- Page-fault COW uses non-blocking `try_lock(PT_LOCK)` and never spins with interrupts disabled (R186-10)
- INVPCID for precise TLB invalidation

**COW Correctness**
- Refcount per physical frame (atomic)
- Decrement + removal atomic under write lock (R69-1 ABA prevention)
- COW fault outcomes are typed as `Handled`, `NotCow`, `Busy`, or `Fatal`; only `Busy` retries
- Z-8 two-phase fork: plan → pre-allocate → apply

**Charge/Uncharge Balance**
- R138-1: Fork charges worst-case COW footprint
- M2-1 SLICE-2/3: Origin-keyed `mem_pinned` tripwire
- R171-CG1x0: PT frame ledger (pt_charged_frames, pt_inherited_bytes)
- Delete gate samples pinned counters before removal (R170-2)

---

### User Stack Safety

**M0-7 Stack Protection**
- SLICE-1: Guard page unmapped below stack (downward overflow → SIGSEGV)
- SLICE-4: Charge-correct `try_grow_user_stack` (charges `elf_charged_bytes`, not `vm_charged_bytes`)
- SLICE-5 (deferred): Demand-grow with bounded-batch #PF arm
- Stack reservation: 2 MiB window, 511 pages usable (guard at low end)

---

## Concurrency Model

### Process State Machine

**States:**
```
Init → Ready → Running → Blocked/Sleeping/Stopped → Zombie → Terminated
```

**Transitions:**
- Init: Just allocated, not yet scheduled
- Ready: Enqueued in scheduler, waiting for CPU
- Running: Executing on CPU
- Blocked: Waiting for event (futex, pipe, socket)
- Sleeping: Nanosleep, poll timeout
- Stopped: SIGSTOP/SIGTSTP (R98-1: orthogonal to state)
- Zombie: Exited, awaiting reap
- Terminated: Fully cleaned up

**Scheduler Algorithm:**
- Multi-level feedback queue (priority buckets)
- Time-slice: 100ms default
- CPU-intensive processes deprioritized
- Work stealing across CPUs
- Priority inheritance for PI-futex

---

### Futex & Priority Inheritance

**Futex Operations:**
- FUTEX_WAIT: Block until value changes
- FUTEX_WAKE: Wake N waiters
- FUTEX_LOCK_PI: Priority-inheritance lock acquire
- FUTEX_UNLOCK_PI: Priority-inheritance lock release

**Priority Inheritance (E.4 PI):**
- Futex holder inherits max waiter priority
- Prevents priority inversion attacks
- R172-8: Claim/bump under FUTEX_TABLE lock (TOCTOU fix)

**Robust Futex (R172-18, deferred):**
- COW deadlock hazard (foreign-CR3 access)
- No libc consumer in M0
- Full ABI deferred to post-M0

---

### Signal Delivery (M0-5)

**Synchronous Delivery (SLICE 1a, DONE)**
- rt_sigaction(13): Install signal handler (requires SA_RESTORER)
- rt_sigprocmask(14): Modify signal mask
- rt_sigreturn(15): Return from signal handler (SROP-defended)
- Delivery at syscall exit (after kill check)
- rt_sigframe layout: 128B red zone + 512B FXSAVE + handler context

**Blocked Syscall EINTR (SLICE 1b-1, DONE)**
- Handler signal wakes blocked waiter
- Returns EINTR via per-site bail + under-lock re-check
- R172-11/12/13: Lost-wakeup fix (re-check after publishing Blocked)
- Cross-CPU kick: Reschedule IPI broadcast

**Frame Republish (SLICE 1b-2, DONE)**
- saved_frame_ptr/owner snapshot at syscall entry
- Republish at delivery tail
- Handler lands at SAME return as EINTR (not one syscall later)

**Preemptive Delivery (SLICE 3, DONE 2026-07-02)**
- Timer-IRQ delivers handlers on Ring-3 return
- Try-lock-or-defer strategy (no block in IRQ)
- Three lens fixes: FIX A (try_lock), FIX B (double-delivery invariants), FIX C (atomic FPU capture)

**SROP Defense:**
- Canonical + low-half RIP/RSP checks
- RFLAGS sanitize (strip privileged bits)
- Executed before SYSRET

---

### SMP & CPU Affinity

**Per-CPU Scheduler:**
- Independent run queues per CPU
- Work stealing when idle
- Load balancing (deferred)

**Inter-CPU Operations:**
- Reschedule IPI: Wake idle CPU or interrupt running task
- TLB shootdown IPI: INVPCID on remote CPUs
- Timer tick: Per-CPU, independent

**CPU Affinity:**
- Per-process `allowed_cpus` bitmask (CLONE_AFFINITY)
- Cpuset subsystem: Per-process cpuset_id
- R163-4: notify_cpuset_task_{joined,left} after fallible ops

---

## IRQ-Context Constraints

### sync_safe Safety Model

> **Status (R192-8, 2026-10-23): dormant.** No kernel crate depends on
> `sync_safe`, so the runtime check below never runs. Its `lock()` panics in
> debug builds whenever `RFLAGS.IF=0`, which rejects correct IRQ-masked
> locking; see `kernel/sync_safe/README.md`. The section is kept as the
> crate's design record.

**Philosophy:**
- Defense-in-Depth Layer 2: Runtime detection
- Part of R173 fail-closed syscall strategy
- Debug builds check `RFLAGS.IF` (interrupt flag)

**sync_safe::Mutex Wrapper:**
```rust
pub struct Mutex<T> {
    inner: spin::Mutex<T>,
}

impl<T> Mutex<T> {
    // Panics in debug if interrupts disabled
    pub fn lock(&self) -> MutexGuard<'_, T>
    
    // Safe to call from IRQ context
    pub fn try_lock(&self) -> Option<MutexGuard<'_, T>>
}
```

**Detection Logic:**
```rust
#[cfg(debug_assertions)]
fn assert_irq_safe(&self, context: &str) {
    let rflags: u64;
    unsafe {
        asm!("pushfq; pop {}", out(reg) rflags);
    }
    const RFLAGS_IF: u64 = 1 << 9;
    if (rflags & RFLAGS_IF) == 0 {
        panic!("FATAL: {} called with IRQs disabled", context);
    }
}
```

---

### Safe-to-Call from IRQ

**Operations:**
- Atomic ops (AtomicU64, AtomicBool, AtomicPtr)
- Per-CPU reads (cpu_local data)
- Static const data access
- `try_lock()` variants (non-blocking)

**Specific Functions:**
- `schedule()` on timer tick (reads Process via Arc, doesn't hold lock)
- Signal delivery attempt (try-lock-or-defer, M0-5-SLICE-3)
- Cgroup stats: `cpu_time_ns.fetch_add()` (R169-2)

---

### MUST NOT Call from IRQ

**Operations:**
- `with_current_manager()` — blocks on PT_LOCK (R67-5)
- Memory allocation (may sleep on OOM killer)
- Blocking I/O (pipe, socket, file)
- Recursive Process table lookups + locks
- Any VFS operation holding PT_LOCK

**Specific Functions:**
- `copy_from_user()` / `copy_to_user()` (page faults)
- `allocate_frame()` (buddy allocator may block)
- File read/write (VFS operations)
- Futex wait (blocking)

---

## Critical Code Paths

### 1. Fork (sys_fork → fork_inner)

**Sequence:**
1. Acquire parent Process lock
2. Check cgroup pids.max (R152-5: attach BEFORE copy to prevent TOCTOU)
3. Snapshot parent state (cgroup_id, cpuset_id, rlimits, FD table)
4. Release parent lock
5. `create_process()` — allocate child PID + kernel stack
6. Re-acquire parent lock
7. Cgroup attach (moved here by R152-5)
8. `fork_inner()`:
   - COW page table copy (under PT_LOCK, R67-6)
   - MmState clone (R138-1 worst-case charge)
   - FD table deep copy (bounded by MAX_FD=256, R162-14)
   - Seccomp state `try_clone()` (R162-6 fallible)
   - Signal state + capability table clone
   - notify_cpuset_task_joined (R163-4: after fallible ops)
9. Multi-controller charge (FD + memory, rollback on failure R123-4)
10. Scheduler: `add_process()` (enqueue in ready queue)

**Lock Order:** Parent Process → PT_LOCK → Child Process → Cgroup locks

**Charging:**
- Worst-case COW footprint pre-charged (R138-1)
- PT frames tracked separately (pt_inherited_bytes, R171-CG1x0)
- Rollback on any failure (no partial state)

---

### 2. Exec (sys_execve → exec_from_bytes)

**Sequence:**
1. Path resolution loop (M0-4: iterative shebang, depth-cap 4)
2. VFS read (single-resolution, feeds SAME bytes to LSM + ELF loader)
3. EBUSY check (multithread/CLONE_VM, unbypassable)
4. `exec_in_progress` flag armed (blocks concurrent exec)
5. `create_fresh_address_space()` — allocate new PML4
6. KPTI user PML4 (H.3, entry-island 4-GiB, R118-5)
7. `load_elf()` — PT_LOAD mappings, stack/brk reservation
8. `build_initial_user_stack()` — auxv + argc/argv/envp (M0-1)
9. Signal state preserved (handler table + mask)
10. **Atomic commit** (single Process+MmState lock hold):
    - Uncharge old AS (4-term: data + code + PT + stack)
    - Charge new AS
    - PT ledger transfer
    - Update Process fields
    - Free old PML4
11. `exec_in_progress` flag cleared

**Lock Order:** Process → MmState → Cgroup locks

**Critical Invariant:** All uncharges + PT-ledger in ONE lock hold (R171-CG1x0)

---

### 3. Schedule (enhanced_scheduler::schedule)

**Sequence:**
1. Current process: check state + time-slice
2. If expired/yielding: transition to Ready, enqueue
3. Find highest-priority ready process in current CPU's run queue
4. If none: `steal_one()` from other CPUs
5. If stealing: check `on_cpu` + `switch_in_progress` flags (R172-03)
6. Context switch branch selection:
   - **switch_context**: Both tasks have valid contexts (kernel→kernel)
   - **enter_usermode**: Next task is fresh or returning to Ring 3
7. **enter_usermode branch** (R172-01/04/05 fixes):
   - Save outgoing task RIP/RFLAGS (R172-01)
   - Restore FS_BASE/GS_BASE from PCB (R172-04)
   - Clear `syscall_active` + `frame_ptr` (R172-05)
8. Set `current_pid`, switch CR3 if different AS
9. SYSRET to user mode (canonical RIP/RSP validated)

**Lock Order:** Scheduler internal → Process (brief Arc clone, no hold)

---

### 4. Timer Interrupt (on_timer_interrupt)

**Sequence:**
1. Current process tick (decrement time_slice, update cpu_time_ns)
2. CPU quota charge (R110-2: snapshot limits before charge, skip if `refreshing==true`)
3. Check quota throttle: if exceeded, transition to Ready + reschedule
4. Per-CPU deferred work drain (R169-5: full drain, no locks held)
5. Preemptive signal delivery attempt (M0-5-SLICE-3):
   - `try_lock()` Process (defer if contended)
   - Check `has_deliverable_signal()` (handler-only, not in_handler)
   - Capture frame (try_lock or defer)
   - Write rt_sigframe (FXSAVE + red zone + handler context)
   - Update saved RIP → handler, RSP → frame
6. Check `need_resched` flag: if set, call `schedule()`

**IRQ Safety:** All operations non-blocking (try_lock variants, atomics)

---

### 5. Page Fault (#PF)

**Sequence (COW path):**
1. Attempt non-blocking acquisition of `PT_LOCK` through `try_with_current_manager`.
2. If contended, return `CowFaultResult::Busy`; the #PF handler returns through IRETQ so IF is
   restored, then the unchanged write instruction re-faults after pending shootdown IPIs run.
3. Under `PT_LOCK`, re-read the PTE flags; return `NotCow` if another CPU already resolved it.
4. Resolve the unique-reference case in place, or allocate and copy a private physical frame.
5. Update the PTE (clear COW, set writable, install the private physical address).
6. Consume/decrement the old frame tracking reference atomically.
7. Perform the required local/remote TLB invalidation before any retired frame is released.
8. Return `Handled`; allocator or metadata failures return a typed `Fatal` reason.

**Sequence (Stack grow, M0-7 SLICE-5, deferred):**
1. Validate fault address (within stack window, below current RSP)
2. Check `stack_grow_pending_bytes` budget
3. Bounded-batch allocation (max N pages per fault, DoS prevention)
4. Charge to `elf_charged_bytes` (NOT `vm_charged_bytes`, FA-04 fix)
5. Map pages, update stack floor
6. TLB flush

**Lock Rule:** #PF may only `try_lock(PT_LOCK)`; it must never block or spin on the lock while IF=0.

---

## Dependency Graph

```
kernel (main binary)
  │
  ├─ kernel_core (hub, process/syscall/signal/cgroup/namespace)
  │   ├─ mm (memory management)
  │   ├─ audit (security audit log)
  │   ├─ cap (capability table)
  │   ├─ lsm (LSM hooks)
  │   ├─ seccomp (BPF filtering)
  │   ├─ security (hardening)
  │   ├─ net (networking, cycle avoided via no kernel_core dep)
  │   ├─ trace (observability)
  │   └─ compliance (standards)
  │
  ├─ sched (scheduler, work stealing, priority)
  │   ├─ kernel_core (process state)
  │   ├─ arch (context switch)
  │   ├─ cpu_local (per-CPU data)
  │   └─ trace (counters)
  │
  ├─ mm (memory manager, buddy, page tables, COW)
  │   ├─ drivers (MMIO, framebuffer)
  │   ├─ cpu_local (per-CPU alloc state)
  │   ├─ tlb_ops (TLB shootdown)
  │   └─ klog (logging)
  │
  ├─ arch (x86_64 abstraction, context switch, syscall entry)
  │   ├─ cpu_local (per-CPU GDT/TSS/IST)
  │   ├─ drivers (APIC, HPET)
  │   └─ klog
  │
  ├─ ipc (futex, pipe, message queue)
  │   ├─ kernel_core (process lookup)
  │   ├─ mm (user pointer validation)
  │   ├─ vfs (pipe integration)
  │   └─ trace (timeout callbacks)
  │
  ├─ vfs (filesystems: ramfs, devfs, procfs, cgroupfs, ext2, initramfs)
  │   ├─ kernel_core (credentials, FD table)
  │   ├─ lsm (security hooks)
  │   ├─ block (disk I/O)
  │   ├─ mm (page cache)
  │   ├─ cap (capability checks)
  │   └─ klog
  │
  ├─ net (TCP/IP stack, VirtIO-net, firewall, conntrack)
  │   ├─ mm (DMA buffers)
  │   ├─ virtio (device transport)
  │   ├─ cap (socket capabilities)
  │   ├─ lsm (firewall policy)
  │   ├─ security (RNG for ISN/SYN cookies)
  │   ├─ iommu (device isolation)
  │   └─ klog
  │
  ├─ security (W^X, KASLR, kptr guard, Spectre, RNG)
  │   ├─ mm (page table validation)
  │   ├─ drivers (RDRAND)
  │   └─ klog
  │
  ├─ iommu (VT-d, DMA remapping)
  │   ├─ mm (MMIO mapping)
  │   └─ klog
  │
  ├─ block (VirtIO-blk)
  │   ├─ virtio (device transport)
  │   └─ mm (DMA)
  │
  ├─ lsm (LSM hooks, policy engine)
  │   ├─ cap (capability checks)
  │   └─ audit (conditional event logging)
  │
  ├─ (sync_safe: dormant since R192-8, not linked)
  ├─ cpu_local (per-CPU storage)
  ├─ drivers (framebuffer, keyboard, VGA, device I/O)
  ├─ klog (structured logging)
  ├─ trace (counters, profiler, watchdog, kdump)
  ├─ tlb_ops (TLB shootdown primitives)
  ├─ virtio (VirtIO transport layer)
  ├─ livepatch (runtime kernel patching, ECDSA-P256)
  ├─ crypto (cryptographic primitives)
  ├─ cap (capability system)
  ├─ audit (security audit log)
  └─ compliance (standards compliance checks)
```

**Key Anti-Cycle Constraints:**
- **kernel_core** uses callback pattern for LSM/audit (no dep cycle)
- **mm** is a leaf (both kernel_core and net depend on it)
- **net** avoids kernel_core dep (no cycle via mm)
- **vfs** depends on kernel_core but not vice versa (syscall layer hooks)
- **lsm** uses callback registration (register_context_provider)

---

## High-Risk Areas for Audit

### Priority 1: CRITICAL (Context Switch & Memory Safety)

**1. arch/context_switch.rs (Lines 372-646)**
- Risk: Incomplete register save/restore → privilege escalation
- R172-01: RIP/RFLAGS omission allowed user code in Ring 0
- Audit: Verify ALL registers saved (RIP, RSP, RFLAGS, segments, FPU)
- Verify: `clone(); sched_yield()` test passes

**2. arch/syscall.rs (Syscall Entry/Exit, Lines 1120-1130)**
- Risk: FS_BASE/GS_BASE leak across tasks → TLS corruption
- R172-04: No restore on switch_context branch
- Audit: Verify FS/GS restored from PCB on ALL return paths
- Verify: arch_prctl(SET_FS) + CLONE_SETTLS + yield test

**3. mm/page_table.rs (COW Fault Handler)**
- Risk: Refcount double-decrement → use-after-free
- R69-1/R186-10: PT_LOCK serialization + atomic decrement + typed contention result
- Audit: Verify #PF uses non-blocking PT_LOCK acquisition and `Busy` retries only after IRETQ
- Audit: Verify refcount==0 check before dealloc

**4. mm/buddy_allocator.rs (Physical Frame Allocation)**
- Risk: Double-free, overlapping blocks, reservation violations
- R74-4/R104-1/R152-15/R167-B: Order tracking + merge validation + reservations
- Audit: Verify merge checks buddy free-list presence
- Audit: Verify reservation regions never allocated

---

### Priority 2: HIGH (Process Lifecycle & Cgroups)

**5. kernel_core/fork.rs (fork_inner, Lines 200-600)**
- Risk: Incomplete COW setup → memory corruption
- R138-1: Worst-case charge prevents double-uncharge
- Audit: Verify charge/uncharge symmetry
- Audit: Verify PT_LOCK held during page table copy

**6. kernel_core/syscall.rs (Exec Path, Lines 4400-4700)**
- Risk: Half-committed exec state → AS leak
- R172-02: ELF bounds validation before iterator
- Audit: Verify ExecSpaceGuard rollback on ALL error paths
- Audit: Verify atomic commit (uncharge + charge in ONE lock hold)

**7. kernel_core/cgroup.rs (Charge/Uncharge, Lines 1500-2100)**
- Risk: Charge/uncharge asymmetry → delete gate fail-closed forever
- R170-2/M2-1: Origin-keyed pinned counters + delete gate sampling
- Audit: Verify EVERY charge has paired uncharge
- Audit: Verify delete gate samples pinned (not display) counters

**8. kernel_core/process.rs (free_process_resources, Lines 4100-4400)**
- Risk: Resource leak on exit
- R171-F5: FD release OUTSIDE PROCESS_TABLE lock (TOCTOU fix)
- Audit: Verify 4-term memory uncharge (data + code + PT + stack)
- Audit: Verify PT-ledger reconciliation

---

### Priority 3: MEDIUM (Concurrency & IPC)

**9. sched/enhanced_scheduler.rs (schedule, Lines 1300-1800)**
- Risk: Steal-before-save → half-saved task execution
- R172-03: `on_cpu` + `switch_in_progress` guard
- Audit: Verify Ready marking AFTER save complete
- Audit: Verify `syscall_active` cleared on enter_usermode

**10. ipc/futex.rs (Futex Operations, Lines 200-900)**
- Risk: TOCTOU between table lookup and claim → orphaned bucket
- R172-08: Claim/bump under FUTEX_TABLE lock
- Audit: Verify bucket claim atomic with table lock
- Audit: Verify priority inheritance correctly propagates

**11. vfs/ramfs.rs (Rename, Lines 800-900)**
- Risk: Half-mutation → lost directory entry
- R172-14/15: Atomic insert-first/remove-after + global lock
- Audit: Verify RAMFS_RENAME_LOCK held across BOTH insert AND remove
- Audit: Verify ancestry re-check under lock

---

### Priority 4: DEFENSE-IN-DEPTH (Hardening & Isolation)

**12. security/memory_hardening.rs (W^X Enforcement)**
- Risk: Transient writable+executable window
- D1-BOOT-NX-KASLR-LAYOUT (R166): Single-pass enforcement
- Audit: Verify no page transiently both W+X
- Audit: Verify enforcement before first user instruction

**13. arch/interrupts.rs (#PF Handler, Lines 1100-1300)**
- Risk: Kernel panic on legitimate user fault
- Audit: Verify user #PF → SIGSEGV, not panic
- Audit: Verify COW fault path correct (copy-on-write)
- Audit: Verify stack guard page generates SIGSEGV (not silent corruption)

**14. kernel_core/usercopy.rs (User Pointer Access)**
- Risk: SMAP bypass → kernel access to unmapped user memory
- R106-4: Narrow per-chunk STAC/CLAC window
- Audit: Verify STAC/CLAC pairing on ALL paths
- Audit: Verify fault handling converts to EFAULT

**15. net/src/socket.rs (Port Binding, Lines 300-600)**
- Risk: Port exhaustion DoS
- J2-8/R170-2: Per-cgroup ephemeral port budget
- Audit: Verify origin-keyed port_pinned tracking
- Audit: Verify delete gate samples port_pinned

---

## Known Audit Findings

### R172 (2026-06-23) — CRITICAL Context-Switch Cluster [ALL FIXED 2026-06-29]

**R172-01 [CRITICAL]** — Context-switch RIP/RFLAGS omission
- **Impact:** Privilege escalation (user `.text` executes in Ring 0)
- **Root Cause:** `save_context` omitted RIP/RFLAGS on outgoing task
- **Trigger:** `clone(); sched_yield()` → parent resumes at creation entry in Ring 0
- **Fix:** Capture RIP/RFLAGS on ALL switch branches + self-test
- **Status:** FIXED 2026-06-23, verified 2-core SMP

**R172-03 [HIGH]** — Steal-before-save race
- **Impact:** Half-saved task execution → undefined state
- **Root Cause:** Schedule marks Ready before save complete
- **Fix:** `on_cpu` + `switch_in_progress` guard
- **Status:** FIXED 2026-06-23

**R172-04 [HIGH]** — FS_BASE/GS_BASE leak
- **Impact:** TLS corruption across tasks
- **Root Cause:** No restore on switch_context/syscall-return branch
- **Fix:** Restore FS/GS from PCB on ALL return paths
- **Status:** FIXED 2026-06-23

**R172-05 [HIGH]** — `syscall_active` leak
- **Impact:** CPU wedge (-EBUSY on all syscalls)
- **Root Cause:** Not cleared on enter_usermode switch-out
- **Fix:** Clear `syscall_active` + `frame_ptr` on that branch
- **Status:** FIXED 2026-06-23

**R172-14/15 [HIGH]** — ramfs rename half-mutation + ancestry TOCTOU
- **Impact:** Lost directory entry or detached Arc-cycle
- **Root Cause:** Remove before insert under SEPARATE locks
- **Fix:** Atomic insert-first/remove-after under global RAMFS_RENAME_LOCK
- **Status:** FIXED 2026-06-23 + follow-on 2026-06-29

**R172-08 [HIGH]** — Futex bucket TOCTOU
- **Impact:** Orphaned bucket → lost-wakeup or PI double-owner
- **Root Cause:** Drop FUTEX_TABLE lock before claim
- **Fix:** Claim/bump under table lock
- **Status:** FIXED 2026-06-23

**R172-11/12/13 [HIGH/MED]** — Lost-wakeup cluster
- **Impact:** Blocked task never wakes despite event
- **Root Cause:** Missing under-lock re-check after publishing Blocked
- **Fix:** Re-check at ALL 4 sites (wait4, stdin, socket, prepare_to_wait)
- **Status:** FIXED 2026-06-23

---

### R171 (2026-06-12) — IOMMU & AP Initialization [FIXED 2026-06-14]

**R171-1 [CRITICAL]** — IOMMU never initialized
- **Impact:** DMA isolation dormant (devices can access all memory)
- **Root Cause:** `find_dmar_table` hard stub returns NotFound
- **Fix:** Port ACPI finder into iommu module, use mm::phys_to_virt
- **Status:** FIXED 2026-06-14

**R171-2 [CRITICAL]** — AP SYSCALL MSRs never programmed
- **Impact:** Ring 0 RIP=0 on SYSCALL from AP
- **Root Cause:** MSR setup only in BSP path
- **Fix:** Program LSTAR/CSTAR in AP init
- **Status:** FIXED 2026-06-14

**R171-CG1x0 [HIGH]** — PT frame leak on exec migration
- **Impact:** Memory charge stranded → delete gate fail-closed
- **Root Cause:** PT frames charged to wrong cgroup
- **Fix:** pt_charged_frames ledger (PhysFrame-keyed)
- **Status:** FIXED 2026-06-14 (M2-1 SLICE-0)

**R171-S-R170-2-01 [HIGH]** — Memory delete-gate strand
- **Impact:** Cgroup delete fail-closed forever
- **Root Cause:** `mem_pinned` not sampled in delete gate
- **Fix:** Origin-keyed pinned counters + delete gate sampling
- **Status:** FIXED 2026-06-14 (M2-1 SLICE-2/3)

---

### R170 (2026-06-10) — Cgroup Resource Tracking [ALL FIXED 2026-06-10]

**R170-1/2 [HIGH]** — FD/port budget tracking
- **Impact:** Resource leak → delete gate fail-closed
- **Root Cause:** Controller-bit-keyed (not origin-keyed)
- **Fix:** Origin-keyed fds_pinned/ports_pinned + delete gate sampling
- **Status:** FIXED 2026-06-10

---

### R166 (2026-06-03) — Boot NX Layout [FIXED 2026-06-03]

**D1-BOOT-NX-KASLR-LAYOUT [CRITICAL]** — Transient NX on live `.text`
- **Impact:** Speculative execution window → Meltdown leak
- **Root Cause:** Two-pass W^X enforcement (mark NX, then clear for code)
- **Fix:** Single-pass enforcement (never mark `.text` NX)
- **Verification:** 30/30 KASLR boots clean, amplified reproducer 4/4 fault on old code
- **Status:** FIXED 2026-06-03, `make boot-check` CI gate added

---

### R167 (2026-06-04) — PMM Reservation Hardening [FIXED 2026-06-04]

**R167-PMM-RESERVATION-HARDENING [MEDIUM latent]** — Physical memory double-ownership
- **Impact:** Heap/kernel/framebuffer overlap → corruption
- **Root Cause:** No per-page reservation tracking
- **Fix:** Reservation-aware buddy (Parts A/B/C)
  - Conventional-only admission
  - Per-page heap/kernel/framebuffer/UEFI reservations
  - Fail-closed overflow checks
  - BootInfo ABI v1
- **Status:** FIXED 2026-06-04, Codex-converged + 4-lens Workflow review

---

### R165 (2026-05-29) — Medium/Low Cluster [ALL FIXED 2026-05-29]

**8 MEDIUM findings FIXED:**
- R165-1/2: brk reservation protocol (brk_in_progress guard)
- R165-3/4: Fallible BTreeMap operations
- R165-5/6: Cpuset notification timing
- R165-7/8: Signal delivery edge cases

**12/13 LOW findings FIXED:**
- R165-9..21: Various defensive hardening
- R165-14: Mitigated (BTreeMap no_std tech-debt)

---

### Deferred Findings

**R172-18 [MEDIUM]** — Futex PI/robust ABI
- **Impact:** Incomplete pthread_mutex support
- **Root Cause:** Robust list access requires foreign-CR3 → COW deadlock hazard
- **Deferral Reason:** No libc consumer in M0; unsafe-as-designed
- **Future Work:** Phase U post-M0 ABI item

**R121-2 [MEDIUM]** — BTreeMap allocation
- **Impact:** OOM panic (not ENOMEM)
- **Root Cause:** no_std BTreeMap uses infallible allocator
- **Deferral Reason:** Requires custom BTreeMap or upstream change
- **Mitigation:** FallibleOrderedMap wrapper (R172-22 follow-on)

---

## Development Status (M0 User-Mode ABI)

**Current Phase:** M0 (User-Mode ABI Foundation)  
**Gate:** Static musl `hello` runs to completion, exit 0  
**Status:** 90% complete (items 1-5 DONE, item 6 SLICE 1-4 DONE, item 7 SLICE 1/3/4 DONE)

### Completed (2026-06-21 → 2026-07-02)

✅ **M0-1** — Auxv builder + SysV entry stack  
✅ **M0-2** — Startup syscalls (clock_gettime, readv, rt_sigprocmask)  
✅ **M0-3** — Conformance harness (`make musl-check` gate)  
✅ **M0-4** — Exec disambiguation (confused-deputy killed)  
✅ **M0-5** — Signal delivery (SLICE 1a/1b-1/1b-2/3 DONE)  
✅ **M0-6** — Syscall fill (SLICE 1: RLIMIT, SLICE 2: rename, SLICE 4: fcntl/pipe2/pread/pwrite)  
✅ **M0-7** — Stack guard page (SLICE 1) + demand-grow primitive (SLICE 4) + preemptive IRQ delivery (SLICE 3)

### Remaining (Tracked in next-phase-plan.md)

- **M0-6 SLICE 3** — readlink/symlink (ramfs Symlink node + resolver fix)
- **M0-6 SLICE 5+** — Additional syscalls (poll, mremap, statx, termios)
- **M0-7 SLICE 5** — Demand-grow #PF arm (geometry split + bounded-batch)
- **M0-7 SLICE 6** — SMP demand-grow (cross-CPU synchronization)

### Post-M0 Roadmap (Phase U)

**S0** — M0 complete (static musl gate green)  
**S1** — Exec-split hardened  
**S2** — Native core cap-wiring (`fd_table`→`CapId`, 5 native syscalls @ 600–631)  
**S3** — Synchronous IPC + shared memory (gate: measure cold-path latency)  
**S4** — Personality stand-up (**POINT OF NO RETURN**)  
**S5** — Dynamic linking (ld.so/PIE/vDSO)  
**S6** — glibc compatibility  
**S7** — OCI (Open Container Initiative) compliance

---

## Verification Gates

### Build & Test

**Commands (ALL remote-only via ssh-skill):**
```bash
make build          # Compile kernel
make lint           # Clippy (4/4 lints)
make test           # Single-core in-kernel tests (17 passed)
make boot-check     # Boot to Ring 3, 0 NX faults
make musl-check     # Static musl hello end-to-end (M0 gate)
```

**SMP Testing:**
```bash
make test           # 2-core SMP (22 passed, CPU 1 online)
```

**Current Status (2026-07-02):**
- Build: 0 errors
- Lint: 4/4 OK
- Test (single-core): 17 passed, 0 failed
- Test (2-core SMP): 22 passed, 0 failed
- Boot-check: 0 NX violations (`v=0e e=0011`)
- Musl-check: **PASS** (both libc markers + exit 0)

---

### Security Posture

**Audit Rounds:** 186 (R1 → R186)  
**Findings:** ~1,333 filed IDs, ~1,177 fixed/resolved  
**Open:** 0 CRITICAL, 1 HIGH (`R186-4`), 0 MEDIUM  
**0-HIGH Streak:** 0/3  
**1.0-Preview Gate:** implementation/design blockers are `R186-4` and its design parent;
the independent zero-HIGH streak remains 0/3

**Last 5 Audit Rounds:**
- R188 (2026-08-07 audit, 2026-08-27 remediation): standalone full-codebase audit; 3 HIGH
  (`U06-1`, `U09-1/2/3`, `U34-1`) + 24 MEDIUM fixed; residuals `U37-1`/`U55-6`/`U29-3` open;
  not an R-series round; streak unchanged 0/3; gate still blocked on `R186-4`
- R187 (2026-08-08): KCOV observability surface; 7 findings fixed, 7/7 PASS, 8/8
  review-fix defects repaired, 0 escalated; no new open HIGH; gate still blocked on `R186-4`
- R186 (2026-07-28→30): 18 filed; 16/17 actionables fixed; `R186-18`
  fixed and review-verified; 24/24 ReviewFix defects repaired with 0 escalations;
  final eight-file source payload and remote runtime ladder verified; `R186-4` open
- R185 (2026-07-23): clean caller/lock-context verification round
- R184 (2026-07-23): real CRITICAL/HIGH findings fixed; subsequent review-fix converged

---

## Architecture Evolution & Future Work

### Near-Term (M0 Completion)

1. **Syscall Coverage** — Fill remaining ~20 syscalls (poll, mremap, statx, termios)
2. **Stack Demand-Grow** — Wire #PF arm with bounded-batch allocation (SLICE 5)
3. **SMP Demand-Grow** — Cross-CPU synchronization for stack growth (SLICE 6)
4. **Symlink Support** — ramfs Symlink node + readlink resolver fix

### Mid-Term (Phase U: S1-S3)

1. **Exec Hardening** — Strengthen shebang parser, DAC/LSM integration
2. **Capability Wiring** — `fd_table`→`CapId` mapping (5 native syscalls)
3. **Synchronous IPC** — Measure cold-path latency, optimize critical path
4. **Shared Memory** — mmap(MAP_SHARED), SysV shm (if latency budget allows)

### Long-Term (Phase U: S4-S7)

1. **Personality Layer** — De-privileged Linux personality (POINT OF NO RETURN)
2. **Dynamic Linking** — ld.so, PIE, vDSO (AT_SYSINFO_EHDR)
3. **glibc Compatibility** — Full glibc test suite green
4. **OCI Compliance** — Container runtime (runc/crun integration)

### Deferred Technical Debt

1. **BTreeMap Fallibility** — Custom allocator or FallibleOrderedMap everywhere
2. **Futex Robust List** — COW deadlock hazard (requires different approach)
3. **VFS Path Resolution OOM** — `normalize_path` infallible String allocs
4. **Load Balancing** — SMP work stealing is manual; auto-balance deferred
5. **RLIMIT Enforcement** — Advisory only (stored but not enforced)

---

## References

### Key Documents

- **`docs/next-phase-plan.md`** — Development roadmap, audit findings, M0 work items
- **`docs/roadmap.md`** — Long-term vision, Phase U sequencing
- **`docs/review/qa-YYYY-MM-DD.md`** — Per-audit-round detailed findings
- **`CLAUDE.md`** — Project instructions, dual-environment setup
- **`CONTRIBUTING.md`** — Contributor guide, pre-push hooks

### Key Files (Hot Spots)

**CRITICAL (modify with extreme caution):**
- `kernel_core/process.rs` (~4,500 lines) — Process lifecycle
- `kernel_core/syscall.rs` (~6,800 lines) — Syscall dispatcher
- `kernel_core/cgroup.rs` (~2,100 lines) — Resource accounting
- `mm/buddy_allocator.rs` (~800 lines) — Physical memory
- `mm/page_table.rs` (~1,200 lines) — Virtual memory, COW
- `arch/context_switch.rs` (~600 lines) — Register save/restore
- `sched/enhanced_scheduler.rs` (~1,800 lines) — SMP scheduler

**HIGH (complex, well-tested):**
- `kernel_core/fork.rs` — Fork/COW setup
- `ipc/futex.rs` (~900 lines) — Priority inheritance
- `vfs/ramfs.rs` (~1,100 lines) — In-memory filesystem
- `net/src/socket.rs` (~1,000 lines) — Socket table

---

## Glossary

**AP** — Application Processor (non-BSP CPU core)  
**BSP** — Bootstrap Processor (primary CPU core)  
**COW** — Copy-On-Write (deferred page copy on fork)  
**KPTI** — Kernel Page Table Isolation (Meltdown mitigation)  
**LSM** — Linux Security Module (pluggable security hooks)  
**MMIO** — Memory-Mapped I/O (device registers in address space)  
**NX** — No-Execute (page table flag, data pages not executable)  
**PCB** — Process Control Block (Process struct)  
**PI** — Priority Inheritance (futex prevents priority inversion)  
**SMAP** — Supervisor Mode Access Prevention (block kernel user-memory access)  
**SMEP** — Supervisor Mode Execution Prevention (block kernel user-code exec)  
**SROP** — Sigreturn-Oriented Programming (attack via forged sigframe)  
**TLB** — Translation Lookaside Buffer (page table cache)  
**TOCTOU** — Time-Of-Check-Time-Of-Use (race between validation and use)  
**VMA** — Virtual Memory Area (contiguous region in address space)  
**W^X** — Write-XOR-Execute (no page both writable and executable)

---

## Conclusion

Nilix is a production-oriented, security-first microkernel with:

- **Zero open CRITICAL and one open HIGH finding** (`R186-4`)
- **Comprehensive audit coverage** (187 R-series rounds, ~1,184 fixes/resolutions; plus the 2026-08-07 standalone full-codebase audit remediated 2026-08-27 — see [`docs/security-audit-status.md`](../../security-audit-status.md))
- **Defense-in-depth security** (hardware + software + runtime detection)
- **Rigorous lock ordering** (10-level hierarchy, documented)
- **Proven user-mode ABI** (static musl gate green)

**Next Milestone:** close `R186-4`, complete the zero-HIGH streak, and continue Phase U ABI coverage  
**Long-Term Vision:** Capability-first OS with de-privileged Linux personality (Phase U)

This architecture map is authoritative for reconstruction efforts. All subsystems, boundaries, and constraints are documented. Refer to `docs/review/nextplan/` for active development items.

---

**Document Version:** 1.0  
**Last Updated:** 2026-07-02  
**Maintainer:** Zero-OS Kernel Team  
**Status:** Authoritative Architecture Reference
