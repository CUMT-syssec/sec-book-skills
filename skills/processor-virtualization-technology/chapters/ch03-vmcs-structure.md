# 第 3 章：VMCS 结构

## Core Idea

VMCS（Virtual Machine Control Structure）是一次 VM-entry/VM-exit 的控制面与状态交换面：处理器不承诺其内部字段的物理布局，VMM 必须通过字段编码和 VMX 指令访问它，并同时维护 VMCS 的状态机、控制依赖与 guest/host 上下文。

## Frameworks Introduced

- **VMCS 三属性状态机**：分别跟踪 `activity`（active/inactive）、`current`（current/not current）和 `launch`（clear/launched）。
  - When to use：选择 `VMCLEAR`、`VMPTRLD`、`VMLAUNCH` 或 `VMRESUME` 前。
  - How：`VMCLEAR` → inactive、not current、clear；`VMPTRLD` → active、current；成功 `VMLAUNCH` → launched。`VMLAUNCH` 要求 clear，`VMRESUME` 要求 launched。
- **六区域职责分离**：用“谁提供状态、谁控制切换、谁记录结果”组织 VMCS。
  - When to use：初始化 VMCS、定位 VM-entry/VM-exit 配置错误或解释某字段为何存在。
  - How：先区分状态区、控制区和信息区，再依据字段宽度与类型编码访问。
- **编码访问模型**：把字段 ID 当成稳定接口，而不是内存偏移。
  - When to use：实现 VMCS buffer、封装 `VMREAD`/`VMWRITE`、适配不同处理器实现。
  - How：解析 access type、index、field type 和 width；再按当前执行模式处理 full/high 访问。
- **能力位约束模型**：控制字段不是任意位图，必须按 VMX capability MSR 的 allowed-0/allowed-1 约束设置，并满足控制位之间的前置依赖。

## Key Concepts

- **current-VMCS**：一个逻辑处理器同一时刻唯一的当前 VMCS；`VMREAD`、`VMWRITE`、`VMLAUNCH`、`VMRESUME` 隐式作用于它。
- **current-VMCS pointer**：由处理器维护；清除当前 VMCS 后变为 `FFFFFFFF_FFFFFFFFh`，后续隐式访问会 `VMfailInvalid`。
- **VMCS region**：4 KiB 对齐的物理区域；大小取自 `IA32_VMX_BASIC[44:32]`，上限 4 KiB；缓存类型能力取自 `[53:50]`。
- **VMCS header**：前 8 字节包含 VMCS revision identifier 与 VMX-abort indicator；revision identifier 必须匹配 `IA32_VMX_BASIC[31:0]`。
- **VMX-abort**：VM-exit 内部严重错误路径；处理器写入非零 abort ID，并进入 shutdown。
- **natural-width**：在支持 64 位架构的处理器上为 64 位，否则为 32 位；只有 full 编码，没有 high 编码。
- **VM-exit information**：只读地记录退出原因、qualification、向量事件、指令信息，以及 VMX 指令的有效失败码。

## Mental Models

- **把 VMCS 看成硬件协议对象**：软件拥有字段语义，不拥有字段物理偏移；跨代兼容依赖字段 ID 和 capability MSR。
- **把六区域看成双向检查点**：VM-entry 从 guest-state 加载，VM-exit 把 guest 当前状态写回并从 host-state 恢复；三组控制区规定沿途动作，信息区解释退出结果。
- **把控制位看成依赖图**：例如 `unrestricted guest` 依赖 EPT；`virtual-NMIs` 依赖 `NMI exiting`；secondary controls 仅在 `activate secondary controls=1` 时生效。
- **把 MSR 列表看成批量 WRMSR 事务**：地址、对齐、数量、索引合法性和每个 16 字节表项都属于协议的一部分。

## Anti-patterns

- **用 `MOV` 直接读写 VMCS region**：字段位置是实现相关的，可能覆盖不可见或错误字段并产生不可预测行为。
- **混淆 active、current 与 launched**：三类属性相互独立；“已加载过”不等于“当前”，也不等于“可用 `VMRESUME`”。
- **在另一逻辑处理器直接加载 active VMCS**：迁移前应先 `VMCLEAR`，使其 inactive，再在目标处理器 `VMPTRLD`。
- **在 32 位模式先写 high 再写 full**：写 full 会写低 32 位并清高 32 位，顺序错误会丢失高半部。
- **硬编码控制位或保留位**：必须从相应 VMX capability MSR 推导允许值；忽略依赖会在 VM-entry 检查阶段失败。
- **把 VM-exit 信息当普通可写状态**：多数信息字段由处理器产生，用于诊断而非驱动 guest 配置。

## Code Examples

```text
write_vmcs_u64(field_full, field_high, value):
    if running_in_64_bit_mode:
        VMWRITE field_full, value
    else:
        VMWRITE field_full, low32(value)
        VMWRITE field_high, high32(value)
```

- **What it demonstrates**：固定 64 位字段在 32 位模式必须按 full→high 顺序写；natural-width 字段不存在 high 编码。

## Reference Tables

### VMCS 六个数据区域

| 区域 | VM-entry/VM-exit 职责 |
|---|---|
| guest-state | entry 时加载 guest；exit 时保存 guest 当前状态 |
| host-state | exit 时恢复 VMM/host 环境 |
| VM-execution controls | 决定 non-root 执行行为及触发 VM-exit 的条件 |
| VM-exit controls | 决定退出处理、保存/加载动作及 host 返回环境 |
| VM-entry controls | 决定进入处理、guest 模式、MSR 加载和事件注入 |
| VM-exit information | 记录退出原因、qualification、事件与指令明细、VMX 指令错误 |

### 字段 ID 编码

| 位域 | 含义 | 取值 |
|---|---|---|
| bit 0 | access type | `0=full`；`1=high`（固定 64 位字段高 32 位） |
| bits 9:1 | index | 同一类别中的字段索引 |
| bits 11:10 | field type | `0=control`，`1=read-only`，`2=guest-state`，`3=host-state` |
| bits 14:13 | width | `0=16`，`1=64`，`2=32`，`3=natural-width` |

### 宽度访问规则

| 字段宽度 | 32 位模式 | 64 位模式 |
|---|---|---|
| 16/32 位 | 读取时高位清零；写入只取有效低位 | 同左，忽略源操作数高位 |
| 固定 64 位 | full 访问低 32 位，high 访问高 32 位；完整写入必须 full→high | full 直接访问完整 64 位；high 仍只访问高半部 |
| natural-width | 在 64 位处理器的 32 位模式只访问低 32 位，写入会清高半部 | 直接访问完整 64 位 |

## Key Takeaways

1. VMCS 是处理器定义的接口对象，不是可按偏移解析的 C 结构体。
2. 发起 entry 前先验证 VMCS 状态：首次进入用 clear+`VMLAUNCH`，退出后恢复用 launched+`VMRESUME`。
3. 六区域分别承载 guest/host 状态、三类控制与退出信息；定位问题时先判断责任区域。
4. 字段 ID 同时编码访问半部、索引、类别和宽度；固定 64 位字段与 natural-width 字段不可混为一谈。
5. 控制字段必须同时满足 capability MSR 和跨字段依赖，保留位也属于可验证契约。
6. VM-entry 的 MSR-load 列表每项 16 字节，适合批量恢复状态，但必须严格校验地址、数量与索引。

## Connects To

- **第 2 章**：VMX operation、VMX 指令失败语义与 capability MSR 是 VMCS 合法配置的前提。
- **第 4 章**：VM-entry 按阶段检查本章定义的控制区、host-state、guest-state、MSR 列表和事件注入字段。
- **第 5 章**：VM-exit 使用 host-state 恢复 VMM，并在 VM-exit information 区域留下诊断证据。
- **第 6、7 章**：EPT/VPID 与 APIC 虚拟化依赖本章的 execution-control 字段和地址类字段。
