# 03 · BGP 深入与排障

> 基础见 [ENCOR 第 8 章](../02-ENCOR-350-401/08-BGP.md)。本章聚焦**排障、策略操控和大规模部署**。

## ① BGP 排障黄金流程

```
   ① show ip bgp summary
      邻居状态是 Established 吗？
        ├─ Idle    → TCP 都没开始（路由不通/地址写错）
        ├─ Active  → ★ TCP 建立失败 ★（179 被拦/源地址不对/单向路由）
        ├─ OpenSent→ AS 号不匹配 / Router ID 冲突
        └─ 数字    → ✅ 已建立，收到 N 条前缀
        ↓
   ② show ip bgp <前缀>
      ★ 这是分界线 ★
      BGP 表里有这条路由吗？
        ├─ 没有 → 问题在【上游】（对方没发/被 in 方向策略过滤）
        └─ 有   → 走 ③
        ↓
   ③ 看标记
        ├─ 没有 * → ★ 下一跳不可达 ★（next-hop-self？）
        ├─ 有 * 没 > → 不是最优（对照选路 11 步分析）
        ├─ 有 r  → ★ RIB-failure ★（有 AD 更低的路由）
        └─ 有 *> → 应该在路由表里
        ↓
   ④ show ip route <前缀>
      真的进路由表了吗？
        ↓
   ⑤ show ip bgp neighbors X advertised-routes / routes
      我发给它什么？它发给我什么？
```

> **★ 第 ② 步的 BGP 表是分界线**（和 OSPF 的 LSDB 一样的道理）：BGP 表是"我收到了什么"，路由表是"我采纳了什么"。

---

## ② 邻居问题深入

### 2.1 状态机与卡点

| 状态 | 含义 | 卡住的原因 |
|:--|:--|:--|
| **Idle** | 初始/出错，未尝试连接 | 路由不通、邻居地址完全写错、被 `shutdown` |
| **Connect** | 正在建 TCP | 短暂状态 |
| **Active** | ⚠️ **TCP 建立失败，正在重试** | **TCP 179 被拦**、源地址不对、单向路由、`ebgp-multihop` 缺失 |
| **OpenSent** | 已发 OPEN，等对方 | **AS 号不匹配**、Router ID 冲突、版本不兼容 |
| **OpenConfirm** | 等 Keepalive | 短暂状态 |
| **Established** | ✅ 正常 | — |

> **`Active` 是最容易误解的状态**。它不是"活跃正常"，而是"正在尝试重连"。**看到 Active 就是有问题。**

### 2.2 完整排查流程

```cisco
! ── ① 能 ping 通吗（★ 用正确的源地址）──
R1# ping 10.0.12.2
R1# ping 2.2.2.2 source Loopback0            ! 如果配了 update-source

! ── ② TCP 179 通吗（★ 最关键的一步）──
R1# telnet 10.0.12.2 179
Trying 10.0.12.2, 179 ... Open               ← ✅ 通
! "Connection refused" → 对端没跑 BGP 或明确拒绝
! 超时              → ★ 被 ACL/防火墙静默丢弃 ★

! ── ③ 配置检查 ──
R1# show run | section router bgp
! · remote-as 对吗？
! · update-source 配了吗？（用 Loopback 时必须）
! · ebgp-multihop 配了吗？（eBGP 用 Loopback 时必须）
! · 是不是被 shutdown 了？

! ── ④ 双方 AS 号 ──
R1# show ip bgp summary                      ! 看 AS 列

! ── ⑤ 认证 ──
R1# debug ip bgp
%TCP-6-BADAUTH: Invalid MD5 digest from 10.0.12.2:179

! ── ⑥ 对端也配了我吗 ──
! BGP 是双向配置，两端都要有 neighbor 语句
```

### 2.3 常见配置陷阱

**陷阱 1：用 Loopback 建 iBGP 但忘了 `update-source`**
```cisco
! ❌ 错误
R1(config-router)# neighbor 2.2.2.2 remote-as 65001
! R1 用出接口 IP 发起连接，但 R2 配的 neighbor 是 R1 的 Loopback
! → 源地址不匹配 → R2 拒绝 → Active

! ✅ 正确
R1(config-router)# neighbor 2.2.2.2 remote-as 65001
R1(config-router)# neighbor 2.2.2.2 update-source Loopback0
```

**陷阱 2：eBGP 用 Loopback 但忘了 `ebgp-multihop`**
```cisco
! eBGP 的默认 TTL 是 1，只能直连
! 用 Loopback 建邻居通常要跨一跳 → TTL 不够
R1(config-router)# neighbor 2.2.2.2 ebgp-multihop 2
R1(config-router)# neighbor 2.2.2.2 update-source Loopback0

! 更安全的写法（GTSM，防远程伪造）
R1(config-router)# neighbor 2.2.2.2 ttl-security hops 2
```

> **`ebgp-multihop` vs `ttl-security`**：
> - `ebgp-multihop 2` = **发送** TTL=2 的包
> - `ttl-security hops 2` = **只接受** TTL ≥ 254 的包（意味着来源最多 2 跳）
>
> **`ttl-security` 更安全**——它能防止远程攻击者伪造 BGP 报文（因为远程的包 TTL 会很低）。**两者不能同时配。**

**陷阱 3：`network` 命令不生效**
```cisco
R1(config-router)# network 192.168.0.0 mask 255.255.252.0
```
**BGP 的 `network` 命令要求路由表里有【完全匹配】的路由**（前缀和掩码都一致）。

```cisco
R1# show ip route 192.168.0.0 255.255.252.0
% Network not in table                        ← ★ 所以 network 不生效
```

**解法**：
```cisco
R1(config)# ip route 192.168.0.0 255.255.252.0 Null0    ! 造一条静态路由
```

**验证**：
```cisco
R1# show ip bgp
     Network          Next Hop     Metric LocPrf Weight Path
 *>  192.168.0.0/22   0.0.0.0           0         32768 i    ← 现在有了 ✓
```

---

## ③ 路由标记详解（★ 排障核心）

```cisco
R1# show ip bgp
BGP table version is 15, local router ID is 1.1.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete

     Network          Next Hop         Metric LocPrf Weight Path
 *>  192.168.1.0/24   0.0.0.0               0         32768 i
 *>i 200.1.1.0/24     3.3.3.3               0    100      0 200 i
 *   200.1.1.0/24     10.0.13.3             0             0 500 600 i
 r>  10.1.1.0/24      10.0.12.2             0             0 200 i
 s>  192.168.1.0/24   0.0.0.0               0         32768 i
     200.2.2.0/24     10.0.99.9             0             0 300 i
```

| 标记 | 含义 | 排障意义 |
|:--|:--|:--|
| **`*`** | valid（**下一跳可达**） | **没有 `*` = 下一跳不可达** |
| **`>`** | best（被选为最优） | 只有 `*>` 才会进路由表 |
| **`i`**（第 3 列） | 从 iBGP 学到 | — |
| **`r`** | **RIB-failure** | ★ BGP 选它为最优，但**装不进路由表**（有 AD 更低的路由） |
| **`s`** | suppressed | 被聚合抑制（`aggregate-address ... summary-only`） |
| **`d`** | damped | 被路由阻尼抑制 |
| **`h`** | history | 曾经存在，现在不可用（阻尼历史） |
| **`S`** | Stale | GR（优雅重启）期间的陈旧路由 |
| **`m`** | multipath | 参与多路径负载均衡 |
| **`b`** | backup-path | BGP PIC 的备份路径 |
| **`x`** | best-external | 最优的外部路径（有更优的 iBGP 时） |

**★ 三种"看起来有但不生效"的情况**：

**① 只有 `*` 没有 `>`** → **不是最优路径**
```cisco
 *   200.1.1.0/24     10.0.13.3    ...  500 600 i
```
对照选路 11 步分析为什么输了。

**② 什么标记都没有** → **下一跳不可达**
```cisco
     200.2.2.0/24     10.0.99.9    ...  300 i
     ↑ 没有 *
     
R1# show ip route 10.0.99.9
% Network not in table                       ← 找到原因
```

**③ 有 `r`（RIB-failure）** → **BGP 想装但装不进去**
```cisco
 r>  10.1.1.0/24      10.0.12.2    ...  200 i

R1# show ip bgp rib-failure
Network          Next Hop        RIB-failure              RIB-NH Matches
10.1.1.0/24      10.0.12.2       Higher admin distance    n/a

R1# show ip route 10.1.1.0
Routing entry for 10.1.1.0/24
  Known via "static", distance 1                ← ★ 静态路由 AD=1 < eBGP 20
```

> **`r` 标记的实际影响**：BGP 依然会把这条路由**通告给邻居**（因为它是 BGP 的最优路径），但本机转发时用的是那条 AD 更低的路由。
>
> **如果那条 AD 更低的路由指向错误的方向，会造成"我通告了但我自己走别的路"的不一致**——排障时很迷惑。

---

## ④ 选路操控实战

### 4.1 选路 11 步（回顾）

| 步 | 属性 | 规则 | 实战用途 |
|:--|:--|:--|:--|
| 0 | 下一跳可达 | 不可达淘汰 | — |
| **1** | **WEIGHT** | 大的赢（Cisco 私有，本机） | 只影响本机 |
| **2** | **LOCAL_PREF** | 大的赢（AS 内传播） | ★ **控出向流量** |
| 3 | 本地产生 | 自己 network/aggregate 的优先 | — |
| **4** | **AS_PATH 长度** | 短的赢 | ★ **控入向流量** |
| 5 | ORIGIN | i > e > ? | — |
| 6 | MED | 小的赢 | 建议入向（不可靠） |
| 7 | eBGP > iBGP | eBGP 优先 | — |
| 8 | IGP metric | 小的赢 | 热土豆路由 |
| 9 | 更老的 eBGP | 稳定优先 | — |
| 10 | Router ID 小 | 平局决胜 | — |
| 11 | 邻居 IP 小 | 最终决胜 | — |

**口诀**：`We Love Oranges As Apples, Mmm, Eat In Rome`

### 4.2 四种操控手段的对比

| 手段 | 选路步骤 | 影响范围 | 方向 | 可靠性 |
|:--|:--|:--|:--|:--|
| **WEIGHT** | 1 | **只本机** | in | ★★★★★ |
| **LOCAL_PREF** | 2 | **整个 AS** | in | ★★★★★ |
| **AS_PATH Prepend** | 4 | 外部 AS | out | ★★★★ |
| **MED** | 6 | 邻居 AS | out | ★★ |

### 4.3 实战配置

```cisco
! ═══ 控制出向流量：LOCAL_PREF ═══
R1(config)# ip prefix-list ALL seq 5 permit 0.0.0.0/0 le 32
R1(config)# route-map TELECOM-IN permit 10
R1(config-route-map)#  match ip address prefix-list ALL
R1(config-route-map)#  set local-preference 200
R1(config-router)# neighbor <电信> route-map TELECOM-IN in

! ═══ 控制入向流量：AS-Path Prepending ═══
R1(config)# route-map PREPEND-OUT permit 10
R1(config-route-map)#  set as-path prepend 65001 65001 65001
R1(config-router)# neighbor <联通> route-map PREPEND-OUT out

! ═══ 只影响本机：WEIGHT ═══
R1(config-router)# neighbor 10.0.12.2 weight 200
! 或用 route-map 精细控制
R1(config)# route-map SET-WEIGHT permit 10
R1(config-route-map)#  match ip address prefix-list IMPORTANT
R1(config-route-map)#  set weight 500
R1(config-router)# neighbor 10.0.12.2 route-map SET-WEIGHT in

! ═══ 用 Community 打标签，后续批量处理 ═══
R1(config)# ip bgp-community new-format
R1(config)# route-map TAG-CUSTOMER permit 10
R1(config-route-map)#  set community 65001:100
R1(config-router)# neighbor <客户> route-map TAG-CUSTOMER in
R1(config-router)# neighbor <上游> send-community        ! ★ 必须开

! 后续根据 Community 做策略
R1(config)# ip community-list standard CUSTOMER permit 65001:100
R1(config)# route-map POLICY permit 10
R1(config-route-map)#  match community CUSTOMER
R1(config-route-map)#  set local-preference 300
```

> **`send-community` 必须显式开启**，Cisco 默认不发送 Community。忘了配 → 对端收不到 → 基于 Community 的策略全部失效。**这是很隐蔽的配置遗漏。**

### 4.4 route-map 的隐含 deny（★ 极易踩坑）

```cisco
R1(config)# route-map MY-POLICY permit 10
R1(config-route-map)#  match ip address prefix-list IMPORTANT
R1(config-route-map)#  set local-preference 200
! ★ 结束了，没有第二条 ★

R1(config-router)# neighbor 10.0.12.2 route-map MY-POLICY in
```

**后果**：**除了匹配 `IMPORTANT` 的路由，其他所有路由都被拒绝了！**

因为 route-map 末尾有**隐含的 `deny any`**（和 ACL 一样）。

**修复：必须加一条兜底**
```cisco
R1(config)# route-map MY-POLICY permit 20
! ★ 空的 permit 语句 = 匹配所有剩余的，不做任何修改，放行 ★
```

**验证**：
```cisco
R1# show ip bgp summary
Neighbor    V   AS  MsgRcvd MsgSent TblVer InQ OutQ Up/Down  State/PfxRcd
10.0.12.2   4  200     2451    2438      8   0    0 02:15:33        3
                                                                     ↑
                                          本来应该收到 1245 条，现在只有 3 条
                                          → ★ 被 route-map 的隐含 deny 拦了 ★
```

> **这是 BGP 策略配置最常见的事故**：改完 route-map 一软重置，**大量路由消失，业务中断**。
>
> **写 route-map 的铁律：永远记得加兜底的 `permit` 语句。**

---

## ⑤ 路由过滤的四种方法

| 方法 | 匹配什么 | 灵活性 | 性能 |
|:--|:--|:--|:--|
| **prefix-list** | **前缀 + 掩码长度范围** | 中 | ★ 最好 |
| **filter-list (AS-Path ACL)** | **AS_PATH 正则** | 中 | 好 |
| **route-map** | **任意属性组合** + 可修改属性 | ★ 最灵活 | 中 |
| distribute-list | 前缀（用 ACL） | 低 | 差（已过时） |

### 5.1 prefix-list

```cisco
! 语法：ip prefix-list NAME seq N {permit|deny} 网络/长度 [ge X] [le Y]

! 只接受 /24 及更短的前缀（拒绝碎片）
R1(config)# ip prefix-list SANE seq 5 permit 0.0.0.0/0 le 24

! 只接受精确的 10.1.0.0/16
R1(config)# ip prefix-list EXACT seq 5 permit 10.1.0.0/16

! 接受 10.1.0.0/16 内所有 /24
R1(config)# ip prefix-list SUBNETS seq 5 permit 10.1.0.0/16 ge 24 le 24

! 拒绝默认路由和私网
R1(config)# ip prefix-list CLEAN seq 5 deny 0.0.0.0/0
R1(config)# ip prefix-list CLEAN seq 10 deny 10.0.0.0/8 le 32
R1(config)# ip prefix-list CLEAN seq 15 deny 172.16.0.0/12 le 32
R1(config)# ip prefix-list CLEAN seq 20 deny 192.168.0.0/16 le 32
R1(config)# ip prefix-list CLEAN seq 25 deny 0.0.0.0/0 ge 25       ! 拒绝 /25 以上
R1(config)# ip prefix-list CLEAN seq 30 permit 0.0.0.0/0 le 24

R1(config-router)# neighbor 10.0.12.2 prefix-list CLEAN in
```

**`ge` / `le` 的理解（★ 考点）**：
```
   ip prefix-list X permit 10.0.0.0/8 ge 24 le 24
                              ↑        ↑     ↑
                          网络范围   最小长度 最大长度
   
   含义：在 10.0.0.0/8 这个范围内，长度正好是 24 的前缀
   匹配：10.1.1.0/24 ✓、10.255.99.0/24 ✓
   不匹配：10.0.0.0/8 ✗（长度不是24）、10.1.0.0/16 ✗、10.1.1.0/25 ✗
```

| 写法 | 含义 |
|:--|:--|
| `permit 10.0.0.0/8` | **只匹配 10.0.0.0/8 本身** |
| `permit 10.0.0.0/8 le 32` | 10.0.0.0/8 及其内部**所有**子网 |
| `permit 10.0.0.0/8 ge 24` | 10.0.0.0/8 内长度 ≥24 的 |
| `permit 10.0.0.0/8 ge 24 le 26` | 长度在 24–26 之间的 |
| `permit 0.0.0.0/0` | **只匹配默认路由** |
| `permit 0.0.0.0/0 le 32` | **匹配所有前缀** |

> **`permit 0.0.0.0/0` 和 `permit 0.0.0.0/0 le 32` 的区别是最常见的错误**。前者只匹配默认路由，后者匹配一切。

### 5.2 filter-list（AS-Path 正则）

```cisco
! 只接受直连 AS 200 产生的路由（AS_PATH 只有 200）
R1(config)# ip as-path access-list 1 permit ^200$

! 接受 AS 200 及其下游
R1(config)# ip as-path access-list 2 permit ^200_

! 只接受本地产生的（AS_PATH 为空）
R1(config)# ip as-path access-list 3 permit ^$

! 拒绝经过 AS 300 的路由
R1(config)# ip as-path access-list 4 deny _300_
R1(config)# ip as-path access-list 4 permit .*

R1(config-router)# neighbor 10.0.12.2 filter-list 1 in
```

**正则速查**：

| 符号 | 含义 | 例子 |
|:--|:--|:--|
| `^` | 开头 | `^200` = 以 200 开头 |
| `$` | 结尾 | `200$` = 以 200 结尾 |
| `_` | 分隔符（空格/开头/结尾/逗号） | `_300_` = 包含 300 |
| `.` | 任意单个字符 | |
| `*` | 前面的重复 0 次或多次 | `.*` = 匹配任意 |
| `+` | 重复 1 次或多次 | |
| `?` | 0 次或 1 次 | |
| `[]` | 字符集 | `[123]` |
| `\|` | 或 | `^(200\|300)$` |

**常用组合**：

| 正则 | 含义 |
|:--|:--|
| **`^$`** | **本 AS 产生的路由**（AS_PATH 为空） |
| **`^200$`** | 只经过 AS 200（直连邻居产生的） |
| **`^200_`** | 从 AS 200 学来的（AS 200 是第一跳） |
| **`_300$`** | AS 300 产生的（AS 300 是最后一跳） |
| **`_300_`** | 路径中包含 AS 300 |
| **`.*`** | 匹配所有 |

**测试正则**：
```cisco
R1# show ip bgp regexp ^200$
R1# show ip bgp regexp _300_
```

---

## ⑥ 大规模 BGP：RR 与聚合

### 6.1 Route Reflector 深入

**RR 的反射规则**：

| 路由来源 | 反射给 |
|:--|:--|
| **Client** | 所有其他 Client + 所有非 Client |
| **非 Client**（普通 iBGP） | **只给 Client** |
| **eBGP** | 所有邻居 |

**两个防环属性**：

| 属性 | 作用 |
|:--|:--|
| **`ORIGINATOR_ID`** | 记录路由的**最初产生者**。收到 ORIGINATOR_ID 是自己的 → 丢弃 |
| **`CLUSTER_LIST`** | 记录经过的 **RR 集群 ID**。RR 收到含自己 Cluster-ID 的 → 丢弃 |

```cisco
RR(config)# router bgp 65001
RR(config-router)# bgp cluster-id 1.1.1.1
RR(config-router)# neighbor 10.0.0.2 route-reflector-client
RR(config-router)# neighbor 10.0.0.3 route-reflector-client
```

**验证**：
```cisco
RC1# show ip bgp 200.1.1.0
BGP routing table entry for 200.1.1.0/24
  200
    3.3.3.3 from 1.1.1.1 (1.1.1.1)
      Origin IGP, localpref 100, valid, internal, best
      Originator: 3.3.3.3, Cluster list: 1.1.1.1
      ↑                    ↑
   最初产生者          经过的 RR 集群
```

**RR 冗余设计**：
```cisco
! 两台 RR 配【相同的 cluster-id】
RR1(config-router)# bgp cluster-id 1.1.1.1
RR2(config-router)# bgp cluster-id 1.1.1.1
! 每个 Client 同时连两台 RR
```

> **相同 cluster-id 的意义**：两台 RR 属于同一个集群，Client 从 RR1 收到的路由不会再被 RR2 反射回来（CLUSTER_LIST 里已有该 cluster-id），**避免冗余路由和潜在环路**。

**⚠️ RR 的一个副作用：路径隐藏**

RR 只反射**最优路径**。如果 Client-A 有一条到某前缀的路径，Client-B 有另一条，RR 只会反射它认为最优的那条。**其他 Client 看不到备选路径**，可能导致次优路由或收敛慢。

**解法：BGP Add-Path**
```cisco
RR(config-router)# bgp additional-paths select all
RR(config-router)# bgp additional-paths send receive
RR(config-router)# neighbor 10.0.0.2 additional-paths send
RR(config-router)# neighbor 10.0.0.2 advertise additional-paths all
```

### 6.2 路由聚合

```cisco
! ── 基本聚合（同时发聚合和明细）──
R1(config-router)# aggregate-address 192.168.0.0 255.255.252.0

! ── 只发聚合，抑制明细（★ 最常用）──
R1(config-router)# aggregate-address 192.168.0.0 255.255.252.0 summary-only

! ── 保留 AS_PATH 信息（防环）──
R1(config-router)# aggregate-address 192.168.0.0 255.255.252.0 as-set

! ── 有选择地抑制某些明细 ──
R1(config)# route-map SUPPRESS-MAP permit 10
R1(config-route-map)#  match ip address prefix-list HIDE-THESE
R1(config-router)# aggregate-address 192.168.0.0 255.255.252.0 suppress-map SUPPRESS-MAP

! ── 有选择地放行某些被抑制的明细 ──
R1(config-router)# aggregate-address 192.168.0.0 255.255.252.0 summary-only \
                    unsuppress-map UNSUPPRESS-MAP
```

**`as-set` 的作用（★ 考点）**：

**不加 `as-set`**：聚合路由的 AS_PATH **只有本 AS**，丢失了明细路由的 AS_PATH 信息。
```
   明细：200 300 (来自 AS300)
   明细：400 500 (来自 AS500)
        ↓ 聚合（无 as-set）
   聚合：65001                          ← ★ AS_PATH 信息全丢了 ★
        ↓
   ★ 可能形成环路 ★（AS300 可能会接受这条路由，因为它的 AS_PATH 里没有 300）
```

**加了 `as-set`**：
```
   聚合：65001 {200,300,400,500}        ← ★ 用花括号保留所有 AS ★
        ↓
   AS300 收到后发现自己在 AS_SET 里 → 丢弃 → ✅ 防环
```

**代价**：任何一条明细路由的 AS_PATH 变化，都会导致聚合路由的属性变化 → **更频繁的 BGP 更新**。

**验证**：
```cisco
R1# show ip bgp
     Network          Next Hop     Metric LocPrf Weight Path
 *>  192.168.0.0/22   0.0.0.0                        32768 {200,300,400} i
                                                            ↑ as-set 的效果
 s>  192.168.0.0/24   10.0.12.2         0             0 200 300 i
 s>  192.168.1.0/24   10.0.13.3         0             0 400 i
     ↑ s = suppressed（被抑制的明细）
```

**Null0 防环路由**：
```cisco
R1# show ip route | include Null0
B    192.168.0.0/22 [200/0] via 0.0.0.0, 00:02:15, Null0
```
落在聚合范围内但没有明细的流量直接丢弃，**防止和上游来回弹**。

### 6.3 路由阻尼（Route Dampening）

抑制频繁震荡的路由，避免它们不断触发全网 BGP 更新。

```cisco
R1(config-router)# bgp dampening 15 750 2000 60
!                            ↑    ↑    ↑    ↑
!                   半衰期(分) 重用 抑制 最大抑制时间(分)
!                             阈值 阈值
```

**机制**：
```
   路由每 flap 一次，惩罚值 +1000
        ↓
   惩罚值 > 抑制阈值(2000) → ★ 路由被抑制（不使用、不通告）★
        ↓
   惩罚值按半衰期(15分钟)指数衰减
        ↓
   惩罚值 < 重用阈值(750) → 恢复使用
```

```cisco
R1# show ip bgp dampening flap-statistics
R1# show ip bgp dampening dampened-paths
R1# clear ip bgp dampening 10.1.1.0 255.255.255.0
```

> ⚠️ **路由阻尼有争议**：RFC 7196 指出传统的阻尼参数**过于激进**，会导致合法的路由被过度抑制（一次短暂抖动可能被抑制 1 小时）。
>
> **现代建议**：要么不用，要么用更宽松的参数（RIPE-378 推荐）。**企业网基本不需要**，主要是运营商在用。

---

## ⑦ 配套实验：BGP 故障注入

### 实验 A：Active 状态排查

```cisco
! 制造故障：用 ACL 拦掉 TCP 179
R2(config)# ip access-list extended BLOCK-BGP
R2(config-ext-nacl)#  deny tcp any any eq 179
R2(config-ext-nacl)#  deny tcp any eq 179 any
R2(config-ext-nacl)#  permit ip any any
R2(config)# interface GigabitEthernet0/1
R2(config-if)# ip access-group BLOCK-BGP in
```

**观察**：
```cisco
R1# show ip bgp summary
Neighbor    V   AS  MsgRcvd MsgSent TblVer InQ OutQ Up/Down  State/PfxRcd
10.0.12.2   4  200        0       0      0   0    0 never    Active
                                                              ↑↑↑↑↑↑

R1# ping 10.0.12.2
!!!!!                                        ← ping 通！

R1# telnet 10.0.12.2 179
Trying 10.0.12.2, 179 ...
% Connection timed out                       ← ★ 找到了：TCP 179 不通
```

**✅ 这个对比（ping 通但 telnet 179 不通）直接定位到 ACL/防火墙问题。**

### 实验 B：next-hop-self 缺失

```cisco
R1(config-router)# no neighbor 3.3.3.3 next-hop-self
R1# clear ip bgp 3.3.3.3 soft out
```

**在 R3 上观察**：
```cisco
R3# show ip bgp
     Network          Next Hop     Metric LocPrf Weight Path
 *  i200.1.1.0/24     10.0.12.2         0    100      0 200 i
 ↑
 没有 > 标记

R3# show ip bgp 200.1.1.0
BGP routing table entry for 200.1.1.0/24
  Paths: (1 available, no best path)
  200
    10.0.12.2 (inaccessible) from 1.1.1.1 (1.1.1.1)
              ↑↑↑↑↑↑↑↑↑↑↑↑ 找到了
```

**修复**：
```cisco
R1(config-router)# neighbor 3.3.3.3 next-hop-self
R1# clear ip bgp 3.3.3.3 soft out
```

### 实验 C：route-map 隐含 deny

```cisco
R1(config)# ip prefix-list IMPORTANT seq 5 permit 200.1.1.0/24
R1(config)# route-map POLICY permit 10
R1(config-route-map)#  match ip address prefix-list IMPORTANT
R1(config-route-map)#  set local-preference 200
! ★ 故意不加兜底 ★

R1(config-router)# neighbor 10.0.12.2 route-map POLICY in
R1# clear ip bgp 10.0.12.2 soft in
```

**观察**：
```cisco
R1# show ip bgp summary
Neighbor    V   AS  MsgRcvd MsgSent TblVer InQ OutQ Up/Down State/PfxRcd
10.0.12.2   4  200     2451    2438      8   0    0 02:15:33      1
                                                                  ↑
                                        本来 1245 条，现在只剩 1 条！
```

**修复**：
```cisco
R1(config)# route-map POLICY permit 20      ! ★ 空的 permit = 兜底放行
R1# clear ip bgp 10.0.12.2 soft in
```

**✅ 这个实验能让你永远记住 route-map 的隐含 deny。**

### 实验 D：RIB-failure

```cisco
! 手工加一条静态路由，与 BGP 学到的前缀冲突
R1(config)# ip route 200.1.1.0 255.255.255.0 10.0.13.3
```

**观察**：
```cisco
R1# show ip bgp
     Network          Next Hop     Metric LocPrf Weight Path
 r>  200.1.1.0/24     10.0.12.2         0             0 200 i
 ↑
 RIB-failure

R1# show ip bgp rib-failure
Network          Next Hop        RIB-failure              RIB-NH Matches
200.1.1.0/24     10.0.12.2       Higher admin distance    n/a

R1# show ip route 200.1.1.0
Routing entry for 200.1.1.0/24
  Known via "static", distance 1                ← 静态 AD=1 赢了 eBGP AD=20
```

**注意**：即使有 `r`，R1 **依然会把这条路由通告给它的 BGP 邻居**——因为它是 BGP 的最优路径。**但本机转发走的是静态路由。** 这种不一致在排障时很迷惑。

### 实验 E：AS-Path 正则测试

```cisco
R1# show ip bgp regexp ^200$
! 只显示 AS_PATH 正好是 "200" 的路由

R1# show ip bgp regexp _300_
! 显示路径中包含 AS 300 的路由

R1# show ip bgp regexp ^$
! 显示本 AS 产生的路由
```

**应用到策略**：
```cisco
! 只接受直连 AS 200 自己的路由，不接受它转发的其他 AS 的路由
R1(config)# ip as-path access-list 1 permit ^200$
R1(config-router)# neighbor 10.0.12.2 filter-list 1 in
R1# clear ip bgp 10.0.12.2 soft in

R1# show ip bgp summary | include 10.0.12.2
10.0.12.2   4  200   2451  2438   8  0  0 02:15:33      5
                                                         ↑ 前缀数大幅减少
```

---

## ⑧ 排障速查表

| 症状 | 根因 | 验证 |
|:--|:--|:--|
| 邻居 **Idle** | 路由不通 / 地址写错 / shutdown | `ping`、`show run \| sec router bgp` |
| 邻居 **Active** | **TCP 179 被拦** / 源地址不对 | **`telnet <IP> 179`** |
| 卡 **OpenSent** | AS 号不匹配 / Router ID 冲突 | `debug ip bgp` |
| 用 Loopback 建 iBGP 起不来 | 缺 `update-source` | `show run \| sec router bgp` |
| 用 Loopback 建 eBGP 起不来 | 缺 `ebgp-multihop` | 同上 |
| **收到路由但没有 `>`** | 不是最优 | 对照选路 11 步 |
| **路由没有 `*`** | **下一跳不可达** | `show ip route <下一跳>`、配 `next-hop-self` |
| 路由有 **`r`** | **AD 更低的路由抢赢了** | `show ip bgp rib-failure` |
| **前缀数突然锐减** | **route-map 隐含 deny** | `show route-map`、加兜底 permit |
| `network` 不生效 | 路由表无完全匹配的路由 | `show ip route <网段>`，加 Null0 |
| Community 策略不生效 | 缺 `send-community` | `show run \| inc send-community` |
| 改了策略不生效 | 没软重置 | `clear ip bgp X soft in/out` |
| 看不到 received-routes | 缺 `soft-reconfiguration inbound` | 配上或用 Route Refresh |
| 邻居频繁震荡 | 链路 / CPU / **maximum-prefix 触发** | `show logging`、`show ip bgp summary` |
| 聚合后明细还在 | 缺 `summary-only` | `show ip bgp` 看 `s` 标记 |

### 关键排障命令

```cisco
! ── 我发给对方什么 ──
R1# show ip bgp neighbors 10.0.12.2 advertised-routes

! ── 对方发给我什么（经过 in 策略过滤后的）──
R1# show ip bgp neighbors 10.0.12.2 routes

! ── 对方发给我什么（★ 原始的，未过滤）──
R1# show ip bgp neighbors 10.0.12.2 received-routes
! ⚠️ 需要先配 soft-reconfiguration inbound

! ── 策略调试 ──
R1# show route-map POLICY
route-map POLICY, permit, sequence 10
  Match clauses:
    ip address prefix-lists: IMPORTANT
  Set clauses:
    local-preference 200
  Policy routing matches: 0 packets, 0 bytes
                          ↑ 匹配计数

! ── 前缀列表命中 ──
R1# show ip prefix-list detail CLEAN
ip prefix-list CLEAN:
   count: 6, range entries: 4, sequences: 5 - 30, refcount: 2
   seq 5 deny 0.0.0.0/0 (hit count: 12, refcount: 1)
   seq 30 permit 0.0.0.0/0 le 24 (hit count: 1245, refcount: 1)
                                       ↑ 有命中说明规则在起作用
```

---

## ⑨ 自测题

**1.** BGP 表里的 `r` 标记是什么意思？影响是什么？

<details><summary>答案</summary>

**`r` = RIB-failure：BGP 选它为最优路径，但【装不进路由表】。**

**最常见的原因：有 AD 更低的路由占据了同一个前缀。**

```cisco
R1# show ip bgp
     Network          Next Hop     Metric LocPrf Weight Path
 r>  200.1.1.0/24     10.0.12.2         0             0 200 i
 ↑↑
 r = RIB-failure, > = BGP 认为它是最优

R1# show ip bgp rib-failure
Network          Next Hop        RIB-failure              RIB-NH Matches
200.1.1.0/24     10.0.12.2       Higher admin distance    n/a
                                 ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑

R1# show ip route 200.1.1.0
Routing entry for 200.1.1.0/24
  Known via "static", distance 1                 ← ★ 静态 AD=1 < eBGP AD=20
  Routing Descriptor Blocks:
  * 10.0.13.3
```

**AD 对照**：

| 来源 | AD |
|:--|:--|
| 直连 | 0 |
| 静态 | **1** |
| **eBGP** | **20** |
| EIGRP 内部 | 90 |
| OSPF | 110 |
| **iBGP** | **200** |

所以**静态路由、EIGRP、OSPF 都可能抢赢 iBGP**（AD 200）；静态路由能抢赢 eBGP（AD 20）。

**★ 影响（最关键的一点）**：

**BGP 依然会把这条路由通告给邻居**（因为它是 BGP 的最优路径），**但本机转发时用的是那条 AD 更低的路由**。

```
   ★ 产生了"控制平面"和"数据平面"的不一致 ★
   
   我告诉邻居："去 200.1.1.0/24 走我"
   但我自己转发时走的是静态路由指向的方向
        ↓
   如果那条静态路由指向错误的地方 → ★ 黑洞或环路 ★
```

**这种不一致在排障时极其迷惑**：邻居说"我把流量发给你了"，你说"我通告了这条路由啊"，但流量就是不通。

**其他 RIB-failure 的原因**：

| 原因 | `show ip bgp rib-failure` 显示 |
|:--|:--|
| **AD 更低的路由** | `Higher admin distance` |
| 路由表满 | `Routing table limit exceeded` |
| 内存不足 | `No memory` |
| 下一跳不匹配 | `Nexthop mismatch` |

**处理方式**：

**① 确认是不是有意的**
```cisco
R1# show ip route 200.1.1.0
! 如果那条静态路由是你有意配的（比如流量工程），那 r 标记是正常的
```

**② 如果是误配，删掉冲突的路由**
```cisco
R1(config)# no ip route 200.1.1.0 255.255.255.0 10.0.13.3
```

**③ 如果确实需要共存，调整 AD**
```cisco
R1(config)# ip route 200.1.1.0 255.255.255.0 10.0.13.3 250    ! 调高静态 AD
! 或调低 BGP AD
R1(config-router)# distance bgp 20 200 200
```

**④ 让 BGP 不通告 RIB-failure 的路由**（保持一致性）
```cisco
R1(config-router)# bgp suppress-inactive
```
这条命令让 BGP **不通告那些没能装进路由表的路由**，消除控制平面和数据平面的不一致。**推荐配置。**

**日常巡检建议**：把 `show ip bgp rib-failure` 加进巡检脚本。**正常情况下应该没有输出。**
</details>

**2.** 你配了一个 route-map 应用到 BGP 邻居的 in 方向，结果收到的前缀数从 1245 条变成 3 条。为什么？

<details><summary>答案</summary>

**route-map 末尾有隐含的 `deny any`，你只写了 permit 的条件，其他所有路由都被拒绝了。**

```cisco
! ❌ 有问题的配置
R1(config)# ip prefix-list IMPORTANT seq 5 permit 200.1.1.0/24
R1(config)# route-map POLICY permit 10
R1(config-route-map)#  match ip address prefix-list IMPORTANT
R1(config-route-map)#  set local-preference 200
! ★ 结束了，没有第二条 ★

R1(config-router)# neighbor 10.0.12.2 route-map POLICY in
```

**执行逻辑**：
```
   收到一条路由
        ↓
   序列 10：匹配 IMPORTANT 吗？
        ├─ 是 → permit，设 local-pref 200 ✓
        └─ 否 → 继续往下看
                 ↓
             没有更多序列了
                 ↓
        ★ 隐含的 deny → 拒绝 ★
```

**结果**：只有匹配 `IMPORTANT` 的路由被接受，**其他 1242 条全被拒绝了**。

**在生产环境这就是业务中断事故。**

**✅ 正确配置：必须加兜底**
```cisco
R1(config)# route-map POLICY permit 10
R1(config-route-map)#  match ip address prefix-list IMPORTANT
R1(config-route-map)#  set local-preference 200

R1(config)# route-map POLICY permit 20
! ★ 空的 permit 语句 ★
! 没有 match = 匹配所有剩余的
! 没有 set   = 不修改任何属性
! → 原样放行
```

**验证**：
```cisco
R1# show route-map POLICY
route-map POLICY, permit, sequence 10
  Match clauses:
    ip address prefix-lists: IMPORTANT
  Set clauses:
    local-preference 200
  Policy routing matches: 0 packets, 0 bytes
route-map POLICY, permit, sequence 20
  Match clauses:                              ← 空 = 匹配所有
  Set clauses:                                ← 空 = 不修改
  Policy routing matches: 0 packets, 0 bytes

R1# clear ip bgp 10.0.12.2 soft in
R1# show ip bgp summary | include 10.0.12.2
10.0.12.2   4  200   2451  2438   8  0  0 02:15:33   1245    ← 恢复 ✓
```

**同样的陷阱也存在于**：
- **prefix-list**（末尾隐含 deny）
- **ACL**（末尾隐含 deny）
- **as-path access-list**（末尾隐含 deny）

```cisco
! prefix-list 也要兜底
R1(config)# ip prefix-list CLEAN seq 5 deny 10.0.0.0/8 le 32
R1(config)# ip prefix-list CLEAN seq 10 permit 0.0.0.0/0 le 32    ← ★ 兜底

! as-path access-list 也要
R1(config)# ip as-path access-list 1 deny _300_
R1(config)# ip as-path access-list 1 permit .*                     ← ★ 兜底
```

**安全操作流程（生产环境改 BGP 策略）**：

```cisco
! ① 先记录当前状态
R1# show ip bgp summary | redirect flash:before.txt

! ② 用 configure revert 保险
R1# write memory
R1# configure terminal revert timer 5

! ③ 改策略（★ 记得加兜底）
R1(config)# route-map POLICY permit 10
R1(config-route-map)#  ...
R1(config)# route-map POLICY permit 20
R1(config)# end

! ④ 软重置
R1# clear ip bgp 10.0.12.2 soft in

! ⑤ 立即对比前缀数
R1# show ip bgp summary | include 10.0.12.2
! 数字大幅下降 → 有问题，等 revert 自动回滚

! ⑥ 确认无误
R1# configure confirm
R1# write memory
```

**★ 写 route-map 的三条铁律**：
1. **永远加兜底的 `permit` 语句**
2. **改完立即对比前缀数**
3. **生产环境用 `configure revert` 保险**
</details>

**3.** `ip prefix-list X permit 10.0.0.0/8` 和 `ip prefix-list X permit 10.0.0.0/8 le 32` 有什么区别？

<details><summary>答案</summary>

| 写法 | 匹配什么 |
|:--|:--|
| **`permit 10.0.0.0/8`** | **只匹配 `10.0.0.0/8` 这一条前缀本身** |
| **`permit 10.0.0.0/8 le 32`** | **匹配 10.0.0.0/8 及其内部的所有子网** |

**具体对比**：

| 前缀 | `10.0.0.0/8` | `10.0.0.0/8 le 32` |
|:--|:--|:--|
| `10.0.0.0/8` | ✅ | ✅ |
| `10.1.0.0/16` | ❌ | ✅ |
| `10.1.1.0/24` | ❌ | ✅ |
| `10.1.1.1/32` | ❌ | ✅ |
| `11.0.0.0/8` | ❌ | ❌ |

**`ge` / `le` 的完整语法**：
```
   ip prefix-list NAME seq N {permit|deny} 网络/长度 [ge 最小长度] [le 最大长度]
                                             ↑
                                    定义"在哪个网络范围内"
```

**规则**：`网络长度 ≤ ge ≤ le ≤ 32`

**常用组合速查**：

| 写法 | 含义 |
|:--|:--|
| `permit 10.0.0.0/8` | 只有 10.0.0.0/8 本身 |
| `permit 10.0.0.0/8 le 32` | 10.0.0.0/8 及所有子网 |
| `permit 10.0.0.0/8 ge 24` | 10.0.0.0/8 内长度 ≥ 24 的 |
| `permit 10.0.0.0/8 ge 24 le 24` | 10.0.0.0/8 内**正好 /24** 的 |
| `permit 10.0.0.0/8 ge 16 le 24` | 长度在 16–24 之间的 |
| **`permit 0.0.0.0/0`** | **只匹配默认路由** ★ |
| **`permit 0.0.0.0/0 le 32`** | **匹配所有前缀** ★ |
| `permit 0.0.0.0/0 le 24` | 所有长度 ≤ 24 的前缀 |
| `permit 0.0.0.0/0 ge 25` | 所有长度 ≥ 25 的（碎片） |

> **`permit 0.0.0.0/0` 和 `permit 0.0.0.0/0 le 32` 的混淆是最常见的错误。**
>
> 如果你想"放行所有路由"却写了 `permit 0.0.0.0/0`，结果是**只放行了默认路由，其他全被隐含 deny 拦掉**。

**实战应用：过滤运营商发来的路由**

```cisco
R1(config)# ip prefix-list SANE-IN seq 5   deny 0.0.0.0/0                ! 拒默认路由
R1(config)# ip prefix-list SANE-IN seq 10  deny 0.0.0.0/8 le 32          ! 拒 0.x
R1(config)# ip prefix-list SANE-IN seq 15  deny 10.0.0.0/8 le 32         ! 拒私网
R1(config)# ip prefix-list SANE-IN seq 20  deny 127.0.0.0/8 le 32        ! 拒环回
R1(config)# ip prefix-list SANE-IN seq 25  deny 169.254.0.0/16 le 32     ! 拒链路本地
R1(config)# ip prefix-list SANE-IN seq 30  deny 172.16.0.0/12 le 32      ! 拒私网
R1(config)# ip prefix-list SANE-IN seq 35  deny 192.168.0.0/16 le 32     ! 拒私网
R1(config)# ip prefix-list SANE-IN seq 40  deny 224.0.0.0/4 le 32        ! 拒组播
R1(config)# ip prefix-list SANE-IN seq 45  deny 0.0.0.0/0 ge 25          ! 拒 /25 以上碎片
R1(config)# ip prefix-list SANE-IN seq 50  permit 0.0.0.0/0 le 24        ! ★ 放行其余

R1(config-router)# neighbor <运营商> prefix-list SANE-IN in
```

**只通告自己的网段（防止变成中转 AS）**：
```cisco
R1(config)# ip prefix-list MY-NETS seq 5 permit 202.100.0.0/22
R1(config-router)# neighbor <运营商> prefix-list MY-NETS out
```

**验证命中**：
```cisco
R1# show ip prefix-list detail SANE-IN
ip prefix-list SANE-IN:
   count: 10, range entries: 8, sequences: 5 - 50, refcount: 2
   seq 5 deny 0.0.0.0/0 (hit count: 2, refcount: 1)
   seq 15 deny 10.0.0.0/8 le 32 (hit count: 45, refcount: 1)     ← 拦到了私网
   seq 50 permit 0.0.0.0/0 le 24 (hit count: 1198, refcount: 1)
```

**hit count 能告诉你每条规则实际拦了多少**，是验证策略是否符合预期的好办法。
</details>

**4.** RR 的 `ORIGINATOR_ID` 和 `CLUSTER_LIST` 分别防什么环路？

<details><summary>答案</summary>

**背景**：RR 打破了 iBGP 的水平分割规则（"从 iBGP 学到的不能传给其他 iBGP"），所以**需要新的防环机制**。

| 属性 | 记录什么 | 防什么环路 |
|:--|:--|:--|
| **ORIGINATOR_ID** | 路由的**最初产生者**（在本 AS 内第一个通告它的路由器的 Router ID） | **防止路由回到产生它的路由器** |
| **CLUSTER_LIST** | 经过的**所有 RR 集群的 Cluster-ID** 列表 | **防止路由在多个 RR 之间循环** |

**ORIGINATOR_ID 的工作方式**：
```
   RC1 通告一条路由给 RR
        ↓
   RR 设置 ORIGINATOR_ID = RC1 的 Router ID
        ↓
   RR 反射给 RC2、RC3... 以及 RC1（如果配置不当）
        ↓
   ★ RC1 收到后发现 ORIGINATOR_ID 是自己 → 丢弃 ★
```

**CLUSTER_LIST 的工作方式**：
```
   RR1（cluster-id 1.1.1.1）反射一条路由
        ↓
   在 CLUSTER_LIST 里加上 1.1.1.1
        ↓
   路由传到 RR2（cluster-id 2.2.2.2）
        ↓
   RR2 加上 2.2.2.2 → CLUSTER_LIST = {1.1.1.1, 2.2.2.2}
        ↓
   如果路由又传回 RR1
        ↓
   ★ RR1 发现 CLUSTER_LIST 里有自己的 1.1.1.1 → 丢弃 ★
```

**查看**：
```cisco
RC2# show ip bgp 200.1.1.0
BGP routing table entry for 200.1.1.0/24, version 5
Paths: (1 available, best #1)
  200
    3.3.3.3 (metric 20) from 1.1.1.1 (1.1.1.1)
      Origin IGP, localpref 100, valid, internal, best
      ★ Originator: 3.3.3.3, Cluster list: 1.1.1.1 ★
                    ↑                      ↑
             最初产生者是 RC1        经过了 cluster 1.1.1.1
```

**RR 冗余的正确配置**：

```cisco
! 两台 RR 配【相同的 cluster-id】
RR1(config-router)# bgp cluster-id 1.1.1.1
RR2(config-router)# bgp cluster-id 1.1.1.1

! 每个 Client 同时连两台 RR
RC1(config-router)# neighbor <RR1> remote-as 65001
RC1(config-router)# neighbor <RR2> remote-as 65001
```

**为什么两台 RR 要用相同的 cluster-id**：

```
   相同 cluster-id：
   RC1 → RR1 → RC2
   RC1 → RR2 → RC2
        ↓
   RC2 从 RR2 收到的路由，CLUSTER_LIST 里有 1.1.1.1
   RC2 从 RR1 收到的路由，CLUSTER_LIST 里也有 1.1.1.1
        ↓
   ★ 两条是同一个集群的，RC2 只保留一条 ★
   → 减少冗余路由，节省内存
   
   不同 cluster-id：
        ↓
   RC2 会收到两条独立的路由（CLUSTER_LIST 不同）
   → 冗余更好（一条失效还有另一条）
   → 但内存开销更大
```

**⚠️ 实践中的争议**：
- **相同 cluster-id**：省内存，但如果 RR1 和 RC2 之间的路径断了，RC2 可能失去所有路径（因为它把 RR2 的路由当成了重复的）
- **不同 cluster-id**：更好的冗余，但 BGP 表更大

**现代建议：用不同的 cluster-id**（默认就是各自的 Router ID），除非内存确实紧张。

**RR 的另一个副作用：路径隐藏**

RR **只反射自己认为最优的那条路径**。
```
   RC1 有到 200.1.1.0/24 的路径 A
   RC2 有到 200.1.1.0/24 的路径 B
        ↓
   RR 选出最优（假设是 A），只反射 A
        ↓
   ★ RC3 只知道路径 A，不知道有路径 B ★
   → 可能次优路由
   → A 失效时收敛慢（要等 RR 重新选路再反射）
```

**解法：BGP Add-Path**
```cisco
RR(config-router)# bgp additional-paths select all
RR(config-router)# bgp additional-paths send receive
RR(config-router)# neighbor <client> additional-paths send
RR(config-router)# neighbor <client> advertise additional-paths all
```

这让 RR 可以反射多条路径，Client 能看到全部备选。
</details>

**5.** 路由聚合的 `as-set` 参数是干什么的？不加会有什么风险？

<details><summary>答案</summary>

**`as-set` 让聚合路由保留所有被聚合明细路由的 AS_PATH 信息（用花括号 `{}` 表示无序集合）。**

**不加 `as-set` 的问题**：

```
   R1 聚合了两条明细：
   · 192.168.0.0/24  AS_PATH: 200 300
   · 192.168.1.0/24  AS_PATH: 400 500
        ↓
   聚合成 192.168.0.0/22
        ↓
   ★ AS_PATH 变成：65001（只有本 AS）★
   ★ 200/300/400/500 的信息全丢了 ★
```

**风险：路由环路**

```
   AS 300 收到这条聚合路由 192.168.0.0/22
        ↓
   检查 AS_PATH：65001
        ↓
   ★ 没有自己的 AS 号 → 认为是合法路由 → 接受 ★
        ↓
   但这条聚合路由里包含了 AS 300 自己产生的 192.168.0.0/24
        ↓
   ★ AS 300 可能把去往自己网段的流量发回给 AS 65001 → 环路 ★
```

**加了 `as-set` 后**：

```cisco
R1(config-router)# aggregate-address 192.168.0.0 255.255.252.0 summary-only as-set
```

```cisco
R1# show ip bgp
     Network          Next Hop     Metric LocPrf Weight Path
 *>  192.168.0.0/22   0.0.0.0                        32768 {200,300,400,500} i
                                                            ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
                                                    花括号表示 AS_SET（无序集合）
 s>  192.168.0.0/24   10.0.12.2         0             0 200 300 i
 s>  192.168.1.0/24   10.0.13.3         0             0 400 500 i
```

**现在 AS 300 收到时**：
```
   检查 AS_PATH：{200,300,400,500}
        ↓
   ★ 发现自己的 AS 300 在里面 → 丢弃 ★ ✅ 防环成功
```

**`as-set` 的代价**：

**任何一条明细路由的 AS_PATH 变化，都会导致聚合路由的属性变化。**

```
   明细路由 A 的 AS_PATH 变了（比如上游改了路径）
        ↓
   聚合路由的 AS_SET 也要变
        ↓
   ★ 触发新的 BGP UPDATE 通告给所有邻居 ★
        ↓
   如果明细路由不稳定，聚合路由也跟着震荡
```

**这就削弱了"聚合能隔离震荡"的价值。**

**权衡建议**：

| 场景 | 建议 |
|:--|:--|
| 聚合的明细**全部来自本 AS** | **不需要 `as-set`**（不存在环路风险） |
| 聚合的明细**来自多个外部 AS** | **建议加 `as-set`**（防环） |
| 明细路由很不稳定 | 不加 `as-set`（避免震荡传播），但要用其他手段防环 |

**其他聚合选项**：

```cisco
! 基本聚合（同时发聚合和明细）
aggregate-address 192.168.0.0 255.255.252.0

! ★ 只发聚合，抑制所有明细（最常用）
aggregate-address 192.168.0.0 255.255.252.0 summary-only

! 保留 AS 信息
aggregate-address 192.168.0.0 255.255.252.0 as-set

! 有选择地抑制部分明细
aggregate-address 192.168.0.0 255.255.252.0 suppress-map SUPPRESS-THESE

! 对特定邻居放行被抑制的明细
neighbor 10.0.12.2 unsuppress-map SHOW-DETAILS

! 设置聚合路由的属性
aggregate-address 192.168.0.0 255.255.252.0 attribute-map SET-ATTRS
```

**自动生成的 Null0 防环路由**：
```cisco
R1# show ip route | include Null0
B    192.168.0.0/22 [200/0] via 0.0.0.0, 00:02:15, Null0
```

**作用**：如果流量的目的地在聚合范围内但**没有对应的明细路由**，直接丢弃。

**没有这条路由会怎样**：
```
   包的目的是 192.168.3.10（在 /22 范围内但没有明细）
        ↓
   R1 匹配默认路由 → 发给上游
        ↓
   上游匹配 R1 通告的 192.168.0.0/22 → 发回给 R1
        ↓
   ★ 来回弹，直到 TTL 耗尽 ★
```

**Null0 路由把这个环路掐死在本地。**

**验证聚合是否生效**：
```cisco
R1# show ip bgp
! 看聚合路由有 *> 标记
! 看明细路由有 s（suppressed）标记

R1# show ip bgp 192.168.0.0/22
BGP routing table entry for 192.168.0.0/22, version 8
  Paths: (1 available, best #1, table default)
  Advertised to update-groups: 1
  Refresh Epoch 1
  Local, (aggregated by 65001 1.1.1.1)              ← 确认是聚合产生的
    0.0.0.0 from 0.0.0.0 (1.1.1.1)
      Origin IGP, localpref 100, weight 32768, valid, aggregated, local, atomic-aggregate, best
                                                              ↑
                                              atomic-aggregate 表示丢失了明细信息
                                              （加了 as-set 后这个标记会消失）
```
</details>

---

**上一章** ← [02 OSPF 深入与排障](02-OSPF深入与排障.md) ｜ **下一章** → [04 路由重分发与路由策略](04-路由重分发与路由策略.md)
