# 07 · OSPF 进阶

> ⚠️ **全书权重最高的一章。** ENCOR 和 ENARSI 里 OSPF 的分量最重，而 **LSA 类型表**是所有 OSPF 题目的公共前提。
> 通过标准：**能默写 1/2/3/4/5/7 类 LSA 的产生者、传播范围、作用。**

## ① 这章解决什么问题

[Stage 1 第 4 章](../01-CCNA补齐篇/04-OSPF单区域.md) 里，所有路由器都在 Area 0，大家共享一份完全相同的 LSDB。

这在 20 台设备时没问题。但当网络增长到 200 台：

1. **LSDB 巨大**，每台设备都要存全网拓扑，内存吃紧
2. **任何一条链路抖动，全网所有设备都要重跑 SPF**，CPU 飙升
3. **路由表巨大**，转发性能下降

**多区域（Multi-Area）就是解法**：把网络切成若干区域，**区域内部的拓扑细节不泄露到区域外**。区域 A 里的链路抖动，区域 B 完全不知道，也不需要重算。

而要理解多区域，就必须理解 **LSA 类型**——因为不同类型的 LSA 有不同的传播范围，这正是"隔离"得以实现的机制。

---

## ② 区域与路由器角色

### 2.1 区域的规则

```
                    ┌──────────────────────────┐
                    │      Area 0 (骨干)        │
                    │   ┌────┐      ┌────┐     │
                    │   │ R1 │──────│ R2 │     │
                    │   └──┬─┘      └─┬──┘     │
                    └──────┼──────────┼────────┘
                       ABR │          │ ABR
              ┌────────────┼──┐   ┌───┼──────────────┐
              │  Area 1    │  │   │   │   Area 2     │
              │        ┌───▼┐ │   │ ┌─▼──┐           │
              │        │ R3 │ │   │ │ R4 │           │
              │        └────┘ │   │ └──┬─┘           │
              └───────────────┘   └────┼─────────────┘
                                  ASBR │
                                  ┌────▼────┐
                                  │  外部网络 │ (BGP/静态/RIP)
                                  └─────────┘
```

**三条硬性规则**：

1. **必须有 Area 0（骨干区域）**
2. **所有非骨干区域必须直接连接到 Area 0**（否则需要虚链路 Virtual-Link）
3. **区域间的流量必须经过 Area 0**（不允许 Area1 → Area2 直接走）

> **为什么规则 3 存在**：OSPF 用"区域间路由必须经过骨干"来**防止区域间路由环路**。因为 ABR 之间不会互相传递 Type-3 LSA（除非通过 Area 0），天然形成了星型结构。

### 2.2 路由器角色

| 角色 | 全称 | 定义 | 产生什么 LSA |
|:--|:--|:--|:--|
| **IR** | Internal Router | 所有接口都在**同一个区域** | Type-1 |
| **BR** | Backbone Router | 至少一个接口在 **Area 0** | Type-1 |
| **ABR** | **Area Border Router** | 接口**跨越多个区域**（至少一个在 Area 0） | **Type-3, Type-4** |
| **ASBR** | **Autonomous System Boundary Router** | 把**外部路由**引入 OSPF（重分发） | **Type-5 / Type-7** |
| **DR/BDR** | Designated Router | 广播网络上的指定路由器 | **Type-2** |

> **一台路由器可以同时是多个角色**。比如一台既连着 Area 0 和 Area 1（ABR），又把静态路由重分发进来（ASBR）。

---

## ③ LSA 类型（★★★ 全章核心）

### 3.1 LSA 类型总表（必须能默写）

| Type | 名称 | **谁产生** | **传播范围** | **作用** | 路由表标记 |
|:--|:--|:--|:--|:--|:--|
| **1** | **Router LSA** | **每台 OSPF 路由器** | **本区域内** | 描述本路由器的接口、链路状态、cost | `O` |
| **2** | **Network LSA** | **DR** | **本区域内** | 描述多路访问网络上有哪些路由器 | `O` |
| **3** | **Summary LSA**<br>(Network Summary) | **ABR** | **跨区域**（除 Totally Stub） | 把一个区域的**网段**通告给其他区域 | **`O IA`** |
| **4** | **ASBR Summary LSA** | **ABR** | 跨区域 | 告诉其他区域**怎么到达 ASBR** | （无独立标记） |
| **5** | **AS External LSA** | **ASBR** | **泛洪到整个 AS**（Stub/NSSA 除外） | 外部路由（重分发进来的） | **`O E1` / `O E2`** |
| **6** | Group Membership | — | — | 组播 OSPF（MOSPF），**已废弃** | — |
| **7** | **NSSA External LSA** | **NSSA 区域内的 ASBR** | **仅在 NSSA 区域内** | NSSA 里的外部路由，到 ABR 后**转成 Type-5** | **`O N1` / `O N2`** |
| 8 | Link LSA | — | — | **仅 OSPFv3**（IPv6 链路本地地址） | — |
| 9-11 | Opaque LSA | — | 链路/区域/AS | 扩展用（**MPLS-TE 用 Type-10**） | — |

### 3.2 逐个理解

#### Type-1 (Router LSA) —— "我是谁，我连着什么"

**每台 OSPF 路由器都会产生**，描述自己的所有接口和链路。

```cisco
R1# show ip ospf database router

            OSPF Router with ID (1.1.1.1) (Process ID 1)

                Router Link States (Area 0)

  LS age: 245
  Options: (No TOS-capability, DC)
  LS Type: Router Links
  Link State ID: 1.1.1.1
  Advertising Router: 1.1.1.1
  Number of Links: 3

    Link connected to: another Router (point-to-point)
     (Link ID) Neighboring Router ID: 2.2.2.2
     (Link Data) Router Interface address: 10.0.12.1
      Number of TOS metrics: 0
       TOS 0 Metrics: 100

    Link connected to: a Stub Network
     (Link ID) Network/subnet number: 192.168.1.0
     (Link Data) Network Mask: 255.255.255.0
       TOS 0 Metrics: 100
```

**四种链路类型**（考点）：

| Type | 描述 | Link ID | Link Data |
|:--|:--|:--|:--|
| 1 | Point-to-Point | 邻居的 Router ID | 本接口 IP |
| 2 | Transit Network（多路访问） | **DR 的接口 IP** | 本接口 IP |
| 3 | **Stub Network** | 网络地址 | 子网掩码 |
| 4 | Virtual Link | 邻居 Router ID | 本接口 IP |

**Type-1 LSA 只在本区域内泛洪，绝不跨区域。** 这就是"区域内拓扑细节不外泄"的机制。

#### Type-2 (Network LSA) —— "这个广播网上有谁"

**只有 DR 产生**，描述多路访问网络（以太网）上连接了哪些路由器。

```cisco
R1# show ip ospf database network

                Net Link States (Area 0)

  LS Type: Network Links
  Link State ID: 10.0.12.1 (address of Designated Router)
  Advertising Router: 1.1.1.1
  Network Mask: /24
        Attached Router: 1.1.1.1
        Attached Router: 2.2.2.2
        Attached Router: 3.3.3.3
```

**关键推论**：
- **点对点链路（P2P）没有 DR，所以不产生 Type-2 LSA**
- 把以太网接口改成 `ip ospf network point-to-point`，可以**减少一个 Type-2 LSA**，让 LSDB 更小

#### Type-3 (Summary LSA) —— "隔壁区域有这些网段"

**由 ABR 产生**，把一个区域的**网段信息**（不是拓扑细节！）通告给其他区域。

```
   Area 1 里有：
     R3-R4 之间的链路（拓扑细节）
     R4-R5 之间的链路（拓扑细节）
     192.168.10.0/24
     192.168.20.0/24
        ↓
   ABR 只告诉 Area 0：
     "有 192.168.10.0/24，cost 100"
     "有 192.168.20.0/24，cost 150"
     ★ 拓扑细节完全不传 ★
```

**这就是区域隔离的核心价值**：Area 0 的路由器**不知道 Area 1 内部长什么样**，所以 Area 1 内部的链路抖动**不会触发 Area 0 的 SPF 重算**。

**路由表标记：`O IA`**（Inter-Area，区域间）

```cisco
R1# show ip route ospf
O IA  192.168.10.0/24 [110/200] via 10.0.12.2, 00:05:12, GigabitEthernet0/1
  ↑↑
 区域间路由
```

> **注意**：Type-3 LSA 传的是"网段 + cost"，**不是拓扑**。所以区域间路由是**距离矢量式**的（相信 ABR 说的），而区域内是链路状态式的。**OSPF 在区域间实际上退化成了距离矢量协议。**
>
> 这也是为什么"所有区域必须连 Area 0"——用星型拓扑防止区域间环路，就像 RIP 用跳数限制防环一样。

#### Type-4 (ASBR Summary LSA) —— "怎么到达 ASBR"

**问题场景**：
```
   Area 1                Area 0              Area 2
   [ASBR] ──── [ABR-1] ──── [ABR-2] ──── [R5]
      │
   外部路由 (Type-5 LSA 会泛洪到全 AS)
```

R5 收到了 Type-5 LSA，知道"有一条外部路由 `200.1.1.0/24`，是 ASBR (Router ID 3.3.3.3) 通告的"。

**但 R5 不知道怎么到达 3.3.3.3！** 因为 ASBR 在 Area 1，它的 Type-1 LSA 不会传到 Area 2。

**Type-4 LSA 就是解决这个的**：ABR 告诉其他区域"我知道怎么到 ASBR 3.3.3.3，cost 是 X"。

```cisco
R5# show ip ospf database asbr-summary

                Summary ASB Link States (Area 2)

  LS Type: Summary Links (AS Boundary Router)
  Link State ID: 3.3.3.3 (AS Boundary Router address)
  Advertising Router: 2.2.2.2                    ← ABR 通告的
  TOS: 0  Metric: 200
```

**记忆**：
- **Type-5 说"有哪些外部路由"**
- **Type-4 说"怎么到达通告这些外部路由的那台 ASBR"**
- **两者必须配合，缺一不可** —— 只有 Type-5 没有 Type-4，路由是不可达的

> **例外**：如果 ASBR 就在本区域内（本区域的路由器能通过 Type-1 LSA 知道怎么到它），就**不需要 Type-4**。

#### Type-5 (AS External LSA) —— 外部路由

**由 ASBR 产生**，把重分发进来的外部路由（静态、BGP、RIP、EIGRP、直连）通告给整个 OSPF 域。

**泛洪范围：整个 AS**（不受区域限制！），除了 Stub 和 NSSA 区域。

**两种度量类型（★ 高频考点）**：

| 类型 | 标记 | Metric 计算 | 特点 |
|:--|:--|:--|:--|
| **E2**（**默认**） | `O E2` | **只算外部 cost，不加内部 cost** | 无论走多远，metric 不变 |
| **E1** | `O E1` | **外部 cost + 到 ASBR 的内部 cost** | 反映真实路径开销 |

```cisco
! 重分发时指定类型
R1(config-router)# redistribute static subnets metric-type 1     ! E1
R1(config-router)# redistribute static subnets metric-type 2     ! E2（默认）
```

**什么时候用哪个**：

```
        ┌─────────────────────────────┐
        │  外部网络 200.1.1.0/24       │
        └───┬─────────────────────┬───┘
       cost 20                cost 20
        ┌───▼───┐             ┌───▼───┐
        │ ASBR1 │             │ ASBR2 │
        └───┬───┘             └───┬───┘
       内部 cost 10          内部 cost 100
            │                     │
            └──────────┬──────────┘
                    [ R5 ]
```

**用 E2（默认）**：R5 看到两条路径的 metric 都是 20（只算外部 cost）→ **等价，做负载均衡**
→ 一半流量走了内部 cost 100 的远路 ❌

**用 E1**：R5 看到 ASBR1 路径 = 20+10 = 30，ASBR2 路径 = 20+100 = 120 → **选 ASBR1** ✓

> **实践建议**：**有多个 ASBR 时用 E1**（能正确选择最近的出口）。只有一个 ASBR 时 E2 更简单（metric 稳定，不会因内部拓扑变化而波动）。
>
> **考点**：E1 和 E2 比较时，**E1 永远优于 E2**（无论 metric 大小）。

#### Type-7 (NSSA External LSA) —— 特殊的外部路由

在 **NSSA（Not-So-Stubby Area）** 里，不允许 Type-5 LSA 进入，但如果这个区域内部有 ASBR（需要引入外部路由），怎么办？

**用 Type-7**：
```
   NSSA 区域内的 ASBR 产生 Type-7 LSA
        ↓ （只在 NSSA 区域内泛洪）
   到达 ABR
        ↓
   ★ ABR 把 Type-7 转换成 Type-5 ★
        ↓
   Type-5 泛洪到整个 OSPF 域
```

**路由表标记**：`O N1` / `O N2`（对应 E1/E2，默认 **N2**）

---

## ④ 特殊区域类型（★ 高频考点）

**设计目的：进一步减少 LSA 数量和路由表规模。**

### 4.1 四种特殊区域对照表（必背）

| 区域类型 | Type-1,2 | **Type-3** | **Type-4,5** | **Type-7** | 默认路由 | 配置命令 |
|:--|:--|:--|:--|:--|:--|:--|
| **标准区域** | ✅ | ✅ | ✅ | ❌ | — | 默认 |
| **Stub** | ✅ | ✅ | ❌ **拒绝** | ❌ | ✅ **ABR 自动注入 Type-3 默认路由** | `area X stub` |
| **Totally Stub** | ✅ | ❌ **拒绝** | ❌ **拒绝** | ❌ | ✅ 只有默认路由 | `area X stub no-summary`（**仅 ABR**） |
| **NSSA** | ✅ | ✅ | ❌ **拒绝** | ✅ **允许** | ⚠️ **不自动注入**，需 `default-information-originate` | `area X nssa` |
| **Totally NSSA** | ✅ | ❌ **拒绝** | ❌ **拒绝** | ✅ **允许** | ✅ 自动注入 | `area X nssa no-summary`（**仅 ABR**） |

**记忆方法**：

```
Stub          = 拒绝【外部路由】(Type-4,5)
Totally Stub  = 拒绝【外部路由 + 区域间路由】(Type-3,4,5)
NSSA          = Stub + 【允许自己产生外部路由】(Type-7)
Totally NSSA  = Totally Stub + 【允许自己产生外部路由】(Type-7)

带 "Totally" 的 = 额外拒绝 Type-3
带 "NSSA" 的   = 额外允许 Type-7
```

### 4.2 配置要点

```cisco
! ── Stub 区域：区域内【所有】路由器都要配 ──
R-ABR(config)# router ospf 1
R-ABR(config-router)# area 1 stub
R-Internal(config-router)# area 1 stub          ! 内部路由器也要配

! ── Totally Stub：no-summary 只在 ABR 上配 ──
R-ABR(config-router)# area 1 stub no-summary    ! ★ 仅 ABR
R-Internal(config-router)# area 1 stub          ! 内部路由器还是 stub

! ── NSSA ──
R-ABR(config-router)# area 1 nssa
R-Internal(config-router)# area 1 nssa

! NSSA 默认不注入默认路由，需要手工
R-ABR(config-router)# area 1 nssa default-information-originate

! ── Totally NSSA ──
R-ABR(config-router)# area 1 nssa no-summary    ! ★ 仅 ABR
R-Internal(config-router)# area 1 nssa
```

> ⚠️ **两个必须记住的规则**：
> 1. **区域内所有路由器的 Stub/NSSA 标志必须一致**，否则**邻居建立不起来**（这是建立邻居的 5 个条件之一）。
> 2. **`no-summary` 只在 ABR 上配**，内部路由器不需要（也不能）配。

### 4.3 验证

```cisco
R1# show ip ospf | include Area|Stub|NSSA
    Area 1
        Number of interfaces in this area is 2
        It is a stub area                     ← 确认区域类型
        generates stub default route with cost 1

R1# show ip ospf database summary             ! Type-3
R1# show ip ospf database external            ! Type-5
R1# show ip ospf database nssa-external       ! Type-7

! 内部路由器上验证
R-Internal# show ip route ospf
O IA  0.0.0.0/0 [110/2] via 10.0.13.1, 00:02:15, GigabitEthernet0/1
   ↑ Totally Stub 区域里只有一条默认路由
```

---

## ⑤ 路由汇总

**OSPF 有两种汇总，位置和命令都不同**：

| 汇总类型 | 在哪配 | 汇总什么 | 命令 |
|:--|:--|:--|:--|
| **区域间汇总** | **ABR** | **Type-3 LSA**（区域内路由） | `area X range <网络> <掩码>` |
| **外部路由汇总** | **ASBR** | **Type-5 LSA**（外部路由） | `summary-address <网络> <掩码>` |

```cisco
! ── ABR 上汇总 Area 1 的路由 ──
R-ABR(config)# router ospf 1
R-ABR(config-router)# area 1 range 192.168.0.0 255.255.252.0
R-ABR(config-router)# area 1 range 192.168.4.0 255.255.252.0 cost 100   ! 手工指定 cost
R-ABR(config-router)# area 1 range 10.99.0.0 255.255.0.0 not-advertise  ! 抑制不通告

! ── ASBR 上汇总外部路由 ──
R-ASBR(config)# router ospf 1
R-ASBR(config-router)# summary-address 200.1.0.0 255.255.0.0
```

**汇总的价值**：
1. **缩小路由表**
2. **★ 隔离故障** —— 明细路由抖动时，汇总路由保持稳定，**不会触发其他区域的 LSA 泛洪和 SPF 重算**

> **第 2 点是汇总最重要的价值**，比缩小路由表重要得多。一条链路每天抖 100 次，如果没有汇总，全网每天要重算 100 次 SPF。

**汇总的 cost 计算**：
- **默认**：取被汇总的明细路由中**最小的 cost**
- 可以用 `cost` 关键字手工指定

**⚠️ 汇总要配 Null0 防环**：
```cisco
! Cisco 会自动为 area range 生成一条指向 Null0 的丢弃路由
R-ABR# show ip route | include Null0
O    192.168.0.0/22 is a summary, 00:05:23, Null0
```
**作用**：落在汇总范围内但没有明细路由的流量，直接丢弃，**不会在设备间来回弹形成环路**。

---

## ⑥ 虚链路（Virtual Link）

**解决两个问题**：

### 场景 1：区域没有直连 Area 0

```
   Area 0 ──── [ABR-1] ──── Area 1 ──── [ABR-2] ──── Area 2
                                                        ↑
                                          Area 2 没有直连 Area 0！违反规则
```

**解法**：在 ABR-1 和 ABR-2 之间**穿过 Area 1** 建立虚链路，逻辑上把 Area 2 连到 Area 0。

```cisco
! ABR-1（Router ID 1.1.1.1）
R-ABR1(config-router)# area 1 virtual-link 2.2.2.2       ! 对端的 Router ID

! ABR-2（Router ID 2.2.2.2）
R-ABR2(config-router)# area 1 virtual-link 1.1.1.1
```

**注意**：`area 1` 是**穿越区域（Transit Area）**，不是 Area 0 也不是 Area 2。

### 场景 2：Area 0 被分割

```
   Area 0 (左半) ──── Area 1 ──── Area 0 (右半)
                        ↑
             骨干被切断了！需要虚链路修复
```

**虚链路的限制（考点）**：
- **穿越区域不能是 Stub / NSSA / Totally Stub**
- 穿越区域必须有完整的路由信息（不能有汇总或过滤阻断）
- 两端必须都是 ABR
- **虚链路上不能配认证**（除非穿越区域配了区域认证）

```cisco
R1# show ip ospf virtual-links
Virtual Link OSPF_VL0 to router 2.2.2.2 is up
  Run as demand circuit
  DoNotAge LSA allowed.
  Transit area 1, via interface GigabitEthernet0/1
  Topology-MTID    Cost    Disabled     Shutdown      Topology Name
        0           100      no            no            Base
```

> **虚链路是"临时补救"，不是"设计方案"。** 如果你的网络需要虚链路，说明**区域规划有问题**，应该考虑重新设计。虚链路增加了复杂度和排障难度，而且是许多诡异故障的来源。

---

## ⑦ OSPF 认证与安全

```cisco
! ── 接口级 MD5 认证（传统）──
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip ospf message-digest-key 1 md5 MySecretKey
R1(config)# router ospf 1
R1(config-router)# area 0 authentication message-digest

! ── 更现代的 SHA 认证（IOS 15+，推荐）──
R1(config)# key chain OSPF-KEYS
R1(config-keychain)# key 1
R1(config-keychain-key)#  key-string MyVerySecretKey
R1(config-keychain-key)#  cryptographic-algorithm hmac-sha-256

R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip ospf authentication key-chain OSPF-KEYS

! ── passive-interface（最重要的安全措施）──
R1(config-router)# passive-interface default
R1(config-router)# no passive-interface GigabitEthernet0/1

! ── 限制 LSA 数量（防 LSA 泛洪攻击/故障）──
R1(config-router)# max-lsa 10000

! ── 验证 ──
R1# show ip ospf interface Gi0/1 | include auth
  Cryptographic authentication enabled
    Sending SA: Simple, 1, algorithm HMAC-SHA-256, key-chain "OSPF-KEYS"
```

---

## ⑧ OSPF 优化与快速收敛

```cisco
! ── ① 参考带宽（★ 必改，全网统一）──
R1(config-router)# auto-cost reference-bandwidth 100000    ! 100 Gbps

! ── ② SPF 与 LSA 节流（指数退避，防震荡）──
R1(config-router)# timers throttle spf 10 100 5000
!                                    ↑   ↑    ↑
!                              首次延迟 增量 最大值(毫秒)
R1(config-router)# timers throttle lsa 10 100 5000
R1(config-router)# timers lsa arrival 80

! ── ③ BFD（★ 亚秒级故障检测）──
R1(config)# interface GigabitEthernet0/1
R1(config-if)# bfd interval 50 min_rx 50 multiplier 3
R1(config)# router ospf 1
R1(config-router)# bfd all-interfaces

! ── ④ 点对点网络类型（省掉 DR 选举和 Type-2 LSA）──
R1(config-if)# ip ospf network point-to-point

! ── ⑤ 调快 Hello/Dead（不如 BFD 高效，但简单）──
R1(config-if)# ip ospf hello-interval 1
R1(config-if)# ip ospf dead-interval 4

! ── ⑥ 增量 SPF / LSA 组步调 ──
R1(config-router)# ispf
R1(config-router)# timers pacing lsa-group 120

! ── ⑦ 默认路由注入 ──
R1(config-router)# default-information originate
R1(config-router)# default-information originate always metric 10 metric-type 1
```

**BFD vs 调快 Hello**：

| | **调快 OSPF Hello (1/4)** | **BFD (50ms×3)** |
|:--|:--|:--|
| 检测时间 | 4 秒 | **150 毫秒** |
| CPU 开销 | 高（OSPF 进程处理每个 Hello） | **低**（BFD 在硬件/专用进程处理） |
| 扩展性 | 差（邻居多时 CPU 压力大） | **好** |
| 支持多协议 | 只对 OSPF | **OSPF/BGP/EIGRP/静态路由都能用** |

> **推荐：用 BFD，保持 OSPF Hello 为默认值。** BFD 是专门为快速故障检测设计的轻量协议，效率远高于调快路由协议自身的定时器。

---

## ⑨ 配套实验：多区域 + 特殊区域 + 汇总

**拓扑**：
```
   Area 1                Area 0                 Area 2 (NSSA)
                                                       
  [R3]───────[R1/ABR]───────[R2/ABR]───────[R4]───[R5/ASBR]
192.168.10.0/24    │              │                    │
192.168.11.0/24    │              │              [外部静态路由]
                   └── Area 0 ────┘               200.1.1.0/24
```

### Step 1：基础多区域

```cisco
! ── R1 (ABR: Area 0 + Area 1) ──
R1(config)# router ospf 1
R1(config-router)# router-id 1.1.1.1
R1(config-router)# auto-cost reference-bandwidth 100000
R1(config-router)# network 10.0.12.1 0.0.0.0 area 0
R1(config-router)# network 10.0.13.1 0.0.0.0 area 1

! ── R3 (Area 1 内部路由器) ──
R3(config)# router ospf 1
R3(config-router)# router-id 3.3.3.3
R3(config-router)# auto-cost reference-bandwidth 100000
R3(config-router)# network 10.0.13.3 0.0.0.0 area 1
R3(config-router)# network 192.168.10.0 0.0.0.255 area 1
R3(config-router)# network 192.168.11.0 0.0.0.255 area 1

! ── R2 (ABR: Area 0 + Area 2) ──
R2(config)# router ospf 1
R2(config-router)# router-id 2.2.2.2
R2(config-router)# network 10.0.12.2 0.0.0.0 area 0
R2(config-router)# network 10.0.24.2 0.0.0.0 area 2
```

### Step 2：观察 LSA 类型（★ 核心实验）

**在 R3（Area 1 内部）上看**：
```cisco
R3# show ip ospf database

            OSPF Router with ID (3.3.3.3) (Process ID 1)

                Router Link States (Area 1)              ← Type-1
Link ID         ADV Router      Age   Seq#       Checksum Link count
1.1.1.1         1.1.1.1         245   0x80000005 0x00A1B2  2
3.3.3.3         3.3.3.3         238   0x80000004 0x00C3D4  4

                Summary Net Link States (Area 1)         ← Type-3
Link ID         ADV Router      Age   Seq#       Checksum
10.0.12.0       1.1.1.1         240   0x80000002 0x001234
10.0.24.0       1.1.1.1         235   0x80000002 0x005678
192.168.20.0    1.1.1.1         230   0x80000002 0x009ABC
```

**关键观察**：
- R3 **看不到 Area 0 和 Area 2 的 Type-1 LSA** ← 区域隔离生效 ✓
- R3 只通过 **Type-3 LSA** 知道其他区域有哪些网段 ✓

**在 R1（ABR）上看**：
```cisco
R1# show ip ospf database

                Router Link States (Area 0)              ← Area 0 的 Type-1
                Net Link States (Area 0)                 ← Area 0 的 Type-2
                Summary Net Link States (Area 0)         ← Type-3
                
                Router Link States (Area 1)              ← Area 1 的 Type-1
                Summary Net Link States (Area 1)         ← Type-3
```

**ABR 同时维护两个区域的 LSDB** ✓

### Step 3：引入外部路由，观察 Type-5 和 Type-4

```cisco
! R5 作为 ASBR，重分发静态路由
R5(config)# ip route 200.1.1.0 255.255.255.0 Null0
R5(config)# router ospf 1
R5(config-router)# redistribute static subnets
```

**在 R3 上看**：
```cisco
R3# show ip ospf database external

                Type-5 AS External Link States

  LS Type: AS External Link
  Link State ID: 200.1.1.0 (External Network Number)
  Advertising Router: 5.5.5.5                      ← ASBR
  Network Mask: /24
        Metric Type: 2 (Larger than any link state path)
        Metric: 20
```

```cisco
R3# show ip ospf database asbr-summary

                Summary ASB Link States (Area 1)         ← Type-4

  LS Type: Summary Links (AS Boundary Router)
  Link State ID: 5.5.5.5 (AS Boundary Router address)
  Advertising Router: 1.1.1.1                      ← ABR 告诉 Area 1 怎么到 ASBR
  TOS: 0  Metric: 300
```

**路由表**：
```cisco
R3# show ip route ospf
O IA  10.0.12.0/30 [110/200] via 10.0.13.1, 00:10:23, Gi0/1
O IA  192.168.20.0/24 [110/300] via 10.0.13.1, 00:10:23, Gi0/1
O E2  200.1.1.0/24 [110/20] via 10.0.13.1, 00:02:15, Gi0/1
  ↑↑                    ↑
外部路由 E2         metric 只有 20（不加内部 cost）
```

### Step 4：把 Area 1 改成 Stub，观察变化

```cisco
! ★ 区域内所有路由器都要配
R1(config-router)# area 1 stub
R3(config-router)# area 1 stub
```

**R3 上的变化**：
```cisco
R3# show ip ospf database external
! 空输出！Type-5 被拒绝了 ✓

R3# show ip route ospf
O IA  10.0.12.0/30 [110/200] via 10.0.13.1, ...
O IA  192.168.20.0/24 [110/300] via 10.0.13.1, ...
O*IA  0.0.0.0/0 [110/101] via 10.0.13.1, ...        ← ABR 自动注入的默认路由 ✓
! 200.1.1.0/24 不见了，被默认路由覆盖
```

**验证连通性**：R3 依然能访问 `200.1.1.0/24`，因为走默认路由。**路由表小了，但连通性不变。**

### Step 5：改成 Totally Stub

```cisco
! ★ no-summary 只在 ABR 上配
R1(config-router)# area 1 stub no-summary
! R3 保持 area 1 stub 不变
```

**R3 上的变化**：
```cisco
R3# show ip route ospf
O*IA  0.0.0.0/0 [110/101] via 10.0.13.1, 00:01:30, Gi0/1
! ★ 只剩一条默认路由！Type-3 也被拒绝了 ✓

R3# show ip ospf database summary
! 只有一条 0.0.0.0 的 Type-3
```

**这就是 Totally Stub 的威力**：分支路由器的路由表从几十条变成 1 条，内存和 CPU 压力大幅降低。

### Step 6：Area 2 配成 NSSA，观察 Type-7 → Type-5 转换

```cisco
R2(config-router)# area 2 nssa
R4(config-router)# area 2 nssa
R5(config-router)# area 2 nssa
R5(config-router)# redistribute static subnets           ! R5 在 NSSA 里做 ASBR
```

**在 R4（NSSA 内部）上看**：
```cisco
R4# show ip ospf database nssa-external

                Type-7 AS External Link States (Area 2)

  LS Type: AS External Link
  Link State ID: 200.1.1.0
  Advertising Router: 5.5.5.5
        Metric Type: 2
        Metric: 20

R4# show ip route ospf
O N2  200.1.1.0/24 [110/20] via 10.0.45.5, 00:01:15, Gi0/1
  ↑↑
 NSSA 外部路由
```

**在 R1（Area 1，跨区域）上看**：
```cisco
R1# show ip ospf database external

                Type-5 AS External Link States           ← 已经被 R2 转换成 Type-5！

  Link State ID: 200.1.1.0
  Advertising Router: 2.2.2.2                     ← 注意：通告者变成了 ABR R2
```

**✅ 验证了 Type-7 → Type-5 的转换发生在 ABR 上。**

### Step 7：路由汇总

```cisco
! R1 汇总 Area 1 的 192.168.10.0/24 和 192.168.11.0/24
R1(config-router)# area 1 range 192.168.10.0 255.255.254.0
```

**在 R2 上看**：
```cisco
! 汇总前
R2# show ip route ospf | include 192.168.1
O IA  192.168.10.0/24 [110/300] via 10.0.12.1, ...
O IA  192.168.11.0/24 [110/300] via 10.0.12.1, ...

! 汇总后
R2# show ip route ospf | include 192.168.1
O IA  192.168.10.0/23 [110/300] via 10.0.12.1, ...     ← 两条变一条 ✓
```

**在 R1 上验证防环路由**：
```cisco
R1# show ip route | include Null0
O    192.168.10.0/23 is a summary, 00:02:15, Null0     ← 自动生成的丢弃路由
```

### Step 8：验证汇总的故障隔离价值（★ 最重要的验证）

```cisco
! 在 R2 上开启 OSPF 事件调试
R2# debug ip ospf events

! 在 R3 上反复 shut/no shut 192.168.10.0 所在的接口
R3(config)# interface Loopback10
R3(config-if)# shutdown
R3(config-if)# no shutdown
```

**汇总前**：R2 会看到大量 LSA 更新和 SPF 重算。
**汇总后**：R2 **完全不受影响**——因为汇总路由 `192.168.10.0/23` 只要还有一条明细存在就保持稳定。

> **这就是路由汇总最核心的价值**：不是"缩小路由表"，而是**"隔离故障，防止本地抖动扩散到全网"**。

---

## ⑩ 排障速查表

| 症状 | 怀疑点 | 验证命令 |
|:--|:--|:--|
| 邻居卡 **EXSTART** | **MTU 不匹配** | `show ip ospf interface \| inc MTU` |
| 邻居卡 **INIT** | ACL 拦了组播 / 单向链路 | `show access-lists`、抓包 |
| 停在 **2WAY** | DROTHER 之间（**正常**） | `show ip ospf neighbor` 看角色 |
| 邻居建不起来 | 5 个条件之一 | `show ip ospf interface` 逐项对比 |
| **配了 Stub 后邻居断了** | **Stub 标志不一致** | 区域内所有路由器都要配 |
| 邻居反复翻转 | Router ID 重复 / 链路抖动 | `show logging \| inc DUP_RTRID` |
| 学不到区域间路由 | ABR 配置 / Type-3 被过滤 | `show ip ospf database summary` |
| **学不到外部路由** | **缺 Type-4**，或区域是 Stub | `show ip ospf database asbr-summary` |
| 外部路由 metric 不合理 | E2 不加内部 cost | 改成 `metric-type 1` |
| 万兆和千兆等价 | **参考带宽没改** | `show ip ospf \| inc Reference` |
| 汇总后某网段不可达 | **过度汇总** | `show ip route <目标>` |
| 虚链路不起来 | 穿越区域是 Stub / 路由不通 | `show ip ospf virtual-links` |
| LSDB 不一致 | MTU / 认证 / 内存 | 逐台 `show ip ospf database` 对比 |
| SPF 频繁运行 | 拓扑震荡 | `show ip ospf statistics`、`show ip ospf \| inc SPF` |

### 定位 SPF 震荡

```cisco
R1# show ip ospf statistics
            OSPF Router with ID (1.1.1.1) (Process ID 1)

  Area 0: SPF algorithm executed 1245 times      ← 次数异常高
  
  SPF calculation time
  Delta T   Intra D-Intra Summ D-Summ Ext D-Ext Total Reason
  00:00:12       0        0     0      0    0     0     0    R
  00:00:23       0        0     0      0    0     0     0    R, N
                                                            ↑
                                       R = Router LSA 变化
                                       N = Network LSA 变化
                                       SN/SA/X = Summary/ASBR/External

R1# show ip ospf | include SPF
 SPF algorithm last executed 00:00:12.345 ago
```

**如果 SPF 执行次数持续快速增长**，说明有链路在抖动。用 `show ip ospf database router` 看哪个 LSA 的 Seq# 增长最快，就能定位到抖动的设备。

---

## ⑪ 三厂商对照 + 考点自测

### 三厂商对照

| 目的 | Cisco | H3C | 华为 |
|:--|:--|:--|:--|
| 多区域 | `network X area 1` | `area 1` → `network X` | `area 1` → `network X` |
| Stub | `area 1 stub` | `stub`（area 视图下） | `stub` |
| Totally Stub | `area 1 stub no-summary` | `stub no-summary` | `stub no-summary` |
| NSSA | `area 1 nssa` | `nssa` | `nssa` |
| 区域间汇总 | `area 1 range X Y` | `abr-summary X Y`（area 视图） | `abr-summary X Y` |
| 外部汇总 | `summary-address X Y` | `asbr-summary X Y` | `asbr-summary X Y` |
| 虚链路 | `area 1 virtual-link 2.2.2.2` | `vlink-peer 2.2.2.2` | `vlink-peer 2.2.2.2` |
| 参考带宽 | `auto-cost reference-bandwidth 100000` | `bandwidth-reference 100000` | `bandwidth-reference 100000` |
| 默认路由 | `default-information originate` | `default-route-advertise` | `default-route-advertise` |
| 查 LSDB | `show ip ospf database` | `display ospf lsdb` | `display ospf lsdb` |
| 查邻居 | `show ip ospf neighbor` | `display ospf peer` | `display ospf peer` |

### 考点

- **LSA 类型表（1/2/3/4/5/7）**：谁产生、传播范围、作用 ★★★
- **四种特殊区域**对各类 LSA 的过滤规则 ★★★
- **`no-summary` 只在 ABR 配**
- **Stub/NSSA 标志必须区域内一致**（否则邻居断）
- **E1 vs E2 的 metric 计算**，E1 优于 E2
- **Type-7 → Type-5 转换发生在 ABR**
- **`area range`（ABR）vs `summary-address`（ASBR）**
- **虚链路的穿越区域不能是 Stub**
- **参考带宽必须全网统一**

### 自测题

**1.** 默写 OSPF 的 1/2/3/4/5/7 类 LSA：谁产生、传播范围、作用。

<details><summary>答案</summary>

| Type | 名称 | **谁产生** | **传播范围** | **作用** | 路由标记 |
|:--|:--|:--|:--|:--|:--|
| **1** | Router LSA | **每台 OSPF 路由器** | **本区域内** | 描述本路由器的接口、链路、cost | `O` |
| **2** | Network LSA | **DR** | **本区域内** | 描述多路访问网络上连了哪些路由器 | `O` |
| **3** | Summary LSA | **ABR** | **跨区域** | 把区域内的**网段**（非拓扑）通告给其他区域 | **`O IA`** |
| **4** | ASBR Summary LSA | **ABR** | 跨区域 | 告诉其他区域**怎么到达 ASBR** | 无独立标记 |
| **5** | AS External LSA | **ASBR** | **整个 AS**（Stub/NSSA 除外） | 外部路由（重分发进来的） | **`O E1`/`O E2`** |
| **7** | NSSA External LSA | **NSSA 内的 ASBR** | **仅 NSSA 区域内** | NSSA 里的外部路由，到 ABR 转成 Type-5 | **`O N1`/`O N2`** |

**理解要点**：

**① Type-1 和 Type-2 是"区域内的拓扑细节"，绝不跨区域。**
这是区域隔离的机制——Area 1 的链路抖动只会重新生成 Area 1 的 Type-1 LSA，Area 0 完全感知不到。

**② Type-3 传的是"网段 + cost"，不是拓扑。**
所以区域间路由本质上是**距离矢量式**的（相信 ABR 报的 cost），这就是为什么 OSPF 要求"所有区域必须连 Area 0"——用星型拓扑防止区域间环路。

**③ Type-4 和 Type-5 必须配对。**
- Type-5 说"有这些外部路由，是 ASBR X 通告的"
- Type-4 说"怎么到达 ASBR X"
- 只有 Type-5 没有 Type-4 → **路由学到了但不可达**

例外：ASBR 在本区域内时不需要 Type-4（本区域的 Type-1 LSA 已经描述了怎么到它）。

**④ Type-7 只是 Type-5 的"区域内替身"。**
NSSA 不允许 Type-5 进来，但区域内部又需要引入外部路由，于是用 Type-7 在区域内传播，到 ABR 转成 Type-5 再泛洪出去。

**排障时的应用**：
```cisco
show ip ospf database                    ! 总览
show ip ospf database router             ! Type-1
show ip ospf database network            ! Type-2
show ip ospf database summary            ! Type-3
show ip ospf database asbr-summary       ! Type-4
show ip ospf database external           ! Type-5
show ip ospf database nssa-external      ! Type-7
```

**"学不到某条路由"的排查逻辑**：先看它应该是哪类 LSA，再看对应的 database 里有没有。
- **有 LSA 但路由表没有** → 问题在本地（AD 被抢、被过滤、Type-4 缺失）
- **连 LSA 都没有** → 问题在上游（没通告、被区域类型过滤、被汇总）
</details>

**2.** Stub、Totally Stub、NSSA、Totally NSSA 分别拒绝哪些 LSA？

<details><summary>答案</summary>

| 区域类型 | Type-1,2 | **Type-3** | **Type-4,5** | **Type-7** | 默认路由 |
|:--|:--|:--|:--|:--|:--|
| **标准区域** | ✅ | ✅ | ✅ | ❌ | — |
| **Stub** | ✅ | ✅ | ❌ **拒绝** | ❌ | ✅ 自动注入 |
| **Totally Stub** | ✅ | ❌ **拒绝** | ❌ **拒绝** | ❌ | ✅ 自动注入 |
| **NSSA** | ✅ | ✅ | ❌ **拒绝** | ✅ **允许** | ⚠️ **不自动**，需手工 |
| **Totally NSSA** | ✅ | ❌ **拒绝** | ❌ **拒绝** | ✅ **允许** | ✅ 自动注入 |

**记忆公式**：
```
Stub          = 拒绝【外部路由】(Type-4,5)
Totally Stub  = Stub + 拒绝【区域间路由】(Type-3)
NSSA          = Stub + 允许【本区域产生外部路由】(Type-7)
Totally NSSA  = Totally Stub + 允许 Type-7

"Totally" → 额外拒绝 Type-3
"NSSA"    → 额外允许 Type-7
```

**配置要点（考点）**：

```cisco
! Stub：区域内【所有】路由器都要配
R-ABR(config-router)#      area 1 stub
R-Internal(config-router)# area 1 stub

! Totally Stub：no-summary 【只在 ABR 上】配
R-ABR(config-router)#      area 1 stub no-summary    ← 只有 ABR
R-Internal(config-router)# area 1 stub               ← 内部还是 stub

! NSSA 需要手工注入默认路由
R-ABR(config-router)# area 1 nssa default-information-originate
```

**⚠️ 两个必须记住的规则**：

**① 区域内所有路由器的 Stub/NSSA 标志必须一致。**
这是"建立邻居的 5 个条件"之一。不一致 → **邻居直接建不起来**。

**这是配 Stub 时最常见的事故**：只在 ABR 上配了 `area 1 stub`，内部路由器忘了配 → 整个区域的邻居全断。

**② `no-summary` 只在 ABR 上有意义。**
它的作用是让 ABR 不向该区域发送 Type-3 LSA。内部路由器配了也没用（它本来就不产生 Type-3）。

**选型建议**：

| 场景 | 推荐 |
|:--|:--|
| 分支机构，只有一个出口，无外部路由 | **Totally Stub**（路由表最小） |
| 分支机构，但有本地外部路由（比如本地静态路由要引进来） | **Totally NSSA** |
| 需要看到区域间明细（做流量工程） | Stub |
| 有多个 ABR，需要选最优出口 | 标准区域或 Stub（保留 Type-3 以便比较 cost） |

**为什么 Totally Stub 效果最好**：分支路由器的路由表可能从几百条降到 1 条（一条默认路由）。内存、CPU、收敛速度全面改善。而连通性完全不受影响——反正所有流量都要走那一个出口。
</details>

**3.** OSPF 外部路由的 E1 和 E2 有什么区别？什么时候用哪个？

<details><summary>答案</summary>

| | **E2**（默认） | **E1** |
|:--|:--|:--|
| Metric 计算 | **只算外部 cost** | **外部 cost + 到 ASBR 的内部 cost** |
| 特点 | 无论离 ASBR 多远，metric 都不变 | 反映真实的端到端开销 |
| 路由标记 | `O E2` | `O E1` |
| 优先级 | 低 | **高**（E1 永远优于 E2） |

**关键场景：有多个 ASBR 时**

```
              [外部网络 200.1.1.0/24]
               │                  │
          外部cost 20        外部cost 20
          ┌────▼────┐        ┌────▼────┐
          │  ASBR1  │        │  ASBR2  │
          └────┬────┘        └────┬────┘
          内部cost 10         内部cost 100
               │                  │
               └────────┬─────────┘
                     [ R5 ]
```

**用 E2（默认）**：
```cisco
R5# show ip route 200.1.1.0
O E2  200.1.1.0/24 [110/20] via <ASBR1>, ...
                       ↑ 都是 20
      200.1.1.0/24 [110/20] via <ASBR2>, ...
                       ↑ 都是 20
! → R5 认为两条路径等价，做 ECMP 负载均衡
! → 一半流量走了内部 cost 100 的远路 ❌
```

**用 E1**：
```cisco
ASBR1(config-router)# redistribute static subnets metric-type 1
ASBR2(config-router)# redistribute static subnets metric-type 1
```
```cisco
R5# show ip route 200.1.1.0
O E1  200.1.1.0/24 [110/30] via <ASBR1>, ...
                       ↑ 20 + 10 = 30，被选中 ✓
! ASBR2 的路径 metric = 20 + 100 = 120，被淘汰
```

**选择建议**：

| 场景 | 推荐 | 理由 |
|:--|:--|:--|
| **多个 ASBR** | **E1** | 能正确选择"离自己最近的出口" |
| **单个 ASBR** | **E2**（默认） | metric 稳定，不会因内部拓扑变化而波动，减少不必要的 LSA 更新 |
| 多个 ASBR 但希望负载均衡 | E2 | 故意让它们等价 |
| 需要明确的主备关系 | E1 + 调整内部 cost | 精确控制 |

**其他要点**：

**① E1 永远优于 E2**（无论 metric 数值大小）
```
O E1  200.1.1.0/24 [110/500]     ← 会被选中
O E2  200.1.1.0/24 [110/20]      ← 即使 metric 小得多，也输
```

**② OSPF 路由选择的完整优先级**：
```
① 区域内路由 (O, Intra-area)
② 区域间路由 (O IA, Inter-area)
③ 外部 Type-1 (O E1 / O N1)
④ 外部 Type-2 (O E2 / O N2)
```
**先比类型，类型相同再比 metric。** 一条 cost 10000 的区域内路由会赢过 cost 1 的 E2 路由。

**③ N1/N2 是 NSSA 版本的 E1/E2**，规则完全一样（默认也是 N2）。

**④ 重分发时的默认 metric**：
```cisco
! OSPF 重分发的默认 metric 是 20（BGP 是 1）
R1(config-router)# redistribute static subnets                     ! metric 默认 20
R1(config-router)# redistribute static subnets metric 100          ! 手工指定
R1(config-router)# default-metric 50                               ! 全局默认值
```
</details>

**4.** ABR 上的 `area 1 range` 和 ASBR 上的 `summary-address` 有什么区别？

<details><summary>答案</summary>

| | **`area X range`** | **`summary-address`** |
|:--|:--|:--|
| 配在哪 | **ABR** | **ASBR** |
| 汇总什么 | **Type-3 LSA**（区域内路由） | **Type-5 / Type-7 LSA**（外部路由） |
| 影响 | 区域间的路由通告 | 外部路由的通告 |
| 命令位置 | `router ospf` 下 | `router ospf` 下 |

```cisco
! ── ABR：汇总 Area 1 的内部路由，通告给其他区域 ──
R-ABR(config)# router ospf 1
R-ABR(config-router)# area 1 range 192.168.0.0 255.255.252.0
R-ABR(config-router)# area 1 range 10.99.0.0 255.255.0.0 not-advertise   ! 抑制
R-ABR(config-router)# area 1 range 192.168.4.0 255.255.252.0 cost 100    ! 指定 cost

! ── ASBR：汇总重分发进来的外部路由 ──
R-ASBR(config)# router ospf 1
R-ASBR(config-router)# summary-address 200.1.0.0 255.255.0.0
R-ASBR(config-router)# summary-address 200.99.0.0 255.255.0.0 not-advertise
```

**记忆**：
- **`area X range`** = "把 **area X 里的**路由汇总起来" → 只有 ABR 有跨区域的视角
- **`summary-address`** = "把**外部**路由汇总起来" → 只有 ASBR 才有外部路由

**汇总的 cost 计算规则**：
- **默认**：取被汇总的明细路由中**最小的 cost**（Cisco 的行为；RFC 建议取最大值，其他厂商可能不同）
- 用 `cost` 关键字可以手工指定

**自动生成的防环路由**：
```cisco
R-ABR# show ip route | include Null0
O    192.168.0.0/22 is a summary, 00:05:23, Null0
```
**作用**：如果有流量的目的地落在汇总范围内，但实际没有对应的明细路由，这条 Null0 路由会**直接丢弃**它。

**为什么需要**：没有这条路由的话，包会匹配到默认路由被送回上游，上游又匹配汇总路由送回来 → **来回弹形成环路**，直到 TTL 耗尽。

**汇总最重要的价值（很多人只知道"缩小路由表"）**：

**★ 隔离故障。**

```
   Area 1 里有 192.168.10.0/24，链路每天抖动 100 次
   
   没有汇总：
   每次抖动 → Type-3 LSA 更新 → 泛洪到所有区域 → 所有路由器重跑 SPF
   → 全网每天重算 100 次
   
   有汇总（192.168.10.0/23）：
   明细抖动 → 只要还有至少一条明细存在，汇总路由就不变
   → 其他区域完全不知道，0 次重算 ✓
```

**这才是汇总的核心价值。** 路由表小一点是次要的，防止本地故障扩散到全网才是关键。

**⚠️ 过度汇总的风险**：
```
   ABR 通告 192.168.0.0/22（覆盖 0,1,2,3）
   但 Area 1 里其实只有 192.168.0.0/24 和 192.168.1.0/24
   
   → 192.168.2.0/24 和 192.168.3.0/24 的流量会被吸引到这个 ABR
   → 到了以后无路可走（被 Null0 丢弃）→ 黑洞
```

**规划原则**：**地址规划必须支持汇总**（见 [第 1 章](01-企业网络架构与设计.md)），且**汇总范围要精确匹配实际拥有的地址块**。
</details>

**5.** 你在 ABR 上配了 `area 1 stub`，结果 Area 1 里的所有邻居都断了。为什么？

<details><summary>答案</summary>

**因为 Stub 标志是"建立邻居的 5 个条件"之一，区域内所有路由器必须一致。**

**建立 OSPF 邻居的 5 个条件**（回顾 [Stage 1 第 4 章](../01-CCNA补齐篇/04-OSPF单区域.md)）：
1. Area ID 相同
2. Hello / Dead 间隔相同
3. 认证类型和密钥相同
4. **★ Stub / NSSA 标志相同**
5. 子网掩码相同（广播网络）

你只在 ABR 上配了 `area 1 stub`：
```
ABR 的 Hello 包里：E-bit = 0（我是 Stub 区域）
内部路由器的 Hello 包里：E-bit = 1（我是标准区域）
        ↓
   ★ 不匹配，Hello 包被直接丢弃 ★
        ↓
   邻居关系全部断开
```

**日志会明确提示**：
```
%OSPF-4-ERRRCV: Received invalid packet: mismatched area stub/transit flag 
from 10.0.13.3, GigabitEthernet0/1
```

**正确做法：区域内所有路由器都要配**
```cisco
! ABR
R1(config)# router ospf 1
R1(config-router)# area 1 stub

! Area 1 内的每一台路由器
R3(config)# router ospf 1
R3(config-router)# area 1 stub

R4(config)# router ospf 1
R4(config-router)# area 1 stub
```

**Totally Stub 的特殊之处**：
```cisco
! ABR 上配 no-summary
R1(config-router)# area 1 stub no-summary       ← 只有 ABR

! 内部路由器只配 stub（不带 no-summary）
R3(config-router)# area 1 stub                  ← 不需要 no-summary
```

**为什么内部路由器不需要 no-summary**：
- `stub` 标志会写进 Hello 包，**必须一致**
- `no-summary` **不影响 Hello 包**，它只是告诉 ABR"不要向这个区域发 Type-3 LSA"
- 内部路由器本来就不产生 Type-3，配了也没意义

**同样的规则适用于 NSSA**：
```cisco
R1(config-router)# area 1 nssa                  ! ABR
R3(config-router)# area 1 nssa                  ! 内部，必须一致
R1(config-router)# area 1 nssa no-summary       ! 只在 ABR
```

**⚠️ 生产环境的操作建议**：

改区域类型会**中断该区域内所有邻居关系**，导致业务中断。所以：

1. **必须在维护窗口做**
2. **准备好完整的配置脚本**，逐台快速执行（越快邻居恢复越快）
3. **从远端往 ABR 方向配**（先配最远的内部路由器，最后配 ABR）——这样 ABR 一改完，所有邻居同时恢复
   - 反过来先配 ABR 的话，中间过程会有更长时间的断连
4. **用 `configure terminal revert timer` 做保险**
5. **有带外管理通道**（改区域类型可能让你失去到远端设备的路由）

**验证**：
```cisco
R3# show ip ospf | include Area 1 -A 5
    Area 1
        Number of interfaces in this area is 2
        It is a stub area                        ← 确认
        generates stub default route with cost 1

R3# show ip ospf neighbor                        ! 确认邻居都回来了
```
</details>

---

**上一章** ← [06 EIGRP](06-EIGRP.md) ｜ **下一章** → [08 BGP](08-BGP.md)
