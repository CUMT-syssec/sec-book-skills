# Patterns

## 虚拟化三条件检查
**When to use**：评审一个“虚拟化”或“加速”方案。

**How**：分别验证等价性、高效性和资源控制；把不能安全直通的敏感操作列入 trap 策略。

**Trade-offs**：增强等价性常增加模拟面；过度 trap 会破坏高效性。

## VM-exit 成本阶梯
**When to use**：定位 Guest 高频操作的延迟或吞吐瓶颈。

**How**：记录路径是否停留在 Guest、KVM 内核、用户空间 VMM、Host 协议栈；优先消除最频繁且最深的切换。

**Trade-offs**：下沉内核/硬件会提高速度，但增加内核复杂度、隔离风险或可观测难度。

## VMCS 状态交接审计
**When to use**：调试 VM-entry 失败、寄存器错乱或异常 VM-exit。

**How**：核对 Guest/Host state、execution controls、exit reason/qualification，以及软件补充保存的上下文。

**Trade-offs**：只读单一 VMCS 字段容易忽略字段间约束和硬件校验。

## 四地址链分解
**When to use**：调试 Guest 缺页、错误映射或 DMA 地址问题。

**How**：明确 GVA→GPA、GPA→HVA、HVA→HPA 的拥有者、页大小、权限和失效点；设备 DMA 另画 IOVA→HPA。

**Trade-offs**：合并表示法更简洁，但会掩盖是哪一级翻译失败。

## 影子页表双阶段缺页
**When to use**：理解无 EPT 环境的 Guest 缺页。

**How**：先遍历 Guest 页表；若 GVA→GPA 不存在，向 Guest 注入缺页；存在后再建立 GVA→HPA 影子映射并缓存。

**Trade-offs**：兼容旧硬件，但同步 CR3/PTE、缓存和失效的成本高。

## EPT 二级翻译
**When to use**：硬件支持 EPT 且希望减少 CR3 trap/软件 page walk。

**How**：Guest 页表负责 GVA→GPA，EPT 负责 GPA→HPA；为 EPT violation 建立映射、权限和失效策略。

**Trade-offs**：降低退出次数，但 page walk 更深，TLB/EPTP 失效策略仍影响性能。

## 中断投递状态机
**When to use**：排查丢中断、重复中断或中断延迟。

**How**：逐项验证请求、路由、IRR/pending、目标 VCPU、注入字段、ACK、ISR 与 EOI；同时核对 edge/level。

**Trade-offs**：硬件辅助减少退出，但状态分散在虚拟芯片、VMCS 和硬件描述符中。

## IRQ Routing 适配层
**When to use**：同一设备模型需兼容 PIC、IOAPIC 与 MSI。

**How**：把 GSI/消息映射成路由表项与投递回调，设备只调用统一注入入口。

**Trade-offs**：统一接口降低耦合，但错误或重复表项会造成双重投递。

## PCI 设备地址窗口模拟
**When to use**：模拟 PCI 配置空间和 BAR。

**How**：先模拟配置读写与 BAR 探测，再注册对应 PIO/MMIO 区间，最后把 trap 分派到设备状态机。

**Trade-offs**：兼容原生驱动，但高频寄存器访问会产生大量 VM-exit。

## 透传闭环
**When to use**：把 PF/VF 直接分配给 Guest。

**How**：同时配置虚拟配置空间、IOMMU 域、DMA 映射、设备重置、MSI 中断重映射和撤销顺序。

**Trade-offs**：性能高；迁移、共享、故障隔离与回收更难。

## Virtqueue 所有权交接
**When to use**：实现或排查 Virtio 驱动/设备。

**How**：驱动构建 descriptor chain→发布 available→通知；设备消费→发布 used→通知；驱动回收；每步定义索引、屏障和所有者。

**Trade-offs**：批处理和通知抑制提高吞吐，但增加延迟与卡队列诊断难度。

## ioeventfd 轻量退出
**When to use**：Guest 高频 notify 不值得让 VCPU 线程完整返回用户空间。

**How**：把通知地址注册到 ioeventfd；KVM 内核触发 eventfd 并直接 re-entry，独立设备线程通过 epoll 处理 I/O。

**Trade-offs**：减少上下文切换，但引入异步并发、生命周期和队列过载问题。

## OVS 首包慢路径与缓存快路径
**When to use**：解释控制面规则正确但首包慢，或数据面行为与表面拓扑不同。

**How**：首包上送控制面匹配 OpenFlow，决策下发 datapath cache；后续同类包在内核直接执行归约动作。

**Trade-offs**：快路径高效；缓存陈旧、匹配掩码或优先级错误会造成难以直观看到的偏差。

## Overlay 标识转换
**When to use**：跨计算节点隔离重叠租户网段。

**How**：节点内以 VLAN/端口区分租户，在 br-tun 边界映射为 VNI，VTEP 用 Underlay IP/UDP 封装；入站反向转换。

**Trade-offs**：扩展性强，但 MTU、隧道状态、FDB/流表和标识映射成为新增故障面。

## 双向包路径追踪
**When to use**：虚拟机访问外网或 Floating IP 单向不通。

**How**：固定五元组，分别画出出站与回程的 TAP、Linux bridge、OVS、VXLAN、namespace、路由/NAT、br-ex 节点；在每个语义变化点取证。

**Trade-offs**：步骤多，但能避免只在单桥抓包就误判整条路径。
