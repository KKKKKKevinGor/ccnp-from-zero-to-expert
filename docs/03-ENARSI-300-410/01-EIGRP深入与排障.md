# 01 · EIGRP 深入与排障

> 基础见 [ENCOR 第 6 章](../02-ENCOR-350-401/06-EIGRP.md)。本章聚焦**排障和进阶特性**。

## ① 排障框架

```
   ① 邻居建立了吗？
      show ip eigrp neighbors
        ├─ 没有邻居 → 走 ② 邻居排查
        └─ 有邻居但翻转 → 走 ③ 稳定性排查
        ↓
   ② 路由学到了吗？
      show ip eigrp topology
        ├─ 拓扑表没有 → 对方没通告 / 被过滤
        └─ 拓扑表有但路由表没有 → AD 被抢 / 被过滤
        ↓
   ③ 路径最优吗？
      show ip eigrp topology <网段>
        └─ 看 FD/RD，判断 Successor 和 FS
        ↓
   ④ 收敛快吗？
      有 FS 吗？没有的话会进 Active 查询
```

---

## ② 邻居问题排查

### 2.1 建立邻居的条件（对比 OSPF）

| 条件 | EIGRP | OSPF |
|:--|:--|:--|
| **AS 号 / 进程号** | ✅ **必须相同** | ❌ 可以不同 |
| **K 值** | ✅ **必须相同** | — |
| **在同一子网** | ✅ 必须 | ✅ 必须 |
| **认证** | ✅ 必须匹配 | ✅ 必须匹配 |
| **Hello / Hold 时间** | ❌ **不需要相同** | ✅ **必须相同** |
| **MTU** | ❌ **不检查** | ✅ **必须相同**（否则卡 EXSTART） |
| **Router ID** | 不强制唯一（但重复影响外部路由） | ✅ 必须唯一 |
| Stub 标志 | — | ✅ 必须一致 |

> **考点**：EIGRP 的 Hello/Hold **不需要匹配**（一端 5 秒一端 60 秒也能建邻居），这和 OSPF 完全不同。但 **Hold 时间过短会导致邻居误判断开**——如果一端 Hello 60 秒而另一端 Hold 15 秒，邻居会不断超时重建。

### 2.2 常见邻居故障

**故障 A：AS 号不匹配**
```cisco
R1(config)# router eigrp 100
R2(config)# router eigrp 200          ! 不同
```
**症状**：完全没有邻居，也没有任何日志（EIGRP 直接丢弃 AS 号不匹配的包）。
**排查**：
```cisco
R1# show ip protocols | include Routing Protocol
Routing Protocol is "eigrp 100"
R2# show ip protocols | include Routing Protocol
Routing Protocol is "eigrp 200"       ← 找到了
```

**故障 B：K 值不匹配**
```cisco
R2(config-router)# metric weights 0 1 1 1 0 0      ! 改了 K2
```
**症状**：邻居建不起来，**有明确日志**：
```
%DUAL-5-NBRCHANGE: EIGRP-IPv4 100: Neighbor 10.0.12.2 (Gi0/1) is down: K-value mismatch
```
**排查**：
```cisco
R1# show ip protocols | include K1
  EIGRP metric weight K1=1, K2=0, K3=1, K4=0, K5=0
R2# show ip protocols | include K1
  EIGRP metric weight K1=1, K2=1, K3=1, K4=0, K5=0    ← K2 不同
```

> **不要随便改 K 值**。把 Load（K2）和 Reliability（K4/K5）加入计算会让 metric 随流量波动，导致**路由持续震荡**。默认的 K1/K3（带宽+延迟）是稳定的。

**故障 C：认证不匹配**
```cisco
R1(config-if)# ip authentication key-chain eigrp 100 KEY1
R2(config-if)# ip authentication key-chain eigrp 100 KEY2      ! 密钥不同
```
**症状**：
```
%DUAL-5-NBRCHANGE: ... is down: Auth failure
```
**排查**：
```cisco
R1# show ip eigrp interfaces detail Gi0/1 | include Auth
  Authentication mode is md5, key-chain is "KEY1"

R1# debug eigrp packets
EIGRP: ignored packet from 10.0.12.2, opcode = 5 (invalid authentication)
```

**★ key-chain 的时间陷阱**：
```cisco
R1(config)# key chain KEY1
R1(config-keychain)# key 1
R1(config-keychain-key)#  key-string MySecret
R1(config-keychain-key)#  accept-lifetime 00:00:00 Jan 1 2026 23:59:59 Dec 31 2026
R1(config-keychain-key)#  send-lifetime 00:00:00 Jan 1 2026 23:59:59 Dec 31 2026
```
如果**系统时间不对**（NTP 没同步），key 会被认为"未生效"或"已过期"，认证失败。**排查认证问题时先看 `show clock`。**

```cisco
R1# show key chain
Key-chain KEY1:
    key 1 -- text "MySecret"
        accept lifetime (00:00:00 Jan 1 2026) - (23:59:59 Dec 31 2026) [valid now]
        send lifetime (00:00:00 Jan 1 2026) - (23:59:59 Dec 31 2026) [valid now]
                                                                      ↑ 必须是 valid now
```

**故障 D：接口是 passive**
```cisco
R1(config-router)# passive-interface GigabitEthernet0/1
```
**症状**：该接口的邻居消失，但网段还在通告。
**排查**：
```cisco
R1# show ip protocols | section Passive
  Passive Interface(s):
    GigabitEthernet0/1                ← 找到了

R1# show ip eigrp interfaces
! Gi0/1 不在列表里
```

**故障 E：单播邻居（NBMA/隧道场景）**
```cisco
R1(config-router)# neighbor 10.0.12.2 GigabitEthernet0/1
```
配了静态邻居后，**该接口的组播 Hello 会被禁用**，只用单播。如果只有一端配了静态邻居，邻居建不起来。

---

## ③ SIA 深入

### 3.1 SIA 的完整机制

```
   ① Successor 失效，且【没有 FS】
        ↓
   ② 路由进入 ★ Active 状态 ★
        ↓
   ③ 向所有邻居（Stub 除外）发 QUERY
        ↓
   ④ 邻居收到 QUERY：
      ├─ 我知道 → 回 REPLY
      ├─ 我不知道，但我有别的邻居 → ★ 继续向下游发 QUERY ★（扩散！）
      └─ 我也没路径 → 回 REPLY（unreachable）
        ↓
   ⑤ 等待所有 REPLY
        ↓
   ⑥ Active Timer（默认 3 分钟）超时
        ↓
   ⑦ ★ SIA 触发 → 强制断开没回应的邻居 ★
        ↓
   ⑧ 邻居重建 → 可能引发连锁震荡
```

**根本问题：查询的扩散范围不可控。**

### 3.2 SIA-Query / SIA-Reply（IOS 12.4+）

现代 IOS 增加了一个中间步骤，**避免误杀健康但慢的邻居**：

```
   Active Timer 走到一半（1.5 分钟）
        ↓
   发送 ★ SIA-Query ★："你还活着吗？还在处理吗？"
        ↓
   ├─ 收到 SIA-Reply → 邻居活着只是慢 → ★ 延长等待，不断邻居 ★
   └─ 没收到 → 确认有问题 → 到时断开
```

```cisco
R1# show ip eigrp topology active
A 10.1.1.0/24, 0 successors, FD is Inaccessible
  1 replies, active 00:01:52, query-origin: Local origin
        via 10.0.13.3 (Infinity/Infinity), r, Q, GigabitEthernet0/2
                                            ↑  ↑
                                     等待reply  已发SIA-Query
```

### 3.3 三种防 SIA 手段的对比

| 手段 | 原理 | 效果 | 代价 |
|:--|:--|:--|:--|
| **路由汇总** | 汇总点收到明细的 QUERY 时**立即回 REPLY**（"我有汇总覆盖它"），查询到此为止 | ★★★★★ | 需要地址规划支持 |
| **Stub 路由** | 上游**根本不向 Stub 邻居发 QUERY** | ★★★★ | Stub 不能做中转 |
| **调 Active Timer** | 给慢邻居更多时间 | ★ 治标 | 延长故障时间 |

**汇总的工作原理（关键理解）**：

```
   R-Core ──汇总 10.1.0.0/16──> R-Dist ──明细──> R-Access (10.1.1.0/24)
   
   R-Core 上没有 10.1.1.0/24 的明细，只有汇总
        ↓
   收到关于 10.1.1.0/24 的 QUERY
        ↓
   ★ R-Core 检查：我有 10.1.0.0/16 的汇总，覆盖了它 ★
        ↓
   ★ 立即回 REPLY，不再向自己的其他邻居扩散查询 ★
        ↓
   查询范围被限制在 R-Dist 以下
```

**这就是"汇总建立查询边界（Query Boundary）"** —— EIGRP 大规模部署的核心设计原则。

### 3.4 Stub 的六个选项

```cisco
R1(config-router)# eigrp stub [connected] [summary] [static] [redistributed] [receive-only] [leak-map NAME]
```

| 选项 | 通告什么 | 用途 |
|:--|:--|:--|
| `connected` | 直连路由 | ★ 分支的 LAN 网段 |
| `summary` | 汇总路由 | ★ 分支做了汇总时 |
| `static` | 静态路由 | 需配合 `redistribute static` |
| `redistributed` | 重分发进来的路由 | 分支连了其他协议 |
| `receive-only` | **什么都不通告** | 纯接收端（罕见） |
| `leak-map NAME` | 有选择地泄露特定路由 | 精细控制 |

**默认（只写 `eigrp stub`）= `connected summary`**

**分支路由器标准配置**：
```cisco
Branch(config)# router eigrp 100
Branch(config-router)# eigrp stub connected summary
```

**验证上游是否真的不发查询**：
```cisco
Core# show ip eigrp neighbors detail
H   Address      Interface   Hold Uptime   SRTT  RTO  Q  Seq
0   10.0.13.3    Gi0/2         12  00:15:23  10  100  0  18
    Version 23.0/2.0, Retrans: 0, Retries: 0, Prefixes: 5
    Topology-ids from peer - 0
    Stub Peer Advertising ( CONNECTED SUMMARY ) Routes
    ★ Suppressing queries ★                ← 确认不发查询
```

**⚠️ Stub 的陷阱：Stub 路由器不能做中转。**

```
   R-Core ──── R-Stub ──── R-Remote
   
   如果 R-Stub 配了 stub，R-Core 不会向它发查询，
   也不会通过它学到 R-Remote 的路由（除非 leak-map）
        ↓
   ★ R-Remote 变成孤岛 ★
```

**所以：只有真正的"末梢"路由器才能配 Stub。** 中间的汇聚设备绝对不能配。

---

## ④ 路由汇总深入

### 4.1 两种汇总

```cisco
! ── 接口级汇总（经典模式）──
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip summary-address eigrp 100 10.1.0.0 255.255.0.0
R1(config-if)# ip summary-address eigrp 100 10.2.0.0 255.255.0.0 200    ! 指定 AD

! ── 命名模式 ──
R1(config)# router eigrp CORP
R1(config-router)# address-family ipv4 unicast autonomous-system 100
R1(config-router-af)#  af-interface GigabitEthernet0/1
R1(config-router-af-interface)#   summary-address 10.1.0.0 255.255.0.0
```

### 4.2 汇总的三个副作用（★ 排障重点）

**副作用 1：自动生成 Null0 路由**
```cisco
R1# show ip route | include Null0
D    10.1.0.0/16 is a summary, 00:05:23, Null0
```
**作用**：落在汇总范围内但没有明细的流量直接丢弃，**防止路由环路**。

**副作用 2：抑制明细路由的通告**

配了汇总后，该接口**只发汇总，不发明细**。如果下游需要明细做精细选路，会失效。

**副作用 3（★ 最容易踩坑）：汇总路由的 AD 是 5**

```cisco
R1# show ip route 10.1.0.0
D    10.1.0.0/16 [5/2816] is a summary, Null0
                  ↑ AD = 5，不是 90！
```

**这可能导致意外的路由选择**：
```
   R1 有：
   · EIGRP 汇总路由 10.1.0.0/16，AD=5
   · 静态路由 10.1.0.0/16 → 某个下一跳，AD=1
   
   静态路由 AD=1 < 5，静态赢 ✓（符合预期）
   
   但如果是：
   · EIGRP 汇总 10.1.0.0/16，AD=5
   · OSPF 学到 10.1.0.0/16，AD=110
   
   ★ EIGRP 汇总赢 ★，流量走 Null0 被丢弃！
```

**修复**：
```cisco
R1(config-if)# ip summary-address eigrp 100 10.1.0.0 255.255.0.0 250
!                                                                 ↑ 把 AD 调高
```

### 4.3 Leak Map（汇总时泄露特定明细）

**场景**：做了汇总，但某条明细路由必须让下游看到（比如它需要走不同的路径）。

```cisco
R1(config)# ip prefix-list LEAK-ME seq 5 permit 10.1.5.0/24
R1(config)# route-map LEAK-MAP permit 10
R1(config-route-map)#  match ip address prefix-list LEAK-ME

R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip summary-address eigrp 100 10.1.0.0 255.255.0.0 leak-map LEAK-MAP
```

**结果**：下游同时收到 `10.1.0.0/16`（汇总）和 `10.1.5.0/24`（明细）。

---

## ⑤ 不等价负载均衡深入

```cisco
R1(config-router)# variance 2
R1(config-router)# maximum-paths 6
R1(config-router)# traffic-share balanced        ! 按 metric 反比（默认）
! 或
R1(config-router)# traffic-share min across-interfaces    ! 只用最优路径，但保留备份
```

**参与条件（两个都要满足）**：
1. `FD ≤ variance × 最优FD`
2. **必须是 Feasible Successor**（满足 FC）

**排障：配了 variance 但没生效**
```cisco
R1# show ip eigrp topology 10.1.1.0/24
P 10.1.1.0/24, 1 successors, FD is 3072
        via 10.0.12.2 (3072/2816), Gi0/1        ← Successor
! 只显示一条 → 说明另一条不是 FS

R1# show ip eigrp topology all-links
P 10.1.1.0/24, 1 successors, FD is 3072
        via 10.0.12.2 (3072/2816), Gi0/1
        via 10.0.13.3 (5500/4000), Gi0/2        ← RD=4000 > FD=3072，不满足 FC
                                    ↑ 即使 FD 5500 ≤ 6144，也不能参与
```

**修复思路**：调整接口 delay，让备份路径的 RD 变小。
```cisco
R3(config)# interface GigabitEthernet0/1
R3(config-if)# delay 10                          ! 降低延迟 → 降低 RD
```

---

## ⑥ EIGRP 认证与安全

```cisco
! ── MD5 认证（传统）──
R1(config)# key chain EIGRP-KEYS
R1(config-keychain)# key 1
R1(config-keychain-key)#  key-string OldSecret
R1(config-keychain-key)#  accept-lifetime 00:00:00 Jan 1 2026 infinite
R1(config-keychain-key)#  send-lifetime 00:00:00 Jan 1 2026 23:59:59 Jun 30 2026
R1(config-keychain)# key 2
R1(config-keychain-key)#  key-string NewSecret
R1(config-keychain-key)#  accept-lifetime 00:00:00 Jun 1 2026 infinite
R1(config-keychain-key)#  send-lifetime 00:00:00 Jul 1 2026 infinite

R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip authentication mode eigrp 100 md5
R1(config-if)# ip authentication key-chain eigrp 100 EIGRP-KEYS

! ── SHA-256（命名模式，更安全）──
R1(config)# router eigrp CORP
R1(config-router)# address-family ipv4 unicast autonomous-system 100
R1(config-router-af)#  af-interface GigabitEthernet0/1
R1(config-router-af-interface)#   authentication mode hmac-sha-256 MyStrongPassword
```

**★ 密钥轮换的正确做法（考点）**：

上面的 key-chain 配置演示了**平滑轮换**：
```
   key 1: accept 永久有效，send 到 6/30 为止
   key 2: accept 从 6/1 开始，send 从 7/1 开始
   
   6/1 - 6/30：发 key1，接受 key1 和 key2  ← ★ 重叠期 ★
   7/1 起：   发 key2，接受 key1 和 key2
        ↓
   全网都切换完后，再删除 key 1
```

**重叠期是关键**：让全网设备有时间逐台切换，期间新旧密钥都能工作，**不会中断邻居**。

**验证**：
```cisco
R1# show key chain EIGRP-KEYS
Key-chain EIGRP-KEYS:
    key 1 -- text "OldSecret"
        accept lifetime (00:00:00 Jan 1 2026) - (infinite) [valid now]
        send lifetime (00:00:00 Jan 1 2026) - (23:59:59 Jun 30 2026)
    key 2 -- text "NewSecret"
        accept lifetime (00:00:00 Jun 1 2026) - (infinite) [valid now]
        send lifetime (00:00:00 Jul 1 2026) - (infinite) [valid now]
```

---

## ⑦ 配套实验：故障注入全套

**拓扑**：
```
              ┌──────┐
        ┌─────┤  R1  ├─────┐
        │     └──────┘     │
    ┌───┴──┐          ┌────┴─┐
    │  R2  ├──────────┤  R3  │
    └───┬──┘          └────┬─┘
        │                  │
   10.1.1.0/24        10.1.2.0/24
```

### 实验 A：制造 SIA

**Step 1：确认当前有 FS（正常状态）**
```cisco
R1# show ip eigrp topology 10.1.1.0/24
P 10.1.1.0/24, 1 successors, FD is 3072
        via 10.0.12.2 (3072/2816), Gi0/1
        via 10.0.13.3 (5120/2560), Gi0/2        ← RD 2560 < FD 3072，是 FS ✓
```

**Step 2：破坏 FS 条件**
```cisco
R3(config)# interface GigabitEthernet0/2
R3(config-if)# delay 50000                       ! 大幅增加延迟
```
```cisco
R1# show ip eigrp topology 10.1.1.0/24
P 10.1.1.0/24, 1 successors, FD is 3072
        via 10.0.12.2 (3072/2816), Gi0/1
! 只剩一条 → R3 的路径不再是 FS
```

**Step 3：断主链路，观察 Active 状态**
```cisco
R1(config)# interface GigabitEthernet0/1
R1(config-if)# shutdown
```
**立即执行**：
```cisco
R1# show ip eigrp topology active
A 10.1.1.0/24, 0 successors, FD is Inaccessible
  1 replies, active 00:00:03, query-origin: Local origin
        via 10.0.13.3 (Infinity/Infinity), r, Gi0/2
↑                                          ↑
Active                              等待 R3 的 reply
```

**Step 4：模拟邻居不回复（制造真正的 SIA）**
```cisco
! 在 R3 上阻断 EIGRP 报文
R3(config)# ip access-list extended BLOCK-EIGRP
R3(config-ext-nacl)#  deny eigrp any any
R3(config-ext-nacl)#  permit ip any any
R3(config)# interface GigabitEthernet0/2
R3(config-if)# ip access-group BLOCK-EIGRP out
```

**等待 3 分钟**：
```cisco
R1# show logging | include SIA
%DUAL-3-SIA: Route 10.1.1.0/24 stuck-in-active state in IP-EIGRP(0) 100. Cleaning up
%DUAL-5-NBRCHANGE: EIGRP-IPv4 100: Neighbor 10.0.13.3 (Gi0/2) is down: stuck in active
                                                                        ↑ 邻居被强制断开
```

**✅ 你亲手制造了一次 SIA。** 现在你会认得这个症状了。

### 实验 B：用汇总建立查询边界

```cisco
! 在 R2 上向 R1 通告汇总
R2(config)# interface GigabitEthernet0/1
R2(config-if)# ip summary-address eigrp 100 10.1.0.0 255.255.0.0
```

**验证 R1 只看到汇总**：
```cisco
R1# show ip route eigrp
D    10.1.0.0/16 [90/3072] via 10.0.12.2, 00:01:15, GigabitEthernet0/1
! 没有明细了
```

**现在再断 10.1.1.0/24**：
```cisco
R2(config)# interface Loopback1
R2(config-if)# shutdown
```

**观察 R1**：
```cisco
R1# show ip eigrp topology active
! 空输出！★ R1 完全不知道有明细路由消失了 ★
! 因为它只知道汇总 10.1.0.0/16，而汇总依然有效（还有 10.1.2.0/24）
```

**✅ 汇总成功阻断了查询扩散。** 这就是防 SIA 最有效的手段。

### 实验 C：Stub 验证

```cisco
R3(config)# router eigrp 100
R3(config-router)# eigrp stub connected summary
```

```cisco
R1# show ip eigrp neighbors detail | include Stub -A 1
    Stub Peer Advertising ( CONNECTED SUMMARY ) Routes
    Suppressing queries                          ← ★ R1 不会向 R3 发查询
```

**现在再重复实验 A 的 Step 3**：主链路断了，R1 不会向 R3 发查询，**直接标记路由不可达，不会进入长时间的 Active 状态**。

### 实验 D：汇总 AD=5 导致的黑洞

```cisco
! R1 上同时有 EIGRP 汇总和 OSPF 学到的明细
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip summary-address eigrp 100 10.1.0.0 255.255.0.0
```

```cisco
R1# show ip route 10.1.0.0
D    10.1.0.0/16 [5/2816] is a summary, 00:02:15, Null0
                  ↑ AD=5

! 假设 OSPF 也学到了 10.1.0.0/16（AD=110）
! → EIGRP 汇总（AD 5）赢 → 流量走 Null0 被丢弃 ★ 黑洞 ★
```

**验证**：
```cisco
R1# ping 10.1.99.99
Success rate is 0 percent (0/5)

R1# show ip route 10.1.99.99
Routing entry for 10.1.0.0/16
  Known via "eigrp 100", distance 5, metric 2816, type internal
  Routing Descriptor Blocks:
  * directly connected, via Null0                ← 找到黑洞了
```

**修复**：
```cisco
R1(config-if)# ip summary-address eigrp 100 10.1.0.0 255.255.0.0 250
```

---

## ⑧ 排障速查表

| 症状 | 根因 | 验证命令 |
|:--|:--|:--|
| 完全没邻居，无日志 | **AS 号不同** | `show ip protocols \| inc Routing Protocol` |
| 邻居 down，日志 `K-value mismatch` | K 值不同 | `show ip protocols \| inc K1` |
| 邻居 down，日志 `Auth failure` | 认证/密钥/**时间** | `show key chain`、`show clock` |
| 邻居 down，日志 `Interface Goodbye received` | 对端优雅关闭 | 正常，对方在重启进程 |
| 邻居 down，日志 `retry limit exceeded` | 单向链路 / 丢包 | `show interfaces`、`show ip eigrp interfaces` |
| 邻居 down，日志 `stuck in active` | **SIA** | `show ip eigrp topology active` |
| 某接口没有邻居 | passive-interface | `show ip protocols \| section Passive` |
| 学不到某网段 | 对方没 network / 被汇总 / 被过滤 | 对方 `show ip protocols` |
| 收敛慢 | **没有 FS** | `show ip eigrp topology all-links` |
| variance 不生效 | 备份路径不是 FS | 同上 |
| **流量进黑洞** | **汇总路由 AD=5 抢赢了** | `show ip route <目的>` 看是否 Null0 |
| 路由被莫名汇总 | `auto-summary` 开着 | `show ip protocols \| inc summar` |
| CPU 高 | 频繁 Active 查询 | `show ip eigrp topology active`、`show processes cpu` |

### 日志速查

```cisco
R1# show logging | include DUAL
```

| 日志 | 含义 |
|:--|:--|
| `Neighbor ... is up: new adjacency` | 邻居建立 ✓ |
| `Neighbor ... is down: holding time expired` | Hold 超时（链路问题/CPU忙） |
| `Neighbor ... is down: K-value mismatch` | K 值不同 |
| `Neighbor ... is down: Auth failure` | 认证失败 |
| `Neighbor ... is down: retry limit exceeded` | 重传超限（单向链路） |
| `Neighbor ... is down: Interface Goodbye received` | 对端优雅关闭 |
| **`Route ... stuck-in-active`** | **SIA** |
| `DUAL-3-SIA` | SIA 详细信息 |

---

## ⑨ 自测题

**1.** EIGRP 出现 SIA 的根本原因是什么？两个最有效的预防手段？

<details><summary>答案</summary>

**根本原因：查询（QUERY）的扩散范围不可控。**

```
   Successor 失效且没有 FS
        ↓
   路由进入 Active，向所有邻居发 QUERY
        ↓
   邻居不知道 → ★ 继续向它的邻居发 QUERY ★
        ↓
   查询呈扩散式传播，可能涉及几十台设备
        ↓
   任何一台响应慢（CPU 忙、链路单向、软件 bug）
        ↓
   查询者一直等待 → 超过 Active Timer（3 分钟）→ ★ SIA ★
        ↓
   强制断开那个邻居 → 邻居重建 → 可能引发连锁震荡
```

**两个最有效的预防手段**：

**① 路由汇总（★ 最有效）**

**原理**：配了汇总的路由器收到关于**明细路由**的 QUERY 时，会立即回复 REPLY（"我有汇总覆盖它，不用往下问了"），**查询到此为止，不再扩散**。

```cisco
R-Dist(config)# interface GigabitEthernet0/1
R-Dist(config-if)# ip summary-address eigrp 100 10.1.0.0 255.255.0.0
```

**这叫"建立查询边界（Query Boundary）"**。汇总点把网络分割成若干个查询域，任何一个域内的故障，查询都不会扩散到域外。

**② Stub 路由**

**原理**：上游路由器**根本不向 Stub 邻居发送 QUERY**（因为知道它是末梢，不可能有其他路径）。

```cisco
Branch(config-router)# eigrp stub connected summary
```

**验证**：
```cisco
Core# show ip eigrp neighbors detail
    Stub Peer Advertising ( CONNECTED SUMMARY ) Routes
    ★ Suppressing queries ★
```

**⚠️ Stub 的限制：Stub 路由器不能做中转。** 只有真正的末梢设备才能配。中间的汇聚层配了会导致下游变成孤岛。

**其他辅助手段**：
- **优化拓扑**：避免全 mesh（查询扩散呈指数级）
- **调 Active Timer**：`timers active-time 5`（治标，只是给更多时间）
- **SIA-Query/SIA-Reply**：IOS 12.4+ 自动支持，超时前先探测邻居是否还活着，避免误杀

**实践原则**：
> **EIGRP 网络必须做汇总 + Stub，否则规模一大必出 SIA。**
>
> 这是 EIGRP 相比 OSPF 的主要运维负担——OSPF 靠 Area 天然限制了 LSA 泛洪范围，EIGRP 需要工程师主动设计查询边界。

**规划检查清单**：
```
□ 所有末梢/分支路由器配了 eigrp stub
□ 汇聚层向核心通告汇总路由
□ 地址规划保证可汇总
□ 巡检脚本监控 show ip eigrp topology active（正常应无输出）
```
</details>

**2.** EIGRP 汇总路由的 AD 是多少？这可能带来什么问题？

<details><summary>答案</summary>

**EIGRP 汇总路由的 AD 是 5**（不是内部路由的 90，也不是外部路由的 170）。

```cisco
R1# show ip route 10.1.0.0
D    10.1.0.0/16 [5/2816] is a summary, 00:05:23, Null0
                  ↑ AD = 5
```

**为什么设计成 5**：为了让汇总路由**优先于大多数动态路由协议**（OSPF 110、RIP 120、iBGP 200、EIGRP 自身 90/170），确保汇总生效。

**带来的问题：可能形成黑洞**

```
   场景：
   · R1 配了 EIGRP 汇总 10.1.0.0/16，AD=5，指向 Null0
   · R1 同时通过 OSPF 学到 10.1.0.0/16（AD=110）
        ↓
   ★ EIGRP 汇总（AD 5）赢 ★
        ↓
   去往 10.1.x.x 的流量走 Null0 → 全部被丢弃
        ↓
   ★ 黑洞 ★
```

**更隐蔽的情况：过度汇总**

```
   R1 汇总了 10.1.0.0/16，但实际只有 10.1.1.0/24 和 10.1.2.0/24
        ↓
   10.1.3.0/24 实际在别的地方（通过 OSPF 学到）
        ↓
   R1 的汇总路由（Null0，AD 5）压过了 OSPF 学到的明细
        ↓
   ★ 去往 10.1.3.0/24 的流量被 R1 丢进 Null0 ★
```

**诊断**：
```cisco
R1# ping 10.1.3.10
Success rate is 0 percent (0/5)

R1# show ip route 10.1.3.10
Routing entry for 10.1.0.0/16
  Known via "eigrp 100", distance 5, metric 2816, type internal
  Routing Descriptor Blocks:
  * directly connected, via Null0              ← ★ 找到黑洞
```

**看到 `via Null0` 且不是你有意配的，就是这个问题。**

**修复方案**：

**① 调高汇总路由的 AD**
```cisco
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip summary-address eigrp 100 10.1.0.0 255.255.0.0 250
!                                                                 ↑ AD=250，比任何协议都差
```
这样汇总只在没有其他路由时才生效。

**② 精确汇总（不要过度汇总）**
```cisco
! 只汇总自己实际拥有的
R1(config-if)# ip summary-address eigrp 100 10.1.1.0 255.255.254.0    ! 只覆盖 1 和 2
```

**③ 用 leak-map 泄露特殊明细**
```cisco
R1(config)# ip prefix-list LEAK seq 5 permit 10.1.3.0/24
R1(config)# route-map LEAK-MAP permit 10
R1(config-route-map)#  match ip address prefix-list LEAK
R1(config-if)# ip summary-address eigrp 100 10.1.0.0 255.255.0.0 leak-map LEAK-MAP
```

**AD 值速查（EIGRP 相关）**：

| 路由类型 | AD |
|:--|:--|
| **EIGRP 汇总** | **5** |
| EIGRP 内部 | 90 |
| EIGRP 外部（重分发进来的） | 170 |
| 直连 | 0 |
| 静态 | 1 |
| eBGP | 20 |
| OSPF | 110 |
| RIP | 120 |
| iBGP | 200 |

**排障口诀**：**看到路由指向 Null0 且不是有意配的，先查汇总。**
</details>

**3.** EIGRP 建立邻居时，Hello/Hold 时间需要匹配吗？MTU 呢？

<details><summary>答案</summary>

**都不需要！这是 EIGRP 和 OSPF 最容易混淆的差异。**

| 条件 | **EIGRP** | **OSPF** |
|:--|:--|:--|
| **Hello / Hold 时间** | ❌ **不需要匹配** | ✅ **必须匹配** |
| **MTU** | ❌ **不检查** | ✅ **必须匹配**（否则卡 EXSTART） |
| AS 号 / 进程号 | ✅ **必须相同** | ❌ 可以不同 |
| K 值 | ✅ **必须相同** | — |
| 认证 | ✅ 必须匹配 | ✅ 必须匹配 |
| 同一子网 | ✅ 必须 | ✅ 必须 |
| Router ID 唯一 | 不强制 | ✅ 必须 |
| Stub/NSSA 标志 | — | ✅ 必须一致 |

**为什么 EIGRP 不需要匹配 Hello/Hold**：

因为 **EIGRP 把 Hold 时间放在 Hello 报文里告诉对方**：
```
   R1 发 Hello：Hello=5秒, "我的 Hold 是 15 秒"
        ↓
   R2 记录：如果 15 秒没收到 R1 的 Hello，就认为它挂了
   
   R2 发 Hello：Hello=60秒, "我的 Hold 是 180 秒"
        ↓
   R1 记录：如果 180 秒没收到 R2 的 Hello，就认为它挂了
   
   ★ 各自用对方声明的 Hold 时间，互不干扰 ★
```

OSPF 则是**双方必须约定相同的值**（Hello 参数在 Hello 包里，不一致直接丢弃）。

**⚠️ 但配置不当仍会出问题**：

```cisco
R1(config-if)# ip hello-interval eigrp 100 60      ! Hello 60 秒
R1(config-if)# ip hold-time eigrp 100 15           ! ★ Hold 只有 15 秒
```

这个配置会导致：R2 收到 R1 的 Hello 后，等 15 秒；但 R1 要 60 秒才发下一个 Hello → **R2 每次都会超时断开邻居**，然后重建，无限循环。

**规则：Hold ≥ 3 × Hello**（默认就是这个比例）。

**为什么 EIGRP 不检查 MTU**：

EIGRP 用自己的 **RTP（Reliable Transport Protocol）** 传输，有确认和重传机制。如果一个包太大被丢弃，RTP 会重传（并可能分片）。它不像 OSPF 那样在 DD 交换阶段有显式的 MTU 检查。

**但 MTU 不匹配仍可能导致问题**（更隐蔽）：
```
   大的 UPDATE 报文被丢弃
        ↓
   RTP 不断重传
        ↓
   重传次数超限 → ★ 邻居 down: retry limit exceeded ★
```

```cisco
R1# show logging | include DUAL
%DUAL-5-NBRCHANGE: EIGRP-IPv4 100: Neighbor 10.0.12.2 (Gi0/1) is down: retry limit exceeded
```

**看到 `retry limit exceeded`，怀疑三件事**：
1. **单向链路**（一端能发不能收）
2. **MTU 不匹配**导致大包被丢
3. **严重丢包**

```cisco
R1# show ip eigrp interfaces detail Gi0/1
                     Xmit Queue   Mean   Pacing Time   Multicast    Pending
Interface  Peers  Un/Reliable  SRTT   Un/Reliable   Flow Timer   Routes
Gi0/1        1        0/2       1200      0/1          5000         12
                        ↑                                            ↑
                   队列有积压                                   有待发路由
```

**验证 MTU**：
```cisco
R1# ping 10.0.12.2 size 1500 df-bit
```
</details>

**4.** `eigrp stub` 有哪些选项？为什么中间的汇聚路由器不能配 Stub？

<details><summary>答案</summary>

**六个选项**：

```cisco
R1(config-router)# eigrp stub [connected] [summary] [static] [redistributed] [receive-only] [leak-map NAME]
```

| 选项 | 通告什么 | 用途 |
|:--|:--|:--|
| **`connected`** | 直连路由 | ★ 分支的 LAN 网段 |
| **`summary`** | 汇总路由 | ★ 分支做了汇总时 |
| `static` | 静态路由 | 需配合 `redistribute static` |
| `redistributed` | 重分发进来的路由 | 分支连了其他协议 |
| `receive-only` | **什么都不通告，只接收** | 纯接收端（罕见） |
| `leak-map NAME` | 有选择地泄露特定路由 | 精细控制 |

**默认（只写 `eigrp stub`）= `connected summary`**

**分支路由器的标准配置**：
```cisco
Branch(config)# router eigrp 100
Branch(config-router)# eigrp stub connected summary
```

**为什么中间的汇聚路由器不能配 Stub**：

**因为 Stub 路由器不能做"中转"（transit）。**

```
   R-Core ──── R-Dist ──── R-Access ──── LAN
                  ↑
            如果这里配了 stub
                  ↓
   ① R-Core 【不会向 R-Dist 发查询】
   ② R-Dist 【不会通告从 R-Access 学到的路由】给 R-Core
      （因为 stub connected summary 只通告自己的直连和汇总，
        不通告"学到的路由"）
        ↓
   ★ R-Access 后面的所有网段，R-Core 都学不到 ★
   ★ R-Access 变成孤岛 ★
```

**Stub 的语义是**："我是末梢，我后面没有别的路由器，你不用问我别的路径。"

如果 R-Dist 后面还有 R-Access，这个声明就是**假的**，会导致路由丢失。

**判断能否配 Stub 的标准**：

```
   问自己：★ 这台路由器后面还有其他路由器吗？★
        ├─ 没有（只有 LAN 和终端）→ ✅ 可以配 Stub
        └─ 有 → ❌ 不能配
```

**典型的正确部署**：
```
                 [Core]
                    │
              ┌─────┴─────┐
          [Dist-1]     [Dist-2]        ← ❌ 不能配 stub（下面还有设备）
              │             │
        ┌─────┴───┐    ┌────┴────┐
    [Branch-1] [Branch-2] ...          ← ✅ 配 stub（末梢）
        │
      [LAN]
```

**如果 Stub 路由器确实需要中转少量路由，用 `leak-map`**：
```cisco
Branch(config)# ip prefix-list TRANSIT seq 5 permit 10.9.9.0/24
Branch(config)# route-map STUB-LEAK permit 10
Branch(config-route-map)#  match ip address prefix-list TRANSIT

Branch(config)# router eigrp 100
Branch(config-router)# eigrp stub connected summary leak-map STUB-LEAK
```

**验证 Stub 生效**：
```cisco
Core# show ip eigrp neighbors detail
H   Address     Interface  Hold Uptime  SRTT  RTO  Q  Seq
0   10.0.13.3   Gi0/2        12  00:15:23  10  100  0  18
    Version 23.0/2.0, Retrans: 0, Retries: 0, Prefixes: 5
    Topology-ids from peer - 0
    ★ Stub Peer Advertising ( CONNECTED SUMMARY ) Routes ★
    ★ Suppressing queries ★
```

**误配 Stub 的排障**：
```
   症状：某个分支后面的网段，核心学不到了
        ↓
   检查：Core# show ip eigrp neighbors detail | include Stub
        ↓
   如果中间设备显示 "Stub Peer" → 找到问题
        ↓
   修复：中间设备去掉 stub，或改用 leak-map
```
</details>

**5.** EIGRP 的 `retry limit exceeded` 日志说明什么？怎么排查？

<details><summary>答案</summary>

```
%DUAL-5-NBRCHANGE: EIGRP-IPv4 100: Neighbor 10.0.12.2 (Gi0/1) is down: retry limit exceeded
```

**含义**：EIGRP 的 RTP（可靠传输协议）**多次重传某个报文都没收到确认**，判定邻居不可达。

**默认重传 16 次后放弃。**

**三个可能的根因（按概率排序）**：

**① 单向链路（Unidirectional Link）**
```
   R1 能收到 R2 的包，但 R2 收不到 R1 的包
        ↓
   R1 认为邻居正常（一直收到 Hello）
   R1 发的 UPDATE 需要确认，但 R2 收不到 → 不会回 ACK
        ↓
   R1 重传 16 次后 → retry limit exceeded
```

**常见原因**：光纤 TX/RX 有一个方向坏了、光衰过大、接头氧化、单侧 ACL。

**排查**：
```cisco
! 两端都看接口计数器
R1# show interfaces GigabitEthernet0/1 | include packets|errors
     12345 packets input, 1234567 bytes
     98765 packets output, 9876543 bytes           ← 有输出

R2# show interfaces GigabitEthernet0/1 | include packets|errors
     0 packets input, 0 bytes                      ← ★ 一个包都没收到！
     45678 packets output, 4567890 bytes

! 光模块诊断
R1# show interfaces GigabitEthernet0/1 transceiver detail
        Optical   Optical
     Tx Power  Rx Power
       -2.5      -28.5 dBm                         ← 接收光功率过低
```

**解决**：配 **UDLD aggressive**（见 [ENCOR 第 4 章](../02-ENCOR-350-401/04-交换进阶-STP进阶与排障.md)）
```cisco
R1(config)# udld aggressive
```

**② MTU 不匹配**
```
   EIGRP 的 UPDATE 报文可能很大（路由多的时候）
        ↓
   超过对端 MTU → 被丢弃
        ↓
   RTP 重传 → 还是被丢 → retry limit exceeded
```

**特征**：**邻居能建立（Hello 包小），但一交换路由就断**。

**排查**：
```cisco
R1# ping 10.0.12.2 size 1500 df-bit
M.M.M                                              ← 大包不通

R1# show ip eigrp interfaces detail Gi0/1
                     Xmit Queue   Mean   Pacing Time   Multicast    Pending
Interface  Peers  Un/Reliable  SRTT   Un/Reliable   Flow Timer   Routes
Gi0/1        1        0/12      1200      0/1          5000         45
                        ↑↑                                          ↑↑
                  队列严重积压                                  大量待发路由
```

**③ 严重丢包 / CPU 过载**
```cisco
R1# show interfaces GigabitEthernet0/1 | include drops|errors
     Total output drops: 12345                     ← 大量丢包
     
R1# show processes cpu sorted | include CPU
CPU utilization for five seconds: 98%/45%          ← CPU 打满
```

**CPU 高时，EIGRP 进程来不及处理 ACK，也会导致重传超限。**

**完整排查流程**：

```
① 两端接口计数器对比
   R1# show interfaces Gi0/1 | include packets input|packets output
   R2# show interfaces Gi0/1 | include packets input|packets output
   → 一端 output 有数字，另一端 input 是 0 → ★ 单向链路 ★

② MTU 测试
   R1# ping 10.0.12.2 size 1500 df-bit
   → 小包通大包不通 → ★ MTU 问题 ★

③ 光模块诊断（光纤链路）
   R1# show interfaces Gi0/1 transceiver detail
   → Rx Power < -25 dBm 或 N/A → ★ 光路问题 ★

④ CPU 和队列
   R1# show processes cpu sorted
   R1# show ip eigrp interfaces detail Gi0/1
   → Xmit Queue 积压严重 → ★ 拥塞或 CPU 问题 ★

⑤ ACL 检查
   R1# show access-lists | include eigrp
   R2# show access-lists | include eigrp
   → 某一端拦了 EIGRP（协议号 88）
```

**预防措施**：
```cisco
! 光纤链路开 UDLD
R1(config)# udld aggressive

! 限制 EIGRP 占用的带宽（默认 50%，低速链路上可能太多）
R1(config-if)# ip bandwidth-percent eigrp 100 30

! 确保接口带宽配置正确（影响 EIGRP 的发送速率计算）
R1(config-if)# bandwidth 100000
```

> **`bandwidth` 命令的隐藏影响**：EIGRP 用接口的 `bandwidth` 值来计算自己能占用多少带宽（默认 50%）。如果接口实际是 10Mbps 但配了 `bandwidth 1000000`（1Gbps），EIGRP 会以 500Mbps 的速率发送 → **把链路打爆 → 丢包 → retry limit exceeded**。
>
> **确保 `bandwidth` 反映真实带宽**，尤其在子速率接口和隧道上。
</details>

---

**下一章** → [02 OSPF 深入与排障](02-OSPF深入与排障.md)
