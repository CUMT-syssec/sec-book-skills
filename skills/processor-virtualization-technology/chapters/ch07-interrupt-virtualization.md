# Chapter 7: 中断虚拟化

## Core Idea

中断虚拟化的核心是保持事件归属与 delivery 状态：VMM 可以拦截异常、NMI、外部中断和 APIC 访问，但 guest 自身产生的事件通常仍须被正确反射。VMX 的 virtual-APIC、virtual-interrupt delivery 与 posted-interrupt 把高频 APIC 状态变更和中断提交下沉到硬件，以减少 VM-exit。

## Frameworks Introduced

- **异常归因决策**：先判断异常由 guest 自身条件还是 VMM 人为条件引发。
  - Guest 原因：反射原异常；VMM 条件：撤销/修复条件后恢复，必要时恢复原始 vectoring 事件。
  - How: 同时检查 `VM-exit interruption information` 与 `IDT-vectoring information.valid`；delivery 中的故障不能只看当前退出异常。
- **异常升级状态机**：嵌套异常按体系结构组合为 `#DF`；`#DF` delivery 再失败可形成 triple fault。
  - Decision rule: 需要合成为 `#DF` 时注入向量 8、硬件异常、error-code valid，错误码为 0；triple fault 后不再注入，转 shutdown 或终止 VM。
- **APIC 虚拟化能力栈**：从访问拦截逐层提升到硬件状态模拟。
  - xAPIC：APIC-access page；x2APIC：virtualize x2APIC mode；两者不能同时开启。
  - `use TPR shadow` 提供 VTPR；`APIC-register virtualization` 提供更多影子寄存器；`virtual-interrupt delivery` 负责仲裁与 guest-IDT delivery；posted-interrupt 负责免退出接收通知中断。
- **虚拟中断状态机**：VIRR/RVI 表示请求，VISR/SVI 表示服务中，VTPR/VPPR 表示优先级门限。
  - Evaluate: `interrupt-window exiting=0` 且 `RVI[7:4] > VPPR[7:4]` 才组织 pending；IF=1 且无 STI/MOV-SS 阻塞才 delivery。
- **posted-interrupt 七步流水线**：比较通知向量→清 ON→local APIC EOI→PIR OR 到 VIRR 并清 PIR→更新 RVI→评估→delivery。
  - Why it works: 通知中断本身不进入 guest handler；它只把并发张贴的 PIR 批量转入 virtual-APIC 请求队列。

## Key Concepts

- **Exception bitmap**：按向量决定异常是否导致 VM-exit。
- **NMI unblocking**：异常发生在 IRET 时可能记录解除 NMI blocking；反射时 VM-entry interruption information 的保留位必须清零，并按场景恢复 interruptibility state。
- **APIC-access page**：xAPIC 内存映射访问的受控入口；线性访问可被重定向至 virtual-APIC page，GPA/物理访问语义不同。
- **Virtual-APIC page**：4K 影子页，承载 VTPR、VEOI、VICR、VIRR、VISR 等虚拟寄存器。
- **TPR/PPR/EOI/Self-IPI virtualization**：更新虚拟优先级、结束 in-service 中断、产生本地虚拟中断，并按条件触发评估或 trap 型 VM-exit。
- **RVI / SVI**：guest interrupt status 中的 requesting/servicing virtual interrupt 向量。
- **PIR / ON**：posted-interrupt descriptor 的 256 位请求集合与 outstanding notification 位；运行中的目标 VM 可由其他处理器用 locked RMW 更新。
- **Acknowledge interrupt on exit**：为1时退出时取得外部中断向量；为0时请求留在控制器，VMM 开 IF 后再接收。

## Mental Models

- **把异常反射看作“恢复 guest 应看到的历史”**：不是简单复制当前异常；若原事件 delivery 被中断，需恢复原事件或按组合规则生成 `#DF`。
- **把 APIC-access page 当作入口、virtual-APIC page 当作状态**：前者匹配 guest 的访问地址，后者保存虚拟寄存器；不要把二者当成同一物理页面语义。
- **把虚拟中断看作队列加优先级门**：VIRR/PIR 是队列，RVI 是队首候选，VPPR 是门限，interruptibility/activity 是最终闸门。
- **把 posted interrupt 看作数据面快路径**：普通外部中断仍可能退出；只有 physical vector 等于 notification vector 时进入免退出搬运流程。

## Anti-patterns

- **吞掉 guest 自身异常**：会破坏 guest OS 的错误处理与可观察状态。
- **直接复制 interruption 字段但不清 NMI-unblocking 位**：VM-entry 保留位非法可导致进入失败。
- **嵌套异常时只注入当前异常**：可能应合成为 `#DF`，或在 triple fault 时根本不应注入。
- **同时启用 xAPIC 与 x2APIC 原生虚拟化**：控制组合不合法。
- **只改 RVI/VIRR 就期待立即 delivery**：评估只由 VM-entry、TPR/EOI/Self-IPI 或 posted-interrupt 等规定入口触发。
- **并发修改 posted-interrupt descriptor 不使用 locked RMW**：会丢 PIR 请求或破坏 ON 协议。

## Reference Tables

| 场景 | VMM 决策 |
|---|---|
| guest 自身异常直接退出 | 反射 interruption 信息和错误码 |
| 原事件 delivery 期间发生可组合异常 | 注入合成后的 `#DF` |
| triple fault | 不注入；shutdown 或终止 VM |
| VMM 人为条件导致异常 | 修复条件，恢复当前指令或原始 vectoring 事件 |
| 外部中断需反射 guest | `acknowledge interrupt on exit=1`，取得向量后注入 |
| 通知向量匹配 posted interrupt | 不 VM-exit；PIR→VIRR，更新 RVI 并评估 |

```text
if physical_vector != notification_vector:
    vmexit(EXTERNAL_INTERRUPT)
else:
    atomic_clear(PI_desc.ON)
    local_apic_eoi()
    pending = atomic_take(PI_desc.PIR)
    VIRR |= pending
    if pending is not empty:
        RVI = max(RVI, highest_vector(pending))
    evaluate_virtual_interrupt()
```

## Key Takeaways

1. 异常处理先判归因，再判直接/间接 delivery；反射目标可能是当前异常、原事件或合成 `#DF`。
2. virtual-APIC 的请求、服务和优先级状态必须作为一个状态机维护，不能零散更新。
3. 虚拟中断只有通过规定入口触发评估，并同时满足优先级、IF、interruptibility 与 activity 条件才 delivery。
4. posted interrupt 用通知向量把 PIR 批量转入 VIRR；不匹配的外部中断仍按 `external-interrupt exiting` 退出。
5. NMI 与外部中断的 blocking、acknowledge 和反射规则不同，不能共享普通 maskable interrupt 模板。

## Connects To

- **Ch 5**：interruption/vectoring 信息、trap 型退出和 NMI 状态保存定义了本章的证据边界。
- **Ch 6**：APIC-access page 的 EPT 映射与缓存失效影响 APIC 访问是否真正触发虚拟化路径。
- **Guest IDT / Local APIC**：硬件虚拟化减少退出，但最终 delivery 仍必须保持 x86 异常与中断语义。
