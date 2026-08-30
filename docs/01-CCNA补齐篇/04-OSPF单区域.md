# 04 · OSPF 单区域

> ⚠️ **本篇最重要的一章。** ENCOR 和 ENARSI 里 OSPF 的分量最重，而所有进阶内容（多区域、LSA 类型、虚链路、路由重分发）都建立在这一章的邻居建立、LSDB、SPF 计算之上。
> 通过标准：**能背出邻居状态机的 7 个状态，并说清每个状态卡住时该查什么。**

## ① 这章解决什么问题

上一章我们用静态路由让两个网段互通了。但如果网络里有 30 台路由器、200 个网段呢？

- 手工配 200 条静态路由，配到手断，还容易配错
- 加一个新网段，要去所有路由器上加路由
- 链路断了，路由不会自动切换（除非配浮动路由 + IP SLA）

**动态路由协议就是答案**：路由器之间互相交换信息，**自动**计算出最优路径，拓扑变化时**自动**重新收敛。

OSPF 是企业网络里最主流的内部网关协议（IGP）。理解它的方式不是背命令，而是理解**"每台路由器都拥有一张完整的全网地图，然后各自算出到每个目的地的最短路"** 这个核心思想。

---

## ② 原理讲透

### 2.1 OSPF 是什么类型的协议

| 特性 | OSPF |
|:--|:--|
| 类型 | **链路状态型（Link-State）** |
| 算法 | **SPF / Dijkstra 最短路径优先** |
| 传输 | 直接跑在 IP 之上，**协议号 89** |
| 组播地址 | `224.0.0.5`（所有 OSPF 路由器）、`224.0.0.6`（DR/BDR） |
| 管理距离 | **110** |
| 度量值 | **Cost = 参考带宽 / 接口带宽** |
| 支持 VLSM/CIDR | ✅ 无类协议 |
| 收敛速度 | 快（拓扑变化立即触发 LSA 泛洪） |
| 标准 | OSPFv2 (IPv4) = RFC 2328；OSPFv3 (IPv6) = RFC 5340 |

### 2.2 链路状态 vs 距离矢量（理解 OSPF 的钥匙）

| | 距离矢量（RIP、EIGRP 部分） | **链路状态（OSPF、IS-IS）** |
|:--|:--|:--|
| 传什么 | "我到 X 网段要 3 跳" | "我这台路由器有这些接口，连着这些邻居" |
| 知道什么 | 只知道方向和距离，**不知道全貌** | **每台路由器都有全网完整拓扑图** |
| 类比 | 问路：路人说"往前 3 个路口左转" | 看地图：自己拿着完整地图规划路线 |
| 环路风险 | 高（需要水平分割、毒性逆转等机制） | **低**（有全局视野，天然防环） |
| 资源消耗 | 低 | 高（要存 LSDB、跑 SPF 算法） |
| 收敛 | 慢（逐跳传递，有计时器） | 快（LSA 立即泛洪全区域） |

**OSPF 的工作流程（三步）**：

```
① 建立邻居 (Neighbor)
   └─ Hello 报文互相发现，协商参数
        ↓
② 同步数据库 (LSDB)
   └─ 交换 LSA，每台路由器都拥有完全相同的"全网地图"
        ↓
③ 计算路由 (SPF)
   └─ 各自以自己为根，跑 Dijkstra 算法，算出到每个网段的最短路径
        ↓
      装入路由表
```

**关键理解**：**同一个区域内，所有路由器的 LSDB 必须完全一致。** 如果不一致，说明同步出了问题——这是 OSPF 排障的第一原则。

### 2.3 Router ID（路由器的身份证）

每台 OSPF 路由器需要一个 **32 位的唯一标识**，格式像 IP 地址。

**选举顺序**：
```
1. 手工配置的 router-id            ← 强烈推荐，永远手工配
        ↓ 没配
2. 所有 up 的 Loopback 接口中最大的 IP
        ↓ 没有 Loopback
3. 所有 up 的物理接口中最大的 IP
```

**为什么必须手工配**：
- 如果靠接口 IP 自动选，某个接口 down 了可能导致 **Router ID 变化 → 所有邻居关系重建 → 全网重新收敛**
- Router ID 变化后不会自动更新，需要 `clear ip ospf process`（业务中断）

```cisco
R1(config)# router ospf 1
R1(config-router)# router-id 1.1.1.1        ! ★ 第一件事就配这个
```

**最佳实践**：用 Loopback 地址做 Router ID，且 Loopback 地址和 Router ID 保持一致，方便排障时一眼认出是哪台设备。

> **改 Router ID 后必须重置进程才生效**：
> ```cisco
> R1# clear ip ospf process
> ```
> ⚠️ 这会**中断所有 OSPF 邻居关系并重新收敛**，生产环境必须在维护窗口做。

### 2.4 邻居状态机（★ 核心考点）

```
   Down
     │  收到 Hello
     ▼
   Init            ← 我收到了对方的 Hello，但对方的 Hello 里还没有我
     │  在对方 Hello 里看到自己的 Router ID
     ▼
   2-Way           ← 双向通信建立！此时选举 DR/BDR
     │  （在广播网络中，非 DR/BDR 之间就停在这里，这是正常的！）
     │  决定要建立邻接关系
     ▼
   ExStart         ← 协商主从关系和 DD 序列号
     │
     ▼
   Exchange        ← 交换 DD 报文（数据库摘要）
     │
     ▼
   Loading         ← 用 LSR 请求缺失的 LSA，对方用 LSU 回应
     │
     ▼
   Full            ← ✅ 数据库完全同步，邻接关系建立完成
```

**七个状态的卡点诊断（必背表）**：

| 卡在哪 | 含义 | 最可能的原因 | 检查什么 |
|:--|:--|:--|:--|
| **Down** | 完全没收到 Hello | 接口 down / 没 `network` 进 OSPF / 接口是 passive | `show ip ospf interface`、`show ip int brief` |
| **Init** | 单向通信 | **对方收不到我的 Hello** —— ACL 拦了组播、单向链路 | ACL、`show access-lists`、抓包 |
| **2-Way** | 双向通信但不建邻接 | **广播网络中双方都不是 DR/BDR —— 这是正常状态！** | `show ip ospf neighbor` 看是否 DROTHER |
| **ExStart** | 主从协商卡住 | **MTU 不匹配**（最经典）/ 单向链路 | `show ip ospf interface` 对比 MTU |
| **Exchange** | DD 交换卡住 | MTU 不匹配 / MTU 太大被丢弃 | 同上 |
| **Loading** | LSA 请求卡住 | LSA 损坏 / 内存不足（罕见） | `show ip ospf database` |
| **Full** | ✅ 正常 | — | — |

> **两个最重要的诊断**：
> 1. **卡在 `ExStart`/`Exchange` → 99% 是 MTU 不匹配。** OSPF 的 DD 报文会用接口 MTU 大小，两端 MTU 不同时无法完成交换。
> 2. **停在 `2-Way` 不一定是故障。** 在广播网络（以太网）中，DROTHER 之间只保持 2-Way 是**设计如此**——它们只跟 DR/BDR 建立 Full 邻接。

**MTU 不匹配的解决**：
```cisco
! 方法 1（推荐）：统一两端 MTU
R1(config-if)# ip mtu 1500

! 方法 2：忽略 MTU 检查（治标不治本，但应急有用）
R1(config-if)# ip ospf mtu-ignore
```

### 2.5 建立邻居的五个必要条件（★ 高频考点）

Hello 报文里携带这些参数，**任何一项不匹配都无法建立邻居**：

| # | 条件 | 说明 | 检查命令 |
|:--|:--|:--|:--|
| 1 | **Area ID 相同** | 接口必须在同一个区域 | `show ip ospf interface` |
| 2 | **Hello / Dead 间隔相同** | 默认广播网 10/40 秒，NBMA 30/120 秒 | `show ip ospf interface` |
| 3 | **认证类型和密钥相同** | 明文/MD5，密钥必须一致 | `show ip ospf interface` |
| 4 | **Stub 区域标志相同** | Stub/NSSA 标志位必须一致 | `show ip ospf` |
| 5 | **子网掩码相同**（广播/NBMA 网络） | 两端必须在同一网段 | `show ip interface brief` |

**外加两个"不在 Hello 里但同样致命"的条件**：

| # | 条件 | 说明 |
|:--|:--|:--|
| 6 | **Router ID 不能重复** | 重复会导致邻居翻转、LSDB 混乱 |
| 7 | **MTU 必须匹配** | 不影响 Hello，但会卡在 ExStart/Exchange |

**一条命令看全**：
```cisco
R1# show ip ospf interface GigabitEthernet0/0
GigabitEthernet0/0 is up, line protocol is up
  Internet Address 192.168.12.1/24, Area 0        ← 条件 1、5
  Process ID 1, Router ID 1.1.1.1, Network Type BROADCAST, Cost: 1
  Transmit Delay is 1 sec, State DR, Priority 1
  Designated Router (ID) 1.1.1.1, Interface address 192.168.12.1
  Backup Designated router (ID) 2.2.2.2, Interface address 192.168.12.2
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
                                     ↑ 条件 2
  ...
  Message digest authentication enabled              ← 条件 3
    Youngest key id is 1
```

### 2.6 DR / BDR 选举（广播网络特有）

**为什么需要 DR**：

在一个多路访问网络（以太网）上，如果有 N 台路由器，两两建立邻接关系需要 `N×(N-1)/2` 条：
- 5 台 → 10 条邻接
- 10 台 → **45 条邻接**

每条邻接都要同步 LSDB，泛洪的 LSA 数量呈平方增长，浪费带宽和 CPU。

**DR 的作用**：选出一个"中心节点"，所有路由器只跟 DR（和 BDR）建立 Full 邻接。

```
没有 DR：                       有 DR：
   R1 ─── R2                      R1     R2
   │ ╲   ╱ │                       ╲     ╱
   │  ╲ ╱  │                        ╲   ╱
   │   ╳   │          →             [DR]
   │  ╱ ╲  │                        ╱   ╲
   │ ╱   ╲ │                       ╱     ╲
   R3 ─── R4                      R3     R4
   
   6 条邻接                        4 条邻接（+BDR 备份）
```

**选举规则**：

```
1. 比接口的 OSPF 优先级 (Priority)     ← 大的赢！（注意跟 STP 相反）
   默认 1，范围 0-255
   Priority = 0 的路由器【永远不参与选举】
        ↓ 优先级相同
2. 比 Router ID                         ← 大的赢！
```

**⚠️ 三个必须记住的特点**：

1. **优先级和 Router ID 都是"大的赢"**（和 STP 的"小的赢"正好相反，极易记混）
2. **选举是非抢占的（Non-preemptive）**：DR 选出来后，即使后来加入一台优先级更高的路由器，**也不会抢占**。只有 DR 挂了才重选（此时 BDR 直接上位）。
   - 想强制重选，必须 `clear ip ospf process`（会中断业务）
3. **DR/BDR 是"接口级"概念，不是"路由器级"**：同一台路由器可以在一个网段是 DR，在另一个网段是 DROTHER。

**哪些网络类型需要 DR**：

| 网络类型 | 需要 DR？ | 默认 Hello/Dead | 需要手工配邻居？ |
|:--|:--|:--|:--|
| **Broadcast**（以太网） | ✅ 需要 | 10 / 40 | ❌ |
| **Non-Broadcast (NBMA)**（帧中继） | ✅ 需要 | 30 / 120 | ✅ 需要 `neighbor` |
| **Point-to-Point**（串口/PPP） | ❌ 不需要 | 10 / 40 | ❌ |
| **Point-to-Multipoint** | ❌ 不需要 | 30 / 120 | ❌ |
| **Point-to-Multipoint Non-Broadcast** | ❌ 不需要 | 30 / 120 | ✅ |
| **Loopback** | — | — | — |

> **实战技巧**：两台路由器之间的以太网链路（点对点使用），**手工改成 point-to-point 类型**能省掉 DR 选举，收敛更快：
> ```cisco
> R1(config-if)# ip ospf network point-to-point
> ```
> 这在数据中心和核心互联链路上是标准做法。

### 2.7 Cost 计算

```
Cost = 参考带宽 (Reference Bandwidth) / 接口带宽
```

**默认参考带宽 = 100 Mbps**（`10^8` bps）

| 接口带宽 | 计算 | Cost |
|:--|:--|:--|
| 10 Mbps | 100/10 | 10 |
| 100 Mbps | 100/100 | 1 |
| **1 Gbps** | 100/1000 = 0.1 → 取整 | **1** ← 问题！ |
| **10 Gbps** | 100/10000 = 0.01 → 取整 | **1** ← 问题！ |
| **100 Gbps** | | **1** ← 问题！ |

**⚠️ 严重问题**：默认参考带宽是 1991 年定的（当时 100M 是最快的）。现在**千兆、万兆、十万兆的 Cost 全是 1**，OSPF 无法区分它们的优劣，会把万兆链路和千兆链路当成等价路径做负载均衡——**性能灾难**。

**必须修改参考带宽**：
```cisco
R1(config-router)# auto-cost reference-bandwidth 100000     ! 单位 Mbps，即 100 Gbps
```

修改后：

| 接口带宽 | Cost |
|:--|:--|
| 100 Mbps | 1000 |
| 1 Gbps | 100 |
| 10 Gbps | 10 |
| 100 Gbps | 1 |

> ⚠️ **必须在全网所有 OSPF 路由器上统一修改！** 只改一部分会导致 Cost 计算不一致，产生次优路由甚至路由环路。修改时 IOS 会警告：
> ```
> % OSPF: Reference bandwidth is changed.
>         Please ensure reference bandwidth is consistent across all routers.
> ```

**手工指定接口 Cost**（优先级高于自动计算）：
```cisco
R1(config-if)# ip ospf cost 50
```

### 2.8 network 命令与通配符掩码

```cisco
R1(config)# router ospf 1
R1(config-router)# network 192.168.1.0 0.0.0.255 area 0
                            ↑            ↑          ↑
                        网络地址    通配符掩码    区域
```

**`network` 命令做两件事**：
1. **在匹配的接口上启用 OSPF**（发送/接收 Hello，建立邻居）
2. **把该接口所在的网段通告出去**（写进 LSA）

**通配符掩码写法**（回顾 [基础篇第 3 章](../00-基础篇/03-IP编址与子网划分.md)）：

```cisco
! 精确匹配一个接口（推荐，最安全）
network 192.168.1.1 0.0.0.0 area 0

! 匹配整个 /24
network 192.168.1.0 0.0.0.255 area 0

! 匹配 10.x.x.x 全部
network 10.0.0.0 0.255.255.255 area 0

! 匹配所有接口（偷懒写法，生产慎用）
network 0.0.0.0 255.255.255.255 area 0
```

> **最佳实践：用 `0.0.0.0` 精确匹配单个接口 IP。**
>
> 好处：意图明确，不会误把新加的接口带进 OSPF。用 `0.0.0.0 255.255.255.255` 的话，以后加一个连着外网的接口，会自动被 OSPF 纳管并把外网 Hello 发出去——既是安全隐患，也可能建立意外的邻居。

**现代替代方案：直接在接口下启用（IOS 15+，推荐）**
```cisco
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip ospf 1 area 0
```
更直观，不用算通配符掩码，接口级别可控。

### 2.9 passive-interface（重要的安全实践）

有些接口需要**把网段通告进 OSPF，但不希望在上面发 Hello**——比如接终端的 LAN 口。

```cisco
R1(config-router)# passive-interface GigabitEthernet0/0
```

**效果**：
- ✅ 该接口的网段仍然被通告（其他路由器能学到这个网段）
- ❌ 不再发送/接收 Hello，不会在此建立邻居

**为什么必须配**：
1. **安全**：防止有人接一台路由器进来伪造 OSPF 邻居，注入恶意路由
2. **性能**：不必要的 Hello 浪费带宽和 CPU
3. **稳定**：避免与不该建邻居的设备建立邻接

**最佳实践：默认全 passive，只放开需要的**
```cisco
R1(config-router)# passive-interface default              ! 全部设为 passive
R1(config-router)# no passive-interface GigabitEthernet0/1  ! 只放开互联口
R1(config-router)# no passive-interface GigabitEthernet0/2
```
这是"默认拒绝"的安全思维，强烈推荐。

---

## ③ 配置命令

### Cisco

```cisco
! ═══ 基础配置 ═══
R1(config)# router ospf 1                        ! 1 是进程号，本地有效，两端可以不同
R1(config-router)# router-id 1.1.1.1             ! ★ 必配
R1(config-router)# network 192.168.12.1 0.0.0.0 area 0
R1(config-router)# network 192.168.1.0 0.0.0.255 area 0
R1(config-router)# passive-interface default
R1(config-router)# no passive-interface GigabitEthernet0/1
R1(config-router)# auto-cost reference-bandwidth 100000    ! ★ 全网统一

! ═══ 接口下启用（现代写法，推荐）═══
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip ospf 1 area 0

! ═══ 接口参数调整 ═══
R1(config-if)# ip ospf cost 50                   ! 手工 cost
R1(config-if)# ip ospf priority 255              ! DR 优先级（大的赢）
R1(config-if)# ip ospf priority 0                ! 永不当 DR/BDR
R1(config-if)# ip ospf network point-to-point    ! 改网络类型，省掉 DR 选举
R1(config-if)# ip ospf hello-interval 5          ! 加快检测（两端必须一致）
R1(config-if)# ip ospf dead-interval 20
R1(config-if)# ip ospf mtu-ignore                ! 忽略 MTU 检查（应急）

! ═══ 认证（安全，生产必配）═══
! 接口级 MD5 认证
R1(config-if)# ip ospf message-digest-key 1 md5 MySecretKey
R1(config-router)# area 0 authentication message-digest

! 或用更现代的 SHA 认证（IOS 15+）
R1(config)# key chain OSPF-KEYS
R1(config-keychain)# key 1
R1(config-keychain-key)#  key-string MySecretKey
R1(config-keychain-key)#  cryptographic-algorithm hmac-sha-256
R1(config-if)# ip ospf authentication key-chain OSPF-KEYS

! ═══ 默认路由注入 ═══
R1(config-router)# default-information originate            ! 有默认路由才通告
R1(config-router)# default-information originate always     ! 无论如何都通告

! ═══ 查看命令（★ 排障核心）═══
R1# show ip ospf neighbor                         ! ① 邻居状态，最常用
R1# show ip ospf neighbor detail
R1# show ip ospf interface                        ! ② 接口参数，对比两端
R1# show ip ospf interface brief
R1# show ip ospf                                  ! ③ 进程信息、Router ID、区域
R1# show ip ospf database                         ! ④ LSDB
R1# show ip ospf database router
R1# show ip route ospf                            ! ⑤ 学到的路由
R1# show ip protocols                             ! ⑥ 协议配置概览

! ═══ 调试与重置 ═══
R1# debug ip ospf adj                             ! 调试邻接建立（慎用）
R1# debug ip ospf hello
R1# undebug all
R1# clear ip ospf process                         ! ⚠️ 会中断所有邻居
R1# clear ip ospf neighbor <ip>                   ! 只重置一个邻居
```

### 输出解读

```cisco
R1# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2           1   FULL/DR         00:00:35    192.168.12.2    GigabitEthernet0/1
3.3.3.3           1   FULL/BDR        00:00:38    192.168.13.3    GigabitEthernet0/2
4.4.4.4           1   2WAY/DROTHER    00:00:33    192.168.12.4    GigabitEthernet0/1
    ↑             ↑     ↑      ↑          ↑
 邻居的        优先级  状态  对方角色   Dead 倒计时
 Router ID                            （归零就断邻居）
```

**状态含义**：
- `FULL/DR` = 完全邻接，对方是 DR ✅
- `FULL/BDR` = 完全邻接，对方是 BDR ✅
- `FULL/-` = 完全邻接（点对点网络，没有 DR 概念）✅
- **`2WAY/DROTHER`** = 双方都不是 DR/BDR，**这是正常的**，不是故障
- `EXSTART/DR` = ❌ 卡在 ExStart，查 MTU
- `INIT/-` = ❌ 单向通信，对方收不到我的 Hello

**Dead Time 的读法**：从 40 秒往下倒计时，每收到一个 Hello 就重置回 40。如果这个数字一直在减小接近 0，说明**收不到对方的 Hello 了**。

### 三厂商对照

| 目的 | Cisco | H3C | 华为 |
|:--|:--|:--|:--|
| 启动进程 | `router ospf 1` | `ospf 1` | `ospf 1` |
| Router ID | `router-id 1.1.1.1` | `ospf 1 router-id 1.1.1.1` | `ospf 1 router-id 1.1.1.1` |
| 通告网段 | `network 192.168.1.0 0.0.0.255 area 0` | `area 0` → `network 192.168.1.0 0.0.0.255` | `area 0` → `network 192.168.1.0 0.0.0.255` |
| 接口启用 | `ip ospf 1 area 0` | `ospf enable 1 area 0` | `ospf enable 1 area 0` |
| 静默接口 | `passive-interface Gi0/0` | `silent-interface Gi0/0` | `silent-interface Gi0/0` |
| 接口 cost | `ip ospf cost 50` | `ospf cost 50` | `ospf cost 50` |
| DR 优先级 | `ip ospf priority 255` | `ospf dr-priority 255` | `ospf dr-priority 255` |
| 网络类型 | `ip ospf network point-to-point` | `ospf network-type p2p` | `ospf network-type p2p` |
| 参考带宽 | `auto-cost reference-bandwidth 100000` | `bandwidth-reference 100000` | `bandwidth-reference 100000` |
| 查邻居 | `show ip ospf neighbor` | `display ospf peer` | `display ospf peer` |
| 查接口 | `show ip ospf interface` | `display ospf interface` | `display ospf interface` |
| 查 LSDB | `show ip ospf database` | `display ospf lsdb` | `display ospf lsdb` |
| 查路由 | `show ip route ospf` | `display ip routing-table protocol ospf` | `display ip routing-table protocol ospf` |

> **结构差异**：Cisco 的 `network` 命令直接带 `area` 参数；**H3C/华为要先进 `area` 视图，再写 `network`**：
> ```
> [H3C] ospf 1 router-id 1.1.1.1
> [H3C-ospf-1] area 0
> [H3C-ospf-1-area-0.0.0.0] network 192.168.1.0 0.0.0.255
> ```
>
> **术语差异**：Cisco 叫 `passive-interface`，H3C/华为叫 `silent-interface`。

---

## ④ 配套实验：三路由器单区域 OSPF

**拓扑**：
```
                  ┌──────────────┐
                  │      R1      │ Lo0: 1.1.1.1/32
                  │  RID 1.1.1.1 │
                  └──┬────────┬──┘
        192.168.12.0/24│      │192.168.13.0/24
                .1     │      │     .1
                       │      │
                .2     │      │     .3
      ┌────────────────┴┐    ┌┴─────────────────┐
      │       R2        │    │       R3         │
      │   RID 2.2.2.2   │    │   RID 3.3.3.3    │
      │  Lo0: 2.2.2.2   │    │  Lo0: 3.3.3.3    │
      └────────┬────────┘    └────────┬─────────┘
               │  192.168.23.0/24      │
          .2   └───────────────────────┘  .3
               
   LAN2: 192.168.2.0/24        LAN3: 192.168.3.0/24
```

### Step 1：R1 配置

```cisco
R1(config)# interface Loopback0
R1(config-if)# ip address 1.1.1.1 255.255.255.255

R1(config)# interface GigabitEthernet0/1
R1(config-if)# description ### To-R2 ###
R1(config-if)# ip address 192.168.12.1 255.255.255.0
R1(config-if)# no shutdown

R1(config)# interface GigabitEthernet0/2
R1(config-if)# description ### To-R3 ###
R1(config-if)# ip address 192.168.13.1 255.255.255.0
R1(config-if)# no shutdown

! ── OSPF ──
R1(config)# router ospf 1
R1(config-router)# router-id 1.1.1.1
R1(config-router)# auto-cost reference-bandwidth 100000
R1(config-router)# passive-interface default
R1(config-router)# no passive-interface GigabitEthernet0/1
R1(config-router)# no passive-interface GigabitEthernet0/2
R1(config-router)# network 192.168.12.1 0.0.0.0 area 0
R1(config-router)# network 192.168.13.1 0.0.0.0 area 0
R1(config-router)# network 1.1.1.1 0.0.0.0 area 0
```

### Step 2：R2 配置

```cisco
R2(config)# interface Loopback0
R2(config-if)# ip address 2.2.2.2 255.255.255.255

R2(config)# interface GigabitEthernet0/1
R2(config-if)# ip address 192.168.12.2 255.255.255.0
R2(config-if)# no shutdown

R2(config)# interface GigabitEthernet0/2
R2(config-if)# ip address 192.168.23.2 255.255.255.0
R2(config-if)# no shutdown

R2(config)# interface GigabitEthernet0/0
R2(config-if)# ip address 192.168.2.1 255.255.255.0     ! LAN 网关
R2(config-if)# no shutdown

R2(config)# router ospf 1
R2(config-router)# router-id 2.2.2.2
R2(config-router)# auto-cost reference-bandwidth 100000
R2(config-router)# passive-interface default
R2(config-router)# no passive-interface GigabitEthernet0/1
R2(config-router)# no passive-interface GigabitEthernet0/2
R2(config-router)# network 192.168.12.2 0.0.0.0 area 0
R2(config-router)# network 192.168.23.2 0.0.0.0 area 0
R2(config-router)# network 192.168.2.0 0.0.0.255 area 0      ! LAN 也通告（但 passive）
R2(config-router)# network 2.2.2.2 0.0.0.0 area 0
```

**R3 配置类似，Router ID 用 3.3.3.3，LAN 是 192.168.3.0/24。**

### Step 3：验证邻居

```cisco
R1# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2           1   FULL/BDR        00:00:35    192.168.12.2    GigabitEthernet0/1
3.3.3.3           1   FULL/BDR        00:00:38    192.168.13.1    GigabitEthernet0/2
```
✅ 两个邻居都是 **FULL**。

**如果不是 FULL，对照 2.4 节的卡点诊断表排查。**

### Step 4：验证 LSDB（关键理解点）

```cisco
R1# show ip ospf database

            OSPF Router with ID (1.1.1.1) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age  Seq#       Checksum Link count
1.1.1.1         1.1.1.1         245  0x80000005 0x00A1B2 5
2.2.2.2         2.2.2.2         238  0x80000004 0x00C3D4 5
3.3.3.3         3.3.3.3         241  0x80000004 0x00E5F6 5

                Net Link States (Area 0)

Link ID         ADV Router      Age  Seq#       Checksum
192.168.12.1    1.1.1.1         245  0x80000002 0x001234
192.168.13.1    1.1.1.1         240  0x80000002 0x005678
192.168.23.2    2.2.2.2         238  0x80000002 0x009ABC
```

**★ 关键验证**：在 R2 和 R3 上执行同样的命令，**输出应该完全一致**（除了顶部的 "OSPF Router with ID" 和 Age 略有差异）。

**这就是链路状态协议的本质：区域内所有路由器拥有完全相同的 LSDB（全网地图）。**

如果 LSDB 不一致，说明同步有问题——这是 OSPF 排障的第一原则。

### Step 5：验证路由表

```cisco
R1# show ip route ospf

O     2.2.2.2/32 [110/101] via 192.168.12.2, 00:05:12, GigabitEthernet0/1
O     3.3.3.3/32 [110/101] via 192.168.13.3, 00:05:10, GigabitEthernet0/2
O     192.168.2.0/24 [110/101] via 192.168.12.2, 00:05:12, GigabitEthernet0/1
O     192.168.3.0/24 [110/101] via 192.168.13.3, 00:05:10, GigabitEthernet0/2
O     192.168.23.0/24 [110/200] via 192.168.13.3, 00:05:10, GigabitEthernet0/2
                       [110/200] via 192.168.12.2, 00:05:12, GigabitEthernet0/1
                        ↑     ↑                              ↑
                       AD  Metric                    两条等价路径，ECMP 负载均衡
```

**注意 Cost 计算**（参考带宽改成 100000 后）：
- 千兆接口 Cost = 100000/1000 = **100**
- 到 R2 的 Loopback：出接口 Gi0/1 (100) + R2 的 Loopback (1) = **101**
- 到 192.168.23.0/24：经 R2 或 R3 都是 100+100 = **200**，等价 → ECMP

### Step 6：验证 ECMP 负载均衡

```cisco
R1# show ip route 192.168.23.0
Routing entry for 192.168.23.0/24
  Known via "ospf 1", distance 110, metric 200, type intra area
  Routing Descriptor Blocks:
  * 192.168.13.3, from 3.3.3.3, 00:06:22 ago, via GigabitEthernet0/2
      Route metric is 200, traffic share count is 1
    192.168.12.2, from 2.2.2.2, 00:06:22 ago, via GigabitEthernet0/1
      Route metric is 200, traffic share count is 1
                                                  ↑ 两条路径均分流量
```

**OSPF 默认支持 4 条等价路径负载均衡**，可以调整：
```cisco
R1(config-router)# maximum-paths 8
```

### Step 7：故障注入实验（重点）

**故障 A：MTU 不匹配（造成 ExStart 卡死）**

```cisco
R2(config)# interface GigabitEthernet0/1
R2(config-if)# ip mtu 1400                       ! R1 侧是默认 1500
R1# clear ip ospf process
```

**观察**：
```cisco
R1# show ip ospf neighbor
Neighbor ID     Pri   State      Dead Time   Address         Interface
2.2.2.2           1   EXSTART/DR  00:00:35   192.168.12.2    GigabitEthernet0/1
                      ↑↑↑↑↑↑↑ 卡住了
```

**日志**：
```
%OSPF-5-ADJCHG: Process 1, Nbr 2.2.2.2 on GigabitEthernet0/1 from EXSTART to DOWN, 
Neighbor Down: Too many retransmissions
```

**排查**：
```cisco
R1# show ip ospf interface Gi0/1 | include MTU
  MTU is 1500 bytes

R2# show ip ospf interface Gi0/1 | include MTU
  MTU is 1400 bytes                              ← 找到问题
```

**修复**：
```cisco
R2(config-if)# ip mtu 1500
! 或应急处理
R1(config-if)# ip ospf mtu-ignore
R2(config-if)# ip ospf mtu-ignore
```

**故障 B：Hello 间隔不匹配**

```cisco
R2(config-if)# ip ospf hello-interval 5          ! R1 是默认 10
```

**观察**：邻居直接消失（连 Init 都到不了），因为 Hello 参数不匹配的报文会被直接丢弃。

**排查**：
```cisco
R1# show ip ospf interface Gi0/1 | include Timer
  Timer intervals configured, Hello 10, Dead 40, ...
R2# show ip ospf interface Gi0/1 | include Timer
  Timer intervals configured, Hello 5, Dead 20, ...    ← 不一致
```

> 注意：改 hello-interval 会**自动把 dead-interval 改成 4 倍**。

**故障 C：Area 不匹配**

```cisco
R2(config-router)# no network 192.168.12.2 0.0.0.0 area 0
R2(config-router)# network 192.168.12.2 0.0.0.0 area 1
```

**观察**：邻居消失。**日志会明确提示**：
```
%OSPF-4-ERRRCV: Received invalid packet: mismatched area ID from backbone area 
from 192.168.12.2, GigabitEthernet0/1
```

**故障 D：Router ID 重复**

```cisco
R3(config-router)# router-id 2.2.2.2              ! 和 R2 重复
R3# clear ip ospf process
```

**观察**：邻居状态不断在 FULL 和 DOWN 之间翻转，日志刷屏：
```
%OSPF-4-DUP_RTRID_NBR: OSPF detected duplicate router-id 2.2.2.2 from 
192.168.13.3 on interface GigabitEthernet0/2
```

**这是个非常好的实验**，因为在真实网络里 Router ID 重复往往是"复制粘贴配置"造成的，症状很迷惑（时通时不通）。

---

## ⑤ 排障思路

### 标准排查流程（记住这个顺序）

```
① show ip ospf neighbor
   └─ 有邻居吗？状态是 FULL 吗？
      ├─ 没有邻居        → 走第 ② 步
      ├─ 卡在 INIT       → 对方收不到我的 Hello（ACL？单向链路？）
      ├─ 卡在 EXSTART    → 【MTU 不匹配】99% 是这个
      ├─ 停在 2WAY       → 检查是否 DROTHER 之间（正常）
      └─ FULL 但翻转     → Router ID 重复 / 链路不稳 / CPU 高

② show ip ospf interface <接口>
   └─ 逐项对比两端的五个条件：
      Area ID / Hello&Dead / 认证 / Stub标志 / 掩码
      同时看 MTU 和 Network Type

③ show ip ospf database
   └─ LSDB 是否与邻居一致？
      不一致 → 同步有问题，检查 MTU / 认证 / 内存

④ show ip route ospf
   └─ 路由学到了吗？
      LSDB 有但路由表没有 → 检查 AD、是否有更优路由、是否被过滤

⑤ show ip protocols
   └─ 看 network 语句、passive-interface、参考带宽、Router ID 是否符合预期
```

### 症状 → 根因 速查表

| 症状 | 最可能的根因 | 验证命令 |
|:--|:--|:--|
| 完全没有邻居 | 接口没进 OSPF / 是 passive / 接口 down | `show ip ospf interface brief` |
| 卡在 **INIT** | ACL 拦了 224.0.0.5 组播 / 单向链路 | `show access-lists`、抓包 |
| 卡在 **EXSTART/EXCHANGE** | **MTU 不匹配** | `show ip ospf interface \| inc MTU` |
| 停在 **2WAY** | DROTHER 之间（**正常**） | `show ip ospf neighbor` 看角色 |
| 邻居反复翻转 | Router ID 重复 / 链路抖动 / CPU 过高 | `show logging`、`show processes cpu` |
| 邻居 FULL 但学不到路由 | 对方没 `network` 该网段 / 被过滤 | 对方 `show ip protocols` |
| 路由存在但走了次优路径 | Cost 计算问题 / 参考带宽不一致 | `show ip ospf interface \| inc Cost` |
| 万兆和千兆被当成等价 | **参考带宽没改** | `show ip ospf \| inc Reference` |
| Hello 参数看似一致但仍不通 | 认证不匹配 | `show ip ospf interface \| inc auth` |
| LSDB 不一致 | MTU / 认证 / 内存不足 | `show ip ospf database` 逐台对比 |

### OSPF 必备 ACL 放行

如果链路上有 ACL，**必须放行 OSPF**：
```cisco
! OSPF 是 IP 协议号 89，不是 TCP/UDP，不能写端口
access-list 101 permit ospf any any

! 更精确的写法（放行组播和单播）
access-list 101 permit ospf any host 224.0.0.5
access-list 101 permit ospf any host 224.0.0.6
access-list 101 permit ospf any any
```

**这是"卡在 INIT 状态"最常见的原因**：ACL 只放行了 TCP/UDP，把 OSPF 的组播报文拦掉了。

---

## ⑥ 考点提示 + 自测题

### 考点

- **邻居状态机 7 个状态**和每个状态的卡点原因（**必背**）。
- **建立邻居的 5 个条件**（Area / Hello&Dead / 认证 / Stub标志 / 掩码）。
- **MTU 不匹配 → ExStart 卡死**，是最经典的送分题。
- **DR/BDR 选举：优先级大的赢，非抢占**（和 STP 相反）。
- **默认参考带宽 100Mbps 的问题**及修改方法。
- **2WAY 是正常状态**（DROTHER 之间）。
- **Router ID 选举顺序**和手工配置的重要性。

### 自测题

**1.** OSPF 邻居卡在 `EXSTART` 状态，最可能是什么原因？为什么？

<details><summary>答案</summary>

**MTU 不匹配。**

**原理**：ExStart 阶段路由器要交换 **DD（Database Description）报文**来协商主从关系和数据库摘要。DD 报文会按照**接口 MTU 的大小**来构造。

如果 R1 的 MTU 是 1500，R2 是 1400：
- R1 发出一个 1500 字节的 DD 报文
- R2 的接口 MTU 只有 1400，**收到后直接丢弃**（OSPF 有显式的 MTU 检查机制，收到 MTU 大于自己接口 MTU 的 DD 会拒绝）
- R1 收不到响应，超时重传，一直卡在 ExStart

**日志会提示**：
```
%OSPF-5-ADJCHG: Process 1, Nbr 2.2.2.2 on Gi0/1 from EXSTART to DOWN, 
Neighbor Down: Too many retransmissions
```

**排查**：
```cisco
R1# show ip ospf interface Gi0/1 | include MTU
R2# show ip ospf interface Gi0/1 | include MTU
```

**修复（两种）**：
```cisco
! 方法 1（推荐）：统一 MTU
R2(config-if)# ip mtu 1500

! 方法 2（应急）：两端都忽略 MTU 检查
R1(config-if)# ip ospf mtu-ignore
R2(config-if)# ip ospf mtu-ignore
```

**为什么方法 2 是治标**：忽略检查后邻居能起来，但如果实际路径 MTU 确实不足，大的 LSU 报文仍然会被丢弃，导致 LSDB 同步不完整——问题会以更隐蔽的形式出现。**优先统一 MTU。**

**MTU 不匹配的常见来源**：
- 一端配了 GRE/IPsec 隧道，改小了 MTU
- 一端启用了巨帧（jumbo frame）
- 混合厂商设备的默认 MTU 定义不同（比如某些平台把 MTU 理解为含/不含以太网头）
</details>

**2.** 两台路由器的 OSPF 邻居状态一直是 `2WAY`，这一定是故障吗？

<details><summary>答案</summary>

**不一定，很可能是完全正常的。**

在**广播网络（以太网）**上，OSPF 会选举 DR 和 BDR。规则是：
- 所有路由器与 **DR** 建立 **FULL** 邻接
- 所有路由器与 **BDR** 建立 **FULL** 邻接
- **DROTHER 之间只保持 `2WAY`**，不建立 FULL 邻接

**这是设计如此**，目的是减少邻接数量和 LSA 泛洪开销。DROTHER 之间不需要直接同步数据库——它们都通过 DR 获得完整的 LSDB。

**如何判断是正常还是故障**：
```cisco
R1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2           1   FULL/DR         00:00:35    192.168.1.2     Gi0/1
3.3.3.3           1   FULL/BDR        00:00:38    192.168.1.3     Gi0/1
4.4.4.4           1   2WAY/DROTHER    00:00:33    192.168.1.4     Gi0/1
                      ↑↑↑↑ ↑↑↑↑↑↑↑↑
                    正常！对方是 DROTHER，我也是 DROTHER
```

**什么时候 2WAY 是故障**：
1. **点对点链路上出现 2WAY** —— P2P 网络没有 DR 概念，应该直接 FULL
2. **网络里所有路由器都是 2WAY，没有任何 FULL** —— 说明 DR 选举出了问题，可能是所有接口的 `ip ospf priority` 都设成了 0
3. **本该是 DR/BDR 的设备也停在 2WAY** —— 检查 DR 选举

**验证 DR/BDR 是谁**：
```cisco
R1# show ip ospf interface Gi0/1 | include Designated
  Designated Router (ID) 2.2.2.2, Interface address 192.168.1.2
  Backup Designated router (ID) 3.3.3.3, Interface address 192.168.1.3
```
</details>

**3.** 默认参考带宽是 100Mbps，为什么这在现代网络里是个严重问题？

<details><summary>答案</summary>

**因为 Cost 计算会取整，导致千兆及以上的链路 Cost 全部为 1，OSPF 无法区分它们。**

```
Cost = 参考带宽(100Mbps) / 接口带宽

100 Mbps  → 100/100   = 1
1 Gbps    → 100/1000  = 0.1  → 取整为 1
10 Gbps   → 100/10000 = 0.01 → 取整为 1
100 Gbps  →                  → 1
```

**后果**：一条 10Gbps 链路和一条 1Gbps 链路的 Cost 都是 1，OSPF 认为它们**完全等价**，会做 ECMP 负载均衡，把一半流量塞进千兆链路——**万兆链路的价值被浪费，千兆链路被打满，业务性能严重受损**，而且这种问题非常难被发现（路由表看起来完全正常）。

**修复**：
```cisco
R1(config-router)# auto-cost reference-bandwidth 100000     ! 单位是 Mbps = 100 Gbps
```

修改后：

| 带宽 | Cost |
|:--|:--|
| 100 Mbps | 1000 |
| 1 Gbps | 100 |
| 10 Gbps | 10 |
| 40 Gbps | 2 |
| 100 Gbps | 1 |

**⚠️ 三个必须注意的点**：

1. **必须在全网所有 OSPF 路由器上统一修改。** 只改一部分会导致不同路由器对同一条路径算出不同的 Cost，产生次优路由甚至路由环路。IOS 会警告你这一点。

2. **修改是全局的，影响所有接口。** 已经手工配了 `ip ospf cost` 的接口不受影响（手工值优先）。

3. **参考带宽要留余量。** 设成 100000（100G）而不是 10000（10G），是为了给未来的 400G 链路留空间。如果现在设 10000，将来上了 100G 链路，Cost 又会挤在一起。

**同样的问题也存在于 EIGRP**（默认最大带宽 10Gbps），修复方法是 `metric rib-scale` 或使用 EIGRP 的宽度量（wide metrics）。
</details>

**4.** DR 选举的规则是什么？如果后来接入一台优先级更高的路由器，会抢占吗？

<details><summary>答案</summary>

**选举规则（两步）**：
1. **比接口的 OSPF 优先级（Priority），大的赢。** 默认 1，范围 0–255。**优先级为 0 的路由器永不参与选举。**
2. **优先级相同时，比 Router ID，大的赢。**

> ⚠️ **注意方向**：OSPF 是 **"大的赢"**，而 STP 是 **"小的赢"**。这两个极易记混，是考试常见的陷阱题。

**不会抢占。OSPF 的 DR 选举是非抢占的（Non-preemptive）。**

**原因**：DR 变更代价很高——所有路由器要与新 DR 重新建立 FULL 邻接、重新同步 LSDB，期间可能出现短暂的路由黑洞。如果允许抢占，每次有新设备加入（或者某台设备重启）都会触发一次全网震荡，非常不稳定。

**实际行为**：
- 新加入的高优先级路由器只会成为 **DROTHER**
- 只有当 **DR 失效** 时，BDR 自动升级为 DR，然后在剩下的路由器中重新选举 BDR
- 想强制重选，必须在**所有**路由器上执行 `clear ip ospf process`（会中断业务）

**实践建议**：
```cisco
! 规划阶段就指定好 DR/BDR
CoreR1(config-if)# ip ospf priority 255       ! 期望的 DR
CoreR2(config-if)# ip ospf priority 100       ! 期望的 BDR
AccessR3(config-if)# ip ospf priority 0       ! 永不参选
AccessR4(config-if)# ip ospf priority 0
```

**更好的办法：干脆不要 DR。** 如果链路实际上是点对点使用的（两台设备之间的直连），改成 P2P 类型：
```cisco
R1(config-if)# ip ospf network point-to-point
R2(config-if)# ip ospf network point-to-point
```
好处：
- 省掉 DR 选举，收敛更快
- 不产生 Type-2 (Network) LSA，LSDB 更小
- 没有 DR 抢占的困扰

**这在数据中心和核心互联链路上是标准做法。**
</details>

**5.** R1 和 R2 的 OSPF 邻居是 FULL，但 R1 学不到 R2 后面 LAN 网段的路由。可能是什么原因？

<details><summary>答案</summary>

邻居 FULL 说明**邻居关系本身没问题**，问题出在"R2 有没有把那个网段通告出来"。

**排查顺序**：

**① R2 有没有把该网段 `network` 进 OSPF？**
```cisco
R2# show ip protocols
Routing Protocol is "ospf 1"
  ...
  Routing for Networks:
    192.168.12.2 0.0.0.0 area 0
    192.168.23.2 0.0.0.0 area 0
    ! ← 如果这里没有 LAN 网段，就是根因
    
R2# show ip ospf interface brief
Interface  PID  Area  IP Address/Mask   Cost  State Nbrs F/C
Gi0/1      1    0     192.168.12.2/24   100   DR    1/1
Gi0/2      1    0     192.168.23.2/24   100   DR    1/1
! ← LAN 接口 Gi0/0 不在列表里 = 没启用 OSPF
```
**修复**：
```cisco
R2(config-router)# network 192.168.2.0 0.0.0.255 area 0
```

**② 该接口是否 up？**
```cisco
R2# show ip interface brief | include Gi0/0
GigabitEthernet0/0   192.168.2.1   YES manual  down   down     ← 接口 down 就不会通告
```
OSPF 只通告 **up 状态**接口的网段。

**③ R2 上是否配了 `passive-interface`（这不影响通告）？**

注意：`passive-interface` **不会**阻止网段被通告，它只是不发 Hello。所以这不是原因。但如果误用了 `passive-interface default` 而 LAN 接口的 network 语句也没写，那网段就不会被通告。

**④ 是否有路由过滤？**
```cisco
R2# show ip protocols | include Filter|distribute
  Outgoing update filter list for all interfaces is not set
  
R1(config-router)# ! 检查 R1 侧是否有 distribute-list in
```

**⑤ LSDB 里有没有？（关键判断）**
```cisco
R1# show ip ospf database router adv-router 2.2.2.2

  Link connected to: a Stub Network
     (Link ID) Network/subnet number: 192.168.2.0
     (Link Data) Network Mask: 255.255.255.0
     ! ← 如果这里有，说明 R2 通告了，问题在 R1 侧
     ! ← 如果没有，说明 R2 根本没通告
```

**这是个关键的分界点**：
- **LSDB 里有，路由表里没有** → 问题在 R1 侧：可能是有更优的路由（比如静态路由 AD=1）抢占了，或者被 `distribute-list in` 过滤了
- **LSDB 里也没有** → 问题在 R2 侧：没通告

**⑥ 如果 LSDB 有但路由表没有，检查是否被更优路由覆盖**：
```cisco
R1# show ip route 192.168.2.0
Routing entry for 192.168.2.0/24
  Known via "static", distance 1                    ← 静态路由 AD=1 赢了 OSPF 的 110
```

**排障思维总结**：**用 LSDB 作为分界线**。LSDB 是"我收到了什么"，路由表是"我采纳了什么"。两者的差异告诉你问题在发送侧还是接收侧。
</details>

---

**上一章** ← [03 EtherChannel 链路聚合](03-EtherChannel链路聚合.md) ｜ **下一章** → [05 ACL 访问控制列表](05-ACL访问控制列表.md)
