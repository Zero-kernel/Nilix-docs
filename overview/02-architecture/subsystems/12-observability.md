# 运维监控模块分析文档

> 分析范围：`kernel/livepatch/`、`kernel/trace/`、`kernel/audit/`、`kernel/compliance/`、`kernel/klog/`
> 分析日期：2026-05-13
> Zero-OS 版本：main 分支

---

## 1. 模块概述

Zero-OS 的运维监控体系由五个内核 crate 协同构成，覆盖了从热补丁安全加载、运行时可观测性、安全审计、合规策略执行到结构化日志输出的完整链路：

| 模块 | 职责 | 核心安全属性 |
|------|------|-------------|
| `livepatch` | 内核函数热替换（INT3 detour） | ECDSA P-256 签名验证 + KAT 门控 + 依赖拓扑排序 |
| `trace` | 可观测性基础设施（tracepoint / profiler / watchdog / kdump） | per-CPU 无锁计数、加密崩溃转储、指针脱敏 |
| `audit` | 安全审计日志 | SHA-256 hash chain + 可选 HMAC-SHA256、强制开启 |
| `compliance` | 合规策略引擎 | Hardening profile + PolicySurface（boot-time locked）+ FIPS 模式（sticky）+ KAT 自测 |
| `klog` | 结构化内核日志 | Profile-aware 日志过滤、Secure 模式零输出、指针安全规范 |

五个模块之间存在明确的依赖关系：
- `compliance` 调用 `audit::crypto`（SHA-256/HMAC KAT）和 `livepatch::ecdsa_p256`（ECDSA KAT）
- `compliance` 的 FIPS 状态被 `kdump` 查询以决定是否抑制输出
- `audit` 被 `compliance` 用于记录 FIPS 状态变更事件和 PolicySurface 验证事件
- `klog` 通过 `KlogProfile` 原子状态实现 profile-aware 日志输出，由 `compliance` 驱动
- `livepatch` 通过回调机制将生命周期事件注入 `audit` 的 hash chain
- `trace` 与 `audit` 相互独立，但共享类似的 capability gate 设计模式
- `livepatch` 和 `audit` 共享 `crypto` crate 中的 SHA-256 实现（R148-I9 修复）

---

## 2. 文件结构

### 2.1 行数统计表

| 文件路径 | 行数 | 主要职责 |
|---------|------|---------|
| `kernel/livepatch/lib.rs` | 2083 | 热补丁核心：patch 格式解析、签名验证、INT3 detour、状态机、依赖管理、拓扑排序、回滚策略、审计集成、syscall 入口 |
| `kernel/livepatch/ecdsa_p256.rs` | ~451 | ECDSA P-256 验签：KAT 状态机、RFC 6979 测试向量、多密钥验证 |
| `kernel/trace/lib.rs` | 607 | 可观测性框架：tracepoint 注册/触发、read guard、TraceRecord |
| `kernel/trace/profiler.rs` | ~433 | PC 采样分析器：per-CPU seqlock ring buffer、控制面锁 |
| `kernel/trace/watchdog.rs` | ~525 | 挂起任务看门狗：heartbeat、one-shot trip、generation ABA 防护 |
| `kernel/trace/kdump.rs` | ~964 | 加密崩溃转储：ChaCha20 加密、KptrGuard 指针脱敏、FIPS 抑制 |
| `kernel/trace/counters.rs` | ~290 | per-CPU 原子计数器：15 种预定义指标、聚合快照 |
| `kernel/audit/lib.rs` | ~2650 | 安全审计：SHA-256 hash chain、HMAC-SHA256、ring buffer、事件分类、时间戳回调 |
| `kernel/compliance/lib.rs` | 912 | 合规引擎：Hardening profile、PolicySurface（boot-time locked）、FIPS 模式、KAT 编排 |
| `kernel/klog/lib.rs` | 265 | 内核日志：profile-aware 宏（klog!/klog_always!/klog_force!/kprintln!） |
| **合计** | **~9180** | |

### 2.2 目录结构

```
kernel/
├── livepatch/
│   ├── lib.rs              # 热补丁核心逻辑 + 依赖管理 + 拓扑排序
│   └── ecdsa_p256.rs       # ECDSA P-256 验签模块
├── trace/
│   ├── lib.rs              # 可观测性框架入口
│   ├── profiler.rs         # PC 采样分析器
│   ├── watchdog.rs         # 挂起任务看门狗
│   ├── kdump.rs            # 加密崩溃转储
│   └── counters.rs         # per-CPU 原子计数器
├── audit/
│   └── lib.rs              # 安全审计子系统
├── compliance/
│   └── lib.rs              # 合规策略引擎 + PolicySurface
└── klog/
    └── lib.rs              # 结构化内核日志宏
```

---

## 3. 逐文件分析

### 3.1 kernel/livepatch/lib.rs（2083 行）

#### 3.1.1 Patch 镜像格式（ZLP2）

livepatch 定义了一套紧凑的二进制 patch 镜像格式，所有整数采用小端序：

```
偏移   大小  字段                说明
0x00   4    magic              = "ZLP2"
0x04   2    version            = 1
0x06   2    header_len         = 72
0x08   4    flags              bit 0: HAS_DEPS（含依赖表）
0x0C   4    reserved0          = 0
0x10   8    target_addr        目标函数入口的内核虚拟地址
0x18   8    handler_addr       替换处理函数地址（0 表示 handler 在 patch_data 中）
0x20   4    patch_data_len     patch_data 字节数
0x24   4    reserved1          = 0
0x28   32   patch_data_sha256  patch_data 的 SHA-256 摘要
0x48   64   signature          ECDSA P-256 签名（r||s，覆盖 header||patch_data）
0x88   ..   patch_data         可选的依赖表（HAS_DEPS 时）+ handler 代码 blob
```

安全设计要点：
- 签名覆盖 `header || patch_data` 的 SHA-256 摘要，确保完整性和真实性
- `patch_data_sha256` 字段提供独立的数据完整性校验（先验证 data hash，再验证签名）
- 要求 `buf.len() == total_len`（精确长度匹配），防止签名未覆盖的隐藏字节
- `MAX_PATCH_BYTES = 64 * 1024`（64KB）限制 patch 镜像最大尺寸
- **编译时安全门控**：`#[cfg(all(feature = "insecure-ecdsa-stub", not(any(test, debug_assertions))))]` 触发 `compile_error!`，确保 insecure stub 不会出现在 release 构建中

#### 3.1.2 依赖管理与拓扑排序（P1-4）

livepatch 实现了基于 SHA-256 UID 的 patch 依赖追踪系统：

**依赖元数据**（`PatchMeta`）：
- `patch_uid: [u8; 32]`：patch 的唯一标识符（取自 `patch_data_sha256`）
- `dep_count: u8`：依赖数量（0..=4，`MAX_PATCH_DEPS = 4`）
- `deps: [[u8; 32]; 4]`：前置依赖的 UID 列表

**依赖表格式**（HAS_DEPS 标志位设置时）：
```
patch_data 前缀：
[0]       dep_count   (1..=4)
[1..33]   dep_uid[0]  (SHA-256)
[33..65]  dep_uid[1]  (如果 dep_count >= 2)
...
```

**依赖检查**：
- `check_dependencies(slot_index)`：验证所有前置依赖处于 `Enabled` 状态，在 enable 前调用
- `check_no_dependents(slot_index)`：验证无活跃 patch 依赖当前 patch，在 disable/unload 前调用
- 自依赖通过 `ct_eq` 常量时间比较检测并拒绝

**拓扑排序**（`topo_sort_patch_ids`）：
- 使用 **Kahn's 算法** 对所有非空 patch 按依赖关系排序
- 返回依赖优先顺序（根节点在前，叶节点在后）
- 检测依赖循环（返回 `EINVAL`）和缺失依赖（返回 `ENOENT`）
- 处于过渡状态的 patch 导致返回 `EBUSY`
- `Failed` 状态的 patch 被跳过，不参与批量操作

**批量操作**：
- `sys_kpatch_enable_all`（syscall 514）：按拓扑顺序启用所有 patch，失败时回滚已启用的 patch
- `sys_kpatch_disable_all`（syscall 515）：按逆拓扑顺序禁用所有 patch，失败时重新启用已禁用的 patch

**元数据存储安全**（P2-3）：
- `PATCH_META_TABLE` 使用 `Mutex<[Option<PatchMeta>; MAX_PATCHES]>` 替代 `static mut`，消除 UB
- 双锁机制（`PATCH_REG_LOCK` + `PATCH_META_TABLE.lock()`）在冷路径上可接受

#### 3.1.3 INT3 单字节 Detour 机制

livepatch 的核心 detour 策略是将目标函数入口的第一个字节替换为 `INT3 (0xCC)`：

1. **安装**（`install_int3_detour`）：
   - 获取 `TEXT_PATCH_LOCK` 序列化并发文本修改
   - 调用 `make_text_writable` 临时解除内核 .text 写保护
   - 保存原始字节（首次安装时记录，重新启用时验证未被篡改）
   - 通过 `atomic_write_u8` 执行单字节原子写入（SMP 安全）
   - `flush_icache` + `sync_cores` 确保所有 CPU 看到新指令
   - 恢复 .text 只读保护

2. **分发**（`breakpoint_dispatch`）：
   - 从 `#BP` 异常处理器调用
   - CPU 推送的 RIP 指向 INT3 之后，因此实际 INT3 地址为 `RIP - 1`
   - 遍历 patch table 查找匹配的 target 地址
   - 递增 `in_flight` 计数器（R103-2 修复：防止卸载时 UAF）
   - 通过 `InterruptStackFrame::update` 将 RIP 重写为 handler 地址

3. **卸载安全**（`kpatch_unload`）：
   - 要求 patch 处于 `Disabled` 状态
   - P1-4：检查无活跃依赖者（`check_no_dependents`）
   - 执行 `sync_cores()` IPI 屏障
   - 有界自旋等待 `in_flight` 计数器归零（最多 10,000 次迭代）
   - 若超时未 quiesce，恢复 exec 地址并返回 `EBUSY`（防 UAF）
   - 清除元数据时持有 `PATCH_REG_LOCK`，防止与并发注册竞争

#### 3.1.4 状态机与生命周期

```
Empty → Loading → Registered → Enabling → Enabled → Disabling → Disabled → (Unload → Empty)
                                                                    ↑
                                                              Failed (终态)
```

关键状态转换均使用 `compare_exchange` 原子操作，确保并发安全。额外的序列化锁：
- `PATCH_REG_LOCK`：序列化注册操作，防止同一 target 的重复 patch 竞争，同时拒绝重复 UID
- `TEXT_PATCH_LOCK`：序列化文本修改，防止并发页权限冲突

#### 3.1.5 Patch ID 溢出保护（P2-8）

`NEXT_PATCH_ID` 使用 `fetch_update` + `checked_add` 替代简单的 `fetch_add`，当 64 位 ID 耗尽时返回 `EOVERFLOW` 而非回绕，同时将 slot 从 `Loading` 重置为 `Empty` 防止泄漏。

#### 3.1.6 审计集成（P1-4）

livepatch 通过回调机制集成到 audit 的 hash chain 中：
- `LivepatchAuditFn` 回调签名：`fn(action, patch_id, target_addr, extra, timestamp)`
- `register_audit_callback()` 在 audit 初始化后由 `main.rs` 注册
- 回调在锁外调用（先复制函数指针再释放锁），避免跨子系统死锁
- **KASLR 保护**（R121-5）：release 构建中审计事件的地址字段被清零，debug 构建保留完整地址
- action 编码：0=load, 1=enable, 2=disable, 3=unload

#### 3.1.7 权限与 LSM 门控

syscall 路径（`sys_kpatch_load/enable/disable/unload`）实施双重权限检查：
1. `is_privileged()`：要求 CAP_ADMIN 或等效特权
2. `check_lsm_kpatch_*`：LSM 策略检查，**默认实现为 fail-closed（返回 EPERM）**

#### 3.1.8 W^X 与可执行内存管理

当 `handler_addr == 0`（handler 代码在 patch_data 中）时：
1. 计算 `handler_offset`（跳过可选的依赖表）
2. `alloc_exec` 分配可执行内存区域
3. 拷贝 handler blob 到分配区域
4. **强制调用 `seal_exec`** 完成 RW → RX 权限转换（W^X 合规）
5. `flush_icache` 刷新指令缓存
6. 验证分配地址在内核高半区（canonical + bit 63 set）

#### 3.1.9 Panic Rollback 策略

`rollback_recent_patches` 提供 best-effort 的崩溃路径回滚：
- 使用 `try_lock` 避免死锁（若锁被持有则放弃）
- 不执行堆分配
- 可跳过 `sync_cores`（其他 CPU 可能已挂起）
- 仅回滚 TSC 时间窗口内的 patch（通过 `enabled_tsc` 判断）
- 读取 TSC 在获取锁之后，防止 TSC 偏差导致新 patch 被误认为过旧
- **限制**：回滚后其他 CPU 可能保留 stale INT3 在指令缓存中，仅适用于系统即将停机的场景

#### 3.1.10 共享 SHA-256 实现（R148-I9）

livepatch 不再包含独立的 `mod sha256` 实现，而是通过 `crypto` crate 复用与 audit 相同的 SHA-256 代码路径：

```rust
mod sha256 {
    pub use crypto::sha256::Sha256;
    pub fn sha256(data: &[u8]) -> [u8; 32] {
        Sha256::digest(data)
    }
}
```

这确保 FIPS KAT 测试的是实际使用的代码路径，避免实现漂移。

#### 3.1.11 签名验证失败语义

| 失败原因 | 返回错误 | 含义 |
|---------|---------|------|
| KAT 未通过 | `ENOSYS` | 密码学子系统不可用 |
| 无可信公钥（全零） | `ENOSYS` | 密码学子系统未配置 |
| 公钥格式无效 | `ENOSYS` | 密码学子系统不可用 |
| 签名不匹配 | `EACCES` | 认证失败 |
| 数据 hash 不符 | `EACCES` | 完整性校验失败 |

---

### 3.2 kernel/livepatch/ecdsa_p256.rs（~451 行）

#### 3.2.1 KAT 状态机

ECDSA 验签模块实现了 FIPS 风格的 Known Answer Test 门控：

```
Uninit → Running → Pass（启用验签）
                 → Fail（永久禁用，sticky）
```

- `run_kat_if_needed()`：幂等调用，首次执行 KAT，后续直接返回缓存结果
- KAT 使用 RFC 6979 Appendix A.2.5 确定性测试向量（私钥 d、消息 "sample"、SHA-256 预哈希）
- 失败后状态永久锁定为 `Fail`，所有后续验签请求返回 `Err(VerifyError::KatFailed)`

#### 3.2.2 多密钥验签 API

核心公共 API 为 `verify_prehash_any()`，接受可信公钥切片进行多密钥验签：

```rust
pub fn verify_prehash_any(
    trusted_pubkeys_uncompressed: &[[u8; 65]],
    digest: &[u8; 32],
    sig_rs: &[u8; 64],
) -> Result<(), VerifyError>
```

- 公钥以 SEC1 未压缩格式传入（65 字节：`04 || X || Y`）
- 全零公钥通过 `ct_is_all_zero()` 常量时间检测并跳过
- 任一公钥验签成功即返回 `Ok(())`

#### 3.2.3 P-256 曲线运算

模块依赖 `p256` 和 `ecdsa` crate，使用 `ecdsa::hazmat::verify_prehashed` 执行底层验签，避免引入 `sha2` crate（在裸机目标上存在 LLVM 问题）。

---

### 3.3 kernel/trace/lib.rs（607 行）

#### 3.3.1 Tracepoint 注册与触发

tracepoint 系统提供了一个轻量级的内核事件追踪框架：

**注册**：
- `register_tracepoint(tp: &'static Tracepoint)` 注册静态 tracepoint 引用
- 最多支持 `MAX_TRACEPOINTS`（**64**）个 tracepoint
- 每个 tracepoint 包含：`TraceId`、名称、`TraceCategory`（8 种：Sched/Syscall/Memory/Ipc/Net/Watchdog/Storage/Security/Custom）、字段名元数据、启用标志（`AtomicBool`）、可选回调（`TraceSink`）
- 回调通过 `Tracepoint::install_callback(cb)` 安装，使用 CAS 防止并发覆盖
- 回调存储为 `AtomicUsize`（R89-3：CHERI/MPK 安全，避免数据指针与函数指针混淆）

**触发热路径**（`Tracepoint::emit`）：
```rust
if !self.is_enabled() {  // 单次 Acquire 原子加载
    return Ok(());
}
// 在栈上构造 TraceRecord 并调用回调
```

关键性能特征：
- 禁用时开销：单次 `Acquire` 原子加载
- 启用时：在栈上构造 TraceRecord 并拷贝 args（无堆分配），然后调用已注册的 sink 回调
- 无论是否有回调，启用时都会递增 `TracepointsEmitted` 计数器

#### 3.3.2 TraceRecord 结构

```rust
pub struct TraceRecord {
    pub id: TraceId,                    // TraceId(u32)
    pub category: TraceCategory,        // 事件分类
    pub name: &'static str,             // 人类可读名称
    pub timestamp: u64,                 // 单调时间戳（毫秒）
    pub cpu: usize,                     // 当前 CPU ID
    pub args: TraceArgs,                // [u64; TRACE_ARGS_MAX] — 最多 6 个参数
    pub arg_count: u8,                  // 有效参数数量
    pub fields: &'static [&'static str], // 参数字段名
}
```

#### 3.3.3 Read Guard（访问控制）

trace 模块实现了可选的 read guard 机制：
- `install_read_guard(guard_fn)` 注册访问控制回调（使用 `AtomicUsize` 存储，R89-3 CHERI/MPK 安全）
- `ensure_metrics_read_allowed()` 在数据读取前检查权限

**重要差异**：与 audit 模块的 fail-closed 设计不同，trace 的 read guard **在未安装时默认放行（fail-open）**。

#### 3.3.4 声明宏

`declare_tracepoint!` 宏简化静态 tracepoint 声明：
```rust
declare_tracepoint!(
    pub SCHED_SWITCH, 0x0001, TraceCategory::Sched,
    ["prev_pid", "next_pid", "prev_state"]
);
```

---

### 3.4 kernel/trace/profiler.rs（~433 行）

#### 3.4.1 Per-CPU Ring Buffer 采样器

**数据结构**：
```rust
pub struct ProfilerSample {
    pub timestamp_ns: u64,  // 单调时间（纳秒）
    pub pid: u64,           // 当前进程 PID
    pub rip: u64,           // 采样时的指令指针
    pub cpu: u32,           // CPU ID
    _reserved: u32,         // 对齐填充
}
```

`ProfilerSample` 为 32 字节（`#[repr(C)]`），最小化高频采样时的缓存影响。

**Ring Buffer 设计**：
- 每个 CPU 独立的 ring buffer（避免跨核缓存行争用）
- 满时采用 **drop-oldest** 语义
- 每个槽位使用 **seqlock** 发布

**控制面**：
- `PROFILER_CONTROL_LOCK`：序列化 start/stop/snapshot 操作
- `record_pc_sample()`：数据面热路径，**不获取控制锁**

---

### 3.5 kernel/trace/watchdog.rs（~525 行）

#### 3.5.1 Hung-Task 看门狗

**核心机制**：
- 每个被监控的任务通过 `register_watchdog(cfg, now_ms)` 获得 `WatchdogHandle`
- 定期调用 `heartbeat(handle, now_ms)` 重置超时
- 定时器中断中通过 `poll_watchdogs(now_ms)` 检查超时

**安全设计**：
- **One-shot trip**：首次超时触发后设置 `tripped` 标志，防止重复告警
- **Heartbeat 恢复**：超时后若收到 heartbeat，清除 `tripped`
- **Generation/ABA 防护**：handle 包含 generation 计数器
- **最小超时保护**：`MIN_TIMEOUT_MS` 防止误报

---

### 3.6 kernel/trace/kdump.rs（~964 行）

#### 3.6.1 加密崩溃转储

**加密方案**：
- ChaCha20 流加密（仅保密性）
- 每次 dump 单次 44 字节 RNG 调用生成密钥+Nonce
- 输出格式：`--BEGIN ZOS-KDUMP--` / `--END ZOS-KDUMP--`

**KptrGuard 指针脱敏**：
- 内核指针经 `KptrGuard::from_addr(val).obfuscated_value()` 脱敏
- ASCII-hex 指针也被扫描和脱敏

**安全失败策略**：

| 条件 | 行为 | 原因 |
|------|------|------|
| RNG 不可用 | 不输出（无明文回退） | 防止未加密泄露 |
| FIPS Enabled | 抑制输出 | ChaCha20 非 FIPS 批准 |
| FIPS Failed | 抑制输出 | 保守处理 |
| 空 dump | 不输出且不消耗 emit-once 标志 | 允许后续有效 dump |

---

### 3.7 kernel/trace/counters.rs（~290 行）

15 种预定义的内核性能计数器，per-CPU 无锁设计（`CpuLocal` + `Relaxed fetch_add`）。快照聚合使用 `Acquire` 加载 + `saturating_add`，受 read guard 保护。

---

### 3.8 kernel/audit/lib.rs（~2650 行）

#### 3.8.1 架构概述

- **防篡改**：SHA-256 hash chain（R148-I9：使用共享 `crypto` crate 实现）
- **IRQ 安全**：所有锁操作通过 `interrupts::without_interrupts()` 包裹
- **固定 Ring Buffer**：有界内存使用，溢出时逐出最旧事件
- **零分配热路径**：事件发射路径避免堆分配
- **强制开启**：`disable()` 为 no-op

#### 3.8.2 时间戳回调（R148-I8）

audit 支持注册时间戳提供者回调：
- `register_timestamp_callback(kernel_core::time::current_timestamp_ms)` 在 audit 初始化后注册
- 解决了先前审计事件 timestamp 字段为 0 的问题

#### 3.8.3 HMAC 密钥安全

- 密钥最小 16 字节（`MIN_HMAC_KEY_SIZE`，R110-4：FIPS 140-2 要求 >= 128 位）
- 密钥最大 32 字节（`MAX_HMAC_KEY_SIZE`）
- 只能设置一次（`KeyAlreadySet` 错误）
- 中间缓冲区通过 `secure_zeroize` 清零（volatile write + compiler fence）

#### 3.8.4 事件分类体系

**AuditKind**（8 种）：Syscall、Ipc、Fs、Process、Signal、Security、Network、Internal

**安全事件子分类**（`AuditSecurityClass`，4 种）：
- `Lsm`（LSM hook 拒绝）
- `Seccomp`（seccomp/pledge 过滤）
- `Capability`（能力生命周期）
- `CgroupDelegation`（P1-3：cgroup 委派操作）

#### 3.8.5 公共 Crypto API

```rust
pub mod crypto {
    pub fn sha256_digest(data: &[u8]) -> [u8; 32];
    pub fn hmac_sha256_digest(key: &[u8], msg: &[u8]) -> [u8; 32];
    pub struct StreamingSha256 { ... }
}
```

供 `compliance` 模块执行 FIPS KAT。

#### 3.8.6 路径哈希工具

- `hash_path(path)` — SHA-256 截断至 64 位，带 `"AUDIT-PATH-SHA256-V1:"` domain separator
- `hash_bytes(data)` — SHA-256 截断至 64 位，带 `"AUDIT-BYTES-SHA256-V1:"` domain separator
- `hash_binary_prefix(data, max_len)` — 二进制前缀哈希（防 argv[0] 欺骗）

---

### 3.9 kernel/compliance/lib.rs（912 行）

#### 3.9.1 PolicySurface（G.fin.1）

compliance 模块新增了 boot-time locked 的 `PolicySurface` 结构，作为所有子系统安全策略决策的**单一来源**：

```rust
pub struct PolicySurface {
    pub profile: HardeningProfile,
    pub source: ProfileSource,              // Default/BootCmdline/Runtime
    pub panic_redact_details: bool,         // Secure: true
    pub kaslr_fail_closed: bool,            // Secure: true
    pub kpti_fail_closed: bool,             // Secure: true
    pub audit_ring_capacity: usize,         // 256/128/64
    pub debug_interfaces_enabled: bool,     // Secure: false
    pub spectre_mitigations: bool,          // Secure/Balanced: true
    pub kptr_guard: bool,                   // Secure/Balanced: true
    pub strict_wxorx: bool,                 // Secure: true
}
```

**生命周期**：
- `init_policy_surface(profile, source)` 在 boot 期间调用一次，构建并锁定 PolicySurface
- 重复调用同一 profile/source 是幂等的，不同值会 panic
- 设置 `ACTIVE_PROFILE` 原子值并阻止后续 `set_profile()` 更改
- `policy()` 返回只读引用，未初始化时 panic

**审计集成**：
- `emit_profile_validation_audit_event()` 发射策略验证事件
- `emit_deferred_policy_audit()` 在 audit 初始化后重新发射（因初次发射时 audit 未就绪）

**ProfileSource**（审计溯源）：
- `Default`：内核内置默认
- `BootCmdline`：启动命令行 `profile=secure` 显式选择
- `Runtime`：运行时/测试环境设置

#### 3.9.2 Hardening Profile

三级安全加固配置（与旧版一致），现在通过 `PolicySurface` 暴露完整的策略字段集。

| 配置项 | Secure | Balanced | Performance |
|--------|--------|----------|-------------|
| W^X 策略 | strict（违规 panic） | warn（仅警告） | off |
| Spectre 缓解 | 全部启用 | 全部启用 | 禁用 |
| kptr 混淆 | 启用 | 启用 | 禁用 |
| NX 保护 | 启用 | 启用 | 启用（始终） |
| 审计容量 | 256 | 128 | 64 |
| 安全自测 | 启用 | 禁用 | 禁用 |
| Panic 输出抑制 | 启用 | 禁用 | 禁用 |
| KASLR fail-closed | 启用 | 禁用 | 禁用 |
| 调试接口 | 禁用 | 启用 | 启用 |

**Fail-Closed 回退**（R94-4）：
```rust
pub fn current_profile() -> HardeningProfile {
    match ACTIVE_PROFILE.load(Ordering::Acquire) {
        0 => HardeningProfile::Secure,
        1 => HardeningProfile::Balanced,
        2 => HardeningProfile::Performance,
        _ => HardeningProfile::Secure, // 内存损坏时回退到最严格
    }
}
```

#### 3.9.3 FIPS 模式

FipsState 和 `fips_state()` 现已迁移到 `security::fips` 模块（R140-6），通过 `pub use` 重新导出，避免 RNG 模块循环依赖。

**并发安全**（R93-1）：
- `FIPS_ENABLING` 原子标志序列化并发 enable 尝试
- `EnableGuard` RAII 确保所有退出路径释放锁

#### 3.9.4 FIPS KAT 编排

compliance 模块编排三项 Known Answer Test：

1. **SHA-256 KAT**（`kat_sha256`）：
   - 空消息、"abc" 短消息、1,000,000 个 'a' 流式测试
   - 所有比较使用常量时间 `ct_eq`

2. **HMAC-SHA256 KAT**（`kat_hmac_sha256`）：
   - 64 字节 key、32 字节 key、100 字节 key（测试密钥哈希路径）

3. **ECDSA P-256 KAT**（`kat_ecdsa_p256`）：
   - 委托给 `livepatch::ecdsa_p256::run_kat_if_needed()`

---

### 3.10 kernel/klog/lib.rs（265 行）

#### 3.10.1 Profile-Aware 日志系统

klog 提供四种宏，覆盖从调试到关键输出的全部场景：

| 宏 | Release 构建 | Profile 感知 | 用途 |
|---|---|---|---|
| `kprintln!` | 编译移除 | 否 | 调试诊断（替代 `println!`） |
| `klog!` | 活跃 | 是 | 运行时日志（5 级：Trace/Debug/Info/Warn/Error） |
| `klog_always!` | 活跃 | Secure 时抑制 | 启动标语、状态消息 |
| `klog_force!` | 活跃 | 否 | Pre-panic 诊断、关键错误 |

**过滤策略**：
- **Secure**：全部抑制（零信息泄露）
- **Balanced**：仅 Error + Warn
- **Performance**：全部级别

热路径为单次 `Relaxed` 原子加载 + 整数比较。

#### 3.10.2 安全设计

- `LOG_ALWAYS_ENABLED` 原子标志默认 `false`（fail-closed），直到 `set_profile()` 调用
- Secure profile 下 `klog_always!` 被抑制，最小化信息泄露面
- `klog_force!` 永不抑制，仅用于安全策略执行消息和关键硬件错误
- klog 不依赖 `security`（kptr_guard）以避免循环依赖，调用者必须自行包装指针

---

## 4. 模块整体评估

### 4.1 安全设计评价

**优秀实践**：

1. **Fail-Closed 一致性**：audit（未注册 authorizer → AccessDenied）、compliance（内存损坏 → Secure/Failed）、livepatch（未接入 LSM → EPERM）、klog（未 set_profile → 全部抑制）均采用 fail-closed 设计。唯一例外是 trace 的 read guard（fail-open），需要运维注意。

2. **密钥材料生命周期管理**：audit 的 HMAC 密钥实现了完整的安全生命周期——只能设置一次、最小 128 位（R110-4）、`Drop` 时 volatile 清零、HMAC 中间缓冲区 `secure_zeroize`。

3. **防篡改审计链**：SHA-256 hash chain + domain separation + 可选 HMAC + 时间戳回调（R148-I8）提供了完整的审计保障。

4. **W^X 合规**：livepatch 的可执行内存管理严格遵循 W^X 原则。

5. **KAT 架构**：compliance 不重复实现密码学，而是调用实际使用的代码路径进行自测。livepatch 和 audit 共享 `crypto` crate 的 SHA-256 实现（R148-I9），彻底消除实现漂移。

6. **依赖拓扑排序**（P1-4）：Kahn's 算法实现的拓扑排序确保 patch 按正确的依赖顺序启用/禁用，批量操作失败时自动回滚。

7. **PolicySurface 单一来源**（G.fin.1）：boot-time locked 的策略结构确保所有子系统查询同一策略，消除策略漂移。

8. **KASLR 地址保护**（R120-2/R121-5）：livepatch 日志和审计事件在 release 构建中不暴露内核地址。

**需要关注的风险点**：

1. **trace read guard 默认 fail-open**：如果 boot 期间未安装 read guard，所有 trace/profiler/counter 数据对任何调用者可见。

2. **kdump 仅加密无认证**：ChaCha20 流加密不提供完整性保护。

3. **livepatch panic rollback 的局限性**：best-effort 回滚可能留下 stale INT3，仅适用于系统即将停机的场景。

### 4.2 代码质量评价

| 维度 | 评分 | 说明 |
|------|------|------|
| 文档完整性 | ★★★★★ | 每个模块都有详尽的 rustdoc，包含安全设计说明和变更历史 |
| 错误处理 | ★★★★★ | 所有公共 API 返回 `Result`，错误类型语义清晰 |
| 并发安全 | ★★★★★ | 原子操作 + spin lock + IRQ disable + RAII guard（FIPS EnableGuard） |
| 测试覆盖 | ★★★★☆ | audit 有完整单元测试，compliance 有 KAT，klog 的 profile 行为依赖集成测试 |
| 内存安全 | ★★★★★ | `PATCH_META_TABLE` 从 `static mut` 迁移到 `Mutex`（P2-3），消除 UB |
| 性能设计 | ★★★★★ | per-CPU 无锁计数器、单原子加载快速路径、klog 热路径单次 Relaxed load |

### 4.3 模块间依赖关系图

```
                    ┌──────────────┐
                    │  compliance  │
                    │ (PolicySurface│
                    │  + 策略引擎)  │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │  audit   │ │ livepatch│ │  trace   │
        │(审计日志) │ │(热补丁)  │ │(可观测性) │
        └────┬─────┘ └────┬─────┘ └──────────┘
             │            │
             │   ┌────────┘ livepatch 审计回调
             │   │
             ▼   ▼
        共享 crypto crate (SHA-256)
        compliance KAT → audit::crypto + livepatch::ecdsa_p256

        ┌──────────┐
        │   klog   │ ← compliance 驱动 profile
        │(结构化日志)│ ← 所有子系统使用
        └──────────┘
```

### 4.4 运维建议

1. **Boot 检查清单**：确保以下初始化步骤按序完成：
   - `compliance::init_policy_surface(profile, source)` → `compliance::lock_profile()`
   - `klog::set_profile()` 绑定日志级别
   - `audit::init(capacity)` → 注册 authorizer → `audit::set_hmac_key()` → `audit::register_timestamp_callback()`
   - `compliance::emit_deferred_policy_audit()` 重发策略验证事件
   - `livepatch::register_audit_callback()` 接入审计链
   - `trace::install_read_guard()` （**必须**，否则 trace 数据无访问控制）

2. **PolicySurface 验证**：boot 日志中确认 PolicySurface enforcement 摘要输出，验证所有安全特性符合预期。

3. **监控指标**：通过 `trace::snapshot_counters()` 定期采集 15 种内核计数器。

4. **FIPS 合规部署**：启用 FIPS 模式前确认所有使用非 FIPS 算法的子系统已接入 `is_algorithm_permitted()` 检查。
