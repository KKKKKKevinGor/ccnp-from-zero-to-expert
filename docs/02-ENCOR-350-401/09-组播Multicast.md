# 09 · 组播 Multicast

## ① 这章解决什么问题

公司要做全员直播，1000 人同时观看一路 2Mbps 的视频流。

**用单播（Unicast）**：服务器要发 1000 份相同的数据 → **2Gbps 出口带宽**，服务器和网络都扛不住。

**用广播（Broadcast）**：所有人都收到，包括不想看的人；而且**广播跨不了三层**，只能在本网段。

**用组播（Multicast）**：服务器**只发一份**，网络设备在分叉点**复制**给需要的接收者 → **2Mbps 出口带宽**，且只有订阅者收到。

组播在 ENCOR 里权重不高（★★），但它是**视频会议、IPTV、金融行情推送、Windows 部署（WDS/SCCM）** 的基础，实战中遇到组播故障时，不懂原理就完全无从下手。

---

## ② 原理讲透

### 2.1 三种通信模式对比

| | 单播 Unicast | 广播 Broadcast | **组播 Multicast** |
|:--|:--|:--|:--|
| 接收者 | 一个 | **所有人**（同一广播域） | **订阅了的那些人** |
| 源发几份 | N 份（N 个接收者） | 1 份 | **1 份** |
| 跨三层 | ✅ | ❌ | ✅ |
| 带宽效率 | 差（N 倍） | 中 | **★ 最好** |
| 传输层 | TCP/UDP | UDP | **只能 UDP** |
| 典型应用 | Web、SSH | ARP、DHCP | 视频直播、IPTV、行情推送、路由协议 |

> **组播只能用 UDP**：因为 TCP 是面向连接、需要确认和重传的，而组播是"一对多、不知道有多少接收者"的模型，无法维护连接状态。这也意味着**组播不保证可靠传输**——丢了就丢了，靠应用层自己处理（比如 FEC 前向纠错）。

### 2.2 组播地址

**IPv4 组播地址范围：`224.0.0.0 – 239.255.255.255`（D 类，`224.0.0.0/4`）**

| 范围 | 名称 | 用途 |
|:--|:--|:--|
| **224.0.0.0 – 224.0.0.255** | **本地链路组播** | **不被路由器转发**（TTL=1） |
| 224.0.1.0 – 238.255.255.255 | 全局范围 | 互联网组播 |
| **232.0.0.0/8** | **SSM（源特定组播）** | 指定源 |
| 233.0.0.0/8 | GLOP | 基于 AS 号分配 |
| **239.0.0.0/8** | **管理范围（私有）** | ★ **企业内部使用**（相当于组播的私网地址） |

**必背的本地链路组播地址**：

| 地址 | 用途 |
|:--|:--|
| **224.0.0.1** | **所有主机** |
| **224.0.0.2** | **所有路由器** |
| **224.0.0.5** | **OSPF 所有路由器** |
| **224.0.0.6** | **OSPF DR/BDR** |
| **224.0.0.9** | RIPv2 |
| **224.0.0.10** | **EIGRP** |
| **224.0.0.13** | **PIM** |
| **224.0.0.18** | **VRRP** |
| 224.0.0.22 | IGMPv3 |
| 224.0.0.102 | HSRPv2 / GLBP |

> **企业内部部署组播，用 `239.0.0.0/8`**（管理范围）。这相当于组播世界的 RFC1918 私网地址，不会和互联网组播冲突。

### 2.3 组播 MAC 地址映射（★ 考点）

组播 IP 需要映射到组播 MAC 才能在二层传输。

**映射规则**：
```
固定前缀：01:00:5E              （25 位）
+ 组播 IP 的【低 23 位】        （23 位）
= 48 位 MAC 地址
```

**示例：`239.1.1.1`**
```
239 . 1 . 1 . 1
239 = 11101111
1   = 00000001
1   = 00000001
1   = 00000001

取低 23 位：0000001 00000001 00000001
                ↑ 第二字节的最高位被丢弃

MAC = 01:00:5E:01:01:01
```

**★ 关键问题：32:1 的地址重叠**

组播 IP 有 28 位可变（32 位 − 4 位的 `1110` 前缀），但只有 **23 位**映射进 MAC。

**28 − 23 = 5 位丢失 → 2^5 = 32 个不同的组播 IP 映射到同一个 MAC 地址。**

```
239.1.1.1    →  01:00:5E:01:01:01
239.129.1.1  →  01:00:5E:01:01:01     ← 相同！
224.1.1.1    →  01:00:5E:01:01:01     ← 也相同！
225.129.1.1  →  01:00:5E:01:01:01     ← 还是相同！
```

**后果**：主机订阅了 `239.1.1.1`，但会在二层收到 `239.129.1.1` 的流量（因为 MAC 相同），只能在三层丢弃——**浪费主机 CPU 和网络带宽**。

**规避方法**：**规划组播地址时，避免第二字节相差 128 的组合**。比如统一用 `239.1.x.x`，不要同时用 `239.129.x.x`。

### 2.4 IGMP：主机怎么告诉网络"我要订阅"

**IGMP (Internet Group Management Protocol)** 运行在**主机和本地路由器之间**。

| 版本 | 关键特性 |
|:--|:--|
| **IGMPv1** | 只有 Query 和 Report，**离开组要靠超时**（最长 3 分钟），已淘汰 |
| **IGMPv2**（最常用） | 增加 **Leave Group 消息**（主动离开，快速停止）+ 查询器选举 |
| **IGMPv3** | 支持 **SSM（源特定组播）**：可以指定"我只要来自源 A 的 239.1.1.1" |

**IGMPv2 工作流程**：
```
   路由器                                    主机
     │                                        │
     │  ① Membership Query（周期性，60秒）      │
     │─────────── 224.0.0.1 ─────────────────>│  "有人要收组播吗？"
     │                                        │
     │  ② Membership Report                   │
     │<───────────────────────────────────────│  "我要 239.1.1.1"
     │                                        │
     │        （开始转发组播流量）               │
     │                                        │
     │  ③ Leave Group                         │
     │<─────────── 224.0.0.2 ─────────────────│  "我不要了"
     │                                        │
     │  ④ Group-Specific Query（确认还有人吗）  │
     │──────────────────────────────────────>│
     │  （无人响应 → 停止转发）                  │
```

**Report 抑制机制**：如果主机 A 已经发了 Report，主机 B 听到后**就不再发**（避免重复）。所以路由器只知道"这个网段有人要"，**不知道具体有几个人**。

### 2.5 IGMP Snooping（二层优化，★ 实战必配）

**问题**：交换机是二层设备，看到组播帧（目的 MAC 是组播地址）**默认会像广播一样泛洪到所有端口**。

```
   没有 IGMP Snooping：
   
   组播源 ──> [交换机] ──┬──> PC1（订阅了）✓
                        ├──> PC2（没订阅）← 白白收到，浪费带宽
                        ├──> PC3（没订阅）← 白白收到
                        └──> PC4（没订阅）← 白白收到
                        
   一路 10Mbps 的视频流会占满所有端口
```

**IGMP Snooping** 让交换机"偷听" IGMP 报文，学习**哪个端口下有订阅者**，只往那些端口转发。

```cisco
! Cisco 默认开启
SW1(config)# ip igmp snooping
SW1(config)# ip igmp snooping vlan 10

! 配置查询器（★ 没有三层路由器时必须配）
SW1(config)# ip igmp snooping querier
SW1(config)# ip igmp snooping vlan 10 querier address 192.168.10.1

! 查看
SW1# show ip igmp snooping
SW1# show ip igmp snooping groups
SW1# show ip igmp snooping mrouter
```

> **⚠️ 最常见的组播故障**：二层网络里**没有三层设备做 IGMP 查询器**。没有周期性的 Query，交换机的 IGMP Snooping 表项会**老化**，然后要么停止转发（视频卡住），要么退化成泛洪。
>
> **解法**：在核心交换机上配 `ip igmp snooping querier`，让它扮演查询器的角色。**这是纯二层组播部署的必配项。**

### 2.6 PIM：路由器之间怎么建组播转发树

**PIM (Protocol Independent Multicast)** 运行在**路由器之间**，负责建立组播分发树。

**"Protocol Independent" 的含义**：PIM 不自己算路由，**直接用现有的单播路由表**（OSPF/EIGRP/BGP 算出来的）做 RPF 检查。

#### RPF 检查（★ 组播的核心机制）

**RPF (Reverse Path Forwarding) 检查**：

```
   收到一个组播包
        ↓
   查单播路由表：到【组播源】的最优路径，出接口是哪个？
        ↓
   ★ 这个包是从那个接口收到的吗？★
        ├─ 是 → RPF 检查通过 → 转发
        └─ 否 → ★ 丢弃 ★
```

**为什么需要 RPF**：**防止组播环路**。

组播是"一对多"的树状转发，如果没有 RPF 检查，一个包可能在环路里无限复制（比二层广播风暴更可怕，因为每一跳都会复制多份）。

RPF 保证**组播流量只沿着"回到源的最短路径"的反方向传播**，天然形成一棵无环的树。

**RPF 失败是组播排障的头号问题**：
```cisco
R1# show ip rpf 192.168.1.100
RPF information for ? (192.168.1.100)
  RPF interface: GigabitEthernet0/1
  RPF neighbor: ? (10.0.12.2)
  RPF route/mask: 192.168.1.0/24
  RPF type: unicast (ospf 1)
  RPF recursion count: 0
  Doing distance-preferred lookups across tables
```

**常见的 RPF 失败原因**：
1. **单播路由和组播流量路径不一致**（非对称路由）
2. 单播路由指向了错误的接口
3. 隧道场景（GRE 隧道的单播路由和实际组播路径不同）

**解法**：
```cisco
! 配置静态组播路由，覆盖 RPF 检查
R1(config)# ip mroute 192.168.1.0 255.255.255.0 10.0.13.3
!                    ↑ 源网段              ↑ 组播的 RPF 邻居
```

#### PIM 的三种模式

| 模式 | 全称 | 机制 | 适用 |
|:--|:--|:--|:--|
| **PIM-DM** | Dense Mode（密集模式） | **推 (Push)**：先泛洪到所有地方，不要的再剪枝（Prune） | 接收者密集、小网络 |
| **PIM-SM** | Sparse Mode（稀疏模式） | **拉 (Pull)**：默认不发，有人明确请求（Join）才发 | ★ **接收者稀疏、大网络（主流）** |
| **PIM-SSM** | Source-Specific Multicast | 接收者指定"我要来自源 A 的组 G" | IPTV、单一已知源 |
| PIM-BIDIR | Bidirectional | 双向树，适合多源多接收者 | 视频会议 |

**PIM-DM 的问题**：
- 泛洪-剪枝周期（默认 **3 分钟**）会周期性重新泛洪一遍，**周期性占用全网带宽**
- 扩展性差
- **实际部署几乎不用**，主要是考点

**PIM-SM 是主流**，但它需要一个关键角色：**RP**。

### 2.7 RP（汇聚点）与共享树

**PIM-SM 的核心问题**：接收者想收 `239.1.1.1`，但**它不知道源在哪里**。怎么找？

**解法：设立一个"中介"——RP (Rendezvous Point，汇聚点)。**

```
   ① 源开始发送组播
        ↓
   源的第一跳路由器（DR）把流量【注册】到 RP
   （用 PIM Register 消息，单播封装发给 RP）
        ↓
   ② 接收者发 IGMP Report
        ↓
   接收者的第一跳路由器向 ★ RP ★ 发送 PIM Join
        ↓
   ③ 建立【共享树 RPT】：源 → RP → 接收者
        ↓
   ④ 流量开始流动后，接收者的路由器发现
      "直接到源的路径更短"
        ↓
   ⑤ ★ SPT 切换 ★：切换到【最短路径树 SPT】：源 → 接收者（不经 RP）
```

**两种树**：

| 树 | 记法 | 说明 |
|:--|:--|:--|
| **共享树 RPT** | **`(*, G)`** | 星号表示"任意源"，**以 RP 为根** |
| **最短路径树 SPT** | **`(S, G)`** | S 是具体的源，**以源为根，路径最短** |

```cisco
R1# show ip mroute
(*, 239.1.1.1), 00:05:23/00:02:45, RP 10.0.0.1, flags: SJC
                                   ↑ 共享树，指向 RP
  Incoming interface: GigabitEthernet0/1, RPF nbr 10.0.12.2
  Outgoing interface list:
    GigabitEthernet0/2, Forward/Sparse, 00:05:23/00:02:45

(192.168.1.100, 239.1.1.1), 00:03:12/00:02:50, flags: JT
 ↑ 具体的源                                            ↑ T = 已切换到 SPT
  Incoming interface: GigabitEthernet0/3, RPF nbr 10.0.13.3
  Outgoing interface list:
    GigabitEthernet0/2, Forward/Sparse, 00:03:12/00:02:50
```

**重要标志位**：

| 标志 | 含义 |
|:--|:--|
| `S` | Sparse Mode |
| `D` | Dense Mode |
| **`T`** | **SPT-bit set（已切换到最短路径树）** |
| `J` | Join SPT（正在切换） |
| `C` | Connected（有直连的接收者） |
| `L` | Local（本机是接收者） |
| `P` | Pruned（已剪枝） |
| `F` | Register flag（源的 DR） |

### 2.8 RP 的三种配置方式

| 方式 | 配置 | 优缺点 |
|:--|:--|:--|
| **静态 RP** | 每台路由器手工指定 | ✅ 简单、可预测<br>❌ 无冗余，改动要动所有设备 |
| **Auto-RP**（Cisco 私有） | 候选 RP 通过 `224.0.1.39/40` 通告 | ✅ 自动<br>❌ Cisco 私有 |
| **BSR**（标准，PIMv2） | Bootstrap Router 机制 | ✅ **标准协议，推荐** |

```cisco
! ── ① 静态 RP（最简单，小网络够用）──
R1(config)# ip multicast-routing
R1(config)# ip pim rp-address 10.0.0.1
! 所有路由器都要配同样的一条

! ── ② Auto-RP ──
! 候选 RP
RP(config)# ip pim send-rp-announce Loopback0 scope 10
! 映射代理
MA(config)# ip pim send-rp-discovery Loopback0 scope 10
! 所有路由器（需要监听 Auto-RP 组）
R1(config)# ip pim autorp listener

! ── ③ BSR（标准，推荐）──
RP(config)# ip pim rp-candidate Loopback0 priority 0
BSR(config)# ip pim bsr-candidate Loopback0 0 priority 100

! ── 接口启用 PIM ──
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip pim sparse-mode
! 或 sparse-dense-mode（Auto-RP 场景常用）

! ── 查看 ──
R1# show ip pim rp mapping
R1# show ip pim neighbor
R1# show ip pim interface
R1# show ip mroute
R1# show ip mroute count
R1# show ip rpf <源IP>
R1# show ip igmp groups
R1# show ip igmp interface
```

---

## ③ 配套实验：PIM-SM 组播转发

**拓扑**：
```
   [组播源]                                    [接收者]
  192.168.1.100                              192.168.3.10
       │                                          │
    ┌──▼──┐        ┌─────┐        ┌─────┐    ┌────▼──┐
    │ R1  │────────│ R2  │────────│ R3  │────│  SW   │
    │(DR) │        │(RP) │        │(DR) │    └───────┘
    └─────┘        └─────┘        └─────┘
                Lo0: 10.0.0.2
                
   组播组：239.1.1.1
```

### Step 1：基础配置

```cisco
! ── 所有路由器 ──
R1(config)# ip multicast-routing
R1(config)# ip pim rp-address 10.0.0.2           ! 静态 RP，指向 R2

! 所有参与组播的接口都要启用 PIM
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip pim sparse-mode
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip pim sparse-mode

! ── R2（RP）──
R2(config)# interface Loopback0
R2(config-if)# ip address 10.0.0.2 255.255.255.255
R2(config-if)# ip pim sparse-mode                ! ★ RP 的 Loopback 也要启用 PIM
R2(config)# ip pim rp-address 10.0.0.2
```

> **★ 常见错误**：忘了在 **RP 的 Loopback 接口**上启用 `ip pim sparse-mode`。这会导致 RP 无法正常工作，所有 Join 都失败。

### Step 2：验证 PIM 邻居

```cisco
R2# show ip pim neighbor
PIM Neighbor Table
Neighbor          Interface              Uptime/Expires    Ver   DR
Address                                                          Prio/Mode
10.0.12.1         GigabitEthernet0/1     00:15:23/00:01:32 v2    1 / S P G
10.0.23.3         GigabitEthernet0/2     00:15:20/00:01:28 v2    1 / DR S P G
                                                                   ↑ DR
```

### Step 3：验证 RP 映射

```cisco
R1# show ip pim rp mapping
PIM Group-to-RP Mappings

Group(s): 224.0.0.0/4, Static
    RP: 10.0.0.2 (?)
        ↑ 所有组播组都用这个 RP
```

**★ 所有路由器上的 RP 地址必须一致**，否则会出现"源注册到 RP-A，接收者去 RP-B 请求"的情况，永远收不到流量。

### Step 4：接收者加入组播组

```
! 在接收者 PC 上（Linux）
$ ip route add 224.0.0.0/4 dev eth0
$ socat UDP4-RECVFROM:5000,ip-add-membership=239.1.1.1:eth0,fork -

! 或用 iperf
$ iperf -s -u -B 239.1.1.1 -i 1
```

**验证 IGMP**：
```cisco
R3# show ip igmp groups
IGMP Connected Group Membership
Group Address    Interface          Uptime    Expires   Last Reporter
239.1.1.1        GigabitEthernet0/0 00:02:15  00:02:45  192.168.3.10
                                                         ↑ 接收者
```

**验证共享树建立**：
```cisco
R3# show ip mroute 239.1.1.1
(*, 239.1.1.1), 00:02:15/00:02:45, RP 10.0.0.2, flags: SJC
  Incoming interface: GigabitEthernet0/1, RPF nbr 10.0.23.2
                                          ↑ 朝 RP 的方向
  Outgoing interface list:
    GigabitEthernet0/0, Forward/Sparse, 00:02:15/00:02:45
                                        ↑ 朝接收者的方向
```

**在 RP 上验证**：
```cisco
R2# show ip mroute 239.1.1.1
(*, 239.1.1.1), 00:02:20/00:03:10, RP 10.0.0.2, flags: S
  Incoming interface: Null, RPF nbr 0.0.0.0        ← RP 是树根，没有入接口
  Outgoing interface list:
    GigabitEthernet0/2, Forward/Sparse, 00:02:20/00:03:10
```

### Step 5：源开始发送

```
! 在源 PC 上
$ iperf -c 239.1.1.1 -u -T 32 -t 300 -b 2M
```

**验证注册**：
```cisco
R1# show ip mroute 239.1.1.1
(192.168.1.100, 239.1.1.1), 00:00:15/00:03:20, flags: FT
                                                        ↑ F = 我是源的 DR，正在注册
  Incoming interface: GigabitEthernet0/0, RPF nbr 0.0.0.0
  Outgoing interface list:
    GigabitEthernet0/1, Forward/Sparse, 00:00:15/00:03:15
```

### Step 6：观察 SPT 切换（★ 核心观察点）

**刚开始（共享树）**：
```cisco
R3# show ip mroute 239.1.1.1
(*, 239.1.1.1), flags: SJC
  Incoming interface: GigabitEthernet0/1, RPF nbr 10.0.23.2    ← 经 RP
```

**几秒后（切换到 SPT）**：
```cisco
R3# show ip mroute 239.1.1.1
(*, 239.1.1.1), 00:05:23/stopped, RP 10.0.0.2, flags: SJC
  Incoming interface: GigabitEthernet0/1, RPF nbr 10.0.23.2
  Outgoing interface list:
    GigabitEthernet0/0, Forward/Sparse

(192.168.1.100, 239.1.1.1), 00:00:45/00:02:50, flags: JT
                                                       ↑ ★ T = 已切换到 SPT
  Incoming interface: GigabitEthernet0/1, RPF nbr 10.0.23.2
  Outgoing interface list:
    GigabitEthernet0/0, Forward/Sparse
```

**关掉自动 SPT 切换（观察对比）**：
```cisco
R3(config)# ip pim spt-threshold infinity
! 永远不切换，一直用共享树
```

**为什么要切换到 SPT**：共享树的路径 `源 → RP → 接收者` 可能绕远路。SPT 是 `源 → 接收者` 的最短路径，延迟更低、带宽占用更少。

**为什么不一开始就用 SPT**：因为接收者一开始**不知道源在哪**。必须先通过 RP 建立连接，收到第一个包后才知道源地址，然后才能切到 SPT。

### Step 7：故障注入

**故障 A：RPF 失败**
```cisco
! 人为制造非对称路由：让单播路由走 R2，但组播想走另一条路
R3(config)# ip route 192.168.1.0 255.255.255.0 <另一条路径>
```
**观察**：
```cisco
R3# show ip mroute count
! 收到的包数不增长

R3# show ip rpf 192.168.1.100
RPF interface: GigabitEthernet0/2         ← 单播路由指向这个接口
! 但组播流量从 Gi0/1 进来 → RPF 失败 → 丢弃

R3# debug ip mpacket
IP(0): s=192.168.1.100 (GigabitEthernet0/1) d=239.1.1.1 
       id=1234, ttl=61, prot=17, len=1050, not RPF interface
                                              ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ 找到问题
```
**修复**：
```cisco
R3(config)# ip mroute 192.168.1.0 255.255.255.0 10.0.23.2
!           静态组播路由，覆盖 RPF 检查
```

**故障 B：RP 地址不一致**
```cisco
R3(config)# ip pim rp-address 10.0.0.99      ! 错误的 RP
```
**观察**：接收者的 IGMP Report 正常，但收不到流量。
```cisco
R3# show ip pim rp mapping
    RP: 10.0.0.99 (?)                        ← 和其他路由器不一致

R3# show ip mroute 239.1.1.1
(*, 239.1.1.1), flags: SJC
  Incoming interface: Null, RPF nbr 0.0.0.0  ← 到不了这个 RP
```

**故障 C：忘了在 RP 的 Loopback 上启用 PIM**
```cisco
R2(config)# interface Loopback0
R2(config-if)# no ip pim sparse-mode
```
**观察**：整个组播完全不工作，但错误信息很不明显。
```cisco
R1# show ip mroute 239.1.1.1
(192.168.1.100, 239.1.1.1), flags: FT
! 注册一直在重试，但 RP 收不到
```

**故障 D：IGMP Snooping 没有查询器（纯二层场景）**
```cisco
SW1(config)# no ip igmp snooping querier
```
**观察**：初始能收到流量，但 **60-260 秒后视频卡住**（Snooping 表项老化）。
```cisco
SW1# show ip igmp snooping groups
! 表项消失了

SW1# show ip igmp snooping querier
! 没有查询器
```
**修复**：
```cisco
SW1(config)# ip igmp snooping querier
SW1(config)# ip igmp snooping vlan 10 querier address 192.168.10.1
```

---

## ④ 排障速查表

| 症状 | 怀疑点 | 验证命令 |
|:--|:--|:--|
| 完全收不到组播 | **RPF 失败** | `show ip rpf <源IP>`、`debug ip mpacket` |
| | RP 不一致 | `show ip pim rp mapping`（逐台对比） |
| | 接口没启 PIM | `show ip pim interface` |
| | **RP 的 Loopback 没启 PIM** | `show run interface Lo0` |
| 主机没发 IGMP Report | 应用/主机侧问题 | `show ip igmp groups`、主机抓包 |
| **二层收不到（跨交换机）** | **IGMP Snooping 无查询器** | `show ip igmp snooping querier` |
| 视频先能看后卡住 | Snooping 表项老化 | 同上，配 querier |
| 流量走了绕路 | 还在共享树 | `show ip mroute` 看有没有 `T` 标志 |
| 部分接收者收不到 | 交换机端口/VLAN | `show ip igmp snooping groups` |
| 收到了不该收的组播 | **MAC 地址重叠（32:1）** | 检查组播 IP 规划 |
| PIM 邻居建不起来 | 接口模式不匹配 | `show ip pim neighbor`、`show ip pim interface` |
| 组播占满带宽 | PIM-DM 周期性泛洪 | 改用 PIM-SM |

### 组播排障的标准流程

```
① 源在发吗？
   R-源侧# show ip mroute count
   R-源侧# show ip mroute <组地址>     ← 应该有 (S,G) 条目
   
② 接收者请求了吗？
   R-接收侧# show ip igmp groups       ← 应该看到组地址和接收者 IP
   
③ RP 一致吗？
   所有路由器# show ip pim rp mapping   ← 必须完全一致
   
④ RPF 通过吗？（★ 最常见的问题）
   R# show ip rpf <源IP>
   R# debug ip mpacket                  ← 看有没有 "not RPF interface"
   
⑤ 转发树建立了吗？
   R# show ip mroute <组地址>
   ← Incoming interface 不能是 Null
   ← Outgoing interface list 不能是空的
   
⑥ 二层转发正常吗？
   SW# show ip igmp snooping groups
   SW# show ip igmp snooping querier   ← 必须有查询器
```

---

## ⑤ 三厂商对照 + 考点自测

### 三厂商对照

| 目的 | Cisco | H3C | 华为 |
|:--|:--|:--|:--|
| 开启组播路由 | `ip multicast-routing` | `multicast routing` | `multicast routing-enable` |
| 接口启 PIM | `ip pim sparse-mode` | `pim sm` | `pim sm` |
| 静态 RP | `ip pim rp-address 10.0.0.2` | `static-rp 10.0.0.2`（pim 视图） | `static-rp 10.0.0.2` |
| BSR 候选 | `ip pim bsr-candidate Lo0` | `c-bsr 10.0.0.2` | `c-bsr 10.0.0.2` |
| RP 候选 | `ip pim rp-candidate Lo0` | `c-rp 10.0.0.2` | `c-rp 10.0.0.2` |
| IGMP Snooping | `ip igmp snooping` | `igmp-snooping enable` | `igmp-snooping enable` |
| Snooping 查询器 | `ip igmp snooping querier` | `igmp-snooping querier` | `igmp-snooping querier enable` |
| 查组播表 | `show ip mroute` | `display multicast routing-table` | `display multicast routing-table` |
| 查 RPF | `show ip rpf <源>` | `display multicast rpf-info <源>` | `display multicast rpf-info <源>` |
| 查 IGMP | `show ip igmp groups` | `display igmp group` | `display igmp group` |

### 考点

- **组播地址范围 `224.0.0.0/4`**，`224.0.0.0/24` 不被路由
- **必背地址**：224.0.0.5(OSPF)、224.0.0.10(EIGRP)、224.0.0.13(PIM)、224.0.0.18(VRRP)
- **组播 MAC 映射 `01:00:5E` + 低 23 位**，**32:1 重叠**
- **IGMPv2 有 Leave 消息，IGMPv3 支持 SSM**
- **RPF 检查用单播路由表，防环**
- **PIM-SM 是拉模式，PIM-DM 是推模式**
- **(*, G) 共享树 vs (S, G) 最短路径树**
- **IGMP Snooping 需要查询器**

### 自测题

**1.** 组播 IP `239.1.1.1` 对应的组播 MAC 是什么？为什么会有地址重叠问题？

<details><summary>答案</summary>

**MAC 地址：`01:00:5E:01:01:01`**

**计算过程**：
```
固定前缀：01:00:5E （前 25 位，最后一位固定为 0）
+ 组播 IP 的【低 23 位】

239.1.1.1 的二进制：
11101111 . 00000001 . 00000001 . 00000001
           ↑ 这一位被丢弃（只取低 23 位）

低 23 位 = 0000001 00000001 00000001
        = 01:01:01（十六进制）

MAC = 01:00:5E:01:01:01
```

**为什么会重叠（★ 考点）**：

组播 IP 的可变部分有 **28 位**（32 位减去 `1110` 这 4 位固定前缀），但只有 **23 位**被映射进 MAC。

**28 − 23 = 5 位信息丢失 → 2^5 = 32 个不同的组播 IP 映射到同一个 MAC。**

**重叠示例**（这 4 个 IP 的 MAC 完全相同）：
```
224.1.1.1    →  01:00:5E:01:01:01
225.129.1.1  →  01:00:5E:01:01:01
239.1.1.1    →  01:00:5E:01:01:01
239.129.1.1  →  01:00:5E:01:01:01
```

**实际后果**：

主机订阅了 `239.1.1.1`，网卡会按 MAC `01:00:5E:01:01:01` 过滤。但如果网络里还有 `239.129.1.1` 的流量，**它的 MAC 也是这个**，网卡会一起收进来，然后**在 IP 层才发现不对再丢弃**。

浪费的是：
- 交换机端口带宽
- 主机网卡到 CPU 的 PCIe 带宽
- **主机 CPU**（要处理和丢弃这些包）

在高码率视频组播场景（比如多路 4K 流），这个浪费可能很可观。

**规避方法**：

**规划组播地址时，避免使用第二字节相差 128 的组合。**

```
✅ 好的规划：统一用 239.1.x.x 段
   239.1.1.1  （直播频道 1）
   239.1.1.2  （直播频道 2）
   239.1.2.1  （监控视频）
   
❌ 坏的规划：同时用 239.1.x.x 和 239.129.x.x
   239.1.1.1   和  239.129.1.1  → MAC 冲突
```

**企业内部组播地址规划建议**：
- 用 **`239.0.0.0/8`**（管理范围，相当于组播的私网地址）
- **只用第二字节的低 7 位范围**（0-127），避开 128-255
- 按业务分段：`239.1.x.x` 视频、`239.2.x.x` 监控、`239.3.x.x` 数据分发
</details>

**2.** 什么是 RPF 检查？为什么组播必须做这个检查？

<details><summary>答案</summary>

**RPF (Reverse Path Forwarding) 检查**：

```
   路由器收到一个组播包
        ↓
   查【单播路由表】：到【组播源】的最优路径，出接口是哪个？
        ↓
   ★ 这个组播包，是从那个接口收进来的吗？★
        ├─ 是 → RPF 通过 → 复制并转发到下游
        └─ 否 → ★ 直接丢弃 ★
```

**为什么必须做（防环）**：

组播是**树状转发**——每个分叉点都会**复制**数据包。如果存在环路：

```
   单播环路：包绕圈，TTL 递减到 0 就死了，流量是恒定的
   
   组播环路：★ 每经过一个分叉点，包的数量就翻倍 ★
             环路里的流量呈【指数级】增长
             比二层广播风暴更可怕
```

RPF 保证组播流量**只沿着"回到源的最短路径"的反方向**传播，这天然形成一棵**无环的树**。

**RPF 的巧妙之处**：它不需要额外的协议或状态，**直接复用已有的单播路由表**。这就是 PIM 名字里 "Protocol Independent"（协议无关）的含义——它不关心单播路由是 OSPF、EIGRP 还是 BGP 算出来的，直接拿来用。

**RPF 失败是组播排障的头号问题**：

```cisco
R1# debug ip mpacket
IP(0): s=192.168.1.100 (GigabitEthernet0/1) d=239.1.1.1 
       id=1234, ttl=61, prot=17, len=1050, not RPF interface
                                              ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
                                          包从错误的接口进来，被丢弃

R1# show ip rpf 192.168.1.100
RPF information for ? (192.168.1.100)
  RPF interface: GigabitEthernet0/2         ← 期望从这个接口收
  RPF neighbor: ? (10.0.13.3)
  RPF route/mask: 192.168.1.0/24
  RPF type: unicast (ospf 1)                ← 依据是 OSPF 路由
```

**常见的 RPF 失败原因**：

| 原因 | 说明 |
|:--|:--|
| **非对称路由** | 单播路由走 A 路径，但组播实际从 B 路径来 |
| **GRE 隧道场景** | 单播路由指向隧道，但组播走物理接口（或反之） |
| **多路径 ECMP** | 单播有多条等价路径，RPF 只认其中一条 |
| **重分发导致路由变化** | 单播路由指向了非预期的接口 |
| **单播路由缺失** | 到源的路由根本不存在 → RPF 无法判断 → 全丢 |

**解法**：

```cisco
! 方案 1：配静态组播路由，专门覆盖 RPF 检查
R1(config)# ip mroute 192.168.1.0 255.255.255.0 10.0.13.3
!                    ↑ 源网段            ↑ 期望的 RPF 邻居

! 方案 2：修复单播路由，让它和组播路径一致（治本）

! 方案 3：用 MBGP（多协议 BGP）承载独立的组播路由表
!         这样单播和组播可以有不同的拓扑
```

**验证修复**：
```cisco
R1# show ip rpf 192.168.1.100
  RPF type: static mroute                   ← 现在用的是静态组播路由
```
</details>

**3.** PIM-SM 和 PIM-DM 有什么区别？为什么 PIM-SM 是主流？

<details><summary>答案</summary>

| | **PIM-DM（密集模式）** | **PIM-SM（稀疏模式）** |
|:--|:--|:--|
| 机制 | **推 (Push)** | **拉 (Pull)** |
| 初始行为 | **先泛洪到所有地方** | **默认不发任何流量** |
| 后续 | 不需要的路由器发 Prune（剪枝） | 需要的路由器发 Join |
| 需要 RP | ❌ 不需要 | ✅ **需要** |
| 周期性行为 | **每 3 分钟重新泛洪一次**（Prune 超时） | 无 |
| 适用 | 接收者密集、网络小 | ★ **接收者稀疏、网络大** |
| 实际使用 | 几乎不用 | **主流** |

**PIM-DM 的致命问题：周期性泛洪**

```
   T=0：   源开始发送 → 泛洪到全网所有路由器
   T=0+：  不需要的分支发 Prune → 停止转发
   T=3min：★ Prune 超时，重新泛洪一遍 ★
   T=3min+：又收到 Prune → 又停止
   T=6min：★ 再泛洪 ★
   ...无限循环
```

**后果**：即使只有 1 个接收者，**全网每 3 分钟都要被一路组播流冲刷一次**。如果有 10 路 5Mbps 的视频流，每 3 分钟就有 50Mbps 涌向每一条链路。

**在大网络里这是灾难性的。**

**PIM-SM 的优势**：

```
   默认状态：什么都不发（省带宽）
        ↓
   有接收者发 IGMP Report
        ↓
   接收者的路由器向 RP 发 Join
        ↓
   ★ 只在需要的路径上建立转发状态 ★
        ↓
   流量开始后，切换到最短路径树 SPT
```

**只有真正需要的链路上才有组播流量。**

**PIM-SM 的代价：需要 RP**

RP（汇聚点）是一个"中介"——因为接收者不知道源在哪，必须通过 RP 建立初始连接。

这带来了额外的复杂度：
- **所有路由器必须知道相同的 RP 地址**（不一致 → 组播完全不通）
- RP 是单点，需要冗余（Anycast RP、BSR 冗余）
- RP 的位置影响初始路径（可能绕远）

**SPT 切换缓解了 RP 绕路的问题**：流量开始后，接收者的路由器会主动切换到"直接到源"的最短路径，不再经过 RP。

**其他模式**：

| 模式 | 说明 |
|:--|:--|
| **PIM-SSM** | 接收者用 IGMPv3 直接指定 `(S, G)`，**完全不需要 RP**。适合 IPTV 这种"源固定且已知"的场景。**地址范围 232.0.0.0/8** |
| **PIM-BIDIR** | 双向共享树，适合"多源多接收者"（视频会议）。不做 SPT 切换，状态最少 |

**选型建议**：

| 场景 | 推荐 |
|:--|:--|
| 企业内部通用组播 | **PIM-SM** |
| IPTV、单一已知源 | **PIM-SSM**（最简单，无 RP） |
| 多方视频会议 | PIM-BIDIR |
| 实验室/极小网络 | PIM-DM（省事，但不推荐） |
</details>

**4.** 纯二层网络里部署组播，为什么必须配 IGMP Snooping 查询器？

<details><summary>答案</summary>

**因为 IGMP Snooping 是"偷听" IGMP 报文来学习的，而 IGMP Report 是【响应 Query 才发】的。没有查询器 = 没有 Query = 没有 Report = 交换机学不到（或学到后老化）。**

**IGMP 的工作模型**：
```
   路由器（查询器）                          主机
        │                                    │
        │  ① Membership Query（每 60 秒）     │
        │──────────────────────────────────>│
        │                                    │
        │  ② Membership Report               │
        │<───────────────────────────────────│
        │                                    │
   交换机在中间"偷听"这些报文，
   学习"哪个端口下有订阅者"
```

**如果网络里只有交换机，没有三层路由器**（比如一个纯二层的视频监控网段）：

```
   没有查询器
        ↓
   没有周期性 Query
        ↓
   主机只在【刚加入组时】主动发一次 Report（Unsolicited Report）
        ↓
   交换机学到了表项
        ↓
   ★ 表项默认 260 秒后老化 ★
        ↓
   老化后，交换机不知道谁要这个组播
        ↓
   要么停止转发（视频卡死），要么退化成泛洪（浪费带宽）
```

**症状特征（很典型）**：
- 视频**刚开始能看**，**几分钟后卡住**
- 重新打开播放器又能看几分钟（因为重新发了 Report）
- 时间间隔很规律（约 260 秒 = 交换机的 Snooping 老化时间）

**解决方案**：
```cisco
! 在核心/汇聚交换机上配置查询器
SW1(config)# ip igmp snooping querier
SW1(config)# ip igmp snooping vlan 10 querier address 192.168.10.1
SW1(config)# ip igmp snooping vlan 10 querier version 2

! 查看
SW1# show ip igmp snooping querier
Vlan      IP Address     IGMP Version   Port         Max Response Time
10        192.168.10.1   v2             Switch       10

SW1# show ip igmp snooping groups
Vlan   Group        Type    Version  Port List
10     239.1.1.1    igmp    v2       Gi1/0/5, Gi1/0/8
                                     ↑ 只往这两个端口转发 ✓
```

**查询器的选举**：
- 如果网段里有多个查询器（多台交换机都配了，或有路由器），**IP 地址最小的当选**
- 所以配置时要注意**指定一个合理的地址**，避免选出意外的查询器

**⚠️ 另一个相关的坑：IGMP Snooping 全关的后果**

有人为了"图省事"关掉 IGMP Snooping：
```cisco
SW1(config)# no ip igmp snooping
```
结果是**所有组播流量泛洪到所有端口**。一路 10Mbps 视频，48 口交换机 → 每个端口都收到 10Mbps 无用流量。如果有 10 路视频，就是 100Mbps 的垃圾流量灌满每个端口。

**这在监控网络里特别常见**——几十路摄像头组播，关掉 Snooping 后整个网络瘫痪。

**部署检查清单**：
- [ ] 全局和相关 VLAN 启用 IGMP Snooping
- [ ] **有查询器**（三层设备或配置 `snooping querier`）
- [ ] 查询器地址合理，不会被意外抢占
- [ ] 验证 `show ip igmp snooping groups` 有正确的端口列表
- [ ] 长时间观察（> 10 分钟），确认表项不会消失
</details>

**5.** `show ip mroute` 输出里的 `(*, G)` 和 `(S, G)` 分别是什么意思？`T` 标志代表什么？

<details><summary>答案</summary>

| 记法 | 名称 | 含义 |
|:--|:--|:--|
| **`(*, G)`** | **共享树 RPT** | `*` = 任意源。以 **RP 为根**的树 |
| **`(S, G)`** | **最短路径树 SPT** | S = 具体的源 IP。以**源为根**的树，路径最短 |

**为什么需要两种树**：

```
   问题：接收者想收 239.1.1.1，但【不知道源在哪】
        ↓
   ① 先通过 RP 建立 (*, G) 共享树
      路径：源 → RP → 接收者（可能绕远）
        ↓
   ② 收到第一个包后，接收者的路由器【知道源地址了】
        ↓
   ③ ★ 切换到 (S, G) 最短路径树 ★
      路径：源 → 接收者（最短）
        ↓
   ④ 剪掉共享树上多余的分支
```

**`T` 标志 = SPT-bit set = 已经切换到最短路径树。**

**实际输出解读**：
```cisco
R3# show ip mroute 239.1.1.1

(*, 239.1.1.1), 00:05:23/stopped, RP 10.0.0.2, flags: SJC
 ↑                                  ↑              ↑↑↑
 共享树                          RP 地址        S=Sparse
                                                J=Join
                                                C=Connected(有直连接收者)
  Incoming interface: GigabitEthernet0/1, RPF nbr 10.0.23.2
                                          ↑ 朝 RP 的方向
  Outgoing interface list:
    GigabitEthernet0/0, Forward/Sparse, 00:05:23/00:02:45

(192.168.1.100, 239.1.1.1), 00:00:45/00:02:50, flags: JT
 ↑ 具体的源                                             ↑↑
                                                    J=Join
                                                ★ T=已切换到 SPT ★
  Incoming interface: GigabitEthernet0/1, RPF nbr 10.0.23.2
  Outgoing interface list:
    GigabitEthernet0/0, Forward/Sparse, 00:00:45/00:02:50
```

**完整的标志位表**：

| 标志 | 含义 |
|:--|:--|
| `S` | Sparse Mode |
| `D` | Dense Mode |
| **`T`** | **SPT-bit set（已切换到最短路径树）** |
| `J` | Join SPT（正在切换或可以切换） |
| `C` | Connected（本路由器有直连的接收者） |
| `L` | Local（路由器自己是接收者） |
| `P` | Pruned（已剪枝，没有下游接收者） |
| `F` | Register flag（本路由器是源的 DR，正在向 RP 注册） |
| `R` | RP-bit set（用于剪掉共享树分支） |
| `X` | Proxy Join Timer |

**控制 SPT 切换**：
```cisco
! 立即切换（默认行为，收到第一个包就切）
R1(config)# ip pim spt-threshold 0

! 永不切换（一直用共享树）
R1(config)# ip pim spt-threshold infinity

! 按流量阈值切换（超过 X kbps 才切）
R1(config)# ip pim spt-threshold 100
```

**什么时候不想切换到 SPT**：
- **状态爆炸**：每个 `(S, G)` 都要维护独立的转发状态。如果有 1000 个源、1000 个组，就是 100 万条状态。在超大规模组播网络（运营商 IPTV）里，会用 `spt-threshold infinity` 保持在共享树上，减少状态。
- **RP 位置本来就很优**：如果 RP 就在源和接收者之间的最优路径上，切换没有收益。

**排障中的应用**：

| 观察 | 含义 |
|:--|:--|
| 只有 `(*, G)`，没有 `(S, G)` | **源还没开始发送**，或注册失败 |
| 有 `(S, G)` 但没有 `T` | 还在共享树上（可能绕路） |
| `Incoming interface: Null` | **RPF 失败**或本机就是 RP |
| `Outgoing interface list: Null` | **没有下游接收者**（或被剪枝） |
| `flags` 里有 `P` | 已剪枝，不转发 |

**"组播不通"的快速判断**：
```cisco
R# show ip mroute <组地址>
! ① Incoming interface 是 Null？→ RPF 问题
! ② Outgoing interface list 是空？→ 下游没人请求（IGMP 问题）
! ③ 两个都正常但收不到？→ 二层问题（IGMP Snooping）
```
</details>

---

**上一章** ← [08 BGP](08-BGP.md) ｜ **下一章** → [10 QoS 服务质量](10-QoS服务质量.md)
