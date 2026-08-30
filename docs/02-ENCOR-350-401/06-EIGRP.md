# 06 · EIGRP

## ① 这章解决什么问题

OSPF 已经能自动学路由了，为什么还要 EIGRP？

因为它们的设计哲学不同：

- **OSPF**：每台路由器都存一份完整的全网地图，自己跑 Dijkstra 算最短路。**准确、无环，但内存和 CPU 开销大，且拓扑变化时全区域都要重算**。
- **EIGRP**：只跟邻居交换"我到某地要多远"，但**预先算好备份路径**。**收敛极快（有备份路径时几乎瞬时），资源开销小，但需要精巧的机制保证无环**。

理解 EIGRP 的关键是理解 **DUAL 算法**——它用一个巧妙的数学条件（可行性条件）保证了"备份路径一定不会形成环路"，从而可以放心地预先计算并立即切换。

**这个思路很值得学**：不是每次都重算，而是**提前算好一个数学上保证安全的备份，故障时直接用**。

---

## ② 原理讲透

### 2.1 EIGRP 是什么

| 特性 | EIGRP |
|:--|:--|
| 类型 | **高级距离矢量 / 混合型** |
| 算法 | **DUAL (Diffusing Update Algorithm)** |
| 传输 | 直接跑在 IP 之上，**协议号 88** |
| 组播地址 | **224.0.0.10** |
| 可靠性 | 自研 **RTP (Reliable Transport Protocol)** |
| 管理距离 | **内部 90，外部 170，汇总 5** |
| 度量值 | 复合度量（带宽 + 延迟 + 可靠性 + 负载） |
| 支持 VLSM | ✅ |
| 状态 | 2013 年后 Cisco 公开为 **RFC 7868**（信息性），但实际仍主要在 Cisco 设备上 |

**EIGRP 的三张表**（对比 OSPF 的 LSDB）：

| 表 | 内容 | 查看命令 |
|:--|:--|:--|
| **邻居表** (Neighbor Table) | 直连的 EIGRP 邻居 | `show ip eigrp neighbors` |
| **拓扑表** (Topology Table) | **所有已知路径**，包括备份路径 | `show ip eigrp topology` |
| **路由表** (Routing Table) | 拓扑表中的最优路径 | `show ip route eigrp` |

**关键理解**：拓扑表里有 **Successor（最优）和 Feasible Successor（备份）**，路由表里只放 Successor。这就是 EIGRP 能快速收敛的基础。

### 2.2 DUAL 算法的四个核心概念（★★★ 必须彻底搞懂）

这四个概念是 EIGRP 的灵魂，也是考试重点。

```
        R1 ──── 10 ──── R2 ──── 20 ──── [目标网络 X]
         │                       │
         └──── 30 ──── R3 ──── 15 ─┘
```

| 术语 | 英文 | 定义 | 站在 R1 的视角 |
|:--|:--|:--|:--|
| **RD** | Reported Distance<br>（也叫 AD, Advertised Distance） | **邻居告诉我的、它到目标的距离** | R2 说它到 X 是 20；R3 说它到 X 是 15 |
| **FD** | Feasible Distance | **我到目标的总距离**（我到邻居 + 邻居到目标） | 经 R2 = 10+20 = **30**<br>经 R3 = 30+15 = **45** |
| **Successor** | 后继 | **FD 最小的那条路径的下一跳** | R2（FD=30 < 45） |
| **FS** | Feasible Successor<br>（可行后继） | **满足可行性条件的备份路径** | 看 R3 是否满足 FC |

#### 可行性条件（Feasibility Condition, FC）—— ★ 最重要的公式

```
   邻居的 RD  <  当前的 FD
   
   （邻居到目标的距离，必须严格小于我到目标的最优距离）
```

**验算上面的例子**：
- 当前 FD = 30（经 R2）
- R3 的 RD = 15
- **15 < 30 ✅ 满足 FC → R3 是 Feasible Successor**

**如果 R3 说它到 X 是 35 呢？**
- 35 < 30？**否** ❌
- R3 **不是** FS，只是拓扑表里的一条普通路径

#### 为什么这个条件能保证无环？（关键理解）

**反证法思考**：

如果 R3 到 X 的距离（15）**小于** R1 到 X 的距离（30），那说明：
- R3 有一条**不经过 R1** 的路径到 X
- 因为如果 R3 要经过 R1 才能到 X，那它的距离至少是 `30 + R3到R1的距离` > 30

**所以：只要邻居的 RD < 我的 FD，就能数学上保证它不是"绕回我自己"的路径。**

这就是 EIGRP 敢于**预先把 FS 存起来，故障时立即切换、不做任何计算**的底气。

> **对比 OSPF**：OSPF 靠"每台路由器有完整地图 + 跑 SPF 算法"来保证无环，代价是每次拓扑变化都要重算。EIGRP 靠一个简单的不等式保证无环，代价是这个条件比较保守——**有些实际无环的路径会被排除在 FS 之外**。

### 2.3 收敛过程：有 FS vs 没有 FS

#### 情况 A：有 FS —— 瞬时切换

```
   Successor (R2) 失效
        ↓
   路由表中该路由被移除
        ↓
   ★ 直接把 FS (R3) 提升为 Successor，装入路由表 ★
        ↓
   完成！耗时：几十毫秒
   
   路由始终处于 Passive 状态（P），没有查询过程
```

#### 情况 B：没有 FS —— 进入 Active 状态查询

```
   Successor 失效，且拓扑表里没有满足 FC 的备份
        ↓
   路由进入 ★ Active 状态 ★（A）
        ↓
   向所有邻居发送 QUERY："你们谁能到 X？距离多少？"
        ↓
   等待所有邻居的 REPLY
        ↓
   收齐所有 REPLY 后，重新计算，选出新的 Successor
        ↓
   路由回到 Passive 状态（P）
   
   耗时：取决于查询范围和邻居响应速度
```

> **术语澄清（极易误解）**：
> - **Passive (P)** = **正常状态**！路由稳定，有可用路径。
> - **Active (A)** = **异常状态**！正在查询，路由暂时不可用。
>
> 这和直觉相反——很多人以为 Active 是"活跃可用"。**在 EIGRP 里，看到 Active 就是有问题。**

#### SIA（Stuck In Active）—— EIGRP 最严重的故障

如果某个邻居**一直不回 REPLY**（设备 CPU 过载、链路单向、软件 bug），查询者会一直等待。

**默认 Active Timer 是 3 分钟**，超时后：
- 路由器会**强制断开那个没回应的邻居关系**
- 日志：`%DUAL-3-SIA: Route 10.1.1.0/24 stuck-in-active state in IP-EIGRP(0) 100. Cleaning up`
- **后果：邻居关系重建，可能引发连锁震荡**

**SIA 的根本原因：查询范围太大。** 查询是**扩散式**的——R1 问 R2，R2 如果也不知道就去问 R3……一圈问下来可能涉及几十台设备。任何一台响应慢，整个查询就卡住。

**解决 SIA 的方法（按有效性排序）**：

| 方法 | 说明 |
|:--|:--|
| **① 路由汇总** | ★ **最有效**。汇总点会自动回复 REPLY（"我有汇总路由，不用往下问了"），**从根本上限制查询范围** |
| **② Stub 路由** | 分支路由器配成 Stub，**上游不会向它发送查询** |
| **③ 优化网络设计** | 减少冗余路径的复杂度，避免全 mesh |
| **④ 调整 Active Timer** | 治标不治本，`timers active-time <分钟>` |
| **⑤ SIA-Query/SIA-Reply** | IOS 12.4+ 自动支持，在超时前先探测邻居是否还活着，避免误断邻居 |

> **实战原则：EIGRP 网络必须做汇总和 Stub，否则规模一大必出 SIA。** 这是 EIGRP 相比 OSPF 的一个主要运维负担。

### 2.4 度量值计算

**经典公式（默认只用带宽和延迟）**：

```
Metric = 256 × [ K1×Bandwidth + (K2×Bandwidth)/(256−Load) + K3×Delay ] × [ K5/(Reliability+K4) ]

默认 K 值：K1=1, K2=0, K3=1, K4=0, K5=0
简化为：
Metric = 256 × ( Bandwidth + Delay )
```

其中：
```
Bandwidth = 10^7 / 路径上【最小】的接口带宽(Kbps)
Delay     = 路径上【所有】接口延迟之和(微秒) / 10
```

**计算示例**：
```
R1 ──1Gbps──> R2 ──100Mbps──> [目标]

Bandwidth = 10^7 / 100000 (取最小带宽 100Mbps = 100000 Kbps) = 100
Delay     = (10 + 100) / 10 = 11
            ↑ 1Gbps 接口延迟 10us，100Mbps 接口延迟 100us

Metric = 256 × (100 + 11) = 28416
```

**关键点**：
- **带宽取路径上的最小值**（瓶颈决定）
- **延迟是所有接口的累加**
- **默认只有 K1(带宽) 和 K3(延迟) 生效**

> ⚠️ **K 值必须两端一致**，否则邻居建不起来。而且**不要随便改 K 值**——把 Load 和 Reliability 加入计算会导致 Metric 不断变化，引发路由震荡。

**Wide Metrics（EIGRP 命名模式，IOS 15+）**：

经典度量值最大只能表示到 10Gbps（因为 `10^7/带宽` 在更高带宽下都趋近于 1，无法区分）。**这和 OSPF 参考带宽的问题一模一样。**

命名模式用 64 位宽度量解决：
```cisco
R1(config)# router eigrp CORP
R1(config-router)# address-family ipv4 unicast autonomous-system 100
R1(config-router-af)#  metric rib-scale 128
R1(config-router-af)#  metric version 64bit
```

### 2.5 建立邻居的条件

| # | 条件 | 说明 |
|:--|:--|:--|
| 1 | **AS 号相同** | 必须完全一致 |
| 2 | **K 值相同** | 默认 K1=1,K2=0,K3=1,K4=0,K5=0 |
| 3 | **在同一子网** | 主地址必须在同一网段 |
| 4 | **认证匹配** | 密钥链和密钥必须一致 |
| 5 | 接口不是 passive | — |

**注意 EIGRP 和 OSPF 的差异**：
- **EIGRP 的 Hello/Hold 时间不需要匹配**（不像 OSPF 必须一致）
- **EIGRP 没有 Router ID 必须唯一的强要求**（但重复会影响外部路由）
- **EIGRP 不检查 MTU**（不会像 OSPF 那样卡在 ExStart）

**默认定时器**：
- 带宽 > T1 (1.544Mbps)：Hello **5 秒**，Hold **15 秒**
- 带宽 ≤ T1：Hello **60 秒**，Hold **180 秒**

---

## ③ 配置命令

### Cisco

```cisco
! ═══ 经典模式（Classic Mode）═══
R1(config)# router eigrp 100                        ! 100 是 AS 号，全网必须一致
R1(config-router)# eigrp router-id 1.1.1.1
R1(config-router)# network 192.168.1.0 0.0.0.255
R1(config-router)# network 10.0.12.1 0.0.0.0        ! 精确匹配单个接口（推荐）
R1(config-router)# no auto-summary                  ! ★ 必须关闭自动汇总！
R1(config-router)# passive-interface default
R1(config-router)# no passive-interface GigabitEthernet0/1

! ═══ 命名模式（Named Mode，IOS 15+ 推荐）═══
R1(config)# router eigrp CORP
R1(config-router)# address-family ipv4 unicast autonomous-system 100
R1(config-router-af)#  eigrp router-id 1.1.1.1
R1(config-router-af)#  network 10.0.0.0 0.255.255.255
R1(config-router-af)#  af-interface default
R1(config-router-af-interface)#   passive-interface
R1(config-router-af-interface)#   exit
R1(config-router-af)#  af-interface GigabitEthernet0/1
R1(config-router-af-interface)#   no passive-interface
R1(config-router-af-interface)#   hello-interval 2
R1(config-router-af-interface)#   hold-time 6
R1(config-router-af-interface)#   summary-address 10.1.0.0 255.255.0.0
R1(config-router-af-interface)#   authentication mode md5
R1(config-router-af-interface)#   authentication key-chain EIGRP-KEYS

! ═══ 路由汇总（★ 防 SIA 的关键）═══
! 经典模式：在接口下配
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip summary-address eigrp 100 10.1.0.0 255.255.0.0

! ═══ Stub 路由（★ 分支必配）═══
R1(config-router)# eigrp stub connected summary
! 选项：connected / summary / static / redistributed / receive-only

! ═══ 不等价负载均衡（EIGRP 独有！）═══
R1(config-router)# variance 2                       ! 允许 FD ≤ 2倍最优FD 的路径参与
R1(config-router)# maximum-paths 4                  ! 最多 4 条

! ═══ 认证 ═══
R1(config)# key chain EIGRP-KEYS
R1(config-keychain)# key 1
R1(config-keychain-key)#  key-string MyEigrpSecret
R1(config-keychain-key)#  cryptographic-algorithm hmac-sha-256

R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip authentication mode eigrp 100 md5
R1(config-if)# ip authentication key-chain eigrp 100 EIGRP-KEYS

! ═══ 带宽控制（EIGRP 默认最多占用接口带宽的 50%）═══
R1(config-if)# ip bandwidth-percent eigrp 100 30

! ═══ 查看命令 ═══
R1# show ip eigrp neighbors                         ! ① 邻居
R1# show ip eigrp neighbors detail
R1# show ip eigrp topology                          ! ② 拓扑表（★ 最重要）
R1# show ip eigrp topology all-links                ! 显示所有路径，包括非 FS
R1# show ip eigrp topology 10.1.1.0/24              ! 某条路由的详情
R1# show ip eigrp topology active                   ! ★ 看有没有卡在 Active
R1# show ip eigrp interfaces
R1# show ip eigrp interfaces detail
R1# show ip route eigrp                             ! ③ 路由表
R1# show ip protocols                               ! 配置概览
R1# show ip eigrp events                            ! 事件历史，排障用

! ═══ 调试 ═══
R1# debug eigrp packets
R1# debug ip eigrp
R1# debug eigrp neighbors
```

### `show ip eigrp topology` 输出解读（★ 核心）

```cisco
R1# show ip eigrp topology

EIGRP-IPv4 Topology Table for AS(100)/ID(1.1.1.1)
Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status, s - sia Status

P 10.1.1.0/24, 1 successors, FD is 3072
        via 10.0.12.2 (3072/2816), GigabitEthernet0/1        ← Successor
        via 10.0.13.3 (5120/2560), GigabitEthernet0/2        ← Feasible Successor
↑           ↑              ↑    ↑
状态      下一跳          FD    RD
```

**读法**：
- **`P`** = Passive = **正常**；**`A`** = Active = **正在查询，有问题**
- **`1 successors`** = 有 1 条最优路径
- **`FD is 3072`** = 当前最优距离
- 括号里 **`(FD/RD)`**：第一个是"我到目标的距离"，第二个是"邻居到目标的距离"

**判断第二条是不是 FS**：
```
邻居的 RD = 2560
当前的 FD = 3072
2560 < 3072  ✅ 满足 FC → 是 Feasible Successor
```

**看所有路径（包括不满足 FC 的）**：
```cisco
R1# show ip eigrp topology all-links
P 10.1.1.0/24, 1 successors, FD is 3072, serno 12
        via 10.0.12.2 (3072/2816), GigabitEthernet0/1
        via 10.0.13.3 (5120/2560), GigabitEthernet0/2
        via 10.0.14.4 (7680/4096), GigabitEthernet0/3       ← 这条不是 FS
                              ↑ RD=4096 > FD=3072，不满足 FC
```

> **排障技巧**：如果某条路由收敛慢或出现 SIA，先看 `show ip eigrp topology` 有没有 FS。**没有 FS 的路由在故障时必然要走查询流程**，这就是慢的原因。

### 三厂商对照

| 目的 | Cisco | H3C | 华为 |
|:--|:--|:--|:--|
| 启动进程 | `router eigrp 100` | 不支持（用 OSPF/IS-IS） | 不支持 |
| 通告网段 | `network 10.0.0.0 0.255.255.255` | — | — |
| 关自动汇总 | `no auto-summary` | — | — |
| 查邻居 | `show ip eigrp neighbors` | — | — |

> **重要**：**EIGRP 是 Cisco 私有协议**（虽然 2013 年以 RFC 7868 形式公开了基本规范，但只是信息性文档，其他厂商基本没有实现）。
>
> **H3C 和华为的设备不支持 EIGRP。** 混合厂商环境必须用 **OSPF 或 IS-IS**。
>
> **这也是选型时的重要考量**：如果你的网络将来可能引入非 Cisco 设备，**一开始就应该选 OSPF**，否则迁移会非常痛苦。

---

## ④ 配套实验：DUAL 手算 + 快速收敛验证

**拓扑**：
```
                    ┌──────┐
              10Mbps│  R2  │100Mbps
            ┌───────┤      ├───────┐
            │       └──────┘       │
       ┌────┴───┐              ┌───┴────────┐
       │   R1   │              │  [网络 X]   │
       │        │              │ 10.1.1.0/24│
       └────┬───┘              └───┬────────┘
            │       ┌──────┐       │
            └───────┤  R3  ├───────┘
             1Gbps  └──────┘ 100Mbps
```

### Step 1：手算（先别看设备）

**假设**（简化计算，用示意值）：
- R2 到 X 的 metric（RD）= 2816
- R3 到 X 的 metric（RD）= 2560
- R1 到 R2 的 metric 增量 = 256
- R1 到 R3 的 metric 增量 = 2560

**计算表**：

| 路径 | RD（邻居报的） | 增量 | FD（我的总距离） | 是 Successor？ | 满足 FC？ |
|:--|:--|:--|:--|:--|:--|
| 经 R2 | 2816 | 256 | **3072** | ? | — |
| 经 R3 | 2560 | 2560 | **5120** | ? | ? |

<details><summary>答案</summary>

**Successor**：FD 最小的 → 经 R2（FD = 3072）

**判断 R3 是否为 FS**：
```
可行性条件：邻居的 RD < 当前的 FD
R3 的 RD = 2560
当前 FD  = 3072
2560 < 3072  ✅ 满足
```
**→ R3 是 Feasible Successor**

**结论**：
- 路由表里装的是 **经 R2 的路径**
- 拓扑表里同时保存了 **经 R3 的备份路径**
- **R2 链路一断，R1 立即用 R3 的路径，不需要查询，几十毫秒完成切换**

**反例思考**：如果 R3 报的 RD 是 4000 呢？
```
4000 < 3072？  ❌ 不满足
→ R3 不是 FS
→ R2 断了以后，R1 必须进入 Active 状态，向所有邻居发 QUERY
→ 收敛慢，且有 SIA 风险
```
</details>

### Step 2：设备验证

```cisco
R1(config)# router eigrp 100
R1(config-router)# eigrp router-id 1.1.1.1
R1(config-router)# network 10.0.0.0 0.255.255.255
R1(config-router)# no auto-summary
```

```cisco
R1# show ip eigrp topology 10.1.1.0/24

EIGRP-IPv4 Topology Entry for AS(100)/ID(1.1.1.1) for 10.1.1.0/24
  State is Passive, Query origin flag is 1, 1 Successor(s), FD is 3072
  Descriptor Blocks:
  10.0.12.2 (GigabitEthernet0/1), from 10.0.12.2, Send flag is 0x0
      Composite metric is (3072/2816), route is Internal
                            ↑     ↑
                           FD    RD
  10.0.13.3 (GigabitEthernet0/2), from 10.0.13.3, Send flag is 0x0
      Composite metric is (5120/2560), route is Internal
                            ↑     ↑
                           FD    RD  ← 2560 < 3072，是 FS ✓
```

### Step 3：验证快速收敛（有 FS）

```cisco
! 在 PC 上持续 ping 10.1.1.10
! 断开 R1-R2 链路
R1(config)# interface GigabitEthernet0/1
R1(config-if)# shutdown
```

**观察**：
```cisco
R1# show ip eigrp topology 10.1.1.0/24
  State is Passive, 1 Successor(s), FD is 5120
                    ↑ 状态一直是 Passive！没有进入 Active
  10.0.13.3 (GigabitEthernet0/2), ...
      Composite metric is (5120/2560)
      ↑ FS 直接被提升为 Successor
```

**ping 丢包：0–1 个包。**

**日志**（注意没有 Active/Query 的记录）：
```
%DUAL-5-NBRCHANGE: EIGRP-IPv4 100: Neighbor 10.0.12.2 (Gi0/1) is down: interface down
```

### Step 4：制造"没有 FS"的场景，观察查询过程

```cisco
! 手工把 R3 的 metric 调高，让它不满足 FC
R3(config)# interface GigabitEthernet0/2
R3(config-if)# delay 10000                     ! 大幅增加延迟
```

```cisco
R1# show ip eigrp topology 10.1.1.0/24
  State is Passive, 1 Successor(s), FD is 3072
  10.0.12.2 (GigabitEthernet0/1), ...
      Composite metric is (3072/2816)
  ! 注意：R3 的路径不再显示（因为不满足 FC，不是 FS）

R1# show ip eigrp topology all-links
  ! 现在能看到 R3 的路径了，但 RD > FD
  10.0.13.3 (GigabitEthernet0/2), ...
      Composite metric is (10240/9984)
                                  ↑ RD=9984 > FD=3072，不满足 FC
```

**再断 R1-R2**：
```cisco
R1(config)# interface GigabitEthernet0/1
R1(config-if)# shutdown
```

**立即查看**：
```cisco
R1# show ip eigrp topology active
IP-EIGRP Topology Table for AS(100)/ID(1.1.1.1)

A 10.1.1.0/24, 0 successors, FD is Inaccessible
  1 replies, active 00:00:03, query-origin: Local origin
        via 10.0.13.3 (Infinity/Infinity), r, GigabitEthernet0/2
↑                                          ↑
Active 状态！                        r = 等待 reply
```

**这次的 ping 丢包明显更多**，因为要等查询-回复的往返。

### Step 5：验证 Stub 防 SIA

```cisco
! 把 R3 配成 Stub（模拟分支路由器）
R3(config)# router eigrp 100
R3(config-router)# eigrp stub connected summary
```

**验证**：
```cisco
R1# show ip eigrp neighbors detail
H   Address      Interface   Hold  Uptime   SRTT   RTO  Q  Seq
0   10.0.13.3    Gi0/2         12  00:15:23   10   100  0  18
    Version 23.0/2.0, Retrans: 0, Retries: 0, Prefixes: 5
    Topology-ids from peer - 0
    Stub Peer Advertising ( CONNECTED SUMMARY ) Routes
    ↑ 确认 R3 是 Stub
    Suppressing queries
    ↑ ★ R1 不会向 R3 发送查询
```

**这就是 Stub 防 SIA 的机制**：上游路由器**根本不会向 Stub 邻居发送 QUERY**，从而缩小了查询扩散的范围。

### Step 6：不等价负载均衡（EIGRP 独有）

**OSPF 只能做等价负载均衡（ECMP），EIGRP 可以做不等价！**

```cisco
R1(config)# router eigrp 100
R1(config-router)# variance 2
```

**规则**：允许 `FD ≤ variance × 最优FD` 且**满足 FC** 的路径同时装入路由表。

```
最优 FD = 3072
variance = 2
→ 允许 FD ≤ 6144 的 FS 路径

经 R3 的 FD = 5120 ≤ 6144  ✅ 且是 FS
→ 两条路径都进路由表
```

**验证**：
```cisco
R1# show ip route 10.1.1.0
Routing entry for 10.1.1.0/24
  Known via "eigrp 100", distance 90, metric 3072
  Routing Descriptor Blocks:
  * 10.0.12.2, from 10.0.12.2, via GigabitEthernet0/1
      Route metric is 3072, traffic share count is 5      ← 分担比例
    10.0.13.3, from 10.0.13.3, via GigabitEthernet0/2
      Route metric is 5120, traffic share count is 3      ← 按 metric 反比分配
```

> **`traffic share count` 按 metric 反比分配** —— metric 小的分到更多流量。这比"平均分"更合理，因为它反映了链路的实际能力差异。
>
> ⚠️ **只有满足 FC 的路径才能参与不等价负载均衡**。不满足 FC 的路径即使 FD 在 variance 范围内也不会被使用（因为无法保证无环）。

---

## ⑤ 排障速查

| 症状 | 怀疑点 | 验证命令 |
|:--|:--|:--|
| 邻居建不起来 | AS 号不同 | `show ip protocols \| include Routing Protocol` |
| | K 值不同 | `show ip protocols \| include K1` |
| | 不在同一子网 | `show ip interface brief` |
| | 认证不匹配 | `show ip eigrp interfaces detail`、`debug eigrp packets` |
| | 接口是 passive | `show ip protocols \| section Passive` |
| 邻居反复翻转 | 链路抖动 / CPU 高 / 单向链路 | `show ip eigrp events`、`show processes cpu` |
| 收敛慢 | **没有 FS** | `show ip eigrp topology` 看有几条 via |
| **SIA 故障** | **查询范围太大** | `show ip eigrp topology active`、`show logging \| inc SIA` |
| 学不到某网段 | 对方没 network / 被过滤 | 对方 `show ip protocols` |
| **路由被莫名汇总** | **`auto-summary` 开着** | `show ip protocols \| inc auto-summary` |
| 负载均衡没生效 | variance / 不满足 FC | `show ip route <网段>`、`show ip eigrp topology` |
| EIGRP 占满带宽 | 默认可用 50% | `ip bandwidth-percent eigrp 100 30` |

### `no auto-summary` 的重要性

**EIGRP 在经典模式下默认开启自动汇总**（`auto-summary`），会在**主类边界**自动汇总。

**危害示例**：
```
R1 有 10.1.1.0/24，R2 有 10.2.2.0/24，中间是 192.168.12.0/24

auto-summary 开着时：
R1 向 R2 通告：10.0.0.0/8    ← 汇总到了 A 类主网边界！
R2 向 R1 通告：10.0.0.0/8    ← 同样

结果：双方都收到 10.0.0.0/8，路由环路或黑洞
```

```cisco
R1(config-router)# no auto-summary        ! ★ 必须
```

> **IOS 15.x 之后默认是关闭的**，但**老版本默认开启**。看到 EIGRP 出现莫名其妙的路由问题，第一件事就是检查这个。
>
> ```cisco
> R1# show ip protocols | include summar
>   Automatic Summarization: disabled       ← 应该是 disabled
> ```

---

## ⑥ 考点提示 + 自测题

### 考点

- **FD / RD / Successor / Feasible Successor** 四个概念（★★★）。
- **可行性条件：邻居 RD < 当前 FD**，以及它为什么能保证无环。
- **Passive = 正常，Active = 异常**（反直觉）。
- **SIA 的成因和解法**（汇总、Stub）。
- **EIGRP 支持不等价负载均衡（variance）**，OSPF 不支持。
- **AD：内部 90，外部 170，汇总 5**。
- **`no auto-summary` 必配**。
- **EIGRP 是 Cisco 私有**，混合厂商用 OSPF。

### 自测题

**1.** 什么是可行性条件（FC）？为什么满足 FC 就能保证无环？

<details><summary>答案</summary>

**可行性条件（Feasibility Condition）**：
```
   邻居的 RD  <  当前的 FD
   
   RD (Reported Distance)  = 邻居告诉我的、它到目标的距离
   FD (Feasible Distance)  = 我到目标的最优距离
```

**为什么能保证无环（反证法）**：

假设邻居 R3 的路径**要经过我（R1）** 才能到达目标 X。

那么：
```
R3 到 X 的距离 = R3 到 R1 的距离 + R1 到 X 的距离
              = (某个正数) + FD
              > FD
```

**所以：如果一条路径要绕回我自己，那它的 RD 必然 ≥ 我的 FD。**

反过来：**只要 RD < FD，就数学上保证了这条路径不经过我。**

这就是为什么 EIGRP 敢于把 FS **预先存起来，故障时立即切换、不做任何计算、也不用担心成环**。

**具体例子**：
```
       R1 ──10──> R2 ──20──> [X]
        │                     ↑
        └────30───> R3 ──15───┘

R1 的 FD（经 R2）= 10 + 20 = 30
R3 的 RD = 15

15 < 30  ✅ 满足 FC

含义：R3 有一条只要 15 的路径到 X。如果 R3 是经过 R1 到 X 的，
      它的距离至少是 30 + (R3到R1的距离) > 30，不可能是 15。
      → R3 一定有独立的路径 → 用它做备份是安全的 ✓
```

**FC 的保守性（重要理解）**：

FC 是**充分条件，不是必要条件**。也就是说：
- 满足 FC 的路径**一定**无环 ✓
- 不满足 FC 的路径**不一定**有环，但 EIGRP 宁可不用它

这种保守性是有代价的——**有些实际可用的备份路径被排除在 FS 之外**，导致故障时不得不进入 Active 状态查询。

**这就是"快速收敛"和"绝对无环"之间的权衡**。EIGRP 选择了"宁可慢一次，也不成一次环"。

**实践含义**：设计 EIGRP 网络时，应该**尽量让备份路径满足 FC**（比如通过调整接口 delay 让备份路径的 RD 更小），从而最大化拥有 FS 的路由比例，减少 Active 查询的发生。
</details>

**2.** EIGRP 的路由处于 Active 状态意味着什么？

<details><summary>答案</summary>

**意味着这条路由出问题了，正在向邻居查询新路径。**

**这个术语极其反直觉**——很多人以为 Active 是"活跃可用"，其实正好相反：

| 状态 | 缩写 | 含义 |
|:--|:--|:--|
| **Passive** | **P** | ✅ **正常状态**。路由稳定，有可用的 Successor |
| **Active** | **A** | ❌ **异常状态**。Successor 失效且没有 FS，正在向所有邻居发 QUERY |

**进入 Active 的条件**：
```
Successor 失效
    AND
拓扑表里没有满足 FC 的 Feasible Successor
    ↓
路由进入 Active 状态
    ↓
向所有邻居（除了 Stub 邻居）发送 QUERY
    ↓
等待所有邻居的 REPLY
    ↓
收齐后重新计算 → 回到 Passive
```

**查看**：
```cisco
R1# show ip eigrp topology active
IP-EIGRP Topology Table for AS(100)/ID(1.1.1.1)

A 10.1.1.0/24, 0 successors, FD is Inaccessible
  2 replies, active 00:02:47, query-origin: Local origin
        via 10.0.13.3 (Infinity/Infinity), r, GigabitEthernet0/2
        via 10.0.14.4 (Infinity/Infinity), r, GigabitEthernet0/3
↑                                          ↑
Active                              r = 还在等这个邻居的 reply
```

**`active 00:02:47` 是关键指标**——已经 Active 了 2 分 47 秒。**默认 Active Timer 是 3 分钟**，超时就会触发 SIA。

**如果一直卡在 Active（SIA - Stuck In Active）**：
```
%DUAL-3-SIA: Route 10.1.1.0/24 stuck-in-active state in IP-EIGRP(0) 100. Cleaning up
```
路由器会**强制断开那个没回应的邻居**，可能引发连锁震荡。

**排障时的操作**：
```cisco
! 1. 看有没有路由卡在 Active
R1# show ip eigrp topology active

! 2. 如果有，看是在等谁的 reply（带 r 标记的）
!    然后去那台设备检查：CPU？链路？软件 bug？

! 3. 看历史事件
R1# show ip eigrp events | include SIA|Active

! 4. 看日志
R1# show logging | include DUAL-3-SIA
```

**日常巡检建议**：把 `show ip eigrp topology active` 加进巡检脚本。**正常情况下这条命令应该没有输出**。有输出就说明网络正在经历收敛或有问题。
</details>

**3.** 什么是 SIA？怎么预防？

<details><summary>答案</summary>

**SIA（Stuck In Active）= 路由卡在 Active 状态超过 Active Timer（默认 3 分钟）。**

**成因**：
```
路由器发出 QUERY
    ↓
某个邻居迟迟不回 REPLY
（原因：CPU 100%、内存耗尽、链路单向、软件 bug、
        或者它自己也在等更下游的 REPLY）
    ↓
查询者一直等待
    ↓
超过 3 分钟 → SIA 触发
    ↓
★ 强制断开那个邻居关系 ★
    ↓
邻居关系重建 → 触发新的收敛 → 可能引发连锁震荡
```

**根本原因：查询是"扩散式"的，范围可能非常大。**

```
R1 问 R2："谁能到 X？"
  R2 不知道，问 R3、R4
    R3 不知道，问 R5、R6
      R5 不知道，问 R7...
      
一次查询可能涉及几十台设备。
任何一台响应慢，整条链都卡住。
```

**预防措施（按有效性排序）**：

**① 路由汇总（★ 最有效）**
```cisco
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip summary-address eigrp 100 10.1.0.0 255.255.0.0
```
**原理**：配了汇总的路由器收到关于明细路由的 QUERY 时，会**立即回复 REPLY**（"我有汇总路由覆盖它，不用再往下问了"），**查询到此为止，不会继续扩散**。

这是**从根本上限制查询范围**的方法，也是 EIGRP 大规模部署的必备手段。

**② Stub 路由（分支必配）**
```cisco
Branch(config)# router eigrp 100
Branch(config-router)# eigrp stub connected summary
```
**原理**：上游路由器**根本不会向 Stub 邻居发送 QUERY**（因为知道它是末梢，不可能有别的路径）。

**验证**：
```cisco
Core# show ip eigrp neighbors detail
    Stub Peer Advertising ( CONNECTED SUMMARY ) Routes
    Suppressing queries                       ← 确认不发查询
```

**Stub 的选项**：

| 选项 | 通告什么 |
|:--|:--|
| `connected` | 直连路由 |
| `summary` | 汇总路由 |
| `static` | 静态路由（需配合重分发） |
| `redistributed` | 重分发进来的路由 |
| `receive-only` | **什么都不通告**，只接收 |
| `leak-map` | 有选择地泄露特定路由 |

**分支路由器的典型配置**：`eigrp stub connected summary`

**③ 优化网络设计**
- 避免全 mesh 拓扑（查询会呈爆炸式扩散）
- 采用清晰的分层结构（核心-汇聚-接入）
- 地址规划保证可汇总（这又回到了 [第 1 章](01-企业网络架构与设计.md) 的地址规划原则）

**④ 调整 Active Timer（治标）**
```cisco
R1(config-router)# timers active-time 5          ! 改成 5 分钟
R1(config-router)# timers active-time disabled   ! ⚠️ 永不超时，危险
```
这只是给慢的邻居更多时间，**不解决根本问题**。

**⑤ SIA-Query / SIA-Reply（IOS 12.4+ 自动支持）**
在 Active Timer 到期前的一半时间，路由器会发一个 **SIA-Query** 探测："你还活着吗？还在处理吗？"
- 邻居回 **SIA-Reply** → 说明它还活着只是慢，**延长等待时间，不断邻居**
- 邻居不回 → 确认它有问题，才断邻居

这大幅减少了"误杀"健康邻居的情况。

**实战总结**：
> **EIGRP 网络必须做汇总 + Stub，否则规模一大必出 SIA。**
>
> 这是 EIGRP 相比 OSPF 的一个主要运维负担——OSPF 靠区域（Area）天然限制了 LSA 泛洪范围，而 EIGRP 需要工程师主动设计汇总点。

**规划检查清单**：
- [ ] 所有分支/末梢路由器配了 `eigrp stub`
- [ ] 汇聚层向核心通告汇总路由
- [ ] 地址规划保证可汇总
- [ ] 巡检脚本监控 `show ip eigrp topology active`
</details>

**4.** EIGRP 的 `variance` 是干什么的？它和 OSPF 的负载均衡有什么区别？

<details><summary>答案</summary>

**`variance` 启用不等价负载均衡（Unequal-Cost Load Balancing）—— 这是 EIGRP 相比其他 IGP 的独门武器。**

**规则**：
```
允许 FD ≤ variance × 最优FD  且  满足可行性条件(FC)  的路径
同时装入路由表并参与负载分担
```

**示例**：
```
最优路径 FD = 3072（经 R2）
备份路径 FD = 5120（经 R3，且满足 FC）

variance 1（默认）：只有 FD = 3072 的路径能用 → 单路径
variance 2：允许 FD ≤ 3072 × 2 = 6144 → 5120 符合 → 两条路径都用 ✓
```

**配置**：
```cisco
R1(config)# router eigrp 100
R1(config-router)# variance 2
R1(config-router)# maximum-paths 4
```

**验证**：
```cisco
R1# show ip route 10.1.1.0
Routing entry for 10.1.1.0/24
  Known via "eigrp 100", distance 90, metric 3072
  Routing Descriptor Blocks:
  * 10.0.12.2, from 10.0.12.2, via GigabitEthernet0/1
      Route metric is 3072, traffic share count is 5       ← 分到 5/8 的流量
    10.0.13.3, from 10.0.13.3, via GigabitEthernet0/2
      Route metric is 5120, traffic share count is 3       ← 分到 3/8 的流量
```

**`traffic share count` 按 metric 反比分配** —— metric 越小分到的流量越多。这比平均分更合理，因为它反映了链路的实际能力。

**与 OSPF 的对比**：

| | **EIGRP** | **OSPF** |
|:--|:--|:--|
| 等价负载均衡 | ✅ 支持 | ✅ 支持（ECMP） |
| **不等价负载均衡** | ✅ **支持（variance）** | ❌ **不支持** |
| 分担比例 | **按 metric 反比** | 平均分 |
| 默认最大路径数 | 4 | 4 |

**为什么 OSPF 不支持不等价负载均衡**：

OSPF 是链路状态协议，靠 SPF 算法保证无环。**如果允许流量走非最短路径，SPF 的无环保证就失效了**——因为其他路由器不知道你在做这件事，可能形成环路。

而 EIGRP 有 **FC（可行性条件）** 这个数学保证：**只有满足 FC 的路径才能参与不等价负载均衡**，而满足 FC 的路径在数学上保证不会绕回自己。所以 EIGRP 可以安全地这么做。

**这是 DUAL 算法带来的独特优势。**

**⚠️ 两个重要限制**：

**① 只有 FS 才能参与**
```cisco
R1# show ip eigrp topology all-links
P 10.1.1.0/24, 1 successors, FD is 3072
        via 10.0.12.2 (3072/2816), Gi0/1      ← Successor
        via 10.0.13.3 (5120/2560), Gi0/2      ← FS，可以参与 ✓
        via 10.0.14.4 (5500/4000), Gi0/3      ← 不是 FS（RD 4000 > FD 3072）
                                                 即使 FD 5500 ≤ 6144 也不能参与 ✗
```

**② variance 设太大的风险**
```cisco
R1(config-router)# variance 10        ! ⚠️ 危险
```
可能把**质量差得多的路径**也拉进来（比如把 4G 备份链路和千兆专线一起用），导致：
- 部分流量走了极慢的路径，用户体验反而下降
- 延迟差异大导致 TCP 乱序，性能下降

**建议 variance 不超过 2–3**，且部署前评估各路径的实际质量差异。

**实战应用场景**：
- 主备链路带宽相近（比如 1G 主 + 500M 备），想让备份链路也承载部分流量
- 多条不同运营商的链路，想同时利用
- 数据中心内多条不完全等价的路径
</details>

**5.** EIGRP 和 OSPF 在什么场景下应该选哪个？

<details><summary>答案</summary>

**决策矩阵**：

| 考量因素 | 选 **EIGRP** | 选 **OSPF** |
|:--|:--|:--|
| **厂商环境** | 纯 Cisco | ★ **混合厂商**（H3C/华为/Juniper 都只支持 OSPF） |
| **配置复杂度** | ★ 简单（一条 network 就能跑） | 需要规划区域、ABR/ASBR |
| **收敛速度** | ★ **有 FS 时极快**（几十毫秒） | 需要重跑 SPF（但现代设备也很快） |
| **不等价负载均衡** | ★ **支持（variance）** | ❌ 不支持 |
| **大规模网络** | 需要精心设计汇总和 Stub 防 SIA | ★ **区域天然限制泛洪范围** |
| **拓扑要求** | 灵活，任意拓扑 | ★ 需要 Area 0 骨干，非骨干区域必须连骨干 |
| **CPU/内存** | ★ 较低 | 较高（要存 LSDB、跑 SPF） |
| **人才储备** | 只有 Cisco 背景的人熟悉 | ★ **通用技能，招人容易** |
| **未来演进** | Cisco 在推 SD-Access（底层用 IS-IS） | ★ 生态更广，标准化 |
| **认证考试** | ENCOR/ENARSI 都考 | ★ 权重更高 |

**实践建议**：

**选 OSPF 的情况（大多数）**：
1. **网络里有或将来可能有非 Cisco 设备** —— 这是决定性因素
2. 需要和运营商、合作伙伴对接
3. 团队技能通用性优先
4. 大规模网络，需要清晰的区域划分

**选 EIGRP 的情况**：
1. 100% 纯 Cisco 环境，且**确定长期不会变**
2. 有大量不等价的冗余链路，需要 variance
3. 网络规模中等，团队熟悉 EIGRP
4. 追求极致的收敛速度（配合 FS 设计）

> **我的建议：新建网络优先 OSPF。**
>
> 理由不是技术优劣（EIGRP 在很多方面确实更优雅），而是**厂商锁定风险**。设备采购的决定权往往不在网络工程师手上——三年后老板决定采购华为设备降本，你的 EIGRP 网络就要推倒重来。而 OSPF 从第一天就是安全的。
>
> 这不是技术判断，是工程判断。**技术选型要考虑的不只是"现在哪个好"，还有"三年后我会不会后悔"。**

**混合部署的现实**：

很多大企业是**两者并存**的：
```
   核心/骨干：OSPF 或 IS-IS（对接运营商、多厂商）
        ↕ 路由重分发
   分支/接入：EIGRP（历史遗留，纯 Cisco）
```

这时候 **路由重分发** 就成了必修课——而重分发是路由环路的高发区，详见 [ENARSI 第 4 章](../03-ENARSI-300-410/04-路由重分发与路由策略.md)。

**AD 值的影响（容易被忽略的坑）**：
```
EIGRP 内部：90    ← 比 OSPF 优先
OSPF：      110
EIGRP 外部：170   ← 比 OSPF 差
```
在同时跑两个协议的路由器上，**EIGRP 内部路由会压倒 OSPF 路由**。如果规划不当，会导致流量走了非预期的路径。重分发时尤其要注意。
</details>

---

**上一章** ← [05 一二层排障与高可用](05-一层二层排障与高可用-HSRP-VRRP.md) ｜ **下一章** → [07 OSPF 进阶](07-OSPF进阶.md)
