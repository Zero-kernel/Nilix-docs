# 内存管理 (mm) 模块分析文档

> **模块路径**: `kernel/mm/`
> **编程语言**: Rust (nightly, `no_std` + `alloc`)
> **目标架构**: x86_64 (UEFI 启动)
> **分析日期**: 2026-05-13
> **文档版本**: 基于当前 `main` 分支最新提交

---

## 1. 模块概述

mm 模块是 Zero-OS 微内核的内存管理子系统，负责从物理页帧分配到虚拟地址映射的完整内存管理链路。该模块遵循 Zero-OS "安全 > 效率 > 速度" 的设计原则，在多个层面实施了安全加固措施。

### 1.1 核心职责

- **物理内存管理**: 基于 Buddy 算法的物理页帧分配与回收
- **虚拟内存管理**: x86_64 四级页表的创建、映射、取消映射与标志位管理
- **堆内存管理**: 内核堆的初始化与 KASLR 随机化
- **页缓存**: 文件系统页缓存的全局管理、LRU 淘汰与脏页回写
- **TLB 一致性**: SMP 环境下的跨 CPU TLB 失效（shootdown）机制
- **OOM 处理**: 内存不足时的页缓存回收与进程终止策略
- **DMA 缓冲区**: 安全的 DMA 内存分配与 IOMMU 映射管理

### 1.2 设计原则

- **安全优先**: fail-closed 策略（TLB shootdown 失败则 panic）、DMA 缓冲区 scrub、IOMMU 映射失败时泄漏而非复用
- **依赖反转**: 通过函数指针注入避免 crate 循环依赖（IPI sender、IOMMU ops、OOM callbacks）
- **RAII 生命周期**: `DmaBuffer::drop()` 和 `PageCacheEntry::drop()` 自动回收物理页
- **两阶段操作**: 先收集候选（持锁），再执行慢操作（释锁），减少锁持有时间
- **日志安全**: R148-8 地址日志门控（`#[cfg(debug_assertions)]`），防止 release 构建泄漏 KASLR 信息

### 1.3 依赖关系

```
mm (本模块)
├── drivers      — 日志输出宏 (println!)
├── klog         — 分级日志宏 (klog!, klog_always!, kprintln!)
├── cpu_local    — Per-CPU 数据结构、TLB 邮箱、LAPIC ID 查询
├── tlb_ops      — INVPCID 指令封装
├── x86_64       — 页表结构、CR3/CR4 寄存器、TLB flush 指令
├── spin         — 自旋锁 (Mutex, RwLock)
├── lazy_static  — 延迟初始化全局单例
├── linked_list_allocator — 内核堆分配器
└── bit-vec      — 位图数据结构
```

---

## 2. 文件结构

| 文件 | 行数 | 功能 |
|------|------|------|
| `lib.rs` | 41 | 模块根，声明子模块并 re-export 公共 API |
| `Cargo.toml` | 18 | crate 配置与依赖声明 |
| `memory.rs` | 707 | BootInfo 解析、堆初始化（含 KASLR）、FrameAllocator 适配 |
| `buddy_allocator.rs` | 532 | Buddy 物理页帧分配器，含 order 追踪与自测 |
| `page_table.rs` | 872 | x86_64 页表管理、递归映射、2MB 降级、MMIO 映射 |
| `page_cache.rs` | 1049 | 全局页缓存、LRU 淘汰、脏页回写、内存压力处理 |
| `tlb_shootdown.rs` | 1313 | IPI-based TLB shootdown、CR3 追踪、PCID/INVPCID 支持 |
| `oom_killer.rs` | 289 | OOM killer、进程评分、缓存回收、审计事件 |
| `dma.rs` | 488 | DMA 缓冲区分配、IOMMU 映射、scrub、fail-safe 策略 |

---

## 3. 逐文件分析

### 3.1 lib.rs — 模块入口（41 行）

#### 功能概述

模块根文件，声明 `#![no_std]` 和 `#![feature(abi_x86_interrupt)]` 属性，引入 `alloc`、`drivers`、`klog` 外部 crate，声明所有子模块并通过 `pub use` 导出公共 API。

#### 核心结构

无独立数据结构。仅包含模块声明和 re-export。

导出的公共 API 涵盖：
- `memory`: `BootInfo`, `FrameAllocator`, `MemoryMapInfo`
- `oom_killer`: `get_stats`, `on_allocation_failure`, `register_audit_callback`, `register_callbacks`, `OomProcessInfo`, `OomStats`
- `page_cache`: `find_or_create_page`, `init`, `read_page`, `reclaim_pages`, `sync_inode`, `writeback_dirty_pages`, `writeback_page`, `AddressSpace`, `GlobalPageCache`, `InodeId`, `MemoryPressureHandler`, `PageCacheEntry`, `PageCacheStats`, `PageIndex`, `PageState`, `WritebackStats`, `PAGE_CACHE`, `PAGE_SIZE`, `PRESSURE_HANDLER`
- `page_table`: `map_mmio`, `phys_to_virt`, `with_current_manager`, `MapError`, `PageTableManager`, `UnmapError`, `UpdateFlagsError`, `PHYSICAL_MEMORY_OFFSET`
- `tlb_shootdown`: `flush_current_as_all`, `flush_current_as_page`, `flush_current_as_range`, `get_stats`

#### 核心函数

- `init()`: 调用 `klog_always!` 打印初始化消息，实际初始化由各子模块独立完成。

#### 模块间依赖

- 依赖 `drivers` 和 `klog` crate 的日志宏
- 导出所有子模块的公共 API 供内核其他模块使用

---

### 3.2 Cargo.toml — 依赖配置（18 行）

#### 功能概述

定义 mm crate 的元数据和依赖关系。

#### 关键依赖分析

- `x86_64 = "0.15"`: 提供页表结构、物理/虚拟地址类型、CR3/CR4 操作
- `spin = "0.9"`: 自旋锁实现，适用于 `no_std` 内核环境
- `lazy_static` (spin_no_std): 延迟初始化全局变量
- `linked_list_allocator = "0.10"`: 内核堆分配器（`#[global_allocator]`）
- `bit-vec = "0.6"` (no default-features): Buddy 分配器的位图
- `cpu_local` / `tlb_ops` / `klog`: 内部 workspace crate

---

### 3.3 memory.rs — 内存初始化与堆管理（707 行）

#### 功能概述

负责内核启动阶段的内存初始化，包括：
1. 解析 UEFI BootInfo 中的内存映射
2. 内核堆的初始化（含 Partial KASLR 堆地址随机化）
3. 物理内存区域选择与 Buddy 分配器初始化
4. `FrameAllocator` 适配器（桥接 Buddy 分配器与 x86_64 crate 的 trait）

#### 核心数据结构

```rust
// BootInfo — Bootloader 传入的启动信息
#[repr(C)]
pub struct BootInfo {
    pub memory_map: MemoryMapInfo,    // UEFI 内存映射
    pub framebuffer: FramebufferInfo, // GOP 帧缓冲区
    pub kaslr_slide: u64,             // KASLR 偏移
    pub rsdp_address: u64,            // ACPI RSDP 地址
    pub cmdline_len: usize,           // P1-1: 命令行长度
    pub cmdline: [u8; 256],           // P1-1: 命令行缓冲区
}

// MemoryMapInfo — UEFI 内存映射描述
#[repr(C)]
pub struct MemoryMapInfo {
    pub buffer: u64,           // 内存映射缓冲区地址
    pub size: usize,           // 缓冲区总大小
    pub descriptor_size: usize,// 单个描述符大小
    pub descriptor_version: u32,
}

// FramebufferInfo — GOP 帧缓冲区信息
#[repr(C)]
pub struct FramebufferInfo {
    pub base: u64,             // 帧缓冲区物理地址
    pub size: usize,           // 帧缓冲区大小
    pub width: u32,            // 水平分辨率
    pub height: u32,           // 垂直分辨率
    pub stride: u32,           // 每行字节数
    pub pixel_format: PixelFormat, // 像素格式 (Rgb/Bgr/Unknown)
}

// FrameAllocator — 物理帧分配器适配器
pub struct FrameAllocator;  // 零大小类型，委托给全局 Buddy 分配器

// MemoryStats — 内存统计信息
pub struct MemoryStats {
    pub total_physical_pages: usize,
    pub free_physical_pages: usize,
    pub used_physical_pages: usize,
    pub fragmentation_percent: u32,
    pub heap_used_bytes: usize,
    pub heap_total_bytes: usize,
}
```

#### 核心函数分析

**`init_with_bootinfo(boot_info: &BootInfo)`**
- 主初始化入口，接收 Bootloader 传递的 BootInfo
- 流程：
  1. 调用 `select_heap_base_from_bootinfo()` 尝试从 UEFI 内存映射中选择经过验证的堆地址
  2. 若失败，回退到 `select_heap_base()` 使用未验证的随机化
  3. 调用 `init_heap_allocator_at()` 初始化堆
  4. R148-8 修复：地址日志使用 `#[cfg(debug_assertions)]` 门控，防止 release 构建泄漏 KASLR 信息
  5. 调用 `select_region_from_bootinfo()` 选择最大的可用物理内存区域
  6. 初始化 Buddy 分配器
  7. debug 构建下运行 `buddy_allocator::run_self_test()`

**`select_heap_base_from_bootinfo(boot_info: &BootInfo) -> Option<(usize, bool)>`**
- R72-4 修复：堆地址选择时验证 UEFI 内存映射
- 随机化窗口：`[0xffffffff80400000, 0xffffffff90000000)`，2MB 对齐
- 算法：RDRAND 获取熵 -> 随机选择 slot -> 将候选 heap 虚拟地址换算为物理地址（减去 `PHYSICAL_MEMORY_OFFSET`）-> 调用 `heap_range_usable()` 验证候选区间完全落在可用 UEFI 区域（`EFI_CONVENTIONAL_MEMORY` / `EFI_BOOT_SERVICES_CODE` / `EFI_BOOT_SERVICES_DATA`），且满足 `MIN_SAFE_ADDRESS..HIGH_HALF_MAP_LIMIT` 直映访问约束 -> 若随机 slot 不可用则遍历所有 slot 寻找有效位置

**`rdrand64_early() -> Option<u64>`**
- 早期 RDRAND 访问，不依赖 ChaCha20 CSPRNG
- 最多重试 32 次，使用内联汇编直接调用 `rdrand` 指令
- 使用 `nomem, nostack` 选项

**`rdrand_supported_early() -> bool`**
- R72-4 修复：正确处理 CPUID 的 `rbx` 破坏（push/pop 保存），不使用 `nostack`/`nomem`
- 查询 CPUID.01H:ECX.RDRAND[bit 30]

**`select_region_from_bootinfo(boot_info: &BootInfo) -> Option<(u64, usize)>`**
- 遍历 UEFI 内存映射，选择最大的可用区域
- 过滤条件：仅 `EFI_CONVENTIONAL_MEMORY`/`EFI_BOOT_SERVICES_*`，地址 >= 1MB，< 1GB（直映限制）
- 跨越 1GB 边界的区域截断到 1GB
- 最大使用量上限：256MB

**`init()` — 后备初始化函数**
- 无 BootInfo 时使用硬编码后备区域（`FALLBACK_PHYS_MEM_START = 0x10000000`，64MB）

#### 关键常量

| 常量 | 值 | 说明 |
|------|------|------|
| `HEAP_DEFAULT_BASE` | `0xffffffff80400000` | 堆默认基地址 |
| `HEAP_WINDOW_END` | `0xffffffff90000000` | 随机化窗口上界 |
| `HEAP_ALIGNMENT` | 2MB | 大页兼容对齐 |
| `HEAP_SIZE` | 1MB | 堆大小 |
| `HIGH_HALF_MAP_LIMIT` | 1GB | 直映上限 |
| `MIN_SAFE_ADDRESS` | 1MB | 最低安全地址 |
| `MIN_USABLE_REGION` | 2MB | 最小可用区域 |

#### 模块间依赖

- 上游：Bootloader 提供 `BootInfo`
- 下游：`buddy_allocator::init_buddy_allocator()` 初始化物理页分配
- 全局：`#[global_allocator]` 使用 `linked_list_allocator::LockedHeap`

#### 当前不足与改进建议

1. **堆大小固定 1MB**: `HEAP_SIZE = 1024 * 1024`，对于复杂内核操作（大量 `Vec`/`BTreeMap`）可能不足，且无法动态扩展。
2. **直映窗口限制 1GB**: `HIGH_HALF_MAP_LIMIT = 1GB`，Bootloader 仅映射物理 0-1GB 到高半区。超过 1GB 的物理内存完全不可用。
3. **UEFI 内存映射信任边界**: `select_region_from_bootinfo()` / `heap_range_usable()` 直接解引用 `MemoryMapInfo.buffer` 指针读取描述符（`unsafe`），虽已检查 `descriptor_size == 0` / `buffer == 0` / `size == 0`，但仍缺少对 `descriptor_size >= size_of::<EfiMemoryDescriptor>()` 及 `buffer` 对齐的校验。
4. **堆随机化熵源单一**: 仅依赖 RDRAND，若 CPU 不支持则回退到固定地址。随机化窗口约 126 个 slot（约 7 bit 熵）。

---

### 3.4 buddy_allocator.rs — Buddy 物理页帧分配器（532 行）

#### 功能概述

实现经典的 Buddy 内存分配算法，管理物理页帧的分配与回收。支持 2^0 到 2^10 阶（4KB 到 4MB）的连续物理内存分配，并在分配失败时触发 OOM killer。

#### 核心数据结构

```rust
pub struct BuddyAllocator {
    free_lists: [Vec<usize>; MAX_ORDER],  // 每阶的空闲块链表（块索引）
    bitmap: BitVec,                        // 页面分配状态位图
    alloc_order: Vec<u8>,                  // R74-4: 每页的分配 order 记录
    split_bitmap: BitVec,                  // 块分割状态位图
    base_addr: PhysAddr,                   // 管理区域起始物理地址
    total_pages: usize,                    // 总页数
    free_pages: usize,                     // 空闲页数
}
// MAX_ORDER = 11, PAGE_SIZE = 4096
// order 范围 [0, MAX_ORDER)，最大连续分配 = 2^10 * 4KB = 4MB
```

#### 核心函数分析

**`alloc_pages(&mut self, order: usize) -> Option<PhysFrame>`**
- 从 `order` 开始向上搜索空闲块
- 找到后调用 `split_block()` 逐级分割到目标大小
- 调用 `mark_allocated()` 标记位图，`record_alloc_order()` 记录分配 order
- 时间复杂度：O(MAX_ORDER) 查找 + O(order_diff) 分割

**`free_pages(&mut self, frame: PhysFrame, order: usize)`**
- R74-4 安全检查链：
  1. 地址范围验证（在管理区域内）
  2. 对齐验证（`block_idx & (pages - 1) == 0`）
  3. `block_idx + pages > total_pages` 越界检查
  4. 分配 order 匹配验证（`alloc_order[block_idx]` 编码为 `order+1`，0 表示空闲）
  5. 双重释放检测（位图检查所有页面均为已分配状态）
- 通过所有检查后标记为空闲，调用 `merge_blocks()` 尝试合并

**`merge_blocks(&mut self, block_idx: usize, order: usize)`**
- 循环尝试与 buddy 合并，直到无法继续或达到 MAX_ORDER-1
- buddy 索引计算：`block_idx ^ (1 << order)`（XOR 翻转对应位）
- R152-15 修复：合并前验证 buddy 在对应阶的 `free_lists[order]` 中存在（`iter().position()`），若不存在则中止合并，防止重叠空闲块
- 从空闲链表中移除 buddy，合并后加入更高阶空闲链表

**`is_buddy_free(&self, buddy_idx: usize, order: usize) -> bool`**
- R104-1 修复：检查完整块范围 `buddy_idx + pages <= total_pages`（而非仅起始索引），使用 `checked_add` 防溢出
- 逐页检查 bitmap 确认所有页面未分配

**`alloc_physical_pages(count: usize) -> Option<PhysFrame>`** (全局函数)
- R152-17 修复：使用 `checked_next_power_of_two()` 替代 `next_power_of_two()`，防止大值溢出到 0
- count=0 直接返回 None
- 第一次分配失败后触发 `oom_killer::on_allocation_failure()`
- OOM 处理后重试一次

**`free_physical_pages(frame: PhysFrame, count: usize)`** (全局函数)
- R100-4 修复：count=0 为 no-op（`0.next_power_of_two() == 1` 会错误释放 1 页）
- 使用 `checked_next_power_of_two()` 防溢出

**`init_buddy_allocator(base_addr: PhysAddr, size: usize)`**
- R132-3 修复：使用 `kprintln!`（debug-only）输出物理基地址，防止 release 构建泄漏信息

#### 模块间依赖

- 被 `memory::FrameAllocator` 包装，提供 `x86_64::FrameAllocator` trait 实现
- 被 `page_cache::PageCacheEntry::drop()` 调用释放物理帧
- 被 `dma::alloc_dma_buffer()` 调用分配 DMA 内存
- 分配失败时调用 `oom_killer::on_allocation_failure()`

#### 当前不足与改进建议

1. **空闲链表使用 `Vec<usize>`**: 查找 buddy 时需要线性扫描（`iter().position()`），时间复杂度 O(n)。应使用链表或哈希集合实现 O(1) 查找和删除。
2. **全局单锁**: `BUDDY_ALLOCATOR: Mutex<Option<BuddyAllocator>>` 是全局瓶颈。在 SMP 环境下，所有 CPU 的物理页分配/释放都竞争同一把锁。
3. **碎片率计算不精确**: `calculate_fragmentation()` 仅考虑最大空闲块与总空闲页的比值，不反映实际碎片分布。
4. **自测代码与生产代码混合**: `run_self_test()` 和自定义 `assert!` 宏在同一文件中，应分离到测试模块。

---

### 3.5 page_table.rs — 页表管理器（872 行）

#### 功能概述

提供 x86_64 四级页表的完整管理功能，包括：
1. 虚拟页到物理帧的映射/取消映射
2. 页表标志位修改
3. 递归页表访问（通过 PML4[510] 自映射）
4. 2MB 大页到 4KB 页的降级（用于 MMIO 隔离和 W^X 强制）
5. MMIO 区域映射（RW+NX+uncached）
6. R141-9: 非 PRESENT PTE 的物理帧回收
7. 全局页表锁保证 SMP 安全

#### 核心数据结构

```rust
// 页表管理器 — 封装 x86_64 crate 的 OffsetPageTable
pub struct PageTableManager {
    mapper: OffsetPageTable<'static>,
}

// 错误类型
pub enum MapError {
    FrameAllocationFailed,
    ParentEntryHugePage,
    PageAlreadyMapped,
    InvalidRange,  // R32-MM-2: 溢出保护
}

pub enum UnmapError {
    PageNotMapped,
    ParentEntryHugePage,
    InvalidFrameAddress,
    InvalidRange,  // R35-MM-2: 溢出保护
}

pub enum UpdateFlagsError {
    PageNotMapped,
    ParentEntryHugePage,
}

// 关键常量
pub const PHYSICAL_MEMORY_OFFSET: u64 = 0xffff_ffff_8000_0000;
pub const RECURSIVE_INDEX: usize = 510;
pub const VGA_VIRT_ADDR: u64 = PHYSICAL_MEMORY_OFFSET + 0x000b_8000;
pub const APIC_VIRT_ADDR: u64 = 0xffff_ffff_fee0_0000;
```

#### 核心函数分析

**`with_current_manager(physical_memory_offset, f) -> T`** (unsafe)
- 基于当前 CR3 构建临时 `PageTableManager` 并在闭包中使用
- R32-MM-1 修复：执行期间禁用中断，防止 CR3 切换
- R67-5 修复：获取全局 `PT_LOCK`，防止多 CPU 并发修改
- 参数 `physical_memory_offset` 仅为接口兼容保留，内部固定使用 `get_phys_offset()`

**`with_pt_lock(f) -> T`**
- R67-5 修复：公共辅助函数，获取全局 `PT_LOCK` 后执行闭包

**`take_nonpresent_leaf_frame(&mut self, page: Page) -> Option<PhysFrame>`**
- R141-9 修复：回收 `mprotect(PROT_NONE)` 清除 PRESENT 位后仍编码在 PTE 中的物理帧
- 从 `mapper.level_4_table_mut()` 开始手动遍历 PML4->PDPT->PD->PT
- 若叶 PTE 非 PRESENT 且物理地址非零，提取帧并清除 PTE
- 解决了 `unmap_page()` 对非 PRESENT 条目返回 `PageNotMapped` 导致 `sys_munmap` 跳过帧释放的内存泄漏

**`map_page(&mut self, page, frame, flags, allocator) -> Result<(), MapError>`**
- 委托给 `OffsetPageTable::map_to()`，映射后自动 flush 本地 TLB

**`map_range(&mut self, start_virt, start_phys, size, flags, allocator)`**
- R32-MM-2 修复：所有偏移计算使用 `checked_add`/`checked_mul` 防止溢出
- R34-MM-1 修复：失败时回滚已映射的页面，防止孤立映射

**`with_active_level_4_table(f) -> T`** (unsafe)
- R152-16 修复：断言 `cpu_local::num_online_cpus() <= 1`，仅允许 SMP 启动前调用
- R153-10 注释：绕过 PT_LOCK，限制为单 CPU 早期启动

**`split_2m_entry(pd_entry, allocator) -> Result<&'static mut PageTable, MapError>`** (unsafe)
- 将 2MB 大页 PD 条目拆分为 512 个 4KB PTE
- 分配新页表帧，复制原始标志（移除 HUGE_PAGE/DIRTY）

**`ensure_pte_level(page, allocator) -> Result<(), MapError>`** (unsafe)
- 确保虚拟页由 4KB PTE 支持
- X-7 & R68-2 修复：大页拆分后对所有 CPU 执行 2MB 范围的 TLB shootdown

**`map_mmio(virt, phys, size, allocator) -> Result<(), MapError>`** (unsafe)
- R67-5 修复：顶层获取一次 `PT_LOCK`，内部使用 unlocked 版本避免死锁
- 流程：ensure PTE 粒度 -> 逐页映射（已映射则更新标志）-> 跨 CPU TLB shootdown
- MMIO 标志：`PRESENT | WRITABLE | NO_EXECUTE | NO_CACHE | WRITE_THROUGH`
- 在 `interrupts::without_interrupts` 块中执行整个 PTE 分割和映射

#### 递归页表访问函数

- `recursive_pml4()` / `recursive_pdpt(idx)` / `recursive_pd(idx, idx)` / `recursive_pt(idx, idx, idx)`
- 通过 PML4[510] 自映射实现对任意物理页表帧的虚拟地址访问
- 地址计算基于固定的递归映射基地址常量

#### 模块间依赖

- 被 `kernel_core` 的进程管理、COW 处理、mmap/munmap 调用
- 被 `security` 模块的 W^X 验证和页表遍历调用
- 调用 `tlb_shootdown::flush_current_as_range()` 进行跨 CPU TLB 失效
- 依赖 `cpu_local::num_online_cpus()` 进行 SMP 安全断言
- 依赖 `memory::FrameAllocator` 分配中间页表帧

#### 当前不足与改进建议

1. **全局 `PT_LOCK` 是性能瓶颈**: 所有页表修改（包括不同地址空间）都竞争同一把锁。应演进为 per-address-space 锁。
2. **`with_current_manager()` 关中断范围过大**: 整个闭包执行期间中断被禁用。
3. **`PAGE_TABLE_MANAGER` 全局实例与 `with_current_manager()` 并存**: 后者每次从 CR3 重建管理器，前者可能过时。
4. **`map_range` 回滚不释放中间页表帧**: 失败回滚时仅 unmap 已映射的页面，中间页表帧不会被回收。
5. **`split_2m_entry` 缺少对 1GB 大页的支持**: 仅处理 2MB 大页降级。

---

### 3.6 page_cache.rs — 全局页缓存（1049 行）

#### 功能概述

实现文件系统的全局页缓存，提供：
1. 基于哈希桶的全局页面查找（256 个桶，每桶一个 `RwLock<BTreeMap>`）
2. LRU 链表管理页面淘汰顺序
3. 脏页追踪与回写机制
4. Per-inode `AddressSpace` 索引
5. 内存压力处理接口

#### 核心数据结构

```rust
// 页缓存条目
pub struct PageCacheEntry {
    pub pfn: u64,              // 物理帧号
    pub inode_id: InodeId,     // 所属 inode
    pub index: PageIndex,      // inode 内页索引
    dirty: AtomicBool,         // 脏标志
    refcount: AtomicU32,       // 引用计数（用于显式 pin）
    state: AtomicU32,          // 页状态 (Invalid/Uptodate/Reading/Writeback/Error)
    io_lock: Mutex<()>,        // I/O 序列化锁
    lru_index: AtomicU64,      // LRU 链表中的位置索引
}

// 页状态枚举
#[repr(u8)]
pub enum PageState {
    Invalid = 0,
    Uptodate = 1,
    Reading = 2,
    Writeback = 3,
    Error = 4,
}

// LRU 链表（数组实现的双向链表 + 空闲链表）
struct LruList {
    entries: Vec<LruEntry>,  // 预分配的条目数组
    head: u64,               // 最近使用（MRU）
    tail: u64,               // 最久未用（LRU）
    count: usize,
    free_head: u64,          // 空闲条目链表头
}

// 全局页缓存
pub struct GlobalPageCache {
    buckets: Vec<RwLock<BTreeMap<(InodeId, PageIndex), Arc<PageCacheEntry>>>>,
    lru: Mutex<LruList>,
    nr_pages: AtomicU64,
    nr_dirty: AtomicU64,
    max_pages: u64,  // 默认 16384 (64MB)
}

// Per-inode 地址空间
pub struct AddressSpace {
    inode_id: InodeId,
    pages: RwLock<BTreeMap<PageIndex, Arc<PageCacheEntry>>>,
    nr_pages: AtomicU64,
    nr_dirty: AtomicU64,
}
```

#### 核心函数分析

**`find_or_create_page(inode_id, index, alloc_pfn) -> Option<Arc<PageCacheEntry>>`**
- 先查找缓存，命中则返回
- 未命中则分配新物理页，创建 `PageCacheEntry`，尝试加入缓存
- 竞态处理：若另一线程先插入，丢弃本地页（Drop 自动释放物理帧）

**`read_page(inode_id, index, alloc_pfn, read_from_disk) -> Option<Arc<PageCacheEntry>>`**
- 调用 `find_or_create_page` 获取页面
- 若页面非 Uptodate，获取 `io_lock` 后执行磁盘读取
- 双重检查：获取锁后再次检查状态，避免重复 I/O

**`GlobalPageCache::add_to_cache(inode_id, index, page)`**
- R28-3 修复：LRU 满时回滚 bucket 插入，防止孤立页面
- R42-4 修复：移除冗余的 `page.get()` 调用，Arc clone 已提供引用计数

**`GlobalPageCache::shrink(&self, nr_to_reclaim) -> usize`**
- R42-5 修复：持续扫描 LRU 而非在首个不可回收页停止
- R42-4 修复：使用 `PageCacheEntry::can_reclaim()` 静态方法（基于 `Arc::strong_count == 2`）
- 两阶段操作：持 LRU 锁弹出尾部候选 -> 释放 LRU 锁检查可回收性 -> 不可回收则放回 LRU 头部
- 锁序：bucket -> LRU（与 find/add 一致）

**`writeback_dirty_pages(max_pages, write_fn) -> WritebackStats`**
- 两阶段：持 LRU 锁收集脏页列表 -> 释放锁后逐页回写
- 使用 `try_lock()` 跳过正在 I/O 的页面

**`PageCacheEntry::drop()`**
- R36 修复：释放物理帧回 Buddy 分配器，防止内存泄漏

#### 模块间依赖

- 被 VFS 层调用进行文件 I/O 缓存
- 调用 `buddy_allocator::free_physical_pages()` 释放物理帧
- 被 `oom_killer` 通过 `MemoryPressureHandler` trait 调用回收页面

#### 当前不足与改进建议

1. **LRU 全局单锁**: `lru: Mutex<LruList>` 是热路径瓶颈。
2. **`can_reclaim` 使用 `Arc::strong_count`**: 并发环境下可能不准确（TOCTOU）。
3. **哈希函数质量**: `hash_key()` 使用简单的乘法哈希。
4. **缺少异步回写**: `writeback_dirty_pages()` 是同步操作。
5. **内存压力处理不完整**: `PageCachePressureHandler::on_memory_pressure()` 仅回收干净页，无法触发同步回写。

---

### 3.7 tlb_shootdown.rs — TLB Shootdown 基础设施（1313 行）

#### 功能概述

提供 SMP 环境下的跨 CPU TLB 失效机制，是内存管理安全性的关键组件。主要功能：
1. IPI-based TLB shootdown（向量 0xFE）
2. Per-address-space CR3 追踪（定向 shootdown 优化）
3. 队列化邮箱（支持批量请求，深度 4）
4. PCID/INVPCID 感知的 TLB flush
5. ACK 超时重试与 fail-closed 策略
6. P2-7: ShootdownCpuPin 防止 CPU 迁移
7. R106-5: 邮箱饱和时优雅降级到全 flush

#### 核心数据结构

```rust
// TLB shootdown 统计
pub struct TlbShootdownStats {
    pub full_flushes: u64,
    pub range_flushes: u64,
    pub pages_flushed: u64,
    pub coalesced_fallbacks: u64,     // R106-5: 邮箱饱和回退计数
}

// ShootdownCpuPin — P2-7: 防止 CPU 迁移的 RAII guard
struct ShootdownCpuPin {
    cpu_id: usize,
    _not_send: PhantomData<*const ()>, // !Send + !Sync
}

// 关键全局状态
static ONLINE_CPU_COUNT: AtomicU64;
static ONLINE_CPU_MASK: AtomicU64;
static NEXT_SHOOTDOWN_GEN: AtomicU64;
static TLB_IPI_SENDER: AtomicUsize;
static INVPCID_SUPPORTED: AtomicBool;

// Per-address-space 追踪
static CPU_ACTIVE_CR3: CpuLocal<AtomicU64>;
static ASID_CPU_MASKS: RwLock<BTreeMap<u64, u64>>;
static ASID_WRITER_PENDING: AtomicBool;
```

#### 核心函数分析

**`flush_current_as_all()`**
- P2-7 修复：使用 `ShootdownCpuPin` 固定 CPU，防止迁移
- 流程：
  1. `assert_single_core_mode()` 安全检查
  2. `dispatch_shootdown(0, 0)` 发送请求到远程 CPU
  3. `flush_all_local()` 本地 flush
  4. 自身邮箱 self-ACK（仅当 `request_gen == generation`）
  5. `wait_for_acks_with_retry()` 等待所有 ACK
- R68-5 修复：ACK 失败直接 panic（fail-closed）

**`flush_current_as_range(start: VirtAddr, len: usize)`**
- `normalize_range()` 对齐到页边界，超过 16 页则回退到全 flush
- R106-5 修复：若有 CPU 邮箱饱和（`fallback_cpus` 非空），本地也使用全 flush
- P2-7 修复：同样使用 `ShootdownCpuPin`

**`flush_all_local()`**
- R115-4 修复：标记为 `pub`，作为内核统一的本地 TLB flush API
- 三路径本地 flush：
  1. INVPCID 可用：`invpcid_all_nonglobal()`
  2. PCID 启用但无 INVPCID：`flush_all_pcid_without_invpcid()`（R114-4 修复：包裹在 `without_interrupts` 中）
  3. 无 PCID：`tlb::flush_all()`

**`flush_all_pcid_without_invpcid()`** (unsafe)
- R74-1 增强：无 INVPCID 时的全 PCID flush
- R114-4 修复：`debug_assert!` 检查中断已禁用
- Intel SDM Vol. 3A Section 4.10.4.1 方法：保存 CR3/CR4 -> CR3[11:0]=0 -> 清除 CR4.PCIDE -> 重载 CR3 -> 恢复 CR4/CR3

**`track_cr3_switch(new_cr3: u64)`**
- 上下文切换时更新 CR3->CPU 映射
- R69-4 修复：设置 `ASID_WRITER_PENDING` 标志，获取写锁后清除

**`collect_target_cpus(target_cr3: u64) -> Vec<usize>`**
- R68 优化：根据 CR3 追踪定向选择目标 CPU
- R69-4 修复：获取读锁前检查写者优先标志，短暂自旋让步（100 次）
- 回退：若 CR3 未追踪或追踪为空，广播到所有在线 CPU

**`enqueue_mailbox(mailbox, cr3, start, len, generation) -> bool`**
- R106-5 修复：队列满时返回 `false`（优雅降级），不再 panic
- CAS 竞争 tail 槽位，最多自旋 500K 次
- 内存序：字段 Relaxed 写入，generation Release 发布

**`post_requests(targets, cr3, start, len, generation) -> Vec<usize>`**
- R94-8 修复：目标 CPU 邮箱缺失时 panic（fail-closed）
- R106-5 修复：返回邮箱饱和的 CPU 列表

**`handle_shootdown_ipi()`**
- IPI 处理函数，在中断上下文执行
- R72 队列化：从邮箱队列 FIFO 排空所有待处理请求
- R74-1 修复：即使目标 CR3 非当前活动，也执行 `flush_all_local()`（PCID 可能缓存旧条目）
- R106-5 修复：排空队列后检查 `request_gen > ack_gen`，执行隐式全 flush 补偿邮箱饱和
- ACK 单调递增：仅当 `generation > current_ack` 时更新

**`wait_for_acks_with_retry(targets, generation, sender) -> bool`**
- 最多 `ACK_TIMEOUT_RETRIES`(3) 次重试
- R93-9 修复：`wait_for_acks` 使用 `.unwrap_or(false)`（fail-closed）
- R93-9 修复：`get_unacked_cpus` 使用 `.unwrap_or(true)`（fail-closed）
- 最终超时：debug 构建 panic，release 构建打印警告

**`ShootdownCpuPin`** (P2-7)
- 禁止抢占的 RAII guard，防止 TLB shootdown 期间 CPU 迁移
- 构造时 retry loop：采样 CPU ID -> 确认未迁移 -> preempt_disable -> 再次确认
- 标记为 `!Send + !Sync`

#### 模块间依赖

- 被 `page_table::map_mmio()`、`ensure_pte_level()` 调用
- 被 `kernel_core` 的 COW 处理、munmap、进程退出调用
- 依赖 `cpu_local` 提供 per-CPU 邮箱和 LAPIC ID
- 依赖 `tlb_ops` 提供 INVPCID 指令封装
- IPI 发送函数由 `arch` 层通过 `register_ipi_sender()` 注入

#### 当前不足与改进建议

1. **`ASID_CPU_MASKS` 使用 `BTreeMap` + 堆分配**: 在 IPI 热路径中涉及堆分配。
2. **64 CPU 上限**: `ONLINE_CPU_MASK` 使用 `AtomicU64` 位掩码。
3. **ACK 超时策略不一致**: debug 构建 panic，release 构建仅警告。
4. **缺少 shootdown 合并/批处理**: 多个连续的 shootdown 请求各自独立发送 IPI。

---

### 3.8 oom_killer.rs — OOM Killer（289 行）

#### 功能概述

当物理内存分配失败时，OOM killer 负责：
1. 首先尝试从页缓存回收内存
2. 若仍不足，评分并选择最佳受害进程终止
3. 通过注册的审计回调发出审计事件

#### 核心数据结构

```rust
// 进程快照（用于 OOM 评分）
pub struct OomProcessInfo {
    pub pid: ProcessId,
    pub tgid: ProcessId,
    pub uid: u32,
    pub gid: u32,
    pub rss_pages: u64,
    pub nice: i32,
    pub oom_score_adj: i32,
    pub has_mm: bool,
    pub is_kernel_thread: bool,
}

// 回调函数类型（依赖反转）
type SnapshotFn = fn() -> Vec<OomProcessInfo>;
type KillFn = fn(ProcessId, i32);
type CleanupFn = fn(ProcessId);
type TimestampFn = fn() -> u64;
type AuditEmitFn = fn(u32, u32, u64, u64, i64, u64); // R106-8

// OOM 统计
pub struct OomStats {
    pub is_running: bool,
    pub low_watermark_pct: usize,  // 5%
    pub min_reclaim_pages: usize,  // 64
}
```

#### 核心函数分析

**`on_allocation_failure(nr_pages_needed: usize)`**
- OOM 处理入口
- 重入保护：`OOM_RUNNING` AtomicBool CAS
- nr_pages_needed=0 直接返回
- 流程：页缓存回收 -> `still_under_pressure()` 检查 -> `kill_best_candidate()`

**`score_process(info: &OomProcessInfo) -> i64`**
- 免杀：`oom_score_adj <= -1000` -> `i64::MIN`
- R28-10 修复：pid 1 或 tgid 1（init）-> `i64::MIN`
- 内核线程或无 mm -> `i64::MIN`
- 基础分 = `rss_pages * 100`，加 `oom_score_adj`，加 `nice * 10`（使用 `saturating_*` 防溢出）

**`register_audit_callback(audit_emit: AuditEmitFn)`**
- R106-8 修复：独立注册审计回调，支持审计子系统延迟初始化

**`emit_audit(victim, needed, ts_cb, audit_cb)`**
- R106-8 修复：在 `klog!` 控制台日志之外，调用注册的审计回调将事件写入防篡改的哈希链审计环形缓冲区

#### 模块间依赖

- 被 `buddy_allocator::alloc_physical_pages()` 调用
- 通过 `register_callbacks()` 注入 `kernel_core` 的进程管理函数
- 通过 `register_audit_callback()` 注入审计子系统函数
- 调用 `page_cache::PRESSURE_HANDLER` 回收缓存页

---

### 3.9 dma.rs — DMA 缓冲区管理（488 行）

#### 功能概述

提供统一的 DMA 缓冲区分配 API，支持：
1. 物理连续内存分配（通过 Buddy 分配器）
2. 按需 IOMMU 映射（通过注入的 `IommuOps`）
3. 防御性内存清零（分配和释放时 scrub）
4. Fail-safe 策略（映射失败时泄漏而非复用）

#### 核心数据结构

```rust
// DMA 缓冲区
pub struct DmaBuffer {
    phys: u64,          // 物理地址
    iova: u64,          // I/O 虚拟地址（identity mapping: iova == phys）
    size: usize,        // 分配大小（页对齐）
    domain_id: DomainId,// IOMMU 域 ID
}

// IOMMU 操作接口（依赖反转）
pub struct IommuOps {
    pub kernel_domain_id: DomainId,
    pub map_range: fn(DomainId, u64, u64, usize, bool) -> Result<(), DmaError>,
    pub unmap_range: fn(DomainId, u64, usize) -> Result<(), DmaError>,
}

// DMA 错误类型
pub enum DmaError {
    InvalidSize,
    NoMem,
    OutOfDirectMapRange,
    IommuMapFailed,       // 映射状态不确定（必须泄漏页面）
    IommuMapRejected,     // R95-4: 映射被拒绝（可安全释放）
    IommuUnmapFailed,
}
```

#### 核心函数分析

**`alloc_dma_buffer(size: usize) -> Result<DmaBuffer, DmaError>`**
- 流程：
  1. 大小对齐到 4KB
  2. 从 Buddy 分配器分配物理页
  3. R153-I5 修复：使用 `checked_next_power_of_two()` 匹配 buddy 分配器模式
  4. 验证物理地址在直映窗口内（< 1GB）
  5. 清零缓冲区（`scrub_range`）
  6. 若 IOMMU 已注册，执行 identity mapping（iova == phys）
  7. 映射失败处理：
     - `IommuMapRejected`：安全释放页面
     - `IommuMapFailed`：scrub 后泄漏页面

**`DmaBuffer::drop()`**
- R95-8 文档：驱动必须在 drop 前静默设备
- R153-I5 修复：使用 `checked_next_power_of_two()` 防溢出
- 流程：IOMMU unmap -> 失败则 scrub 后泄漏 -> scrub 整个分配区域 -> 释放物理页

**`scrub_range(phys: u64, bytes: usize)`**
- 使用 `ptr::write_volatile` 逐字节清零
- `compiler_fence(SeqCst)` 防止编译器优化掉清零操作

**`register_iommu_ops(ops: IommuOps)`**
- 使用 `spin::Once` 确保仅注册一次

#### 模块间依赖

- 调用 `buddy_allocator::alloc_physical_pages()` / `free_physical_pages()`
- 使用 `PHYSICAL_MEMORY_OFFSET` 进行物理到虚拟地址转换
- IOMMU 操作由 `kernel/iommu` 通过 `register_iommu_ops()` 注入

#### 当前不足与改进建议

1. **Identity IOVA 映射**: `iova == phys` 限制了 IOMMU 的隔离能力。应引入 IOVA 分配器。
2. **`scrub_range` 逐字节清零效率低**: 应使用 `rep stosb` 或按缓存行批量清零。
3. **直映窗口限制**: `MAX_DIRECT_MAP_PHYS = 1GB`。
4. **缺少 scatter-gather 支持**: 仅支持物理连续缓冲区。
5. **`DmaStats` 过于简单**: 仅包含 `iommu_enabled` 布尔值。

---

## 4. 模块整体评估

### 4.1 架构设计评估

#### 优点

1. **清晰的分层**: 物理内存（buddy）-> 虚拟内存（page_table）-> 缓存（page_cache）-> DMA，层次分明。
2. **依赖反转模式**: 通过函数指针注入（IPI sender、IOMMU ops、OOM callbacks、audit callback）成功避免了 crate 循环依赖。
3. **RAII 资源管理**: `DmaBuffer` 和 `PageCacheEntry` 的 Drop 实现确保物理页不会泄漏。
4. **两阶段操作**: page_cache 的 shrink/writeback 和 TLB shootdown 都采用"持锁收集 -> 释锁执行"模式。
5. **安全加固意识**: 多轮 R-编号修复（R28/R32/R34/R36/R42/R67-R74/R93-R100/R104/R106/R114-R115/R132/R141/R148/R152-R153），覆盖溢出、竞态、fail-open、日志泄漏等问题。
6. **日志安全**: R148-8 地址信息门控，R132-3 物理基地址门控，防止 release 构建泄漏 KASLR/布局信息。

#### 不足

1. **全局锁瓶颈**: `PT_LOCK`（页表）、`BUDDY_ALLOCATOR`（物理页）、`lru`（页缓存）都是全局单锁。
2. **堆分配在热路径**: TLB shootdown 的 `collect_target_cpus()` 返回 `Vec<usize>`。
3. **初始化顺序隐式**: 各子模块的初始化分散在不同调用点。
4. **缺少 VMA 抽象**: 没有虚拟内存区域（VMA）管理。
5. **直映窗口 1GB 限制**: 整个模块假设物理内存 < 1GB。

### 4.2 安全性评估

#### 已实施的安全措施

| 威胁 | 机制 | 代码位置 |
|------|------|----------|
| Stale TLB -> UAF | IPI-based shootdown + fail-closed ACK | `tlb_shootdown.rs` R68-5 |
| PCID stale entries | 无条件 flush（即使 CR3 不匹配） | `handle_shootdown_ipi()` R74-1 |
| 页表并发修改 | 全局 `PT_LOCK` + 关中断 | `page_table.rs` R67-5 |
| 整数溢出 | `checked_add`/`checked_mul`/`checked_next_power_of_two` | `page_table.rs` / `buddy_allocator.rs` R152-17 |
| 部分映射泄漏 | 失败回滚 | `map_range()` R34-MM-1 |
| Buddy 部分释放 | 对齐检查 + order 匹配验证 | `buddy_allocator.rs` R74-4 |
| Buddy 越界合并 | `buddy_idx + pages <= total_pages` | `is_buddy_free()` R104-1 |
| Buddy 重叠合并 | free_list 存在性验证 | `merge_blocks()` R152-15 |
| DMA 信息泄露 | 分配/释放时 scrub | `dma.rs` |
| DMA 映射不确定 | Fail-safe 泄漏策略 | `dma.rs` R95-4 |
| OOM kill init | pid 1/tgid 1 免杀 | `oom_killer.rs` R28-10 |
| 堆地址预测 | KASLR 堆随机化 | `memory.rs` |
| 堆覆盖保留内存 | UEFI 内存映射验证 | `memory.rs` R72-4 |
| 非 PRESENT PTE 帧泄漏 | `take_nonpresent_leaf_frame()` | `page_table.rs` R141-9 |
| 地址日志泄漏 KASLR | `#[cfg(debug_assertions)]` 门控 | `memory.rs` R148-8 |
| CR4.PCIDE 操作中断安全 | `without_interrupts` + `debug_assert!` | `tlb_shootdown.rs` R114-4 |
| 早期 PML4 并发修改 | `num_online_cpus() <= 1` 断言 | `page_table.rs` R152-16 |
| CPU 迁移导致 TLB 不一致 | `ShootdownCpuPin` preempt_disable | `tlb_shootdown.rs` P2-7 |
| 邮箱饱和 DoS | 优雅降级到全 flush | `tlb_shootdown.rs` R106-5 |
| 邮箱缺失静默跳过 | panic（fail-closed） | `post_requests()` R94-8 |

### 4.3 性能评估

#### 热路径分析

| 操作 | 锁竞争 | 堆分配 | IPI 开销 | 评估 |
|------|--------|--------|----------|------|
| 物理页分配 | 全局 Mutex | 无 | 无 | 中等瓶颈 |
| 页表映射 | PT_LOCK + 关中断 | 可能（中间页表） | 无 | 高瓶颈 |
| TLB shootdown | ASID_CPU_MASKS RwLock | Vec/BTreeMap | 每目标 CPU 一次 | 高开销 |
| 页缓存查找 | bucket RwLock + LRU Mutex | 无 | 无 | LRU 锁是瓶颈 |
| 页缓存回收 | LRU Mutex + bucket RwLock | 无 | 无 | 两锁交替 |
| DMA 分配 | Buddy Mutex | 无 | 无 | 中等 |

---

## 5. 并发模型与锁序

### 5.1 锁清单

| 锁 | 类型 | 保护对象 | 热路径 |
|----|------|----------|--------|
| `PT_LOCK` | `Mutex<()>` | 全局页表修改 | map/unmap/MMIO |
| `BUDDY_ALLOCATOR` | `Mutex<Option<BuddyAllocator>>` | 物理页分配/释放 | alloc/free |
| `PAGE_TABLE_MANAGER` | `Mutex<Option<PageTableManager>>` | 全局页表管理器实例 | 初始化 |
| `PAGE_CACHE.buckets[i]` | `RwLock<BTreeMap>` | 第 i 个哈希桶 | find/add/remove |
| `PAGE_CACHE.lru` | `Mutex<LruList>` | LRU 链表 | find(touch)/add/shrink |
| `PageCacheEntry.io_lock` | `Mutex<()>` | 单页 I/O 序列化 | read/writeback |
| `ASID_CPU_MASKS` | `RwLock<BTreeMap>` | CR3->CPU 映射 | shootdown/context switch |
| `CALLBACKS` | `Mutex<Callbacks>` | OOM 回调函数 | OOM 触发 |
| `OOM_RUNNING` | `AtomicBool` | OOM 重入保护 | OOM 触发 |
| `ALLOCATOR` | `LockedHeap` | 内核堆 | 所有堆分配 |

### 5.2 锁序规则

1. **page_cache**: `bucket[i].write()` -> `lru.lock()`（在 `add_to_cache` 和 `remove_from_cache` 中）
2. **page_table**: `PT_LOCK.lock()` -> `interrupts::without_interrupts()`（在 `with_current_manager` 中）
3. **shootdown**: `page_table::ensure_pte_level()` / `map_mmio()` 会在持有 `PT_LOCK` 的情况下调用 `tlb_shootdown::flush_current_as_range()`，因此存在 `PT_LOCK` -> `ASID_CPU_MASKS`(read) 的嵌套
4. **OOM**: `OOM_RUNNING` CAS -> `PRESSURE_HANDLER.on_memory_pressure()`（触发 `PAGE_CACHE` 的 LRU/bucket 锁）-> `buddy_allocator::get_allocator_stats()`（获取 `BUDDY_ALLOCATOR` 锁）-> `CALLBACKS.lock()`

### 5.3 中断上下文约束

- `handle_shootdown_ipi()`: 在 IPI 中断上下文执行，不可获取任何可能被中断前代码持有的锁
- `with_current_manager()`: 关中断执行，防止 CR3 切换
- `map_mmio()`: 关中断 + PT_LOCK，最长持锁段
- `flush_all_pcid_without_invpcid()`: R114-4 要求中断已禁用

---

## 6. 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| PFN | Page Frame Number | 物理帧号 |
| PTE | Page Table Entry | 页表条目（最低级） |
| PDE | Page Directory Entry | 页目录条目 |
| PML4 | Page Map Level 4 | 四级页表根 |
| CR3 | Control Register 3 | 页表基地址寄存器 |
| PCID | Process Context Identifier | 进程上下文标识符（TLB 标签） |
| INVPCID | Invalidate PCID | 按 PCID 失效 TLB 的指令 |
| IOVA | I/O Virtual Address | 设备可见的虚拟地址 |
| TLB | Translation Lookaside Buffer | 地址转换缓存 |
| COW | Copy-On-Write | 写时复制 |
| LRU | Least Recently Used | 最近最少使用（淘汰策略） |
| OOM | Out Of Memory | 内存不足 |
| DMA | Direct Memory Access | 直接内存访问 |
| IOMMU | I/O Memory Management Unit | I/O 内存管理单元 |
| KASLR | Kernel Address Space Layout Randomization | 内核地址空间布局随机化 |
| UEFI | Unified Extensible Firmware Interface | 统一可扩展固件接口 |
| BSP | Bootstrap Processor | 引导处理器 |
| AP | Application Processor | 应用处理器 |
| IPI | Inter-Processor Interrupt | 处理器间中断 |
| MMIO | Memory-Mapped I/O | 内存映射 I/O |
| RAII | Resource Acquisition Is Initialization | 资源获取即初始化 |
