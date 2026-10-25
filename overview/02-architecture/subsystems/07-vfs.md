# Zero-OS VFS 模块深度分析

> 分析日期：2026-05-13

## 1. 模块概述

### 1.1 定位与职责

VFS（Virtual File System）模块是 Zero-OS 内核的文件系统抽象层，位于 `kernel/vfs/` 目录下。该模块提供统一的文件系统接口，向上对接系统调用层（sys_open、sys_read、sys_write、sys_stat、sys_openat2 等），向下管理多种具体文件系统实现。

VFS 模块承担以下核心职责：

- **文件系统抽象**：通过 `FileSystem` / `Inode` trait 定义统一接口，所有具体文件系统（ramfs、ext2、procfs、devfs、cgroupfs、initramfs）均实现该接口
- **路径解析**：支持 openat2 风格的路径解析，包含 `RESOLVE_BENEATH`、`RESOLVE_IN_ROOT`、`RESOLVE_NO_XDEV`、`RESOLVE_NO_SYMLINKS`、`RESOLVE_NO_MAGICLINKS` 等安全标志
- **挂载管理**：基于命名空间的挂载表管理，支持 `CLONE_NEWNS` 语义的 Copy-on-Write 挂载表复制，R144-5 每命名空间挂载数限制（MAX_MOUNTS_PER_NS=4096）
- **权限控制**：完整的 POSIX DAC（自主访问控制）实现，包括 umask、setuid/setgid 位清理、sticky-bit 语义；R135-1 修复使用宿主映射凭证（host-mapped credentials）替代命名空间相对凭证
- **MAC 集成**：通过 LSM（Linux Security Module）hook 实现强制访问控制，在 DAC 检查之前执行 MAC 策略；R153-2 stat() LSM 门控、R154-1 fd-backed stat MAC 门控
- **设备管理**：通过 devfs 提供 `/dev` 虚拟文件系统，支持块设备动态注册
- **PID 命名空间感知**：procfs 支持 PID 命名空间隔离（R133-4/R134-1/R141-5/R142-3），目录条目显示命名空间本地 PID
- **cgroup 委托**：cgroupfs 支持 P1-3 委托模式，非 root 用户可管理受委托的 cgroup 子树

### 1.2 架构概览

```text
+-------------------+
|   Syscalls        |  sys_open, sys_read, sys_write, sys_stat, sys_openat2
+-------------------+
         |
         v
+-------------------+
|   VFS Manager     |  路径解析, 挂载表, DAC/MAC 权限检查, 命名空间隔离
+-------------------+
         |
   +-----+-----+-----+-----+-----+-----+
   |     |     |     |     |     |     |
   v     v     v     v     v     v     v
 ramfs  ext2  procfs devfs cgroup initramfs mount_ns
```

### 1.3 依赖关系

VFS 模块依赖以下内核子系统（见 `Cargo.toml`）：

| 依赖 | 用途 |
|------|------|
| `kernel_core` | 进程凭证（含宿主映射 euid/egid）、FileOps trait、系统调用接口、挂载命名空间、PID 命名空间 |
| `drivers` | 键盘输入读取（console 设备）、`print!` / `println!` 宏 |
| `lsm` | LSM hook 接口（MAC 强制访问控制） |
| `block` | BlockDevice trait（块设备抽象） |
| `mm` | 页缓存（page_cache）、伙伴分配器（buddy_allocator）、物理内存偏移 |
| `cap` | NamespaceId 类型定义 |
| `klog` | 内核日志宏 |
| `spin` | 自旋锁（Mutex / RwLock） |
| `lazy_static` | 全局静态变量延迟初始化 |
| `bitflags` | MountFlags 位标志定义 |

## 2. 文件结构与代码规模

| 文件 | 行数 | 职责 |
|------|------|------|
| `lib.rs` | 87 | 模块入口，pub mod 声明与 re-export，`vfs_init()` 入口 |
| `traits.rs` | 302 | 核心 trait 定义：FileSystem、Inode、FileHandle；R154-1 fd-backed stat MAC 门控 |
| `types.rs` | 518 | 类型定义：FileMode、FileType、FsError（19 种）、OpenFlags（含 O_PATH R130-6）、ResolveFlags、Stat、DirEntry |
| `manager.rs` | 1737 | VFS 管理器：挂载表、路径解析、DAC/MAC 权限、系统调用回调；R135-1/R135-2 宿主凭证、R144-5 挂载限制、R142-7 组件边界路径前缀、R128-2 目录遍历权限 |
| `ramfs.rs` | 740 | 内存文件系统：配额管理、文件/目录 CRUD；R112-2 溢出安全 ID |
| `ext2.rs` | 1577 | ext2 文件系统：超级块验证、块组描述符、inode 读写、页缓存集成、写支持；R106-6 扇区大小验证缓存、R133-7/R134-6 目录条目类型修复 |
| `procfs.rs` | 1745 | /proc 文件系统：PID 目录、进程状态、内存信息、PID 命名空间隔离（R133-4/R134-1/R141-5/R142-3）、cgroup 虚拟化 meminfo（R140-8） |
| `devfs.rs` | 654 | /dev 设备文件系统：null/zero/console 字符设备、块设备节点 |
| `cgroupfs.rs` | 1044 | cgroup v2 文件系统：控制文件读写、cgroup 委托（P1-3）、确定性 inode（R154-2）、内存迁移（R143-1/R148-5） |
| `initramfs.rs` | 670 | CPIO 归档解析：只读 initramfs、硬链接支持 |
| `mount_namespace.rs` | 798 | 挂载命名空间：层级管理、Copy-on-Write 挂载表、深度限制（32 层）、R144-5 挂载数限制、R151-1 原子 unmount |
| `Cargo.toml` | 20 | 依赖声明 |
| **合计** | **~9892** | |

## 3. 逐文件深度分析

### 3.1 lib.rs（87 行）

#### 3.1.1 功能概述

模块入口文件，声明 `#![no_std]` 环境并导入 `alloc`、`drivers`、`klog` 外部 crate。通过 `pub mod` 声明所有子模块，并通过 `pub use` 提供便捷的 re-export。

#### 3.1.2 核心结构

- **模块声明**：cgroupfs、devfs、ext2、initramfs、manager、mount_namespace、procfs、ramfs、traits、types
- **Re-export 策略**：将各子模块的关键类型（如 `RamFs`、`Ext2Fs`、`ProcFs`、`DevFs`、`CgroupFs`、`Initramfs`）和管理函数（`init`、`mount`、`open`、`stat`、`readdir`）统一导出
- **`vfs_init()` 函数**：顶层初始化入口，委托给 `manager::init()`，完成后通过 `klog_always!` 输出初始化日志

#### 3.1.3 依赖关系

- 向下依赖所有子模块
- `#[macro_use] extern crate drivers` 引入 `print!` / `println!` 宏
- `#[macro_use] extern crate klog` 引入 `klog!` / `klog_always!` 宏

---

### 3.2 traits.rs（302 行）

#### 3.2.1 功能概述

定义 VFS 核心抽象接口，是所有文件系统实现的契约层。

#### 3.2.2 核心数据结构与 trait

**`FileSystem` trait**（Send + Sync）：
- `fs_id() -> u64`：文件系统唯一标识
- `fs_type() -> &'static str`：文件系统类型名
- `root_inode() -> Arc<dyn Inode>`：根 inode
- `lookup(parent, name) -> Result<Arc<dyn Inode>>`：子项查找
- `create(parent, name, mode) -> Result<Arc<dyn Inode>>`：创建文件/目录（默认返回 NotSupported）
- `unlink(parent, name)`：删除条目（默认返回 NotSupported）
- `rename(old_parent, old_name, new_parent, new_name)`：重命名（默认返回 NotSupported）
- `sync()`：刷新缓存到存储

**`Inode` trait**（Send + Sync）：
- `ino() / fs_id() / stat()`：基本元数据
- `open(flags) -> Result<Box<dyn FileOps>>`：打开 inode 返回文件操作句柄
- `is_dir() / is_file()`：类型判断（默认通过 stat 推导）
- `readdir(offset)`：目录遍历
- `read_at(offset, buf) / write_at(offset, data)`：数据读写（默认返回 NotSupported）
- `truncate(len)`：截断文件
- `as_any()`：向下转型支持

**`FileHandle` struct**：
- 封装 `Arc<dyn Inode>` + `Arc<spin::Mutex<u64>>` offset + `OpenFlags` + `seekable`
- R41-3 修复：实现 `Clone`，通过 `Arc` 共享 offset，允许在释放进程锁后执行 I/O
- 实现 `FileOps` trait，桥接 VFS 层与 kernel_core 系统调用接口
- R41-1 修复：`stat()` 方法返回真实 inode 元数据（而非空壳）
- R154-1 修复：fd-backed stat 执行 MAC 门控，防止通过继承/预策略 fd 绕过路径级 R153-2 检查
- `seek()` 支持 Set/Cur/End 三种模式，使用 `checked_add` 防止溢出

#### 3.2.3 设计评价

- trait 设计简洁清晰，默认实现（返回 NotSupported）降低了只读文件系统的实现负担
- `FileHandle` 的 offset 共享机制（`Arc<Mutex<u64>>`）正确处理了 dup/fork 场景下的文件位置共享语义
- `as_any()` 方法提供了类型安全的向下转型，避免了 unsafe 代码
- R154-1 在 FileOps::stat() 中加入 LSM hook，与路径级 R153-2 形成纵深防御

---

### 3.3 types.rs（518 行）

#### 3.3.1 功能概述

定义 VFS 层的所有核心类型，包括文件类型、权限模式、错误码、打开标志、路径解析标志、时间戳、目录条目等。

#### 3.3.2 核心数据结构

**`FileType` 枚举**（`#[repr(u8)]`）：
- Regular / Directory / CharDevice / BlockDevice / Symlink / Fifo / Socket
- `to_mode_bits()` 转换为 POSIX st_mode 高 4 位

**`FileMode` struct**：
- `file_type: FileType` + `perm: u16`（低 12 位：rwxrwxrwx + setuid/setgid/sticky）
- 构造函数：`regular()` / `directory()` / `char_device()` / `block_device()`
- `to_raw()` 合并为 u32 st_mode 值

**`Stat` struct**：
- 完整的 POSIX stat 结构：dev、ino、mode、nlink、uid、gid、rdev、size、blksize、blocks、atime/mtime/ctime
- `Default` 实现使用 `TimeSpec::now()` 获取当前时间

**`FsError` 枚举**（19 种错误）：
- 覆盖 POSIX 常见错误：NotFound(ENOENT)、PermDenied(EACCES)、Exists(EEXIST)、NotDir(ENOTDIR)、IsDir(EISDIR)、CrossDev(EXDEV)、SymlinkLoop(ELOOP)、Seek(ESPIPE) 等
- `to_errno()` 方法返回负的 errno 值
- R41-1 修复：实现 `From<FsError> for SyscallError` 和 `From<Stat> for VfsStat` 类型转换
- `NotEmpty` 正确映射到 errno -39 (ENOTEMPTY)

**`OpenFlags` struct**（`u32` 包装）：
- 标准 POSIX 标志：O_RDONLY/O_WRONLY/O_RDWR/O_CREAT/O_EXCL/O_TRUNC/O_APPEND/O_NONBLOCK
- 扩展标志：O_DIRECTORY/O_NOFOLLOW/O_PATH
- R130-6 修复：`is_readable()` / `is_writable()` 对 O_PATH fd 返回 false，防止通过 O_PATH 绕过 DAC
- 完整的查询方法：`is_create()` / `is_path()` / `is_directory()` / `is_nofollow()` 等

**`ResolveFlags` struct**（`u64` 包装）：
- openat2 兼容标志：RESOLVE_NO_XDEV(0x01)、RESOLVE_NO_MAGICLINKS(0x02)、RESOLVE_NO_SYMLINKS(0x04)、RESOLVE_BENEATH(0x08)、RESOLVE_IN_ROOT(0x10)、RESOLVE_CACHED(0x20)

**`SeekWhence` 枚举**（`#[repr(i32)]`）：Set(0) / Cur(1) / End(2)

**`DirEntry` struct**：name(String) + ino(u64) + file_type(FileType)

---

### 3.4 manager.rs（1737 行）

#### 3.4.1 功能概述

VFS 管理器是整个文件系统层的核心枢纽，负责挂载表管理、路径解析、权限检查、系统调用回调注册。这是 VFS 模块中最复杂的文件。

#### 3.4.2 核心数据结构

**`Vfs` struct**：
- `mount_tables: RwLock<BTreeMap<NamespaceId, Arc<NamespaceMountTable>>>`：命名空间 -> 挂载表映射
- `devfs: RwLock<Option<Arc<DevFs>>>`：devfs 句柄，用于块设备注册

**`NamespaceMountTable` struct**：
- `mounts: RwLock<BTreeMap<String, Mount>>`：路径 -> 挂载点映射
- `root_fs: RwLock<Option<Arc<dyn FileSystem>>>`：根文件系统
- `clone_from(parent)` 方法实现 CLONE_NEWNS 语义的挂载表复制

**`Mount` struct**（manager 内部版本）：
- `path: String`：挂载路径
- `fs: Arc<dyn FileSystem>`：文件系统实例

#### 3.4.3 核心函数分析

**DAC 权限检查**：
- `check_access_permission(stat, need_read, need_write, need_exec)`：完整 POSIX DAC 实现
  - R135-1 修复：使用 `current_host_euid()` / `current_host_egid()` 宿主映射凭证，防止用户命名空间 root (ns-euid==0) 绕过宿主 inode 权限检查
  - 无进程上下文时（早期引导/内核初始化）允许访问
  - 宿主 root (host-mapped euid=0) 绕过所有检查
  - 文件所有者使用 owner 位（0o700）
  - 主组或补充组成员使用 group 位（0o070）
  - 其他用户使用 other 位（0o007）
- `apply_umask(perm)`：应用进程 umask 掩码
- `strip_suid_sgid_if_needed(perm, is_dir)`：R135-1 修复 — 使用 `current_is_host_root()` 替代命名空间 euid，防止命名空间 root 在宿主文件系统上创建 setuid/setgid 文件

**挂载管理**：
- `init()`：初始化默认挂载（ramfs@/、devfs@/dev、procfs@/proc、sysfs+cgroupfs@/sys/fs/cgroup）
- `mount(path, fs)`：R135-2 修复 — 仅宿主 root 可执行 mount（`current_is_host_root()`）；R26-2 修复 — LSM MAC 门控
- `mount_in_namespace(ns, path, fs)`：R144-5 修复 — 每命名空间挂载数限制（MAX_MOUNTS_PER_NS=4096）；R152-9 修复 — 持锁更新 root_fs 防止并发不一致
- `umount(path)`：R135-2 修复 — 仅宿主 root 可执行 umount
- `ensure_namespace_table(ns)`：延迟创建命名空间挂载表，递归确保父命名空间表已存在
- `materialize_namespace(ns)`：R74-2 修复 — 强制立即物化挂载表，防止延迟物化导致的隔离绕过

**路径解析**：
- `lookup_path_with_flags(path, resolve_flags, follow_final_symlink)`：核心路径解析引擎
  - 最大符号链接深度：40 层
  - R142-7 修复：组件边界感知路径前缀检查（`is_path_within()`），防止 "/proc" 错误匹配 "/process"
  - R128-2 修复：在每个目录组件上检查 execute/search 权限，修复多组件路径中最终组件父目录的权限检查遗漏
  - RESOLVE_BENEATH / RESOLVE_IN_ROOT：限制路径不超出锚点目录
  - RESOLVE_NO_XDEV：禁止跨越挂载边界（通过 fs_id 比较）
  - RESOLVE_NO_SYMLINKS：遇到任何符号链接返回 ELOOP
  - RESOLVE_NO_MAGICLINKS：阻止 procfs magic 符号链接
  - R41-2 修复：当设置 BENEATH/IN_ROOT 时拒绝绝对路径
  - R65-20 修复：跨越挂载点前验证目录执行权限

**文件操作**：
- `open_with_resolve(path, flags, create_mode, resolve_flags)`：openat2 兼容的文件打开
  - O_CREAT：创建文件，应用 umask 和 setuid/setgid 清理
  - O_EXCL：排他创建
  - C.4 修复：创建前重新验证权限以缩小 TOCTOU 窗口
  - R25-9 修复：DAC 检查前先执行 LSM hook（MAC 优先于 DAC）
  - R26-1 修复：文件创建前执行 MAC 门控
- `stat(path)`：R153-2 修复 — 返回 stat 元数据前执行 LSM hook_file_permission，防止攻击者在 MAC 拒绝下探测文件元数据
- `unlink(path)`：删除文件/目录
  - R135-1 修复：sticky-bit 检查使用 `current_host_euid()`
  - C.4 修复：三次 inode 验证（初始查找 -> 权限重验证 -> 最终确认）防止 TOCTOU
- `readdir(path)`：目录枚举，需要 read + execute 权限（W-2 安全修复）

**系统调用回调**：
- `register_syscall_callbacks()`：注册 VFS 回调到 kernel_core
- 回调包含：open、open_with_resolve、stat、lseek、create、unlink、readdir、truncate、mount_ns_materialize
- `vfs_readdir_callback`：R114-2 修复 — max_bytes 预算参数，限制目录条目收集以防止大目录 OOM；R131-5 修复 — 拒绝 O_PATH fd 的 getdents64
- `vfs_truncate_callback`：R132-2 修复 — 在进程锁外执行 inode 回调防止锁序反转；R129-1 修复 — 要求 fd 为写入模式

**路径工具**：
- `normalize_path(path)`：R32-VFS-1 修复 — 拒绝 `..` 越过根目录的路径遍历攻击
- `split_path(path)`：分离父目录和文件名
- `hash_path(path)`：FNV-1a 64 位哈希，用于 LSM 上下文

#### 3.4.4 安全修复清单

| 修复编号 | 描述 |
|----------|------|
| X-4 | mount/umount 限制为 root 用户 |
| R25-9 | LSM hook 集成（MAC 在 DAC 之前） |
| R26-1 | 文件创建前 MAC 门控 |
| R26-2 | mount/umount 操作 MAC 门控 |
| R32-VFS-1 | 路径规范化防止目录遍历 |
| R37-4 | 目录读取前 MAC 检查 |
| R41-2 | RESOLVE_BENEATH/IN_ROOT 拒绝绝对路径 |
| R65-20 | 挂载点跨越前验证目录权限 |
| R74-2 | 命名空间挂载表立即物化 |
| R114-2 | readdir max_bytes 预算防 OOM |
| R128-2 | 每个目录组件执行权限检查 |
| R129-1 | ftruncate 要求 fd 可写 |
| R130-6 | O_PATH fd 禁止 read/write |
| R131-5 | O_PATH fd 禁止 getdents64 |
| R132-2 | truncate 回调锁序修复 |
| R135-1 | 宿主映射凭证替代命名空间凭证 |
| R135-2 | mount/umount 使用宿主 root 检查 |
| R142-7 | 组件边界路径前缀检查 |
| R144-5 | 每命名空间挂载数限制 |
| R152-9 | 持锁更新 root_fs 一致性 |
| R153-2 | stat() LSM 门控 |
| C.4 | TOCTOU 窗口缩小（多次重验证） |
| V-1 | 打开文件前 DAC 权限检查 |
| W-2 | 目录枚举需要 read+execute 权限 |

---

### 3.5 ramfs.rs（740 行）

#### 3.5.1 功能概述

内存文件系统实现，用于临时存储和引导文件。支持文件/目录的创建、读写、删除和重命名操作，并实现了全局配额管理防止内存耗尽。

#### 3.5.2 核心数据结构

**配额管理**：
- `MAX_FILE_SIZE: usize = 16 * 1024 * 1024`（16 MiB 单文件上限）
- `MAX_TOTAL_BYTES: usize = 64 * 1024 * 1024`（64 MiB 全局上限）
- `TOTAL_BYTES_USED: AtomicUsize`：全局已用字节计数器
- `quota_try_alloc(bytes)`：CAS 循环原子分配，溢出安全
- `quota_release(bytes)`：`fetch_update` + `saturating_sub` 原子释放

**`RamFs` struct**：
- `fs_id: u64`：文件系统唯一 ID（R112-2 溢出安全分配）
- `root: Arc<RamFsInode>`：根目录 inode
- `next_ino: AtomicU64`：inode 号分配器（R112-2 溢出安全）

**`RamFsInode` struct**：
- `fs_id / ino / meta: RwLock<Meta>`：基本元数据
- `kind: NodeKind`：File（`data: RwLock<Vec<u8>>`）或 Dir（`entries: RwLock<BTreeMap<String, Arc<RamFsInode>>>`）
- `self_ref: RwLock<Option<Weak<RamFsInode>>>`：弱引用自身，用于 FileHandle 创建
- 实现 `Drop`：文件 inode 释放时归还配额

---

### 3.6 ext2.rs（1577 行）

#### 3.6.1 功能概述

完整的 ext2 文件系统实现，是 VFS 模块中最复杂的文件之一。支持超级块验证、块组描述符加载、inode 读写、目录遍历、页缓存集成的文件读取、以及基本的文件写入（仅直接块）。

#### 3.6.2 核心数据结构

**磁盘结构（`#[repr(C)]`）**：
- `Ext2Superblock`（1024 字节）：inodes_count、blocks_count、log_block_size、blocks_per_group、inodes_per_group、magic(0xEF53)、inode_size 等
- `Ext2GroupDesc`（32 字节）：block_bitmap、inode_bitmap、inode_table、free_blocks_count、free_inodes_count
- `Ext2InodeRaw`：mode、uid、size_lo/size_high、block[15]（直接+间接块指针）、flags（immutable/append-only）
- `Ext2DirEntryHead`（`#[repr(C, packed)]`）：inode、rec_len、name_len、file_type

**内存结构**：
- `Ext2Fs`：dev(BlockDevice)、superblock(RwLock)、group_descs(RwLock)、block_size、sector_size（R106-6 缓存已验证扇区大小）、blocks_count（R99-4 缓存不可变副本）、meta_lock(Mutex)
- `Ext2Inode`：fs(Weak<Ext2Fs>)、ino、raw(RwLock)、size(AtomicU64)、write_lock(Mutex)
- `Ext2File`：inode(Arc<Ext2Inode>)、offset(Mutex<u64>)；R154-1 修复 — stat() 中执行 MAC 门控

**安全辅助函数**：
- `read_u32_le(buf, index)`：R96-1 修复 — 安全的小端 u32 读取，避免 `Vec<u8>` 上的未对齐访问 UB
- `validated_sector_size(dev)`：R106-6 修复 — 集中式扇区大小验证（非零、2 的幂），替代散落各处的 ad-hoc 检查

**目录条目类型映射**（R133-7/R134-6 修复）：
- `EXT2_FT_FIFO=5` / `EXT2_FT_SOCK=6`：正确映射到 FileType::Fifo / Socket
- `EXT2_FT_UNKNOWN=0`：回退到 inode mode 推导文件类型，支持未启用 filetype 特性的 ext2 镜像

#### 3.6.3 安全修复清单

| 修复编号 | 描述 |
|----------|------|
| R28-4 | 目录条目 rec_len/name_len 边界验证 |
| R28-5 | 间接块指针验证 |
| R36 | 页缓存引用计数平衡 |
| R95-3 | read_unaligned 避免未对齐 UB |
| R96-1 | read_u32_le 安全小端读取 |
| R96-2 | checked_shl/checked_add 防溢出 |
| R96-8 | 目录条目头 read_unaligned |
| R97-2 | first_data_block 一致性验证 |
| R97-3 | u32::try_from 替代截断转换 |
| R99-1 | read_block 前置块号验证 |
| R99-2 | BGDT 全链路 checked 算术 |
| R99-4 | 缓存 blocks_count 无锁验证 |
| R100-3 | inode_size 验证 |
| R105-4 | write_group_desc checked 算术 |
| R106-6 | 集中式扇区大小验证与缓存 |
| R112-2 | 溢出安全 fs_id 分配 |
| R133-7 | FIFO/Socket 目录条目类型映射 |
| R134-6 | EXT2_FT_UNKNOWN 回退 inode mode |
| R154-1 | Ext2File::stat() MAC 门控 |
| R65-EXT2-1/2/3/4/5 | 超级块字段验证、组数限制、组描述符越界检查 |

---

### 3.7 procfs.rs（1745 行）

#### 3.7.1 功能概述

/proc 虚拟文件系统实现，提供进程信息和系统状态的只读接口。与真实进程表（`PROCESS_TABLE`）集成，支持 PID 访问控制、PID 命名空间隔离和 cgroup 虚拟化。

#### 3.7.2 核心数据结构

**inode 类型**：
- `ProcRootInode`：/proc 根目录
- `ProcSelfSymlink`：/proc/self 符号链接（R141-5 使用命名空间本地 PID）
- `ProcPidDirInode`：/proc/[pid]/ 目录（R142-3 使用命名空间本地 PID 作为 inode 号）
- `ProcPidStatusInode`：/proc/[pid]/status（R140-7 输出命名空间本地 PID）
- `ProcPidCmdlineInode`：/proc/[pid]/cmdline
- `ProcPidStatInode`：/proc/[pid]/stat（R140-7 输出命名空间本地 PID）
- `ProcPidMapsInode`：/proc/[pid]/maps（R117-2 输出限制、R144-2 权限位解码）
- `ProcPidFdDirInode` / `ProcPidFdSymlink`：/proc/[pid]/fd/（R42-1 PID 重用防护）
- `ProcMeminfoInode`：/proc/meminfo（R140-8 cgroup 虚拟化）
- `ProcCpuinfoInode` / `ProcUptimeInode` / `ProcVersionInode`

#### 3.7.3 核心函数分析

**PID 命名空间隔离**（R133-4/R134-1/R141-5/R142-3）：
- `is_pid_visible_in_caller_ns(pid)`：R134-1 修复 — 检查目标进程是否在调用者 PID 命名空间子树中可见
- `caller_ns_local_pid(global_pid)`：R141-5 修复 — 将全局 PID 转换为调用者命名空间本地 PID
- `resolve_ns_pid_to_global(ns_pid)`：R141-5 修复 — 将命名空间本地 PID 解析为全局 PID
- `list_pids()`：R133-4 修复 — 仅返回调用者命名空间中可见的 PID

**PID 访问控制**（R31-1/R37-6/R139-2/R140-5）：
- `can_access_pid(pid)`：
  - 自身进程始终允许
  - R139-2 修复：使用 `current_host_euid()` 宿主映射 euid 进行 root 检查
  - R140-5 修复：使用宿主映射 UID 比较调用者和目标，基于 Option 比较防止 OVERFLOW_UID 误匹配
  - R37-6 修复：移除同 GID 访问权限

**cgroup 虚拟化 meminfo**（R140-8）：
- 非根 cgroup 且配置了 memory.max 时返回 cgroup 相对内存统计
- 根 cgroup 或无限制时返回宿主全局物理内存统计

#### 3.7.4 安全修复清单

| 修复编号 | 描述 |
|----------|------|
| R29-1 | 使用真实进程表替代硬编码数据 |
| R31-1 | PID 目录访问控制 |
| R37-6 | 移除同 GID 访问权限 |
| R39-3 | 使用共享凭证读取 |
| R42-1 | fd 符号链接 PID 重用防护 |
| R98-1 | 正确处理正交 stopped 标志 |
| R117-2 | maps 输出限制（1000 条/64KB） |
| R121-4 | mmap 长度标志位剥离 |
| R133-4 | PID 命名空间目录列表过滤 |
| R134-1 | PID 命名空间 lookup 可见性检查 |
| R139-2 | 宿主映射 euid root 检查 |
| R140-5 | 宿主映射 UID 比较 |
| R140-7 | status/stat 输出命名空间本地 PID |
| R140-8 | cgroup 虚拟化 meminfo |
| R141-5 | PID 命名空间全链路隔离 |
| R142-3 | 命名空间本地 PID inode 号 |
| R144-2 | maps 权限位解码 |

---

### 3.8 devfs.rs（654 行）

#### 3.8.1 功能概述

/dev 设备文件系统实现，提供标准字符设备（null、zero、console）和动态块设备注册。

#### 3.8.2 核心数据结构

**设备 inode 类型**：
- `DevDirInode`：/dev 根目录，`entries: RwLock<BTreeMap<String, Arc<dyn Inode>>>`
- `NullDevInode`：/dev/null — 读返回 EOF，写丢弃数据；rdev = make_dev(1, 3)
- `ZeroDevInode`：/dev/zero — 读返回无限零，写丢弃数据；rdev = make_dev(1, 5)
- `ConsoleDevInode`：/dev/console — 读从键盘缓冲区读取，写通过 `print!` 输出；rdev = make_dev(5, 1)；权限 0o620
- `BlockDevInode`：/dev/vdX — 块设备节点，`device: Arc<dyn BlockDevice>`，`rw_lock: Arc<Mutex<()>>`

#### 3.8.3 核心函数分析

- `DevFs::new()`：创建标准设备（null/zero/console），初始化 self_ref 弱引用
- `register_block_device(name, device)`：R112-2 溢出安全 inode 分配（从 100 开始）
- `BlockDevInode::read_at(offset, buf)`：块设备读取，扇区对齐批量读优化
- `BlockDevInode::write_at(offset, data)`：非对齐时读-修改-写，`rw_lock` 序列化
- `make_dev(major, minor)`：生成设备号（Linux 兼容格式）

---

### 3.9 cgroupfs.rs（1044 行）

#### 3.9.1 功能概述

cgroup v2 文件系统实现，挂载于 `/sys/fs/cgroup`。目录层级映射 cgroup 树，通过控制文件读写管理资源限制。与 `kernel_core::cgroup` 模块集成。支持 P1-3 委托模式和确定性 inode。

#### 3.9.2 核心数据结构

**确定性 inode（R154-2 修复）**：
- `CGROUPFS_INO_STRIDE = 16`：每个 cgroup 占 16 个 inode 槽位
- `cgroup_dir_ino(cgroup_id) = (cgroup_id + 1) * 16`
- `cgroup_ctrl_ino(cgroup_id, ctrl_index) = (cgroup_id + 1) * 16 + ctrl_index + 1`
- 替代原有的 AtomicU64 计数器，解决跨 mount/remount 时 inode 号不稳定问题

**`CtrlKind` 枚举**（12 种控制文件）：
- cgroup 核心：Procs / Controllers(只读) / SubtreeControl
- CPU 控制器：CpuWeight / CpuMax
- 内存控制器：MemoryMax / MemoryHigh / MemoryCurrent(只读)
- PID 控制器：PidsMax / PidsCurrent(只读)
- I/O 控制器：IoMax / IoStat(只读)

#### 3.9.3 P1-3 委托模式

- `effective_owner(cgroup_id)`：遍历祖先链查找最近委托点的 UID，用于 DAC 元数据
- `is_privileged_or_delegate(cgroup_id)`：R134-2 修复 — 使用宿主映射 root 检查和宿主映射 euid
- `apply_limit(cgroup, limits, is_delegate)`：委托管理器只能设置不超过祖先限制的值
- R153-8 修复：控制文件 stat 反映实际写权限（委托/root 可写返回 0o644，否则 0o444）

#### 3.9.4 进程迁移安全

- R148-5 修复：阻止 CLONE_VM 共享地址空间的任务迁移，防止 memory.max 绕过
- R143-1 修复：迁移时转移 cgroup 内存计费，防止源/目标计费泄漏
- CF-1 修复：正确解析 PID 并在 cgroup 间迁移任务（原子 detach+attach）

---

### 3.10 initramfs.rs（670 行）

#### 3.10.1 功能概述

CPIO "newc" 格式的只读 initramfs 实现。解析内存中的 CPIO 归档，构建文件/目录/符号链接树，支持硬链接。所有写操作返回 `ReadOnly`。

#### 3.10.2 核心数据结构

- `CPIO_MAGIC: b"070701"`
- `CPIO_HEADER_LEN: 110`
- `MAX_INITRAMFS_FILE_SIZE: 64 MiB`（R28-9 修复 — 防止 OOM）
- `GENERATED_INO_START: 1 << 32`（隐式目录的 inode 号起始值）
- `NodeKind`：Directory / File（`Arc<[u8]>`）/ Symlink（`Arc<[u8]>`）

---

### 3.11 mount_namespace.rs（798 行）

#### 3.11.1 功能概述

挂载命名空间实现，提供文件系统视图隔离。支持 CLONE_NEWNS 语义的 Copy-on-Write 挂载表复制、层级管理、深度限制。

#### 3.11.2 核心数据结构

**`MountFlags` bitflags**（`u64`）：
- RDONLY / NOSUID / NODEV / NOEXEC / NOATIME / RELATIME
- BIND / REC / PRIVATE / SHARED

**`MountNsError` 枚举**（10 种错误）：
- MaxDepthExceeded / MaxMountsExceeded（R144-5 新增）/ MountExists / MountNotFound / MountBusy / InvalidPath / PermissionDenied 等

**`Mount` struct**（mount_namespace 版本）：
- path / fs / flags / source / fstype
- `is_readonly()` / `is_nosuid()` / `is_nodev()` / `is_noexec()` 查询方法

**`MountNamespace` struct**：
- `id: NamespaceId` / `level: u8` / `parent: Option<Arc<MountNamespace>>`
- `mounts: RwLock<BTreeMap<String, Mount>>`
- `root_path: RwLock<String>`
- `new_child(parent)`：创建子命名空间，R112-2 溢出安全 ID 分配
- `find_mount(path)`：最长前缀匹配
- `has_submounts(path)`：检查子挂载

**R151-1 修复**：
- `remove_mount()`：路径规范化匹配 add_mount() 语义；先获取写锁，子挂载检查和移除原子执行

**R144-5 修复**：
- `MAX_MOUNTS_PER_NS = 4096`：每命名空间挂载数限制
- `add_mount()` 在插入前检查限制

---

### 3.12 Cargo.toml（20 行）

#### 3.12.1 依赖分析

```toml
[dependencies]
spin = "0.9"
lazy_static = { version = "1.4", features = ["spin_no_std"] }
bitflags = "2.4"
kernel_core = { path = "../kernel_core" }
drivers = { path = "../drivers" }
lsm = { path = "../lsm", features = ["lsm"] }
block = { path = "../block" }
mm = { path = "../mm" }
cap = { path = "../cap" }
klog = { path = "../klog" }
```

新增依赖：`klog`（内核日志宏），其余与之前版本一致。

---

## 4. 模块整体评估

### 4.1 架构评估

#### 4.1.1 优势

1. **trait 抽象设计良好**：`FileSystem` / `Inode` trait 定义清晰，默认实现降低了只读文件系统的实现成本。6 种文件系统（ramfs/ext2/procfs/devfs/cgroupfs/initramfs）均通过统一接口接入，扩展性强。

2. **openat2 兼容路径解析**：`ResolveFlags` 提供了安全路径限制能力。R142-7 引入组件边界感知的路径前缀检查，R128-2 修复了多组件路径的目录权限检查遗漏。

3. **命名空间隔离**：基于 `NamespaceId` 的挂载表隔离，支持 `CLONE_NEWNS` 语义。R74-2 即时物化、R144-5 挂载数限制、R151-1 原子 unmount 构成完整的命名空间安全链。

4. **PID 命名空间感知的 procfs**：R133-4/R134-1/R141-5/R142-3 系列修复实现了完整的 PID 命名空间隔离，容器内进程仅能看到自身命名空间的 PID。

5. **宿主凭证安全模型**：R135-1/R135-2 系列修复使用宿主映射凭证替代命名空间相对凭证，防止用户命名空间 root 绕过宿主 inode 权限。

6. **ext2 实现完整度高**：1577 行代码覆盖了超级块验证、块组描述符、inode 读写、目录遍历、页缓存集成、基本写入支持。R106-6 集中式扇区大小验证消除了分散的除零风险。

7. **cgroup 委托模型**：P1-3 委托模式允许非 root 用户管理受委托 cgroup 子树，R154-2 确定性 inode 确保跨 mount 一致性。

#### 4.1.2 不足

1. **挂载管理分裂**：manager.rs 和 mount_namespace.rs 存在两套并行的挂载管理实现，MountFlags 未在路径解析中执行
2. **缺少 dentry/inode 缓存**：ext2 每次路径查找都从磁盘读取
3. **VFS 抽象不完整**：缺少 `symlink()`、`readlink()`、`chmod()`、`chown()`、`utimes()`、`statfs()` 等标准操作
4. **ext2 写入限制**：仅支持直接块（12 块 = 48KB@4K块），无间接块写入支持

### 4.2 安全评估

#### 4.2.1 已实现的安全机制

| 安全机制 | 实现位置 | 状态 |
|----------|----------|------|
| POSIX DAC（owner/group/other） | manager.rs `check_access_permission` | 完整（R135-1 宿主映射） |
| umask 应用 | manager.rs `apply_umask` | 完整 |
| setuid/setgid 位清理 | manager.rs `strip_suid_sgid_if_needed` | 完整（R135-1 宿主 root 检查） |
| sticky-bit 语义 | manager.rs `unlink` | 完整（R135-1 宿主 euid） |
| LSM MAC hook | manager.rs（file_open/create/mkdir/unlink/mount/umount/stat/truncate） | 完整（R153-2/R154-1） |
| O_PATH 语义 | types.rs OpenFlags | 完整（R130-6/R131-5） |
| 路径遍历防护 | manager.rs `normalize_path` (R32-VFS-1) | 完整 |
| 符号链接循环检测 | manager.rs（MAX_SYMLINK_DEPTH=40） | 完整 |
| mount/umount 权限 | manager.rs（R135-2：仅宿主 root） | 完整 |
| TOCTOU 缓解 | manager.rs（C.4：多次重验证） | 部分 |
| PID 访问控制 | procfs.rs `can_access_pid` | 完整（R139-2/R140-5 宿主 UID） |
| PID 命名空间隔离 | procfs.rs（R133-4/R134-1/R141-5/R142-3） | 完整 |
| ext2 磁盘数据验证 | ext2.rs（R28-R134 系列） | 完整 |
| ramfs 配额管理 | ramfs.rs（V-3：64MiB 全局 + 16MiB 单文件） | 完整 |
| 命名空间隔离 | manager.rs + mount_namespace.rs | 完整（R144-5 限制/R151-1 原子） |
| cgroup 委托安全 | cgroupfs.rs（P1-3/R134-2/R148-5/R143-1） | 完整 |
| 确定性 inode | cgroupfs.rs（R154-2） | 完整 |
| 目录遍历权限 | manager.rs（R128-2） | 完整 |
| 组件边界路径检查 | manager.rs（R142-7） | 完整 |
| readdir OOM 防护 | manager.rs（R114-2 预算） | 完整 |

### 4.3 安全修复完整索引

以下为 VFS 模块中所有已标注的安全修复编号及其位置：

| 修复编号 | 文件 | 描述 |
|----------|------|------|
| R25-9 | manager.rs | LSM hook 集成（MAC 在 DAC 之前） |
| R26-1 | manager.rs | 文件创建前 MAC 门控 |
| R26-2 | manager.rs | mount/umount MAC 门控 |
| R28-4 | ext2.rs | 目录条目边界验证 |
| R28-5 | ext2.rs | 间接块指针验证 |
| R28-9 | initramfs.rs | initramfs 文件大小限制 |
| R29-1 | procfs.rs | 真实进程表集成 |
| R31-1 | procfs.rs | PID 目录访问控制 |
| R32-VFS-1 | manager.rs | 路径规范化防遍历 |
| R36 | ext2.rs | 页缓存引用计数平衡 |
| R37-4 | manager.rs | 目录读取 MAC 检查 |
| R37-6 | procfs.rs | 移除同 GID 访问权限 |
| R39-3 | procfs.rs | 共享凭证读取 |
| R41-1 | traits.rs | FileHandle stat 返回真实元数据 |
| R41-2 | manager.rs | RESOLVE_BENEATH 拒绝绝对路径 |
| R41-3 | traits.rs | FileHandle Clone 共享 offset |
| R42-1 | procfs.rs | fd 符号链接 PID 重用防护 |
| R65-20 | manager.rs | 挂载点跨越权限验证 |
| R65-EXT2-1 | ext2.rs | 超级块关键字段验证 |
| R65-EXT2-2 | ext2.rs | 组数上限限制 |
| R65-EXT2-3 | ext2.rs | 组描述符越界检查 |
| R65-EXT2-4 | ext2.rs | 位图容量验证 |
| R65-EXT2-5 | ext2.rs | inode 读取 checked 算术 |
| R74-2 | manager.rs | 命名空间挂载表立即物化 |
| R95-3 | ext2.rs | read_unaligned 避免 UB |
| R96-1 | ext2.rs | read_u32_le 安全小端读取 |
| R96-2 | ext2.rs | checked_shl/checked_add 防溢出 |
| R96-8 | ext2.rs | 目录条目头 read_unaligned |
| R97-2 | ext2.rs | first_data_block 一致性验证 |
| R97-3 | ext2.rs | u32::try_from 替代截断转换 |
| R98-1 | procfs.rs | 正交 stopped 标志处理 |
| R99-1 | ext2.rs | read_block 前置块号验证 |
| R99-2 | ext2.rs | BGDT 全链路 checked 算术 |
| R99-4 | ext2.rs | 缓存 blocks_count 无锁验证 |
| R100-3 | ext2.rs | inode_size 验证 |
| R105-4 | ext2.rs | write_group_desc checked 算术 |
| R106-6 | ext2.rs | 集中式扇区大小验证与缓存 |
| R112-2 | 多文件 | 溢出安全 ID/inode 分配 |
| R114-2 | manager.rs | readdir max_bytes 预算防 OOM |
| R117-2 | procfs.rs | maps 输出限制 |
| R121-4 | procfs.rs | mmap 长度标志位剥离 |
| R128-2 | manager.rs | 每个目录组件执行权限检查 |
| R129-1 | manager.rs | ftruncate 要求 fd 可写 |
| R130-6 | types.rs | O_PATH fd 禁止 read/write |
| R131-5 | manager.rs | O_PATH fd 禁止 getdents64 |
| R132-2 | manager.rs | truncate 回调锁序修复 |
| R133-4 | procfs.rs | PID 命名空间目录列表过滤 |
| R133-7 | ext2.rs | FIFO/Socket 目录条目类型 |
| R134-1 | procfs.rs | PID 命名空间 lookup 可见性 |
| R134-2 | cgroupfs.rs | 宿主映射 euid 委托身份 |
| R134-6 | ext2.rs | EXT2_FT_UNKNOWN 回退 inode mode |
| R135-1 | manager.rs | 宿主映射凭证 DAC 检查 |
| R135-2 | manager.rs | mount/umount 宿主 root 检查 |
| R139-2 | procfs.rs | 宿主映射 euid root 绕过 |
| R140-5 | procfs.rs | 宿主映射 UID 跨命名空间比较 |
| R140-7 | procfs.rs | status/stat 命名空间本地 PID |
| R140-8 | procfs.rs | cgroup 虚拟化 meminfo |
| R141-5 | procfs.rs | PID 命名空间全链路隔离 |
| R142-3 | procfs.rs | 命名空间本地 PID inode 号 |
| R142-7 | manager.rs | 组件边界路径前缀检查 |
| R143-1 | cgroupfs.rs | cgroup 内存计费迁移 |
| R144-2 | procfs.rs | maps 权限位解码 |
| R144-5 | mount_namespace.rs, manager.rs | 每命名空间挂载数限制 |
| R148-5 | cgroupfs.rs | 阻止 CLONE_VM 任务迁移 |
| R151-1 | mount_namespace.rs | 原子 unmount（路径规范化+持锁） |
| R152-9 | manager.rs | 持锁更新 root_fs 一致性 |
| R153-2 | manager.rs | stat() LSM 门控 |
| R153-8 | cgroupfs.rs | 控制文件 stat 反映写权限 |
| R154-1 | traits.rs, ext2.rs | fd-backed stat MAC 门控 |
| R154-2 | cgroupfs.rs | 确定性 inode 计算 |
| C.4 | manager.rs | TOCTOU 窗口缩小 |
| V-1 | manager.rs | 打开文件前 DAC 检查 |
| V-3 | ramfs.rs | ramfs 配额管理 |
| W-2 | manager.rs | 目录枚举权限检查 |
| X-4 | manager.rs | mount/umount 限制为 root |
| CF-1/3/4/5 | cgroupfs.rs | cgroup 控制文件修复 |
| P1-3 | cgroupfs.rs | cgroup 委托模式 |

### 4.4 总结

Zero-OS VFS 模块以约 9892 行 Rust 代码实现了一个功能较为完整的虚拟文件系统层，涵盖 6 种文件系统、openat2 兼容路径解析、POSIX DAC + LSM MAC 双层权限模型、命名空间隔离、PID 命名空间感知 procfs、cgroup 委托模型等核心能力。安全修复覆盖面广，形成了从 R25 到 R154 的系统化安全改进链，总计 70+ 个编号安全修复。

相较 2026-02-12 版本，主要变更包括：
- **宿主凭证安全模型**（R135-1/R135-2）：DAC/权限检查全部使用宿主映射凭证
- **PID 命名空间完整隔离**（R133-4/R134-1/R141-5/R142-3/R140-7）：procfs 全链路命名空间感知
- **stat LSM 门控**（R153-2/R154-1）：路径级和 fd 级 stat 均受 MAC 保护
- **挂载安全增强**（R144-5/R151-1/R152-9）：挂载数限制、原子 unmount、一致性保证
- **ext2 加固**（R106-6/R133-7/R134-6）：扇区大小集中验证、目录条目类型完整映射
- **cgroupfs 增强**（R154-2/R153-8/R143-1/R148-5）：确定性 inode、内存迁移安全、委托权限反映
- **路径解析修复**（R128-2/R142-7）：目录权限全组件检查、组件边界路径匹配
- **O_PATH 语义闭合**（R130-6/R131-5）：O_PATH fd 禁止 read/write/getdents64
