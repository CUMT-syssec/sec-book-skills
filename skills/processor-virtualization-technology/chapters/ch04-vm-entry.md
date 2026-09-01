# 第 4 章：VM-entry 处理

## Core Idea

VM-entry 不是一次简单的上下文恢复，而是一条分阶段、可分类失败的硬件事务：先验证指令和 VMCS 状态，再验证控制区与可恢复的 host 环境，最后验证并加载 guest 状态、MSR 和注入事件；只有全部通过，guest 的第一条指令才可能执行。

## Frameworks Introduced

- **三阶段 VM-entry 验证流水线**：
  - When to use：定位 `VMLAUNCH`/`VMRESUME` 失败以及设计 VMCS 初始化顺序。
  - How：①指令基本检查；②VM-execution、VM-exit、VM-entry controls 与 host-state 检查；③guest-state 检查和 MSR 加载。
- **三类失败出口**：
  - 异常：非法模式或权限产生 `#UD`/`#GP`。
  - `VMfailInvalid`/`VMfailValid`：不进入 guest，继续执行 VMX 指令的下一条指令；分别以 CF/ZF 标识。
  - VM-entry failure VM-exit：guest-state 或 MSR 加载失败，转入 host-RIP，并在 exit reason 中置 bit 31。
- **“先证明能回来”原则**：控制区与 host-state 在 guest-state 前检查，保证后续 guest 检查或加载失败时能安全恢复 host。
- **向量化 VM-entry**：把中断、异常或 pending MTF 事件安排在 guest 第 1 条指令之前 delivery。

## Key Concepts

- **`VMLAUNCH`**：用于 clear 的 current-VMCS；成功后 VMCS 变为 launched。
- **`VMRESUME`**：用于 launched 的 current-VMCS；典型场景是 VM-exit 后恢复 guest。
- **`blocking by MOV-SS`**：若发起 entry 时存在，会产生 `VMfailValid`。
- **invalid guest state**：第 3 阶段验证失败，exit reason 基本码 33，bit 31=1。
- **MSR-load failure**：guest 状态加载阶段失败，exit reason 基本码 34，bit 31=1；qualification 可给出失败表项序号。
- **vectorized VM-entry**：`VM-entry interruption information.valid=1` 的进入；事件经 guest IDT/IVT delivery。
- **pending debug exception**：注入事件完成后评估；可触发 `#DB` 或相应 VM-exit。
- **direct post-entry VM-exit**：进入完成但 guest 第 1 条指令前立即退出的事件族。

## Mental Models

- **把 VM-entry 看成提交协议**：阶段 1/2 失败时尚未提交 guest；阶段 3 的某些失败通过特殊 VM-exit 回滚到已验证的 host 环境。
- **把控制字段看成约束图，而非开关集合**：例如 secondary controls 依赖 activation 位，`unrestricted guest` 依赖 EPT，posted interrupts 又依赖 virtual-interrupt delivery 与 exit acknowledge。
- **把 guest 模式看成交叉约束**：`IA-32e mode guest`、`unrestricted guest`、CR0/CR4、RFLAGS.VM 与段属性共同决定实模式、legacy、virtual-8086、compatibility 或 64 位模式。
- **把事件注入看成“第一条指令前发生的架构事件”**：fault 类返回到当前 RIP；软件中断/软件异常等 trap 类用 instruction length 修正返回 RIP。

## Anti-patterns

- **只检查 `VMLAUNCH`/`VMRESUME` 的一个标志**：必须区分 CF=`VMfailInvalid` 与 ZF=`VMfailValid`，有效失败还要读取 VM-instruction error。
- **把所有失败都当普通 VM-exit**：指令异常、VMfail 与 VM-entry failure VM-exit 的控制流、状态更新和诊断字段不同。
- **先调 guest-state，忽略 host-state**：host 字段不合法会让处理器无法安全处理后续 entry 失败。
- **独立设置 execution controls**：地址对齐、MAXPHYADDR、EPTP 格式、VPID 非零、APIC/posted-interrupt 依赖必须一起验证。
- **把 MSR-load 列表当无条件 WRMSR**：保留 MSR、x2APIC 范围、FS/GS base、SMM-only MSR 或会引发 `#GP` 的值都可能导致原因码 34。
- **对软件型注入事件省略 instruction length**：类型 4、5、6 要求长度 1–15，否则 `VMfailValid`，且 trap 返回地址会错误。
- **认为注入事件满足 exiting 条件就必然直接退出**：普通注入先 delivery；只有 delivery 中的次生条件或注入 pending MTF 等路径才退出。

## Code Examples

```text
VMCLEAR vmcs_pa
VMPTRLD vmcs_pa
VMWRITE ...                 # controls, host-state, guest-state
VMLAUNCH                    # first entry; later use VMRESUME
if CF == 1: handle_vmfail_invalid()
if ZF == 1: handle_vmfail_valid(VM_INSTRUCTION_ERROR)
```

- **What it demonstrates**：VMCS 状态转换、首次/后续进入的指令选择，以及两种 VMfail 的分流。

## Reference Tables

### 检查阶段与失败形态

| 阶段 | 主要检查 | 失败结果 |
|---|---|---|
| 1. 指令基本检查 | VMX 模式、CPL=0、current-VMCS、MOV-SS 阻塞、clear/launched 状态 | `#UD`/`#GP`、`VMfailInvalid` 或 `VMfailValid` |
| 2. 控制区与 host-state | capability 位、跨字段依赖、地址/对齐、host CR/RIP/selector/base/MSR | `VMfailValid`，继续执行下一条指令 |
| 3a. guest-state | CR、RIP/RFLAGS、段、GDTR/IDTR、MSR、activity/interruptibility、VMCS link、PDPTE | VM-entry failure VM-exit，原因码 33 |
| 3b. 状态与 MSR 加载 | guest 寄存器、PDPTE、cache/APIC 状态、MSR-load 列表 | MSR 失败时原因码 34 |
| 完成后 | 注入事件、pending #DB、直接退出条件 | delivery guest 事件或在第 1 条指令前 VM-exit |

### 高频控制依赖

| 设置 | 必须同时满足 |
|---|---|
| secondary controls 有效 | `activate secondary controls=1` |
| `unrestricted guest=1` | `enable EPT=1`，EPTP 合法 |
| `virtual-NMIs=1` | `NMI exiting=1` |
| `process posted-interrupts=1` | virtual-interrupt delivery、external-interrupt exiting、acknowledge interrupt on exit；descriptor 64B 对齐 |
| `enable VPID=1` | VPID 非 0 |
| MSR 列表 count 非 0 | address 16B 对齐；`address + count×16 - 1` 不越过 MAXPHYADDR |

### 事件注入判定

| 类型 bits 10:8 | 事件 | 关键约束 |
|---|---|---|
| 0 | external interrupt | guest RFLAGS.IF 必须允许；通常使用 32–255 向量 |
| 2 | NMI | vector=2 |
| 3 | hardware exception | vector≤31；仅 #DF/#TS/#NP/#SS/#GP/#PF/#AC 携带错误码 |
| 4/5/6 | software interrupt / privileged software exception / software exception | instruction length 必须为 1–15；返回 RIP 加该长度 |
| 7 | other event | vector=0；注入 pending MTF VM-exit |

### guest 第 1 条指令前的直接 VM-exit 优先级

`TPR below threshold > pending MTF > pending debug exception > VMX-preemption timer > NMI-window exiting > interrupt-window exiting`

## Key Takeaways

1. 首次 entry 用 `VMLAUNCH`，恢复 launched VMCS 用 `VMRESUME`；选择错误会 `VMfailValid`。
2. 先按失败形态分诊，再查字段：异常、VMfail、原因码 33 和原因码 34 不能混为一类。
3. 第二阶段先验证 host 可恢复性；第三阶段的 guest/MSR 失败才可安全转到 host-RIP。
4. guest-state 合法性取决于运行模式的交叉约束，尤其是 CR0/CR4、EFER、RFLAGS 和段属性。
5. 加载顺序会影响最终值：MSR-load 列表可覆盖此前从 guest-state 字段加载的同名 MSR。
6. 注入事件发生在 guest 第 1 条指令前；软件型 trap 事件需要 instruction length 修正返回点。
7. entry 完成不保证执行 guest 指令；多个直接退出条件按固定优先级仲裁。

## Connects To

- **第 3 章**：本章所有检查、加载与注入都以 VMCS 六区域及字段编码为输入。
- **第 5 章**：原因码 33/34、exit qualification 与向量化信息通过 VM-exit 信息区交给 VMM。
- **第 6 章**：EPT、VPID、PDPTE 与 TLB/paging-structure cache 处理决定地址转换环境。
- **第 7 章**：virtual-APIC、RVI/SVI、PPR、posted interrupts 与窗口退出在 entry 尾部参与仲裁。
