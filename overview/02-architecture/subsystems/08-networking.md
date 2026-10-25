# Zero-OS net 网络栈模块分析文档

分析日期：2026-05-13

## 一、模块概述

`net` 模块是 Zero-OS 内核的完整网络协议栈实现，位于 `kernel/net/src/` 目录下。该模块基于 `no_std` 环境构建，使用 `alloc` crate 进行动态内存分配，实现了从链路层（Ethernet）到传输层（TCP/UDP）的完整协议处理链路。

模块总代码量约 **20,152 行**（含 Cargo.toml），涵盖 17 个文件（16 个 `.rs` 源文件 + 1 个 `Cargo.toml`）。核心设计理念为 **安全优先（Security-First）**，在每个协议层均内置了严格的输入验证、速率限制、DoS 防护和资源边界控制。

### 核心特性

- **完整协议栈**：Ethernet → ARP → IPv4 → ICMP/UDP/TCP，支持 IP 分片重组
- **安全加固**：默认 DROP 防火墙策略、SYN Cookie、RFC 5961 Challenge ACK 与严格 RST 验证、速率限制、IOMMU DMA 隔离
- **能力安全（Capability-Based）Socket API**：基于 `cap` crate 的命名空间隔离，LSM 钩子集成
- **有状态防火墙**：per-namespace conntrack 连接跟踪 + 优先级规则匹配，支持 ESTABLISHED/RELATED 状态放行
- **TCP SACK 支持（RFC 2018/6675）**：选择性确认发送/接收，OOO 段缓存与重组，SACK 丢失检测与重传
- **TCP Keepalive（RFC 1122）**：可配置空闲探测，定时器驱动的活性检测
- **VirtIO-net 驱动**：PCI Modern Transport，DMA 缓冲区生命周期管理，恶意设备防护
- **Per-Namespace 隔离**：防火墙规则表、conntrack 表、分片重组缓存均按网络命名空间隔离
- **资源边界控制**：全局/每源 IP 分片限制、每命名空间 Socket 配额、TCP 连接数上限、全局 UDP 内存上限

### 依赖关系

| 依赖 crate | 用途 |
|-----------|------|
| `spin` | 自旋锁（Mutex/RwLock/Once） |
| `mm` | 内存管理，DMA 缓冲区分配（`alloc_dma_buffer`） |
| `virtio` | VirtIO 设备抽象（队列、特性协商） |
| `drivers` | 串口输出（`println!` 宏） |
| `x86_64` | 物理/虚拟地址类型（`PhysAddr`/`VirtAddr`） |
| `cap` | 能力安全原语（`NamespaceId`、`ProcessCtx`） |
| `lsm` | Linux Security Module 风格钩子 |
| `security` | 安全策略框架 |
| `iommu` | IOMMU DMA 隔离 |
| `klog` | 内核日志宏（`klog!`、`klog_always!`） |

### Feature 标志

- `conntrack`（默认启用）：启用有状态连接跟踪，为防火墙提供 ESTABLISHED/RELATED 状态判定

---

## 二、文件结构

| 文件 | 行数 | 职责简述 |
|------|------|---------|
| `Cargo.toml` | 20 | 包元数据、依赖声明、feature 定义 |
| `lib.rs` | 465 | 模块根：子模块声明、公共 re-export、网络常量、设备注册表、MMIO 映射、初始化 |
| `ethernet.rs` | 214 | Ethernet II 帧解析与构建 |
| `ipv4.rs` | 561 | IPv4 报文解析、头部构建、校验和计算 |
| `arp.rs` | 999 | ARP 协议：缓存管理、防投毒、per-interface 速率限制 |
| `icmp.rs` | 638 | ICMP 协议：Echo 处理、速率限制令牌桶 |
| `tcp.rs` | 3,068 | TCP 协议：状态机、拥塞控制、SYN Cookie、窗口缩放、SACK、OOO 重组、Keepalive |
| `udp.rs` | 515 | UDP 协议：解析、校验和、构建 |
| `socket.rs` | 7,145 | Socket 层：能力安全 API、LSM 集成、TCP 被动/主动打开、连接管理、全局 UDP 内存上限 |
| `firewall.rs` | 809 | 有状态防火墙：per-namespace 规则表、优先级规则、CIDR 匹配、conntrack 集成 |
| `conntrack.rs` | 1,256 | 连接跟踪：TCP/UDP/ICMP 状态机、per-namespace 配额、LRU 淘汰、超时清理 |
| `fragment.rs` | 1,104 | IP 分片重组：RFC 815 空洞算法、RFC 5722 重叠检测、per-namespace 隔离、DoS 防护 |
| `buffer.rs` | 525 | 网络缓冲区：DMA 兼容 NetBuf、BufPool 缓冲池 |
| `device.rs` | 366 | 网络设备抽象：NetDevice trait、能力协商、链路状态 |
| `virtio_net.rs` | 911 | VirtIO-net 驱动：PCI Modern Transport、DMA 管理、恶意设备防护 |
| `pci.rs` | 370 | PCI 总线扫描：VirtIO-net 设备探测、BAR 读取、IOMMU 绑定 |
| `stack.rs` | 1,186 | 协议栈主循环：帧处理入口、协议分发、TX 路径、防火墙裁决、定时器 |

---

## 三、逐文件分析

### 3.1 `Cargo.toml`（20 行）

#### 功能说明

定义 `net` crate 的包元数据和依赖关系。

#### 关键配置

```toml
[features]
default = ["conntrack"]
conntrack = []
```

`conntrack` 作为默认 feature 启用有状态连接跟踪。该 feature 可通过编译时关闭以减少代码体积，但会丧失有状态防火墙能力。

#### 依赖分析

10 个依赖 crate 中，`mm`、`virtio`、`drivers`、`cap`、`lsm`、`security`、`iommu`、`klog` 均为内核内部 path 依赖；`spin` 和 `x86_64` 为外部 crate。依赖链清晰，无循环依赖风险。

---

### 3.2 `lib.rs`（465 行）

#### 功能说明

模块根文件，负责子模块声明、公共符号 re-export、网络常量定义、全局设备注册表和 MMIO 映射基础设施。

#### 核心数据结构

- **`NetDeviceRegistry`**：全局网络设备注册表，`RwLock<Vec<RegisteredDevice>>` 保护，最多支持 `MAX_NET_DEVICES = 8` 个设备
- **`RegisteredDevice`**：注册设备条目，包含名称、索引和 `Arc<Mutex<Box<dyn NetDevice>>>` 句柄

#### 核心函数

| 函数 | 说明 |
|------|------|
| `register_device()` | 注册网络设备，检查重名和数量上限 |
| `get_device()` | 按名称获取设备句柄 |
| `get_device_by_index()` | 按索引获取设备句柄 |
| `init()` | 网络栈初始化入口：PCI 探测 → MMIO 映射 → VirtIO 设备创建 → 设备注册 |
| `map_pci_mmio()` | 将物理 MMIO 区域映射到内核虚拟地址空间（安全加固后身份映射为只读） |
| `map_virtio_pci_regions()` | 映射 VirtIO PCI 设备的所有 capability 区域 |

#### 网络常量

```rust
pub const DEFAULT_MTU: usize = 1500;
pub const DEFAULT_HEADROOM: usize = 64;   // Eth(14) + IP(20) + TCP(20) = 54, 对齐到 64
pub const DEFAULT_TAILROOM: usize = 64;
pub const VIRTIO_NET_HDR_SIZE: usize = 12;
pub const ETH_HEADER_SIZE: usize = 14;
pub const ETH_MIN_FRAME_SIZE: usize = 60;
pub const ETH_MAX_FRAME_SIZE: usize = 1514;
pub const MAX_NET_DEVICES: usize = 8;
```

#### MMIO 映射基础设施

安全加固后身份映射（identity mapping）变为只读，因此 PCI MMIO 区域需要显式映射到 `NET_MMIO_VIRT_BASE = 0xffff_ffff_5000_0000` 开始的 64 MB 虚拟地址空间。MMIO 偏移使用 `AtomicU64` 原子分配，失败时记录错误并禁用设备 Bus Master（R82-4 修复）。

---

### 3.3 `ethernet.rs`（214 行）

#### 功能说明

Ethernet II 帧的解析与构建，提供链路层基础设施。

#### 核心数据结构

- **`EthAddr`**：6 字节 MAC 地址，提供 `is_broadcast()`、`is_multicast()`、`is_unicast()`、`is_local()` 判断，含 `ZERO` 和 `BROADCAST` 常量
- **`EthHeader`**：解析后的以太网帧头（dst MAC + src MAC + EtherType）
- **`EthError`**：`Truncated`（帧过短）、`UnsupportedProtocol`

#### 核心函数

| 函数 | 说明 |
|------|------|
| `parse_ethernet()` | 解析原始帧字节 → `(EthHeader, &[u8] payload)` |
| `build_ethernet_frame()` | 构建完整以太网帧（header + payload） |

#### EtherType 常量

```rust
pub const ETHERTYPE_IPV4: u16 = 0x0800;
pub const ETHERTYPE_ARP: u16 = 0x0806;
```

---

### 3.4 `ipv4.rs`（561 行）

#### 功能说明

IPv4 协议处理：报文解析、头部构建、校验和计算。实现严格的安全验证。

#### 核心数据结构

- **`Ipv4Addr`**：4 字节 IPv4 地址，提供 `is_broadcast()`、`is_multicast()`、`is_loopback()`、`is_unspecified()`、`is_valid_source()` 等判断，含 `UNSPECIFIED`、`LOCALHOST`、`BROADCAST` 常量
- **`Ipv4Header`**：解析后的 IPv4 头部，包含 version、ihl、total_len、identification、flags_fragment、ttl、protocol、checksum、src、dst、options_len。提供 `is_fragment()`、`fragment_offset()`、`more_fragments()`、`dont_fragment()`、`proto()` 方法
- **`Ipv4Proto`**：协议号枚举（ICMP=1, TCP=6, UDP=17）
- **`Ipv4Error`**：详细错误类型（Truncated、BadVersion、BadIhl、BadTotalLen、ChecksumMismatch、SourceRoutingForbidden、InvalidSource、InvalidTtl）

#### 核心函数

| 函数 | 说明 |
|------|------|
| `parse_ipv4()` | 解析 IPv4 报文，验证版本、IHL、校验和、长度一致性、TTL、源地址、源路由选项 |
| `build_ipv4_header()` | 构建 20 字节 IPv4 头部（含校验和计算） |
| `compute_checksum()` | RFC 1071 互联网校验和算法 |
| `calculate_checksum_with_pseudo()` | 带伪头部的校验和计算（用于 TCP/UDP） |

#### 安全特性

- 严格验证 version == 4、IHL >= 5、total_len >= IHL*4
- 校验和强制验证（不跳过）
- TTL 检查（TTL == 0 返回 InvalidTtl）
- **源地址验证加固（R44-3）**：拒绝广播、组播、未指定、回环、0/8 网络和 .255 结尾的源地址
- **源路由选项拒绝**：检测并拒绝 LSRR（0x83）和 SSRR（0x89）选项

---

### 3.5 `arp.rs`（999 行）

#### 功能说明

RFC 826 ARP 协议实现，包含安全加固的 ARP 缓存管理。

#### 核心数据结构

- **`ArpPacket`**：解析后的 ARP 报文（sender_hw/ip、target_hw/ip、op）
- **`ArpCache`**：ARP 缓存表，`VecDeque<ArpEntry>` 存储，支持 LRU 淘汰
  - 最大条目数：`DEFAULT_CACHE_MAX_ENTRIES = 256`
  - TTL：`DEFAULT_CACHE_TTL_MS = 300,000`（5 分钟）
  - **R102-12 修复**：内含 per-interface `rx_rate_limiter` 和 `tx_rate_limiter`（`TokenBucket`），防止单一恶意主机耗尽全局令牌桶
- **`ArpEntry`**：缓存条目（ip、mac、kind、updated_at），区分 Static/Dynamic
- **`ArpStats`**：统计计数器（rx_packets、rx_requests、rx_replies、tx_replies、rx_errors、rx_rate_limited、cache_conflicts、cache_hits、cache_misses）

#### 核心函数

| 函数 | 说明 |
|------|------|
| `parse_arp()` | 解析 ARP 报文，验证硬件/协议类型和地址长度 |
| `process_arp()` | ARP 报文处理主入口：per-interface + 全局速率限制 → 缓存更新 → 应答生成 |
| `build_arp_reply()` | 构建 ARP 应答帧 |
| `build_arp_request()` | 构建 ARP 请求帧 |
| `build_gratuitous_arp()` | 构建免费 ARP（地址冲突检测） |
| `serialize_arp()` | 序列化 ARP 报文为 28 字节 |

#### 安全特性

- **双层速率限制（R102-12）**：per-interface 和全局令牌桶双重控制，RX 处理 50 pps / burst 100，TX 应答 20 pps / burst 40
- **防投毒**：缓存冲突检测，拒绝修改已有条目 MAC 的更新（`CacheConflict` 错误）
- **静态条目保护**：Static 类型条目永不被动态学习覆盖
- **R62-5 修复**：缓存满时只淘汰 Dynamic 条目，永不淘汰 Static 条目
- **源验证**：拒绝广播/组播/全零 MAC 和全零 IP 作为发送方
- **反射攻击防护**：拒绝 sender_ip == our_ip 但 sender_hw != our_mac 的报文
- **R65-7 修复**：仅从 ARP Reply（或允许的 gratuitous ARP）学习地址映射，不从普通 ARP Request 学习
- **R101-11 修复**：gratuitous ARP 不更新静态条目（保护网关等关键绑定）
- **R45 修复**：先清除过期条目再检查冲突，丢弃非目标为我方的 ARP Reply

---

### 3.6 `icmp.rs`（638 行）

#### 功能说明

ICMP 协议处理，包含 Echo Request/Reply、Destination Unreachable、Time Exceeded 等消息类型。

#### 核心数据结构

- **`IcmpHeader`**：解析后的 ICMP 头部（icmp_type、code、checksum、rest_of_header）
- **`TokenBucket`**：通用令牌桶速率限制器，用于 ICMP 和 ARP 速率控制
  - 参数：rate（令牌/秒）、burst（最大突发）
  - 使用 CAS 原子操作确保 SMP 安全
- **`IcmpError`**：错误类型（Truncated、ChecksumInvalid、InvalidType 等）

#### 核心函数

| 函数 | 说明 |
|------|------|
| `parse_icmp()` | 解析 ICMP 报文，验证校验和 |
| `build_echo_reply()` | 构建 Echo Reply 报文 |
| `build_dest_unreachable()` | 构建 Destination Unreachable 报文 |
| `build_time_exceeded()` | 构建 Time Exceeded 报文 |

#### 安全特性

- **速率限制**：全局 ICMP 速率限制器 `ICMP_RATE_LIMITER`，防止 ICMP 洪泛
- **Smurf 攻击防护**：在 `stack.rs` 中实现，不对广播/组播目的地址回复 Echo Reply
- **校验和强制验证**

#### ICMP 类型常量

```rust
pub const ICMP_TYPE_ECHO_REPLY: u8 = 0;
pub const ICMP_TYPE_DEST_UNREACHABLE: u8 = 3;
pub const ICMP_TYPE_ECHO_REQUEST: u8 = 8;
pub const ICMP_TYPE_TIME_EXCEEDED: u8 = 11;
```

---

### 3.7 `tcp.rs`（3,068 行）

#### 功能说明

RFC 793 兼容的 TCP 协议实现，是模块中代码量第二大的文件（仅次于 socket.rs）。包含完整的 TCP 状态机、拥塞控制（RFC 5681）、SYN Cookie（RFC 4987）、窗口缩放（RFC 7323）、**SACK（RFC 2018/6675）**、**乱序段缓存与重组**、**TCP Keepalive（RFC 1122）** 和安全加固。

#### 核心数据结构

- **`TcpState`**：RFC 793 TCP 状态枚举（Closed → Listen → SynSent → SynReceived → Established → FinWait1/2 → CloseWait → Closing → LastAck → TimeWait）
- **`TcpControlBlock`**：TCP 控制块（TCB），包含：
  - 发送序列空间：iss、snd_una、snd_nxt、snd_wnd、snd_wl1、snd_wl2
  - 接收序列空间：irs、rcv_nxt、rcv_wnd
  - 拥塞控制：cwnd、ssthresh、dup_ack_count、congestion_state、recover
  - 窗口缩放：snd_wscale、rcv_wscale、wscale_requested、wscale_received
  - **SACK 状态（R115-3）**：sack_requested、sack_received、highest_sacked、ooo_bytes
  - **乱序队列**：ooo_queue（`VecDeque<OooSegment>`），上限 `TCP_OOO_MAX_SEGMENTS = 64`
  - 重传状态：rto_ms、srtt_us、rttvar_us、retries
  - 缓冲区：send_buffer（带 `TCP_MAX_SEND_BUFFER_BYTES = 4 MB` 上限）、recv_buffer
  - **Keepalive 状态（R148-I3）**：keepalive_enabled、keepalive_idle_ms（默认 7200s）、keepalive_interval_ms（默认 75s）、keepalive_probes_max（默认 9）、keepalive_probes_sent
- **`TcpCongestionState`**：拥塞控制状态（SlowStart、CongestionAvoidance、FastRecovery）
- **`TcpConnKey`**：4 元组连接键（local_ip、local_port、remote_ip、remote_port）
- **`TcpHeader`**：解析后的 TCP 头部
- **`TcpOptions`**：TCP 选项（MSS、WindowScale、SackPermitted、**Sack blocks**、Timestamps）
- **`SackBlock`**：SACK 块结构（left_edge、right_edge），最多 `TCP_SACK_MAX_BLOCKS = 4` 个
- **`TcpSegment`**：发送缓冲区段（seq_num、data、flags、retrans_count、**sacked**、**lost**）
- **`OooSegment`**：乱序接收段（seq、data、fin），用于 OOO 队列
- **`SynCookieData`**：SYN Cookie 解码数据

#### SYN Cookie 实现

```rust
pub const TCP_SYN_COOKIE_MSS_TABLE: [u16; 8] = [536, 1024, 1220, 1460, 4312, 8960, 16384, 32768];
pub const TCP_SYN_COOKIE_MAX_AGE_MS: u64 = 120_000;  // 120 秒最大年龄
pub const TCP_SYN_COOKIE_TIME_GRANULARITY_MS: u64 = 4_000;
```

- `generate_syn_cookie_isn()`：生成 SYN Cookie ISN，编码时间戳、MSS 索引和 MAC
- `validate_syn_cookie()`：验证 SYN Cookie，检查时间窗口和 MAC 完整性
- `syn_cookie_select_mss()`：从 Cookie 中恢复 MSS 值

#### 拥塞控制（RFC 5681）

- **慢启动**：cwnd 指数增长直到 ssthresh
- **拥塞避免**：cwnd 线性增长（每 RTT 增加约 1 MSS）
- **快速重传**：3 个重复 ACK 触发
- **快速恢复（NewReno，R55-1）**：partial ACK 处理，recover 点跟踪
- **有限传输（RFC 3042，R56-1）**：前 2 个 dup ACK 时发送新数据

#### SACK 实现（RFC 2018/6675，R115-3）

- **`sack_enabled()`**：双方均在 SYN/SYN-ACK 中交换 SACK-Permitted 后启用
- **`ooo_insert()`**：将乱序段插入 OOO 队列，自动合并重叠/相邻段，受窗口预算和 `TCP_OOO_MAX_SEGMENTS` 限制（R113-2 修复）
- **`ooo_drain_contiguous()`**：从 OOO 队列前端取出可拼接数据推进 rcv_nxt，处理 FIN 段（R133-3 修复）
- **`generate_sack_blocks()`**：从 OOO 队列生成最多 4 个 SACK 块（最近插入的优先）
- **`process_sack_blocks()`**：处理对端 SACK 块更新发送方记分板，O(n + m log m) 算法（R115-3 修复：从 O(n*m) 优化）
- **`sack_mark_lost()` / `sack_pipe()` / `sack_find_lost_segment()`**：RFC 6675 丢失检测与重传

#### OOO 段安全加固

- **R113-2 修复**：OOO 段总字节数受通告接收窗口约束，防止攻击者发送超窗口 OOO 段膨胀内存
- **R133-3 修复**：OOO 段中 FIN 处理修正；超出 FIN 位置的 OOO 段数据被截断清理，防止注入 FIN 之后的数据
- **R144 修复**：OOO 合并时正确处理 FIN 位置截断，防止合并扩展超过 FIN 的数据范围

#### 窗口缩放（RFC 7323）

```rust
pub const TCP_MAX_WINDOW_SCALE: u8 = 14;
pub const TCP_MAX_SCALED_WINDOW: u32 = (u16::MAX as u32) << TCP_MAX_WINDOW_SCALE;
pub const TCP_DEFAULT_RCV_WINDOW_BYTES: u32 = 256 * 1024;  // 256 KB
```

#### 关键安全常量

```rust
pub const TCP_MAX_RETRIES: u8 = 15;
pub const TCP_INITIAL_RTO_MS: u64 = 1000;
pub const TCP_MIN_RTO_MS: u64 = 1000;
pub const TCP_MAX_RTO_MS: u64 = 120_000;
pub const TCP_TIME_WAIT_MS: u64 = 120_000;      // 2*MSL
pub const TCP_FIN_WAIT_2_TIMEOUT_MS: u64 = 60_000;
pub const TCP_SYN_TIMEOUT_MS: u64 = 30_000;
pub const TCP_MAX_SYN_BACKLOG: usize = 128;
pub const TCP_MAX_ACCEPT_BACKLOG: usize = 128;
pub const TCP_MAX_ACTIVE_CONNECTIONS: usize = 4096;
pub const TCP_MAX_SEND_SIZE: usize = 64 * 1024;
pub const TCP_MAX_SEND_BUFFER_BYTES: usize = 4 * 1024 * 1024;  // 4 MB
pub const TCP_INITIAL_SSTHRESH: u32 = 64 * 1024;
pub const TCP_OOO_MAX_SEGMENTS: usize = 64;
pub const TCP_SACK_MAX_BLOCKS: usize = 4;
```

---

### 3.8 `udp.rs`（515 行）

#### 功能说明

RFC 768 UDP 协议实现，安全优先设计。

#### 核心数据结构

- **`UdpHeader`**：解析后的 UDP 头部（src_port、dst_port、length、checksum）
- **`UdpError`**：错误类型（Truncated、LengthMismatch、ChecksumInvalid、ZeroChecksum、NoListener、PayloadTooLarge、BroadcastDest、InvalidSource）
- **`UdpStats`**：统计计数器（rx/tx_packets/bytes、rx_errors、checksum_errors、no_listener）

#### 核心函数

| 函数 | 说明 |
|------|------|
| `parse_udp()` | 解析 UDP 报文 + 校验和验证 |
| `parse_udp_header()` | 仅解析头部（不验证校验和） |
| `build_udp_datagram()` | 构建完整 UDP 数据报 |
| `compute_udp_checksum()` | 计算 UDP 校验和（含 IPv4 伪头部） |
| `verify_udp_checksum()` | 验证 UDP 校验和 |

#### 安全特性

- **强制校验和**：拒绝 checksum == 0 的报文（比 RFC 768 更严格，RFC 允许 IPv4 UDP 省略校验和）
- **长度验证**：header.length 必须与实际数据长度一致
- **载荷上限**：`UDP_MAX_PAYLOAD = 65507`（65535 - 20 IP - 8 UDP），防止 IPv4 total_len 溢出
- **广播/组播源地址拒绝**

#### 常量

```rust
pub const UDP_HEADER_LEN: usize = 8;
pub const UDP_MAX_PAYLOAD: usize = 65507;
pub const UDP_DEFAULT_MTU: usize = 1472;        // 1500 - 20 (IP) - 8 (UDP)
pub const EPHEMERAL_PORT_START: u16 = 49152;
pub const EPHEMERAL_PORT_END: u16 = 65535;
```

---

### 3.9 `socket.rs`（7,145 行）

#### 功能说明

模块中最大的文件，实现了完整的能力安全 Socket API。这是用户空间与网络栈交互的唯一入口，集成了 LSM 安全钩子、TCP 被动打开（listen/accept）、TCP 主动连接（connect）、UDP 收发、连接生命周期管理、TCP 定时器以及 Keepalive 探测发送。

#### 核心数据结构

- **`SocketTable`**：全局 Socket 管理表
  - `sockets: RwLock<BTreeMap<u64, Arc<SocketState>>>`：所有 Socket 实例
  - `udp_bindings: Mutex<BTreeMap<(NamespaceId, u16), Weak<SocketState>>>`：UDP 端口绑定
  - `tcp_bindings: Mutex<BTreeMap<(NamespaceId, u16), Weak<SocketState>>>`：TCP 端口绑定
  - `tcp_conns: Mutex<BTreeMap<TcpLookupKey, Weak<SocketState>>>`：活跃 TCP 连接
  - `per_ns_counts: Mutex<BTreeMap<NamespaceId, u64>>`：每命名空间 Socket 计数
  - `MAX_SOCKETS_PER_NS: u64 = 8192`：每命名空间 Socket 配额上限

- **`SocketState`**：单个 Socket 的完整状态
  - 包含 socket_id、domain、sock_type、protocol、label、net_ns_id
  - TCP 状态：`tcp_state: Mutex<Option<TcpSocketState>>`
  - TCP 监听状态：`listen_state: Mutex<Option<TcpListenState>>`
  - UDP 接收队列：`rx_queue: Mutex<VecDeque<PendingDatagram>>`（上限 `MAX_RX_QUEUE = 64`）
  - **Drop 实现（R132-4）**：销毁时释放未读 UDP 数据报的全局字节计数

- **`TcpListenState`**：TCP 被动打开状态
  - `syn_queue: BTreeMap<TcpLookupKey, PendingSyn>`：半开连接队列
  - `accept_queue: VecDeque<Arc<SocketState>>`：已完成连接队列
  - `syn_backlog` / `accept_backlog`：队列容量上限

- **`WaitQueue`**：调度器集成的等待队列
  - `wakeup_count: AtomicU64`：唤醒计数器
  - `closed: AtomicBool`：关闭标志
  - 通过 `SocketWaitHooks` trait 与调度器交互

- **`SocketLabel`**：安全标签（creator ProcessCtx + secmark）

#### 核心函数

| 函数 | 说明 |
|------|------|
| `create_tcp_socket()` | 创建 TCP Socket，LSM 检查 + 命名空间配额 |
| `create_udp_socket()` | 创建 UDP Socket，LSM 检查 + 命名空间配额 |
| `bind_udp()` / `bind_tcp()` | 绑定端口，LSM `hook_net_bind` 检查 |
| `listen()` | TCP 被动打开，创建 `TcpListenState` |
| `poll_accept_ready()` | 检查 accept_queue 是否有就绪连接 |
| `connect()` | TCP 主动连接（SYN 发送） |
| `send_to_udp()` / `tcp_send()` | 数据发送，LSM 检查 |
| `recv_from_udp()` / `tcp_recv()` | 数据接收，LSM 检查 |
| `tcp_shutdown()` | TCP 连接关闭 |
| `deliver_udp()` | UDP 收包投递（从 stack.rs 调用） |
| `process_tcp_segment()` | TCP 收包处理主入口（从 stack.rs 调用） |
| `run_tcp_timers()` | TCP 定时器：重传、FIN 重传、FIN_WAIT_2 超时、**Keepalive 探测（R148-I3）** |
| `sweep_time_wait()` | TIME_WAIT 状态清理 |
| `abort_tcp_connect()` | 中止 TCP 连接（发送 RST） |
| `close()` | 关闭 Socket 并释放所有资源 |

#### 全局 UDP 内存上限（R132-4）

```rust
const MAX_GLOBAL_UDP_QUEUED_BYTES: usize = 16 * 1024 * 1024;  // 16 MB
static GLOBAL_UDP_QUEUED_BYTES: AtomicUsize = AtomicUsize::new(0);
```

使用原子 CAS 操作在 `enqueue_rx()` 中递增，在 `pop_rx()` 和 `SocketState::Drop` 中递减。防止多个 UDP Socket 累积大数据报导致内核堆耗尽。

#### 安全特性

- **LSM 钩子全覆盖**：socket 创建、bind、connect、listen、send、recv、shutdown 均经过 LSM 策略检查
- **能力安全**：Socket 操作需要 `ProcessCtx`（来自 `cap` crate），支持命名空间隔离
- **RFC 5961 严格 RST 验证（R151-7）**：同步状态下仅接受 SEG.SEQ == RCV.NXT 的 RST；窗口内非精确 RST 触发 Challenge ACK；窗口外 RST 静默丢弃。防止盲 RST 注入攻击
- **Challenge ACK 速率限制（R54-2/R121-6/R155-13）**：100/秒 CAS 令牌桶，窗口重置使用 compare_exchange 防止 SMP 双重 refill
- **RST 速率限制（R63-4/R121-6/R154-15）**：100/秒令牌桶，同样使用 CAS 修复 SMP 竞态
- **全局 TCP 连接限制**：
  - `GLOBAL_MAX_HALF_OPEN = 1024`：半开连接上限，CAS 原子递增
  - `TCP_MAX_ACTIVE_CONNECTIONS = 4096`：活跃连接上限
- **每命名空间 Socket 配额（R76-3/R77-4）**：
  - `MAX_SOCKETS_PER_NS = 8192`
  - TCP 监听器创建子 Socket 时也检查配额（R77-4 修复绕过路径）
  - 失败路径正确回滚配额计数
- **Conntrack 种子（R155-9）**：TCP 被动打开（SYN 队列路径和 SYN Cookie 路径）均在发送 SYN-ACK 后种子 conntrack 条目，确保后续 ACK 包被正确分类为 Established 而非 Invalid

---

### 3.10 `firewall.rs`（809 行）

#### 功能说明

基于优先级的有状态防火墙，集成 conntrack 状态进行包过滤决策。**R121-1 修复后为 per-namespace 隔离**。

#### 核心数据结构

- **`FirewallTable`**：防火墙规则表
  - `rules: RwLock<Vec<FirewallRule>>`：规则列表（按优先级降序排列）
  - `default_action: FirewallAction`：默认动作（**DROP**，R94-12 修复后的 fail-closed 设计）
  - `stats: FirewallStats`：统计计数器

- **`FirewallRule`**：单条防火墙规则
  - 匹配条件：src_ip（CIDR）、dst_ip（CIDR）、src_port（范围）、dst_port（范围）、proto、ct_state
  - 动作：Accept / Drop / Reject { icmp_code }
  - 优先级：priority（降序评估，相同优先级按 rule ID 升序）
  - 日志：per-rule 可选日志

- **`FirewallPacket`**：待评估的包元数据（含 `net_ns_id` 字段用于命名空间路由）
- **`FirewallVerdict`**：评估结果（action + rule_id + log）
- **`IpCidrMatch`**：CIDR 地址匹配器（addr + prefix_len）
- **`PortRange`**：端口范围匹配器（start..=end）
- **`CtStateMask`**：conntrack 状态位掩码（NEW=0x01, ESTABLISHED=0x02, RELATED=0x04, INVALID=0x08）
- **`FirewallRuleBuilder`**：规则构造器 builder 模式

#### Per-Namespace 隔离（R121-1）

```rust
static FIREWALL_TABLES: RwLock<BTreeMap<u64, Arc<FirewallTable>>> = RwLock::new(BTreeMap::new());
```

- `firewall_table_for_ns(ns_id)`：获取（懒初始化）命名空间专属防火墙表，每个命名空间拥有独立规则集
- `firewall_table()`：获取根命名空间（namespace 0）防火墙表
- `firewall_remove_ns(ns_id)`：命名空间销毁时清理防火墙表，防止 map 无限增长

#### 默认规则集（R94-12 修复）

```rust
// Rule 1: DROP INVALID 包（priority 1000）
// Rule 2: ACCEPT ESTABLISHED|RELATED 包（priority 900）
// 默认策略：DROP（fail-closed 安全设计）
```

**关键安全设计**：R94-12 修复将默认策略从 ACCEPT 改为 DROP。新连接必须有显式 ACCEPT 规则才能通过。

#### 安全特性

- **规则原子替换**：通过 `RwLock` 保证规则表替换的原子性
- **批量统计更新（R64-6）**：每包一次原子操作而非每规则一次，减少原子操作开销
- **日志速率限制（R64-4/R65-1/R154-I7）**：100/秒令牌桶防止控制台洪泛；CAS 修复令牌下溢竞态；窗口重置使用 compare_exchange 防止 SMP 双重 refill

---

### 3.11 `conntrack.rs`（1,256 行）

#### 功能说明

独立于 Socket 层的连接跟踪模块，为防火墙提供有状态过滤能力。跟踪 TCP、UDP 和 ICMP 流的状态转换。**R107-2 修复后全面支持 per-namespace 隔离**。

#### 核心数据结构

- **`ConntrackTable`**：连接跟踪表
  - `entries: RwLock<BTreeMap<FlowKey, Mutex<ConntrackEntry>>>`：条目存储
  - `ns_entry_counts: Mutex<BTreeMap<u64, usize>>`：per-namespace 条目计数（R140-9）
  - `lru_index: Mutex<BinaryHeap<Reverse<LruIndexEntry>>>`：LRU 淘汰堆（R65-3）
  - `lru_clock: AtomicU64`：单调递增 generation 计数器
  - 最大全局条目数：`CT_MAX_ENTRIES = 65536`
  - **每命名空间条目上限（R140-9）**：`CT_MAX_ENTRIES_PER_NS = 16384`

- **`FlowKey`**：归一化流键（**net_ns_id** + proto + ip_lo/hi + port_lo/hi），双向匹配
- **`ConntrackEntry`**：连接条目
  - state（CtProtoState）、last_seen_ms、bytes/packets_orig/reply、seen_reply
  - `initiator_dir`（R63-1）：真实发起方向（修复归一化导致的方向混淆）
  - `lru_gen`（R65-3）：LRU 堆验证 generation

- **`CtDecision`**：决策枚举（New、Established、Related、Invalid）
- **`TcpCtState`**：TCP 连接跟踪状态（None → SynSent → SynRecv → Established → FinWait → **FinWait2**（R146-NET-2） → CloseWait → LastAck → TimeWait → Close）
- **`UdpCtState`**：UDP 状态（Unreplied → Replied）
- **`IcmpCtState`**：ICMP 状态（EchoRequest → EchoReply）
- **`L4Meta`**：L4 元数据（tcp_flags、payload_len）

#### 核心函数

| 函数 | 说明 |
|------|------|
| `update_on_packet()` | 包到达时更新连接状态（主入口），含过期条目刷新（R151-10） |
| `lookup()` | 按 FlowKey 查找条目 |
| `sweep()` | 定时清理过期条目（预算 `CT_SWEEP_BUDGET = 256`） |
| `tcp_transition()` | TCP 状态机转换，含同时打开（R150-I1）、RST 不转换（R145-4）、FIN_WAIT→FIN_WAIT2（R146-NET-2） |
| `udp_transition()` | UDP 状态转换 |
| `icmp_transition()` | ICMP 连接跟踪 |
| `ct_process_tcp/udp/icmp()` | 便捷包装函数 |

#### 超时配置

| 状态 | 超时 |
|------|------|
| TCP SYN_SENT / SYN_RECV | 60 秒 |
| **TCP ESTABLISHED** | **7,200 秒（2 小时，R147-4 从 5 分钟上调）** |
| **TCP FIN_WAIT / FIN_WAIT2** | **60 秒（R155-10 从 120s 下调以对齐 socket 层）** |
| TCP CLOSE_WAIT | 60 秒 |
| TCP LAST_ACK | 30 秒 |
| TCP TIME_WAIT | 120 秒（2*MSL） |
| TCP CLOSE | 10 秒 |
| UDP Unreplied | 30 秒 |
| UDP Replied | 180 秒（3 分钟） |
| ICMP | 30 秒 |

#### 安全特性

- **LRU 淘汰（R63-3/R65-3）**：表满时淘汰最久未使用的条目，O(log n) 堆操作
- **堆压缩（R96-3）**：LRU 堆在高 PPS 下会膨胀（每包一次 push），当堆大小超过 `entries.len() * 4 + 1024` 时从当前条目重建堆
- **方向修正（R63-1）**：FlowKey 归一化使用字典序，但状态机需要真实发起方向，通过 `initiator_dir` 字段修正
- **状态转换验证（R95-2）**：无效状态转换返回 `CtDecision::Invalid`；SynSent 状态返回 New 而非 Established；teardown 状态（Close/TimeWait）中的 SYN 和非 SYN 包均返回 Invalid，防止 RST→Close→SYN 绕过防火墙
- **RST 不转换状态（R145-4）**：conntrack 层缺乏序列号验证 RST 合法性，RST 保持当前状态不变，让 socket 层（RFC 5961）执行正确验证
- **同时打开支持（R150-I1）**：SynSent 状态收到对端裸 SYN 时转换到 SynRecv
- **过期条目刷新（R151-10）**：查找时遇到过期条目视为不存在，防止迟到包复活过期流
- **Per-namespace 配额（R140-9）**：每命名空间最多 16384 条目，防止单一命名空间垄断全局表
- **Per-namespace FlowKey 隔离（R107-2）**：FlowKey 包含 net_ns_id，不同命名空间的相同 5 元组流不会冲突

#### 已知限制：ICMP RELATED 未解析嵌入原始包

`ct_process_icmp()` 函数中对 ICMP 错误消息（Type 3/11/12）的处理存在不足：当前实现将 ICMP 错误消息作为 NEW 流处理，而非解析其嵌入的原始 IP 头部来关联到已有连接。这意味着在默认 DROP 防火墙策略下，合法的 ICMP 错误消息（如 Path MTU Discovery 的 "Fragmentation Needed"）可能被丢弃。

---

### 3.12 `fragment.rs`（1,104 行）

#### 功能说明

IPv4 分片重组模块，实现了 RFC 815 空洞算法和 RFC 5722 重叠检测，配备全面的 DoS 防护机制。**R140-4 修复后支持 per-namespace 分片隔离**。

#### 核心数据结构

- **`FragmentCache`**：全局分片重组缓存
  - `queues: Mutex<BTreeMap<FragmentKey, FragmentQueue>>`：活跃重组队列
  - `per_src_counts: Mutex<BTreeMap<(u64, u32), usize>>`：每（命名空间, 源 IP）队列计数（R141-2 修复为命名空间隔离）
  - `stats: FragmentStats`：统计计数器

- **`FragmentKey`**：重组键（**net_ns_id** + src + dst + protocol + identification），派生 Ord 用于 BTreeMap（R140-4）
- **`FragmentQueue`**：单个 IP 数据报的重组队列
  - `holes: Vec<FragmentHole>`：RFC 815 空洞列表
  - `frags: BTreeMap<u16, Vec<u8>>`：按偏移存储的分片数据
  - `rate_limiter: RateLimiter`：每源令牌桶速率限制
  - 状态标志：have_first、have_last、l4_header_ok、total_len
- **`FragmentDropReason`**：丢弃原因枚举（RateLimited、TooLarge、QueueFragLimit、QueueByteLimit、FirstTooSmall、Overlap、GlobalQueueLimit、GlobalFragLimit、GlobalByteLimit、PerSourceLimit、Timeout、ZeroLength、Duplicate）

#### 安全限制常量

```rust
pub const FRAG_TIMEOUT_MS: u64 = 30_000;           // 重组超时 30 秒（R101-12 从 45s 降低）
pub const MAX_PACKET_SIZE: usize = 65_535;          // IPv4 最大包长
pub const MAX_FRAGS_PER_QUEUE: usize = 64;          // 每队列最大分片数
pub const MAX_BYTES_PER_QUEUE: usize = 512 * 1024;  // 每队列最大字节数 512KB（R62-2）
pub const MAX_QUEUES_PER_SRC: usize = 256;           // 每源 IP 最大队列数
pub const GLOBAL_MAX_QUEUES: usize = 4096;           // 全局最大队列数
pub const GLOBAL_MAX_FRAGS: usize = 32_768;          // 全局最大分片数
pub const GLOBAL_MAX_FRAG_BYTES: usize = 64 * 1024 * 1024;  // 全局最大字节数 64MB（R62-2）
pub const MIN_L4_HEADER_BYTES: usize = 8;            // 首片最小 L4 头部
pub const RATE_LIMIT_TOKENS: u32 = 128;              // 每源速率限制令牌
pub const RATE_LIMIT_WINDOW_MS: u64 = 1000;          // 速率限制窗口
```

#### DoS 防护机制（多层防御）

1. **全局资源限制**：队列数（4096）、分片数（32768）、字节数（64MB）
2. **每（命名空间, 源 IP）限制（R141-2）**：最多 256 个并发重组队列，命名空间隔离防止跨命名空间 DoS
3. **每队列限制**：最多 64 个分片、512KB 字节
4. **速率限制**：每源 128 分片/秒令牌桶
5. **首片 L4 头部要求**：首片必须包含至少 8 字节 L4 头部（覆盖 UDP 头部或 TCP 源/目的端口）
6. **RFC 5722 重叠检测**：重叠分片触发整个重组队列丢弃
7. **固定超时**：30 秒超时不因新分片到达而刷新（防止滴水攻击保持队列存活）
8. **原子资源预留（R66-11）**：使用 CAS 操作预留全局分片槽位和字节配额，失败时回滚
9. **LRU 淘汰（R101-12）**：全局队列满时淘汰最老队列而非直接拒绝
10. **checked 算术（R99-3/R102-L1）**：防止整数溢出绕过边界检查
11. **Per-namespace 隔离（R140-4）**：FragmentKey 包含 net_ns_id，不同命名空间的分片互不干扰

---

### 3.13 `buffer.rs`（525 行）

#### 功能说明

DMA 兼容的网络包缓冲区实现，提供 headroom/tailroom 管理和缓冲池。

#### 核心数据结构

- **`NetBuf`**：网络包缓冲区
  - `dma: DmaBuffer`：IOMMU 映射的 DMA 缓冲区（R98-2）
  - 内存布局：`[headroom | data | tailroom]`
  - 跟踪 data_offset 和 data_len 实现零拷贝头部操作

- **`BufPool`**：预分配缓冲池
  - `buffers: Mutex<Vec<NetBuf>>`：可用缓冲区栈
  - 默认布局：MTU=1500, headroom=64, tailroom=64（总计 1628 字节，适配单页）

#### 核心函数

| 函数 | 说明 |
|------|------|
| `NetBuf::new()` | 从 DMA 缓冲区创建 NetBuf，checked 算术防溢出（R99-3） |
| `push_head()` | 向前扩展数据区（消耗 headroom），用于添加协议头 |
| `push_tail()` | 向后扩展数据区（消耗 tailroom），用于接收数据 |
| `pull_head()` | 从前端收缩数据区（回收 headroom），用于剥离协议头 |
| `pull_tail()` | 从后端收缩数据区（回收 tailroom） |
| `reset()` | 重置缓冲区并清零全部内存（R43-5 防信息泄漏） |
| `BufPool::alloc()` / `free()` | 缓冲区分配/归还 |

#### 安全特性

- **DMA 缓冲区（R98-2）**：使用 `alloc_dma_buffer()` 分配 IOMMU 映射内存
- **全缓冲区清零（R43-5 v3）**：`reset()` 清零整个缓冲区（含 tailroom），防止 DMA 设备读取残留数据
- **Release 模式边界检查（R43-4）**：`buffer_slice()` / `buffer_slice_mut()` 使用 `assert!` 配合 `checked_add()` 而非 `debug_assert!`
- **checked 算术（R99-3）**：`new()` 中使用 `checked_add()` 防止 headroom+mtu+tailroom 溢出绕过 DMA_PAGE_SIZE 检查
- **Drop 安全**：`NetBuf` 的 `Drop` 委托给 `DmaBuffer`，后者处理 IOMMU 解映射、内存擦除和物理页释放

---

### 3.14 `device.rs`（366 行）

#### 功能说明

网络设备抽象层，定义 `NetDevice` trait 作为所有网络驱动的统一接口。

#### 核心数据结构

- **`NetDevice` trait**：核心网络设备接口
  - 身份与配置：`name()`、`mac_address()`、`set_mac_address()`、`capabilities()`
  - 链路状态：`link_status()`
  - 操作模式：`operating_mode()`、`set_operating_mode()`、`enable/disable_interrupts()`
  - 发送路径：`transmit()`（消费 NetBuf）、`reclaim_tx()`、`tx_queue_space()`
  - 接收路径：`receive()`（返回 NetBuf）、`replenish_rx()`、`rx_queue_depth()`
  - 轮询/中断：`poll()`、`handle_interrupt()`
  - 统计：tx/rx_packets/bytes/errors、`rx_dropped()`（R66-8）

- **`DeviceCaps`**：设备能力（tx/rx_checksum、tso、ufo、gso、rss、vlan_offload、promiscuous、multicast_filter、mtu）
- **`LinkStatus`**：链路状态（up、speed_mbps、full_duplex），含 `UP_UNKNOWN` 常量
- **`OperatingMode`**：操作模式（Polling、Interrupt、Napi）
- **`MacAddress`**：类型别名 `[u8; 6]`
- **错误类型**：`NetError`、`TxError`、`RxError`

#### 设计原则

1. **非阻塞 I/O**：transmit/receive 均为非阻塞操作
2. **缓冲区所有权**：transmit 消费 NetBuf，receive 返回 NetBuf
3. **双模式操作**：支持中断驱动和轮询模式
4. **能力协商**：设备通过 `DeviceCaps` 通告硬件卸载能力

---

### 3.15 `virtio_net.rs`（911 行）

#### 功能说明

VirtIO-net 网络设备驱动，支持 PCI Modern Transport（也支持 MMIO Transport）。实现了 `NetDevice` trait，包含大量针对恶意/故障设备的防护措施。

#### 核心数据结构

- **`VirtioNetDevice`**：VirtIO 网络设备
  - `transport: VirtioTransport`：PCI 或 MMIO 传输层
  - `rx_queue` / `tx_queue`：VirtIO 虚拟队列
  - `rx_queue_dma` / `tx_queue_dma`：队列 DMA 缓冲区所有权（R95-7）
  - `rx_inflight: Vec<Option<RxInflight>>`：RX 在途缓冲区跟踪，含 driver 端 capacity 记录
  - `tx_inflight: Vec<Option<NetBuf>>`：TX 在途缓冲区
  - `tx_chain_next` / `rx_chain_next: Vec<Option<u16>>`：驱动端描述符链跟踪（R44-1）
  - `rx_ready: Vec<NetBuf>`：已接收包就绪队列（上限 `MAX_RX_READY_QUEUE = 256`，R66-8）
  - `rx_recycle: Vec<NetBuf>`：待回收 RX 缓冲区（R96-4）

- **`RxInflight`**：RX 在途缓冲区元数据（R65-24）
  - `buf: NetBuf`：实际缓冲区
  - `capacity: usize`：驱动端跟踪的投递容量，用于 DMA 溢出钳位

#### 安全特性（恶意设备防护）

- **驱动端链跟踪（R44-1）**：`tx_chain_next` / `rx_chain_next` 由驱动维护，不信任设备返回的 next 指针，防止恶意设备操纵描述符链导致 UAF 或双重释放
- **DMA 缓冲区所有权（R95-7）**：`setup_queue()` 将 DMA 缓冲区存储在 `rx_queue_dma`/`tx_queue_dma` 字段而非 `mem::forget()` 泄漏，确保设备销毁时正确释放
- **DMA 溢出钳位（R65-24）**：`RxInflight.capacity` 跟踪驱动端投递的缓冲区容量，设备报告的已用字节数被钳位到此值，防止恶意设备声称处理了超出实际大小的数据
- **缓冲区回收（R96-4）**：`recycle_rx_buffer()` 将处理完的 RX 缓冲区放入 `rx_recycle` 待重新投递，防止 BufPool 在对抗条件下耗尽
- **有界 RX 就绪队列（R66-8）**：`MAX_RX_READY_QUEUE = 256`，溢出时缓冲区进入 `rx_recycle` 而非丢弃（R96-4）
- **正确的 Drop 实现（R95-7）**：禁用队列 → 重置设备 → SeqCst 内存屏障 → 释放 DMA 缓冲区

#### 特性协商

```rust
// 必需特性
VIRTIO_F_VERSION_1      // VirtIO 1.0 现代设备
// 可选特性
VIRTIO_NET_F_MAC         // 设备提供 MAC 地址
VIRTIO_NET_F_STATUS      // 设备提供链路状态
```

---

### 3.16 `pci.rs`（370 行）

#### 功能说明

PCI 总线扫描模块，负责探测 VirtIO-net 设备并读取 PCI capability 信息。

#### 核心数据结构

- **`PciSlot`**：PCI 设备槽位（bus、device、function）
- **`VirtioNetPciDevice`**：探测到的 VirtIO-net PCI 设备信息
  - PCI 槽位、VirtIO PCI 地址结构（`VirtioPciAddrs`）

#### 核心函数

| 函数 | 说明 |
|------|------|
| `probe_virtio_net()` | 扫描 PCI 总线探测 VirtIO-net 设备 |
| `read_virtio_caps()` | 遍历 PCI capability 链表读取 VirtIO capability |
| `read_bar()` | 读取 BAR 寄存器（支持 32 位和 64 位） |
| `disable_bus_master()` | 禁用总线主控（安全回退） |

#### PCI 扫描范围

- 总线：0-7
- 设备：0-31
- 支持多功能设备检测

#### 设备识别

```rust
// VirtIO 过渡设备
vendor_id == 0x1AF4 && device_id == 0x1000 && subsystem_id == 1

// VirtIO 现代设备
vendor_id == 0x1AF4 && device_id == 0x1041
```

#### 安全特性

- **IOMMU 先绑定后启用（fail-closed）**：在启用 Bus Master 之前先将设备绑定到 IOMMU，绑定失败则跳过设备（NotAvailable 除外：IOMMU 不存在时记录警告继续，R94-14 修复）
- **现代设备验证（R82-1）**：缺少 Modern Transport capability 的设备被禁用 Bus Master 并跳过

---

### 3.17 `stack.rs`（1,186 行）

#### 功能说明

协议栈主处理循环，是所有收包处理的入口点，也是发包的出口。集成了所有协议层和安全组件。

#### 核心数据结构

- **`NetStats`**：全局网络统计（原子计数器）
  - rx_packets、rx_errors、ipv4_rx、icmp_rx/echo_rx/echo_tx、rate_limited、unsupported_proto、fragments_rx/reassembled/dropped、arp_stats、udp_stats
- **`ProcessResult`**：帧处理结果枚举（Handled、Reply(Vec<u8>)、Dropped(DropReason)）
- **`DropReason`**：丢弃原因枚举，覆盖所有协议层错误，含 Firewall 变体（rule_id + rejected）
- **`NetConfig`**：网络配置（固定 QEMU SLIRP 地址）
- **`NetConfigSnapshot`**：公共网络配置快照
- **`NetState`**：全局网络状态（config + ARP cache）

#### 网络配置（简化路由）

```rust
const DEFAULT_OUR_IP: Ipv4Addr = Ipv4Addr([10, 0, 2, 15]);
const DEFAULT_GATEWAY_IP: Ipv4Addr = Ipv4Addr([10, 0, 2, 2]);
const DEFAULT_GATEWAY_MAC: EthAddr = EthAddr([0x52, 0x54, 0x00, 0x12, 0x34, 0x56]);
```

**注意**：当前路由为硬编码的 QEMU SLIRP 网络配置，所有非本地子网流量发往固定网关 MAC。这是开发阶段的简化设计，生产环境需要实现动态路由表和 ARP 解析。

#### 核心函数

| 函数 | 说明 |
|------|------|
| `process_frame()` | 帧处理入口：Ethernet 解析 → IPv4/ARP 分发，含命名空间 ID 传递（R90-2） |
| `process_ipv4()` | IPv4 处理：分片重组（含命名空间隔离 R140-4）→ ICMP/UDP/TCP 分发 |
| `process_icmp()` | ICMP 处理：conntrack → per-namespace 防火墙 → Echo Reply 生成（含 Smurf 防护） |
| `process_udp()` | UDP 处理：conntrack → per-namespace 防火墙 → Socket 投递 |
| `process_tcp()` | TCP 处理：conntrack → per-namespace 防火墙 → Socket 层委托 |
| `build_frame_and_transmit()` | 构建以太网帧并发送，使用 DMA 缓冲区（R98-2） |
| `transmit_tcp_segment()` | TCP 段发送（IP 头 + TCP 头 + 以太网帧） |
| `transmit_udp_datagram()` | UDP 数据报发送 |
| `apply_firewall_verdict()` | 应用防火墙裁决（REJECT 时对 TCP 发送 RFC 793 RST，对其他协议发送 ICMP 不可达，R64-5） |
| `handle_timer_tick()` | 定时器处理：分片清理 |
| `network_config()` | 获取网络配置快照（懒初始化 MAC 地址） |

#### 处理流程

```
收包路径：
  process_frame(frame, our_mac, our_ip, arp_cache, stats, net_ns_id, now_ms)
    +--> parse_ethernet()
    +--> EtherType::ARP --> process_arp()
    +--> EtherType::IPv4 --> process_ipv4()
        +--> parse_ipv4() + 校验和验证
        +--> 分片重组（fragment::process_fragment(net_ns_id, ...)）
        +--> ICMP --> process_icmp()
        |   +--> conntrack (per-namespace)
        |   +--> firewall (per-namespace, R121-1)
        |   +--> Echo Request --> build_echo_reply()（Smurf 防护）
        +--> UDP --> process_udp()
        |   +--> conntrack (per-namespace, R107-2)
        |   +--> firewall (per-namespace, R121-1)
        |   +--> apply_firewall_verdict()
        |   +--> socket_table().deliver_udp(net_ns_id, ...)
        +--> TCP --> process_tcp()
            +--> parse_tcp_header() + verify_tcp_checksum()
            +--> conntrack (per-namespace, R107-2)
            +--> firewall (per-namespace, R121-1)
            +--> apply_firewall_verdict()
            +--> socket_table().process_tcp_segment(net_ns_id, ...)
```

#### 安全特性

- **Smurf 攻击防护**：不对广播/组播目的地址回复 ICMP Echo Reply（双重检查：IP 层 + MAC 层）
- **防火墙裁决执行（R64-5）**：TCP REJECT 发送 RFC 793 RST（根据 ACK 标志正确计算 seq/ack）；非 TCP 协议发送 ICMP Destination Unreachable；广播/组播目的地址不发送 ICMP 错误
- **防火墙 REJECT 速率限制（R64-1）**：复用 ICMP_RATE_LIMITER 防止反射/放大攻击
- **命名空间隔离（R90-2）**：所有收包路径传递 `net_ns_id`，Socket 层按命名空间投递
- **DMA 发送（R98-2）**：TX 路径使用 `alloc_dma_buffer()` 分配 IOMMU 映射的 NetBuf

---

## 四、模块总体评估

### 4.1 架构评估

#### 分层设计

`net` 模块采用经典的分层网络协议栈架构：

```
用户空间 syscall
    |
+---------------------------------------------+
|  socket.rs (7145 行)                         |  Socket 层
|  能力安全 API / LSM 钩子 / TCP 连接管理       |  （用户接口）
|  TCP Keepalive / 全局 UDP 内存上限            |
+---------------------------------------------+
|  tcp.rs (3068 行) / udp.rs (515 行)          |  传输层
|  TCP 状态机 / 拥塞控制 / SACK / OOO 重组      |
|  UDP 收发                                    |
+---------------------------------------------+
|  firewall.rs (809 行) / conntrack.rs (1256 行)|  安全层
|  per-namespace 有状态防火墙 / 连接跟踪        |
+---------------------------------------------+
|  ipv4.rs (561 行) / icmp.rs (638 行)         |  网络层
|  fragment.rs (1104 行) / arp.rs (999 行)     |
+---------------------------------------------+
|  ethernet.rs (214 行)                        |  链路层
+---------------------------------------------+
|  stack.rs (1186 行)                          |  协议栈集成
|  帧处理入口 / 协议分发 / TX 路径 / 防火墙裁决  |
+---------------------------------------------+
|  virtio_net.rs (911 行) / pci.rs (370 行)    |  设备驱动
|  device.rs (366 行) / buffer.rs (525 行)     |
+---------------------------------------------+
```

**优点**：
- 各层职责清晰，模块间通过明确的函数接口交互
- `stack.rs` 作为集成层统一协调各协议模块，避免了层间直接耦合
- `device.rs` 的 `NetDevice` trait 为未来支持更多网卡驱动（e1000、Intel NIC）提供了扩展点
- Per-namespace 隔离贯穿防火墙、conntrack、分片重组三大子系统

**不足**：
- `socket.rs` 承载了过多职责（7145 行），包含 TCP 被动打开、主动连接、SYN Cookie 处理、TIME_WAIT 管理、TCP 定时器、Keepalive 等，建议拆分为 `socket_tcp.rs`、`socket_udp.rs` 等子模块
- 路由功能硬编码在 `stack.rs` 的 `NetConfig` 中，缺少独立的路由表模块

### 4.2 安全评估

#### 安全默认值（Security Defaults）

`net` 模块在安全设计上表现出色，贯彻了"默认安全"原则：

| 安全机制 | 实现位置 | 说明 |
|---------|---------|------|
| 默认 DROP 防火墙 | `firewall.rs` R94-12 | fail-closed 设计，新连接需显式 ACCEPT 规则 |
| Per-namespace 防火墙 | `firewall.rs` R121-1 | 每个命名空间独立规则集，互不影响 |
| SYN Cookie | `tcp.rs` | RFC 4987 无状态 SYN 洪泛防护，8 级 MSS 表 |
| RFC 5961 严格 RST 验证 | `socket.rs` R151-7 | 精确序列号匹配；窗口内触发 Challenge ACK；防盲 RST 注入 |
| Challenge ACK/RST 速率限制 | `socket.rs` R54-2/R121-6/R154-15/R155-13 | 100/秒 CAS 令牌桶，SMP 安全 |
| IOMMU DMA 隔离 | `pci.rs` / `buffer.rs` | fail-closed：IOMMU 绑定失败则禁用设备 |
| DMA 缓冲区清零 | `buffer.rs` R43-5 | reset() 清零全缓冲区防信息泄漏 |
| 分片 DoS 防护 | `fragment.rs` | 11 层防御：全局/每源/每队列限制 + 速率限制 + 重叠检测 + per-namespace 隔离 |
| ARP 防投毒 | `arp.rs` | 缓存冲突检测 + 静态条目保护 + 源验证 + 双层速率限制 |
| 命名空间隔离 | `socket.rs` / `stack.rs` / `conntrack.rs` / `firewall.rs` / `fragment.rs` | Socket 配额 + 收包路由 + conntrack + 防火墙 + 分片 |
| 恶意设备防护 | `virtio_net.rs` | 驱动端链跟踪 + DMA 溢出钳位 + 有界队列 + 正确 Drop |
| TCP SACK 安全 | `tcp.rs` R115-3 | O(n+m log m) 处理算法防 CPU 放大 |
| OOO 段窗口约束 | `tcp.rs` R113-2 | OOO 总字节数受通告窗口约束防内存耗尽 |
| 全局 UDP 内存上限 | `socket.rs` R132-4 | 16 MB 全局 CAS 计数器防堆耗尽 |
| Conntrack per-NS 配额 | `conntrack.rs` R140-9 | 每命名空间最多 16384 条目 |
| Conntrack RST 不转换 | `conntrack.rs` R145-4 | 保留状态让 socket 层验证 RST 合法性 |
| Conntrack 过期条目刷新 | `conntrack.rs` R151-10 | 查找时过期条目视为不存在 |
| TCP Keepalive | `tcp.rs`/`socket.rs` R148-I3 | RFC 1122 合规，可配置参数，定时器驱动 |
| TCP 被动打开 conntrack 种子 | `socket.rs` R155-9 | SYN/SYN-ACK 后种子 conntrack 确保 ACK 分类正确 |

### 4.3 性能评估

#### 优化措施

- **缓冲池预分配**：`BufPool` 避免热路径上的内存分配
- **零拷贝头部操作**：`NetBuf` 的 headroom/tailroom 设计避免数据拷贝
- **批量统计更新**：防火墙规则评估使用本地计数器，每包一次原子操作（R64-6）
- **LRU 堆压缩**：conntrack LRU 堆在膨胀时从当前条目重建（R96-3）
- **读写锁分离**：conntrack 表使用 `RwLock`，读路径（查找）不阻塞
- **SACK 优化（R115-3）**：process_sack_blocks 从 O(n*m) 优化为 O(n + m log m)
- **Per-namespace 防火墙快路径**：Arc clone 后释放全局 map 锁，评估时无锁竞争

#### 性能瓶颈

- **自旋锁竞争**：`BufPool`、`SocketTable` 的多个 `Mutex` 在高并发下可能成为瓶颈
- **防火墙线性扫描**：规则评估 O(n)，大规则集场景需要优化
- **conntrack 双锁**：`entries`（RwLock）和 `lru_index`（Mutex）的锁获取顺序需要注意
- **分片重组双锁**：`queues` 和 `per_src_counts` 同时持有可能增加延迟

### 4.4 改进路线图

#### 高优先级

1. **实现 conntrack ICMP RELATED 关联**：解析 ICMP 错误消息中嵌入的原始 IPv4 头部，在 conntrack 表中查找对应连接，返回 `CtDecision::Related`。这对 Path MTU Discovery 至关重要

2. **Socket 配额 RAII 化**：将命名空间 Socket 计数的递减操作移入 `SocketState` 的 `Drop` 实现或使用 RAII 守卫，消除配额泄漏风险

3. **socket.rs 拆分**：将 7145 行的 socket.rs 拆分为 TCP/UDP 子模块，提升可维护性

#### 中优先级

4. **动态路由表**：替换硬编码的 QEMU SLIRP 地址，实现基本的路由表（静态路由 + 默认网关），支持多网卡场景

5. **防火墙规则索引**：为大规则集场景实现分层索引（按协议/端口预分组），将评估复杂度从 O(n) 降低

6. **ARP 缓存数据结构优化**：当前使用 `VecDeque` 线性扫描，大规模场景可考虑 `BTreeMap` 或 `HashMap` 提升查找效率

#### 低优先级

7. **IPv6 支持**：当前仅支持 IPv4

8. **VLAN 支持**：`ETHERTYPE_VLAN` 常量在旧版中定义但当前未在 lib.rs re-export 中使用

9. **硬件卸载利用**：`DeviceCaps` 已定义 checksum/TSO/GSO 等卸载能力，但协议栈未利用这些能力进行优化

10. **NAPI 模式实现**：`OperatingMode::Napi` 已定义但未实现中断触发轮询的混合模式
