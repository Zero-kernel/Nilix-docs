# Zero-OS 项目全景分析文档

> **分析日期**: 2026-05-13  
> **分析版本**: next-phase-plan v12.8 (post-R155 audit)  
> **分析方法**: 8 个并行子 agent 按功能域深度扫描全部 128 个 Rust 源文件  
> **目标**: 全面理解项目现状，评估与生产级内核 (Linux) 的距离

---

## 1. 项目概述

### 1.1 定位

Zero-OS 是一个用 **Rust** 编写的 **x86_64 UEFI 内核**，采用 **安全优先混合内核** 架构，设计原则为：

> **Security > Correctness > Efficiency > Performance**

目标不是完全复刻 Linux，而是构建一个能在 **UEFI + QEMU/KVM** 环境中运行的安全内核，具备容器级隔离能力。

### 1.2 量化概览

| 指标 | 数值 |
|------|------|
| 内核代码 | 128 个 .rs 文件, ~115,000 行 |
| 引导加载器 | 1 个文件, ~1,006 行 |
| 用户空间 | 6 个文件, ~2,611 行 |
| **总代码量** | **~118,617 行** |
| 子系统数 | 24 个独立 crate |
| 安全审计轮次 | **155 轮** |
| 发现问题总数 | **881 个** |
| 已修复 | **844 个 (95.8%)** |
| Open CRITICAL | **0** |
| Open HIGH | **0** |
| 系统调用 | ~85 个已实现 |
| 文件系统 | 6 种 (ramfs, ext2, procfs, devfs, cgroupfs, initramfs) |
| 网络协议 | IPv4 + TCP + UDP + ICMP + ARP |
| 容器支持 | 5 种 Namespace + Cgroups v2 (4 控制器) |

### 1.3 架构总览

```
┌─────────────────────────────────────────────────────────────┐
│                       USER SPACE (Ring 3)                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │  Shell   │  │  Test    │  │  musl    │  │  clone   │    │
│  │ (交互式)  │  │ (Ring3)  │  │  (ELF)   │  │  (线程)   │    │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘    │
│       └──────────────┴──────────────┴──────────────┘         │
│                    SYSCALL / SYSRET (MSR)                     │
├──────────────────────────────────────────────────────────────┤
│                      KERNEL SPACE (Ring 0)                    │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ 安全框架: LSM (60+ hooks) + Capabilities + Seccomp/Pledge│ │
│  │         + Audit (SHA-256 Hash Chain) + Compliance       │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │ Scheduler│ │   MM     │ │   IPC    │ │  VFS     │       │
│  │ MLFQ     │ │ Buddy+PT │ │ Futex+PI │ │ ext2+   │       │
│  │ Per-CPU  │ │ COW+KPTI │ │ Pipe     │ │ procfs  │       │
│  │ 工作窃取  │ │ PageCache│ │ Endpoint │ │ cgroupfs│       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
│                                                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │  Net     │ │  Block   │ │ Namespace│ │ Cgroups  │       │
│  │ TCP/UDP  │ │ VirtIO   │ │ PID/Mount│ │ v2       │       │
│  │ Firewall │ │ blk/net  │ │ IPC/Net  │ │ CPU/Mem  │       │
│  │ Conntrack│ │ IOMMU    │ │ User     │ │ PID/IO   │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
│                                                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │ Arch     │ │ Security │ │ Drivers  │ │ Infra    │       │
│  │ GDT/IDT  │ │ KASLR    │ │ VGA/FB   │ │ Klog     │       │
│  │ APIC/SMP │ │ W^X/NX   │ │ Keyboard │ │ Trace    │       │
│  │ SYSCALL  │ │ CSPRNG   │ │ Serial   │ │ Livepatch│       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
├──────────────────────────────────────────────────────────────┤
│                      HARDWARE (x86_64)                       │
│  CPU (LAPIC/IOAPIC) │ Memory │ VirtIO │ PCI │ IOMMU/VT-d   │
└──────────────────────────────────────────────────────────────┘
```

### 1.4 开发阶段进度

| 阶段 | 名称 | 状态 | 完成度 |
|------|------|------|--------|
| Phase A | 安全基础 | ~90% | KASLR 部分, KPTI 已启用 |
| Phase B | 能力/MAC 框架 | ✅ 完成 | LSM+Seccomp+Cap 全部集成 |
| Phase C | 存储基础 | ✅ 完成 | VirtIO-blk + ext2 + Page Cache |
| Phase D | 网络基础 | ✅ 完成 | 完整 TCP/IP 栈 + 有状态防火墙 |
| Phase E | SMP & 并发 | ✅ 完成 | 多核调度 + TLB Shootdown + RCU |
| Phase F | 资源治理 | ✅ 完成 | 5 种 Namespace + Cgroups v2 + IOMMU |
| Phase G | 生产就绪 | 安全完成 | 可观测性 + 热补丁 + 合规 |
| Phase H | 高级特性 | 进行中 | KASLR 全文本 + KPTI + ABI 审计 |
| Phase I-M | 未来 | 计划中 | 高级网络/存储/虚拟化 |

---

## 2. 子系统目录

| # | 子系统 | 路径 | 详细文档 |
|---|--------|------|----------|
| 1 | 引导加载器 | `bootloader/` | [01_bootloader.md](01_bootloader.md) |
| 2 | 体系结构 | `kernel/arch/` | [02_arch.md](02_arch.md) |
| 3 | 内存管理 | `kernel/mm/` | [03_mm.md](03_mm.md) |
| 4 | 调度器 | `kernel/sched/` | [04_sched.md](04_sched.md) |
| 5 | 进程间通信 | `kernel/ipc/` | [05_ipc.md](05_ipc.md) |
| 6 | 内核核心 | `kernel/kernel_core/` | [06_kernel_core.md](06_kernel_core.md) |
| 7 | 虚拟文件系统 | `kernel/vfs/` | [07_vfs.md](07_vfs.md) |
| 8 | 网络栈 | `kernel/net/` | [08_net.md](08_net.md) |
| 9 | 安全子系统 | `kernel/security/` | [09_security.md](09_security.md) |
| 10 | 驱动/块/VirtIO/IOMMU | `kernel/drivers/`, `kernel/block/`, etc. | [10_drivers_block_virtio_iommu.md](10_drivers_block_virtio_iommu.md) |
| 11 | LSM/Cap/Seccomp | `kernel/lsm/`, `kernel/cap/`, `kernel/seccomp/` | [11_lsm_cap_seccomp.md](11_lsm_cap_seccomp.md) |
| 12 | 热补丁/追踪/审计/合规 | `kernel/livepatch/`, `kernel/trace/`, etc. | [12_livepatch_trace_audit_compliance.md](12_livepatch_trace_audit_compliance.md) |
| 13 | CPU Local / TLB Ops | `kernel/cpu_local/`, `kernel/tlb_ops/` | [13_cpu_local_tlb_ops.md](13_cpu_local_tlb_ops.md) |
| 14 | 用户空间 | `userspace/` | [14_userspace.md](14_userspace.md) |
| 15 | 内核入口 | `kernel/src/main.rs` | [15_kernel_entry.md](15_kernel_entry.md) |
| **16** | **与 Linux 差距分析** | — | **[16_gap_analysis_vs_linux.md](16_gap_analysis_vs_linux.md)** |

---

## 3. 与 Linux 内核差距总览

### 3.1 定量对比

| 指标 | Linux 6.x | Zero-OS | 差距倍数 |
|------|-----------|---------|----------|
| 总代码量 | ~30,000,000 行 | ~118,617 行 | **253x** |
| 系统调用 | 450+ | ~85 | 5.3x |
| 文件系统 | 30+ | 6 | 5x |
| 驱动代码 | ~10,000,000 行 | ~3,000 行 | 3333x |
| CPU 架构 | 20+ | 1 (x86_64) | 20x |
| 安全模块 | LSM + SELinux + AppArmor + SMACK | LSM + Seccomp + Cap | 策略缺失 |
| 容器支持 | Namespace + Cgroups v2 | ✅ 同等 | **接近持平** |
| 网络协议 | 完整 TCP/IP + IPv6 + Netfilter | IPv4 + TCP/UDP + 基础防火墙 | 3x |

### 3.2 功能差距矩阵

| 类别 | 完成度 | 关键差距 |
|------|--------|----------|
| **引导/初始化** | ██████████ 95% | ACPI 解析有限, 无 Secure Boot 链 |
| **内存管理** | ████████░░ 80% | 无 Slab/NUMA/Swap/内存压缩 |
| **进程/线程** | ████████░░ 85% | 无 ptrace, 线程组不完整 |
| **调度器** | ████████░░ 80% | 无 CFS vruntime, 无实时调度类 |
| **IPC** | ██████░░░░ 60% | 无 SysV IPC/Unix Socket/epoll |
| **VFS** | ███████░░░ 70% | ext2 写限于直接块, 无 journal |
| **网络** | ███████░░░ 70% | 无 IPv6/路由表/NAT/隧道 |
| **安全** | ████████░░ 80% | 无 SELinux 策略/Keyring/IMA |
| **容器** | █████████░ 90% | ✅ 5 种 NS + Cgroups v2 完整 |
| **驱动** | ███░░░░░░░ 30% | 无统一驱动框架/设备树/ACPI |
| **可观测性** | ███████░░░ 70% | 有 Trace/Profiler/Watchdog, 无 perf |
| **虚拟化** | █████░░░░░ 50% | IOMMU/VT-d 完整, 无 KVM |

### 3.3 对于 UEFI 运行目标的评估

由于项目的目标不是完全复刻 Linux，而是在 **UEFI 环境 (QEMU/KVM)** 中能运行，以下是针对该目标的评估：

| 能力 | 状态 | 说明 |
|------|------|------|
| UEFI 启动 | ✅ 完成 | BOOTX64.EFI → ELF 加载 → 高半区映射 → 内核入口 |
| 基本 I/O | ✅ 完成 | VGA + 帧缓冲 + 键盘 + 串口 |
| 内存管理 | ✅ 完成 | Buddy + 4 级页表 + COW + PageCache |
| 多核 SMP | ✅ 完成 | LAPIC/IOAPIC + AP 启动 + Per-CPU + TLB Shootdown |
| 用户态程序 | ✅ 完成 | ELF 加载 + Ring 3 + Shell + 系统调用 |
| 磁盘 I/O | ✅ 完成 | VirtIO-blk + ext2 读写 |
| 网络通信 | ✅ 完成 | VirtIO-net + TCP/UDP + ARP + 防火墙 |
| 进程隔离 | ✅ 完成 | Namespace + Cgroups + SMAP/SMEP + W^X |
| 安全审计 | ✅ 完成 | SHA-256 哈希链 + LSM + Seccomp |
| 热补丁 | ✅ 完成 | INT3 重定向 + ECDSA 签名验证 |

**结论**: 对于 UEFI 运行目标，Zero-OS **已经具备了一个功能完整的操作系统内核的所有核心能力**。它可以：
- 从 UEFI 启动
- 管理多核 CPU
- 运行用户态程序
- 进行磁盘和网络 I/O
- 提供容器级别的进程隔离
- 进行安全审计和热补丁

**距离生产级的主要差距**集中在：驱动生态、高级文件系统、IPv6、性能优化等领域。

---

*详细子系统分析请参阅各章节文档。完整差距分析参阅 [16_gap_analysis_vs_linux.md](16_gap_analysis_vs_linux.md)。*
