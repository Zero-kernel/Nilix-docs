# Zero-OS 内核缺口实现总设计（2026-09-13）

**状态（2026-09-20 更新）：** 已实现既有 3.1–3.3 子集与私有匿名 mremap 切片。共享映射生命周期修复经独立源码审查，Ring-3 内存门 13/13 通过，含跨页 usercopy 首触、父子共享、PT 最后叶子退账、真实 cgroup 迁移与 rmdir。完整并发/失败路径及 shared-mmap 专用 musl marker 仍是 verification pending；文件映射与共享 mremap 保持 fail-closed。
**范围：** README 的 “What works, and what is missing” 表格中列出的全部缺口。
**依据：** [README](https://github.com/Zero-kernel/Nilix/blob/main/README.md)、[roadmap](../roadmap.md)、[当前 next-phase plan](https://github.com/Zero-kernel/Nilix-docs-private/blob/main/nextplan/next-phase-plan-2026-09-13.md)、当前工作树源码。
**源码身份：** `git rev-parse HEAD = 8660ba62ed0226f25154456d1908dec048e07c85`；工作树有用户已有修改，本文没有改动任何内核代码。
**设计原则：** Safety > Correctness > Efficiency > Performance。本文是交接设计，不授予实现、提交或发布权限。

English version: [English](kernel-gap-implementation-plan-2026-09-13.md)。真实硬件所需的验证在当前环境不可执行，相关项目均标记为 deferred，不能用 QEMU 结果替代。

本文把“已经有代码但语义不完整”和“明确不支持的能力”分开处理。一个工作包只有在实现、失败清理、并发证明、针对性测试和所需独立审查都完成后，才能从 DRAFT 变为 READY/VERIFIED。

2026-09-13 的 P0-A 跟进已在工作树中完成：fork 在 `Vec::try_reserve_exact` 前为精确的 CoreProcess 快照 charge 预留容量，将一次 reservation 转移到 `AdmittedMap`，并移除无失败处理的 `shrink_to_fit`。构造器错误同时携带 backing 与 armed reservation，调用方按 backing 后 reservation 顺序清理；映射替换不匹配返回 `CapacityInvariant` 并保留现有 map/charge，空 map 回收在异常时 fail-closed 而不触发 invariant panic。两次独立 U23 审查均通过这些安全 ownership 路径。最终工作树的远程 ladder 已在 `40c-devbox-ts` 的 `/tmp/zero-os-codex-final-20260913` 上完成并记录（hosted/MM/core/build/lint 通过，QEMU 套件零失败），2026-09-14 又在未变更的源码上做了本地复验（见 3.1 节）。

工作树同时修正了 ST-K2 shared fork 路径：shared child leaf 保留原始可写权限，父侧不会被重新写入 COW 标记；聚焦的 COW self-test 现在断言这两个性质。这仍是源码级 Phase 2 ownership slice，不能据此宣称 first-touch、usercopy、migration 或完整 teardown 已验收。

## 1. 总体依赖和推进顺序

先完成会影响所有后续分配和对象生命周期的基础工作，再扩大 ABI 和平台范围：

```text
P0-A/R186-4 准入闭环
        │
        ├── ST-K2-P1：mmap flags 诚实化 + stress 管道报告
        │       └── ST-K2-P2：共享匿名内存
        ├── 进程/线程/回收（waitid、PID1、clone 共享对象）
        ├── VFS 对象和目录相对路径
        │       └── ST-K4：fsync/fdatasync/sync/sync_file_range + 恢复
        └── cgroup 初始归属、资源迁移和故障计数
                │
                ├── 原生 capability/IPC、TSYNC
                ├── netns 数据面（veth、路由、控制面、设备迁移）
                └── 用户态 personality、动态 ELF、pthread/OCI

独立但不能宣称已认证：KPTI、早期 W+X、物理 VT-d、retpoline、livepatch、
长期 SMP/性能/热稳定性。它们需要各自的平台或工具链证据。
```

已有的三份深度设计继续作为对应工作包的主体：

* [P0-A/R186-4 准入闭环](https://github.com/Zero-kernel/Nilix-docs-private/blob/main/design/p0-a-r186-4-admission-closure-design.md)；
* [ST-K2 mmap/shared](https://github.com/Zero-kernel/Nilix-docs-private/blob/main/design/st-k2-map-shared-design.md)；
* [ST-K4 durability/writeback](https://github.com/Zero-kernel/Nilix-docs-private/blob/main/design/st-k4-fsync-writeback-design.md)。

下面的章节给出每个 README 组件的当前证据、实现步骤和验收 oracle。实现代理应先阅读本章对应小节，再阅读链接的深度设计，最后重新核对当前源码行号。

## 2. UEFI、启动和早期内存保护（P2/P3，平台依赖）

### 当前缺口

`bootloader/src/main.rs` 已有 PIE 重定位、KASLR、BootInfo 和内存图交接，但启动阶段仍有身份映射和高半内核映射的临时 W+X 窗口（R188 U55-6），并只验证 x86_64/OVMF 这类已测配置。README 还明确没有更广泛固件、硬件和早期 W+X 资格证明。

### 设计

1. 在 bootloader 解析 EFI memory map 后建立一个不可变的 `BootMapReceipt`：每个映射记录物理区间、最终权限、来源 descriptor 和是否临时。所有区间使用半开区间和 `checked_add`；描述符重叠、地址溢出、未对齐或未知类型直接停止启动。
2. 分两阶段切换权限：先建立 RX `.text`、R `.rodata`、RW NX 数据/堆、设备 MMIO 的非执行映射，再切换 CR3；切换前扫描所有 alias，确保没有同一物理页同时出现 W+X。切换完成后只保留必须的启动 trampoline，随后撤销低 4 GiB identity map。
3. `kernel/src/main.rs` 只消费 BootInfo 中已验证的区间，禁止根据常量重新推算物理基址；启动失败必须输出 descriptor index/原因并停止，不得降级成“继续运行”。
4. 增加 UEFI 版本、memory-map 排序、NX/SMEP/SMAP 可用性和固件 RNG 的 profile；不支持的 profile 明确记录为 unsupported。

### 失败、清理和测试

* 注入大于 64 页的 memory map、重叠区间、末端溢出、非规范地址和 framebuffer 越界，期望 fail-closed 且不访问越界地址。
* 在 QEMU 中在权限切换前后读取页表，验证 `.text` 从未以可写权限暴露；验证切换后低身份映射不可执行/不可写。
* `make boot-check` 必须看到串口到达 Ring 3，并记录实际映像 hash；不同 UEFI/CPU 能力用独立结果，不用默认 QEMU 结果代表全部平台。

## 3. 内存管理和虚拟内存（P0/P1）

### 3.1 准入闭环（P0-A/R186-4）

`kernel/kernel_core/fork.rs` 在 `fork_inner` 组装快照和页表时负责子进程资源路径；`kernel/mm/heap_admission.rs` 负责 reserved/committed 双计数，`kernel/mm/admitted.rs` 负责 admitted 容器。当前实现已在分配前为精确快照 charge 预留容量（`fork.rs:773-776`），将一次 reservation 转移到最终的 `AdmittedMap`（`fork.rs:788` 的 `from_sorted_vec_with_reservation`），并不再执行可能 panic 的 shrink（`fork.rs` 与 `kernel/mm/*.rs` 全树无 `shrink_to_fit`）。构造器错误通过 `into_parts()` 同时返回 backing 与 armed reservation（`fork.rs:791-795`），调用方按 backing 后 reservation 顺序释放。诱导的 COW/PT/子进程发布清理 oracle、映射替换不匹配路径（`admitted.rs` 的 `CapacityInvariant` 系列）和构造器 owner 清理 oracle 均已在本地通过；两次独立 U23 审查通过这些安全 ownership 路径。

实现顺序和不变量以 P0-A 设计为准：

1. 根据实际 capacity 计算一次 charge，在任何快照分配前调用 `try_reserve`；将 reservation 通过一次 transfer 交给最终 `AdmittedMap`，禁止重复 reserve 或先释放再重新获取。
2. COW 父页表变更是 commit 点。此前所有可失败步骤（PCB、fd/cgroup charge、child root、KPTI root、COW plan）失败都必须由 RAII guard 释放一次；commit 后路径不得再返回可恢复错误。
3. 普通 `CorruptState` 错误必须向上传播；`HeapReservation`/`HeapCharge` 的 Drop fail-stop 只作为既有计数损坏后的显式边界，并且通过线性安全 API 不可达。映射替换不匹配返回 `CapacityInvariant`，而不是依赖 invariant panic。
4. `remove` 只移除逻辑条目，保留 backing 的 capacity charge 直到实际 retirement；fork/exec/unmap/reap 都使用同一所有权表。

验收：记录 reserved/committed 在正常、OOM、页表失败、child publish 失败、压力回收后的精确 delta；诱导 COW/PT 错误，确认父进程映射、child PID、fd/cgroup 计数和页表物理帧都回到基线。本地证据为 `cargo test -p mm --features host_harness --lib --locked`（26/26）、`cargo test -p kernel_core --features host_harness --lib --locked`（67/67）、`cargo check -p kernel --locked`（通过）和 `cargo fmt --all -- --check`（通过；两个受影响 Rust 文件已统一为仓库要求的 CRLF）。远程阶梯已在 `/tmp/zero-os-codex-final-20260913` 上完成：MM 26/26、kernel-core 68/68、hosted allowlist 438 次执行（外加 CpuLocal doctest 与 3 项编译检查）、`make build` 与 `make lint` 通过；runtime 35/39/0、boot 28/46/0、四核 SMP 37/37/0 零失败，musl 因缺少 guest marker 仍不完整。2026-09-14 的本地复验仍为 26/26 与 67/67；**67 与 68 的差异是环境差异而非回归**：第 68 项是 `usercopy::robust_usercopy_tests::ksa002_cmpxchg_mapped_success_and_mismatch`（`kernel/kernel_core/usercopy.rs:1689`），带 `#[cfg(target_os = "linux")]`，在 Windows target 上被 cfg 排除。本轮没有重跑远程阶梯。

### 3.2 mmap flags、MAP_SHARED/MAP_FIXED 和文件映射（ST-K2）

当前入口为 `kernel/kernel_core/syscall.rs:13798` 的 `sys_mmap`：匿名映射已分三阶段并使用 `MmState.mmap_regions`。当前工作树在 LSM hook 后调用 `validate_mmap_flags`（`syscall.rs:11835`，调用点 `:13850`）校验完整 flags：只有 `MAP_PRIVATE|MAP_ANONYMOUS` 和 `MAP_SHARED|MAP_ANONYMOUS` 进入匿名路径；MAP_FIXED、不完整的 shared/validate 形态和未知位在修改 VMA/cgroup/PT 前返回 `EOPNOTSUPP`。文件描述符映射（`fd >= 0`）在 `:13860-13866` 返回 `EOPNOTSUPP`，shared `PROT_NONE` 在 `:13879-13883` 返回 `EOPNOTSUPP`。

**Phase 1（先做，参见 ST-K2 设计）：**

1. **Phase 1 已实现：** 在 LSM hook 之后、地址空间修改之前解析完整 `flags`。接受 `MAP_PRIVATE|MAP_ANONYMOUS` 和 shared-anonymous 形态；未知位、缺少 `MAP_ANONYMOUS` 的 `MAP_SHARED`、`MAP_SHARED_VALIDATE`、`MAP_FIXED` 返回 `EOPNOTSUPP`，不能悄悄按 private 映射成功。`run_mmap_flags_self_test()` 已接入 integration memory-mapping gate。
2. `addr != 0` 仍按页对齐、用户上限、stack window、brk reservation、现有 VMA 和 map-count 检查；只有未来专门的 MAP_FIXED 工作包才能允许覆盖，且必须先拆除旧 VMA 并准备完整回滚。
3. `stress_runner.c` 的 report page 和 CPU worker 结果已改为有界 pipe 记录；父子端关闭未使用 fd，处理短读/短写、EINTR、EOF、child failure 和超长记录，缺失记录不能被解释成 0 次工作。CPU profile 为每个 worker 使用独立 pipe；SMP 和 combined 仍有意使用 shared counters，继续属于 Phase 2 边界。

**Phase 2（共享匿名，已实现到按需首触）：**

共享匿名映射使用 admitted 的 `SharedAnonRegion` side map（`kernel/kernel_core/fork.rs:1909`）：槽位 backing 与 `HeapCharge` 在 `try_new`（`:1918`）中于任何分配之前全量预留，charge 保留到 region Arc 最终销毁（`Drop` 在 `:1974`）。`sys_mmap` 只发布元数据、不分配数据页（`syscall.rs:14033-14043`）；首次用户态或 usercopy fault 由 `handle_shared_page_fault`（`fork.rs:1704`）处理：全程只用 try-lock，在 `PT_LOCK` 之前预留有界的 DATA+PT 额度（`:1747-1775`），零化并映射一页，在 region 槽位记录 owner 与 cgroup 归属；`map_page` 失败只拆除已证明为空的中间页表，并把可达 PT 帧计入保留账（`:1825-1850`）。fork 复制 side-map Arc 与共享叶子并保留可写权限，同时只增加一个 mapping 引用（`:805-839`）；父进程 PTE 不变。shared `mprotect` 在任何边界拆分之前返回 `EOPNOTSUPP`（`syscall.rs:15036-15045`）。

2026-09-20 跟进：fault 检查 VMA transient/fork 状态，在 charge 前识别已被其他 CPU 解决的缺页；munmap 按实际拆除叶子执行跨 CPU TLB 刷新。新增预留容量的 per-AS shared-fault PT 物理帧账，PT 随地址空间存活和迁移，DATA 保持首触者归属，不能把 PT 费用绑在 region Drop 上。详见[生命周期设计及验证](https://github.com/Zero-kernel/Nilix-docs-private/blob/main/design/st-k2-shared-fault-lifecycle-design.md)。待验收项是完整 SMP fault/unmap/exec/exit 竞争、精确物理帧最后引用释放、PTE-only syscall preflight、通用 private rollback 的账目回收，以及 shared-mmap 专用 musl marker。

1. `PAGE_REF_COUNT` 已是物理帧的唯一 owner；VMA 与 side-map 只保存共享对象和页槽，不在每个进程重复释放帧。
2. 首次 fault 的 charge 与提交顺序已按上述实现；charge 失败保持未映射，不出现“有页无 charge”。
3. fork/exec/munmap/进程退出按引用计数释放，最后一个引用才回收帧；2026-09-20 guest 已验证：任务迁走后仍由共享 DATA 持有的原 cgroup 对 rmdir 返回 EBUSY，最后 munmap 后费用归零且 rmdir 成功；未首触 region 尚无 DATA owner。

**文件映射与 mremap 的当前边界：**

文件映射 mmap 仍返回 EOPNOTSUPP；sys_mremap 已实现私有匿名/PROT_NONE 的原地扩展、缩小与 MAYMOVE，guest 用例实际覆盖这些路径。共享 mremap、FIXED/DONTUNMAP 和文件映射仍未实现，既有私有 mremap 的完整独立事务审查/OOM/竞争验收仍待完成。以下是未支持边界的目标设计，不是当前实现证据：通过 fd 表得到 pinned `FileHandle`/`Inode`，页缓存条目使用 `(fs_id, ino, page_index)`，fault 时验证 inode 仍有效并持有 mount lifetime；`sys_mremap` 采用 prepare/commit 事务，先检查旧 VMA 完整覆盖、长度和溢出，再为扩展或新地址预留 VMA，复制/移动 PTE，成功后一次性更新 cgroup 和 VMA，任一失败撤销新 VMA、TLB shootdown 和已分配帧。`MREMAP_FIXED` 必须显式带 `MAYMOVE`，目标区间不能覆盖 stack/brk/其他 VMA；共享映射迁移只移动引用，不复制帧。以上内容不是已实现的证据。

验收必须区分执行环境。run_mmap_flags_self_test、run_cow_refcount_self_test、run_cow_failure_cleanup_self_test 和 run_memory_capability_self_test 的集成调用仍由 guest 执行，hosted 编译不等于执行这些路径。hosted kernel-core 现有共享 region 元数据测试和新加的 PT 账本来源测试，仍不执行真实 mmap/fault。2026-09-20 make test-ring3-mm 为 13 passed / 0 failed，覆盖私有 mremap、共享父子可见性/子先首触、跨页 uname usercopy、相邻 private VMA 保留 PT 的三轮回收，以及真实迁移/rmdir。2026-09-25 在 §4 的 waitid/F2/ROOT-INIT 改动落地后，于同一台 x86_64 Linux 主机重跑该门：报告 `Results: 13 passed, 0 failed`、`Process 1 terminated with exit code 0`，门禁退出码 0——八条契约腿全部 `[PASS]`，cgroup 迁移腿是直接通过而非 qualified，唯一的 `[SKIP]` 是被允许的 PTE-only preflight 备选项。该次重跑的意义在于证明进程生命周期改动没有回归内存子系统，并不构成新的内存覆盖。同日稍后的一次运行在同一 fixture 中加入了 §4 的 `waitid` 腿，报告 `Results: 14 passed, 0 failed`，27 条 `[PASS]` / 1 条 `[SKIP]` / 0 条 `[FAIL]`，所有内存腿依然通过，门禁 `FATAL` 正则零命中。基础 musl 门通过，但不是 shared-mmap 专用 musl 证据。stress_runner.c 的 CPU/SMP 消费者已实际运行；额外压力门被 host 端旧 gate_log import 路径阻断，不能以工作负载 marker 宣称门禁通过。完整限制与源码/日志身份见上述跟进设计。`kernel/src/runtime_tests/regression_tests_p0.rs` 中有三个看似属于 §3 覆盖的运行时用例其实是占位符，什么也不验证：`r23_1_cow_tlb_shootdown`、`r174_b4_brk_va_reservation_toctou` 与 `m0_7_stack_guard_page` 在每条路径上都返回 `TestResult::Deferred`。其中 COW shootdown 用例最容易误读：单 CPU 时以 "requires 2+ CPUs" 延期，两个及以上 CPU 时则以 "full COW test requires fork + concurrent memory access" 延期，因此多核运行只会改变延期信息，仍然什么也不检查。在这些用例体被实现之前，COW TLB shootdown 与 brk VA 预留 TOCTOU 都未经验证；单靠增加 CPU 无法关闭它们。栈保护页行为现已改由 Ring 3 验证（见下），但 `m0_7_stack_guard_page` 运行时用例体本身仍是占位符。**用户栈保护页的 guest 覆盖（2026-09-25 新增）。** 用户栈窗口是固定的（无栈 ASLR），因此 `test_user_stack_guard` 直接按布局寻址，而非用无界递归去触发。它在 fork 出的子进程中执行，因为命中保护页按设计是致命的：在 `USABLE_BASE`（`0x7fffffdff000`，最低的惰性页）写入必须触发按需增长并成功；在保护页顶端字节（`USABLE_BASE - 1`）与底端字节（`STACK_BASE`，`0x7fffffdfe000`）写入必须各自以退出码 139（`128 + SIGSEGV`）杀死子进程。探测保护页两端能捕获增长边界的差一错误——那种错误只会映射保护页顶部。该腿先检查自身栈帧位于 `[EAGER_FLOOR, USER_STACK_TOP)` 内，一旦布局变动就直接失败，而不去探测无关内存。保护页故障只打印 `[PF ENTRY]` 并走 `USER_MODE` 终止路径，从不进入 `[PAGE FAULT]` 内核分支，因此不会触发门禁的 `FATAL` 正则。**`pids.max` 下 fork 拒绝的 guest 覆盖（2026-09-25 新增）。** `test_fork_pids_refusal` 加入 `pids.max = 1` 的 cgroup 后尝试四次 fork：每次都必须返回 `EAGAIN`，cgroup 的 `nr_tasks` 必须不变，且没有任何可收割的子进程。离开该 cgroup 后的一次 fork 得到 pid 7，紧接上一条腿的 pid 6，因此被拒绝的 fork 没有消耗任何 PID。这钉住了可观察的回滚契约，但并不显示是哪个拒绝点生效——cgroup 中已有一个任务时，`check_fork_allowed` 预检会在 `attach_task` 之前拒绝，因此 `fork::sys_fork` 中 attach 阶段的回滚只被间接覆盖。

### 3.3 slab/NUMA/swap/THP

有界能力原语已在 `kernel/mm/capabilities.rs`（375 行）中实现：`SlabCache` 在构造时预留 slot 与 quarantine 两份 `AdmittedVec` backing，并保持非阻塞的固定锁序，使 IRQ/fault 调用者得到 `Contended` 而不是阻塞；`NumaAllocator` 覆盖 64 个 node，用原子页扣减分配，迁移时先预留目标再释放源；`SwapSlotTable` 在保留的 `HeapCharge` 下运行 free/reserved/committed 原子状态机；`ThpMapping` 强制 2 MiB 对齐并提供 split/merge 几何辅助。`run_memory_capability_self_test()`（`capabilities.rs:354`）由 **guest** 集成内存映射门调用（`kernel/src/integration_test.rs:381`），**不是** hosted MM 套件：`capabilities.rs` 没有声明任何 `#[test]`，因此 hosted MM 的 26 个用例没有一个执行它，3.1 节记录的 hosted 通过也不覆盖 3.3。

后端接入仍待完成——先保留 buddy/全局 heap 作为唯一 owner，新增功能必须接入 `HeapClass` 准入和 cgroup charge：

* slab：cache 接入实际 slab pages 的 partial/full/empty 链表；释放不在 IRQ 持锁路径执行最终回收。
* NUMA：BootInfo 记录 node ranges，per-node buddy + 跨 node fallback；charge 按实际 node 总量，迁移必须有 reservation。
* swap：接入 swap slot owner、PTE 状态和写回错误；OOM 时不能把 dirty page 当作可回收页。
* THP：只在完整 2 MiB 对齐、可拆分、COW/munmap/mprotect 已证明后启用；否则返回 `ENOMEM/EOPNOTSUPP`。

每个能力单独设计和性能/压力 oracle，不能因为接口存在就从 README 删除缺口；平台资格仍未完成。

## 4. 进程生命周期、PID1 和等待（P1）

### 当前证据

`sys_wait4` 在 `kernel/kernel_core/syscall.rs:7595`；`sys_waitid` 在 `:23270` 已**实现**（2026-09-25），并在 `:4368` 分发（syscall 247，与 Linux x86_64 一致；seccomp 进程管理组早已以 `SYS_WAITID`（`kernel/seccomp/types.rs:194`）包含它）。两个调用方共用同一套收割引擎 `wait_reap_core`（`:7661`），由 `WaitTarget`（`:7633`）和一个写出各自 ABI 的 `publish` 闭包参数化。`kernel/kernel_core/process.rs:9525` 已有 namespace 感知的孤儿改嫁，`try_commit_reaper` 在 `:9635`。ROOT-INIT 现已在启动时注册，F2 的“无 current user frame”fork 拒绝也已实现（均见下）。

### waitid 设计

实现方式是把 wait4 的 scan/wakeup 路径抽取为 `wait_reap_core` 而非复制一份，因此两个 syscall 共享选择符解析、namespace 过滤、阻塞与回滚。`WaitTarget` 覆盖 `P_ALL`（`:23179`，在 `:23241` 解析）与 `P_PID`；`P_PGID`（`:23183`）在 `:23248` 按失败关闭返回 `EINVAL`，理由与 `wait4` 拒绝进程组相同——进程组成员关系尚不是被跟踪的关系，接受它会静默退化为 `P_ALL` 而收割错误的 child。`idtype` 与整个 options 字在任何状态变更之前完成校验；`WEXITED` 为必需位。

ABI 差异集中在 `publish` 闭包：`wait4` 写出 `c_int` status 并返回 namespace PID，`waitid` 写出 128 字节 `siginfo_t`（`si_signo=SIGCHLD`、`si_code=CLD_EXITED`、`si_pid`=namespace PID、`si_uid`、`si_status`）并返回 0。发布发生在收割提交**之前**：`infop`/`status` 指针出错时恢复 `Ready`、清除 `waiting_child`、让 zombie 仍可被收割并返回 `EFAULT`，与 Linux 一致。`WNOWAIT`（`:7885`）只观察不消费——child 仍在链上且不调用 `cleanup_zombie`，后续 wait 仍可回收，只回滚 parent 自身的等待状态。`WNOHANG` 不阻塞即返回（`wait4` 返回 0，`waitid` 返回全零 `siginfo`）。`WUNTRACED`/`WCONTINUED` 以及未知 option 位都**返回 EINVAL** 而非静默忽略：本内核不记录 parent 可见的 stop/continue 事件，接受这些位等于承诺一个永不到达的通知并挂住调用者。`wait4` 继续忽略它们仅为向后兼容。

### PID1/reaper 与 F2

**ROOT-INIT（2026-09-25 关闭）。** 根 PID namespace 的 init 现在与其他所有 init 在同一处注册，即 `assign_pid_chain` 内部（`pid_namespace.rs:1499-1513`），而不是走单独的 boot hook——根分支只是在 `membership.pid == 1` 时不再被跳过。因此 `reparent_orphans` 的 STEP 3（`process.rs:9602`）从 `ROOT_PID_NAMESPACE.init_global_pid()` 解析兜底 reaper，仅在 pid 1 存在之前才回落到裸常量 `ROOT_INIT_PID`。这一点之所以重要：该注册带身份校验并在 detach 时清除（`clear_init`，`:1229`），因此不可能指向被回收复用的 pid，而数字常量可以。

有两处不对称是刻意的。根注册是**尽力而为**（`let _ = set_init(..)`）：与非根分支不同，`InitAlreadySet` 不能让链分配失败，因为让整个 PID 分配失败严格比保留既有映射更糟。另外根被排除在 init 死亡级联之外（`handle_namespace_init_death` 跳过 `is_root()`，`process.rs:9721`），所以注册它不会启动任何 kill 级联——这是已验证的，不是假设。

Reaper 提交保持既有的一次一锁协议与 generation 检查，杜绝自收养与 ABA。pid 1 永不能成为自己的 reaper；根 init 缺失或处于 Zombie/Terminated 时走 `reparent_logged_leak`——确定的 ppid 加一条泄漏日志——而不是 panic，因为在销毁路径上 panic 会重新打开本切片正要关闭的进程遗弃问题类。

**F2（2026-09-25 关闭）。** `fork_inner` 的无 frame 分支（`fork.rs:660`）过去会被任何 syscall-frame 窗口缺失的父进程进入，并以复制、重定位父进程内核栈（取自 `parent.context`）的方式为子进程播种。这个模型对内核线程成立，对 Ring-3 父进程则是错的：它恰好重建了上方一个分支中 ST-K3 修复所消除的“嵌合”状态——用户段选择子下的内核 `rsp`，最终在 `switch_to_user` 中触发 `ud2`。该分支现以 `ForkError::InvalidUserFrame` 拒绝用户父进程，syscall 层将其映射为 `EFAULT`。

判别式为 `kernel_stack_resume_is_admissible(parent.memory_space)`（`fork.rs:520`），即 `memory_space == 0`——本内核既有的内核线程判定，与 OOM 候选扫描使用的惯用法相同。它刻意从 PCB 读取，而非从 frame 窗口推断，因为调用方之所以走到这里，*恰恰是*因为该窗口缺失或已被放弃；因此一个被放弃的 frame 无法把用户父进程伪装成内核父进程。拒绝发生在发布之前：`fork::sys_fork`（`fork.rs:289-302`，拥有子进程创建的外层包装）会让每个 `fork_inner` 错误依次经过子链接移除、cgroup detach、scheduler permit 释放与 `cleanup_partial_child`（`:300`）后才返回，所以不会残留子 PCB、PID 链条目或 cgroup 计费；`syscall.rs` 中的 `sys_fork`（`:4524`）只负责映射所得的 `ForkError`。选用 `EFAULT` 而非瞬时错误码正是要点——没有任何重试能补上缺失的 frame，`ENOMEM`/`EAGAIN` 只会诱发空转重试。

测试状态是部分完成，值得直说。选择符/option 校验与 `siginfo_t` 编码的宿主判定已于 2026-09-25 在 x86_64 Linux 主机上**执行并通过**（见 §19）；它们在 macOS/arm64 上无法编译，因此只能在远端运行。两个 F2 判定（`f2_frameless_fork_tests`，`fork.rs:470`）仅覆盖谓词本身——`memory_space` 为零时允许内核栈续跑、非零时不允许——这是有意为之，因为 `fork_inner` 本身需要活地址空间，而任何宿主 fixture 都提供不了；拒绝动作位于发布之前这一点是从 `fork::sys_fork` 错误路径读出的，未经执行。**`waitid` 的 guest 覆盖（2026-09-25 新增）。** 一条 Ring-3 腿——`userspace/src/syscall_test.rs` 中的 `test_waitid`——现经真实分发臂驱动 syscall 247，作用于真实 zombie，并作为具名条目写入 `ring3_mm_oracle.sh` 的腿契约，因此删除或改名都会使门禁失败。它覆盖了宿主判定做不到的部分：六种畸形参数各自返回 `EINVAL`；无子进程时即便带 `WNOHANG` 也返回 `ECHILD`（`wait_reap_core` 中的无子进程检查先于非阻塞分支）；`WNOWAIT` 窥视向被投毒的 Ring-3 缓冲区写回 `si_signo=SIGCHLD`、`si_code=CLD_EXITED`、子进程 pid 与 `si_status=42`；出错的 `infop` 返回 `EFAULT`；zombie 此后在两者之后依然存活，并仍能被普通 `WEXITED` 等待收割；第二次收割返回 `ECHILD`，证明它已消失；`si_status` 之后的 `siginfo_t` 尾部读出为零而非毒值；`P_ALL` 以 `si_status=7` 收割第二个子进程。每条失败路径都会收割每个 fork 出的子进程，因此出错的子检查不会把 zombie 泄漏给后续各腿。**ROOT-INIT 孤儿改嫁的 guest 覆盖（2026-09-25 新增）。** Ring-3 门以根 namespace 的 PID 1 身份启动该 fixture，因此第二条腿 `test_root_init_reparent` 能端到端地检验 ROOT-INIT 修复：PID 1 fork 出中间进程，中间进程再 fork 出孙进程后退出，`reparent_orphans` 必须把孙进程交给 PID 1。只有当 PID 1 能以 `P_PID` *按 pid* 收割该孙进程时这条腿才通过——这只有在内核把它链入 PID 1 的 `children` 列表后才可能，单纯改写陈旧的 `ppid` 并不满足——而孙进程自身的退出码报告 `getppid()` 已变为 1，而非变到别处或根本未变。之后 PID 1 不再留有任何子进程（`WNOHANG` 下为 `ECHILD`）。其首次运行失败源于测试本身的竞态，而非内核：孙进程通过采样 `getppid()` 来得知原始父进程，当它在中间进程已退出之后才被调度时，该采样值已是 1，于是“是否变化”的检查永远不会触发（退出码 93，`GC_NEVER_ORPHANED`）。现在中间进程在 fork 前记录自身 pid，孙进程改为与之比较。`pids.max` 下的 fork 拒绝已经 guest 验证（见 §3）。PID1 *退出*、reaper 竞态，以及具体的 attach 阶段与 fd 计费回滚路径，需要超出单棵进程树的活进程状态，目前仍没有任何判定覆盖。2026-09-25 的启动门（`make test`）能正常启动并收割：`GATE-LOG QUALIFIED (not a strict pass): passed=35 deferred=39 failed=0 warning_markers=1 skipped_markers=39`，harness `gate status=3 (qualified)`，Ring-3 init 运行至 `Process 1 exited with code 0` / `Process 1 terminated with exit code 0`。QUALIFIED 由 39 个 deferred 与 39 个 skipped 标记导致，而非任何失败——`failed=0`。该门留存的日志只保留了截断的 serial 尾部，因此逐条判定改从 Ring-3 门的完整 serial 工件（`/tmp/tmp.VT4mIwQwyr`，同一源码树、同一启动序列）读取：其中记录了 `[PASS] RF180-40 process Arc/Weak exact-lifetime admission`、`[PASS] RF180-44 process-table deferred retirement`、`[PASS] D2-ARC process-registry transaction` 与 `[PASS] RF178-33 selector/aging/identity-cleanup + RF178-35 fatal wake + RF178-36 identity resume`，`[FAIL]` 计数为 0。这证明 ROOT-INIT 注册与重构后的 `wait4` 路径能正常启动并收割；`waitid` 专属的 guest 证据是上文的 Ring-3 腿。经执行验证的：`waitid` 参数契约、`siginfo_t` 编码与回写、WNOWAIT 存活、EFAULT 先于收割、`P_ALL`（宿主与 guest 均有）；F2 谓词（仅宿主）。ROOT-INIT 孤儿收养、`pids.max` 下的 fork 拒绝与用户栈保护页已经 guest 验证。未经执行验证的：活 fork 中 F2 的拒绝顺序、PID1 退出、reaper 竞态，以及 attach 阶段与 fd 计费回滚路径。

## 5. 线程、TLS 和 pthread 前置条件（P1）

### 当前缺口

`sys_clone` 的 flag 定义和限制在 `kernel/kernel_core/syscall.rs:4610-4685`；当前明确拒绝 `CLONE_THREAD/CLONE_FILES/CLONE_FS/CLONE_SIGHAND` 组合，仅保留受限 `CLONE_VM|TLS`。`Process` 已有独立 `rlimits`、`cap_table`、`signal` 和 fd 字段，不能直接把字段改成共享而不处理 Drop。

### 实现方案

1. 引入四个显式共享对象：`MmHandle(Arc<Mutex<MmState>>)`、`FilesHandle(Arc<...>)`、`FsContextHandle(Arc<...>)`、`SignalTableHandle(Arc<...>)`。每个对象带 generation 和 owner count；PCB 只持 handle，不复制底层 map。
2. 先实现 `CLONE_VM|CLONE_SIGHAND|CLONE_THREAD`，要求独立用户栈、同一 tgid、不能同时创建 namespace；然后实现 `CLONE_FILES` 和 `CLONE_FS`，最后组合。任何不满足依赖的 flag 继续 `EINVAL`。
3. `CLONE_FILES` 的 fd refcount 只在共享表 clone/close/exec-CLOEXEC 处改变一次；表扩容走 `AdmittedMap`，超过 `RLIMIT_NOFILE` 返回 `EMFILE`。
4. `CLONE_SIGHAND` 共享 disposition，但 blocked mask、pending queue、robust list 和 clear-tid 仍按线程保存；`exit_group` 先设置共享 exiting generation，再唤醒并清理全部线程。
5. TLS 设置在 child 尚未 runnable 时写入，context switch/interrupt return 恢复 FS/user-GS；失败时销毁 child，不发布 tgid 成员。

### 验收

用真实静态 pthread-like workload 验证共享内存、fd offset/status、cwd/root、signal disposition、TLS、fork/exec/exit、robust futex 和线程组等待。加入 SMP 迁移、同时 close/dup、exec 与 clone、线程组退出和用户栈 fault；所有 cleanup 必须在锁外执行 Arc 最终 Drop。

## 6. 调度、SMP 和资源压力（P2/P3）

### 缺口和方案

当前 `kernel/sched/enhanced_scheduler.rs` 是 per-CPU MLFQ/steal/balance，CPU-local 设有 64 CPU ceiling，但 README 明确没有高核数、长时间或物理并发资格。ST-K3 残余 F7（8-page kernel stack/PCB 碎片）、F4/F6（持锁诊断和 switch sentinel）、W-3/W-4（wait 选择/嵌套 zombie）仍在计划中。

1. 先做资源盘点：统计 kernel stack、PCB、run queue、RCU callback 的实际容量和连续页需求；为 8-page stack 提供 admitted contiguous allocation，失败返回可诊断错误，不在 allocator 中 panic。
2. 所有 scheduler/PCB 锁内日志改成固定大小事件，锁外格式化；context switch 写入编译期 sentinel，resume 后核对 task generation 和 CPU id。
3. steal 路径保持 `READY_QUEUE -> SCHEDULER_STATS` 顺序，不能持 ready lock 调用 blocking cpuset/registry read；跨 CPU 只使用 try-lock 或先复制 id。
4. wait queue 只唤醒 selector 匹配的 waiter，单次 wake 有上限并记录 over-wake 计数；嵌套 zombie 用原始 wait4 rubric 再验证，不能用 KSA-011 的 namespace 结果代替。
5. 以 2/4/8/16 CPU profile 运行 fork/exec/exit、affinity/cpuset、TLB shootdown、futex PI、net RX、page fault；每个 profile 保留实际 QEMU `-smp`、CPU list、运行时长、迁移计数和错误计数。

验收要求长跑无 deadlock/lost wakeup/RCU backlog，CPU 负载和 task accounting 在迁移前后守恒。64 只作为拒绝超限的上界，不能作为“已测试拓扑”。

## 7. IPC、futex、信号和时间（P1）

### 7.1 原生同步 IPC和共享内存

`kernel/ipc` 已有 pipe、endpoint、futex 原语，但 README 不宣称完整同步消息或共享内存；原生 capability API 也尚未闭合。实现按 capability 先行：

1. endpoint object 保存 bounded message slots、sender/receiver wait queues、closing generation；发送者先 reserve slot 和 payload charge，再发布消息，失败撤销 reservation。
2. `endpoint_call` 使用 request/reply capability，reply cap 只能被一次消费；超时、对端退出和 namespace teardown 都唤醒等待者并返回明确 errno。
3. 共享内存对象复用 ST-K2 的 shared-anonymous frame owner；映射权限、长度、offset 和 cgroup charge 在 cap invoke 时一次校验。

### 7.2 futex 和信号

`sys_futex`（`kernel/kernel_core/syscall.rs:16349`）当前只覆盖少量 WAIT/WAKE/PI 操作。新增 opcode 前先完成 key 解析（PRIVATE/SHARED）、bitset、requeue、robust list 和 PI chain 的深度/超时策略；任何用户地址 fault 都在持 bucket lock 前完成验证。

信号方面新增 per-thread alternate stack、完整 `siginfo_t`、queued RT signal bounded queue 和 `SA_RESTART` 重启点表。signal frame 必须在用户栈/altstack 映射范围内一次性 reserve，copyout 失败不改变 pending 状态；`rt_sigreturn` 验证 CS/SS/RFLAGS/RIP/RSP 和所有用户指针。

epoll/eventfd/timerfd 只有在 pipe/futex wait ownership 稳定后实现，全部使用统一 wait-key/generation，避免为每个 fd 创建无限 watcher。

### 验收

覆盖 waiter 超时、EINTR、signal interrupt/restart、PI 链、owner death、requeue、共享 futex 跨进程、altstack 溢出、RT 队列上限和 endpoint 对端退出。测试必须包含 SMP 竞争和用户 copy fault，不能只测成功路径。

## 8. 文件描述符和 rlimit（P1/P2）

当前 `Process` 在 `kernel/kernel_core/process.rs:1687-1717` 定义 16 个 rlimit 槽位，`RLIMIT_NOFILE` 已用于 fd cap；其它 fcntl/线程共享语义仍缺失。

### 实现步骤

1. 把 fd 表操作拆成 `FdTable` API：lookup、reserve、install、replace、close，所有 API 返回 `Result`，禁止调用者直接操作底层 map。reserve 返回 token，只有 commit 才发布可见 fd。
2. 实现 `F_GETFD/F_SETFD`（CLOEXEC）、`F_GETFL/F_SETFL`（共享 open-file-description status）、`F_DUPFD/F_DUPFD_CLOEXEC`；`F_SETFL` 只允许可变位，O_APPEND/O_NONBLOCK 通过共享原子状态影响所有 dup。
3. 完善 `getrlimit/setrlimit/prlimit64`：soft 不得超过 hard，降低 hard 需要权限；实时 fd 数量和 `files.max`/cgroup charge 一起检查。fork/exec/CLONE_FILES/exit 每个 ownership 变化只有一个 charge/uncharge 点。
4. 将 `RLIMIT_AS/DATA/CPU/CORE/FSIZE/STACK/NPROC` 标注为 enforced 或 advisory；没有真正 enforcement 的槽位返回文档化的 unsupported，不在 ABI 中假装成功。

### 验收

测试 dup/close race、fork/exec CLOEXEC、共享表、fd 0/1/2 被普通文件替换、fd table OOM、NOFILE 限制、setrlimit 权限和 cgroup files.max 事务。任何错误路径都必须关闭 token、撤销 cap refcount 并在锁外 Drop。

## 9. VFS、路径和文件元数据（P1）

### 当前证据

`kernel/vfs/manager.rs`/`traits.rs` 已有 `FileHandle`、`FileLifetime`、ramfs/ext2/JBD2、cwd/root/pivot 和 symlink resolver；但 `sys_statx`（`kernel/kernel_core/syscall.rs:17265`）、`sys_chown/fchown/lchown`（`:21797-21808`）仍为 ENOSYS，普通 dirfd-relative 组合被显式拒绝，hard link、terminal fidelity 和完整 POSIX 未实现。

### 统一路径对象设计

1. 在 `FileHandle` 中保存 pinned `FileLifetime`（Filesystem 或 Mount）和 inode Arc。相对路径解析先在 PCB 锁下 clone dirfd handle，再释放 PCB 锁，解析期间不重新从当前 namespace 查找，避免 pivot/umount retarget。
2. `PathContext` 包含 base directory inode、root jail、mount generation、follow-final-symlink 标志；每一步 lookup 验证 mount generation 和 execute permission，循环深度有固定上限。
3. 所有 `*at` syscall 先验证 dirfd 是目录且可执行；`AT_FDCWD` 使用线程 FsContext。绝对路径忽略 dirfd，非 AT_FDCWD 的相对路径在支持前返回 `ENOTSUP`，不能默认为 cwd。

### 元数据和操作

* inode 增加 uid/gid/mode、nlink、timestamps、fs id/mount id、file flags，更新通过 filesystem transaction；DAC 由 credential/user namespace 转换后检查，CAP_CHOWN 等 authority 由 LSM hook 决定。
* `chown/fchown/lchown` 共享一个 `set_owner` 核心：path 版本选择 follow/no-follow，fd 版本直接用 pinned inode；uid/gid 为 `UINT32_MAX` 表示不变，权限和 setid 清除规则集中处理。
* hard link 只允许同一 filesystem、非目录、link count 未溢出；先 reserve directory entry 和 inode nlink，再原子提交，失败完整回滚。unlink/rename 与 link 使用同一 inode lock 顺序。
* `statx` 定义 `#[repr(C)]` 256-byte 结构和 mask/flags 解析，先填基础 stat，再按 mask 填 btime/mount id/attributes；`copy_to_user` 只写 caller buffer 长度，`STATX_*_SYNC` 不得承诺不存在的同步。
* terminal/termios 另设 device FileOps 状态机；在此之前对不支持的 ioctl 返回 `ENOTTY`，不要返回静态成功。

### 验收

创建/删除/重命名/硬链接并发、跨 mount 拒绝、chown 权限和 setid、statx mask/短 buffer、dirfd 被关闭或指向非目录、pivot/umount 与 open fd 并发。保留 symlink/readlink 现有测试，新增 no-follow 和循环 oracle。

## 10. 存储、持久化和缓存（P1，ST-K4）

完整设备 flush、四个 syscall 和 guest block/recovery workload 按 [ST-K4 设计](https://github.com/Zero-kernel/Nilix-docs-private/blob/main/design/st-k4-fsync-writeback-design.md) 执行；其当前 source bridge 已指出 numeric fd、current namespace lookup、BIO ownership 和 virtio 无 FLUSH 时“静默成功”都是不能沿用的旧前提。

实现要点：

1. 通过 FileHandle pinned filesystem 解析 fsync 目标，释放 PCB/registry lock 后再执行 I/O；fd 0/1/2 若已 dup 成普通文件，行为按文件对象而非数字决定。
2. 在 `traits.rs` 增加 `Inode::fsync(datasync)` 和 filesystem `sync`；Ext2 先 `ensure_io_healthy`，对 Busy/NoMem 有界重试，失败返回 EIO，不把 fsync 的暂时资源耗尽误记成永久 poison。
3. block trait 暴露真实 durability capability：硬件 flush、明确 write-through guarantee 或 unsupported。virtio 缺少 `VIRTIO_BLK_F_FLUSH` 时必须传播 unsupported；不能用 `Ok(())` 冒充持久化。
4. `GlobalPageCache::clear_dirty` 只在确认数据已写入并完成 barrier 后调用，按 fs_id/inode 过滤；补上 dirty error channel、LRU detach race、error-page refill 和 dirty-unreclaimable pressure。
5. umount 先阻止新 lookup，再同步/失效 cache，等待 pinned FileLifetime；namespace table retirement 不能让仍打开的 inode 失去 owner。
6. `sync_file_range` 先做 signed offset/nbytes、flags allowlist 和 checked_add，再按 write-through 语义返回；不得把它文档化为完整 Linux durability。
7. 扩展 `userspace/stress_runner.c` 的 block profile：slot/generation 写- flush-读-校验、QMP kill/restart、fsck/recovery，输出 `valid_slots/read_bytes/write_bytes/checksum`，host validator 严格拒绝缺失或假成功。

验收包含注入 device Busy/NoMem/IO error、无 FLUSH 设备、并发 fsync、pivot/umount、已 unlink 但仍 open 文件、kill 前后 checksum/generation；所有失败路径保留 dirty/error 证据。

## 11. 网络和 netns 数据面（P1/P2）

### 当前缺口与缺陷

`kernel/net` 已有 virtio-net、IPv4/TCP/UDP、conntrack 和 default-DROP，但 veth、子 namespace RX、路由管理、per-netns firewall 管理、完整 loopback、IPv6 和设备迁移均未闭合。R188 RF188-5 还确认 `kernel/net/src/fragment.rs` 的 64-lane hash 会让不同 live namespace 冲突，且清理没有每 tick 上限；这是需要优先修根因的实际缺陷。设备 transfer 由 `MOVE_NET_DEVICE_ARMED=false` 保持禁用。

### 实现分层

1. **fragment isolation 修复：** 把 key 从 lane hash 扩展为 `(netns_id, fragment_id, src,dst,protocol)`，lane 只做索引提示，最终 map lookup 必须包含 namespace id；每 tick 最多扫描固定条目，超额延后并计数。netns Drop 先停止 ingress，再 drain 该 namespace 的所有 fragment。
2. **veth：** 创建 pair object，端点各自 pinned netns、MAC、ifindex 和 generation；TX 只把 owned packet 转移到对端 bounded RX queue，队列满返回/drop 并计数，不在 softirq 中分配无界内存。
3. **child RX 和 loopback：** netns 初始化时创建 loopback；RX worker 根据 device owner generation 路由到目标 namespace，namespace teardown 先 revoke owner、停止 worker、drain queue 后再释放 buffers。
4. **路由和防火墙管理：** 添加 route table、最长前缀匹配、gateway/interface existence 校验；所有 mutate ioctl/netlink-like calls 通过 user namespace 映射后的 `CAP_NET_ADMIN`，authority 绑定 netns generation，不能使用全局 ADMIN bit。防火墙规则采用 admitted bounded vector，replace 通过 prepare/commit。
5. **设备迁移（最后）：** 只有 namespace-FD authority、generation、child drain reachability、RX/TX quiesce、revocation 和 backstop 都 ready 才翻转 `MOVE_NET_DEVICE_ARMED`；迁移中旧 owner 仍接收完成事件但不接收新包，失败恢复原 owner。
6. **IPv6：** 先完成邻居/路由/ICMPv6/UDPv6/TCPv6 checksum 和 socket address ABI，再做 wire interoperability；不要把 IPv6 parser 的存在当作可用能力。

### 验收

host tap/virtio profile 要记录真实 root MAC、QEMU NIC、namespace id 和 packet capture。验证不同 namespace 同 fragment id 不串包、veth 双向 RX/TX、路由隔离、default-DROP、规则 generation、loopback、自设备 teardown、迁移中 packet drain；保留无设备 negative control。必须有独立 wire observation，host unit tests 不能替代。

## 12. Namespace、cgroup 和容器资源（P1/P2）

当前已有 PID/mount/IPC/net/user namespace 及 CPU/memory/PIDs/I/O/files/ports 控制器，但不是完整 cgroups-v2/OCI。`kernel/kernel_core/cgroup.rs` 的 attach/migrate 已处理层级 pids/memory/fd/port charge，README 仍标记 init membership、delegation、namespace/clone 组合和 container networking 未完成。

### 实现步骤

1. 启动 init 通过 `attach_task` 一次加入 root cgroup；记录 attach generation，重复 attach、fork、exec、delete 都有守恒 oracle。禁止用测试 guest 的事后 fork 修补 membership。
2. cgroupfs 实现统一 `cgroup.controllers`, `subtree_control`, `procs`, `cgroup.procs`, `events`, `stat` 语义；每个 controller 声明 enforcement（hard reject）或 observation（只计数），`memory.current`, `kmem.current`, `io.stat`, `pids.current` 使用实际 owner bytes。
3. delegation 需要 cgroup namespace owner、权限检查和 ancestor boundary；删除前 freeze children、迁移或返回 EBUSY，不能留下 registry 中的 orphan。
4. 完成 `setns/unshare` 的 fd authority、namespace generation 和组合矩阵；CLONE_NEW* 与 CLONE_THREAD、shared fd/mm 的不兼容组合在 prepare 阶段拒绝。
5. OCI/container networking 依赖第 11 节 veth/route/firewall；rootfs、mount propagation、uid/gid map、PID1 和 fd 继承必须以同一 transaction 发布。
6. 补齐 slab/conntrack/page-table kmem 的 cgroup 账本；charge 失败不能遗留物理资源，deferred uncharge 只能在 process context drain。

验收：创建/层级 limit/delegation、fork/clone/exec/migrate/delete、near-limit/OOM、四 CPU 竞争、namespace fd close/ABA、container 启停和网络隔离。检查每个 controller 的 current/max/events 与真实对象数量一致。

## 13. Capability、LSM 和 seccomp（P1）

当前 generation-checked cap、credential-bound publication、LSM/seccomp/pledge hook 已存在；native capability syscall family、每 netns administration 和 TSYNC 未完成。`kernel/kernel_core/syscall.rs` 的 clone 路径已经明确在共享对象未闭合前拒绝相关 flags；`kernel/seccomp/types.rs` 已有 waitid 等 pledge 列表修复，但不能把列表存在当作 TSYNC 完成。

### 设计

1. 先实现 `native_cap_op`：create/derive/revoke/query 的 opcode、rights mask、generation、namespace owner 和 bounded table；每个操作先 reserve entry，再发布 cap，失败撤销 reservation。
2. 实现 `native invoke/spawn/endpoint_call/event_wait` 时，cap lookup 得到 immutable object Arc；调用期间不持 cap-table lock，object close 通过 generation/revocation 唤醒调用者。
3. LSM hook 对 capability、netns admin、mount、chown、mmap、device transfer 使用同一 credential snapshot；hook 返回错误时不得先发布资源。
4. seccomp TSYNC 只有在线程组共享 `SignalTable`/filter vector、线程创建和 exit generation 已完成后启用。安装流程为 prepare filter -> freeze thread group -> append filter -> publish generation -> thaw；任一线程无法同步则全组回滚。
5. pledge/seccomp dispatch 采用 syscall-number 单一表生成 `DISPATCHED_PROMISED`、filter 和 runtime allowlist，避免三处漂移；unsupported syscall 必须返回文档化 errno 或 kill，不能 fail-open。

验收：cap generation exhaustion/reuse、revocation race、跨 namespace admin、LSM deny rollback、TSYNC 与并发 clone/exit、filter OOM、pledge promise parity。高风险 authority 变化需要独立安全审查。

## 14. 内存和 CPU 安全加固（P1/P3）

### 当前边界

`kernel/kernel_core/fork.rs:2516` 的 `create_kpti_user_pml4` 已有双根接口，但 entry island 仍复制高半地址顶部 4 GiB（注释明确要求后续 page-granular trampoline）；README 规定 `FULL_KPTI_ISOLATION_SUPPORTED=false`，双根仍含 kernel mapping。`bootloader/src/main.rs` 的早期 identity/high-half W+X 是 U55-6。编译器 retpoline feature 当前明确不支持，CPU mitigation 的 IBRS/STIBP/SSBD/IBPB 依赖硬件。

### 全 KPTI 设计

1. 将 syscall entry、interrupt entry、per-CPU scratch、GDT/IDT/TSS 和最小 IST stack 放入专用 linker sections，页粒度建立 trampoline island；user CR3 只映射这些页和 user half，删除整个 direct-map/heap/kernel stack alias。
2. 每个 Process 保存 user/kernel CR3 + PCID generation；syscall/interrupt entry 先在 trampoline 中切 kernel CR3，再访问 GS/PCB；exit/NMI 反向切换，任何中途 fault 都走已映射的 panic path。
3. fork/exec/teardown 以 shared user tables 的 refcount 作为 owner，kernel root/private island 单独释放；TLB shootdown 在 root 切换前后记录 generation，防止 stale PCID。
4. 在证明 entry/NMI/switch/return 全路径后才将 `FULL_KPTI_ISOLATION_SUPPORTED` 改为 true；只改报告字段不能视为实现。

### 其他加固

* U55-6：启动页表权限扫描必须在每次 alias 和 CR3 切换完成后运行；失败停止启动。
* retpoline：在 pinned nightly/LLVM 能生成并反汇编验证所有间接 branch 后，增加独立 cargo feature；缺少 compiler support 时继续 compile-time reject。
* secure matrix：对缺失 IOMMU、SMEP/SMAP/UMIP、RNG、KPTI、retpoline 每个字段做负向测试，结果分 supported/active/advisory/unsupported。

验收要求 speculative probe、CR3/CPL3、NMI/IRQ、SMP migration、fork/exec/exit 全路径证据；QEMU 平衡结果不能代替物理 CPU 证据。

## 15. VT-d、DMA 和设备模型（P2/P3，物理前置）

当前 `kernel/iommu` 已有 DMAR/Q35/EDU translated DMA、MSI、invalidation、quarantine 和 IRTE reuse 的 QEMU 证据。未完成的是物理 qualification、MSI-X、多 DRHD/bridge/segment、RMRR、scalable mode、ATS/PASID/ACCESS_PLATFORM 和通用 VM-passthrough lifecycle；`kernel/iommu/lib.rs:636` 仍有“Map RMRR regions in all domains” TODO。

### 分阶段实现

1. **RMRR：** 解析每个 RMRR 的 device scope，验证 base<=limit、segment 和 bridge path；将 region 按设备 domain 映射且禁止普通 user allocation 进入，domain destroy 时先 quiesce device 再撤销映射。
2. **拓扑：** 实现 multi-DRHD lookup（segment、bus/device/function、bridge hierarchy），找不到唯一 unit 时 fail closed；保留 QEMU 单 DRHD 作为独立 profile。
3. **中断：** 在 MSI 基础上加入 MSI-X table/PBA 映射、vector ownership、IRTE generation 和 teardown quiesce；设备写入 stale vector 必须被拒绝并计数。
4. **现代模式：** 只有检测到完整 scalable-mode capability、PASID/ATS page-request fault 处理和安全 invalidation 后才 enable；virtio `ACCESS_PLATFORM` negotiation 失败时禁止把设备加入 translated domain。
5. **VM passthrough：** 先有实际 consumer lifecycle（assign、DMA map/unmap、interrupt bind、reset/revoke）再实现，不为没有 caller 的 `U29-3` 伪造支持。

### 验收

建立硬件矩阵（主板/DMAR/endpoint/firmware/CPU）并逐行记录；注入越界 DMA、RMRR outside range、bridge lookup failure、MSI-X stale vector、reset/reuse、IOTLB timeout。没有物理设备时标记 verification pending，不能由 EDU QEMU 结果升级为 physical supported。

## 16. Livepatch（当前应保持禁用；P3 长期）

`kernel/livepatch` 具有 parser、ECDSA KAT、依赖排序和 syscall wrapper，但 README/`docs/livepatch-support.md` 将生产 livepatch 标为 unsupported，默认路径返回 ENOSYS/权限错误。安全顺序是先保持 fail-closed，再分阶段实现：

1. **密钥与镜像：** 启动时从不可变、版本化 key manifest 载入公钥；拒绝空 key、未知算法、过期/回滚版本和未对齐 section。签名覆盖 target hash、依赖、ABI、权限和版本。
2. **加载：** userspace buffer 先 bounded copy/admission，再只读解析 ELF/relocation；不能在锁内分配或执行 patch code。所有目标符号必须来自 allowlist 并与当前 text hash 匹配。
3. **启用：** stop-machine/RCU quiescence 使所有 CPU 离开旧指令，原子写跳板，执行 instruction-cache/TLB flush；任何 CPU 超时回滚整批修改。
4. **禁用/卸载：** 反向依赖顺序，等待 active callsites/refcount 为零后恢复原 bytes；unload 之前清掉所有 function pointer 和 per-CPU references。
5. 只有完成 SMP/interrupt/NMI/rollback fault injection 和独立安全审查后才翻转支持字段；在此之前所有 mutation syscall 必须稳定返回 ENOSYS，不得有部分启用路径。

验收包括错误签名、key rotation、依赖 cycle、并发 enable/disable/unload、CPU 卡死、page fault、reboot persistence 和 rollback hash。测试失败必须留下 patch disabled 且无 dangling code。

## 17. Audit、观测和 FIPS/运维（P2/P3）

当前 audit ring、hash/HMAC、trace/counters、watchdog/profiler、serial kdump 和 persistence hook 已存在，但没有生产 durable/remote backend、完整 key operation、performance envelope 或 FIPS certification。

### 设计

1. persistence backend 使用 bounded admitted queue 和 sequence cursor；snapshot 先 capture/drop ring lock，再由 process-context worker 写 pinned ext2 文件或远端 transport。写失败保留未确认 cursor，重启后检测 gap/duplicate。
2. 远端发送增加认证 session、重放窗口、backpressure 和 fail-closed policy；内核不能在 IRQ/关键锁中做网络或文件 I/O。
3. key lifecycle 记录 provision/rotate/revoke/use，敏感 buffer 用统一 crypto scrub；FIPS KAT 通过不等于认证，报告只写 measured state。
4. counters 统一单位（字节、ns、sector），饱和时设置 overflow bit；性能数据带 workload、CPU、频率、温度、方差和源码/image identity。

验收：掉电/重启后的日志序列、磁盘满/远端断开、审计 ring overflow、key revoke、时间回拨、counter overflow 和持锁延迟。任何缺少真实测量的 performance script 只能输出 deferred/qualified。

## 18. 用户态 ABI、动态 ELF 和兼容层（P2/P3）

当前 loader 主要支持 ET_EXEC/static musl；`kernel/kernel_core/elf_loader.rs` 对 ET_DYN/PT_INTERP/动态 relocation 没有完整链，README 也不宣称 glibc、pthread、BusyBox、OCI 或 userspace personality。

### 推进方案

1. 先完成静态 ABI residual：`repr(C)` 的 stat/statx/siginfo/poll/rlimit/clone 参数做 compile-time size/offset oracle，失败构建；用户指针和 errno 对齐 musl 行为。
2. ET_DYN loader：验证 `PT_LOAD` 对齐/溢出和权限，选择 ASLR base，应用 RELA（仅 allowlist 类型），建立 `AT_BASE/AT_PHDR/AT_ENTRY`；禁止 text relocation 和 W+X。
3. PT_INTERP/ld.so：把 interpreter 作为 pinned file object 映射，递归加载依赖有深度/总大小上限；任何 relocation、依赖或 page fault 失败都销毁新 MmState，不污染旧 exec。
4. vDSO：把只读、无 kernel data alias 的 time/getcpu page 映射到用户空间，版本化 symbol/ABI；fallback syscall 保留并可观测。
5. pthread/glibc 只有在第 5 节线程共享、第 7 节信号/futex、第 8 节 fd 和第 3 节 shared memory 都通过真实 workload 后再启用。personality/OCI 依赖 native IPC/cap、VFS、netns 和 cgroup 全链。

验收：ET_EXEC 回归、ET_DYN 多次 ASLR、重定位错误/依赖循环、interpreter 缺失、vDSO fallback、glibc/pthread smoke、TLS/clone/signal/robust futex 和 container init。不能通过“能启动一个 hello”宣称动态兼容。

## 19. 测试、fuzzing、stress-v2 和性能资格（横向 P1/P2/P3）

当前 allowlist 固定为 454 个 hosted execution，且 `make test-hosted-subcrates` 已于 2026-09-25 在 x86_64 Linux（`nightly-2025-12-08`）上按此数字观测为绿。该数字的构成是：十五个 `run_suite` 固定值合计 435，加上三个 `run_standalone_rust_suite` 固定值合计 19（`kernel-coverage` 3、`iommu-register-window` 11、`iommu-init-state` 5），也就是门禁自身汇总行实际强制的口径；此前仓库文字写作 433/435/438/439，均与脚本不一致——原因是求和时漏掉了这三个 standalone 套件，另有由 `kernel/tests/test_coverage.rs` 消费的 `NILIX_TEST_TOTAL=74` oracle 断言的 74 个 `RuntimeTest` 注册（这是扫描器口径，不是 `impl RuntimeTest for` 的直接 grep）和真实 KCOV/QEMU smoke，但这些计数不证明完备性。deferred guest tests、六个 stress-v2 profile、性能/热稳定性和硬件证据仍未完成。3.1–3.3 记录的 MM/ST-K2 集成 self-test 是 `kernel/src/integration_test.rs` 中的 **guest** 门检查，不是 hosted 条目，因此该总数不包含它们。脚本 `scripts/gates/performance/perf_regression_test.sh` 对未实现 workload 明确输出 deferred；`userspace/stress_runner.c` 目前把 block/fsync 缺口报告为 `fsync_unsupported`。本地 C userspace 编译和 musl guest marker 合约仍为 verification pending；已记录远程副本上的 hosted/MM/core/build/lint 阶梯已完成。

ST-K3-WAITID 的判定用例是 `kernel/kernel_core/syscall.rs` 中的 `st_k3_waitid_oracle_tests`，与 `mremap_oracle_tests` 一样以 `all(test, feature = "host_harness", not(target_os = "none"))` 门控。它执行生产代码的 `waitid_validate_args` 与 `waitid_encode_siginfo`——正是为此从 `sys_waitid` 抽取出来，使判定驱动真正出货的路径而非副本。已覆盖：选择符解析与 `P_PGID`/`id <= 0`/未知 idtype 的失败关闭拒绝；option 合约（`WEXITED` 必需，`WUNTRACED`/`WCONTINUED` 及未知位返回 EINVAL）；x86_64 `siginfo_t` 字段偏移；以及两条置零不变量——`si_status` 之后的 union 尾部保持为零，且 `WNOHANG` 无事可收时返回完整 128 字节全零，因此不会泄漏陈旧用户字节。收割/摘链、阻塞、EFAULT 恢复与 WNOWAIT-后续收割需要活进程状态和用户地址空间，仍属 guest 门工作，**不在**本判定覆盖范围内。

**验证状态（2026-09-25）：** `cargo fmt --all -- --check` 与 `cargo clippy -p kernel_core --target x86_64-unknown-none` 均通过；余下四条 Clippy 警告是 `virtio`、`mm`、`net` 与 `fork.rs` 中的既有问题，改动文件内无新增。hosted 测试在本地无法编译：`cargo check -p kernel_core --features host_harness --profile test` 在仅支持 x86 的 `drivers` 依赖中失败（`kernel/drivers/vga_buffer.rs` 内联汇编寄存器 `dx`/`al` 在 aarch64 上未知），因此它们是**在 x86_64 Linux 远程主机上执行**的：`cargo test --features host_harness --lib --locked` 报告 `79 passed; 0 failed`，四条 `st_k3_waitid_oracle_tests` 与两条 F2 frameless-fork 谓词用例全部通过。该次运行同时确定了计数固定值：`scripts/tools/hosted_subcrate_tests.sh` 原为 `run_suite kernel-core 73 0`，会触发精确计数门禁失败，故固定值从 73 改为 79。随后完整的 `make test-hosted-subcrates` 阶梯端到端为绿，报告 `454 unit tests`，三个 `run_check` 编译检查全部通过；该运行需要预先放置 `kernel/src/hello.elf` 及其他受版本控制的 guest ELF fixture——缺失时 `kernel-tests` 编译检查会因生成物缺失而失败，并非代码缺陷。`test-hosted-subcrates` 的 Makefile 帮助文本先前写作 "169 tests"、后为 "435"，现已更正为 454，与门禁汇总行一致。

### 统一测试协议

1. 每个工作包维护“行为 oracle + 资源 oracle + 失败 oracle + identity”：行为验证结果，reserved/current/refs 等守恒，错误 errno/日志，源码/image/QEMU 配置 hash。
2. stress-v2 六个 profile（memory、cpu、smp、process、block、combined）都运行记录要求的 900 秒；`BEGIN/READY/HEARTBEAT/PASS` 顺序、valid slots、checksum、resource/recovery 字段缺一项即 incomplete/fail，不能当 PASS。
3. guest workload 通过 pipe、真实 fd、真实网络包、真实 block flush 产生记录；mock/host parser 只验证协议，不代替内核行为。
4. fuzzing：KCOV 控制面继续 host-root-only；QEMU fuzz 每个输入记录 disk/image/revision identity，单次 transient error 只丢弃该迭代，不终止整个 campaign。corpus 加载错误必须日志化，文件名使用单调唯一 id。
5. 高风险 MM/entry/IRQ/IOMMU/ownership 改动完成 build、fmt、Clippy、hosted、boot、musl、SMP/resource、相关 fuzz 和独立 review；缺少环境时标为 verification pending。
6. 性能基线使用真实 userspace workload、固定 CPU/频率/内存/温度窗口、重复样本和方差阈值；热/物理结论必须在 named hardware 上测量，不能从 TCG 或脚本 placeholder 推导。

## 20. 新人执行清单和交接规则

接手任一工作包时按以下顺序操作：

1. 阅读本文件对应小节、roadmap 章节、当前 nextplan 行和已有设计；用 `git status --short` 保留用户修改。
2. 重新运行 `rg` 定位本文列出的函数/常量，核对行号和调用者；若实现前提变化，先在设计的“Deviation and Amendment Log”记录日期、证据和安全影响。
3. 写一份单项设计（高风险按 `docs/review/design` 完整模板），列出状态、owner、锁顺序、allocation/Drop、失败矩阵、测试 oracle 和独立 review 需求。
4. 只实现一个有明确依赖边界的 coherent slice；所有 prepare/commit/rollback 先在 hosted 单元测试覆盖，再运行适用的 guest/stress gate。
5. 任何“unsupported”路径要有稳定 errno、无副作用、文档边界和 negative test；任何“implemented”声明必须有真实行为证据，不能仅凭 dispatch、结构体或 parser 存在。
6. 更新 nextplan 的 item 状态和受影响设计；没有独立审查或所需环境证据时保持 `DRAFT`/`verification pending`，不要伪造 READY/VERIFIED。

### 当前推荐手次

* **第一手：** P0-A/R186-4 收尾：本地 COW/PT/child-publication、map mismatch、constructor-owner oracle，两次 U23 safe-path review，以及远程阶梯（MM 26/26、kernel-core 68/68、hosted 438、build/lint 通过，QEMU 零失败）均已就位；余下是 R186-4 acceptance 记录本身的闭合。
* **第二手：** ST-K2-P1 flags/pipe 最终验证，随后 ST-K4 durability/block workload。
* **第三手：** 补齐 ST-K2-P2 的 guest first-touch/teardown/migration oracle 与 musl marker；关闭 waitid 的 guest 门环节（WNOWAIT-后续收割、PID1 退出、reaper 竞态、copyout 故障、cgroup/fd 回滚）；再处理 clone shared-object、ST-K1 init membership 和 F2/F7/F4/F6/F10 residuals。
* **第四手：** native capability/IPC、VFS metadata/dirfd、netns dataplane；每个工作包独立设计和 wire/lifecycle 测试。
* **第五手：** 六 profile stress、SMP/secure/device qualification；最后再考虑动态 userspace、OCI、full KPTI、物理 VT-d、retpoline 和 livepatch。

## 21. Open limitations

本文没有把有界的 P0-A/ST-K2 实现切片或 R188/KSA 的 scoped PASS 转换成完整 Linux/硬件/发布资格。guest usercopy/teardown/failure 覆盖、musl marker 合约、物理 VT-d、完整 KPTI、compiler retpoline、生产 livepatch、长期性能/热稳定性证据、原始 R188 review lineage、六 profile stress 验收和 clean full-audit streak 仍由当前 nextplan 管理。3.3 的能力原语已实现，但其 buddy/PTE/device 后端尚未接入，文件映射与 `mremap` 仍是未实现的 fail-closed 边界；这些都不能按“已支持”记入 README。

## 22. 参考

* [README capability table](https://github.com/Zero-kernel/Nilix/blob/main/README.md#what-works-and-what-is-missing)
* [Roadmap component gaps](../roadmap.md#5-capabilities-and-missing-contracts)
* [Current next-phase plan](https://github.com/Zero-kernel/Nilix-docs-private/blob/main/nextplan/next-phase-plan-2026-09-13.md)
* [R188 review-fix](https://github.com/Zero-kernel/Nilix-docs-private/blob/main/reviewfix/reviewfix-2026-09-13.md)
* [P0-A design](https://github.com/Zero-kernel/Nilix-docs-private/blob/main/design/p0-a-r186-4-admission-closure-design.md)
* [ST-K2 design](https://github.com/Zero-kernel/Nilix-docs-private/blob/main/design/st-k2-map-shared-design.md)
* [ST-K4 design](https://github.com/Zero-kernel/Nilix-docs-private/blob/main/design/st-k4-fsync-writeback-design.md)
* [ST-K3 进程生命周期记录](https://github.com/Zero-kernel/Nilix-docs-private/blob/main/design/st-k3-process-lifecycle-design.md)——§4 的 waitid、ROOT-INIT、F2 与 Ring-3 生命周期判定
* [VT-d support matrix](../vtd-support-matrix.md)
* [Livepatch support boundary](../livepatch-support.md)

## 23. 设计审查与修订记录

| 日期 | 章节 | 证据/变更 | 安全影响 | 审查结果 |
|---|---|---|---|---|
| 2026-09-13 | 全文 | 基于 README、roadmap、nextplan 和只读源码检查建立当前缺口清单；物理硬件资格明确 deferred。 | 保持不支持和未测能力 fail-closed 且可见。 | DRAFT；实现前的高风险独立设计审查待完成。 |
| 2026-09-13 | 3.1、19、21 | 记录 P0-A/R186-4 实现跟进：fork 先准入再分配快照、构造器错误携带 reservation、先释放 backing 再释放 charge，以及 26 项 MM 回归。 | 保持一次容量大小的 owner transfer，避免 map/构造器失败路径在 backing 仍存活时释放准入；计数损坏后的 Drop fail-stop 和远程 ladder 仍作为明确限制。 | 两次独立 U23 安全路径审查 PASS；最终工作树远程复验待完成。 |
| 2026-09-13 | 3.2、19 | 记录 ST-K2-P1 在 LSM hook 后校验 flags、integration flags oracle，以及 stress report/CPU worker 的有界 pipe 传输。 | 不支持 flags 在 VMA/cgroup/PT 修改前 fail-closed；缺失、短、失败或超长记录保持 incomplete。SMP/combined shared counters 继续留在 Phase 2 边界。 | 本地 MM/kernel_core 测试、kernel check 和格式检查 PASS；C userspace 编译和远程 ladder 仍为 verification pending。 |
| 2026-09-13 | 3.2 | 记录 ST-K2-P2 第一段 eager shared ownership：持久 shared VMA 标记、初始 `PAGE_REF_COUNT` 跟踪、shared-range-aware fork plan/build、可写 child PTE、父侧 COW 排除、refcount-aware rollback/release 以及 child flag 回归断言。 | 防止已实现的 shared leaf 进入 private COW，并让帧释放继续由现有 refcount ledger 决定；first-toucher、side-map、usercopy、migration 和完整 teardown 语义明确保持开放。 | 当前工作树实现记录；Phase 2 完整独立审查以及 guest/remote qualification 待完成。 |
| 2026-09-13 | 3.2、3.3 | 完成共享匿名按需路径：admitted region side-map、按页首触 cgroup charge 与固定存储 #PF 分配器、fork Arc 继承、teardown 后 region 释放，以及有界的 slab/NUMA/swap/THP 原语和集成 oracle。 | 共享页保持可写并由 refcount 持有；charge 与元数据清理在页表拆除前后有序执行。未支持的平台后端保持禁用，直到有专门设计与压力证据。 | 本地 kernel/MM 检查通过；guest SMP/usercopy、C userspace 编译与远程阶梯仍为 verification pending。 |
| 2026-09-13 | 3.2 | 加固共享 #PF 失败路径：在 `PT_LOCK` 之前预留有界 DATA+PT 额度，`map_page` 失败后只拆除已证明为空的中间页表，把可达 PT 帧计入继承 basis，并避免在 PT 锁下访问 cgroup registry。 | 防止首次触映射失败时泄漏中间页表或发布未计费的保留表；在计费不变量失败时采取保守策略。 | 本地编译/格式检查通过。该源码派生补丁的远程同步曾被自动审查判定为未验证上传而拒绝，因此最终工作树远程复验当时仍待完成。 |
| 2026-09-14 | 3.1–3.3 | 在 `40c-devbox-ts` 的 `/tmp/zero-os-codex-final-20260913` 上复验最终源码，包含 shared-fault 中间页表回滚与锁安全的 DATA+PT 预留加固。 | 最终远程源码身份由 fork/capabilities 哈希记录；hosted/MM/core/build/lint 通过，QEMU 套件零失败。 | 已实现范围的最终工作树远程验证 PASS；musl guest marker 与 usercopy/teardown/failure 资格仍待完成。 |
| 2026-09-14 | 3.1–3.3 | 针对未变更工作树的“实现与描述一致性”核对（两个已记录哈希仍匹配）。修正行证据（`sys_mmap` `:13798`、`validate_mmap_flags` `:11835` 调用点 `:13850`、`sys_mremap` `:22016`）；把文件映射/`mremap` 段落改写为明确未实现的 `EOPNOTSUPP`/`ENOSYS` 边界；把 3.2/3.3 的 oracle 声明从 hosted allowlist 移到 guest 集成门（`kernel/src/integration_test.rs:53-57`、`:379-381`）；把 3.3 段落重述为“原语已实现、后端未接入”；修正 §19 的 hosted 计数因果表述与过时的“远程阶梯待完成”。 | 未改动内核代码。避免记录被读成文件映射/`mremap` 与 hosted 3.3 覆盖已存在，并让启用/禁用能力边界对下一位实现者保持可见。 | 未变更工作树上的本地复验：`cargo test --manifest-path kernel/mm/Cargo.toml --features host_harness --lib --locked` → 26 通过 / 0 失败；`... kernel/kernel_core ...` → 67 通过 / 0 失败（第 68 项为 `usercopy::robust_usercopy_tests::ksa002_cmpxchg_mapped_success_and_mismatch`，带 `#[cfg(target_os = "linux")]`）；`cargo check -p kernel --locked` 与 `cargo fmt --all -- --check` → exit 0。本轮未重跑远程阶梯。 |
| 2026-09-25 | 4、19 | 实现 `sys_waitid`（`syscall.rs:23227`，分发 `:4368`）：把 wait4 的 scan/wakeup 路径抽取为共享的 `wait_reap_core`（`:7661`），配合 `WaitTarget` 选择符与各自 ABI 的 `publish` 闭包；并通过在 `assign_pid_chain`（`pid_namespace.rs:1499-1513`）内部注册根 PID namespace init（而非单独 boot hook）关闭 ROOT-INIT。新增 `st_k3_waitid_oracle_tests`，直接驱动抽取出的 `waitid_validate_args`/`waitid_encode_siginfo`。 | 一套收割引擎而非两套会分叉的实现。`P_PGID`、`WUNTRACED`/`WCONTINUED` 与未知 option 位以 `EINVAL` 失败关闭，而不是静默退化为 `P_ALL` 或承诺本内核永不发送的通知。发布先于收割提交，因此 `infop` 出错时恢复 `Ready` 并让 zombie 仍可收割。带身份校验的 init 注册不可能指向被回收复用的 pid；根注册为尽力而为，故 `InitAlreadySet` 绝不使 PID 分配失败，且根被排除在 init 死亡级联之外（已在 `process.rs:9721` 验证）。根 init 缺失或 Zombie 时走 `reparent_logged_leak` 而非 panic。 | `cargo clippy -p kernel_core --target x86_64-unknown-none` 无新增警告。四个新判定用例已在 x86_64 Linux 远程主机上**执行并通过**（`cargo test --features host_harness --lib --locked` 报告 `79 passed; 0 failed`），随后完整 `make test-hosted-subcrates` 阶梯为绿，报告 `454 unit tests` 与 3 个编译检查全通过；本机 macOS/arm64 因 `kernel/drivers/vga_buffer.rs` 仅支持 x86 的内联汇编而无法构建 hosted 测试。guest 门环节（WNOWAIT-后续收割、PID1 退出、reaper 竞态、copyout 故障、cgroup/fd 回滚）与高风险独立审查仍待完成。 |
| 2026-09-25 | 3、4、19 | 第 3–4 章的 guest 门验证轮次，并附带引用审计。按当前源码树重新解析 §3–§4 中所有 `file:line`，修正已漂移者（`P_ALL`/`P_PGID` 常量 `:23179`/`:23183`、`WNOWAIT` 分支 `:7885`、`sys_waitid` `:23270`、`SYS_WAITID` `seccomp/types.rs:194`、F2 判定模块 `fork.rs:470`）；同步了仍把 F2 描述为开放的中文 §4。较早各行引用的是各改动落地时的行号。 | 未改动内核代码。记录实际执行了什么，而非设计上应执行什么，并标出唯一一条没有任何 guest 工作负载驱动的路径——`waitid`。 | `make build` 退出码 0。`make test` 为 QUALIFIED：`passed=35 deferred=39 failed=0`，harness `gate status=3`，Ring-3 init `exit code 0`。`make test-ring3-mm` 退出码 0：`Results: 13 passed, 0 failed`，serial 工件 `/tmp/tmp.VT4mIwQwyr` 中 26 条 `[PASS]` / 1 条被允许的 `[SKIP]` / 0 条 `[FAIL]`，该工件同时载有 §4 生命周期判定（`RF180-40`、`RF180-44`、`D2-ARC`、`RF178-33/35/36`）。hosted 阶梯在 `454 unit tests` 下为绿，四条 `st_k3_waitid_oracle_tests` 全部通过。途中修复两处 harness 故障，均属环境问题：一个游离的上级 `Cargo.toml` 遮蔽了真实 workspace，以及 `make` 继承的非登录 `PATH` 中缺少 `cargo`。没有任何 guest 工作负载发出 syscall 247，因此 `waitid` 仍只有宿主判定覆盖。 |
| 2026-09-25 | 4、3、19 | 关闭 `waitid` 的 guest 覆盖缺口。在 `userspace/src/syscall_test.rs` 中新增 Ring-3 腿 `test_waitid`，经真实分发臂以真实 zombie 驱动 syscall 247，并将其写入 `ring3_mm_oracle.sh` 的腿契约。未改动内核代码。 | 使 `waitid` 在参数契约、`siginfo_t` 回写至 Ring-3 内存、WNOWAIT 存活、EFAULT 先于收割与 `P_ALL` 上，从仅有宿主判定变为经 guest 验证。它是契约腿，因此覆盖不会悄然消失。每条失败路径都会收割每个 fork 出的子进程。 | 在 x86_64 Linux 远端运行 `make test-ring3-mm`：`Results: 14 passed, 0 failed`，27 条 `[PASS]` / 1 条被允许的 `[SKIP]` / 0 条 `[FAIL]`，`[PASS] waitid argument contract, WNOWAIT survival, EFAULT-before-reap, P_ALL`，子进程按预期以 `42` 与 `7` 退出，门禁 `FATAL` 正则零命中。PID1 退出、reaper 竞态与 cgroup/fd 回滚仍未覆盖；高风险独立审查仍待完成。 |
| 2026-09-25 | 4、3、19 | 以第二条 Ring-3 腿 `test_root_init_reparent` 对 ROOT-INIT 改嫁做 guest 验证，并写入 `ring3_mm_oracle.sh` 腿契约；记录三个 §3 运行时用例（`r23_1_cow_tlb_shootdown`、`r174_b4_brk_va_reservation_toctou`、`m0_7_stack_guard_page`）是始终返回 `Deferred` 的占位符。未改动内核代码。 | ROOT-INIT 从源码阅读升级为 guest 验证：被孤立的孙进程由 PID 1 收养且可按 pid 收割，这需要真实的 `children` 链接，而非仅改写 `ppid`。占位符说明避免把 COW TLB shootdown 误读为只受 CPU 数量阻塞——多核运行只会改变其延期信息，什么也不验证。 | 首次运行因测试本身的竞态失败（孙进程在中间进程退出后才采样 `getppid()`；退出码 93），修复在测试中而非内核。在 x86_64 Linux 远端重跑 `make test-ring3-mm`：`RING3-MM OK`，门禁退出码 0，`Results: 15 passed, 0 failed`，28 条 `[PASS]` / 1 条被允许的 `[SKIP]` / 0 条 `[FAIL]`，孙进程退出码 91（`GC_REPARENTED_TO_INIT`），`FATAL` 正则零命中。PID1 退出、reaper 竞态、cgroup/fd 回滚与三个占位符用例体仍待完成。 |
| 2026-09-25 | 3、4、19 | 再增两条 Ring-3 契约腿：`test_fork_pids_refusal`（在 `pids.max = 1` 下 fork）与 `test_user_stack_guard`（惰性区增长加保护页两端）。未改动内核代码。 | 栈保护页行为从未验证升级为 guest 验证，且不依赖 `m0_7_stack_guard_page` 占位符；`pids.max` 下 fork 的拒绝契约被钉住：`EAGAIN`、任务计数不变、无可收割子进程、不消耗 PID。attach 阶段回滚仍只被间接覆盖，并如实记录。 | fork 拒绝运行：`Results: 16 passed, 0 failed`，门禁退出码 0，探测 fork 在 pid 6 之后得到 pid 7。栈保护运行：`RING3-MM OK`，门禁退出码 0，`Results: 17 passed, 0 failed`，30 条 `[PASS]` / 0 条 `[FAIL]`。PID1 退出、reaper 竞态、attach 阶段与 fd 计费回滚，以及 COW shootdown 与 brk TOCTOU 占位符仍待完成。 |
