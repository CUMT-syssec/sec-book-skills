# 第 11 章：网络栈、sk_buff、NAPI、netdevice、socket 与 namespace

## Core Idea

Linux 网络路径由多个不同生命周期对象串接：用户态 fd 对应 `struct socket`，协议层使用 `struct sock` 保存连接状态，设备层用 `net_device` 表示接口，包在层间主要由 `sk_buff` 携带。接收侧通常由硬中断只做“停/抑制中断并调度 NAPI”，随后 `NET_RX_SOFTIRQ` 在预算内轮询驱动，构造 skb，进入协议分派，最后排队到 socket 接收队列并唤醒进程。

这套设计的关键不是函数列表，而是上下文切换：硬中断要短，NAPI poll 位于 softirq、不可睡眠，socket 系统调用位于进程上下文、可以阻塞。网络 namespace 又给设备、路由、协议状态和 socket 增加隔离边界；对象不能跨 netns 随意引用。

## Frameworks Introduced

### 1. skb 所有权框架

- **When**：驱动收包、协议转发、clone 或丢包时。
- **How**：明确当前函数是消费 skb、借用 skb 还是返回所有权；需要共享时使用 clone/copy 及其引用规则，失败路径调用适当 free/consume 接口。
- **Why**：`sk_buff` 同时包含元数据和对数据区的引用，clone 后两者共享程度不同。
- **Failure**：向一个会消费 skb 的函数提交后继续访问，或错误路径重复释放，都会 UAF/double free。

### 2. NAPI 预算框架

- **When**：高包速率下平衡吞吐与调度公平。
- **How**：中断调度 NAPI；poll 最多处理 budget 个包。若工作未尽返回 budget，让核心继续轮询；若完成则调用 `napi_complete_done()` 并重新打开设备中断。
- **Why**：中断风暴转为批处理，同时预算防止单设备垄断 softirq。
- **Failure**：有剩余工作却完成 NAPI 会丢唤醒；已清空仍总返回 budget 会造成 busy loop。

### 3. netdevice 状态框架

- **When**：实现注册、打开、发包和注销。
- **How**：`alloc_netdev*` 后初始化 ops/features，`register_netdev()` 使设备可见；注销先停止新流量并等待 RCU/引用边界，再释放私有资源和 netdev。
- **Why**：设备同时暴露给 sysfs、协议栈、namespace 和通知链。
- **Failure**：注册中途失败需撤销 NAPI、队列和硬件；注销后仍由 work/timer 持有指针会 UAF。

### 4. per-net 初始化框架

- **When**：协议或模块维护 namespace 私有状态。
- **How**：注册 `pernet_operations` 的 init/exit；`setup_net()` 按注册顺序初始化，失败时按反序执行 pre-exit、exit 和 free，并跨越 RCU 同步。
- **Why**：新 netns 必须获得一套一致子系统状态。
- **Failure**：init 成功部分未反向清理会泄漏；exit 忽略并发可见性会留下读者访问已释放数据。

## Key Concepts

- `sk_buff`：头尾指针、协议头偏移、设备、路由/控制块和共享数据引用。
- `net_device`：设备状态、队列、features、`net_device_ops`、NAPI 实例的承载者。
- `socket` 与 `sock`：前者连接 VFS/fd，后者属于协议实现与网络状态机。
- softirq/backlog/RPS：收包可在不同 CPU 排队和处理，不能假定中断 CPU 等于协议处理 CPU。
- netns：设备、路由表、防火墙和协议对象的作用域；`struct net` 通过 refcount 管理。

## Mental Models

把收包想成“硬中断按门铃，NAPI 搬一批箱子，协议栈逐层拆箱，socket 排队交付”。每一层都可能消费或重定向 skb。再给整条路径套上 namespace 标签：查设备、路由和协议状态时，都必须来自同一个 `struct net` 上下文。

## Anti-patterns

- 在 NAPI poll 或普通 softirq 中用 mutex、等待 completion 或 `GFP_KERNEL` 分配。
- 不记录 skb 所有权，错误出口同时 `kfree_skb()` 和让下层消费。
- NAPI 完成后才打开中断但没有正确的竞态协议，造成包到达窗口丢事件。
- 使用全局静态表保存本应 per-net 的协议状态。
- 注销 netdevice 时不先取消 timer/work/NAPI，释放后异步回调仍可能运行。
- 把 `netif_receive_skb()` 返回成功理解为包必达 socket；源码说明处理中仍可能丢包。

## Commands & APIs

- `ip -details link`、`ip -s link`：设备能力、状态与计数。
- `ss -naptu`：socket、协议状态和所属进程。
- `ip netns list`、`ip netns exec <ns> ...`：观察 namespace 隔离。
- `cat /proc/net/softnet_stat`：每 CPU 收包处理、丢弃/挤压线索。
- `napi_schedule()`、`napi_complete_done()`：NAPI 调度/完成配对。
- `netif_receive_skb()`：仅用于 softirq 且中断应开启；进入核心接收路径。
- `sock_hold()` / `sock_put()`、`get_net()` / `put_net()`：引用必须成对。

## Worked Example

典型 poll 的关键是 budget 和中断重开顺序：

```c
static int demo_poll(struct napi_struct *napi, int budget)
{
    struct demo_priv *p = container_of(napi, struct demo_priv, napi);
    int work = demo_rx(p, budget);

    if (work < budget && napi_complete_done(napi, work))
        demo_enable_rx_irq(p);

    return work;
}
```

`demo_rx()` 不能睡眠，并须为每个收到的包明确 skb 的交付或释放。真实驱动还需用设备寄存器协议保证“完成轮询—重新开中断”窗口内到达的包不会永久失去通知。

## Key Takeaways

1. socket/sock、net_device 与 skb 分属用户接口、协议状态和包载体三个层次。
2. NAPI 用预算把硬中断转为 softirq 批处理，poll 路径不得睡眠。
3. 每次跨层调用前先确认 skb 所有权是否被消费。
4. netns 初始化是有序事务，失败必须反序退出并等待并发读者。

## Source Anchors

- `$KERNEL_SRC/include/linux/skbuff.h:716` — `struct sk_buff`
- `$KERNEL_SRC/include/linux/netdevice.h:1890` — `struct net_device`
- `$KERNEL_SRC/include/linux/net.h:114` — `struct socket`
- `$KERNEL_SRC/net/core/dev.c:5667` — `netif_receive_skb()` 的上下文契约
- `$KERNEL_SRC/net/core/dev.c:6848` — `napi_poll()`
- `$KERNEL_SRC/net/socket.c:1484` — `sock_create()`
- `$KERNEL_SRC/net/socket.c:1508` — `__sys_socket()`
- `$KERNEL_SRC/net/core/net_namespace.c:330` — `setup_net()` 与反序回滚

## Connects To

- 第 13 章：softirq 执行与延后工作构成 NAPI 接收路径的运行基础。
- 第 5 章：内存序与 per-CPU 同步决定网络快路径的并发正确性。
- 第 6 章：RCU 和引用计数保护 netdevice、skb 及 namespace 生命周期。
- 第 7 章：socket 阻塞等待最终通过调度器唤醒任务。
- 第 12 章：物理网卡通过 device model 绑定驱动并参与 PM。
