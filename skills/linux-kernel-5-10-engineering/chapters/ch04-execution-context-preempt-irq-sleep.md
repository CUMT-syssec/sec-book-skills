# 第 4 章：执行上下文、抢占、IRQ 与可睡眠性

## Core Idea

内核 API 能否调用，首先取决于**当前上下文是否允许阻塞、是否允许迁移、是否可能被中断以及数据还被谁并发访问**。同一个函数在进程上下文中安全，在硬中断、软中断、持有自旋锁或禁用抢占时可能立即违规。

5.10.240 用 `preempt_count()` 的不同位域追踪 hardirq、softirq/NMI 和抢占状态；但源码明确警告 `in_atomic()` 不能可靠判断“一般情况下能否睡眠”，驱动不应据此动态选择路径。正确做法是让接口契约静态清晰，并用 `might_sleep()`/调试配置捕获违规。

## Frameworks Introduced

### 上下文五问

- **When**：调用可能分配、等待、访问用户内存或拿锁的 API 前。
- **How**：问：来自 task/IRQ/softirq/NMI 哪层？本地 IRQ 是否关闭？抢占是否关闭？持有什么锁？调用链是否可能睡眠？
- **Why**：上下文限制具有传递性；深层 helper 睡眠仍由外层违规负责。
- **Failure**：只看当前函数没有显式 `schedule()`，忽略 `GFP_KERNEL`、mutex、fault 或等待队列的间接睡眠。

### 上半部采样、下半部处理

- **When**：IRQ 路径需要复杂计算、阻塞 I/O 或等待资源时。
- **How**：中断侧只确认/采集最小状态并排队，在线程化 IRQ、workqueue 或专用 kthread 中完成可睡眠工作。
- **Why**：缩短不可抢占区和 IRQ 延迟。
- **Failure**：延后处理却未建立对象生命周期或顺序保证，形成 use-after-free 或丢事件。

## Key Concepts

- **进程上下文**：有 `current`，通常可睡眠，但持有 spinlock、禁 IRQ/抢占后仍进入原子区。
- **硬中断/NMI**：不能睡眠；NMI 限制更强，只能使用明确 NMI-safe 的设施。
- **软中断/tasklet**：仍不可睡眠；workqueue worker 属于进程上下文，通常可以睡眠。
- **抢占控制**：`preempt_disable()` 防止当前任务被内核抢占/迁移，不等于关闭本地中断，也不是跨 CPU 数据锁。
- **IRQ 控制**：`local_irq_disable()` 只影响本 CPU；若共享数据也被其他 CPU 访问，仍需合适的同步。
- **保存恢复**：未知调用者 IRQ 状态时使用 `local_irq_save(flags)`/restore 或 `spin_lock_irqsave()` 成对恢复，不能无条件 enable。
- **分配标志**：可能睡眠的路径通常用 `GFP_KERNEL`；原子上下文只能使用适用的原子分配，但 `GFP_ATOMIC` 不是架构设计的万能补丁。

## Mental Models

把上下文视为一组逐层收紧的能力：普通 task 能做的最多；禁抢占、持 spinlock、softirq、hardirq、NMI 逐步减少可用能力。函数契约应声明它需要的最低能力，而不是在内部猜测调用者。

另一个模型是“时间预算 + 所有权”：IRQ 路径不仅不能睡，还必须快速交还 CPU；把任务移到 workqueue 后，时间约束放宽，但工作项必须持有对象引用，并有取消/销毁协议。

## Anti-patterns

- `if (in_atomic()) GFP_ATOMIC else GFP_KERNEL`：掩盖接口上下文不明确，且 `in_atomic()` 本身有盲区。
- 在 spinlock 区域调用 `mutex_lock()`、`copy_to_user()` 或可能 fault 的访问。
- 用 `preempt_disable()` 当互斥锁；它只约束当前 CPU 的调度行为。
- 在持锁 IRQ handler 中排队工作后立刻释放工作项依赖的对象。
- 在不知道原状态时 `local_irq_disable(); ...; local_irq_enable();`，破坏调用者已禁用 IRQ 的状态。

## Commands & APIs

关键 API：`preempt_disable()/preempt_enable()`、`local_irq_save()/local_irq_restore()`、`spin_lock_irqsave()`、`might_sleep()`、`schedule_work()`、threaded IRQ。对可能阻塞的 helper，优先在入口放置 `might_sleep()`，让调试配置检查 IRQ/BH、禁抢占和持锁等非法原子上下文；若接口契约明确要求 task context，可用 `WARN_ON_ONCE(!in_task())` 断言，但它不是“可以睡眠”的替代判断，IRQ/BH 状态、抢占状态和持锁情况仍需分别检查。

建议的静态调查：

```bash
rg -n 'spin_lock|local_irq|preempt_disable' path/to/subsystem
rg -n 'mutex_lock|wait_event|GFP_KERNEL|copy_(to|from)_user' path/to/subsystem
rg -n 'schedule_work|queue_work|request_threaded_irq' path/to/subsystem
```

动态验证可启用 `CONFIG_DEBUG_ATOMIC_SLEEP` 等调试项并运行能覆盖该调用链的负载；未实际启动目标内核时只能报告“配置/代码已准备”，不能报告运行验证成功。

## Worked Example：IRQ 中需要更新设备状态并通知用户

错误设计：handler 持自旋锁更新状态后直接 `mutex_lock()`，再发起可能睡眠的总线事务。

改造思路：handler 用 `spin_lock_irqsave()` 保护短状态更新，记录事件并 `schedule_work()`；work 函数在进程上下文中取 mutex、执行可睡眠 I/O、发布通知。移除设备时先禁止新 IRQ，再同步 IRQ、取消 work，最后释放对象。这样不仅解决“能否睡”，也闭合延后工作的生命周期。

审查时仍要问：多个 IRQ 是否合并会丢语义？work 是否需要引用？状态在 IRQ 和 worker 之间由同一锁保护还是用原子/内存序发布？这些问题连接到第 5、6 章。

## Key Takeaways

- 判断 API 合法性要沿完整调用链追踪上下文。
- `in_atomic()` 不是通用“可睡眠探测器”。
- 禁抢占、禁 IRQ、加锁解决的是不同问题。
- 延后工作必须同时设计事件语义和对象生命周期。

## Source Anchors

- `$KERNEL_SRC/include/linux/preempt.h:80` — hardirq/softirq 位域计数。
- `$KERNEL_SRC/include/linux/preempt.h:88` — 各类上下文判定语义说明。
- `$KERNEL_SRC/include/linux/preempt.h:103` — `in_task()` 定义。
- `$KERNEL_SRC/include/linux/preempt.h:142` — `in_atomic()`；其紧邻注释说明局限与驱动禁用建议。
- `$KERNEL_SRC/include/linux/preempt.h:169` — `preempt_disable()`。
- `$KERNEL_SRC/include/linux/preempt.h:183` — `preemptible()` 同时检查计数与 IRQ。
- `$KERNEL_SRC/include/linux/irqflags.h:192` — `local_irq_disable()` 调试封装。
- `$KERNEL_SRC/include/linux/irqflags.h:200` — `local_irq_save(flags)`。
- `$KERNEL_SRC/include/linux/kernel.h:223` — `might_sleep()` 宏；上方注释定义契约和用途。
- `$KERNEL_SRC/kernel/sched/core.c:7258` — `___might_sleep()` 运行检查实现。

## Connects To

第 5 章回答这些上下文中应选择何种锁和内存序；第 6 章回答 IRQ/work/RCU 读者跨时段持有对象时如何安全销毁。
