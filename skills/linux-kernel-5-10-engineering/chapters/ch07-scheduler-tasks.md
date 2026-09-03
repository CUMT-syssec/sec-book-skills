# 第 7 章：调度器、任务状态与唤醒

## Core Idea

调度器不是“挑一个优先级最高的进程”这么简单，而是在每 CPU 运行队列上维护可运行实体，并在任务状态、队列归属和 CPU 归属同时变化时保证一致性。工程上要把 `task_struct` 看成跨子系统的生命周期对象：创建时逐层取得资源，运行时由调度类管理，睡眠时先发布状态再检查条件，退出时按固定顺序释放，最终还需由父进程回收。

`TASK_RUNNING` 同时表示“正在 CPU 上运行”或“在 runqueue 上等待”；睡眠态才表示当前不可运行。唤醒并非简单写回 `TASK_RUNNING`：`try_to_wake_up()` 要用 `p->pi_lock` 串行化并发唤醒，依据 `on_rq`、`on_cpu` 与 CPU 亲和性决定是否迁移和重新入队，并用内存屏障与等待端的 `set_current_state()` 配对。

## Frameworks Introduced

### 1. 任务生命周期框架

- **When**：分析 `fork/clone` 失败、僵尸进程、资源泄漏或退出死锁时。
- **How**：沿 `copy_process()` 的资源获取和错误标签向下读，再沿 `do_exit()` 的释放顺序读；不要只看系统调用入口。
- **Why**：任务不是单一对象，而是 mm、files、fs、signal、namespace、凭据和调度实体的组合。
- **Failure**：创建中途必须按已完成阶段逆序回滚；退出不能发生在中断上下文，递归退出甚至会转为不可中断睡眠等待重启。

### 2. 睡眠—唤醒协议

- **When**：写等待队列、完成量、条件变量式内核代码时。
- **How**：等待端先设置可被目标 wake mask 命中的状态，再检查条件；唤醒端先写条件，再调用 `wake_up*()`。
- **Why**：状态写入与条件读写必须建立可见性顺序，否则可能丢失唤醒。
- **Failure**：遗漏循环复查会把伪唤醒当成功；持有自旋锁或禁抢占区域中调用可能睡眠的路径会破坏上下文约束。

### 3. 每 CPU runqueue

- **When**：追查负载均衡、亲和性或唤醒延迟。
- **How**：区分 `p->pi_lock`（稳定任务调度属性）与 `rq->lock`（保护运行队列）；迁移时关注锁顺序和 CPU 重选。
- **Why**：每 CPU 队列减少全局争用，但引入远端唤醒、IPI 与迁移竞态。
- **Failure**：脱离锁读取 `on_rq/on_cpu` 并据此修改任务，会制造重复入队或永久丢失任务。

## Key Concepts

- `task_struct`：身份、状态、调度实体、地址空间、文件表和 namespace 的汇合点。
- 调度类：stop、deadline、RT、fair、idle 以统一接口接入核心调度器；普通任务主要由 CFS 管理。
- 自愿调度与抢占：`schedule()` 调用 `__schedule(false)`；是否再次循环由 `need_resched()` 决定。
- 特殊状态：`set_special_state()` 面向不允许普通唤醒竞争的状态；普通等待优先使用 `set_current_state()`。
- 退出与回收：`do_exit()` 释放运行资源并进入退出态，但 PID/退出状态的最终回收还依赖 wait 路径。

## Mental Models

把任务想成一张“状态 + 所在位置”二元表：状态回答能否运行，`on_rq/on_cpu` 回答实体在哪里。任何唤醒实现都必须把两个维度一起闭合。再把一次阻塞想成两方握手：等待者发布“我可被唤醒”，唤醒者发布“条件已成立”；屏障确保双方不会各自只看到旧值。

## Anti-patterns

- 用 `current->state = ...` 手写等待协议，绕过带屏障的状态设置宏。
- 以为 `TASK_RUNNING` 等于“此刻占用 CPU”，据此做无锁资源判断。
- 在持有 spinlock、关闭中断或 atomic context 中直接调用可能阻塞的分配和调度函数。
- 创建任务时只返回错误而不镜像撤销已复制的 namespaces、files 或 mm。
- 唤醒后立即假设目标任务已经执行；唤醒只让它成为可运行候选。

## Commands & APIs

- `ps -eo pid,ppid,stat,psr,pri,ni,comm`：观察状态、CPU 与优先级。
- `cat /proc/<pid>/sched`、`cat /proc/sched_debug`：查看实体和 runqueue 统计（后者依配置而定）。
- `chrt -p <pid>`、`taskset -pc <cpulist> <pid>`：读取/设置策略与亲和性。
- `set_current_state()` / `__set_current_state()`：前者包含排序语义，后者只在已有锁等保证时使用。
- `schedule()`、`wake_up_process()`、`wake_up_interruptible()`：阻塞与唤醒常用入口。

## Worked Example

一个可中断等待必须循环检查条件，并在离开前恢复运行态：

```c
DEFINE_WAIT(wait);

for (;;) {
    prepare_to_wait(&wq, &wait, TASK_INTERRUPTIBLE);
    if (READ_ONCE(ready))
        break;
    if (signal_pending(current)) {
        ret = -ERESTARTSYS;
        break;
    }
    schedule();
}
finish_wait(&wq, &wait);
```

生产者应先更新 `ready`，再 `wake_up_interruptible(&wq)`。`prepare_to_wait()` 负责入队和状态发布，`finish_wait()` 负责无论成功还是信号中断都撤销等待者状态；这就是可审计的生命周期闭环。

## Key Takeaways

1. 调度正确性来自状态、runqueue、CPU 归属和内存序的联合协议。
2. 睡眠必须允许当前上下文调度；唤醒只改变可运行性，不保证立即执行。
3. `copy_process()` 与 `do_exit()` 应作为资源获取/释放的镜像来读。
4. 等待代码要有循环、信号处理和统一清理出口。

## Source Anchors

- `$KERNEL_SRC/include/linux/sched.h:643` — `struct task_struct`
- `$KERNEL_SRC/kernel/sched/core.c:2846` — `try_to_wake_up()` 的锁与屏障契约
- `$KERNEL_SRC/kernel/sched/core.c:4618` — `schedule()`
- `$KERNEL_SRC/kernel/fork.c:1950` — `copy_process()`
- `$KERNEL_SRC/kernel/exit.c:762` — `do_exit()`

## Connects To

- 第 5 章：并发原语与内存序决定唤醒协议是否成立。
- 第 8 章：缺页和直接回收可能让任务睡眠。
- 第 10 章：I/O completion 常是任务重新变为可运行的触发点。
