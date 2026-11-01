# Nilix versus Linux — Gap Analysis and Next Plan

**Date:** 2026-11-01
**Source baseline:** `e615f0d` (branch `fix/selective-wait-wakeup`, merged to `main` as `78dd11a` by PR #56). Line numbers refer to this tree and will drift.
**Linux baseline:** x86_64 syscall table from `kernel-headers-7.2.4` (`/usr/include/asm/unistd_64.h`); feature comparisons use long-stable mainline Linux behavior.
**Supersedes:** the May 2026 comparison at [overview/02-architecture/design-decisions/gap-analysis-vs-linux.md](overview/02-architecture/design-decisions/gap-analysis-vs-linux.md).
**Status:** analysis and plan rationale. The §8.2 amendments are in the private plan as v16.14, which remains the plan of record. Capability labels stay in the [roadmap](roadmap.md).

## 0. Method and limits

| Evidence | Meaning |
| --- | --- |
| **Table** | Extracted by script from the `sys_*` dispatch match in `kernel/kernel_core/syscall.rs` and compared with the Linux header. |
| **Source** | Read or traced in this tree, cited as `path:line`. |
| **Recorded** | Taken from a dated project record and not rerun here. |

**What was read.** These current status documents were read in full: roadmap, public plan pointer, security status, the 2026-09-13 plan (v16.13), the 2026-10-25 handoff, the May gap analysis, the gap implementation plan and the 2026-10-31 audit report. The 2026-10-27 QA record (its qualification section), the 2026-10-29 audit report and the ST-K4 design were read in part.

**What was not read.** The 251,000 lines of kernel Rust were not read line by line. They were covered with the extracted syscall table, a tree-wide census of `ENOSYS`, `EOPNOTSUPP`, `TODO` and deferred markers, and targeted reading of the code behind each claim below. Absence claims mean no matching code path or name was found by those searches. Older dated plans, audits and overview documents were not read individually; this analysis relies on the current plan's open-item conservation record (plan §8) to carry them forward.

**What was not run.** No kernel build, guest gate or stress profile ran for this document. Behavior claims come from source traces or dated records. The Linux source tree was not available on this host, so Linux feature rows are not checked against the 7.2 source.

**Publication.** `docs/` is a public repository. This document names capability gaps only. Unrepaired audit leads and hardening notes stay in the private records.

Gap tiers used below:

| Tier | Meaning |
| --- | --- |
| **T0** | Blocks 1.0-Preview under the project's recorded release conditions |
| **T1** | Needed to run unmodified static POSIX programs |
| **T2** | Needed for dynamic linking and event-driven servers |
| **T3** | Needed for containers |
| **T4** | Platform, hardware and hardening breadth |
| **ND** | Not planned without a named consumer (§8.5) |

## 1. Conclusions

1. **Nilix is a deeply audited kernel with a narrow ABI.** It dispatches 103 of 370 live Linux syscall numbers, and 7 of those always refuse. Memory, process and thread lifecycle depth is real; breadth is not.
2. **The biggest blockers to real programs are capacity limits, not missing features.** Executables above 256 KiB cannot be exec'd, the page cache holds 1 MiB, the kernel heap is 2 MiB and usable RAM stops at 1 GiB (§3). Adding syscalls without lifting these would not produce a usable system.
3. **The persistent filesystem is nearly append-only.** `Ext2Fs` implements `create` for regular files and `rename`. It has no `unlink`, `mkdir`, `symlink` or `truncate`, and a growing file can only allocate its 12 direct blocks (48 KiB at 4 KiB blocks).
4. **The recorded plan has a hole.** ST-K4 is expected to unblock the `block` and `combined` stress profiles, but `combined` fails on Ext2 truncate. It then needs Ext2 indirect allocation and unlink, none of which ST-K4 provides (§6.2).
5. **Several implemented subsystems cannot be reached from Ring 3.** No code constructs a namespace file descriptor, so `setns` has nothing to act on. No syscall changes a task's user ID, so the DAC, credential and user-namespace machinery cannot be fully exercised by a normal program. Native endpoint IPC has no syscall.
6. **No external conformance suite is in use.** The repository and docs contain no LTP, musl `libc-test`, xfstests or kselftest integration. Current ABI evidence is written by the project itself.
7. **Next plan:** keep the recorded order (ST-K4, then ST-K2-P2 race probes, then stress acceptance), insert Ext2 mutation and the thread-heartbeat root cause, then target one concrete outcome: static BusyBox running from the journaled disk, measured by external suites (§8).

## 2. Linux ABI surface

### 2.1 Coverage (Table)

| Measure | Count |
| --- | --- |
| Numbers in the Linux x86_64 header | 385 |
| Dead or removed numbers (`uselib`, `_sysctl`, `create_module`, `tuxcall` and similar) | 15 |
| **Live Linux numbers** | **370** |
| Linux-numbered dispatch arms in Nilix | 103 |
| Dispatched but always refusing | 7 (`link` EPERM; `chmod`, `fchmod` EOPNOTSUPP; `chown`, `fchown`, `lchown`, `statx` ENOSYS) |
| **Dispatched with an actual operation** | **96** (26% of live numbers) |
| Live numbers with no arm (return ENOSYS) | 267 |
| Private numbers (500-524) | 24, of which the 6 `kpatch` calls are unsupported and the 5 `kcov` calls need the `kcov` feature and host-root authority |

A dispatched number means some operation exists, not that its semantics match Linux (§2.3).

### 2.2 Missing numbers by family (Table)

The rows total 282. Tier is the earliest tier that needs most of the family.

| Family | Missing | Examples | Tier |
| --- | --- | --- | --- |
| `*at` paths, metadata, xattr | 48 | `unlinkat`, `mkdirat`, `readlinkat`, `faccessat`, `utimensat`, `fchownat`, `truncate`, `fchdir`, `statfs`, `getxattr` | T1 |
| Event and fd objects | 21 | `epoll_*`, `eventfd2`, `inotify_*`, `memfd_create`, `pidfd_*`, `userfaultfd` | T2 |
| Memory management | 24 | `madvise`, `msync`, `mincore`, `mlock*`, `process_vm_*`, `mbind`, `pkey_*`, `mseal` | T1/T2 |
| Durability and file I/O | 19 | `fsync`, `fdatasync`, `sync`, `sync_file_range`, `flock`, `fallocate`, `sendfile`, `splice`, `preadv` | T0/T1 |
| Timers and clocks | 18 | `alarm`, `setitimer`, `timer_*`, `clock_getres`, `clock_nanosleep`, `timerfd_*` | T1/T2 |
| Scheduling and accounting | 18 | `sched_setscheduler`, `setpriority`, `getcpu`, `getrusage`, `times`, `sysinfo` | T1 |
| SysV and POSIX IPC | 18 | `shmget`, `semop`, `msgsnd`, `mq_open` | T2/T3 |
| Process and thread API | 16 | `vfork`, `clone3`, `rseq`, `membarrier`, `futex_waitv` | T2 |
| Identity and credentials | 14 | `setuid`, `setresuid`, `getresuid`, `setgroups`, `capget`, `capset` | T1 |
| Mount API | 12 | `mount`, `umount2`, `fsopen`, `move_mount` | T3 |
| Signals and restart | 10 | `sigaltstack`, `rt_sigsuspend`, `rt_sigtimedwait`, `rt_sigpending`, `pause`, `signalfd4` | T1 |
| Modules, kexec, power, host admin | 10 | `init_module`, `kexec_load`, `reboot`, `swapon`, `sethostname` | T4/ND |
| Socket API | 9 | `getsockname`, `getpeername`, `setsockopt`, `getsockopt`, `sendmsg`, `recvmsg`, `socketpair` | T1 |
| Async I/O | 9 | `io_setup`, `io_uring_*` | ND |
| Security and keys | 9 | `keyctl`, `landlock_*`, `lsm_*` | T3/T4 |
| Sessions and job control | 6 | `setsid`, `setpgid`, `getpgid`, `getsid` | T1 |
| Tracing and programmability | 6 | `ptrace`, `perf_event_open`, `bpf`, `syslog` | T4/ND |
| Dead or removed numbers | 15 | Not a gap | — |

### 2.3 Dispatched calls with narrower semantics (Source)

| Call | Current contract | Evidence |
| --- | --- | --- |
| `mmap` | Exactly `MAP_PRIVATE\|MAP_ANONYMOUS` or `MAP_SHARED\|MAP_ANONYMOUS`, optionally `MAP_FIXED_NOREPLACE`. `MAP_FIXED`, `MAP_STACK`, `MAP_NORESERVE`, `MAP_POPULATE` and any file descriptor are refused | `syscall.rs:17238`, `validate_mmap_flags` |
| `mremap` | Private anonymous only; `MREMAP_FIXED` and `MREMAP_DONTUNMAP` refused | Recorded, roadmap §5.1 |
| `open`/`openat`/`fstatat`/`renameat` | A relative path with a real directory fd returns EOPNOTSUPP | `syscall.rs:21120`, `:21258`, `:21332` |
| `renameat2` | `RENAME_NOREPLACE` only | `syscall.rs:21138` |
| `readv` | Reads only the first non-empty segment | `syscall.rs:13619` |
| `clone` | `CLONE_VFORK`, `CLONE_PTRACE` and `CLONE_PARENT` refused | `syscall.rs:5099` |
| `waitid` | No `P_PGID`, `WUNTRACED` or `WCONTINUED` (EINVAL) | Recorded, roadmap §5.2 |
| `fcntl` | `F_DUPFD`, `F_DUPFD_CLOEXEC`, `F_GETFD`, `F_SETFD`, `F_GETFL`, `F_SETFL` only; no record locks | `sys_fcntl` |
| `ioctl` | `TCGETS`, `TCSETS*`, `TIOC[GS]WINSZ`, `FIONREAD` on the console; all else ENOTTY. No block, socket or interface ioctls | `sys_ioctl` |
| `prctl` | `NO_NEW_PRIVS` and seccomp get/set, plus test-only knobs | `sys_prctl` |
| `socket` | `AF_INET` stream and datagram | `net/src/socket.rs:1174` |
| `setns` | Mount namespace type only, and no fd source exists (§4.10) | `syscall.rs:11774` |
| `clock_gettime` | Every accepted clock reads one milliseconds-since-boot counter | `syscall.rs:21985` |
| `futex` | All keys are private to an address space, so no cross-process shared futex | `ipc/futex.rs:2628` |
| `execve` | `ET_EXEC` only; `#!` scripts handled | `elf_loader.rs:565`, `syscall.rs:8426` |

## 3. Capacity ceilings

These limits follow from the project's admission and fail-closed design and are not defects. They are still the most important gap, because each one sets a hard upper bound for every workload (Source).

| Resource | Nilix | Linux | Effect | Evidence |
| --- | --- | --- | --- | --- |
| Executable file | **256 KiB**, staged whole in the kernel heap | Demand-paged from the file | Larger binaries, typically including static BusyBox builds, cannot exec | `syscall.rs:2946`, `mm/heap_budget.rs:134` |
| ELF `PT_LOAD` | 1 MiB per segment, 32 segments | No comparable limit | Larger text or data refused | `elf_loader.rs:73`, `:205` |
| Page cache | **256 pages (1 MiB)** | All free RAM | File I/O effectively uncached | `mm/page_cache.rs:34`, `heap_budget.rs:121` |
| Kernel heap | **2 MiB**, one `linked_list_allocator` | Slab plus vmalloc, scaling with RAM | Caps every kernel object family | `mm/memory.rs:328` |
| Usable RAM | **Under 1 GiB**: one window inside the direct map | Terabytes | Larger guests boot but cannot use the rest | `mm/memory.rs:420` |
| File growth on Ext2 | **48 KiB** by allocation (12 direct blocks at 4 KiB) | Effectively none | No large new files | `vfs/ext2.rs:12477` |
| Descriptors | 256 per process | `nofile` tunable to over 1 million | Servers refused | `process.rs:432` |
| Conntrack | **256 flows**, 64 per namespace | Scales with RAM, tunable | Low connection ceiling while the firewall is active | `net/src/conntrack.rs:103`, `heap_budget.rs:115` |
| CPUs | 64 | Up to 8192 by configuration | Rejected above the ceiling | `cpu_local/lib.rs:35` |
| PIDs | 32,768 fixed | Tunable to about 4 million | Adequate for now | `process.rs:826` |

**Eager allocation.** `brk`, private `mmap`, `mprotect` Path A and `mremap` growth back every page before returning (`kernel_core/populate.rs:1`). Only shared-anonymous regions and stack growth are demand-paged. Linux overcommits and faults lazily, so runtimes that reserve large sparse ranges will fail here even when they would touch little of the range.

## 4. Subsystem gaps

State labels follow [roadmap §0](roadmap.md#0-how-to-read-status), plus **Absent** for no code path.

### 4.1 Boot, CPU and platform

| Capability | Nilix | Linux | Tier |
| --- | --- | --- | --- |
| Architectures | x86_64 only | Over 20 | ND |
| Boot | UEFI loader with KASLR, 256-byte command line; no BIOS or Secure Boot path found | UEFI, BIOS, Secure Boot chain, many loaders | T4 |
| ACPI | RSDT/XSDT, MADT, HPET, DMAR only | Full ACPI with AML interpreter | T4 |
| Extended FPU state | **Absent.** FXSAVE only; no XSAVE or XCR0 code | XSAVE family, AVX, AVX-512, AMX | T2 |
| x2APIC | Unsupported; firmware-enabled x2APIC is refused (`arch/apic.rs:729`) | Supported | T4 |
| PCI | Legacy port I/O config; virtio-net scan covers buses 0-7 (`net/src/pci.rs:69`); no ECAM | Full PCIe, ECAM, hotplug, AER | T4 |
| Power | Ring-0 shell `reboot` and `poweroff` only; no `reboot(2)` | ACPI power states, suspend, cpufreq, cpuidle | T4 |
| CPU hotplug | Absent | Supported | ND |
| Machine check | Exception counter seen; no decode or recovery found | MCE and EDAC | T4 |
| Early-boot W+X | Open debt U55-6 (Recorded) | Tightly controlled | T4 |

### 4.2 Memory management

| Capability | Nilix | Linux | Tier |
| --- | --- | --- | --- |
| Buddy, COW fork, page tables, TLB shootdown, PCID | Implemented, with heavy guest and audit evidence | Same core | At parity for this subset |
| Fallible admission and per-class heap budgets | Implemented; stricter than Linux | Overcommit and OOM | Nilix ahead |
| Slab, NUMA, swap, THP | Primitives only. `mm/capabilities.rs` has no consumer except a self-test | Full subsystems | T2 (slab), ND (others) |
| Lazy private anonymous memory | Absent (§3) | Default | T2 |
| File-backed `mmap` | **Absent**, fail-closed | Core feature | **T2** |
| `MAP_FIXED` replacement | Refused | Core feature; needed by loaders | **T2** |
| `madvise`, `msync`, `mlock`, `mincore` | Absent | Present | T1 |
| Shared-anonymous `mremap` | Refused | Present | T2 |
| Page cache as primary I/O path | 1 MiB; write-through Ext2; no production writeback caller | Dirty writeback with flusher threads | T2 |
| Reclaim, compaction, KSM, memory tiering | Cache reclaim and OOM killer only | Full | ND beyond reclaim |
| KASAN or KFENCE equivalent | Absent; hosted fuzz builds use ASan only | Present | T4 |

### 4.3 Processes, threads and signals

| Capability | Nilix | Linux | Tier |
| --- | --- | --- | --- |
| fork, exec, exit, wait4, waitid, reaping | Tested subset | Full | Depth comparable for the subset |
| Thread groups, TLS, musl pthreads on 1-16 CPUs | Tested subset, default on since `c62b278` (Recorded) | NPTL | Core present |
| Process groups, sessions, controlling TTY | **Absent**; no `pgid` or session state in the PCB | Full job control | **T1** |
| User ID changes, setuid exec | **Absent**; no `set*id` syscalls and no `S_ISUID` handling | Full | **T1** |
| `SA_RESTART`, `sigaltstack`, `SA_ONSTACK` | Accepted without the semantics, or absent (`signal.rs:269`) | Full | **T1** |
| `rt_sigsuspend`, `rt_sigtimedwait`, queued RT signals, full `siginfo` | Absent or minimal | Full | T1 |
| Signal handler during a blocked `LOCK_PI` | Handler waits until the call returns (Recorded) | Restart machinery | T1 |
| `ptrace` | Absent | Full | T4 |
| `rseq`, `membarrier`, `clone3`, `vfork` | Absent | Present | T2 |
| Dynamic ELF, `PT_INTERP`, user PIE, vDSO, stack ASLR | **Absent.** The stack address is fixed; only the mmap base is randomized | Standard | **T2** |

### 4.4 Scheduler

| Capability | Nilix | Linux | Tier |
| --- | --- | --- | --- |
| Per-CPU run queues, work stealing, balancing, affinity, cpuset | Implemented; 2/4/8/16-CPU QUALIFIED (Recorded) | Full topology-aware domains | Core present |
| Policy | MLFQ with 256 priority buckets | EEVDF, RT, deadline, `sched_ext` | T1 (API) / ND (classes) |
| `nice`, `setpriority`, `sched_setscheduler` | `nice` field with no setter | Full | T1 |
| Kernel preemption | **Non-preemptible kernel** (Recorded, plan §3.4) | Configurable up to full preemption | T2 |
| Tick | Fixed 1 kHz; no tickless mode found | NO_HZ | ND |
| Deferred work | No softirq, workqueue or general kthread framework found; RX runs in process context | Full | T2 |
| Latency under load | Thread profile heartbeats reached 94-114 s at 16 vCPUs, bound 18 s (Recorded 2026-10-27) | Not comparable | **T0** |

### 4.5 Time

| Capability | Nilix | Linux | Tier |
| --- | --- | --- | --- |
| Wall clock | **Absent.** No RTC read found; `CLOCK_REALTIME` is time since boot | RTC, NTP adjustment | **T1** |
| Clock resolution | 1 ms for all clocks | Nanosecond | T1 |
| Interval and POSIX timers, `alarm`, `timerfd` | Absent | Full | T1/T2 |
| `settimeofday`, `clock_settime`, `adjtimex` | Absent | Full | T2 |

### 4.6 IPC

| Capability | Nilix | Linux | Tier |
| --- | --- | --- | --- |
| Pipes | Tested; capability-backed | Same plus splice | Core present |
| Futex | WAIT, WAKE, REQUEUE, WAKE_OP, bitsets, PI, requeue-PI, robust | Same plus cross-process keys, `futex_waitv` | T2 for shared keys |
| Named FIFOs | Absent (no `mknod`) | Present | T1 |
| `AF_UNIX` | **Absent** | Core local IPC | **T2** |
| `epoll`, `eventfd`, `timerfd`, `signalfd` | **Absent** | Core server APIs | **T2** |
| SysV IPC, POSIX queues | Absent; an IPC namespace object exists with nothing to isolate | Present | T2/T3 |
| Native endpoint IPC | Code exists (`ipc/ipc.rs`), no syscall arm | Not applicable | Project goal, deferred (§8.4) |

### 4.7 VFS and filesystems

| Capability | Nilix | Linux | Tier |
| --- | --- | --- | --- |
| VFS object model | `FileSystem` and `Inode` traits have no `link`, `mknod`, `setattr`, `chmod`, `chown` or xattr methods (`vfs/traits.rs`) | Full inode and dentry operations | **T1** |
| Dentry and inode caches | Absent | Core | T2 |
| ramfs | Create, unlink, rename, symlink, truncate | tmpfs with quotas and swap | Core present |
| Ext2 read | Direct and indirect mapping, directories | Same | Core present |
| Ext2 write | In-place to mapped blocks; direct-block allocation; `create` for regular files; `rename`. **No `unlink`, `mkdir`, `symlink`, `truncate`** (`ext2.rs:11519`, `:11567`, `:12743`) | Full | **T0** (see §6.2) |
| Plain Ext2 read-write | Refused: writable mounts require a journal (`ext2.rs:5697`) | Allowed | Deliberate |
| JBD2 | Constrained ordered-data, self-checkpointing inline commits | Full, asynchronous | T2 |
| ext4 | Refused; only `FILETYPE` and `RECOVER` incompatible features accepted (`ext2.rs:83`) | Primary filesystem | T4 |
| Mount from Ring 3 | **Absent.** `/`, `/dev`, `/proc`, `/sys`, `/sys/fs/cgroup` and `/mnt` are fixed at boot (`vfs/operations.rs:73`) | Full | **T3** |
| Permission changes | `chmod`, `chown`, `link`, `statx`, `utimensat` refused or absent | Full | **T1** |
| Directory-fd paths | Refused except `AT_FDCWD` and absolute | Full | **T1** |
| File locks | Absent | Advisory and OFD locks | T1 |
| procfs | `cmdline`, `cpuinfo`, `fd`, `maps`, `meminfo`, `schedstat`, `self`, `stat`, `status`, `task`, `uptime`, `version`, `waitstat`. No `mounts`, `ns`, `net`, `sys`, `limits` or `exe` | Hundreds of entries | T1/T3 |
| devfs | `null`, `zero`, `console` and block nodes. No `random`, `urandom`, `tty`, `ptmx` or `pts` (`devfs.rs:36`) | Full | **T1** |
| sysfs | `/sys` is a ramfs holding the cgroup mount point | Device model | T4 |
| Other filesystems | None (no overlayfs, FUSE, NFS, 9p, virtiofs) | Many | T3 (overlay), ND (rest) |
| inotify, fanotify | Absent | Present | T2 |
| Terminal | Console ioctls only; no line discipline or pty | Full tty layer | T1/T2 |

### 4.8 Block and storage

| Capability | Nilix | Linux | Tier |
| --- | --- | --- | --- |
| Virtio-blk, owned BIO, 512B/4096B geometry | Tested | Same | Core present |
| Durability syscalls | **Absent.** `FLUSH` is negotiated and the Ext2 write path flushes, but `fsync` and friends have no arm | Full | **T0** |
| Crash-recovery proof | No recovery workload; the block profile refuses to run (`stress_runner.c:1707`) | xfstests and fsstress in wide use | **T0** |
| Request layer | Single-device path; no I/O scheduler or multi-queue layer | blk-mq, schedulers | ND |
| Device mapper, loop, RAID, NVMe, AHCI | Absent | Present | T4 (NVMe, AHCI), ND (rest) |

### 4.9 Networking

| Capability | Nilix | Linux | Tier |
| --- | --- | --- | --- |
| IPv4, ICMP, ARP, fragments, UDP, TCP with NewReno, window scaling, SYN cookies | Implemented; hosted net 139 tests (count pin) | Full | Core present; interoperability unqualified |
| Conntrack and default-DROP firewall | Implemented, capped at 256 flows | Netfilter and nftables | Core present |
| **Loopback and self-addressed traffic** | **Fails closed** (`net/src/stack.rs:1610`, `:1783`) | Core | **T1** |
| `getsockname`, `setsockopt`, `sendmsg`, `recvmsg` | **Absent** | Core | **T1** |
| Address and route configuration | Static defaults for QEMU SLIRP (`stack.rs:1451`); no interface ioctls, no netlink, no route table, no DHCP | Full | T2 |
| IPv6 | Absent (ethertype constant only) | Full | T2 |
| `AF_PACKET`, raw sockets, `AF_NETLINK` | Absent | Full | T3 |
| veth, bridge, VLAN, tunnels, NAT | Absent | Full | T3 (veth, NAT), ND (rest) |
| Offloads, multi-queue, NAPI | Absent; RX runs in process context | Full | ND until a performance baseline exists |
| NIC drivers | Virtio-net only | Thousands | T4 |
| Wire interoperability | No independent packet oracle (NET-QUAL, Recorded) | Deployed worldwide | T2 |

### 4.10 Namespaces, cgroups and containers

| Capability | Nilix | Linux | Tier |
| --- | --- | --- | --- |
| PID, mount, IPC, net and user namespace objects | Implemented with transactional rollback | Same | Core present |
| UTS, cgroup and time namespaces | Absent | Present | T3 |
| Namespace file descriptors | **Unreachable.** No code constructs `MountNamespaceFd` or `NetNamespaceFd` (`syscall.rs:12004`), and there is no `/proc/<pid>/ns` | Standard | **T3** |
| cgroupfs files | `cgroup.controllers`, `cgroup.procs`, `cgroup.subtree_control`, `cpu.max`, `cpu.weight`, `io.max`, `io.stat`, `memory.current`, `memory.high`, `memory.max`, `pids.current`, `pids.max`, plus private `files.*` and `ports.*` | Also `cgroup.events`, `cgroup.stat`, `cgroup.freeze`, `cpu.stat`, `memory.stat`, `memory.events`, `cpuset.*` and others | T3 |
| Container networking | Child-namespace RX, veth, routes and per-namespace administration open; device transfer disarmed | Standard | **T3** |
| OCI runtime | Not attempted | Standard | T3 |

### 4.11 Security

| Capability | Nilix | Linux | Tier |
| --- | --- | --- | --- |
| Usercopy discipline, W^X, SMEP/SMAP/UMIP, KASLR, kptr redaction | Implemented and audited | Same | At parity for this subset |
| Generation-checked capabilities, pledge, hash-chain audit | Implemented | No direct equivalent | Nilix ahead in design |
| Seccomp | Strict and filter; TSYNC rejected | Full, including user notification | T2 |
| LSM | 56 `hook_*` entry points; four built-in policies including one test policy; policy fixed at boot | Several hundred hooks; SELinux, AppArmor, Smack, Landlock, BPF LSM | T4 |
| POSIX capabilities API | Absent; ambient gates reduce to host root | Full | T1/T3 |
| Full KPTI | **Unsupported.** `FULL_KPTI_ISOLATION_SUPPORTED = false` (`security/kaslr.rs:100`) | Supported | T4 |
| Compiler retpoline | Unsupported; the feature fails compilation (Recorded) | Supported | T4 |
| CET, shadow stacks, protection keys | Absent | Present | T4 |
| Keyring, IMA, EVM, dm-verity, fs-verity | Absent | Present | ND |
| Livepatch | Unsupported, ENOSYS | Supported | ND |

### 4.12 Observability and debugging

| Capability | Nilix | Linux | Tier |
| --- | --- | --- | --- |
| KCOV, counters, tracepoints, watchdog, profiler, serial kdump | Implemented | Same families and more | Core present |
| `/proc/schedstat`, `/proc/waitstat` | Implemented | Richer equivalents | Core present |
| ftrace, kprobes, perf, eBPF | Absent | Full | ND, except a minimal perf baseline |
| `dmesg`/`syslog(2)` | Absent; serial console only | Full | T1 |
| Durable or remote audit backend | Absent (Recorded) | auditd | T4 |

### 4.13 Drivers

Implemented: virtio-blk, virtio-net, VGA and framebuffer console, serial, PS/2 keyboard, APIC, HPET, VT-d (QEMU-qualified scope). Absent: driver model, loadable modules, AHCI, NVMe, physical NICs, USB, graphics, input layer and sound. This is the largest gap by volume and the least urgent while QEMU is the target. A small physical set (NVMe, AHCI, one Intel NIC) becomes relevant only when physical qualification is scheduled.

### 4.14 Userspace and testing

| Capability | Nilix | Linux ecosystem | Tier |
| --- | --- | --- | --- |
| Static musl, including pthreads | Tested | Works | Core present |
| glibc, dynamic linking, BusyBox, distributions | Not established | Standard | T1/T2 |
| External conformance suites | **Absent** | LTP, kselftest, xfstests, libc-test | **T1** |
| Fuzzing | 11 scheduled libFuzzer campaigns and a QEMU KCOV executor | syzkaller at scale | Good for project size |
| Guest runtime tests | `make test` 35 passed / 39 deferred / 0 failed; boot 28/46/0; SMP-4 37/37/0 (Recorded) | Not comparable | **T0**: deferrals block strict qualification |

## 5. Where Nilix is deliberately different

These should be preserved while closing gaps. They are design strengths, not evidence of overall maturity relative to Linux.

- Rust kernel with explicit `unsafe` boundaries and exact usercopy fixups.
- Fallible, admitted kernel allocation with per-class budgets. Linux has no equivalent system-wide contract.
- Unsupported operations fail closed with stable errors, backed by census tests for hooks and pledge parity.
- Generation-checked capabilities, pledge-style promises and a tamper-evident audit chain.
- A dense audit and independent review history. The zero-clean-round streak shows this process is still finding defects.

The roadmap's long-term goal of moving the Linux personality into less-privileged userspace also differs from Linux. Today, services run in Ring 0 ([roadmap §3.1](roadmap.md#31-current-execution-and-planned-personality)), so Nilix is a monolithic kernel in practice.

## 6. Project status

### 6.1 Delivered since the last roadmap revision (2026-09-14)

| Area | Delivered | Source |
| --- | --- | --- |
| Threads | Thread groups on by default, exec from any thread, shared tables, 16-CPU evidence | Plan v16.13 |
| Futex | Linux opcodes for private keys, including `WAKE_OP` and requeue-PI; PI ABBA `EDEADLK` detection still missing | `syscall.rs:19915`, README |
| Boot | RAM above 1 GiB no longer panics (F8, F9) | Plan §9 |
| Isolation | R192 identity-map removal, kernel-stack window repair | Plan §3.3, §3.4 |
| MM | Bounded eager population, `MAP_FIXED_NOREPLACE`, relocation staging, shared-COW fix | Plan §3.3 |
| Console | Serial lock (TD-SERIAL-1), console input ownership | Plan §3.3, PR #49 |
| Audits | R190-R192 rounds; 2026-10-29 and 2026-10-31 audit runs with repairs merged | Issues #42, #44, #48; PR #49 |
| Wait | Exact child wait filters published before blocking | PR #56 |

Hosted test pins now total 616 (`scripts/tools/hosted_subcrate_tests.sh`).

### 6.2 Release conditions today

| Condition | State |
| --- | --- |
| Clean full audit rounds | **0/3.** Both October audit runs had confirmed findings and reported partial coverage; the latest left three units deferred |
| R188 lineage | 123 PASS, 3 PARTIAL, 5 verification pending (Recorded 2026-09-13) |
| R190 and R191 review-fix | Handled outside the plan by user decision; status per external records |
| Strict guest qualification | Open; 39 runtime deferrals, plus ST-K2-P2 races |
| Stress profiles | See below |
| Extended CI | Last three runs not green: failure on 2026-10-18 (twice), cancelled on 2026-10-25 after about 3 hours |
| Main CI | `78dd11a` passed |

Stress state, recorded 2026-10-27 and not rerun on this tree:

| Profile | Result | What is blocking it |
| --- | --- | --- |
| memory | QUALIFIED | — |
| cpu | QUALIFIED (85 rounds after accounting fix) | — |
| smp | QUALIFIED | — |
| process | QUALIFIED (1,339 rounds after oracle fix) | — |
| thread | **FAIL** at 16 vCPUs | Heartbeat gaps of 94-114 s; no root cause recorded |
| block | **FAIL** | `fsync` absent; the workload itself is a refusal stub (`stress_runner.c:1707`) |
| combined | **FAIL** | `combined_open` ENOSYS; see the trace below |

**Root cause of `combined_open` (Source, not rerun).** The workload opens `/mnt/test/stress-combined.tmp` with `O_RDWR|O_CREAT|O_TRUNC` (`stress_runner.c:1342`). The open path calls `file.inode.truncate(0)` (`vfs/manager.rs:462`). `Ext2Inode` does not override `truncate`, so the default returns `FsError::NotSupported` (`vfs/traits.rs:270`), which maps to `ENOSYS` (`vfs/manager.rs:361`). This matches the recorded errno. Two further failures follow once that is fixed:

1. The profile writes 16 × 4 KiB (64 KiB) per round (`stress_protocol.py:442`), so the 13th write reaches an unsupported indirect hole.
2. It then calls `unlink` (`stress_runner.c:1366`), which `Ext2Fs` does not implement.

ST-K4 as designed adds none of these. The block profile fits within direct blocks (12 slots of 4 KiB), so ST-K4 alone can complete it, but the guest workload has to be written.

## 7. Documentation drift found

These rows were corrected on 2026-11-01 (DOC-SYNC), except the ST-K4 design, which is refreshed as the first step of that item. Their old text is kept here because it explains why earlier planning missed some of these gaps.

| Document | Problem | Correct state |
| --- | --- | --- |
| [roadmap.md](roadmap.md) header | Revision 6.1, 2026-09-14, links plan v16.4 | Plan is v16.13 |
| roadmap §1, §13 | 438 hosted tests | 616 pinned |
| README | "515 hosted unit tests" | 616 pinned |
| roadmap §5.3 | Lists `WAKE_OP` and requeue-PI as available, then says they are refused | Available for private keys (`syscall.rs:19915`) |
| roadmap §5.3 | "No completed epoll/eventfd/timerfd surface" | Correct, but understates it: no code exists |
| roadmap §6 | "restricted clone" | Thread groups accepted in every build |
| [next-phase-plan.md](next-phase-plan.md) | Points to v16.12 | v16.13 |
| [security-audit-status.md](security-audit-status.md) | Header dated 2026-09-14; no R190-R192 or October runs | Needs update |
| [docs/README.md](README.md) | "September 12 rollover"; links `stress-gate-status.md`, which does not exist | v16.13; remove or restore the link |
| Roadmap and plan | "Six" stress profiles; README and `stress_test.sh` have seven | Seven, including `thread` |
| README scheduler row | Soaks qualify on 4 vCPUs, and lists process as missing | Process qualified 2026-10-27; thread fails at 16 vCPUs |
| Plan §5 | IDs F10 and F11 mean different things than GAP-6 F10 and F11 | Rename one set |
| ST-K4 design | Says it unblocks `combined` | It does not (§6.2) |
| May 2026 gap analysis | Claims such as hot patching present and 30-40% coverage | Superseded by this document |
| CONTRIBUTING | "hybrid microkernel" | Services run in Ring 0 today |

## 8. Next plan

### 8.1 Decisions

1. **Keep the recorded order** (user decision, 2026-10-25): ST-K4, then ST-K2-P2 race probes, then stress acceptance. Nothing found here invalidates it.
2. **Add the missing prerequisites** instead of weakening stress workloads. Pointing `combined` at ramfs or shrinking its writes would pass the gate while hiding the gap, so that option is rejected.
3. **Count seven stress profiles.** The superset is the safer reading of the "all profiles" condition.
4. **After Preview work, go for outcomes rather than syscall counts.** Each milestone names a real program and an external suite.
5. **Lift capacity ceilings before widening APIs that depend on them.**
6. **ABI breadth comes before native IPC and the userspace personality.** A personality can only be validated against programs that already run. This changes the sequence in plan §6 (F-10 listed personality first) and leaves F-1b through F-2 in the backlog.
7. **Roadmap policy stands:** capability milestones, no calendar commitments.

### 8.2 Amendments to plan v16.13 (applied as v16.14)

| ID | Change | Reason |
| --- | --- | --- |
| **EXT2-MUT** (new, T0) | Ext2 truncate, indirect-block allocation and unlink as journaled transactions; `mkdir` and `symlink` follow | Required by `combined`; §6.2 |
| **THREAD-HB** (new, T0) | Root-cause the thread heartbeat gaps before changing any bound | Recorded failure with no owner |
| **EXT-CI** (new, T0) | Make `Extended tests` complete within its budget | It is the acceptance vehicle and is not green |
| **STRESS-7** (rename) | STRESS-6 becomes seven profiles | §7 |
| **DOC-SYNC** (new) | Correct the §7 rows | Current docs mislead planning |
| **ST-K4 scope note** | State that the guest block workload is in scope and `combined` is not | §6.2 |

### 8.3 Milestone N1 — Preview chain (T0)

Serial order; each item names its design need per [plan §3](next-phase-plan.md).

| Order | Item | Work | Acceptance |
| --- | --- | --- | --- |
| 0 | DOC-SYNC | Done 2026-11-01: §7 rows fixed; October audit runs recorded in the security status | Link check; counts match the hosted script |
| **1 (NEXT)** | **ST-K4** | Refresh the design on current `main`; `fsync`, `fdatasync`, `sync`, `sync_file_range` bound to `FileHandle`; honest flush reporting; seccomp and pledge census; real guest block workload | `block` QUALIFIED for 900 s with QMP kill, recovery and `e2fsck`; injected flush failure reports EIO |
| 2 | EXT2-MUT | Slices: truncate; indirect allocation; unlink with orphan handling. Design first, with independent review before and after code | `combined` passes its I/O stage; crash-injected journals recover; `e2fsck` clean; hosted plus Ring-3 legs |
| 3 | ST-K2-P2 race probes | Same-CPU exception reentry, cross-CPU holder and shootdown ACK, final-owner and parent-first unmap, private rollback | Probes run in guest; P0-A strict qualification closes |
| 4 | THREAD-HB | Use `/proc/schedstat` and stall dumps to locate the stall; consider non-preemptible holds and `PT_LOCK` contention | `thread` QUALIFIED at 4 and 16 vCPUs with the existing bound, or a bound changed only with evidence |
| 5 | EXT-CI + STRESS-7 | Fit the Extended budget; run all seven profiles on one tree | All seven QUALIFIED on one source identity |
| 6 | STRICT-Q | Give each runtime deferral an owner: implement, or reclassify with a reason. Reconcile P1-A and R188 residuals. Includes the placeholder bodies `r23_1_cow_tlb_shootdown` and `r174_b4_brk_va_reservation_toctou` | Strict profile passes for the declared scope |
| 7 | Audit rounds | Run on a frozen tree, including the three deferred audit units | Three clean full rounds, counted by the external process |

ST-K2-P2 and THREAD-HB can run alongside items 1-2 when no shared files are touched. Items 1 and 2 share the Ext2 commit path and must stay in series.

### 8.4 Later milestones

**N2 — Static POSIX baseline (T1).** Outcome: unmodified static BusyBox (`ash` and core applets) runs from the journaled disk with a working shell pipeline, job control and file manipulation.

| Slice | Content |
| --- | --- |
| N2.0 Syscall table | One generated table drives dispatch, pledge and filter lists (the implementation plan's §13 intent). Split `syscall.rs` (27,419 lines) by family, keeping its census tests working |
| N2.1 Exec limits | Stage exec images in frames instead of the heap; raise the 256 KiB and 1 MiB segment limits; raise the fd ceiling |
| N2.2 Conformance harness | Add musl `libc-test` and an LTP `syscalls` subset with a committed, count-pinned pass list, following existing gate conventions |
| N2.3 Paths and metadata | General directory-fd resolution, `*at` family, `fchdir`, `truncate`, VFS `setattr`, real `chmod`, `chown`, `statx`, `utimensat`, hard links, `statfs`, FIFOs. Remaining Ext2 `mkdir` and `symlink` if not already done |
| N2.4 Identity | `set*id`, `getres*`, groups, setuid-bit exec, `capget`, `capset`, with audit coverage of the now-reachable DAC paths |
| N2.5 Job control | Process groups, sessions, controlling terminal, `WUNTRACED`, `WCONTINUED`, `P_PGID` |
| N2.6 Signals and time | `SA_RESTART`, `sigaltstack`, `rt_sigsuspend`, `rt_sigtimedwait`, queued RT signals, `alarm`, itimers, POSIX timers, RTC wall clock, nanosecond clocks |
| N2.7 Device and proc surface | `/dev/urandom`, `/dev/random`, `/dev/tty`, `/proc/mounts`, `/proc/<pid>/exe`, `getrusage`, `times`, `sysinfo`, `setpriority`, `getcpu`, `syslog` |
| N2.8 Socket basics | Loopback delivery, `getsockname`, `getpeername`, `setsockopt`, `getsockopt`, `sendmsg`, `recvmsg` |

Acceptance: a BusyBox boot gate plus an LTP and libc-test pass list that only grows.

**N3 — Memory and dynamic userspace (T2).** Outcome: dynamically linked musl and glibc programs run.

Order: heap and slab scaling (`HEAP-SCALE`, with a named consumer for `SlabCache`) → direct map and buddy above 1 GiB → page cache sizing with real writeback → lazy private anonymous memory with a stated overcommit policy → `MAP_FIXED`, `madvise`, `msync`, `mlock` → file-backed private, then shared, `mmap` → `ET_DYN`, `PT_INTERP`, stack ASLR, vDSO → XSAVE state → kernel-preemption or long-hold review.

**N4 — Event-driven servers (T2).** Outcome: an unmodified `nginx` or `redis`, with `curl` as client, passes a scripted run with an independent packet capture.

Content: `AF_UNIX`, `epoll` on a common wait-key contract, `eventfd`, `timerfd`, `signalfd`, `memfd_create`, `rseq`, `membarrier`, `clone3`, interface configuration, DHCP or static route table, NET-QUAL wire oracle, higher conntrack and socket budgets, IPv6 as a separate design, PERF-BASELINE.

**N5 — Containers (T3).** Outcome: `runc` or `crun` starts a BusyBox OCI image under QEMU with network connectivity.

Content: `mount` and `umount2`, namespace fds and `/proc/<pid>/ns`, UTS and cgroup namespaces, complete `setns`, seccomp TSYNC, veth and child RX, per-namespace firewall and routes, NAT, cgroup v2 completion and delegation, ST-K1 root membership, overlayfs, SysV IPC.

**N6 — Platform and hardening (T4).** Each is qualified independently and depends on external prerequisites.

Content: full KPTI (U37-1b), retpoline, CET, early W+X (U55-6), x2APIC, ECAM, MSI-X, NVMe, AHCI, one physical NIC, physical VT-d (P3-2), ptrace for debuggers, durable audit, native IPC and personality (F-1b through F-10) re-evaluated against then-current needs.

### 8.5 Not planned without a named consumer

Non-x86 architectures, loadable modules, eBPF, `io_uring`, KVM host, swap, NUMA policy, ext4, btrfs and XFS, device mapper, keyring, IMA, CPU hotplug, livepatch enablement.

`io_uring`, eBPF and modules each add a large attack surface, which conflicts with the Safety > Correctness > Efficiency > Performance rule, so they should stay out until a consumer justifies the review cost.

### 8.6 Rules for every new ABI slice

- One per-item design for MM, IRQ, ABI, security or lifetime changes, with independent review for high-risk items.
- Admission before allocation, a failure matrix with exact rollback and no blocking under registry locks.
- LSM and seccomp disposition before shipping, with census lists updated together.
- Stable errno and no side effects for any refused form, with a negative test.
- A Ring-3 leg through the real syscall entry, hosted pins updated, and external suite results where they apply.
- Roadmap and README updated in the same change, keeping the labels honest.

### 8.7 Immediate next steps

DOC-SYNC and the §8.2 amendments are applied, and `78dd11a` passed CI. ST-K4 is implemented on branch `feat/st-k4-durability` and awaits review and its 900-second run. This document stays a 2026-11-01 snapshot: its tables still show the pre-ST-K4 state.

## 9. References

[Roadmap](roadmap.md) · [Next-phase pointer](next-phase-plan.md) · [Security status](security-audit-status.md) · [CI guide](ci-testing.md) · [Quality gates](quality-gates.md) · [VT-d matrix](vtd-support-matrix.md) · [Livepatch](livepatch-support.md) · [Gap implementation design](design/kernel-gap-implementation-plan-2026-09-13.md)

Private records (maintainers): `review/nextplan/next-phase-plan-2026-09-13.md`, `review/design/next-handoff-2026-10-25.md`, `review/audits/qa-2026-10-27.md`, `review/design/st-k4-fsync-writeback-design.md`.
