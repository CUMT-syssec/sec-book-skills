# 第3章：中断虚拟化

## Core Idea

中断虚拟化的核心，是把物理世界的“设备置位—中断控制器仲裁—CPU 响应”改写成一条可观察、可路由、可注入的状态机。纯软件方案依赖 KVM 在 `VM entry` 前评估虚拟 PIC/LAPIC 并写入 VMCS；APICv、虚拟中断递交和 Posted Interrupt 则把寄存器状态、仲裁逻辑与递交通知逐步下沉到硬件，目标是让正在 Guest 模式运行的 vCPU 不经 `VM exit` 也能接收中断。

## Frameworks Introduced

- **软件中断注入流水线**
  - When to use：理解传统 KVM IRQ 注入、排查中断延迟或 vCPU 唤醒问题。
  - How：设备请求写入虚拟中断芯片 → IRR 记录 pending → 仲裁屏蔽与优先级 → ACK 更新 ISR/IRR → KVM 在中断窗口打开时把向量写入 `VM_ENTRY_INTR_INFO_FIELD` → Guest 通过 IDT 进入处理程序。
  - Why / failure mode：Guest 正在运行或 vCPU 因 `hlt` 睡眠时，必须 kick vCPU，造成 IPI、调度及成对 `VM exit/entry` 开销。

- **PIC/8259A 状态机**
  - When to use：兼容单核或传统 PC 中断模型，理解边沿/电平触发、屏蔽和 EOI。
  - How：`IRR` 保存请求，`IMR` 屏蔽请求，优先级判别器比较 `IRR & ~IMR` 与 `ISR`，ACK 后边沿中断从 IRR 清除并进入 ISR；非 AEOI 模式由 Guest 写 EOI 清 ISR。
  - Why / failure mode：只记录一个“有中断”布尔值会丢失触发类型、优先级和在服务状态；边沿中断若没有先拉低再拉高，也不会形成新请求。

- **APIC 分层路由模型**
  - When to use：多 vCPU、设备中断亲和性和 IPI 场景。
  - How：外设管脚进入 I/O APIC；重定向表按 pin 保存 vector、destination、delivery mode、trigger mode；I/O APIC 选定目标后调用其虚拟 LAPIC；vCPU 间 IPI 则由源 LAPIC 写 ICR，直接选择目标 LAPIC。
  - Why / failure mode：PIC 只有单一 INTR/INTA，无法表达多 CPU 分发；把 I/O APIC 与 LAPIC 混为一体会丢失“全局路由”和“每核接收”两个责任边界。

- **IRQ routing 统一分派**
  - When to use：同一 VM 同时支持 PIC、I/O APIC 和 MSI/MSI-X。
  - How：以 GSI 匹配 `kvm_kernel_irq_routing_entry`，由表项 `set` 回调分派到 PIC、I/O APIC 或 MSI 实现；用户空间可用 `KVM_SET_GSI_ROUTING` 建立/更新路由。
  - Why / failure mode：按 IRQ 号硬编码“同时调用 PIC 和 IOAPIC”无法扩展到 MSI，也容易重复递交。

- **MSI/MSI-X 消息化中断**
  - When to use：PCI/PCIe 设备需要低延迟、多队列和独立中断配置。
  - How：Guest 驱动配置 PCI Capability；设备以 message address 表达目标 APIC，以 message data 表达 vector；KVM 的 MSI 路由解析消息后直接调用目标 LAPIC，绕过 I/O APIC。MSI-X 通过 BIR + Table Offset 定位表，每个向量可独立配置。
  - Why / failure mode：只创建 MSI-X Capability 而未建立 IRQ routing 表项，设备即使“启用 MSI-X”也没有可执行的递交路径。

- **APICv / Posted Interrupt 快路径**
  - When to use：高 IOPS、网络多队列或中断密集型虚拟机，需要削减 VM exit。
  - How：VMCS 指向 `virtual-APIC page`；虚拟中断递交用 `GUEST_INTR_STATUS` 的 RVI/SVI 维护待服务/服务中向量；发送端将 vector 置入 posted-interrupt descriptor，再以专用 notification vector 通知目标物理 CPU。
  - Why / failure mode：APICv 不是“所有 APIC 写都无退出”；写 ICR 等有副作用的操作仍可能需要 VMM。目标 vCPU 不在 Guest 模式时仍需 kick/重调度。

## Key Concepts

- **PIC / 8259A**：最多 8 路输入、可级联的传统可编程中断控制器，适用于单处理器兼容路径。
- **IRR / IMR / ISR**：分别表示待处理请求、屏蔽位和正在服务的中断，是中断生命周期的最小状态集合。
- **LAPIC**：每个处理器/虚拟 CPU 的本地 APIC，接收外设转发中断及 IPI。
- **I/O APIC**：连接外设管脚，通过中断重定向表把请求转换为目标 LAPIC、向量和递交模式。
- **IPI**：处理器间中断；虚拟化中既承载 Guest 的核间通信，也用于把正在 Guest 模式运行的目标 vCPU “踢出”或通知其处理 posted interrupt。
- **MSI / MSI-X**：以 PCI 写消息代替物理管脚的中断机制；MSI-X 支持更多且可独立配置的向量。
- **Interrupt-window exiting**：Guest 暂时不能接收中断时设置的 VMX 控制；一旦 IF 和可中断条件满足，CPU 触发退出以便软件注入。
- **APICv**：对 APIC 寄存器虚拟化、虚拟中断递交等硬件能力的总称，主要价值是减少 APIC 访问和中断递交引起的退出。
- **Posted-interrupt descriptor**：按位记录待递交向量及 notification 状态的数据结构，其地址与通知向量由 VMCS 指定。

## Mental Models

- **把中断看成状态机，而不是一次函数调用**：请求、屏蔽、仲裁、服务、EOI 任一阶段状态错误，都可能表现为丢中断或重复中断。
- **把 APIC 看成两级交换网络**：I/O APIC 做“入口分类与选路”，LAPIC 做“每核排队与递交”；IPI 跳过入口层。
- **把 MSI-X 看成“设备内置的重定向表”**：它把 I/O APIC 的部分路由功能下沉到设备，但最终仍进入目标 LAPIC。
- **把 APICv 看成执行位置迁移**：功能没有消失，而是从用户态/内核态软件逐步迁入 Guest 模式可见的硬件状态与逻辑。

## Anti-patterns

- **只在下一次自然 VM entry 注入**：忙碌或睡眠 vCPU 会产生不可控延迟；必须根据 vCPU 状态 wake 或 kick。
- **混淆 Guest IF 与 PIC IMR**：`cli` 只让 CPU 暂不响应，PIC 仍可锁存/发送；IMR 则在控制器层屏蔽输入。
- **忽略 ACK/EOI 配对**：非 AEOI 下不清 ISR，会阻塞相同或低优先级请求；过早清除电平触发 IRR 也可能丢失仍有效的电平。
- **为每种 irqchip 写独立调用分支**：会导致 PIC/APIC/MSI 逻辑分叉；应以 IRQ routing 将 GSI 与具体递交函数解耦。
- **假设 MSI-X 自动绕过全部虚拟化层**：配置空间、路由建立和 LAPIC 递交仍需正确建模。
- **把 Posted Interrupt 当作无条件直达**：只有目标 vCPU 正在 Guest 模式且硬件/VMCS 配置正确时才能免退出；否则仍走调度路径。

## Code Examples

```c
/* 用户态设备发起管脚中断 */
struct kvm_irq_level irq = { .irq = gsi, .level = 1 };
ioctl(vm_fd, KVM_IRQ_LINE, &irq);

/* KVM 路由后的抽象 */
for_each_route(e, gsi)
    e->set(e, kvm, level);  /* PIC / IOAPIC / MSI */

/* 传统 VM-entry 注入 */
vmcs_write32(VM_ENTRY_INTR_INFO_FIELD,
             vector | INTR_TYPE_EXT_INTR | INTR_INFO_VALID_MASK);
```

- **What it demonstrates**：设备只提交“哪个 GSI、什么电平”，KVM 通过路由选择 irqchip；传统路径最终把向量编码进 VMCS。

```c
/* Posted Interrupt 的关键动作 */
pi_test_and_set_pir(vector, &vcpu->pi_desc);
if (!pi_test_and_set_on(&vcpu->pi_desc) && vcpu->mode == IN_GUEST_MODE)
    send_ipi(vcpu->cpu, POSTED_INTR_VECTOR);
else
    kvm_vcpu_kick(vcpu);
```

- **What it demonstrates**：先发布 descriptor 状态，再根据 vCPU 是否正在 Guest 模式选择专用通知或普通 kick。

## Reference Tables

| 机制 | 路由入口 | 每核接收 | 典型注入点 | 主要开销/约束 |
|---|---|---|---|---|
| PIC/8259A | IR0–IR7，级联扩展 | 无独立每核控制 | VM entry 写 VMCS | 单核兼容；仲裁和 EOI 全由软件模拟 |
| I/O APIC + LAPIC | pin → redirection entry | 每 vCPU 一个 LAPIC | VM entry 或 APICv | 需维护目标、向量、触发和 delivery mode |
| MSI | PCI Capability 消息 | 直接选择目标 LAPIC | 路由到 LAPIC | 向量数有限，配置共享粒度较粗 |
| MSI-X | BIR + Table Offset 指向表 | 每表项独立目标/向量 | 路由到 LAPIC | 配置空间与表访问仍须正确虚拟化 |
| Posted Interrupt | descriptor + notification IPI | 目标 vCPU descriptor | Guest 模式内 | 要求 APICv/VMCS 支持和正确 vCPU 状态 |

| 8259A 状态 | 写入时机 | 读/清除时机 | 错误症状 |
|---|---|---|---|
| IRR | 识别到有效边沿或电平 | ACK 后按触发类型更新 | 丢中断、重复 pending |
| IMR | Guest 编程屏蔽 | 仲裁时参与 `IRR & ~IMR` | 被屏蔽请求意外递交 |
| ISR | 非 AEOI ACK 后置位 | EOI 后清除 | 低优先级中断长期饥饿 |

## Worked Example

以启用 MSI-X 的 virtio-net 队列完成一次收包为例：

1. Guest PCI 驱动枚举设备，在配置空间 Capability List 中发现 MSI-X；根据 BIR 选择 BAR，并用 Table Offset 定位 MSI-X table。
2. Guest 为某个 virtqueue 分配 vector，写入该表项的 message address、message data，并通过 `queue_msix_vector` 告知虚拟设备队列与向量的绑定。
3. 用户态设备从 MSI-X 表项提取 address/data，调用 `KVM_SET_GSI_ROUTING` 建立 `KVM_IRQ_ROUTING_MSI` 表项；此时 GSI 已映射到目标 APIC ID 与 vector。
4. 数据到达后，虚拟设备调用统一的 IRQ 入口。KVM 匹配 GSI，进入 `kvm_set_msi`，解析目标与 vector，调用目标 vCPU 的 `kvm_apic_set_irq`。
5. 传统路径把 vector 置入 LAPIC IRR，必要时 `kvm_vcpu_kick`；目标在下一次 VM entry 时完成仲裁并写 `VM_ENTRY_INTR_INFO_FIELD`。
6. 若启用 APICv/Posted Interrupt，则 KVM 将 vector 置入目标 vCPU 的 PIR，并发送 `POSTED_INTR_VECTOR`。目标 CPU 正在 Guest 模式时直接评估 RVI/SVI 并进入 Guest ISR，无需成对退出/进入。
7. Guest 驱动处理队列并完成相应的 EOI/队列收尾。若向量长期停留在服务中，应优先检查 EOI、触发模式和路由表，而不是只检查设备是否“发过中断”。

## Key Takeaways

1. 先画清楚请求在哪里被记录、在哪里仲裁、在哪里递交，再调试“丢中断”。
2. PIC 的 IRR/IMR/ISR 与 ACK/EOI 构成不可省略的生命周期；APIC 只是把路由扩展到多核。
3. IRQ routing 是 KVM 统一 PIC、I/O APIC 与 MSI/MSI-X 的关键解耦层。
4. MSI-X 绕过 I/O APIC，不绕过配置、路由和 LAPIC。
5. APICv 与 Posted Interrupt 的价值是减少 VM exit，而不是改变 Guest 可见的中断语义。
6. vCPU 是否在 Guest 模式、是否睡眠、是否打开中断窗口，决定了实际递交快慢路径。

## Connects To

- **Ch 1：CPU 虚拟化**：VMCS、VM entry/exit 和 Guest interruptibility 是软件中断注入的执行基础。
- **Ch 4：设备虚拟化**：设备透传依赖中断重映射，并可与 Posted Interrupt 结合形成外设到 Guest 的直达链路。
- **Virtio 虚拟化**：virtqueue 的通知与完成路径通常通过 MSI-X 多向量映射到不同 vCPU。
