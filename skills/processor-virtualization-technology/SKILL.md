---
name: processor-virtualization-technology
description: 'Knowledge base from "处理器虚拟化技术" by 邓志. Use when referencing Intel VMX, VMCS, VM-entry/VM-exit, EPT/VPID, address-translation caches, exception reflection, or Local APIC virtualization.'
---

<!-- argument-hint: [topic, framework name, or chapter number] -->

# 处理器虚拟化技术

**Author**: 邓志 | **Pages**: 646（PDF 671） | **Chapters**: 7 | **Generated**: 2026-09-01

## How to Use This Skill

- **无参数** — 加载下列核心模型，快速定位 VMX 问题属于哪个阶段。
- **按主题** — 询问 `EPT violation`、`VM-entry failure`、`posted interrupt` 等；我会读取相关章节。
- **按章节** — 询问 `ch03` 或“第 3 章”，直接加载对应参考文件。
- **浏览** — 询问“有哪些章节/主题”，查看索引。

核心区未覆盖的位定义、流程细节或例子必须先读取对应章节文件，再回答。

---

## Core Frameworks & Mental Models

### 1. 把 VMX 看成受约束的状态机

用 `VMX operation → root/non-root operation → VM-entry/VM-exit` 状态转换解释处理器行为。先问“当前逻辑处理器在哪个状态、current VMCS 是谁、VMCS 是 clear 还是 launched”，再解释指令结果。不要把 VMM 与 VMX root、guest 与 VMX non-root 机械等同；它们是常见用法，不是定义本身。

### 2. 用能力 MSR 约束配置，而不是写死控制位

设置 VM-execution、VM-entry、VM-exit 等控制字段时，先读取对应能力 MSR 的 allowed-0/allowed-1 信息：必须为 1 的位要保留，不允许为 1 的位要清除。只有处理器明确支持后才启用 secondary controls、EPT、VPID、APIC 虚拟化或 VM functions。用这个模型处理跨 CPU 差异和“不支持但写入了”的入口失败。

### 3. 把 VMCS 当成一次状态转换的完整契约

VMCS 不是一包任意字段，而是六类契约：

- **VM-execution controls**：guest 运行时哪些操作由硬件放行或拦截。
- **VM-exit controls**：退出时如何保存 guest、加载 host。
- **VM-entry controls**：进入时如何加载 guest、注入事件。
- **Guest-state**：即将运行或刚被保存的 guest 上下文。
- **Host-state**：退出后必须可恢复的 VMM 上下文。
- **VM-exit information**：这次退出的原因、限定信息与事件现场。

修改某类字段时，同时检查它依赖的能力位、地址宽度、对齐、保留位以及相关区域。

### 4. 用分层检查解释 VM-entry

按固定层级诊断 VMLAUNCH/VMRESUME：

1. 指令执行环境与 current/launch 状态。
2. VM-execution、VM-exit、VM-entry 控制字段。
3. host-state 合法性。
4. guest-state 合法性。
5. guest 环境和 MSR 加载、事件注入、APIC 状态更新。

区分三种结果：VMfailInvalid、VMfailValid、以及 VM-entry failure 导致的 VM-exit。不要把“进入后立即退出”误报成“指令没有执行”。

### 5. 用退出证据驱动 VM-exit 分派

先读取 `exit reason`，再按原因选择有效的 `exit qualification`、interruption information、IDT-vectoring information、instruction length、guest-linear address 与 GPA。之后才决定推进 RIP、模拟指令、修复映射、反射异常或终止 VM。不要对所有退出套同一个“RIP += instruction length”模板。

### 6. 分开两级地址转换与两类故障

用 `guest-linear → GPA → HPA` 分解地址问题：guest 页表负责第一段，EPT 负责第二段。guest `#PF` 属于 guest 分页语义；EPT violation 表示 EPT 权限拒绝；EPT misconfiguration 表示 EPT 表项格式非法。只有完成分层，才能决定向 guest 反射异常、由 VMM 补映射，还是修正 EPT 表项。

### 7. 用缓存域选择最小失效范围

把转换缓存分为 linear mapping、guest-physical mapping 和 combined mapping，并用 PCID、VPID、EPTP 标识域。修改映射后，按受影响域选择 INVLPG、INVVPID 或 INVEPT；优先地址级或单上下文失效，只有边界无法确定时才扩大范围。避免用全局刷新掩盖域管理错误。

### 8. 把中断虚拟化视为事件连续性问题

VM-exit 可能发生在事件产生、递送或 guest 处理阶段。结合 interruption information、IDT-vectoring information、interruptibility state 和 NMI blocking 判断现场，再决定直接恢复、重新注入、反射新异常、形成 #DF 或处理 triple fault。目标是保持 guest 观察到的事件顺序与 x86 规则一致。

### 9. 按退出成本逐级采用 APIC 硬件辅助

先能正确监控 guest 的 APIC 基址与寄存器访问，再按能力逐步使用 APIC-access page、virtual-APIC page、TPR shadow、EOI bitmap、APIC-register virtualization、virtual-interrupt delivery 和 posted interrupt。每加一层硬件辅助，都先验证 VMCS 依赖字段和软件后备路径。

---

## Chapter Index

| # | Title | Key Frameworks |
|---|---|---|
| [ch01](chapters/ch01-system-platform.md) | 系统平台 | 三阶段启动、PCB/SDA、多处理器初始化 |
| [ch02](chapters/ch02-vmx-architecture.md) | VMX 架构基础 | root/non-root、能力检测、VMX 指令状态 |
| [ch03](chapters/ch03-vmcs-structure.md) | VMCS 结构 | 六类区域、字段编码、执行/入口/退出控制 |
| [ch04](chapters/ch04-vm-entry.md) | VM-entry 处理 | 分层校验、guest 加载、事件注入 |
| [ch05](chapters/ch05-vm-exit.md) | VM-exit 处理 | 退出优先级、信息记录、guest/host 切换 |
| [ch06](chapters/ch06-memory-virtualization.md) | 内存虚拟化 | EPT、两级转换、VPID、缓存失效 |
| [ch07](chapters/ch07-interrupt-virtualization.md) | 中断虚拟化 | 异常恢复、任务切换、Local APIC、posted interrupt |

## Topic Index

- **APIC-access / virtual-APIC** → ch03, ch04, ch07
- **EPT / EPTP / EPTP switching** → ch02, ch03, ch06
- **EPT violation / misconfiguration** → ch03, ch06, ch07
- **Exception bitmap / PFEC mask-match** → ch03, ch07
- **Exit qualification / exit reason** → ch03, ch05, ch06, ch07
- **IDT-vectoring / 事件重注入** → ch03, ch05, ch07
- **INVEPT / INVVPID / INVLPG** → ch02, ch06
- **Local APIC / x2APIC / EOI** → ch03, ch07
- **MSR bitmap / I/O bitmap** → ch03, ch05
- **PCB / SDA / 多处理器启动** → ch01
- **Posted interrupt / virtual-interrupt delivery** → ch03, ch04, ch07
- **TPR shadow / PPR** → ch03, ch04, ch07
- **VMCS 生命周期与字段编码** → ch02, ch03
- **VM-entry failure / VMfailValid** → ch02, ch04
- **VM-exit 分派与优先级** → ch03, ch05
- **VMXON / VMXOFF / VMLAUNCH / VMRESUME** → ch02, ch04
- **VPID / 转换缓存域** → ch02, ch03, ch06
- **guest-linear / GPA / HPA** → ch03, ch06
- **异常反射 / #DF / triple fault** → ch05, ch07

## Supporting Files

- [glossary.md](glossary.md) — 关键术语与章节定位
- [patterns.md](patterns.md) — 配置、入口、退出、内存和中断处理模式
- [cheatsheet.md](cheatsheet.md) — 故障分流、失效范围与事件恢复速查

---

## Scope & Limits

本 skill 只覆盖本书内容，并从扫描版 PDF 的中文 OCR 结果综合生成；少量表格字符、位号或代码标识可能存在识别误差。涉及精确保留位、型号能力或生产实现时，应同时核对目标处理器对应版本的 Intel 架构手册与实际 CPUID/MSR 结果。
