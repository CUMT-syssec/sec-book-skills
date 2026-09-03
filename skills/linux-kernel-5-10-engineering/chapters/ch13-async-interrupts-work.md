# 第 13 章 异步执行：IRQ、softirq、timer、workqueue 与等待

## Core Idea

内核异步代码的核心不是“把函数晚点执行”，而是选择正确的**执行上下文、并发模型和生命周期屏障**。硬 IRQ 顶半部必须短且不能睡眠；softirq/timer 仍处于原子上下文；workqueue 把工作交给进程上下文；wait queue 和 completion 则让进程等待状态或一次事件。工程上要同时回答三件事：回调能否睡眠、谁拥有其数据、销毁前用什么同步原语证明回调已不再访问数据。

典型链路是：设备中断只确认来源并采样最少状态，随后唤醒 threaded IRQ 或排队 work；timer 只负责触发状态机，不执行可能阻塞的 I/O；关闭路径按“禁止新事件 → 同步取消异步源 → 释放对象”的顺序收口。

## Frameworks Introduced

### 1. Threaded IRQ

- **When**：设备需要快速应答中断，但后续处理可能睡眠或耗时。
- **How**：`request_threaded_irq()` 注册 primary handler 与 `thread_fn`；顶半部返回 `IRQ_WAKE_THREAD`，线程函数在可调度上下文运行。
- **Why**：把中断延迟和业务处理解耦，又保持每条 IRQ 的屏蔽/唤醒语义。
- **Failure**：共享 IRQ 缺少唯一 `dev_id` 会使 `free_irq()` 无法精确解绑；线程化处理仍需处理设备拔除与并发中断。

### 2. Softirq、timer 与 workqueue

- **When**：softirq 适合系统级高频延后工作；timer 适合基于 `jiffies` 的超时；workqueue 适合可睡眠的普通异步任务。
- **How**：softirq 由 `open_softirq()` 建立动作、`raise_softirq()` 置 pending；`timer_setup()` 后用 `mod_timer()` 安排；`INIT_WORK()` 后用 `queue_work()` 投递。
- **Why**：三者分别覆盖原子上下文、时间触发和进程上下文。
- **Failure**：timer 回调中睡眠、重复初始化 pending work、或仅 `timer_delete()`/`cancel_work()` 后立刻释放承载对象，都会留下 use-after-free 窗口。

### 3. Wait queue 与 completion

- **When**：等待“条件变真”用 wait queue；等待“一次工作完成”用 completion。
- **How**：`wait_event*()` 总是在唤醒后重新检查条件；`complete()` 增加 done token 并唤醒一个等待者，`complete_all()` 唤醒全部。
- **Why**：条件与事件语义分开，避免自行拼装 task state 和链表操作。
- **Failure**：无锁检查条件会丢失状态变化；栈上 completion 若等待者可能晚于函数返回，会变成悬空对象；`reinit_completion()` 与正在发生的 `complete()` 并发会吞事件。

## Key Concepts

- **上下文预算**：hardirq/softirq/timer 不得调用可能睡眠的 API；workqueue worker 可以睡眠，但不能无限占住共享 worker。
- **投递不是执行**：`queue_work()` 返回 `false` 只表示该 work 已处于 pending/已在队列，不能据此推断它正在执行，也不代表失败需要重投。
- **同步取消**：`cancel_work_sync()` 等到目标 work 不再 pending/running；新代码用 `timer_delete_sync()` 停止 timer 并等待另一 CPU 上可能正在执行的 callback。`del_timer_sync()` 在该树中只是兼容内联别名。
- **共享数据发布**：IRQ 与进程上下文之间仍需 spinlock、原子变量或明确的 acquire/release 关系，排队动作本身不是任意数据竞争的万能屏障。
- **等待可中断性**：用户可见路径通常选 `wait_event_interruptible*()`，必须传播 `-ERESTARTSYS`；内部 teardown 才常用不可中断等待。

## Mental Models

把异步对象看成两个正交状态机：一条是业务状态（idle/running/done），另一条是调度状态（not queued/pending/executing）。锁保护业务状态，取消 API 收敛调度状态。析构必须同时把两条状态机带回静止点。

另一个实用模型是“门、排水、拆除”：先关门阻止新 IRQ/timer/work，再用同步取消排干已进入的回调，最后拆除内存和设备资源。只做后两步而不关门，生产者可能在排水期间再次投递。

## Anti-patterns

- 在 hardirq、softirq 或 timer callback 中调用 `mutex_lock()`、`msleep()` 或可能 fault 的用户访问。
- 用 `schedule_work()` 规避所有并发设计，却没有定义退出时的 `cancel_work_sync()`。
- 将 `flush_workqueue()` 当成取消；它等待先前排队工作，却不阻止生产者继续排队。
- `wait_event(wq, flag)` 中的 `flag` 没有受锁或 `READ_ONCE()`/匹配写保护。
- completion 完成后盲目 `reinit_completion()`，未证明没有并发 waiter/completer。

## Commands & APIs

```text
request_threaded_irq / free_irq / synchronize_irq
open_softirq / raise_softirq
timer_setup / mod_timer / timer_delete_sync
INIT_WORK / queue_work / cancel_work_sync / flush_work
wait_event_interruptible_timeout / wake_up_all
init_completion / complete / wait_for_completion_timeout
```

运行时观察可先看 `/proc/interrupts` 与 `/proc/softirqs`，再用 tracefs 事件确认生产与消费是否配对；这些计数只能说明活动发生，不能证明回调生命周期安全。

## Worked Example

下面展示一个对象的最小“timer 触发 work、退出时同步收口”骨架：

```c
static void demo_workfn(struct work_struct *w)
{
	struct demo *d = container_of(w, struct demo, work);
	mutex_lock(&d->lock);
	/* 可睡眠的设备处理 */
	mutex_unlock(&d->lock);
}

static void demo_timer(struct timer_list *t)
{
	struct demo *d = from_timer(d, t, timer);
	if (!READ_ONCE(d->stopping))
		queue_work(system_wq, &d->work);
}
static void demo_stop(struct demo *d)
{
	WRITE_ONCE(d->stopping, true);
	timer_delete_sync(&d->timer);
	cancel_work_sync(&d->work);
}
```

关键不是代码短，而是 stop 位于所有资源释放之前，且 timer/work 都观察同一个关闭门。若 IRQ 也能投递 work，还必须先禁用并同步 IRQ。

## Key Takeaways

1. 先按“能否睡眠”选择执行机制，再设计并发与延迟。
2. 异步对象必须有明确的关闭门和同步取消步骤。
3. wait queue 等条件，completion 等事件；两者都不是数据锁。
4. `*_sync()` 是生命周期证明的一部分，不能被普通删除/取消替代；5.10.240 新代码应写 `timer_delete_sync()`。

## Source Anchors

- `$KERNEL_SRC/kernel/irq/manage.c:2000` — `request_threaded_irq()` 契约；`$KERNEL_SRC/kernel/irq/manage.c:1921` — `free_irq()`。
- `$KERNEL_SRC/kernel/softirq.c:456` — `raise_softirq_irqoff()`；`$KERNEL_SRC/kernel/softirq.c:489` — `open_softirq()`。
- `$KERNEL_SRC/kernel/time/timer.c:1093` — `mod_timer()` 并发说明；`$KERNEL_SRC/kernel/time/timer.c:1345` — `timer_delete_sync()`；`$KERNEL_SRC/include/linux/timer.h:189` — `del_timer_sync()` 兼容别名。
- `$KERNEL_SRC/kernel/workqueue.c:1515` — `queue_work_on()`；`$KERNEL_SRC/kernel/workqueue.c:3161` — `cancel_work_sync()`。
- `$KERNEL_SRC/kernel/sched/wait.c:287` — `prepare_to_wait_event()`；`$KERNEL_SRC/kernel/sched/wait.c:365` — `finish_wait()`。
- `$KERNEL_SRC/kernel/sched/completion.c:28` — `complete()`；`$KERNEL_SRC/kernel/sched/completion.c:127` — `wait_for_completion()`。

## Connects To

- 第 5 章的锁与内存序：异步边界上的共享状态仍需同步。
- 第 15 章的模块卸载：退出函数必须排干 IRQ、timer 和 work。
- 第 17 章的 tracepoint/ftrace：用事件时间线定位延迟与重复投递。
- 第 18 章的 fault injection：验证超时、取消和错误回退路径。
