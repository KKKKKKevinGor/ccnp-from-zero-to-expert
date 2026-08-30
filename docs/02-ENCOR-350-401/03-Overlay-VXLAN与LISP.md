# 03 · Overlay：VXLAN 与 LISP

## ① 这章解决什么问题

数据中心里，有一台虚拟机要从机架 A 热迁移到机架 B。

**要求：迁移过程中 IP 不能变**（否则所有连接中断，业务感知到故障）。

IP 不变意味着**迁移前后必须在同一个二层网络**。但机架 A 和机架 B 之间是三层网络（Spine-Leaf 架构，全部用路由链路）。

**怎么在三层网络上"变出"一个二层网络？**

这就是 **Overlay（叠加网络）** 要解决的问题。而 VXLAN 是目前最主流的答案。

同时还有一个更根本的问题：**IP 地址天生承担了两个矛盾的职责**——既是"身份标识"（我是谁），又是"位置标识"（我在哪）。设备一移动，位置变了，身份也被迫跟着变。**LISP 就是要把这两个职责分开。**

---

## ② Underlay 与 Overlay

```
   ┌─────────────────────────────────────────────────┐
   │  Overlay（叠加网络）                              │
   │  VM-A ─────── 虚拟的二层网络 ─────── VM-B         │
   │  10.1.1.10                        10.1.1.20      │
   │  （它们以为自己在同一个 VLAN 里）                   │
   └────────────────────┬────────────────────────────┘
                        │ 封装/解封装
   ┌────────────────────▼────────────────────────────┐
   │  Underlay（底层网络）                             │
   │  Leaf-1 ── Spine ── Leaf-2                       │
   │  192.168.1.1      192.168.1.2                    │
   │  （纯三层，跑 OSPF/BGP，ECMP，无 STP）              │
   └─────────────────────────────────────────────────┘
```

| | **Underlay** | **Overlay** |
|:--|:--|:--|
| 是什么 | 真实的物理网络 | 在物理网络上"叠加"出的虚拟网络 |
| 协议 | OSPF / IS-IS / BGP | VXLAN / GRE / LISP / MPLS |
| 关注 | 设备之间的 IP 可达性 | 业务之间的连通性和策略 |
| 变化频率 | 低（物理拓扑稳定） | 高（业务频繁变动） |
| 类比 | 高速公路网 | 物流公司的运输路线 |

**核心思想：把"物理连通性"和"业务连通性"解耦。**

Underlay 只需要保证"任意两台设备的 IP 能互通"，简单稳定。所有业务的复杂性（VLAN、隔离、策略、迁移）都在 Overlay 里解决，不需要动物理网络。

**这个思想贯穿了所有现代网络架构**：SD-WAN、SD-Access、ACI、Kubernetes 的 CNI（Calico/Flannel），全都是 Underlay + Overlay 的结构。

---

## ③ VXLAN

### 3.1 VXLAN 是什么

**VXLAN (Virtual eXtensible LAN) = 把二层以太网帧封装进 UDP 包，从而能跨三层网络传输。**

```
原始二层帧：
┌────────┬───────┬──────┬──────────┐
│ 目的MAC │ 源MAC │ 类型 │   数据    │
└────────┴───────┴──────┴──────────┘

VXLAN 封装后：
┌─────────┬────────┬──────────┬──────────┬─────────────────────┐
│ 外层MAC  │ 外层IP  │ UDP:4789 │ VXLAN头  │  完整的原始二层帧     │
│ 14 字节  │ 20 字节 │  8 字节  │  8 字节  │                     │
└─────────┴────────┴──────────┴──────────┴─────────────────────┘
    └──────────── 总开销 50 字节 ────────────┘
    
外层 IP：Leaf-1 的 IP → Leaf-2 的 IP （在 Underlay 里正常路由）
```

**关键参数**：

| 参数 | 值 | 说明 |
|:--|:--|:--|
| **UDP 目的端口** | **4789** | IANA 标准（Cisco 早期用 8472，注意兼容性） |
| **VNI** | **24 位** | VXLAN Network Identifier，**1600 万个** |
| **封装开销** | **50 字节** | MTU 从 1500 → 1450，**Underlay 建议开启巨帧 9000** |
| **VTEP** | — | VXLAN Tunnel EndPoint，做封装/解封装的设备 |

### 3.2 VXLAN 解决了 VLAN 的三个问题

| VLAN 的问题 | VXLAN 的解法 |
|:--|:--|
| **只有 4094 个**（VID 12 位） | **VNI 24 位 = 1600 万个**，多租户够用了 |
| **必须在同一个二层域内**，跨不了三层 | 封装进 UDP，**可以跨任意三层网络** |
| **依赖 STP**，一半链路被阻塞 | Underlay 是三层 + ECMP，**所有链路 100% 利用** |

> **考点**：VLAN → VXLAN 的演进动机是 ENCOR 的高频考点。记住这条因果链：
> **4094 个不够 → VNI 24 位；跨不了三层 → UDP 封装；STP 浪费带宽 → 三层 Underlay + ECMP。**

### 3.3 VTEP 与转发过程

**VTEP (VXLAN Tunnel EndPoint)** 是做封装/解封装的设备，通常是 Leaf 交换机（也可以是服务器上的虚拟交换机）。

**转发过程**：

```
VM-A (10.1.1.10, MAC-A)              VM-B (10.1.1.20, MAC-B)
      │                                      │
   [Leaf-1 / VTEP-1]                  [Leaf-2 / VTEP-2]
   Loopback: 1.1.1.1                  Loopback: 2.2.2.2
      │                                      │
      └────────── Underlay (三层) ───────────┘

① VM-A 发一个二层帧给 VM-B（目的 MAC-B）
② VTEP-1 收到，查表：MAC-B 在 VTEP-2 后面（IP 2.2.2.2），VNI = 10000
③ VTEP-1 封装：
     外层 IP: 1.1.1.1 → 2.2.2.2
     UDP: 源端口随机（用于 ECMP 哈希），目的 4789
     VXLAN 头: VNI = 10000
     载荷: 原始的完整二层帧
④ Underlay 正常路由这个 UDP 包（可以走 ECMP 多路径）
⑤ VTEP-2 收到，剥掉外层封装，看 VNI=10000 找到对应的二层域
⑥ 把原始帧发给 VM-B
```

> **细节**：**外层 UDP 的源端口是根据内层帧的哈希值生成的**。这让 Underlay 的 ECMP 能把不同的流哈希到不同路径上，实现负载均衡。如果源端口固定，所有 VXLAN 流量会挤在一条路径上。

### 3.4 VTEP 怎么知道"MAC-B 在哪个 VTEP 后面"

这是 VXLAN 的核心问题，有两代解法：

#### 第一代：Flood & Learn（泛洪学习，已过时）

用**组播**模拟广播域：
- 每个 VNI 对应一个组播组
- 未知单播、广播、组播（**BUM 流量**）通过组播泛洪到所有 VTEP
- VTEP 通过观察收到的包学习 MAC ↔ VTEP 的对应关系

**问题**：
- 需要 Underlay 支持组播（PIM），复杂度高
- 泛洪流量大，扩展性差
- 学习是被动的，收敛慢

#### 第二代：VXLAN + EVPN（★ 现代标准）

用 **MP-BGP EVPN**（Ethernet VPN，RFC 7432）作为**控制平面**：

```
   控制平面（EVPN/BGP）：
   VTEP-1 ──BGP──> Route Reflector <──BGP── VTEP-2
        "我这里有 MAC-A、IP 10.1.1.10、VNI 10000"
                          ↓
        VTEP-2 提前知道了，不需要泛洪学习
   
   数据平面（VXLAN）：
   VTEP-1 ═══════ VXLAN 封装 ═══════ VTEP-2
```

**EVPN 的价值**：

| 能力 | 说明 |
|:--|:--|
| **控制平面学习** | MAC/IP 信息通过 BGP 主动通告，**不需要泛洪** |
| **减少 BUM 流量** | ARP 可以本地代答（ARP Suppression） |
| **多归属（Multi-homing）** | 一台服务器可以双上联到两个 Leaf，全活 |
| **集成路由与桥接（IRB）** | 同一个设备同时做二层转发和三层路由 |
| **快速收敛** | BGP 撤销路由，比等 MAC 老化快得多 |

**EVPN 的五种路由类型（了解即可）**：

| Type | 名称 | 作用 |
|:--|:--|:--|
| Type-2 | MAC/IP Advertisement | **通告 MAC 和 IP**（最常用） |
| Type-3 | Inclusive Multicast | 建立 BUM 流量的转发列表 |
| Type-5 | IP Prefix | 通告三层前缀（对接外部网络） |
| Type-1 | Ethernet Auto-Discovery | 多归属场景 |
| Type-4 | Ethernet Segment | 多归属的 DF 选举 |

> **ENCOR 层面只需理解 VXLAN 的封装原理和 VNI 的作用**，EVPN 的细节属于 CCIE DC 范围。但要知道"现代 VXLAN 部署一定配 EVPN 控制平面"这个结论。

### 3.5 VXLAN 配置示例（Nexus / Catalyst 9000）

```cisco
! ── ① Underlay：三层网络 + Loopback ──
Leaf-1(config)# interface Loopback0
Leaf-1(config-if)# ip address 1.1.1.1 255.255.255.255       ! VTEP 源地址

Leaf-1(config)# interface Ethernet1/1
Leaf-1(config-if)# no switchport
Leaf-1(config-if)# ip address 192.168.1.1 255.255.255.252
Leaf-1(config-if)# ip ospf network point-to-point

Leaf-1(config)# router ospf UNDERLAY
Leaf-1(config-router)# router-id 1.1.1.1

! ── ② 开启功能 ──
Leaf-1(config)# feature nv overlay
Leaf-1(config)# feature vn-segment-vlan-based
Leaf-1(config)# feature bgp
Leaf-1(config)# nv overlay evpn

! ── ③ VLAN 映射到 VNI ──
Leaf-1(config)# vlan 100
Leaf-1(config-vlan)#  vn-segment 10000                      ! VLAN 100 ↔ VNI 10000

! ── ④ 创建 NVE 接口（VTEP）──
Leaf-1(config)# interface nve1
Leaf-1(config-if-nve)#  no shutdown
Leaf-1(config-if-nve)#  source-interface loopback0
Leaf-1(config-if-nve)#  host-reachability protocol bgp      ! 用 EVPN 控制平面
Leaf-1(config-if-nve)#  member vni 10000
Leaf-1(config-if-nve-vni)#   suppress-arp                   ! ARP 本地代答
Leaf-1(config-if-nve-vni)#   ingress-replication protocol bgp

! ── ⑤ BGP EVPN ──
Leaf-1(config)# router bgp 65001
Leaf-1(config-router)# neighbor 10.10.10.10 remote-as 65001  ! Spine 作为 RR
Leaf-1(config-router-neighbor)#  update-source loopback0
Leaf-1(config-router-neighbor)#  address-family l2vpn evpn
Leaf-1(config-router-neighbor-af)#   send-community extended

! ── ⑥ MTU（★ 必须）──
Leaf-1(config)# system jumbomtu 9216
Leaf-1(config)# interface Ethernet1/1
Leaf-1(config-if)# mtu 9216

! ── 查看 ──
Leaf-1# show nve peers
Leaf-1# show nve vni
Leaf-1# show bgp l2vpn evpn
Leaf-1# show l2route evpn mac all
Leaf-1# show mac address-table
```

### 3.6 VXLAN 的 MTU 问题（必须处理）

**VXLAN 增加 50 字节开销**：
```
原始帧 1518 字节（含以太网头和 FCS）
+ VXLAN 封装 50 字节
= 1568 字节
```

**Underlay 的接口 MTU 必须至少 1550，强烈建议直接开巨帧 9000+。**

```cisco
! Nexus
Leaf-1(config)# system jumbomtu 9216
Leaf-1(config)# interface Ethernet1/1
Leaf-1(config-if)# mtu 9216

! Catalyst 9000
Leaf-1(config)# system mtu 9198
```

> **如果不开巨帧**：所有超过 1450 字节的用户流量都会失败（因为封装后超过 Underlay MTU 被丢弃）。症状同样是"ping 通但业务不通"、"大文件传不动"。
>
> **VXLAN 部署的第一个检查项永远是 MTU。**

---

## ④ LISP

### 4.1 LISP 解决什么问题

**核心洞察：IP 地址同时承担了两个矛盾的角色。**

```
IP 地址 = 身份 (Who you are) + 位置 (Where you are)
              ↓                      ↓
        我是 10.1.1.10          我在这个网段/这台路由器后面
```

**矛盾在哪**：
- 设备移动了 → **位置变了** → 按理 IP 要变 → 但**身份也跟着变了**，所有连接中断
- 想保持 IP 不变 → **位置信息就失效了** → 路由系统不知道往哪送

**这个矛盾造成的实际问题**：

| 问题 | 表现 |
|:--|:--|
| **移动性差** | 手机从 WiFi 切到 4G，IP 变了，所有 TCP 连接断开 |
| **路由表爆炸** | 为了支持多归属和 TE，运营商往互联网注入大量明细路由，全球 BGP 表已超 90 万条 |
| **多归属复杂** | 一个企业接两个 ISP，要跑 BGP、申请 AS 号、申请 PI 地址 |

### 4.2 LISP 的解法：分离身份与位置

```
   EID (Endpoint Identifier)  =  身份，终端用的地址，不随移动改变
   RLOC (Routing Locator)     =  位置，路由器的地址，标识"在哪"
   
   映射系统（Mapping System）维护 EID → RLOC 的对应关系
```

```
   主机 A (EID: 10.1.1.10)              主机 B (EID: 10.2.1.20)
        │                                      │
   [xTR-1 / RLOC: 200.1.1.1]           [xTR-2 / RLOC: 203.2.2.2]
        │                                      │
        └────── 互联网（只路由 RLOC）───────────┘
                        │
                 [映射系统 MS/MR]
                 10.1.1.0/24 → 200.1.1.1
                 10.2.1.0/24 → 203.2.2.2
```

**转发过程**：
```
1. A 发包给 B（目的 EID 10.2.1.20）
2. ITR (xTR-1) 收到，查本地映射缓存 → 没有
3. ITR 向映射系统查询："10.2.1.20 在哪？"
4. 映射系统回答："在 RLOC 203.2.2.2"
5. ITR 封装：外层 IP = 200.1.1.1 → 203.2.2.2（LISP 用 UDP 4341）
6. 互联网正常路由这个包（只需要知道 RLOC 的路由）
7. ETR (xTR-2) 收到，解封装，把原始包交给 B
```

**关键**：**互联网核心只需要路由 RLOC（数量少、稳定），不需要知道任何 EID**。EID 的位置变化只影响映射系统，不影响全球路由表。

### 4.3 LISP 角色

| 角色 | 全称 | 作用 |
|:--|:--|:--|
| **ITR** | Ingress Tunnel Router | 入口，**封装**（查映射，加 LISP 头） |
| **ETR** | Egress Tunnel Router | 出口，**解封装** |
| **xTR** | — | 同时是 ITR 和 ETR（通常一台设备两个角色都有） |
| **MS** | Map-Server | **存储** EID → RLOC 映射（ETR 向它注册） |
| **MR** | Map-Resolver | **响应查询**（ITR 向它提问） |
| **PxTR** | Proxy xTR | 让非 LISP 网络也能访问 LISP 站点 |
| **ALT** | Alternative Topology | 映射数据库的分布式组织方式 |

> **记忆**：**I**ngress = **I**n = 进入隧道 = 封装；**E**gress = **E**xit = 离开隧道 = 解封装。

### 4.4 LISP 的应用场景

| 场景 | 价值 |
|:--|:--|
| **SD-Access** | ★ **Cisco 园区网 SDN 的控制平面就是 LISP** |
| 数据中心 VM 迁移 | VM 换机房，EID 不变，只更新映射 |
| 移动性 | 设备跨网络移动保持 IP 不变 |
| 多归属 | 不需要 BGP 和 PI 地址就能实现 |
| IPv6 过渡 | EID 用 IPv6，RLOC 用 IPv4（或反之） |

**SD-Access 中的映射**：

```
   LISP    → 控制平面（Control Plane）：谁在哪里
   VXLAN   → 数据平面（Data Plane）：怎么传输
   TrustSec → 策略平面（Policy Plane）：谁能访问谁（SGT）
```

**这三者的组合就是 SD-Access 的技术底座**，详见 [第 15 章](15-SD-Access与SD-WAN.md)。

### 4.5 LISP 简要配置

```cisco
! ── xTR 配置 ──
xTR-1(config)# router lisp
xTR-1(config-router-lisp)#  locator-set MY-RLOC
xTR-1(config-router-lisp-locator-set)#   200.1.1.1 priority 1 weight 100
xTR-1(config-router-lisp-locator-set)#   exit

xTR-1(config-router-lisp)#  eid-table default instance-id 0
xTR-1(config-router-lisp-eid-table)#   database-mapping 10.1.1.0/24 locator-set MY-RLOC
xTR-1(config-router-lisp-eid-table)#   exit

xTR-1(config-router-lisp)#  ipv4 itr map-resolver 100.1.1.1
xTR-1(config-router-lisp)#  ipv4 etr map-server 100.1.1.1 key MyLispKey
xTR-1(config-router-lisp)#  ipv4 itr
xTR-1(config-router-lisp)#  ipv4 etr

! ── 查看 ──
xTR-1# show lisp session
xTR-1# show lisp site
xTR-1# show ip lisp map-cache
xTR-1# show ip lisp database
xTR-1# lig 10.2.1.20                    ! LISP Internet Groper，查询某个 EID
```

---

## ⑤ 配套实验：理解 VXLAN 封装

**由于完整的 VXLAN + EVPN 实验需要 Nexus 或 Catalyst 9000 镜像，这里提供一个概念验证实验和一个抓包分析任务。**

### 实验 A：用 Linux 建 VXLAN 隧道（最容易上手）

这个实验用两台 Linux 主机就能做，能直观理解 VXLAN 的封装。

```bash
# ── 主机 A (物理 IP 192.168.1.10) ──
ip link add vxlan10 type vxlan id 10000 \
    remote 192.168.1.20 dstport 4789 dev eth0
ip addr add 10.1.1.10/24 dev vxlan10
ip link set vxlan10 up

# ── 主机 B (物理 IP 192.168.1.20) ──
ip link add vxlan10 type vxlan id 10000 \
    remote 192.168.1.10 dstport 4789 dev eth0
ip addr add 10.1.1.20/24 dev vxlan10
ip link set vxlan10 up

# ── 验证 ──
# 在主机 A 上
ping 10.1.1.20              # 应该通

# ── 抓包看封装（在物理接口上抓）──
tcpdump -i eth0 -nn udp port 4789 -v
```

**你会看到**：
```
IP 192.168.1.10.51234 > 192.168.1.20.4789: VXLAN, flags [I] (0x08), vni 10000
IP 10.1.1.10 > 10.1.1.20: ICMP echo request
   ↑ 外层是物理网络的 IP + UDP 4789
   ↑ 内层是虚拟网络的 IP
```

**这就是 Overlay 的全部秘密**：内层网络完全不知道外层的存在。

### 实验 B：MTU 验证

```bash
# 不调 MTU 时
ping -M do -s 1472 10.1.1.20    # 1472+28=1500，加 VXLAN 50 字节 = 1550
# 如果物理网络 MTU 是 1500 → 失败

# 查看 vxlan 接口的 MTU
ip link show vxlan10
# 通常自动设成 1450 (1500-50)

# 物理网络开巨帧后
ip link set eth0 mtu 9000
ip link set vxlan10 mtu 8950
ping -M do -s 8900 10.1.1.20    # 现在可以了
```

### 实验 C：抓包分析任务

用 Wireshark 打开一个 VXLAN 抓包文件（或自己抓的），完成下面的分析表：

| 层次 | 字段 | 值 |
|:--|:--|:--|
| 外层以太网 | 源/目的 MAC | ? |
| 外层 IP | 源/目的 IP | ? |
| UDP | 源端口 / 目的端口 | ? / **4789** |
| VXLAN | **VNI** | ? |
| VXLAN | Flags | 0x08 (I 位置位) |
| 内层以太网 | 源/目的 MAC | ? |
| 内层 IP | 源/目的 IP | ? |

**思考题**：
1. 外层 UDP 的**源端口**是怎么来的？为什么它很重要？
2. 如果 Underlay 有 4 条 ECMP 路径，同一对 VM 之间的多条 TCP 流会走同一条路径吗？

<details><summary>答案</summary>

**1.** 外层 UDP 源端口是**根据内层帧的哈希值生成的**（通常哈希内层的源/目的 MAC、IP、端口）。

**为什么重要**：Underlay 的 ECMP 负载均衡是基于五元组哈希的。如果所有 VXLAN 包的 UDP 源端口都相同，五元组中只有源端口这一项能提供熵值——固定的话，**所有 VXLAN 流量会哈希到同一条路径**，其他 ECMP 路径闲置。

用内层流的哈希做源端口，保证了**不同的内层流会被分散到不同的 Underlay 路径**。

**2.** **不会走同一条路径。**

不同的 TCP 流有不同的源端口，内层哈希不同 → 外层 UDP 源端口不同 → Underlay 的 ECMP 哈希结果不同 → 走不同路径。

**但同一条 TCP 流的所有包会走同一条路径**（内层五元组固定 → 外层源端口固定 → 哈希结果固定），这保证了**不会乱序**。

**这个设计很精妙**：既实现了负载均衡，又保证了单流的顺序性。跟 EtherChannel 的按流哈希是同一个思路。
</details>

---

## ⑥ 排障速查

| 症状 | 怀疑点 | 检查 |
|:--|:--|:--|
| VXLAN 建立不了 | Underlay 不通 | `ping <对端 Loopback>`，检查 OSPF/BGP |
| VTEP peer 是 down | Loopback 没通告 | `show nve peers`、检查路由 |
| 小包通大包不通 | **MTU** | Underlay 必须支持 ≥1550，建议 9000 |
| 同 VNI 内不通 | VLAN↔VNI 映射 | `show nve vni`、`show vlan` |
| MAC 学不到 | EVPN 控制平面 | `show bgp l2vpn evpn`、`show l2route evpn mac all` |
| 流量走单条路径 | ECMP 哈希 | 检查 UDP 源端口是否变化 |
| 跨 VNI 不通 | 需要 IRB/L3VNI | 检查三层网关配置 |
| LISP 封装不了 | 映射查询失败 | `show ip lisp map-cache`、`lig <EID>` |

---

## ⑦ 考点提示 + 自测题

### 考点

- **VXLAN 的 VNI 是 24 位 = 1600 万个**，解决 VLAN 4094 不够的问题。
- **UDP 目的端口 4789**，封装开销 **50 字节**。
- **VTEP** 的作用。
- **Underlay vs Overlay** 的概念区分。
- **VXLAN 需要 Underlay 支持大 MTU（≥1550，建议巨帧）**。
- **LISP 分离 EID（身份）和 RLOC（位置）**。
- **ITR/ETR/xTR/MS/MR** 各自的角色。
- **SD-Access = LISP（控制面）+ VXLAN（数据面）+ TrustSec（策略面）**。

### 自测题

**1.** VXLAN 的 VNI 是多少位？它解决了 VLAN 的什么问题？

<details><summary>答案</summary>

**VNI（VXLAN Network Identifier）是 24 位**，可标识 **2^24 = 16,777,216（约 1600 万）** 个虚拟网络。

**解决的三个问题**：

**① 标识符数量不足**
- VLAN 的 VID 只有 **12 位 = 4094 个**可用
- 在大型数据中心和公有云的多租户场景下远远不够（一个中等规模的云可能有上万租户，每个至少需要几个网段）
- VXLAN 的 1600 万个彻底解决了这个问题

**② 二层域无法跨三层网络**
- VLAN 只能在同一个二层广播域内有效，跨路由器就失效了
- VXLAN 把二层帧封装进 **UDP（端口 4789）**，可以在任意三层网络上传输
- 这让"虚拟机跨机架、跨机房迁移时 IP 不变"成为可能

**③ STP 导致的带宽浪费**
- 传统大二层网络依赖 STP 防环，必然有链路被阻塞（可能浪费 50% 带宽）
- VXLAN 的 Underlay 是纯三层网络（OSPF/BGP + ECMP），**所有链路 100% 利用**
- 收敛也更快（路由协议 vs STP）

**代价**：
- **50 字节封装开销** → Underlay 必须支持更大的 MTU（≥1550，实践中直接开巨帧 9000）
- 需要控制平面来学习 MAC ↔ VTEP 映射（现代方案用 **BGP EVPN**）
- 排障更复杂（要同时看 Underlay 和 Overlay 两层）

**记忆这条演进链**：
```
VLAN 12 位 4094 个不够 ──> VXLAN VNI 24 位 1600 万
VLAN 跨不了三层        ──> VXLAN 封装进 UDP 4789
STP 浪费一半带宽       ──> 三层 Underlay + ECMP
```
</details>

**2.** Underlay 和 Overlay 分别是什么？为什么要分开？

<details><summary>答案</summary>

| | **Underlay（底层网络）** | **Overlay（叠加网络）** |
|:--|:--|:--|
| 是什么 | 真实的物理网络 | 在物理网络之上虚拟出的逻辑网络 |
| 目标 | **设备之间 IP 可达** | **业务之间连通 + 策略** |
| 协议 | OSPF / IS-IS / BGP | VXLAN / GRE / LISP / MPLS |
| 变化频率 | **低**（物理拓扑很少变） | **高**（业务天天变） |
| 关注点 | 稳定、快速收敛、ECMP | 隔离、迁移、策略、多租户 |

**为什么要分开——核心是"解耦"**：

**① 物理网络可以保持简单稳定**
Underlay 只需要做一件事：让每台设备的 Loopback 互通。配置极简（一个路由协议 + ECMP），几乎不需要改动。

**② 业务变化不需要动物理网络**
- 新增一个租户 → 加一个 VNI，Underlay 完全不知情
- VM 迁移 → 更新 Overlay 的映射，Underlay 完全不知情
- 调整隔离策略 → 改 Overlay 策略，Underlay 完全不知情

**在传统架构里，这些操作可能需要在多台交换机上改 VLAN、改 Trunk、改 STP——牵一发动全身。**

**③ 故障域清晰**
排障时可以分层：
```
先测 Underlay: ping <对端 Loopback>
   通了 → 问题在 Overlay（VNI 映射、EVPN、策略）
   不通 → 问题在 Underlay（路由、链路、MTU）
```

**④ 各自可以独立演进**
Underlay 从 10G 升级到 100G，Overlay 完全无感。Overlay 从 Flood&Learn 升级到 EVPN，Underlay 完全无感。

**这个思想的普适性**（值得记住）：

| 领域 | Underlay | Overlay |
|:--|:--|:--|
| 数据中心 | Spine-Leaf 三层网络 | VXLAN + EVPN |
| 园区网 | 三层路由网络 | SD-Access (LISP + VXLAN) |
| 广域网 | 互联网 / MPLS | SD-WAN (IPsec 隧道) |
| 容器 | 主机网络 | Kubernetes CNI (Calico/Flannel) |
| 云 | 物理数据中心网络 | VPC / 虚拟网络 |

**"用一层简单稳定的传输网，承载多个灵活多变的逻辑网"——这是现代网络架构最重要的一个思想。**
</details>

**3.** VXLAN 部署时，为什么必须先检查 MTU？

<details><summary>答案</summary>

**因为 VXLAN 封装增加 50 字节开销，如果 Underlay 的 MTU 不够，所有大包都会被静默丢弃。**

**开销明细**：
```
外层以太网头     14 字节
外层 IP 头       20 字节
UDP 头            8 字节
VXLAN 头          8 字节
─────────────────────────
总计             50 字节
```

**后果**：
```
虚拟机发出 1500 字节的包
+ 50 字节 VXLAN 封装
= 1550 字节
> Underlay 接口 MTU 1500
→ 被丢弃
```

**症状**（非常典型且隐蔽）：
- ✅ ping 通（小包）
- ✅ SSH 能连（小包）
- ❌ **网页打不开、大文件传不动、数据库同步失败**（大包）
- ❌ 症状随机出现，很难复现

**这和 GRE/IPsec 的 MTU 问题是同一类**，但 VXLAN 更严重，因为它是数据中心的基础设施，影响面大。

**解决方案**：

```cisco
! 方案 1（★ 推荐）：Underlay 开启巨帧
Leaf-1(config)# system jumbomtu 9216            ! Nexus
Leaf-1(config)# interface Ethernet1/1
Leaf-1(config-if)# mtu 9216

Leaf-1(config)# system mtu 9198                  ! Catalyst 9000

! 方案 2：最低要求，把 Underlay MTU 设到 1600
Leaf-1(config-if)# mtu 1600

! 方案 3（兜底）：在虚拟机侧降低 MTU
! Linux: ip link set eth0 mtu 1450
! 但这需要改所有虚拟机，不现实
```

**为什么推荐直接开巨帧 9000**：
1. 一劳永逸，未来叠加更多封装（VXLAN + IPsec + MPLS）也不怕
2. 巨帧本身能提升大流量场景的性能（更少的包头开销、更少的中断）
3. 数据中心内部没有 MTU 协商的兼容性问题（都是自己的设备）

**⚠️ 必须全路径一致**：Underlay 路径上的**每一台设备、每一个接口**都要配。有一台没配，问题依然存在。

```cisco
! 检查全路径 MTU
Leaf-1# ping 2.2.2.2 size 9000 df-bit
Leaf-1# show interface Ethernet1/1 | include MTU
```

> **VXLAN 部署检查清单第一条永远是 MTU。** 我见过太多团队在 VXLAN 上线后遇到"随机的业务异常"，查了几天最后发现是某一台 Spine 的某个接口忘了配 MTU。
</details>

**4.** LISP 中的 EID 和 RLOC 分别是什么？为什么要分离？

<details><summary>答案</summary>

| | **EID** (Endpoint Identifier) | **RLOC** (Routing Locator) |
|:--|:--|:--|
| 含义 | **身份**——"我是谁" | **位置**——"我在哪" |
| 谁在用 | 终端主机（VM、PC、手机） | 网络设备（xTR 路由器） |
| 变化 | **移动时不变** | 随位置改变 |
| 谁知道它 | 只有 LISP 站点内部和映射系统 | **互联网核心路由** |

**为什么要分离——传统 IP 的根本矛盾**：

传统 IP 地址**同时**扮演两个角色：
```
10.1.1.10 既表示"这台主机的身份"
          又表示"它在 10.1.1.0/24 这个网段里，通过某台路由器可达"
```

**这个二合一带来三个问题**：

**① 移动性差**
设备移动到另一个网络 → 位置变了 → 必须换 IP → **但换 IP 等于换身份** → 所有基于 IP 的 TCP 连接、会话、ACL 全部失效。

手机从 WiFi 切换到 4G 时，所有下载中断、视频卡住——就是这个原因。

**② 全球路由表爆炸**
为了支持多归属（一个企业接两个 ISP）和流量工程，企业需要向互联网注入自己的明细路由。结果是**全球 BGP 路由表已超过 90 万条**，而且还在增长。核心路由器的 FIB 内存和收敛时间都承受着巨大压力。

**③ 多归属复杂**
想要多归属，企业必须：申请 AS 号 → 申请 PI（提供商无关）地址 → 跑 BGP → 和两个 ISP 建立对等。门槛极高。

**LISP 的解法**：

```
   主机的 EID 是 10.1.1.10  ← 永远不变，就是它的身份
        │
   [xTR，RLOC = 200.1.1.1]  ← 位置，会变
        │
   映射系统：10.1.1.10 → 200.1.1.1
        │
   互联网只路由 RLOC（数量少、稳定）
```

**带来的好处**：

| 好处 | 说明 |
|:--|:--|
| **移动不换 IP** | 主机移动到新位置，EID 不变，**只需更新映射系统里的 RLOC** |
| **路由表瘦身** | 互联网核心只需知道 RLOC 的路由（运营商聚合后数量很少），**EID 完全不进全球路由表** |
| **多归属简化** | 一个 EID 可以映射到多个 RLOC（带 priority/weight），**不需要 BGP 和 AS 号** |
| **IPv6 过渡** | EID 用 IPv6，RLOC 用 IPv4（或反之），实现平滑过渡 |
| **按需查询** | ITR 只在需要时查询映射并缓存，**不需要预先知道全网信息** |

**实际应用**：

**Cisco SD-Access 的控制平面就是 LISP**：
```
LISP     → 控制平面：Control Plane Node 维护"哪个终端在哪个交换机后面"
VXLAN    → 数据平面：实际的流量封装传输
TrustSec → 策略平面：SGT 标签决定谁能访问谁
```

在 SD-Access 里，**终端在整个园区内任意移动，IP 和安全策略都不变**——这正是 LISP 身份/位置分离的直接价值。

**类比**：
- **传统 IP** = 你的身份证号是"某某小区3栋502"。搬家就得换身份证号，所有认识你的人都要重新认识你。
- **LISP** = 身份证号（EID）永远不变；搬家只需去派出所更新住址（RLOC）；别人找你时先查派出所的登记（映射系统）。
</details>

**5.** SD-Access 用到了本章的哪些技术？各自扮演什么角色？

<details><summary>答案</summary>

**SD-Access 是 LISP + VXLAN + TrustSec 的组合**：

| 技术 | 平面 | 角色 |
|:--|:--|:--|
| **LISP** | **控制平面** | 维护"哪个终端（EID）在哪个交换机（RLOC）后面"的映射。终端接入或移动时更新映射 |
| **VXLAN** | **数据平面** | 实际的流量封装。VNI 区分不同的虚拟网络（VN），跨三层 Fabric 传输 |
| **Cisco TrustSec (SGT)** | **策略平面** | 给每个终端打上**安全组标签（SGT）**，策略基于 SGT 而不是 IP |
| **IS-IS**（通常） | **Underlay** | Fabric 内部的三层路由，保证各节点 Loopback 互通 |

**架构角色**：

```
        ┌──────────────────────────────────────┐
        │       DNA Center（管理与编排）         │
        └──────────────────┬───────────────────┘
                           │
        ┌──────────────────▼───────────────────┐
        │  Control Plane Node（LISP Map-Server）│
        │  维护 EID → RLOC 映射                  │
        └──────────────────┬───────────────────┘
                           │
   ┌───────────────────────┼───────────────────────┐
   │                       │                       │
┌──▼──────────┐    ┌───────▼──────┐    ┌───────────▼──┐
│ Edge Node   │    │  Edge Node   │    │ Border Node  │
│ (接入交换机) │    │              │    │ (对接外部网络) │
│ VTEP + xTR  │    │              │    │              │
└──┬──────────┘    └──────────────┘    └──────────────┘
   │
 [终端]
```

| 节点角色 | 职责 |
|:--|:--|
| **Control Plane Node** | LISP 的 MS/MR，维护终端映射数据库 |
| **Edge Node** | 接入交换机，是 VTEP（VXLAN 封装）+ xTR（LISP 封装），终端的默认网关 |
| **Border Node** | Fabric 与外部网络的边界，做路由泄露和 SGT 转换 |
| **Fabric WLC / AP** | 无线接入，AP 也成为 Fabric 的一部分 |
| **DNA Center** | 图形化管理、自动化配置下发、策略定义、遥测分析 |

**这个组合解决了什么**：

| 传统园区网的痛点 | SD-Access 的解法 |
|:--|:--|
| VLAN 数量有限、跨设备麻烦 | **VXLAN VNI**，1600 万个，跨三层自由延伸 |
| 终端移动要换 IP/VLAN | **LISP**，身份与位置分离，移动时 IP 和策略不变 |
| 策略基于 IP/VLAN，地址一变策略就失效 | **SGT**，策略基于身份标签，与 IP 完全解耦 |
| STP 浪费带宽、收敛慢 | **三层 Underlay + ECMP**，无 STP |
| 配置分散在几十台设备上，容易出错 | **DNA Center 集中编排**，意图驱动 |

**举个具体例子**：

一个员工的笔记本从 3 楼移动到 8 楼：
- **传统网络**：换了接入交换机 → 换 VLAN → 换 IP → 基于 IP 的 ACL 全部失效 → 需要重新配策略
- **SD-Access**：Edge Node 检测到终端接入 → 向 Control Plane Node 更新 LISP 映射 → **IP 不变、SGT 不变、策略自动跟随**

**知识点串联**（这是 ENCOR 最喜欢考的"技术如何组合"）：

```
第 1 章：Spine-Leaf 架构      → SD-Access 的物理拓扑基础
第 2 章：VRF                  → SD-Access 的 Virtual Network (VN) 就是 VRF
第 3 章：VXLAN + LISP         → 数据平面 + 控制平面
第 13 章：TrustSec / 802.1X   → 策略平面 + 终端认证
第 14 章：DNA Center / API    → 自动化编排
第 15 章：SD-Access 整合      → 全部组装起来
```

**所以这一章不是孤立的知识点，它是理解第 15 章的必要前提。**
</details>

---

**上一章** ← [02 网络虚拟化：VRF/GRE/IPsec](02-网络虚拟化-VRF-GRE-IPsec.md) ｜ **下一章** → [04 交换进阶：STP 进阶与排障](04-交换进阶-STP进阶与排障.md)
