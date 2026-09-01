# 第6章：网络虚拟化

## Core Idea

Overlay 网络把租户二层报文封装成宿主机三层网络中的 UDP/VXLAN 载荷，使租户地址、广播域和拓扑独立于物理 IDC 网络。完整实现不是单个“虚拟交换机”，而是 TAP、Linux bridge、veth、Open vSwitch、VTEP、namespace、iptables/NAT 与物理网卡共同组成的分层数据路径。

## Frameworks Introduced

- **Underlay / Overlay 双地址空间框架**：内层包表达租户通信，外层包只负责在计算节点和网络节点之间运输。
  - When to use：设计多租户隔离、允许私网地址重叠，或排查“隧道可达但虚拟机不通”时。
  - How：在计算节点给租户流量分配本地 VLAN Tag；进入 `br-tun` 时把 VLAN 映射为 VXLAN ID；VTEP 使用宿主机 IP 封装；对端解封后再映射回本地 VLAN。
  - Why / failure mode：VLAN 只有约 4096 个标识且平坦二层限制用户自定义子网；Overlay 扩大租户标识空间并解耦拓扑。若用内层 IP 识别租户，重叠网段会产生歧义。

- **计算节点接入链框架**：`VM ↔ TAP ↔ Linux bridge(qbr) ↔ veth(qvb/qvo) ↔ br-int ↔ patch ↔ br-tun ↔ VTEP`。
  - When to use：定位单 VM 的二层接入、安全组、VLAN 标记或隧道出口问题。
  - How：QEMU 通过 TAP 文件接口写入以太帧；TAP 网络设备把 skb 交给 Linux bridge；bridge 经过 netfilter；veth 跨接 Linux bridge 与 OVS；`br-int` 做租户本地交换，`br-tun` 做 VLAN/VXLAN 映射。
  - Why / failure mode：早期 OVS 接收路径绕开 Linux netfilter，因此在 VM 与 OVS 间插入 Linux bridge 承载 iptables 安全组。把 patch 口用于 Linux bridge 或把 veth 当 OVS 内部 patch 口都会破坏连接语义。

- **网络节点三桥三域框架**：`br-tun` 终结 Overlay，`br-int` 接入租户路由器端口，`br-ex` 接入外部物理网络。
  - When to use：分析南北向流量、Floating IP、网关或外网桥问题。
  - How：VXLAN 包经物理口和 UDP 4789 socket 到 `br-tun`；解封后进入 `br-int` 的 `qr-*` 内部端口与 router namespace；`qg-*` 再接 `br-ex` 与外部网卡。
  - Why / failure mode：网络 namespace 把每个租户的路由表、ARP、iptables 隔离开；若在宿主机根 namespace 查规则，常会得出“没有 NAT”的错误结论。

- **控制面决策、数据面缓存框架**：OpenFlow/OVS 用户态先按多表流水线算出动作，datapath 缓存最终 flow 并直接转发后续同类报文。
  - When to use：解释逻辑路径与抓到的内核实际路径不一致时。
  - How：控制面匹配入端口、VLAN、MAC、VXLAN ID并执行 `resubmit/mod_vlan_vid/set_tunnel/output`；首包完成决策后，下发“qvo 直达 VXLAN”或“VXLAN 直达 qr/qvo”的数据面动作。
  - Why / failure mode：逻辑上报文经过 `br-int → patch → br-tun`，实际 datapath 可跨桥直达目标端口。只根据拓扑图推断逐跳抓包位置，会漏掉快速路径。

- **Floating IP 双向 NAT 框架**：出向以 SNAT 把 private IP 替换为 Floating IP，入向以 DNAT 把 Floating IP 还原为 private IP。
  - When to use：让外部主机访问虚拟机，或让外部看到稳定的租户出口地址时。
  - How：在租户 router namespace 的 `iptables`/netfilter 中应用 NAT；`qg-*` 配置 Floating IP 辅地址并代理 ARP；路由在 `qr-*`（租户侧）和 `qg-*`（外部侧）之间转发。
  - Why / failure mode：NAT、路由和 ARP 必须一致。只添加 NAT 规则但未让 `qg-*` 回答 Floating IP 的 ARP，外部报文到不了网络节点。

## Key Concepts

- **Overlay**：在现有三层 underlay 之上承载独立虚拟二层网络的机制；内层租户包作为外层宿主机包的 payload。
- **VXLAN**：常以 UDP 4789 承载以太帧的 Overlay 封装，并用 VXLAN Network Identifier 区分租户网络。
- **VTEP**：VXLAN Tunnel Endpoint，负责封装/解封；本章场景中由计算节点或网络节点上的 OVS VXLAN 端口承担。
- **TAP**：TUN/TAP 的二层模式；一端是供 QEMU 等用户进程读写的文件接口，一端是可加入 bridge 的内核网络设备。
- **Linux bridge**：内核二层交换机；在该部署中以 `qbr-*` 承载可由 netfilter/iptables 实施的安全组过滤。
- **veth pair**：成对虚拟网卡，一端收到的帧从另一端出现；`qvb-*` 接 Linux bridge，`qvo-*` 接 `br-int`。
- **network namespace**：隔离网络设备、路由表、ARP 和 netfilter 状态；`qrouter-*` 与 `qdhcp-*` 分别承载租户路由和 DHCP。
- **br-int / br-tun / br-ex**：分别承担租户集成交换、隧道转换、外部网络接入；职责不同但可由 OVS datapath 优化为直接动作。
- **VLAN ↔ VXLAN 映射**：VLAN Tag 是节点本地的租户标记，VXLAN ID 是跨节点隧道标记；同一租户在不同节点的本地 VLAN 值可以不同。
- **SNAT / DNAT**：分别改写源地址和目的地址，建立 private IP 与 Floating IP 的双向可达关系。

## Mental Models

- 把一次虚拟机报文看成“三次换装”：VM 以原始以太帧出发，本地交换时穿 VLAN 标签，穿 underlay 时再套 VXLAN/UDP/IP 外壳。
- 用“三张图”排障：逻辑拓扑图说明组件责任，namespace/端口图说明连接关系，datapath flow 说明报文实际走法。
- 把租户身份看成随边界转换的标签：计算节点本地 VLAN、隧道中的 VXLAN ID、对端本地 VLAN 三者相关但不必相等。
- 把网络节点看成协议边界：一侧是 Overlay 二层租户网络，另一侧是可路由、可 NAT、可 ARP 的外部三层网络。

## Anti-patterns

- **用租户 IP 标识网络**：不同租户可复用 `192.168.0.0/16`；必须依靠 VLAN/VXLAN ID 与 namespace 隔离。
- **把 TAP 当成隧道设备**：TAP 只把 VM 进程接入宿主机二层网络，VXLAN 封装由 VTEP/OVS 完成。
- **把 veth 与 OVS patch 混用**：veth 连接不同类型的网络栈对象；patch 是 OVS 桥之间的高效内部连接。
- **只查根 namespace 的路由和 iptables**：租户 NAT、网关接口和路由通常存在 `qrouter-*` namespace 内。
- **认为拓扑中的每个桥都对应实际逐跳转发**：OVS datapath 缓存可直接从 qvo 到 VXLAN，或从 VXLAN 到 qr/qvo。
- **忽略 ARP 与 FDB 学习**：流表知道“MAC 在哪个 VTEP”通常依赖广播/学习；未知单播或错误 FDB 会被洪泛或送错端口。
- **只做 DNAT/SNAT，不验证返回路径**：状态跟踪、路由、反向 VTEP 和安全组任一缺失都会形成单向可达。

## Code Examples

```bash
# 关键观测组合：连接、namespace、OVS 控制面与数据面
ip -d link show
ip netns exec qrouter-... ip route
ip netns exec qrouter-... iptables-save
ovs-ofctl dump-flows br-tun
ovs-dpctl dump-flows
```

```text
# 典型逻辑动作：本地租户标签 → 隧道标签 → 远端 VTEP
match(in_port=qvo, vlan=3, dst=gateway_mac)
actions=strip_vlan,set_tunnel:0x4,output:vxlan_to_network_node
```

```text
# 入向 NAT
Floating IP 10.75.234.7 → DNAT → private IP 192.168.0.3
```

- **What it demonstrates**：正确诊断需要同时观察 Linux 网络对象、namespace 规则、OVS 逻辑流表与内核 datapath 的最终动作。

## Reference Tables

| 组件 | 所在位置 | 层次/角色 | 关键连接或转换 |
|---|---|---|---|
| TAP | 计算节点 | 二层 VM 接口 | QEMU 文件接口 ↔ 内核 netdev |
| `qbr-*` | 计算节点 | Linux bridge / 安全组 | TAP ↔ `qvb-*`，经过 netfilter |
| `qvb-*` / `qvo-*` | 计算节点 | veth pair | Linux bridge ↔ OVS `br-int` |
| `br-int` | 计算/网络节点 | 租户集成交换 | 端口 VLAN Tag、MAC/FDB 转发 |
| patch-tun / patch-int | 计算/网络节点 | OVS patch | `br-int` ↔ `br-tun` |
| `br-tun` | 计算/网络节点 | Overlay 转换 | VLAN ↔ VXLAN ID，选择 VTEP |
| VXLAN port / VTEP | 两类节点 | UDP 隧道端点 | 内层以太帧 ↔ 外层 UDP/IP |
| `qrouter-*` | 网络节点 | router namespace | `qr-*` 租户侧 ↔ `qg-*` 外部侧，NAT/路由 |
| `br-ex` | 网络节点 | 外部交换 | `qg-*`、同名 internal 口、物理网卡 |

| 方向 | 网络节点 NAT | 封装方向 | 关键地址变化 |
|---|---|---|---|
| VM → 外部 | SNAT | 计算节点封装，网络节点解封 | `src private → Floating IP` |
| 外部 → VM | DNAT | 网络节点封装，计算节点解封 | `dst Floating IP → private` |
| 同子网跨计算节点 | 无需网络节点 NAT | 计算节点 VTEP 直连 | 内层地址不变，外层目标为对端计算节点 |

## Worked Example

以 vm1 为例：private IP `192.168.0.3`，Floating IP `10.75.234.7`，运行在计算节点 `10.76.36.36`；其租户网络 VXLAN ID 为 `4`，网关位于网络节点 `10.73.189.17`。

### 出向：vm1 访问 IDC 外部主机

1. vm1 生成目的 MAC 为租户网关的以太帧。QEMU 向 `tap03247d72-8f` 文件接口写帧；TAP netdev 把 skb 送入 Linux bridge `qbr03247d72-8f`。
2. `qbr` 应用 netfilter 安全组并依据 FDB 转发到 `qvb`；帧从 veth 对端 `qvo` 进入计算节点 `br-int`。该端口对应本地 VLAN Tag。
3. OVS 控制面逻辑经 patch 口进入 `br-tun`，把本地 VLAN 转为 `tun_id=0x4`，依据网关 MAC 选择通往网络节点 `10.73.189.17` 的 VXLAN 端口。数据面缓存后可直接执行 `qvo → set tunnel → VXLAN port`。
4. VTEP 添加外层 UDP/VXLAN/IP 头：外层源/目的 IP 是计算节点与网络节点，内层仍是 vm1 到租户网关的帧。underlay 只需路由这两个宿主机 IP。
5. 网络节点 UDP 4789 socket 接收并解封，OVS 根据 `tun_id=4` 映射成本地 VLAN。逻辑路径是 `br-tun → patch → br-int → qr-*`，datapath 可把 VXLAN 端口直接转发到对应 `qr-*`。
6. 报文进入 vm1 的 `qrouter-*` namespace。iptables 执行 SNAT，把源 IP `192.168.0.3` 改为 `10.75.234.7`；路由选择外部侧 `qg-*`。
7. `qg-*` 接入 `br-ex`，后者依据外部网关 MAC/FDB 从物理口 `xgbe0` 发往 IDC。外部主机看到的源地址是 Floating IP，而看不到租户 private IP。

### 入向：IDC 主机访问 vm1

1. 外部主机 ARP 查询 `10.75.234.7`。该 Floating IP 作为 `/32` 辅地址配置在对应 router namespace 的 `qg-*` 上，因此网络节点代为应答，外部报文进入 `br-ex → qg-*`。
2. router namespace 的 netfilter 在 PREROUTING/OUTPUT 路径执行 DNAT：目的地址 `10.75.234.7 → 192.168.0.3`。租户路由表把报文从 `qr-*` 送入网络节点 `br-int`。
3. `br-int` 加本地 VLAN；`br-tun` 根据 VLAN 和 vm1 的目的 MAC 选择计算节点 `10.76.36.36`，去掉 VLAN 并设置 VXLAN ID 4。datapath 可直接执行 `qr → set tunnel → VXLAN port`。
4. 计算节点的 UDP 4789 socket 解封。`br-tun` 将 VXLAN ID 4 映射为该节点本地 VLAN，再按 vm1 MAC 选择 `qvo`；快速路径可直接 `VXLAN port → qvo`。
5. 报文经 veth 的 `qvo → qvb` 进入 `qbr`，再次受安全组过滤，再由 FDB 转发到 `tap`，最终被 QEMU/虚拟机网卡接收。

这条双向路径揭示了排障顺序：先验证 Floating IP 的 ARP 与 `br-ex`，再验证 namespace 内 NAT/路由，再看网络节点 VTEP，最后检查计算节点的 VXLAN 映射、`qvo/qvb/qbr/tap` 和安全组。反向验证则按相反方向逐层推进。

## Key Takeaways

1. Overlay 把租户拓扑与物理网络解耦，租户身份跨节点依靠 VXLAN ID，而非可重叠的 private IP。
2. TAP、Linux bridge、veth、OVS patch 和 VXLAN port 各解决不同边界，不能互相替代。
3. 计算节点负责 VM 接入、安全组和隧道封装；网络节点负责 Overlay 终结、租户路由、NAT 与外网接入。
4. VLAN 是节点本地标签，VXLAN ID 是跨节点标签；排障必须核对双向映射。
5. OVS 控制面展示逻辑流水线，datapath flow 才展示稳定流量的实际快速路径。
6. Floating IP 可达性是 ARP、DNAT/SNAT、namespace 路由、VTEP 与返回路径共同成立的结果。

## Connects To

- **Ch 5：Virtio 虚拟化**：Virtio-net 将 Guest 网络缓冲交给 Host；TAP 是进入本章宿主机网络路径的常见下一站。
- **Linux network namespace**：为每个租户构造独立的路由器、DHCP、ARP 与 netfilter 控制域。
- **OpenFlow / SDN**：控制面以多表规则表达策略，OVS datapath 把结果压缩为高效缓存动作。
- **Netfilter / NAT**：安全组、SNAT、DNAT 与双向规则决定南北向通信是否成立。
- **Underlay 路由**：Overlay 不修复物理网络；VTEP IP、MTU、UDP 4789 和宿主机路由仍必须可达。
