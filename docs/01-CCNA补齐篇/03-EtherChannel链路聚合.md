# 03 · EtherChannel 链路聚合

## ① 这章解决什么问题

两台核心交换机之间跑千兆链路，业务增长后带宽不够了。

**方案一：换万兆。** 要换模块、换光纤，贵，而且可能设备不支持。

**方案二：再接一根千兆线。** 但是——上一章刚学过，两条并行链路会成环，**STP 会把第二条阻塞掉**，带宽还是 1Gbps，白接。

**EtherChannel 就是方案三**：把多条物理链路捆绑成**一条逻辑链路**。STP 把它看成一个端口（不会阻塞），带宽叠加，还自带冗余——断一根，剩下的继续工作，**不触发 STP 重收敛**。

这是企业网络里最常用、性价比最高的一个技术。理解它，也为后面的 vPC、StackWise、MLAG 打下基础。

---

## ② 原理讲透

### 2.1 核心思想

```
不用 EtherChannel：                用了 EtherChannel：

  SW1 ═══Gi0/1═══ SW2               SW1 ═══Gi0/1═══ SW2
      ───Gi0/2─── (被STP阻塞)           ═══Gi0/2═══
                                          ↓
  可用带宽：1 Gbps                  逻辑上是一个 Port-channel1
  冗余切换：STP 收敛 (秒级)          可用带宽：2 Gbps
                                    冗余切换：< 1 秒，STP 无感知
```

**关键点**：STP 看到的是 `Port-channel1` 这一个逻辑端口，而不是两个物理口，所以**没有环路，不需要阻塞**。

**能捆多少条**：Cisco 一般支持 **最多 8 条活动链路**（部分平台支持 8 活动 + 8 备用，共 16 条配置）。

### 2.2 三种协商协议

| 协议 | 标准 | 模式 | 说明 |
|:--|:--|:--|:--|
| **LACP** | **802.3ad（标准）** | `active` / `passive` | **推荐**，跨厂商通用 |
| **PAgP** | Cisco 私有 | `desirable` / `auto` | 只在 Cisco 设备间可用 |
| **Static (ON)** | 无协商 | `on` | 强制捆绑，不发协商帧 |

**模式组合表（必背考点）**：

#### LACP

| SW1 \ SW2 | active | passive | on |
|:--|:--|:--|:--|
| **active** | ✅ **成功** | ✅ **成功** | ❌ 失败 |
| **passive** | ✅ **成功** | ❌ **失败** | ❌ 失败 |
| **on** | ❌ 失败 | ❌ 失败 | ✅（但不推荐） |

#### PAgP

| SW1 \ SW2 | desirable | auto | on |
|:--|:--|:--|:--|
| **desirable** | ✅ **成功** | ✅ **成功** | ❌ 失败 |
| **auto** | ✅ **成功** | ❌ **失败** | ❌ 失败 |
| **on** | ❌ 失败 | ❌ 失败 | ✅ |

**记忆规则（一句话记住两张表）**：
> **至少要有一方主动**（`active` / `desirable`），双方都被动（`passive+passive` / `auto+auto`）会互相等待，永远起不来。

**`on` 模式的危险**：
- `on` 不发送任何协商帧，**盲目捆绑**
- 如果一端 `on`、另一端不是 `on`（或者根本没配 EtherChannel），**会形成二层环路**——因为一端认为是聚合口（不阻塞），另一端认为是两个独立口
- **生产环境永远用 LACP `active`**，除非对端设备不支持协商

> **LACP vs PAgP 怎么选**：如果对端是 Cisco 也用 LACP。因为 LACP 是标准协议，未来换厂商、接服务器、接防火墙都能通用。PAgP 除了历史遗留没有任何优势。

### 2.3 捆绑成功的前提条件（必须完全一致）

所有成员端口的以下参数**必须完全相同**，否则端口会被踢出聚合组（进入 `suspended` 或 `independent` 状态）：

| 参数 | 说明 |
|:--|:--|
| **速率 (Speed)** | 不能千兆和百兆混捆 |
| **双工 (Duplex)** | 都必须 full |
| **交换模式** | 都是 access 或都是 trunk，不能混 |
| **Access VLAN** | 如果是 access 口，VLAN 必须相同 |
| **Trunk 的 Native VLAN** | 必须相同 |
| **Trunk 的 Allowed VLAN 列表** | 必须完全相同 |
| **STP 参数** | cost、priority 等应一致 |
| **MTU** | 必须相同 |

> **实战最重要的一条**：**先配好 Port-channel 逻辑接口，再把物理口加进去**。反过来（先加物理口再配逻辑口）容易出现参数不同步。
>
> 更稳妥的做法：**在 Port-channel 接口上配置的参数会自动下发到所有成员口**。所以捆好之后，所有 VLAN/Trunk 相关配置都应该在 `interface Port-channel1` 下改，而不是逐个改物理口。

### 2.4 负载分担（Load Balancing）

**最容易被误解的一点：EtherChannel 不是"把每个包轮流分到各条链路上"。**

它是**基于哈希算法，按"流"分配**的：对每个数据帧的某些字段做哈希，结果决定走哪条物理链路。**同一条流（相同的源/目的）永远走同一条链路。**

**为什么这么设计**：如果按包轮询，同一个 TCP 连接的包会走不同链路，因为链路延迟不同会**产生乱序**，接收端要重排序，性能反而更差。按流哈希保证了同一条流的包顺序不变。

**Cisco 支持的哈希依据**：

```cisco
SW1(config)# port-channel load-balance ?
  dst-ip           Dst IP Addr
  dst-mac          Dst Mac Addr
  dst-mixed-ip-port Dst IP Addr and TCP/UDP Port
  dst-port         Dst TCP/UDP Port
  src-dst-ip       Src XOR Dst IP Addr
  src-dst-mac      Src XOR Dst Mac Addr
  src-dst-mixed-ip-port  Src XOR Dst IP Addr and TCP/UDP Port
  src-ip           Src IP Addr
  src-mac          Src Mac Addr
  src-port         Src TCP/UDP Port
```

**默认值**：多数平台是 `src-mac` 或 `src-dst-ip`（因平台而异，用 `show etherchannel load-balance` 确认）。

**场景化选择**：

| 场景 | 推荐算法 | 原因 |
|:--|:--|:--|
| 交换机之间（多对多流量） | `src-dst-ip` | IP 组合多样，哈希分布均匀 |
| 服务器 → 交换机（一台服务器） | `src-dst-port` | 源目 IP 固定，只有端口能区分 |
| 有大量单向流量（备份、存储） | `src-dst-mixed-ip-port` | 最大化熵值 |
| 接路由器（所有流量目的 MAC 相同） | **不要用 `dst-mac`** | 所有流量哈希到同一条链路 |

> **经典故障**：捆了 4 条千兆，但监控显示只有一条在跑满，其他三条闲置。
>
> **根因**：负载分担算法选得不对。比如用了 `dst-mac`，而所有流量都是发往同一个网关（同一个目的 MAC），哈希结果永远相同 → 全挤在一条链路上。
>
> **修复**：改成 `src-dst-ip` 或 `src-dst-mixed-ip-port`，增加哈希输入的多样性。

**另一个常被忽略的点**：**捆绑的链路数最好是 2 的幂（2、4、8）**。因为哈希结果通常是取模分配，链路数是 2 的幂时分布最均匀。捆 3 条或 5 条会导致某些链路承载更多流量。

### 2.5 Layer 2 vs Layer 3 EtherChannel

**二层聚合**（最常见，交换机互联）：
```cisco
SW1(config)# interface range Gi1/0/23-24
SW1(config-if-range)# channel-group 1 mode active
SW1(config)# interface Port-channel1
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk allowed vlan 10,20,30
```

**三层聚合**（路由口聚合，核心之间的三层互联）：
```cisco
SW1(config)# interface range Gi1/0/23-24
SW1(config-if-range)# no switchport            ! ★ 必须先转成路由口
SW1(config-if-range)# channel-group 1 mode active

SW1(config)# interface Port-channel1
SW1(config-if)# no switchport
SW1(config-if)# ip address 10.0.0.1 255.255.255.252
```

> **顺序很重要**：三层聚合必须**先在物理口上 `no switchport`**，再加入 channel-group。如果先加入了再改，可能报错或行为异常。

---

## ③ 配置命令

### Cisco

```cisco
! ═══ 标准 LACP 二层聚合（推荐做法）═══
! Step 1: 先清理成员口的旧配置（可选但推荐）
SW1(config)# default interface range GigabitEthernet1/0/23-24

! Step 2: 把物理口加入聚合组
SW1(config)# interface range GigabitEthernet1/0/23-24
SW1(config-if-range)# switchport
SW1(config-if-range)# switchport mode trunk
SW1(config-if-range)# switchport trunk allowed vlan 10,20,30
SW1(config-if-range)# switchport trunk native vlan 999
SW1(config-if-range)# channel-protocol lacp
SW1(config-if-range)# channel-group 1 mode active      ! ★ 自动创建 Po1
SW1(config-if-range)# no shutdown

! Step 3: 在逻辑口上做后续配置（会自动同步到成员口）
SW1(config)# interface Port-channel1
SW1(config-if)# description ### To-SW2-Po1 ###
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk allowed vlan 10,20,30

! ═══ LACP 高级参数 ═══
! LACP 系统优先级（决定哪端是决策方，默认 32768）
SW1(config)# lacp system-priority 100

! 端口优先级（决定哪些口活动、哪些备用，默认 32768，数字小的优先）
SW1(config-if)# lacp port-priority 100

! 快速 LACP（1秒 hello，默认是 30秒）
SW1(config-if)# lacp rate fast

! 最少活动链路数（少于这个数就整个 Po 下线，防止带宽不足还硬撑）
SW1(config-if)# port-channel min-links 2

! ═══ 负载分担 ═══
SW1(config)# port-channel load-balance src-dst-ip
SW1# show etherchannel load-balance

! ═══ 三层聚合 ═══
SW1(config)# interface range GigabitEthernet1/0/23-24
SW1(config-if-range)# no switchport
SW1(config-if-range)# channel-group 2 mode active
SW1(config)# interface Port-channel2
SW1(config-if)# no switchport
SW1(config-if)# ip address 10.0.0.1 255.255.255.252

! ═══ 查看 ═══
SW1# show etherchannel summary                 ! ★ 最常用
SW1# show etherchannel 1 detail
SW1# show etherchannel 1 port-channel
SW1# show lacp neighbor
SW1# show lacp 1 internal
SW1# show lacp counters
SW1# show interfaces Port-channel1
SW1# show interfaces Port-channel1 etherchannel
SW1# show spanning-tree vlan 10                ! 确认 Po 作为一个端口参与 STP
```

### `show etherchannel summary` 输出解读（最重要的排障命令）

```cisco
SW1# show etherchannel summary

Flags:  D - down        P - bundled in port-channel
        I - stand-alone s - suspended
        H - Hot-standby (LACP only)
        R - Layer3      S - Layer2
        U - in use      f - failed to allocate aggregator
        M - not in use, minimum links not met
        u - unsuitable for bundling
        w - waiting to be aggregated
        d - default port

Number of channel-groups in use: 1
Number of aggregators:           1

Group  Port-channel  Protocol    Ports
------+-------------+-----------+-----------------------------------------
1      Po1(SU)        LACP       Gi1/0/23(P)  Gi1/0/24(P)
          ↑↑                            ↑           ↑
       S=二层 U=在用                  P=已捆绑    P=已捆绑
```

**标志位速查（排障核心）**：

| 标志 | 含义 | 说明 |
|:--|:--|:--|
| **`(P)`** | bundled | ✅ **正常，已成功捆绑** |
| **`(I)`** | stand-alone | ❌ 独立工作，**没捆上**——协商失败或参数不一致 |
| **`(s)`** | suspended | ❌ 被挂起——LACP 协商失败，对端没响应 |
| **`(D)`** | down | ❌ 端口物理 down |
| `(H)` | Hot-standby | 热备（超过 8 条活动链路时的备用口） |
| `(w)` | waiting | 正在等待协商 |
| **`(SU)`** | Layer2 + in use | ✅ 二层聚合正常工作 |
| **`(RU)`** | Layer3 + in use | ✅ 三层聚合正常工作 |
| **`(SD)`** | Layer2 + down | ❌ 聚合口整体 down |
| `(SM)` | not in use, min-links | 活动链路数少于 `min-links` 设定值 |

> **排障口诀**：**看到 `(P)` 就对了，看到 `(I)` 或 `(s)` 就有问题。**
> - `(I)` → 通常是两端参数不一致（VLAN、模式、速率）
> - `(s)` → 通常是 LACP 协商失败（对端是 passive+passive，或没配）

### 三厂商对照

| 目的 | Cisco | H3C | 华为 |
|:--|:--|:--|:--|
| 创建聚合口 | `interface Port-channel1` | `interface Bridge-Aggregation 1` | `interface Eth-Trunk 1` |
| 三层聚合口 | `interface Port-channel1` + `no switchport` | `interface Route-Aggregation 1` | `interface Eth-Trunk 1` + `undo portswitch` |
| 加入成员 | `channel-group 1 mode active` | `port link-aggregation group 1` | `eth-trunk 1` |
| 启用 LACP | `mode active` | `link-aggregation mode dynamic` | `mode lacp-static` |
| 静态捆绑 | `mode on` | 默认（静态） | `mode manual load-balance` |
| 负载分担 | `port-channel load-balance src-dst-ip` | `link-aggregation load-sharing mode source-ip destination-ip` | `load-balance src-dst-ip` |
| 查看 | `show etherchannel summary` | `display link-aggregation verbose` | `display eth-trunk` |

> **术语对照**：
> - Cisco：**EtherChannel / Port-channel**
> - H3C：**链路聚合 / Bridge-Aggregation（二层）、Route-Aggregation（三层）**
> - 华为：**Eth-Trunk**
> - 业界通用：**LAG (Link Aggregation Group)** 或 **Bonding**（Linux 侧）
>
> **重要差异**：**H3C 的聚合默认是静态模式（相当于 Cisco 的 `on`）**，需要显式配 `link-aggregation mode dynamic` 才启用 LACP。跟 Cisco 对接时如果忘了这一步，一端 LACP `active` 一端静态 → 捆不起来（Cisco 侧显示 `(s)` suspended）。

---

## ④ 配套实验：LACP 聚合 + 故障切换

**拓扑**：
```
              Po1 (LACP)
   ┌──────┐ Gi1/0/23 ═══ Gi1/0/23 ┌──────┐
   │ SW1  │ Gi1/0/24 ═══ Gi1/0/24 │ SW2  │
   └──┬───┘                       └───┬──┘
      │Gi1/0/1                        │Gi1/0/1
     PC1 (VLAN 10)                   PC2 (VLAN 10)
   192.168.10.10                   192.168.10.20
```

### Step 1：不配聚合，先看 STP 阻塞（对照组）

```cisco
! 两台都配好 VLAN 10 和两条 Trunk（暂不聚合）
SW1(config)# interface range Gi1/0/23-24
SW1(config-if-range)# switchport mode trunk
SW1(config-if-range)# switchport trunk allowed vlan 10
```

```cisco
SW1# show spanning-tree vlan 10
Interface   Role Sts Cost   Prio.Nbr Type
----------- ---- --- ------ -------- ------
Gi1/0/23    Desg FWD 4      128.23   P2p
Gi1/0/24    Desg FWD 4      128.24   P2p

SW2# show spanning-tree vlan 10
Gi1/0/23    Root FWD 4      128.23   P2p
Gi1/0/24    Altn BLK 4      128.24   P2p      ← 一条被阻塞了，带宽浪费
```

**结论**：接了两条线，实际只能用一条。

### Step 2：配置 LACP 聚合

```cisco
! ── SW1 ──
SW1(config)# interface range GigabitEthernet1/0/23-24
SW1(config-if-range)# shutdown                       ! 先关，避免配置过程中成环
SW1(config-if-range)# switchport mode trunk
SW1(config-if-range)# switchport trunk allowed vlan 10
SW1(config-if-range)# switchport trunk native vlan 999
SW1(config-if-range)# channel-protocol lacp
SW1(config-if-range)# channel-group 1 mode active
SW1(config-if-range)# no shutdown

! ── SW2 ── 参数必须完全一致
SW2(config)# interface range GigabitEthernet1/0/23-24
SW2(config-if-range)# shutdown
SW2(config-if-range)# switchport mode trunk
SW2(config-if-range)# switchport trunk allowed vlan 10
SW2(config-if-range)# switchport trunk native vlan 999
SW2(config-if-range)# channel-protocol lacp
SW2(config-if-range)# channel-group 1 mode active
SW2(config-if-range)# no shutdown
```

### Step 3：验证捆绑成功

```cisco
SW1# show etherchannel summary
Group  Port-channel  Protocol    Ports
------+-------------+-----------+------------------------------
1      Po1(SU)        LACP       Gi1/0/23(P)  Gi1/0/24(P)
                                          ↑ 两个都是 (P) = 成功 ✓

SW1# show lacp neighbor
Flags:  S - Device is requesting Slow LACPDUs
        F - Device is requesting Fast LACPDUs
        A - Device is in Active mode    P - Device is in Passive mode

Channel group 1 neighbors
                  LACP port                        Admin  Oper   Port
Port      Flags   Priority  Dev ID          Age    Key    Key    Number
Gi1/0/23  SA      32768     0000.0000.2222  12s    0x0    0x1    0x17
Gi1/0/24  SA      32768     0000.0000.2222  15s    0x0    0x1    0x18
                  ↑ SA = Slow + Active，对端确实在 active 模式 ✓
```

### Step 4：验证 STP 把它当一个端口

```cisco
SW1# show spanning-tree vlan 10
Interface   Role Sts Cost   Prio.Nbr Type
----------- ---- --- ------ -------- ------
Po1         Desg FWD 3      128.65   P2p       ← 只有 Po1，没有物理口
                       ↑
              注意 cost 变成 3（聚合后带宽翻倍，开销降低）
```

**关键观察**：
1. 物理口 Gi1/0/23、Gi1/0/24 **不再单独出现在 STP 里**
2. **没有任何端口被阻塞** → 两条链路都在转发
3. Po1 的 cost 是 **3**（2Gbps），比单条 1Gbps 的 4 小

### Step 5：验证带宽叠加

```cisco
SW1# show interfaces Port-channel1
Port-channel1 is up, line protocol is up
  Hardware is EtherChannel, address is 0000.0000.1117
  MTU 1500 bytes, BW 2000000 Kbit/sec, DLY 10 usec,
                     ↑ 2 Gbps ✓
  ...
  Members in this channel: Gi1/0/23 Gi1/0/24
```

### Step 6：故障切换测试（重点）

```cisco
! 在 PC1 上持续 ping PC2：ping 192.168.10.20 -t

! 断掉其中一条成员链路
SW1(config)# interface GigabitEthernet1/0/23
SW1(config-if)# shutdown
```

**预期结果**：
```cisco
SW1# show etherchannel summary
1      Po1(SU)        LACP       Gi1/0/23(D)  Gi1/0/24(P)
                                          ↑D=down    ↑仍在工作

SW1# show interfaces Port-channel1 | include BW
  MTU 1500 bytes, BW 1000000 Kbit/sec           ← 带宽自动降到 1Gbps
```

**ping 结果：丢 0~1 个包。**

**对比 Step 1 的场景**（不聚合，靠 STP 冗余）：断链路会丢 1-3 个包（Rapid-PVST+）甚至 30-50 个包（传统 STP）。

> **这就是 EtherChannel 相比"靠 STP 做冗余"的核心优势**：切换在聚合层完成，**STP 完全不感知拓扑变化**，不触发重收敛。

**恢复**：
```cisco
SW1(config-if)# no shutdown
```

### Step 7：制造并排查典型故障

**故障 A：一端 active，一端 passive+passive**

```cisco
SW1(config)# interface range Gi1/0/23-24
SW1(config-if-range)# channel-group 1 mode passive
SW2(config)# interface range Gi1/0/23-24
SW2(config-if-range)# channel-group 1 mode passive
```

```cisco
SW1# show etherchannel summary
1      Po1(SD)        LACP       Gi1/0/23(s)  Gi1/0/24(s)
                                          ↑ s = suspended
```
**根因**：双方都被动等待，没人发起协商。
**修复**：至少一端改成 `active`。

**故障 B：两端 Allowed VLAN 不一致**

```cisco
SW2(config)# interface range Gi1/0/23-24
SW2(config-if-range)# switchport trunk allowed vlan 10,20     ! SW1 只有 10
```

```cisco
SW1# show etherchannel summary
1      Po1(SU)        LACP       Gi1/0/23(P)  Gi1/0/24(P)
! 可能仍显示捆绑成功，但 VLAN 20 的流量在 SW1 侧不通
```

**注意**：Allowed VLAN 不一致**不一定阻止捆绑**（跨设备参数不需要严格一致，只有同一台设备的成员口之间必须一致）。真正的问题是 VLAN 20 的流量会被 SW1 丢弃。

**排查**：`show interfaces trunk` 对比两端。

**故障 C：同一台设备的成员口参数不一致**

```cisco
SW1(config)# interface Gi1/0/24
SW1(config-if)# switchport access vlan 20        ! 改成了 access
```

```cisco
SW1# show etherchannel summary
1      Po1(SU)        LACP       Gi1/0/23(P)  Gi1/0/24(I)
                                                        ↑ I = 独立，被踢出去了

SW1# show etherchannel 1 detail | include Reason
! 或看日志
%EC-5-CANNOT_BUNDLE2: Gi1/0/24 is not compatible with Gi1/0/23 and will be suspended (vlan mode of Gi1/0/24 is access, Gi1/0/23 is trunk)
```
**根因**：同一聚合组内成员口的交换模式必须一致。
**修复**：改回 trunk，或在 `interface Port-channel1` 下统一配置。

**故障 D：一端 LACP，一端静态 on**

```cisco
SW2(config)# interface range Gi1/0/23-24
SW2(config-if-range)# channel-group 1 mode on
```

```cisco
SW1# show etherchannel summary
1      Po1(SD)        LACP       Gi1/0/23(s)  Gi1/0/24(s)
```

**⚠️ 危险**：SW2 侧因为是 `on` 模式（不协商，强制捆绑），**它认为链路已经聚合，不会被 STP 阻塞**；而 SW1 侧协商失败，两个口是独立的 Trunk。这种状态下**极易形成二层环路**。

**这就是为什么绝不要用 `on` 模式。**

---

## ⑤ 排障思路

| 症状 | 标志位 | 怀疑点 | 验证 | 根因 |
|:--|:--|:--|:--|:--|
| 端口显示 `(s)` suspended | LACP 协商失败 | `show lacp neighbor` | 对端没配 / 双方都 passive / 对端是 `on` |
| 端口显示 `(I)` stand-alone | 参数不一致 | `show etherchannel 1 detail` | 同组成员口速率/双工/VLAN/模式不同 |
| 端口显示 `(D)` down | 物理层 | `show interfaces status` | 链路断、光模块坏 |
| Po 显示 `(SM)` | min-links 未满足 | `show run int Po1` | 活动链路数 < `port-channel min-links` |
| 捆绑成功但只有一条在跑流量 | 负载分担算法 | `show etherchannel load-balance` | 哈希输入熵值太低，改 `src-dst-ip` |
| 捆绑后出现环路 | 一端 `on` | 对比两端 `show run` | 模式不匹配，一端强制捆绑 |
| 带宽没有叠加 | 同上 | `show int Po1 \| inc BW` | 只有一条成员口是 `(P)` |
| 混合厂商捆不起来 | 协商模式 | 对比配置 | H3C 默认静态，需 `link-aggregation mode dynamic` |

### 标准排查流程

```
1. show etherchannel summary
   └─ 看标志位：(P) 正常，(I)/(s)/(D) 异常
   
2. 如果是 (s) suspended：
   SW1# show lacp neighbor
   └─ 有输出？→ 对端在协商，检查两端模式组合
   └─ 无输出？→ 对端根本没发 LACP，检查对端配置或线序
   
3. 如果是 (I) stand-alone：
   SW1# show etherchannel 1 detail
   SW1# show logging | include EC-5
   └─ 日志会明确告诉你哪个参数不一致
   
4. 如果捆绑成功但流量不均：
   SW1# show etherchannel load-balance
   SW1# show interfaces Gi1/0/23 | include rate
   SW1# show interfaces Gi1/0/24 | include rate
   └─ 对比各成员口流量，改哈希算法
   
5. 如果怀疑成环：
   SW1# show spanning-tree vlan 10
   └─ 应该只看到 Po1，看到物理口说明捆绑没生效
```

### 检查两端配置一致性的快捷方法

```cisco
SW1# show run interface Po1
SW1# show run interface Gi1/0/23
! 与 SW2 的对应输出逐行比对
```

或者用管道快速对比：
```cisco
SW1# show interfaces trunk | include Po1
SW2# show interfaces trunk | include Po1
```

---

## ⑥ 考点提示 + 自测题

### 考点

- **LACP/PAgP 模式组合表**必背（尤其"双被动不成功"）。
- **`show etherchannel summary` 的标志位** `(P)/(I)/(s)/(SU)/(RU)` 是排障核心。
- **负载分担基于流哈希，不是逐包轮询**——这是最常见的理解误区。
- **`on` 模式的危险性**（可能成环）。
- **三层聚合要先 `no switchport`**。
- **EtherChannel 对 STP 是一个逻辑端口**，不会被阻塞。

### 自测题

**1.** SW1 配了 `channel-group 1 mode passive`，SW2 也配了 `passive`。聚合能起来吗？

<details><summary>答案</summary>

**不能。**

LACP 的 `passive` 模式是"**被动响应**"：它不主动发送 LACPDU，只在收到对方的 LACPDU 后才回应。

双方都 passive → **谁都不先开口** → 永远协商不成功。

`show etherchannel summary` 会显示：
```
1      Po1(SD)        LACP       Gi1/0/23(s)  Gi1/0/24(s)
```
`(s)` = suspended（挂起）

**规则记忆**：**至少要有一方主动。**
- LACP：`active + active` ✓、`active + passive` ✓、`passive + passive` ✗
- PAgP：`desirable + desirable` ✓、`desirable + auto` ✓、`auto + auto` ✗

**最佳实践**：**两端都配 `active`**。这样任何一端重启或重配后都能主动重新协商，收敛更快。
</details>

**2.** 捆了 4 条千兆链路，理论带宽 4Gbps。但监控发现只有一条链路跑到 900Mbps，其他三条几乎没流量。为什么？怎么修？

<details><summary>答案</summary>

**根因：负载分担算法的哈希输入熵值太低。**

EtherChannel **不是逐包轮询**，而是对每个帧的特定字段做哈希，用结果决定走哪条链路。**同一条流永远走同一条链路**（这样才能保证不乱序）。

如果哈希输入的多样性不够，所有流量会哈希到同一条链路上。常见情况：

| 场景 | 错误算法 | 为什么失效 |
|:--|:--|:--|
| 所有流量发往同一个路由器/网关 | `dst-mac` | 目的 MAC 全是网关的 MAC，哈希结果恒定 |
| 只有一台服务器在传输 | `src-ip` / `src-mac` | 源地址只有一个 |
| 两台设备之间的单一大流（如存储同步） | 任何算法 | **单条 TCP 流本身就无法拆分** |

**诊断**：
```cisco
SW1# show etherchannel load-balance
EtherChannel Load-Balancing Configuration:
        dst-mac                              ← 找到问题

SW1# show interfaces Gi1/0/21 | include rate
SW1# show interfaces Gi1/0/22 | include rate
SW1# show interfaces Gi1/0/23 | include rate
SW1# show interfaces Gi1/0/24 | include rate
! 对比四个口的实际速率
```

**修复**：
```cisco
SW1(config)# port-channel load-balance src-dst-ip
! 或熵值更高的
SW1(config)# port-channel load-balance src-dst-mixed-ip-port
```

**⚠️ 无法解决的情况**：如果瓶颈是**单条 TCP 流**（比如一次大文件传输），无论什么算法都只能走一条链路——因为拆分会导致乱序。**EtherChannel 提升的是总吞吐，不是单流带宽。** 单流要快只能升级链路速率。

**这是必须让业务方理解的一点**：给他们捆了 4Gbps，不等于一次文件传输能跑到 4Gbps。
</details>

**3.** 为什么绝对不要在生产环境用 `channel-group 1 mode on`？

<details><summary>答案</summary>

因为 `on` 模式**不发送任何协商帧，盲目地强制捆绑**。

**危险场景**：一端配了 `on`，另一端配置错了、还没配、或者线接错了口。

```
SW1 (mode on)                    SW2 (未配 EtherChannel)
Gi0/23 ═══════════════════════ Gi0/23  (独立 Trunk)
Gi0/24 ═══════════════════════ Gi0/24  (独立 Trunk)
   ↓                                ↓
认为是一个 Po1，                 认为是两个独立端口，
STP 只看到一个端口，              STP 会阻塞其中一个
不阻塞任何链路
```

**但 STP 计算在两端不对称** —— SW1 认为只有一个逻辑口不需要阻塞，SW2 虽然会阻塞一个口，但由于 SW1 侧的两个物理口都在转发，**从 SW1 发出的广播会同时从两条链路到达 SW2**，而 SW2 的两个口在 SW1 眼中是同一个 Po 的成员——**MAC 地址表震荡 + 潜在的广播风暴**。

**用 LACP 就不会有这个问题**：协商不成功时，端口会被 `suspended`，**不转发任何流量**，是"故障安全（fail-safe）"的行为。

**唯一可以考虑用 `on` 的场景**：对端设备不支持 LACP（极老的设备或某些特殊硬件）。即便如此，也应该：
1. 双方都严格确认配置无误
2. 配置期间先 `shutdown` 物理口
3. 配好后再 `no shutdown`
4. 配 `storm-control` 作为兜底

**记住：LACP 的协商机制不是麻烦，是保护。**
</details>

**4.** 三层 EtherChannel（路由口聚合）配置时，`no switchport` 应该在哪里执行？顺序有讲究吗？

<details><summary>答案</summary>

**必须先在物理成员口上执行 `no switchport`，然后再 `channel-group`。**

**正确顺序**：
```cisco
SW1(config)# interface range GigabitEthernet1/0/23-24
SW1(config-if-range)# no switchport                    ! ① 先转路由口
SW1(config-if-range)# no ip address                    ! ② 清掉可能的残留 IP
SW1(config-if-range)# channel-group 1 mode active      ! ③ 再加入聚合组

SW1(config)# interface Port-channel1
SW1(config-if)# no switchport                          ! ④ 逻辑口也要
SW1(config-if)# ip address 10.0.0.1 255.255.255.252
```

**为什么顺序重要**：`channel-group` 命令会检查成员口的属性来决定创建二层还是三层 Port-channel。如果物理口还是 switchport 状态就加入聚合组，IOS 会创建**二层** Port-channel，之后再改就可能报错或需要拆了重建。

**另一个细节**：`no switchport` 会**清除该接口所有的二层配置**（VLAN、trunk 设置等），并可能短暂 flap 接口。所以这个操作要在维护窗口做。

**验证**：
```cisco
SW1# show etherchannel summary
1      Po1(RU)        LACP       Gi1/0/23(P)  Gi1/0/24(P)
          ↑↑
        R = Layer3 ✓   U = in use ✓
```
看到 **`(RU)`** 就说明三层聚合配对了。如果是 `(SU)`，说明还是二层。
</details>

**5.** Cisco 交换机和 H3C 交换机之间做链路聚合，Cisco 侧配了 `mode active`，但 `show etherchannel summary` 显示 `(s)` suspended。H3C 侧可能少配了什么？

<details><summary>答案</summary>

**H3C 侧很可能用的是默认的静态聚合模式，没有启用 LACP（动态聚合）。**

**H3C 的默认行为**：`interface Bridge-Aggregation 1` 创建后**默认是静态聚合**（相当于 Cisco 的 `mode on`），**不发送 LACPDU**。

Cisco 侧是 `active`，会主动发 LACPDU，但 H3C 不回应 → Cisco 认为协商失败 → 端口 `suspended`。

**H3C 侧正确配置**：
```
[SW-H3C] interface Bridge-Aggregation 1
[SW-H3C-Bridge-Aggregation1] link-aggregation mode dynamic     ← ★ 关键，启用 LACP
[SW-H3C-Bridge-Aggregation1] port link-type trunk
[SW-H3C-Bridge-Aggregation1] port trunk permit vlan 10 20 30
[SW-H3C-Bridge-Aggregation1] quit

[SW-H3C] interface GigabitEthernet 1/0/23
[SW-H3C-GigabitEthernet1/0/23] port link-aggregation group 1
[SW-H3C] interface GigabitEthernet 1/0/24
[SW-H3C-GigabitEthernet1/0/24] port link-aggregation group 1
```

**验证**：
```
[SW-H3C] display link-aggregation verbose Bridge-Aggregation 1
Aggregation Interface: Bridge-Aggregation1
Aggregation Mode: Dynamic                    ← 应该是 Dynamic
...
  Port             Status  Priority Oper-Key  Flag
  GE1/0/23         S       32768    1         {ACDEF}
  GE1/0/24         S       32768    1         {ACDEF}
                   ↑ S = Selected（已选中，正常）
```

**H3C 标志位对照**：
| H3C | 含义 | 对应 Cisco |
|:--|:--|:--|
| `S` (Selected) | 已选中，参与转发 | `(P)` |
| `U` (Unselected) | 未选中 | `(I)` / `(s)` |

**华为侧的对应配置**：
```
[SW-HW] interface Eth-Trunk 1
[SW-HW-Eth-Trunk1] mode lacp-static            ← 启用 LACP
[SW-HW-Eth-Trunk1] port link-type trunk
[SW-HW-Eth-Trunk1] port trunk allow-pass vlan 10 20 30
[SW-HW] interface GigabitEthernet 0/0/23
[SW-HW-GigabitEthernet0/0/23] eth-trunk 1
```

**混合厂商聚合的检查清单**：
1. 两端都启用了 LACP（不是静态模式）
2. 两端至少一方是 active
3. 速率/双工一致
4. Trunk 的 allowed VLAN 一致
5. LACP 的 rate（fast/slow）建议一致
</details>

---

**上一章** ← [02 STP 生成树协议](02-STP生成树协议.md) ｜ **下一章** → [04 OSPF 单区域](04-OSPF单区域.md)
