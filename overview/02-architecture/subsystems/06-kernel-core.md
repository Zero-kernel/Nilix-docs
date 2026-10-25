# kernel_core 模块深度分析

> 分析日期：2026-05-13
> 源码版本：涵盖 R65-R155 修复、E.4/E.5/F.1/F.2/G.1/H.3 特性、KPTI 集成、三阶段 mmap、ExecSpaceGuard

## 1. 模块概述

`kernel_core` 是 Zero-OS 内核的核心子系统，承担进程管理、系统调用分发、内存隔离、信号处理、资源控制及安全执行等职责。该模块以 `#![no_std]` 运行于 x86_64 裸机环境，总代码量约 26,800 行 Rust，是整个内核中规模最大、依赖最广、安全敏感度最高的模块。

### 1.1 核心设计理念

- **Linux 兼容性**：系统调用编号、ABI 结构体、信号语义、命名空间模型均对标 Linux x86_64
- **安全纵深**：SMAP + W^X + KPTI + Seccomp + Capabilities + LSM + Audit 多层防御
- **SMP 可扩展**：`CpuLocal` per-CPU 数据 + RCU 无锁读 + 原子操作，减少全局锁争用
- **回调解耦**：通过 `scheduler_hook` 等钩子机制避免模块间循环依赖
- **失败可回滚**：ELF 加载、COW 页表复制、命名空间设备迁移等关键路径均具备回滚能力
- **Cgroup 一致性**：所有内存计费路径（mmap/brk/exec/fork）均跟踪 pending 字节，迁移时完整转移

### 1.2 外部依赖

| 依赖 crate | 用途 |
|---|---|
| `spin` | 自旋锁（Mutex/RwLock），适用于 `no_std` 环境 |
| `lazy_static` | 延迟初始化全局静态变量（`spin_no_std` feature） |
| `bitflags` | 位标志类型（MountFlags, CgroupControllers 等） |
| `x86_64` | x86_64 架构抽象（页表、寄存器、中断） |
| `xmas-elf` | ELF 文件解析 |
| `mm` | 内存管理（页表操作、帧分配器、物理地址转换） |
| `cpu_local` | Per-CPU 数据存储 |
| `drivers` | 设备驱动（VGA 缓冲区输出、kprintln! 宏） |
| `klog` | 统一内核日志宏（klog!, klog_always!） |
| `cap` | 能力（Capability）系统 |
| `seccomp` | Seccomp/Pledge 沙箱过滤 |
| `lsm` | Linux Security Module 钩子 |
| `audit` | 安全审计日志 |
| `security` | 安全 RNG、KPTI 状态、KASLR 滑动 |
| `net` | 网络协议栈 |
| `trace` | 可观测性（计数器、Watchdog） |
| `compliance` | 合规性配置（FIPS 模式等） |
| `livepatch` | 热补丁支持 |

---

## 2. 文件结构

```
kernel/kernel_core/
├── Cargo.toml              # 包配置与依赖声明（26行）
├── lib.rs                  # 模块入口与公共 API 再导出（278行）
├── process.rs              # 进程控制块(PCB)与进程管理（4423行）
├── fork.rs                 # Fork/COW 实现（1562行）
├── syscall.rs              # 系统调用分发与实现（11181行）
├── signal.rs               # POSIX 信号处理（466行）
├── elf_loader.rs           # ELF64 二进制加载器（709行）
├── cgroup.rs               # Cgroup v2 资源控制器（2297行）
├── rcu.rs                  # Read-Copy-Update 同步原语（875行）
├── scheduler_hook.rs       # 调度器回调钩子（176行）
├── usercopy.rs             # 用户空间内存拷贝（容错）（1468行）
├── exception_table.rs      # x86_64 异常表（132行）
├── time.rs                 # 时间管理与 TCP 定时器（214行）
├── mount_namespace.rs      # 挂载命名空间（322行）
├── ipc_namespace.rs        # IPC 命名空间（492行）
├── net_namespace.rs        # 网络命名空间（606行）
├── user_namespace.rs       # 用户命名空间（921行）
└── pid_namespace.rs        # PID 命名空间（653行）
```

**代码量统计**：总计约 26,800 行 Rust 代码（不含 Cargo.toml）。

---

## 3. 逐文件深度分析

### 3.1 lib.rs — 模块入口与公共 API（278行）

**功能概述**：作为 crate 根文件，声明所有子模块并通过 `pub use` 将关键类型和函数再导出为统一的公共 API。同时提供 LSM（Linux Security Module）上下文适配器函数。

**关键特性**：
- `#![no_std]` + `#![feature(abi_x86_interrupt)]` + `#![feature(negative_impls)]`
- 声明 17 个公共子模块（`pub mod process`, `pub mod fork`, ...）
- 大量 `pub use` 再导出，形成扁平化的公共接口（约 150+ 个符号）
- LSM 适配器函数（`lsm_get_pid()`, `lsm_get_credentials()`, `lsm_get_ticks()`）桥接 kernel_core 与 lsm crate
- `init()` 函数执行跨子系统初始化粘合

**`init()` 初始化序列**（第233-278行）：
1. `process::init()` — 必须最先，缓存 BOOT_CR3
2. `time::init()` — 记录启动 TSC
3. LSM context provider 注册
4. Seccomp current-process 评估回调注册（R29-2 修复）
5. Socket wait hooks 注册
6. Socket timeout checker 注册为定时器回调
7. Audit snapshot authorizer 注册（R26-4 修复；R133-1：使用 host-mapped root check）
8. OOM killer 回调注册
9. `cgroup::init()` — Cgroup v2 子系统初始化

**新增再导出**（相比早期版本）：
- `create_kpti_user_pml4`, `free_kpti_user_pml4`, `sync_kpti_cr3`（H.3 KPTI）
- `register_kpti_cr3_callback`, `KptiCr3UpdateCallback`（H.3 Per-CPU CR3 更新）
- `current_host_euid`, `current_host_egid`, `current_is_host_root`, `current_host_supplementary_groups`（R133-1/R135-1 host-mapped DAC）
- `force_init_resched_locals`（R151-5 Per-CPU 预初始化）
- `drain_deferred_stdin_wakes`（R149-1）
- `mmap_flags_to_perms`（R144-2 procfs mmap 权限解码）
- `FutexKey`（E.4 优先级继承）

**当前不足**：
- 再导出列表已超过 150 个符号，缺乏分组注释，维护成本高
- LSM 适配器函数直接写在 lib.rs 中，随着安全钩子增多应考虑独立模块

---

### 3.2 process.rs — 进程控制块与进程管理（4423行）

**功能概述**：定义进程控制块（PCB）`Process` 结构体及其全生命周期管理。是整个内核最核心的数据结构所在。相比早期版本（~2875行），已增长至 4423 行，新增了大量安全修复和特性。

**关键常量**：

| 常量 | 值 | 说明 |
|---|---|---|
| `KSTACK_BASE` | `0xFFFF_FFFF_0000_0000` | 内核栈虚拟地址基址 |
| `KSTACK_STRIDE` | `0x5000` (20KB) | 每进程内核栈步长（16KB 栈 + 4KB 守护页） |
| `KSTACK_PAGES` | 4 | 内核栈页数（16KB） |
| `MAX_PID` | ~209,715 | PID 上限（受内核栈地址空间约束） |
| `PID_MAX` | 32,768 | R106-11：用户可见 PID 上界（Linux 默认值），独立于 MAX_PID |
| `MAX_FD` | 256 | 每进程最大文件描述符数 |
| `DEFAULT_MMAP_BASE` | `0x4000_0000` | mmap 默认起始地址 |
| `WATCHDOG_TIMEOUT_MS` | 10,000 | 挂起任务检测超时（10秒） |

**核心数据结构**：

`Process` 结构体（PCB）包含以下关键字段（50+ 字段）：

```
Process {
    // === 进程标识 ===
    pid, generation,                // R106-1: 代际计数器防 PID 复用攻击
    tid, tgid, ppid,
    is_thread, name,

    // === 状态管理 ===
    state, stopped,                 // R98-1: 正交 stopped 标志
    pending_signals,
    thread_group_exiting,           // R153-3: exit_group 原子标志（Arc<AtomicBool>）
    pending_kill, pending_exit_code, // R115-1: 跨 CPU 退出请求

    // === 调度 ===
    priority, dynamic_priority,
    base_dynamic_priority,          // E.4 PI: 未应用 PI 的基线优先级
    pi_boosts, waiting_on_futex,    // E.4: 优先级继承支持
    time_slice, cpu_time,
    wait_ticks,                     // R65-19: 饥饿计数器
    allowed_cpus, cpuset_id,

    // === CPU 上下文 ===
    context: Context,               // 16 通用寄存器 + RIP/RFLAGS + CS/SS + FxSaveArea
    fpu_used,                       // 惰性 FPU 跟踪

    // === 内存 ===
    kernel_stack, kernel_stack_top,
    user_stack, memory_space,
    user_memory_space,              // H.3 KPTI: 用户 PML4 物理地址
    mmap_regions, next_mmap_addr,
    vm_charged_bytes,               // R131-6: Per-task cgroup 内存计费
    brk_pending_growth,             // R144-1: brk 扩展 pending 计费
    mprotect_pending_bytes,         // R147-1: mprotect pending 计费
    exec_pending_bytes,             // R149-3: exec pending 计费
    elf_charged_bytes,              // R137-1: ELF 加载器 cgroup 计费

    // === 文件系统 ===
    fd_table, cloexec_fds,          // R39-4: close-on-exec 标记
    cap_table,                      // 能力表（Arc，fork 时 CLOFORK 过滤）
    credentials,                    // Arc<RwLock<Credentials>>
    umask,
    brk_start, brk,
    fs_base, gs_base,               // TLS 状态

    // === 安全沙箱 ===
    seccomp_state, pledge_state,
    seccomp_installing,             // R26-3: 安装期间拒绝新线程

    // === 命名空间 ===
    pid_ns_chain,                   // F.1: PID 命名空间成员链
    pid_ns_for_children,
    mount_ns, mount_ns_for_children,
    ipc_ns, ipc_ns_for_children,
    net_ns, net_ns_for_children,
    user_ns, user_ns_for_children,

    // === 资源控制 ===
    cgroup_id,                      // F.2: Cgroup v2 成员

    // === 可观测性 ===
    watchdog_handle,                // G.1: 看门狗句柄

    // === OOM ===
    nice, oom_score_adj,

    // === 线程支持 ===
    clear_child_tid, set_child_tid,
    robust_list_head, robust_list_len,

    // === 子进程与等待 ===
    exit_code, waiting_child, children,
    created_at,
}
```

**`Context` 结构体**（第345-411行）：
- 16 个通用寄存器 + RIP + RFLAGS + CS/SS + 4个 padding u64 + `FxSaveArea`
- `#[repr(C, align(64))]` 保证 64 字节对齐
- 默认 RFLAGS = `0x202`（IF 位使能），CS = 0x08（内核代码段），SS = 0x10（内核数据段）

**`ProcessState` 枚举**（第295-311行）：`Ready | Running | Blocked | Stopped | Sleeping | Zombie | Terminated`

**`Process::new()`**（第807行）：
- R101-1 修复：默认凭证为 nobody（uid=65534），非 root
- 内核进程（ppid=0）由 `create_process()` 显式提升为 root
- E.4 PI：`base_dynamic_priority` 和 `pi_boosts` 初始化
- R98-1：`stopped` 初始化为 false
- `next_mmap_addr` 使用 `security::randomized_mmap_base()` 随机化
- H.2 KASLR：`kernel_stack_slot()` 应用 `security::kernel_stack_slide()` 随机偏移

**`kernel_stack_slot()`**（第203行）：
- R103-I2 修复：全部使用 checked 算术，溢出返回 `Err(KernelStackError::AddressOverflow)` 而非 panic
- H.2 KASLR：基址加 `kernel_stack_slide()` 随机偏移

**`allocate_kernel_stack()`**（第241行）：
- R128-1 修复：内核栈页标志不使用 GLOBAL（防止 PID 回收后 stale TLB）
- 分配连续物理帧，映射栈页，守护页不映射

**核心函数**：

| 函数 | 说明 |
|---|---|
| `create_process()` | 创建新进程，分配 PID、内核栈、初始化 PCB、注册看门狗 |
| `terminate_process()` | 终止进程，释放资源，转为 Zombie |
| `terminate_self_and_halt()` | R117-1：自身终止后禁中断+切换 boot CR3+HLT，不返回 |
| `request_process_exit()` | R115-1：设置跨 CPU pending_kill 标志 |
| `drain_deferred_irq_terminates()` | R155-6：在进程上下文排空 IRQ 延迟终止 |
| `free_process_resources()` | 释放进程持有的所有资源（fd、内存、cgroup 退费等） |
| `activate_memory_space()` | 切换 CR3；H.3 KPTI：同时更新 per-CPU 双 CR3 |
| `sync_kpti_cr3()` | H.3：PCB user_memory_space 更新后重新同步 per-CPU KPTI 状态 |
| `get_process()` / `current_pid()` | 通过 PID 查找进程 / 获取当前 PID |
| `current_host_euid()` / `current_is_host_root()` | R133-1：host-mapped euid/root 检查 |
| `compute_cgroup_charged_bytes()` | 汇总所有 pending 计费字段，用于迁移时完整转移 |

**`FileOps` trait**（第122-140行）：
```rust
pub trait FileOps: Send + Sync {
    fn clone_box(&self) -> Box<dyn FileOps>;
    fn as_any(&self) -> &dyn Any;
    fn type_name(&self) -> &'static str;
    fn stat(&self) -> Result<VfsStat, SyscallError>; // R41-1: fstat 支持
}
```

**凭证模型**：`credentials: Arc<RwLock<Credentials>>` — 线程组内共享凭证（CLONE_THREAD 场景），fork 时创建独立副本。`Credentials` 包含 uid/gid/euid/egid/supplementary_groups。

**当前不足**：
- 文件体量巨大（4423 行），`Process` 结构体字段 50+，应考虑按职责拆分
- `PROCESS_TABLE` 使用全局 `Mutex<Vec<Option<Arc<Mutex<Process>>>>>`，双层锁嵌套存在死锁风险
- `MAX_FD = 256` 偏低，生产环境通常需要 1024+
- 缺少进程组（process group）和会话（session）抽象

---

### 3.3 fork.rs — Fork 与写时复制（COW）（1562行）

**功能概述**：实现 `sys_fork()` 系统调用及完整的写时复制（Copy-On-Write）页表复制机制。相比早期版本（~1202行），增长至 1562 行，新增了大量安全修复和 cgroup 计费逻辑。

**核心数据结构**：

`ForkResult` 枚举：`Parent(ProcessId) | Child | Error(ForkError)`

`ForkError` 枚举：`NoCurrentProcess | ProcessNotFound | MemoryAllocationFailed | PageTableCopyFailed | ProcessCreationFailed | CgroupPidsLimitExceeded | MmapTransientState`
- **R122-1 新增**：`MmapTransientState` — fork 期间 mmap_regions 含 PENDING_MAP/PENDING_UNMAP 条目时返回 EAGAIN

**核心函数**：

| 函数 | 说明 |
|---|---|
| `sys_fork()` | Fork 入口；F.2 Cgroup pids.max 检查；R152-5：先 attach_task 再 fork_inner 消除 TOCTOU；R138-1：COW 全量 cgroup 计费 |
| `fork_inner()` | 内部实现：R122-1 三阶段 mmap 瞬态检查；页表 COW 复制；H.3 KPTI 用户 PML4 创建；上下文/fd/凭证/命名空间继承 |
| `copy_page_table_cow()` | 两阶段 COW：规划 → 预分配 → 应用（Z-8 修复） |
| `handle_cow_page_fault()` | COW 页错误处理；通过 `try_lock(PT_LOCK)` 返回 `Handled/NotCow/Busy/Fatal` 精确结果（R186-10） |
| `create_fresh_address_space()` | 创建新用户地址空间，深拷贝恒等映射并拆分大页 |
| `create_kpti_user_pml4()` | H.3 KPTI：为指定内核 PML4 创建用户 PML4 shadow |
| `free_kpti_user_pml4()` | H.3 KPTI：释放用户 PML4 根帧（不递归共享子表） |
| `cleanup_partial_child()` | 清理失败 fork 的部分子进程；H.0.9：PID 命名空间链 detach；G.1：看门狗注销 |

**关键设计**：

1. **两阶段 COW（Z-8 修复）**：
   - 阶段 1（规划）：遍历页表收集叶子修改计划，不修改任何 PTE
   - 阶段 2（预分配）：预分配所有中间页表帧，失败时父进程未被修改
   - 阶段 3（应用）：使用预分配帧应用 COW 修改，保证不会失败

2. **R152-5 Cgroup 时序修复**：`sys_fork()` 在 `fork_inner()` 之前执行 `attach_task()`，消除并发 fork 全部通过 `check_fork_allowed` 但串行 attach 失败的 TOCTOU 窗口

3. **R138-1 COW 全量 cgroup 计费**：fork 后对子进程 cgroup 收取父进程完整虚拟内存足迹（mmap_regions + brk + elf_charged_bytes），保守但 fail-closed

4. **R122-1 三阶段 mmap 瞬态检查**：`fork_inner()` 入口检查 `mmap_regions` 中是否包含 PENDING_MAP/PENDING_UNMAP 瞬态标志，有则返回 `MmapTransientState`（EAGAIN）

5. **H.3 KPTI 集成**：`fork_inner()` 为子进程创建独立的用户 PML4（`create_kpti_user_pml4()`），失败时回滚内核 PML4

6. **COW 标志位**：使用 x86_64 页表 BIT_9（软件可用位）标记 COW 页

7. **跨 CPU 序列化（R67-6 修复）**：COW 设置期间持有全局页表锁（PT_LOCK）

8. **TLB Shootdown（R68-4 修复）**：COW 解析后使用跨 CPU TLB 刷新

**`fork_inner()` 继承清单**：
- 页表（COW 复制）、CPU 上下文（RAX 置 0）、FPU 状态
- CPU 亲和性、cpuset（含 task_joined 回调）
- 内核栈（独立分配，内容复制，RSP/RBP 重定位）
- 文件描述符表（clone_box 逐 fd 克隆）、cloexec_fds
- 能力表（clone_for_fork 过滤 CLOFORK）
- 凭证（独立 Arc 克隆）、umask
- 堆管理（brk_start, brk）、elf_charged_bytes
- TLS 状态（fs_base, gs_base）
- 命名空间（IPC/Net/User 及 for_children）
- Seccomp/Pledge 沙箱状态
- Cgroup 成员关系
- mmap_regions（剥离瞬态标志）

**当前不足**：
- `PT_LOCK` 仍是全局锁，SMP 下 COW 页错误在同一页表修改域内串行化；#PF 竞争通过 `Busy` 退出并重试，禁止关中断阻塞
- `PhysicalPageRefCount` 使用 `BTreeMap` 存储引用计数
- 不支持大页（Huge Page）的 COW

---

### 3.4 syscall.rs — 系统调用分发与实现（11181行）

**功能概述**：实现 Linux x86_64 兼容的系统调用分发器，包含 100+ 个系统调用的完整实现。是整个内核中代码量最大的单文件，从早期的 9181 行增长至 11181 行。

**关键常量与 ABI 结构体**：

- Seccomp 相关：`SECCOMP_SET_MODE_STRICT/FILTER`、`PR_GET/SET_SECCOMP`、`PR_SET/GET_NO_NEW_PRIVS`
- Linux ABI 结构体：`OpenHow`（openat2）、`TimeVal`/`TimeSpec`、`UtsName`、`LinuxDirent64`
- mmap 区域标志：`MMAP_REGION_FLAG_PROT_NONE`、`MMAP_REGION_FLAG_TRANSIENT_MASK`（R122-1 三阶段 mmap）

**核心函数**：

| 函数 | 说明 |
|---|---|
| `syscall_dispatcher()` | 系统调用分发入口，按编号路由 |
| `sys_read/write/open/close` | 基础文件 I/O |
| `sys_fork/clone/exec/exit/wait` | 进程生命周期 |
| `sys_mmap/munmap/brk/mprotect` | 内存管理；R121-4 三阶段 mmap；R144-1 brk pending 计费 |
| `sys_socket/bind/listen/accept/connect` | 网络套接字 |
| `sys_kill` | 信号发送 |
| `sys_seccomp/prctl` | 安全沙箱 |
| `sys_clone_ns/setns/unshare` | 命名空间操作 |
| `sys_cgroup_*` | Cgroup 管理 |
| `sys_exec` | R149-3：exec_pending_bytes 追踪；ExecSpaceGuard RAII 回滚 |
| `sys_exit_group` | R153-3：thread_group_exiting 原子标志防竞态 |

**三阶段 mmap/munmap 协议（R121-4）**：
1. Phase 1（Reserve）：在 `mmap_regions` 中插入带 `PENDING_MAP`/`PENDING_UNMAP` 标志的条目
2. Phase 2（PT Operation）：释放进程锁，执行页表映射/解映射
3. Phase 3（Commit）：重新获取锁，清除 PENDING 标志（或移除条目）

**安全集成**：
- 每个系统调用入口处执行 Seccomp 过滤检查
- 关键操作前调用 LSM 钩子
- 所有系统调用通过 Audit 子系统记录入口/出口事件
- Per-CPU 计数器跟踪系统调用频率（G.1）

**回调注册机制**：通过 `register_*_callback()` 函数注册 VFS、管道、futex 等操作的回调，避免 kernel_core 直接依赖 VFS/IPC crate。新增：`register_vfs_open_with_resolve_callback`、`register_vfs_truncate_callback`。

**新增导出**：`mmap_flags_to_perms()`（R144-2）用于 procfs `/proc/<pid>/maps` 的 mmap 区域权限解码。

**当前不足**：
- 单文件 11181 行，严重影响可维护性
- 回调注册模式导致调用链隐式化
- 缺少 32-bit 兼容层（compat syscall）

---

### 3.5 signal.rs — POSIX 信号处理（466行）

**功能概述**：实现 POSIX 兼容的信号子系统，支持信号 1-64。相比早期版本（~419行），增长至 466 行，新增了内核权威信号路径和跨 CPU 安全终止。

**核心数据结构**：

`Signal(u8)`：信号标识符（1-64），定义标准 POSIX 信号常量（SIGHUP 到 SIGTTOU）。新增辅助方法：`is_stop()`、`is_continue()`、`is_uncatchable()`。

`PendingSignals { bits: u64 }`：挂起信号位图。

`SignalAction` 枚举：`Ignore | Terminate | Stop | Continue`

`SignalError` 枚举：`InvalidSignal | NoSuchProcess | PermissionDenied`。实现 `From<SignalError> for SyscallError`。

**核心函数**：

| 函数 | 说明 |
|---|---|
| `send_signal()` | 用户态信号发送，强制 POSIX 权限检查 |
| `send_signal_kernel()` | R115-2：内核权威信号（bypass 权限检查），用于 PID namespace cascade/OOM/seccomp |
| `send_signal_inner()` | 共享内部实现，`enforce_permissions` 参数控制权限检查 |
| `default_action()` | 获取信号默认动作 |
| `signal_name()` | 信号名称字符串（调试用） |
| `register_resume_callback()` | 注册调度器恢复回调 |

**安全设计**：
- **R65-26 修复**：获取目标进程 Arc 后一次性持有，消除 TOCTOU 窗口
- **R134-7 修复**：`send_signal_inner` 使用 `current_is_host_root()` 进行 host-mapped root 检查
- PID 1 (init) 特殊保护：仅自身可发信号

**R98-1 作业控制修复**：
- SIGSTOP：设置正交 `stopped` 标志，不覆盖 Blocked/Sleeping 状态
- SIGCONT：通过调度器 `resume_stopped()` 回调处理，不直接清除 `stopped` 标志

**H.0.7 跨 CPU 安全终止**：
- 自身终止（same CPU）：调用 `terminate_self_and_halt()`（R117-1），不返回
- 远程终止（other CPU）：通过 `request_process_exit()` 设置 pending_kill 标志，目标在 syscall 返回路径自行终止

**当前不足**：
- 不支持用户自定义信号处理器（sigaction）
- 缺少信号掩码（sigprocmask）
- 信号队列使用位图，不支持实时信号排队语义
- 缺少 SIGCHLD 的 waitpid 集成

---

### 3.6 elf_loader.rs — ELF64 二进制加载器（709行）

**功能概述**：解析并加载 ELF64 可执行文件到用户地址空间。相比早期版本（~599行），增长至 709 行，新增了多项安全加固。

**关键常量**：

| 常量 | 值 | 说明 |
|---|---|---|
| `USER_BASE` | `0x0040_0000` (4MB) | 用户程序加载基址 |
| `USER_STACK_TOP` | `0x7FFF_FFFF_E000` | 用户栈顶（减去 8KB 守护页） |
| `USER_STACK_SIZE` | `0x20_0000` (2MB) | 用户栈大小 |
| `MAX_ELF_SEGMENT_PAGES` | 256 | R151-2：每段最大页数（1MB），防 DoS |
| `MAX_ELF_LOAD_SEGMENTS` | 32 | R154-5：最大 PT_LOAD 段数，防 DoS |

**`ElfLoadError` 枚举**（新增成员）：
- `TooManySegments`（R154-5）：PT_LOAD 段数超过 32
- `OverlappingSegments`（R154-12）：段虚拟地址范围重叠
- `InvalidSegment`（R155-16）：段 vaddr+memsz 算术溢出

**`ElfLoadResult`**（新增字段）：
- `charged_bytes: u64`（R125-1）：cgroup 已计费字节数，用于 exec 失败时回滚

**`load_elf()` 签名变更**：
```rust
pub fn load_elf(image: &[u8], cgroup_id: cgroup::CgroupId) -> Result<ElfLoadResult, ElfLoadError>
```
- R149-3 修复：接收调用方在进程锁下捕获的 `cgroup_id`，消除 TOCTOU

**安全加固**：
- **W^X 强制**：拒绝同时可写可执行的段
- **R24-9**：入口点验证（用户空间范围 + canonical 地址）
- **R151-2**：每段最大 256 页（1MB）
- **R154-5**：最大 32 个 PT_LOAD 段
- **R154-12**：段重叠检测
- **R155-16**：使用 `checked_add` 替代 `saturating_add` 防止溢出
- **R154-14**：`all_mappings` 使用 fallible 分配，避免 heap 耗尽 panic
- **R93-6**：Cgroup 内存计费
- **Z-10**：失败时统一回滚所有映射

**当前不足**：
- 不支持动态链接（无 PT_INTERP/PT_DYNAMIC 处理）
- 不支持 PIE（Position Independent Executable）
- 用户栈大小固定 2MB
- 缺少 ASLR（地址空间布局随机化，但 `next_mmap_addr` 已有随机化）

---

### 3.7 cgroup.rs — Cgroup v2 资源控制器（2297行）

**功能概述**：实现 Linux Cgroup v2 风格的层级化资源控制。相比早期版本（~1714行），增长至 2297 行，新增了委托模式和强化的计费一致性。

**关键常量**：

| 常量 | 值 | 说明 |
|---|---|---|
| `MAX_CGROUP_DEPTH` | 8 | 最大层级深度 |
| `MAX_CGROUPS` | 4096 | 最大 cgroup 数量 |
| `DEFAULT_WEIGHT` | 100 | CPU 默认权重 |
| `IO_BURST_SECS` | 4 | IO 令牌桶突发窗口 |

**核心数据结构**：
- `CgroupControllers` 位标志：`CPU | MEMORY | PIDS | IO`
- `CgroupLimits`：资源限额配置（cpu_weight, cpu_max, memory_max/high, pids_max, io_max_bytes/iops）
- `CgroupStats`：无锁统计（全部 `AtomicU64/AtomicU32`）
- `CpuQuotaState`：CPU 配额跟踪
- `CgroupNode`：层级节点

**核心函数**：

| 函数 | 说明 |
|---|---|
| `init()` | 初始化根 cgroup |
| `create_cgroup()` | 创建子 cgroup（含深度/数量限制检查） |
| `delete_cgroup()` | 删除 cgroup（需无任务且无子节点） |
| `delegate_cgroup()` | 委托 cgroup 管理权限给非 root 用户 |
| `migrate_task()` | 任务迁移到目标 cgroup（含计费转移） |
| `check_fork_allowed()` | Fork 前检查 pids.max |
| `try_charge_memory()` | 原子 CAS 内存计费 |
| `uncharge_memory()` | 内存释放退费 |
| `charge_cpu_quota()` | CPU 配额计费 |
| `charge_io()` / `wait_for_io_window()` | IO 令牌桶计费与节流 |

**当前不足**：
- 内存控制器缺少 OOM killer 实现
- CPU 配额 `refresh_window()` 非原子
- IO 控制器不区分设备

---

### 3.8 rcu.rs — Read-Copy-Update 同步原语（875行）

**功能概述**：实现基于 epoch 的 RCU 同步机制。

**核心函数**：

| 函数 | 说明 |
|---|---|
| `rcu_read_lock()` | 进入读侧临界区 |
| `rcu_read_unlock()` | 退出读侧临界区 |
| `synchronize_rcu()` | 阻塞等待所有预存读者完成 |
| `call_rcu()` | 注册延迟回调（R71-1 CAS 修复） |
| `poll()` | 排空已完成 grace period 的回调（MAX_CALLBACKS_PER_POLL=16） |
| `rcu_timer_tick()` | 定时器驱动的 grace period 推进 |

**当前不足**：
- `synchronize_rcu()` 使用忙等待
- 回调队列使用全局 Mutex
- 未实现 SRCU

---

### 3.9 scheduler_hook.rs — 调度器回调钩子（176行）

**功能概述**：提供调度器与其他模块之间的解耦接口。

**核心数据结构**：
- `TIMER_CBS: Mutex<Vec<TimerCallback>>` — R148-I6 修复：注册时 `without_interrupts` 防死锁
- `RESCHED_CB: Mutex<Option<ReschedCallback>>` — R152-11 修复：注册时 `without_interrupts`
- `IRQ_RESCHED_PENDING: CpuLocal<AtomicBool>` — R67-4：Per-CPU 中断延迟抢占标志

**新增函数**：
- `force_init_resched_locals()`（R151-5）：在 IRQ 启用前预初始化 Per-CPU 标志，防止首次 IRQ 在 `Once::call_once()` 堆分配中死锁

**`reschedule_if_needed()`**（第131行）— syscall 返回路径，新增排空点：
1. `drain_deferred_tcp_timers()`（R65-6）
2. `drain_deferred_stdin_wakes()`（R149-1）
3. `drain_deferred_irq_terminates()`（R155-6）
4. `rcu::poll()`（E.4 RCU）
5. 消费 IRQ_RESCHED_PENDING 并调用 resched 回调

**`on_scheduler_tick()`**（第88行）：
- 使用固定大小栈数组（`MAX_TIMER_CALLBACKS = 4`）复制回调，避免 IRQ 上下文堆分配
- R148-I6：使用 blocking `lock()` 而非 `try_lock()`，因注册侧已包裹在 `without_interrupts` 中
- R72：调用 `rcu_timer_tick()` 推进 grace period

---

### 3.10 usercopy.rs — 用户空间内存拷贝（1468行）

**功能概述**：提供内核与用户空间之间的容错内存拷贝操作。相比早期版本（~1327行），增长至 1468 行。

**核心数据结构**：

`UserCopyState`：Per-CPU 用户拷贝状态（V-5 修复：Per-CPU）
- `active`, `pid`（H-36：PID 绑定）, `remaining`, `start`, `end`

`UserAccessGuard`：SMAP RAII 守卫
- S-5 修复：嵌套支持（depth 计数器）
- V-5 修复：Per-CPU 存储
- R25-10 修复：最外层禁中断

`UserPtr<T>`：类型安全的用户空间单对象指针

`UserSlice<T>`：类型安全的用户空间切片指针

**内联汇编（H-26 修复）**：
- `__zero_os_usercopy_get_u8` / `__zero_os_usercopy_put_u8`：异常表辅助函数
- 使用 PC-relative `.ex_table` 条目，兼容 PIE/KASLR

**当前不足**：
- 逐字节拷贝（`get_u8`/`put_u8`），性能远低于批量拷贝
- `MAX_CSTRING_LEN` 硬编码限制

---

### 3.11 exception_table.rs — x86_64 异常表（132行）

**功能概述**：实现 Linux 风格的异常表，将故障指令地址映射到修复地址。

**核心数据结构**：
```rust
#[repr(C)]
pub struct ExceptionTableEntry {
    pub fault_ip: usize,
    pub fixup_ip: usize,
}
```

**当前不足**：线性搜索 O(n)，应改为排序表 + 二分查找。

---

### 3.12 time.rs — 时间管理与 TCP 定时器（214行）

**功能概述**：提供基于时钟中断的时间戳支持，集成双频率 TCP 定时器扫描和延迟定时器工作排空机制。

**定时器频率**：
- `TCP_TIMER_SWEEP_INTERVAL_MS = 200` — 快速定时器
- `TIME_WAIT_SWEEP_INTERVAL_MS = 1000` — 慢速定时器

**R65-6 修复**：IRQ 上下文锁争用时设置延迟标志，在 syscall 返回路径排空。

---

### 3.13 mount_namespace.rs — 挂载命名空间（322行）

`MAX_MNT_NS_LEVEL = 32`。引用计数使用 `Release/Acquire` 内存序。

---

### 3.14 ipc_namespace.rs — IPC 命名空间（492行）

`MAX_IPC_NS_LEVEL = 32`，`MAX_IPC_NAMESPACES = 1024`（R76-2）。`NsCountGuard` RAII 模式。

---

### 3.15 net_namespace.rs — 网络命名空间（606行）

设备迁移（`move_device()`）支持权限检查、原子性和回滚。

---

### 3.16 user_namespace.rs — 用户命名空间（921行）

UID/GID 映射（单次写入语义）、`MAX_MAPPINGS = 5`、父命名空间包含性验证。

---

### 3.17 pid_namespace.rs — PID 命名空间（653行）

PID 链分配（`assign_pid_chain()` 含回滚）、级联终止（`get_cascade_kill_pids()`）、双向映射。`MAX_PID_NS_LEVEL = 32`。

---

## 4. 模块整体评估

### 4.1 架构优势

1. **安全纵深完备**：SMAP + W^X + KPTI + Seccomp/Pledge + Capability + LSM + Audit，从硬件到策略的完整安全栈
2. **Linux 兼容的隔离栈**：PID/Mount/IPC/Net/User 五种命名空间 + Cgroup v2 四控制器
3. **SMP 可扩展性设计**：`CpuLocal` per-CPU 数据 + RCU 无锁读 + 原子操作 + per-CPU IRQ 标志
4. **关键路径回滚能力**：ELF 段映射、COW 页表复制（两阶段）、三阶段 mmap、exec ExecSpaceGuard、命名空间设备迁移、PID 链分配
5. **回调解耦架构**：scheduler_hook、信号恢复回调、IPC 清理回调、cpuset task_joined/left 回调
6. **Cgroup 计费一致性**：所有内存操作路径（mmap/brk/exec/fork）跟踪 pending 字节，迁移时完整转移
7. **跨 CPU 安全终止**：R115-1 pending_kill + R117-1 terminate_self_and_halt + H.0.7 remote signal 延迟终止

### 4.2 安全评估

**已实现的安全机制**：
- 用户空间指针验证（`UserPtr<T>`/`UserSlice<T>` 类型安全 API）
- SMAP 守卫（嵌套支持 + per-CPU + 中断禁用）
- 异常表容错恢复（EFAULT 语义，PC-relative for KASLR）
- POSIX 信号权限检查（R134-7 host-mapped root）
- Capability 表 fork 时 CLOFORK 过滤
- Seccomp BPF 过滤 + Pledge 沙箱
- Cgroup pids.max fork 限制（R152-5 TOCTOU 修复）
- 内核栈守护页（R128-1 非 GLOBAL）
- W^X ELF 加载策略
- 用户命名空间映射的父包含性验证
- H.2 KASLR 内核栈随机偏移
- H.3 KPTI 双页表（kernel/user CR3）
- R101-1 默认 nobody 凭证
- R106-1 进程代际计数器
- R151-2/R154-5/R154-12/R155-16 ELF 加载器 DoS 防护

**安全风险点**：
- `syscall.rs` 11181 行单文件
- 回调/反转控制的隐式调用链
- `unsafe` 代码集中在 usercopy 汇编、页表操作、COW 引用计数
- 跨子系统资源一致性的生命周期交织

### 4.3 性能评估

**性能优势**：
- RCU 无锁读侧路径
- 惰性 FPU 保存/恢复
- COW 延迟物理页复制
- Cgroup 统计使用无锁原子操作
- 定时器回调使用栈数组避免 IRQ 堆分配
- R118-6 热路径直接传递 user_memory_space 避免 PROCESS_TABLE 扫描

**性能瓶颈**：
- usercopy 逐字节拷贝
- 异常表线性搜索 O(n)
- `PT_LOCK` 全局页表修改锁
- `PROCESS_TABLE` 双层 Mutex 嵌套
- RCU 回调队列全局 Mutex

### 4.4 并发模型与锁顺序

**已知锁顺序约定**：
1. `PROCESS_TABLE` → `Process`（进程表锁先于进程锁）
2. `PidNamespace` 内部锁 → `PidNamespaceMembership`
3. 页错误入口仅可 `try_lock(PT_LOCK)`；IF=0 时禁止阻塞/自旋（R186-10）
4. `TIMER_CBS`/`RESCHED_CB` — 注册时 `without_interrupts` 防 IRQ 死锁

**潜在死锁风险**：
- 命名空间操作涉及多个子系统锁（进程表、命名空间内部锁、cgroup 锁），缺少全局锁顺序文档

### 4.5 改进路线图

**P0（关键）**：
- 拆分 `syscall.rs`（11181 行）为按功能域的子模块
- 实现批量 usercopy（`rep movsb` 或 `get_u64`）
- 异常表改为排序表 + 二分查找

**P1（重要）**：
- `Process` 结构体按职责拆分
- 将全局 `PT_LOCK` 逐步收窄为 per-address-space/page-table 粒度
- 实现 cgroup 级别的 OOM 回收策略
- PID 回收机制

**P2（增强）**：
- ASLR 支持（ELF 加载基址随机化）
- 动态链接器（PT_INTERP）
- 进程组和会话抽象
- SRCU（Sleepable RCU）
- 信号处理器（sigaction）完整实现
- `/proc` 文件系统命名空间感知

**P3（优化）**：
- `PhysicalPageRefCount` 改用哈希表或 per-page 内联计数
- RCU 回调队列改为 per-CPU 无锁队列
- TSC 频率校准
- TCP 定时器逻辑迁移到网络子系统
