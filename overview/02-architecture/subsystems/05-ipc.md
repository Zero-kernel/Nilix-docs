# IPC 模块分析文档

> 分析日期：2026-05-13

## 1. 模块概述

IPC（Inter-Process Communication，进程间通信）模块是 Zero-OS 内核的核心子系统之一，负责提供进程间数据交换与同步的基础设施。该模块位于 `kernel/ipc/` 目录下，以独立 crate 形式组织，通过 `kernel_core` 提供的回调注册机制与系统调用层对接。

### 1.1 设计哲学

IPC 模块遵循以下核心设计原则：

- **能力（Capability）模型**：端点通信采用基于能力的访问控制，发送权限通过显式白名单（`allowed_senders`）管理，发送者身份由内核自动填充，不可伪造。R106-1 修复后引入 generation 机制，防止 PID 复用后的越权发送。
- **命名空间隔离**：端点按 IPC 命名空间分区（R75-2 修复），不同命名空间的端点互不可见、互不可访问，为容器化场景提供隔离基础。
- **有界资源**：消息队列、端点数量、消息大小均设有上限，防止 OOM 和资源耗尽攻击。端点 ID 和管道 ID 使用 `checked_add` 防止溢出回绕（R105-5、R111-3）。
- **阻塞语义**：管道和端点均支持阻塞 I/O，通过 `WaitQueue` 同步原语实现进程挂起与唤醒。
- **Lost-wakeup 防护**：同步原语（KMutex、Semaphore、CondVar）和管道均使用 `prepare_to_wait/finish_wait` 模式，消除"检查条件-入队"之间的唤醒信号丢失窗口。
- **安全清理**：进程退出时自动清理所有 IPC 资源（端点、futex、管道），防止资源泄漏和僵尸等待者。

### 1.2 功能组成

| 子模块 | 功能 | 对应文件 |
|--------|------|----------|
| 端点通信 | 基于能力的消息传递，命名空间隔离，PID generation 防复用 | `ipc.rs` |
| 匿名管道 | 单向字节流通道，环形缓冲区 | `pipe.rs` |
| 同步原语 | WaitQueue、KMutex、Semaphore、CondVar | `sync.rs` |
| Futex | 用户空间快速互斥锁，含优先级继承（PI） | `futex.rs` |
| 模块入口 | 初始化、系统调用回调注册、错误转换 | `lib.rs` |

### 1.3 依赖关系总览

```
lib.rs (入口)
  ├── ipc.rs (端点通信) ──→ sync.rs (WaitQueue)
  ├── pipe.rs (管道) ──→ sync.rs (WaitQueue)
  ├── futex.rs (Futex) ──→ sync.rs (WaitQueue)
  └── sync.rs (同步原语) ──→ kernel_core (进程管理、调度)
```

外部依赖：
- `kernel_core`：进程管理（`current_pid`、`current_generation`、`get_process`）、命名空间（`current_ipc_ns_id`）、调度（`force_reschedule`）、系统调用回调注册
- `mm`：页表管理（futex 地址验证）
- `vfs`：文件系统操作（fd 读写回调中的 VFS 文件处理）
- `drivers`：调试输出宏
- `klog`：结构化日志宏（`klog_always!`、`kprintln!`）
- `trace`：性能计数器（IPC 消息追踪）
- `spin`：自旋锁
- `lazy_static`：延迟初始化全局状态
- `x86_64`：中断控制、页表标志

### 1.4 调用上下文矩阵

| 函数/路径 | 执行上下文 | 可获取的锁 | 可执行的操作 |
|-----------|-----------|-----------|-------------|
| `send_message` / `receive_message` | 系统调用（进程上下文） | ENDPOINTS | 堆分配、调度 |
| `futex_wait` / `futex_wake` | 系统调用（进程上下文） | FUTEX_TABLE、bucket | 堆分配、调度、用户空间访问 |
| `pipe read/write` | 系统调用（进程上下文） | PipeInner | 调度（阻塞模式） |
| `waitqueue_timer_tick` | 定时器中断（IRQ 上下文） | TIMED_WAITERS、waiters | **禁止**堆分配、**禁止**调度 |
| `ipc_cleanup` | 进程退出（持有进程锁） | ENDPOINTS、FUTEX_TABLE | 需避免重入进程锁 |

---

## 2. 文件结构

```
kernel/ipc/
├── Cargo.toml      # crate 配置，声明依赖
├── lib.rs          # 模块入口（402行）：初始化、系统调用回调、错误转换
├── ipc.rs          # 端点通信（733行）：能力模型、命名空间隔离、generation 防复用、阻塞接收
├── pipe.rs         # 匿名管道（556行）：环形缓冲区、阻塞I/O、EOF/EPIPE语义
├── sync.rs         # 同步原语（873行）：WaitQueue、KMutex、Semaphore、CondVar、超时
└── futex.rs        # Futex实现（859行）：WAIT/WAKE/PI锁/robust语义、清理
```

总代码量约 3,423 行（含注释和空行），其中 `sync.rs` 和 `futex.rs` 最为复杂。

---

## 3. 逐文件分析

### 3.1 lib.rs -- 模块入口与系统调用桥接

**文件路径**：`kernel/ipc/lib.rs`（402 行）

#### 3.1.1 功能概述

模块入口文件，承担以下职责：

1. **子模块声明与 re-export**（第 1-38 行）：声明 `futex`、`ipc`、`pipe`、`sync` 四个子模块，并将各模块的公共 API 通过 `pub use` 导出，形成统一的对外接口。
2. **系统调用回调实现**（第 44-341 行）：实现 pipe 创建、fd 读写关闭、futex 操作、IPC 清理等回调函数。
3. **错误类型转换**（第 190-222 行）：`PipeError -> SyscallError` 和 `FsError -> SyscallError` 的映射。
4. **初始化入口**（第 380-402 行）：`init()` 函数注册所有回调并初始化子系统。

#### 3.1.2 核心数据结构

本文件不定义独立数据结构，主要使用其他模块的类型。

#### 3.1.3 核心函数逐段分析

##### `pipe_create_callback()`（第 47-85 行）

创建管道的系统调用回调。执行流程：

1. 获取当前进程 PID 和进程对象（第 51-52 行）
2. 调用 `create_pipe()` 创建管道，获得读端和写端句柄（第 55-56 行）
3. 获取进程锁，分配读端 fd（第 63-65 行）
4. 分配写端 fd，**失败时回滚已分配的读端 fd**（第 68-75 行）
5. 返回 `(read_fd, write_fd)` 元组

**安全设计**：写端 fd 分配失败时通过 `proc.remove_fd(rfd)` 回滚读端，确保不会泄漏半初始化的管道。

##### `fd_read_callback()` / `fd_write_callback()`（第 97-167 行）

文件描述符读写回调，统一处理管道和 VFS 文件。执行流程：

1. 获取当前进程并加锁（第 101-102 行）
2. 通过 `get_fd(fd)` 获取 fd 对象（第 106 行）
3. 使用 `as_any().downcast_ref::<PipeHandle>()` 尝试管道类型（第 109 行）
4. 若为管道：**克隆句柄 -> 释放进程锁 -> 执行 I/O**（第 110-112 行）
5. 若非管道：尝试 `downcast_ref::<FileHandle>()`，同样克隆后释放锁再 I/O（第 116-123 行）
6. 都不匹配返回 `EBADF`（第 125 行）

**关键安全修复**：
- **R32-IPC-1**：单次锁获取确定 fd 类型并获取句柄，避免 TOCTOU（先检查类型、再获取句柄之间 fd 被替换）。
- **R41-3**：克隆文件句柄后释放进程锁再执行 I/O。如果持锁执行阻塞 I/O（如管道读取），其他线程无法操作该进程的 fd 表，造成 DoS。

##### `fd_close_callback()`（第 172-187 行）

文件描述符关闭回调。**R155-3 修复**：先在持锁期间提取 fd 对象（`remove_fd`），然后在**锁外** Drop 该对象，避免锁反转（Process lock -> socket waiters -> Process lock）。

```rust
let removed = {
    let mut proc = process.lock();
    proc.remove_fd(fd).ok_or(SyscallError::EBADF)?
};
drop(removed);  // 在锁外 Drop，避免锁反转
```

##### `futex_callback()`（第 245-341 行）

Futex 系统调用分发入口。执行流程：

1. **地址验证**（第 258-289 行）：
   - 4 字节对齐检查（`uaddr & 0x3 != 0`）
   - 用户空间范围检查（`uaddr < PHYSICAL_MEMORY_OFFSET`）
   - 页表标志验证（PRESENT + USER_ACCESSIBLE），检查 u32 的起始和结束地址所在页
2. **获取 TGID**（第 291-299 行）：R37-2 修复，使用线程组 ID 而非线程 PID，确保 `CLONE_THREAD` 兄弟线程共享 futex 键空间
3. **操作分发**（第 301-341 行）：根据 `op` 分发到 `futex_wait`、`futex_wake`、`futex_lock_pi`、`futex_unlock_pi`

**R39-6 修复**：`FUTEX_WAIT` 和 `FUTEX_WAIT_TIMEOUT` 统一处理，区别仅在于是否传入 `timeout_ns`。

##### `ipc_cleanup()`（第 351-354 行）

进程退出时的 IPC 清理回调。接受 `pid`、`tgid`、`ipc_ns_id` 三个参数（R37-2 和 R75-2 修复：直接接受 TGID 和命名空间 ID，避免在清理回调中重新锁定进程对象导致死锁）。依次调用 `cleanup_process_endpoints()` 和 `cleanup_process_futexes()`。

##### `futex_wake_callback()`（第 370-373 行）

线程退出时 `clear_child_tid` 机制的 futex 唤醒回调。使用 TGID 作为键唤醒等待者，确保 `CLONE_THREAD` 兄弟线程可互相唤醒。

##### `init()`（第 380-402 行）

IPC 子系统初始化入口。按顺序执行：

1. 初始化 WaitQueue 超时定时器（R39-6）
2. 注册 IPC 清理回调到进程管理子系统
3. 注册 futex 唤醒回调（用于 `clear_child_tid`）
4. 注册系统调用回调（pipe、fd_read、fd_write、fd_close、futex）
5. 调用 `ipc::init()` 初始化端点系统
6. 使用 `klog_always!` 输出初始化日志

##### 错误转换函数（第 190-222 行）

`pipe_error_to_syscall()` 和 `fs_error_to_syscall()` 提供完整的错误码映射：

| PipeError | SyscallError | | FsError | SyscallError |
|-----------|-------------|---|---------|-------------|
| WouldBlock | EAGAIN | | NotFound | ENOENT |
| BrokenPipe | EPIPE | | IsDir | EISDIR |
| Closed | EPIPE | | PermDenied | EACCES |
| InvalidPipe / InvalidOperation | EBADF | | NoSpace/NoMem | ENOMEM |
| PipeIdExhausted | ENOMEM | | Io | EIO |
| NoCurrentProcess | ESRCH | | CrossDev | EXDEV |
|  |  | | SymlinkLoop | ELOOP |
|  |  | | NotSupported | ENOSYS |
|  |  | | Pipe | EPIPE |
|  |  | | NotEmpty / Busy | EBUSY |

#### 3.1.4 模块间依赖

- **kernel_core**：系统调用回调注册（`register_pipe_callback` 等 5 个注册函数 + `register_ipc_cleanup`）、进程管理（`current_pid`、`get_process`）、`FileOps` trait、`SyscallError`
- **mm::page_table**：`with_current_manager`、`PHYSICAL_MEMORY_OFFSET`（futex 地址验证）
- **vfs::traits::FileHandle**：VFS 文件读写的 downcast 目标
- **x86_64**：`PageTableFlags`、`VirtAddr`（futex 地址验证）

#### 3.1.5 当前不足与改进建议

1. **futex 错误转换代码重复**：`futex_callback` 中 `FUTEX_WAIT`、`FUTEX_LOCK_PI`、`FUTEX_UNLOCK_PI` 三个分支的 `FutexError -> SyscallError` 转换代码完全相同，应提取为公共函数 `futex_error_to_syscall()`。

2. **futex 地址验证与 futex_wait 内部的二次读取存在重复页表遍历**：`futex_callback` 验证地址后，`futex_wait` 内部的 `read_user_u32()` 会再次遍历页表。虽然二次检查是防 lost-wakeup 的必要措施，但两次页表遍历有性能开销。注意：不能简单缓存验证结果，因为页表映射对用户态是可变的，缓存会引入新的 TOCTOU。

---

### 3.2 ipc.rs -- 端点通信系统

**文件路径**：`kernel/ipc/ipc.rs`（733 行）

#### 3.2.1 功能概述

实现基于能力（capability）的端点消息传递系统。每个端点属于一个进程（owner），只有 owner 可以接收消息。发送权限通过 `allowed_senders` 白名单控制，发送者身份由系统自动从 `current_pid()` 获取，不可伪造。

R75-2 修复引入了 IPC 命名空间隔离：端点注册表按 `NamespaceId` 分区，不同命名空间的端点互不可见、互不可访问。

R106-1 修复引入了 PID generation 机制：`allowed_senders` 使用 `BTreeMap<ProcessId, u64>`（PID -> generation 映射）替代原来的 `BTreeSet<ProcessId>`，防止 PID 复用后新进程继承旧进程的 IPC 发送权限。

文件分为两大部分：
- **第 1-584 行**：核心端点通信（注册、发送、接收、权限管理、清理）
- **第 586-733 行**：阻塞 IPC 扩展（等待队列、阻塞接收、超时重试）

#### 3.2.2 核心数据结构

##### `Message` / `ReceivedMessage`（第 37-52 行）

```rust
pub struct Message {
    pub sender: ProcessId,  // 由系统自动填充，不可伪造
    pub data: Vec<u8>,
}
pub struct ReceivedMessage {
    pub sender: ProcessId,
    pub data: Vec<u8>,
}
```

两个结构体字段完全相同，`ReceivedMessage` 是面向接收方的视图。设计意图是将内部 `Message`（可能包含更多元数据）与外部 `ReceivedMessage` 解耦，但当前实现中两者无差异。

##### `IpcError`（第 55-73 行）

```rust
pub enum IpcError {
    NoCurrentProcess,     // 无当前进程上下文
    ProcessNotFound,      // R106-1: 目标进程不存在（无法查询 generation）
    EndpointNotFound,     // 端点不存在或不在当前命名空间
    AccessDenied,         // 无发送权限或非端点所有者
    QueueFull,            // 消息队列已满（背压）
    MessageTooLarge,      // 消息超过 MAX_MESSAGE_SIZE
    TooManyEndpoints,     // 端点数量超过 MAX_ENDPOINTS_PER_PROCESS
    EndpointIdExhausted,  // R105-5: 全局端点 ID 空间耗尽
}
```

相比旧版新增 `ProcessNotFound` 和 `EndpointIdExhausted` 两个错误变体。

##### `Endpoint`（第 82-153 行）

```rust
struct Endpoint {
    owner: ProcessId,
    owner_generation: u64,                    // R106-1: 所有者 generation
    allowed_senders: BTreeMap<ProcessId, u64>, // R106-1: PID -> generation 映射
    queue: VecDeque<Message>,
}
```

端点内部结构，不对外暴露。关键不变量：
- `owner` 始终在 `allowed_senders` 中（构造时自动插入）
- `queue.len() <= MAX_MESSAGES_PER_ENDPOINT`（由 `push_message` 保证）
- `owner` 的权限不可通过 `revoke_access` 撤销
- R106-1：`can_send()` 同时验证 PID 和 generation，防止 PID 复用后的越权访问

##### `EndpointRegistry`（第 159-272 行）

```rust
struct EndpointRegistry {
    per_ns: BTreeMap<NamespaceId, BTreeMap<ProcessId, BTreeMap<EndpointId, Endpoint>>>,
    owner_index: BTreeMap<EndpointId, (NamespaceId, ProcessId)>,
}
```

全局端点注册表，R75-2 修复后按命名空间分区。采用三层嵌套 BTreeMap：

- 第一层：`NamespaceId` -> 命名空间内的所有端点
- 第二层：`ProcessId` -> 该进程拥有的所有端点
- 第三层：`EndpointId` -> 具体端点

`owner_index` 是反向索引，用于从 `EndpointId` 快速定位所属命名空间和进程，避免全表扫描。

R105-5 修复：`register_endpoint()` 使用 `fetch_update` + `checked_add` 分配端点 ID，防止 u64 溢出回绕到 0。

R106-1 修复：`register_endpoint()` 接受预解析的 `(pid, generation)` 对，所有 PROCESS_TABLE 查询必须在获取 ENDPOINTS 锁之前完成（防止锁序反转死锁）。

##### 常量配置（第 28-34 行）

| 常量 | 值 | 说明 |
|------|-----|------|
| `MAX_MESSAGES_PER_ENDPOINT` | 64 | 每端点最大消息数（背压阈值） |
| `MAX_ENDPOINTS_PER_PROCESS` | 32 | 每进程最大端点数 |
| `MAX_MESSAGE_SIZE` | 4096 | 单条消息最大字节数 |

##### 全局状态（第 275-280 行）

```rust
static NEXT_ENDPOINT_ID: AtomicU64 = AtomicU64::new(1);
static ref ENDPOINTS: Mutex<EndpointRegistry> = Mutex::new(EndpointRegistry::default());
```

`NEXT_ENDPOINT_ID` 使用 `fetch_update` + `checked_add` 保证全局唯一性且不回绕。`ENDPOINTS` 是全局单锁保护的注册表。

#### 3.2.3 核心函数逐段分析

##### `Endpoint::new()`（第 99-120 行）

构造新端点。R106-1 修复：接受 `owner_generation` 和预解析的 `resolved_senders: &[(ProcessId, u64)]`，将 `owner` 连同其 generation 自动加入 `allowed_senders`。

##### `Endpoint::can_send()`（第 125-127 行）

R106-1 修复：同时验证 PID 和 generation（`self.allowed_senders.get(&sender).copied() == Some(sender_generation)`），防止 PID 复用后的越权发送。

##### `register_endpoint()`（第 305-325 行）

1. 获取 owner PID、generation 和 IPC 命名空间 ID（第 306-308 行）
2. R106-1：**在获取 ENDPOINTS 锁之前**解析所有 sender 的 generation（第 312-320 行），避免锁序反转
3. 获取 ENDPOINTS 锁，调用 `EndpointRegistry::register_endpoint()`（第 322-324 行）

##### `send_message()`（第 349-374 行）

1. 自动获取发送者 PID、generation 和 IPC 命名空间 ID（第 351-353 行）
2. 检查消息大小限制（第 356-358 行）
3. 获取全局锁，查找端点（命名空间隔离）（第 360-363 行）
4. R106-1：检查发送权限（PID + generation 双重验证）（第 366-368 行）
5. 推入消息队列（背压检查在 `push_message` 内部）（第 370 行）
6. 递增 IPC 消息计数器（第 372 行）

**注意**：此函数**不唤醒**阻塞接收者。只有 `send_message_notify()` 才会唤醒。

##### `receive_message()`（第 397-421 行）

非阻塞接收。R106-1 修复：验证调用者是端点 owner 时同时检查 PID 和 generation。队列为空时返回 `Ok(None)`。

##### `grant_access()`（第 433-456 行）

R106-1 修复：在获取 ENDPOINTS 锁之前查询目标进程的 generation，显式 `drop(target_proc)` 确保不跨 ENDPOINTS 锁持有进程引用。

##### `destroy_endpoint()`（第 501-526 行）

端点销毁。执行顺序至关重要（X-6 修复）：

1. 获取锁验证 owner 权限（含 R106-1 generation 验证）（第 506-514 行）
2. **释放锁**（第 516 行）
3. **重新获取锁**移除端点注册（第 519 行）
4. 清理等待队列，唤醒所有阻塞接收者（第 523 行）

**顺序约束**：必须先移除端点注册，再清理等待队列。这确保被唤醒的线程在下一次 `receive_message` 时立即看到 `EndpointNotFound`，避免重新创建等待队列导致再次阻塞。

##### `cleanup_process_endpoints()`（第 542-563 行）

进程退出时的端点清理。同样遵循"先移除注册，再清理等待队列"的顺序：

1. 收集该进程的所有端点 ID（持锁，第 544-552 行）
2. 移除端点注册（持锁，第 557 行）
3. 逐个清理等待队列（不持锁，第 560-562 行）

##### `get_queue_length()`（第 568-584 行）

R106-1 修复：查看队列状态时同时验证 PID 和 generation。

##### `send_message_notify()`（第 620-635 行）

发送消息并唤醒等待者。先调用 `send_message()`，然后：

```rust
let wq = {
    let queues = ENDPOINT_WAIT_QUEUES.lock();
    queues.get(&endpoint_id).cloned()  // 克隆 Arc
};
if let Some(wq) = wq {
    wq.wake_one();  // 在锁外唤醒
}
```

**X-6 修复**：克隆 `Arc` 后释放锁，避免在持有 `ENDPOINT_WAIT_QUEUES` 锁时调用 `wake_one()`（可能触发调度）。

##### `receive_message_blocking()`（第 654-682 行）

阻塞接收。核心循环：

```rust
loop {
    match receive_message(endpoint_id)? {
        Some(msg) => return Ok(msg),
        None => {
            let wq = get_or_create_wait_queue(endpoint_id);
            if wq.is_closed() { return Err(EndpointNotFound); }
            let waited = wq.wait();
            if !waited {
                if wq.is_closed() { return Err(EndpointNotFound); }
                return Err(NoCurrentProcess);
            }
        }
    }
}
```

**X-6 安全增强**：使用 `Arc<WaitQueue>` 避免 use-after-free；检查 `is_closed()` 避免在已销毁端点上永久阻塞。

##### `cleanup_wait_queue()`（第 720-733 行）

端点销毁时清理等待队列。使用 `close()` 而非 `wake_all()`：

1. 从 `ENDPOINT_WAIT_QUEUES` 中移除并获取 `Arc`（第 723-726 行）
2. 调用 `wq.close()` 设置 closed 标志并唤醒所有等待者（第 731 行）

#### 3.2.4 模块间依赖

- `kernel_core::process`：`current_pid()`、`current_generation()`、`ProcessId`、`get_process()`
- `kernel_core::current_ipc_ns_id()`：获取当前 IPC 命名空间 ID
- `trace::counters`：`increment_counter(TraceCounter::IpcMessages, 1)` 追踪 IPC 消息
- `sync::WaitQueue`：阻塞接收的等待队列

#### 3.2.5 当前不足与改进建议

1. **`receive_message_blocking()` 存在 lost-wakeup 竞态**（第 654-682 行）：`receive_message()` 返回 `None` 到 `wq.wait()` 之间存在窗口。如果发送方在此窗口内调用 `send_message_notify()` 并执行 `wake_one()`，由于接收方尚未入队，唤醒信号丢失，接收方将永久阻塞。管道模块已使用 `prepare_to_wait/finish_wait` 模式解决此问题，端点接收应采用相同模式。

2. **`send_message()` 与 `send_message_notify()` 的语义割裂**：`send_message()` 不唤醒阻塞接收者，如果发送方误用 `send_message()` 而非 `send_message_notify()`，接收方可能永远收不到消息。建议将唤醒逻辑合并到 `send_message()` 中，或移除 `send_message()` 的公共接口。

3. **`destroy_endpoint()` 的两次锁获取**（第 506-519 行）：先获取锁验证 owner，释放后再获取锁移除端点。两次锁获取之间存在竞态窗口。虽然当前设计中只有 owner 可以销毁端点，但仍建议合并为单次锁获取。

4. **`EndpointRegistry` 三层嵌套 BTreeMap 的性能**：每次端点查找需要三次 BTreeMap 查询（O(log n) 每次）。对于高频 IPC 场景，可考虑使用 HashMap 或扁平化为单层 `BTreeMap<EndpointId, Endpoint>` + 辅助索引。

5. **消息数据使用 `Vec<u8>` 堆分配**：每条消息都需要堆分配。高频小消息场景下分配器压力大。可考虑小消息内联优化（如 `SmallVec<[u8; 64]>`）。

6. **IPC 计数器在发送和接收时都递增**：`send_message`（第 372 行）和 `receive_message`（第 418 行）都调用 `increment_counter(TraceCounter::IpcMessages, 1)`，导致一条消息被计数两次。建议仅在发送时计数，或使用不同的计数器区分发送/接收。

7. **`receive_message_with_retries()` 的 busy-wait 模式**（第 696-710 行）：通过 `force_reschedule()` 让出 CPU 后重试，本质上是轮询。在高负载下浪费 CPU 时间，建议使用 `receive_message_blocking()` 的超时变体替代。

8. **`allowed_senders` 无数量上限**：`BTreeMap<ProcessId, u64>` 可以无限增长。恶意进程可通过反复调用 `grant_access()` 耗尽内存。建议添加 `MAX_ALLOWED_SENDERS` 限制。

---

### 3.3 pipe.rs -- 匿名管道实现

**文件路径**：`kernel/ipc/pipe.rs`（556 行）

#### 3.3.1 功能概述

实现经典的 Unix 匿名管道，提供进程间单向数据通道。核心特性：

- **环形缓冲区**：使用固定大小的 `Vec<u8>` 作为环形缓冲区，通过 `read_pos` 和 `len` 管理读写位置
- **阻塞/非阻塞模式**：通过 `PipeFlags::nonblock` 控制
- **EOF/EPIPE 语义**：写端全关闭时读端返回 EOF(0)；读端全关闭时写端返回 `BrokenPipe`
- **引用计数**：通过 `readers`/`writers` 计数管理管道生命周期
- **FileOps 集成**：实现 `FileOps` trait，可存储在进程 fd 表中
- **ID 溢出保护**：R111-3 修复，使用 `fetch_update` + `checked_add` 防止管道 ID 回绕到 0

#### 3.3.2 核心数据结构

##### `PipeError`（第 24-40 行）

```rust
pub enum PipeError {
    NoCurrentProcess,
    BrokenPipe,
    WouldBlock,
    Closed,
    InvalidPipe,
    InvalidOperation,   // 在错误端执行读/写
    PipeIdExhausted,    // R111-3: ID 分配耗尽
}
```

相比旧版新增 `InvalidOperation` 和 `PipeIdExhausted` 两个变体。

##### `PipeInner`（第 64-77 行）

```rust
struct PipeInner {
    buffer: Vec<u8>,    // 环形缓冲区（创建时分配，大小固定）
    capacity: usize,    // 缓冲区容量
    read_pos: usize,    // 读位置（head）
    len: usize,         // 当前数据长度
    readers: usize,     // 读端引用计数
    writers: usize,     // 写端引用计数
}
```

**不变量**：
- `0 <= len <= capacity`
- `0 <= read_pos < capacity`
- 写位置 = `(read_pos + len) % capacity`（隐式计算，不存储）
- `buffer.len() == capacity`（创建后不变）

##### `Pipe`（第 155-164 行）

```rust
pub struct Pipe {
    id: PipeId,
    inner: Mutex<PipeInner>,
    read_wait: WaitQueue,   // 等待读取的进程队列
    write_wait: WaitQueue,  // 等待写入的进程队列
}
```

使用两个独立的 `WaitQueue` 分别管理读等待者和写等待者，避免虚假唤醒。

##### `PipeHandle`（第 357-364 行）

```rust
pub struct PipeHandle {
    pipe: Arc<Pipe>,        // 底层管道（共享所有权）
    end_type: PipeEndType,  // 端类型（Read/Write）
    flags: PipeFlags,       // 标志（nonblock, cloexec）
}
```

管道端句柄，通过 `Arc<Pipe>` 共享底层管道。`Clone` 实现委托给 `duplicate()`，会增加对应端的引用计数。

#### 3.3.3 核心函数逐段分析

##### `PipeInner::read()`（第 112-129 行）

从环形缓冲区读取数据：

1. 计算实际可读字节数 `to_read = min(dst.len(), self.len)`（第 113 行）
2. 循环处理环形回绕：每次读取 `min(剩余, capacity - read_pos)` 字节（第 119-124 行）
3. 更新 `read_pos`（取模回绕）和 `len`（第 123、127 行）

##### `PipeInner::write()`（第 132-151 行）

向环形缓冲区写入数据：

1. 计算实际可写字节数 `to_write = min(src.len(), self.space())`（第 133 行）
2. 计算写位置 `write_pos = (read_pos + len) % capacity`（第 138 行）
3. 循环处理环形回绕（第 142-147 行）
4. 更新 `len`（第 149 行）

##### `Pipe::read()`（第 202-238 行）

管道读取的完整逻辑，包含 Z-11 lost-wakeup 修复：

```
loop {
    获取 inner 锁 {
        if 有数据 -> 读取 + 唤醒写者 + return Ok(n)
        if 写端全关闭 -> return Ok(0)  // EOF
        if 非阻塞 -> return Err(WouldBlock)
        // Z-11: 在持有锁时加入等待队列
        should_wait = read_wait.prepare_to_wait()
    }  // 释放锁
    if should_wait -> read_wait.finish_wait()  // 实际阻塞
    // 被唤醒后循环重新检查
}
```

**Z-11 修复关键**：`prepare_to_wait()` 在持有 `inner` 锁时调用，确保"检查条件"和"入队"是原子的。即使写者在释放锁后立即写入并调用 `wake_one()`，由于读者已在队列中，唤醒信号不会丢失。

##### `Pipe::write()`（第 254-301 行）

管道写入逻辑，支持部分写入：

```
while total_written < src.len() {
    获取 inner 锁 {
        if 读端全关闭 -> return BrokenPipe (或已写入的字节数)
        if 有空间 -> 写入 + 唤醒读者
        if 写满 -> return Ok(total_written)
        if 非阻塞 -> return WouldBlock (或已写入的字节数)
        // Z-11: 在持有锁时加入等待队列
        should_wait = write_wait.prepare_to_wait()
    }
    if should_wait -> write_wait.finish_wait()
}
```

##### `PipeHandle::read()` / `PipeHandle::write()`（第 396-409 行）

读写方法先验证端类型（`PipeEndType::Read` / `PipeEndType::Write`），类型不匹配返回 `InvalidOperation` 错误。

##### `PipeHandle::duplicate()`（第 427-439 行）

复制句柄（用于 fork）：

1. 根据端类型调用 `add_reader()` 或 `add_writer()` 增加引用计数
2. 克隆 `Arc<Pipe>` 和标志

##### `Drop for PipeHandle`（第 459-465 行）

句柄销毁时根据端类型调用 `close_read()` 或 `close_write()`：

- `close_read()`：减少 `readers` 计数，唤醒所有写者（让它们检测到 EPIPE）
- `close_write()`：减少 `writers` 计数，唤醒所有读者（让它们检测到 EOF）

##### `FileOps for PipeHandle`（第 469-507 行）

实现 `FileOps` trait 使管道句柄可存储在进程 fd 表中：

- `clone_box()`：调用 `duplicate()` 创建 `Box<dyn FileOps>`
- `as_any()`：支持 downcast 到 `PipeHandle`
- `type_name()`：返回 `"PipeRead"` 或 `"PipeWrite"`
- `stat()`：R41-1 修复，返回 `S_IFIFO | 0o666` 模式

##### `create_pipe()` / `create_pipe_with_capacity()`（第 516-534 行）

管道工厂函数。R111-3 修复：使用 `fetch_update` + `checked_add` 原子递增 `NEXT_PIPE_ID`，防止溢出回绕到 0。创建 `Arc<Pipe>`，返回读端和写端句柄。

#### 3.3.4 模块间依赖

- `sync::WaitQueue`：读写阻塞等待（`prepare_to_wait`、`finish_wait`、`wake_one`、`wake_all`）
- `kernel_core::FileOps`：文件描述符 trait
- `kernel_core::VfsStat`：fstat 返回值结构
- `kernel_core::SyscallError`：错误类型

#### 3.3.5 当前不足与改进建议

1. **引用计数非原子操作**：`readers` 和 `writers` 是 `usize` 类型，通过 `Mutex<PipeInner>` 保护。`add_reader()` / `add_writer()` 需要获取整个 `PipeInner` 锁仅为修改一个计数器。可考虑使用 `AtomicUsize` 减少锁竞争，但需注意与 `close_read/close_write` 中的唤醒逻辑的原子性。

2. **缓冲区大小固定**：创建后缓冲区大小不可调整。Linux 支持通过 `fcntl(F_SETPIPE_SZ)` 动态调整管道大小（最大 1MB）。

3. **缺少 PIPE_BUF 原子写入保证**：POSIX 要求小于 `PIPE_BUF`（通常 4096 字节）的写入是原子的——要么全部写入，要么不写入。当前实现允许部分写入后阻塞，不满足此语义。

4. **缺少 `poll()` / `select()` 支持**：没有实现非阻塞 I/O 多路复用接口，限制了管道在事件驱动架构中的使用。

5. **`PipeId` 使用 `AtomicUsize`**（第 511 行）：而 `EndpointId` 使用 `AtomicU64`，类型不一致。在 64 位系统上效果相同，但语义上应统一。

6. **测试代码不完整**（第 536-556 行）：`test_pipe_basic()` 函数未标记 `#[test]`，且在内核环境中无法直接运行（依赖进程上下文）。

7. **`close_read()` / `close_write()` 使用 `saturating_sub`**（第 304-317 行）：虽然防止了下溢，但如果计数器意外为 0 时被调用，说明存在 bug。建议在 debug 模式下添加断言。

---

### 3.4 futex.rs -- Futex 实现

**文件路径**：`kernel/ipc/futex.rs`（859 行）

#### 3.4.1 功能概述

实现用户空间快速互斥锁（Futex）的内核支持，是 pthread mutex/condvar 的底层基础设施。这是 IPC 模块中最复杂的文件之一，包含五种操作和完整的优先级继承（PI）支持。

支持的操作：

| 操作码 | 常量值 | 说明 |
|--------|--------|------|
| `FUTEX_WAIT` | 0 | 如果 `*uaddr == val` 则阻塞，否则返回 EAGAIN |
| `FUTEX_WAKE` | 1 | 唤醒最多 n 个等待者 |
| `FUTEX_WAIT_TIMEOUT` | 2 | 同 WAIT，但支持纳秒级超时（R39-6 修复） |
| `FUTEX_LOCK_PI` | 3 | 带优先级继承的互斥锁加锁（E.4 PI） |
| `FUTEX_UNLOCK_PI` | 4 | 带优先级继承的互斥锁解锁（E.4 PI） |

#### 3.4.2 核心数据结构

##### `FutexBucket`（第 49-60 行）

```rust
struct FutexBucket {
    queue: Arc<WaitQueue>,                     // 等待队列
    waiter_count: usize,                       // 活跃等待者计数
    owner: Option<ProcessId>,                  // PI: 当前持有者
    owner_dead: bool,                          // PI: 持有者已死亡（robust）
    pi_waiters: BTreeMap<ProcessId, Priority>,  // PI: 等待者优先级表
}
```

**不变量**：
- `waiter_count >= pi_waiters.len()`（非 PI 等待者不在 `pi_waiters` 中）
- `owner.is_some()` 时 `owner` 不在 `pi_waiters` 中
- `owner_dead == true` 仅在 `owner` 退出但尚未被继任者接管时成立

##### `FUTEX_TABLE`（第 74-81 行）

```rust
static ref FUTEX_TABLE: Mutex<BTreeMap<FutexKey, Arc<Mutex<FutexBucket>>>> =
    Mutex::new(BTreeMap::new());
```

全局 futex 表，以 `(tgid, uaddr)` 为键。使用两层锁：外层 `Mutex` 保护表结构，内层 `Mutex<FutexBucket>` 保护单个桶。

**R37-2 修复**：键使用 TGID 而非 PID，确保 `CLONE_THREAD` 兄弟线程共享 futex 键空间，保持 pthread 语义。

##### `FutexError`（第 32-46 行）

```rust
pub enum FutexError {
    WouldBlock,        // 值不匹配
    InvalidOperation,  // 无效操作（如非持有者解锁）
    Fault,             // 内存访问错误
    NoProcess,         // 无当前进程
    TimedOut,          // 等待超时（R39-6）
    OwnerDied,         // 锁持有者已退出（E.4 robust）
}
```

#### 3.4.3 核心函数逐段分析

##### `read_user_u32()`（第 87-127 行）

安全读取用户空间 u32 值，用于 futex_wait 的二次检查。

1. 使用 `with_current_manager` 获取页表管理器（第 95-96 行）
2. 计算结束地址并检查溢出（第 98-100 行）
3. 验证起止两页的 PRESENT + USER_ACCESSIBLE 标志（第 103-116 行）
4. P1-6 修复：使用 `copy_from_user_safe` 读取，移除了冗余的外层 `UserAccessGuard`（`copy_from_user_safe` 内部自带 guard）（第 121-122 行）

##### `futex_wait()`（第 144-220 行）

FUTEX_WAIT / FUTEX_WAIT_TIMEOUT 的核心实现。这是 futex 最关键的代码路径：

```
1. 检查 current_value == expected（第 152-153 行）
2. 获取/创建 bucket（第 160 行）
3. 增加 waiter_count + 获取 queue Arc + 释放 bucket 锁（第 164-168 行）
4. 二次读取 futex 值（第 172-196 行）
   - 值仍匹配 -> 继续阻塞
   - 值已变化 -> 回滚 waiter_count + return WouldBlock
   - 读取失败 -> 回滚 + return Fault
5. 调用 queue.wait_with_timeout(timeout_ns)（第 201 行）
6. 被唤醒后减少 waiter_count（第 204-209 行）
7. 尝试清理空桶（第 212 行）
8. 根据 WaitOutcome 返回结果（第 215-219 行）
```

**关键设计**：步骤 3 中释放 bucket 锁后再调用 `wait_with_timeout()`，避免持有 bucket 锁时阻塞导致死锁。步骤 4 的二次读取防止 lost-wakeup 竞态（值在步骤 1 和步骤 5 之间被修改）。

##### `futex_wake()`（第 235-262 行）

FUTEX_WAKE 实现，相对简单：

1. 查找 bucket（不创建，不存在则返回 0）（第 240-243 行）
2. 获取 queue Arc 后释放 bucket 锁（第 247-250 行）
3. 调用 `queue.wake_n(n)` 唤醒最多 n 个等待者（第 251 行）
4. 尝试清理空桶（第 257-259 行）

##### `futex_lock_pi()`（第 282-446 行）

带优先级继承的互斥锁加锁，是整个模块最复杂的函数。

**快路径**（第 300-331 行）：
```
获取 bucket 锁 {
    清理已死亡的 owner（robust）
    if 锁空闲 -> 设置 owner = pid + return Ok(0)
    if 自己已持有 -> return Err(InvalidOperation)  // 防死锁
    增加 waiter_count
}
```

**慢路径**（第 333-446 行）：
```
1. 记录 waiting_on_futex（用于链式 PI）（第 334-336 行）
2. 获取 queue Arc，queue.prepare_to_wait()（先入队）（第 340-351 行）
3. 记录 pi_waiters（入队后再记录，避免 race）（第 354-357 行）
4. R73-1 窗口修复：再次检查 owner 状态（第 362-386 行）
   - 如果 owner 已为 None -> 直接获取锁 + cancel_wait
5. 触发 PI 传播到当前 owner（第 389 行）
6. queue.finish_wait()（实际阻塞）（第 392 行）
7. 被唤醒后：
   - 减少 waiter_count + 移除 pi_waiters（第 401-405 行）
   - 检查是否已被 unlock_pi 设置为 owner（第 410-423 行）
   - 如果 owner 为 None -> 接管锁（第 426-429 行）
8. 清除 waiting_on_futex + 重新计算 PI + 清理空桶
```

**R73-1 窗口修复**（第 359-386 行）：在 `prepare_to_wait()` 和 `pi_waiters.insert()` 之间，如果 owner 已经 unlock，此时 owner 会是 None。如果不检查就直接 `finish_wait()`，会永久阻塞。修复方案是在 `pi_waiters.insert()` 后再次检查 owner 状态。

##### `futex_unlock_pi()`（第 464-528 行）

带优先级继承的互斥锁解锁：

1. 验证当前线程是持有者（第 476-477 行）
2. 清理已退出的等待者（第 481-482 行）
3. 选择最高优先级等待者作为继任者（第 485 行）
4. 转移所有权（第 487-495 行）
5. 清除自身 PI 提升（第 502-507 行）
6. 传递剩余 PI 到新 owner + 链式传播（第 509-515 行）
7. 唤醒新 owner（`wake_specific`）（第 521 行）

##### `cleanup_process_futexes()`（第 556-690 行）

进程退出时的 futex 清理，分三个阶段：

**阶段 1：清理 waiter 条目**（第 562-616 行，R72-1 修复）

遍历所有以 TGID 为键的 bucket，移除退出进程的 pi_waiters 条目和 WaitQueue 条目。这防止 PID 复用后新进程被错误唤醒。

R72-1（Codex review）修复：只递减一次 `waiter_count`——即使进程同时出现在 `pi_waiters` 和 WaitQueue 中，也只计数一次。

**阶段 2：处理 owner 死亡**（第 618-667 行，E.4 robust 语义）

遍历退出进程持有的所有 futex：
- 标记 `owner_dead = true`
- 选择最高优先级等待者作为继任者
- 转移所有权并唤醒继任者

**阶段 3：线程组最后一个线程退出时清理桶**（第 669-690 行）

如果线程组还有其他活跃线程（`group_size > 1`），只做 owner_dead 标记，不移除桶。只有最后一个线程退出时才移除所有桶并唤醒所有等待者。

##### `cleanup_empty_bucket()`（第 704-728 行）

清理空的 futex 桶，使用双重检查防止竞态：

1. 检查 bucket 是否为空（waiter_count == 0 && queue.is_empty() && owner.is_none() && pi_waiters.is_empty()）
2. 释放 bucket 锁，获取 FUTEX_TABLE 锁
3. 使用 `Arc::ptr_eq` 确认是同一个桶（防止竞态条件下移除新创建的桶）
4. 再次检查是否为空
5. 确认后移除

##### `apply_pi_and_propagate()`（第 777-837 行）

链式优先级继承传播。R72-2 修复后使用迭代式实现（避免递归栈溢出）：

```
work_stack = [(key, owner, donated)]
while let Some((cur_key, cur_owner, donation)) = work_stack.pop() {
    depth += 1
    if depth > MAX_PI_CHAIN_DEPTH(64) -> break

    获取 cur_owner 的进程锁 {
        apply_pi_boost 或 clear_pi_boost
        获取 next_wait = get_waiting_on_futex()
    }

    if changed -> request_resched_from_irq()

    if next_wait 存在且未访问过 {
        查找 next_owner -> push to work_stack
    }
}
```

**R72-2 修复**：原始实现使用递归调用，恶意用户可构造 N 个进程的等待链（A -> B -> C -> ... -> N），导致 O(N) 栈帧溢出 16KB 内核栈。迭代版本使用 O(1) 栈空间。

#### 3.4.4 模块间依赖

- `kernel_core::process`：`current_pid`、`get_process`、`FutexKey`、`Priority`、`ProcessId`、`thread_group_size`
- `kernel_core::request_resched_from_irq`：PI 优先级变更后触发重调度
- `kernel_core::usercopy`：`copy_from_user_safe`（SMAP 保护）
- `mm::page_table`：`with_current_manager`、`PageTableManager`（地址验证）
- `sync::WaitQueue`：等待队列（`wait_with_timeout`、`wake_n`、`wake_specific`、`prepare_to_wait`、`finish_wait`、`cancel_wait`）

#### 3.4.5 当前不足与改进建议

1. **全局 FUTEX_TABLE 单锁瓶颈**：所有 futex 操作都需要先获取 `FUTEX_TABLE` 的全局 Mutex 来查找/创建 bucket。Linux 使用 256 个哈希桶减少竞争。建议采用类似方案：`static FUTEX_BUCKETS: [Mutex<BTreeMap<...>>; 256]`，以 `hash(tgid, uaddr) % 256` 分桶。

2. **PI 链传播中的 Vec 堆分配**（第 785 行）：`apply_pi_and_propagate()` 中的 `work_stack` 使用 `Vec::with_capacity(8)`。虽然预分配了 8 个元素，但深链可能超出导致堆分配。在某些调用路径中（如 `cleanup_process_futexes` 被 IRQ 触发时），堆分配可能不安全。建议使用固定大小的栈数组。

3. **`cleanup_process_futexes()` 的两次全表扫描**（第 562-667 行）：先扫描清理 waiter 条目，再扫描处理 owner 死亡。两次扫描都需要获取 FUTEX_TABLE 锁并 collect 到 Vec。建议合并为单次扫描。

4. **PI futex 不支持超时**：`futex_lock_pi()` 使用 `finish_wait()` 而非 `wait_with_timeout()`，不支持 `FUTEX_LOCK_PI` 的超时变体（Linux 的 `FUTEX_LOCK_PI2`）。

5. **`FutexTable` 类型别名为空元组**（第 736 行）：`pub type FutexTable = ()` 仅用于向后兼容，应清理移除。

6. **`futex_wait()` 的二次读取窗口**（第 170-201 行）：二次读取发生在 `wait_with_timeout()` 的入队之前。虽然概率极低，但理论上 waker 可在"二次读之后、入队之前"完成 wake 导致永久阻塞。可考虑使用 `prepare_to_wait` + 二次读取 + `finish_wait` 模式（类似 `futex_lock_pi` 的做法）。

7. **`select_highest_waiter()` 的 O(n) 复杂度**（第 743-748 行）：遍历所有等待者找最小优先级。如果等待者数量大，可考虑使用 `BTreeMap` 按优先级排序或维护最小堆。

---

### 3.5 sync.rs -- 同步原语

**文件路径**：`kernel/ipc/sync.rs`（873 行）

#### 3.5.1 功能概述

提供内核空间的同步机制，是所有阻塞 IPC 操作的基础设施。包含四种同步原语：

1. **WaitQueue**（等待队列）：最基础的阻塞/唤醒原语，被管道、端点、futex 广泛使用
2. **KMutex**（内核互斥锁）：可阻塞的互斥锁，适用于长时间持有锁的场景。R155-7 修复后使用 `prepare_to_wait/cancel_wait` 模式防止 lost-wakeup。
3. **Semaphore**（计数信号量）：控制对有限资源的并发访问。R154-6 修复后使用 `prepare_to_wait/cancel_wait` 模式防止 lost-wakeup。
4. **CondVar**（条件变量）：等待特定条件成立。R153-7 修复后使用 `prepare_to_wait/finish_wait` 模式防止 lost-wakeup。

此外还包含 R39-6 修复引入的超时等待支持基础设施。

#### 3.5.2 核心数据结构

##### `WaitQueue`（第 64-71 行）

```rust
pub struct WaitQueue {
    waiters: Mutex<VecDeque<ProcessId>>,  // 等待的进程 ID 列表
    closed: AtomicBool,                    // 关闭标志（X-6 安全增强）
    timed_out: Mutex<BTreeSet<ProcessId>>, // 因超时被唤醒的进程集合（R39-6）
}
```

**状态机**：

```
                    +----------+
                    |   Open   | <-- new()
                    +----+-----+
                         | close()
                    +----v-----+
                    |  Closed  | --> 拒绝新等待者，唤醒所有现有等待者
                    +----------+
```

等待者状态机：

```
                prepare_to_wait()          finish_wait()
    [不在队列] ----------------> [在队列/Blocked] ------------> [阻塞中]
        ^                            |                            |
        |                            | cancel_wait()              | wake_one/wake_all/timeout
        |                            v                            v
        +---------------------- [不在队列/Ready] <------------- [被唤醒/Ready]
```

##### `WaitOutcome`（第 22-31 行）

```rust
pub enum WaitOutcome {
    Woken,      // 被正常唤醒
    TimedOut,   // 等待超时
    Closed,     // 队列已关闭
    NoProcess,  // 无当前进程
}
```

##### `TimedWaiter`（第 34-42 行）

```rust
#[derive(Debug, Clone, Copy)]
struct TimedWaiter {
    queue: usize,           // WaitQueue 的裸地址
    pid: ProcessId,         // 等待的进程 ID
    deadline_tick: u64,     // 超时截止时间（tick）
}
```

R143-5 修复：`register_timed_wait()` 中添加 `debug_assert!` 验证 `queue` 地址位于内核静态地址空间（>= 0xFFFF_FFFF_8000_0000），防止堆/栈上的 WaitQueue 意外注册定时等待导致 use-after-free。

##### `KMutex`（第 478-485 行）

```rust
pub struct KMutex {
    locked: AtomicBool,
    wait_queue: WaitQueue,
    owner: Mutex<Option<ProcessId>>,  // 调试用
}
```

##### `Semaphore`（第 584-589 行）

```rust
pub struct Semaphore {
    count: AtomicU32,
    wait_queue: WaitQueue,
}
```

##### `CondVar`（第 698-701 行）

```rust
pub struct CondVar {
    wait_queue: WaitQueue,
}
```

##### 全局超时状态（第 44-48 行）

```rust
static TIMED_WAITERS: Mutex<Vec<TimedWaiter>> = Mutex::new(Vec::new());
static WAITQUEUE_TIMER_INIT: AtomicBool = AtomicBool::new(false);
```

#### 3.5.3 核心函数逐段分析

##### `WaitQueue::wait_with_timeout()`（第 104-185 行）

带超时的阻塞等待，是所有阻塞操作的核心：

```
1. 获取当前 PID（第 105-108 行）
2. 快速检查 closed 标志（Acquire 序）（第 111-113 行）
3. 零超时直接返回 TimedOut（第 116-118 行）
4. R154-10: 计算 deadline_tick，使用 checked_add 防止溢出（第 125-129 行）
5. without_interrupts {
     二次检查 closed（Relaxed 序）
     R153-1: 重复入队检查，避免同一 PID 多次入队
     设置 ProcessState::Blocked
   }（第 134-158 行）
6. 注册超时（如果有 deadline）（第 166-169 行）
7. force_reschedule()（让出 CPU）（第 172 行）
8. 取消超时登记（第 175-177 行）
9. 检查 timed_out 标志（第 180-184 行）
```

**R154-10 修复**：使用 `checked_add` 防止用户提供巨大的 `timeout_ns` 值（如 `u64::MAX`）时溢出，产生接近零的 tick 值。

**R153-1 修复**：入队前检查是否已存在，防止虚假重调度导致同一 PID 多次入队，消耗其他等待者的唤醒信号。

##### `WaitQueue::timeout_wake()`（第 191-224 行）

超时唤醒处理。R155-2 修复后使用 `try_lock()` 避免 IRQ 上下文中死锁：

1. R155-2：对进程锁使用 `try_lock()` 而非 `lock()`，如果锁被占用则返回 `false`（下一个 tick 重试）
2. R155-12：如果进程已不存在（被杀死），只从 waiters 中移除，不插入 `timed_out`（防止无人消费的标记泄漏和 PID 复用误判）
3. 从 waiters 中移除指定 PID
4. 如果进程状态为 Blocked，设置为 Ready
5. 在 `timed_out` 集合中标记该 PID

##### `WaitQueue::prepare_to_wait()`（第 365-402 行）

Z-11 修复引入的分步等待 API。在持有外部锁时调用，将当前进程加入等待队列并设置为 Blocked 状态，但不触发调度。

R152-8 修复：入队前检查是否已存在，防止重复入队。如果已在队列中，直接返回 `true`。

**用途**：解决"检查条件 -> 入队"之间的 lost-wakeup 竞态。调用者在持有保护条件的锁时调用 `prepare_to_wait()`，确保"检查条件为假"和"入队"是原子的。

##### `WaitQueue::finish_wait()`（第 408-411 行）

完成等待，调用 `force_reschedule()` 实际阻塞。必须在 `prepare_to_wait()` 返回 true 后调用，且在释放外部锁之后。

##### `WaitQueue::cancel_wait()`（第 421-447 行）

取消等待。如果在 `prepare_to_wait()` 后发现条件已满足，调用此函数从队列移除并恢复 Ready 状态。同时取消定时等待。

##### `WaitQueue::wake_one()`（第 234-250 行）

唤醒队列中的一个进程：

1. 在 `without_interrupts` 中弹出队首 PID
2. 取消该进程的定时等待（R39-6）
3. 如果进程状态为 Blocked，设置为 Ready

##### `WaitQueue::wake_n()`（第 286-313 行）

唤醒最多 n 个进程，用于 FUTEX_WAKE。

**注意**：只有状态为 Blocked 的进程才计入 `woken` 计数。如果进程已被其他路径唤醒（状态不是 Blocked），从队列移除但不计入返回值。

##### `WaitQueue::wake_specific()`（第 326-344 行）

E.4 PI 引入的精确唤醒。在队列中查找指定 PID 并唤醒，用于 `FUTEX_UNLOCK_PI` 选择最高优先级等待者。

##### `WaitQueue::close()`（第 462-465 行）

关闭队列并唤醒所有等待者（X-6 安全增强）：

```rust
pub fn close(&self) {
    self.closed.store(true, Ordering::Release);
    self.wake_all();
}
```

关闭后的队列不再接受新的等待者（`wait_with_timeout` 和 `prepare_to_wait` 会检查 `closed` 标志）。

##### `KMutex::lock()`（第 500-531 行）

R155-7 修复：使用 `prepare_to_wait/cancel_wait` 模式防止 lost-wakeup：

```rust
pub fn lock(&self) {
    loop {
        if CAS(false -> true) 成功 -> 记录 owner + return

        // R155-7: 先入队，再重试 CAS
        if !prepare_to_wait() -> return

        if CAS(false -> true) 成功 -> cancel_wait + 记录 owner + return

        // CAS 仍然失败，实际阻塞
        finish_wait()
    }
}
```

与旧版对比，消除了 CAS 失败到入队之间的 lost-wakeup 窗口。

##### `KMutex::unlock()`（第 554-567 行）

R154-16 修复：添加 `debug_assert!` 验证调用者是锁的持有者。

```rust
pub fn unlock(&self) {
    debug_assert!({ owner.is_none() || *owner == current_pid() });
    *self.owner.lock() = None;
    self.locked.store(false, Ordering::Release);
    self.wait_queue.wake_one();
}
```

##### `Semaphore::wait()`（第 615-655 行）

R154-6 修复：使用 `prepare_to_wait/cancel_wait` 模式防止 lost-wakeup：

```rust
pub fn wait(&self) {
    loop {
        // 快路径：尝试 CAS 递减
        if CAS(count -> count-1) 成功 -> return

        // R154-6: 先入队
        if !prepare_to_wait() -> return

        // 入队后重新检查 count
        if CAS(count -> count-1) 成功 -> cancel_wait + return

        // count 仍为 0，实际阻塞
        finish_wait()
    }
}
```

##### `Semaphore::signal()`（第 682-687 行）

R154-17 修复：使用 `saturating_add` 防止 u32 溢出回绕（count 从 MAX 回到 0 会清除所有 permit）。

##### `CondVar::wait()`（第 727-743 行）

R153-7 修复：使用 `prepare_to_wait/finish_wait` 模式，消除了旧版中 `unlock()` 和 `wait()` 之间的 lost-wakeup 窗口：

```rust
pub fn wait(&self, mutex: &KMutex) {
    // R153-7: 先入队（在持有 mutex 时）
    if !self.wait_queue.prepare_to_wait() {
        return;
    }

    // 释放锁 -- notifiers can now run
    mutex.unlock();

    // 实际阻塞（此时已在队列中，不会错过 wake）
    self.wait_queue.finish_wait();

    // 重新获取锁
    mutex.lock();
}
```

##### 超时支持辅助函数（第 762-873 行）

**`register_timed_wait()`**（第 784-808 行）：

R143-5 修复：`debug_assert!` 验证 WaitQueue 地址位于内核静态区域。

R154-10 修复：入队前去重检查，防止同一 `(queue, pid)` 对的重复登记。

**`process_waitqueue_timeouts()`**（第 826-867 行）：

在定时器中断中处理超时。使用固定大小栈数组（`MAX_TIMEOUTS_PER_TICK = 16`）避免 IRQ 中堆分配：

1. 获取 `TIMED_WAITERS` 锁
2. 遍历列表，将已超时的条目移到栈数组中（最多 16 个）
3. 释放锁
4. 对每个超时条目，将 `usize` 转换回 `*const WaitQueue` 并调用 `timeout_wake()`
5. R155-2 修复：若 `timeout_wake()` 返回 false（进程锁争用），将条目重新插回 `TIMED_WAITERS`，下一个 tick 重试

**`waitqueue_timer_tick()`**（第 870-873 行）：定时器回调入口，获取当前 tick 并调用 `process_waitqueue_timeouts()`。

#### 3.5.4 模块间依赖

- `kernel_core::process`：`current_pid`、`get_process`、`ProcessId`、`ProcessState`
- `kernel_core::force_reschedule`：触发调度让出 CPU
- `kernel_core::get_ticks`：获取当前 tick 计数
- `kernel_core::register_timer_callback`：注册定时器回调
- `x86_64::instructions::interrupts`：`without_interrupts` 中断控制

#### 3.5.5 当前不足与改进建议

1. **`TimedWaiter.queue` 使用裸指针（usize）的 use-after-free 风险**（第 35-42 行、第 849 行）：`process_waitqueue_timeouts()` 在定时器中断中将 `usize` 转换回 `*const WaitQueue` 并解引用。R143-5 的 `debug_assert!` 只在 debug 模式下生效，release 模式仍有风险。建议改为 `Arc<WaitQueue>` 或 `Weak<WaitQueue>` + generation ID。

2. **超时处理在 IRQ 上下文中执行**：`waitqueue_timer_tick()` 在定时器中断中调用，会获取 `TIMED_WAITERS` 锁和 `waiters` 锁，并修改 `ProcessState`。这拉长了中断关闭时间。R155-2 修复通过 `try_lock()` 缓解了部分死锁风险，但 `TIMED_WAITERS` 锁的进程/IRQ 上下文竞争仍然存在。建议将超时标记延迟到进程上下文处理（如使用 per-CPU 的超时队列）。

3. **`MAX_TIMEOUTS_PER_TICK = 16` 的限制**（第 817 行）：每个 tick 最多处理 16 个超时。如果大量 futex 同时超时（如压力测试），剩余的会延迟到下一个 tick 处理，导致超时精度下降。

4. **`wake_n()` 中的 woken 计数不准确**（第 286-313 行）：只有当进程状态为 Blocked 时才递增 `woken` 计数。如果进程已被其他路径唤醒（状态不是 Blocked），从队列中移除但不计入 woken，导致返回值可能小于实际从队列中移除的数量。

5. **`TIMED_WAITERS` 使用 `Vec` 线性扫描**（第 45 行）：每个 tick 都需要遍历整个列表检查超时。大量定时等待者时 O(n) 扫描开销显著。建议使用最小堆（`BinaryHeap`）按 deadline 排序，将检查复杂度降为 O(k)（k 为超时数量）。

6. **`receive_message_blocking()` 存在 lost-wakeup 竞态**：`receive_message()` 返回 `None` 到 `wq.wait()` 之间的窗口仍未使用 `prepare_to_wait/finish_wait` 模式修复。

---

## 4. 模块整体评估

### 4.1 架构设计评估

**优点**：

- **分层清晰**：sync（基础设施）-> pipe/futex/ipc（具体机制）-> lib.rs（系统调用桥接），依赖方向单一，无循环依赖。
- **能力模型安全**：端点通信的 `allowed_senders` 白名单 + 不可伪造的发送者身份 + R106-1 generation 防 PID 复用，提供了强访问控制。
- **命名空间隔离**：R75-2 修复后端点按 IPC 命名空间分区，为容器化场景提供了隔离基础。
- **PI futex 完整性**：实现了链式优先级继承传播、robust futex 语义、PID 复用防护，功能完整度高。
- **同步原语 lost-wakeup 修复**：R153-7（CondVar）、R154-6（Semaphore）、R155-7（KMutex）、Z-11（pipe）均已使用 `prepare_to_wait/finish_wait` 模式修复。
- **回调注册模式**：通过 `kernel_core` 的回调注册机制解耦 IPC 模块与系统调用层，便于模块化。
- **fd_close 锁反转修复**：R155-3 修复使 fd_close 在锁外 Drop 对象，消除了原来的 Process -> socket waiters -> Process 锁反转。

**不足**：

- **端点阻塞接收的 lost-wakeup 问题**：`receive_message_blocking` 仍未使用 `prepare_to_wait/finish_wait` 模式，是模块内唯一残留的 lost-wakeup 竞态。
- **全局单锁设计**：`ENDPOINTS`、`FUTEX_TABLE`、`ENDPOINT_WAIT_QUEUES`、`TIMED_WAITERS` 都是全局 `spin::Mutex`，在高并发场景下是瓶颈。
- **超时基础设施的安全性**：`TimedWaiter` 使用裸指针存储 WaitQueue 地址，是 use-after-free 的隐患（R143-5 的 debug_assert 仅缓解不根治）。

### 4.2 安全性评估

**已实施的安全措施**：

| 修复标签 | 安全措施 | 涉及文件 |
|----------|----------|----------|
| R32-IPC-1 | fd 读写单次锁获取避免 TOCTOU | lib.rs |
| R32-IPC-2 | futex 地址对齐/范围/页表验证 | lib.rs |
| R41-3 | 克隆句柄后释放锁再 I/O | lib.rs |
| R24-5 | 跨页安全读取 + SMAP 保护 | futex.rs |
| P1-6 | 移除冗余 UserAccessGuard | futex.rs |
| R37-2 | TGID 作为 futex 键（pthread 语义） | futex.rs, lib.rs |
| R39-6 | futex 超时支持 | futex.rs, sync.rs |
| R72-1 | 退出时清理 waiter 条目防 PID 复用 | futex.rs |
| R72-2 | 迭代式 PI 传播防栈溢出 | futex.rs |
| R73-1 | PI lock 窗口修复 | futex.rs |
| R75-2 | IPC 命名空间隔离 | ipc.rs |
| R105-5 | 端点 ID checked_add 防溢出 | ipc.rs |
| R106-1 | PID generation 防复用越权 | ipc.rs |
| R111-3 | 管道 ID checked_add 防溢出 | pipe.rs |
| R143-5 | TimedWaiter 地址静态区域 assert | sync.rs |
| R152-8 | prepare_to_wait 重复入队检查 | sync.rs |
| R153-1 | wait_with_timeout 重复入队检查 | sync.rs |
| R153-7 | CondVar prepare_to_wait 修复 | sync.rs |
| R154-6 | Semaphore lost-wakeup 修复 | sync.rs |
| R154-10 | 超时计算 checked_add + 去重 | sync.rs |
| R154-16 | KMutex unlock owner 断言 | sync.rs |
| R154-17 | Semaphore signal saturating_add | sync.rs |
| R155-2 | timeout_wake try_lock 防 IRQ 死锁 | sync.rs |
| R155-3 | fd_close 锁外 Drop 防锁反转 | lib.rs |
| R155-7 | KMutex lost-wakeup 修复 | sync.rs |
| R155-12 | 已死进程超时清理不标记 timed_out | sync.rs |
| X-6 | 端点销毁时唤醒等待者 + closed 标志 | ipc.rs, sync.rs |
| Z-11 | prepare_to_wait/finish_wait 防 lost-wakeup | sync.rs, pipe.rs |
| E.4 | PI futex + robust 语义 | futex.rs |

**残留安全风险**（按优先级排序）：

| 优先级 | 风险 | 位置 | 影响 |
|--------|------|------|------|
| P0 | `TimedWaiter.queue` 裸指针 use-after-free | sync.rs:849 | 内核崩溃 |
| P0 | `receive_message_blocking()` lost-wakeup | ipc.rs:654 | 永久阻塞 |
| P0 | `futex_wait()` 二次读与入队之间的 lost-wakeup 窗口 | futex.rs:170-201 | 永久阻塞 |
| P1 | `timeout_wake()` 与正常唤醒竞态导致错误返回 TimedOut | sync.rs:191 | 语义错误 |
| P1 | 端点/管道退出清理路径中 wake_all 可能重入进程锁 | ipc.rs/pipe.rs | 死锁 |
| P2 | PI 链传播 Vec 堆分配（最坏情况内存/延迟不可控） | futex.rs:785 | 性能/鲁棒性 |

### 4.3 性能评估

**优点**：

- futex 快路径（值不匹配立即返回）避免不必要的内核阻塞
- 管道环形缓冲区读写 O(n) 且无堆分配
- 超时处理使用固定大小栈数组避免 IRQ 中堆分配
- IPC 消息计数器使用 Per-CPU 计数器（`trace::counters`）
- R155-2：timeout_wake 使用 try_lock 避免 IRQ 中自旋等待，失败时重新入队下一 tick 重试

**瓶颈**：

| 瓶颈 | 位置 | 影响 | 建议 |
|------|------|------|------|
| FUTEX_TABLE 全局 Mutex | futex.rs:79 | 所有 futex 操作串行化 | 哈希桶分片 |
| ENDPOINTS 全局 Mutex | ipc.rs:279 | 所有端点操作串行化 | Per-namespace 锁 |
| TIMED_WAITERS 全局 Mutex | sync.rs:45 | 定时器中断竞争 | Per-CPU 队列 |
| 三层嵌套 BTreeMap | ipc.rs:162 | 3 次 O(log n) 查找 | 扁平化 + HashMap |
| TIMED_WAITERS 线性扫描 | sync.rs:826 | O(n) 每 tick | 最小堆 |

### 4.4 锁序与依赖图

当前模块涉及的锁及其获取顺序。注意：许多路径刻意在获取下一层锁前释放上一层锁（如 futex 路径中 clone queue Arc 后释放 bucket 锁），因此实际嵌套关系需要区分"同时持有"和"顺序获取"。

**实际同时持有的锁嵌套链**：

```
链 1（futex 桶元数据）：
  FUTEX_TABLE -> FutexBucket
  （注：bucket 锁释放后才进入 WaitQueue，不同时持有）

链 2（WaitQueue 阻塞/唤醒）：
  WaitQueue::waiters -> Process lock（设置 ProcessState）
  WaitQueue::waiters -> TIMED_WAITERS（cancel_timed_wait）

链 3（端点通信）：
  ENDPOINTS -> （无嵌套锁）
  ENDPOINT_WAIT_QUEUES -> （clone Arc 后释放，不嵌套）

链 4（管道）：
  PipeInner -> WaitQueue::waiters（prepare_to_wait 在持有 PipeInner 锁时调用）
            -> Process lock

链 5（超时处理，IRQ 上下文）：
  TIMED_WAITERS -> WaitQueue::waiters -> Process lock (try_lock, R155-2)
```

**关中断边界**：`WaitQueue` 的入队/出队/唤醒操作大量包在 `interrupts::without_interrupts` 内。这意味着在这些临界区内不会被 timer ISR 抢占。

**R155-2 改善**：`timeout_wake()` 中对进程锁使用 `try_lock()`，如果锁被占用则不自旋等待而是返回 `false`，由 `process_waitqueue_timeouts()` 将条目重新插回 `TIMED_WAITERS` 下一个 tick 重试。这消除了之前进程上下文持有进程锁时被 timer 中断打断导致自旋死锁的风险。

### 4.5 改进路线图

#### P0 -- 必须修复（内存安全 / 永久阻塞）

1. **将 `TimedWaiter.queue: usize` 改为 `Arc<WaitQueue>` 或 `Weak<WaitQueue>`**
   - 消除 use-after-free 风险
   - 使用 `Weak` 可在 WaitQueue 释放后优雅降级（`upgrade()` 返回 None）

2. **修复 `receive_message_blocking()` 的 lost-wakeup**
   - 采用 `prepare_to_wait/finish_wait` 模式：在持有 ENDPOINTS 锁检查队列为空后 `prepare_to_wait`，释放后 `finish_wait`

3. **修复 `futex_wait()` 的二次读取窗口**
   - 改为 `prepare_to_wait` -> 二次读取 -> `finish_wait` 模式，或让"值检查+入队"在同一桶锁临界区内原子化

#### P1 -- 应当修复（性能 / 锁安全 / 语义正确性）

1. **FUTEX_TABLE 哈希桶分片**：参考 Linux 的 256 桶设计
2. **修复 `timeout_wake()` 与正常唤醒的竞态**：仅在成功从 waiters 中移除 PID 时才标记 `timed_out`，确保 prefer-wake 语义
3. **提取 futex 错误转换为公共函数**
4. **合并 `send_message` 和 `send_message_notify`**：消除语义割裂

#### P2 -- 可选优化

1. 管道支持 `fcntl(F_SETPIPE_SZ)` 动态调整大小
2. 管道支持 PIPE_BUF 原子写入语义
3. 端点消息小消息内联优化（`SmallVec`）
4. 支持 poll/select 非阻塞 I/O 多路复用
5. PI futex 支持超时变体
6. `TIMED_WAITERS` 改用最小堆
7. `allowed_senders` 添加数量上限
8. 清理 `FutexTable = ()` 类型别名
9. PI 链传播 `work_stack` 改用固定大小栈数组（消除最坏情况下的堆分配和延迟不可控）
