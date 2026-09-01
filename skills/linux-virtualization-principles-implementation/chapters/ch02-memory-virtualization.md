# 第2章：内存虚拟化

## Core Idea

内存虚拟化必须同时回答两个问题：VMM 如何向 Guest 呈现一套可信的“物理内存”，以及 Guest 地址最终如何落到 Host 的真实页面。实践中要始终区分 **GVA、GPA、HVA、HPA**；影子页表用软件合成 GVA→HPA，EPT 则把翻译拆成硬件协作的 GVA→GPA 与 GPA→HPA 两阶段。

## Frameworks Introduced

- **四地址空间链路**：以 GVA（Guest Virtual Address）、GPA（Guest Physical Address）、HVA（Host Virtual Address）、HPA（Host Physical Address）拆解每一步翻译责任。
  - When to use：排查缺页、内存槽、MMIO、页表同步或 EPT violation 时。
  - How：先确认当前地址属于哪个空间，再标出翻译者：Guest 页表负责 GVA→GPA；memslot 负责 GPA→HVA；Host MM 负责 HVA→HPA；EPT 可在硬件中直接维护 GPA→HPA。
  - Why / failure mode：把 GPA 当 HVA 解引用或把 Guest CR3 当 HPA 使用，会跨越地址空间边界并访问错误页面。

- **“呈现内存”与“支撑内存”分离**：E820/BIOS 告诉 Guest 哪些 GPA 可用，`KVM_SET_USER_MEMORY_REGION` 告诉 KVM 哪段 HVA 支撑这些 GPA。
  - When to use：创建虚拟机内存布局、处理低端保留区、热插拔或多 memslot 时。
  - How：VMM 构造 E820 段；把 `int 0x15` 处理函数放入虚拟 BIOS 区；设置 IVT 0x15 项；再注册 `slot + guest_phys_addr + memory_size + userspace_addr`。
  - Why / failure mode：只注册 memslot 而不呈现 E820，Guest 不知道可用范围；只伪造 E820 而无 HVA 支撑，访问会在 KVM 侧失败。

- **按需缺页建映射**：只预建根页表，真正访问时再分配 Host 页面并补齐各级页表。
  - When to use：虚拟 RAM 较大但工作集较小，或希望利用 Host 虚拟内存与换页机制时。
  - How：根据故障地址定位 memslot；换算页帧；固定/获得用户页；逐级创建中间页表页；写入末级映射。
  - Why / failure mode：为整个虚拟内存条预分配 `struct page` 会浪费未触达页面，并限制内存超分与换页。

- **影子页表（Shadow Page Table）**：KVM 为每个 Guest 地址空间维护一张直接映射 GVA→HPA 的页表，并让硬件 CR3 指向它。
  - When to use：没有 EPT/NPT 等二阶段硬件支持时。
  - How：拦截 Guest 写 CR3，记录 Guest 根页表；以 Guest 根页帧号为 key 查找/缓存影子根；缺页时遍历 Guest 页表得到 GPA，再解析 memslot 得到 HPA，填充影子页表。
  - Why / failure mode：Guest 页表尚无映射时必须向 Guest 注入缺页异常；Guest 建好 GVA→GPA 后，第二次访问才可建立影子映射。页表写入若未被截获并同步，会产生陈旧映射。

- **EPT 两阶段翻译**：Guest MMU 原生完成 GVA→GPA，EPT 硬件完成 GPA→HPA。
  - When to use：CPU 支持 EPT 且 KVM 已启用时，这是常规选择。
  - How：VMCS 的 `GUEST_CR3` 指向 Guest 自己的页表，`EPT_POINTER` 指向每 VM 一张 EPT 根表；EPT violation 时从 `GUEST_PHYSICAL_ADDRESS` 取 GPA，分配/定位 HPA，执行 direct map。
  - Why / failure mode：若仍开启 CR3-load/store exiting，会为 Guest 进程切换制造无意义的 VM-exit；若把 EPT 根写入 Guest CR3，则 Guest 页表语义被破坏。

## Key Concepts

- **逻辑地址/线性地址**：段基址与段内偏移组成逻辑地址；段单元输出线性地址。平坦模型中段基址为 0，两者近似等价。
- **平坦内存模型**：内核/用户代码段和数据段覆盖相同地址空间，主要依靠分页而非分段实现隔离。
- **页表层级**：多级页表用按需分配的中间表换取空间效率；CR3 指向唯一根表。
- **E820**：BIOS `int 0x15, eax=0xE820` 返回的物理内存段描述；`ebx` 是续传索引，`es:di` 是输出缓冲区，`ecx` 是记录尺寸，`edx=0x534D4150` 是 SMAP 魔数。
- **memslot**：KVM 中一段连续 GPA 区间的描述；核心是 `base_gfn`、`npages`、flags 与对应 `userspace_addr`。
- **Virtual-8086 模式**：在保护环境中运行实模式代码；KVM 通过 VMCS 中 Guest RFLAGS 的 VM 位进入该模式。
- **虚拟 MMU 上下文**：KVM 根据 nonpaging、32 位分页、PAE、长模式选择不同的根级别与缺页处理函数。
- **CR3-load exiting**：VMCS 控制位；影子页表模式用它截获 Guest 地址空间切换，EPT 模式通常清除它。
- **EPT violation**：EPT 中 GPA→HPA 映射缺失或权限不符时的 VM-exit；故障 GPA 由 VMCS 提供。

## Mental Models

- **把地址看成“带命名空间的数值”**：相同的 `0x1000` 在 GVA、GPA、HVA、HPA 中不是同一个对象；任何转换函数都应明确输入与输出空间。
- **把虚拟内存分成控制面与数据面**：E820/BIOS 是 Guest 可见拓扑的控制面，memslot 与页表是实际承载访问的数据面。
- **把影子页表想成“编译产物”**：输入是 Guest 页表和 Host 页面布局，输出是 GVA→HPA 快速路径；输入一变，产物就需要失效或重编译。
- **把 EPT 想成“串联的第二个 MMU”**：Guest 仍管理自己的页表，EPT 只接收 GPA 并继续翻译；两者职责分离而非合并。
- **把缺页分层诊断**：Guest page fault 说明 GVA→GPA 未就绪；EPT violation 说明 GPA→HPA 未就绪；Host page fault 可能说明 HVA 尚未获得物理页。

## Anti-patterns

- **省略地址类型后缀**：变量只叫 `addr`，容易把 GPA、HVA 或 HPA 交给错误 API；实践中应使用 `gva/gpa/gfn/hva/hpa/pfn` 明确命名。
- **把 E820 当成真实内存分配器**：E820 只是 Guest 可见描述，不能替代 memslot/HVA 注册。
- **预先为所有 Guest RAM 分配物理页**：破坏按需分配、换页与内存超分，未使用页面也占用 HPA。
- **影子页表缺页时直接假定 Guest PTE 存在**：若 Guest 尚未建立 GVA→GPA，正确动作是注入 Guest page fault，而不是虚构 GPA。
- **Guest CR3 改变后从零重建所有影子页表**：进程切换频繁，成本过高；应以 Guest 根页帧为 key 缓存并复用影子根。
- **启用 EPT 后继续拦截每次 CR3 访问**：EPT 每 VM 一张，不随 Guest 进程切换；额外退出只有成本没有正确性收益。
- **把页表地址“取出”与“访问页表内容”混为一谈**：读取 CR3/PDE 中的 GPA 数值本身不经过 EPT；只有用该 GPA 访存读取 PDE/PTE 时才进行 GPA→HPA。

## Code Examples

注册虚拟内存条的关键 API 组合：

```c
struct kvm_userspace_memory_region m = {
    .slot = 0, .guest_phys_addr = 0,
    .memory_size = ram_size, .userspace_addr = ram_hva,
};
ioctl(vm_fd, KVM_SET_USER_MEMORY_REGION, &m);
```

地址转换的最小表达：

```text
gfn = gpa >> PAGE_SHIFT
hva = slot.userspace_addr + (gfn - slot.base_gfn) * PAGE_SIZE
hpa = host_mm_resolve(hva) + page_offset(gpa)
```

EPT 装载职责：

```text
VMCS.EPT_POINTER = ept_root_hpa
VMCS.GUEST_CR3   = guest_page_table_gpa
```

- **What it demonstrates**：同一 VM 同时持有 Guest 自己的页表根和 Host 控制的二阶段页表根，两者不能互换。

## Reference Tables

| 地址 | 含义 | 典型持有者 | 典型下一步 |
|---|---|---|---|
| GVA | Guest 进程虚拟地址 | Guest CPU/进程 | Guest 页表翻译为 GPA |
| GPA | Guest 看到的物理地址 | Guest 页表、E820、设备模型 | memslot/EPT 翻译 |
| HVA | VMM 进程虚拟地址 | QEMU/kvmtool | Host MM 解析为 HPA |
| HPA | 机器真实物理地址 | Host 内核、硬件 | 送往内存控制器 |

| 模式 | 生效页表 | 最终映射 | CR3 写入处理 | 典型缺页路径 |
|---|---|---|---|---|
| 实模式 Guest（无 EPT） | KVM nonpaging 页表 | GPA→HPA | 通常无需频繁切换 | GPA 缺失 → KVM direct map |
| 保护模式 + 影子页表 | 每 Guest 地址空间一张影子页表 | GVA→HPA | 必须 VM-exit，切换影子根 | 走 Guest 页表；必要时先注入 Guest fault |
| 保护模式 + EPT | Guest 页表 + 每 VM 一张 EPT | GVA→GPA→HPA | 通常不退出 | Guest fault 留在 Guest；EPT violation 进入 KVM |

| E820 调用寄存器 | 作用 |
|---|---|
| `eax=0xE820` | 选择完整内存映射查询 |
| `ebx` | 当前记录索引/续传标记；结束时返回 0 |
| `es:di` | 一条 `e820_entry` 的目标缓冲区 |
| `ecx=20` | 记录大小 |
| `edx=0x534D4150` | `SMAP` 魔数 |

| 机制 | 主要收益 | 主要成本/风险 |
|---|---|---|
| 影子页表 | 命中后一次硬件遍历得到 HPA | VM-exit、软件 walk、同步与多根缓存复杂度 |
| EPT | Guest 页表原生运行，二阶段硬件化 | EPT walk 与 EPT violation；仍需维护 GPA→HPA |

## Worked Example

以下重构一个启用 EPT 的 32 位、两级 Guest 页表访问流程。Guest 要读取 GVA `v`，页面大小为 4KB：

1. Guest CR3 中保存 Guest 页目录的 GPA。CPU 只是取出这个数值，还没有访问内存，因此此刻不需要 EPT。
2. CPU 用 `v[31:22]` 计算 PDE 索引，得到 PDE 的 GPA。为了真正读取 PDE 内容，硬件把这个 GPA 交给 EPT；EPT 将其翻译为 HPA 后完成内存读取。
3. PDE 给出 Guest 页表的 GPA。再次强调：取出这个数值无需 EPT；当 CPU 用 `v[21:12]` 算出 PTE 的 GPA 并读取 PTE 时，才再次经 EPT 翻译。
4. PTE 给出目标 Guest 页帧 GPA。CPU 加上 `v[11:0]` 页内偏移形成最终 GPA；因为要访问实际数据，第三次通过 EPT 得到最终 HPA。
5. 若 Guest PDE/PTE 不存在，产生普通 Guest page fault，由 Guest 内核建立 GVA→GPA，CPU 不必退出到 Host。
6. 若上述任一次 EPT 查询不存在 GPA→HPA 映射，产生 EPT violation。CPU VM-exit，KVM 从 VMCS `GUEST_PHYSICAL_ADDRESS` 读取故障 GPA；以 `gfn=gpa>>12` 定位 memslot，换算 HVA，获得 Host PFN/HPA；随后 `tdp_page_fault`/direct-map 路径补齐 EPT 表项，再 VM-entry 重试。

这个例子揭示一个常见误判：一次 Guest 数据访问可能触发多次 EPT 查表，因为 Guest 页表本身也存放在 Guest“物理内存”里；但 Guest 进程切换只更换 Guest CR3，不需要更换每 VM 唯一的 EPT 根，也不应因此产生 CR3-load VM-exit。

## Key Takeaways

1. 先给每个地址标注 GVA/GPA/HVA/HPA，再讨论转换；地址值脱离命名空间没有可操作意义。
2. E820 负责向 Guest 描述内存拓扑，memslot 负责把 GPA 区间连接到 Host 用户地址空间，两者缺一不可。
3. 按需缺页分配使虚拟 RAM 可以利用 Host 虚拟内存、换页和超分；预分配全部物理页会丢失这些能力。
4. 影子页表把两阶段关系合成为 GVA→HPA，但代价是拦截 CR3、软件遍历 Guest 页表和维护一致性。
5. EPT 恢复 Guest 页表的原生职责，并以每 VM 一张 EPT 表承接 GPA→HPA，显著减少 VM-exit 和页表数量。
6. Guest page fault、EPT violation、Host page fault 分属不同层次，诊断时不能混为一种“缺页”。

## Connects To

- **Ch 1 CPU 虚拟化**：CR3-load exiting、EPT violation 和缺页异常都通过 VMCS 控制字段或退出信息连接到 VM-exit 处理。
- **设备虚拟化**：不属于 RAM memslot 的 GPA 可能是 MMIO；内存异常路径需要分流到设备模拟。
- **中断与启动**：实模式 BIOS、IVT、`int 0x15` 与 VCPU 启动状态共同建立 Guest 最早期运行环境。
- **Host 内存管理**：`get_user_pages`、换页、页帧固定和脏页跟踪决定虚拟内存条最终如何消耗物理资源。
- **迁移与快照**：memslot、脏页位图、Guest 可见 E820 和二阶段映射共同界定需要迁移的内存状态。
