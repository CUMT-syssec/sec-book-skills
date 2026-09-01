# 第4章：设备虚拟化

## Core Idea

设备虚拟化是在兼容性、性能、共享性与隔离性之间分层取舍：完全虚拟化忠实模拟寄存器和总线协议，半虚拟化重设计 Guest/Host 交互，设备透传则让数据面绕过 VMM。透传并不等于“完全放行”，配置空间、BAR、DMA 与中断必须分别建立受 VMM 控制的映射和安全边界。

## Frameworks Introduced

- **设备虚拟化演进阶梯**
  - When to use：选择设备模型或定位开销来自哪里。
  - How：完全虚拟化复现硬件接口；vhost 将 dataplane 下沉内核但保留用户态控制面；Virtio 用 virtqueue 取代繁复寄存器交互；VT-d/VFIO 将设备数据面直接交给 Guest；SR-IOV 将 PF 管理出的多个 VF 分别透传。
  - Why / failure mode：越靠近直通，数据路径越短，但硬件依赖、迁移难度和隔离配置责任越高。

- **PCI 配置空间代理模型**
  - When to use：虚拟 PCI 设备枚举、Capability/MSI-X 配置或 VF 透传。
  - How：Guest 通过 `CONFIG_ADDRESS`(0xCF8) 指定 BDF 与寄存器，访问 `CONFIG_DATA`(0xCFC)；VMM 用设备表定位虚拟配置头并读写对应偏移。PCI 传统配置空间为 256 B，PCIe 扩展为 4096 B；头部含 Vendor ID、Device ID、Header Type、BAR，设备相关区承载 Capability。
  - Why / failure mode：透传 VF 的数据面可直达，但配置空间必须代理和过滤；否则 Guest 可篡改 MSI 或映射 Host 资源。

- **BAR 地址请求与双地址视图**
  - When to use：为虚拟设备分配 PIO/MMIO 区域，或将 VF 板上内存映射给 Guest。
  - How：设备用最多 6 个 BAR 声明区域大小、类型和属性；固件/虚拟机监控器分配地址。模拟设备可直接把 Guest 端基址写入虚拟 BAR；透传设备则读取真实 region，分配 GPA，用 `mmap` 获得 HVA，再注册 GPA→HVA 映射，并向 Guest 暴露加工后的 BAR。
  - Why / failure mode：把真实 HPA 原样暴露给 Guest 不可用也不安全；只改 BAR 而不建立 GPA→HVA/HPA 映射，会在真正 MMIO 时失败。

- **VFIO 透传控制面**
  - When to use：将 PF/VF 安全地分配给虚拟机。
  - How：VFIO 枚举设备 region，代理配置空间，映射 BAR，配置 MSI/MSI-X，并把 Guest 内存 bank 注册到 IOMMU domain。数据传输绕过用户态 VMM，敏感配置仍由 VMM/内核过滤。
  - Why / failure mode：把“数据面直通”误解为“所有访问直通”会破坏隔离；配置与中断编程必须继续受控。

- **VT-d / IOMMU DMA 重映射**
  - When to use：任何允许物理设备直接 DMA 到 Guest 内存的透传方案。
  - How：以设备 BDF 选择 domain 页表；VMM/VFIO 用 `VFIO_IOMMU_MAP_DMA` 建立 IOVA（通常等于 GPA）→HPA 映射；设备 DMA 地址先经过 IOMMU，再到内存总线。
  - Why / failure mode：没有 DMA 重映射，恶意或失控设备可访问 Host 或其他 VM 内存；只依赖 CPU 的 EPT 无法约束设备 DMA。

- **中断重映射与 VT-d Posted Interrupt**
  - When to use：透传 MSI/MSI-X 设备既要防伪造中断，又要降低递交延迟。
  - How：中断消息携带 handle/subhandle，硬件索引 IRTE；普通 remapped interrupt 由 IRTE 给出目标 CPU/vector，posted 模式则给出 posted-interrupt descriptor 地址与通知向量。Guest 写 MSI-X 表时触发受控配置，VFIO/IOMMU 更新 IRTE。
  - Why / failure mode：仅有 DMA 隔离不能阻止中断攻击；若 IRTE 来源校验或 domain 绑定错误，设备仍可能干扰 Host/其他 Guest。

- **PIO 完全虚拟化闭环**
  - When to use：实现传统串口、调试 `KVM_EXIT_IO` 或理解用户态设备模拟。
  - How：Guest 执行 `out/in` → VM exit → KVM 从 Exit Qualification 解析方向、宽度、string 标志与端口 → 通过共享的 `kvm_run` 页面把请求交给用户态 → 模拟设备处理 → 对 IN 操作，KVM 在下次进入 Guest 前把返回数据写回 RAX。
  - Why / failure mode：每次端口访问都跨越 Guest、内核、用户态，完全虚拟化的高兼容性以高退出和上下文切换成本为代价。

## Key Concepts

- **完全虚拟化**：按真实设备规范模拟寄存器、PIO/MMIO、中断和状态机，Guest 可使用原生驱动。
- **半虚拟化 / Virtio**：Guest 使用专门驱动和 virtqueue 协议，以更少的陷入和更高批处理效率换取驱动感知。
- **PCI 配置空间**：设备发现与控制入口；传统 PCI 为 256 B，PCIe 为 4096 B，前部为标准头，后部含 Capability。
- **BDF**：Bus/Device/Function 三元组，是 PCI 设备与 IOMMU 选择上下文的基本标识。
- **BAR**：Base Address Register；声明板上 region 映射到 MMIO 还是 PIO，以及其分配后基址。
- **MMIO / PIO**：设备寄存器映射到内存地址空间，或由 `in/out` 专用指令访问的两种 I/O 方式。
- **VFIO**：向用户态 VMM 提供受控设备 region、配置空间、中断和 IOMMU 管理的内核接口。
- **VT-d / IOMMU**：面向设备 DMA 的地址翻译和隔离硬件；其页表与 CPU 的 EPT 是两套独立保护面。
- **SR-IOV PF/VF**：PF 负责设备及 VF 管理，VF 提供可独立透传的数据资源、队列和中断。
- **IRTE**：Interrupt Remapping Table Entry；验证和改写透传设备中断的目标、vector，或指向 posted-interrupt descriptor。

## Mental Models

- **把设备虚拟化拆成控制面与数据面**：配置、资源分配和安全策略留在 VMM/VFIO；高频数据搬运尽量走内核或硬件快路径。
- **把 BAR 看成契约，不是内存本身**：BAR 说明“需要多大、属于哪种地址空间、映射到哪里”；真正访问还依赖 Host Bridge/Root Complex 与页表映射。
- **把透传看成两条独立通道**：数据/DMA 通道由 IOMMU 约束，中断通道由 interrupt remapping 约束；缺一条都不安全。
- **把地址翻译画成两张图**：CPU 访问设备 region 是 GVA→GPA→HPA/PCI region；设备 DMA 是 IOVA/GPA→IOMMU→HPA。不要用 EPT 解释 DMA。
- **把完全虚拟化看成总线协议解释器**：VMM 根据端口/地址片选设备，根据访问宽度和方向更新设备状态，并在必要时生成中断。

## Anti-patterns

- **把全部设备模拟塞进内核**：复杂控制逻辑扩大内核攻击面；仅把高频 dataplane 下沉更合理。
- **向 Guest 暴露真实 BAR/HPA**：Guest 地址视图不同，且可能越权访问 Host 资源；必须重写为 GPA 并注册映射。
- **只配置 VFIO region，不配置 IOMMU domain**：设备能够 DMA，但没有内存边界。
- **只启用 IOMMU，不启用中断重映射**：内存隔离成立，中断来源与目的仍可能被恶意控制。
- **把 PF 当作可任意复制的 VF**：VF 有独立数据资源但仍由 PF 管理；共享能力、复位粒度和隔离程度受具体硬件约束。
- **忽略不存在 PCI 设备的返回语义**：配置读应返回全 1；返回零可能让 Guest 错误识别出设备。
- **创建 vCPU 后才创建 irqchip**：该 vCPU 可能没有正确关联虚拟中断芯片，串口接收链路无法闭合。
- **复制长 I/O 缓冲而不用 `kvm_run` 共享映射**：增加用户态/内核态传递成本，也容易错用 Guest 虚拟地址。

## Code Examples

```text
# Pseudocode: BDF/寄存器由 0xCF8 选择，0xCFC 传数据
offset = (cfg.register_number << 2) + (port - PCI_CONFIG_DATA);
hdr = pci_devices[cfg.device_number];
value = hdr ? load(hdr + offset, size) : all_ones(size);
```

- **What it demonstrates**：VMM 模拟 Host Bridge 的地址译码；不存在设备按 PCI 语义返回全 1。

```c
/* 透传内存：分别建立 CPU 与设备看到的映射 */
hva = mmap(vfio_fd, region_offset, region_size);
kvm__register_dev_mem(gpa, region_size, hva);  /* CPU: GPA -> HPA */

struct vfio_iommu_type1_dma_map m = {
    .iova = guest_phys_addr, .vaddr = host_userspace_addr, .size = len
};
ioctl(container, VFIO_IOMMU_MAP_DMA, &m);      /* DMA: IOVA -> HPA */
```

- **What it demonstrates**：同一 Guest 内存必须分别对 CPU 页表路径与设备 IOMMU 路径建立翻译。

```c
/* 用户态 PIO 模拟 */
ioctl(vcpu_fd, KVM_RUN);
if (run->exit_reason == KVM_EXIT_IO)
    emulate(run->io.port, run->io.direction,
            (char *)run + run->io.data_offset);
```

- **What it demonstrates**：`kvm_run` 同时携带退出元数据和共享 I/O 数据偏移，用户态无需从 Guest 地址直接取数。

## Reference Tables

| 模型 | Guest 驱动 | 数据路径 | 共享/扩展性 | 主要代价 |
|---|---|---|---|---|
| 完全虚拟化 | 原生驱动 | Guest → KVM → 用户态设备 | 易复制多个虚拟设备 | VM exit、上下文切换、完整状态机 |
| vhost | Virtio 驱动 | dataplane 主要在内核 | 软件共享良好 | 内核数据面复杂度 |
| Virtio | 专用半虚拟化驱动 | virtqueue 批量共享内存 | 软件共享良好 | Guest 需支持协议 |
| Direct Assignment | 原生设备驱动 | Guest 直达物理设备 | 整设备独占 | 不易共享、迁移困难 |
| SR-IOV VF | VF 驱动 | Guest 直达 VF | 一个 PF 提供多个 VF | 依赖硬件、PF 管理与隔离能力 |

| 资源/通道 | Guest 可见地址 | Host/硬件实际目标 | 保护机制 |
|---|---|---|---|
| CPU 访问 Guest RAM | GPA | HPA | EPT/影子页表 |
| CPU 访问透传 BAR | GPA/MMIO | PCI region 对应 HPA | VMM 重写 BAR + KVM memory slot |
| 设备 DMA | IOVA，常等于 GPA | Guest RAM 的 HPA | VT-d/IOMMU domain 页表 |
| 设备中断 | MSI handle/subhandle | 目标 LAPIC 或 PI descriptor | Interrupt Remapping / IRTE |

| I/O 类型 | 寻址方式 | 典型虚拟化截获 |
|---|---|---|
| PIO | `in/out` + 端口号 | I/O instruction VM exit，解析 Exit Qualification |
| MMIO | 普通 load/store + 内存地址 | EPT violation 或已注册 MMIO handler |
| PCI 配置 PIO | 0xCF8/0xCFC | VMM 代理 BDF 与寄存器读写 |
| PCIe 配置 MMIO | ECAM/扩展配置空间 | MMIO 路径，配置空间可达 4096 B |

## Worked Example

以“把一个支持 MSI-X 的 SR-IOV VF 透传给 Guest，并完成一次 DMA 与中断”为例：

1. Host 通过 PF 创建 VF，并把目标 VF 交给 VFIO。VMM 注册一个虚拟 PCI 设备，使 Guest 按 BDF 枚举到它；配置读写仍由 VMM/VFIO 代理。
2. VMM 读取 VFIO 的 config region 和 BAR0–BAR5 region。对每个可映射 BAR，在 Guest 物理地址空间分配 GPA；`mmap` 取得 region 的 HVA，再向 KVM 注册 GPA→HVA。虚拟配置空间中的 BAR 写入 GPA，而不是 Host 固件分配的 HPA。
3. VMM 遍历 Guest memory bank，调用 `VFIO_IOMMU_MAP_DMA`。内核把传入的 HVA 解析为 HPA，在该 VF 所属 IOMMU domain 中建立 IOVA/GPA→HPA 页表；设备因此只能 DMA 到这台 Guest 的页。
4. Guest 驱动配置 MSI-X table。该敏感写被 VMM 截获，VFIO 为 Host IRQ 建立中断重映射上下文；MSI 消息中的 handle/subhandle 索引 IRTE，Guest 不能自行指定 Host 或其他 VM 的 LAPIC。
5. Guest 把描述符地址（GPA/IOVA）写入 VF 队列。VF 发起 DMA 时，IOMMU 用 BDF 选中 domain，将 IOVA 翻译为 Guest RAM 的 HPA并完成数据传输；CPU 的 EPT 不参与这次设备侧翻译。
6. VF 完成请求后发送 MSI-X。普通重映射模式由 IRTE 生成安全的目标 CPU/vector；若启用 VT-d Posted Interrupt，IRTE 指向目标 vCPU 的 posted-interrupt descriptor，重映射硬件置位 PIR 并发送 notification，目标可在 Guest 模式直接处理。
7. 若数据正常但无中断，应沿“MSI-X table → VFIO Host IRQ → IRTE → LAPIC/PI descriptor”检查；若中断正常但 DMA fault，应沿“VF BDF → IOMMU domain → IOVA mapping → HPA”检查。两条链路互不替代。

## Key Takeaways

1. 选择设备模型时同时衡量兼容性、退出次数、共享方式、迁移性与隔离责任。
2. PCI 配置空间是控制入口；BAR 只描述和承载映射结果，不等同于板上内存本身。
3. 透传的快路径是数据面，配置空间与安全敏感操作仍必须被代理。
4. CPU 内存访问依靠 EPT，设备 DMA 依靠 IOMMU；排错时必须分开。
5. DMA 重映射保护内存，中断重映射保护中断目的与来源，两者共同构成 VT-d 隔离。
6. VT-d Posted Interrupt 将中断重映射与 APICv 连接起来，使合规透传设备的中断可免 VMM 直达 vCPU。
7. 完全虚拟化串口例子揭示了所有模拟设备的基本闭环：截获、译码、模拟、返回数据、注入中断。

## Connects To

- **Ch 2：内存虚拟化**：BAR 的 GPA→HPA 与 Guest RAM 的 EPT 映射共享 CPU 侧地址翻译基础，但 DMA 另走 IOMMU。
- **Ch 3：中断虚拟化**：MSI/MSI-X、LAPIC、Posted Interrupt 是透传中断直达的终点机制。
- **Virtio 虚拟化**：在不具备或不适合硬件透传时，用 virtqueue 与 vhost 缩短软件数据路径。
