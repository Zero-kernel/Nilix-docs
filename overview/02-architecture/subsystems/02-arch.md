# arch 模块 — x86_64 硬件抽象层深度分析

> 路径: `kernel/arch/`
> 语言: Rust (no_std, nightly, `abi_x86_interrupt` feature gate)
> 依赖: `kernel_core`, `drivers`, `cpu_local`, `mm`, `tlb_ops`, `trace`, `klog`, `x86_64` v0.15, `spin` v0.9, `lazy_static` v1.4
> 分析日期: 2026-05-13

---

## 1. 模块概述

`arch` 模块是 Zero-OS 内核与 x86_64 硬件之间的唯一抽象层，封装了所有 CPU 特权级管理、
中断/异常处理、SMP 多核启动、系统调用入口、定时器校准、KPTI 双页表切换以及 CPU 安全加固功能。

### 1.1 文件结构

| 文件 | 行数 | 职责 |
|------|------|------|
| `Cargo.toml` | 19 | crate 依赖声明 |
| `lib.rs` | 64 | 模块入口，re-export 公共 API |
| `gdt.rs` | 409 | Per-CPU GDT/TSS 初始化与管理，IST 栈 + 保护页 |
| `interrupts.rs` | ~1773 | IDT、异常/IRQ/IPI 处理器、8259 PIC、用户异常统一处理 |
| `context_switch.rs` | ~652 | 进程上下文保存/恢复/切换，KPTI 用户态入口 |
| `syscall.rs` | ~1111 | SYSCALL/SYSRET MSR 配置，KPTI 双 CR3 切换，系统调用入口汇编 |
| `apic.rs` | ~1111 | LAPIC/I/O APIC 硬件操作，HPET/PIT 定时器校准 |
| `smp.rs` | ~1516 | AP 启动 (INIT-SIPI-SIPI)，ACPI MADT/RSDP 解析 |
| `hpet.rs` | 458 | HPET 高精度定时器初始化与读取 |
| `ipi.rs` | 411 | 处理器间中断 (IPI) 基础设施与发送 |
| `cpu_protection.rs` | 374 | SMEP/SMAP/UMIP 加固，虚拟化检测 |
| `invpcid.rs` | 43 | INVPCID 指令封装 (re-export tlb_ops) |

### 1.2 初始化顺序

```
arch::init()
  -> gdt::init()                    // BSP GDT + TSS (含 DF IST + NMI IST)
  -> context_switch::init_fpu()     // CR0/CR4 FPU/SSE 位
  -> syscall::register_frame_callback()  // syscall 帧回调注册到 kernel_core
```

后续由内核主流程调用:
```
cpu_protection::enable_protections()  // SMEP/SMAP/UMIP
cpu_protection::require_smap_support() // SMAP 硬性要求检查
hpet::init()                          // HPET 定时器
apic::init()                          // LAPIC 初始化 (BSP)
interrupts::init()                    // IDT 加载 + 8259 PIC 初始化
syscall::init_syscall_msr()           // SYSCALL MSR
syscall::init_syscall_percpu(0)       // BSP per-CPU GS base + SWAPGS
gdt::install_bsp_ist_guard_page()     // R149-I5: BSP DF IST 保护页
gdt::install_bsp_nmi_guard_page()     // R148-I5: BSP NMI IST 保护页
ipi::init()                           // IPI 子系统 (TLB shootdown 回调注册)
smp::start_aps()                      // AP 启动
```

---

## 2. 逐文件分析

### 2.1 Cargo.toml

**依赖分析**:

| 依赖 | 类型 | 用途 |
|------|------|------|
| `kernel_core` | 内部 (path) | 系统调用分发、进程管理、调度器钩子 |
| `drivers` | 内部 (path) | `println!` 宏 (串口/VGA 输出) |
| `cpu_local` | 内部 (path) | Per-CPU 数据结构、CPU ID 管理 |
| `mm` | 内部 (path) | 页表操作、TLB shootdown、MMIO 映射 |
| `tlb_ops` | 内部 (path) | INVPCID 指令封装 |
| `trace` | 内部 (path) | 性能追踪、看门狗、PC 采样分析器 |
| `klog` | 内部 (path) | 结构化内核日志宏 |
| `x86_64` | 外部 v0.15 | 页表类型、段寄存器、中断帧 |
| `spin` | 外部 v0.9 | 自旋锁、Once 初始化 |
| `lazy_static` | 外部 v1.4 | 静态延迟初始化 (spin_no_std feature) |

### 2.2 lib.rs — 模块入口

**功能**: 声明所有子模块并 re-export 公共符号，提供统一的 `arch::init()` 入口。

**Feature Gate**: `#![feature(abi_x86_interrupt)]` — 启用 x86 中断调用约定。

**Re-export 策略**:
- `context_switch`: `Context`, `FxSaveArea`, `switch_context`, `enter_usermode`, `jump_to_usermode`, `save_context`, `restore_context`, `validate_kernel_context`, `assert_kernel_context`, `init_fpu`, `USER_CODE_SELECTOR`, `USER_DATA_SELECTOR`
- `gdt`: `Selectors`, `selectors`, `set_kernel_stack`, `get_kernel_stack`, `set_ist_stack`, `default_kernel_stack_top`, `init_for_ap`, `KERNEL_STACK_SIZE`, `DOUBLE_FAULT_STACK_SIZE`, `DOUBLE_FAULT_IST_INDEX`
- `syscall`: `SyscallFrame`, `init_syscall_msr`, `with_current_syscall_frame`, `register_frame_callback`, `arch_set_kpti_cr3s`, `is_initialized` (as `syscall_initialized`)
- `cpu_protection`: `CpuProtectionStatus`, `check_cpu_features`, `enable_protections`, `require_smap_support`, `HypervisorType`, `detect_hypervisor`, `hypervisor_present`, `is_software_emulated`, `is_virtualized`
- `cpu_local` (外部 crate): `current_cpu_id`, `max_cpus`, `CpuLocal`, `PerCpuData`, `RawTaskPtr`, `PER_CPU_DATA`, `init_bsp`, `num_online_cpus`, `register_cpu_id`, `current_cpu`
- `smp`: `ap_rust_entry`, `online_cpus`, `set_rsdp_address`, `smp_initialized`, `start_aps`
- `invpcid`: 全部 INVPCID 函数

**`init()` 函数**:
```rust
pub fn init() {
    gdt::init();
    context_switch::init_fpu();
    syscall::register_frame_callback();
    klog_always!("Arch module initialized (FPU/SIMD enabled)");
}
```

### 2.3 gdt.rs — 全局描述符表与任务状态段

**功能**: 管理 x86_64 分段机制，提供特权级切换所需的 TSS，支持 Per-CPU GDT/TSS (R70-3)，为 Double Fault 和 NMI 提供独立 IST 栈及保护页。

#### 2.3.1 核心常量

| 常量 | 值 | 说明 |
|------|-----|------|
| `MAX_CPUS` | 64 | 最大支持 CPU 数 |
| `DOUBLE_FAULT_IST_INDEX` | 0 | 双重错误使用的 IST 索引 |
| `NMI_IST_INDEX` | 1 | R148-I5: NMI 使用的 IST 索引 |
| `KERNEL_STACK_SIZE` | 65536 (64 KB) | 内核栈大小 |
| `DOUBLE_FAULT_STACK_SIZE` | 32768 (32 KB) | 双重错误 IST 栈大小 |
| `NMI_STACK_SIZE` | 32768 (32 KB) | R148-I5: NMI IST 栈大小 |

#### 2.3.2 核心数据结构

**`AlignedStack<const SIZE: usize>`**: `#[repr(C, align(4096))]` 页对齐栈结构。R148-4 修复将对齐从 16 字节提升到 4096 字节，使 IST 栈底页可以安全 unmap 作为保护页。

**`Selectors`**: 段选择子集合:
| 字段 | 选择子值 | 用途 |
|------|---------|------|
| `kernel_code` | 0x08 | 内核代码段 (Ring 0) |
| `kernel_data` | 0x10 | 内核数据段 (Ring 0) |
| `user_data` | 0x18 | 用户数据段 (Ring 3) |
| `user_code` | 0x20 | 用户代码段 (Ring 3) |
| `tss` | 动态 | TSS 段选择子 |

**Per-CPU 存储**:
- `PER_CPU_TSS: [TaskStateSegment; MAX_CPUS]` — 每 CPU 独立 TSS
- `PER_CPU_GDT: [Option<(GlobalDescriptorTable, Selectors)>; MAX_CPUS]` — 每 CPU 独立 GDT
- `PER_CPU_INIT: [AtomicBool; MAX_CPUS]` — 初始化标志
- `SELECTORS_CACHE: Once<Selectors>` — 全局选择子缓存 (所有 CPU 相同)

**BSP IST 栈**:
- `BSP_KERNEL_STACK` — BSP 内核特权栈 (64 KB)
- `BSP_DOUBLE_FAULT_STACK` — BSP 双重错误 IST 栈 (32 KB)
- `BSP_NMI_STACK` — R148-I5: BSP NMI IST 栈 (32 KB)

**AP IST 栈**:
- `AP_DOUBLE_FAULT_STACKS: [AlignedStack; MAX_CPUS]` — R145-6: 每 AP 独立 DF IST 栈
- `AP_NMI_STACKS: [AlignedStack; MAX_CPUS]` — R148-I5: 每 AP 独立 NMI IST 栈

#### 2.3.3 核心函数

**`init_per_cpu_gdt(cpu_id, kernel_stack_top, df_stack_top, nmi_stack_top)`** (unsafe):
- 初始化指定 CPU 的 TSS (设置 RSP0、IST[0] = DF 栈、IST[1] = NMI 栈)
- 创建 GDT: 内核代码/数据段 + 用户数据/代码段 + TSS 描述符
- 注意: 用户数据段在用户代码段之前添加 (SYSRET 要求)
- 存储到 `PER_CPU_GDT[cpu_id]`，设置 `PER_CPU_INIT` 标志

**`load_per_cpu_gdt(cpu_id)`** (unsafe):
- 加载 GDT (`gdt.load()`)
- 设置段寄存器: CS = kernel_code, DS = SS = kernel_data
- 加载 TSS (`load_tss`)

**`init()`**: BSP 初始化，使用静态分配的 `BSP_KERNEL_STACK`、`BSP_DOUBLE_FAULT_STACK` 和 `BSP_NMI_STACK`。

**`init_for_ap(cpu_id, kernel_stack_top)`**: AP 初始化 (R70-3)。
- 断言 `cpu_id != 0` (BSP 不应调用)
- R145-6 修复: AP 使用 `AP_DOUBLE_FAULT_STACKS[cpu_id]` 独立 DF IST 栈
- R148-4 修复: unmap DF IST 栈底页作为保护页 (溢出触发 #PF 而非静默损坏)
- R148-I5 修复: AP 使用 `AP_NMI_STACKS[cpu_id]` 独立 NMI IST 栈 + 保护页

**`install_bsp_ist_guard_page()`** (R149-I5):
- 在 mm::page_table::init() 完成后调用
- unmap BSP DF IST 栈底页作为保护页

**`install_bsp_nmi_guard_page()`** (R148-I5):
- 在 mm::page_table::init() 完成后调用
- unmap BSP NMI IST 栈底页作为保护页

**`set_kernel_stack(stack_top)`**: 更新当前 CPU 的 TSS RSP0，用于进程切换前设置返回栈。

**`set_ist_stack(index, stack_top)`**: 更新当前 CPU 指定 IST 栈顶。

**`default_kernel_stack_top()`**: 返回当前 CPU 的默认内核栈顶 (R70-3: 不再仅返回 BSP 栈)。

#### 2.3.4 与旧版差异

| 变更 | 旧版 | 当前版本 |
|------|------|----------|
| IST 栈 | 仅 DF (IST[0]) | DF (IST[0]) + NMI (IST[1]) (R148-I5) |
| AP DF IST 栈 | 复用内核栈 | 独立 per-AP 栈 (R145-6) |
| 栈对齐 | align(16) | align(4096) 页对齐 (R148-4) |
| IST 保护页 | 无 | BSP + AP 均设置底页 guard page (R148-4, R149-I5) |
| `init_per_cpu_gdt` 参数 | 3 个 | 4 个 (增加 nmi_stack_top) |

### 2.4 context_switch.rs — 上下文切换与用户态入口

**功能**: 实现进程上下文的保存/恢复/切换，以及从内核态进入用户态的 IRETQ 路径 (含 KPTI CR3 切换)。

#### 2.4.1 核心常量

| 常量 | 值 | 说明 |
|------|-----|------|
| `USER_CODE_SELECTOR` | `0x23` | 用户代码段 (GDT index 4, RPL=3) |
| `USER_DATA_SELECTOR` | `0x1B` | 用户数据段 (GDT index 3, RPL=3) |
| `FXSAVE_OFFSET` | `0xC0` (192) | FPU 保存区在 Context 中的偏移 |

#### 2.4.2 核心数据结构

**`FxSaveArea`** (`#[repr(C, align(64))]`):
- 512 字节 FXSAVE/FXRSTOR 缓冲区
- 64 字节对齐 (FXSAVE 要求最低 16 字节，64 字节避免缓存行分裂)
- `Default`: 设置 FCW = 0x037F (双精度、所有异常屏蔽)，MXCSR = 0x1F80

**`Context`** (`#[repr(C, align(64))]`):
```
偏移 0x00: rax, rbx, rcx, rdx, rsi, rdi, rbp, rsp
偏移 0x40: r8-r15
偏移 0x80: rip, rflags
偏移 0x90: cs, ss
偏移 0xA0: _padding (4 x u64，对齐到 64 字节边界)
偏移 0xC0: fx (FxSaveArea, 512 字节)
```
- 注意: 不包含 cr3/fs_base/gs_base (页表切换由调度器单独处理)
- `new()`: 默认 RFLAGS = 0x202 (IF=1), CS = 0x08, SS = 0x10
- `init_for_process(entry, stack_top)`: 内核线程初始化
- `init_for_user_process(entry, stack_top)`: 用户线程初始化 (CS=0x23, SS=0x1B)

#### 2.4.3 核心函数

**`switch_context(old: *mut Context, new: *const Context)`** (`#[unsafe(naked)]`):

这是最关键的上下文切换函数，完全用内联汇编实现:

1. **R67-10 修复**: `pushfq; pop [rdi+0x88]` 原子保存 RFLAGS 在 CLI 之前
2. **CLI**: 禁用中断保护切换原子性
3. **R69-2 惰性 FPU**: 不在此处执行 FXSAVE (由 #NM 处理器按需保存)
4. **寄存器保存**: 保存 callee-saved (RBX, RBP, R12-R15) + RSP + RCX/RDX
5. **Z-5 修复**: RDI/RSI 按 caller-saved 处理，切换后清零 (不保留)
6. **RIP 保存**: `mov rax, [rsp]` 从栈顶取返回地址
7. **R102-3 修复**: 清除 per-CPU syscall 状态 (`frame_ptr = 0`, `syscall_active = 0`)，通过 GS 相对寻址访问，偏移量从 `syscall.rs` 导入 (R103-3 单一来源)
8. **R69-2 惰性 FPU**: 设置 CR0.TS 位触发下次 FPU 使用时的 #NM 异常
9. **恢复阶段**: 从新 Context 恢复所有寄存器，恢复 RFLAGS (含 IF 位)

**`enter_usermode(ctx: *const Context) -> !`** (`#[unsafe(naked)]`):

通过 IRETQ 进入用户态 (含 H.3 KPTI CR3 切换):

1. **Y-6 规范地址验证**: 检查 RIP/RSP 是否为规范地址 (`shl 16; sar 16; cmp`)
2. **高半区检查**: `bt rip/rsp, 47` 检测内核地址，拒绝跳转到内核空间
3. **RFLAGS 净化**: 清除 IOPL/NT/RF 位，强制设置 IF
4. **FXRSTOR64**: 恢复用户 FPU 状态
5. **通用寄存器恢复**: 恢复全部 GPR
6. **IRETQ 栈帧构建**: 按 SS, RSP, RFLAGS, CS, RIP 顺序压栈
7. **段选择子强制**: 忽略参数中的选择子，强制使用 `USER_CODE_SELECTOR` / `USER_DATA_SELECTOR`
8. **H.3 KPTI CR3 切换** (R118-2): 在 IRETQ 前通过 `gs:[percpu_kpti_user_cr3]` 读取用户 CR3 并加载，若 user_cr3 == kernel_cr3 (无 KPTI) 则跳过。使用 `kpti_tmp` 临时保存 RDX
9. **SWAPGS**: 在 IRETQ 前执行 SWAPGS 切换到用户 GS
10. **失败路径**: 非规范地址触发 UD2 (未定义指令异常)

**`jump_to_usermode(entry_point, user_stack) -> !`**: 便捷接口，构造 Context 后调用 `enter_usermode`。

**`save_context(ctx: &mut Context)` / `restore_context(ctx: &Context)`**:
- 使用 `pushfq; cli` 保护 FPU 保存/恢复的原子性 (R67-10)
- 保存/恢复所有 GPR + FPU 状态
- `popfq` 恢复原 IF 状态

**`validate_kernel_context(ctx: *const Context) -> bool`** (R65-16):
- 验证 CS RPL 位为 0 (内核态)
- 验证 SS RPL 位为 0
- 防止通过伪造 Context 的段选择子实现特权提升

**`init_fpu()`**:
- 设置 CR0: 清除 EM (模拟), 清除 TS (任务切换), 设置 MP (监控协处理器)
- 设置 CR4: 启用 OSFXSR (SSE) 和 OSXMMEXCPT (SIMD 异常)

#### 2.4.4 与旧版差异

| 变更 | 旧版 | 当前版本 |
|------|------|----------|
| KPTI CR3 切换 | 无 | `enter_usermode` 中读取 gs:[kpti_user_cr3] 加载用户 CR3 (H.3/R118-2) |
| GS 偏移常量 | 无导入 | 从 `crate::syscall` 导入 (R103-3) |
| `jump_to_usermode` | 不存在 | 新增便捷接口 |

### 2.5 interrupts.rs — 中断描述符表与处理器

**功能**: 定义完整的 IDT，实现 20 个异常处理器、3 个硬件 IRQ 处理器、4 个 IPI 处理器 (含 Panic + Spurious)，统一的用户异常处理，以及 8259 PIC 初始化。

#### 2.5.1 Per-CPU FPU 保存机制

**数据结构**:
- `IRQ_FPU_AREAS: CpuLocal<IrqFpuSaveArea>` — Per-CPU 单缓冲区 FPU 保存区 (512 字节)
- `IRQ_FPU_DEPTH: CpuLocal<AtomicU32>` — Per-CPU 中断嵌套深度计数器
- `IRQ_FPU_TS_WAS_SET: CpuLocal<AtomicBool>` — Per-CPU 惰性 FPU CR0.TS 状态保存

**`force_init_irq_cpu_locals()`** (R151-5):
- 在 BSP/AP init 阶段、中断启用之前强制初始化 IRQ 路径 CpuLocal 静态变量
- 防止首次 IRQ 触发 `irq_save_fpu()` 时 `Once::call_once()` 尝试堆分配导致死锁

**`irq_save_fpu()`**:
- R68-7 修复: 使用 `fetch_update` + `checked_add` 防止深度溢出攻击
- 仅最外层 (depth == 0) 执行实际 FXSAVE64
- 嵌套中断 (depth > 0) 直接返回，不重复保存
- R69-2: 保存并清除 CR0.TS 位，防止 FXSAVE 触发 #NM 递归

**`irq_restore_fpu()`**:
- R68-7 修复: 使用 `checked_sub` 防止深度下溢
- 仅最外层恢复时执行 FXRSTOR64
- 恢复 CR0.TS 位到中断前状态

#### 2.5.2 用户异常统一处理 (R107-1)

**`handle_user_exception(stack_frame, exit_code) -> bool`**:
- 检查 CS RPL 判断异常来源 (用户态/内核态)
- 用户态异常: 通过 `terminate_self_and_halt()` (R117-1) 终止进程，不返回
- 内核态异常: 返回 false，由调用者 panic
- 防止用户可触发的内核 panic

**退出码映射** (POSIX 128 + signal):
| 信号 | 编号 | 退出码 | 异常 |
|------|------|--------|------|
| SIGILL | 4 | 132 | #UD |
| SIGBUS | 7 | 135 | #AC |
| SIGFPE | 8 | 136 | #DE, #OF, #MF, #XM |
| SIGSEGV | 11 | 139 | #BR, #GP, #TS, #NP, #SS, #PF |

**`check_pending_kill_on_exception_return(stack_frame)`** (R117-4):
- 在 #DB, #BP, #NM 等非致命异常返回用户态前检查 pending_kill 标志
- 确保跨 CPU 退出请求在不经过 syscall-return 或 timer IRQ 路径的情况下也能被执行

#### 2.5.3 SMAP 辅助

**`clac_if_smap()`**:
- R65-17 修复: 先执行 CLD (清除方向标志)
- V-4 SMP 修复: 直接读取 CR4 判断 SMAP 是否启用 (而非缓存标志)
- 若 SMAP 启用则执行 CLAC

#### 2.5.4 中断统计

**`AtomicInterruptStats`**: 无锁中断计数器
- 字段: `breakpoint`, `page_fault`, `double_fault`, `general_protection_fault`, `invalid_opcode`, `divide_error`, `overflow`, `bound_range_exceeded`, `invalid_tss`, `segment_not_present`, `stack_segment_fault`, `alignment_check`, `machine_check`, `simd_floating_point`, `virtualization`, `timer`, `keyboard`
- `InterruptStatsSnapshot`: 快照结构用于读取

#### 2.5.5 IDT 配置

通过 `lazy_static!` 定义静态 IDT:

| 向量 | 异常/中断 | IST | 说明 |
|------|----------|-----|------|
| 0 | Divide Error | - | 除零异常 (R107-1: 用户态终止) |
| 1 | Debug | - | 调试异常 (R117-4: pending_kill 检查) |
| 2 | NMI | IST[1] | R148-I5: 使用独立 NMI IST 栈 |
| 3 | Breakpoint | - | 断点 (R117-4: pending_kill 检查) |
| 4 | Overflow | - | 溢出 (R107-1: 用户态终止) |
| 5 | Bound Range | - | 边界检查 (R107-1: 用户态终止) |
| 6 | Invalid Opcode | - | 无效操作码 (R107-1: 用户态终止, R107-4: KASLR 保护) |
| 7 | Device Not Available | - | #NM 惰性 FPU (R69-2, R154-4 try_lock, R155-1 CR0.TS 恢复) |
| 8 | Double Fault | IST[0] | 双重错误 (独立栈) |
| 10 | Invalid TSS | - | R151-4: 用户态终止 |
| 11 | Segment Not Present | - | R151-4: 用户态终止 |
| 12 | Stack Segment | - | R151-4: 用户态终止 |
| 13 | General Protection | - | 通用保护错误 (R107-1: 用户态终止) |
| 14 | Page Fault | - | 页错误 (COW + usercopy + R117-1 terminate_self_and_halt) |
| 16 | x87 FP Exception | - | R107-1: 用户态终止 |
| 17 | Alignment Check | - | R107-1: 用户态终止 |
| 18 | Machine Check | - | 机器检查 (不返回) |
| 19 | SIMD FP Exception | - | R107-1: 用户态终止 |
| 20 | Virtualization | - | 虚拟化异常 |
| 32 | Timer | - | PIT/LAPIC 定时器 |
| 33 | Keyboard | - | PS/2 键盘 |
| 36 | Serial COM1 | - | 串口中断 |
| 0xFB | IPI Reschedule | - | 重调度 IPI (R155-11: FPU 保存/恢复) |
| 0xFD | IPI Panic | - | R120-3: Panic 广播 (CLI + HLT 循环) |
| 0xFE | IPI TLB Shootdown | - | TLB 刷新 IPI (R155-11: FPU 保存/恢复) |
| 0xFF | Spurious | - | R120-3: LAPIC SIVR 伪中断 (不发送 EOI) |

#### 2.5.6 关键异常处理器

**`device_not_available_handler`** (#NM, R69-2 + R154-4 + R155-1):
- 惰性 FPU 恢复: 清除 CR0.TS 位，启用 FPU 访问
- 禁用中断保护 FPU 所有权转移
- R154-4: 使用 `try_lock()` 避免死锁 (若 #NM 中断已持有 Process 锁的代码)
- R155-1: 若任一 try_lock 失败，恢复 CR0.TS 使指令稍后重试 #NM
- 两步转移: 先 FXSAVE 前任所有者 FPU 状态，再 FXRSTOR 当前进程 FPU 状态
- 只有在前任成功保存后才恢复当前进程 (防止前任状态丢失)
- R117-4: 处理完毕后检查 pending_kill

**`page_fault_handler`**:
- 读取 CR2 获取故障地址
- **COW 处理**: 检查 `PROTECTION_VIOLATION | CAUSED_BY_WRITE`，调用 `kernel_core::fork::handle_cow_page_fault()`
- **H-26 usercopy 异常表**: 检查 RIP 是否在 usercopy 异常表中，若是则跳转到修复地址
- **用户态页错误**: 通过 `terminate_self_and_halt()` 终止进程 (R116-1, R117-1)
- **内核态页错误**: panic
- X-8 修复: release 构建中不泄露内核地址
- G.1: 集成 per-CPU 观测计数器 (PageFaults, CowFaults)

**`general_protection_handler`** (R107-1):
- 区分用户态/内核态 (检查 CS RPL)
- 用户态 GPF: 通过 `handle_user_exception()` 终止进程
- 内核态 GPF: panic
- R107-4: debug_assertions 门控串口输出

#### 2.5.7 硬件 IRQ 处理器

**Timer (向量 32)**:
- `irq_enter()` / `irq_exit()`: R69-3 IRQ 上下文跟踪
- BSP: PIT 中断 -> `on_timer_tick()` -> 看门狗轮询 -> PC 采样分析器
- AP: LAPIC 定时器 -> 调度器 tick
- FPU 保存/恢复包裹整个处理过程
- **R116-2 pending_kill 检查**: 返回用户态前检查跨 CPU 退出请求
  - R155-6: 通过 `defer_irq_terminate()` 延迟终止到进程上下文
  - 若延迟成功: 标记 Zombie -> 恢复 FPU -> 切换到 boot CR3 -> force_reschedule -> HLT
  - 若延迟队列满: 重新标记 pending 等待下一 tick
- EOI: BSP 发送 PIC + LAPIC EOI，AP 仅发送 LAPIC EOI
- R155-14: AP 首次定时器调试输出使用有界 UART 等待 (MAX_WAIT=1000)

**Keyboard (向量 33)**:
- 读取端口 0x60 获取扫描码
- 转发到 `drivers::push_scancode()`
- R23-5: 唤醒等待 stdin 输入的进程
- 发送 PIC + LAPIC EOI

**Serial COM1 (向量 36)**:
- FIFO 排空: 循环读取端口 0x3F8 直到 LSR bit 0 为 0
- 转发到 `drivers::keyboard::push_char()`
- 收到输入后唤醒等待进程 + 请求重调度
- 发送 PIC + LAPIC EOI

#### 2.5.8 IPI 处理器

**Reschedule IPI (0xFB)**:
- 设置当前 CPU 的 `need_resched` 标志
- R155-11: FPU 保存/恢复 (防止编译器自动向量化 SSE 损坏 FPU 状态)
- 发送 LAPIC EOI

**TLB Shootdown IPI (0xFE)**:
- 调用 `mm::tlb_shootdown::handle_shootdown_ipi()`
- R155-11: FPU 保存/恢复
- 发送 LAPIC EOI

**Panic IPI (0xFD)** (R120-3):
- 发送 LAPIC EOI
- CLI + HLT 永久停止此 CPU

**Spurious (0xFF)** (R120-3):
- Per Intel SDM: 伪中断不得发送 EOI
- Handler 为空操作

#### 2.5.9 8259 PIC 初始化

**`pic_init()`**:
- ICW1-ICW4 标准初始化序列
- 重映射: IRQ 0-7 -> 向量 32-39, IRQ 8-15 -> 向量 40-47
- 启用 IRQ: IRQ0 (定时器), IRQ1 (键盘), IRQ4 (串口 COM1)
- 其余 IRQ 屏蔽

**串口 UART 初始化** (`serial_init_interrupt`):
- 配置 LCR: 8N1 格式
- 配置 FCR: 启用 FIFO + 清空 RX/TX
- 配置 MCR: OUT2 位启用 PIC 中断传递
- 清空残留接收数据 + IIR
- IER 由 `enable_serial_interrupts()` 在 STI 前单独启用

#### 2.5.10 与旧版差异

| 变更 | 旧版 | 当前版本 |
|------|------|----------|
| NMI IST 栈 | 无 | IST[1] (R148-I5) |
| 用户异常处理 | 部分处理器有 | 统一 `handle_user_exception()` (R107-1) |
| pending_kill | 无 | timer + 非致命异常返回前检查 (R116-2, R117-4) |
| terminate 方式 | `terminate_process()` | `terminate_self_and_halt()` (R117-1) |
| Panic IPI | 无 | 0xFD (R120-3) |
| Spurious handler | 无 | 0xFF (R120-3) |
| #TS/#NP/#SS 用户态 | panic | 终止进程 (R151-4) |
| #NM try_lock | 无 | `try_lock()` 避免死锁 (R154-4) |
| #NM CR0.TS 恢复 | 无 | try_lock 失败时恢复 CR0.TS (R155-1) |
| IPI FPU 保存 | 无 | Reschedule/TLB IPI 增加 FPU 保存/恢复 (R155-11) |
| CpuLocal 强制初始化 | 无 | `force_init_irq_cpu_locals()` (R151-5) |

### 2.6 syscall.rs — 系统调用入口

**功能**: 配置 SYSCALL/SYSRET MSR，实现完整的系统调用入口/退出汇编路径，集成 H.3 KPTI 双 CR3 切换。

#### 2.6.1 MSR 常量

| MSR | 地址 | 用途 |
|-----|------|------|
| `IA32_STAR` | `0xC0000081` | 内核/用户段选择子 |
| `IA32_LSTAR` | `0xC0000082` | 64-bit SYSCALL 入口点 |
| `IA32_SFMASK` | `0xC0000084` | SYSCALL RFLAGS 掩码 |
| `IA32_EFER` | `0xC0000080` | 扩展特性寄存器 |
| `IA32_GS_BASE` | `0xC0000101` | 用户态 GS 基址 |
| `IA32_KERNEL_GS_BASE` | `0xC0000102` | 内核态 GS 基址 |

SFMASK 清除位: IF, TF, DF, AC, IOPL, NT, RF — 防止特权/调试位带入内核。

#### 2.6.2 Per-CPU Syscall 元数据

**`SyscallPerCpu`** (`#[repr(C, align(64))]`):
```rust
pub struct SyscallPerCpu {
    pub scratch_top: u64,       // 偏移 0: 临时栈顶 (预计算)
    pub user_rsp_shadow: u64,   // 偏移 8: 用户 RSP 影子
    pub frame_ptr: u64,         // 偏移 16: 当前 syscall 帧指针
    pub syscall_active: u64,    // 偏移 24: syscall 活跃标志 (R67-11)
    pub kpti_kernel_cr3: u64,   // 偏移 32: H.3 KPTI 内核 CR3
    pub kpti_user_cr3: u64,     // 偏移 40: H.3 KPTI 用户 CR3
    pub kpti_tmp: u64,          // 偏移 48: H.3 KPTI 临时寄存器保存
}
```
- 64 字节对齐确保缓存行隔离
- R103-3 修复: 偏移常量通过 `core::mem::offset_of!` 编译时断言 (含 KPTI 字段)
- 通过 GS 段相对寻址访问 (SWAPGS 后 GS 指向当前 CPU 的结构)

**`arch_set_kpti_cr3s(user_cr3, kernel_cr3)`** (H.3):
- 在上下文切换时由 `activate_memory_space()` 调用
- 写入当前 CPU 的 `kpti_kernel_cr3` 和 `kpti_user_cr3`
- 当 KPTI 禁用 (user_cr3 == kernel_cr3) 时，汇编中的 `cmp/je` 跳过 `mov cr3`

**SWAPGS 状态机不变量** (R102-L7):
- 内核态: `IA32_GS_BASE` = per-CPU 数据, `IA32_KERNEL_GS_BASE` = 用户 GS
- 用户态: `IA32_GS_BASE` = 用户 GS, `IA32_KERNEL_GS_BASE` = per-CPU 数据
- 违反此不变量会导致: 空指针解引用、特权提升、信息泄露

#### 2.6.3 SyscallFrame

**`SyscallFrame`** (`#[repr(C)]`):
- 16 个 64 位寄存器，128 字节
- 布局与汇编保存顺序严格匹配
- 用于 clone/fork 读取调用者寄存器状态

**`with_current_syscall_frame<F, R>(f: F) -> Option<R>`** (R102-6):
- 闭包 API 防止帧引用逃逸 (替代旧的 `get_current_syscall_frame() -> &'static`)
- 内部通过 `SYSCALL_PERCPU[cpu_id].frame_ptr` 获取帧指针

#### 2.6.4 syscall_entry_stub — 系统调用入口汇编

**阶段 1: SMAP 安全 & SWAPGS** (R101-6, R102-5):
- `CLAC`: 显式清除 AC 标志 (SMAP 防御纵深，虽然 SFMASK 已清除)
- `SWAPGS` + `LFENCE`: CVE-2019-1125 推测执行防护
- 保存用户 RSP 到 `gs:[PERCPU_USER_RSP_OFFSET]`

**阶段 2: 切换到临时栈**:
- 从 `gs:[PERCPU_SCRATCH_TOP_OFFSET]` 加载 per-CPU 临时栈顶
- `CLD` 清除方向标志

**阶段 3: 保存用户寄存器**:
- 分配 128 字节帧空间
- 保存全部 16 个 GPR
- 清除 DR7/DR6 防止硬件断点打进内核

**阶段 3.5: 嵌套 syscall 检测** (R67-11):
- `lock cmpxchg gs:[PERCPU_SYSCALL_ACTIVE_OFFSET]`: 原子 CAS 0->1
- 若已为 1 (嵌套)，设置 r15 = 1 标记

**阶段 3.75: H.3 KPTI 入口 CR3 切换**:
- 读取 `gs:[percpu_kpti_kernel_cr3]`
- 与 `gs:[percpu_kpti_user_cr3]` 比较
- 若不同 (KPTI 启用)，`mov cr3, rax` 切换到内核页表
- 若相同 (KPTI 禁用)，跳过 `mov cr3`

**阶段 4: 切换到内核栈**:
- 调用 `syscall_get_kernel_rsp0()` 获取 TSS RSP0
- 在内核栈上分配 FPU 保存区 (512B) + 帧 (128B)
- `rep movsq` 复制帧从临时栈到内核栈
- Z-1 修复: `FXSAVE64` 保存用户 FPU 状态

**阶段 5: 调用分发器**:
- 嵌套 syscall 快速失败: 返回 `-EBUSY` (-16)
- 正常路径: `STI` 启用中断，调用 `syscall_dispatcher_bridge()`
- System V ABI: 7 个参数 (rdi, rsi, rdx, rcx, r8, r9, [stack])
- Z-2 修复: 栈对齐 (`sub rsp, 8` + `push arg5`)

**阶段 6: 恢复与返回**:
- `CLI` 禁用中断
- Z-1 修复: `FXRSTOR64` 恢复用户 FPU 状态
- R67-11: 释放 `syscall_active` 标志 (仅非嵌套路径)
- SYSRET 安全检查: 验证用户 RIP/RSP 为规范地址且在低半区
- R67-9: RFLAGS 掩码清除 IOPL/NT/RF/VM/VIF/VIP，强制设置 IF

**阶段 7: H.3 KPTI 出口 CR3 切换 + SYSRETQ**:
- 保存 RDX 到 `gs:[percpu_kpti_tmp]`
- 读取 `gs:[percpu_kpti_user_cr3]` 并与 kernel_cr3 比较
- 若不同: `mov cr3, rdx` 切换到用户页表
- 恢复 RDX
- `SWAPGS` + `LFENCE` + `SYSRETQ` 返回用户态

**回退路径** (R35-SYSRET-1):
- 非规范/高半区地址: 调用 `syscall_bad_return()` -> `terminate_self_and_halt()` 终止进程 (R117-1)
- 不执行 IRETQ (避免 #GP 在错误 GS 状态下)

**`init_syscall_percpu(cpu_id)`** (R67-8):
- 预计算 scratch 栈顶
- 设置 `IA32_KERNEL_GS_BASE` 指向 `SYSCALL_PERCPU[cpu_id]`
- 设置 `IA32_GS_BASE` = 0 (用户初始值)
- R100-2 修复: 立即执行 SWAPGS 使内核从启动开始运行在 post-SWAPGS 状态

#### 2.6.5 与旧版差异

| 变更 | 旧版 | 当前版本 |
|------|------|----------|
| KPTI | 无 | 入口/出口双 CR3 切换 (H.3) |
| SyscallPerCpu 字段 | 4 个 | 7 个 (增加 kpti_kernel_cr3, kpti_user_cr3, kpti_tmp) |
| 编译时断言 | 4 个 | 7 个 (增加 KPTI 字段偏移) |
| `arch_set_kpti_cr3s()` | 不存在 | 新增 |
| SYSCALL_INITIALIZED | `static mut bool` | `AtomicBool` (R102-L3) |
| 日志输出 | 总是输出 | R102-7: LSTAR/STAR 值仅 debug 构建输出 |

### 2.7 apic.rs — 本地 APIC 与 I/O APIC

**功能**: 封装 LAPIC 和 I/O APIC 的 MMIO 操作，提供定时器校准 (HPET/PIT)、IPI 发送、中断路由功能。

#### 2.7.1 LAPIC 寄存器定义

通过子模块组织寄存器常量:
- `lapic`: ID (0x020), VERSION (0x030), TPR (0x080), EOI (0x0B0), SIVR (0x0F0), ICR_LOW/HIGH (0x300/0x310), LVT_TIMER (0x320), TIMER_INIT/CURRENT/DIVIDE (0x380/0x390/0x3E0) 等
- `sivr_bits`: APIC_ENABLED (bit 8)
- `lvt_bits`: MASKED (bit 16), TIMER_PERIODIC (bit 17)
- `timer_divide`: DIV_16 (0x03) 等
- `icr_delivery`: FIXED (0x00), INIT (0x05), STARTUP (0x06)
- `icr_flags`: LEVEL_ASSERT (bit 14)
- `icr_dest`: NO_SHORTHAND, ALL_EXCLUDING_SELF 等

#### 2.7.2 LAPIC MMIO 访问

**`LAPIC_BASE: AtomicU32`**: 存储 LAPIC 基址 (默认 `0xFEE00000`)。
- **截断风险**: 使用 `AtomicU32` 存储 64 位地址，高 32 位被截断。在 x2APIC 或非标准配置下可能出错。

**`lapic_read(offset) / lapic_write(offset, value)`**: 通过恒等映射的 MMIO 地址进行 volatile 读写。

#### 2.7.3 定时器校准

**`calibrate_lapic_timer()`**: 统一入口 (R70-7: 断言中断已禁用)
- 优先 HPET，回退 PIT

**`calibrate_with_hpet()`** (E.1，首选):
- 使用 HPET 主计数器作为参考时钟 (~14 MHz)
- 设置 LAPIC 定时器初始计数 = 0xFFFFFFFF，等待 ~10ms
- 计算: `per_ms = elapsed_lapic / window_ms`
- 精度优于 PIT 方案

**`calibrate_with_pit()`** (回退):
- 使用 PIT 通道 2 单次模式 (Mode 0)
- PIT 频率 = 1,193,182 Hz
- 等待 PIT 输出引脚翻转 (端口 0x61 bit 5)

**`start_lapic_timer()`**:
- 使用校准或默认的初始计数
- 设置 LVT_TIMER: 向量 32, 周期模式, 未屏蔽
- 分频器: DIV_16

#### 2.7.4 LAPIC 初始化

**`init_lapic()`** (BSP):
1. 硬件使能 LAPIC (IA32_APIC_BASE MSR bit 11)
2. 存储 BSP LAPIC ID
3. 清除错误状态寄存器
4. TPR = 0 (接受所有中断)
5. DFR = flat model, LDR = 1 << 24
6. LINT0 = ExtINT (PIC 直通), LINT1 = MASKED
7. SIVR = 0xFF + APIC_ENABLED

**`init_lapic_for_ap()`** (AP):
- 与 BSP 相同，但 LINT0 屏蔽 (AP 不接收 PIC 中断)
- 自动调用 `start_lapic_timer()` 启动定时器

#### 2.7.5 IPI 操作

**`send_ipi_raw(dest_lapic_id, vector, delivery_mode)`**:
- 等待 ICR 空闲
- 写入 ICR_HIGH (目标 LAPIC ID)
- 写入 ICR_LOW (向量 + 传递模式 + LEVEL_ASSERT + TRIGGER_EDGE)

**`send_init_ipi(dest)` / `send_sipi(dest, vector)`**: INIT-SIPI 序列用于 AP 启动。

**`broadcast_ipi_all_except_self(vector)`**: 使用 ICR shorthand `ALL_EXCLUDING_SELF` 广播。

#### 2.7.6 I/O APIC

**`IOAPIC_BASE: AtomicU32`**: I/O APIC 基址 (默认 `0xFEC00000`)。

**`ioapic_read(reg) / ioapic_write(reg, value)`**: 通过 REGSEL/REGDATA 间接访问。

**`ioapic_route_irq(irq, vector, dest_lapic, level_triggered, active_low)`**: 设置重定向表条目。

**`ioapic_mask_irq(irq) / ioapic_unmask_irq(irq)`**: 屏蔽/取消屏蔽 IRQ。

**注意**: `init_ioapic()` 在 `init()` 中被注释掉，当前未使用 I/O APIC 路由。

#### 2.7.7 与旧版差异

| 变更 | 旧版 | 当前版本 |
|------|------|----------|
| `broadcast_ipi_all_except_self` | 无 (旧版 IPI 逐 CPU) | 使用 ICR shorthand 广播 |
| LDR | 0 | `1 << 24` |
| LINT1 | NMI | MASKED |

### 2.8 smp.rs — 对称多处理启动

**功能**: 实现 AP (Application Processor) 的 INIT-SIPI-SIPI 启动序列，包括 ACPI MADT 解析、实模式跳板代码生成、Per-AP 栈分配。

#### 2.8.1 核心常量

| 常量 | 值 | 说明 |
|------|-----|------|
| `TRAMPOLINE_PHYS` | `0x8000` | 跳板物理地址 (< 1MB) |
| `AP_STACK_SIZE` | 16384 (16 KB) | 每 AP 内核栈大小 |
| `PHYSICAL_MEMORY_OFFSET` | `0xFFFFFFFF80000000` | 高半区偏移 |
| `MAX_PHYS_MAPPED` | `0x100000000` (4 GB) | 直接映射最大物理地址 |
| `MAX_CPUS` | 64 | 最大 CPU 数 |
| `AP_READY_TIMEOUT` | 1,000,000 | AP 就绪超时 (自旋迭代) |

#### 2.8.2 跳板数据结构

**`ApTrampolineData`** (`#[repr(C)]`):
```rust
pub struct ApTrampolineData {
    pub pml4_phys: u64,
    pub stack_top: u64,
    pub entry_point: u64,
    pub cpu_index: u32,
    pub lapic_id: u32,
    pub efer: u64,
    pub cr4: u64,
    pub claim_flag_addr: u64,   // R67-2: 数据认领标志地址
}
```

#### 2.8.3 跳板代码生成

**`generate_trampoline() -> [u8; 0x200]`**:

手工汇编的 AP 启动跳板，布局:

| 偏移 | 大小 | 内容 |
|------|------|------|
| 0x00-0x3F | 64B | 16-bit 实模式: CLI, CLD, 段寄存器清零, LGDT, 启用保护模式, far jmp |
| 0x40-0x5F | 32B | 32-bit 保护模式: 加载数据段, 加载 EDI = 数据地址, jmp 到 0x90 |
| 0x60-0x67 | 8B | GDT 指针 (limit + base) |
| 0x70-0x8F | 32B | GDT: null + 32-bit code + 32-bit data + 64-bit code |
| 0x90-0xBF | 48B | 32-bit 续: 加载 CR4 (PAE), CR3 (PML4), EFER (SCE|LME|NXE), 启用分页, far jmp |
| 0xC0-0xFF | 64B | 64-bit 长模式: 加载 RBX = 数据地址, RSP, 参数 (含 claim_flag_addr -> RCX), call entry_point |
| 0x100-0x137 | 56B | ApTrampolineData 结构 |

#### 2.8.4 AP 启动流程

**`start_aps()`**:
1. 获取 BSP LAPIC ID
2. 通过 `enumerate_cpus()` -> `parse_madt()` 枚举所有 CPU
3. 过滤出 AP 列表
4. `make_trampoline_executable()`: 清除跳板页 NX 位 (兼容 L-5 4KB 页拆分)。R142-5: 加 PT_LOCK 序列化
5. 复制跳板到物理 0x8000 (通过高半区虚拟地址)
6. 对每个 AP:
   - R67-2: 重置 `AP_DATA_CLAIMED` 标志
   - 分配 per-AP 栈 (`alloc_ap_stack`)
   - 填充 `ApTrampolineData`
   - 发送 INIT IPI -> 等待 10ms -> SIPI -> 等待 200us -> SIPI
   - R67-2: 等待 AP 认领数据 (防止数据竞争)；超时则提前终止 SMP 初始化
   - 等待 AP 信号就绪
7. R151-6: 标记 `cpu_local::set_smp_init_done()`
8. `make_trampoline_nonexecutable()`: 恢复 NX 位 (R68-2: 使用跨 CPU TLB shootdown)

**`ap_rust_entry(cpu_index, lapic_id, stack_top, claim_flag_addr)`**:
1. R67-2: 原子设置 `claim_flag_addr` = 1 (Release 语义)
2. R67-3: 验证硬件 LAPIC ID 匹配 (防止 CPU 欺骗)
3. 加载内核 GDT/TSS (`init_for_ap`，含 R145-6/R148-4/R148-I5 IST 栈+保护页)
4. 注册 CPU ID 映射
5. 加载 IDT
6. 初始化 LAPIC (init_lapic_for_ap，含启动 LAPIC timer)
7. 初始化 per-CPU 数据
8. R151-5: 强制初始化 IRQ CpuLocal 静态变量
9. R67-8: 初始化 per-CPU syscall 元数据和 GS base
10. R68-1: 先启用中断，再标记在线 (TLB shootdown 正确性)
11. 注册 TLB shootdown 子系统
12. R70-2: 更新 cpu_local 在线计数
13. 进入 `ap_idle_loop()`

**`ap_idle_loop()`** (R70-1):
- 无竞争空闲循环: CLI -> 检查调度器 -> 检查 `need_resched` -> STI;HLT 原子序列
- x86 保证 STI 在下一条指令生效，因此 pending 中断会立即唤醒 HLT

#### 2.8.5 栈分配

**`alloc_ap_stack()`** (R70-5):
- 从伙伴分配器分配 4 页 (16 KB)
- 验证物理地址 < 4GB (高半区直接映射限制)
- 最多重试 16 次寻找低地址帧
- 清零栈内存 (安全)
- 返回 16 字节对齐的栈顶
- R130-5/R131-8: 物理地址仅在 debug 构建输出

#### 2.8.6 ACPI 解析

- `find_rsdp()`: 优先使用 bootloader 提供的 RSDP 地址，回退到 BIOS 区域扫描 (0xE0000-0x100000)
- `validate_rsdp_at()`: 验证签名 "RSD PTR " + 校验和，支持 ACPI 1.0/2.0
- `find_table_rsdt()` / `find_table_xsdt()`: 在 RSDT/XSDT 中按签名查找表
- `parse_madt()`: 解析 MADT 条目类型 0 (Processor Local APIC)，提取启用的 LAPIC ID
- `phys_slice()`: 通过高半区偏移将物理地址转换为虚拟地址切片

#### 2.8.7 与旧版差异

| 变更 | 旧版 | 当前版本 |
|------|------|----------|
| AP IST 栈 | 复用内核栈 | 独立 DF + NMI IST 栈 + 保护页 (R145-6, R148-I5) |
| CpuLocal 初始化 | 无 | `force_init_irq_cpu_locals()` 在中断启用前调用 (R151-5) |
| PT_LOCK | 不获取 | trampoline NX 操作获取 PT_LOCK (R142-5) |
| SMP 完成标志 | 仅 `SMP_INIT_DONE` | 额外调用 `cpu_local::set_smp_init_done()` (R151-6) |
| 地址泄露 | 输出物理地址 | CR3/RSDP 仅 debug 构建输出 (R131-8) |

### 2.9 hpet.rs — 高精度事件定时器

**功能**: 通过 ACPI 表定位 HPET 硬件，创建 MMIO 映射，启用主计数器，提供高精度时间参考。

#### 2.9.1 核心常量

| 常量 | 值 | 说明 |
|------|-----|------|
| `HPET_DEFAULT_BASE` | `0xFED00000` | 默认 HPET 物理基址 |
| `HPET_MMIO_SIZE` | `0x1000` (4 KB) | HPET 寄存器映射大小 |
| `HPET_VIRT_BASE` | `0xFFFFFFFFfed00000` | 高半区 MMIO 虚拟地址 |

#### 2.9.2 寄存器定义

| 偏移 | 名称 | 说明 |
|------|------|------|
| 0x000 | CAPS_ID | 能力与 ID (周期、定时器数、64-bit 标志) |
| 0x010 | CONFIG | 配置 (使能、传统路由) |
| 0x020 | INT_STATUS | 中断状态 |
| 0x0F0 | COUNTER | 主计数器值 |
| 0x100+N*0x20 | TIMER_CONFIG | 定时器 N 配置 |
| 0x108+N*0x20 | TIMER_COMPARATOR | 定时器 N 比较值 |

#### 2.9.3 HpetInfo 结构

```rust
pub struct HpetInfo {
    pub base_phys: u64,
    pub base_virt: u64,
    pub period_fs: u64,
    pub frequency_hz: u64,
    pub comparator_count: u8,
    pub counter_64bit: bool,
}
```

#### 2.9.4 初始化流程

**`init()`**:
1. 快速路径: 若已初始化返回缓存状态
2. E.1 修复: 获取 `HPET_INIT_LOCK` 防止并发初始化
3. 双重检查锁定模式

**`init_internal()`** (unsafe):
1. 通过 `find_rsdp()` -> `find_table_xsdt/rsdt("HPET")` 定位 HPET ACPI 表
2. 验证表签名、长度、校验和
3. 解析 `HpetTable`: 提取 `GenericAddress` 中的物理基址
4. 验证地址空间为系统内存 (非 I/O 端口)
5. 通过 `mm::map_mmio()` 创建高半区 MMIO 映射
6. 读取 CAPS_ID: 提取周期 (飞秒)、定时器数、64-bit 标志
7. 禁用计数器 -> E.1 修复: 解除所有固件配置的比较器 -> 重置计数器为 0
8. 禁用传统路由 -> 启用主计数器
9. 计算频率: `10^15 / period_fs` (128-bit 算术避免溢出)

#### 2.9.5 公共 API

- `read_main_counter() -> Option<u64>`: 读取主计数器 (处理 32-bit 硬件)
- `is_initialized() -> bool`: 检查初始化状态
- `frequency_hz() -> Option<u64>`: 获取频率
- `info() -> Option<HpetInfo>`: 获取完整信息

### 2.10 ipi.rs — 处理器间中断

**功能**: 定义 IPI 向量分配、类型枚举、发送函数、统计计数器，以及 TLB shootdown 集成。

#### 2.10.1 IPI 向量分配

| 向量 | 类型 | 用途 |
|------|------|------|
| 0xFA | Profile | R120-3: 性能采样 (从 0xFD 移动) |
| 0xFB | Reschedule | 请求目标 CPU 运行调度器 |
| 0xFC | Halt | 停止 CPU (关机/panic) |
| 0xFD | Panic | R120-3: panic 广播 (从 0xFF 移动) |
| 0xFE | TLB Shootdown | 跨 CPU TLB 失效 |

注意: 0xFF 保留给 LAPIC SIVR (伪中断)，不得用于 IPI。

#### 2.10.2 IPI 发送

**`send_ipi(target_cpu, ipi_type)`**:
- R68-6 修复: 检查目标 CPU 在线状态 (`mm::tlb_shootdown::is_cpu_online`)
- 查找目标 LAPIC ID (`cpu_local::lapic_id_for_cpu`)
- 调用 `apic::send_ipi_raw()` 发送

**`send_ipi_target(target, ipi_type)`**:
- 支持 `Cpu(id)`, `AllExceptSelf`, `All`, `Mask(u64)` 四种目标

**`init()`**: 注册 TLB shootdown IPI 发送回调到 mm 层，打破 arch<->mm 循环依赖。

#### 2.10.3 IPI 统计

`IpiStats` 通过 `AtomicU64` 计数器实现无锁统计，分别统计 reschedule/tlb_shootdown/halt/profile/panic 发送次数。

#### 2.10.4 与旧版差异

| 变更 | 旧版 | 当前版本 |
|------|------|----------|
| Panic 向量 | 0xFF | 0xFD (R120-3) |
| Profile 向量 | 0xFD | 0xFA (R120-3) |
| IPI_VECTOR_PANIC IDT 注册 | 未注册 | 已注册 (interrupts.rs) |

### 2.11 cpu_protection.rs — CPU 安全加固

**功能**: 检测并启用 SMEP/SMAP/UMIP CPU 保护特性，检测虚拟化环境。

#### 2.11.1 保护特性

| 特性 | CPUID 位 | CR4 位 | 功能 |
|------|---------|--------|------|
| SMEP | 07H:EBX[7] | bit 20 | 阻止内核执行用户页 |
| SMAP | 07H:EBX[20] | bit 21 | 阻止内核访问用户页 (除非 STAC) |
| UMIP | 07H:ECX[2] | bit 11 | 阻止用户态 SGDT/SIDT/SLDT/SMSW/STR |

#### 2.11.2 启用顺序 (I-1 修复)

1. UMIP — 无内核代码依赖
2. SMEP — 独立于其他特性
3. SMAP — 最后启用，因为内核必须已有 STAC/CLAC 包装

#### 2.11.3 SMAP 硬性要求 (R102-5)

**`require_smap_support()`**:
- 内核无条件使用 CLAC/STAC 指令
- 无 SMAP 支持的 CPU 上这些指令会触发 #UD
- 启动时检查，不满足则 panic 停止引导

#### 2.11.4 虚拟化检测

**`HypervisorType`** 枚举:
- `None`, `QemuTcg`, `Kvm`, `VMware`, `HyperV`, `Xen`, `Unknown`

**检测方法**:
- CPUID leaf 0x01 ECX[31] = hypervisor present bit
- CPUID leaf 0x40000000 EBX:ECX:EDX = 12 字节签名
- R72-4: push/pop 不使用 nomem 选项
- R73-4: ECX 初始化为 0

**`is_software_emulated()`**: 检测 QEMU TCG (软件模拟，IPI 延迟高)。

#### 2.11.5 辅助函数

- `stac()` / `clac()`: SMAP 临时禁用/恢复 (inline asm)
- `cpuid_leaf_7()`: 安全的 CPUID 调用 (保存/恢复 RBX)

### 2.12 invpcid.rs — INVPCID 指令封装

**功能**: 从 `tlb_ops` crate re-export INVPCID 操作，提供向后兼容的 `arch::invpcid` 命名空间。

**Re-export 函数**:
- `invpcid_supported()`: 检测 INVPCID 支持
- `invpcid_address(pcid, addr)`: 失效单个 (PCID, 线性地址) 对
- `invpcid_single_context(pcid)`: 失效指定 PCID 的所有映射
- `invpcid_all_nonglobal()`: 失效所有非全局映射
- `invpcid_all_global()`: 失效所有映射 (含全局)
- `flush_address(addr)` / `flush_all_nonglobal()` / `flush_pcid(pcid)`: 高级封装

---

## 3. 模块整体评估

### 3.1 架构设计

**优点**:
- 清晰的模块化: 每个硬件子系统独立文件，通过 lib.rs 统一导出
- Per-CPU 设计贯穿始终: GDT/TSS (R70-3), syscall 元数据 (R67-8), FPU 保存区, 中断统计
- 安全修复标记系统 (R-xx, Z-xx, L-xx, H-xx 等) 提供了完整的修复追溯
- SWAPGS 状态机有详细文档 (R102-L7)
- 编译时断言验证结构体偏移 (R103-3)，含 KPTI 字段
- IST 栈分离: DF (IST[0]) + NMI (IST[1]) + 保护页
- 统一用户异常处理: `handle_user_exception()` + `check_pending_kill_on_exception_return()`
- KPTI 集成: syscall 入口/出口 + IRETQ 路径双 CR3 切换 (H.3)
- 惰性 FPU try_lock 容错: 锁争用时恢复 CR0.TS 重试 (R154-4/R155-1)

**不足**:
- 高半区映射仅 1GB 但多处代码假设 4GB 直接映射 (`MAX_PHYS_MAPPED = 4GB`)
- 8259 PIC 与 LAPIC 混用增加了不必要的复杂性
- I/O APIC 初始化被注释掉，限制了中断路由灵活性
- `static mut` 在多处使用 (GDT, TSS, IPI handlers, syscall stacks)

### 3.2 安全性

**已有防护**:
- CVE-2019-1125 SWAPGS 推测执行防护 (LFENCE)
- CVE-2014-4699/9322 SYSRET 规范地址验证
- SMAP/SMEP/UMIP 硬件加固 (R102-5 硬性要求)
- KPTI 双页表隔离 (H.3: syscall 入口/出口/IRETQ 路径)
- 嵌套 syscall 检测 (R67-11 lock cmpxchg)
- RFLAGS 净化 (R67-9 清除 IOPL/NT/RF/VM/VIF/VIP)
- 内核上下文验证 (R65-16 RPL 检查)
- DR7/DR6 清除防止硬件断点注入
- 跳板页 NX 恢复 + 跨 CPU TLB shootdown (R68-2)
- LAPIC ID 验证防止 CPU 欺骗 (R67-3)
- Usercopy 异常表 (H-26)
- COW 页错误处理
- NMI 独立 IST 栈 (R148-I5)
- IST 栈保护页 (R148-4, R149-I5)
- 用户异常统一终止 (R107-1: #DE/#OF/#UD/#GP/#BR/#AC/#MF/#XM/#TS/#NP/#SS)
- pending_kill 跨 CPU 退出执行 (R116-2, R117-4)
- terminate_self_and_halt 集中化 (R117-1)
- #NM try_lock 防死锁 + CR0.TS 恢复 (R154-4, R155-1)
- KASLR 保护: release 构建不输出内核地址 (R107-4, R102-7, X-8, R131-8)
- Panic IPI 与 SIVR 向量分离 (R120-3)

**安全缺陷**:
1. **LAPIC 地址硬编码** (中等): `cpu_local::current_cpu_id()` 通过硬编码 `0xFEE00020` 读取 LAPIC ID，应使用专用高半区 MMIO 映射
2. **`static mut` 数据竞争** (中等): 多处 `static mut` 依赖调用者保证安全性
3. **NMI SWAPGS 一致性** (低): NMI 处理器使用独立 IST 栈但未执行 SWAPGS 检查。当前 NMI handler 不访问 GS-based 数据，但若引入 per-CPU 数据访问则需修复

### 3.3 性能

**优化点**:
- 惰性 FPU (CR0.TS) 避免不必要的 FPU 保存/恢复
- Per-CPU 缓存行对齐 (SyscallPerCpu align(64)) 避免伪共享
- HPET 定时器校准提供高精度参考
- AP 空闲循环使用 STI;HLT 原子序列避免竞争
- KPTI: 无 KPTI 时 `cmp/je` 跳过 `mov cr3` (零开销)
- Reschedule/TLB IPI FPU 保存/恢复 (R155-11)

**性能问题**:
1. **FXSAVE/FXRSTOR**: 每次 syscall 和中断都保存/恢复 512 字节 FPU 状态
2. **`AllExceptSelf` 逐 CPU 发送** (ipi.rs): 未使用 `apic::broadcast_ipi_all_except_self` (apic.rs 中已有)
3. **`delay_us` 假设 2GHz**: 在不同频率 CPU 上延迟不准确
4. **PIC 定时器**: BSP 使用 PIT 而非 LAPIC 定时器，精度和灵活性受限

### 3.4 改进路线图

| 优先级 | 改进项 | 说明 |
|--------|--------|------|
| P0 | LAPIC MMIO 映射 | 使用专用高半区映射替代恒等映射硬编码地址 |
| P0 | 高半区映射扩展 | 将直接映射从 1GB 扩展到 4GB 或使用按需映射 |
| P1 | XSAVE 支持 | 替换 FXSAVE 以支持 AVX/AVX-512 状态保存 |
| P1 | I/O APIC 启用 | 取消注释 `init_ioapic()`，迁移到 APIC 中断路由 |
| P1 | 消除 `static mut` | 使用 `UnsafeCell` + 原子操作或 `spin::Once` 替代 |
| P2 | IPI broadcast shorthand | ipi.rs 的 `AllExceptSelf` 使用 apic.rs 的 ICR broadcast |
| P2 | TSC 频率校准 | 使用 HPET 校准 TSC 频率替代硬编码 2GHz |
| P2 | x2APIC 支持 | 支持 MADT 类型 9 条目和 MSR 访问模式 |
| P3 | XSAVE compact 模式 | 使用 XSAVEC 减少保存区大小 |
| P3 | 虚拟化检测缓存 | 缓存 `detect_hypervisor()` 结果 |

---

## 4. 附录: 模块间依赖图

```
arch
  +-- kernel_core (syscall_dispatcher, process, scheduler_hook, SyscallFrame,
  |                fork::handle_cow_page_fault, terminate_self_and_halt,
  |                defer_irq_terminate, take_pending_process_exit,
  |                on_scheduler_tick, on_timer_tick, request_resched_from_irq,
  |                wake_stdin_waiters, force_reschedule, exception_table, usercopy)
  +-- drivers (println!/kprintln! 宏, keyboard::push_char, push_scancode)
  +-- cpu_local (current_cpu_id, CpuLocal, PerCpuData, register_cpu_id,
  |              lapic_id_for_cpu, current_cpu, NO_FPU_OWNER, set_smp_init_done,
  |              mark_cpu_online, init_ap)
  +-- mm (page_table::recursive_pd/pt, buddy_allocator, tlb_shootdown,
  |        map_mmio, flush_current_as_all, FrameAllocator, PHYSICAL_MEMORY_OFFSET)
  +-- tlb_ops (invpcid_* 函数)
  +-- trace (counters::increment_counter, watchdog::poll_watchdogs,
  |           profiler::record_pc_sample)
  +-- klog (klog!, klog_always!, kprintln! 宏)
  +-- x86_64 (PageTable, Cr0, Cr4, VirtAddr, PhysAddr, InterruptStackFrame)
  +-- spin (Mutex, Once)
  +-- lazy_static (IDT 静态初始化)
```
