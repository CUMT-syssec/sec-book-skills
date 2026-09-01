# Glossary

**APIC** — 面向多处理器的高级可编程中断控制器体系，由 I/O APIC 与每核 LAPIC 协作。（Ch 3）

**APICv** — 把部分虚拟 APIC 访问、评估和投递逻辑下沉到硬件的虚拟化能力集合。（Ch 3）

**BAR** — PCI Base Address Register，声明设备 PIO/MMIO 地址窗口及其属性。（Ch 4）

**DMA 重映射** — IOMMU 将设备发出的 IOVA/DMA 地址限制并翻译到获授权的 Host 物理页。（Ch 4）

**EOI** — End of Interrupt，处理器/Guest 告知中断控制器当前中断服务结束，可推进后续投递。（Ch 3）

**EPT** — Extended Page Tables，由硬件完成 GPA→HPA 的第二阶段页表遍历。（Ch 2）

**eventfd / ioeventfd** — 以内核事件计数器连接 KVM I/O trap 与用户空间设备线程，使内核可唤醒设备处理而不让 VCPU 线程完整返回用户空间。（Ch 5）

**GPA** — Guest Physical Address，Guest 认为的物理地址。（Ch 2）

**GVA** — Guest Virtual Address，Guest 进程或内核使用的虚拟地址。（Ch 2）

**HPA** — Host Physical Address，真实物理内存地址。（Ch 2）

**HVA** — Host Virtual Address，用户空间 VMM 映射 Guest 内存时使用的 Host 虚拟地址。（Ch 2）

**IOMMU / VT-d** — 为设备 DMA 提供地址翻译、权限隔离及中断重映射的硬件单元/Intel 技术。（Ch 4）

**IOAPIC** — 接收外设管脚中断并依据重定向表选择向量、触发方式和目标 LAPIC。（Ch 3）

**IRR / ISR** — Interrupt Request Register / In-Service Register，分别记录 pending 与正在服务的中断。（Ch 3）

**IRQ routing** — 将统一的 Guest 系统中断号映射到 PIC、IOAPIC 或 MSI 投递函数的路由抽象。（Ch 3）

**KVM_RUN** — 用户空间 VMM 驱动 VCPU 进入 Guest，并通过共享 `kvm_run` 区域交换退出信息的 ioctl。（Ch 1）

**LAPIC** — 每个逻辑处理器关联的 Local APIC，接收外部中断并负责 IPI。（Ch 3）

**memory slot** — KVM 描述一段 GPA、大小及其 HVA 映射关系的 Guest 内存区域。（Ch 2）

**MMIO / PIO** — 内存映射 I/O / 端口 I/O；Guest 访问可被 VM-exit 截获并交由设备模型处理。（Ch 1, Ch 4）

**MSI / MSI-X** — 设备以消息写入形式直接向 LAPIC 请求中断；MSI-X 支持更多且可独立配置的向量。（Ch 3）

**OpenFlow** — 以匹配字段和动作组成流表，分离交换控制逻辑与转发数据面的协议模型。（Ch 6）

**Overlay** — 在 Underlay 物理网络之上封装租户二层/三层网络，使逻辑拓扑独立于物理拓扑。（Ch 6）

**OVS datapath** — Open vSwitch 的内核数据面，缓存控制面决策并执行包处理快路径。（Ch 6）

**PIC / 8259A** — 传统管脚式可编程中断控制器，借助 IRR、ISR、优先级和 EOI 管理中断。（Ch 3）

**posted interrupt** — 硬件把中断记录到投递描述符，并尽可能直接通知运行中的目标 VCPU，减少 VM-exit。（Ch 3）

**SR-IOV** — 单个 PCIe 设备提供 PF 与多个 VF，使 VF 可分别分配给 Guest。（Ch 4）

**TAP** — TUN/TAP 的二层虚拟网络设备，让用户空间 VMM 与 Host 网络栈交换以太网帧。（Ch 6）

**Trap and Emulate** — 让 Guest 普通指令直接执行，敏感操作陷入 VMM 后由其模拟的基本模型。（Ch 1）

**VCPU** — Guest 可见的处理器实例，实质是由 Host 线程、KVM 状态和 VMCS 驱动的执行上下文。（Ch 1）

**Virtio** — Guest 驱动与虚拟设备通过标准化配置和共享队列协作的半虚拟化协议族。（Ch 5）

**Virtqueue** — Virtio 的共享队列，由描述符表、available ring 和 used ring 组成。（Ch 5）

**VLAN / VNI** — VLAN ID 常作节点内局部隔离标签；VXLAN Network Identifier 提供更大的跨节点租户标识空间。（Ch 6）

**VM-entry / VM-exit** — CPU 在 VMX Root 与 non-Root 之间交接执行权的硬件转换。（Ch 1）

**VMCS** — Virtual Machine Control Structure，保存 Guest/Host 状态、执行控制和退出信息。（Ch 1）

**VMX** — Intel 为 x86 增加的虚拟机扩展，提供 Root/non-Root 模式及相关控制指令。（Ch 1）

**VTEP / VXLAN** — VTEP 在 Overlay 边缘封装或解封；VXLAN 用 UDP 承载带 VNI 的租户帧。（Ch 6）

**影子页表** — KVM 软件维护的 GVA→HPA 合成页表，随 Guest 页表和 CR3 变化保持同步。（Ch 2）
