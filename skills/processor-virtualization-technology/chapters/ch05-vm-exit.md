# Chapter 5: VM-exit 处理

## Core Idea

VM-exit 是处理器从 VMX non-root operation 原子地切换到 VMX root operation 的受控状态转换。VMM 的正确性不只取决于“识别退出原因”，还取决于按体系结构顺序读取退出证据、保存 guest、加载 host，并区分直接事件、事件 delivery 期间的间接退出和 trap 型退出。

## Frameworks Introduced

- **退出触发三分法**：把退出源分为无条件指令、有条件指令和事件。
  - When to use: 配置拦截策略或解释某次 VM-exit 时。
  - How: 先判断指令是否天然退出；再检查 pin/primary/secondary controls、bitmap、mask/shadow；最后检查异常、中断、窗口、EPT、APIC 等事件条件。
- **VM-exit 七阶段状态机**：按硬件实际顺序理解退出，而不是把它看成普通函数调用。
  - How: ①记录 VM-exit information；②更新 VM-entry 控制/注入字段；③更新当前处理器状态；④保存 guest-state；⑤执行 VM-exit MSR-store；⑥加载 host-state 并清地址监控；⑦执行 VM-exit MSR-load。
- **直接/间接证据模型**：直接引发退出的向量事件由 `VM-exit interruption information` 描述；delivery 期间发生故障的原始事件由 `IDT-vectoring information` 描述。
  - Why it works: 它保留“当前故障”与“被中断的原始 delivery”两条因果链，决定后续应反射哪个事件。
- **指令异常优先级模型**：先确定指令本身是否因编码、权限或操作数而产生异常，再判断配置的指令退出。
  - How: 如 `lock cpuid` 先产生 `#UD`；CPL=3 执行 `MOV from CR3` 先产生 `#GP`。只有异常被 exception bitmap 拦截时，才以异常原因 VM-exit，而不是 CPUID/CR3-store 原因退出。

## Key Concepts

- **Exit reason**：VM-exit 的主原因码；VM-entry 失败导致的退出会带失败指示。
- **Exit qualification**：特定退出原因的上下文，如控制寄存器访问、`#PF` 线性地址或 EPT violation 权限细节；并非所有原因都定义它。
- **Guest-linear address / Guest-physical address**：分别记录相关线性地址和 EPT 故障引用的 GPA；是否有效取决于退出原因及 qualification。
- **VM-exit interruption information**：直接引发退出的向量号、类型、valid、error-code 等信息。
- **IDT-vectoring information**：原向量事件在 delivery 期间间接引发退出时的证据。
- **VM-exit instruction length/information**：供 VMM 解码、模拟并在成功模拟后推进 guest RIP；不能对 fault 型语义盲目推进。
- **Trap 型 VM-exit**：TPR below threshold、EOI virtualization、APIC-write、monitor trap flag 等在相关操作完成后退出。
- **VMX-abort**：VM-exit 内部阶段失败的不可恢复路径；记录非零 abort ID，并可能进入只能由 RESET 唤醒的 VMX-abort shutdown。

## Mental Models

- **把 VM-exit 当作提交日志**：先读 `exit reason`，再按原因读取 qualification、地址、interruption/vectoring 和 instruction 字段；未由该原因定义的字段视为不可用。
- **把直接退出当作“事件尚未 delivery”**：例如直接 `#PF` 不更新 CR2，故障线性地址在 qualification；直接 NMI 的 blocking 状态在退出完成后才出现。
- **把间接退出当作“delivery 已开始但未完成”**：CR2、调试状态、NMI blocking 或部分栈/描述符状态可能已更新；`IDT-vectoring information.valid=1` 是关键标志。
- **把 host-state 当作恢复契约**：host CR0/CR4 必须满足 fixed-bit 约束，host RIP/base 必须合法；VM-exit 后 RFLAGS 被置为 `0x2`，因此 IF=0，VMM 要显式决定何时重新开中断。

## Anti-patterns

- **只按退出原因分派，不验证字段有效性**：会读取未定义 qualification 或错误地址字段。
- **每次模拟后都递增 RIP**：异常、间接事件和 trap 型退出并不都应跳过当前指令。
- **把异常原因误判为指令拦截**：忽略 `#UD/#GP` 的优先级会破坏 guest 可观察语义。
- **忽略 `IDT-vectoring information`**：可能丢失原始事件，错误注入嵌套异常。
- **把 VMX-abort 当作普通 VM-exit 恢复**：MSR 列表、PDPTE 或 host-state 加载错误可能已使恢复路径失效。

## Reference Tables

| 证据字段 | 何时读取 | 主要用途 |
|---|---|---|
| Exit reason | 每次 VM-exit 首先读取 | 选择处理器与后续字段集合 |
| Exit qualification | 仅对该 reason 有定义时 | 解析访问类型、寄存器、权限或故障细节 |
| VM-exit interruption information | 直接向量事件退出 | 取得当前向量及错误码属性 |
| IDT-vectoring information | `valid=1` | 恢复 delivery 中的原始向量事件 |
| Instruction length/information | 指令类退出 | 解码、模拟、决定是否推进 RIP |
| Guest-linear / physical address | 原因明确提供时 | 定位内存、EPT 或 APIC 访问 |

```text
reason = vmread(EXIT_REASON)
evidence = read_only_fields_defined_for(reason)
if idt_vectoring.valid:
    preserve_original_delivery(evidence)
result = dispatch(reason, evidence)
if result == EMULATED_AND_COMPLETED:
    guest_rip += exit_instruction_length
vmresume()
```

## Key Takeaways

1. VM-exit 处理必须遵循“原因 → 原因限定字段 → 直接/间接事件 → 模拟/反射/恢复”的证据顺序。
2. 直接退出与 delivery 期间的间接退出会保存不同的处理器状态，不能共用无条件恢复模板。
3. 指令异常优先于某些指令拦截条件；exception bitmap 决定异常是否再转化为 VM-exit。
4. VPID 开启时 VM-entry/VM-exit 可保留相关转换缓存；关闭时默认 VPID 0000H 的线性与 combined mapping 会被刷新。
5. MSR-store、host PDPTE、MSR-load 和 host-state 都是退出可靠性边界，错误可升级为 VMX-abort。

## Connects To

- **Ch 4**：VM-entry 的检查、事件注入和 guest-state 恢复与本章状态机互为逆向路径。
- **Ch 6**：EPT violation/misconfiguration 的 qualification、GPA 与缓存失效规则决定内存退出处理。
- **Ch 7**：异常反射、NMI unblocking、APIC trap 型退出和中断注入建立在本章证据模型上。
