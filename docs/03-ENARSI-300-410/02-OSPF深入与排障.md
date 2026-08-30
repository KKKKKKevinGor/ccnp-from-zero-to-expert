# 02 · OSPF 深入与排障

> 基础见 [ENCOR 第 7 章](../02-ENCOR-350-401/07-OSPF进阶.md)。本章聚焦**排障、过滤和进阶特性**。

## ① OSPF 排障黄金流程

**记住这个顺序，90% 的 OSPF 问题能在 5 分钟内定位：**

```
   ① show ip ospf neighbor
      邻居建立了吗？状态对吗？
        ├─ 没有邻居  → 走 ② 接口检查
        ├─ 卡 INIT   → 对方收不到我的 Hello（ACL？单向链路？）
        ├─ 卡 EXSTART→ ★ MTU 不匹配 ★（99%）
        ├─ 停 2WAY   → DROTHER 之间（正常）
        └─ 翻转      → Router ID 重复 / 链路抖动 / CPU
        ↓
   ② show ip ospf interface <接口>
      逐项对比两端的 5 个条件 + MTU + 网络类型
        ↓
   ③ show ip ospf database
      ★ 这是分界线 ★
      LSA 收到了吗？和邻居的 LSDB 一致吗？
        ├─ LSDB 里没有 → 问题在【发送侧】（对方没通告/被过滤）
        └─ LSDB 里有   → 问题在【接收侧】（AD/过滤/SPF）
        ↓
   ④ show ip route ospf
      路由进表了吗？
        ↓
   ⑤ show ip protocols
      network 语句、passive、参考带宽、过滤器
```

> **★ 第 ③ 步的 LSDB 是排障的分界线。** LSDB 是"我收到了什么"，路由表是"我采纳了什么"。两者的差异告诉你问题在哪一侧——这是 OSPF 排障最重要的思维方式。

---

## ② 邻居问题深入

### 2.1 五个条件 + 三个隐藏条件

**Hello 报文里携带的（不匹配直接丢包）**：
1. **Area ID**
2. **Hello / Dead 间隔**
3. **认证类型和密钥**
4. **Stub / NSSA 标志**（E-bit）
5. **子网掩码**（广播/NBMA 网络）

**不在 Hello 里但同样致命**：
6. **MTU**（卡在 EXSTART/EXCHANGE）
7. **Router ID 不能重复**
8. **网络类型必须兼容**（P2P 只能对 P2P，Broadcast 只能对 Broadcast）

### 2.2 网络类型不匹配（容易被忽略的第 8 条）

| 网络类型 | 需要 DR | Hello/Dead | 需要手工邻居 |
|:--|:--|:--|:--|
| **Broadcast** | ✅ | **10 / 40** | ❌ |
| **Non-Broadcast (NBMA)** | ✅ | **30 / 120** | ✅ |
| **Point-to-Point** | ❌ | **10 / 40** | ❌ |
| **Point-to-Multipoint** | ❌ | **30 / 120** | ❌ |
| **Point-to-Multipoint Non-Broadcast** | ❌ | **30 / 120** | ✅ |
| Loopback | — | — | — |

**兼容性矩阵（★ 考点）**：

| 一端 \ 另一端 | Broadcast | P2P | Non-Broadcast | P2MP |
|:--|:--|:--|:--|:--|
| **Broadcast** | ✅ | ❌ | ❌ | ❌ |
| **P2P** | ❌ | ✅ | ❌ | ❌ |
| **Non-Broadcast** | ❌ | ❌ | ✅ | ❌ |
| **P2MP** | ❌ | ❌ | ❌ | ✅ |

**为什么不兼容**：
- **Hello/Dead 时间不同**（10/40 vs 30/120）→ 直接不匹配
- **DR 需求不同** → 一端选 DR 一端不选，行为冲突

```cisco
R1(config-if)# ip ospf network point-to-point
R2(config-if)# ip ospf network broadcast          ! 不匹配！
```

**症状**：邻居建不起来，或者建起来了但不稳定。

**排查**：
```cisco
R1# show ip ospf interface Gi0/1 | include Network Type|Timer
  Process ID 1, Router ID 1.1.1.1, Network Type POINT_TO_POINT, Cost: 100
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5

R2# show ip ospf interface Gi0/1 | include Network Type|Timer
  Process ID 1, Router ID 2.2.2.2, Network Type BROADCAST, Cost: 100
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
                              ↑ 时间一样，但类型不同 → DR 行为冲突
```

> **实践建议**：**两台路由器之间的以太网链路，两端都改成 `point-to-point`**。好处：
> - 省掉 DR 选举，收敛更快
> - 不产生 Type-2 LSA，LSDB 更小
> - 没有 DR 抢占的困扰
>
> 这在数据中心和核心互联链路上是标准做法。**但必须两端都改。**

### 2.3 EXSTART/EXCHANGE 卡死的完整原因

**99% 是 MTU，但还有其他可能**：

| 原因 | 排查 |
|:--|:--|
| **MTU 不匹配**（99%） | `show ip ospf interface \| inc MTU` |
| **单向链路** | 两端接口计数器对比 |
| **ACL 拦了大包** | `show access-lists` |
| **Router ID 重复** | `show logging \| inc DUP_RTRID` |
| **中间设备丢大包** | `ping size 1500 df-bit` |
| MTU 相同但路径 MTU 不足（隧道场景） | 同上 |

```cisco
! 修复方案 1（推荐）：统一 MTU
R2(config-if)# ip mtu 1500

! 修复方案 2（应急）：两端都忽略 MTU 检查
R1(config-if)# ip ospf mtu-ignore
R2(config-if)# ip ospf mtu-ignore
```

> **`mtu-ignore` 是治标不治本**：邻居能起来了，但如果实际路径 MTU 确实不足，大的 **LSU 报文依然会被丢弃**，导致 **LSDB 同步不完整**——问题会以更隐蔽的形式出现（某些路由学不到）。
>
> **优先统一 MTU。**

### 2.4 Router ID 重复

```cisco
R3(config-router)# router-id 2.2.2.2          ! 和 R2 重复
```

**症状**：邻居状态在 FULL 和 DOWN 之间**反复翻转**，日志刷屏。

```cisco
R1# show logging | include DUP_RTRID
%OSPF-4-DUP_RTRID_NBR: OSPF-1 Process detected duplicate router-id 2.2.2.2 
from 192.168.13.3 on interface GigabitEthernet0/2
```

**为什么重复会导致问题**：Router ID 是 LSA 的唯一标识。两台设备用同一个 ID，它们的 Type-1 LSA 会**互相覆盖**，LSDB 陷入混乱。

**修复**：
```cisco
R3(config-router)# router-id 3.3.3.3
R3# clear ip ospf process                     ! ⚠️ 会中断所有邻居
```

**预防**：
```cisco
! 每台设备用 Loopback 地址做 Router ID，且与 Loopback IP 一致
R3(config)# interface Loopback0
R3(config-if)# ip address 3.3.3.3 255.255.255.255
R3(config)# router ospf 1
R3(config-router)# router-id 3.3.3.3
```
这样排障时看到 Router ID 就知道是哪台设备，也不会重复。

---

## ③ OSPF 路由过滤（★ ENARSI 重点）

**OSPF 是链路状态协议，区域内的 LSDB 必须完全一致**——所以**不能在区域内部过滤 LSA**（会导致 LSDB 不一致，SPF 算错）。

**过滤只能在特定的位置做**：

| 方法 | 在哪配 | 过滤什么 | 影响 |
|:--|:--|:--|:--|
| **`area X filter-list`** | **ABR** | **Type-3 LSA（进出某区域）** | **影响 LSDB** |
| **`area X range ... not-advertise`** | **ABR** | 汇总时抑制 | 影响 LSDB |
| **`distribute-list ... in`** | 任意路由器 | **只过滤【进路由表】** | **不影响 LSDB** |
| **`summary-address ... not-advertise`** | **ASBR** | Type-5 LSA | 影响 LSDB |
| **`redistribute ... route-map`** | ASBR | 重分发时过滤 | 影响 LSDB |

### 3.1 `area X filter-list`（ABR 上过滤 Type-3）

```cisco
R-ABR(config)# ip prefix-list BLOCK-10-99 seq 5 deny 10.99.0.0/16 le 32
R-ABR(config)# ip prefix-list BLOCK-10-99 seq 10 permit 0.0.0.0/0 le 32

R-ABR(config)# router ospf 1
R-ABR(config-router)# area 1 filter-list prefix BLOCK-10-99 in
!                                                          ↑
!                          in  = 过滤【进入 Area 1】的 Type-3
!                          out = 过滤【离开 Area 1】的 Type-3
```

**方向的理解（★ 容易搞反）**：

```
   Area 0 ──── [ABR] ──── Area 1
   
   area 1 filter-list ... in
   → 过滤 ABR 【向 Area 1 发送】的 Type-3 LSA
   → 也就是 "从其他区域进入 Area 1" 的路由
   
   area 1 filter-list ... out
   → 过滤 ABR 【从 Area 1 向外发送】的 Type-3 LSA
   → 也就是 "Area 1 的路由通告给其他区域"
```

**记忆**：**方向是相对于"这个 area"的**，不是相对于路由器。

**验证**：
```cisco
R-Area1# show ip ospf database summary
! 被过滤的前缀不会出现
```

### 3.2 `distribute-list in`（只过滤路由表，不影响 LSDB）

```cisco
R1(config)# ip prefix-list NO-10-99 seq 5 deny 10.99.0.0/16 le 32
R1(config)# ip prefix-list NO-10-99 seq 10 permit 0.0.0.0/0 le 32

R1(config)# router ospf 1
R1(config-router)# distribute-list prefix NO-10-99 in
```

**★ 关键理解**：

```cisco
R1# show ip ospf database summary
! ★ LSA 还在！★ 因为 distribute-list 不影响 LSDB

R1# show ip route ospf | include 10.99
! ★ 路由表里没有 ★ 被过滤掉了
```

**为什么这样设计**：OSPF 的 LSDB 必须在区域内保持一致（否则 SPF 会算出不同的拓扑，导致环路）。`distribute-list in` 只影响**本机把 LSDB 转成路由表**这一步，**不影响 LSA 的泛洪**，所以不会破坏一致性。

**这也意味着**：`distribute-list in` **是本地行为**，只影响这一台路由器。其他路由器依然会学到那些路由。

> **`distribute-list out` 在 OSPF 里只对重分发有效**（在 ASBR 上过滤重分发进来的路由），不能用来过滤 OSPF 自己的 LSA。

### 3.3 三种过滤的选择

| 需求 | 用什么 |
|:--|:--|
| 不让某个区域**知道**某些路由（减小 LSDB） | **`area X filter-list`**（ABR） |
| 只是**本机**不想用某条路由 | **`distribute-list in`** |
| 不让外部路由进入某个区域 | **Stub / Totally Stub** 区域类型 |
| 汇总时抑制某个网段 | **`area X range ... not-advertise`** |
| 不通告某条重分发进来的外部路由 | **`summary-address ... not-advertise`**（ASBR） |

---

## ④ 虚链路深入

### 4.1 两种使用场景

**场景 1：区域不直连 Area 0**
```
   Area 0 ──[ABR-1]── Area 1 ──[ABR-2]── Area 2
                                            ↑
                                  Area 2 没有直连 Area 0
```

**场景 2：Area 0 被分割**
```
   Area 0(左) ──[ABR-1]── Area 1 ──[ABR-2]── Area 0(右)
                                                ↑
                                       骨干被切断了
```

### 4.2 配置

```cisco
! ABR-1（Router ID 1.1.1.1）
R-ABR1(config)# router ospf 1
R-ABR1(config-router)# area 1 virtual-link 2.2.2.2
!                           ↑              ↑
!                     穿越区域(Transit)  对端的 Router ID

! ABR-2（Router ID 2.2.2.2）
R-ABR2(config-router)# area 1 virtual-link 1.1.1.1
```

**★ 三个必须记住的点**：
1. **`area 1` 是穿越区域**，不是 Area 0，也不是目标区域
2. **参数是对端的 Router ID**，不是 IP 地址
3. **两端都要配**

### 4.3 虚链路的限制（★ 考点）

| 限制 | 说明 |
|:--|:--|
| **穿越区域不能是 Stub / NSSA / Totally Stub** | 虚链路要传 Type-3/4/5 LSA，Stub 区域会拒绝 |
| **穿越区域必须有到对端 Router ID 的完整路由** | 不能被汇总或过滤阻断 |
| **两端必须都是 ABR** | 非 ABR 不能建虚链路 |
| **认证要匹配** | 用穿越区域的认证配置 |

**为什么穿越区域不能是 Stub**：
```
   Stub 区域拒绝 Type-4/5 LSA
        ↓
   但虚链路的作用就是把 Type-3/4/5 从一端传到另一端
        ↓
   ★ 矛盾 → IOS 会直接拒绝配置 ★
```

```cisco
R1(config-router)# area 1 stub
R1(config-router)# area 1 virtual-link 2.2.2.2
% OSPF: Area 1 is a stub area, cannot configure virtual link
```

### 4.4 排障

```cisco
R1# show ip ospf virtual-links
Virtual Link OSPF_VL0 to router 2.2.2.2 is up
  Run as demand circuit
  DoNotAge LSA allowed.
  Transit area 1, via interface GigabitEthernet0/1
  Topology-MTID    Cost    Disabled     Shutdown      Topology Name
        0           100      no            no            Base

R1# show ip ospf neighbor
Neighbor ID   Pri  State     Dead Time  Address       Interface
2.2.2.2         0  FULL/  -  -          10.0.13.3     OSPF_VL0
                                                       ↑ 虚链路作为一个虚拟接口
```

**虚链路建不起来的排查**：
```
① 两端都配了吗？（Router ID 对不对）
   R1# show run | include virtual-link

② 穿越区域是 Stub 吗？
   R1# show ip ospf | include Area 1 -A 3

③ 到对端 Router ID 的路由通吗？
   R1# show ip route 2.2.2.2
   R1# ping 2.2.2.2
   ★ 必须能通到对端的 Router ID（通常是 Loopback）★

④ 认证匹配吗？
   R1# show ip ospf virtual-links | include auth
```

> **虚链路是"临时补救"，不是"设计方案"。** 需要虚链路说明区域规划有问题。它增加复杂度、增加排障难度，而且是很多诡异故障的来源（比如 `DoNotAge` LSA 导致的 LSDB 不老化问题）。
>
> **能重新规划区域就重新规划，不要长期依赖虚链路。**

---

## ⑤ 其他进阶特性

### 5.1 OSPF 按需电路（Demand Circuit）

用于**按流量计费的链路**（ISDN、卫星），抑制周期性的 Hello 和 LSA 刷新。

```cisco
R1(config-if)# ip ospf demand-circuit
```

**副作用**：LSA 被标记为 **DoNotAge**，永不老化。如果拓扑变化时某台设备没收到更新，**这个错误的 LSA 会永久留在 LSDB 里**。

```cisco
R1# show ip ospf database | include DNA
! DNA = Do Not Age
```

**清理**：`clear ip ospf process`（全网执行）。

### 5.2 TTL 安全（GTSM）

防止远程伪造 OSPF 报文：
```cisco
R1(config-router)# ttl-security all-interfaces hops 1
```
只接受 TTL ≥ 254 的 OSPF 报文（意味着来源必须是 1 跳以内的直连邻居）。

### 5.3 优雅重启（Graceful Restart / NSF）

主控切换时保持转发不中断：
```cisco
R1(config-router)# nsf ietf
! 或 Cisco 私有
R1(config-router)# nsf cisco
```

### 5.4 LSA 限制（防泛洪攻击）

```cisco
R1(config-router)# max-lsa 12000 80 warning-only
!                          ↑     ↑
!                        上限  80%告警
```
**没有这个限制，一次 LSA 泛洪风暴可能耗尽路由器内存导致宕机。**

### 5.5 前缀抑制（减小 LSDB）

不通告传输链路的前缀（只通告 Loopback 和用户网段）：
```cisco
R1(config-router)# prefix-suppression
! 或接口级
R1(config-if)# ip ospf prefix-suppression
```

**在大规模网络中能显著减小 LSDB**——几百条 /30 互联地址不需要被全网知道。

---

## ⑥ 配套实验：OSPF 故障注入全套

**拓扑**：
```
   Area 1              Area 0              Area 2
   [R3]────[R1/ABR]────[R2/ABR]────[R4]────[R5/ASBR]
                                              │
                                        [外部路由]
```

### 实验 A：MTU 不匹配

```cisco
R2(config)# interface GigabitEthernet0/1
R2(config-if)# ip mtu 1400
R1# clear ip ospf process
```

**观察**：
```cisco
R1# show ip ospf neighbor
Neighbor ID  Pri  State       Dead Time  Address      Interface
2.2.2.2        1  EXSTART/DR  00:00:35   10.0.12.2    Gi0/1
                  ↑↑↑↑↑↑↑ 卡住

R1# show logging | include OSPF
%OSPF-5-ADJCHG: Process 1, Nbr 2.2.2.2 on Gi0/1 from EXSTART to DOWN, 
Neighbor Down: Too many retransmissions

R1# debug ip ospf adj
OSPF-1 ADJ Gi0/1: Nbr 2.2.2.2 has larger interface MTU
```

**修复**：`R2(config-if)# ip mtu 1500`

### 实验 B：网络类型不匹配

```cisco
R1(config-if)# ip ospf network point-to-point
! R2 保持默认 broadcast
```

**观察**：邻居不稳定或建不起来。
```cisco
R1# show ip ospf interface Gi0/1 | include Network Type
  Network Type POINT_TO_POINT
R2# show ip ospf interface Gi0/1 | include Network Type
  Network Type BROADCAST
```

**修复**：两端统一。

### 实验 C：Type-3 LSA 过滤

```cisco
! 不让 Area 1 知道 10.99.0.0/16
R1(config)# ip prefix-list BLOCK-99 seq 5 deny 10.99.0.0/16 le 32
R1(config)# ip prefix-list BLOCK-99 seq 10 permit 0.0.0.0/0 le 32

R1(config)# router ospf 1
R1(config-router)# area 1 filter-list prefix BLOCK-99 in
```

**验证（在 R3，Area 1 内部）**：
```cisco
! 过滤前
R3# show ip ospf database summary | include 10.99
10.99.0.0       1.1.1.1  ...

! 过滤后
R3# show ip ospf database summary | include 10.99
! 空 → LSA 都没了 ✓

R3# show ip route ospf | include 10.99
! 空 ✓
```

### 实验 D：distribute-list（对比实验，理解 LSDB 与路由表的区别）

```cisco
R3(config)# ip prefix-list NO-99 seq 5 deny 10.99.0.0/16 le 32
R3(config)# ip prefix-list NO-99 seq 10 permit 0.0.0.0/0 le 32
R3(config)# router ospf 1
R3(config-router)# distribute-list prefix NO-99 in
```

**验证（★ 关键对比）**：
```cisco
R3# show ip ospf database summary | include 10.99
10.99.0.0       1.1.1.1  ...              ← ★ LSA 还在！

R3# show ip route ospf | include 10.99
! 空                                       ← ★ 但路由表里没有
```

**✅ 这个对比清楚地展示了两种过滤的本质区别**：
- `area filter-list` → **不发送 LSA**，影响整个区域
- `distribute-list in` → **收了 LSA 但不装入路由表**，只影响本机

### 实验 E：虚链路

**先制造问题**：
```cisco
! 把 R4-R5 之间改成 Area 2，但 Area 2 不直连 Area 0
R4(config-router)# no network 10.0.45.4 0.0.0.0 area 0
R4(config-router)# network 10.0.45.4 0.0.0.0 area 2
```

**观察**：R5 的路由学不到（Area 2 不合法）。
```cisco
R5# show ip route ospf
! 只有 Area 2 内部的路由
```

**修复：建虚链路（穿越 Area 2... 等等，这里穿越的应该是连接 Area 0 的那个区域）**

调整拓扑为：`Area 0 ──[R2]── Area 1 ──[R4]── Area 2`

```cisco
R2(config-router)# area 1 virtual-link 4.4.4.4
R4(config-router)# area 1 virtual-link 2.2.2.2
```

**验证**：
```cisco
R2# show ip ospf virtual-links
Virtual Link OSPF_VL0 to router 4.4.4.4 is up
  Transit area 1, via interface GigabitEthernet0/1

R5# show ip route ospf
O IA  10.0.0.0/24 [110/300] via ...        ← 学到了 ✓
```

**再测试限制**：
```cisco
R2(config-router)# area 1 stub
% OSPF: Area 1 is a stub area, cannot configure virtual link
                                     ↑ 验证了"穿越区域不能是 Stub"
```

### 实验 F：Router ID 重复

```cisco
R4(config-router)# router-id 2.2.2.2
R4# clear ip ospf process
```

**观察**：
```cisco
R2# show logging | include DUP_RTRID
%OSPF-4-DUP_RTRID_NBR: OSPF-1 Process detected duplicate router-id 2.2.2.2 
from 10.0.24.4 on interface GigabitEthernet0/2

R2# show ip ospf neighbor
! 状态反复在 FULL 和 DOWN 之间翻转
```

---

## ⑦ 排障速查表

| 症状 | 根因 | 验证 |
|:--|:--|:--|
| 无邻居 | 接口没进 OSPF / passive / down | `show ip ospf interface brief` |
| 卡 **INIT** | ACL 拦组播 / 单向链路 | `show access-lists`、抓包 |
| 卡 **EXSTART/EXCHANGE** | **MTU 不匹配** | `show ip ospf interface \| inc MTU` |
| 停 **2WAY** | DROTHER 之间（正常） | `show ip ospf neighbor` 看角色 |
| 邻居翻转 | **Router ID 重复** / 链路抖动 | `show logging \| inc DUP_RTRID` |
| 建不起来但参数看似一致 | **网络类型不匹配** | `show ip ospf interface \| inc Network Type` |
| 配 Stub 后邻居断 | Stub 标志不一致 | 区域内所有路由器都要配 |
| 学不到区域间路由 | ABR / **Type-3 被过滤** | `show ip ospf database summary` |
| **学不到外部路由** | **缺 Type-4** / 区域是 Stub | `show ip ospf database asbr-summary` |
| LSDB 有但路由表没有 | **distribute-list** / AD 被抢 | `show ip protocols \| inc distribute` |
| 虚链路起不来 | 穿越区域是 Stub / 路由不通 | `show ip ospf virtual-links` |
| 万兆千兆等价 | 参考带宽没改 | `show ip ospf \| inc Reference` |
| SPF 频繁运行 | 拓扑震荡 | `show ip ospf statistics` |
| LSDB 有 DNA 标记的 LSA | demand-circuit | `show ip ospf database \| inc DNA` |

### SPF 震荡定位

```cisco
R1# show ip ospf statistics
  Area 0: SPF algorithm executed 1245 times           ← 异常高

  SPF calculation time
  Delta T   Intra D-Intra Summ D-Summ Ext D-Ext Total Reason
  00:00:12       0        0     0      0    0     0     0    R
  00:00:23       0        0     0      0    0     0     0    R, N
                                                            ↑
                        R  = Router LSA 变化
                        N  = Network LSA 变化
                        SN = Summary Net LSA 变化
                        SA = Summary ASBR LSA 变化
                        X  = External LSA 变化
```

**找出抖动源**：
```cisco
R1# show ip ospf database router | include Advertising|Seq
  Advertising Router: 3.3.3.3
  LS Seq Number: 0x80000456                          ← 序列号增长最快的就是抖动源
```

序列号增长最快的那台设备，就是在不断重新生成 LSA 的那台——去查它的接口是否在 flap。

---

## ⑧ 自测题

**1.** OSPF 邻居卡在 EXSTART，除了 MTU 还可能是什么原因？

<details><summary>答案</summary>

**MTU 占 99%，但还有其他可能**：

| 原因 | 排查方法 |
|:--|:--|
| **① MTU 不匹配** | `show ip ospf interface \| include MTU`，两端对比 |
| **② 单向链路** | 两端接口计数器对比：一端 output 有数字，另一端 input 是 0 |
| **③ ACL 拦了大包** | `show access-lists`，检查是否有基于包大小或某些特征的过滤 |
| **④ Router ID 重复** | `show logging \| include DUP_RTRID` |
| **⑤ 中间设备丢大包** | `ping <邻居> size 1500 df-bit` |
| **⑥ 路径 MTU 不足**（隧道场景） | GRE/IPsec 隧道两端 MTU 一致，但实际路径不够 |
| ⑦ CPU 过载导致处理不过来 | `show processes cpu sorted` |

**为什么这些都会卡在 EXSTART**：

EXSTART 阶段要交换 **DD（Database Description）报文**协商主从关系。DD 报文可能比较大。如果：
- MTU 不匹配 → 对端**显式拒绝**（OSPF 有 MTU 检查机制）
- 单向链路 → 发出去了但收不到响应
- 大包被丢 → 超时重传，重传次数超限

都表现为**卡在 EXSTART，然后超时回到 DOWN，再重试**。

**日志的区分**：
```cisco
! MTU 不匹配（有明确提示）
R1# debug ip ospf adj
OSPF-1 ADJ Gi0/1: Nbr 2.2.2.2 has larger interface MTU

! 超时（原因不明确，需要进一步排查）
%OSPF-5-ADJCHG: Process 1, Nbr 2.2.2.2 on Gi0/1 from EXSTART to DOWN, 
Neighbor Down: Too many retransmissions
```

**完整排查流程**：
```
① 对比两端 MTU
   R1# show ip ospf interface Gi0/1 | include MTU
   R2# show ip ospf interface Gi0/1 | include MTU
   
② 测试大包能否通过
   R1# ping 10.0.12.2 size 1500 df-bit
   → 不通 → MTU 或路径问题
   
③ 检查双向流量
   R1# show interfaces Gi0/1 | include packets input|packets output
   R2# show interfaces Gi0/1 | include packets input|packets output
   → 一端 input 为 0 → 单向链路
   
④ 检查 Router ID
   R1# show logging | include DUP_RTRID
   
⑤ 开调试（慎用）
   R1# debug ip ospf adj
```

**修复**：
```cisco
! 方案 1（推荐）：统一 MTU
R2(config-if)# ip mtu 1500

! 方案 2（应急）：忽略 MTU 检查
R1(config-if)# ip ospf mtu-ignore
R2(config-if)# ip ospf mtu-ignore
```

**⚠️ `mtu-ignore` 的风险**：邻居能起来，但如果实际路径 MTU 真的不足，**大的 LSU 报文依然会被丢弃**，导致 LSDB 同步不完整。症状变成"邻居 FULL 但某些路由学不到"，更难查。

**优先统一 MTU，`mtu-ignore` 只作为临时手段。**
</details>

**2.** `area X filter-list` 和 `distribute-list in` 有什么区别？

<details><summary>答案</summary>

| | **`area X filter-list`** | **`distribute-list ... in`** |
|:--|:--|:--|
| 配在哪 | **ABR** | 任意 OSPF 路由器 |
| 过滤什么 | **Type-3 LSA** | **进入路由表的路由** |
| **影响 LSDB？** | ✅ **影响**（LSA 根本不发送） | ❌ **不影响**（LSA 照收） |
| 影响范围 | **整个区域**（所有路由器都收不到） | **只影响本机** |
| 方向 | `in`（进区域）/ `out`（出区域） | 只有 `in` 有效 |

**关键实验对比**：

```cisco
! ── 用 area filter-list ──
R-ABR(config-router)# area 1 filter-list prefix BLOCK-99 in

! 在 Area 1 内部的 R3 上看：
R3# show ip ospf database summary | include 10.99
! ★ 空 —— LSA 根本没发过来 ★
R3# show ip route ospf | include 10.99
! 空
```

```cisco
! ── 用 distribute-list in ──
R3(config-router)# distribute-list prefix NO-99 in

! 在 R3 上看：
R3# show ip ospf database summary | include 10.99
10.99.0.0       1.1.1.1  ...
! ★ LSA 还在！★
R3# show ip route ospf | include 10.99
! ★ 但路由表里没有 ★
```

**为什么 OSPF 不能像 EIGRP 那样随便过滤 LSA**：

OSPF 是**链路状态协议**，**区域内所有路由器的 LSDB 必须完全一致**。

```
   如果 R3 的 LSDB 少了一条 LSA
        ↓
   R3 跑 SPF 算出的拓扑图和其他路由器不同
        ↓
   ★ 可能形成路由环路 ★
   （R3 认为该走 A 路径，R4 认为该走 B 路径，互相指来指去）
```

**所以**：
- **区域内不能过滤 LSA**（`distribute-list` 只能过滤路由表，不能过滤 LSA）
- **区域之间可以过滤**（`area filter-list`）—— 因为区域间路由本来就是"距离矢量式"的（相信 ABR 说的），不需要完整拓扑

**选择建议**：

| 需求 | 用什么 |
|:--|:--|
| 不让整个区域知道某些路由（**减小 LSDB，省内存**） | **`area X filter-list`**（ABR） |
| **只有本机**不想用某条路由（比如本机有更好的路径） | **`distribute-list in`** |
| 不让外部路由进入某区域 | **Stub / Totally Stub** |
| 汇总时抑制某个网段 | **`area X range ... not-advertise`** |
| 过滤重分发进来的外部路由 | **`summary-address ... not-advertise`** 或 route-map |

**方向的理解（`area filter-list` 最容易搞反）**：

```
   Area 0 ──── [ABR] ──── Area 1
   
   area 1 filter-list prefix X in
   → 过滤 ABR ★向 Area 1 发送★ 的 Type-3
   → "外面的路由进不来 Area 1"
   
   area 1 filter-list prefix X out
   → 过滤 ABR ★从 Area 1 向外发送★ 的 Type-3
   → "Area 1 的路由出不去"
```

**方向是相对于"这个 area"的，不是相对于路由器。**
</details>

**3.** 虚链路的穿越区域为什么不能是 Stub？

<details><summary>答案</summary>

**因为虚链路的作用是传递 Type-3 / Type-4 / Type-5 LSA，而 Stub 区域会拒绝 Type-4/5，两者矛盾。**

**虚链路的本质**：
```
   Area 0 ──[ABR-1]── Area 1（穿越区域）──[ABR-2]── Area 2
                          ↑
              虚链路"穿过"这个区域，把 ABR-1 和 ABR-2
              逻辑上连成 ★ Area 0 的一部分 ★
```

虚链路上要传递：
- **Type-3 LSA**（区域间路由）
- **Type-4 LSA**（ASBR 位置）
- **Type-5 LSA**（外部路由）

**Stub 区域的定义就是"拒绝 Type-4/5"**（Totally Stub 连 Type-3 也拒绝）。

**如果穿越区域是 Stub**：
```
   虚链路想传 Type-5 过去
        ↓
   Stub 区域的规则：不接受 Type-5
        ↓
   ★ 直接矛盾 ★
```

**IOS 会直接拒绝配置**：
```cisco
R1(config-router)# area 1 stub
R1(config-router)# area 1 virtual-link 2.2.2.2
% OSPF: Area 1 is a stub area, cannot configure virtual link
```

**反过来也一样**：
```cisco
R1(config-router)# area 1 virtual-link 2.2.2.2
R1(config-router)# area 1 stub
% OSPF: Cannot configure stub area 1 with virtual link
```

**虚链路的其他限制（一起记）**：

| 限制 | 说明 |
|:--|:--|
| **穿越区域不能是 Stub / NSSA / Totally Stub** | 上述原因 |
| **穿越区域必须有到对端 Router ID 的完整路由** | 虚链路靠单播通信，必须能路由到对端 |
| **两端必须都是 ABR** | 非 ABR 不能建虚链路 |
| **参数是 Router ID，不是 IP** | 常见配置错误 |
| **两端都要配** | 单边配置无效 |
| **穿越区域不能有汇总/过滤阻断到 Router ID 的路由** | 否则虚链路建不起来 |

**排障流程**：
```cisco
! ① 两端配置对不对
R1# show run | include virtual-link
 area 1 virtual-link 2.2.2.2

! ② 穿越区域的类型
R1# show ip ospf | include Area 1 -A 3
    Area 1
        Number of interfaces in this area is 2
        (不应该出现 "It is a stub area")

! ③ 能路由到对端 Router ID 吗
R1# show ip route 2.2.2.2
R1# ping 2.2.2.2                        ← ★ 必须通

! ④ 虚链路状态
R1# show ip ospf virtual-links
Virtual Link OSPF_VL0 to router 2.2.2.2 is up
  Transit area 1, via interface GigabitEthernet0/1
```

**⚠️ 虚链路的另一个副作用：`DoNotAge` LSA**

虚链路上传的 LSA 会被标记为 **DNA（Do Not Age）**，永不老化。
```cisco
R1# show ip ospf database | include DNA
```
如果拓扑变化时某台设备没收到更新，**这条错误的 LSA 会永久留在 LSDB 里**，产生持续的路由异常。清理需要全网 `clear ip ospf process`。

**这就是为什么虚链路是"临时补救"而非"设计方案"**：
- 增加复杂度和排障难度
- DNA LSA 可能导致陈旧路由
- 依赖穿越区域的稳定性

**如果你的网络需要虚链路，说明区域规划有问题，应该考虑重新设计**（比如把 Area 2 直接连到 Area 0，或者用 GRE 隧道模拟一条到 Area 0 的直连链路）。
</details>

**4.** OSPF 网络类型不匹配会怎样？哪些类型可以互通？

<details><summary>答案</summary>

**网络类型不匹配会导致邻居建不起来，或者建起来但不稳定。**

**两个不匹配的根源**：
1. **Hello/Dead 时间不同**（10/40 vs 30/120）→ Hello 参数不匹配，报文直接被丢弃
2. **DR 需求不同** → 一端选 DR 一端不选，行为冲突

**六种网络类型**：

| 类型 | 需要 DR | Hello/Dead | 需手工邻居 | 典型场景 |
|:--|:--|:--|:--|:--|
| **Broadcast** | ✅ | **10 / 40** | ❌ | 以太网（默认） |
| **Non-Broadcast (NBMA)** | ✅ | **30 / 120** | ✅ | 帧中继 |
| **Point-to-Point** | ❌ | **10 / 40** | ❌ | 串口、隧道（默认） |
| **Point-to-Multipoint** | ❌ | **30 / 120** | ❌ | 星型帧中继 |
| **P2MP Non-Broadcast** | ❌ | **30 / 120** | ✅ | 特殊场景 |
| Loopback | — | — | — | 环回口 |

**兼容性矩阵**：

| | Broadcast | P2P | NBMA | P2MP |
|:--|:--|:--|:--|:--|
| **Broadcast** | ✅ | ❌ | ❌ | ❌ |
| **P2P** | ❌ | ✅ | ❌ | ❌ |
| **NBMA** | ❌ | ❌ | ✅ | ❌ |
| **P2MP** | ❌ | ❌ | ❌ | ✅ |

**★ 只有相同类型才能建立邻居。**

**排查**：
```cisco
R1# show ip ospf interface GigabitEthernet0/1
GigabitEthernet0/1 is up, line protocol is up
  Internet Address 10.0.12.1/30, Area 0
  Process ID 1, Router ID 1.1.1.1, Network Type POINT_TO_POINT, Cost: 100
                                                 ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5

R2# show ip ospf interface GigabitEthernet0/1
  Process ID 1, Router ID 2.2.2.2, Network Type BROADCAST, Cost: 100
                                                 ↑↑↑↑↑↑↑↑↑ 不匹配
```

**修复**：两端统一。
```cisco
R2(config-if)# ip ospf network point-to-point
```

**★ 实践建议：把两台路由器之间的以太网链路改成 point-to-point**

```cisco
R1(config-if)# ip ospf network point-to-point
R2(config-if)# ip ospf network point-to-point
```

**好处**：

| 好处 | 说明 |
|:--|:--|
| **收敛更快** | 省掉 DR/BDR 选举（Wait 40 秒） |
| **LSDB 更小** | **不产生 Type-2（Network）LSA** |
| **没有 DR 抢占困扰** | 不需要担心非抢占带来的次优 DR |
| **配置更简单** | 不需要调 `ip ospf priority` |

**这在数据中心和核心互联链路上是标准做法。**

**⚠️ 什么时候不能改成 P2P**：
- 该网段上有**3 台或以上**路由器（真正的多路访问）
- 只有一端能改（两端必须都改）

**特殊场景：P2P 类型下的掩码检查**

改成 P2P 后，OSPF **不再检查子网掩码是否匹配**（因为点对点链路上不需要）。这意味着：
```cisco
R1(config-if)# ip address 10.0.12.1 255.255.255.252    ! /30
R2(config-if)# ip address 10.0.12.2 255.255.255.0      ! /24
! Broadcast 类型下：邻居建不起来（掩码不匹配）
! P2P 类型下：★ 邻居能建起来 ★
```

这可以是个"特性"（容忍配置错误），也可以是个"坑"（掩盖了真正的配置问题）。**排障时要留意。**
</details>

**5.** `show ip ospf database` 里有某条 LSA，但 `show ip route ospf` 里没有对应路由。可能是什么原因？

<details><summary>答案</summary>

**LSDB 有但路由表没有，说明"收到了但没采纳"。问题在【本机】，不在上游。**

**这是 OSPF 排障最有价值的一个判断**——它把问题范围缩小了一半。

**六个可能的原因**：

**① 有 AD 更低的路由抢赢了**
```cisco
R1# show ip route 10.1.1.0
Routing entry for 10.1.1.0/24
  Known via "static", distance 1, metric 0        ← ★ 静态路由 AD=1 < OSPF 110
  Routing Descriptor Blocks:
  * 10.0.12.2

R1# show ip route 10.1.1.0 | include distance
```
**这是最常见的原因。** 检查是否有静态路由、EIGRP（AD 90）、eBGP（AD 20）学到了同一个前缀。

**② 被 `distribute-list in` 过滤了**
```cisco
R1# show ip protocols | include distribute
  Incoming update filter list for all interfaces is 10
                                                   ↑ 有过滤器

R1# show access-lists 10
Standard IP access list 10
    10 deny 10.1.1.0, wildcard bits 0.0.0.255      ← 找到了
    20 permit any
```

**③ 下一跳不可达（转发地址问题）**
```cisco
R1# show ip ospf database external 10.1.1.0
  Forward Address: 192.168.99.1                    ← 转发地址
  
R1# show ip route 192.168.99.1
% Network not in table                             ← ★ 转发地址不可达
```

**Type-5 LSA 的 Forward Address（FA）**：
- FA = 0.0.0.0 → 流量发给 ASBR
- FA ≠ 0.0.0.0 → **流量发给 FA 指定的地址**，如果这个地址在路由表里不可达，**路由不会被安装**

**FA 非零的条件**（考点）：ASBR 的下一跳在 OSPF 通告的网段内，且该接口不是 passive。

**④ 缺 Type-4 LSA（外部路由专有）**
```cisco
R1# show ip ospf database external 10.1.1.0
  Advertising Router: 5.5.5.5                      ← ASBR

R1# show ip ospf database asbr-summary 5.5.5.5
! 空！★ 不知道怎么到 ASBR ★
```
**没有 Type-4，就不知道怎么到达 ASBR，路由不可达。**

**⑤ SPF 还没运行完（短暂）**
```cisco
R1# show ip ospf | include SPF
 SPF algorithm last executed 00:00:02.345 ago
```
刚收到 LSA，SPF 还在计算。等几秒再看。

**⑥ LSA 是自己产生的（自己不会用自己的）**
```cisco
R1# show ip ospf database external 10.1.1.0
  Advertising Router: 1.1.1.1                      ← 就是本机
```
路由器不会把自己通告的 LSA 装进路由表（它本来就有那条路由，来自其他协议或直连）。

**⑦ 路由表已满 / 内存不足（罕见）**
```cisco
R1# show ip route summary
R1# show processes memory sorted
```

**完整排查流程**：

```
① 确认 LSA 在 LSDB 里
   R1# show ip ospf database summary 10.1.1.0
   R1# show ip ospf database external 10.1.1.0
        ↓ 有
② 看路由表里这个前缀是什么来源
   R1# show ip route 10.1.1.0
   → "Known via static/eigrp/bgp" → ★ AD 被抢
        ↓ 没有任何路由
③ 检查过滤器
   R1# show ip protocols | include distribute|filter
        ↓ 没有过滤
④ 如果是外部路由，检查 Type-4 和 FA
   R1# show ip ospf database asbr-summary <ASBR-ID>
   R1# show ip ospf database external <前缀> | include Forward
   R1# show ip route <FA地址>
        ↓ 都正常
⑤ 强制重算
   R1# clear ip ospf process        ⚠️ 会中断邻居
```

**反过来的判断（同样重要）**：

| 现象 | 问题在哪 |
|:--|:--|
| **LSDB 有，路由表没有** | ★ **本机**（AD/过滤/FA/Type-4） |
| **LSDB 也没有** | ★ **上游**（对方没通告/被 area filter-list 过滤/区域类型拒绝） |

**这个二分法能让你在 30 秒内确定该去哪台设备排查。**
</details>

---

**上一章** ← [01 EIGRP 深入与排障](01-EIGRP深入与排障.md) ｜ **下一章** → [03 BGP 深入与排障](03-BGP深入与排障.md)
