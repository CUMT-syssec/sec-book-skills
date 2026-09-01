# 第2章：VMX架构基础

## Core Idea

VMX 不是“打开一个开关”即可使用，而是一套由处理器能力、控制寄存器约束、内存区域、VMCS 生命周期、VM-entry/VM-exit 状态转换和指令失败语义共同限定的协议。可靠的 VMM 应先探测、再合成合法配置，最后按状态机顺序执行 VMX 指令。

## Frameworks Introduced

- **root / non-root 双环境模型**
  - When to use：区分 VMM 的物理资源控制权与 guest 的受控执行环境时。
  - How：VMM 运行于 VMX root operation；VM 运行于 VMX non-root operation。`VMLAUNCH`/`VMRESUME` 发起 VM-entry，退出条件或无条件事件触发 VM-exit，VMM 处理退出原因、虚拟化资源后再次进入 guest。
- **VMXON + VMCS 两层状态容器**
  - When to use：建立每 CPU 的 VMM 环境并管理 VM 时。
  - How：每个逻辑处理器准备 VMXON region，每个虚拟处理器准备独立 VMCS；区域使用物理地址、4KB 对齐，首 DWORD 写入 `IA32_VMX_BASIC` 给出的 revision ID。
- **能力驱动配置框架**
  - When to use：设置控制字段、EPT/VPID/VMFUNC 或 CR0/CR4 时。
  - How：先用 CPUID 确认 VMX，再按依赖读取能力 MSR；`IA32_VMX_BASIC[55]=1` 时相应字段采用 TRUE MSR。功能只有在 allowed-1 位允许时才能启用。
- **合法值合成规则**
  - When to use：把期望功能转换为合法控制字段或 CR0/CR4 时。
  - How：`result = (desired | must_be_1) & may_be_1`。控制 MSR 低/高 32 位分别给出 allowed-0/allowed-1；CR0/CR4 用 FIXED0/FIXED1 表达同类约束。
- **VMX 指令三态检查**
  - When to use：执行可能失败的 VMX 指令后。
  - How：CF=ZF=0 为 VMsuccess；CF=1 为 VMfailInvalid；ZF=1 为可读取错误字段的 VMfailValid。`#UD/#GP` 是独立的异常通道。

## Key Concepts

- **Intel VT-x / VMX**：CPU 域虚拟化，与设备侧 VT-d、VT-c 分层。
- **VMM / VM**：VMM 监管资源与退出处理；VM 是由 guest 软件占用的隔离执行实例。
- **VMX operation**：由 `VMXON` 进入、`VMXOFF` 退出的新处理器操作模式，内部包含 root 与 non-root。
- **VMXON region**：逻辑处理器进入 VMX operation 时使用的 VMM 级区域；操作数必须是物理指针。
- **VMCS**：描述虚拟处理器控制与 host/guest state 的结构；每逻辑处理器同时仅一个 current-VMCS。
- **VM-entry / VM-exit**：root→non-root 与 non-root→root 的控制权切换；首次 entry 使用 `VMLAUNCH`，退出后恢复使用 `VMRESUME`。
- **allowed 0-setting / allowed 1-setting**：控制字段每个位允许取 0/1 的能力约束。
- **unrestricted guest**：允许 non-root guest 运行实模式或未分页保护模式的可选能力。
- **EPT / VPID**：分别标识 GPA→HPA 与线性转换上下文，决定 INVEPT/INVVPID 的刷新域。

## Mental Models

- 把 VMX 当成**能力协商协议**：代码表达期望，MSR 表达边界，合成后才是合法配置。
- 把 VM-entry 看成**分阶段提交**：先查指令/指针/launch 状态，再查控制与 host state，最后查 guest state；阶段决定失败路径。
- 把缓存刷新看成**按地址空间标签失效**：INVEPT 面向 EPTP/EP4TA，INVVPID 面向 VPID 与可选线性地址，不能互相替代。

## Anti-patterns

- **只检查 CPUID 就执行 VMXON**：还必须处理 `IA32_FEATURE_CONTROL`、CR0/CR4、A20M、CPL、运行模式和 VMXON region。
- **硬编码控制字段**：处理器与虚拟机暴露能力会不同；未经 allowed-0/1 合成可能导致 VM-entry 失败。
- **向 VMX 指令传虚拟地址**：区域操作数要求物理地址，并满足对齐、地址宽度与 revision ID 检查。
- **只看 CF**：VMX 指令还可能以 ZF 指示 VMfailValid，或通过 `#UD/#GP` 失败；三类结果必须分开诊断。
- **无条件执行刷新类型**：先从 `IA32_VMX_EPT_VPID_CAP` 检测 type；INVVPID 也不能以 VPID=0 为目标。

## Code Examples

从本章规则重构出的最小配置流程：

```text
require CPUID.VMX == 1
configure IA32_FEATURE_CONTROL before it is locked
CR0 := (CR0 | IA32_VMX_CR0_FIXED0) & IA32_VMX_CR0_FIXED1
CR4 := (CR4 | IA32_VMX_CR4_FIXED0) & IA32_VMX_CR4_FIXED1
CR4.VMXE := 1

vmxon_region.revision_id := IA32_VMX_BASIC.revision_id
VMXON physical_address(vmxon_region)
require CF == 0 && ZF == 0

control := (desired | allowed0_low) & allowed1_high
```

- **What it demonstrates**：把环境前置条件、区域初始化、指令结果和能力字段合成放在同一条可审计链上。

## Reference Tables

| 目标 | 指令序列/条件 | 关键状态 |
|---|---|---|
| 进入 VMX operation | 设置许可与固定位 → 初始化 VMXON → `VMXON` | 进入 root；current-VMCS 初始无效 |
| 首次进入 VM | `VMCLEAR` → `VMPTRLD` → `VMWRITE` 配置 → `VMLAUNCH` | VMCS 必须为 clear |
| VM-exit 后恢复 | 处理退出原因/更新 VMCS → `VMRESUME` | VMCS 必须为 launched |
| 退出 VMX operation | 满足双重监控约束 → `VMXOFF` → 清 CR4.VMXE | 返回普通模式 |

## Key Takeaways

1. VMX 初始化顺序是：探测能力 → 满足约束 → 初始化物理区域 → 执行指令 → 检查 CF/ZF/异常。
2. VMXON region 属于逻辑处理器，VMCS 属于虚拟处理器；二者不能做成全局单例。
3. 控制字段应由期望值与 capability MSR 合成，不能把单机结果当常量。
4. VM-entry 失败要按阶段定位：基本/host 检查失败返回下一指令，guest-state 失败走 VM-exit 路径。
5. VMLAUNCH/VMRESUME 由 VMCS launch 状态选择；EPT/VPID 失效也必须使用受支持的域与 type。

## Connects To

- **Ch 1**：Stage 2/3 的分页、PCB 私有状态和每 CPU VMX 初始化提供执行 VMXON 的系统条件。
- **Ch 3**：本章只建立 VMCS 的角色与指令接口；字段编码、current/activity/launch 属性和各状态区在下一章展开。
- **Ch 4**：VM-entry 的控制字段、host-state、guest-state 检查规则决定具体的进入成功条件。
- **Ch 6**：EPT、VPID、INVEPT、INVVPID 的能力位和刷新域将在内存虚拟化中具体应用。
- **Ch 7**：external-interrupt exiting、事件注入和 VM-exit 路径构成中断虚拟化的控制基础。
