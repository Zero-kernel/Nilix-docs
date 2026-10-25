# Zero-OS 设备驱动、块设备、VirtIO 与 IOMMU 子系统深度分析

> 分析日期：2026-05-13

## 目录

1. [概述](#1-概述)
2. [kernel/drivers -- 基础设备驱动模块](#2-kerneldrivers--基础设备驱动模块)
   - 2.1 [模块入口 lib.rs](#21-模块入口-librs)
   - 2.2 [VGA 文本模式驱动 vga_buffer.rs](#22-vga-文本模式驱动-vga_bufferrs)
   - 2.3 [PS/2 键盘驱动 keyboard.rs](#23-ps2-键盘驱动-keyboardrs)
   - 2.4 [帧缓冲驱动 framebuffer.rs](#24-帧缓冲驱动-framebufferrs)
3. [kernel/block -- 块设备抽象层](#3-kernelblock--块设备抽象层)
   - 3.1 [块设备核心 lib.rs](#31-块设备核心-librs)
   - 3.2 [PCI 设备扫描 pci.rs](#32-pci-设备扫描-pcirs)
   - 3.3 [VirtIO 块设备定义 virtio/mod.rs](#33-virtio-块设备定义-virtiomodrs)
   - 3.4 [VirtIO 块设备驱动 virtio/blk.rs](#34-virtio-块设备驱动-virtioblkrs)
4. [kernel/virtio -- VirtIO 通用框架](#4-kernelvirtio--virtio-通用框架)
   - 4.1 [框架入口与基础定义 lib.rs](#41-框架入口与基础定义-librs)
   - 4.2 [VirtQueue 实现 queue.rs](#42-virtqueue-实现-queuers)
   - 4.3 [传输层抽象 transport.rs](#43-传输层抽象-transportrs)
5. [kernel/iommu -- IOMMU/VT-d DMA 隔离子系统](#5-kerneliommu--iommuvt-d-dma-隔离子系统)
   - 5.1 [子系统入口与全局管理 lib.rs](#51-子系统入口与全局管理-librs)
   - 5.2 [ACPI DMAR 表解析 dmar.rs](#52-acpi-dmar-表解析-dmarrs)
   - 5.3 [Intel VT-d 硬件驱动 vtd.rs](#53-intel-vt-d-硬件驱动-vtdrs)
   - 5.4 [IOMMU 域管理 domain.rs](#54-iommu-域管理-domainrs)
   - 5.5 [故障处理 fault.rs](#55-故障处理-faultrs)
   - 5.6 [中断重映射 interrupt.rs](#56-中断重映射-interruptrs)
6. [跨模块架构与数据流](#6-跨模块架构与数据流)
7. [安全设计总结](#7-安全设计总结)

---

## 1. 概述

本文档对 Zero-OS 内核中四个紧密关联的子系统进行深度分析：

| 子系统 | 路径 | 核心职责 |
|--------|------|----------|
| **drivers** | `kernel/drivers/` | 基础硬件驱动：VGA 文本模式、帧缓冲、PS/2 键盘 |
| **block** | `kernel/block/src/` | 块设备抽象层：BIO 框架、请求队列、设备注册表 |
| **virtio** | `kernel/virtio/src/` | VirtIO 通用框架：环结构、传输层抽象（MMIO/PCI） |
| **iommu** | `kernel/iommu/` | Intel VT-d IOMMU：DMA 隔离、中断重映射、故障处理 |

这四个子系统构成了从硬件抽象到安全隔离的完整设备 I/O 栈：

```
用户空间 / 文件系统
       |
       v
+------------------+
|   Block Layer    |  <- BIO 抽象、请求队列、设备注册
|  (kernel/block)  |
+---------+--------+
          |
          v
+------------------+
|  VirtIO Driver   |  <- virtio-blk 驱动、VirtQueue
| (kernel/virtio)  |
+---------+--------+
          | DMA 请求
          v
+------------------+
|   IOMMU (VT-d)   |  <- DMA 地址翻译、隔离、故障检测
| (kernel/iommu)   |
+---------+--------+
          |
          v
    物理内存 / 设备
```

---

## 2. kernel/drivers -- 基础设备驱动模块

### 2.1 模块入口 lib.rs

**文件**: `kernel/drivers/lib.rs`（21 行）

模块入口文件，组织三个子模块并提供统一的初始化接口和键盘 API re-export：

- 采用 `#![no_std]` 裸机环境，依赖 `alloc` crate 提供堆分配
- `init()` 函数为空实现，注释明确说明初始化期间避免使用尚未就绪的输出子系统
- 通过 `pub use keyboard::{...}` 重导出键盘 API（`push_scancode`、`read_buf`、`read_char`、`has_input`、`available`、`clear`）

### 2.2 VGA 文本模式驱动 vga_buffer.rs

**文件**: `kernel/drivers/vga_buffer.rs`（212 行）

提供 VGA 文本模式（80x25）控制台输出，同时将输出镜像到串口，并在帧缓冲可用时自动切换。

**核心数据结构**：

- `ScreenChar`（`#[repr(C)]`）：`ascii_character: u8` + `color_code: ColorCode`
- `Writer`：`column_position: usize` + `color_code: ColorCode` + `buffer: &'static mut Buffer`
- `static WRITER: Mutex<Writer>`：VGA 缓冲区映射到 `0xffffffff800b8000`（高半区内核空间）

**关键设计**：

| 特性 | 实现方式 |
|------|----------|
| MMIO 安全访问 | 使用 `ptr::write_volatile` / `read_volatile` 防止编译器优化 |
| 双输出通道 | 同时写 VGA 缓冲区和串口（`out dx, al` 内联汇编） |
| 帧缓冲优先 | `_print()` 检测 framebuffer 是否已初始化，优先使用像素模式 |
| 固定映射地址 | VGA 缓冲区映射到 `0xffffffff800b8000`（高半区内核空间） |

### 2.3 PS/2 键盘驱动 keyboard.rs

**文件**: `kernel/drivers/keyboard.rs`（427 行）

实现 PS/2 Set 1 扫描码到 ASCII 的转换，采用 IRQ 生产者 + syscall 消费者的环形缓冲架构。

**核心数据结构**：

- `KeyboardBuffer`：`buffer: [u8; 1024]`（环形缓冲区）+ `read_pos` / `write_pos` / `count` + `shift_pressed` / `ctrl_pressed` / `caps_lock`
- `static KEYBOARD_BUFFER: Mutex<KeyboardBuffer>`
- `static DROPPED_BYTES: AtomicU64`：缓冲区满时丢弃字节计数

**IRQ/锁安全策略**：

- 生产者（IRQ 处理路径）：`push_scancode(code)` -> `KEYBOARD_BUFFER.lock()` -> `process_scancode()` -> `push()`
- 消费者（Syscall 路径）：`read_buf()` / `read_char()` -> `without_interrupts(|| { KEYBOARD_BUFFER.lock()... })`
- `push_char(ch)`：串口 IRQ 直接推入 ASCII 字符（`-nographic` 模式）
- 特殊控制键：Ctrl+C 注入 ETX（0x03）、Ctrl+D 注入 EOT（0x04）

### 2.4 帧缓冲驱动 framebuffer.rs

**文件**: `kernel/drivers/framebuffer.rs`（948 行）

基于 UEFI GOP 帧缓冲的像素级控制台，提供 8x16 位图字体渲染。

**核心数据结构**：

- `FramebufferInfo`（`#[repr(C)]`）：`base: u64` / `size: usize` / `width: u32` / `height: u32` / `stride: u32` / `pixel_format: PixelFormat`
- `FramebufferWriter`：fb_base / width / height / stride / pixel_format / col / row / max_col / max_row / fg_color / bg_color / initialized / fb_size

**安全边界保护（R24-8 系列修复）**：

| 检查点 | 保护措施 |
|--------|----------|
| 初始化 | 验证 `stride * height` 不超过 `info.size`，`width <= stride` |
| 像素写入 | 计算偏移前检查 `offset + 4 <= fb_size` |
| 滚屏 | `ptr::copy` 前验证源/目标范围均在 `fb_size` 内 |
| 清屏 | 以 `fb_size` 为上限，防止 OOB 写入 |

**帧缓冲地址映射**：
- 物理地址 < 1GB：使用高半区映射 (`base + PHYS_OFFSET`)
- 物理地址 >= 1GB（QEMU/OVMF 典型 0x80000000）：使用恒等映射，安全模块保留为可写 MMIO
- `get_framebuffer_region()`：暴露物理区域信息供安全模块查询

---

## 3. kernel/block -- 块设备抽象层

### 3.1 块设备核心 lib.rs

**文件**: `kernel/block/src/lib.rs`（992 行）

提供块设备层的核心抽象，包括 BIO（Block I/O）框架、请求队列和设备注册表。

**核心类型体系**：

- `BioOp`：Read / Write / Flush / Discard
- `BioVec`：`phys_addr: u64` + `offset: u32` + `len: u32`（scatter-gather 元素）
- `SecurityTag`：`label: u64` + `integrity_level: u8`（LSM 集成）
- `Bio`：op + sector + vecs + security_tag + completion
- `Bio::id`：R105-5 模式溢出安全 ID 生成（wrapping 语义）

**BlockDevice trait**：
- `submit_bio(&self, bio: Bio) -> Result<(), BlockError>`
- `read_sync(&self, sector: u64, buf: &mut [u8]) -> Result<usize, BlockError>`
- `write_sync(&self, sector: u64, buf: &[u8]) -> Result<usize, BlockError>`
- `capacity_sectors(&self) -> u64`
- `sector_size(&self) -> u32`
- `block_size(&self) -> u32`
- `flush(&self) -> Result<(), BlockError>`
- `is_read_only(&self) -> bool`

**请求队列与注册表**：

- `RequestQueue`：`queue: Mutex<VecDeque<Bio>>` + `max_depth: usize` + `stats: Mutex<RequestQueueStats>`
- `BlockDeviceRegistry`：`devices: RwLock<BTreeMap<String, RegisteredDevice>>`
- `static BLOCK_REGISTRY: Lazy<BlockDeviceRegistry>`

**BIO 校验（安全关键路径）**：
- `Bio::validate()` 在提交前执行严格校验：`checked_add` / `checked_mul` 防止整数溢出、扇区对齐验证、容量边界验证

### 3.2 PCI 设备扫描 pci.rs

**文件**: `kernel/block/src/pci.rs`（304 行）

实现最小化 PCI 配置空间访问和 virtio-blk 设备扫描。

**PCI 配置空间访问**：
- Legacy PCI Mechanism #1（I/O 端口 0xCF8/0xCFC）
- BAR 解析（支持 32/64 位，过滤 I/O BAR）

**VirtIO PCI Capability 解析**：
- 遍历 PCI capability list（最多 48 次迭代，防死循环）
- 提取 virtio vendor capability（ID=0x09）：Common Configuration / Notifications / ISR Status / Device Configuration

**IOMMU 集成（安全关键）**：
- 设备发现后调用 `iommu::attach_device()` 注册到 IOMMU（fail-closed）
- R94-14：NotAvailable 时警告但继续（显式安全权衡）
- R82-2/R82-3：缺少 modern capabilities 或探测失败时关闭 Bus Master

### 3.3 VirtIO 块设备定义 virtio/mod.rs

**文件**: `kernel/block/src/virtio/mod.rs`（87 行）

定义 virtio-blk 协议相关的常量和数据结构：
- `VirtioBlkReqHeader`（`#[repr(C)]`）：req_type + reserved + sector
- `VirtioBlkConfig`（`#[repr(C)]`）：capacity + size_max + seg_max

### 3.4 VirtIO 块设备驱动 virtio/blk.rs

**文件**: `kernel/block/src/virtio/blk.rs`（1384 行）

完整的 virtio-blk 驱动实现，同时支持 MMIO 和 PCI modern 传输层。

**核心数据结构**：

- `VirtioBlkDevice`：transport + queue + capacity + blk_size + readonly + request_pool + inflight + name + dma_buf（R105-3 DMA 缓冲所有权）+ failed（R106-3 设备失败标志）
- `RequestMeta`：kind + head_desc + data_desc + status_desc + dma_buffer + data_len + abandoned

**R105-3 DMA 生命周期修复**：
- `init_virtqueue()` 返回 `DmaBuffer` 而非裸物理地址，保持 IOMMU 映射所有权
- 设备持有 `dma_buf: Option<DmaBuffer>`，确保 VirtQueue 内存在设备生命周期内保持映射

**R106-3 设备重置恢复**：
- `reset_device()`：超时后尝试 VirtIO 设备重置和重新初始化
- 重置流程：设置 `failed=true` -> 清零状态 -> 重新协商特性 -> 重建 VirtQueue -> 回收废弃请求 -> 设置 `failed=false`
- `do_request()` / `read_sync()` / `write_sync()` 前检查 `failed` 标志快速拒绝 I/O

**R152-18 VirtQueue 大小计算修复**：
- avail ring 和 used ring 大小计算包含 +2 字节的 used_event/avail_event 字段（VirtIO 规范要求）

**R150-I2 描述符边界检查**：
- `debug_assert!` 在调试构建中捕获越界描述符访问

**安全修复汇总**：

| 修复编号 | 问题 | 解决方案 |
|----------|------|----------|
| R66-3 | 未初始化 ring 内存暴露给设备 DMA | 初始化时清零整个 ring 结构 |
| R66-5 | 恶意设备伪造 used.idx 跳变 | 校验 pending 不超过 queue size |
| R66-6 | 描述符 double-free | alloc_bitmap 跟踪 + 释放前检查 |
| R39-1 | stale completion 导致 UAF | RequestMeta 按 head 匹配，超时标记 abandoned |
| R37-3 | 超时后设备晚到 DMA 写入已释放内存 | 超时不释放 DMA 缓冲（泄漏/钉住） |
| R28-2 | 扇区/字节计算溢出 | checked_mul/checked_add 全路径 |
| R94-13 | DMA 缓冲 IOMMU 映射泄漏 | DmaBuffer drop 自动 unmap |
| R105-3 | DMA 缓冲所有权丢失 | 保持 DmaBuffer 而非裸物理地址 |
| R106-3 | 设备超时无恢复 | 设备重置恢复机制 |
| R150-I2 | 越界描述符访问 | debug_assert 边界检查 |
| R152-18 | VirtQueue 大小计算缺少 event 字段 | avail/used ring +2 字节 |

---

## 4. kernel/virtio -- VirtIO 通用框架

### 4.1 框架入口与基础定义 lib.rs

**文件**: `kernel/virtio/src/lib.rs`（187 行）

提供 VirtIO 规范定义的通用常量、环结构和内存屏障原语。

**VirtIO 常量体系**：
- 设备类型：VIRTIO_DEVICE_NET(1) / BLK(2) / CONSOLE(3) / RNG(4)
- 设备状态位：ACKNOWLEDGE(1) / DRIVER(2) / DRIVER_OK(4) / FEATURES_OK(8) / FAILED(128)
- 描述符标志：VRING_DESC_F_NEXT(1) / WRITE(2) / INDIRECT(4)

**环结构定义**（`#[repr(C)]`）：
- `VringDesc`：addr(u64) + len(u32) + flags(u16) + next(u16)
- `VringAvail`：flags(u16) + idx(u16) + ring[变长]
- `VringUsed`：flags(u16) + idx(u16) + ring[变长]

**内存屏障**：
- `mb()` = `fence(SeqCst)`、`wmb()` = `fence(Release)`、`rmb()` = `fence(Acquire)`

### 4.2 VirtQueue 实现 queue.rs

**文件**: `kernel/virtio/src/queue.rs`（343 行）

通用 VirtQueue 实现，可被不同 VirtIO 设备驱动共享。

**核心结构**：

- `VirtQueue`：size + notify_off + desc/avail/used 指针 + free_list + alloc_bitmap（R44-8）+ last_used_idx + desc_phys/avail_phys/used_phys

**R152-18 VirtQueue 大小计算修复**：
- `avail_bytes` 计算包含 `+2`（avail_event 字段）
- `used_bytes` 计算包含 `+2`（used_event 字段）
- 此修复确保与 VirtIO 规范一致，防止设备写入 event 字段时越界

**R148-I7 描述符 ID 验证**：
- `pop_used()` 中验证设备返回的描述符 ID 在有效范围内

**R150-I2 / R154-I3 边界检查**：
- `write_desc()` 和 `read_desc()` 中使用 `debug_assert!` 检查描述符索引

**关键操作与安全防护**：

- **描述符分配/释放**：R44-8 固定锁序（先 alloc_bitmap 再 free_list）防死锁；R43-3 边界检查 + double-free 检测
- **Used Ring 安全读取（pop_used）**：R44-5 拒绝不合理的 used.idx 跳变；R48-1/R48-4 后退检测防重放攻击
- **初始化安全（R44-4）**：全量清零所有 ring 内存

### 4.3 传输层抽象 transport.rs

**文件**: `kernel/virtio/src/transport.rs`（477 行）

统一的 VirtIO 传输层接口，支持 MMIO 和 PCI Modern 两种传输方式。

**传输层枚举**：
- `VirtioTransport::Mmio(MmioTransport)` / `Pci(VirtioPciTransport)`

**MMIO 传输**：
- `MmioTransport`：base(*mut u8) + version(u32)
- `probe(mmio_phys, virt_offset)`：检查 magic value (0x74726976) 和 version

**PCI Modern 传输**：
- `VirtioPciTransport`：virtio_device_type + common_cfg + notify_base + notify_len + notify_off_multiplier + isr + device_cfg

**通知安全（R34-VIRTIO-1 / R35-VIRTIO-1）**：
- `checked_mul` 防止通知偏移乘法溢出
- 零长度通知区域前置验证
- 边界检查：`offset + 2` 必须在通知窗口内

**统一接口方法**：device_id / reset / status / set_status / device_features / write_driver_features / queue_max / setup_queue / queue_ready / notify / read_config_struct

---

## 5. kernel/iommu -- IOMMU/VT-d DMA 隔离子系统

### 5.1 子系统入口与全局管理 lib.rs

**文件**: `kernel/iommu/lib.rs`（1332 行）

IOMMU 子系统的核心入口，管理全局状态、设备附加/分离、DMA 映射和 VM 直通。

**编译时安全门控**：
- `#[cfg(all(feature = "unsafe_identity_passthrough", not(debug_assertions)))]` -> `compile_error!`（R94-13：防止 identity passthrough 进入生产构建）

**全局状态管理**：
- `IOMMU_ENABLED` / `IOMMU_INIT_DONE` / `IOMMU_INIT_FAILED`（R90-1 fail-closed）/ `IOMMU_UNIT_COUNT`
- `IOMMU_UNITS` / `DOMAINS` / `VM_DOMAINS` / `VM_DEVICE_IRTES`

**初始化流程**：R90-1 防重入 + fail-closed -> DMAR 表解析 -> DRHD 初始化（含中断重映射）-> AGAW 选择 -> 内核域创建 -> DMA 翻译启用 -> DMA 分配器钩子注册

**Fail-Closed 设计（ensure_iommu_ready）**：
- R90-1：初始化失败后永久拒绝
- R94-14：无硬件也拒绝（不再静默 bypass）

**VM 直通 API**：
- `create_vm_domain(vm_id)`：创建 VM 专用域（PageTable 类型）
- `assign_device_to_vm(device, domain)`：分配设备到 VM，强制中断重映射 + IRTE 分配回滚
- `unassign_device_from_vm(device, domain)`：R88-2 先 detach 再清 IRTE

**设备隔离（isolate_device）**：
- R86-1：验证 segment（legacy I/O 仅支持 segment 0）
- R86-2：PCI_CONFIG_LOCK 串行化配置空间访问
- R86-3：失效设备的 context cache 和 IOTLB

**DMA 映射钩子（R95-4 错误分类）**：
- 安全错误（映射前被拒绝）-> DmaError::IommuMapRejected（可安全释放物理页）
- 不确定错误（可能部分映射）-> DmaError::IommuMapFailed（必须泄漏物理页）
- R96-9：扩展 safe-rejection 列表

### 5.2 ACPI DMAR 表解析 dmar.rs

**文件**: `kernel/iommu/dmar.rs`（600 行）

解析 ACPI DMAR（DMA Remapping）表，发现 IOMMU 硬件单元及其设备作用域。

**DMAR 表结构层次**：
- DMAR Table -> ACPI Header + DMAR Header + Remapping Structures[]
  - DRHD（DMA Remapping Hardware Unit Definition）-> flags + segment + register_base + Device Scopes[]
  - RMRR（Reserved Memory Region Reporting）-> base_address / limit_address + Device Scopes[]

**解析安全措施**：签名验证、校验和验证、结构长度验证、checked_add 边界检查、设备作用域长度验证

### 5.3 Intel VT-d 硬件驱动 vtd.rs

**文件**: `kernel/iommu/vtd.rs`（1510 行）

Intel VT-d IOMMU 硬件单元的完整驱动实现。

**VtdUnit 核心结构**：
- reg_base / cap / ecap / segment / iotlb_offset / fault_offset / num_fault_regs
- root_table_phys（AtomicU64，CAS 分配）/ table_lock / translation_enabled
- ir_table / attached_domains / attached_devices / device_scopes / include_pci_all

**根表/上下文表初始化**：CAS 并发安全（先分配清零页 -> 验证 direct-map 范围 -> CAS 写入 -> 失败则释放）

**设备附加（attach_device）**：
- 前置检查：根表就绪、翻译已启用、设备未重复附加
- Identity 域：R94-13 feature gate + R81-2 ECAP.PT 检查
- PageTable 域：R83-4 AGAW vs CAP.SAGAW 验证
- 写入顺序：先 hi 后 lo（present 位最后可见）+ Release fence
- 后续：失效 context cache + IOTLB

**设备分离（detach_device）**：
- R87-2：先禁用 Bus Master（防 post-detach DMA）
- 清除 context entry + Release fence + 失效 context cache + IOTLB
- R87-1：持锁更新跟踪结构

**中断重映射设置**：R84-1 全程持锁防并发 -> 检查 ECAP.IR -> 分配 IR 表 -> 编程 IRTA -> GCMD.IRE -> R84-2 失败回滚

### 5.4 IOMMU 域管理 domain.rs

**文件**: `kernel/iommu/domain.rs`（884 行）

管理 IOMMU 域的 IOVA->物理地址映射，维护二级页表（Second-Level Page Table）。

**域类型**：

| 类型 | 用途 | 页表 | 安全级别 |
|------|------|------|----------|
| Identity | 调试用 pass-through | 无（IOVA=物理地址） | 低（需 feature gate） |
| PageTable | 内核 DMA / VM 直通 | 4KB 粒度 SLPT | 高（按需映射） |

**R131-9 修复**：映射默认不授予 EXECUTE 权限（DMA 设备极少需要指令获取能力），减少攻击面

**R130-3 修复**：Domain Drop 时释放页表树内存，防止长期运行中的内存泄漏

**两阶段提交（install_mapping，R83-1）**：
- Phase 1：验证与暂存（持 page_table_lock，按 address_width 选择层级，遍历每个 4KB 页，暂存 PTE 写入）
- Phase 2：原子提交（遍历 staged_writes，write_volatile 写入 PTE）
- R80-1：分配后验证物理地址在 direct-map 范围
- R80-4：不设置 A/D 位（部分硬件为保留位）
- R95-5：SeqCst fence 确保 PTE 写入对硬件可见

**清除映射（clear_mapping）**：R80-5 检测并拒绝 superpage（PS 位）条目

### 5.5 故障处理 fault.rs

**文件**: `kernel/iommu/fault.rs`（537 行）

VT-d DMA 故障检测、日志记录和设备隔离。

**故障原因枚举**：12 种（Reserved / RootEntryNotPresent / ContextEntryNotPresent / ContextEntryInvalid / AddressBeyondMgaw / WriteToReadOnly / ReadNotPermitted / PageEntryInvalid / RootEntryReserved / ContextEntryReserved / PageEntryReserved / InvalidTranslationType / Unknown）

**安全相关故障判定**：WriteToReadOnly / ReadNotPermitted / AddressBeyondMgaw / InvalidTranslationType

**故障读取流程（R85 系列修复）**：
- R85-1：从 FRI 开始轮转读取
- R85-2：checked_add 防止 MMIO 指针回绕
- R85-3：同时清除 PFO + PPF + FRI 位
- 最多读取 MAX_FAULT_RECORDS=16 条（防 DoS）

### 5.6 中断重映射 interrupt.rs

**文件**: `kernel/iommu/interrupt.rs`（653 行）

Intel VT-d 中断重映射，防止恶意设备注入任意中断。

**IRTE 结构（128 位）**：
- Lo 64 bits：Present / FPD / DestMode / RH / TriggerMode / DeliveryMode / Vector / SourceID / SIQ / SVT
- Hi 64 bits：DestinationID

**中断重映射表管理**：
- `InterruptRemappingTable`：phys_addr / virt_addr / entries / order / pages / allocation_bitmap / allocated_count
- R84-3：清零整个分配页（不仅仅是 entry 数量）
- `set_entry()`：先写 hi 再写 lo + Release fence（防半初始化 IRTE）
- `free_index()`：释放时同步清空 IRTE

**IrteHandle（MSI 编程接口）**：
- `msi_address_lo()` / `msi_address_hi()` / `msi_data()`：生成 Remappable MSI 格式寄存器值

---

## 6. 跨模块架构与数据流

### 6.1 设备发现到 I/O 完成的完整路径

```
系统启动
  |
  +-- 1. IOMMU 初始化
  |      iommu::init()
  |      +-- 解析 DMAR 表 -> 发现 VT-d 单元
  |      +-- 初始化根表/上下文表
  |      +-- 创建内核域（PageTable，按需映射）
  |      +-- 启用 DMA 翻译
  |      +-- 注册 DMA 分配器钩子
  |
  +-- 2. 块设备探测
  |      block::probe_devices()
  |      +-- PCI 总线扫描 -> 找到 virtio-blk
  |      +-- iommu::attach_device() -> IOMMU 注册
  |      +-- 启用 Bus Master
  |      +-- 解析 VirtIO PCI capabilities
  |      +-- VirtioBlkDevice::probe_pci()
  |            +-- 特性协商
  |            +-- VirtQueue 初始化（DMA 内存分配，R105-3 保持所有权）
  |            +-- 注册到 BLOCK_REGISTRY
  |
  +-- 3. I/O 请求处理
  |      BlockDevice::submit_bio(bio)
  |      +-- Bio::validate() -> 安全校验
  |      +-- do_request(op, sector, buf)
  |      |   +-- 分配 DMA 缓冲 -> mm::dma::alloc_dma_buffer()
  |      |   |   +-- iommu::map_range() -> SLPT 映射
  |      |   +-- 构建描述符链 -> VirtQueue
  |      |   +-- push_avail() + notify()
  |      |   +-- 轮询 pop_used() -> 完成处理
  |      |   +-- 超时 -> R106-3 尝试设备重置
  |      +-- DMA 缓冲释放 -> iommu::unmap_range()
  |
  +-- 4. 故障处理（异步）
         iommu::handle_dma_faults()
         +-- 读取故障记录
         +-- 日志记录（控制台 + 审计）
         +-- 设备隔离（禁用 Bus Master）
```

### 6.2 模块依赖关系

```
kernel/drivers
  +-- framebuffer（独立，bootloader 信息初始化）
  +-- vga_buffer -> framebuffer（优先使用）
  +-- keyboard（独立，IRQ 驱动）

kernel/virtio
  +-- 独立 crate（通用 VirtIO 原语）

kernel/block
  +-- -> kernel/virtio（VirtQueue、Transport）
  +-- -> kernel/iommu（设备附加、DMA 映射）
  +-- -> mm::dma（DMA 缓冲分配）
  +-- -> klog（内核日志）

kernel/iommu
  +-- -> mm::buddy_allocator（页表/IR 表分配）
  +-- -> mm::phys_to_virt（物理->虚拟地址转换）
  +-- -> mm::dma（注册映射钩子）
  +-- -> kernel/drivers（println! 宏）
```

---

## 7. 安全设计总结

### 7.1 安全修复索引

本文档涉及的源代码中包含大量标记为 `Rxx-x FIX` 的安全修复，按子系统分类如下：

**VirtIO/块设备层**：

| 编号 | 模块 | 问题描述 | 修复方案 |
|------|------|----------|----------|
| R24-8 | framebuffer | 帧缓冲越界写入 | 初始化验证 + 运行时边界检查 |
| R28-2 | block/blk | 扇区/字节计算溢出 | checked_mul/checked_add |
| R32-BLK-1 | block/blk | 描述符长度截断 | 限制 buf.len() <= u32::MAX |
| R34-VIRTIO-1 | transport | 通知偏移乘法溢出 | checked_mul + 边界检查 |
| R35-VIRTIO-1 | transport | 零长度通知区域 | 前置验证 notify_len >= 2 |
| R37-3 | block/blk | 超时后 DMA UAF | 不释放 DMA 缓冲（泄漏/钉住） |
| R39-1 | block/blk | stale completion UAF | RequestMeta 匹配 + abandoned 标记 |
| R43-3 | queue | 描述符越界/double-free | 边界检查 + 重复检测 |
| R44-4 | queue | 未初始化 ring 内存泄露 | 全量清零 |
| R44-5 | queue | used.idx 异常跳变 | 跳变检测 + 重同步 |
| R44-8 | queue | 伪造 next 指针攻击 | alloc_bitmap 跟踪 |
| R48-1/4 | queue | used.idx 后退重放 | 后退检测 + 丢弃 |
| R66-3/5/6 | block/blk | ring 信息泄露/伪造 | 清零 + idx 校验 + bitmap |
| R82-2/3 | block/pci | 孤儿设备 DMA | 失败时关闭 Bus Master |
| R105-3 | block/blk | DMA 缓冲所有权丢失 | 保持 DmaBuffer 而非裸地址 |
| R105-5 | block/lib | Bio ID 溢出 | wrapping 语义安全生成 |
| R106-3 | block/blk | 设备超时无恢复 | 设备重置恢复机制 |
| R148-I7 | queue | 设备返回无效描述符 ID | 边界验证 |
| R150-I2 | queue/blk | 越界描述符访问 | debug_assert 边界检查 |
| R152-18 | queue/blk | VirtQueue event 字段缺失 | avail/used ring +2 字节 |
| R154-I3 | queue | 描述符读取边界 | debug_assert 检查 |

**IOMMU 子系统**：

| 编号 | 模块 | 问题描述 | 修复方案 |
|------|------|----------|----------|
| R80-1 | domain | 页表页超出 direct-map | 分配后验证物理地址范围 |
| R80-2 | domain | 并发页表修改 | page_table_lock 互斥 |
| R80-3 | domain | 错误页表层级 | 按 address_width 选择 |
| R80-4 | domain | A/D 位为保留位 | 不设置 A/D 位 |
| R80-5 | domain | 误清 superpage | 检测并拒绝 PS 位条目 |
| R81-1 | vtd | context entry 可见性 | 先 hi 后 lo + fence |
| R81-2 | vtd | 无 PT 支持的 identity 域 | 检查 ECAP.PT |
| R83-1 | domain | 部分映射"幽灵映射" | 两阶段提交 |
| R83-4 | vtd | AGAW 不匹配 | 验证域 AGAW vs CAP.SAGAW |
| R84-1 | vtd | 并发 IR 编程 | 全程持 ir_table 锁 |
| R84-2 | vtd | IR 失败后表内存悬挂 | 回滚 GCMD.IRE + 清 IRTA |
| R84-3 | interrupt | IRTE 未初始化内存 | 清零整个分配页 |
| R85-1 | fault | 丢失 FRI 之后的故障 | FRI 起始轮转 |
| R85-2 | fault | MMIO 指针回绕 | checked_add |
| R85-3 | fault | 陈旧 FRI 重触发 | 清除 FRI 位 |
| R85-4 | lib | 故障中断风暴 | overflow 时禁用中断 |
| R86-1/2/3 | lib | 设备隔离不完整 | segment 验证 + 锁 + cache 失效 |
| R87-1/2 | vtd | detach 竞态/post-detach DMA | 先禁 BM + 持锁更新 |
| R88-1/2 | lib | VM 直通 IR 缺失 | 强制 IR + detach 优先 |
| R90-1 | lib | 初始化失败后 DMA bypass | INIT_FAILED flag + fail-closed |
| R94-13 | lib/vtd | identity passthrough 泄入生产 | compile_error! + feature gate |
| R94-14 | lib | 无 IOMMU 静默 bypass | NotAvailable 错误 |
| R95-4 | lib | DMA 映射错误分类 | safe vs uncertain 分类 |
| R95-5 | domain | PTE 写入对硬件不可见 | SeqCst fence |
| R96-6 | lib | 死代码/冗余检查 | 移除已被 ensure_iommu_ready 覆盖的检查 |
| R96-9 | lib | DMA 错误分类过窄 | 扩展 safe-rejection 列表 |
| R130-3 | domain | Domain Drop 页表内存泄漏 | Drop 释放页表树 |
| R131-9 | domain | DMA 映射默认可执行 | 不授予 EXECUTE 权限 |

### 7.2 核心安全原则

1. **Fail-Closed（失败即关闭）**：
   - IOMMU 初始化失败后，所有 DMA 操作被永久拒绝
   - 无 IOMMU 硬件时不再静默 bypass，强制调用方显式处理
   - 设备附加前必须 IOMMU 就绪且翻译已启用

2. **Defense in Depth（纵深防御）**：
   - VirtQueue: alloc_bitmap + 边界检查 + used.idx 校验（三重防护）
   - IOMMU: 两阶段提交 + fence + cache 失效（三重保证映射一致性）
   - 设备隔离: 禁 Bus Master + 清 context + 失效 IOTLB

3. **Checked Arithmetic（受检算术）**：
   - 所有涉及物理地址、偏移量、大小的计算均使用 `checked_add` / `checked_mul`
   - 防止整数溢出导致的 MMIO 指针回绕、缓冲区越界

4. **Ordering Guarantees（顺序保证）**：
   - Context Entry / IRTE: 先写 hi 再写 lo（present 位最后可见）
   - PTE 写入后 SeqCst fence（确保 IOTLB 失效前硬件可见）
   - VirtQueue: wmb() 在更新 avail.idx 前，rmb() 在读取 used ring 前

5. **Resource Leak Prevention（资源泄漏防护）**：
   - DmaBuffer drop 自动 unmap IOMMU 映射
   - R105-3：VirtQueue DMA 缓冲所有权保持在设备结构中
   - R130-3：Domain Drop 释放页表树内存
   - 超时请求不释放 DMA 缓冲（防 UAF，接受可控泄漏）
   - IR 表分配失败时回滚 IRTA 寄存器
   - 设备探测失败时关闭 Bus Master

6. **Compile-Time Safety（编译时安全）**：
   - `unsafe_identity_passthrough` feature 在非 debug 构建中触发 `compile_error!`
   - `#[cfg(feature = "audit")]` 条件编译审计日志

7. **Device Recovery（设备恢复，R106-3 新增）**：
   - VirtIO 设备超时后尝试完整的设备重置和重新初始化
   - failed 标志快速拒绝新 I/O，防止雪崩
   - 重置成功后回收废弃请求的资源

---

## 附录：源文件清单

| 文件路径 | 行数 | 核心职责 |
|----------|------|----------|
| `kernel/drivers/lib.rs` | 21 | 驱动模块入口 |
| `kernel/drivers/vga_buffer.rs` | 212 | VGA 文本模式 + 串口输出 |
| `kernel/drivers/keyboard.rs` | 427 | PS/2 键盘输入 |
| `kernel/drivers/framebuffer.rs` | 948 | GOP 帧缓冲控制台 |
| `kernel/block/src/lib.rs` | 992 | 块设备抽象层 |
| `kernel/block/src/pci.rs` | 304 | PCI 扫描与 VirtIO capability 解析 |
| `kernel/block/src/virtio/mod.rs` | 87 | VirtIO 块设备定义 |
| `kernel/block/src/virtio/blk.rs` | 1384 | VirtIO 块设备驱动 |
| `kernel/virtio/src/lib.rs` | 187 | VirtIO 通用常量与环结构 |
| `kernel/virtio/src/queue.rs` | 343 | VirtQueue 通用实现 |
| `kernel/virtio/src/transport.rs` | 477 | 传输层抽象（MMIO/PCI） |
| `kernel/iommu/lib.rs` | 1332 | IOMMU 子系统入口与全局管理 |
| `kernel/iommu/dmar.rs` | 600 | ACPI DMAR 表解析 |
| `kernel/iommu/vtd.rs` | 1510 | Intel VT-d 硬件驱动 |
| `kernel/iommu/domain.rs` | 884 | IOMMU 域管理与页表 |
| `kernel/iommu/fault.rs` | 537 | DMA 故障处理 |
| `kernel/iommu/interrupt.rs` | 653 | 中断重映射 |
