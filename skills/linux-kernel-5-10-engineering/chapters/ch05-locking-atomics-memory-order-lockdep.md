# 第 5 章：锁、原子操作、内存序与 lockdep

## Core Idea

并发正确性包含三个独立问题：**互斥**决定谁能同时修改，**生命周期**决定对象是否仍存在，**内存序**决定其他 CPU 以什么顺序观察读写。一个 `atomic_t` 只能保证特定原子操作不可撕裂，通常不能自动维护多字段不变量，也不能自动解决对象释放。

锁的选择从上下文和临界区行为出发：可能睡眠且仅进程上下文竞争时常用 mutex；中断/原子路径需要短临界区时用 spinlock 变体；读多写少和可容忍旧视图时才考虑 RCU。lockdep 用锁类与获取顺序检查潜在死锁，但它不能替你证明无锁算法的所有内存序。

## Frameworks Introduced

### 不变量驱动同步

- **When**：新增共享字段、锁或 atomic 前。
- **How**：先写出不变量和参与字段，再列读写者与上下文，最后选择覆盖该不变量的同步原语。
- **Why**：锁保护的是关系，不只是变量。
- **Failure**：字段 A、B 分别原子更新，但读者观察到不可能组合。

### happens-before 设计

- **When**：发布对象、环形队列、状态机或无锁 fast path。
- **How**：明确发布写、获取读及其要排序的数据；优先使用锁或 acquire/release 封装。
- **Why**：编译器和 CPU 都可重排普通访问。
- **Failure**：用 `volatile`、注释或单独 `READ_ONCE()` 误当完整顺序保证。

## Key Concepts

- **mutex**：等待者可睡眠，不能在 IRQ/原子上下文使用；适合较长、可能阻塞的临界区。
- **spinlock**：等待时忙等，临界区应短且不可睡眠；若同一锁也在本地 IRQ 使用，进程侧通常需 irqsave 变体避免自死锁。
- **atomic_t**：适合独立计数/位状态的原子读改写；引用计数优先 `refcount_t`，因为它带溢出/下溢防护语义。
- **READ_ONCE/WRITE_ONCE**：约束编译器合并、拆分和重复访问；它们不普遍提供跨 CPU 的完整排序。
- **acquire/release**：只有 acquire 观察到同一同步变量上 release 发布的值，或该 release sequence 中的值时，才建立配对的可见性：release 之前的写对 acquire 之后的读可见。仅仅在两端分别调用 acquire/release、却没有这条读值关系，并不会自动建立 happens-before；需要双向全屏障时才考虑 `smp_mb()`。
- **锁即排序**：同一把锁的 unlock/lock 已携带相应排序，不应无依据叠加屏障。
- **lockdep assertion**：`lockdep_assert_held()` 将“调用者必须持锁”变为可检查契约。

## Mental Models

把共享状态画成表：行是访问者，列是字段，每格写 R/W、上下文与所持锁。若一个不变量跨列，就需要同一同步域，或清晰的版本/发布协议。

把内存序画成箭头而非时间线：生产者初始化数据后，在同步变量上 release 发布值；消费者从同一同步变量 acquire 读到该值或其 release sequence 中的值，才取得读取数据的可见性。没有这条读值配对箭头的“先写后读”只是源代码顺序，不是跨 CPU 保证。

## Anti-patterns

- “变量是 atomic，所以结构体线程安全”。
- 持 spinlock 调用可能睡眠函数，或持 mutex 进入硬中断共享路径。
- 为修竞态到处加入 `smp_mb()`，既难审查也可能仍放错位置。
- 用 `volatile` 实现设备寄存器之外的并发协议。
- 不统一锁顺序，靠“实际很少同时发生”避免 ABBA。
- 关闭 lockdep 警告或改锁类来隐藏真实顺序问题。

## Commands & APIs

API 选择：`mutex_lock/unlock`、`spin_lock_irqsave/unlock_irqrestore`、`atomic_cmpxchg`、`READ_ONCE/WRITE_ONCE`、`smp_store_release/smp_load_acquire`、`lockdep_assert_held`。

建议检查：

```bash
rg -n 'spin_lock|mutex_lock|lockdep_assert' path/to/subsystem
rg -n 'READ_ONCE|WRITE_ONCE|smp_(load_acquire|store_release|mb)' path/to/subsystem
rg -n 'PROVE_LOCKING|LOCKDEP' "$KERNEL_SRC/lib/Kconfig.debug"
```

动态 lockdep 结论需要启用相关配置并运行覆盖锁顺序的工作负载。一次无告警运行只是覆盖范围内的证据，不是不存在死锁的数学证明。

## Worked Example：发布初始化完成的对象

生产者构造对象后设置 ready，消费者看到 ready 后读取 payload。若不用锁，可以形成最小 release/acquire 协议：

```c
struct item {
	int payload;
	int ready;
};

static void publish(struct item *p, int value)
{
	p->payload = value;
	smp_store_release(&p->ready, 1);
}

static bool consume(struct item *p, int *value)
{
	if (!smp_load_acquire(&p->ready))
		return false;
	*value = p->payload;
	return true;
}
```

这只解决 payload 的发布顺序；对象本身必须在消费者结束前存活，多个生产者也需要额外仲裁。若状态更复杂，mutex/spinlock 往往比手写无锁协议更正确、更便宜于维护。

## Key Takeaways

- 先定义共享不变量，再选锁或原子操作。
- 原子性、排序、生命周期是三个问题。
- 优先使用锁和 acquire/release 这类语义化原语，屏障应有明确配对。
- lockdep 能检查锁图，断言能把隐含持锁要求变成契约。

## Source Anchors

- `$KERNEL_SRC/kernel/locking/mutex.c:289` — `mutex_lock()` 实现入口。
- `$KERNEL_SRC/include/linux/mutex.h:167` — `mutex_lock()` 嵌套检查封装。
- `$KERNEL_SRC/include/linux/spinlock.h:382` — `spin_lock_irqsave()`。
- `$KERNEL_SRC/include/asm-generic/atomic-instrumented.h:25` — instrumented `atomic_read()`。
- `$KERNEL_SRC/include/asm-generic/atomic-instrumented.h:238` — `atomic_inc()`。
- `$KERNEL_SRC/include/asm-generic/barrier.h:138` — `smp_store_release()` 架构映射。
- `$KERNEL_SRC/include/asm-generic/barrier.h:142` — `smp_load_acquire()` 架构映射。
- `$KERNEL_SRC/Documentation/memory-barriers.txt:496` — 同一变量上的 RELEASE/ACQUIRE 可见性保证。
- `$KERNEL_SRC/include/linux/lockdep.h:308` — `lockdep_assert_held()`。
- `$KERNEL_SRC/lib/Kconfig.debug:1156` — `PROVE_LOCKING`。

## Connects To

第 4 章提供锁选择所需的上下文约束；第 6 章把原子引用计数、RCU 发布和延迟回收组合成完整对象生命周期。
