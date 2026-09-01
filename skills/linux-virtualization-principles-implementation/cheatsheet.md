# Cheatsheet

## 先决定看哪一层

| 症状 | 首要路径 | 第一批证据 |
|---|---|---|
| Guest 指令/寄存器异常 | VM-entry → VM-exit → exit reason/qualification | VMCS 控制位、退出计数、KVM trace |
| Guest 缺页或地址错乱 | GVA → GPA → HVA → HPA | Guest PTE、memory slot、EPT/影子 PTE |
| 丢中断/延迟高 | 请求 → routing → IRR → 目标 VCPU → 注入 → EOI | 路由表、edge/level、IRR/ISR、kick/posted 状态 |
| 设备性能差 | 完全虚拟化 / Virtio / 透传的退出深度 | VM-exit、用户态往返、队列深度、IOMMU fault |
| Virtio 卡队列 | descriptor → avail → notify → used → reclaim | idx、所有者、屏障、eventfd、完成中断 |
| Overlay 单向不通 | TAP → bridge → br-int → br-tun → VTEP → namespace/NAT | VLAN/VNI、流表、FDB、NAT 规则、双向抓包 |

## 地址翻译决策

- 只知道“Guest 地址”时，先确认是 GVA 还是 GPA。
- 无 EPT：优先检查 CR3 trap、Guest page walk、影子页表缓存与失效。
- 有 EPT：Guest 页表错则注入 Guest #PF；第二阶段错则处理 EPT violation。
- DMA 错误不要套 CPU 页表结论；单独检查 IOMMU 域与 IOVA→HPA。

## 设备模型选择

| 约束 | 优先选择 | 代价 |
|---|---|---|
| 原生驱动兼容最重要 | 完全虚拟化 | PIO/MMIO trap 多 |
| 通用高性能且可迁移 | Virtio | 需要协作驱动和队列协议 |
| 极致数据面性能 | VT-d/SR-IOV 透传 | 迁移、共享、回收和隔离更难 |

## 中断快速判断

- **无请求**：设备/队列完成路径未触发。
- **有请求无 pending**：routing、mask 或触发模式错误。
- **有 pending 不注入**：目标 VCPU、可中断窗口或 VM-entry 字段错误。
- **注入一次后停住**：优先查 ACK/EOI 与 level deassert。
- **中断风暴**：查 level 信号未撤销、EOI 丢失或重复路由表项。

## 性能默认规则

- 高频 VM-exit 先分类原因，再优化最高频且最深的路径。
- 能在 KVM 内核完成的退出，不要无条件返回用户空间。
- 队列批处理要同时限制队列深度并设置最大等待时间。
- OVS 控制面展示“如何决策”，datapath cache 才展示实际快路径。
- 优化前后同时看吞吐、尾延迟、VM-exit、CPU 占用和丢包/重试。

## 常见味道

- 把 GPA 当 HPA：忽略了 Host 对 Guest 内存的再次映射。
- 只看 `ping`：看不到二层隔离、NAT 和回程不对称。
- 只看 avail 不看 used：无法证明设备完成和驱动回收。
- 透传只配置 BAR：DMA 与中断仍可能越权或不可达。
- 看到 VM-exit 就归咎 KVM：真正成本可能来自返回用户态后的协议栈。
