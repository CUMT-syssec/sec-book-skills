# Chapter 6: 内存虚拟化

## Core Idea

内存虚拟化把 guest 看到的 GPA 空间与平台 HPA 空间隔离：guest paging structure 完成 GLA→GPA，EPT paging structure 完成 GPA→HPA。正确实现不仅要建立 EPT，还要按映射途径和 cache 域精确执行失效，避免把旧转换留给错误的 VM、地址空间或 EPTP。

## Frameworks Introduced

- **两阶段地址转换**：guest 分页负责 GLA→GPA，EPT 负责 GPA→HPA。
  - When to use: 定位页故障归属或分析一次内存访问的 walk。
  - How: guest 页表检查失败产生 `#PF`，属于 guest；任一 GPA→HPA 转换失败产生 EPT violation/misconfiguration，属于 VMM。
- **EPT 四级 walk**：EPTP 指向 HPA 中的 PML4T；4K 页依次检查 PML4E→PDPTE→PDE→PTE，2M 页止于 PDE，1G 页止于 PDPTE。
  - Decision rule: 每级先判断 present，再判断表项配置，最后把各级 R/W/X 权限按位 AND，检查实际访问。
- **三种映射缓存模型**：`linear mapping`、`guest-physical mapping`、`combined mapping` 分别对应不同转换路径。
  - How: 非 EPT 分页产生 linear；EPT 的 GPA→HPA 产生 guest-physical；开启 guest 分页与 EPT 时可缓存 GLA→HPA 的 combined 结果。
- **三维 cache 域**：PCID 标识 guest 进程地址空间，VPID 标识虚拟处理器，EP4TA（EPTP 的 PML4T 地址）标识 GPA 空间。
  - Domain tags: linear=`VPID+PCID`；guest-physical=`EP4TA`；combined=`VPID+PCID+EP4TA`。
- **最小失效选择法**：按“改变的是线性映射还是 GPA 映射”选择 INVVPID/INVPCID 或 INVEPT，再选 individual/single/all-context 范围。

## Key Concepts

- **GPA / HPA**：guest 私有物理地址与真实平台物理地址；EPT 负责二者隔离。
- **EPTP / EP4TA**：EPTP 提供 PML4T 的 HPA、EPT 页表内存类型、walk 长度及 A/D 开关；其页表根地址 EP4TA 是缓存域标识。
- **EPT access rights**：每级表项的 R/W/X 位共同限制最终访问权限；not-present 表示三位全零。
- **EPT violation**：表项 not-present 或有效权限不足；qualification 给出访问类型、合成权限和故障位置。
- **EPT misconfiguration**：present 表项编码非法，如 write-only、未支持的 execute-only、保留位非零或非法内存类型。
- **A/D bits**：EPTP[6] 开启后，表项 bit 8/9 记录 accessed/dirty；清零这些位可能要求失效。
- **VPID**：启用时必须为 guest 提供非零且彼此隔离的标识；root/未启用时使用 0000H。
- **INVEPT / INVVPID**：前者按 EP4TA 清 GPA/combined 缓存，后者按 VPID 清 linear/combined 缓存。

## Mental Models

- **把页故障看作管辖权问题**：`#PF` 表示 guest 页表语义；EPT 故障表示 VMM 映射或策略语义。不要让 guest 修复 host 的 EPT，也不要让 VMM吞掉正常 guest `#PF`。
- **把失效看作 tag 匹配**：先列出旧 cache 条目的 tag，再选择能覆盖这些 tag 的指令；指令名字不是决策起点。
- **把 EPT violation 当作策略事件，misconfiguration 当作结构错误**：前者可用于按需分配、写保护或执行控制；后者通常应修正页表编码。
- **把“大页”看作 walk 深度与失效粒度的交换**：1G/2M 减少 walk，但扩大权限、内存类型与更新影响范围。

## Anti-patterns

- **不同 VM 复用 VPID 或 EP4TA**：会扩大失效范围，甚至混淆 GPA 空间。
- **修改 EPT 表项后无条件不失效**：旧的 guest-physical/combined mapping 可能继续生效。
- **修复 not-present 后总执行 INVEPT**：该失败转换没有建立缓存，通常无需失效；先判断是否存在旧的成功映射。
- **只刷新 linear mapping**：EPT 地址或权限变化必须覆盖 guest-physical 与 combined mapping。
- **把 EPTP 内存类型与叶表项内存类型混为一谈**：前者用于 EPT 数据结构本身，后者用于映射页面。

## Reference Tables

| 故障 | 判定 | VM-exit 证据 | 处理方向 |
|---|---|---|---|
| EPT violation | not-present 或最终 R/W/X 不允许当前访问 | qualification + GPA；可能有 GLA | 分配/改权限/仿真或拒绝 |
| EPT misconfiguration | present 但编码非法 | GPA；qualification 未定义 | 修复表项结构 |
| `#PF` | guest paging 检查失败 | guest 异常语义 | 通常反射给 guest |

| 指令 | 目标域 | 覆盖的映射 |
|---|---|---|
| INVLPG / INVPCID | 当前 VPID 下的地址/PCID | linear + combined |
| INVVPID | 指定 VPID（跨其 PCID/EP4TA） | linear + combined |
| INVEPT | 指定或全部 EP4TA | guest-physical + combined |

```text
on_ept_fault(gpa, qualification):
    if entry.present and entry.encoding_invalid:
        repair_ept_entry()          # misconfiguration
    else:
        enforce_or_extend_mapping() # violation
    if an_old_successful_translation_may_exist:
        invept(single_context, eptp)
```

## Key Takeaways

1. GLA→GPA 与 GPA→HPA 是两个独立检查层；`#PF` 与 EPT 故障的责任边界不同。
2. EPT walk 每级执行 present→配置合法性检查，最终权限是所有层 R/W/X 的交集。
3. `exit qualification` 是处理 EPT violation 的决策输入；EPT misconfiguration 的 qualification 不可依赖。
4. EPT 地址、权限、页大小、内存类型或 A/D 语义变化后，按旧缓存是否可能存在决定 INVEPT。
5. VPID/PCID/EP4TA 是缓存正确性的身份体系；INVVPID 与 INVEPT 分别切中线性侧和 GPA 侧。

## Connects To

- **Ch 5**：EPT 故障通过 VM-exit information 区域暴露，必须按 reason 限定读取 qualification 与地址字段。
- **Ch 7**：APIC-access page 的映射变化同时涉及 APIC 虚拟化控制与 INVVPID/INVEPT 失效。
- **TLB 与 paging-structure cache**：性能优化只有在 tag 隔离和失效语义正确后才成立。
