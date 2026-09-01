# 第1章：CPU虚拟化

## Core Idea

CPU 虚拟化的核心不是“模拟一颗 CPU”，而是让 Guest 的绝大多数指令直接在物理 CPU 上执行，同时把会影响资源隔离或虚拟机语义的操作收束为可控的 **VM-exit**。VMX 提供 Root/non-Root 两套运行环境，VMCS 保存切换所需状态与控制策略，KVM 则把一次次 VM-entry/VM-exit 组织成 VCPU 的长期运行循环。

## Frameworks Introduced

- **Popek–Goldberg 三条件**：以等价性、高效性、资源控制判断一种方案是否称得上虚拟化。
  - When to use：评估二进制翻译、半虚拟化、硬件辅助虚拟化或纯模拟方案时。
  - How：先检查 Guest 是否看到近似真实机器；再看大多数指令能否原生执行；最后确认 VMM 是否始终掌握物理资源。
  - Why / failure mode：仅做到接口兼容而大量翻译指令属于模拟；Guest 能绕过 VMM 修改物理资源则隔离失效。

- **Trap and Emulate（陷入和模拟）**：普通指令直接执行，敏感操作陷入 VMM，由 VMM 重现其对虚拟硬件应有的效果。
  - When to use：设计指令、寄存器或设备访问的虚拟化路径时。
  - How：识别操作是否敏感；配置退出条件；读取 VM-exit 信息；在内核或用户空间模拟；推进 Guest RIP；重新 VM-entry。
  - Why / failure mode：早期 x86 存在“敏感但非特权”的指令，Ring Compression 无法保证必然陷入；静态翻译又无法覆盖运行时生成代码与动态副作用。

- **VMX 双模式 + VMCS 上下文契约**：VMM 在 VMX Root Mode，Guest 在 VMX non-Root Mode；两种模式内部都保留 ring 0～3。
  - When to use：理解 Guest kernel 为何能处于自己的 ring 0，同时仍受 Host 控制。
  - How：用 VMCS 的 Guest-state area、Host-state area、VM-exit information fields、VM-execution control fields 分别表达“保存什么、恢复什么、为何退出、何时退出”。首次进入使用 `VMLAUNCH`，后续使用 `VMRESUME`。
  - Why / failure mode：VMCS 不是完整的软件线程上下文；通用寄存器、CR2 等未必由硬件自动保存，若 KVM 未在切换边界补齐，Guest 或 Host 状态会被破坏。

- **VCPU 生命周期状态机**：一个 VCPU 由一个 Host 线程承载，持续执行“用户态准备 → `KVM_RUN` → VM-entry → Guest 执行 → VM-exit → 分层处理 → 再进入”。
  - When to use：定位性能损耗、退出路径或用户态设备模拟问题时。
  - How：先按退出原因判断 KVM 内核能否处理；能处理则直接回 Guest（轻量级 VM-exit）；不能处理则通过 `struct kvm_run` 返回 VMM 用户空间。
  - Why / failure mode：把所有退出都上送用户态会额外产生内核/用户态切换；用户态处理后若不再次调用 `KVM_RUN`，VCPU 不会继续执行。

- **语义虚拟化而非指令照搬**：`cpuid`、`hlt`、CR3 写入或 MMIO 写寄存器的虚拟语义不同于物理语义。
  - When to use：一条指令可在硬件执行，但其结果会泄露 Host、停止物理资源或产生设备副作用时。
  - How：让其 VM-exit，读取输入寄存器/退出信息，写回虚拟结果或改变 VCPU/虚拟设备状态，再跳过已模拟指令。
  - Why / failure mode：直接执行 `cpuid` 会破坏迁移兼容性；直接执行 `hlt` 会停止逻辑物理核而非仅挂起 VCPU 线程；只写 MMIO 值而不执行设备回调会丢失副作用。

## Key Concepts

- **敏感指令**：修改系统资源或在不同特权模式下表现不同、因而必须受 VMM 监控的指令；它不一定是特权指令。
- **Ring Compression**：让 Guest kernel 运行在非 ring 0，以期通过特权异常陷入 VMM；受 x86 敏感非特权指令限制。
- **VM-entry / VM-exit**：物理 CPU 在 VMX Root 与 non-Root 间切换；不仅是控制流跳转，也伴随硬件与软件协作的状态保存/恢复。
- **VMCS**：每个 VCPU 的硬件控制与状态容器；当前物理 CPU 通过 `VMPTRLD` 选择要运行的 VMCS。
- **`KVM_RUN`**：用户空间启动或继续一个 VCPU 的 ioctl；返回时应读取共享的 `struct kvm_run` 及 `exit_reason`。
- **轻量级 VM-exit**：退出后完全在 KVM 内核处理并立即重入 Guest，不返回 VMM 用户空间。
- **MMIO**：把设备寄存器映射到内存地址空间；Guest 对保留地址的访存需要被识别并交给虚拟设备处理。
- **PIO**：使用 `in/out/ins/outs` 等专用 I/O 指令；退出信息包含端口、宽度、方向，以及普通 I/O 的值或 string I/O 的地址。
- **SMP 虚拟化**：每个虚拟 CPU 是一个线程；虚拟 LAPIC 将 Guest 的 IPI 转换为目标 VCPU 状态改变、唤醒与中断注入。

## Mental Models

- **把 VCPU 想成“带硬件加速的协程”**：Guest 连续运行，遇到退出点让出控制权；KVM 处理后从精确的 Guest 上下文继续。
- **把 VMCS 想成“硬件认可的切换协议”**：状态区是断点，控制区是拦截策略，退出信息区是故障单；KVM 仍需管理协议未覆盖的状态。
- **把 VM-exit 当作昂贵的异常边界**：正确性先由退出保证，性能则取决于减少退出次数、缩短 Host 停留时间和尽量留在内核处理。
- **把设备寄存器写入看成命令而非赋值**：值只是输入，真正语义可能是发送 IPI、启动设备或改变中断状态。

## Anti-patterns

- **把“特权指令”与“敏感指令”画等号**：会遗漏不抛异常但改变虚拟语义的操作，也会无谓拦截可安全直通的特权操作。
- **认为 VMCS 会保存所有寄存器**：通用寄存器、CR2 等存在软件管理路径；遗漏会造成上下文串扰。
- **每次切换都无条件写 VMCS/CR2**：昂贵且多数时候值未变化；应先比较，只有脏状态才更新。
- **MMIO 模拟只做 `dst = src`**：未调用设备的 `write_emulated`/设备回调，LAPIC ICR 等寄存器的副作用不会发生。
- **模拟完成后不推进 RIP**：Guest 会重新执行同一条指令，形成退出死循环。
- **把所有 VM-exit 都交给用户空间**：增加一次 Host 内核态到用户态的上下文切换，吞吐和延迟都会恶化。
- **让 Guest 直接看到 Host CPUID**：异构集群中会暴露不可迁移特性；VCPU 能力应由用户空间按迁移基线构造。

## Code Examples

VCPU 主循环的最小骨架：

```c
for (;;) {
    ioctl(vcpu_fd, KVM_RUN, 0);
    switch (run->exit_reason) {
    case KVM_EXIT_IO:   emulate_pio(run); break;
    case KVM_EXIT_MMIO: emulate_mmio(run); break;
    default:            handle_or_stop(run);
    }
}
```

VM-entry 的关键选择：

```text
if (!vcpu.launched) VMLAUNCH
else                VMRESUME
```

模拟指令的必要收尾：

```text
decode → read operands → apply virtual semantics
      → device side effect → advance Guest RIP
```

- **What it demonstrates**：用户空间循环、硬件进入指令与模拟流水线共同构成一个可持续运行的 VCPU，而不是一次性调用。

## Reference Tables

| 维度 | 软件陷入/翻译 | VMX 硬件辅助 |
|---|---|---|
| Guest kernel 特权级 | 通常被压缩到非 ring 0 | non-Root ring 0 |
| 敏感操作捕获 | 异常、二进制翻译 | VM-execution controls + VM-exit |
| 普通指令 | 原生或翻译后执行 | 大多原生执行 |
| 上下文载体 | VMM 软件结构 | VMCS + KVM 软件状态 |
| 主要风险 | 敏感非特权指令漏拦截 | 退出配置错误、软件状态未补齐 |

| VMCS 区域 | 作用 | 典型使用时刻 |
|---|---|---|
| Guest-state area | 保存/恢复 Guest CPU 状态 | VM-exit 保存，VM-entry 装载 |
| Host-state area | 保存/恢复 Host CPU 状态 | VM-entry 保存，VM-exit 装载 |
| VM-exit information | 记录退出原因与辅助信息 | KVM 分派退出处理函数 |
| VM-execution controls | 决定哪些行为触发退出 | 配置 CR3、I/O、APIC 等拦截 |

| 事件 | 最小处理层 | 关键动作 |
|---|---|---|
| 外部中断 | Host/KVM 内核 | 处理 Host 中断后快速重入 |
| `cpuid` | KVM 内核 | 按虚拟 CPUID 表写回寄存器 |
| `hlt` | KVM 内核 | 挂起 VCPU 线程，等待中断唤醒 |
| PIO | 内核或用户态设备 | 从 `kvm_run` 读取端口和值/地址 |
| MMIO | 内核或用户态设备 | 解码指令并调用设备副作用回调 |

## Worked Example

以下用一个只执行 16 位无限循环的 Guest 重构完整 CPU 虚拟化链路：

1. 打开 `/dev/kvm`，获得 KVM 子系统文件描述符；调用 `KVM_CREATE_VM` 得到 VM fd。此时只有虚拟机容器，没有内存和处理器。
2. 用 `mmap` 在 Host 用户空间申请一段页对齐内存，填写 `kvm_userspace_memory_region`：`slot=0`、`guest_phys_addr=0`、`memory_size=ram_size`、`userspace_addr=HVA`，再调用 `KVM_SET_USER_MEMORY_REGION`。这一步建立 GPA 区间与 HVA 区间的关系。
3. 调用 `KVM_CREATE_VCPU` 创建 VCPU fd；读取并设置 `kvm_sregs`，令 `cs.selector=0x1000`、`cs.base=0x10000`；设置 `kvm_regs.rip=0`、`rflags=0x2`。于是实模式入口物理地址为 `cs.base + rip = 0x10000`。
4. 把无格式 Guest 镜像复制到 `ram_start + 0x10000`。镜像只有 `jmp` 回自身，不执行主动敏感指令。
5. 用户空间反复调用 `KVM_RUN`。首次由 KVM 配置 VMCS 并执行 `VMLAUNCH`；随后若因外部中断被动 VM-exit，KVM 内核处理后以 `VMRESUME` 快速返回 Guest。
6. 观察运行统计时，VCPU 线程几乎全部处于 Guest 时间，因为 Guest 没有 PIO、MMIO、`cpuid`、`hlt` 等主动退出点。该结果同时验证了“绝大多数指令直接执行”和“VMM 仍可通过被动退出夺回控制权”。

若把 Guest 循环替换为一次 MMIO 写 LAPIC ICR，流程会扩展为：页面异常/专用 APIC-access 退出 → KVM 解码 `mov` → 以异常地址定位虚拟 LAPIC → 写寄存器 → `apic_send_ipi` 匹配目标 VCPU → 注入或唤醒 → 推进 RIP → 重入 Guest。这里“发送 IPI”才是写寄存器的真实完成条件。

## Key Takeaways

1. CPU 虚拟化的性能基础是让非敏感指令原生执行，安全基础是让敏感语义必然经过 VMM。
2. VMX 提供 Root/non-Root 与 VM-entry/VM-exit，VMCS 提供硬件切换协议；KVM 负责补齐软件状态与退出策略。
3. 一个 VCPU 本质上是长期运行的 Host 线程，`KVM_RUN` 只是其运行循环的用户态入口。
4. 退出处理应尽量停留在 KVM 内核；只有设备模型等内核无法完成的工作才返回用户空间。
5. 指令模拟必须同时处理输入、结果、副作用和 RIP 推进，缺一项都不是完整模拟。
6. CPUID 基线、HLT 线程挂起、虚拟 LAPIC IPI 都说明虚拟化重建的是语义，而非机械复刻物理指令。

## Connects To

- **Ch 2 内存虚拟化**：CR3-load exiting、影子页表和 EPT 决定 Guest 写 CR3 是否需要 VM-exit。
- **设备虚拟化**：MMIO/PIO 退出通过 `struct kvm_run` 或内核设备模型连接到具体设备后端。
- **中断虚拟化**：虚拟 LAPIC、INIT/SIPI、APIC-access 退出与中断注入共同完成多 VCPU 协作。
- **调度与性能分析**：VM-exit 频率、内核/用户态退出比例和 Host 调度延迟共同决定 VCPU 的有效运行时间。
