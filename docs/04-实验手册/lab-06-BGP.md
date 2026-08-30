# Lab 06 · BGP 双出口与选路操控

**对应章节**：[ENCOR 08](../02-ENCOR-350-401/08-BGP.md)、[ENARSI 03](../03-ENARSI-300-410/03-BGP深入与排障.md)
**难度**：★★★★　**时长**：3 小时

## 目标

1. 搭建双运营商出口，配置 eBGP 和 iBGP
2. **验证 `next-hop-self` 的必要性**
3. 用 LOCAL_PREF 控制出向流量，用 AS-Path Prepend 控制入向流量
4. **亲手踩一次 route-map 隐含 deny 的坑**
5. 配置路由聚合、前缀过滤、Route Reflector
6. 排查 6 个典型故障

---

## 拓扑

```
      AS 100 (电信)                    AS 200 (联通)
      [ISP-1]                          [ISP-2]
      10.0.11.1                        10.0.22.1
          │ eBGP                            │ eBGP
      10.0.11.2                        10.0.22.2
      ┌───▼────┐                      ┌─────▼──┐
      │   R1   │═══════ iBGP ═════════│   R2   │
      │Lo0:1.1.1.1                    │Lo0:2.2.2.2
      └───┬────┘                      └─────┬──┘
          │        AS 65001                 │
          └────────────┬────────────────────┘
                  ┌────▼────┐
                  │   R3    │  (内网核心)
                  │Lo0:3.3.3.3
                  └─────────┘
                       │
              内网 192.168.0.0/22
```

| 设备 | AS | 角色 |
|:--|:--|:--|
| ISP-1 | 100 | 模拟电信 |
| ISP-2 | 200 | 模拟联通 |
| R1 | 65001 | 连电信的边界路由器 |
| R2 | 65001 | 连联通的边界路由器 |
| R3 | 65001 | 内网核心（后面做 RR） |

---

## Part 1：基础配置

### Step 1：IGP（先让 Loopback 互通）

```cisco
! ── R1/R2/R3 都配 OSPF ──
R1(config)# router ospf 1
R1(config-router)# router-id 1.1.1.1
R1(config-router)# network 1.1.1.1 0.0.0.0 area 0
R1(config-router)# network 10.0.13.1 0.0.0.0 area 0
R1(config-router)# network 10.0.12.1 0.0.0.0 area 0
```

**验证**：
```cisco
R1# ping 2.2.2.2 source Loopback0
!!!!!                                              ← ★ iBGP 的前提
R1# ping 3.3.3.3 source Loopback0
!!!!!
```

### Step 2：eBGP（R1 ↔ ISP-1）

```cisco
R1(config)# router bgp 65001
R1(config-router)# bgp router-id 1.1.1.1
R1(config-router)# bgp log-neighbor-changes

! ── eBGP 到电信 ──
R1(config-router)# neighbor 10.0.11.1 remote-as 100
R1(config-router)# neighbor 10.0.11.1 description ### CHINA-TELECOM ###
R1(config-router)# neighbor 10.0.11.1 password BgpSecret123
R1(config-router)# ★ neighbor 10.0.11.1 soft-reconfiguration inbound ★
R1(config-router)# ★ neighbor 10.0.11.1 maximum-prefix 1000 80 restart 5 ★
```

**`soft-reconfiguration inbound` 的价值**：能看到**未经策略过滤的原始路由**，排障时非常有用。
```cisco
R1# show ip bgp neighbors 10.0.11.1 ★ received-routes ★
! 需要先配 soft-reconfiguration inbound
```

### Step 3：iBGP（★ 三件套）

```cisco
R1(config-router)# neighbor 2.2.2.2 remote-as 65001
R1(config-router)# ★ neighbor 2.2.2.2 update-source Loopback0 ★     ! ① 用 Loopback
R1(config-router)# ★ neighbor 2.2.2.2 next-hop-self ★               ! ② 改下一跳
R1(config-router)# neighbor 2.2.2.2 soft-reconfiguration inbound     ! ③ 便于排障

R1(config-router)# neighbor 3.3.3.3 remote-as 65001
R1(config-router)# neighbor 3.3.3.3 update-source Loopback0
R1(config-router)# neighbor 3.3.3.3 next-hop-self
R1(config-router)# neighbor 3.3.3.3 soft-reconfiguration inbound
```

**★ 为什么用 Loopback 建 iBGP**：
- Loopback 永远 up，**物理链路故障不会导致 BGP 会话断开**（只要还有其他路径可达）
- 配合 `next-hop-self`，下一跳指向 Loopback，IGP 里必然可达

### Step 4：通告内网网段

```cisco
! ★ BGP 的 network 命令要求路由表里有【完全匹配】的路由 ★
R1(config)# ★ ip route 192.168.0.0 255.255.252.0 Null0 ★

R1(config)# router bgp 65001
R1(config-router)# ★ network 192.168.0.0 mask 255.255.252.0 ★
```

**验证 `network` 生效**：
```cisco
R1# show ip bgp
     Network          Next Hop     Metric LocPrf Weight Path
 ★*>★ 192.168.0.0/22   0.0.0.0           0         32768 i
    ↑ 有 *> 说明生效了
```

**试试不加 Null0 路由会怎样**：
```cisco
R1(config)# no ip route 192.168.0.0 255.255.252.0 Null0
R1# show ip bgp | include 192.168.0.0
! ★ 空！network 不生效 ★
```

---

## Part 2：★ 核心实验 —— next-hop-self

### Step 1：先不配 next-hop-self

```cisco
R1(config-router)# ★ no neighbor 3.3.3.3 next-hop-self ★
R1# clear ip bgp 3.3.3.3 soft out
```

### Step 2：在 R3 上观察

```cisco
R3# show ip bgp
     Network          Next Hop         Metric LocPrf Weight Path
 ★ *  i ★ 200.1.1.0/24     10.0.11.1        0    100      0 100 i
    ↑
  ★ 只有 * 没有 > —— 不是最优路径 ★

R3# show ip bgp 200.1.1.0
BGP routing table entry for 200.1.1.0/24
  Paths: (1 available, ★ no best path ★)
  100
    10.0.11.1 ★ (inaccessible) ★ from 1.1.1.1 (1.1.1.1)
              ↑↑↑↑↑↑↑↑↑↑↑↑↑↑
              ★ 下一跳不可达！找到原因 ★

R3# show ip route 10.0.11.1
% Network not in table
! ★ 10.0.11.0/30 是 R1 和 ISP-1 之间的互联网段，没有通告进 IGP ★
```

**★ 理解**：
```
   ① R1 从 ISP-1（eBGP）学到路由，NEXT_HOP = 10.0.11.1
   ② R1 通过 iBGP 传给 R3 时，★ NEXT_HOP 保持不变 ★
   ③ R3 不知道怎么到 10.0.11.1（那是运营商的地址）
   ④ → 路由不可达，不进路由表
```

### Step 3：加回 next-hop-self

```cisco
R1(config-router)# ★ neighbor 3.3.3.3 next-hop-self ★
R1# clear ip bgp 3.3.3.3 soft out
```

```cisco
R3# show ip bgp
     Network          Next Hop         Metric LocPrf Weight Path
 ★ *>i ★ 200.1.1.0/24   ★ 1.1.1.1 ★       0    100      0 100 i
    ↑↑                     ↑
  现在是 best         下一跳变成 R1 的 Loopback，IGP 里可达 ✓

R3# show ip route bgp | include 200.1.1.0
B    200.1.1.0/24 [200/0] via 1.1.1.1, 00:00:15
                    ↑ iBGP 的 AD = 200
```

**✅ 这个实验直观证明了 `next-hop-self` 的必要性。**

**★ iBGP 排障 Top 1 的问题**：症状是"BGP 表里有路由但没有 `>` 标记，路由表里也没有"，看到 `(inaccessible)` 就是它。

---

## Part 3：控制出向流量（LOCAL_PREF）

### 目标

**所有出向流量优先走电信（R1），电信断了才走联通（R2）。**

### Step 1：配置

```cisco
R1(config)# ip prefix-list ALL seq 5 permit 0.0.0.0/0 ★ le 32 ★
!                                                       ↑ 必须加 le 32，否则只匹配默认路由

R1(config)# route-map TELECOM-IN permit 10
R1(config-route-map)#  match ip address prefix-list ALL
R1(config-route-map)#  ★ set local-preference 200 ★

R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.11.1 route-map TELECOM-IN in
R1# ★ clear ip bgp 10.0.11.1 soft in ★
```

### Step 2：在 R2 上验证（★ LOCAL_PREF 通过 iBGP 传播）

```cisco
R2# show ip bgp 200.1.1.0
BGP routing table entry for 200.1.1.0/24
Paths: (2 available, best #1)
  100                                              ← 经 R1（电信）
    1.1.1.1 (metric 20) from 1.1.1.1 (1.1.1.1)
      Origin IGP, ★ localpref 200 ★, valid, ★ internal ★, ★ best ★
                                              ↑            ↑
                              这是 iBGP 路径          但它赢了！
  200                                              ← R2 自己的 eBGP（联通）
    10.0.22.1 from 10.0.22.1 (22.22.22.22)
      Origin IGP, ★ localpref 100 ★, valid, ★ external ★
                                              ↑
                                     eBGP 路径，但输了
```

**★ 关键观察**：**LOCAL_PREF 200 的 iBGP 路径赢过了 LOCAL_PREF 100 的 eBGP 路径**。

**为什么**：
```
   BGP 选路顺序：
   ① WEIGHT
   ★ ② LOCAL_PREF ★        ← 在这一步就分出胜负了
   ③ 本地产生
   ④ AS_PATH
   ⑤ ORIGIN
   ⑥ MED
   ★ ⑦ eBGP > iBGP ★       ← 根本轮不到这一步
```

### Step 3：验证故障切换

```cisco
R1(config)# interface GigabitEthernet0/1
R1(config-if)# shutdown
```

```cisco
R2# show ip bgp 200.1.1.0
  200
    10.0.22.1 from 10.0.22.1
      Origin IGP, localpref 100, valid, external, ★ best ★
                                                    ↑ 自动切到联通 ✓
```

**恢复后自动切回**。

---

## Part 4：控制入向流量（AS-Path Prepend）

### 目标

**让外部访问我们的流量优先从电信进来。**

### Step 1：在连联通的 R2 上做 Prepend

```cisco
R2(config)# route-map PREPEND-TO-UNICOM permit 10
R2(config-route-map)#  ★ set as-path prepend 65001 65001 65001 ★

R2(config)# router bgp 65001
R2(config-router)# neighbor 10.0.22.1 route-map PREPEND-TO-UNICOM ★ out ★
R2# clear ip bgp 10.0.22.1 soft out
```

### Step 2：验证

```cisco
R2# show ip bgp neighbors 10.0.22.1 ★ advertised-routes ★
   Network          Next Hop     Metric LocPrf Weight Path
*> 192.168.0.0/22   10.0.22.2         0         32768 ★ 65001 65001 65001 ★ i
                                                        ↑ Prepend 生效 ✓
```

**在 ISP-2 上看到的**：
```
   经电信路径：  100 65001                        ← AS-Path 长度 2
   经联通路径：  200 65001 65001 65001 65001      ← AS-Path 长度 5
        ↓
   ★ AS-Path 短的赢 → 外部优先从电信进来 ✓ ★
```

### Step 3：为什么不用 MED

```cisco
! 试试 MED
R2(config)# route-map SET-MED permit 10
R2(config-route-map)#  set metric 500
R2(config-router)# neighbor 10.0.22.1 route-map SET-MED out
```

**在 ISP-2 上**：如果 ISP-2 对这条路由设了更高的 LOCAL_PREF，**MED 完全不起作用**（LOCAL_PREF 在第 2 步，MED 在第 6 步）。

**入向控制手段可靠性排序**：

| 方法 | 选路步骤 | 可靠性 |
|:--|:--|:--|
| **只向一个运营商通告** | — | ★★★★★ 最彻底（但失去冗余） |
| **AS-Path Prepend** | **4** | ★★★★ 常用 |
| **运营商提供的 Community** | 影响对方的 **2** | ★★★★ 最精确（需运营商支持） |
| MED | 6 | ★★ 经常不生效 |

**运营商 Community 方案**：
```cisco
! 运营商文档：设置 community 200:120 = 把 LOCAL_PREF 降到 120
R2(config)# route-map LOWER-PREF permit 10
R2(config-route-map)#  set community 200:120
R2(config-router)# neighbor 10.0.22.1 route-map LOWER-PREF out
R2(config-router)# ★ neighbor 10.0.22.1 send-community ★       ! ★ 必须开
```

---

## Part 5：★ 亲手踩坑 —— route-map 隐含 deny

### Step 1：制造问题

```cisco
R1(config)# ip prefix-list IMPORTANT seq 5 permit 200.1.1.0/24

R1(config)# route-map POLICY permit 10
R1(config-route-map)#  match ip address prefix-list IMPORTANT
R1(config-route-map)#  set local-preference 300
! ★ 故意不加兜底 ★

R1(config)# router bgp 65001
R1(config-router)# neighbor 10.0.11.1 route-map POLICY in
R1# clear ip bgp 10.0.11.1 soft in
```

### Step 2：观察灾难

```cisco
R1# show ip bgp summary
Neighbor    V   AS  MsgRcvd MsgSent TblVer InQ OutQ Up/Down State/PfxRcd
10.0.11.1   4  100     2451    2438      8   0    0 02:15:33   ★ 1 ★
                                                                 ↑
                                    ★ 本来 1245 条，现在只剩 1 条！★
```

```cisco
R1# show ip bgp neighbors 10.0.11.1 received-routes | count
Number of lines which match regexp = ★ 1248 ★           ← 原始收到 1245 条

R1# show ip bgp neighbors 10.0.11.1 routes | count
Number of lines which match regexp = ★ 4 ★              ← 过滤后只剩 1 条
```

**★ 这个对比（received-routes vs routes）能一眼看出是被策略过滤了。**

### Step 3：修复

```cisco
R1(config)# ★ route-map POLICY permit 20 ★
! ★ 空的 permit 语句 ★
!   没有 match = 匹配所有剩余的
!   没有 set   = 不修改任何属性
!   → 原样放行

R1# clear ip bgp 10.0.11.1 soft in
```

```cisco
R1# show ip bgp summary | include 10.0.11.1
10.0.11.1   4  100   2451  2438   8  0  0 02:15:33  ★ 1245 ★     ← 恢复 ✓
```

**验证 route-map 结构**：
```cisco
R1# show route-map POLICY
route-map POLICY, permit, sequence 10
  Match clauses:
    ip address prefix-lists: IMPORTANT
  Set clauses:
    local-preference 300
  Policy routing matches: 0 packets, 0 bytes
★ route-map POLICY, permit, sequence 20 ★
  ★ Match clauses: ★                              ← 空 = 匹配所有
  ★ Set clauses: ★                                ← 空 = 不修改
  Policy routing matches: 0 packets, 0 bytes
```

**★ 写 route-map 的三条铁律**：
1. **永远加兜底的 `permit` 语句**
2. **改完立即对比前缀数**
3. **生产环境用 `configure terminal revert timer` 保险**

**同样的陷阱也存在于**：prefix-list、ACL、as-path access-list（末尾都有隐含 deny）。

---

## Part 6：路由聚合与过滤

### Step 1：聚合

```cisco
! 先造几条明细
R1(config)# ip route 192.168.0.0 255.255.255.0 Null0
R1(config)# ip route 192.168.1.0 255.255.255.0 Null0
R1(config)# ip route 192.168.2.0 255.255.255.0 Null0
R1(config)# ip route 192.168.3.0 255.255.255.0 Null0

R1(config)# router bgp 65001
R1(config-router)# network 192.168.0.0 mask 255.255.255.0
R1(config-router)# network 192.168.1.0 mask 255.255.255.0
R1(config-router)# network 192.168.2.0 mask 255.255.255.0
R1(config-router)# network 192.168.3.0 mask 255.255.255.0

! ★ 聚合，抑制明细 ★
R1(config-router)# ★ aggregate-address 192.168.0.0 255.255.252.0 summary-only ★
```

**验证**：
```cisco
R1# show ip bgp
     Network          Next Hop     Metric LocPrf Weight Path
 ★*> ★ 192.168.0.0/22   0.0.0.0                    32768 i     ← 聚合路由
 ★s>★ 192.168.0.0/24   0.0.0.0          0         32768 i     ← s = suppressed
 s>   192.168.1.0/24   0.0.0.0          0         32768 i
 s>   192.168.2.0/24   0.0.0.0          0         32768 i
 s>   192.168.3.0/24   0.0.0.0          0         32768 i

R1# show ip route | include Null0
★ B    192.168.0.0/22 [200/0] via 0.0.0.0, 00:02:15, Null0 ★
! ★ BGP 自动生成的防环路由 ★
```

**验证 Null0 防环的作用**：
```cisco
R1# ping 192.168.99.99                          ! /22 范围外
! 走默认路由

! 假设有个 192.168.2.99（在 /22 内，但明细路由被 shutdown 了）
R1(config)# no ip route 192.168.2.0 255.255.255.0 Null0
R1# ping 192.168.2.99
! ★ 匹配 192.168.0.0/22 的 Null0，直接丢弃，不会往上游弹 ★
```

### Step 2：`as-set` 对比

```cisco
R1(config-router)# ★ aggregate-address 192.168.0.0 255.255.252.0 summary-only as-set ★
```

```cisco
R1# show ip bgp 192.168.0.0/22
BGP routing table entry for 192.168.0.0/22
  Paths: (1 available, best #1)
  Local, (aggregated by 65001 1.1.1.1)
    0.0.0.0 from 0.0.0.0 (1.1.1.1)
      Origin IGP, localpref 100, weight 32768, valid, aggregated, local, best
      ! ★ 加了 as-set 后，atomic-aggregate 标记消失 ★
```

**如果聚合的明细来自不同 AS**，会看到：
```cisco
 *>  192.168.0.0/22   0.0.0.0    ...  ★ {200,300,400} ★ i
                                        ↑ 花括号 = AS_SET
```

### Step 3：前缀过滤（防止收到垃圾路由）

```cisco
R1(config)# ip prefix-list SANE-IN seq 5  ★ deny 0.0.0.0/0 ★              ! 拒默认路由
R1(config)# ip prefix-list SANE-IN seq 10 deny 0.0.0.0/8 le 32
R1(config)# ip prefix-list SANE-IN seq 15 deny 10.0.0.0/8 le 32           ! 拒私网
R1(config)# ip prefix-list SANE-IN seq 20 deny 127.0.0.0/8 le 32
R1(config)# ip prefix-list SANE-IN seq 25 deny 169.254.0.0/16 le 32
R1(config)# ip prefix-list SANE-IN seq 30 deny 172.16.0.0/12 le 32
R1(config)# ip prefix-list SANE-IN seq 35 deny 192.168.0.0/16 le 32
R1(config)# ip prefix-list SANE-IN seq 40 deny 224.0.0.0/4 le 32          ! 拒组播
R1(config)# ip prefix-list SANE-IN seq 45 ★ deny 0.0.0.0/0 ge 25 ★        ! 拒 /25 以上碎片
R1(config)# ip prefix-list SANE-IN seq 50 ★ permit 0.0.0.0/0 le 24 ★      ! ★ 放行其余

R1(config-router)# neighbor 10.0.11.1 prefix-list SANE-IN in
R1# clear ip bgp 10.0.11.1 soft in
```

**验证命中**：
```cisco
R1# show ip prefix-list detail SANE-IN
ip prefix-list SANE-IN:
   count: 10, range entries: 8, sequences: 5 - 50, refcount: 2
   seq 5 deny 0.0.0.0/0 ★ (hit count: 2, refcount: 1) ★
   seq 15 deny 10.0.0.0/8 le 32 ★ (hit count: 45) ★           ← 拦到了私网
   seq 50 permit 0.0.0.0/0 le 24 (hit count: 1198)
```

### Step 4：只通告自己的网段（★ 防止变成中转 AS）

```cisco
R1(config)# ip prefix-list MY-NETS-OUT seq 5 permit 192.168.0.0/22
R1(config-router)# neighbor 10.0.11.1 prefix-list MY-NETS-OUT out
R1# clear ip bgp 10.0.11.1 soft out
```

**验证**：
```cisco
R1# show ip bgp neighbors 10.0.11.1 advertised-routes
   Network          Next Hop     Metric LocPrf Weight Path
*> 192.168.0.0/22   10.0.11.2         0         32768 i
! ★ 只通告一条，不会把从联通学到的路由转发给电信 ★
```

**★ 不做这个过滤的风险（transit AS 泄露）**：
```
   你从电信学到全表，又通告给联通
   联通从你这里学到去电信的路由（AS-Path 更短的话）
        ↓
   ★ 联通的流量会经过你的出口去电信 ★
        ↓
   ★ 你的出口带宽被别人的流量占满 ★
```
**历史上这造成过多次大规模互联网故障。**

### Step 5：AS-Path 过滤

```cisco
! 只接受直连 AS 100 自己产生的路由
R1(config)# ★ ip as-path access-list 1 permit ^100$ ★
R1(config-router)# neighbor 10.0.11.1 filter-list 1 in
R1# clear ip bgp 10.0.11.1 soft in
```

**测试正则**：
```cisco
R1# show ip bgp ★ regexp ^100$ ★
! 只显示 AS_PATH 正好是 "100" 的

R1# show ip bgp regexp _200_
! 显示路径中包含 AS 200 的

R1# show ip bgp regexp ^$
! 显示本 AS 产生的
```

**正则速查**：

| 正则 | 含义 |
|:--|:--|
| **`^$`** | **本 AS 产生的**（AS_PATH 为空） |
| **`^100$`** | 只经过 AS 100 |
| **`^100_`** | AS 100 是第一跳 |
| **`_300$`** | AS 300 产生的 |
| **`_300_`** | 路径中包含 AS 300 |
| **`.*`** | 匹配所有 |

---

## Part 7：Route Reflector

### Step 1：把 R3 配成 RR

```cisco
R3(config)# router bgp 65001
R3(config-router)# ★ bgp cluster-id 3.3.3.3 ★
R3(config-router)# neighbor 1.1.1.1 remote-as 65001
R3(config-router)# neighbor 1.1.1.1 update-source Loopback0
R3(config-router)# ★ neighbor 1.1.1.1 route-reflector-client ★
R3(config-router)# neighbor 2.2.2.2 remote-as 65001
R3(config-router)# neighbor 2.2.2.2 update-source Loopback0
R3(config-router)# ★ neighbor 2.2.2.2 route-reflector-client ★

! ★ R1 和 R2 之间的直接 iBGP 会话可以拆掉了 ★
R1(config-router)# no neighbor 2.2.2.2
R2(config-router)# no neighbor 1.1.1.1
```

### Step 2：验证反射

```cisco
R2# show ip bgp 200.1.1.0
BGP routing table entry for 200.1.1.0/24
  100
    1.1.1.1 (metric 20) from ★ 3.3.3.3 ★ (3.3.3.3)
                              ↑ 从 RR 收到的
      Origin IGP, localpref 200, valid, internal, best
      ★ Originator: 1.1.1.1, Cluster list: 3.3.3.3 ★
        ↑                     ↑
    最初产生者              经过的 RR 集群
```

**★ 这两个属性是 RR 的防环机制**：
- **`Originator`**：收到 Originator 是自己的路由 → 丢弃
- **`Cluster list`**：RR 收到含自己 Cluster-ID 的 → 丢弃

**Client 完全不知道自己是 Client**（配置上没有任何特殊之处）——这是 RR 设计的优雅之处，可以渐进式部署。

---

## Part 8：故障注入

### 故障 A：Active 状态（TCP 179 被拦）

```cisco
ISP-1(config)# ip access-list extended BLOCK-BGP
ISP-1(config-ext-nacl)#  deny tcp any any eq 179
ISP-1(config-ext-nacl)#  deny tcp any eq 179 any
ISP-1(config-ext-nacl)#  permit ip any any
ISP-1(config)# interface GigabitEthernet0/1
ISP-1(config-if)# ip access-group BLOCK-BGP in
```

<details><summary>症状与排查</summary>

```cisco
R1# show ip bgp summary
Neighbor    V   AS  MsgRcvd MsgSent TblVer InQ OutQ Up/Down State/PfxRcd
10.0.11.1   4  100        0       0      0   0    0 never   ★ Active ★

! ★ 关键的二分判断 ★
R1# ping 10.0.11.1
!!!!!                                              ← ping 通！

R1# ★ telnet 10.0.11.1 179 ★
Trying 10.0.11.1, 179 ...
% ★ Connection timed out ★                        ← ★ TCP 179 不通
```

**✅ "ping 通但 telnet 179 超时" 直接定位到 ACL/防火墙问题。**

**三种 telnet 结果的含义**：

| 结果 | 含义 |
|:--|:--|
| **Open** | TCP 通，问题在 BGP 配置层面 |
| **Connection refused** | 收到 RST —— 对端没跑 BGP |
| **超时** | ★ 被 ACL/防火墙**静默丢弃** |

**★ `Active` 不是"活跃正常"，是"TCP 建立失败正在重试"。**
</details>

### 故障 B：缺 `update-source`

```cisco
R1(config-router)# no neighbor 3.3.3.3 update-source Loopback0
R1# clear ip bgp 3.3.3.3
```

<details><summary>症状与排查</summary>

```cisco
R1# show ip bgp summary | include 3.3.3.3
3.3.3.3   4 65001    0    0    0  0  0 never  ★ Active ★

R1# debug ip bgp
BGP: 3.3.3.3 open active, local address ★ 10.0.13.1 ★
                                          ↑ 用的是出接口 IP，不是 Loopback
! R3 配的 neighbor 是 R1 的 Loopback (1.1.1.1)
! → 源地址不匹配 → R3 拒绝
```

**修复**：`neighbor 3.3.3.3 update-source Loopback0`
</details>

### 故障 C：RIB-failure

```cisco
R1(config)# ip route 200.1.1.0 255.255.255.0 10.0.13.3    ! 静态路由，AD=1
```

<details><summary>症状与排查</summary>

```cisco
R1# show ip bgp | include 200.1.1.0
 ★ r> ★ 200.1.1.0/24   10.0.11.1   0   0 100 i
   ↑
 RIB-failure

R1# ★ show ip bgp rib-failure ★
Network          Next Hop        RIB-failure              RIB-NH Matches
200.1.1.0/24     10.0.11.1       ★ Higher admin distance ★  n/a

R1# show ip route 200.1.1.0
Routing entry for 200.1.1.0/24
  Known via "★ static ★", distance ★ 1 ★            ← 静态 AD=1 < eBGP AD=20
```

**★ 影响（最关键的一点）**：
```
   BGP 依然会把这条路由【通告给邻居】（它是 BGP 的最优路径）
   但本机转发时用的是那条静态路由
        ↓
   ★ 控制平面和数据平面不一致 ★
        ↓
   如果静态路由指向错误的方向 → 黑洞或环路
```

**修复**：
```cisco
! 方案 1：删掉冲突的静态路由
R1(config)# no ip route 200.1.1.0 255.255.255.0 10.0.13.3

! 方案 2：调高静态路由的 AD
R1(config)# ip route 200.1.1.0 255.255.255.0 10.0.13.3 250

! ★ 方案 3：让 BGP 不通告 RIB-failure 的路由（推荐，保持一致性）
R1(config-router)# ★ bgp suppress-inactive ★
```

**★ 巡检建议**：把 `show ip bgp rib-failure` 加进脚本，正常应无输出。
</details>

### 故障 D：缺 `send-community`

```cisco
R1(config)# route-map SET-COMM permit 10
R1(config-route-map)#  set community 65001:100
R1(config-router)# neighbor 3.3.3.3 route-map SET-COMM out
! ★ 故意不配 send-community ★
R1# clear ip bgp 3.3.3.3 soft out
```

<details><summary>症状与排查</summary>

```cisco
R3# show ip bgp 200.1.1.0
BGP routing table entry for 200.1.1.0/24
  100
    1.1.1.1 from 1.1.1.1 (1.1.1.1)
      Origin IGP, localpref 200, valid, internal, best
      ! ★ 没有 Community 属性！★
```

**修复**：
```cisco
R1(config-router)# ★ neighbor 3.3.3.3 send-community ★
! 或 send-community both（包含扩展团体）
```

```cisco
R3# show ip bgp 200.1.1.0 | include Community
  ★ Community: 65001:100 ★                        ← 现在有了 ✓
```

**★ Cisco 默认不发送 Community 属性。** 基于 Community 的策略全都依赖这条命令，**忘配是很隐蔽的错误**。
</details>

### 故障 E：prefix-list 写错（`0.0.0.0/0` vs `0.0.0.0/0 le 32`）

```cisco
R1(config)# ip prefix-list WRONG seq 5 ★ permit 0.0.0.0/0 ★
!                                                  ↑ 少了 le 32
R1(config)# route-map POLICY2 permit 10
R1(config-route-map)#  match ip address prefix-list WRONG
R1(config-router)# neighbor 10.0.11.1 route-map POLICY2 in
R1# clear ip bgp 10.0.11.1 soft in
```

<details><summary>症状与排查</summary>

```cisco
R1# show ip bgp summary | include 10.0.11.1
10.0.11.1   4  100   2451  2438   8  0  0 02:15:33  ★ 0 ★
!                                                     ↑ 一条都没收到
```

**原因**：
```
   permit 0.0.0.0/0        →  ★ 只匹配默认路由本身 ★
   permit 0.0.0.0/0 le 32  →  匹配所有前缀
        ↓
   运营商没发默认路由 → 什么都不匹配 → 隐含 deny 拦掉全部
```

**修复**：
```cisco
R1(config)# ip prefix-list WRONG seq 5 permit 0.0.0.0/0 ★ le 32 ★
```

**★ 这是 prefix-list 最常见的错误。** 记住：
| 写法 | 含义 |
|:--|:--|
| `permit 0.0.0.0/0` | **只有默认路由** |
| `permit 0.0.0.0/0 le 32` | **所有前缀** |
</details>

### 故障 F：maximum-prefix 触发

```cisco
R1(config-router)# neighbor 10.0.11.1 maximum-prefix ★ 10 ★
R1# clear ip bgp 10.0.11.1
```

<details><summary>症状与排查</summary>

```cisco
R1# show logging | include PFX
★ %BGP-4-MAXPFX: No. of prefix received from 10.0.11.1 (afi 0) reaches 9, max 10 ★
★ %BGP-3-MAXPFXEXCEED: No. of prefix received from 10.0.11.1 (afi 0): 11 exceed limit 10 ★
★ %BGP-5-ADJCHANGE: neighbor 10.0.11.1 Down BGP Notification sent ★

R1# show ip bgp summary | include 10.0.11.1
10.0.11.1  4  100  ... ★ Idle (PfxCt) ★
                        ↑ 因前缀超限被关闭
```

**恢复**：
```cisco
R1(config-router)# neighbor 10.0.11.1 maximum-prefix 1000 80 ★ restart 5 ★
!                                                       ↑    ↑
!                                                    80%告警  5分钟后自动重连
R1# clear ip bgp 10.0.11.1
```

**★ `maximum-prefix` 是必配项**：如果运营商配置失误向你通告全表（90 万条），**你的路由器内存会瞬间耗尽并崩溃**。这在实际中发生过多次。
</details>

---

## 实验检查清单

```
□ ① eBGP 和 iBGP 邻居全部 Established
□ ② ★ 验证了 next-hop-self 的必要性（对比有无）
□ ③ 用 LOCAL_PREF 控制出向，验证 iBGP 路径赢过 eBGP
□ ④ 用 AS-Path Prepend 控制入向，验证 advertised-routes
□ ⑤ ★ 亲手踩了 route-map 隐含 deny 的坑并修复
□ ⑥ 配置聚合，验证 s（suppressed）标记和 Null0 防环路由
□ ⑦ 配置前缀过滤，验证 hit count
□ ⑧ 配置 RR，验证 Originator 和 Cluster list
□ ⑨ 六个故障都亲手制造并修复
□ ⑩ 验证了故障切换（断电信自动走联通）
□ ⑪ 写了实验笔记
```

## 关键命令总结

| 命令 | 用途 |
|:--|:--|
| **`show ip bgp summary`** | ★ 第一命令：邻居状态和前缀数 |
| **`show ip bgp <前缀>`** | ★ 看选路原因（对照 11 步） |
| **`telnet <邻居> 179`** | ★ Active 状态的关键排查 |
| `show ip bgp neighbors X received-routes` | 原始收到的（需 soft-reconfiguration） |
| `show ip bgp neighbors X routes` | 过滤后的 |
| `show ip bgp neighbors X advertised-routes` | 我发出去的 |
| **`show ip bgp rib-failure`** | ★ 巡检必查 |
| `show route-map` | 验证策略结构（★ 有没有兜底） |
| `show ip prefix-list detail` | hit count 验证规则命中 |
| **`clear ip bgp X soft in/out`** | ★ 软重置，永远不要用 `clear ip bgp *` |

## 标记速查

| 标记 | 含义 | 排障意义 |
|:--|:--|:--|
| `*` | valid（下一跳可达） | **没有 `*` = 下一跳不可达** |
| `>` | best | 只有 `*>` 进路由表 |
| `i`（第3列） | 从 iBGP 学到 | — |
| **`r`** | **RIB-failure** | ★ 有 AD 更低的路由 |
| `s` | suppressed | 被聚合抑制 |
| 什么都没有 | — | ★ 下一跳不可达 |

---

**上一个** ← [Lab 05](lab-05-EIGRP.md) ｜ **下一个** → [Lab 07: 路由重分发环路](lab-07-重分发与路由策略.md)
