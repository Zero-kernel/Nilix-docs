# 内核入口 (kernel/src) 模块分析文档

> 分析范围：`kernel/src/`
> 分析日期：2026-05-13
> Zero-OS 版本：main 分支

## 1. 模块概述

`kernel/src/` 是 Zero-OS 微内核的顶层入口模块，承担**启动编排层（orchestration layer）**的角色。该模块本身不实现具体子系统逻辑，而是按照严格的依赖顺序调用各 crate（`arch`、`mm`、`security`、`sched`、`vfs`、`net`、`block`、`audit`、`trace`、`kernel_core`、`compliance`、`klog`、`livepatch` 等）的初始化接口，将它们组装为一个可运行的内核。

核心职责包括：

- **启动编排**：从 `_start` 入口开始，在中断禁用（IF=0）状态下完成所有关键子系统的初始化，最终启用中断并进入主循环。
- **安全策略绑定**：通过 PolicySurface 选择并锁定 compliance profile（支持 UEFI 启动命令行 `profile=secure` 解析），配置安全加固、CPU 保护特性、KASLR/KPTI，并为审计和可观测性子系统注册 capability gate。
- **故障策略定义**：定义 `alloc_error_handler` 和 `panic_handler`，包含递归保护、按 PolicySurface 的 panic 输出抑制（Secure 时 redacted）与加密 kdump。
- **测试编排**：在启动流程中同步执行编译级集成测试、24 项运行时功能测试，并创建 Ring 3 用户态测试进程（受 PolicySurface `debug_interfaces_enabled` 门控）。
- **演示模块挂载**：包含进程管理、系统调用、中断、内存分配等演示子模块（当前未在主路径调用）。

模块共包含 **9 个源文件**，`main.rs` 为 **1207 行** Rust 代码。

## 2. 文件结构

### 2.1 文件行数统计表

| 文件名 | 行数 | 职责概述 |
|--------|------|----------|
| `main.rs` | 1207 | 内核入口 `_start`、启动链路编排、`alloc_error_handler`、`panic_handler`、栈保护 |
| `runtime_tests.rs` | ~2533 | 运行时测试框架，24 个测试用例覆盖全部核心子系统 |
| `usermode_test.rs` | ~400 | Ring 3 用户态测试，加载内嵌 ELF 并创建用户进程 |
| `process_demo.rs` | ~187 | 进程管理与调度器演示 |
| `stack_guard.rs` | ~179 | 内核栈守护页安装（RSP0 + IST0） |
| `syscall_demo.rs` | ~168 | 系统调用演示 |
| `integration_test.rs` | ~101 | 编译级集成测试 |
| `interrupt_demo.rs` | ~77 | 中断统计与异常处理演示 |
| `demo.rs` | ~29 | 内存分配演示 |

### 2.2 模块依赖关系

`main.rs` 通过 `extern crate` 引入以下外部 crate：

```
drivers (macro_use), arch, block, cap, ipc, kernel_core, mm, net,
sched, security, vfs, livepatch, audit (macro_use), trace, compliance, klog (macro_use)
```

内部子模块声明：

```rust
mod demo;
mod integration_test;
mod interrupt_demo;
mod process_demo;
mod runtime_tests;
mod stack_guard;
mod syscall_demo;
mod usermode_test;
```

## 3. 逐文件分析

### 3.1 main.rs -- 内核入口与启动编排（1207 行）

#### 3.1.1 文件头与全局状态

文件使用 `#![no_std]` + `#![no_main]` 属性，声明为裸机内核入口。通过 `#[macro_use] extern crate drivers`、`#[macro_use] extern crate audit` 和 `#[macro_use] extern crate klog` 引入宏。

关键全局状态：

```rust
const SERIAL_PORT: u16 = 0x3F8;

/// 防止 alloc_error_handler 中递归分配的守卫标志
static COUNTERS_READY: AtomicBool = AtomicBool::new(false);

/// R109-4: 区分早期 boot 阶段和正常运行阶段
static BOOT_PHASE_COMPLETE: AtomicBool = AtomicBool::new(false);
```

`BOOT_PHASE_COMPLETE` 标志在 `sti` 前设置为 `true`，解决了 R109-4 问题：boot 完成后内核线程和中断处理器（`current_credentials() == None`）不应绕过 audit/trace capability gate。

#### 3.1.2 启动命令行解析（P1-1）

`parse_hardening_profile_from_cmdline(boot_info)` 解析 UEFI 启动命令行中的 `profile=<value>` 令牌：
- 扫描 `boot_info.cmdline[..cmdline_len]` 的空白分隔令牌
- 大小写不敏感前缀匹配 `profile=`
- 值通过 `compliance::HardeningProfile::from_str` 解析
- 多个 `profile=` 令牌时取**最后一个有效值**（Linux 内核语义）
- 不识别的值发出 `klog_force!` 警告（运维人员拼写错误检测）
- 返回 `None` 时回退到 `Balanced` 默认

#### 3.1.3 块设备写读验证函数

`test_block_write()` 对探测到的块设备执行端到端写读验证。受 `debug_interfaces_enabled` 门控（P1-1），Secure profile 下跳过此测试。

#### 3.1.4 启动链路详解（`_start` 函数）

`_start` 是内核的 `#[no_mangle] pub extern "C"` 入口点，接收 `BootInfo` 指针。以下按执行顺序分析：

**阶段 0：最早期初始化（中断禁用）**

- 立即 `cli` 禁用中断
- 串口发送第一条诊断消息
- 解析 `BootInfo` 指针

**阶段 0.5：显示输出初始化**

- 从 `BootInfo` 提取 framebuffer 信息，调用 `drivers::framebuffer::init()` 初始化 GOP framebuffer
- 调用 `drivers::vga_buffer::init()` 作为后备

**阶段 0.6：klog 早期绑定（P1-1）**

在堆初始化之前，从启动命令行解析 profile 并调用 `klog::set_profile()`，使 Secure profile 在第一条 `klog_always!` 输出前生效。

**阶段 1 [1/3]：中断处理初始化**

```rust
arch::interrupts::init();
```

加载 IDT，同时完成 GDT/TSS 初始化。注意此时 IF=0。

**阶段 2 [2/3]：内存管理初始化**

- `mm::memory::init_with_bootinfo(info)` 或回退到 `mm::memory::init()`
- `mm::page_table::init(VirtAddr::new(0))`

**阶段 2.5：内核栈守护页安装 + BSP IST 守护页**

```rust
unsafe { stack_guard::install() }
arch::gdt::install_bsp_ist_guard_page();   // R149-I5: 双重错误 IST
arch::gdt::install_bsp_nmi_guard_page();   // NMI IST
```

在 mm 初始化后、中断启用前安装。BSP IST 守护页被延迟安装（因为 `gdt::init()` 在页表初始化前运行）。

**阶段 2.6：安全加固 + PolicySurface 初始化**

这是启动链路中最复杂的阶段：

1. **解析启动命令行**获取 profile 和 source（`BootCmdline` 或 `Default`）
2. **初始化 PolicySurface**：`compliance::init_policy_surface(profile, source)` 构建 boot-time locked 策略
3. **klog 二次绑定**：从 PolicySurface 重新设置 klog 过滤级别（defense-in-depth）
4. **生成 SecurityConfig**：从 profile 和 `phys_offset` 生成安全配置
5. **执行 `security::init()`**：W^X 强制、NX 保护、恒等映射清理、CSPRNG 初始化（ChaCha20 + RDRAND/RDSEED）、kptr guard、Spectre 缓解
6. **锁定 profile**：`compliance::lock_profile()`
7. **输出 PolicySurface enforcement 摘要**：列出所有安全特性状态
8. **Secure 模式失败关闭**：若 `security::init()` 失败且为 Secure profile，直接 `panic!`
9. **RNG 自检**：调用 `security::fill_random()` 验证 RNG（R149-5：使用 FIPS 边界 API）

**阶段 2.65：KASLR/KPTI/PCID 初始化**

```rust
security::init_kaslr(boot_info.map(|info| info.kaslr_slide));
```

**P1-1 PolicySurface 驱动的 fail-closed 策略**：
- `kaslr_fail_closed == true`（Secure profile）时：
  - 无 KASLR → 检查 partial KASLR 作为回退
  - 无任何随机化 → `panic!`
- `kpti_fail_closed == true` 时：警告（KPTI 实现中，暂不 panic）

```rust
mm::tlb_shootdown::init_invpcid_support();
```

**阶段 2.7：CPU 硬件保护特性**

启用 SMEP/SMAP/UMIP，`arch::cpu_protection::require_smap_support()` 将 SMAP 作为**硬性启动前置条件**。

**阶段 2.8：SYSCALL/SYSRET 初始化**

```rust
let syscall_entry = arch::syscall::syscall_entry_stub as *const () as u64;
unsafe { arch::init_syscall_msr(syscall_entry); }
arch::register_frame_callback();
kernel_core::register_kpti_cr3_callback(arch::arch_set_kpti_cr3s);
security::kaslr::enable_kpti();
```

新增 KPTI 支持（R118-3）：
- 注册 per-CPU CR3 更新回调，使 syscall 汇编的 GS-relative CR3 对在上下文切换中保持同步
- 无条件启用 KPTI（所有 pre-Whiskey Lake Intel CPU 易受 Meltdown 攻击）

**阶段 5 [5/8]：kernel_core 初始化**

```rust
kernel_core::init();
```

初始化进程管理和 BOOT_CR3 缓存。

**阶段 5.5 [5.5/8]：APIC/SMP 初始化**

| 步骤 | 说明 |
|------|------|
| RSDP 注入 | 从 BootInfo 传递 ACPI RSDP 地址 |
| BSP LAPIC 初始化 | `arch::apic::init()` |
| HPET 初始化 | 高精度事件计时器（可选） |
| LAPIC timer 校准 | 优先 HPET，回退 PIT |
| BSP per-CPU 数据 | 初始化内核栈、IRQ 栈、Syscall 栈 |
| IRQ CPU-Local 强制初始化 | R151-5：防止 IRQ 路径惰性初始化死锁 |
| BSP syscall per-CPU 状态 | `arch::syscall::init_syscall_percpu(0)` |
| AP 启动 | `arch::start_aps()` 枚举并拉起 AP |

**R151-5 修复**：在中断启用前强制初始化 IRQ 路径的 CpuLocal 静态变量（`force_init_irq_cpu_locals()` + `force_init_resched_locals()`），防止首次 IRQ 在堆锁持有时触发 `Once::call_once()` 死锁。

**阶段 6-7：调度器 / IPC / VFS / 页缓存 / 网络 / 块层**

| 阶段 | 操作 |
|------|------|
| [6/8] | 增强调度器 + cpuset 初始化 |
| [7/8] | IPC capability endpoints |
| [7.5/8] | VFS + devfs 挂载 |
| [7.52/8] | 页缓存初始化 |
| [7.54/8] | 网络层设备探测 |
| [7.55/8] | 块层 virtio-blk + ext2 挂载（块写测试受 debug_interfaces_enabled 门控） |

**阶段 7.6：审计子系统初始化**

审计初始化是入口模块中安全关键度最高的阶段之一：

1. **容量来源**：`compliance::policy().audit_ring_capacity`（从 PolicySurface 获取）
2. **boot 事件**发射作为审计链起始点
3. **延迟策略审计事件**：`compliance::emit_deferred_policy_audit()` 重发初始化阶段被丢弃的 PolicySurface 验证事件
4. **时间戳回调注册**（R148-I8）：`audit::register_timestamp_callback(kernel_core::time::current_timestamp_ms)`
5. **snapshot 授权器**：允许 root（`current_is_host_root()`，R133-1）或 `CAP_AUDIT_READ`
6. **HMAC key 授权器**：允许 root 或 `CAP_AUDIT_WRITE`
7. **R109-4 boot 阶段感知**：授权器在 `BOOT_PHASE_COMPLETE == false` 时放行无凭证请求，完成后拒绝
8. **HMAC 密钥生成与安装**：CSPRNG 32 字节 + volatile 清零 + compiler fence
9. **OOM killer 审计回调**（R106-8）：OOM kill 事件注入 hash chain
10. **Livepatch 审计回调**（P1-4）：livepatch 生命周期事件注入 hash chain

**阶段 7.7：可观测性子系统初始化**

1. `trace::init()`
2. 强制触发 per-CPU 计数器惰性初始化
3. 设置 `COUNTERS_READY = true`
4. 安装 trace 读取守卫（`CAP_TRACE_READ` 或 root，R109-4/R133-1 感知）

**测试执行阶段（PolicySurface 门控）**

```rust
if compliance::policy().debug_interfaces_enabled {
    integration_test::run_all_tests();
    runtime_tests::run_all_runtime_tests();
    usermode_test::run_usermode_test();
} else {
    klog_force!("[POLICY] {} profile: debug/test interfaces disabled", ...);
}
```

P1-1 修复：Secure profile 下 `debug_interfaces_enabled == false`，跳过所有测试以减少攻击面。

**主循环**

```rust
arch::interrupts::enable_serial_interrupts();
BOOT_PHASE_COMPLETE.store(true, Ordering::Release);
unsafe { core::arch::asm!("sti", options(nomem, nostack)); }
sched::enhanced_scheduler::Scheduler::reschedule_now(true);

loop {
    kernel_core::reschedule_if_needed();
    unsafe { core::arch::asm!("hlt", options(nomem, nostack, preserves_flags)); }
}
```

主循环每次唤醒后先调用 `reschedule_if_needed()` 排空延迟工作（R98-3）。

#### 3.1.5 alloc_error_handler

```rust
#[alloc_error_handler]
fn alloc_error_handler(layout: Layout) -> ! {
    if COUNTERS_READY.load(Ordering::Relaxed) {
        increment_counter(TraceCounter::AllocFailures, 1);
    }
    // 串口直接输出分配大小（不依赖堆）
    panic!("Allocation error: {:?}", layout);
}
```

递归保护：仅在 `COUNTERS_READY` 为 true 时递增计数器。

#### 3.1.6 panic_handler

panic 处理器包含以下关键行为：

1. 立即 `cli` 禁用中断
2. `trace::kdump::capture_crash_context(info)` 捕获 CPU 寄存器、栈内容和 panic 信息
3. **PolicySurface 驱动的输出抑制**（G.fin.1）：`compliance::policy().panic_redact_details` 为 true 时，仅输出 `"KERNEL PANIC [Secure mode: details redacted]"`
4. 对于 PolicySurface 尚未初始化的极早期 panic，使用 `compliance::is_policy_initialized()` 判断（fail-open：显示详情以辅助调试）
5. **加密 kdump**：`trace::kdump::emit_encrypted_dump()` 输出 ChaCha20 加密转储

#### 3.1.7 栈保护支持（R148-I4）

```rust
#[no_mangle]
#[used]
pub static __stack_chk_guard: u64 = 0x595e_9fbd_94fd_a766;

#[no_mangle]
pub extern "C" fn __stack_chk_fail() -> ! {
    panic!("stack smashing detected: stack canary corrupted");
}
```

为编译器的 `-fstack-protector` 提供 canary 值和失败处理函数。

### 3.2 其他文件（简述）

**runtime_tests.rs（~2533 行）**：24 项运行时测试，三态结果模型（Pass/Warning/Fail），IPI 测试自适应 QEMU TCG 阈值。

**usermode_test.rs（~400 行）**：通过 Feature Gate 选择 ELF（hello/shell/syscall_test/musl_test/clone_test），四步创建流程（PCB → 地址空间 → ELF 加载 → 上下文配置）。

**stack_guard.rs（~179 行）**：在 `0xFFFF_FFFF_7000_0000`（RSP0）和 `0xFFFF_FFFF_6F00_0000`（IST0）安装带守护页的新栈。

**integration_test.rs（~101 行）**：8 个编译级测试验证子系统链接。

## 4. 模块整体评估

### 4.1 启动链路架构

**优势：**

- **初始化顺序严格正确**：依赖关系被精确编排——中断在内存管理之前初始化，栈守护页在 mm 之后/中断启用前，SYSCALL MSR 在 GDT 之后，调度器在 kernel_core 之后。
- **PolicySurface 单一来源**（G.fin.1）：所有安全策略决策通过 `compliance::policy()` 查询同一不可变结构，消除策略漂移。
- **启动命令行解析**（P1-1）：支持 `profile=secure` 从 UEFI 命令行选择 profile，运维人员无需重新编译内核。
- **KPTI 完整集成**（R118-3）：dual page table roots + CR3 switching + per-CPU 回调。
- **boot 阶段感知**（R109-4）：`BOOT_PHASE_COMPLETE` 标志防止 post-boot 内核线程绕过 capability gate。
- **IRQ 死锁防护**（R151-5）：在中断启用前强制初始化 CpuLocal 静态变量。

**改进建议：**

- **阶段编号不连续**：`[1/3]` 到 `[8/8]` 中间插入大量非整数编号（`[2.5/3]`、`[7.55/8]` 等），建议采用命名阶段。
- **启动时同步测试**：24 项运行时测试在每次启动时执行。建议 Secure profile 下已通过 `debug_interfaces_enabled` 门控解决。
- **演示模块与主路径脱节**：`process_demo.rs` 引用旧调度器 API（`sched::scheduler`）。

### 4.2 安全与合规评估

| 安全特性 | 实现状态 | 关键代码位置 |
|----------|----------|-------------|
| PolicySurface（boot-time locked） | 已实现 | `main.rs` 阶段 2.6 |
| 启动命令行 profile 解析 | 已实现 | `parse_hardening_profile_from_cmdline()` |
| W^X 强制 | 已实现 | via `security::init()` |
| NX 保护 | 已实现 | via `security::init()` |
| CSPRNG (ChaCha20 + RDRAND/RDSEED) | 已实现 | via `security::init()` |
| kptr guard | 已实现 | via `security::init()` |
| Spectre 缓解 | 已实现 | via `security::init()` |
| SMEP/SMAP/UMIP | 已实现 | `main.rs` 阶段 2.7 |
| SMAP 硬性要求 | 已实现 | `require_smap_support()` |
| KASLR fail-closed（Secure） | 已实现 | `main.rs` 阶段 2.65 |
| KPTI + CR3 callback | 已实现 | `main.rs` 阶段 2.8 |
| 审计 HMAC-SHA256 | 已实现 | `main.rs` 阶段 7.6 |
| 审计时间戳回调（R148-I8） | 已实现 | `main.rs` 阶段 7.6 |
| 密钥擦除 (volatile + fence) | 已实现 | `main.rs` 阶段 7.6 |
| Panic 输出抑制 (PolicySurface) | 已实现 | `panic_handler` |
| 加密 kdump | 已实现 | `panic_handler` |
| Profile 锁定 | 已实现 | `compliance::lock_profile()` |
| Livepatch 密钥占位符告警 | 已实现 | `main.rs` 阶段 2.6 后 |
| Livepatch 审计回调（P1-4） | 已实现 | `main.rs` 阶段 7.6 |
| Capability gate (audit/trace) | 已实现 | `main.rs` 阶段 7.6/7.7 |
| OOM 审计回调（R106-8） | 已实现 | `main.rs` 阶段 7.6 |
| 延迟策略审计事件（P1-1） | 已实现 | `emit_deferred_policy_audit()` |
| klog profile-aware 输出 | 已实现 | `main.rs` 阶段 0.6 + 2.6 |
| 栈保护（R148-I4） | 已实现 | `__stack_chk_guard` / `__stack_chk_fail` |
| BSP IST 守护页（R149-I5） | 已实现 | `install_bsp_ist_guard_page()` |
| IRQ CpuLocal 预初始化（R151-5） | 已实现 | `force_init_irq_cpu_locals()` |
| 测试接口 PolicySurface 门控（P1-1） | 已实现 | `debug_interfaces_enabled` 检查 |
| 块写测试 PolicySurface 门控（P1-1） | 已实现 | `debug_interfaces_enabled` 检查 |

### 4.3 启动阶段总览表

| 阶段 | 前置条件 | 动作 | 产物 | 失败策略 |
|------|----------|------|------|----------|
| cli + serial | 无 | 禁用中断，串口输出 | 最小诊断通道 | 无法失败 |
| Framebuffer/VGA | BootInfo | 初始化显示输出 | `println!` 可用 | VGA 后备 |
| klog 早期绑定 | BootInfo | 命令行 profile → klog filter | Secure 抑制生效 | 默认 Balanced |
| 中断 [1/3] | 无 | 加载 IDT/GDT/TSS | 异常处理就绪 | panic |
| 内存 [2/3] | 无 | 堆 + Buddy + 页表 | 动态内存可用 | panic |
| 栈守护 [2.5/3] | mm | guard page + BSP IST/NMI 守护页 | RSP0/IST0/NMI 受保护 | 警告并继续 |
| PolicySurface + 安全加固 [2.6/3] | mm | init_policy_surface + security::init | 安全基线 + 策略锁定 | Secure: panic |
| KASLR/KPTI [2.65/3] | security, PolicySurface | KASLR slide + INVPCID + fail-closed | 地址随机化 | Secure 无 KASLR: panic |
| CPU 保护 [2.7/3] | 无 | SMEP/SMAP/UMIP | 硬件级隔离 | SMAP 缺失: panic |
| SYSCALL + KPTI [2.8/3] | GDT | MSR + 帧回调 + KPTI CR3 回调 + enable_kpti | Ring 3 转换就绪 | panic |
| kernel_core [5/8] | 无 | 进程管理 + BOOT_CR3 | PCB 基础设施 | panic |
| APIC/SMP [5.5/8] | kernel_core | LAPIC/HPET/timer + AP + 预初始化 | 多核就绪 | 单核降级 |
| 调度器 [6/8] | kernel_core | 增强调度器 + cpuset | 调度就绪 | panic |
| IPC [7/8] | kernel_core | capability endpoints | IPC 就绪 | panic |
| VFS [7.5/8] | 无 | devfs + 设备文件 | 文件系统就绪 | panic |
| 页缓存 [7.52/8] | mm | 全局页缓存 | 缓存就绪 | panic |
| 网络 [7.54/8] | 无 | 设备探测 | 网络就绪 | 无设备时警告 |
| 块层 [7.55/8] | VFS, PolicySurface | virtio-blk + ext2 + 门控测试 | 存储就绪 | 无设备时跳过 |
| 审计 [7.6/8] | security, cap, compliance | ring + HMAC + gate + timestamp + livepatch callback | 审计就绪 | 警告并继续 |
| 可观测 [7.7/8] | mm | trace + counters + gate | 监控就绪 | 警告并继续 |
| 测试 | 全部子系统, PolicySurface | 集成 + 运行时 + Ring 3（门控） | 验证报告 | 失败计数警告 |
| BOOT_PHASE_COMPLETE + sti + 主循环 | 全部 | 标记 boot 完成 + 启用中断 | 系统运行 | -- |
