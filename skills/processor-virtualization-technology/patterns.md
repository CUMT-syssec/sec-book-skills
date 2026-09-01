# 技术模式与流程

## 能力先行的控制字段设置
**When to use**: 构造 pin-based、processor-based、VM-entry 或 VM-exit 控制字段时。

**How**: 先读取对应能力 MSR；把必须为 1 的位并入期望值，把不允许为 1 的位清除；只有能力位允许后才启用 EPT、VPID、APIC 虚拟化等二级功能。

**Trade-offs**: 牺牲“写死配置”的简洁性，换取跨处理器型号和 VMX 版本的可运行性。

## VMX operation 前置条件链
**When to use**: 每个逻辑处理器第一次执行 VMXON 前。

**How**: 用 CPUID 确认 VMX；检查 IA32_FEATURE_CONTROL；按 fixed-bit MSR 修正 CR0/CR4；设置 CR4.VMXE；准备物理对齐且 revision identifier 正确的 VMXON region；逐处理器执行 VMXON。

**Trade-offs**: 任一条件遗漏都会让 VMXON 异常或 VMfail；严格链式检查增加初始化代码，但显著缩短故障定位。

## VMCS 生命周期
**When to use**: 创建、切换或销毁虚拟处理器上下文时。

**How**: 分配并写入 revision identifier；VMCLEAR 建立 clear 状态；VMPTRLD 设为 current；通过 VMREAD/VMWRITE 配置；首次用 VMLAUNCH，后续用 VMRESUME；切换前保存 current 指针，结束时 VMCLEAR。

**Trade-offs**: clear/current/launched 是独立属性；混淆它们会导致 VMfailValid，而不是普通 guest 故障。

## VM-entry 分层验证
**When to use**: 分析 VMLAUNCH/VMRESUME 为什么未进入 guest。

**How**: 依次检查指令基本条件、执行/退出/入口控制、host-state、guest-state；通过后加载 guest 环境、MSR-load 列表与注入事件，再更新 APIC/中断状态。

**Trade-offs**: 按层排查比搜索单个字段慢一些，但能区分指令失败、guest-state 入口失败和进入后立即 VM-exit。

## VM-exit 结构化分派
**When to use**: 编写通用 VM-exit handler。

**How**: 先读 exit reason，再按原因读取 exit qualification、instruction length、interruption information、guest-linear/GPA 等附加字段；处理完成后只推进应推进的 guest RIP，并决定恢复、反射事件或终止 VM。

**Trade-offs**: 统一分派表便于扩展，但必须保留每类退出的字段有效性规则，不能无条件读取或修改所有信息。

## 向量事件恢复与反射
**When to use**: VM-exit 与异常、NMI 或中断递送重叠时。

**How**: 结合 VM-exit interruption information 与 IDT-vectoring information 判断事件是在执行中触发还是递送中被打断；维护 NMI blocking/unblocking；必要时构造 VM-entry interruption information 重新注入原事件或反射新异常。

**Trade-offs**: 精确恢复保持 guest 语义；错误组合可能把一次异常升级成 #DF，最终形成 triple fault。

## EPT 故障分类与修复
**When to use**: 收到 EPT violation 或 EPT misconfiguration。

**How**: 先按 exit reason 区分“权限拒绝”与“表项非法”；结合 qualification 判断读/写/执行、GPA 来源及故障层级；只对 violation 建立或调整映射，misconfiguration 必须修正表项格式；修改后按范围执行 INVEPT。

**Trade-offs**: 动态修复支持按需映射，但会增加退出与失效成本；过度放宽权限会破坏隔离。

## 两级地址转换
**When to use**: 把 guest 指针定位到 VMM 可访问的 host 地址。

**How**: 先按 guest 当前分页模式把 guest-linear address 转为 GPA，再用 EPT 把 GPA 转为 HPA，最后映射到 VMM 地址空间；分别报告 guest #PF 与 EPT 类故障。

**Trade-offs**: 分层转换保留错误归属；直接把 guest 线性地址当 GPA 会得到错误映射并绕过 guest 页权限语义。

## 有域的转换缓存失效
**When to use**: 修改 guest 页表、EPT 表、CR3、VPID 或 EPTP 后。

**How**: 先识别缓存属于 linear、guest-physical 还是 combined mapping；能按地址/上下文失效时使用 INVLPG、INVVPID 或 INVEPT 的最小作用域；只有无法界定时才扩大为全上下文失效。

**Trade-offs**: 精确失效需要维护 PCID/VPID/EPTP 关联；全局失效实现简单但会损失其他 VM 的热缓存。

## APIC 虚拟化渐进路径
**When to use**: 降低 guest APIC 访问与中断递送造成的 VM-exit。

**How**: 从 EPT/访问退出的软件模拟起步；按能力逐步启用 APIC-access page、virtual-APIC page、TPR shadow、EOI bitmap、APIC-register virtualization、virtual-interrupt delivery，最后在适用时使用 posted interrupt。

**Trade-offs**: 硬件辅助越深，退出越少，但 VMCS 字段耦合和中断状态同步复杂度越高。
