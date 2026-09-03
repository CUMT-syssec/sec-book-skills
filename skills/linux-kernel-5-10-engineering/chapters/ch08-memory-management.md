# 第 8 章：内存管理、页分配、VMA 与缺页

## Core Idea

Linux 内存管理要同时回答两个问题：物理页从哪里来，以及一个进程的虚拟地址在何时、以什么权限映射到这些页。伙伴系统按 zone、order 和迁移类型提供物理页；`mm_struct` 描述进程地址空间，`vm_area_struct` 描述连续且属性一致的虚拟区间；页表把地址解析落实到具体页。缺页不是天然错误，而是把延迟分配、文件映射和写时复制兑现为页表项的正常控制流。

分配接口的 `gfp_t` 是上下文契约，不只是“内存类型”。是否允许直接回收、文件系统递归或 I/O，决定调用是否可能睡眠、是否可能重入持锁子系统。错误处理也不是只判 `NULL`：缺页返回 `vm_fault_t` 位图，调用者据此区分重试、OOM、SIGBUS 与 SIGSEGV。

## Frameworks Introduced

### 1. GFP 上下文框架

- **When**：驱动、中断、锁内路径需要分配页或 slab 对象时。
- **How**：先判断上下文能否睡眠、是否处于 reclaim/文件系统路径，再选择 `GFP_KERNEL`、`GFP_NOFS` 或 `GFP_ATOMIC` 等组合。
- **Why**：`__GFP_DIRECT_RECLAIM` 会允许分配者进入直接回收；源码明确以 `might_sleep_if()` 检查这一点。
- **Failure**：在 atomic context 用可回收分配可能触发 sleeping-while-atomic；滥用 `GFP_ATOMIC` 则耗尽紧急储备并掩盖设计问题。

### 2. VMA 变换框架

- **When**：实现 `mmap`、`munmap`、权限变更或追查地址空间碎片。
- **How**：在 `mmap_lock` 保护下验证区间、权限和计数，再插入、合并或拆分 VMA；失败时撤销已建立的映射和文件引用。
- **Why**：VMA 是策略区间，不等于已存在的物理页；页表可以仍为空。
- **Failure**：区间溢出、`map_count` 超限和文件偏移溢出会在 `do_mmap()` 前段拒绝；`MAP_FIXED_NOREPLACE` 遇到重叠 VMA 返回 `-EEXIST`，普通 `MAP_FIXED` 则允许替换地址区间内已有映射，调用者必须接受旧映射被撤销的破坏性语义。

### 3. 缺页处理框架

- **When**：CPU 访问无有效 PTE 或违反权限时。
- **How**：体系结构入口定位 VMA并检查权限，在持有 `mmap_lock` 的条件下进入 `handle_mm_fault()`，再分派匿名页、文件页、COW 或大页路径。
- **Why**：把昂贵工作推迟到首次访问，减少不必要分配并支持共享。
- **Failure**：某些文件缺页路径可能释放 `mmap_lock`；调用者必须按返回标志而不是想当然地解锁或重试。

## Key Concepts

- page/zone/order：order 为 0 是单页，order 为 n 表示 `2^n` 个连续页。
- zonelist 与 watermark：快速路径按节点和 zone 水位寻找可用块，失败才进入慢路径回收/压缩/OOM。
- VMA：记录 `[vm_start, vm_end)`、权限、flags、文件与 `vm_ops`。
- 匿名页、page cache 与 COW：来源不同，但都通过 fault 路径建立或更新 PTE。
- 引用与映射计数：页被分配不代表只有一个所有者；释放前必须满足引用协议。

## Mental Models

把虚拟内存看成“三层承诺”：VMA 承诺某段地址如何使用，页表记录当前已兑现的映射，物理分配器提供兑现资源。缺页是兑现动作。GFP 则是一张“执行权限票”：它规定当前分配能否睡眠、能否回收、能触碰哪些子系统。

## Anti-patterns

- 看到缺页计数升高就判断内存异常；minor fault 往往是正常的按需映射。
- 在 spinlock 内调用 `alloc_pages(GFP_KERNEL, ...)`。
- 用高阶连续分配承载可分散的数据，忽略碎片导致的失败概率。
- 修改 VMA 树/链表却不持有正确模式的 `mmap_lock`。
- 忽略 `VM_FAULT_RETRY` 和“锁可能已释放”的返回契约。
- `do_mmap()` 失败后仍使用返回值为地址；其错误以编码后的负值返回。

## Commands & APIs

- `cat /proc/<pid>/maps`、`smaps_rollup`：查看 VMA 与聚合驻留信息。
- `cat /proc/buddyinfo`、`cat /proc/zoneinfo`：查看伙伴空闲块和 zone 水位。
- `vmstat 1`：观察 fault、swap、回收和运行队列变化。
- `alloc_pages(gfp, order)` / `__free_pages(page, order)`：成对管理页。
- `mmap_read_lock(mm)` / `mmap_write_lock(mm)`：地址空间读写序列化。
- `handle_mm_fault()`：内部缺页核心入口，返回 `vm_fault_t` 标志。

## Worked Example

需要可睡眠的单页缓冲区时，生命周期应显式配对：

```c
struct page *page;
void *buf;

page = alloc_page(GFP_KERNEL | __GFP_ZERO);
if (!page)
    return -ENOMEM;

buf = page_address(page);
ret = consume_page(buf);

__free_page(page);
return ret;
```

这段代码只适用于允许睡眠且 `page_address()` 可直接映射该页的平台/内存区域。若 `consume_page()` 建立了异步所有权，便不能在返回前释放；必须把 `put_page()` 或完成回调设计进所有权协议。

## Key Takeaways

1. 物理页、VMA 与页表是三种不同层次的对象。
2. GFP flags 描述调用上下文和回收边界，错误选择可能形成死锁。
3. 缺页是返回标志驱动的状态机，锁所有权可能随返回路径变化。
4. 高阶分配、映射建立与异步共享都必须设计对称回滚。

## Source Anchors

- `$KERNEL_SRC/include/linux/mm_types.h:310` — `struct vm_area_struct`
- `$KERNEL_SRC/include/linux/mm_types.h:393` — `struct mm_struct`
- `$KERNEL_SRC/include/linux/gfp.h:141` — GFP reclaim 语义说明
- `$KERNEL_SRC/mm/page_alloc.c:4973` — `__alloc_pages_nodemask()`，伙伴分配核心
- `$KERNEL_SRC/mm/mmap.c:1411` — `do_mmap()`
- `$KERNEL_SRC/mm/mmap.c:2927` — `do_munmap()`
- `$KERNEL_SRC/mm/memory.c:4702` — `handle_mm_fault()` 的锁契约

## Connects To

- 第 7 章：直接回收和文件缺页都可能导致任务睡眠。
- 第 9 章：文件映射通过 address_space/page cache 连接 VFS。
- 第 10 章：脏页回写最终形成 block I/O。
