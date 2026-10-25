# Zero-OS 辅助模块分析：cpu_local 与 tlb_ops

> **模块路径**: `kernel/cpu_local/` 和 `kernel/tlb_ops/`
> **编程语言**: Rust (nightly, `no_std`)
> **目标架构**: x86_64
> **分析日期**: 2026-05-13
> **文档版本**: 基于当前 `main` 分支最新提交

## 1. 模块概述

`cpu_local` 和 `tlb_ops` 是 Zero-OS 内核的两个底层辅助模块，共同构成 SMP（对称多处理）环境下 CPU 本地数据管理与 TLB 一致性维护的基础设施。

- **cpu_local**：提供 Per-CPU 存储抽象，管理每个 CPU 的本地状态（调度元数据、中断计数、FPU 所有权、TLB shootdown 邮箱、KPTI 上下文等），并实现 LAPIC ID 到逻辑 CPU 索引的 O(1) 反向映射。
- **tlb_ops**：提供 x86_64 架构下的 TLB 失效原语封装，支持 INVPCID 指令的四种失效类型，并在硬件不支持时自动回退到 CR3 重载或 INVLPG 指令。

两者的设计遵循"解耦分层"原则：`tlb_ops` 零外部依赖，`cpu_local` 仅依赖 `spin` crate；上层模块（`arch`、`mm`、`sched`、`kernel_core`、`trace`）按需引用它们，不会产生循环依赖。

### 依赖关系图

```
arch ──────┬──→ cpu_local
mm ────────┤──→ cpu_local
sched ─────┤──→ cpu_local
kernel_core┤──→ cpu_local
trace ─────┘──→ cpu_local

arch ──────┬──→ tlb_ops
mm ────────┘──→ tlb_ops
```

## 2. 文件结构

### cpu_local 模块

| 文件 | 行数 | 说明 |
|------|------|------|
| `kernel/cpu_local/Cargo.toml` | 12 | 包配置，依赖 `spin = "0.9"` |
| `kernel/cpu_local/lib.rs` | 821 | 全部实现：Per-CPU 容器、PerCpuData、TLB shootdown 邮箱、LAPIC 映射、KPTI 字段 |

### tlb_ops 模块

| 文件 | 行数 | 说明 |
|------|------|------|
| `kernel/tlb_ops/Cargo.toml` | 9 | 包配置，零外部依赖 |
| `kernel/tlb_ops/lib.rs` | 290 | 全部实现：INVPCID 检测、描述符、四种失效类型、高层封装 |

## 3. tlb_ops 模块详细分析

### 3.1 文件：`kernel/tlb_ops/lib.rs`（290 行）

#### 3.1.1 INVPCID 支持检测（第 36-80 行）

```rust
static INVPCID_CACHED: AtomicBool = AtomicBool::new(false);
static INVPCID_CHECKED: AtomicBool = AtomicBool::new(false);

pub fn invpcid_supported() -> bool { ... }
```

采用"两段式发布"缓存模式：

1. 首次调用时通过 CPUID leaf 7（subleaf 0）查询 `EBX[10]` 位判断 INVPCID 支持。
2. 先以 `Relaxed` 写入 `INVPCID_CACHED`，再以 `Release` 写入 `INVPCID_CHECKED`。
3. 后续调用以 `Acquire` 读取 `INVPCID_CHECKED`，若已检查则以 `Acquire` 读取缓存值。

内存序分析：`Release`-`Acquire` 配对保证了"先写缓存值、再标记已检查"的顺序对其他 CPU 可见。`INVPCID_CACHED` 的写入使用 `Relaxed` 是安全的，因为它总是在 `INVPCID_CHECKED` 的 `Release` 之前执行，而读端在 `Acquire` `INVPCID_CHECKED` 之后才读取 `CACHED`，形成了完整的 happens-before 链。

内联汇编中手动保存/恢复 `rbx`（LLVM 保留寄存器），不使用 `nostack`/`nomem` 选项（R72-4 修复注释），因为 `push`/`pop` 操作会使用栈和内存。

#### 3.1.2 INVPCID 描述符与底层执行（第 82-110 行）

```rust
#[repr(C, packed)]
struct InvpcidDescriptor {
    pcid: u64,    // 12-bit PCID (0-4095)
    address: u64, // 线性地址（仅类型0使用）
}

unsafe fn invpcid(desc: &InvpcidDescriptor, typ: u64) { ... }
```

描述符为 `packed` 布局，恰好 16 字节，符合 Intel SDM 对 INVPCID 操作数的要求。底层 `invpcid()` 函数使用 `nostack, preserves_flags` 选项——INVPCID 指令本身不修改 EFLAGS 也不使用栈，选项使用正确。

#### 3.1.3 四种 INVPCID 封装（第 112-212 行）

| 函数 | INVPCID 类型 | 失效范围 | 典型场景 |
|------|-------------|---------|---------|
| `invpcid_address(pcid, addr)` | 0 | 单个 (PCID, 线性地址) 对 | COW 解析、单页解映射 |
| `invpcid_single_context(pcid)` | 1 | 指定 PCID 的所有非全局映射 | 进程退出、地址空间销毁 |
| `invpcid_all_nonglobal()` | 2 | 所有 PCID 的非全局映射 | 内核页表全局变更 |
| `invpcid_all_global()` | 3 | 所有映射（含全局） | 内核安全转换、CR4.PGE 切换 |

所有函数均标记 `#[inline(always)]` 和 `pub unsafe`，将安全责任交给调用者。文档注释详尽，包含使用场景和安全约束。

#### 3.1.4 高层自适应封装（第 214-273 行）

```rust
pub unsafe fn flush_pcid(pcid: u16) { ... }
pub unsafe fn flush_address(pcid: u16, addr: u64) { ... }
pub unsafe fn flush_all_nonglobal() { ... }
```

这三个函数自动检测 INVPCID 支持并选择最优路径：

| 函数 | INVPCID 可用时 | 回退方案 |
|------|---------------|---------|
| `flush_pcid` | `invpcid_single_context` | CR3 重载（刷新所有非全局项） |
| `flush_address` | `invpcid_address`（当 pcid != 0） | `INVLPG`（刷新当前上下文单地址） |
| `flush_all_nonglobal` | `invpcid_all_nonglobal` | CR3 重载 |

**语义问题**：`flush_pcid()` 的回退路径（CR3 重载）无法精确刷新指定 PCID 的条目。`flush_address()` 使用 `pcid != 0` 作为是否走 INVPCID 的判断条件，将"PCID 禁用"与"PCID=0（内核有效值）"混为一谈。

#### 3.1.5 单元测试（第 275-290 行）

两个测试验证 `InvpcidDescriptor` 的大小（16 字节）和对齐（1 字节，packed），确保内存布局符合硬件要求。

### 3.2 tlb_ops 模块评估

**优点**：
- 零依赖设计，作为 `arch` 和 `mm` 的共同底座避免循环依赖。
- 内联汇编编写规范，选项标注正确，CPUID 检测的内存序使用无误。
- 文档注释详尽，包含 Intel SDM 表格、使用场景和安全约束。

**潜在问题**：
- 回退路径的语义精度不足：在 PCID 启用但 INVPCID 不可用的场景下，回退行为与函数名暗示的语义不完全匹配。
- 缺少 `flush_all_global` 的高层封装（仅有底层 `invpcid_all_global`），上层调用者需要自行处理回退。

## 4. cpu_local 模块详细分析

### 4.1 文件：`kernel/cpu_local/lib.rs`（821 行）

#### 4.1.1 常量与全局映射表（第 30-78 行）

```rust
const MAX_CPUS: usize = 64;
const INVALID_LAPIC_ID: u32 = u32::MAX;
const INVALID_CPU_ID: usize = usize::MAX;
const LAPIC_ID_REVERSE_MAP_SIZE: usize = 256;
pub const NO_FPU_OWNER: usize = usize::MAX;

static LAPIC_ID_MAP: [AtomicU32; MAX_CPUS] = { ... };
static LAPIC_ID_REVERSE_MAP: [AtomicUsize; LAPIC_ID_REVERSE_MAP_SIZE] = { ... };
```

双向映射设计：
- `LAPIC_ID_MAP`：正向映射，CPU 逻辑索引 -> 硬件 LAPIC ID（64 项）。
- `LAPIC_ID_REVERSE_MAP`：反向映射，硬件 LAPIC ID -> CPU 逻辑索引（256 项），用于 O(1) 查找。

256 项的反向表覆盖了 xAPIC 模式下的 8-bit LAPIC ID 空间。若未来支持 x2APIC（32-bit APIC ID），此表需要扩展或改用哈希表。

#### 4.1.2 SMP 完成标志（第 42-56 行）

```rust
static CPU_LOCAL_SMP_DONE: AtomicBool = AtomicBool::new(false);

pub fn set_smp_init_done() {
    CPU_LOCAL_SMP_DONE.store(true, Ordering::Release);
}
```

R151-6 修复：SMP 初始化完成后，`current_cpu_id()` 对未注册的 LAPIC ID 不再静默回退到 CPU 0，而是 panic。这防止了 Per-CPU 槽别名导致的 TLB shootdown 邮箱、IRQ 计数器、FPU 保存区和调度器状态损坏。

#### 4.1.3 TLB Shootdown 队列（第 88-177 行）

```rust
pub const TLB_SHOOTDOWN_QUEUE_LEN: usize = 4;

pub struct TlbShootdownEntry {
    pub generation: AtomicU64,
    pub cr3: AtomicU64,
    pub start: AtomicU64,
    pub len: AtomicU64,
}

pub struct TlbShootdownMailbox {
    pub request_gen: AtomicU64,
    pub ack_gen: AtomicU64,
    pub head: AtomicU64,
    pub tail: AtomicU64,
    pub entries: [TlbShootdownEntry; TLB_SHOOTDOWN_QUEUE_LEN],
}
```

采用 4 槽有界环形缓冲区（FIFO）设计，替代传统的单槽邮箱：

- **请求方**：写入 entry 字段（Relaxed），以 Release 发布 `entry.generation`，再以 Release 更新 `request_gen`（fetch_max 保持单调性）。
- **处理方**：以 Acquire 加载 `entry.generation`，Relaxed 读取字段，以 Release 写入 `ack_gen` 并清除 entry，以 Release 推进 head。
- **等待方**：以 Acquire 加载 `ack_gen` 确认刷新完成。

`TlbShootdownEntry` 标记为 `#[repr(C)]`，手动实现 `Clone`（因为 `AtomicU64` 不实现 `Clone`），clone 时使用 `Relaxed` 加载。

#### 4.1.4 PerCpuData 结构体（第 179-404 行）

```rust
#[repr(C)]
pub struct PerCpuData {
    pub cpu_id: AtomicUsize,
    pub lapic_id: AtomicU32,
    pub preempt_count: AtomicU32,
    pub irq_count: AtomicU32,
    pub fpu_owner: AtomicUsize,
    pub need_resched: AtomicBool,
    _pad: [u8; 3],
    pub current_task: AtomicPtr<()>,
    pub kernel_stack_top: AtomicUsize,
    pub irq_stack_top: AtomicUsize,
    pub syscall_stack_top: AtomicUsize,
    pub rcu_epoch: AtomicU64,
    pub tlb_mailbox: TlbShootdownMailbox,
    // ---- KPTI per-CPU context (H.3) ----
    pub kpti_seq: AtomicU64,
    pub kpti_user_cr3: AtomicU64,
    pub kpti_kernel_cr3: AtomicU64,
    pub kpti_pcid: AtomicU64,
}
```

这是内核的核心 Per-CPU 数据结构，包含：

| 字段 | 类型 | 用途 | 内存序 |
|------|------|------|--------|
| `cpu_id` | `AtomicUsize` | 逻辑 CPU 索引 | Relaxed（初始化后只读） |
| `lapic_id` | `AtomicU32` | 硬件 LAPIC ID | Relaxed（初始化后只读） |
| `preempt_count` | `AtomicU32` | 抢占禁用嵌套计数 | Relaxed（仅本 CPU 访问） |
| `irq_count` | `AtomicU32` | 中断嵌套计数 | Relaxed（仅本 CPU 访问） |
| `fpu_owner` | `AtomicUsize` | 当前 FPU 所有者 PID | Acquire/Release（跨 CPU 可见） |
| `need_resched` | `AtomicBool` | 重调度标志 | Release 写 / AcqRel swap |
| `current_task` | `AtomicPtr<()>` | 当前任务裸指针 | Acquire/Release |
| `kernel_stack_top` | `AtomicUsize` | 内核栈顶 | Relaxed（初始化后只读） |
| `irq_stack_top` | `AtomicUsize` | 中断栈顶（IST1） | Relaxed（初始化后只读） |
| `syscall_stack_top` | `AtomicUsize` | 系统调用栈顶 | Relaxed（初始化后只读） |
| `rcu_epoch` | `AtomicU64` | RCU epoch 计数 | 由 RCU 子系统管理 |
| `tlb_mailbox` | `TlbShootdownMailbox` | TLB shootdown 邮箱 | 见 4.1.3 |
| `kpti_seq` | `AtomicU64` | KPTI seqlock 序列号（偶=无写） | SeqCst |
| `kpti_user_cr3` | `AtomicU64` | KPTI 用户态 CR3 | 受 kpti_seq 保护 |
| `kpti_kernel_cr3` | `AtomicU64` | KPTI 内核态 CR3 | 受 kpti_seq 保护 |
| `kpti_pcid` | `AtomicU64` | KPTI PCID 值 | 受 kpti_seq 保护 |

使用 `RawTaskPtr = *mut ()` 作为任务指针类型，避免了与调度器模块的循环依赖。手动实现 `Send + Sync`（因为所有字段均为原子类型）。

**关键方法分析**：

- `preempt_disable/enable`：使用 `fetch_add/sub` 原子 RMW 操作。`preempt_enable` 包含 `assert!(old > 0)` 防止下溢。
- `preemptible()`：同时检查 `preempt_count == 0` 和 `irq_count == 0`，确保在中断上下文中也不会触发抢占。
- `clear_fpu_owner_if(pid)`：使用 `compare_exchange` 原子地清除 FPU 所有权，正确处理了竞态条件。
- `irq_enter/exit`：使用 `fetch_add/sub`，`irq_exit` 包含 `assert!(old > 0)` 防止下溢。
- `init()`：初始化所有字段，包括将 `fpu_owner` 设为 `NO_FPU_OWNER`。

#### 4.1.5 CpuLocal<T> 泛型容器（第 406-561 行）

```rust
pub struct CpuLocal<T> {
    init: fn() -> T,
    slots: Once<UnsafeCell<Box<[MaybeUninit<T>]>>>,
}
```

核心设计要点：

1. **堆分配**（R91-2 修复）：使用 `Box::new_uninit_slice(MAX_CPUS)` 在堆上分配槽数组，避免了大型 Per-CPU 类型（如 `SampleRing` ~41KB * 64 CPU = ~2.6MB）导致的栈溢出。
2. **一次性全量初始化**：首次访问任意 CPU 槽时，通过 `spin::Once` 一次性分配并初始化全部 MAX_CPUS（64）个槽。
3. **R151-5 `force_init()` 方法**：强制初始化堆分配，避免首次访问在 IRQ 上下文中发生时因堆分配器锁导致死锁。应在 BSP/AP 初始化阶段、中断启用前调用。
4. **`with(f)`**：访问当前 CPU 的槽，包含边界检查（`assert!(id < MAX_CPUS)`）。
5. **`with_cpu(cpu_id, f)`**：跨 CPU 访问，用于 TLB shootdown 等场景。
6. **`get_cpu(cpu_id)`**：返回 `&'static T` 引用，通过 `core::mem::transmute` 延长生命周期。安全性依赖于"底层存储由 static `Once` 持有、堆分配永不释放"的不变量。

#### 4.1.6 current_cpu_id() 函数（第 562-612 行）

```rust
pub fn current_cpu_id() -> usize {
    let apic_id = unsafe {
        let apic_base = 0xFEE0_0020 as *const u32;
        (core::ptr::read_volatile(apic_base) >> 24) as usize
    };
    // O(1) 反向查找
    let cpu_idx = if apic_id < LAPIC_ID_REVERSE_MAP_SIZE {
        LAPIC_ID_REVERSE_MAP[apic_id].load(Ordering::Relaxed)
    } else { INVALID_CPU_ID };
    if cpu_idx < MAX_CPUS { return cpu_idx; }

    // R151-6 FIX: SMP 完成后对未注册 LAPIC ID panic
    if CPU_LOCAL_SMP_DONE.load(Ordering::Acquire) {
        panic!("current_cpu_id: LAPIC ID {} not registered after SMP init complete", apic_id);
    }

    0 // 早期启动回退
}
```

通过 MMIO 读取 `0xFEE00020`（LAPIC ID 寄存器）的高 8 位获取硬件 APIC ID，然后在反向映射表中 O(1) 查找。

R151-6 修复：SMP 初始化完成后（`CPU_LOCAL_SMP_DONE == true`），未注册的 LAPIC ID 不再静默回退到 CPU 0，而是 panic，防止 Per-CPU 槽别名导致的严重 SMP 数据损坏。

**平台假设**：隐含 xAPIC 模式 + 固定 MMIO 映射 + 8-bit APIC ID 的假设。x2APIC（MSR 模式、32-bit ID）需要不同的读取方式。

#### 4.1.7 CPU 注册与初始化（第 614-776 行）

```rust
pub fn register_cpu_id(cpu_id: usize, lapic_id: u32) { ... }
pub fn init_bsp(lapic_id: u32, kernel_stack_top: usize, ...) { ... }
pub fn init_ap(cpu_id: usize, lapic_id: u32, ...) { ... }
```

- `register_cpu_id`：同时填充正向和反向映射表。对 `cpu_id` 有 `assert!` 边界检查，对 `lapic_id >= 256` 的情况静默跳过反向表填充。
- `init_bsp`：初始化 BSP（CPU 0），先注册映射再初始化 PerCpuData。
- `init_ap`：初始化 AP（CPU 1+），包含 `assert!(cpu_id > 0)` 和 `assert!(cpu_id < MAX_CPUS)` 防止误用。

#### 4.1.8 在线 CPU 管理与 FPU 清理（第 778-821 行）

```rust
static ONLINE_CPU_COUNT: AtomicUsize = AtomicUsize::new(1); // BSP 初始计数
pub fn num_online_cpus() -> usize { ... }
pub fn mark_cpu_online() { ... }
pub fn clear_fpu_owner_all_cpus(pid: usize) { ... }
```

- 在线 CPU 计数初始为 1（BSP），每个 AP 调用 `mark_cpu_online()` 递增。使用 `Release`/`Acquire` 保证跨 CPU 可见性。
- `clear_fpu_owner_all_cpus`：进程退出时遍历所有 CPU 槽（0..MAX_CPUS），使用 `get_cpu` + `clear_fpu_owner_if` 原子地清除匹配 PID 的 FPU 所有权，防止 `#NM` 异常处理程序访问已释放的内存。

## 5. 模块协作关系

两个模块在 TLB shootdown 流程中紧密协作：

```
请求方 CPU                          目标 CPU
    |                                   |
    +-- 写入 TlbShootdownMailbox -------+
    |  (cpu_local 提供数据结构)          |
    +-- 发送 IPI ------------------------+
    |                                   +-- IPI 处理程序读取 mailbox
    |                                   +-- 调用 tlb_ops 原语执行刷新
    |                                   |  (invpcid_all_nonglobal / flush_pcid 等)
    |                                   +-- 写入 ack_gen 确认完成
    +-- 轮询 ack_gen 等待确认 ----------+
    |                                   |
```

关键解耦点：
- `tlb_ops` 不依赖 `cpu_local`，不知道"哪个 CPU"或"邮箱"的概念。
- `cpu_local` 不依赖 `tlb_ops`，不知道具体的 TLB 失效指令。
- 上层模块（`mm`、`arch`）负责编排两者：确定目标 CPU -> 写邮箱 -> 发 IPI -> 目标执行刷新 -> 等待 ACK。

### 5.1 关键风险清单

| # | 风险 | 严重度 | 所在模块 | 说明 |
|---|------|--------|---------|------|
| 1 | `CpuLocal::get_cpu()` 潜在 unsoundness | 高 | cpu_local | `&self` 签名允许非静态实例调用，但内部 transmute 假设 `'static` 生命周期 |
| 2 | PCID 回退语义不精确 | 中 | tlb_ops | `flush_pcid` 回退到 CR3 重载无法按指定 PCID 刷新 |
| 3 | LAPIC ID 注册静默失败 | 中 | cpu_local | `lapic_id >= 256` 时反向表不填充，R151-6 修复后会 panic（而非静默别名） |
| 4 | 同一 LAPIC ID 重复注册 | 中 | cpu_local | 反向表会被覆盖，导致先前注册的 CPU 无法被正确识别 |
| 5 | `CpuLocal` 首次初始化可重入风险 | 低 | cpu_local | `spin::Once` 在中断重入时可能自旋死锁；R151-5 `force_init()` 提供了规避手段 |

## 6. 模块整体评估

### 6.1 架构设计

| 维度 | 评价 |
|------|------|
| 模块边界 | 优秀。两个模块职责单一、边界清晰，成功避免了内核常见的循环依赖问题 |
| 分层抽象 | 优秀。`tlb_ops` 提供纯硬件原语，`cpu_local` 提供数据结构与定位，上层编排策略 |
| 可扩展性 | 良好。MAX_CPUS=64 和 256 项反向表覆盖了当前需求，但 x2APIC 支持需要扩展 |
| 代码组织 | 良好。每个模块单文件实现，结构紧凑；cpu_local 的 821 行包含了较多职责 |

### 6.2 安全性

| 维度 | 评价 |
|------|------|
| unsafe 使用 | 合理。内联汇编、MMIO 读取、生命周期 transmute 均有明确的安全不变量说明 |
| 内存序 | 正确。Release/Acquire 配对覆盖了跨 CPU 可见性需求，本地计数器使用 Relaxed |
| 边界检查 | 良好。`CpuLocal::with` 有 assert 检查，`register_cpu_id` 有范围检查 |
| SMP 安全 | R151-6 修复解决了 post-SMP 静默别名问题；R151-5 修复解决了 IRQ 上下文堆死锁 |
| 潜在 unsoundness | `CpuLocal::get_cpu()` 的 `&self` -> `&'static T` transmute 在非静态实例下不安全 |

### 6.3 性能

| 维度 | 评价 |
|------|------|
| 热路径优化 | 良好。INVPCID 检测结果缓存、O(1) LAPIC 反向查找、所有关键函数 inline |
| 已知瓶颈 | `current_cpu_id()` 的 MMIO 读取在 syscall 热路径上可能较慢 |
| 优化空间 | `preempt_count`/`irq_count` 可改用普通整数 + 编译器屏障；CPU ID 可缓存到 GS base |
| 内存效率 | 良好。堆分配避免栈溢出，TLB shootdown 队列深度 4 平衡了吞吐与内存开销 |

### 6.4 KPTI 扩展（H.3）

PerCpuData 新增了四个 KPTI 相关字段（`kpti_seq`、`kpti_user_cr3`、`kpti_kernel_cr3`、`kpti_pcid`），用于内核页表隔离：

- `kpti_seq`：seqlock 序列计数器，偶数表示无写操作，奇数表示写入中。读端通过 seqlock 协议获得一致的 CR3/PCID 快照。
- `kpti_user_cr3` / `kpti_kernel_cr3`：分别存储当前进程的用户态和内核态 CR3 值，供 syscall/中断入口切换。
- `kpti_pcid`：当前进程的 PCID 值，用于 INVPCID 精确刷新。

这些字段均为 `AtomicU64`，初始化为 0（在 `PerCpuData::new()` 中）。

### 6.5 改进建议

#### 正确性（必须修复或文档化）

1. **CpuLocal::get_cpu 签名**：将签名改为 `get_cpu(&'static self, ...)` 或标记为 `unsafe fn`，在类型层面强制"必须是静态实例"的不变量。
2. **LAPIC ID 防御**：`register_cpu_id` 应对 `lapic_id >= 256` 的情况 `assert!` 或返回错误，并增加重复注册检测。
3. **tlb_ops 回退语义**：明确文档化"仅在 INVPCID 可用时启用 PCID"的全局策略。

#### 健壮性

4. **CpuLocal 初始化时序**：`force_init()` 已提供规避手段（R151-5），但应在文档中明确"首次访问必须在分配器就绪且中断关闭阶段"的约束。
5. **CPU 在线状态统一**：将"CPU 在线状态"收敛到单一权威来源（如 bitmap），避免多模块各自维护导致的不一致。

#### 性能

6. **current_cpu_id 优化**：长期考虑将 CPU ID 缓存到 `GS base` 寄存器，消除 MMIO 读取开销。
7. **本地计数器优化**：`preempt_count`/`irq_count` 可改用普通整数 + 编译器屏障。

#### 未来扩展

8. **x2APIC 兼容**：为 `current_cpu_id()` 预留 x2APIC MSR 读取路径。
9. **flush_all_global 高层封装**：补充 `invpcid_all_global` 的自适应高层封装，提供 CR4.PGE toggle 回退路径。
