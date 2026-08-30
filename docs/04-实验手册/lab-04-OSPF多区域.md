# Lab 04 · OSPF 多区域与特殊区域

**对应章节**：[ENCOR 07](../02-ENCOR-350-401/07-OSPF进阶.md)、[ENARSI 02](../03-ENARSI-300-410/02-OSPF深入与排障.md)
**难度**：★★★　**时长**：3 小时

## 目标

1. 搭建三区域 OSPF，**观察各类 LSA 的产生和传播范围**
2. 依次配置 Stub、Totally Stub、NSSA，对比路由表变化
3. 配置区域间汇总，**验证汇总的故障隔离价值**
4. 配置虚链路
5. 制造并排查 6 个典型故障

---

## 拓扑

```
   Area 1                  Area 0                  Area 2 (后改 NSSA)
                                                          
  [R3]────────[R1/ABR]────────[R2/ABR]────────[R4]────[R5/ASBR]
   │                                                      │
Lo1:192.168.10.0/24                              [静态路由 200.1.1.0/24]
Lo2:192.168.11.0/24
Lo3:192.168.12.0/24
Lo4:192.168.13.0/24
```

| 设备 | Router ID | 角色 |
|:--|:--|:--|
| R1 | 1.1.1.1 | **ABR**（Area 0 + Area 1） |
| R2 | 2.2.2.2 | **ABR**（Area 0 + Area 2） |
| R3 | 3.3.3.3 | Area 1 内部 |
| R4 | 4.4.4.4 | Area 2 内部 |
| R5 | 5.5.5.5 | **ASBR**（Area 2，重分发静态路由） |

| 链路 | 网段 | 区域 |
|:--|:--|:--|
| R1-R2 | 10.0.12.0/30 | Area 0 |
| R1-R3 | 10.0.13.0/30 | Area 1 |
| R2-R4 | 10.0.24.0/30 | Area 2 |
| R4-R5 | 10.0.45.0/30 | Area 2 |

---

## Part 1：基础多区域

### Step 1：配置

```cisco
! ══════ R1（ABR）══════
R1(config)# interface Loopback0
R1(config-if)# ip address 1.1.1.1 255.255.255.255

R1(config)# router ospf 1
R1(config-router)# ★ router-id 1.1.1.1 ★
R1(config-router)# ★ auto-cost reference-bandwidth 100000 ★    ! 全网统一
R1(config-router)# passive-interface default
R1(config-router)# no passive-interface GigabitEthernet0/1     ! 朝 R2
R1(config-router)# no passive-interface GigabitEthernet0/2     ! 朝 R3
R1(config-router)# network 1.1.1.1 0.0.0.0 area 0
R1(config-router)# network 10.0.12.1 0.0.0.0 area 0
R1(config-router)# network 10.0.13.1 0.0.0.0 ★ area 1 ★

! ══════ R3（Area 1 内部）══════
R3(config)# interface Loopback0
R3(config-if)# ip address 3.3.3.3 255.255.255.255
R3(config)# interface Loopback1
R3(config-if)# ip address 192.168.10.1 255.255.255.0
R3(config)# interface Loopback2
R3(config-if)# ip address 192.168.11.1 255.255.255.0
R3(config)# interface Loopback3
R3(config-if)# ip address 192.168.12.1 255.255.255.0
R3(config)# interface Loopback4
R3(config-if)# ip address 192.168.13.1 255.255.255.0

R3(config)# router ospf 1
R3(config-router)# router-id 3.3.3.3
R3(config-router)# auto-cost reference-bandwidth 100000
R3(config-router)# passive-interface default
R3(config-router)# no passive-interface GigabitEthernet0/1
R3(config-router)# network 10.0.13.3 0.0.0.0 area 1
R3(config-router)# network 3.3.3.3 0.0.0.0 area 1
R3(config-router)# network 192.168.10.0 0.0.3.255 area 1      ! 一次通告四个 /24

! ══════ R2（ABR）══════
R2(config)# router ospf 1
R2(config-router)# router-id 2.2.2.2
R2(config-router)# auto-cost reference-bandwidth 100000
R2(config-router)# network 2.2.2.2 0.0.0.0 area 0
R2(config-router)# network 10.0.12.2 0.0.0.0 area 0
R2(config-router)# network 10.0.24.2 0.0.0.0 ★ area 2 ★

! ══════ R4（Area 2 内部）══════
R4(config)# router ospf 1
R4(config-router)# router-id 4.4.4.4
R4(config-router)# auto-cost reference-bandwidth 100000
R4(config-router)# network 10.0.24.4 0.0.0.0 area 2
R4(config-router)# network 10.0.45.4 0.0.0.0 area 2
R4(config-router)# network 4.4.4.4 0.0.0.0 area 2

! ══════ R5（Area 2，暂不做 ASBR）══════
R5(config)# router ospf 1
R5(config-router)# router-id 5.5.5.5
R5(config-router)# auto-cost reference-bandwidth 100000
R5(config-router)# network 10.0.45.5 0.0.0.0 area 2
R5(config-router)# network 5.5.5.5 0.0.0.0 area 2
```

### Step 2：★ 核心观察 —— LSA 的传播范围

**在 R3（Area 1 内部）上看**：
```cisco
R3# show ip ospf database

            OSPF Router with ID (3.3.3.3) (Process ID 1)

                ★ Router Link States (Area 1) ★              ← Type-1
Link ID         ADV Router      Age  Seq#       Checksum Link count
1.1.1.1         1.1.1.1         245  0x80000005 0x00A1B2  2
3.3.3.3         3.3.3.3         238  0x80000004 0x00C3D4  6

                ★ Summary Net Link States (Area 1) ★         ← Type-3
Link ID         ADV Router      Age  Seq#       Checksum
1.1.1.1         1.1.1.1         240  0x80000002 0x001234
2.2.2.2         1.1.1.1         238  0x80000002 0x005678
4.4.4.4         1.1.1.1         235  0x80000002 0x009ABC
5.5.5.5         1.1.1.1         233  0x80000002 0x00DEF0
10.0.12.0       1.1.1.1         240  0x80000002 0x00A1B2
10.0.24.0       1.1.1.1         238  0x80000002 0x00C3D4
10.0.45.0       1.1.1.1         236  0x80000002 0x00E5F6
```

**★ 关键观察**：
- R3 **只看到 Area 1 的 Type-1 LSA**（1.1.1.1 和 3.3.3.3）
- **看不到 Area 0 和 Area 2 的 Type-1 LSA** ← ★ 区域隔离生效 ★
- 通过 **Type-3 LSA** 知道其他区域有哪些网段（但不知道拓扑）

**在 R1（ABR）上看**：
```cisco
R1# show ip ospf database | include Area
                Router Link States (★ Area 0 ★)
                Net Link States (Area 0)
                Summary Net Link States (Area 0)
                Router Link States (★ Area 1 ★)
                Summary Net Link States (Area 1)
                ↑ ★ ABR 同时维护两个区域的 LSDB ★
```

**路由表**：
```cisco
R3# show ip route ospf
★ O IA ★  10.0.12.0/30 [110/200] via 10.0.13.1, 00:05:12, Gi0/1
  O IA   10.0.24.0/30 [110/300] via 10.0.13.1, 00:05:12, Gi0/1
  O IA   10.0.45.0/30 [110/400] via 10.0.13.1, 00:05:12, Gi0/1
  O IA   1.1.1.1/32   [110/101] via 10.0.13.1, 00:05:12, Gi0/1
  O IA   2.2.2.2/32   [110/201] via 10.0.13.1, 00:05:12, Gi0/1
  ↑↑
★ IA = Inter-Area（区域间路由，来自 Type-3 LSA）★
```

---

## Part 2：引入外部路由，观察 Type-4 和 Type-5

### Step 1：R5 成为 ASBR

```cisco
R5(config)# ip route 200.1.1.0 255.255.255.0 Null0
R5(config)# ip route 200.1.2.0 255.255.255.0 Null0
R5(config)# ip route 200.1.3.0 255.255.255.0 Null0

R5(config)# router ospf 1
R5(config-router)# ★ redistribute static subnets ★
```

### Step 2：观察 Type-5（外部路由）

```cisco
R3# show ip ospf database external

            OSPF Router with ID (3.3.3.3) (Process ID 1)

                ★ Type-5 AS External Link States ★

  LS age: 45
  Options: (No TOS-capability, DC)
  LS Type: AS External Link
  Link State ID: 200.1.1.0 (External Network Number)
  ★ Advertising Router: 5.5.5.5 ★                    ← ASBR
  LS Seq Number: 80000001
  Network Mask: /24
        ★ Metric Type: 2 (Larger than any link state path) ★
        Metric: 20
```

**★ 关键**：Type-5 LSA **泛洪到整个 AS**，不受区域限制。R3 在 Area 1，却能收到 Area 2 里 ASBR 产生的 Type-5。

### Step 3：观察 Type-4（怎么到达 ASBR）

```cisco
R3# show ip ospf database asbr-summary

                ★ Summary ASB Link States (Area 1) ★

  LS Type: Summary Links (AS Boundary Router)
  ★ Link State ID: 5.5.5.5 (AS Boundary Router address) ★
  ★ Advertising Router: 1.1.1.1 ★                    ← ABR 告诉 Area 1 怎么到 ASBR
  TOS: 0  Metric: 300
```

**★ 理解 Type-4 的必要性**：
```
   R3 收到 Type-5，知道"有一条外部路由 200.1.1.0/24，是 5.5.5.5 通告的"
        ↓
   ★ 但 R3 不知道怎么到达 5.5.5.5 ★
      （5.5.5.5 在 Area 2，它的 Type-1 LSA 传不到 Area 1）
        ↓
   ★ Type-4 就是解决这个的 ★
      ABR（R1）告诉 Area 1："我知道怎么到 ASBR 5.5.5.5，cost 是 300"
```

**路由表**：
```cisco
R3# show ip route ospf | include 200.1
★ O E2 ★ 200.1.1.0/24 [110/★20★] via 10.0.13.1, 00:02:15, Gi0/1
  O E2   200.1.2.0/24 [110/20] via 10.0.13.1, 00:02:15, Gi0/1
  O E2   200.1.3.0/24 [110/20] via 10.0.13.1, 00:02:15, Gi0/1
  ↑↑                          ↑
外部路由 E2            ★ metric 只有 20，不加内部 cost ★
```

### Step 4：对比 E1 和 E2

```cisco
R5(config)# router ospf 1
R5(config-router)# redistribute static subnets ★ metric-type 1 ★
```

```cisco
R3# show ip route ospf | include 200.1
★ O E1 ★ 200.1.1.0/24 [110/★520★] via 10.0.13.1, ...
  ↑↑                          ↑
                    ★ 20（外部）+ 500（到 ASBR 的内部 cost）★
```

**对比表**：

| 类型 | metric 计算 | 何时用 |
|:--|:--|:--|
| **E2**（默认） | 只算外部 cost | 单个 ASBR，metric 稳定 |
| **E1** | 外部 cost + 到 ASBR 的内部 cost | ★ **多个 ASBR**，能选最近的出口 |

**★ E1 永远优于 E2**（无论 metric 大小）。

---

## Part 3：特殊区域实验（★ 逐步对比路由表）

### Step 1：记录基线（标准区域）

```cisco
R3# show ip route ospf | count
Number of lines which match regexp = ★ 12 ★         ← 记下这个数字

R3# show ip route ospf
O IA  1.1.1.1/32 ...
O IA  2.2.2.2/32 ...
O IA  4.4.4.4/32 ...
O IA  5.5.5.5/32 ...
O IA  10.0.12.0/30 ...
O IA  10.0.24.0/30 ...
O IA  10.0.45.0/30 ...
O E2  200.1.1.0/24 ...
O E2  200.1.2.0/24 ...
O E2  200.1.3.0/24 ...
```

### Step 2：改成 Stub

```cisco
! ★ 区域内所有路由器都要配 ★
R1(config)# router ospf 1
R1(config-router)# ★ area 1 stub ★

R3(config)# router ospf 1
R3(config-router)# ★ area 1 stub ★
```

**观察变化**：
```cisco
R3# show ip ospf database external
! ★ 空输出！Type-5 被拒绝了 ✓ ★

R3# show ip route ospf
O IA  1.1.1.1/32 ...
O IA  2.2.2.2/32 ...
O IA  10.0.12.0/30 ...
O IA  10.0.24.0/30 ...
O IA  10.0.45.0/30 ...
★ O*IA  0.0.0.0/0 [110/101] via 10.0.13.1, Gi0/1 ★     ← ABR 自动注入的默认路由
  ↑
  * 表示这是候选默认路由

R3# show ip route ospf | count
Number of lines which match regexp = ★ 8 ★              ← 从 12 降到 8
```

**验证连通性**：
```cisco
R3# ping 200.1.1.1 source Loopback0
!!!!!                                                    ← 依然通 ✓（走默认路由）
```

**★ 结论：路由表小了，但连通性不变。**

**确认区域类型**：
```cisco
R3# show ip ospf | include Area 1 -A 5
    Area 1
        Number of interfaces in this area is 6
        ★ It is a stub area ★
        generates stub default route with cost 1
```

### Step 3：改成 Totally Stub

```cisco
! ★ no-summary 只在 ABR 上配 ★
R1(config-router)# ★ area 1 stub no-summary ★
! R3 保持 area 1 stub 不变
```

**观察变化**：
```cisco
R3# show ip route ospf
★ O*IA  0.0.0.0/0 [110/101] via 10.0.13.1, 00:01:30, Gi0/1 ★
! ★★ 只剩一条默认路由！★★

R3# show ip route ospf | count
Number of lines which match regexp = ★ 1 ★              ← 从 12 降到 1！

R3# show ip ospf database summary
! 只有一条 0.0.0.0 的 Type-3
```

**验证连通性**：
```cisco
R3# ping 200.1.1.1 source Loopback0
!!!!!                                                    ← 依然通 ✓
R3# ping 2.2.2.2 source Loopback0
!!!!!                                                    ← 依然通 ✓
```

**★★ 这就是 Totally Stub 的威力：路由表从 12 条降到 1 条，连通性完全不变。★★**

**在分支路由器上，这可能是从几百条降到 1 条。**

### Step 4：Area 2 改成 NSSA

```cisco
R2(config-router)# ★ area 2 nssa ★
R4(config-router)# ★ area 2 nssa ★
R5(config-router)# ★ area 2 nssa ★
```

**观察 Type-7**：
```cisco
R4# show ip ospf database nssa-external

                ★ Type-7 AS External Link States (Area 2) ★

  LS Type: AS External Link
  Link State ID: 200.1.1.0
  ★ Advertising Router: 5.5.5.5 ★                     ← NSSA 内的 ASBR
        Metric Type: 2
        Metric: 20

R4# show ip route ospf | include 200.1
★ O N2 ★ 200.1.1.0/24 [110/20] via 10.0.45.5, 00:01:15, Gi0/2
  ↑↑
★ N2 = NSSA 外部路由（Type-2 metric）★
```

**★ 观察 Type-7 → Type-5 的转换（在 Area 1 的 R3 上）**：

先临时把 Area 1 改回标准区域：
```cisco
R1(config-router)# no area 1 stub
R3(config-router)# no area 1 stub
```

```cisco
R3# show ip ospf database external

                ★ Type-5 AS External Link States ★

  Link State ID: 200.1.1.0
  ★ Advertising Router: 2.2.2.2 ★
                        ↑↑↑↑↑↑↑
        ★ 注意！通告者变成了 ABR（R2），不是原始的 ASBR（5.5.5.5）★
```

**✅ 验证了 Type-7 → Type-5 的转换发生在 NSSA 的 ABR 上。**

### Step 5：NSSA 的默认路由

```cisco
R4# show ip route ospf | include 0.0.0.0
! ★ 空！NSSA 不会自动注入默认路由 ★
```

**★ 这是 NSSA 和 Stub 的一个重要区别。**

**手工注入**：
```cisco
R2(config-router)# ★ area 2 nssa default-information-originate ★
```

```cisco
R4# show ip route ospf | include 0.0.0.0
★ O*N2  0.0.0.0/0 [110/1] via 10.0.24.2, 00:00:30, Gi0/1 ★
     ↑↑
   注意是 N2（NSSA 外部），不是 IA
```

**改成 Totally NSSA**：
```cisco
R2(config-router)# ★ area 2 nssa no-summary ★
! 这时会自动注入默认路由，不需要 default-information-originate
```

```cisco
R4# show ip route ospf
O*IA  0.0.0.0/0 [110/1] via 10.0.24.2, Gi0/1
O N2  200.1.1.0/24 [110/20] via 10.0.45.5, Gi0/2      ← ★ Type-7 依然保留
O N2  200.1.2.0/24 ...
O N2  200.1.3.0/24 ...
! ★ 区域间路由（Type-3）被拒绝了，但 Type-7 保留 ★
```

### Step 6：四种区域类型对比总表

**填写你观察到的结果**：

| 区域类型 | R3/R4 的 OSPF 路由条数 | Type-3 | Type-4/5 | Type-7 | 默认路由 |
|:--|:--|:--|:--|:--|:--|
| 标准区域 | 12 | ✅ | ✅ | ❌ | ❌ |
| Stub | 8 | ✅ | ❌ | ❌ | ✅ 自动 |
| Totally Stub | ★ 1 ★ | ❌ | ❌ | ❌ | ✅ 自动 |
| NSSA | ? | ✅ | ❌ | ✅ | ⚠️ 需手工 |
| Totally NSSA | ? | ❌ | ❌ | ✅ | ✅ 自动 |

---

## Part 4：路由汇总与故障隔离（★ 最有价值的实验）

### Step 1：汇总前的基线

```cisco
R2# show ip route ospf | include 192.168.1
O IA  192.168.10.0/24 [110/300] via 10.0.12.1, Gi0/1
O IA  192.168.11.0/24 [110/300] via 10.0.12.1, Gi0/1
O IA  192.168.12.0/24 [110/300] via 10.0.12.1, Gi0/1
O IA  192.168.13.0/24 [110/300] via 10.0.12.1, Gi0/1
! 四条明细
```

### Step 2：在 ABR 上汇总

```cisco
R1(config)# router ospf 1
R1(config-router)# ★ area 1 range 192.168.8.0 255.255.248.0 ★
!                             ↑ /21 覆盖 192.168.8.0 - 192.168.15.255
```

```cisco
R2# show ip route ospf | include 192.168
★ O IA  192.168.8.0/21 [110/300] via 10.0.12.1, Gi0/1 ★
! ★ 四条变一条 ✓ ★
```

**验证 ABR 上的防环路由**：
```cisco
R1# show ip route | include Null0
★ O    192.168.8.0/21 is a summary, 00:02:15, Null0 ★
```

**作用**：落在 /21 范围内但没有明细的流量（比如 192.168.9.0/24）直接丢弃，**防止和上游来回弹**。

### Step 3：★ 验证汇总的故障隔离价值（核心实验）

**在 R2 上开启 OSPF 调试**：
```cisco
R2# debug ip ospf events
R2# debug ip ospf lsa-generation
```

**在 R3 上反复 shut/no shut 一个明细网段**：
```cisco
R3(config)# interface Loopback1                    ! 192.168.10.0/24
R3(config-if)# shutdown
! 等 5 秒
R3(config-if)# no shutdown
! 重复 5 次
```

**观察 R2 的反应**：

<details><summary>预期结果</summary>

**汇总前**（如果没配 `area 1 range`）：
```
R2# debug ip ospf events
OSPF-1 EVENT: Schedule SPF in area 0
OSPF-1 EVENT: Begin SPF at ...
OSPF-1 EVENT: End SPF at ...
★ 每次 shut/no shut 都触发一次 SPF ★
```

**汇总后**：
```
R2# debug ip ospf events
! ★ 完全没有输出 ★
! R2 根本不知道 Area 1 里发生了什么
```

**为什么**：只要 `192.168.11/12/13.0` 中还有至少一条明细存在，**汇总路由 `192.168.8.0/21` 就保持不变**，不会产生新的 Type-3 LSA，**其他区域完全无感知**。

**★★ 这才是路由汇总最核心的价值 ★★**

不是"缩小路由表"（那是次要的），而是**"隔离故障，防止本地抖动扩散到全网"**。

一条链路每天抖 100 次：
- 没汇总 → 全网每天重算 100 次 SPF
- 有汇总 → **0 次**
</details>

### Step 4：验证汇总的 cost

```cisco
R1# show ip ospf database summary 192.168.8.0
  Link State ID: 192.168.8.0
  ★ TOS: 0  Metric: 100 ★
                    ↑ 默认取被汇总明细中【最小】的 cost
```

**手工指定 cost**：
```cisco
R1(config-router)# area 1 range 192.168.8.0 255.255.248.0 ★ cost 500 ★
```

### Step 5：抑制某个网段

```cisco
R1(config-router)# area 1 range 192.168.13.0 255.255.255.0 ★ not-advertise ★
```

```cisco
R2# show ip route ospf | include 192.168.13
! ★ 空 —— 被抑制了 ✓ ★
```

---

## Part 5：虚链路

### Step 1：制造"区域不直连 Area 0"的场景

```cisco
! 把 R4-R5 之间改成 Area 3（不直连 Area 0）
R4(config-router)# no network 10.0.45.4 0.0.0.0 area 2
R4(config-router)# ★ network 10.0.45.4 0.0.0.0 area 3 ★

R5(config-router)# no network 10.0.45.5 0.0.0.0 area 2
R5(config-router)# no network 5.5.5.5 0.0.0.0 area 2
R5(config-router)# ★ network 10.0.45.5 0.0.0.0 area 3 ★
R5(config-router)# ★ network 5.5.5.5 0.0.0.0 area 3 ★
```

**观察问题**：
```cisco
R3# show ip route ospf | include 5.5.5.5
! ★ 学不到 R5 的路由 ★

R2# show ip route ospf | include 5.5.5.5
! ★ 也学不到 ★
```

**原因**：Area 3 通过 Area 2 连接，**不直连 Area 0，违反了 OSPF 的规则**。R4 虽然是 ABR（Area 2 + Area 3），但它没有连 Area 0，**不会把 Area 3 的路由通告出去**。

### Step 2：配置虚链路

```cisco
! ★ 穿越区域是 Area 2（连接两个 ABR 的那个区域）★
R2(config)# router ospf 1
R2(config-router)# ★ area 2 virtual-link 4.4.4.4 ★
!                       ↑              ↑
!                  穿越区域        对端的 Router ID

R4(config)# router ospf 1
R4(config-router)# ★ area 2 virtual-link 2.2.2.2 ★
```

### Step 3：验证

```cisco
R2# show ip ospf virtual-links
★ Virtual Link OSPF_VL0 to router 4.4.4.4 is up ★
  Run as demand circuit
  ★ DoNotAge LSA allowed ★
  Transit area 2, via interface GigabitEthernet0/2
  Topology-MTID    Cost    Disabled     Shutdown      Topology Name
        0           100      no            no            Base

R2# show ip ospf neighbor
Neighbor ID  Pri  State     Dead Time  Address     Interface
4.4.4.4        0  ★FULL/  -★  -        10.0.24.4   ★ OSPF_VL0 ★
                                                     ↑ 虚链路作为一个虚拟接口
```

```cisco
R3# show ip route ospf | include 5.5.5.5
★ O IA  5.5.5.5/32 [110/401] via 10.0.13.1, Gi0/1 ★     ← 学到了 ✓
```

### Step 4：验证虚链路的限制

```cisco
R2(config-router)# area 2 stub
★ % OSPF: Area 2 is a stub area, cannot configure virtual link ★
```

**✅ 验证了"穿越区域不能是 Stub"。**

**观察 DoNotAge LSA**：
```cisco
R2# show ip ospf database | include DNA
! 虚链路上传的 LSA 会被标记为 DNA（永不老化）
```

> **虚链路是临时补救，不是设计方案。** 需要它说明区域规划有问题。DNA LSA 还可能导致陈旧路由永久留在 LSDB 里。

---

## Part 6：故障注入

### 故障 A：MTU 不匹配（卡 EXSTART）

```cisco
R2(config)# interface GigabitEthernet0/1
R2(config-if)# ip mtu 1400
R1# clear ip ospf process
```

<details><summary>症状与排查</summary>

```cisco
R1# show ip ospf neighbor
Neighbor ID  Pri  State         Dead Time  Address     Interface
2.2.2.2        1  ★EXSTART/DR★  00:00:35   10.0.12.2   Gi0/1

R1# show logging | include OSPF
%OSPF-5-ADJCHG: Process 1, Nbr 2.2.2.2 on Gi0/1 from EXSTART to DOWN, 
★ Neighbor Down: Too many retransmissions ★

R1# debug ip ospf adj
★ OSPF-1 ADJ Gi0/1: Nbr 2.2.2.2 has larger interface MTU ★

! 对比 MTU
R1# show ip ospf interface Gi0/1 | include MTU
  MTU is ★ 1500 ★ bytes
R2# show ip ospf interface Gi0/1 | include MTU
  MTU is ★ 1400 ★ bytes                        ← 找到了
```

**修复**：`R2(config-if)# ip mtu 1500`
**或应急**：两端都配 `ip ospf mtu-ignore`（治标）
</details>

### 故障 B：Stub 标志不一致

```cisco
R1(config-router)# area 1 stub
! ★ 故意不在 R3 上配 ★
```

<details><summary>症状与排查</summary>

```cisco
R1# show ip ospf neighbor
! ★ Area 1 的邻居消失了 ★

R1# show logging | include OSPF
%OSPF-4-ERRRCV: Received invalid packet: ★ mismatched area stub/transit flag ★ 
from 10.0.13.3, GigabitEthernet0/2
```

**根因**：Stub 标志是"建立邻居的 5 个条件"之一，**区域内所有路由器必须一致**。

**修复**：`R3(config-router)# area 1 stub`

**★ 生产环境改区域类型的正确流程**：
```
① 准备好完整的配置脚本
② 在维护窗口执行
③ ★ 从远端往 ABR 方向配 ★（先配最远的内部路由器，最后配 ABR）
   → 这样 ABR 一改完，所有邻居同时恢复，中断时间最短
④ 用 configure terminal revert timer 做保险
⑤ 确保有带外管理通道
```
</details>

### 故障 C：Router ID 重复

```cisco
R4(config-router)# router-id 2.2.2.2
R4# clear ip ospf process
```

<details><summary>症状与排查</summary>

```cisco
R2# show logging | include DUP_RTRID
★ %OSPF-4-DUP_RTRID_NBR: OSPF-1 Process detected duplicate router-id 2.2.2.2 
from 10.0.24.4 on interface GigabitEthernet0/2 ★

R2# show ip ospf neighbor
! 状态在 FULL 和 DOWN 之间反复翻转
```

**为什么会翻转**：Router ID 是 LSA 的唯一标识。两台设备用同一个 ID，它们的 Type-1 LSA **互相覆盖**，LSDB 陷入混乱。

**修复**：
```cisco
R4(config-router)# router-id 4.4.4.4
R4# clear ip ospf process                       ! ⚠️ 会中断所有邻居
```

**预防**：**每台设备用 Loopback 地址做 Router ID，且与 Loopback IP 一致**。
</details>

### 故障 D：网络类型不匹配

```cisco
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip ospf network point-to-point
! R2 保持默认 broadcast
```

<details><summary>症状与排查</summary>

```cisco
R1# show ip ospf interface Gi0/1 | include Network Type
  Network Type ★ POINT_TO_POINT ★, Cost: 100

R2# show ip ospf interface Gi0/1 | include Network Type
  Network Type ★ BROADCAST ★, Cost: 100
```

**症状**：邻居建不起来，或建起来但不稳定（DR 行为冲突）。

**修复**：两端统一。

**★ 建议**：两台路由器之间的以太网链路，**两端都改成 point-to-point**：
- 省掉 DR 选举，收敛更快
- **不产生 Type-2 LSA**，LSDB 更小
- 没有 DR 抢占困扰
</details>

### 故障 E：Type-3 LSA 过滤 vs distribute-list（★ 对比实验）

**方法 1：`area filter-list`（影响 LSDB）**
```cisco
R1(config)# ip prefix-list BLOCK-12 seq 5 deny 192.168.12.0/24
R1(config)# ip prefix-list BLOCK-12 seq 10 permit 0.0.0.0/0 le 32

R1(config)# router ospf 1
R1(config-router)# ★ area 1 filter-list prefix BLOCK-12 out ★
!                          ↑ out = 过滤 Area 1 向外通告的 Type-3
```

```cisco
R2# show ip ospf database summary | include 192.168.12
! ★ 空 —— LSA 根本没发过来 ★
R2# show ip route ospf | include 192.168.12
! ★ 空 ★
```

**方法 2：`distribute-list in`（不影响 LSDB）**
```cisco
R1(config-router)# no area 1 filter-list prefix BLOCK-12 out

R2(config)# ip prefix-list NO-12 seq 5 deny 192.168.12.0/24
R2(config)# ip prefix-list NO-12 seq 10 permit 0.0.0.0/0 le 32
R2(config)# router ospf 1
R2(config-router)# ★ distribute-list prefix NO-12 in ★
```

```cisco
R2# show ip ospf database summary | include 192.168.12
★ 192.168.12.0   1.1.1.1  ... ★                 ← ★ LSA 还在！★
R2# show ip route ospf | include 192.168.12
! ★ 但路由表里没有 ★
```

**✅ 这个对比清楚地展示了两种过滤的本质区别**：

| | `area filter-list` | `distribute-list in` |
|:--|:--|:--|
| 配在哪 | **ABR** | 任意路由器 |
| 影响 LSDB | ✅ **影响**（不发 LSA） | ❌ 不影响 |
| 影响范围 | **整个区域** | **只本机** |

**为什么 OSPF 不能像 EIGRP 那样随便过滤 LSA**：区域内所有路由器的 LSDB 必须完全一致，否则 SPF 会算出不同的拓扑，**可能形成环路**。

### 故障 F：忘改参考带宽

```cisco
R1(config-router)# no auto-cost reference-bandwidth 100000
! 恢复默认 100Mbps
```

<details><summary>症状与排查</summary>

```cisco
R1# show ip ospf | include Reference
 Reference bandwidth unit is ★ 100 mbps ★

R1# show ip ospf interface Gi0/1 | include Cost
  Process ID 1, Router ID 1.1.1.1, Network Type BROADCAST, ★ Cost: 1 ★
                                                             ↑ 千兆接口 cost 也是 1

R2# show ip ospf | include Reference
 Reference bandwidth unit is ★ 100000 mbps ★     ← 不一致！
```

**后果**：
- 万兆、千兆、百兆的 cost 都是 1 → OSPF 无法区分
- **两台设备的参考带宽不一致 → 算出的 cost 不同 → 次优路由甚至环路**

**修复**：**全网统一**
```cisco
R1(config-router)# auto-cost reference-bandwidth 100000
```

IOS 会警告：
```
% OSPF: Reference bandwidth is changed.
        ★ Please ensure reference bandwidth is consistent across all routers ★
```
</details>

---

## 实验检查清单

```
□ ① 三区域 OSPF 建立，所有邻居 FULL
□ ② 观察到 Type-1 LSA 不跨区域（区域隔离）
□ ③ 观察到 Type-3 LSA 携带网段但不携带拓扑
□ ④ 观察到 Type-4 和 Type-5 的配对关系
□ ⑤ 对比了 E1 和 E2 的 metric 计算
□ ⑥ 依次配置 Stub / Totally Stub，记录路由条数变化
□ ⑦ 配置 NSSA，观察 Type-7 → Type-5 的转换
□ ⑧ ★ 配置汇总，验证故障隔离价值（debug 观察 SPF 次数）
□ ⑨ 配置虚链路并验证"穿越区域不能是 Stub"
□ ⑩ 六个故障都亲手制造并修复
□ ⑪ 填写了四种区域类型对比表
□ ⑫ 写了实验笔记
```

## 关键命令总结

| 命令 | 用途 |
|:--|:--|
| **`show ip ospf database`** | ★ 排障分界线：LSDB 有没有 |
| `show ip ospf database router/summary/asbr-summary/external/nssa-external` | 分类型查看 |
| **`show ip ospf neighbor`** | 邻居状态 |
| **`show ip ospf interface <接口>`** | ★ 对比两端的 5 个条件 + MTU + 网络类型 |
| `show ip ospf \| include Area` | 确认区域类型 |
| `show ip ospf virtual-links` | 虚链路状态 |
| `show ip ospf statistics` | SPF 运行次数（查震荡） |
| `show ip protocols` | network 语句、过滤器、参考带宽 |

## LSA 类型速记

| Type | 谁产生 | 传播范围 | 路由标记 |
|:--|:--|:--|:--|
| **1** Router | 每台路由器 | **本区域** | `O` |
| **2** Network | **DR** | **本区域** | `O` |
| **3** Summary | **ABR** | **跨区域** | `O IA` |
| **4** ASBR Summary | **ABR** | 跨区域 | — |
| **5** External | **ASBR** | **整个 AS** | `O E1/E2` |
| **7** NSSA External | NSSA 内的 ASBR | **仅 NSSA** | `O N1/N2` |

---

**上一个** ← [Lab 03](lab-03-HSRP高可用.md) ｜ **下一个** → [Lab 05: EIGRP](lab-05-EIGRP.md)
