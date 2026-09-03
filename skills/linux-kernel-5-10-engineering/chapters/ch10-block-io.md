# 第 10 章：块层、bio、blk-mq 与完成路径

## Core Idea

块层把“哪些内存页要在什么扇区读写”与“硬件如何排队执行”分开。`bio` 描述面向块设备的一段逻辑 I/O，可由多个 `bio_vec` 指向页面片段；blk-mq 把一个或多个 bio 组织为 `request`，映射到软件队列和硬件提交队列，再调用驱动的 `queue_rq()`。硬件结束后，完成路径更新状态、结束 request，并最终调用 bio 的 `bi_end_io`。

核心契约是异步所有权：`submit_bio()` 后，在 `bi_end_io` 调用前提交者不得再碰 bio。函数返回不等于 I/O 完成。错误同样沿异步状态传播，因此资源释放、上层唤醒和错误报告通常集中在 completion 回调，而不是 submit 的返回路径。

## Frameworks Introduced

### 1. bio 生命周期框架

- **When**：文件系统、device-mapper 或块驱动构造 I/O 时。
- **How**：分配 bio，设置目标设备/扇区/操作，添加页面，设置 `bi_end_io` 和私有上下文，再提交；完成回调读取 `bi_status` 并释放所有权。
- **Why**：页面集合与设备请求解耦，便于切分、合并和堆叠。
- **Failure**：构造失败要 `bio_put()`；提交后同步释放或复用会 UAF；回调漏释放会泄漏。

### 2. blk-mq 队列框架

- **When**：分析多队列设备吞吐、CPU 映射或 request 卡住。
- **How**：bio 经 `blk_mq_submit_bio()` 得到 request，选择软件 context/hardware context，调度或直接 dispatch，驱动在 `queue_rq()` 接受、资源不足或报错。
- **Why**：每 CPU 软件队列降低锁竞争，硬件队列贴合现代控制器并行度。
- **Failure**：驱动返回资源不足时必须让块层稍后重试；已调用 `blk_mq_start_request()` 的 request 必须最终完成或超时处理。

### 3. 完成与上下文框架

- **When**：在中断中报告硬件完成。
- **How**：驱动保存的 request 指针交给 `blk_mq_complete_request()`；核心可远端投递或直接调用 `mq_ops->complete`。
- **Why**：提交 CPU、硬件中断 CPU 与完成执行 CPU 可能不同。
- **Failure**：完成两次、完成未启动 request 或回调中执行可睡眠操作，都可能破坏计数与上下文约束。

### 4. 前向进展框架

- **When**：堆叠块设备在 submit 路径中再分配/提交 bio。
- **How**：使用 bioset/mempool，并理解 `current->bio_list` 把递归提交转为迭代；必要时 rescuer workqueue 排空待提交 bio。
- **Why**：普通内存回收本身可能依赖块 I/O，分配等待容易闭环死锁。
- **Failure**：在同一受限 mempool 中递归分配多个 bio 而无救援机制，会耗尽储备并互相等待。

## Key Concepts

- `bio`：块范围、页面向量、操作码、状态与完成回调。
- `request`：面向驱动调度/派发的工作单元，可包含多个 bio。
- `request_queue`：限制、调度器和 blk-mq 配置的中心对象。
- tag：硬件在途 request 的有限标识资源。
- plugging：暂存并批量提交，改善合并；任务睡眠前调度器会冲刷 plug 以避免死锁。
- `BLK_STS_*`：块层状态，完成时写入 bio/request，不能随意混用负 errno。

## Mental Models

把路径想成两次聚合和一次反向通知：页面片段聚成 bio，bio 聚成 request，完成再从 request 反向拆到每个 bio。提交路径转移所有权，完成路径归还所有权。任何调试都应同时画出“数据对象链”和“当前负责释放它的人”。

## Anti-patterns

- `submit_bio()` 返回后立即读取 `bio->bi_status` 或 `bio_put()`。
- 在 completion 回调中睡眠，未先转交 workqueue/threaded context。
- 驱动接受 request 后遗漏 `blk_mq_start_request()` 或最终完成。
- 把 `bio_add_page()` 的返回值忽略；在 5.10.240 中它成功时返回完整 `len`，无法加入时返回 0，不会部分加入，因此必须检查是否等于请求的 `len`。
- 为避免分配失败无限叠加 `GFP_ATOMIC`，却不设计 mempool 和回压。
- 将超时完成与正常中断完成并发执行，缺少一次性所有权仲裁。

## Commands & APIs

- `lsblk -o NAME,MAJ:MIN,SIZE,ROTA,SCHED,MOUNTPOINTS`：查看设备与调度器。
- `cat /sys/block/<dev>/queue/nr_requests`、`scheduler`：查看队列配置。
- `iostat -x 1`：观察队列、延迟与利用率（需要 sysstat）。
- `bio_alloc()` / `bio_put()`、`bio_add_page()`、`submit_bio()`：bio 生命周期。
- `blk_mq_start_request()`、`blk_mq_complete_request()`：驱动 request 生命周期关键点。
- `blk_mq_stop_hw_queue()` / `blk_mq_start_hw_queue()`：仅按驱动状态机成对控制派发。

## Worked Example

一个最小 completion 回调应把状态转换和释放集中到一处：

```c
struct io_ctx {
    struct completion done;
    blk_status_t status;
};

static void demo_end_io(struct bio *bio)
{
    struct io_ctx *ctx = bio->bi_private;

    ctx->status = bio->bi_status;
    bio_put(bio);
    complete(&ctx->done);
}
```

提交方在允许睡眠的进程上下文中可等待 completion；若不能睡眠，则必须继续异步传播。真实代码还要保证 `io_ctx` 在回调完成前存活，并处理 bio 构造尚未提交时的同步回滚。

## Key Takeaways

1. bio 描述数据，request 描述可派发工作，blk-mq 连接 CPU 与硬件队列。
2. submit 是所有权转移点，completion 才是资源回收点。
3. 驱动必须闭合 start、timeout/abort 与 complete 状态机。
4. mempool、递归转迭代和回压是内存压力下保持前向进展的关键。

## Source Anchors

- `$KERNEL_SRC/include/linux/blk_types.h:203` — `struct bio`
- `$KERNEL_SRC/include/linux/blkdev.h:135` — `struct request`
- `$KERNEL_SRC/include/linux/blkdev.h:405` — `struct request_queue`
- `$KERNEL_SRC/block/bio.c:437` — `bio_alloc_bioset()` 及 rescuer 语义
- `$KERNEL_SRC/block/blk-core.c:1036` — `submit_bio_noacct()` 递归转迭代
- `$KERNEL_SRC/block/blk-core.c:1071` — `submit_bio()` 异步所有权契约
- `$KERNEL_SRC/block/blk-mq.c:699` — `blk_mq_complete_request()`
- `$KERNEL_SRC/block/blk-mq.c:2171` — `blk_mq_submit_bio()`

## Connects To

- 第 7 章：完成量和 I/O wait 把完成事件连接到任务唤醒。
- 第 8 章：页是 bio 的主要数据载体，回收可能依赖回写完成。
- 第 9 章：文件系统把 page cache 操作转换成块 I/O。
