# Lab 07 · 路由重分发环路（★ 全书最重要的实验）

**对应章节**：[ENARSI 04](../03-ENARSI-300-410/04-路由重分发与路由策略.md)
**难度**：★★★★★　**时长**：3 小时

> ⚠️ **这是全书最重要的实验。**
> 通过标准：**能自己造出一个路由环路，并用三种方法分别修好它。**

## 目标

1. **亲手制造一个双点双向重分发环路**，用 traceroute 看到流量来回弹
2. 用 **Route Tag** 修复（首选方案）
3. 用 **调整 AD** 修复（备选方案）
4. 用 **路由过滤** 修复（特定场景）
5. 配置 PBR 并验证黑洞保护

---

## 拓扑

```
        OSPF 域                            EIGRP 域
                                    
   ┌────────┐            ┌────────┐            ┌────────┐
   │   R3   │────────────│   R1   │────────────│   R4   │
   │3.3.3.3 │            │1.1.1.1 │            │4.4.4.4 │
   └────────┘            │★双向★  │            └────────┘
        │                └────────┘                 │
        │                                           │
        │                ┌────────┐                 │
        └────────────────│   R2   │─────────────────┘
                         │2.2.2.2 │
                         │★双向★  │
                         └────────┘

   R3 的 Loopback: 10.1.1.0/24, 10.1.2.0/24  （OSPF 域原生）
   R4 的 Loopback: 10.2.1.0/24, 10.2.2.0/24  （EIGRP 域原生）
   
   R1、R2 都做【双向重分发】← ★ 环路的温床 ★
```

| 链路 | 网段 | 协议 |
|:--|:--|:--|
| R1-R3 | 10.0.13.0/30 | OSPF Area 0 |
| R2-R3 | 10.0.23.0/30 | OSPF Area 0 |
| R1-R2 | 10.0.12.0/30 | OSPF Area 0 |
| R1-R4 | 10.0.14.0/30 | EIGRP 100 |
| R2-R4 | 10.0.24.0/30 | EIGRP 100 |

---

## Part 1：基础配置（★ 故意不做防环）

### Step 1：R3（OSPF 域）

```cisco
R3(config)# interface Loopback0
R3(config-if)# ip address 3.3.3.3 255.255.255.255
R3(config)# interface Loopback1
R3(config-if)# ip address 10.1.1.1 255.255.255.0
R3(config)# interface Loopback2
R3(config-if)# ip address 10.1.2.1 255.255.255.0

R3(config)# router ospf 1
R3(config-router)# router-id 3.3.3.3
R3(config-router)# auto-cost reference-bandwidth 100000
R3(config-router)# network 10.0.13.3 0.0.0.0 area 0
R3(config-router)# network 10.0.23.3 0.0.0.0 area 0
R3(config-router)# network 3.3.3.3 0.0.0.0 area 0
R3(config-router)# network 10.1.1.0 0.0.0.255 area 0
R3(config-router)# network 10.1.2.0 0.0.0.255 area 0
```

### Step 2：R4（EIGRP 域）

```cisco
R4(config)# interface Loopback0
R4(config-if)# ip address 4.4.4.4 255.255.255.255
R4(config)# interface Loopback1
R4(config-if)# ip address 10.2.1.1 255.255.255.0
R4(config)# interface Loopback2
R4(config-if)# ip address 10.2.2.1 255.255.255.0

R4(config)# router eigrp 100
R4(config-router)# eigrp router-id 4.4.4.4
R4(config-router)# network 10.0.14.4 0.0.0.0
R4(config-router)# network 10.0.24.4 0.0.0.0
R4(config-router)# network 4.4.4.4 0.0.0.0
R4(config-router)# network 10.2.0.0 0.0.255.255
R4(config-router)# no auto-summary
```

### Step 3：R1（★ 双向重分发，无防护）

```cisco
R1(config)# router ospf 1
R1(config-router)# router-id 1.1.1.1
R1(config-router)# auto-cost reference-bandwidth 100000
R1(config-router)# network 10.0.13.1 0.0.0.0 area 0
R1(config-router)# network 10.0.12.1 0.0.0.0 area 0
R1(config-router)# network 1.1.1.1 0.0.0.0 area 0
R1(config-router)# ★ redistribute eigrp 100 subnets ★           ! 无防护！

R1(config)# router eigrp 100
R1(config-router)# eigrp router-id 1.1.1.1
R1(config-router)# network 10.0.14.1 0.0.0.0
R1(config-router)# no auto-summary
R1(config-router)# ★ redistribute ospf 1 metric 100000 100 255 1 1500 ★    ! 无防护！
```

### Step 4：R2（完全相同的配置）

```cisco
R2(config)# router ospf 1
R2(config-router)# router-id 2.2.2.2
R2(config-router)# auto-cost reference-bandwidth 100000
R2(config-router)# network 10.0.23.2 0.0.0.0 area 0
R2(config-router)# network 10.0.12.2 0.0.0.0 area 0
R2(config-router)# network 2.2.2.2 0.0.0.0 area 0
R2(config-router)# ★ redistribute eigrp 100 subnets ★

R2(config)# router eigrp 100
R2(config-router)# eigrp router-id 2.2.2.2
R2(config-router)# network 10.0.24.2 0.0.0.0
R2(config-router)# no auto-summary
R2(config-router)# ★ redistribute ospf 1 metric 100000 100 255 1 1500 ★
```

---

## Part 2：★ 观察环路的形成

### Step 1：初始状态（还没爆发）

```cisco
R1# show ip route 10.1.1.0
Routing entry for 10.1.1.0/24
  Known via "★ ospf 1 ★", distance ★ 110 ★, metric 200, ★ type intra area ★
  Routing Descriptor Blocks:
  * 10.0.13.3, from 3.3.3.3, via GigabitEthernet0/1
! ★ 现在还正常：OSPF 内部路由 ★
```

```cisco
R2# show ip route 10.1.1.0
Routing entry for 10.1.1.0/24
  Known via "ospf 1", distance 110, metric 200, ★ type intra area ★
  * 10.0.23.3, from 3.3.3.3, via GigabitEthernet0/1
! ★ 也正常 ★
```

### Step 2：★ 但回流已经发生了

```cisco
R1# show ip ospf database external | include 10.1.1.0 -A 5
  Link State ID: 10.1.1.0 (External Network Number)
  ★ Advertising Router: 2.2.2.2 ★
                        ↑↑↑↑↑↑↑
        ★ R2 把这条 OSPF 内部路由重分发进 EIGRP，
          又从 EIGRP 重分发【回】了 OSPF！★

R2# show ip ospf database external | include 10.1.1.0 -A 5
  ★ Advertising Router: 1.1.1.1 ★                ← R1 也是
```

**★ 双方都在互相"回灌"对方的路由。**

**现在为什么没出问题**：
```
   R1 有两条 10.1.1.0/24：
   · OSPF 内部路由 O    （从 R3 学到）
   · OSPF 外部路由 O E2 （从 R2 学到，回流的）
        ↓
   OSPF 选路顺序：★ 内部 > 外部 ★
        ↓
   R1 依然选内部路由 ✓ 暂时正常
```

**★ 这是一颗定时炸弹。**

### Step 3：★★ 引爆 —— 断掉原始源

```cisco
R3(config)# interface Loopback1                    ! 10.1.1.0/24 的源
R3(config-if)# ★ shutdown ★
```

**等 30 秒，然后观察**：

```cisco
R1# show ip route 10.1.1.0
Routing entry for 10.1.1.0/24
  Known via "ospf 1", distance 110, metric 20, ★ type extern 2 ★
  Routing Descriptor Blocks:
  * ★ 10.0.12.2 ★, from 2.2.2.2, via GigabitEthernet0/2
        ↑↑↑↑↑↑↑↑↑
   ★ R1 认为要走 R2 ★

R2# show ip route 10.1.1.0
Routing entry for 10.1.1.0/24
  Known via "ospf 1", distance 110, metric 20, ★ type extern 2 ★
  * ★ 10.0.12.1 ★, from 1.1.1.1, via GigabitEthernet0/2
        ↑↑↑↑↑↑↑↑↑
   ★ R2 认为要走 R1 ★
```

**★★★ 环路形成 ★★★**

### Step 4：验证环路

```cisco
R1# traceroute 10.1.1.1 source Loopback0
Type escape sequence to abort.
Tracing the route to 10.1.1.1

  1 ★ 10.0.12.2 ★  4 msec  4 msec  4 msec
  2 ★ 10.0.12.1 ★  8 msec  8 msec  8 msec
  3 ★ 10.0.12.2 ★  4 msec  4 msec  4 msec
  4 ★ 10.0.12.1 ★  8 msec  8 msec  8 msec
  5 10.0.12.2  4 msec  4 msec  4 msec
  ...
  ★ 在 R1 和 R2 之间来回弹，直到 TTL 耗尽 ★
```

```cisco
! 观察链路流量暴涨
R1# show interfaces GigabitEthernet0/2 | include rate
  5 minute input rate ★ 45000000 ★ bits/sec, 5623 packets/sec
  5 minute output rate ★ 45000000 ★ bits/sec, 5623 packets/sec
! ★ 一条本来没什么流量的链路突然被打满 ★
```

**如果有真实流量在跑（比如 PC 在 ping 10.1.1.1）**：
```cisco
R1# show processes cpu sorted | exclude 0.00
CPU utilization for five seconds: ★ 85% ★/60%
! CPU 也会飙高（因为要处理大量 TTL 超时的 ICMP）
```

**✅ 你亲手造出了一个路由环路。**

**★ 记住这个症状**：
| 症状 | 含义 |
|:--|:--|
| **traceroute 看到 IP 往复** | 环路的确定证据 |
| **某条链路流量异常暴涨** | 环路的间接证据 |
| **路由类型从 intra area 变成 extern 2** | 说明用的是回流的路由 |

---

## Part 3：★ 方法 1 —— Route Tag（首选）

### Step 1：恢复环境

```cisco
R3(config)# interface Loopback1
R3(config-if)# no shutdown
```

### Step 2：配置 Route Tag（★ R1 和 R2 配完全相同的内容）

```cisco
! ══════════════════════════════════════════════
! ★ 在 R1 和 R2 上都执行以下【完全相同】的配置 ★
! ══════════════════════════════════════════════

! ── OSPF → EIGRP ──
route-map OSPF-TO-EIGRP ★ deny 5 ★
 ★ match tag 200 ★                       ! 拒绝"来自 EIGRP"的路由（防回流）
route-map OSPF-TO-EIGRP permit 10
 ★ set tag 100 ★                         ! 标记为"来自 OSPF"
 set metric 100000 100 255 1 1500

! ── EIGRP → OSPF ──
route-map EIGRP-TO-OSPF ★ deny 5 ★
 ★ match tag 100 ★                       ! 拒绝"来自 OSPF"的路由（防回流）
route-map EIGRP-TO-OSPF permit 10
 ★ set tag 200 ★                         ! 标记为"来自 EIGRP"

! ── 应用 ──
router eigrp 100
 redistribute ospf 1 route-map OSPF-TO-EIGRP

router ospf 1
 redistribute eigrp 100 subnets route-map EIGRP-TO-OSPF
```

**★ 注意 seq 5 必须排在 seq 10 前面**（route-map 自上而下匹配）。

### Step 3：验证 tag 生效

```cisco
R4# show ip route 10.1.1.0
Routing entry for 10.1.1.0/24
  Known via "eigrp 100", distance 170, metric 2560002816
  ★ Tag 100 ★                                      ← ★ 标记为"来自 OSPF"✓
  Redistributing via eigrp 100
  Last update from 10.0.14.1 on GigabitEthernet0/1
```

```cisco
R3# show ip route 10.2.1.0
Routing entry for 10.2.1.0/24
  Known via "ospf 1", distance 110, metric 20, type extern 2
  ★ Tag 200 ★                                      ← ★ 标记为"来自 EIGRP"✓
```

### Step 4：★ 验证不再回流

```cisco
R1# show ip ospf database external | include 10.1.1.0
! ★ 空输出！★
! R2 不再把这条路由重分发回 OSPF ✓

R2# show ip ospf database external | include 10.1.1.0
! ★ 也是空 ★
```

**对比修复前**：
```cisco
! 修复前
R1# show ip ospf database external | include 10.1.1.0
  Advertising Router: 2.2.2.2                     ← 有回流

! 修复后
R1# show ip ospf database external | include 10.1.1.0
! 空 ✓
```

### Step 5：★ 再次引爆，验证不再环路

```cisco
R3(config)# interface Loopback1
R3(config-if)# shutdown
```

```cisco
R1# show ip route 10.1.1.0
★ % Network not in table ★
! ★★ 正确！路由【消失】而不是形成环路 ★★

R1# traceroute 10.1.1.1
  1  *  *  *
  2  *  *  *
! ★ 不通，但没有环路 ✓ ★
```

**★ 关键理解：修复成功的标志是"路由消失（不可达）"，而不是"通过环路假装可达"。**

不可达是**正确的行为**——源确实没了。环路才是错误的。

### Step 6：观察 route-map 的匹配计数

```cisco
R1# show route-map EIGRP-TO-OSPF
route-map EIGRP-TO-OSPF, ★ deny ★, sequence 5
  Match clauses:
    ★ tag 100 ★
  Set clauses:
  Policy routing matches: 0 packets, 0 bytes
route-map EIGRP-TO-OSPF, permit, sequence 10
  Match clauses:
  Set clauses:
    ★ tag 200 ★
  Policy routing matches: 0 packets, 0 bytes
```

---

## Part 4：方法 2 —— 调整 AD

### Step 1：先撤销 Route Tag

```cisco
R1(config)# router ospf 1
R1(config-router)# no redistribute eigrp 100 subnets route-map EIGRP-TO-OSPF
R1(config-router)# redistribute eigrp 100 subnets

R1(config)# router eigrp 100
R1(config-router)# no redistribute ospf 1 route-map OSPF-TO-EIGRP
R1(config-router)# redistribute ospf 1 metric 100000 100 255 1 1500

! R2 同样撤销
```

**确认环路又回来了**（重复 Part 2 的引爆步骤）。

### Step 2：调整 AD

**思路**：让"回流的 OSPF 外部路由"的 AD 比"EIGRP 外部路由（170）"更差。

```cisco
! ★ R1 和 R2 都配 ★
R1(config)# router ospf 1
R1(config-router)# ★ distance ospf external 180 ★
!                                            ↑ 从 110 调到 180

R2(config)# router ospf 1
R2(config-router)# distance ospf external 180
```

### Step 3：验证

```cisco
! 断掉源
R3(config)# interface Loopback1
R3(config-if)# shutdown
```

```cisco
R1# show ip route 10.1.1.0
Routing entry for 10.1.1.0/24
  Known via "★ eigrp 100 ★", distance ★ 170 ★, metric ...
                                        ↑
                    ★ EIGRP 外部（170）赢了 OSPF 外部（180）★
  Routing Descriptor Blocks:
  * 10.0.14.4, from 10.0.14.4, via GigabitEthernet0/3
                    ↑ 指向 R4（EIGRP 域），不再指向 R2
```

**这也能防环，但**：

### Step 4：验证 AD 方案的缺陷

**缺陷 1：只在配了的设备上生效（局部行为）**
```cisco
! 只在 R1 上配，不在 R2 上配
R2(config)# router ospf 1
R2(config-router)# no distance ospf external 180
```
```cisco
R2# show ip route 10.1.1.0
  Known via "ospf 1", distance ★ 110 ★, type extern 2
  * 10.0.12.1, from 1.1.1.1                       ← R2 依然指向 R1
! ★ 环路依然存在（单向）★
```

**缺陷 2：可能引入次优路由**
```cisco
! 某些场景下 OSPF 外部路由确实是更优的路径
! 但因为 AD 被调到 180，永远选不上
```

**缺陷 3：拓扑变化后可能失效**
```
   如果引入第三个协议（比如 BGP），或者某条路径的协议来源变了
        ↓
   ★ 原来精心计算的 AD 关系可能不再成立 ★
```

**★ 对比结论**：

| 维度 | **Route Tag** | **调整 AD** |
|:--|:--|:--|
| 作用范围 | **路由属性，随路由传播** | **只影响配置它的那台设备** |
| 适应拓扑变化 | ✅ **自动适应** | ❌ 可能失效 |
| 配置一致性 | 两点配**完全相同**的内容 | 需精心计算每台的值 |
| 可读性 | 意图明确 | 一堆数字 |
| 副作用 | 几乎没有 | **可能次优路由** |
| **推荐度** | ★★★★★ | ★★ |

---

## Part 5：方法 3 —— 路由过滤

### Step 1：撤销 AD 调整

```cisco
R1(config-router)# no distance ospf external 180
```

### Step 2：用 prefix-list 明确指定允许重分发的前缀

```cisco
! ★ 只允许 OSPF 域原生的网段被重分发进 EIGRP ★
R1(config)# ip prefix-list OSPF-NATIVE seq 5 permit 10.1.0.0/16 ★ le 32 ★
R1(config)# ip prefix-list OSPF-NATIVE seq 10 permit 3.3.3.3/32
R1(config)# ip prefix-list OSPF-NATIVE seq 15 deny 0.0.0.0/0 le 32

R1(config)# route-map FILTER-OSPF-TO-EIGRP permit 10
R1(config-route-map)#  match ip address prefix-list OSPF-NATIVE
R1(config-route-map)#  set metric 100000 100 255 1 1500

! ★ 只允许 EIGRP 域原生的网段被重分发进 OSPF ★
R1(config)# ip prefix-list EIGRP-NATIVE seq 5 permit 10.2.0.0/16 le 32
R1(config)# ip prefix-list EIGRP-NATIVE seq 10 permit 4.4.4.4/32
R1(config)# ip prefix-list EIGRP-NATIVE seq 15 deny 0.0.0.0/0 le 32

R1(config)# route-map FILTER-EIGRP-TO-OSPF permit 10
R1(config-route-map)#  match ip address prefix-list EIGRP-NATIVE

! ── 应用 ──
R1(config)# router eigrp 100
R1(config-router)# redistribute ospf 1 route-map FILTER-OSPF-TO-EIGRP

R1(config)# router ospf 1
R1(config-router)# redistribute eigrp 100 subnets route-map FILTER-EIGRP-TO-OSPF

! ★ R2 配完全相同的内容 ★
```

### Step 3：验证

```cisco
R1# show ip ospf database external | include 10.1.1.0
! 空 ✓（10.1.x.x 属于 OSPF 域，不会被重分发回 OSPF）

R1# show ip prefix-list detail EIGRP-NATIVE
ip prefix-list EIGRP-NATIVE:
   seq 5 permit 10.2.0.0/16 le 32 ★ (hit count: 4) ★
   seq 15 deny 0.0.0.0/0 le 32 ★ (hit count: 6) ★
                                   ↑ 拦下了 6 条不该重分发的
```

### Step 4：验证过滤方案的缺陷

**缺陷：需要维护前缀列表**

```cisco
! OSPF 域新增一个网段
R3(config)# interface Loopback3
R3(config-if)# ip address 10.5.5.1 255.255.255.0
R3(config)# router ospf 1
R3(config-router)# network 10.5.5.0 0.0.0.255 area 0
```

```cisco
R4# show ip route eigrp | include 10.5.5
! ★ 空！新网段没被重分发过去 ★
! 因为 prefix-list 只允许 10.1.0.0/16
```

**必须去改 prefix-list**：
```cisco
R1(config)# ip prefix-list OSPF-NATIVE seq 7 permit 10.5.5.0/24
R2(config)# ip prefix-list OSPF-NATIVE seq 7 permit 10.5.5.0/24
```

**★ 这就是过滤方案的最大问题：维护成本高，容易遗漏。**

**适合场景**：网段规划非常稳定，且只需要重分发少量固定的前缀。

**实践建议**：**Route Tag 为主，过滤作为二次保险**（纵深防御）。

---

## Part 6：最佳实践 —— Tag + 过滤组合

```cisco
! ══════ R1 和 R2 都配 ══════

! ── ① 定义"合法的"前缀范围（第一道防线）──
ip prefix-list OSPF-NATIVE seq 5 permit 10.1.0.0/16 le 32
ip prefix-list OSPF-NATIVE seq 10 permit 3.3.3.3/32
ip prefix-list OSPF-NATIVE seq 15 deny 0.0.0.0/0 le 32

ip prefix-list EIGRP-NATIVE seq 5 permit 10.2.0.0/16 le 32
ip prefix-list EIGRP-NATIVE seq 10 permit 4.4.4.4/32
ip prefix-list EIGRP-NATIVE seq 15 deny 0.0.0.0/0 le 32

! ── ② Tag 防环（第二道防线）──
route-map OSPF-TO-EIGRP deny 5
 match tag 200                              ! 拒绝回流
route-map OSPF-TO-EIGRP permit 10
 match ip address prefix-list OSPF-NATIVE   ! 且必须在合法范围内
 set tag 100
 set metric 100000 100 255 1 1500

route-map EIGRP-TO-OSPF deny 5
 match tag 100                              ! 拒绝回流
route-map EIGRP-TO-OSPF permit 10
 match ip address prefix-list EIGRP-NATIVE
 set tag 200

! ── ③ 应用 ──
router eigrp 100
 redistribute ospf 1 route-map OSPF-TO-EIGRP

router ospf 1
 redistribute eigrp 100 subnets route-map EIGRP-TO-OSPF
```

**★ 双重防护**：
- Tag 防止"回流"
- prefix-list 防止"误重分发不该重分发的"

---

## Part 7：PBR 实验

### Step 1：基础 PBR

```cisco
! 让某个网段的流量强制走特定出口
R1(config)# ip access-list extended FROM-SALES
R1(config-ext-nacl)#  permit ip 192.168.20.0 0.0.0.255 any

R1(config)# route-map PBR-SALES permit 10
R1(config-route-map)#  match ip address FROM-SALES
R1(config-route-map)#  ★ set ip next-hop 10.0.14.4 ★

R1(config)# route-map PBR-SALES permit 20
! ★ 空 permit：其余流量走正常路由表 ★

R1(config)# interface GigabitEthernet0/0
R1(config-if)# ★ ip policy route-map PBR-SALES ★           ! ★ 在【入接口】上
```

### Step 2：验证

```cisco
R1# show ip policy
Interface      Route map
Gi0/0          PBR-SALES

R1# show route-map PBR-SALES
route-map PBR-SALES, permit, sequence 10
  Match clauses:
    ip address (access-lists): FROM-SALES
  Set clauses:
    ip next-hop 10.0.14.4
  ★ Policy routing matches: 1245 packets, 987654 bytes ★
                            ↑ 有命中 ✓
```

### Step 3：★ 制造黑洞

```cisco
! 断掉 PBR 指定的下一跳
R1(config)# interface GigabitEthernet0/3                   ! 朝 R4
R1(config-if)# shutdown
```

```cisco
! 从 192.168.20.0/24 发起的流量
PC-SALES> ping 10.2.1.1
Request timed out.
! ★ 全部超时 ★

! 但从其他网段发起的流量正常
R1# ping 10.2.1.1 source Loopback0
!!!!!                                                      ← 走正常路由表，通
```

**★ 这就是 PBR 的危险**：`set ip next-hop` **强制走指定下一跳，即使它不可达**。**普通的路由收敛救不了它**（因为 PBR 优先于路由表）。

### Step 4：加 track 保护

```cisco
R1(config)# ip sla 1
R1(config-ip-sla)#  icmp-echo 10.0.14.4
R1(config-ip-sla-echo)#   frequency 5
R1(config-ip-sla-echo)#   timeout 2000
R1(config)# ip sla schedule 1 life forever start-time now

R1(config)# track 1 ip sla 1 reachability
R1(config-track)#  delay down 10 up 30

R1(config)# ip sla 2
R1(config-ip-sla)#  icmp-echo 10.0.12.2
R1(config-ip-sla-echo)#   frequency 5
R1(config)# ip sla schedule 2 life forever start-time now
R1(config)# track 2 ip sla 2 reachability
R1(config-track)#  delay down 10 up 30

! ★ PBR 挂钩 track，配多个备选 ★
R1(config)# route-map PBR-SALES permit 10
R1(config-route-map)#  match ip address FROM-SALES
R1(config-route-map)#  no set ip next-hop 10.0.14.4
R1(config-route-map)#  ★ set ip next-hop verify-availability 10.0.14.4 1 track 1 ★
R1(config-route-map)#  ★ set ip next-hop verify-availability 10.0.12.2 2 track 2 ★
```

### Step 5：验证保护生效

```cisco
R1(config)# interface GigabitEthernet0/3
R1(config-if)# shutdown
```

**等 10 秒（delay down 10）**：
```cisco
R1# show route-map PBR-SALES
route-map PBR-SALES, permit, sequence 10
  Set clauses:
    ip next-hop verify-availability 10.0.14.4 1 track 1  ★ [down] ★
    ip next-hop verify-availability 10.0.12.2 2 track 2  ★ [up] ★
!                                                          ↑ 自动切到序号 2 ✓
```

```cisco
PC-SALES> ping 10.2.1.1
Reply from 10.2.1.1: bytes=32 time=8ms                     ← 恢复了 ✓
```

**如果两个 track 都 down**：
```cisco
R1# show route-map PBR-SALES
    ip next-hop verify-availability 10.0.14.4 1 track 1  [down]
    ip next-hop verify-availability 10.0.12.2 2 track 2  [down]
! ★ PBR 不再强制，流量回落到正常路由表 ★
```

### Step 6：`next-hop` vs `default next-hop` 对比

```cisco
! ── set ip next-hop：PBR 优先于路由表 ──
R1(config-route-map)# set ip next-hop 10.0.14.4
! → 即使路由表里有更好的路径，也强制走这个

! ── set ip default next-hop：路由表优先 ──
R1(config-route-map)# ★ set ip default next-hop 10.0.14.4 ★
! → 先查路由表，路由表【没有匹配】时才用这个
```

**验证差异**：
```cisco
! 用 default next-hop 时
R1# show ip route 10.2.1.0
D EX 10.2.1.0/24 [170/...] via 10.0.14.4        ← 路由表里有

PC-SALES> traceroute 10.2.1.1
! ★ 走路由表的路径，不受 PBR 影响 ★

! 对于路由表里【没有】的目的地
PC-SALES> traceroute 8.8.8.8
! ★ 走 PBR 指定的 10.0.14.4 ★
```

---

## 实验检查清单

```
□ ① ★★ 亲手制造了重分发环路，traceroute 看到 IP 往复 ★★
□ ② 观察到"路由类型从 intra area 变成 extern 2"
□ ③ 观察到链路流量异常暴涨
□ ④ ★ 用 Route Tag 修复，验证不再回流
□ ⑤ ★ 验证修复标志是"路由消失"而非"环路可达"
□ ⑥ 用调整 AD 修复，并验证其三个缺陷
□ ⑦ 用路由过滤修复，并验证维护成本问题
□ ⑧ 配置 Tag + 过滤的组合方案
□ ⑨ 配置 PBR，制造黑洞，再用 track 修复
□ ⑩ 对比 set ip next-hop 和 set ip default next-hop
□ ⑪ 写了实验笔记
```

## 关键命令总结

| 命令 | 用途 |
|:--|:--|
| **`traceroute <目标>`** | ★ 环路的直接证据（IP 往复） |
| **`show ip route <前缀>`** | ★ 看路由的来源和类型（intra area vs extern 2） |
| **`show ip ospf database external \| include <前缀>`** | ★ 检查是否有回流 |
| `show ip route <前缀> \| include Tag` | 验证 tag 生效 |
| `show route-map <名称>` | 验证策略结构 |
| `show ip prefix-list detail` | hit count 验证规则命中 |
| `show interfaces \| include rate` | 环路导致的流量异常 |
| `show ip policy` | PBR 应用情况 |

## 核心结论（★ 必须记住）

| 要点 | 说明 |
|:--|:--|
| **单点重分发安全，双点双向危险** | 冗余的代价 |
| **环路根因：重分发丢失了"路由的血统"** | 无法区分原生路由和回流路由 |
| **修复成功的标志：路由消失，而不是环路可达** | 不可达是正确行为 |
| **Route Tag 是首选**（属性随路由传播，自动适应） | ★★★★★ |
| **调 AD 是备选**（局部行为，可能失效） | ★★ |
| **过滤适合稳定的小规模场景**（维护成本高） | ★★★ |
| **PBR 必须配 verify-availability + track** | 否则黑洞，路由收敛救不了 |
| **route-map 永远加兜底的 permit** | 隐含 deny |

## Route Tag 标准模板（存下来直接用）

```cisco
! ★ 两个重分发点配【完全相同】的内容 ★

route-map OSPF-TO-EIGRP deny 5
 match tag 200
route-map OSPF-TO-EIGRP permit 10
 set tag 100
 set metric 100000 100 255 1 1500

route-map EIGRP-TO-OSPF deny 5
 match tag 100
route-map EIGRP-TO-OSPF permit 10
 set tag 200

router eigrp 100
 redistribute ospf 1 route-map OSPF-TO-EIGRP

router ospf 1
 redistribute eigrp 100 subnets route-map EIGRP-TO-OSPF
```

**Tag 值规划建议**（写进网络文档）：

| Tag | 含义 |
|:--|:--|
| 100 | 来自 OSPF |
| 200 | 来自 EIGRP |
| 300 | 来自静态路由 |
| 400 | 来自 BGP |
| 500 | 来自直连 |
| 999 | 禁止重分发 |

---

**上一个** ← [Lab 06](lab-06-BGP.md) ｜ **下一个** → [Lab 08: GRE over IPsec](lab-08-GRE-over-IPsec.md)
