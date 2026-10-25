# 用户空间 (userspace) 模块分析文档

> 分析范围：`userspace/`
> 分析日期：2026-05-13
> Zero-OS 版本：main 分支

## 1. 模块概述

`userspace/` 是 Zero-OS 的用户态运行时库与应用程序集合。整个模块以 `#![no_std]` 构建，目标平台为 `x86_64-unknown-none`（无操作系统、无标准 C 库），通过自定义链接脚本将用户程序加载到虚拟地址 `0x400000` 处运行于 Ring 3 特权级。

该模块的核心职责包括：

- **系统调用封装层**：提供与 Linux 兼容的 x86_64 syscall 指令封装（遵循 Linux x86_64 syscall 约定，第 4 参数使用 `r10` 而非 `rcx`）
- **极简 C 库替代**：实现内存操作、字符串处理、字符级 I/O、数字转换等基础功能
- **用户态应用程序**：交互式 Shell（15 个内建命令）、Ring 3 验证程序、系统调用测试套件

### 1.1 架构分层

模块采用清晰的四层架构：

```
┌─────────────────────────────────────────────────────┐
│              应用层 (Applications)                    │
│   shell.rs  │  main.rs  │  syscall_test.rs          │
├─────────────────────────────────────────────────────┤
│              C 库抽象层 (libc.rs)                     │
│   内存操作 │ 字符串操作 │ I/O 操作 │ 数字转换        │
├─────────────────────────────────────────────────────┤
│           类型化系统调用封装 (syscall.rs)              │
│   sys_write │ sys_read │ sys_fork │ sys_socket │ ... │
├─────────────────────────────────────────────────────┤
│           系统调用原语 (syscall.rs)                    │
│   syscall0 │ syscall1 │ ... │ syscall6               │
└─────────────────────────────────────────────────────┘
```

### 1.2 构建特性

- `panic = "abort"`：无 unwinding 运行时，panic 直接终止进程
- `opt-level = "z"`：最小体积优化
- `lto = true` + `codegen-units = 1`：全程序链接时优化，生成最紧凑的二进制
- 丢弃 `.eh_frame`/`.dynamic`/`.interp` 等段：纯静态 flat binary，无需动态链接器

## 2. 文件结构

### 2.1 目录布局

```
userspace/
├── Cargo.toml          # 包配置与构建目标定义
├── userspace.ld        # 用户态链接脚本
└── src/
    ├── lib.rs          # crate 根模块（导出 + panic handler）
    ├── syscall.rs      # 系统调用号、原语、类型化封装、数据结构
    ├── libc.rs         # 极简 C 库实现
    ├── main.rs         # hello 程序（Ring 3 验证）
    ├── shell.rs        # 交互式 Shell
    └── syscall_test.rs # 系统调用测试套件
```

### 2.2 行数统计

| 文件 | 行数 | 职责概述 |
|------|------|----------|
| `Cargo.toml` | 36 | 包元数据、lib + 3 bin targets、构建 profile |
| `userspace.ld` | 69 | ELF64 链接脚本，定义内存布局与段权限 |
| `src/lib.rs` | 47 | crate 入口，导出 `syscall`/`libc` 模块，panic handler |
| `src/syscall.rs` | 896 | 系统调用号常量、内联汇编原语、30+ 类型化封装、数据结构 |
| `src/libc.rs` | 527 | 内存/字符串/I/O/数字转换函数 |
| `src/main.rs` | 72 | Ring 3 验证程序（hello world + PID） |
| `src/shell.rs` | 835 | 交互式 Shell，15 个内建命令 |
| `src/syscall_test.rs` | 234 | gettid/set_tid_address/set_robust_list/getrandom 测试 |
| **合计** | **2716** | -- |

## 3. 逐文件分析

### 3.1 Cargo.toml（36 行）

**构建目标定义：**

```toml
[lib]
name = "userspace"
path = "src/lib.rs"
crate-type = ["staticlib", "rlib"]

[[bin]]
name = "hello"       # Ring 3 验证程序
path = "src/main.rs"

[[bin]]
name = "shell"       # 交互式 Shell
path = "src/shell.rs"

[[bin]]
name = "syscall_test" # 系统调用测试
path = "src/syscall_test.rs"
```

- **双 crate-type**：`staticlib` 供 C/链接器直接使用，`rlib` 供 Rust bin targets 依赖
- **构建 profile**：dev 和 release 均设置 `panic = "abort"` + `opt-level = "z"` + `lto = true` + `codegen-units = 1`
- 无外部依赖（`[dependencies]` 为空），完全自包含

### 3.2 userspace.ld（69 行）

用户态 ELF64 链接脚本，定义程序在虚拟地址空间中的布局。

**关键常量：**

```ld
USER_BASE = 0x400000;           /* 用户程序加载基址（4MB） */
USER_STACK_TOP = 0x7FFFFFFFE000; /* 用户栈顶（由内核映射） */
```

**段布局与权限：**

| 段 | 对齐 | 权限 | PT_LOAD FLAGS | 说明 |
|----|------|------|---------------|------|
| `.text` | 4K | R-X | 5 | 代码段，可读可执行 |
| `.rodata` | 4K | R-- | 4 | 只读数据段 |
| `.data` | 4K | RW- | 6 | 可读写数据段 |
| `.bss` | 4K | RW- | 6（与 .data 同 PHDR） | 零初始化数据段 |

**设计分析：**
- **固定地址模型（ET_EXEC）**：`USER_BASE = 0x400000` 是经典 Linux x86_64 默认加载地址，调试友好但**无 PIE/ASLR 支持**
- **W^X 分离**：text 段仅 R-X、data/bss 段仅 RW-
- 丢弃 `.eh_frame` 配合 `panic = "abort"` 意味着**无栈回溯能力**

### 3.3 src/lib.rs（47 行）

Crate 根模块，职责极简：

```rust
#![no_std]

pub mod libc;
pub mod syscall;

#[panic_handler]
fn panic(_info: &core::panic::PanicInfo) -> ! {
    unsafe {
        let msg = b"PANIC in user program\n";
        let _ = syscall::sys_write(2, msg.as_ptr(), msg.len() as u64);
        syscall::sys_exit(1);
    }
}
```

**局限性**：
- panic 信息不包含源文件/行号/panic message（`_info` 参数被忽略）
- `sys_write` 未处理短写

### 3.4 src/syscall.rs（896 行）

本文件是整个用户态模块的核心，包含系统调用号定义、内联汇编原语、类型化封装、数据结构和辅助函数。

#### 3.4.1 系统调用号常量（第 24-117 行）

定义 32 个与 Linux 兼容的系统调用号：

| 分类 | 系统调用号 | 说明 |
|------|-----------|------|
| 文件 I/O | `SYS_READ(0)`, `SYS_WRITE(1)`, `SYS_OPEN(2)`, `SYS_CLOSE(3)`, `SYS_STAT(4)`, `SYS_FSTAT(5)`, `SYS_LSEEK(8)` | 基础文件操作 |
| 内存 | `SYS_MMAP(9)`, `SYS_MUNMAP(11)`, `SYS_BRK(12)` | 内存映射与堆管理 |
| 目录 | `SYS_GETDENTS64(217)`, `SYS_GETCWD(79)`, `SYS_CHDIR(80)` | 目录遍历与导航 |
| 进程 | `SYS_FORK(57)`, `SYS_EXEC(59)`, `SYS_EXIT(60)`, `SYS_WAIT(61)`, `SYS_KILL(62)`, `SYS_GETPID(39)`, `SYS_GETPPID(110)` | 进程生命周期管理 |
| 线程 | `SYS_GETTID(186)`, `SYS_SET_TID_ADDRESS(218)`, `SYS_SET_ROBUST_LIST(273)`, `SYS_EXIT_GROUP(231)` | 线程标识与 musl 兼容 |
| 调度 | `SYS_YIELD(24)` | 主动让出 CPU |
| 网络 | `SYS_SOCKET(41)`, `SYS_CONNECT(42)`, `SYS_SENDTO(44)`, `SYS_RECVFROM(45)`, `SYS_BIND(49)` | BSD socket 接口 |
| 系统 | `SYS_UNAME(63)`, `SYS_GETRANDOM(318)` | 系统信息与随机数 |

**注意**：`SYS_MMAP`/`SYS_MUNMAP`/`SYS_BRK`/`SYS_EXEC` 虽定义了常量，但未提供对应的类型化 wrapper 函数。

#### 3.4.2 Socket 常量（第 122-139 行）

```rust
pub const AF_INET: u32 = 2;
pub const SOCK_STREAM: u32 = 1;
pub const SOCK_DGRAM: u32 = 2;
pub const IPPROTO_TCP: u32 = 6;
pub const IPPROTO_UDP: u32 = 17;
pub const MSG_DONTWAIT: u32 = 0x40;
```

仅覆盖 IPv4/TCP/UDP 基础常量。

#### 3.4.3 系统调用原语（第 144-277 行）

提供 `syscall0` 到 `syscall6` 共 7 个内联汇编函数：

```rust
#[inline(always)]
pub unsafe fn syscall3(num: u64, arg0: u64, arg1: u64, arg2: u64) -> u64 {
    let ret: u64;
    core::arch::asm!(
        "syscall",
        in("rax") num,
        in("rdi") arg0,
        in("rsi") arg1,
        in("rdx") arg2,
        lateout("rax") ret,
        lateout("rcx") _,
        lateout("r11") _,
        options(nostack, preserves_flags),
    );
    ret
}
```

**ABI 合规性**：寄存器约定正确（RAX/RDI/RSI/RDX/R10/R8/R9），`rcx`/`r11` clobber 声明正确。注意 `preserves_flags` 在语义上存在问题（`syscall` 指令会覆写 RFLAGS）。

#### 3.4.4 错误处理（第 283-300 行）

```rust
const MAX_ERRNO: u64 = 4095;

pub fn is_error(result: u64) -> bool {
    result > (u64::MAX - MAX_ERRNO)
}

pub fn errno(result: u64) -> i32 {
    if is_error(result) { -(result as i64) as i32 } else { 0 }
}
```

#### 3.4.5 类型化系统调用封装（第 306-710 行）

提供 30+ 个类型安全的 wrapper 函数，按功能分组：

- **文件操作（7 个）**：sys_write/read/open/close/stat/fstat/lseek
- **目录操作（3 个）**：sys_getdents64/getcwd/chdir
- **进程管理（7 个）**：sys_getpid/getppid/fork/wait/exit/exit_group/kill
- **线程相关（3 个）**：sys_gettid/set_tid_address/set_robust_list
- **调度（1 个）**：sys_yield
- **网络（5 个）**：sys_socket/bind/connect/sendto/recvfrom
- **系统信息（2 个）**：sys_uname/getrandom

所有 wrapper 均标记 `#[inline(always)]` + `unsafe`，零开销抽象。

#### 3.4.6 数据结构（第 716-802 行）

**`Stat`**：`#[repr(C)]` 文件状态结构，与内核 `VfsStat` 布局匹配。

**`Dirent64`**：变长目录项，`d_reclen` 指示完整记录长度。

**`SockAddrIn`**：IPv4 地址结构（16 字节），`port`/`addr` 为网络字节序，`SockAddrIn::new()` 自动调用 `to_be()` 转换。

**`UtsName`**：5 个 65 字节字段（sysname/nodename/release/version/machine），与内核侧 `UtsName` 匹配。

#### 3.4.7 便利函数与网络辅助（第 808-896 行）

- `print(s: &str)` / `eprint(s: &str)`：将 Rust 字符串切片写入 stdout/stderr
- `parse_ipv4(input: &[u8]) -> Option<[u8; 4]>`：解析点分十进制 IPv4 地址，校验每个 octet 范围 `[0, 255]`，要求恰好 4 段

### 3.5 src/libc.rs（527 行）

极简 C 库实现，为用户态程序提供基础运行时支持。

#### 3.5.1 内存操作（第 28-99 行）

| 函数 | 说明 |
|------|------|
| `memcpy` | 逐字节前向拷贝，不处理重叠 |
| `memset` | 逐字节填充 |
| `memcmp` | 逐字节比较，返回差值 |
| `memmove` | 根据地址关系选择前向/后向拷贝，正确处理重叠 |

#### 3.5.2 字符串操作（第 105-228 行）

| 函数 | 说明 |
|------|------|
| `strlen` | 计算 null 终止字符串长度 |
| `strcpy` | 拷贝含 null 终止符 |
| `strncpy` | 限长拷贝，不足部分填零（POSIX 语义） |
| `strcmp` | 字典序比较 |
| `strncmp` | 限长字典序比较 |
| `strchr` | 查找首次出现的字符 |

#### 3.5.3 I/O 操作（第 234-356 行）

- `putchar(c: u8) -> i32`：向 stdout 写入单字节（安全函数）
- `getchar() -> i32`：从 stdin 读取单字节，无输入时返回 -1
- `puts(s: *const u8) -> i32`：输出 null 终止字符串 + 换行符
- `gets_s(buf, size)`：带行编辑功能的阻塞式行读取（轮询 + `sys_yield()`），支持 Backspace/Ctrl+C/Ctrl+D

#### 3.5.4 数字转换（第 362-482 行）

| 函数 | 说明 |
|------|------|
| `itoa` | 有符号整数转字符串，支持 2-36 进制 |
| `utoa` | 无符号整数转字符串 |
| `atoi` | 字符串转有符号整数 |

**已知边界问题**：`itoa` 对 `i64::MIN` 取反会溢出，`atoi` 不检测溢出。

#### 3.5.5 便利输出函数（第 488-527 行）

```rust
pub fn print(s: &str)           // 写 Rust &str 到 stdout
pub fn println(s: &str)         // print + 换行
pub fn print_int(value: i64)    // 十进制整数输出
pub fn println_int(value: i64)  // print_int + 换行
pub fn print_hex(value: u64)    // "0x" 前缀十六进制输出
```

### 3.6 src/main.rs（72 行）

Ring 3 验证程序（bin target: `hello`），是最简单的用户态应用。

**程序流程：**
```
_start() → sys_write("Hello from Ring 3!") → sys_getpid() → print_number(pid) → sys_exit(0)
```

- `_start` 为 ELF 入口点，`extern "C"` + `#[no_mangle]` 确保符号名与链接脚本匹配
- 自带 `print_number` 辅助函数：20 字节栈缓冲区，从尾部向前填充十进制数字

### 3.7 src/shell.rs（835 行）

交互式 Shell，是模块中最复杂的应用程序。

#### 3.7.1 常量与配置（第 53-75 行）

```rust
const MAX_CMD_LEN: usize = 128;
const MAX_PATH_LEN: usize = 256;
const READ_BUF_SIZE: usize = 512;
const DIR_BUF_SIZE: usize = 1024;
const O_RDONLY: i32 = 0;
const O_DIRECTORY: i32 = 0o200000;
const DT_DIR: u8 = 4;
```

所有缓冲区均为栈上定长分配，无动态内存。

#### 3.7.2 主循环与输入处理

```rust
pub extern "C" fn _start() -> ! {
    print(BANNER);
    let mut cmd_buf = [0u8; MAX_CMD_LEN];
    loop {
        print(PROMPT);           // "$ "
        let len = read_line(&mut cmd_buf);
        if len == 0 { continue; }
        execute_command(&cmd_buf[..len]);
    }
}
```

`read_line` 实现轮询式行读取（busy-wait + `sys_yield()`），支持 Backspace 回退、Ctrl+C 取消、Ctrl+D 退出。

#### 3.7.3 命令解析

采用 `strncmp` 逐一匹配的线性扫描策略，`is_end(cmd, n)` 防止前缀误匹配。

#### 3.7.4 辅助函数

| 函数 | 说明 |
|------|------|
| `skip_whitespace(s)` | 跳过前导空格/制表符 |
| `copy_arg(args, out)` | 提取第一个参数到缓冲区 |
| `for_each_dirent(fd, visitor)` | 目录项迭代器（泛型高阶函数） |
| `stream_file(path)` | 流式读取文件内容到 stdout |
| `is_numeric_name(ptr, max)` | 判断纯数字名称（用于 ps 命令） |

#### 3.7.5 内建命令实现（15 个）

| 命令 | 涉及 syscall | 说明 |
|------|-------------|------|
| `help` | -- | 输出帮助文本 |
| `echo` | sys_write | 输出参数文本 + 换行 |
| `pid` | sys_getpid | 显示当前进程 ID |
| `ppid` | sys_getppid | 显示父进程 ID |
| `clear` | sys_write | 输出 25 个换行 |
| `exit` | sys_exit | 退出 shell |
| `ls` | sys_open, sys_getdents64, sys_close | 列出目录内容，跳过 `.`/`..`，目录名后加 `/` |
| `cat` | sys_open, sys_read, sys_write, sys_close | 流式输出文件内容 |
| `pwd` | sys_getcwd | 显示当前工作目录 |
| `cd` | sys_chdir | 切换工作目录 |
| `stat` | sys_stat | 显示文件元数据 |
| `ps` | sys_open, sys_getdents64, sys_close | 遍历 `/proc` 目录过滤纯数字子目录 |
| `free` | sys_open, sys_read, sys_close | 流式输出 `/proc/meminfo` |
| `tcp` | sys_socket, sys_connect, sys_sendto, sys_recvfrom, sys_close | TCP 客户端测试 |
| `uname` | sys_uname | 显示系统名称/节点/版本/架构 |

**`do_tcp`**（最复杂命令）：实现完整的 TCP 客户端流程：
```
解析参数(IP, port, message)
  → parse_ipv4() 验证 IP
  → 手动解析端口号（范围 1-65535）
  → sys_socket(AF_INET, SOCK_STREAM, IPPROTO_TCP)
  → SockAddrIn::new(ip, port)
  → sys_connect()
  → sys_sendto()
  → sys_recvfrom()
  → sys_close()
```

**`do_uname`**：调用 `sys_uname` 获取 `UtsName` 结构，逐字段输出 System/Node/Release/Version/Machine。

### 3.8 src/syscall_test.rs（234 行）

系统调用测试套件（bin target: `syscall_test`），验证 musl libc 所需的关键 syscall。

#### 3.8.1 测试框架

```rust
struct TestResult { passed: usize, failed: usize }
```

极简测试框架：`pass(name)` 和 `fail(name)` 输出 `[PASS]`/`[FAIL]` 前缀并累计计数。

#### 3.8.2 测试用例（4 项）

| 测试 | 验证内容 | 关键断言 |
|------|---------|---------|
| `test_gettid` | `sys_gettid()` 返回值 | 单线程下 TID == PID |
| `test_set_tid_address` | `sys_set_tid_address(&mut tid)` | 返回值等于当前 TID |
| `test_set_robust_list` | `sys_set_robust_list(head, 24)` | 正确 size=24 成功；错误 size=16 被拒绝 |
| `test_getrandom` | `sys_getrandom(buf, 16, 0)` | 返回 16 字节且非全零 |

## 4. 模块整体评估

### 4.1 设计优势

**1. 架构清晰，分层合理**

四层架构职责分明，每层仅依赖下层接口。库以 `staticlib + rlib` 双模式发布。

**2. ABI 兼容性好**

系统调用号沿用 Linux x86_64 约定，数据结构使用 `#[repr(C)]` 确保内存布局兼容。

**3. 零开销抽象**

所有 syscall wrapper 标记 `#[inline(always)]`，配合 LTO 和 `opt-level = "z"`，编译后与直接内联汇编等价。

**4. 功能演示完整**

Shell 覆盖了文件系统（ls/cat/pwd/cd/stat）、进程信息（pid/ppid/ps）、内存信息（free）、网络（tcp）、系统信息（uname）等多个子系统。

**5. 安全边界意识**

链接脚本实现了 W^X 段权限分离，`gets_s` 预留 null 终止符空间防止缓冲区溢出，`parse_ipv4` 校验 octet 范围。

### 4.2 现有局限

**1. libc 覆盖面有限**

| 缺失能力 | 影响 |
|----------|------|
| stdio 缓冲（`FILE*`/`fopen`/`fprintf`） | 每次输出都是裸 syscall |
| 全局 `errno` / `strerror()` | 调用者需手动检查返回值 |
| `malloc`/`free`/`realloc` | 所有数据必须栈上定长分配 |
| `printf` 格式化 | 数字输出需手动调用 `itoa` + `sys_write` |
| `signal`/`sigaction` | 无用户态信号处理框架 |

**2. I/O 健壮性不足**

- `print`/`eprint`/`puts` 不处理短写
- `getchar` 返回 -1 时的 busy-wait 轮询消耗 CPU
- 无 `EINTR` 重试逻辑

**3. Shell 功能偏演示性质**

- 无 tokenizer/lexer、无引号/转义处理、无 glob 展开
- 无管道（`|`）、无重定向（`>`/`<`）、无后台执行（`&`）
- 无命令历史、无 Tab 补全
- `do_tcp` 无超时机制、无 DNS 解析、无 TLS

**4. 安全性差距**

| 安全机制 | 当前状态 |
|----------|---------|
| ASLR | 无。固定地址 `0x400000` |
| PIE | 无。ET_EXEC 固定地址模型 |
| Stack Canary | 无 |
| NX/W^X | 有。链接脚本正确分离段权限 |
| TLS/SSL | 无。TCP 明文传输 |

### 4.3 与 Linux 用户空间的差距矩阵

| 能力维度 | Linux 用户空间 | Zero-OS 当前状态 | 差距等级 |
|----------|---------------|-----------------|---------|
| **运行时/ABI** | AUXV、TLS、VDSO、动态链接器 | 纯静态 `_start` 入口 | 高 |
| **内存管理** | mmap/munmap/mprotect/brk + malloc | 仅定义 syscall 号，无 wrapper 和分配器 | 高 |
| **进程模型** | fork/exec/wait/clone + 完整信号框架 | fork/wait/exit 有 wrapper，无 exec wrapper | 中 |
| **线程** | clone/pthread/futex/TLS | 仅 gettid/set_tid_address/set_robust_list | 高 |
| **文件系统** | 完整 POSIX API | 基础 open/read/write/close/stat/getdents | 中 |
| **I/O 复用** | poll/epoll/select/io_uring | 无 | 高 |
| **网络** | 完整 BSD socket + DNS + TLS + IPv6 | IPv4 TCP 客户端（5 个 syscall wrapper） | 高 |

### 4.4 总结

`userspace/` 模块以约 2700 行 Rust 代码实现了一个功能完整的最小 Ring 3 用户态运行时。其核心价值在于：以极简的代码量覆盖了系统调用封装、C 库替代、交互式 Shell 三个层次，为 Zero-OS 内核提供了有效的功能验证和演示平台。syscall ABI 封装质量高、类型安全设计合理、构建配置紧凑。

主要差距集中在：无动态内存分配、无信号/线程框架、无 I/O 复用、Shell 偏演示性质。这些是从"内核验证工具"向"可用操作系统用户空间"演进的必经之路。当前实现作为 OS 开发的 bring-up 阶段产物，设计取舍合理，代码质量良好。
