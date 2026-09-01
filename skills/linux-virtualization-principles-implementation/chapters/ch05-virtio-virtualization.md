# 第5章：Virtio 虚拟化

## Core Idea

Virtio 不再让虚拟设备机械复刻真实硬件的寄存器和数据通路，而是在 Guest 驱动与 Host 模拟设备之间建立共享内存队列协议。其性能关键不是“消灭 VM-exit”，而是让一次 VM-exit 批量提交任意长度、可散布的 I/O，并把慢速处理、完成通知和资源回收解耦。

## Frameworks Introduced

- **I/O 表示逐层降维框架**：应用字节流依次被表示为文件系统块、`buffer_head`/page cache、`bio`、`request`，最后由 Virtio 驱动变成 descriptor chain。
  - When to use：追踪“一个用户态读写为何会变成哪些设备请求”或定位数据长度、扇区、页片段转换错误时。
  - How：从文件偏移算文件块，再映射扇区；用 `bio` 表示一组 page 片段；由通用块层合并相邻 `bio`；驱动把 request 拆成 header、若干数据描述符、status 描述符。
  - Why / failure mode：只盯 Virtio 层会误以为数据天然连续；事实上 request 可包含多个非连续内存段，遗漏 scatter-gather 会造成截断、越界或多余复制。

- **Virtqueue 三环所有权框架**：descriptor table 描述缓冲区，available ring 由驱动发布待处理链，used ring 由设备发布完成链。
  - When to use：实现设备、驱动、队列诊断或并发审计时。
  - How：驱动分配并维护描述符；把链头 ID 写入 available ring 并推进 `avail.idx`；设备从 `last_avail_idx` 消费，处理后把 `{id,len}` 写入 used ring 并推进 `used.idx`；驱动从 `last_used_idx` 回收。
  - Why / failure mode：`available` 和 `used` 都是相对另一方的状态窗口，不是两份请求副本。混淆生产者、消费者或索引语义会导致重复消费、丢完成或描述符泄漏。

- **提交—通知—处理—完成—回收生命周期**：数据面通过共享内存批量传递，控制面只在必要时 notify 或注入中断。
  - When to use：分析吞吐、延迟、队列卡死以及中断风暴时。
  - How：先完整发布描述符链与 available 条目，再写 `Queue Notify`；设备解析 GPA 指向的数据、执行后端 I/O、填 status 和 used 条目；驱动在中断或轮询中完成上层 request 并归还整条链。
  - Why / failure mode：notify 不是传数据，只是“队列已有工作”的门铃。若先 notify 后发布，设备可能看见不完整状态；若完成后不回收，队列最终耗尽。

- **同步与异步 I/O 决策框架**：极短控制命令可在 Guest 中轮询完成，长耗时数据 I/O 应交给 Host 工作线程并用中断完成。
  - When to use：选择轮询、阻塞、中断或线程池时。
  - How：估计设备处理时间与调度/中断成本；短操作 `kick → get_buf` 轮询，长操作 `kick → Guest 继续运行 → worker 完成 → IRQ → reclaim`。
  - Why / failure mode：同步等待能降低短命令延迟，却会在慢 I/O 上阻塞整个 Guest；异步化把 VCPU 与 I/O worker 解耦，多核上还可并行。

- **轻量 VM-exit（ioeventfd）框架**：KVM 在 Host 内核态处理 notify，只唤醒监听 eventfd 的用户态 I/O 线程，然后直接返回 Guest。
  - When to use：每次 I/O 的用户态往返成本已成为瓶颈时。
  - How：VMM 为 queue 创建 eventfd，以 `KVM_IOEVENTFD` 注册“notify 地址—eventfd”映射；KVM 的 I/O exit handler 命中代理设备后 `eventfd_signal`；VMM 的 epoll 线程执行回调并调度后端任务。
  - Why / failure mode：它减少的是 VCPU 路径上的 Host 内核态↔用户态切换，不等于设备完全绕过用户态，也不替代 used ring 和完成中断。

## Key Concepts

- **Virtio**：为半虚拟化设备定义的驱动—设备交互标准，以共享队列替代对真实硬件接口的逐寄存器模拟。
- **Virtqueue**：Guest 驱动拥有并分配、驱动与设备共同访问的队列；一个设备可有控制、发送、接收等多个 per-queue 实例。
- **Descriptor Table**：描述缓冲区 GPA、长度、方向和链关系的数组；`VRING_DESC_F_NEXT` 连接后继，`VRING_DESC_F_WRITE` 表示设备可写。
- **Available Ring**：驱动向设备发布 descriptor chain 头 ID 的环；有效区间是设备的 `last_avail_idx` 到驱动的 `avail.idx - 1`。
- **Used Ring**：设备向驱动发布已完成链头 ID 与写回长度的环；有效区间是驱动的 `last_used_idx` 到设备的 `used.idx - 1`。
- **Descriptor Chain**：一次请求的 scatter-gather 表示；Virtio blk 常由命令/起始扇区 header、一个或多个数据段、设备写回 status 组成。
- **Notify**：驱动写 `Queue Notify` 触发 VM-exit，告知设备某个 Virtqueue 出现可消费工作；数据本身仍在共享内存。
- **回收**：驱动依据 used 元素的链头 ID 找回原 request，通知通用块层完成，再把链中所有描述符接回 free list。
- **异步 I/O**：VCPU 只提交工作，Host 线程池处理磁盘镜像，完成后注入中断；避免慢 I/O 阻塞 Guest。
- **轻量 VM-exit**：借助 ioeventfd 在 Host 内核态完成事件投递，使 VCPU 无须先返回 VMM 用户态即可重新进入 Guest。

## Mental Models

- 把 Virtqueue 看成“共享内存中的双向记账本”：descriptor 是货物地址，available 是待办清单，used 是签收回执。
- 把 notify 看成门铃而不是运输车：批量数据已在内存中，门铃只负责改变执行方。
- 用“谁分配、谁发布、谁推进索引、谁回收”审计队列；每个字段都应有唯一写入方或严格约定的并发协议。
- 把异步化与轻量退出视为两层独立优化：前者缩短设备处理占用 VCPU 的时间，后者缩短 notify 的 VCPU 退出路径。

## Anti-patterns

- **照搬物理设备寄存器协议**：软件设备没有总线宽度和真实寄存器约束，会制造大量小 I/O 与 VM-exit。
- **假设 request 对应连续缓冲区**：`bio`/page 片段可能离散，必须用 descriptor chain 表达。
- **把 `idx` 当数组下标而忽略环绕**：实际访问需对 ring 大小取模，逻辑计数仍持续递增。
- **只完成上层请求、不释放描述符链**：短期看 I/O 成功，长期必然耗尽 `num_free`。
- **对慢 I/O 忙等**：VCPU 被占住，Guest 其他任务无法获得运行机会。
- **把 ioeventfd 宣称为零 VM-exit**：notify 仍使 CPU 从 Guest 进入 Host 内核，只是避免返回 VMM 主线程再折返。
- **未完成内存发布就 notify**：设备可能读取到旧的索引、未初始化描述符或不完整的链。

## Code Examples

```text
# Pseudocode: 驱动发布一个链并敲门
desc[h] = { .addr = gpa, .len = n, .flags = NEXT, .next = data };
avail->ring[avail->idx % qsz] = h;
avail->idx++;
iowrite16(queue_id, QUEUE_NOTIFY);
```

```text
# Pseudocode: 设备与驱动消费、完成、回收
while (last_avail != avail->idx) process_chain(avail->ring[last_avail++ % qsz]);
used->ring[used->idx++ % qsz] = (used_elem){ .id = head, .len = written };
while (last_used != used->idx) complete_and_detach(used->ring[last_used++ % qsz].id);
```

```c
// 轻量通知关键 API 组合
fd = eventfd(0, 0);
ioctl(vm_fd, KVM_IOEVENTFD, &notify_mapping);
epoll_wait(epfd, events, maxevents, -1);
```

- **What it demonstrates**：共享内存承担数据与状态传递；notify、IRQ/eventfd 只负责跨执行域唤醒。

## Reference Tables

| 结构 | 主要写入方 | 主要读取方 | 核心字段 | 生命周期作用 |
|---|---|---|---|---|
| descriptor table | 驱动；设备仅写 `WRITE` 缓冲 | 设备、回收路径 | `addr,len,flags,next` | 描述命令、数据和状态缓冲 |
| available ring | 驱动 | 设备 | `idx, ring[head_id]` | 发布待处理描述符链 |
| used ring | 设备 | 驱动 | `idx, ring[{id,len}]` | 发布已完成链及写回长度 |
| free list / `data[]` | 驱动 | 驱动 | `free_head,num_free,request` | 关联请求并循环复用描述符 |

| 模式 | VCPU 等待方式 | 完成方式 | 适用场景 | 主要风险 |
|---|---|---|---|---|
| 同步轮询 | 原地 `cpu_relax()` | 读到 used 条目 | 极短控制命令 | 慢操作拖死 Guest |
| 同步设备处理 | VM-exit 后等 VMM 做完 | VMM 注入 IRQ 后返回 | 简单原型 | 整个 Guest 被后端 I/O 阻塞 |
| 异步线程池 | notify 后快速回 Guest | worker 完成并注入 IRQ | 磁盘、网络等长操作 | 需正确处理并发和队列寿命 |
| ioeventfd + 异步 | Host 内核唤醒 worker 后直接回 Guest | worker + IRQ/used ring | 高频 notify | 只优化退出路径，不消除后端成本 |

## Worked Example

以 Virtio blk 写请求为例，沿端到端路径重构一次执行：

1. 应用写文件后，文件系统把字节偏移映射到文件块；page cache 持有脏页。通用块层用 `bio` 描述页、页内偏移和长度，并将相邻扇区的 `bio` 合并为 request。
2. Virtio blk 驱动从 free list 取得描述符，建立链：首描述符指向包含“写命令 + 起始扇区”的 header；中间若干 out 描述符分别指向 request 的数据片段；尾描述符是带 `VRING_DESC_F_WRITE` 的 status 缓冲。
3. 驱动把 Guest 虚拟地址转换为设备可解释的 GPA，保存 `data[head] = request`，把 `head` 写入 `avail.ring[avail.idx % qsz]`，推进 `avail.idx`。
4. 驱动向 `Queue Notify` 写队列号。普通路径发生 VM-exit；配置 ioeventfd 时，KVM 在 Host 内核态命中 notify 映射，唤醒监听 eventfd 的 kvmtool 线程，并让 VCPU 直接重新进入 Guest。
5. kvmtool 回调把该队列的 job 交给线程池。worker 比较 `last_avail_idx` 与 `avail.idx`，取出链头；逐描述符把 GPA 转成 HVA，读取 header 和数据段，将数据写入虚拟磁盘镜像，并在 status 缓冲写结果。
6. 设备把 `{head, written_len}` 追加到 `used.ring`，推进 `used.idx`，再向 Guest 注入 Virtio blk 中断。此时后端 I/O 与 VCPU 可在不同 CPU 上并发。
7. Guest 的 `vp_interrupt → blk_done → get_buf` 路径读取 used 条目，以 `head` 找回 request，检查 status，调用块层完成函数唤醒等待任务。
8. `detach_buf` 遍历整条 descriptor chain，增加 `num_free`，把链重新接到 `free_head`；`last_used_idx` 前移。只有完成这一步，请求生命周期才真正闭合。

这个例子同时说明三个性能来源：scatter-gather 避免强制连续内存和重复拷贝；一次 notify 可对应多个 pending request；异步 worker 与 ioeventfd 分别缩短设备执行阻塞和 VM-exit 用户态往返。

## Key Takeaways

1. Virtio 的核心不是一个设备型号，而是一套以 Virtqueue 为中心的共享内存生产者—消费者协议。
2. descriptor/available/used ring 必须连同各自索引、链头 ID 和所有权一起理解。
3. notify 触发执行，used ring 表达完成，回收 descriptor 才闭合资源生命周期。
4. 短控制命令可轮询，长数据 I/O 应异步；选择依据是完成时间与调度成本，而非接口形式。
5. ioeventfd 让 KVM 在内核态唤醒用户态 worker 后直接回 Guest，是“轻量 VM-exit”，不是“无 VM-exit”。
6. 调试 Virtio 应沿“上层 request → 描述符链 → available → notify → used → IRQ → free list”逐段核对。

## Connects To

- **Ch 1：CPU 虚拟化**：VM-exit handler、I/O 总线与中断注入是 notify 和完成路径的执行基础。
- **Ch 4：设备虚拟化**：Virtio 是从完全模拟转向半虚拟化、降低设备模拟开销的具体方案。
- **Ch 6：网络虚拟化**：Virtio-net 可把虚拟机报文送到 TAP，随后进入 bridge、OVS、Overlay 数据路径。
- **Linux 块层**：`bio`、request、I/O 调度和 page cache 决定驱动最终要编码进 descriptor chain 的数据形态。
