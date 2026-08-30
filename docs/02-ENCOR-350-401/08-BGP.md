# 08 · BGP

> ⚠️ **与 OSPF 并列的最高权重章节。**
> 通过标准：**能背出 BGP 选路 11 步的前 6 步，并说清每一步的实际用途。**

## ① 这章解决什么问题

公司接了两个运营商：电信 200Mbps，联通 100Mbps。需求是：

1. 电信断了自动切联通（**冗余**）
2. 平时优先走电信，但访问联通用户时走联通（**智能选路**）
3. 让外部知道"访问我们公司的这个网段，优先从电信来"（**入向流量控制**）

**OSPF/EIGRP 做不到这些。** 它们是 IGP（内部网关协议），设计目标是"在我自己的网络里找最短路"，靠 cost/metric 这种技术指标选路。

**BGP 是 EGP（外部网关协议）**，它的设计目标完全不同：
- 连接**不同的自治系统（AS）**，也就是不同的组织
- **选路依据是"策略"，不是"最短"** —— 因为跨组织时，"最短"往往不是"最合适"（可能更贵、可能不可信、可能有合同约束）
- 承载**海量路由**（互联网全表 90 万+条）

**理解 BGP 的关键：它是一个"策略引擎"，不是"寻路算法"。**

---

## ② BGP 基础

### 2.1 AS（自治系统）

**AS = 由同一个组织管理、执行统一路由策略的一组网络。**

| 类型 | 范围 | 说明 |
|:--|:--|:--|
| 公有 AS（16 位） | 1 – 64495 | 需向 RIR 申请 |
| **私有 AS（16 位）** | **64512 – 65534** | 企业内部使用 |
| 32 位 AS | 65536 – 4294967295 | 地址耗尽后扩展 |
| 私有 AS（32 位） | 4200000000 – 4294967294 | |

### 2.2 eBGP vs iBGP（★ 核心区别）

| | **eBGP** | **iBGP** |
|:--|:--|:--|
| 邻居所在 AS | **不同 AS** | **相同 AS** |
| 管理距离 (AD) | **20** | **200** |
| 默认 TTL | **1**（必须直连，或用 `ebgp-multihop`） | **255** |
| 下一跳处理 | **改成自己**（next-hop-self 行为） | **不改**（保持原样） |
| AS-Path | **加上自己的 AS 号** | **不加** |
| **路由传递规则** | 学到的路由**可以传给所有邻居** | **★ 从 iBGP 学到的路由，不能再传给其他 iBGP 邻居**（水平分割） |
| 是否需要全互联 | 否 | **★ 是**（或用 RR / Confederation） |

### 2.3 iBGP 水平分割：为什么需要全互联

**规则：从一个 iBGP 邻居学到的路由，不能通告给另一个 iBGP 邻居。**

**为什么有这条规则**：BGP 用 **AS-Path** 防环。但在同一个 AS 内部传递时，AS-Path **不会增加**——所以无法用它检测环路。于是只能用"水平分割"这个更粗暴的规则来防环。

**后果：iBGP 邻居必须全互联（full mesh）。**

```
   4 台路由器全互联需要：4×3/2 = 6 条 iBGP 会话
   10 台需要：45 条
   50 台需要：★ 1225 条 ★  ← 完全不可维护
```

**两种解法**：

#### 解法 1：Route Reflector (RR，路由反射器) ★ 主流

```
              ┌──────────┐
              │    RR    │  ← 路由反射器
              └─┬──┬──┬──┘
                │  │  │
          ┌─────┘  │  └─────┐
       ┌──▼──┐  ┌──▼──┐  ┌──▼──┐
       │ RC1 │  │ RC2 │  │ RC3 │  ← Route Reflector Client
       └─────┘  └─────┘  └─────┘
       
   只需 3 条 iBGP 会话（而不是全互联的 3 条... 但 10 台时是 9 条 vs 45 条）
```

**RR 打破水平分割的规则**：

| 路由来源 | RR 反射给谁 |
|:--|:--|
| **从 Client 学到** | **反射给所有其他 Client + 所有非 Client** |
| **从非 Client（普通 iBGP）学到** | **只反射给 Client** |
| **从 eBGP 学到** | 通告给所有邻居 |

**防环机制**：
- **`ORIGINATOR_ID`**：记录路由的原始产生者。路由器收到 ORIGINATOR_ID 是自己的路由 → 丢弃
- **`CLUSTER_LIST`**：记录经过的 RR 集群。RR 收到含自己 Cluster-ID 的路由 → 丢弃

```cisco
! ── RR 上配置 ──
RR(config)# router bgp 65001
RR(config-router)# neighbor 10.0.0.2 remote-as 65001
RR(config-router)# neighbor 10.0.0.2 route-reflector-client       ! ★
RR(config-router)# neighbor 10.0.0.3 remote-as 65001
RR(config-router)# neighbor 10.0.0.3 route-reflector-client
RR(config-router)# bgp cluster-id 1.1.1.1

! ── Client 上不需要任何特殊配置 ──
RC1(config)# router bgp 65001
RC1(config-router)# neighbor 10.0.0.1 remote-as 65001
```

> **Client 完全不知道自己是 Client** —— 这是 RR 设计的优雅之处，可以渐进式部署。

#### 解法 2：Confederation（联邦）

把一个大 AS 分成多个"子 AS"，子 AS 之间跑 eBGP（但对外看起来还是一个 AS）。

配置复杂，实际用得少，**了解即可**。

---

## ③ BGP 报文与状态机

### 3.1 五种报文

| 报文 | 作用 |
|:--|:--|
| **OPEN** | 建立会话，协商参数（AS 号、Router ID、Hold Time、能力） |
| **UPDATE** | 通告新路由 / 撤销路由（withdraw） |
| **KEEPALIVE** | 保活，默认 **60 秒**（Hold Time 的 1/3） |
| **NOTIFICATION** | 出错，**发完就断开会话** |
| ROUTE-REFRESH | 请求对方重发路由（软重置用） |

**默认定时器**：Keepalive **60 秒**，Hold Time **180 秒**。

### 3.2 状态机（★ 排障核心）

```
   Idle → Connect → Active → OpenSent → OpenConfirm → Established
```

| 状态 | 含义 | **卡在这里说明什么** |
|:--|:--|:--|
| **Idle** | 初始/出错 | **TCP 都建不起来** —— 路由不通、`neighbor` 语句写错、被 ACL 拦、对端不存在 |
| **Connect** | 正在建 TCP | 短暂状态 |
| **Active** | **TCP 建立失败，正在重试** | ⚠️ **名字骗人！Active = 有问题**。TCP 179 不通、单向路由、源地址不对 |
| **OpenSent** | 已发 OPEN，等对方回 | AS 号不匹配、Router ID 冲突 |
| **OpenConfirm** | 已收到 OPEN，等 KEEPALIVE | 短暂状态 |
| **Established** | ✅ **正常，开始交换路由** | — |

> ⚠️ **`Active` 是 BGP 最容易误解的状态。** 和 EIGRP 一样，Active **不是"活跃正常"，而是"正在尝试重连"**——说明 TCP 连接建立失败了。
>
> **看到 Active 就是有问题。**

**`show ip bgp summary` 的读法**：
```cisco
R1# show ip bgp summary
BGP router identifier 1.1.1.1, local AS number 65001

Neighbor      V   AS  MsgRcvd  MsgSent  TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.12.2     4  200     1245     1230       5    0    0 02:15:33      1245
10.0.13.3     4  300        0        0       0    0    0 never        Active
192.168.1.5   4 65001      45       48       5    0    0 00:15:12         3
                                                            ↑            ↑
                                                        建立时长    收到的前缀数
```

**State/PfxRcd 列**：
- **数字** = 已建立，收到了 N 条前缀 ✅
- **`Active`** = TCP 建不起来 ❌
- **`Idle`** = 初始或出错 ❌
- **`Idle (Admin)`** = 被手工 `shutdown` 了

---

## ④ BGP 路径属性

### 4.1 属性分类

| 类别 | 说明 | 例子 |
|:--|:--|:--|
| **公认必遵** (Well-known Mandatory) | 所有实现必须支持，**每条 UPDATE 必须携带** | **AS_PATH、NEXT_HOP、ORIGIN** |
| **公认自选** (Well-known Discretionary) | 必须支持，但可以不带 | **LOCAL_PREF**、ATOMIC_AGGREGATE |
| **可选传递** (Optional Transitive) | 可以不支持，但要原样传给下一个 AS | **COMMUNITY**、AGGREGATOR |
| **可选非传递** (Optional Non-transitive) | 可以不支持，不支持就丢弃 | **MED**、ORIGINATOR_ID、CLUSTER_LIST |

### 4.2 关键属性详解

#### AS_PATH —— 防环 + 选路

记录路由经过的所有 AS 号列表。

**两个作用**：
1. **防环**：收到的路由如果 AS_PATH 里有自己的 AS 号 → **直接丢弃**
2. **选路**：AS_PATH 越短越优（选路第 4 步）

```cisco
R1# show ip bgp 200.1.1.0
BGP routing table entry for 200.1.1.0/24
  Paths: (2 available, best #1)
  200 300 400                                    ← AS_PATH（3 跳）
    10.0.12.2 from 10.0.12.2 (2.2.2.2)
  500 600 700 800                                ← AS_PATH（4 跳）
    10.0.13.3 from 10.0.13.3 (3.3.3.3)
```

**AS_PATH Prepending（人为加长，最常用的入向流量控制手段）**：
```cisco
R1(config)# route-map PREPEND-OUT permit 10
R1(config-route-map)#  set as-path prepend 65001 65001 65001
R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.13.3 route-map PREPEND-OUT out
```
把自己的 AS 号重复几遍，让这条路径看起来更远，**外部就不会优先选它**。

#### NEXT_HOP —— 最容易出问题的属性

| 场景 | NEXT_HOP 行为 |
|:--|:--|
| **eBGP 通告给邻居** | 改成**自己的接口地址** |
| **iBGP 通告给邻居** | **不改，保持原样** ⚠️ |

**iBGP 的经典问题**：

```
   AS 200                    AS 65001
   [R2] ──eBGP── [R1] ──iBGP── [R3]
   10.0.12.2    10.0.12.1      内部
```

1. R1 从 R2（eBGP）学到路由，NEXT_HOP = `10.0.12.2`
2. R1 通过 iBGP 传给 R3 时，**NEXT_HOP 保持 `10.0.12.2` 不变**
3. **R3 不知道怎么到 `10.0.12.2`**（这是 AS 200 的地址，R3 的 IGP 里没有）
4. → 路由**不可达（inaccessible）**，不会被选为最优

**验证**：
```cisco
R3# show ip bgp
     Network          Next Hop            Metric LocPrf Weight Path
 *   200.1.1.0/24     10.0.12.2                0    100      0 200 i
 ↑                    ↑
 没有 > 标记        下一跳不可达
 （不是最优）
```

**解法（三选一）**：
```cisco
! 方案 1（★ 最常用）：R1 上配 next-hop-self
R1(config-router)# neighbor <R3的IP> next-hop-self

! 方案 2：把互联网段通告进 IGP（不推荐，污染 IGP）
R1(config)# router ospf 1
R1(config-router)# network 10.0.12.0 0.0.0.3 area 0

! 方案 3：用 route-map 改写
R1(config)# route-map SET-NH permit 10
R1(config-route-map)#  set ip next-hop 1.1.1.1
```

> **`next-hop-self` 是 iBGP 部署的必配项**。忘了它，路由学到了但全部不可达——这是 BGP 排障 Top 3 的问题。

#### LOCAL_PREF —— 控制"出向"流量

**含义**："在我这个 AS 内部，从哪个出口出去更好"

- **越大越优**（默认 100）
- **只在 AS 内部传递**（iBGP 会传，eBGP 不会传出去）
- **选路第 2 步**（仅次于 Weight）

```cisco
! 让去往某网段的流量优先走电信
R1(config)# route-map SET-LP permit 10
R1(config-route-map)#  set local-preference 200
R1(config)# router bgp 65001
R1(config-router)# neighbor <电信邻居> route-map SET-LP in
```

**这是控制"出向流量"（我方发出的流量）最常用的手段。**

#### MED —— 建议"入向"流量

**含义**："我建议你从哪个入口进来"（Multi-Exit Discriminator）

- **越小越优**（默认 0）
- **只传给邻居 AS，不再往下传**（可选非传递）
- **选路第 7 步**（优先级很低）
- **默认只在同一个邻居 AS 的多条路径之间比较**

```cisco
R1(config)# route-map SET-MED permit 10
R1(config-route-map)#  set metric 50
R1(config-router)# neighbor <邻居> route-map SET-MED out
```

> ⚠️ **MED 是"建议"不是"命令"**。对方 AS 完全可以用 LOCAL_PREF 覆盖你的 MED（LOCAL_PREF 在第 2 步，MED 在第 7 步）。所以**MED 在实践中经常不生效**。
>
> **入向流量控制更可靠的手段是 AS_PATH Prepending**（因为它在第 4 步，比 MED 靠前），或者干脆**只向一个运营商通告某个网段**。

#### WEIGHT —— Cisco 私有，只在本机有效

- **越大越优**（默认：自己产生的路由 32768，学到的 0）
- **Cisco 私有，不传给任何邻居**
- **选路第 1 步**（最高优先级）

```cisco
R1(config-router)# neighbor 10.0.12.2 weight 200
```

**用途**：只想影响**这一台路由器**的选路，不想影响整个 AS。

**LOCAL_PREF vs WEIGHT**：
- **WEIGHT** = 只影响本机（本地策略）
- **LOCAL_PREF** = 影响整个 AS（全局策略）

#### COMMUNITY —— 路由打标签

给路由打一个"标签"，方便批量做策略。

**公认 Community**：

| Community | 含义 |
|:--|:--|
| **`no-export`** | 不通告给 **eBGP 邻居**（可以传给同联邦内的子 AS） |
| **`no-advertise`** | **不通告给任何邻居** |
| `local-AS` | 不通告给联邦外的邻居 |
| `internet` | 通告给所有（默认） |

```cisco
R1(config)# ip bgp-community new-format          ! 用 AS:NN 格式显示

R1(config)# route-map SET-COMM permit 10
R1(config-route-map)#  set community 65001:100 no-export

R1(config-router)# neighbor 10.0.12.2 send-community      ! ★ 必须开，否则不发送
```

> **`send-community` 必须显式开启**，Cisco 默认不发送 Community 属性。忘了配 → 对端收不到 Community → 策略失效。

#### ORIGIN

路由的来源：

| 值 | 显示 | 含义 | 优先级 |
|:--|:--|:--|:--|
| IGP | **`i`** | 用 `network` 命令注入的 | **最优** |
| EGP | `e` | 从 EGP 学的（已废弃） | 中 |
| Incomplete | **`?`** | **重分发进来的** | 最差 |

**选路第 6 步**：`i` > `e` > `?`

---

## ⑤ BGP 选路顺序（★★★ 必背）

**当有多条路径到同一个前缀时，BGP 按以下顺序逐条比较，分出胜负就停止：**

| 步 | 属性 | 规则 | 记忆 | 实战重要性 |
|:--|:--|:--|:--|:--|
| **0** | 下一跳可达吗？ | 不可达直接淘汰 | — | ★★★ |
| **1** | **WEIGHT** | **越大越优**（Cisco 私有，本机有效） | **W**e | ★★★ |
| **2** | **LOCAL_PREF** | **越大越优**（AS 内部） | **L**ove | ★★★★★ |
| **3** | 本地产生的路由 | 自己 `network`/聚合的优先 | **O**ranges | ★★ |
| **4** | **AS_PATH 长度** | **越短越优** | **A**nd | ★★★★★ |
| **5** | **ORIGIN** | **i > e > ?** | **A**pples | ★★★ |
| **6** | **MED** | **越小越优** | **M**mm | ★★★ |
| **7** | **eBGP > iBGP** | eBGP 优先 | **E**at | ★★★★ |
| **8** | **到下一跳的 IGP metric** | **越小越优** | **I**GP | ★★★ |
| **9** | 更老的 eBGP 路径 | 稳定性优先 | — | ★ |
| **10** | Router ID 小的 | 平局决胜 | **R**outer | ★ |
| **11** | 邻居 IP 小的 | 最终决胜 | — | ★ |

**经典记忆口诀**：
```
We Love Oranges As Apples, Mmm, Eat In Rome
 W   L      O    A     A    M    E   I   R
```

**中文记忆版**：
```
① 权重大（Weight）
② 本地优先大（Local_Pref）
③ 自己产生的
④ AS 路径短
⑤ 起源 i > e > ?
⑥ MED 小
⑦ eBGP 优于 iBGP
⑧ IGP 度量小
⑨ 老的稳
⑩ Router ID 小
```

**实战中最常用的三个**：

| 目标 | 用什么 | 方向 |
|:--|:--|:--|
| **控制自己发出的流量走哪个出口** | **LOCAL_PREF**（第 2 步） | 入向 route-map |
| **影响别人从哪个入口进来** | **AS_PATH Prepending**（第 4 步） | 出向 route-map |
| **只影响本机的选路** | **WEIGHT**（第 1 步） | 入向 |

**验证选路结果**：
```cisco
R1# show ip bgp 200.1.1.0/24
BGP routing table entry for 200.1.1.0/24, version 5
Paths: (2 available, best #1, table default)
  Advertised to update-groups: 1
  Refresh Epoch 1
  200 300
    10.0.12.2 from 10.0.12.2 (2.2.2.2)
      Origin IGP, metric 0, localpref 200, valid, external, best
                                        ↑                        ↑
                                  LOCAL_PREF 200            被选中
  500 600 700
    10.0.13.3 from 10.0.13.3 (3.3.3.3)
      Origin IGP, metric 0, localpref 100, valid, external
                                        ↑
                                  LOCAL_PREF 100，输了
```

---

## ⑥ 配置命令

```cisco
! ═══ 基础配置 ═══
R1(config)# router bgp 65001
R1(config-router)# bgp router-id 1.1.1.1
R1(config-router)# bgp log-neighbor-changes

! ── eBGP 邻居 ──
R1(config-router)# neighbor 10.0.12.2 remote-as 200
R1(config-router)# neighbor 10.0.12.2 description ### CHINA-TELECOM ###
R1(config-router)# neighbor 10.0.12.2 password MyBgpSecret         ! MD5 认证
R1(config-router)# neighbor 10.0.12.2 timers 10 30                 ! 加快检测

! ── iBGP 邻居（★ 用 Loopback 建立）──
R1(config-router)# neighbor 3.3.3.3 remote-as 65001
R1(config-router)# neighbor 3.3.3.3 update-source Loopback0        ! ★ 必须
R1(config-router)# neighbor 3.3.3.3 next-hop-self                  ! ★ 必须

! ── eBGP 多跳（用 Loopback 建 eBGP 时需要）──
R1(config-router)# neighbor 2.2.2.2 ebgp-multihop 2
R1(config-router)# neighbor 2.2.2.2 update-source Loopback0
! 或者用更精确的
R1(config-router)# neighbor 2.2.2.2 ttl-security hops 2

! ── 通告网络（★ 必须路由表里已存在完全匹配的路由）──
R1(config-router)# network 192.168.1.0 mask 255.255.255.0
R1(config-router)# redistribute static route-map STATIC-TO-BGP

! ── 路由聚合 ──
R1(config-router)# aggregate-address 192.168.0.0 255.255.252.0
R1(config-router)# aggregate-address 192.168.0.0 255.255.252.0 summary-only    ! 只发聚合
R1(config-router)# aggregate-address 192.168.0.0 255.255.252.0 as-set          ! 保留 AS-Path

! ═══ Route Reflector ═══
RR(config-router)# neighbor 10.0.0.2 route-reflector-client
RR(config-router)# bgp cluster-id 1.1.1.1

! ═══ 策略：Route-Map ═══
R1(config)# ip prefix-list MY-NETS seq 5 permit 192.168.0.0/16 le 24

R1(config)# route-map SET-LOCALPREF permit 10
R1(config-route-map)#  match ip address prefix-list MY-NETS
R1(config-route-map)#  set local-preference 200
R1(config-route-map)# route-map SET-LOCALPREF permit 20            ! ★ 别忘了兜底

R1(config-router)# neighbor 10.0.12.2 route-map SET-LOCALPREF in

! ═══ 策略：AS-Path Prepending ═══
R1(config)# route-map PREPEND permit 10
R1(config-route-map)#  set as-path prepend 65001 65001 65001
R1(config-router)# neighbor 10.0.13.3 route-map PREPEND out

! ═══ 前缀过滤 ═══
R1(config)# ip prefix-list ALLOW-IN seq 5 permit 0.0.0.0/0 le 24    ! 只接受 /24 及以内
R1(config-router)# neighbor 10.0.12.2 prefix-list ALLOW-IN in

! ═══ AS-Path 过滤 ═══
R1(config)# ip as-path access-list 1 permit ^200$                   ! 只接受直连 AS200 的路由
R1(config-router)# neighbor 10.0.12.2 filter-list 1 in

! ═══ 安全：限制前缀数量（★ 防止对端误通告全表打爆你）═══
R1(config-router)# neighbor 10.0.12.2 maximum-prefix 1000 80 restart 5
!                                                    ↑    ↑         ↑
!                                                  上限 80%告警  5分钟后重连

! ═══ 软重置（★ 不断会话就应用新策略）═══
R1(config-router)# neighbor 10.0.12.2 soft-reconfiguration inbound   ! 缓存原始路由
R1# clear ip bgp 10.0.12.2 soft in
R1# clear ip bgp 10.0.12.2 soft out
R1# clear ip bgp * soft
! ⚠️ 不要用 clear ip bgp *（硬重置，会断开所有会话，全表重传）

! ═══ 查看命令 ═══
R1# show ip bgp summary                         ! ① 邻居状态（最常用）
R1# show ip bgp                                 ! ② BGP 表
R1# show ip bgp 200.1.1.0/24                    ! ③ 某条路由的详情（★ 看选路原因）
R1# show ip bgp neighbors 10.0.12.2
R1# show ip bgp neighbors 10.0.12.2 advertised-routes    ! 我发给它什么
R1# show ip bgp neighbors 10.0.12.2 routes               ! 它发给我什么（已过滤）
R1# show ip bgp neighbors 10.0.12.2 received-routes      ! 原始接收（需 soft-reconfiguration）
R1# show ip bgp regexp ^200_                    ! 正则匹配 AS-Path
R1# show ip bgp community 65001:100
R1# show ip route bgp
```

### `show ip bgp` 输出解读

```cisco
R1# show ip bgp
BGP table version is 15, local router ID is 1.1.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath
Origin codes: i - IGP, e - EGP, ? - incomplete

     Network          Next Hop         Metric LocPrf Weight Path
 *>  192.168.1.0/24   0.0.0.0               0         32768 i
 *>i 200.1.1.0/24     3.3.3.3               0    100      0 200 i
 *   200.1.1.0/24     10.0.13.3             0             0 500 600 i
 *   200.2.2.0/24     10.0.12.2             0             0 200 300 i
 ↑↑↑
 │││
 ││└─ i = 从 iBGP 学到的
 │└── > = best（被选为最优，会进路由表）
 └─── * = valid（下一跳可达，有效）
```

**关键标记**：

| 标记 | 含义 |
|:--|:--|
| **`*`** | valid，下一跳可达 |
| **`>`** | **best，被选为最优，会装入路由表** |
| **`i`**（第 3 列） | 从 iBGP 学到 |
| `s` | suppressed（被聚合抑制） |
| `r` | **RIB-failure**（BGP 选它为最优，但没能装进路由表——通常是有 AD 更低的路由） |
| `S` | Stale（GR 期间的陈旧路由） |
| `d` | damped（被阻尼抑制） |

> **只有 `*>` 的路由才会进路由表。** 只有 `*` 没有 `>` 说明它有效但不是最优。什么都没有说明下一跳不可达。

---

## ⑦ 配套实验：双运营商出口 + 策略控制

**拓扑**：
```
   AS 100 (电信)              AS 200 (联通)
        │                          │
    10.0.11.1                 10.0.22.1
        │ eBGP                     │ eBGP
    10.0.11.2                 10.0.22.2
   ┌────▼────┐               ┌─────▼───┐
   │   R1    │══════iBGP═════│   R2    │
   │ Lo0:1.1.1.1             │ Lo0:2.2.2.2
   └────┬────┘               └─────┬───┘
        └───────────┬──────────────┘
             内网 192.168.0.0/22
                 AS 65001
```

### Step 1：基础 BGP 配置

```cisco
! ── R1 ──
R1(config)# router bgp 65001
R1(config-router)# bgp router-id 1.1.1.1
R1(config-router)# bgp log-neighbor-changes

! eBGP 到电信
R1(config-router)# neighbor 10.0.11.1 remote-as 100
R1(config-router)# neighbor 10.0.11.1 description ### CHINA-TELECOM ###
R1(config-router)# neighbor 10.0.11.1 soft-reconfiguration inbound

! iBGP 到 R2（用 Loopback）
R1(config-router)# neighbor 2.2.2.2 remote-as 65001
R1(config-router)# neighbor 2.2.2.2 update-source Loopback0        ! ★
R1(config-router)# neighbor 2.2.2.2 next-hop-self                  ! ★

! 通告内网
R1(config-router)# network 192.168.0.0 mask 255.255.252.0

! ★ 必须：路由表里要有完全匹配的路由，否则 network 不生效
R1(config)# ip route 192.168.0.0 255.255.252.0 Null0
```

**R2 对称配置**（邻居是 AS 200，iBGP 对端是 1.1.1.1）。

> **`network` 命令的关键规则**：BGP 的 `network` **不像 OSPF 那样是"在哪些接口启用协议"**，而是"**把路由表里已有的这条路由注入 BGP**"。
>
> **路由表里必须有完全匹配（前缀+掩码都一致）的路由**，否则 `network` 不生效。所以常配一条指向 Null0 的静态路由来"造出"这条路由。

### Step 2：验证邻居

```cisco
R1# show ip bgp summary
BGP router identifier 1.1.1.1, local AS number 65001

Neighbor    V   AS  MsgRcvd MsgSent  TblVer InQ OutQ Up/Down  State/PfxRcd
2.2.2.2     4 65001     125     128       8   0    0 00:15:12         4
10.0.11.1   4   100    2451    2438       8   0    0 02:15:33      1245
                                                          ↑            ↑
                                                     已建立      收到 1245 条前缀
```

**如果看到 `Active`**，按第 ⑧ 节的排障流程处理。

### Step 3：验证 next-hop-self 的必要性（★ 重点实验）

**先故意不配 next-hop-self**：
```cisco
R1(config-router)# no neighbor 2.2.2.2 next-hop-self
R1# clear ip bgp 2.2.2.2 soft out
```

**在 R2 上看**：
```cisco
R2# show ip bgp 200.1.1.0
BGP routing table entry for 200.1.1.0/24
  Paths: (1 available, no best path)              ← 没有最优路径！
  100
    10.0.11.1 (inaccessible) from 2.2.2.2 (1.1.1.1)
              ↑↑↑↑↑↑↑↑↑↑↑↑
              下一跳不可达！

R2# show ip bgp | include 200.1.1.0
 *  i 200.1.1.0/24    10.0.11.1        0    100      0 100 i
 ↑
 只有 * 没有 >，不会进路由表
```

**加回 next-hop-self**：
```cisco
R1(config-router)# neighbor 2.2.2.2 next-hop-self
R1# clear ip bgp 2.2.2.2 soft out
```

```cisco
R2# show ip bgp | include 200.1.1.0
 *>i 200.1.1.0/24    1.1.1.1          0    100      0 100 i
 ↑↑                  ↑
 现在是 best        下一跳变成 R1 的 Loopback，IGP 里可达 ✓
```

**✅ 这个实验直观证明了 `next-hop-self` 的必要性。**

### Step 4：用 LOCAL_PREF 控制出向流量

**目标**：所有出向流量优先走电信（R1），电信断了才走联通（R2）。

```cisco
! 在 R1 上，把从电信学到的路由的 LOCAL_PREF 设高
R1(config)# route-map TELECOM-IN permit 10
R1(config-route-map)#  set local-preference 200

R1(config-router)# neighbor 10.0.11.1 route-map TELECOM-IN in
R1# clear ip bgp 10.0.11.1 soft in
```

**验证（在 R2 上看，因为 LOCAL_PREF 会通过 iBGP 传播）**：
```cisco
R2# show ip bgp 200.1.1.0
BGP routing table entry for 200.1.1.0/24
  Paths: (2 available, best #1)
  100                                              ← 经 R1（电信）
    1.1.1.1 (metric 20) from 1.1.1.1 (1.1.1.1)
      Origin IGP, localpref 200, valid, internal, best
                            ↑                        ↑
                       LOCAL_PREF 200            被选中 ✓
  200 300                                          ← 经 R2 自己（联通）
    10.0.22.1 from 10.0.22.1 (22.22.22.22)
      Origin IGP, localpref 100, valid, external
                            ↑
                       默认 100，输了
```

**✅ 连 R2 自己的 eBGP 路径都输给了 R1 的路径**——这就是 LOCAL_PREF 的威力（它在第 2 步，比"eBGP 优于 iBGP"的第 7 步靠前得多）。

**验证故障切换**：
```cisco
R1(config)# interface GigabitEthernet0/1
R1(config-if)# shutdown

R2# show ip bgp 200.1.1.0
  200 300
    10.0.22.1 from 10.0.22.1 ...
      Origin IGP, localpref 100, valid, external, best    ← 自动切到联通 ✓
```

### Step 5：用 AS-Path Prepending 控制入向流量

**目标**：让外部访问我们的流量优先从电信进来。

**做法：向联通通告时，把 AS-Path 加长。**

```cisco
! 在 R2（连联通的那台）上配
R2(config)# route-map PREPEND-TO-UNICOM permit 10
R2(config-route-map)#  set as-path prepend 65001 65001 65001

R2(config-router)# neighbor 10.0.22.1 route-map PREPEND-TO-UNICOM out
R2# clear ip bgp 10.0.22.1 soft out
```

**在联通侧（AS 200）看到的**：
```
经电信路径：  100 65001            ← AS-Path 长度 2
经联通路径：  65001 65001 65001 65001   ← AS-Path 长度 4（被 prepend 了 3 次）
```
**→ 外部会优先选择经电信的路径 ✓**

**验证**：
```cisco
R2# show ip bgp neighbors 10.0.22.1 advertised-routes
   Network          Next Hop     Metric LocPrf Weight Path
*> 192.168.0.0/22   10.0.22.2         0         32768 65001 65001 65001 i
                                                       ↑ 确认 prepend 生效
```

> **为什么用 AS-Path Prepending 而不是 MED**：
> - **MED 在选路第 7 步**，很容易被对方的 LOCAL_PREF（第 2 步）覆盖
> - **AS-Path 在第 4 步**，影响力大得多
> - MED 默认只在"同一个邻居 AS 的多条路径"之间比较，跨 AS 场景往往不生效
>
> **入向流量控制的可靠性排序**：
> 1. **只向一个运营商通告该网段**（最彻底，但失去冗余）
> 2. **AS-Path Prepending**（最常用）
> 3. **Community**（如果运营商支持，比如很多运营商提供"设置这个 community 我就降低你的 LOCAL_PREF"）
> 4. MED（最不可靠）

### Step 6：路由聚合与 Null0 防环

```cisco
R1(config-router)# aggregate-address 192.168.0.0 255.255.252.0 summary-only
```

**验证**：
```cisco
R1# show ip bgp
     Network          Next Hop      Metric LocPrf Weight Path
 *>  192.168.0.0/22   0.0.0.0                        32768 i
 s>  192.168.0.0/24   0.0.0.0            0           32768 i     ← s = suppressed
 s>  192.168.1.0/24   0.0.0.0            0           32768 i
 s>  192.168.2.0/24   0.0.0.0            0           32768 i

R1# show ip route | include Null0
B    192.168.0.0/22 [200/0] via 0.0.0.0, 00:02:15, Null0
     ↑ BGP 自动生成的防环路由
```

**为什么需要 Null0**：如果收到目的地在 `192.168.3.0/24`（在聚合范围内但实际不存在）的流量，Null0 会直接丢弃。否则包会匹配默认路由送回上游，上游又匹配聚合路由送回来 → **来回弹**。

### Step 7：安全加固

```cisco
! ── 限制接收的前缀数量（★ 极其重要）──
R1(config-router)# neighbor 10.0.11.1 maximum-prefix 1000 80 restart 5

! ── 只接受合理的前缀 ──
R1(config)# ip prefix-list SANE-IN seq 5 deny 0.0.0.0/0                    ! 拒默认路由
R1(config)# ip prefix-list SANE-IN seq 10 deny 10.0.0.0/8 le 32            ! 拒私网
R1(config)# ip prefix-list SANE-IN seq 15 deny 172.16.0.0/12 le 32
R1(config)# ip prefix-list SANE-IN seq 20 deny 192.168.0.0/16 le 32
R1(config)# ip prefix-list SANE-IN seq 25 deny 0.0.0.0/0 ge 25             ! 拒 /25 以上的碎片
R1(config)# ip prefix-list SANE-IN seq 30 permit 0.0.0.0/0 le 24
R1(config-router)# neighbor 10.0.11.1 prefix-list SANE-IN in

! ── 只通告自己的网段（防止变成中转 AS）──
R1(config)# ip prefix-list MY-NETS-OUT seq 5 permit 192.168.0.0/22
R1(config-router)# neighbor 10.0.11.1 prefix-list MY-NETS-OUT out

! ── MD5 认证 ──
R1(config-router)# neighbor 10.0.11.1 password MyBgpSecret

! ── TTL 安全（防远程伪造）──
R1(config-router)# neighbor 10.0.11.1 ttl-security hops 1
```

> **`maximum-prefix` 是必配项。** 如果运营商配置失误向你通告了全表（90 万条），你的路由器内存会瞬间耗尽并崩溃。这在实际中发生过多次。
>
> **只通告自己的网段** 同样关键——否则如果你接了两个运营商，可能变成它们之间的"免费中转"，你的出口带宽会被别人的流量占满。这叫 **transit AS 泄露**，历史上造成过多次大规模互联网故障。

---

## ⑧ 排障速查表

| 症状 | 状态 | 怀疑点 | 验证 |
|:--|:--|:--|:--|
| 邻居 **Idle** | TCP 建不起来 | 路由不通、邻居地址写错 | `ping <邻居IP>`、`show ip route <邻居>` |
| 邻居 **Active** | TCP 建立失败重试 | **TCP 179 被拦**、单向路由、源地址不对 | `telnet <邻居IP> 179`、`show access-lists` |
| 邻居 **Idle (Admin)** | 被 shutdown | — | `no neighbor X shutdown` |
| 卡 **OpenSent** | AS 号/Router ID | `debug ip bgp` | 核对 `remote-as`、Router ID 是否冲突 |
| **iBGP 学到路由但不用** | **NEXT_HOP 不可达** | `show ip bgp X` 看 `inaccessible` | **配 `next-hop-self`** |
| 用 Loopback 建 iBGP 起不来 | 缺 `update-source` | `show run \| section router bgp` | 加 `update-source Loopback0` |
| 用 Loopback 建 eBGP 起不来 | 缺 `ebgp-multihop` | 同上 | 加 `ebgp-multihop 2` |
| `network` 不生效 | **路由表无完全匹配的路由** | `show ip route <该网段>` | 加 Null0 静态路由 |
| 路由有 `*` 无 `>` | 不是最优 | `show ip bgp X` 看各属性 | 对照选路 11 步分析 |
| 路由有 `r` (RIB-failure) | 有 AD 更低的路由 | `show ip route X` | 检查是否有静态/IGP 路由 |
| 策略改了不生效 | 没做软重置 | — | `clear ip bgp X soft in/out` |
| 收不到某些路由 | 被过滤 | `show ip bgp neighbors X received-routes` | 需先配 `soft-reconfiguration inbound` |
| 邻居频繁震荡 | 链路/CPU/前缀超限 | `show logging`、`show ip bgp summary` | 查 maximum-prefix 是否触发 |
| 路由收敛慢 | BGP 扫描周期 | — | 配 `bgp fast-external-fallover`、BFD |

### BGP 邻居建不起来的标准排查

```
① 能 ping 通邻居吗？
   R1# ping 10.0.12.2
   R1# ping 10.0.12.2 source Loopback0       ← 用实际的源地址测！
   
② TCP 179 通吗？（关键）
   R1# telnet 10.0.12.2 179
   → "Open" = 通 ✓
   → "Connection refused" = 对端没跑 BGP 或拒绝
   → 超时 = 被 ACL/防火墙拦
   
③ 配置对吗？
   R1# show run | section router bgp
   - remote-as 写对了吗？
   - update-source 配了吗？（用 Loopback 时必须）
   - ebgp-multihop 配了吗？（eBGP 用 Loopback 时必须）
   
④ 双方 AS 号一致吗？
   R1# show ip bgp summary          ← 看 AS 列
   
⑤ 认证一致吗？
   R1# debug ip bgp                 ← 会显示认证失败
   
⑥ 对端也配了我吗？
   （BGP 是双向配置的，两端都要有 neighbor 语句）
```

### 软重置 vs 硬重置

```cisco
! ❌ 硬重置：断开会话，重新建立，全表重传
R1# clear ip bgp *
R1# clear ip bgp 10.0.12.2
! 后果：业务中断几十秒到几分钟（全表 90 万条要重传！）

! ✅ 软重置：不断会话，只重新应用策略
R1# clear ip bgp 10.0.12.2 soft in
R1# clear ip bgp 10.0.12.2 soft out
R1# clear ip bgp * soft
```

**软重置的两个前提**：
1. **出方向（soft out）**：任何时候都可以，路由器本来就保存着要发的路由
2. **入方向（soft in）**：需要以下之一
   - 配了 `neighbor X soft-reconfiguration inbound`（缓存对端发来的原始路由，**占内存**）
   - 或双方都支持 **Route Refresh 能力**（现代设备默认支持，**推荐，不占内存**）

```cisco
R1# show ip bgp neighbors 10.0.12.2 | include Route refresh
  Route refresh: advertised and received(new)      ← 支持，可以直接 soft in
```

> **生产环境永远不要用 `clear ip bgp *`。** 这条命令会断开所有 BGP 会话，如果你有全表，重建过程可能需要几分钟，期间业务全断。这是运维事故排行榜上的常客。

---

## ⑨ 三厂商对照 + 考点自测

### 三厂商对照

| 目的 | Cisco | H3C | 华为 |
|:--|:--|:--|:--|
| 启动 BGP | `router bgp 65001` | `bgp 65001` | `bgp 65001` |
| Router ID | `bgp router-id 1.1.1.1` | `router-id 1.1.1.1` | `router-id 1.1.1.1` |
| 邻居 | `neighbor 1.1.1.1 remote-as 100` | `peer 1.1.1.1 as-number 100` | `peer 1.1.1.1 as-number 100` |
| 更新源 | `neighbor X update-source Lo0` | `peer X connect-interface Lo0` | `peer X connect-interface Lo0` |
| next-hop-self | `neighbor X next-hop-self` | `peer X next-hop-local` | `peer X next-hop-local` |
| eBGP 多跳 | `neighbor X ebgp-multihop 2` | `peer X ebgp-max-hop 2` | `peer X ebgp-max-hop 2` |
| RR 客户端 | `neighbor X route-reflector-client` | `peer X reflect-client` | `peer X reflect-client` |
| 通告网络 | `network 192.168.0.0 mask 255.255.252.0` | `network 192.168.0.0 22` | `network 192.168.0.0 22` |
| 聚合 | `aggregate-address X Y summary-only` | `aggregate X Y detail-suppressed` | `aggregate X Y detail-suppressed` |
| 应用策略 | `neighbor X route-map RM in` | `peer X route-policy RP import` | `peer X route-policy RP import` |
| 前缀过滤 | `neighbor X prefix-list PL in` | `peer X ip-prefix PL import` | `peer X ip-prefix PL import` |
| 查邻居 | `show ip bgp summary` | `display bgp peer` | `display bgp peer` |
| 查 BGP 表 | `show ip bgp` | `display bgp routing-table` | `display bgp routing-table` |
| 软重置 | `clear ip bgp X soft in` | `refresh bgp X import` | `refresh bgp X import` |

> **术语差异**：
> - Cisco 的 **route-map** ↔ H3C/华为的 **route-policy**
> - Cisco 的 **in/out** ↔ H3C/华为的 **import/export**
> - Cisco 的 **next-hop-self** ↔ H3C/华为的 **next-hop-local**

### 考点

- **BGP 选路 11 步**（★★★ 必背前 6-8 步）
- **eBGP vs iBGP** 的 AD、TTL、下一跳、水平分割规则
- **iBGP 需要全互联，用 RR 解决**
- **`next-hop-self` 的必要性**
- **`network` 命令要求路由表有完全匹配的路由**
- **Active 状态 = TCP 建不起来**（反直觉）
- **LOCAL_PREF 控出向，AS-Path Prepending 控入向**
- **软重置 vs 硬重置**
- **`maximum-prefix` 的重要性**

### 自测题

**1.** BGP 选路顺序的前 6 步是什么？

<details><summary>答案</summary>

**（第 0 步：下一跳必须可达，不可达直接淘汰）**

| 步 | 属性 | 规则 |
|:--|:--|:--|
| **1** | **WEIGHT** | **越大越优**（Cisco 私有，只在本机有效，默认 0） |
| **2** | **LOCAL_PREF** | **越大越优**（AS 内部传递，默认 100） |
| **3** | **本地产生的路由** | 自己用 `network` / `aggregate` 产生的优先 |
| **4** | **AS_PATH 长度** | **越短越优** |
| **5** | **ORIGIN** | **`i` (IGP) > `e` (EGP) > `?` (incomplete)** |
| **6** | **MED** | **越小越优**（默认 0，只在同一邻居 AS 的路径间比较） |

**后续步骤**：
```
7. eBGP > iBGP
8. 到下一跳的 IGP metric 越小越优
9. 更老的 eBGP 路径（稳定性优先）
10. Router ID 小的
11. 邻居 IP 小的
```

**记忆口诀**：
```
We Love Oranges As Apples, Mmm, Eat In Rome
 W   L      O    A     A    M    E   I   R
Weight, Local_pref, Originate, AS_path, Age(Origin), MED, eBGP, IGP, Router-id
```

**实战中最重要的三个**：

| 目标 | 用什么 | 为什么 |
|:--|:--|:--|
| **控制出向流量** | **LOCAL_PREF**（第 2 步） | 优先级高，且在整个 AS 内传播 |
| **控制入向流量** | **AS_PATH Prepending**（第 4 步） | 比 MED（第 7 步）可靠得多 |
| **只影响本机** | **WEIGHT**（第 1 步） | 不传播，不影响别人 |

**为什么 MED 不可靠**：它在第 6 步，很容易被对方的 LOCAL_PREF（第 2 步）覆盖。你说"从这边进来更好"，对方说"我不管，我就要从那边出去"——LOCAL_PREF 赢。

**验证选路原因**：
```cisco
R1# show ip bgp 200.1.1.0/24
  200 300
    10.0.12.2 from 10.0.12.2 (2.2.2.2)
      Origin IGP, metric 0, localpref 200, valid, external, best
                                    ↑                          ↑
                            关注这些属性                  谁赢了
```
**逐条对照 11 步，就能知道它为什么赢。**
</details>

**2.** 为什么 iBGP 邻居需要全互联？有什么替代方案？

<details><summary>答案</summary>

**因为 iBGP 有"水平分割"规则：从一个 iBGP 邻居学到的路由，不能通告给另一个 iBGP 邻居。**

**为什么有这条规则**：

BGP 用 **AS_PATH** 防环——收到 AS_PATH 里有自己 AS 号的路由就丢弃。

但**在同一个 AS 内部传递时，AS_PATH 不会增加**。所以 iBGP 之间无法用 AS_PATH 检测环路。

于是只能用一条更粗暴的规则防环：**iBGP 学到的路由不再往其他 iBGP 邻居传**。

**后果：必须全互联（full mesh）**

```
n 台路由器需要 n(n-1)/2 条 iBGP 会话：

  4 台 → 6 条
 10 台 → 45 条
 50 台 → ★ 1225 条 ★
100 台 → 4950 条
```

不仅配置量爆炸，每条会话还要消耗内存和 CPU（各自维护 TCP 连接和路由表）。**完全不可维护。**

**替代方案 1：Route Reflector (RR) ★ 主流**

```
              ┌──────────┐
              │    RR    │
              └─┬──┬──┬──┘
          ┌─────┘  │  └─────┐
       ┌──▼──┐  ┌──▼──┐  ┌──▼──┐
       │ RC1 │  │ RC2 │  │ RC3 │
       └─────┘  └─────┘  └─────┘
       
   n 台只需 n-1 条会话（而不是 n(n-1)/2）
```

**RR 打破水平分割**：

| 路由来源 | 反射给谁 |
|:--|:--|
| 从 **Client** 学到 | **所有其他 Client + 所有非 Client** |
| 从**非 Client** 学到 | **只给 Client** |
| 从 **eBGP** 学到 | 所有邻居 |

**RR 的防环机制**（因为打破了水平分割，需要新的防环手段）：
- **`ORIGINATOR_ID`**：记录路由的最初产生者。收到 ORIGINATOR_ID 是自己的 → 丢弃
- **`CLUSTER_LIST`**：记录经过的 RR 集群 ID。RR 收到含自己 Cluster-ID 的 → 丢弃

```cisco
! RR 上配置（Client 不需要任何特殊配置）
RR(config-router)# neighbor 10.0.0.2 route-reflector-client
RR(config-router)# neighbor 10.0.0.3 route-reflector-client
RR(config-router)# bgp cluster-id 1.1.1.1
```

**RR 的冗余设计**：
```
   部署两台 RR，配相同的 cluster-id
   每个 Client 同时连两台 RR
   → 单台 RR 故障不影响路由传播
```

**替代方案 2：Confederation（联邦）**

把大 AS 分成多个"子 AS"（用私有 AS 号），子 AS 之间跑 eBGP，对外统一显示为一个 AS 号。

```cisco
R1(config)# router bgp 65001                    ! 子 AS
R1(config-router)# bgp confederation identifier 100      ! 对外的 AS 号
R1(config-router)# bgp confederation peers 65002 65003
```

**RR vs Confederation**：

| | **RR** | **Confederation** |
|:--|:--|:--|
| 部署难度 | ★ 简单（Client 无需配置） | 复杂（要重新规划 AS） |
| 渐进式部署 | ✅ 容易 | ❌ 难 |
| 实际使用 | **★ 绝大多数场景** | 极少 |

**结论：用 RR。** Confederation 主要是考点，实际部署中很少见。
</details>

**3.** iBGP 邻居学到了路由，但路由表里没有，`show ip bgp` 显示 `inaccessible`。为什么？

<details><summary>答案</summary>

**NEXT_HOP 不可达 —— 因为 iBGP 通告路由时不改变 NEXT_HOP。**

**故障过程**：
```
   AS 200                     AS 65001
   [R2] ──eBGP── [R1] ──iBGP── [R3]
  10.0.12.2     10.0.12.1

① R1 从 R2（eBGP）学到 200.1.1.0/24，NEXT_HOP = 10.0.12.2
② R1 通过 iBGP 传给 R3 时，★ NEXT_HOP 保持 10.0.12.2 不变 ★
③ R3 的路由表里没有到 10.0.12.2 的路由
   （因为 10.0.12.0/30 是 AS 200 和 AS 65001 之间的互联段，
     通常不会通告进内部 IGP）
④ → NEXT_HOP 不可达 → 路由无效，不进路由表
```

**验证**：
```cisco
R3# show ip bgp 200.1.1.0
BGP routing table entry for 200.1.1.0/24
  Paths: (1 available, no best path)                 ← 没有最优路径
  200
    10.0.12.2 (inaccessible) from 1.1.1.1 (1.1.1.1)
              ↑↑↑↑↑↑↑↑↑↑↑↑ 找到问题

R3# show ip bgp | include 200.1.1.0
 *  i 200.1.1.0/24    10.0.12.2   ...
 ↑
 只有 * 没有 >
```

**解决方案（三选一，推荐第 1 个）**：

**① `next-hop-self`（★ 最常用）**
```cisco
R1(config)# router bgp 65001
R1(config-router)# neighbor 3.3.3.3 next-hop-self
```
R1 在向 iBGP 邻居通告时，**把 NEXT_HOP 改成自己的地址**（通常是 Loopback，IGP 里可达）。

**② 把互联网段通告进 IGP**
```cisco
R1(config)# router ospf 1
R1(config-router)# network 10.0.12.0 0.0.0.3 area 0
```
❌ **不推荐**：会把运营商的互联地址引入内部 IGP，污染路由表，而且如果有几十个 eBGP 邻居，就要引入几十个网段。

**③ 用 route-map 改写**
```cisco
R1(config)# route-map SET-NH permit 10
R1(config-route-map)#  set ip next-hop 1.1.1.1
R1(config-router)# neighbor 3.3.3.3 route-map SET-NH out
```
更灵活但更复杂。

**最佳实践模板（iBGP 必配三件套）**：
```cisco
R1(config-router)# neighbor 3.3.3.3 remote-as 65001
R1(config-router)# neighbor 3.3.3.3 update-source Loopback0      ! ① 用 Loopback
R1(config-router)# neighbor 3.3.3.3 next-hop-self                ! ② 改下一跳
R1(config-router)# neighbor 3.3.3.3 soft-reconfiguration inbound  ! ③ 便于排障
```

**为什么用 Loopback 建 iBGP**：
- Loopback 永远 up，**物理链路故障不会导致 BGP 会话断开**（只要还有其他路径可达）
- 有多条路径时，BGP 会话保持稳定
- 配合 `next-hop-self`，下一跳指向 Loopback，IGP 里必然可达

> **`next-hop-self` 是 iBGP 排障 Top 1 的问题。** 症状很典型：`show ip bgp` 里能看到路由，但没有 `>` 标记，路由表里也没有。看到 `inaccessible` 就是它。
</details>

**4.** BGP 邻居状态一直是 `Active`，什么原因？

<details><summary>答案</summary>

**⚠️ `Active` 不是"活跃正常"，而是"TCP 连接建立失败，正在重试"。**

这是 BGP 最容易被误解的状态（和 EIGRP 的 Active 一样反直觉）。

**BGP 状态机**：
```
Idle → Connect → Active → OpenSent → OpenConfirm → Established
                   ↑                                    ↑
             TCP 建不起来                            ✅ 正常
```

**排查顺序**：

**① 邻居的 IP 可达吗？**
```cisco
R1# ping 10.0.12.2
R1# ping 10.0.12.2 source Loopback0        ← ★ 用实际的源地址测！
R1# show ip route 10.0.12.2
```
**注意**：如果配了 `update-source Loopback0`，BGP 会用 Loopback 作为源地址。你必须用同样的源测试，否则测出来的结果不代表实际情况。

**② TCP 179 端口通吗？（最关键的一步）**
```cisco
R1# telnet 10.0.12.2 179
Trying 10.0.12.2, 179 ... Open           ← ✅ 通
```
| 结果 | 含义 |
|:--|:--|
| **Open** | TCP 通，问题在 BGP 配置层面 |
| **Connection refused** | 收到 RST——对端没跑 BGP，或明确拒绝了你 |
| **超时** | **被 ACL / 防火墙静默丢弃** |

**③ 是不是用 Loopback 建邻居但忘了 `update-source`？**
```cisco
R1(config-router)# neighbor 2.2.2.2 update-source Loopback0
```
不配的话，R1 用出接口 IP 发起连接，但对端配的 neighbor 是 R1 的 Loopback → **源地址不匹配，对端拒绝**。

**④ eBGP 用 Loopback 建邻居但忘了 `ebgp-multihop`？**
```cisco
R1(config-router)# neighbor 2.2.2.2 ebgp-multihop 2
```
**eBGP 的默认 TTL 是 1**，只能直连。用 Loopback 建邻居时通常要跨一跳，TTL 不够，包在路上就被丢了。

**⑤ 单向路由？**
```cisco
! 在 R1 上
R1# ping 10.0.12.2 source Loopback0        ← 通

! 在 R2 上（★ 必须双向都测）
R2# ping 1.1.1.1 source Loopback0          ← 不通！找到问题
```
**TCP 需要双向可达**。R1 能到 R2 但 R2 回不来，TCP 三次握手完成不了。

**⑥ ACL 拦了 TCP 179？**
```cisco
R1# show access-lists | include 179|bgp
```

**⑦ 认证不匹配？**
```cisco
R1# debug ip bgp
%TCP-6-BADAUTH: Invalid MD5 digest from 10.0.12.2:179 to 1.1.1.1:11000
```

**⑧ 对端也配了我吗？**
BGP 是**双向配置**的，两端都必须有对方的 `neighbor` 语句。只有一边配了，永远建不起来。

**其他状态的含义**：

| 状态 | 含义 |
|:--|:--|
| **Idle** | TCP 都没开始尝试——路由不通、邻居地址完全写错 |
| **Idle (Admin)** | 被手工 `neighbor X shutdown` 了 |
| **Active** | TCP 建立失败，正在重试 |
| **OpenSent** | 已发 OPEN，等对方响应——**AS 号不匹配**、Router ID 冲突 |
| **OpenConfirm** | 短暂状态，正常情况下很快到 Established |
| **Established** | ✅ 正常 |

**日志排查**：
```cisco
R1(config)# router bgp 65001
R1(config-router)# bgp log-neighbor-changes       ! 建议默认开启
R1# show logging | include BGP
%BGP-5-ADJCHANGE: neighbor 10.0.12.2 Down BGP Notification sent
%BGP-3-NOTIFICATION: sent to neighbor 10.0.12.2 2/2 (peer in wrong AS) 2 bytes 00C8
                                                    ↑ AS 号写错了
```
</details>

**5.** 你要让公司的出向流量优先走电信，同时让外部访问你的流量也优先从电信进来。分别怎么做？

<details><summary>答案</summary>

**这是两个方向的问题，用两种不同的手段。**

## 出向流量（我方发出的）→ 用 **LOCAL_PREF**

```cisco
! 在连电信的路由器上，把学到的路由 LOCAL_PREF 调高
R1(config)# route-map TELECOM-IN permit 10
R1(config-route-map)#  set local-preference 200

R1(config)# router bgp 65001
R1(config-router)# neighbor <电信邻居IP> route-map TELECOM-IN in
R1# clear ip bgp <电信邻居IP> soft in
```

**为什么有效**：
- LOCAL_PREF 在选路**第 2 步**，优先级极高
- 它**通过 iBGP 在整个 AS 内传播**，所以连接联通的路由器 R2 也会知道"经 R1 走电信的路径 LOCAL_PREF=200，比我自己的 100 高"，从而把流量转给 R1
- **默认值 100，设成 200 即可**

**验证**：
```cisco
R2# show ip bgp 200.1.1.0
  100                                          ← 经 R1（电信）
    1.1.1.1 (metric 20) from 1.1.1.1
      Origin IGP, localpref 200, valid, internal, best     ← 赢 ✓
  200 300                                      ← R2 自己的 eBGP（联通）
    10.0.22.1 from 10.0.22.1
      Origin IGP, localpref 100, valid, external
```
注意：**LOCAL_PREF 200 的 iBGP 路径赢过了 LOCAL_PREF 100 的 eBGP 路径**——因为 LOCAL_PREF 在第 2 步，"eBGP 优于 iBGP"在第 7 步。

## 入向流量（外部访问我的）→ 用 **AS_PATH Prepending**

```cisco
! 在连【联通】的路由器上，向联通通告时把 AS-Path 加长
R2(config)# route-map PREPEND-TO-UNICOM permit 10
R2(config-route-map)#  set as-path prepend 65001 65001 65001

R2(config)# router bgp 65001
R2(config-router)# neighbor <联通邻居IP> route-map PREPEND-TO-UNICOM out
R2# clear ip bgp <联通邻居IP> soft out
```

**效果**：
```
外部看到的两条路径：
  经电信：  100 65001                        ← AS-Path 长度 2
  经联通：  200 65001 65001 65001 65001      ← AS-Path 长度 5

→ AS-Path 短的赢，外部优先从电信进来 ✓
```

**为什么不用 MED**：

| | AS_PATH Prepending | MED |
|:--|:--|:--|
| 选路步骤 | **第 4 步** | 第 6 步 |
| 被覆盖的可能 | 低 | **高**（对方一个 LOCAL_PREF 就压过去了） |
| 跨 AS 有效性 | ✅ 所有 AS 都看得到 | ⚠️ 默认只在同一邻居 AS 的路径间比较 |
| 实际效果 | **可靠** | 经常不生效 |

**入向控制手段的可靠性排序**：

| 方法 | 可靠性 | 代价 |
|:--|:--|:--|
| **只向一个运营商通告该网段** | ★★★★★ 最彻底 | 失去该网段的冗余 |
| **AS_PATH Prepending** | ★★★★ | 无 |
| **Community**（运营商提供的） | ★★★★ | 需运营商支持 |
| **MED** | ★★ | 无 |
| **更精细的前缀**（比如通告 /25 给电信，/24 给联通） | ★★★★★ | 很多运营商过滤 /24 以上的前缀 |

**运营商 Community 方案**（很多大运营商提供）：
```cisco
! 运营商文档里会写：设置 community 200:120 = 把 LOCAL_PREF 降到 120
R2(config)# route-map LOWER-PREF permit 10
R2(config-route-map)#  set community 200:120
R2(config-router)# neighbor <联通> route-map LOWER-PREF out
R2(config-router)# neighbor <联通> send-community          ! ★ 必须
```
这比 Prepending 更精确——它直接操作了对方的 LOCAL_PREF（第 2 步），效果最好。**部署前查阅运营商的 BGP Community 文档。**

## 完整方案总结

```cisco
! ═══ R1（连电信）═══
! 出向：抬高从电信学到的路由的 LOCAL_PREF
route-map TELECOM-IN permit 10
 set local-preference 200
router bgp 65001
 neighbor <电信> route-map TELECOM-IN in
 ! 向电信正常通告（不做 prepend）

! ═══ R2（连联通）═══
! 出向：保持默认 LOCAL_PREF 100（自然成为备份）
! 入向：向联通通告时 prepend
route-map PREPEND-TO-UNICOM permit 10
 set as-path prepend 65001 65001 65001
router bgp 65001
 neighbor <联通> route-map PREPEND-TO-UNICOM out
```

**⚠️ 两个必须注意的点**：

1. **两条链路都要保持通告**（不要为了控制流量就只通告一边），否则失去冗余——这违背了接双运营商的初衷。

2. **验证故障切换**：
```cisco
! 断掉电信链路
R1(config)# interface Gi0/1
R1(config-if)# shutdown

! 确认流量自动切到联通
R2# show ip bgp 200.1.1.0        ← 应该变成 best
! 从外部 ping 你的公网 IP，确认依然可达
```

**很多人配完策略就不测故障切换**，结果真出故障时发现根本切不过去——因为 prepend 加太多导致外部完全不选联通，或者某条 route-map 缺了兜底的 `permit` 语句把路由全过滤掉了。
</details>

---

**上一章** ← [07 OSPF 进阶](07-OSPF进阶.md) ｜ **下一章** → [09 组播 Multicast](09-组播Multicast.md)
