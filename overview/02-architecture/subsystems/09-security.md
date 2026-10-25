# Zero-OS 安全加固模块（kernel/security）深度分析

> 分析日期：2026-05-13

## 1. 模块概述

`kernel/security` 是 Zero-OS 内核的安全加固子系统，提供从内存保护到推测执行缓解的多层纵深防御体系。该模块在内核初始化早期阶段运行，通过统一的 7 步初始化流程完成所有安全加固操作，并生成可审计的安全报告。

### 1.1 核心安全能力

| 能力 | 描述 | 实现状态 |
|------|------|----------|
| W^X 强制 | 确保无内存页同时可写且可执行 | 完整实现 |
| 身份映射清理 | 移除或加固引导加载器的恒等映射 | 完整实现 |
| NX 位强制 | 对数据页设置不可执行位 | 完整实现 |
| 硬件 RNG | RDRAND/RDSEED + ChaCha20 CSPRNG（FKE 构造） | 完整实现 |
| kptr 指针混淆 | 防止通过日志泄露内核地址 | 完整实现 |
| Spectre/Meltdown 缓解 | IBRS/IBPB/STIBP/SSBD/RSB 填充 | Spectre 系列完整实现；Meltdown 依赖 KPTI |
| KASLR | PCID 管理、Text KASLR（PIE 引导加载器配合）、Partial KASLR（堆/栈/mmap 随机化） | Text KASLR 完整（H.2）；Partial KASLR 完整实现 |
| KPTI 框架 | per-CPU seqlock CR3 切换、双页表基础设施 | 基础设施就绪（H.3），per-CPU 存储已迁移 |
| FIPS 模式 | FIPS 140-2/140-3 合规状态机，ChaCha20 FIPS 门控 | 完整实现（R140-6） |
| 运行时安全自测 | 轻量级安全不变量验证 | 完整实现 |

### 1.2 设计原则

- **纵深防御（Defense in Depth）**：多层保护机制叠加
- **安全失败（Fail-Secure）**：错误导致更严格的状态
- **最小权限（Least Privilege）**：默认最小权限配置
- **审计追踪（Audit Trail）**：所有安全事件均有日志记录

---

## 2. 文件结构

| 文件 | 行数 | 功能概述 |
|------|------|----------|
| `Cargo.toml` | 18 | 包配置，依赖 spin、lazy_static、x86_64、mm、drivers、klog、cpu_local |
| `lib.rs` | 501 | 模块入口：SecurityConfig/SecurityReport/SecurityError 定义，7 步初始化编排 |
| `memory_hardening.rs` | 874 | 内存加固：身份映射清理、NX 位强制、W^X 段保护、MMIO 保护 |
| `kaslr.rs` | 1663 | KASLR/KPTI 基础设施：PCID 管理、Text KASLR、Partial KASLR（堆/栈/mmap）、per-CPU KPTI seqlock |
| `wxorx.rs` | 320 | W^X 策略验证：4 级页表遍历检查 WRITABLE+EXECUTABLE 违规 |
| `rng.rs` | 700 | 硬件随机数：RDRAND/RDSEED 熵源 + ChaCha20 CSPRNG（FKE 构造）+ FIPS 门控 |
| `kptr.rs` | 282 | 内核指针混淆：SplitMix64 混合的地址保护 |
| `spectre.rs` | 618 | Spectre/Meltdown 缓解：CPUID 探测 + MSR 配置 + RSB 填充 |
| `fips.rs` | 71 | FIPS 模式状态机：Disabled/Enabled/Failed 三态，单调转换 |
| `tests.rs` | 426 | 运行时安全自测框架：5 项内置测试 |

---

## 3. 各文件详细分析

### 3.1 lib.rs — 模块入口与初始化编排

**文件路径**: `kernel/security/lib.rs`（501 行）

#### 3.1.1 功能概述

`lib.rs` 是安全模块的顶层入口，承担三项核心职责：

1. **类型聚合与重导出**：将各子模块的公共类型统一导出，为内核其他模块提供单一入口
2. **配置与报告体系**：定义 `SecurityConfig`（安全配置）和 `SecurityReport`（安全报告）
3. **初始化编排**：通过 `init()` 函数按固定顺序执行 7 步安全加固

#### 3.1.2 子模块声明

```rust
pub mod fips;
pub mod kaslr;
pub mod kptr;
pub mod memory_hardening;
pub mod rng;
pub mod spectre;
pub mod tests;
pub mod wxorx;
```

与早期版本相比，新增了 `fips` 模块（R140-6 FIX）。

#### 3.1.3 公共 API 边界（FIPS 收紧）

```rust
// R141-8 FIX: ChaCha20Rng 和 rdrand64_early 从公共导出中移除，
// 强制 FIPS 边界。外部调用者使用 fill_random() 获取随机字节，
// chacha20_xor_keystream() 用于加密。
// R149-5 FIX: random_u32, random_u64 也从公共导出中移除。
pub use rng::{chacha20_xor_keystream, fill_random, rdrand_available, try_fill_random, RngError};
```

KASLR 模块导出扩展为包含 `text_kaslr_status`、`TextKaslrStatus`、`PartialKaslrFeature` 等新类型。

#### 3.1.4 核心数据结构

**SecurityConfig** — 安全加固配置：

```rust
pub struct SecurityConfig {
    pub phys_offset: VirtAddr,
    pub cleanup_strategy: IdentityCleanupStrategy,
    pub enforce_nx: bool,
    pub validate_wxorx: bool,
    pub initialize_rng: bool,
    pub strict_wxorx: bool,
    pub enable_kptr_guard: bool,
    pub enable_spectre_mitigations: bool,
    pub run_security_tests: bool,
}
```

提供三种预设配置：
- `default()` — 平衡模式：启用大部分加固，W^X 违规仅警告
- `strict()` — 严格模式：W^X 违规即返回错误（初始化失败），启用安全自测
- `permissive()` — 宽松模式：仅用于调试，关闭大部分加固

**SecurityReport** — 安全加固报告：

```rust
pub struct SecurityReport {
    pub identity_cleanup: CleanupOutcome,
    pub nx_summary: Option<NxEnforcementSummary>,
    pub wxorx_summary: Option<ValidationSummary>,
    pub rng_ready: bool,
    pub kptr_guard_active: bool,
    pub spectre_status: Option<MitigationStatus>,
    pub test_report: Option<TestReport>,
    pub total_violations: usize,
}
```

`is_secure()` 判定语义：`total_violations == 0 && rng_ready && test_report.failed == 0`（warnings 不影响安全判定）。

**SecurityError** — 统一错误类型：

```rust
pub enum SecurityError {
    Memory(HardeningError),
    Wxorx(WxorxError),
    Rng(RngError),
    Spectre(SpectreError),
}
```

#### 3.1.5 核心函数

**`init(config, frame_allocator)`** — 7 步初始化流程：

| 步骤 | 操作 | 说明 |
|------|------|------|
| 0 | `kaslr::set_kaslr_fail_closed(config.strict_wxorx)` | R102-11 FIX：Secure 配置下 KASLR RNG 失败 panic |
| 1 | kptr guard 初始化 | 最先执行，保护后续所有日志输出 |
| 2 | 身份映射清理 | 移除/加固引导加载器的恒等映射 |
| 3 | NX 位强制 | 对内核数据段设置不可执行位 |
| 4 | W^X 验证 | 遍历页表检查违规 |
| 5 | RNG 初始化 | 初始化 CSPRNG，验证可用性，用强熵重播种 kptr |
| 6 | Spectre 缓解 | 启用 IBRS/STIBP/SSBD 等 |
| 7 | 安全自测 | 运行内置安全测试（可选） |

**`quick_check(phys_offset)`** — 运行时轻量 W^X 检查，可周期性调用。

#### 3.1.6 模块间依赖

- 依赖 `mm::memory::FrameAllocator` 进行页表修改
- 依赖 `mm::page_table::PHYSICAL_MEMORY_OFFSET` 获取物理内存偏移
- 编排所有子模块的初始化顺序

---

### 3.2 memory_hardening.rs — 内存保护加固

**文件路径**: `kernel/security/memory_hardening.rs`（874 行）

#### 3.2.1 功能概述

该文件是安全模块中最核心的实现，提供三大内存保护能力：

1. **身份映射清理**：移除或加固引导加载器创建的低半区恒等映射（PML4[0]）
2. **NX 位强制**：对内核高半区各段落施加正确的页表权限
3. **W^X 段保护**：确保 text 段 R-X、rodata 段 R--、data/bss 段 RW-NX

#### 3.2.2 核心数据结构

```rust
pub enum IdentityCleanupStrategy {
    Unmap,           // 完全移除恒等映射
    RemoveWritable,  // 保留映射但去除可写标志并添加 NX
    Skip,            // 跳过（仅调试用）
}

pub enum CleanupOutcome {
    Unmapped,
    ReadOnlyUpdated { updated_entries: usize },
    AlreadyAbsent,
    Skipped,
}

pub enum HardeningError {
    PageTableMissing(&'static str),
    FrameAllocFailed,
    InconsistentTopology,
    InvalidAddress,
    UnsafeOperation(&'static str),
}

pub struct NxEnforcementSummary {
    pub text_rx_pages: usize,
    pub ro_pages: usize,
    pub data_nx_pages: usize,
}
```

#### 3.2.3 核心函数

**`cleanup_identity_map(phys_offset, strategy)`**

身份映射清理的主入口。关键设计变更：

- **R144-4 / R152-19 FIX**：添加 `assert!(cpu_local::num_online_cpus() <= 1)` 强制在 SMP 启动前调用，因为该函数仅做本地 TLB 刷新（`flush_all_local`），在 SMP 后调用会导致远端 CPU 的 TLB 陈旧
- **R115-4 FIX**：使用 PCID 感知的 `mm::tlb_shootdown::flush_all_local()` 替代原始 `tlb::flush_all()`

算法流程：
1. 确保 MMIO 区域可达（`protect_mmio_regions`）
2. 读取当前 RSP 以保护引导加载器栈
3. 通过 `with_active_level_4_table` 访问 PML4[0]
4. 对于 `RemoveWritable` 策略：通过递归页表槽（PML4[510]）遍历 PDPT/PD，拆分含 MMIO 的 2MB huge page，逐页硬化

**`enforce_nx_for_kernel(phys_offset, frame_allocator)`**

内核段 NX 强制的主入口。同样增加了 pre-SMP 断言（R144-4/R152-19 FIX）。算法流程：

1. 从链接器符号获取各段边界（text/rodata/data/bss）
2. 调用 `ensure_pte_range` 将相关范围的 huge page 降级为 4KB 粒度
3. **基线保护**：`mark_all_nx()` 先对 PD 下所有叶子页添加 NX 位
4. 按段精调权限：text→R-X、rodata→R--、data/bss→RW-NX
5. 使用 `flush_all_local()` 刷新 TLB

**辅助函数**：

| 函数 | 功能 |
|------|------|
| `mark_all_nx()` | 为 PD 下所有叶子页添加 NX 位（W^X 基线） |
| `apply_section()` | 对指定段范围的 4KB 页应用权限调整闭包 |
| `harden_identity_pt()` | 加固恒等映射的页表，保留 MMIO/栈/SMP trampoline |
| `harden_identity_pd_entry_recursive()` | 通过递归映射加固单个 PD 条目，含 MMIO huge page 拆分（L-5 FIX） |
| `is_mmio_page()` / `is_mmio_2mb_region()` | 检测 MMIO 区域（VGA/APIC/GOP framebuffer） |
| `is_smp_trampoline_page()` | 检测 SMP trampoline 页（0x8000） |
| `is_stack_page()` | 检测引导加载器栈页（RSP +/- 8 页） |

#### 3.2.4 模块间依赖

- 强依赖 `mm::page_table`：递归页表访问（`recursive_pdpt/pd/pt`）、`split_2m_entry`、`ensure_pte_range`、`map_mmio`
- 强依赖 `mm::memory::FrameAllocator`：页表拆分时分配新帧
- 依赖 `drivers::framebuffer::get_framebuffer_region()`：动态获取 GOP framebuffer 区域
- 依赖 `cpu_local::num_online_cpus()`：pre-SMP 安全断言
- 依赖 `mm::tlb_shootdown::flush_all_local()`：PCID 感知 TLB 刷新

---

### 3.3 kaslr.rs — KASLR/KPTI 基础设施

**文件路径**: `kernel/security/kaslr.rs`（1663 行）

#### 3.3.1 功能概述

该文件是安全模块中最大的文件，提供 KASLR（内核地址空间布局随机化）和 KPTI（内核页表隔离）的完整基础设施。主要功能：

- **PCID 检测与 CR4.PCIDE 启用** — 完整实现
- **PCID 位图分配器（1-4095）** — 完整实现
- **Text KASLR**（H.2 完成）：引导加载器通过 `.rela.dyn` 重定位引擎加载内核到随机物理地址，内核从链接器符号检测 slide
- **Partial KASLR**：堆基址随机化（mm::memory 接入）、内核栈基址随机化（CSPRNG 生成 2 MiB 对齐 slide，最大 256 MiB）、用户空间 mmap ASLR（每进程 4KB 粒度，最大 256 MiB 偏移）
- **KernelLayout**：从链接器符号构建运行时内核位置信息，支持 KASLR slide 检测
- **KPTI per-CPU seqlock**（H.3 SMP FIX）：KPTI 上下文从全局原子变量迁移到 per-CPU 存储（`cpu_local` crate）

#### 3.3.2 核心数据结构

**KernelLayout** — 内核内存布局：

```rust
pub struct KernelLayout {
    pub virt_base: u64,      // 虚拟基地址（始终为 KERNEL_VIRT_BASE，R119-4 FIX）
    pub phys_base: u64,      // 物理基地址（包含 slide，R39-7 FIX）
    pub kaslr_slide: u64,    // KASLR 滑动值
    pub text_start: u64,     // text 段起始
    pub text_size: u64,
    pub rodata_start: u64,
    pub rodata_size: u64,
    pub data_start: u64,
    pub data_size: u64,
    pub bss_start: u64,
    pub bss_size: u64,
    pub heap_start: u64,     // 堆起始（可能被 Partial KASLR 随机化）
    pub heap_size: u64,
    pub phys_offset: u64,
}
```

通过 `build_kernel_layout_from_linker()` 从链接器符号构建运行时布局。`virt_to_phys()` 使用固定 `PHYSICAL_MEMORY_OFFSET` 常量（R119-4 FIX：不随 slide 变化）。

**TextKaslrStatus** — Text KASLR 状态（新增）：

```rust
pub struct TextKaslrStatus {
    pub enabled: bool,           // slide 非零则启用
    pub slide: u64,              // 引导加载器传入的 slide
    pub slide_aligned_2m: bool,  // 是否满足 2 MiB 对齐合约
}
```

**PartialKaslrStatus** — 部分 KASLR 特性状态：

```rust
pub struct PartialKaslrStatus {
    pub heap_base: bool,       // 堆基址随机化
    pub kernel_stacks: bool,   // 内核栈随机化
    pub userspace_aslr: bool,  // 用户空间 ASLR（mmap/stack/brk）
}
```

**KptiContext** — KPTI 页表上下文：

```rust
pub struct KptiContext {
    pub user_cr3: u64,    // 用户态页表根（物理地址，不含 PCID 位）
    pub kernel_cr3: u64,  // 内核态页表根（物理地址，不含 PCID 位）
    pub pcid: u16,        // 进程上下文 ID
}
```

提供 `user_cr3_with_pcid()` / `kernel_cr3_with_pcid()` 方法，防御性掩码确保 CR3 4 KiB 对齐（R102-14 FIX）。

**KPTI 上下文存储** — per-CPU seqlock 设计（H.3 SMP FIX）：

```text
原设计（全局原子变量）：
  static KPTI_USER_CR3: AtomicU64;
  static KPTI_KERNEL_CR3: AtomicU64;
  static KPTI_PCID: AtomicU64;
  static KPTI_SEQ: AtomicU64;

新设计（per-CPU 存储）：
  cpu_local::current_cpu().kpti_user_cr3
  cpu_local::current_cpu().kpti_kernel_cr3
  cpu_local::current_cpu().kpti_pcid
  cpu_local::current_cpu().kpti_seq
```

每个 CPU 独立存储 KPTI 上下文，避免 SMP 下多 CPU 同时切换进程互相覆盖。seqlock 协议仅保护同一 CPU 上的中断级并发。

#### 3.3.3 核心函数

| 函数 | 功能 |
|------|------|
| `init(boot_slide)` | 初始化：PCID 启用 → 链接器符号布局 → R39-7 slide 验证 → 内核栈 slide → Partial KASLR 跟踪 |
| `enable_pcid_if_supported()` | 检测 CPUID 并设置 CR4.PCIDE |
| `allocate_pcid()` / `free_pcid()` | 位图式 PCID 分配/释放（1-4095） |
| `generate_kaslr_slide()` | 通过 CSPRNG 生成 2 MiB 对齐的随机滑动值（Secure 配置下 RNG 失败 panic） |
| `generate_kstack_slide()` | 生成内核栈基址 slide（2 MiB 粒度，最大 256 MiB） |
| `randomized_mmap_base(default_base)` | 每进程 mmap 基址随机化（4 KiB 粒度，最大 256 MiB 偏移） |
| `build_kernel_layout_from_linker()` | 从链接器符号构建运行时 KernelLayout |
| `text_kaslr_status()` | 返回 Text KASLR 状态（slide 及对齐信息） |
| `install_kpti_context(ctx)` | per-CPU seqlock 写协议安装 KPTI 上下文（**必须中断禁用**） |
| `current_kpti_context()` | per-CPU seqlock 读协议获取一致性 KPTI 快照 |
| `enter_kernel_mode()` / `return_to_user_mode()` | KPTI CR3 切换（syscall/中断入口出口） |
| `kernel_stack_slide()` | 返回当前内核栈基址 slide（0 = 禁用） |

**PCID 分配器设计**：使用 64 个 `u64` 组成的位图（4096 位），通过 `Mutex` 保护。PCID 0 保留，有效范围 1-4095。

**R118-4 FIX**：移除 `install_kpti_context` 中对 `KPTI_ENABLED` 的动态切换。`KPTI_ENABLED` 是系统级策略标志，仅在启动时由 `enable_kpti()` 设置一次，不再随上下文切换变化。

#### 3.3.4 日志安全

敏感地址信息（KASLR slide、堆基址、内核段地址）通过 `#[cfg(debug_assertions)]` 门控，生产模式下不输出（R101-2 FIX）。

#### 3.3.5 模块间依赖

- 依赖 `crate::rng`：KASLR/栈/mmap 随机值生成
- 依赖 `mm::memory`：获取堆基址（`heap_base()`、`heap_size()`、`heap_randomized()`）
- 依赖 `cpu_local`：per-CPU KPTI 上下文存储（H.3 SMP FIX）
- 依赖 `x86_64::registers::control::Cr4`：PCID 启用
- 被 `lib.rs` 调用进行初始化
- 被内核其他模块查询内核布局和 KPTI 状态

---

### 3.4 wxorx.rs — W^X 策略验证

**文件路径**: `kernel/security/wxorx.rs`（320 行）

#### 3.4.1 功能概述

实现 W^X（Write XOR Execute）策略的运行时验证。通过遍历当前活动的 4 级 x86_64 页表层次结构，检查是否存在同时可写且可执行的内存页。

#### 3.4.2 核心数据结构

```rust
pub struct ValidationSummary {
    pub scanned_entries: usize,
    pub violations: usize,
}

pub struct Violation {
    pub virt_base: VirtAddr,
    pub phys: PhysAddr,
    pub level: PageLevel,
    pub flags: PageTableFlags,
}

pub enum PageLevel { P4, P3, P2, P1, Huge1G, Huge2M }

pub enum WxorxError {
    Violation(Violation),
    PolicyViolation(ValidationSummary),  // X-3 FIX
    IncompleteWalk(&'static str),
    InvalidStructure,
}
```

#### 3.4.3 核心函数

**`validate_active(phys_offset)`** — 主验证入口：

1. 从 CR3 寄存器读取当前活动页表根
2. 遍历 PML4 → PDPT → PD → PT 四级页表
3. 跳过递归页表槽（`RECURSIVE_INDEX`），避免将页表中间条目误判为叶子 PTE
4. 对每个叶子条目检查 `WRITABLE && !NO_EXECUTE`
5. API 语义（X-3 FIX）：`Ok` 表示零违规，`Err(PolicyViolation)` 表示存在违规

**`check_address(virt_addr, phys_offset)`** — 单地址 W^X 合规检查。

**`is_wxorx_violation(flags)`** — 核心判定逻辑：

```rust
fn is_wxorx_violation(flags: PageTableFlags) -> bool {
    let writable = flags.contains(PageTableFlags::WRITABLE);
    let executable = !flags.contains(PageTableFlags::NO_EXECUTE);
    writable && executable
}
```

**`canonicalize(addr)`** — x86_64 地址规范化（bit 47 符号扩展）。

#### 3.4.4 模块间依赖

- 依赖 `mm::page_table::RECURSIVE_INDEX`：跳过递归页表槽
- 依赖 `mm::page_table::with_current_manager`：单地址查询
- 依赖 `x86_64::registers::control::Cr3`：读取活动页表根
- 被 `lib.rs` 的 `init()` 和 `quick_check()` 调用

---

### 3.5 rng.rs — 硬件随机数生成器与 CSPRNG

**文件路径**: `kernel/security/rng.rs`（700 行）

#### 3.5.1 功能概述

提供密码学安全的随机数生成能力，采用分层架构：

1. **硬件熵源层**：RDSEED（真随机，较慢）→ RDRAND（条件随机，较快）
2. **CSPRNG 层**：ChaCha20 流密码作为 PRNG，采用 Fast-Key-Erasure（FKE）构造（R109-3 FIX）
3. **自动重播种**：每生成 1MB 数据后从硬件熵源重播种
4. **FIPS 门控**（R140-6 FIX）：FIPS Enabled 时回退到直接硬件熵，FIPS Failed 时 fail-closed

#### 3.5.2 核心数据结构

```rust
pub enum RngError {
    RdRandUnsupported,
    RdSeedUnsupported,
    InsufficientEntropy,
    NotInitialized,
    FipsBlocked,       // R140-6 FIX：FIPS 状态 Failed 时阻止所有加密操作
    ReseedRequired,
}

// ChaCha20 CSPRNG（R141-8 FIX：限制为 pub(crate)，不公开导出）
pub(crate) struct ChaCha20Rng {
    state: [u32; 16],
    buffer: [u8; 64],
    available: usize,
    bytes_generated: u64,
}
```

全局 RNG 实例：`static GLOBAL_RNG: Mutex<Option<ChaCha20Rng>>`

#### 3.5.3 核心函数

| 函数 | 可见性 | 功能 |
|------|--------|------|
| `init_global()` | pub | 初始化全局 CSPRNG：32 字节密钥 + 12 字节 nonce + 额外 16 字节混入 |
| `fill_random(out)` | pub | FIPS 门控后从全局 CSPRNG 填充随机字节（R108-3 FIX：4KB 分块释放锁） |
| `try_fill_random(out)` | pub | 非阻塞版本：尝试 CSPRNG，失败则直接读硬件熵（panic 安全） |
| `chacha20_xor_keystream(key, nonce, buf)` | pub | R141-8 FIX：ChaCha20 加密 API，不暴露 ChaCha20Rng 类型 |
| `random_u64()` / `random_u32()` | pub(crate) | R149-5 FIX：限制为 crate 内部使用 |
| `random_range(max)` | pub(crate) | 无偏随机范围（拒绝采样法），限制为 crate 内部 |
| `rdrand64_early()` | pub(crate) | R141-8 FIX：早期启动熵源，限制为 crate 内部 |
| `rdrand_available()` | pub | 检查 RDRAND 是否可用（安全的公共查询） |
| `is_ready()` | pub | 检查 CSPRNG 是否就绪 |

**Fast-Key-Erasure（FKE）构造**（R109-3 FIX）：

每次 `refill()` 生成 64 字节 ChaCha20 输出，前 32 字节用于重新生成密钥（覆写 state[4..12]），后 32 字节作为可用输出。块计数器在每次 rekey 后重置为 0。这确保即使 RNG 状态在 refill 后被泄露，攻击者也无法回溯计算先前输出。

**R108-3 FIX：分块生成**：

`fill_random()` 按 `RNG_FILL_CHUNK_SIZE`（4096 字节）分块生成，每块生成后释放全局 spin-mutex，减少大缓冲区（如 `sys_getrandom` 的 1MB）导致的跨核竞争。

**FIPS 门控**（R140-6 FIX）：

```rust
match fips_state() {
    FipsState::Enabled => return fill_entropy(out),   // 回退到直接硬件熵
    FipsState::Failed => return Err(RngError::FipsBlocked),  // fail-closed
    FipsState::Disabled => {}  // 正常使用 ChaCha20
}
```

**安全措施**：
- `explicit_bzero()`：使用 `write_volatile` + `compiler_fence` 安全清零敏感数据
- 拒绝采样法避免模偏差
- 熵获取失败时不回退到零值（Z-6 修复）
- R109-3 FIX：`mix_in()` 后重置块计数器并强制 rekey

#### 3.5.4 模块间依赖

- 依赖 `crate::fips`：FIPS 状态查询
- 被 `kaslr.rs` 调用生成 KASLR/栈/mmap 随机值
- 被 `kptr.rs` 调用重播种指针混淆密钥
- 被 `lib.rs` 初始化并验证

---

### 3.6 kptr.rs — 内核指针混淆

**文件路径**: `kernel/security/kptr.rs`（282 行）

#### 3.6.1 功能概述

提供内核指针混淆机制，防止通过内核日志泄露内核地址从而绕过 KASLR。类似于 Linux 的 `%pK` 格式说明符。

核心思路：用 `KptrGuard` 包装指针，实现 `Display`/`Debug`/`Pointer` trait，在全局开关开启时输出混淆后的地址。

#### 3.6.2 核心数据结构

```rust
pub struct KptrGuard {
    addr: u64,
}

static KPTR_GUARD_ENABLED: AtomicBool;  // 全局开关
static KPTR_SECRET: AtomicU64;           // 混淆密钥
static KPTR_COUNTER: AtomicU64;          // 防重放计数器
```

#### 3.6.3 核心函数

| 函数 | 功能 |
|------|------|
| `KptrGuard::new(ptr)` / `from_addr(addr)` / `from_mut(ptr)` | 创建指针守卫 |
| `guarded_value()` | 根据全局开关返回混淆/原始值 |
| `obfuscated_value()` | 始终返回混淆值 |
| `raw()` | 返回原始地址（仅内核内部使用） |
| `is_kernel_address()` | 检查是否为高半区地址 |
| `init()` | 用 TSC 初始化密钥（早期启动） |
| `reseed_from_entropy()` | 用硬件 RNG 重播种（RNG 就绪后） |
| `reseed_with(secret)` | 用指定熵重播种（混入计数器） |
| `enable()` / `disable()` | 全局开关控制 |
| `format_ptr(ptr)` / `format_addr(addr)` | 便捷日志格式化 |

**混淆算法**：

```rust
fn obfuscate(addr: u64) -> u64 {
    let secret = ensure_secret();
    let mixed = addr ^ secret;
    mix64(mixed) ^ (addr.rotate_right(17) & 0xFFFF_0000_0000_0000)
}
```

使用 SplitMix64 混合函数提供良好的雪崩特性。

**密钥初始化策略**：
1. 早期启动：TSC（时间戳计数器）提供初始熵
2. RNG 就绪后：`reseed_from_entropy()` 用 CSPRNG 强熵重播种
3. 竞态安全：`ensure_secret()` 使用 `compare_exchange` 保证初始化原子性

---

### 3.7 spectre.rs — Spectre/Meltdown 缓解

**文件路径**: `kernel/security/spectre.rs`（618 行）

#### 3.7.1 功能概述

提供推测执行漏洞的检测与缓解，覆盖以下攻击面：

| 漏洞 | 缓解措施 | 状态 |
|------|----------|------|
| Spectre v1 | LFENCE 屏障 | 部分（syscall 路径） |
| Spectre v2 | IBRS/IBPB/STIBP | 支持则启用 |
| Meltdown | KPTI | H.3 基础设施就绪 |
| SSB (Spectre v4) | SSBD | 支持则启用 |
| RSB 下溢 | RSB 填充 | 完整实现（R108-1 FIX） |
| SWAPGS (CVE-2019-1125) | SWAPGS + LFENCE | 已在 syscall.rs 实现 |

#### 3.7.2 核心数据结构

```rust
pub struct MitigationStatus {
    pub ibrs_supported: bool,
    pub ibrs_enabled: bool,
    pub ibpb_supported: bool,
    pub stibp_supported: bool,
    pub stibp_enabled: bool,
    pub retpoline_compiler: bool,
    pub retpoline_required: bool,
    pub ssbd_supported: bool,
    pub ssbd_enabled: bool,
    pub swapgs_mitigated: bool,      // 始终 true
    pub rsb_stuffing_enabled: bool,  // 始终 true
}

pub struct VulnerabilityInfo {
    pub meltdown_susceptible: bool,
    pub spectre_v1_susceptible: bool,
    pub spectre_v2_susceptible: bool,
    pub ssb_susceptible: bool,
    pub mds_susceptible: bool,
}

pub enum SpectreError {
    Unsupported(&'static str),
    MsrUnavailable(&'static str),
    MsrIo(&'static str),
    RetpolineRequired,
}
```

注意：`security::init()` 对 `SpectreError::Unsupported` 不计入 violations（仅对其他错误类型计入）。

#### 3.7.3 核心函数

| 函数 | 功能 |
|------|------|
| `detect()` | CPUID 探测 CPU 缓解能力 |
| `init()` | BSP 初始化：启用 IBRS → STIBP → IBPB → SSBD |
| `init_cpu()` | AP 初始化（R65-25 FIX：per-CPU MSR 配置） |
| `enable_ibrs()` / `enable_stibp()` / `enable_ssbd()` | 通过 IA32_SPEC_CTRL MSR 启用各缓解 |
| `issue_ibpb()` / `try_ibpb()` | 通过 IA32_PRED_CMD MSR 发出 IBPB |
| `rsb_fill()` | RSB 填充：32 个 CALL + LFENCE 序列（R108-1 FIX：移除 `nostack` 选项） |
| `rsb_fill_on_context_switch()` | 上下文切换时的 RSB 填充包装 |
| `fill_rsb_if_needed()` | R109-5 FIX：标记为 `unsafe pub(crate)`，限制外部调用 |
| `get_vulnerabilities()` | 读取 IA32_ARCH_CAPABILITIES 获取漏洞信息 |
| `read_spec_ctrl()` | 读取当前 SPEC_CTRL MSR 值 |

**RSB 填充修复**（R108-1 FIX）：

移除了 `options(nostack)` 声明。`rsb_fill()` 中的 `call` 指令会将 32 个返回地址压入栈（256 字节），之后通过 `add rsp, 256` 恢复。声明 `nostack` 是不安全的，因为编译器可能假设 RSP 未被修改。

**MSR 常量**：

| MSR | 地址 | 用途 |
|-----|------|------|
| IA32_SPEC_CTRL | 0x48 | 推测控制（IBRS/STIBP/SSBD） |
| IA32_PRED_CMD | 0x49 | 预测器命令（IBPB，只写） |
| IA32_ARCH_CAPABILITIES | 0x10A | 架构能力（RDCL_NO/IBRS_ALL/RSBA/SSB_NO/MDS_NO） |

---

### 3.8 fips.rs — FIPS 模式状态机

**文件路径**: `kernel/security/fips.rs`（71 行）

#### 3.8.1 功能概述

R140-6 FIX 新增模块。存储 FIPS 模式的规范状态标志，使 `security` crate 中的加密原语（如 `rng::fill_random`）能够在不引入对 `compliance` crate 的循环依赖的情况下执行 FIPS 策略。

#### 3.8.2 核心数据结构

```rust
#[repr(u8)]
pub enum FipsState {
    Disabled = 0,  // FIPS 模式未启用
    Enabled = 1,   // FIPS 模式启用（sticky，直到重启）
    Failed = 2,    // FIPS 启用失败（如自测失败）或状态损坏
}

static FIPS_MODE: AtomicU8 = AtomicU8::new(FipsState::Disabled as u8);
```

#### 3.8.3 核心函数

| 函数 | 功能 |
|------|------|
| `fips_state()` | 读取当前 FIPS 状态（R94-3 FIX：未知值 fail-closed 返回 Failed） |
| `set_fips_state(state)` | 设置 FIPS 状态（R141-6 FIX：CAS 操作，单调转换，拒绝 Disabled→Disabled） |

**安全设计**：
- 单调转换：一旦设为 Enabled 或 Failed，不可恢复为 Disabled
- CAS 原子操作：防止并发写竞态（R141-6 FIX）
- Fail-closed：`fips_state()` 对未知/损坏的原子值返回 `Failed`

---

### 3.9 tests.rs — 运行时安全自测框架

**文件路径**: `kernel/security/tests.rs`（426 行）

#### 3.9.1 功能概述

提供轻量级运行时安全测试框架，可在启动期或运行期验证系统安全态势。通过 `SecurityTest` trait 定义测试接口，内置 5 项安全测试。

#### 3.9.2 核心数据结构

```rust
pub enum TestResult { Pass, Warning(&'static str), Fail(&'static str) }

pub struct TestOutcome { pub name: &'static str, pub result: TestResult }

pub struct TestReport {
    pub passed: usize,
    pub failed: usize,
    pub warnings: usize,
    pub outcomes: Vec<TestOutcome>,
}

pub struct TestContext { pub phys_offset: VirtAddr }

pub trait SecurityTest {
    fn name(&self) -> &'static str;
    fn run(&self, ctx: &TestContext) -> TestResult;
    fn description(&self) -> &'static str { "Security validation test" }
}
```

#### 3.9.3 内置测试

| 测试 | 功能 | 验证内容 |
|------|------|----------|
| `QuickValidationTest` | 快速 W^X 检查 | 调用 `quick_check()` |
| `WxorxFullValidationTest` | 完整 W^X 验证 | 调用 `wxorx::validate_active()` |
| `RngEntropyTest` | RNG 健康检查 | 全零检测、重复值检测、顺序模式检测、位分布检查 |
| `KptrGuardTest` | 指针混淆验证 | 混淆有效性、一致性检查 |
| `SpectreStatusTest` | Spectre 缓解状态 | 检查 `hardened()` 状态 |

#### 3.9.4 核心函数

| 函数 | 功能 |
|------|------|
| `run_security_tests(ctx)` | 执行所有内置测试，返回聚合报告 |
| `run_test(name, ctx)` | 按名称执行单项测试 |
| `self_test()` | 测试框架自检 |

**`security_assert!` 宏**：仅在 debug 模式下触发 panic 的安全断言。

---

## 4. 模块间依赖关系

### 4.1 初始化调度依赖

```text
lib.rs::init()
  │
  ├─[0] kaslr::set_kaslr_fail_closed()  ← R102-11: 传播 fail-closed 策略
  │
  ├─[1] kptr::init() + kptr::enable()     ← 最先执行，保护后续日志
  │
  ├─[2] memory_hardening::cleanup_identity_map()
  │       └─ mm::page_table (递归映射)
  │       └─ mm::memory::FrameAllocator
  │       └─ drivers::framebuffer (MMIO 检测)
  │       └─ cpu_local::num_online_cpus() (pre-SMP 断言)
  │
  ├─[3] memory_hardening::enforce_nx_for_kernel()
  │       └─ mm::page_table::ensure_pte_range
  │       └─ cpu_local::num_online_cpus() (pre-SMP 断言)
  │
  ├─[4] wxorx::validate_active()
  │       └─ mm::page_table::RECURSIVE_INDEX
  │       └─ x86_64::Cr3
  │
  ├─[5] rng::init_global()
  │       └─ fips::fips_state() (FIPS 门控)
  │       └─ kptr::reseed_from_entropy()   ← 用强熵重播种
  │
  ├─[6] spectre::init()
  │       └─ x86_64::Msr (IA32_SPEC_CTRL/PRED_CMD)
  │
  └─[7] tests::run_security_tests()
          └─ 调用上述所有子模块
```

### 4.2 代码级依赖

```text
lib.rs ──→ kptr, memory_hardening, wxorx, rng, spectre, kaslr, fips, tests
kptr ──→ rng (重播种)
kaslr ──→ rng (随机值生成), mm::memory (堆基址), cpu_local (per-CPU KPTI)
memory_hardening ──→ mm::page_table, mm::memory, drivers::framebuffer, cpu_local, mm::tlb_shootdown
wxorx ──→ mm::page_table
rng ──→ fips (FIPS 状态门控)
spectre ──→ x86_64::Msr
fips ──→ (无外部依赖)
tests ──→ wxorx, rng, kptr, spectre, lib::quick_check
```

### 4.3 隐含前置条件

- 页表拓扑：PML4[0]=低半区、PML4[510]=递归槽、PML4[511]/PDPT[510]=内核高半区
- 链接器符号：`kernel_start/end`、`text_start/end`、`rodata_start/end`、`data_start/end`、`bss_start/end`
- 硬件要求：x86_64 CPU，支持 RDRAND（RNG 子系统达到 `rng_ready` 的前提），RDSEED/IBRS/STIBP/SSBD/PCID 可选
- SMP 时序：`cleanup_identity_map` 和 `enforce_nx_for_kernel` 必须在 SMP bring-up 之前调用

---

## 5. 模块整体评估

### 5.1 架构优势

1. **初始化流程清晰可控**：配置→执行→报告的闭环设计，每步都有明确的输入输出和错误处理
2. **页表权限加固与验证闭环**：先通过 `memory_hardening` 设置权限，再通过 `wxorx` 验证，形成"加固→验证"的完整闭环
3. **KPTI per-CPU seqlock 设计**（H.3 SMP FIX）：解决了原全局原子变量在 SMP 下互相覆盖的问题，per-CPU 存储确保每个 CPU 独立管理 KPTI 上下文
4. **日志安全意识**：通过 `#[cfg(debug_assertions)]` 门控 KASLR 滑动值、堆基址等敏感信息的日志输出
5. **panic 安全 RNG**：`try_fill_random()` 提供非阻塞的硬件熵回退，确保 kdump 等 panic 路径也能获取熵
6. **W^X 基线保护**：`mark_all_nx()` 先将所有页标记为 NX 再按段精调，确保遗漏页面默认不可执行
7. **SMP 感知**：`spectre::init_cpu()` 为 AP 提供独立的 MSR 配置路径；内存加固函数强制 pre-SMP 断言
8. **FIPS 合规门控**：ChaCha20（非 FIPS 认证算法）在 FIPS 模式下自动回退到直接硬件熵
9. **Fast-Key-Erasure**：RNG 的 FKE 构造提供前向安全性
10. **分块 RNG 生成**：4KB 分块释放锁，减少大缓冲区的跨核竞争
11. **RSB 填充修复**：R108-1 正确移除了 `options(nostack)`，消除了编译器可能做出的错误栈帧假设
12. **Partial KASLR 三特性**：堆、内核栈、用户空间 mmap 均实现随机化，提供纵深防御

### 5.2 改进路线建议

| 优先级 | 改进项 | 影响范围 |
|--------|--------|----------|
| P1 | 补齐 MMIO 高半区映射（`protect_mmio_regions` 仍为空操作） | memory_hardening.rs |
| P1 | 验证 EFER.NXE 和 CR0.WP 已启用 | memory_hardening.rs, wxorx.rs |
| P2 | 解耦 `strict_wxorx` 与 `KASLR_FAIL_CLOSED` | lib.rs, kaslr.rs |
| P2 | RNG 迁移到 per-CPU 或无锁快路径 | rng.rs |
| P2 | kptr 混淆算法升级为带密钥 PRF（如 SipHash） | kptr.rs |
| P3 | W^X 验证记录违规位置样本 | wxorx.rs |
| P3 | 补充 SMEP/SMAP/UMIP 状态测试 | tests.rs |
| P3 | 多源熵混合（时序抖动、设备中断） | rng.rs |

### 5.3 总结

Zero-OS 的安全加固模块展现了成熟的安全工程思维：纵深防御、安全失败、最小权限的设计原则贯穿始终。模块化设计清晰，初始化流程有序，关键路径（如 KPTI 上下文读取）采用了正确的 per-CPU 无锁并发方案。

自初始版本以来的显著进展包括：Text KASLR 完整实现（H.2，引导加载器 PIE 支持）、KPTI per-CPU 迁移（H.3 SMP FIX）、RNG Fast-Key-Erasure 构造（R109-3）、FIPS 模式集成（R140-6）、RNG 分块生成（R108-3）、RSB 填充安全修复（R108-1）、Partial KASLR 三特性完整实现（堆/栈/mmap），以及多轮安全审查驱动的 API 边界收紧（R141-8/R149-5 FIPS 边界强制）。
