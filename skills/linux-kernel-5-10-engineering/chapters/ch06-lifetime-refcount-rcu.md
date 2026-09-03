# 第 6 章：refcount、kref、RCU 与对象销毁

## Core Idea

对象生命周期协议必须回答三件事：谁创建初始所有权，谁能取得新引用，最后一个引用在哪里触发销毁。`refcount_t` 提供引用计数专用的饱和与误用检查；`kref` 在其上包装“最后一次 put 调 release”。RCU 则解决读者与移除并发：先让新读者不可见，再等待旧读者离开，最后回收。

引用计数和 RCU 经常组合，但作用不同：引用保证取得后对象存活；RCU 保证查找期间对象不会被立即释放，并提供安全取得引用的窗口。计数已经归零后不能用普通 `kref_get()` 复活；查找路径必须使用 `kref_get_unless_zero()` 并检查返回值。

## Frameworks Introduced

### 生命周期状态机

- **When**：对象会被异步工作、IRQ、文件句柄、定时器或并发查找持有时。
- **How**：写出 `allocated → published → detached → quiescent → freed`，为每条边指定锁、引用或 grace period。
- **Why**：很多 UAF 发生在“从容器移除”与“真正释放”被错误合并时。
- **Failure**：删除列表节点后立即 `kfree()`，仍在 RCU 临界区的读者访问悬空对象。

### 查找并取引用

- **When**：从共享表/链表获得对象并在临界区外继续使用。
- **How**：在锁或 RCU 保护下定位对象，使用非零增引用，失败则视为对象正在死亡；成功后可退出查找保护。
- **Why**：定位与增引用之间存在归零竞态。
- **Failure**：先返回裸指针，稍后再 `kref_get()`。

## Key Concepts

- **初始引用**：`kref_init()` 将计数设为 1；它代表一个真实所有者，不能仅为“防止归零”而永久泄漏。
- **get/put 配对**：每个成功交付给异步消费者的引用必须有明确 put；排队失败或取消路径也要闭合。
- **release 回调**：`kref_put()` 在 `refcount_dec_and_test()` 成功时调用 release；release 通常由 `container_of()` 取回宿主并释放资源。
- **不可复活**：从共享容器查找时用 `kref_get_unless_zero()`；返回 false 表示对象已进入终止状态。
- **RCU 读侧**：`rcu_read_lock()` 到 unlock 之间用 `rcu_dereference()` 读取受保护指针；读侧通常应短小且遵守对应 RCU flavor 约束。
- **发布/替换**：写侧用锁维护更新者之间互斥，用 `rcu_assign_pointer()` 发布。移除后，可在允许阻塞的上下文调用 `synchronize_rcu()` 同步等待；原子/IRQ 路径应使用 `call_rcu()` 异步安排回收。
- **回调上下文**：`call_rcu()` 回调可能在 softirq 上下文执行，因此不得睡眠。若宽限期后的析构包含 mutex、阻塞 I/O 或其他可睡眠步骤，RCU callback 只负责把工作转交 workqueue，由 worker 完成复杂析构。
- **`kfree_rcu()`**：适合对象嵌入 `rcu_head` 且最终动作只是释放；复杂析构用显式 callback。

## Mental Models

把“可达性”和“存活性”分开：容器中的指针提供可达性，引用提供存活性。写侧先切断可达性，旧持有者仍可靠引用或 RCU 宽限期完成访问，最后才释放。

销毁不是一个点，而是一条栅栏序列：停止新入口 → 从索引摘除 → 同步 IRQ/work/timer → 等待 RCU 读者 → 丢弃拥有者引用 → release。实际对象可根据依赖调整顺序，但每个异步来源都必须有关闭动作。

## Anti-patterns

- 使用 `atomic_t` 自制引用计数，遗漏溢出、下溢或归零后的获取规则。
- `kref_read()==1` 后据此决定释放；读取与后续动作之间可竞态。
- 从哈希表拿到裸指针，解锁后才增引用。
- `list_del_rcu()` 后立即释放，误以为“删除”会等待读者。
- 在 RCU 读侧保存指针供临界区外使用，却没有获得独立引用。
- release 中只 `kfree()` 宿主，忘记取消 work、timer 或注销回调。

## Commands & APIs

核心 API：`refcount_set/inc/inc_not_zero/dec_and_test`、`kref_init/get/get_unless_zero/put`、`rcu_read_lock/unlock`、`rcu_dereference`、`rcu_assign_pointer`、`synchronize_rcu`、`call_rcu`、`kfree_rcu`。

审计模板：

```bash
rg -n 'kref_(init|get|put|get_unless_zero)' path/to/subsystem
rg -n 'call_rcu|kfree_rcu|synchronize_rcu' path/to/subsystem
rg -n 'cancel_.*work|del_timer_sync|synchronize_irq' path/to/subsystem
```

对每个 get 建表记录成功路径、失败路径和 put 位置；对每个异步注册记录注销/同步位置。这比只搜索 `kfree()` 更容易发现遗漏。

## Worked Example：RCU 查表后跨临界区使用对象

```c
static struct obj *obj_lookup_get(int id)
{
	struct obj *p;

	if (id < 0 || id >= ARRAY_SIZE(table))
		return NULL;
	rcu_read_lock();
	p = rcu_dereference(table[id]);
	if (p && !kref_get_unless_zero(&p->ref))
		p = NULL;
	rcu_read_unlock();
	return p;
}

static void obj_release(struct kref *ref)
{
	struct obj *p = container_of(ref, struct obj, ref);
	kfree(p);
}
```

写侧必须先在适当锁下把 `table[id]` 替换为 NULL，再等待 RCU 查找窗口结束，随后释放表拥有的引用。这里的同步等待只能发生在可阻塞上下文；不能阻塞时改用 `call_rcu()`，且 callback 本身不得睡眠。成功查找者可在 RCU 临界区外使用对象，完成后 `kref_put(&p->ref, obj_release)`。如果对象还有 work/timer，最后引用归零前还必须保证这些来源已停止或各自持有引用。

## Key Takeaways

- 引用计数管理存活，RCU 管理并发可达与延迟回收。
- 查找与增引用必须组成一个受保护协议，禁止归零复活。
- 销毁要先关闭入口和异步来源，再最终释放。
- 对象生命周期最适合用状态机和 get/put 账本审查。

## Source Anchors

- `$KERNEL_SRC/include/linux/kref.h:19` — `struct kref`。
- `$KERNEL_SRC/include/linux/kref.h:29` — `kref_init()`。
- `$KERNEL_SRC/include/linux/kref.h:62` — `kref_put()` 与 release 回调。
- `$KERNEL_SRC/include/linux/kref.h:109` — `kref_get_unless_zero()` 实现入口。
- `$KERNEL_SRC/include/linux/refcount.h:130` — `refcount_set()`。
- `$KERNEL_SRC/include/linux/refcount.h:231` — `refcount_inc_not_zero()`。
- `$KERNEL_SRC/include/linux/refcount.h:319` — `refcount_dec_and_test()`。
- `$KERNEL_SRC/include/linux/rcupdate.h:40` — `call_rcu()`。
- `$KERNEL_SRC/include/linux/rcupdate.h:460` — `rcu_assign_pointer()`。
- `$KERNEL_SRC/include/linux/rcupdate.h:627` — `rcu_dereference()`。
- `$KERNEL_SRC/include/linux/rcupdate.h:673` — `rcu_read_lock()` 语义。
- `$KERNEL_SRC/include/linux/rcupdate.h:928` — `kfree_rcu()`。
- `$KERNEL_SRC/kernel/rcu/tree.c:3714` — `synchronize_rcu()` 同步等待入口。
- `$KERNEL_SRC/kernel/rcu/tree.c:3725` — `synchronize_rcu()` 进入 `wait_rcu_gp()`。
- `$KERNEL_SRC/Documentation/RCU/UP.rst:125` — RCU callback 可从 softirq 上下文调用。

## Connects To

第 4 章的 IRQ/work 延后处理必须持有可靠引用；第 5 章的锁和 acquire/release 负责发布顺序。后续设备、VFS、网络等章节都可用本章状态机审计注册、查找与 teardown。
