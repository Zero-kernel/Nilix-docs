# 安全框架 (LSM/CAP/Seccomp) 模块分析文档

> 分析日期：2026-05-13

## 1. 模块概述

### 1.1 安全目标与威胁模型

Zero-OS 的安全框架由三个互补的子系统构成，共同实现纵深防御：

| 子系统 | 核心职责 | 防御层次 |
|--------|----------|----------|
| **LSM** (Linux Security Module) | 可插拔安全策略 hook 框架，拦截内核操作并做访问控制裁决 | 强制访问控制 (MAC) |
| **CAP** (Capability) | 基于能力的对象访问控制，管理不可伪造的句柄与权限 | 对象级权限隔离 |
| **Seccomp/Pledge** | 系统调用过滤与承诺机制，限制进程可用的 syscall 集合 | 攻击面收缩 |

另有 **Crypto** 模块提供共享密码学原语（SHA-256），供审计和 livepatch 子系统使用。

**信任边界**：用户态进程不可信，内核态代码可信但需最小权限。三个子系统在 syscall 入口处形成串联裁决链：

```text
用户态 syscall 请求
        |
        v
+-------------------+
| Seccomp/Pledge    |  <- 第一道：syscall 级白名单过滤
| (fail-closed)     |     Kill/Trap -> 直接终止，不进入后续流程
+--------+----------+
         | Allow
         v
+-------------------+
|   LSM Hooks       |  <- 第二道：策略级访问控制
| (policy-based)    |     Denied -> 返回 EPERM
+--------+----------+
         | Ok(())
         v
+-------------------+
|  CAP 权限检查     |  <- 第三道：对象级能力验证
| (per-object)      |     InsufficientRights -> 返回 EACCES
+--------+----------+
         | Ok
         v
    执行实际操作
```

### 1.2 共同设计模式：回调解耦

三个子系统都面临与 `kernel_core` 的循环依赖问题，统一采用**回调注册模式**解耦：

- **LSM**：`register_context_provider()` 注册获取当前进程 PID/凭证/时钟的回调
- **Seccomp**：`register_current_hooks()` 注册当前进程的 seccomp 评估器回调
- **CAP**：`types.rs` 中本地定义 `FileOps` trait / `ProcessId` 类型，避免直接依赖 `kernel_core`

这种模式确保各安全模块可独立编译，在内核初始化阶段通过回调桥接运行时依赖。

### 1.3 Fork/Exec 状态继承矩阵

| 状态 | fork() 行为 | exec() 行为 |
|------|-------------|-------------|
| Seccomp filter stack | 子进程继承（Arc 共享） | 保持不变 |
| Pledge promises | 子进程继承 | `exec_promises` 生效（若有） |
| `no_new_privs` | 子进程继承 | 不可清除 |
| CapTable 条目 | 复制（跳过 CLOFORK 标记的） | 撤销 CLOEXEC 标记的 |
| CapTable generation | 子进程继承父进程计数器 | 不变 |
| LSM 策略 | 全局共享，不随进程变化 | 触发 `task_exec` hook |

---

## 2. 文件结构

### 2.1 行数统计表

| 模块 | 文件 | 行数 | 主要职责 |
|------|------|------|----------|
| LSM | `kernel/lsm/lib.rs` | 1777 | Hook 分发器（60+ hook）、策略注册表、上下文类型、审计集成 |
| LSM | `kernel/lsm/policy.rs` | 796 | `LsmPolicy` trait 定义（51 方法）、PermissivePolicy、DenyAllPolicy |
| CAP | `kernel/cap/lib.rs` | 592 | `CapTable` 实现（allocate/lookup/revoke/delegate/fork/exec） |
| CAP | `kernel/cap/types.rs` | 577 | `CapId`/`CapRights`/`CapFlags`/`CapObject`/`CapEntry`/错误类型 |
| Seccomp | `kernel/seccomp/lib.rs` | 553 | 入口评估、strict/pledge 过滤器生成、审计通知 |
| Seccomp | `kernel/seccomp/types.rs` | 1110 | BPF 指令集、`SeccompFilter`/`SeccompState`、`PledgePromises`、验证逻辑、Pledge 命名空间拒绝 |
| Crypto | `kernel/crypto/lib.rs` | 213 | SHA-256（FIPS 180-4）实现，供审计/livepatch 共享 |
| **合计** | **7 个文件** | **5618** | |

### 2.2 目录结构

```
kernel/
+-- lsm/
|   +-- lib.rs          # Hook 分发 + 策略注册 + 上下文类型
|   +-- policy.rs       # LsmPolicy trait + 内置策略
+-- cap/
|   +-- lib.rs          # CapTable 核心实现
|   +-- types.rs        # 能力系统类型定义
+-- seccomp/
|   +-- lib.rs          # Seccomp 入口 + 过滤器生成
|   +-- types.rs        # BPF VM + Pledge + 状态管理
+-- crypto/
    +-- lib.rs          # SHA-256 共享实现
```

---

## 3. 逐文件分析

### 3.1 LSM 模块

#### 3.1.1 `kernel/lsm/lib.rs`（1777 行）

**功能定位**：LSM 子系统的核心文件，包含策略注册表、hook 分发器、上下文类型定义和审计集成。

**关键数据结构**：

```rust
// 策略槽包装器 -- 解决 trait fat pointer 的原子性问题
#[repr(C)]
struct PolicySlot {
    policy: &'static dyn LsmPolicy,
}

// 全局策略指针（AtomicPtr 实现 IRQ-safe 无锁读取）
static ACTIVE_POLICY_SLOT: AtomicPtr<PolicySlot>;

// R103-4 FIX: 用 Mutex<Option<&'static PolicySlot>> 替代 static mut
// Box::leak 产生稳定堆地址，消除 Rust 2024 edition 的 static mut UB
static USER_POLICY_SLOT: Mutex<Option<&'static PolicySlot>>;
```

**策略注册机制（`set_policy` / `policy`）**：

- `set_policy()` 通过 `Box::leak` 在堆上分配 `PolicySlot`，获得 `&'static` 引用后存入 `AtomicPtr`（`Ordering::Release`）
- `policy()` 通过 `AtomicPtr::load(Ordering::Acquire)` 无锁读取当前策略
- 为何需要 `PolicySlot` 包装层：`dyn LsmPolicy` 是 fat pointer（data + vtable 两个指针），无法用单个 `AtomicPtr` 原子存储。将其包装在 `PolicySlot` 结构体中，只需原子操作一个 thin pointer，避免 torn read
- 旧策略被有意泄漏（`set_policy` 仅在启动时调用一两次，泄漏可接受）

**上下文类型体系**（7 种 hook 上下文）：

| 上下文类型 | 用途 | 关键字段 |
|-----------|------|----------|
| `SyscallCtx` | syscall 入口/出口 | pid, uid/gid, euid/egid, syscall_nr, args[6], cap |
| `ProcessCtx` | 进程生命周期操作 | pid, tgid, uid/gid, euid/egid, cap |
| `FileCtx` | VFS 文件操作 | inode, mode, path_hash, cap |
| `IpcCtx` | IPC 消息传递 | endpoint_id, bytes, cap |
| `SignalCtx` | 信号/ptrace | target_pid, sig, cap |
| `NetCtx` | 网络 socket 操作 | socket_id, proto, local/remote addr:port, cap |
| `NetControlCtx` | 控制面流量（R62-7 FIX） | op(NetControlOp), src_ip, dst_ip, proto_data |

**NetControlOp** 枚举（R62-7 FIX）：`ArpLearn` / `ArpReply` / `IcmpEchoReply` / `IcmpError`，用于拦截绕过 socket 层的控制面流量。

**Hook 分发器设计**：

文件包含 **60+ 个 hook 函数**，覆盖以下类别：

- **Syscall**（2）：`hook_syscall_enter` / `hook_syscall_exit`
- **进程生命周期**（8）：`hook_task_fork` / `hook_task_exec` / `hook_task_exit` / `hook_task_setuid` / `hook_task_setresuid` / `hook_task_setgroups` / `hook_task_prctl` / `hook_task_cap_modify`
- **VFS 文件**（16）：`hook_file_lookup` / `hook_file_open` / `hook_file_create` / `hook_file_mmap` / `hook_file_chmod` / `hook_file_chown` / `hook_file_unlink` / `hook_file_rename` / `hook_file_link` / `hook_file_symlink` / `hook_file_mkdir` / `hook_file_rmdir` / `hook_file_truncate` / `hook_file_permission` / `hook_file_mount` / `hook_file_umount`
- **内存**（4，R29-3/R30-3 FIX）：`hook_memory_mmap` / `hook_memory_mprotect` / `hook_memory_brk` / `hook_memory_munmap`
- **IPC**（5）：`hook_ipc_send` / `hook_ipc_recv` / `hook_ipc_pipe` / `hook_ipc_futex` / `hook_ipc_shm`
- **信号**（2）：`hook_signal_send` / `hook_ptrace`
- **网络**（10）：`hook_net_socket` / `hook_net_bind` / `hook_net_connect` / `hook_net_listen` / `hook_net_accept` / `hook_net_setsockopt` / `hook_net_send` / `hook_net_recv` / `hook_net_shutdown` / `hook_net_control`
- **Livepatch**（4，R102-13/R104-6）：`hook_kpatch_load` / `hook_kpatch_enable` / `hook_kpatch_disable` / `hook_kpatch_unload`

每个 hook 的统一模式（注意：`hook_syscall_exit` 和 `hook_task_exit` 是例外，它们仅调用策略但不触发 denial audit）：

```rust
#[inline]
pub fn hook_xxx(/* 参数 */) -> LsmResult {
    #[cfg(feature = "lsm")]
    {
        let res = policy().xxx(/* 参数 */);
        if let Err(ref err) = res {
            emit_denial_audit(subject, object, "hook_name", err);
        }
        res
    }
    #[cfg(not(feature = "lsm"))]
    {
        let _ = (/* 参数 */);
        Ok(())  // 零开销：编译期消除
    }
}
```

**Feature Gate 机制**：`#[cfg(feature = "lsm")]` 控制编译行为：
- 启用时：hook 调用活跃策略 + 审计
- 禁用时：所有 hook 内联为 `Ok(())`，零运行时开销

**能力操作常量模块**（R65-13 FIX）：

```rust
pub mod cap_op {
    pub const ALLOCATE: u32 = 1;
    pub const REVOKE: u32 = 2;
    pub const DELEGATE: u32 = 3;
}
```

确保 LSM hook 与审计事件的操作代码一致。

**审计集成**：
- 大多数 hook 在策略拒绝时（`Err`）触发审计事件（`hook_syscall_exit` 和 `hook_task_exit` 除外）
- 通过 `emit_lsm_denial()` 发送到 audit 子系统
- 审计信息包含：Subject（pid/uid/gid/cap）、Object（inode/mode/path_hash 或 Memory/Process）、hook 名称、拒绝原因
- `hook_net_control` 使用内核上下文（pid=0），按 `NetControlOp` 生成审计 hook 名称

**初始化**：`lsm::init()` 将 `ACTIVE_POLICY_SLOT` 设置为指向 `PERMISSIVE_SLOT`，确保 `policy()` 在初始化后不再走 null 分支。

#### 3.1.2 `kernel/lsm/policy.rs`（796 行）

**功能定位**：定义 `LsmPolicy` trait 接口及两个内置策略实现。

**`LsmPolicy` trait**：

```rust
pub trait LsmPolicy: Send + Sync {
    fn name(&self) -> &'static str;
    fn priority(&self) -> u32;
    // 共 51 个 hook 方法（含默认实现）
    fn syscall_enter(&self, ctx: &SyscallCtx) -> LsmResult { Ok(()) }
    fn task_fork(&self, parent: &ProcessCtx, child: &ProcessCtx) -> LsmResult { Ok(()) }
    // ...
}
```

**设计要点**：
- `Send + Sync` 约束：策略必须线程安全，可从任意 CPU / 中断上下文调用
- 默认实现全部返回 `Ok(())`，**除了 livepatch hook**
- R104-6 FIX：`kpatch_load` / `kpatch_enable` / `kpatch_disable` / `kpatch_unload` 的默认实现返回 `Err(LsmError::Denied)`（default-deny），因为 livepatch 会向内核注入可执行代码

**Hook 方法分类**（与 `lib.rs` 的 hook 分发器一一对应）：

| 类别 | 方法数量 | 代表方法 |
|------|---------|----------|
| Syscall | 2 | `syscall_enter`, `syscall_exit` |
| 进程生命周期 | 7 | `task_fork`, `task_exec`, `task_exit`, `task_setuid`, `task_setresuid`, `task_setgroups`, `task_prctl` |
| 能力操作 | 1 | `task_cap_modify` |
| VFS 文件 | 16 | `file_lookup` ~ `file_umount`（含 mount/umount） |
| 内存 | 4 | `memory_mmap`, `memory_mprotect`, `memory_brk`, `memory_munmap` |
| IPC | 5 | `ipc_send`, `ipc_recv`, `ipc_pipe`, `ipc_futex`, `ipc_shm` |
| 信号 | 2 | `signal_send`, `ptrace` |
| 网络 | 10 | `net_socket` ~ `net_shutdown`, `net_control`（含控制面） |
| Livepatch | 4 | `kpatch_load`, `kpatch_enable`, `kpatch_disable`, `kpatch_unload` |
| **合计** | **51** | |

**内置策略实现**：

1. **`PermissivePolicy`**：所有 hook 返回 `Ok(())`，优先级 `u32::MAX`（最低）。R104-6 FIX：显式覆盖 kpatch_* 为 `Ok(())`，维持"允许一切"的语义合约。

2. **`DenyAllPolicy`**：所有 hook 返回 `Err(LsmError::Denied)`，优先级 `0`（最高）。完整覆盖了 trait 的全部 51 个方法，确保无遗漏。

**错误类型**：

```rust
pub enum LsmError {
    Denied,    // 策略拒绝操作 -> 映射为 EPERM
    Internal,  // 策略内部错误（如配置错误）
}
```

---

### 3.2 CAP 模块

#### 3.2.1 `kernel/cap/types.rs`（577 行）

**功能定位**：定义能力系统的全部核心类型，是 CAP 模块的类型基础。

**`CapId` -- 能力标识符**：

```text
63              16 15              0
+------------------+----------------+
|    Generation    |     Index      |
|     (48 bits)    |   (16 bits)    |
+------------------+----------------+
```

- **Index**（低 16 位）：CapTable 中的槽位索引，最大 65535 个槽位
- **Generation**（高 48 位）：每次分配时从全局计数器递增获取，槽位重用时新分配的 generation 必然更高，使旧 CapId 永久失效，防止 use-after-free
- `CapId::INVALID`：值为 0（generation=0 保留为无效标记）
- R29-4 FIX：从原来的 32-bit generation + 32-bit index 扩展为 48+16 编码，提供约 281 万亿次分配的空间

**`CapRights` -- 权限位掩码**（`bitflags!` 宏，u64）：

| 类别 | 权限位 | 说明 |
|------|--------|------|
| 通用 (0-4) | READ, WRITE, EXEC, IOCTL, ADMIN | 基本读写执行与管理 |
| 内存 (5-6) | MAP, MAP_EXEC | mmap 映射权限 |
| 网络 (7-10,14) | BIND, CONNECT, LISTEN, ACCEPT, NET_BIND_SERVICE | socket 操作权限 |
| 进程 (11-13) | SIGNAL, WAIT, PTRACE | 进程控制权限 |
| 特权 (30-31) | BYPASS_DAC, BYPASS_MAC | 绕过 DAC/MAC（仅 root） |
| 审计 (40-41) | AUDIT_READ, AUDIT_WRITE | 审计日志读写权限（R66-10 FIX） |
| 可观测 (42) | TRACE_READ | 追踪计数器/watchdog 读取 |
| 组合 | RW, RWX, ALL_SOCKET, ALL_PROCESS | 便捷组合常量 |

核心方法：
- `allows(required)` -- 检查是否包含所需权限
- `restrict(mask)` -- AND 操作实现单调递减的权限委派

**`CapFlags` -- 行为标志**（`bitflags!` 宏，u32）：

| 标志 | 含义 |
|------|------|
| `CLOEXEC` | exec() 时撤销此能力 |
| `CLOFORK` | fork() 时不继承此能力 |
| `O_PATH` | 仅用于路径操作，不可读写 |
| `NOXFER` | 禁止委派给其他进程 |

**`CapObject` -- 能力引用的内核对象**：

```rust
pub enum CapObject {
    File(Arc<dyn FileOps>),    // VFS/fd 对象
    Endpoint(EndpointId),       // IPC 端点
    Socket(Arc<Socket>),        // 网络 socket
    Shm(Arc<Shm>),             // 共享内存
    Timer(Arc<Timer>),          // 定时器
    Process(ProcessId),         // 进程句柄
    Namespace(NamespaceId),     // 命名空间句柄
}
```

**`CapEntry`**：将 `CapObject` + `CapRights` + `CapFlags` 组合为一个表项，提供 `inherits_on_exec()` / `inherits_on_fork()` / `allows()` 便捷方法。

**`Socket` 结构体**（R75-1 增强）：包含 `socket_id`（全局 socket 标识）和 `net_ns_id`（网络命名空间 ID），支持容器化网络隔离。

**`NamespaceId`**：命名空间标识符（`#[repr(transparent)]` 包装 `u64`），用于 mount/ipc/net/user/pid 命名空间。

**`CapError` 错误枚举**：`InvalidCapId` / `InsufficientRights` / `TableFull` / `NoCurrentProcess` / `DelegationDenied` / `InvalidOperation` / `GenerationExhausted`。

#### 3.2.2 `kernel/cap/lib.rs`（592 行）

**功能定位**：`CapTable` 的核心实现，提供能力的分配、查找、撤销、委派、fork 克隆和 exec 清理。

**架构设计**：

```rust
pub struct CapTable {
    inner: Mutex<CapTableInner>,  // spinlock 保护，IRQ-safe
}

struct CapTableInner {
    slots: Vec<Option<CapSlot>>,  // 槽位数组
    free: Vec<u16>,               // 空闲槽位索引栈（R29-4 FIX：u16）
    next_generation: u64,         // 下一个 generation（从 1 开始，0 保留）
}

struct CapSlot {
    generation: u64,   // 此槽位的 generation（R29-4 FIX：u64）
    entry: CapEntry,
}
```

**配置常量**：
- `DEFAULT_CAP_SLOTS = 64`：默认初始容量
- `MAX_CAP_SLOTS = 65535`：最大槽位数（匹配 u16 索引范围，R30-1 FIX）

**核心操作**：

**`allocate(entry) -> Result<CapId, CapError>`**：
1. 从 `free` 栈弹出空闲索引；若无空闲则尝试扩展 `slots`（不超过 `MAX_CAP_SLOTS`）
2. 分配当前 `next_generation` 并递增（`checked_add` 防溢出，R32-CAP-1 FIX）
3. 检查 48-bit generation 上限（`0x0000_FFFF_FFFF_FFFF`），耗尽则返回 `GenerationExhausted`
4. 存入 `CapSlot`，返回 `CapId::from_parts(index, generation)`
5. 所有操作在 `interrupts::without_interrupts` 中执行，确保 IRQ-safe

**`lookup(cap_id) -> Result<CapEntry, CapError>`**：
1. 验证 `cap_id.is_valid()`（generation > 0）
2. 检查 index 范围
3. 匹配 slot 的 generation 与 cap_id 的 generation，不匹配则返回 `InvalidCapId`

**`revoke(cap_id) -> Result<CapEntry, CapError>`**：
1. 验证 + generation 匹配（同 lookup）
2. `take()` 取出 slot 内容，将 index 推回 `free` 栈
3. 返回被撤销的 `CapEntry`
4. 下次该 slot 被重用时会获得更高的 generation，使旧 CapId 永久失效

**`delegate(cap_id, rights_mask, flags) -> Result<CapId, CapError>`**：
1. 查找源能力，检查 `NOXFER` 标志
2. **R25-1 FIX**：强制继承源能力的 flags（`flags | source_entry.flags`），防止通过 flag stripping 提权攻击
3. 权限单调递减：`source_rights.restrict(rights_mask)` = AND 操作
4. 分配新槽位存储委派后的能力

**`clone_for_fork() -> CapTable`**：
- 遍历所有 slot，跳过带 `CLOFORK` 标志的条目
- 子表继承父表的 `next_generation` 计数器，防止子进程 generation 过早回绕
- 重建空闲列表

**`apply_cloexec()`**：
- 遍历所有 slot，撤销带 `CLOEXEC` 标志的条目
- 被撤销的 slot 推回 `free` 栈，重用时获得新 generation

**`has_rights(required) -> bool`**：
- 扫描全表，检查是否有任何能力授予所需权限
- 用于环境权限检查（如 `AUDIT_READ`、`TRACE_READ`）

**`check_rights(cap_id, required) -> Result<bool, CapError>`**：
- 按 CapId 查找并验证特定能力的权限

**安全设计要点**：
- CapTable 是低层结构，**刻意不直接调用 LSM hook 或发出审计事件**（R66-9）
- LSM 和审计集成由 syscall 层包装完成（`sys_cap_allocate` 等）
- 内核内部路径（fork/exec）允许绕过 LSM hook（策略在父进程获取能力时已检查）

---

### 3.3 Seccomp 模块

#### 3.3.1 `kernel/seccomp/types.rs`（1110 行）

**功能定位**：定义 Seccomp 和 Pledge 的全部核心类型，包括 BPF 指令集、过滤器、状态管理和验证逻辑。

**集中化 syscall 号定义**（R149-I3 FIX）：

types.rs 顶部定义了所有 Linux x86_64 syscall 号常量（约 40 个），作为 pledge/seccomp 过滤的唯一来源：

```rust
pub(crate) const SYS_READ: u64 = 0;
pub(crate) const SYS_WRITE: u64 = 1;
pub(crate) const SYS_OPEN: u64 = 2;
pub(crate) const SYS_OPENAT: u64 = 257;
// ... 等等
```

`pledge_to_filter()`（lib.rs）和 `promise_allows_syscall()`（types.rs）两条路径都使用这些常量，消除了原来因硬编码数字导致的覆盖偏差。

**`SeccompAction` -- 过滤裁决动作**：

```rust
pub enum SeccompAction {
    Allow,       // severity 0 - 放行
    Log,         // severity 1 - 记录但放行
    Errno(i32),  // severity 2 - 返回错误码
    Trap,        // severity 3 - 触发 SIGSYS
    Kill,        // severity 4 - 终止进程
}
```

严重度排序：`Kill > Trap > Errno > Log > Allow`。多 filter 叠加时，最严格的 action 胜出。

**`SeccompInsn` -- BPF-like 指令集**：

| 指令 | 功能 |
|------|------|
| `LdSyscallNr` | 加载 syscall 号到累加器 |
| `LdArg(u8)` | 加载 syscall 参数（0-5）到累加器 |
| `LdConst(u64)` | 加载常量 |
| `And(u64)` / `Or(u64)` / `Shr(u8)` | 位运算 |
| `JmpEq/Ne/Lt/Le/Gt/Ge(val, t, f)` | 条件跳转（true_offset, false_offset） |
| `Jmp(u8)` | 无条件跳转 |
| `Ret(SeccompAction)` | 返回裁决 |

`MAX_INSNS = 64`：程序最大长度限制，防止 DoS。

**`SeccompFilter` -- 编译后的过滤器**：

```rust
pub struct SeccompFilter {
    pub default_action: SeccompAction,
    pub prog: Arc<[SeccompInsn]>,       // BPF 程序（Arc 共享，fork 友好）
    pub fast_allow: FastAllowSet,       // 512-bit 快速放行位图
    pub id: u64,                        // FNV-1a 哈希 ID
    pub flags: SeccompFlags,            // LOG / TSYNC / NEW_THREADS
}
```

**`evaluate()` 方法 -- BPF 解释器**：
1. **快速路径**：若 `syscall_nr < 512` 且 `fast_allow` 位图对应位为 1，直接返回 `Allow`
2. **慢速路径**：逐条解释 BPF 程序，维护累加器 `acc` 和程序计数器 `pc`
3. **R32-SECCOMP-2 FIX**：所有跳转指令在 pc 更新后检查边界，越界则 fail-closed 返回 `Trap`
4. 程序执行完毕未遇到 `Ret` 则使用 `default_action`

**`FastAllowSet` -- 快速放行位图**：
- 512 bits = 8 个 u64，覆盖 syscall 号 0-511
- **R152-I1 FIX**：`compute_fast_allow()` 现在遍历 JmpEq 链的 false-branch，而非仅检查 prog[1]，正确设置所有 pledge/strict 过滤器允许的 syscall 位

**`validate_program()` -- 程序验证**（R32-SECCOMP-1 FIX）：
- 检查程序非空且不超过 `MAX_INSNS`
- 验证参数索引 < 6
- **跳转目标验证**：使用 `>=` 而非 `>` 比较 `prog.len()`，防止跳转到程序末尾之后绕过策略
- 确保程序包含至少一条 `Ret` 指令

**`compute_filter_id()` -- 过滤器哈希**（R100-6 FIX）：
- 使用 FNV-1a 算法对每条指令的结构化字段进行哈希
- 避免直接对 enum 做字节切片（可能包含未初始化的 padding 字节导致 UB）

**`PledgePromises` -- OpenBSD 风格承诺**（`bitflags!` 宏，u32，18 个标志）：

| 承诺 | 允许的操作 |
|------|-----------|
| `STDIO` | read, write, close, fstat, lseek, getpid, getuid/gid/euid/egid, getppid, sched_yield |
| `RPATH` | open(RD), openat(RD), stat, lstat, readlink, getdents64 |
| `WPATH` | open(WR), openat(WR), rename, unlink, symlink |
| `CPATH` | open(O_CREAT), openat(O_CREAT), mkdir, rmdir, link |
| `TMPPATH` | open/openat 临时文件操作 |
| `PROC` | fork, vfork, clone(无 CLONE_VM/namespace), wait4, waitid, kill |
| `THREAD` | clone(CLONE_THREAD\|CLONE_VM\|CLONE_SIGHAND), futex, set_tid_address |
| `EXEC` | execve |
| `UNIX` / `INET` / `DNS` | 网络相关 |
| `FATTR` | chmod, chown, fchmod, fchown |
| `RLIMIT` | getrlimit, setrlimit |
| `TIME` | clock_gettime, getrandom |
| `SENDSIG` | kill（发送信号） |
| `PTRACE` | 调试/追踪 |
| `PROT_EXEC` | mmap/mprotect 带 PROT_EXEC |
| `VM` | mmap, mprotect, munmap, brk, mremap |

`PledgePromises::parse()` 支持空格分隔的字符串解析（如 `"stdio rpath wpath"`），无效承诺返回 `SeccompError::InvalidPromise`。

**`promise_allows_syscall()` -- Pledge 语义评估器**：

该函数实现 flag 感知的 syscall 裁决，与 `pledge_to_filter()` 生成的 BPF 过滤器互补（R150-3 FIX 同步两条路径的覆盖范围）：

- **open/openat flag 检查**：读取需要 RPATH（或任意 path 承诺），写入需要 WPATH/CPATH/TMPPATH，创建需要 CPATH/TMPPATH
- **mmap/mprotect PROT_EXEC 门控**：需要 VM + PROT_EXEC 承诺
- **clone() 命名空间拒绝**（R147-2 FIX）：PROC 承诺下的 clone() 拒绝所有 CLONE_NEW* 命名空间标志（CLONE_NEWNS/NEWUTS/NEWIPC/NEWUSER/NEWPID/NEWNET），防止通过命名空间创建提权
- **clone() CLONE_VM 拒绝**（R151-2 FIX）：PROC 承诺下的 clone(CLONE_VM) 仅在 THREAD 承诺也存在且包含 CLONE_THREAD|CLONE_VM|CLONE_SIGHAND 时允许
- **THREAD clone 限制**（R147-2 FIX）：THREAD 承诺仅允许线程式 clone（要求 CLONE_THREAD|CLONE_VM|CLONE_SIGHAND 全部设置）
- **THREAD 命名空间拒绝**（R154-11 FIX）：THREAD 承诺下也显式拒绝 CLONE_NEW* 标志
- `exit`/`exit_group` 始终允许（R145-7 FIX）
- 未匹配任何承诺则返回 `false`（fail-closed）

**`SeccompState` -- 进程级 Seccomp 状态**：

```rust
pub struct SeccompState {
    pub filters: Vec<Arc<SeccompFilter>>,
    pub no_new_privs: bool,
    pub log_violations: bool,
}
```

`evaluate()` 遍历所有 filter，取最严格的 action；遇到 `Kill` 立即短路返回。

#### 3.3.2 `kernel/seccomp/lib.rs`（553 行）

**功能定位**：Seccomp 子系统的入口模块，提供当前进程评估、过滤器生成和审计通知。

**回调注册与 Fail-Closed 设计**：

```rust
static CURRENT_EVALUATOR: RwLock<Option<SeccompEvaluator>>;
static CURRENT_ENABLED_CHECK: RwLock<Option<SeccompEnabledCheck>>;
static SECCOMP_INITIALIZED: AtomicBool;  // R65-14 FIX
```

`evaluate_current()` 的三阶段行为（R65-14 FIX）：

| 阶段 | 条件 | 行为 |
|------|------|------|
| 启动前 | `SECCOMP_INITIALIZED = false` | 返回 `Allow`（内核自身需要 syscall） |
| 正常运行 | evaluator 已注册 | 委托给注册的评估器 |
| 异常状态 | 已初始化但 evaluator 为 None | 返回 `Kill`（fail-closed，防止绕过） |

**Strict Mode 过滤器**（R144-3 FIX）：

```rust
const STRICT_ALLOWED: &[u64] = &[
    0,   // read
    1,   // write
    15,  // rt_sigreturn -- R144-3 FIX
    60,  // exit
    231, // exit_group
];
```

R144-3 FIX：添加 `rt_sigreturn(15)` 到 strict 白名单。没有它，strict 模式下的进程在收到信号后无法从信号处理器返回，因为 sigreturn syscall 被 seccomp 拒绝，导致强制终止而非优雅信号处理。

**Pledge -> Filter 编译器**（`pledge_to_filter()`）：

将 `PledgePromises` 位集编译为 BPF 过滤器。R150-3 FIX 确保与 `promise_allows_syscall()` 路径的 syscall 覆盖范围同步：

| 承诺 | 允许的 syscall |
|------|---------------|
| STDIO | read(0), write(1), close(3), fstat(5), lseek(8), getpid(39), getuid(102), getgid(104), geteuid(107), getegid(108), getppid(110), sched_yield(24) |
| RPATH | open(2), openat(257), stat(4), lstat(6), readlink(89), getdents64(217) |
| WPATH | open(2), openat(257), rename(82), unlink(87), symlink(88) |
| CPATH | open(2), openat(257), mkdir(83), rmdir(84), link(86) |
| TMPPATH | open(2), openat(257) |
| VM | mmap(9), mprotect(10), munmap(11), brk(12), mremap(25) |
| PROC | clone(56), fork(57), vfork(58), wait4(61), kill(62), waitid(247) |
| EXEC | execve(59) |
| THREAD | clone(56), futex(202), set_tid_address(218) |
| TIME | clock_gettime(228), getrandom(318) |
| SENDSIG | kill(62) |
| FATTR | chmod(90), chown(92), fchmod(93), fchown(94) |
| RLIMIT | getrlimit(97), setrlimit(160) |

exit(60) 和 exit_group(231) 始终包含。

**审计通知**（`notify_violation()`）：

当 seccomp 过滤器返回非 Allow 裁决时，通过 `emit_seccomp_violation()` 发送审计事件，包含：
- 进程身份（pid/uid/gid）
- 被阻止的 syscall 号
- 过滤器 ID
- 动作类型
- errno 映射（Kill->9, Trap->31）

---

### 3.4 Crypto 模块

#### 3.4.1 `kernel/crypto/lib.rs`（213 行）

**功能定位**：R148-I9 FIX 新增模块。整合原先分散在 audit 和 livepatch 子系统中的 SHA-256 实现为单一规范的 FIPS 180-4 实现，消除维护漂移风险。

**核心 API**：

```rust
pub mod sha256 {
    pub const BLOCK_SIZE: usize = 64;   // 512 bits
    pub const DIGEST_SIZE: usize = 32;  // 256 bits

    pub struct Sha256 {
        state: [u32; 8],
        buffer: [u8; 64],
        buffer_len: usize,
        total_len: u64,
    }

    impl Sha256 {
        pub const fn new() -> Self;           // 流式初始化
        pub fn digest(data: &[u8]) -> [u8; 32]; // 一次性摘要
        pub fn update(&mut self, data: &[u8]);   // 流式输入
        pub fn finalize(self) -> [u8; 32];       // 完成计算
    }
}
```

实现标准 FIPS 180-4 SHA-256：64 轮 Sigma/Ch/Maj 压缩、Merkle-Damgard 填充、大端序输出。

---

## 4. 模块整体评估

### 4.1 架构优势

**纵深防御**：三层安全机制串联，任一层的绕过不会导致完全失防：
- Seccomp 收缩攻击面（syscall 级）
- LSM 实施策略控制（操作级）
- CAP 验证对象权限（资源级）

**零开销抽象**：LSM 的 `#[cfg(feature = "lsm")]` feature gate 在禁用时将所有 hook 编译为空操作，不产生运行时开销。Seccomp 的 `FastAllowSet` 位图为常见 syscall 提供 O(1) 快速路径。

**Fail-Closed 设计**：
- Seccomp：初始化后 evaluator 丢失则 Kill（R65-14）
- Seccomp BPF：跳转越界返回 Trap（R32-SECCOMP-2）
- Pledge：未匹配任何承诺返回 false
- CAP：generation 不匹配返回 InvalidCapId
- LSM livepatch hook：默认 Denied（R104-6）

**IRQ-Safe**：
- CapTable 使用 `spin::Mutex` + `interrupts::without_interrupts`
- LSM 策略读取使用 `AtomicPtr`（无锁）
- Seccomp 回调使用 `spin::RwLock`

**防 Use-After-Free**：CapId 的 48-bit generation 计数器在槽位重用时递增，使旧引用永久失效。281 万亿次分配的空间足以支撑长期运行系统。

**Pledge 命名空间隔离**：R147-2/R151-2/R154-11 FIX 系列确保 PROC 和 THREAD 承诺下的 clone() 拒绝所有命名空间创建标志和不当的 CLONE_VM 组合，防止通过命名空间/地址空间共享提权。

### 4.2 安全加固记录

文档中可见多轮安全审查的修复痕迹：

| 修复编号 | 模块 | 问题 | 修复措施 |
|---------|------|------|----------|
| R25-1 | CAP | 委派时 flag stripping 提权 | 强制继承源能力的 flags |
| R25-2 | CAP | generation 溢出回绕 | `checked_add` + `GenerationExhausted` 错误 |
| R25-3 | LSM | 部分 hook 缺少 denial audit | 为 setresuid/setgroups/prctl/cap_modify 等 hook 补充审计 |
| R29-2 | Seccomp | `evaluate_current` 始终返回 Allow | 改为回调委托模式 |
| R29-3 | LSM | 缺少内存操作 hook | 新增 memory_mmap/mprotect/brk |
| R29-4 | CAP | 32-bit generation 不足 | 扩展为 48-bit generation + 16-bit index |
| R30-1 | CAP | MAX_CAP_SLOTS 超出 u16 范围 | 限制为 65535 |
| R30-3 | LSM | 缺少 munmap hook | 新增 memory_munmap |
| R32-SECCOMP-1 | Seccomp | 跳转验证用 `>` 允许越界 | 改为 `>=` |
| R32-SECCOMP-2 | Seccomp | 运行时跳转越界未处理 | 越界返回 Trap |
| R32-CAP-1 | CAP | generation 用 wrapping_add | 改为 checked_add |
| R62-7 | LSM | 控制面流量（ARP/ICMP）绕过 LSM | 新增 net_control hook + NetControlCtx/NetControlOp |
| R65-13 | LSM | cap 操作常量与审计枚举不一致 | 新增 cap_op 模块统一常量定义 |
| R65-14 | Seccomp | 初始化后 evaluator 丢失仍 Allow | Fail-closed 返回 Kill |
| R66-9 | CAP | CapTable 方法的 LSM/audit 职责不清 | 明确文档：CapTable 不调用 LSM/audit，由 syscall 层负责 |
| R66-10 | CAP | 缺少审计写权限位 | 添加 AUDIT_WRITE 权限 |
| R75-1 | CAP | Socket 缺少网络命名空间 | 添加 net_ns_id 字段 |
| R100-6 | Seccomp | filter ID 哈希读取 padding 字节 | 改为结构化字段哈希 |
| R102-13 | LSM | 缺少 livepatch 安全控制 | 新增 kpatch_load/enable/disable/unload hook |
| R103-4 | LSM | `static mut` 在 Rust 2024 edition 为 UB | 改为 `Box::leak` + `Mutex` |
| R104-6 | LSM | livepatch hook 默认 Ok 允许注入 | 改为默认 Err(Denied)，PermissivePolicy 显式覆盖 |
| R144-3 | Seccomp | strict 模式缺少 rt_sigreturn | 添加 rt_sigreturn(15) 到白名单 |
| R145-7 | Seccomp | exit_group 未始终允许 | pledge 评估器始终放行 exit/exit_group |
| R147-2 | Seccomp | PROC/THREAD pledge 允许创建命名空间 | 拒绝所有 CLONE_NEW* 标志 |
| R148-I9 | Crypto | SHA-256 在 audit 和 livepatch 中重复实现 | 整合到 crypto 模块 |
| R149-I3 | Seccomp | syscall 号硬编码散布两处 | 集中定义到 types.rs 模块级常量 |
| R150-3 | Seccomp | pledge_to_filter 与 promise_allows_syscall 覆盖不一致 | 同步两条路径（CPATH/FATTR/RLIMIT/SENDSIG/TMPPATH 等） |
| R151-2 | Seccomp | PROC 允许 clone(CLONE_VM) 绕过 THREAD 边界 | CLONE_VM 需 THREAD 承诺且满足线程 flag 要求 |
| R152-I1 | Seccomp | fast_allow 仅检查 prog[1]，遗漏后续 syscall | 遍历 JmpEq false-branch 链 |
| R154-11 | Seccomp | THREAD clone 未拒绝命名空间标志 | 显式拒绝 CLONE_NEW* |

### 4.3 职责边界与交互

**CAP 与 LSM 的分工**：
- CapTable 是纯粹的数据结构，不调用 LSM hook 或审计
- "是否允许用户发起 cap 操作"由 syscall 层的 `lsm::hook_task_cap_modify()` 裁决
- 内核内部路径（fork 时的能力复制）绕过 LSM hook，因为策略在父进程获取能力时已检查

**Seccomp 与 LSM 的顺序**：
- Seccomp 在 LSM 之前评估（syscall 入口最外层）
- Seccomp Kill/Trap 直接终止，不进入 LSM 流程
- Seccomp Allow 后才进入 LSM hook 检查
- 具体的 errno 返回、信号投递、进程终止等行为由 `kernel_core/syscall.rs` 编排

**Crypto 模块的定位**：
- 纯粹的共享密码学原语库，不包含策略逻辑
- 当前仅提供 SHA-256，供 audit（HMAC 完整性）和 livepatch（模块哈希）使用
- no_std 兼容，无外部依赖

### 4.4 潜在改进方向

1. **LSM 多策略支持**：当前为单策略模型，未来可考虑策略链（类似 Linux 的 LSM stacking），但需权衡复杂度与性能
2. **Seccomp 安装语义**：`sys_seccomp`/`prctl` 的安全语义（只增不减、自动置 `no_new_privs`、多线程同步）在 `kernel_core/syscall.rs` 中实现，本模块未直接体现
3. **CapTable 并发优化**：当前使用全表 spinlock，高并发场景下可考虑分段锁或 RCU 读优化
4. **审计粒度**：LSM 审计仅在拒绝时触发，成功操作不记录，可能影响安全审计的完整性
5. **Pledge 网络承诺**：UNIX/INET/DNS 承诺的 syscall 映射尚未在 `pledge_to_filter()` 中实现
