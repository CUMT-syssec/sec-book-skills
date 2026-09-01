# 术语表

**A20M** — 兼容早期 x86 地址回绕行为的模式；进入 VMX operation 前需要按处理器约束处理。(Ch 2)

**Access rights** — VMCS 中描述段类型、特权级、present、粒度及 unusable 等属性的字段集合。(Ch 3, Ch 4)

**Activity state** — 描述 guest 处于 active、HLT、shutdown 或 wait-for-SIPI 等活动状态的 VMCS 字段。(Ch 3, Ch 4, Ch 5)

**APIC-access page** — 用于捕获 guest 对内存映射 Local APIC 访问的专用页面。(Ch 3, Ch 7)

**APIC-register virtualization** — 让部分 APIC 寄存器访问在硬件辅助下完成虚拟化的二级执行控制。(Ch 3, Ch 7)

**Combined mapping** — 将 guest 线性地址经 guest 分页与 EPT 两级转换后形成的组合映射缓存。(Ch 6)

**CR3-target** — 允许指定若干 CR3 值，使匹配的 CR3-load 不必触发 VM-exit。(Ch 3, Ch 4)

**EOI-exit bitmap** — 决定 guest 对特定中断向量执行 EOI 时是否触发 VM-exit 的位图。(Ch 3, Ch 7)

**EPT** — Extended Page Tables；把 GPA 转换为 HPA 的第二级页表机制。(Ch 2, Ch 3, Ch 6)

**EPT misconfiguration** — EPT 表项存在保留位、权限或内存类型等非法组合时产生的配置错误退出。(Ch 6)

**EPT violation** — EPT 权限不允许当前读、写或执行访问时产生的 VM-exit。(Ch 3, Ch 6, Ch 7)

**EPTP** — 指向 EPT PML4 表并携带遍历长度、内存类型等控制信息的指针。(Ch 3, Ch 6)

**EPTP switching** — 通过 VM function 在多个预先登记的 EPTP 之间切换。(Ch 3, Ch 6)

**Exit qualification** — 对特定 VM-exit 原因补充操作数、地址、访问类型或控制寄存器细节的信息字段。(Ch 3, Ch 5)

**Exit reason** — 标识 VM-exit 根因及入口失败等属性的基本退出信息字段。(Ch 3, Ch 5)

**GPA** — Guest-physical address；guest 分页得到并由 EPT 继续转换的地址。(Ch 6)

**Guest-linear address** — guest 指令形成、尚未经过 guest 分页转换的线性地址。(Ch 3, Ch 6)

**HPA** — Host-physical address；两级地址转换最终访问的物理地址。(Ch 6)

**IDT-vectoring information** — 描述 VM-exit 发生时正在递送的向量事件，用于恢复或重新注入。(Ch 3, Ch 5, Ch 7)

**INVEPT** — 按指定范围失效 EPT 派生转换缓存的指令。(Ch 2, Ch 6)

**INVVPID** — 按 VPID 与地址范围失效 guest 线性地址转换缓存的指令。(Ch 2, Ch 6)

**Interruptibility state** — 记录 STI/MOV SS 阻塞、SMI/NMI 阻塞等可中断状态的 guest-state 字段。(Ch 3, Ch 4, Ch 5)

**Local APIC virtualization** — 对 guest 的 APIC 基址、寄存器、EOI、TPR 和中断递送进行监控或硬件辅助处理。(Ch 7)

**MSR bitmap** — 按 MSR 编号及读写方向选择是否触发 VM-exit 的位图。(Ch 3, Ch 5)

**NMI-window exiting** — 当 guest 可接收 NMI 时请求 VM-exit，以便 VMM 安全注入待处理 NMI。(Ch 3, Ch 7)

**PCB** — Processor Control Block；系统平台为每个逻辑处理器维护 VMX 与运行期状态的数据结构。(Ch 1)

**Posted interrupt** — 通过描述符和通知向量把虚拟中断直接投递给运行中的虚拟处理器。(Ch 3, Ch 7)

**SDA** — System Data Area；系统平台共享的全局管理数据区域。(Ch 1)

**TPR shadow** — 用 virtual-APIC 页上的影子 TPR 减少 guest 访问 TPR 产生的退出。(Ch 3, Ch 7)

**Unrestricted guest** — 允许 guest 在未开启分页或保护模式等传统受限状态下运行的 VMX 能力。(Ch 3, Ch 4, Ch 6)

**Virtual-APIC page** — 保存 guest APIC 虚拟状态并供硬件辅助访问的页面。(Ch 3, Ch 4, Ch 7)

**Virtual-interrupt delivery** — 在 APIC 虚拟化状态上评估并递送虚拟中断的硬件机制。(Ch 3, Ch 4, Ch 7)

**VMCS** — Virtual Machine Control Structure；保存执行控制、退出控制、入口控制、guest-state、host-state 与退出信息的核心结构。(Ch 2–5)

**VM-entry** — 从 VMM/host 环境进入或恢复 guest 的受检状态转换。(Ch 2, Ch 4)

**VM-exit** — 处理器停止 guest 并按照 VMCS 保存 guest、记录原因、加载 host 的状态转换。(Ch 2, Ch 5)

**VMfailInvalid** — VMX 指令因 current VMCS 指针无效等条件失败，且无法在 VMCS 中记录指令错误。(Ch 2, Ch 4)

**VMfailValid** — VMX 指令在有效 current VMCS 下失败，并在 VM-instruction error 字段记录原因。(Ch 2, Ch 4)

**VMLAUNCH / VMRESUME** — 分别用于首次启动 clear VMCS 和恢复 launched VMCS 的 VM-entry 指令。(Ch 2, Ch 4)

**VMX non-root operation** — 通常运行 guest 的 VMX 执行域；受 VMCS 执行控制约束。(Ch 2)

**VMX root operation** — 通常运行 VMM 的 VMX 执行域；VM-exit 的目标域。(Ch 2)

**VMXON region** — 进入 VMX operation 时由 VMXON 指令引用、包含 revision identifier 的物理区域。(Ch 2, Ch 3)

**VPID** — Virtual-processor identifier；为不同虚拟处理器标记线性地址转换缓存域。(Ch 2, Ch 3, Ch 6)
