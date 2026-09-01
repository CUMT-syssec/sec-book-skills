# 处理器虚拟化决策速查

## 先定位问题层

| 现象 | 先查 | 下一步 |
|---|---|---|
| VMXON 失败 | CPUID、IA32_FEATURE_CONTROL、CR0/CR4 fixed bits、CR4.VMXE | 检查 VMXON region 对齐与 revision identifier |
| VMLAUNCH/VMRESUME 返回失败 | VMfailInvalid/VMfailValid、VM-instruction error | 按 VM-entry 分层验证 |
| 进入后立即退出 | exit reason、entry-failure 位、注入字段 | 区分入口失败、待处理事件和正常退出 |
| guest 地址无法解析 | guest 分页模式、CR3、PTE 权限 | 若已得到 GPA，再检查 EPT |
| EPT 类退出 | violation 与 misconfiguration | 前者看权限/映射，后者修表项格式 |
| 中断丢失或重复 | interruption information、IDT-vectoring、interruptibility | 检查 NMI/STI/MOV SS 阻塞及重注入 |

## 控制字段决策

- 若处理器能力 MSR 不允许某位为 1，就不要通过“试运行”判断；先降级功能。
- 若 primary controls 未启用 secondary controls，EPT、VPID 与 APIC 二级控制都不会生效。
- 若某访问只需对少数地址或 MSR 退出，优先 bitmap/target 机制，不要打开无条件退出。
- 若目标是减少 Local APIC 退出，按 `访问捕获 → 寄存器虚拟化 → 虚拟中断递送 → posted interrupt` 逐级启用并逐级验证。

## VM-entry / VM-exit 顺序

```text
VMLAUNCH / VMRESUME
  → 指令条件
  → 控制字段与 host-state
  → guest-state
  → 加载 guest / 注入事件
  → VMX non-root
  → 退出条件或事件
  → 记录退出信息
  → 保存 guest / 加载 host
  → VMX root handler
```

不要把三种结果混为一谈：

- 指令直接失败：尚未进入 guest。
- VM-entry failure 导致退出：入口检查或加载阶段失败。
- 正常 VM-exit：guest 已运行，按 exit reason 分派。

## 地址与失效选择

| 改动对象 | 主要缓存域 | 优先工具 |
|---|---|---|
| 当前 guest 单页映射 | linear mapping | INVLPG 或定址 INVVPID |
| 某 VPID 的 guest 转换 | linear/combined mapping | 单上下文 INVVPID |
| 单个 EPTP 的映射 | guest-physical/combined mapping | 单上下文 INVEPT |
| 无法确定影响范围 | 多上下文 | 扩大 INVVPID/INVEPT 作用域 |

规则：先标识 PCID、VPID、EPTP 和地址，再选择最小安全失效范围；不要默认全局刷新。

## EPT 分流

- `violation`：表项格式可用，但当前读/写/执行权限不满足；检查 qualification 后修复映射或保留拒绝。
- `misconfiguration`：表项自身非法；不要按缺页处理，先修正保留位、层级、内存类型与权限组合。
- guest `#PF` 与 EPT 故障属于不同层：前者应由 guest 语义处理，后者由 VMM/EPT 管理处理。

## 事件恢复规则

- 若 IDT-vectoring valid，说明退出打断了事件递送；先恢复原事件语义，再处理新退出原因。
- 若注入条件尚不满足，使用 interrupt-window/NMI-window，而不是反复强制注入。
- 若异常组合按 x86 规则应形成 #DF，就反射 #DF；再次失败形成 triple fault 时按 VM 策略终止或重置。
