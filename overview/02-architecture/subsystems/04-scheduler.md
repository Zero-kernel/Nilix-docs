# 调度器 (sched) 模块深度分析文档

> 分析日期：2026-05-13
> 源码版本：涵盖 R65-R155 修复、E.4/E.5/F.2/G.1 特性、R152-3 饥饿防止再桶化修复

## 1. 模块概述

`sched` 模块是 Zero-OS 内核的进程调度子系统，负责 CPU 时间分配、进程上下文切换、CPU 隔离与负载均衡。模块实现了基于多级反馈队列（MLFQ）思想的动态优先级调度算法，支持 SMP 多核架构下的 Per-CPU 就绪队列、Work Stealing 负载均衡、CPU 亲和性（cpuset）隔离、Cgroup CPU 配额执行，以及完整的 Ring 0/Ring 3 上下文切换（含 FPU 惰性保存、TLS/SWAPGS 支持）。

模块还包含一套运行时锁依赖检测（Lockdep）框架，用于在 debug 构建中检测死锁风险。

### 1.1 设计原则

- **Per-CPU 架构**：就绪队列、当前进程、重调度标志均为 Per-CPU 存储，消除全局锁竞争
- **安全优先**：通过 Lockdep、上下文验证（`assert_kernel_context`）、cpuset 隔离保证等机制防止特权提升和调度逃逸
- **回调驱动**：调度器通过 `kernel_core` 的回调注册机制与定时器中断、系统调用返回、进程生命周期事件集成
- **渐进式演进**：保留旧版调度器（`scheduler.rs`）作为参考/回退，增强型调度器（`enhanced_scheduler.rs`）为系统主线
- **Cgroup 感知**：调度决策集成 cpu.max 配额节流，被节流的进程不会被选中运行

### 1.2 模块依赖关系图

```text
                    ┌─────────────┐
                    │   sched     │
                    │  (lib.rs)   │
                    └──────┬──────┘
           ┌───────────────┼───────────────┬──────────────┐
           ▼               ▼               ▼              ▼
  ┌────────────────┐ ┌──────────┐ ┌──────────────┐ ┌───────────┐
  │ enhanced_      │ │scheduler │ │lock_ordering │ │  cpuset   │
  │ scheduler.rs   │ │.rs (旧版)│ │.rs           │ │  .rs      │
  └───────┬────────┘ └────┬─────┘ └──────────────┘ └─────┬─────┘
          │               │                               │
          ▼               ▼                               ▼
  ┌───────────────────────────────────────────────────────────┐
  │                    kernel_core                             │
  │  (process, cgroup, scheduler_hook, time, callbacks)        │
  └──────────────────────┬────────────────────────────────────┘
                         │
          ┌──────────────┼──────────────┬──────────────┐
          ▼              ▼              ▼              ▼
     ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
     │  arch   │  │ cpu_local │  │  trace   │  │ drivers  │
     │(ctx_sw, │  │(CpuLocal, │  │(counters,│  │(kprintln)│
     │ IPI,    │  │ PerCpu)   │  │ watchdog)│  │          │
     │ syscall)│  │           │  │          │  │          │
     └─────────┘  └──────────┘  └──────────┘  └──────────┘
```

## 2. 文件结构

```
kernel/sched/
├── Cargo.toml                 # 包配置与依赖声明（23行）
├── lib.rs                     # 模块入口（39行）：子模块声明、re-export、初始化
├── process.rs                 # 旧版 PCB 定义（279行）：未被模块树引用，遗留代码
├── scheduler.rs               # 旧版优先级调度器（315行）：全局单锁，仅 demo 使用
├── enhanced_scheduler.rs      # 增强型调度器（1535行）：系统主线调度器
├── cpuset.rs                  # CPU 集合隔离（480行）：层级 cpuset 管理
└── lock_ordering.rs           # 锁顺序与 Lockdep 框架（837行）：9级锁层级 + 运行时检测
```

**代码量统计**：总计约 3,508 行 Rust 代码（不含 Cargo.toml）。

---

## 3. 逐文件分析

---

### 3.1 Cargo.toml（包配置，23行）

#### 功能概述

定义 `sched` crate 的元数据、feature flags 和依赖关系。

#### 逐段分析

```toml
[package]
name = "sched"
version = "0.1.0"
edition = "2021"
```
- 使用 Rust 2021 edition，版本号 0.1.0 表明处于早期开发阶段。

```toml
[features]
default = []
lockdep_graph = []
```
- `lockdep_graph` feature：启用锁依赖图的环检测功能。默认关闭，仅在 debug 构建中有意义。该 feature 控制 `lock_ordering.rs` 中 `dep_graph` 模块的编译。

```toml
[dependencies]
spin = "0.9"
lazy_static = { version = "1.4", features = ["spin_no_std"] }
kernel_core = { path = "../kernel_core" }
drivers = { path = "../drivers" }
arch = { path = "../arch" }
cpu_local = { path = "../cpu_local" }
x86_64 = "0.15"
trace = { path = "../trace" }
klog = { path = "../klog" }
```
- 新增 `klog` 依赖：统一的内核日志宏 crate，替代原有的直接 `println!` 调用。

#### 当前不足与改进建议

1. **缺少 `no_std` 声明**：虽然 `lib.rs` 中有 `#![no_std]`，但 Cargo.toml 未显式声明 `default-features = false` 对某些依赖的约束
2. **`lockdep_graph` feature 未与 debug profile 联动**：需要手动启用，建议在 `.cargo/config.toml` 中为 debug profile 默认启用

---

### 3.2 lib.rs（模块入口，39行）

#### 功能概述

模块入口文件，职责包括：声明子模块、re-export 关键类型、提供模块级初始化函数。

#### 逐行分析

```rust
// 第1-3行：no_std 环境声明与 alloc 引入
#![no_std]
#![feature(abi_x86_interrupt)]
extern crate alloc;
```
- `abi_x86_interrupt` 是 nightly feature，允许定义 x86 中断处理函数。此处声明是因为模块可能间接涉及中断上下文。

```rust
// 第6-9行：外部 crate 引入
#[macro_use]
extern crate drivers;
#[macro_use]
extern crate klog;        // 新增：统一日志宏 (klog!, klog_always!, kprintln!)
extern crate kernel_core;
extern crate lazy_static;
extern crate spin;
```

```rust
// 第15行：关键 re-export
pub use kernel_core::process;
```
- **重要设计决策**：`sched` crate 不定义自己的 `process` 模块，而是直接 re-export `kernel_core::process`。这意味着 `sched::process::Process`、`sched::process::ProcessId` 等类型实际来自 `kernel_core`。

```rust
// 第17-20行：子模块声明
pub mod cpuset;
pub mod enhanced_scheduler;
pub mod lock_ordering;
pub mod scheduler;
```
- 注意：**没有 `pub mod process;`**，因此 `process.rs` 文件虽然存在于目录中，但不会被编译进 crate。

```rust
// 第22-29行：类型 re-export
pub use enhanced_scheduler::Scheduler;
pub use lock_ordering::{LockClassKey, LockdepMutex, LockLevel};
pub use cpuset::{CpusetError, CpusetId, CpusetNode};
```

```rust
// 第31-35行：初始化函数
pub fn init() {
    klog_always!("Scheduler module initialized");
    enhanced_scheduler::init();
    // Note: cpuset::init() should be called after CPU enumeration in main.rs
}
```
- 仅初始化增强型调度器，**不初始化旧版 `scheduler.rs`**。
- cpuset 初始化被延迟到 `main.rs` 中 CPU 枚举完成后调用。
- 使用 `klog_always!` 替代原有的 `println!`，提供统一日志等级支持。

```rust
// 第37-39行：空的 schedule 函数
pub fn schedule() {
    // 简单的调度函数
}
```
- **问题**：函数体为空，实际调度由 `enhanced_scheduler` 的回调机制驱动。此函数容易误导调用者。

#### 当前不足与改进建议

1. **空的 `schedule()` 函数**：应移除或转发到 `enhanced_scheduler::Scheduler::schedule()`，避免误导
2. **`process.rs` 未声明为模块**：文件存在但不编译，应删除或移至 `docs/` 目录
3. **cpuset 初始化分离**：`cpuset::init()` 需要在 `main.rs` 中单独调用，增加了初始化顺序的心智负担

---

### 3.3 process.rs（旧版进程控制块，279行 — 遗留代码）

#### 功能概述

定义了一套独立的进程控制块（`ProcessControlBlock`）、进程状态（`ProcessState`）、优先级（`Priority`）等类型。**此文件未被模块树引用（`lib.rs` 中无 `mod process;`），不会被编译，属于遗留代码。**

#### 核心数据结构

**`ProcessState` 枚举**（第27-41行）：
```rust
pub enum ProcessState {
    New,         // 新创建
    Ready,       // 就绪态
    Running,     // 运行态
    Blocked,     // 阻塞态
    Zombie,      // 僵尸态
    Terminated,  // 已终止
}
```
- 与 `kernel_core::process::ProcessState` 相比，缺少 `Sleeping`、`Stopped` 状态。

**`Priority` 结构体**（第44-53行）：
```rust
pub struct Priority(pub u8);
impl Priority {
    pub const IDLE: Priority = Priority(0);
    pub const LOW: Priority = Priority(5);
    pub const NORMAL: Priority = Priority(10);
    pub const HIGH: Priority = Priority(15);
    pub const REALTIME: Priority = Priority(20);
}
```

**`allocate_pid()`**（第20-24行）：
- **P2-8 修复**：使用 `fetch_update` + `checked_add` 防止 u64 溢出归零（遵循 R105-5 模式），返回 `Option<Pid>`。

#### 核心函数分析

| 函数 | 行号 | 作用 | 分析 |
|------|------|------|------|
| `allocate_pid()` | 20-24 | 全局 PID 分配 | P2-8 修复：`fetch_update` + `checked_add`，返回 `Option<Pid>` |
| `new()` | 114-135 | 创建内核进程 PCB | 调用 `arch::Context::init_for_process()` |
| `new_user_process()` | 138-164 | 创建用户态进程 PCB | 调用 `arch::Context::init_for_user_process()` |
| `reset_time_slice()` | 194-203 | 按优先级重置时间片 | IDLE:50, LOW:75, NORMAL:100, HIGH:150, REALTIME:200 |
| `consume_time_slice()` | 206-214 | 消耗时间片 | 返回是否耗尽，同时累加 cpu_time |
| `allocate_fd()` | 222-235 | 分配文件描述符 | 线性扫描空闲槽位，O(n) 复杂度 |

#### 当前不足与改进建议

1. **与 `kernel_core::process::Process` 完全重复**：应删除此文件，避免概念混淆
2. **`page_table_root` 初始化为 0**：无后续赋值逻辑，属于未完成的设计
3. **文件描述符实现过于简化**：`FileDescriptor` 仅有 fd 和 flags，缺少实际文件句柄引用

---

### 3.4 scheduler.rs（旧版优先级调度器，315行）

#### 功能概述

实现了一个基于全局 `Mutex` 保护的优先级调度器 `PriorityScheduler`，使用 `BTreeMap<Priority, Vec<ProcessId>>` 组织就绪队列。这是系统早期的调度器实现，目前仅作为参考/demo 保留，**不在系统主线初始化路径中**。

#### 核心数据结构

**`PriorityScheduler` 结构体**（第9-18行）：
```rust
pub struct PriorityScheduler {
    ready_queues: BTreeMap<Priority, alloc::vec::Vec<ProcessId>>,
    current_process: Option<ProcessId>,
    stats: SchedulerStats,
}
```
- 使用 `Vec<ProcessId>` 作为同优先级队列，**真正的 FIFO 语义**（`queue.remove(0)` 从头部取出）。

**`SchedulerStats` 结构体**（第217-223行）：
```rust
#[derive(Debug, Default, Clone, Copy)]
pub struct SchedulerStats {
    pub scheduled: u64,
    pub enqueued: u64,
    pub woken: u64,
    pub ticks: u64,
}
```
- `print()` 方法使用 `klog!(Info, ...)` 替代原有 `println!`。

**全局实例**（第236-238行）：
```rust
static ref SCHEDULER: Mutex<PriorityScheduler> = Mutex::new(PriorityScheduler::new());
```

#### 核心函数逐段分析

**`add_process()`**（第30-44行）：
- 从全局进程表获取 PCB，读取动态优先级后设置 Ready 状态。
- **问题**：先 `drop(proc)` 再入队，存在 TOCTOU 窗口。

**`schedule()`**（第58-146行）— 核心调度逻辑：
- **R98-1 修复**：在选择进程时检查 `!proc.stopped`（第129行）并在 Running 状态中检查 `proc.stopped`（第69行），防止选中被 job-control 暂停的进程。
- **性能问题**：`queue.remove(0)` 是 O(n) 操作。

**`tick()`**（第149-172行）：
- 饥饿防止机制较为粗糙：每 100 tick 调用一次 `update_dynamic_priority()`。

**`init()`**（第241-251行）：
- 创建 init 进程（PID 0），失败时 panic。
- **注意**：此 `init()` 在当前系统中**不会被调用**。

**`run()`**（第294-315行）— 测试用主循环：
- 使用 `kprintln!` 替代原有 `println!` 进行日志输出。
- 仅用于测试/demo，不执行真正的上下文切换。

#### 当前不足与改进建议

1. **全局单锁瓶颈**：所有操作通过 `SCHEDULER.lock()` 串行化
2. **`Vec::remove(0)` 是 O(n)**：应使用 `VecDeque::pop_front()`
3. **`remove_process()` 线性扫描**：遍历所有优先级桶查找进程
4. **无 CPU 亲和性支持**
5. **无 cgroup 集成**
6. **`wake_up()` 未检查 stopped 标志**：与 R98-1 修复不一致
7. **建议处置**：标记为 `#[deprecated]` 或移至 `examples/`

---

### 3.5 enhanced_scheduler.rs（增强型调度器，1535行 — 系统核心）

#### 功能概述

Zero-OS 的核心调度器实现，是系统主线调度路径。通过 `kernel_core` 的回调注册机制深度接入定时器中断、系统调用返回、进程生命周期事件（fork/exit/SIGCONT）。实现了完整的 SMP 感知调度系统：

- Per-CPU 就绪队列（`CpuLocal<Mutex<ReadyQueues>>`）消除全局锁竞争
- 动态优先级反馈调度（优先级分桶 + 时间片 + CPU 密集惩罚 + yield 奖励 + 饥饿提升）
- Work Stealing 负载均衡（本地队列为空时从最忙 CPU 偷取任务）
- 周期性负载均衡（CPU0 每 64 tick 执行一次全局均衡）
- 完整上下文切换（Ring 0 `switch_context` + Ring 3 `enter_usermode`/IRETQ）
- CPU 亲和性（cpuset 感知的有效掩码计算，含 R95-6 防逃逸修复）
- Cgroup CPU 配额执行（`cpu.max` 节流，F.2 集成）
- FPU 惰性保存/恢复（lazy FPU with CR0.TS，含 R69-2 跨 CPU 迁移修复）
- IPI kick 唤醒空闲 CPU（R70-2/R70-7 修复）
- TLS 支持（FS/GS base MSR 保存/恢复 + R100-2 KERNEL_GS_BASE 修复）
- 看门狗心跳（G.1 hung-task 检测）
- Per-CPU 上下文计数器（G.1 Observability）
- R152-3 饥饿进程再桶化修复（deferred re-bucket pattern）
- H.3 KPTI：`activate_memory_space` 支持双 CR3（kernel/user PML4）
- R118-6：`reschedule_now` 直接传递 `user_memory_space` 避免热路径 PROCESS_TABLE 扫描

#### 核心数据结构详解

**类型别名与就绪队列结构**（第76-84行）：
```rust
pub type Pid = ProcessId;
pub type ProcessControlBlock = Arc<Mutex<Process>>;
type ReadyQueues = BTreeMap<Priority, BTreeMap<Pid, ProcessControlBlock>>;
```
- `ReadyQueues` 是双层 BTreeMap：外层按优先级排序（数值越小越优先），内层按 PID 排序。
- **R146-6 说明**：注释已更新承认 PID 排序 ≠ 入队顺序。CPU 密集型任务通过 `decrease_dynamic_priority()` 分离，I/O 密集型任务可能存在低 PID 偏向。

**`ContextShadow` 结构体**（第98-125行）：
```rust
struct ContextShadow {
    buf: UnsafeCell<process::Context>,
}
```
- Per-CPU 上下文影子缓冲区，解决 R70-4 修复的"use-after-unlock"问题。
- `store()` 方法将 `process::Context` 复制到影子缓冲区并返回 `*const ArchContext` 指针。
- **安全性依赖**：Per-CPU 独占访问（通过 `CpuLocal` + 中断禁用保证）。

**编译期 ABI 断言**（第133-137行）：
```rust
const _: () = {
    assert!(size_of::<process::Context>() == size_of::<ArchContext>());
    assert!(align_of::<process::Context>() == align_of::<ArchContext>());
};
```

**全局状态变量**：

| 变量 | 类型 | 行号 | 说明 |
|------|------|------|------|
| `NEXT_CONTEXT_SHADOW` | `CpuLocal<ContextShadow>` | 129 | Per-CPU 上下文影子缓冲区 |
| `BOOTSTRAP_CONTEXT` | `Mutex<ArchContext>` | 140 | 首次调度的哑上下文 |
| `CURRENT_PROCESS` | `CpuLocal<Mutex<Option<Pid>>>` | 146 | Per-CPU 当前进程跟踪 |
| `READY_QUEUE` | `CpuLocal<Mutex<ReadyQueues>>` | 152 | Per-CPU 就绪队列（pub 可见性） |
| `SCHEDULER_STATS` | `Mutex<SchedulerStats>` | 156 | 全局调度统计（lazy_static） |
| `BALANCE_TICKER` | `AtomicU64` | 160 | 负载均衡计时器（仅 CPU0 驱动） |

**常量配置**（第52-64行）：

| 常量 | 值 | 说明 |
|------|-----|------|
| `SCHED_DEBUG` | `false` | 调度调试输出开关（编译期消除） |
| `LOAD_BALANCE_INTERVAL_TICKS` | 64 | 负载均衡间隔（tick 数） |
| `LOAD_IMBALANCE_THRESHOLD` | 1 | 触发迁移的最小队列长度差 |
| `TICK_NS` | 1,000,000 | 每 tick 纳秒数（1ms） |

#### 核心函数逐段分析

**调度决策路径**（三阶段流水线）：

##### `on_clock_tick()`（第986行）— 时钟中断处理

在 `without_interrupts` 中执行，是调度器的"感知"阶段：

1. 获取 Per-CPU 当前进程和就绪队列（第990-992行）
2. 减少当前进程时间片（第1002行），累加 CPU 时间
3. **F.2 Cgroup**：记账 CPU 时间（`cgroup::account_cpu_time`）并执行 cpu.max 配额检查（第1010-1022行），超额时立即抢占
4. 时间片耗尽时降低动态优先级并设置 `need_resched`（第1026-1032行）
5. **R65-19 饥饿防止**：遍历就绪队列，增加 `wait_ticks` 并检查饥饿提升（第1038-1089行）
6. **R152-3 修复**：收集需要再桶化的进程到 `rebucket` 向量，迭代结束后在新锁段中应用，防止 BTreeMap 迭代期间修改键（第1039-1088行）
7. 更新 SCHEDULER_STATS（第1091-1095行）
8. 执行周期性负载均衡（`maybe_balance()`，仅 CPU0）（第1098行）
9. **不执行实际调度**，仅设置 `need_resched` 标志

##### `schedule()`（第1142行）— 调度决策

调度器的"决策"阶段：

1. 清除当前 CPU 的 `need_resched` 标志（第1145行）
2. 获取 Per-CPU 当前进程（第1148行）
3. 调用 `select_next_process()` 从本地队列选择或通过 Work Stealing 获取任务（第1152行）
4. 如果选中新进程，保存旧进程状态为 Ready，更新 `CURRENT_PROCESS` 和 `kernel_core::set_current_pid`（第1158-1188行）
5. 返回 `Option<(Pid, usize)>` 供上下文切换使用
6. **不执行 CR3 切换或上下文切换**

##### `reschedule_now(force)`（第1265行）— 完整上下文切换

调度器的"执行"阶段，是最关键的函数（约250行）：

1. 检查 `need_resched` 标志（force=true 时跳过）（第1268行）
2. 检查可抢占性（`preemptible()`：irq_count 和 preempt_count 必须为 0）（第1274行），不可抢占时设标志延迟
3. 调用 `schedule()` 获取调度决策（第1283行）
4. 保存旧进程的 FS/GS base（R100-2 修复：使用 KERNEL_GS_BASE MSR 0xC0000102）（第1309-1325行）
5. **R69-2 FPU 惰性保存**：清除 CR0.TS → fxsave64 到进程 FPU 缓冲区 → 清除 FPU ownership（第1348-1383行）
6. 通过 `NEXT_CONTEXT_SHADOW` 复制新进程上下文到 Per-CPU 影子缓冲区（第1399-1410行），同时提取 `user_memory_space`（R118-6 修复）和 `watchdog_handle`（G.1）
7. 更新 TSS.rsp0 为新进程内核栈顶，无栈时回退到 `default_kernel_stack_top()`（第1432-1439行）
8. G.1 上下文切换计数器递增（第1445行）
9. G.1 看门狗心跳（第1450-1453行）
10. 激活新进程地址空间（`activate_memory_space`，含 KPTI 双 CR3 支持）（第1455行）
11. 根据目标特权级选择切换方式：
    - Ring 3：`save_context` + 写入 FS/GS MSR + `enter_usermode`（IRETQ，不返回）（第1466-1498行）
    - Ring 0：`assert_kernel_context` + `switch_context`（第1501-1507行）

**进程管理 API**：

| 函数 | 行号 | 作用 | 关键实现 |
|------|------|------|----------|
| `add_process()` | 787 | 添加进程到就绪队列 | `target_cpu_for_new_work()` 选择最轻负载 CPU；`remove_from_all_queues()` 防重复；`kick_cpu()` 唤醒空闲 CPU（R70-7） |
| `remove_process()` | 841 | 移除进程 | `remove_from_all_queues()` 遍历所有 CPU 队列 |
| `resume_stopped()` | 866 | 恢复 SIGCONT 暂停的进程 | R98-1 修复：检查 `stopped` 正交标志；临时设为 Blocked 防止竞态；仅对 Ready/Stopped 状态重新入队 |
| `yield_cpu()` | 1199 | 主动让出 CPU | 设置 Ready 状态，奖励动态优先级（`update_dynamic_priority()`），调用 schedule() |
| `set_current()` / `get_current()` | 952/961 | Per-CPU 当前进程管理 | 通过 CpuLocal 访问 |
| `process_count()` | 1220 | 获取就绪进程总数 | 遍历所有 CPU 队列求和 |

**负载均衡**：

| 函数 | 行号 | 作用 |
|------|------|------|
| `maybe_balance()` | 632 | 每 64 tick 在 CPU0 上触发负载均衡 |
| `balance_queues()` | 644 | 找到最忙和最闲 CPU，迁移一个任务 |
| `migrate_one_ready()` | 684 | R69-3 修复：从源 CPU 弹出一个就绪进程，检查有效亲和性（cpuset 感知）后放入目标 CPU |
| `steal_one()` | 564 | 从最忙 CPU 偷取一个就绪进程到本地队列 |
| `least_loaded_cpu()` | 458 | 查找最轻负载 CPU（排除指定 CPU，尊重亲和性掩码） |
| `target_cpu_for_new_work()` | 486 | 为新任务选择目标 CPU（含亲和性验证） |
| `cpu_pool_size()` | 449 | 可用 CPU 数量（`min(max_cpus, max(num_online_cpus, 1))`） |

**CPU 亲和性（E.5 Cpuset 集成）**：

| 函数 | 行号 | 作用 |
|------|------|------|
| `effective_allowed_cpus()` | 350 | 计算有效 CPU 掩码：online ∩ cpuset ∩ task_affinity；R95-6 修复：空交集回退到 cpuset-only，全空回退到 CPU 0（fail-closed） |
| `cpu_allowed()` | 382 | 检查 CPU 是否在亲和性掩码中（0 表示无限制，R70-3 修复） |
| `cpu_allowed_for_test()` | 393 | 公开的测试包装器 |
| `kick_idle_cpus()` | 420 | 向允许的空闲 CPU 发送 IPI |

**选择逻辑 — `select_next_locked()`**（第216行）：

- R70-3：使用 `effective_allowed_cpus()` 进行 cpuset 感知的亲和性检查
- R98-1：跳过 `stopped == true` 的进程
- F.2：跳过被 cgroup cpu.max 节流的进程（`cpu_quota_is_throttled()`）
- R146-6 说明：同优先级内按 PID 升序选择（非 FIFO，已知偏向）

**初始化与回调注册**（`init()`，第1514行）：

注册到 `kernel_core` 的回调：
- `register_cleanup_notifier` → `Scheduler::remove_process`
- `register_scheduler_add` → `Scheduler::add_process`
- `register_timer_callback` → `Scheduler::on_clock_tick`
- `register_resched_callback` → `Scheduler::reschedule_now`
- `register_resume_callback` → `Scheduler::resume_stopped`

启动日志输出 Per-CPU、Work Stealing、SMP kick、上下文切换等特性状态。

#### 模块间依赖

- `kernel_core::process`：进程管理（get_process, current_pid, activate_memory_space, set_current_pid）
- `kernel_core::cgroup`：CPU 配额计费（account_cpu_time, charge_cpu_quota, cpu_quota_is_throttled）和节流
- `kernel_core::time`：时间戳（current_timestamp_ms）
- `arch`：上下文切换（switch_context, enter_usermode, save_context, assert_kernel_context）、IPI（send_ipi）、TSS（set_kernel_stack, default_kernel_stack_top）
- `cpu_local`：Per-CPU 数据（CpuLocal, current_cpu, current_cpu_id, max_cpus, num_online_cpus, NO_FPU_OWNER）
- `cpuset`：CPU 集合隔离（effective_cpus）
- `trace::counters`：性能计数器（ContextSwitches）
- `trace::watchdog`：看门狗心跳（heartbeat, WatchdogHandle）
- `security`：KPTI 状态查询（is_kpti_enabled）

#### 当前不足与改进建议

1. **同优先级公平性问题**：就绪桶使用 `BTreeMap<Pid, PCB>`，PID 升序而非 FIFO（R146-6 已记录，建议改用 `VecDeque`）
2. **`current_ready_queue()` unsafe 生命周期扩展**（第302行）
3. **ContextShadow 跨类型转换**（第98-137行）：仅有 size/align 断言，未检查字段偏移量
4. **就绪队列可能包含非就绪任务**：影响负载均衡准确性
5. **负载均衡仅在 CPU0 执行**：多核场景下 CPU0 成为均衡瓶颈
6. **`SCHEDULER_STATS` 全局 Mutex**：建议改为 Per-CPU 计数器
7. **`remove_from_all_queues()` 遍历所有 CPU**：O(CPU数 * 优先级桶数)

---

### 3.6 cpuset.rs（CPU 集合隔离，480行）

#### 功能概述

实现 CPU 集合（cpuset）子系统，允许将进程限制在 CPU 子集上运行。支持层级结构，是 Cgroup CPU 控制器和容器 CPU 隔离的基础。

有效 CPU 掩码：`online_mask ∩ cpuset_mask ∩ task_affinity`

#### 核心数据结构

| 类型 | 说明 |
|------|------|
| `CpusetId` | u32 包装，ROOT=0, INVALID=u32::MAX |
| `CpusetNode` | id、cpus_allowed（AtomicU64）、parent（Weak）、task_count（AtomicU32） |
| `CpusetRegistry` | sets（BTreeMap）、next_id（使用 checked_add 防溢出） |
| `CpusetError` | 9 种错误枚举（NotInitialized, NotFound, InvalidParent, InvalidMask, EmptyMask, CannotDestroyRoot, NotEmpty, TooManySets, PermissionDenied） |

#### 核心函数分析

| 函数 | 行号 | 作用 |
|------|------|------|
| `init()` | 168 | 创建 root cpuset；注册 fork/exit 回调（`kernel_core::register_cpuset_task_joined/left`） |
| `cpuset_create()` | 227 | 验证掩码是父集子集、非空；分配 ID（checked_add 防溢出） |
| `cpuset_destroy()` | 259 | R73-3 修复：检查无任务和无子 cpuset |
| `cpuset_set_cpus()` | 299 | R73-3 修复：验证父引用有效性（Weak::upgrade） |
| `effective_cpus()` | 353 | online ∩ cpuset ∩ task_affinity（task_affinity=0 视为无限制） |
| `task_joined()`/`task_left()` | 382/389 | 原子递增/递减任务计数 |
| `cpuset_task_joined_callback()`/`cpuset_task_left_callback()` | 399/407 | E.5 回调包装器，匹配 `fn(u32)` 签名 |

#### 初始化日志

- 使用 `klog_always!` 打印 CPU 数量
- Debug 构建下额外打印 root 掩码值（I.1 klog containment）

#### 当前不足

1. 最大 64 CPU 限制（u64 位掩码）
2. 层级遍历使用递归（`effective_cpus()` 递归调用 `parent.effective_cpus()`，可能栈溢出）
3. Weak 引用可能失效
4. 缺少 CPU 热插拔支持

---

### 3.7 lock_ordering.rs（锁顺序与 Lockdep 框架，837行）

#### 功能概述

定义 9 级锁顺序层级 + Lockdep 运行时检测框架。提供 `LockdepMutex<T>` 包装器，在 debug 构建中执行锁顺序验证。

#### 锁顺序层级

```
Level 1: IRQ Control（最高优先级）
Level 2: Per-CPU Data
Level 3: Scheduler（READY_QUEUE, CURRENT_PROCESS, SCHEDULER_STATS）
Level 4: Memory Management（PT_LOCK, BUDDY_ALLOCATOR, HEAP_ALLOCATOR）
Level 5: Process/Thread（PROCESS_TABLE, Process::inner）
Level 6: VFS
Level 7: IPC（ENDPOINT_REGISTRY, Pipe, Futex）
Level 8: Device/Driver（VGA_BUFFER, SERIAL_PORT）
Level 9: Audit/Security（AUDIT_RING, RNG_STATE, KPTR_KEY）
```

#### 核心组件

**`LockClassKey`**（第247行）：使用 `&'static str` 标识锁类，支持 const 构造。

**`LockdepMutex<T>`**（第293行）：
- debug 构建：`lock()` 和 `try_lock()` 在 `without_interrupts` 中执行 check → acquire → record 序列（R71-3 IRQ 安全修复）
- release 构建：零开销编译为普通 `spin::Mutex`
- `LockdepMutexGuard` 的 `Drop` 实现同样在 `without_interrupts` 中更新跟踪栈

**Per-CPU 锁栈**（`lock_tracking` 模块，第471行）：
- `LockStack`：固定大小数组（MAX_LOCK_DEPTH=32），使用 `Cell<usize>` 追踪深度
- `validate()`：检查新锁的 level 是否大于栈顶锁的 level（同类递归锁除外）
- `pop()`：验证 LIFO 释放顺序

**依赖图模块**（`dep_graph`，第618行，仅 `lockdep_graph` feature）：
- `DependencyGraph`：使用 `BTreeMap<LockClassKey, BTreeSet<LockClassKey>>` 存储有向边
- `would_create_cycle()`：检查添加边是否形成环（迭代 DFS，防栈溢出）
- `validate_edges()`/`record_edges()`：在锁获取时更新图

#### R186-10 页错误锁规则

`COW_FAULT_LOCK` 已移除。COW 页错误在中断门（IF=0）内只能非阻塞获取 `PT_LOCK`；竞争时返回
`Busy`，经 IRETQ 恢复中断后重试原指令，避免阻塞当前 CPU 并饿死 TLB shootdown IPI。

#### 当前不足

1. Lockdep 未在关键路径中使用（`enhanced_scheduler.rs` 使用原生 `spin::Mutex`）
2. 依赖图全局 Mutex 影响 debug 性能
3. release 构建完全无操作

---

## 4. 模块整体评估

### 4.1 架构设计评估

**优点**：
- Per-CPU 就绪队列消除全局锁竞争
- 回调驱动松耦合（5 个回调注册点）
- 三阶段调度流水线（感知/决策/执行）职责清晰
- cpuset 层级结构支持容器化
- Cgroup cpu.max 集成实现配额节流
- R152-3 deferred re-bucket 模式保证 BTreeMap 迭代安全

**不足**：
- 新旧调度器并存（scheduler.rs 和 process.rs 遗留）
- 同优先级 BTreeMap 违反 FIFO
- 调度器状态通过全局变量管理

### 4.2 安全性评估

**已实施**：
- `assert_kernel_context` 防特权提升
- cpuset 隔离保证（R95-6 防逃逸）
- FPU 惰性保存（R69-2 跨 CPU 迁移安全）
- 可抢占性检查（R69-3）
- R98-1 正交 stopped 标志防丢失唤醒

**残留风险**：
- ContextShadow 跨类型转换（仅 size/align 断言）
- `current_ready_queue()` unsafe 生命周期

### 4.3 性能评估

**优点**：
- Per-CPU + Work Stealing 业界标准
- IPI kick 快速拾取新任务
- SCHED_DEBUG 编译期消除
- R152-3 deferred re-bucket 避免迭代中修改

**瓶颈**：
- `SCHEDULER_STATS` 全局 Mutex
- `remove_from_all_queues` O(CPU数)
- `select_next_locked` O(n) 遍历

### 4.4 改进路线图

**P0（正确性/安全性）**：
1. 清理 process.rs 遗留文件
2. 消除 `current_ready_queue()` unsafe 生命周期扩展

**P1（性能/可维护性）**：
1. SCHEDULER_STATS 改为 Per-CPU 计数器
2. debug 配置中在调度器热路径使用 LockdepMutex
3. 统一 Context 类型（消除跨类型转换）
4. 同优先级桶改用 VecDeque 实现 FIFO

**P2（可选优化）**：
1. 支持 >64 CPU 位图
2. 自适应负载均衡间隔
3. 分布式负载均衡（非仅 CPU0）
4. 标记 scheduler.rs deprecated
