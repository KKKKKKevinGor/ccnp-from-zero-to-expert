# Lab 05 · EIGRP DUAL 与 SIA

**对应章节**：[ENCOR 06](../02-ENCOR-350-401/06-EIGRP.md)、[ENARSI 01](../03-ENARSI-300-410/01-EIGRP深入与排障.md)
**难度**：★★★　**时长**：2 小时

## 目标

1. **手算 FD/RD，判断 Successor 和 Feasible Successor**，再用设备验证
2. 对比"有 FS"和"没有 FS"时的收敛速度
3. **亲手制造一次 SIA 故障**
4. 用汇总和 Stub 建立查询边界
5. 验证不等价负载均衡

---

## 拓扑

```
                 ┌──────┐
           4     │  R2  │    4
      ┌──────────┤      ├──────────┐
      │          └──────┘          │
  ┌───┴──┐                    ┌────┴───┐
  │  R1  │                    │   R4   │──── 10.1.1.0/24
  └───┬──┘                    └────┬───┘
      │          ┌──────┐          │
      └──────────┤  R3  ├──────────┘
          19     │      │    4
                 └──────┘
                 
   R1-R2: 1Gbps (delay 10)
   R2-R4: 1Gbps (delay 10)
   R1-R3: 100Mbps (delay 100)
   R3-R4: 1Gbps (delay 10)
```

---

## Part 1：手算 DUAL（★ 先别碰设备）

### 概念回顾

| 术语 | 定义 |
|:--|:--|
| **RD** (Reported Distance) | **邻居告诉我的、它到目标的距离** |
| **FD** (Feasible Distance) | **我到目标的总距离** = 我到邻居 + 邻居到目标 |
| **Successor** | FD 最小的那条路径的下一跳 |
| **FS** (Feasible Successor) | **满足可行性条件的备份路径** |

**★ 可行性条件（FC）**：
```
   邻居的 RD  <  当前的 FD
```

### 练习

**站在 R1 的视角，看去往 `10.1.1.0/24`**：

假设（用简化的示意值）：
- R2 报告它到 10.1.1.0/24 的距离（RD）= **2816**
- R3 报告它到 10.1.1.0/24 的距离（RD）= **3072**
- R1 到 R2 的增量 = **256**
- R1 到 R3 的增量 = **2560**

**填表**：

| 路径 | 邻居的 RD | 增量 | **FD** | 是 Successor？ | 满足 FC？ |
|:--|:--|:--|:--|:--|:--|
| 经 R2 | 2816 | 256 | ? | ? | — |
| 经 R3 | 3072 | 2560 | ? | ? | ? |

<details><summary>答案</summary>

| 路径 | RD | 增量 | **FD** | Successor？ | FC？ |
|:--|:--|:--|:--|:--|:--|
| 经 R2 | 2816 | 256 | **3072** | ✅ **是**（FD 最小） | — |
| 经 R3 | 3072 | 2560 | **5632** | ❌ | ? |

**判断 R3 是否为 FS**：
```
   可行性条件：邻居的 RD < 当前的 FD
   
   R3 的 RD = 3072
   当前 FD  = 3072
   
   ★ 3072 < 3072？ 否！（必须【严格小于】）★
   
   → R3 ★ 不是 ★ Feasible Successor
```

**★ 后果**：
```
   R1-R2 链路断了
        ↓
   Successor 失效，且没有 FS
        ↓
   ★ 路由进入 Active 状态，向所有邻居发 QUERY ★
        ↓
   收敛慢，有 SIA 风险
```

**如果 R3 的 RD 是 2560 呢？**
```
   2560 < 3072  ✅ 满足 FC
   → R3 是 Feasible Successor
   → R1-R2 断了，★ 立即切到 R3，几十毫秒完成 ★
```

**★ 这就是 EIGRP 收敛快慢的分水岭。**
</details>

---

## Part 2：设备验证

### Step 1：基础配置

```cisco
! ══════ 所有路由器 ══════
Rx(config)# router eigrp 100
Rx(config-router)# ★ eigrp router-id X.X.X.X ★
Rx(config-router)# network 10.0.0.0 0.255.255.255
Rx(config-router)# ★ no auto-summary ★                  ! ★★★ 必须
Rx(config-router)# passive-interface default
Rx(config-router)# no passive-interface <互联接口>
```

### Step 2：验证邻居

```cisco
R1# show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(100)
H   Address      Interface  Hold Uptime   SRTT   RTO  Q  Seq
                            (sec)         (ms)       Cnt Num
1   10.0.13.3    Gi0/2        12  00:05:23   45   270  0  8
0   10.0.12.2    Gi0/1        13  00:05:45   38   228  0  7
                              ↑                       ↑
                        Hold 倒计时              ★ Q Cnt 应为 0 ★
```

**★ `Q Cnt` 不为 0 说明有报文积压**——可能是链路问题或对端 CPU 忙。

### Step 3：★ 核心验证 —— 拓扑表

```cisco
R1# show ip eigrp topology 10.1.1.0/24

EIGRP-IPv4 Topology Entry for AS(100)/ID(1.1.1.1) for 10.1.1.0/24
  ★ State is Passive ★, Query origin flag is 1, 1 Successor(s), FD is 3072
        ↑ Passive = 正常！
  Descriptor Blocks:
  10.0.12.2 (GigabitEthernet0/1), from 10.0.12.2, Send flag is 0x0
      ★ Composite metric is (3072/2816) ★, route is Internal
                             ↑    ↑
                            FD    RD
```

**只显示了一条？** 说明另一条不满足 FC，不是 FS。

**看所有路径（包括非 FS）**：
```cisco
R1# show ip eigrp topology ★ all-links ★

P 10.1.1.0/24, 1 successors, FD is 3072, serno 12
        via 10.0.12.2 (3072/2816), Gi0/1              ← Successor
        via 10.0.13.3 (5632/3072), Gi0/2              ← ★ 不是 FS
                              ↑
                    RD=3072 不 < FD=3072
```

**✅ 和你的手算一致。**

---

## Part 3：收敛速度对比

### 实验 A：没有 FS 时的收敛

```cisco
! 在 PC 上持续 ping 10.1.1.1

R1(config)# interface GigabitEthernet0/1
R1(config-if)# shutdown
```

**立即查看（几秒内）**：
```cisco
R1# show ip eigrp topology ★ active ★

EIGRP-IPv4 Topology Table for AS(100)/ID(1.1.1.1)

★ A ★ 10.1.1.0/24, 0 successors, FD is Inaccessible
  1 replies, ★ active 00:00:03 ★, query-origin: Local origin
        via 10.0.13.3 (Infinity/Infinity), ★ r ★, Gi0/2
                                            ↑
                                   等待 R3 的 reply
```

**记录丢包数**：__________

<details><summary>预期结果</summary>

**丢 5-20 个包**（取决于查询往返时间）。

**过程**：
```
   ① Successor 失效
   ② 没有 FS
   ③ ★ 路由进入 Active，发 QUERY ★
   ④ 等待所有邻居的 REPLY
   ⑤ 收齐后重新计算
   ⑥ 回到 Passive
```
</details>

### 实验 B：制造 FS，再测

```cisco
! 恢复链路
R1(config-if)# no shutdown

! ★ 调低 R3 到目标的延迟，让它的 RD 变小 ★
R3(config)# interface GigabitEthernet0/2                ! 朝 R4
R3(config-if)# ★ delay 1 ★
```

**验证 FS 出现了**：
```cisco
R1# show ip eigrp topology 10.1.1.0/24
  State is Passive, 1 Successor(s), FD is 3072
  Descriptor Blocks:
  10.0.12.2 (Gi0/1), ...
      Composite metric is (3072/2816), route is Internal
  ★ 10.0.13.3 (Gi0/2), ... ★
      ★ Composite metric is (5120/2560) ★, route is Internal
                                     ↑
                        ★ RD=2560 < FD=3072 → 是 FS ✓ ★
```

**再次断主链路**：
```cisco
R1(config)# interface GigabitEthernet0/1
R1(config-if)# shutdown
```

**立即查看**：
```cisco
R1# show ip eigrp topology 10.1.1.0/24
  ★ State is Passive ★, 1 Successor(s), FD is 5120
        ↑↑↑↑↑↑↑ ★ 一直是 Passive！没有进入 Active ★
  10.0.13.3 (Gi0/2), ...
      Composite metric is (5120/2560)
      ↑ FS 直接被提升为 Successor
```

**记录丢包数**：__________

<details><summary>预期结果</summary>

**丢 0-1 个包**。

**日志（注意没有 Active/Query 的记录）**：
```
%DUAL-5-NBRCHANGE: EIGRP-IPv4 100: Neighbor 10.0.12.2 (Gi0/1) is down: interface down
! ★ 没有 "stuck in active"，没有查询过程 ★
```

**★ 对比结论**：

| 情况 | 收敛时间 | 过程 |
|:--|:--|:--|
| **有 FS** | **几十毫秒** | 直接提升 FS，无查询 |
| **没有 FS** | 几秒到几十秒 | Active → QUERY → 等 REPLY → 重算 |

**★ 设计原则：尽量让备份路径满足 FC。** 可以通过调整接口 delay 来实现（降低备份路径的 RD）。
</details>

---

## Part 4：★ 制造 SIA 故障

### Step 1：破坏 FS 条件

```cisco
R3(config)# interface GigabitEthernet0/2
R3(config-if)# ★ delay 50000 ★                          ! 大幅增加延迟
```

```cisco
R1# show ip eigrp topology all-links
P 10.1.1.0/24, 1 successors, FD is 3072
        via 10.0.12.2 (3072/2816), Gi0/1
        via 10.0.13.3 (★ 大数值/大数值 ★), Gi0/2         ← 不再是 FS
```

### Step 2：让 R3 无法回复 REPLY

```cisco
! ★ 在 R3 上阻断 EIGRP 报文（协议号 88）★
R3(config)# ip access-list extended BLOCK-EIGRP
R3(config-ext-nacl)#  ★ deny eigrp any any ★
R3(config-ext-nacl)#  permit ip any any
R3(config)# interface GigabitEthernet0/2
R3(config-if)# ip access-group BLOCK-EIGRP out
```

### Step 3：断主链路，触发 SIA

```cisco
R1(config)# interface GigabitEthernet0/1
R1(config-if)# shutdown
```

**持续观察**：
```cisco
! 立即
R1# show ip eigrp topology active
A 10.1.1.0/24, 0 successors, FD is Inaccessible
  1 replies, ★ active 00:00:05 ★, query-origin: Local origin
        via 10.0.13.3 (Infinity/Infinity), r, Gi0/2

! 1.5 分钟后（Active Timer 走到一半）
R1# show ip eigrp topology active
A 10.1.1.0/24, 0 successors, FD is Inaccessible
  1 replies, ★ active 00:01:32 ★, query-origin: Local origin
        via 10.0.13.3 (Infinity/Infinity), r, ★ Q ★, Gi0/2
                                               ↑
                                    ★ 已发送 SIA-Query ★

! 3 分钟后
R1# show logging | include SIA
★ %DUAL-3-SIA: Route 10.1.1.0/24 stuck-in-active state in IP-EIGRP(0) 100. Cleaning up ★
★ %DUAL-5-NBRCHANGE: EIGRP-IPv4 100: Neighbor 10.0.13.3 (Gi0/2) is down: 
   stuck in active ★
                     ↑ ★ 邻居被强制断开 ★
```

**✅ 你亲手制造了一次 SIA。现在你会认得这个症状了。**

**恢复**：
```cisco
R3(config)# interface GigabitEthernet0/2
R3(config-if)# no ip access-group BLOCK-EIGRP out
R3(config-if)# no delay 50000
R1(config)# interface GigabitEthernet0/1
R1(config-if)# no shutdown
```

---

## Part 5：防 SIA —— 汇总与 Stub

### 实验 A：汇总建立查询边界

**准备**：让 R4 有多个明细网段。
```cisco
R4(config)# interface Loopback1
R4(config-if)# ip address 10.1.1.1 255.255.255.0
R4(config)# interface Loopback2
R4(config-if)# ip address 10.1.2.1 255.255.255.0
R4(config)# interface Loopback3
R4(config-if)# ip address 10.1.3.1 255.255.255.0
R4(config)# interface Loopback4
R4(config-if)# ip address 10.1.4.1 255.255.255.0
```

**汇总前**：
```cisco
R1# show ip route eigrp | include 10.1
D    10.1.1.0/24 [90/3072] via 10.0.12.2, ...
D    10.1.2.0/24 [90/3072] via 10.0.12.2, ...
D    10.1.3.0/24 [90/3072] via 10.0.12.2, ...
D    10.1.4.0/24 [90/3072] via 10.0.12.2, ...
```

**在 R2 上配汇总**：
```cisco
R2(config)# interface GigabitEthernet0/1                ! 朝 R1
R2(config-if)# ★ ip summary-address eigrp 100 10.1.0.0 255.255.248.0 ★
```

**汇总后**：
```cisco
R1# show ip route eigrp | include 10.1
★ D    10.1.0.0/21 [90/3072] via 10.0.12.2, ... ★
! 四条变一条 ✓

R2# show ip route | include Null0
★ D    10.1.0.0/21 is a summary, 00:02:15, Null0 ★
! 自动生成的防环路由
```

**★ 验证查询边界（核心实验）**：

```cisco
! 在 R1 上开调试
R1# debug eigrp packets query reply

! 在 R4 上断一个明细网段
R4(config)# interface Loopback1
R4(config-if)# shutdown
```

**观察**：

<details><summary>预期结果</summary>

**汇总前**（如果 R2 没配汇总）：
```
R1# debug eigrp packets query reply
EIGRP: Received QUERY on Gi0/1 nbr 10.0.12.2
EIGRP: Sending REPLY on Gi0/1 nbr 10.0.12.2
★ R1 参与了查询过程 ★
```

**汇总后**：
```
R1# debug eigrp packets query reply
! ★ 完全没有输出 ★
! R1 根本不知道 R4 那边有明细路由消失了
```

**为什么**：
```
   R2 上配了汇总 10.1.0.0/21
        ↓
   R2 收到关于 10.1.1.0/24 的 QUERY
        ↓
   ★ R2 检查：我有 10.1.0.0/21 的汇总，覆盖了它 ★
        ↓
   ★ R2 立即回 REPLY，不再向 R1 扩散查询 ★
        ↓
   ★ 查询边界建立成功 ★
```

**这就是 EIGRP 大规模部署的核心设计原则：用汇总建立查询边界。**
</details>

### 实验 B：Stub 防 SIA

```cisco
! 把 R3 配成 Stub（模拟分支路由器）
R3(config)# router eigrp 100
R3(config-router)# ★ eigrp stub connected summary ★
```

**验证上游不发查询**：
```cisco
R1# show ip eigrp neighbors ★ detail ★
H   Address      Interface  Hold Uptime   SRTT  RTO  Q  Seq
1   10.0.13.3    Gi0/2        12  00:15:23   10  100  0  18
    Version 23.0/2.0, Retrans: 0, Retries: 0, Prefixes: 5
    Topology-ids from peer - 0
    ★ Stub Peer Advertising ( CONNECTED SUMMARY ) Routes ★
    ★ Suppressing queries ★
      ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
      R1 不会向 R3 发送 QUERY
```

**再次触发故障，观察不会 SIA**：
```cisco
R1(config)# interface GigabitEthernet0/1
R1(config-if)# shutdown
```

```cisco
R1# show ip eigrp topology active
! ★ 空输出或很快恢复 Passive ★
! 因为不向 R3 发查询，直接标记路由不可达
```

**★ Stub 的限制验证**：
```cisco
! R3 后面接一台 R5
R5(config)# router eigrp 100
R5(config-router)# network 192.168.5.0 0.0.0.255

R1# show ip route eigrp | include 192.168.5
! ★ 空！★
! 因为 R3 是 Stub，只通告自己的 connected 和 summary，不做中转
```

**★ 结论：只有真正的末梢路由器才能配 Stub。**

**用 leak-map 泄露特定路由**：
```cisco
R3(config)# ip prefix-list LEAK seq 5 permit 192.168.5.0/24
R3(config)# route-map STUB-LEAK permit 10
R3(config-route-map)#  match ip address prefix-list LEAK

R3(config)# router eigrp 100
R3(config-router)# eigrp stub connected summary ★ leak-map STUB-LEAK ★
```

---

## Part 6：不等价负载均衡

### Step 1：确认有 FS

```cisco
R1# show ip eigrp topology 10.1.0.0/21
  State is Passive, 1 Successor(s), FD is 3072
  10.0.12.2 (Gi0/1), Composite metric is (3072/2816)      ← Successor
  10.0.13.3 (Gi0/2), Composite metric is (5120/2560)      ← ★ FS（2560 < 3072）
```

### Step 2：配置 variance

```cisco
R1(config)# router eigrp 100
R1(config-router)# ★ variance 2 ★
R1(config-router)# maximum-paths 4
```

**规则**：允许 `FD ≤ variance × 最优FD` **且满足 FC** 的路径。
```
   最优 FD = 3072
   variance = 2
   → 允许 FD ≤ 6144
   
   备份路径 FD = 5120 ≤ 6144  ✅ 且是 FS
   → 两条都进路由表
```

### Step 3：验证

```cisco
R1# show ip route 10.1.0.0
Routing entry for 10.1.0.0/21
  Known via "eigrp 100", distance 90, metric 3072
  Routing Descriptor Blocks:
  ★ 10.0.12.2 ★, from 10.0.12.2, via GigabitEthernet0/1
      Route metric is 3072, ★ traffic share count is 5 ★
  ★ 10.0.13.3 ★, from 10.0.13.3, via GigabitEthernet0/2
      Route metric is 5120, ★ traffic share count is 3 ★
                                                    ↑
                        ★ 按 metric 反比分配：metric 小的分到更多流量 ★
```

### Step 4：验证"不满足 FC 的路径不能参与"

```cisco
! 把 R3 的 RD 调大，让它不满足 FC
R3(config)# interface GigabitEthernet0/2
R3(config-if)# delay 5000
```

```cisco
R1# show ip route 10.1.0.0
Routing entry for 10.1.0.0/21
  Routing Descriptor Blocks:
  * 10.0.12.2, ...
  ! ★ 只剩一条！即使 FD 在 variance 范围内，不满足 FC 就不能用 ★
```

**★ 这是 EIGRP 不等价负载均衡的关键约束**：只有 FS 才能参与（因为只有 FS 在数学上保证无环）。

---

## Part 7：故障注入

### 故障 A：AS 号不匹配

```cisco
R2(config)# router eigrp 200                            ! 其他是 100
```

<details><summary>症状与排查</summary>

**症状**：邻居完全消失，**且没有任何日志**（EIGRP 直接丢弃 AS 号不匹配的包）。

```cisco
R1# show ip protocols | include Routing Protocol
Routing Protocol is "★ eigrp 100 ★"

R2# show ip protocols | include Routing Protocol
Routing Protocol is "★ eigrp 200 ★"                    ← 找到了
```

**★ 这是最"安静"的故障**——没有日志，只能靠对比配置发现。
</details>

### 故障 B：K 值不匹配

```cisco
R2(config-router)# ★ metric weights 0 1 1 1 0 0 ★       ! 改了 K2
```

<details><summary>症状与排查</summary>

```cisco
R1# show logging | include DUAL
★ %DUAL-5-NBRCHANGE: EIGRP-IPv4 100: Neighbor 10.0.12.2 (Gi0/1) is down: 
   K-value mismatch ★

R1# show ip protocols | include K1
  EIGRP metric weight ★ K1=1, K2=0, K3=1 ★, K4=0, K5=0

R2# show ip protocols | include K1
  EIGRP metric weight K1=1, ★ K2=1 ★, K3=1, K4=0, K5=0    ← 不同
```

**★ 不要随便改 K 值**：把 Load（K2）和 Reliability（K4/K5）加入计算会让 metric 随流量波动，导致**路由持续震荡**。
</details>

### 故障 C：认证不匹配 + 时间问题

```cisco
R1(config)# key chain EIGRP-KEYS
R1(config-keychain)# key 1
R1(config-keychain-key)#  key-string Secret1
R1(config-keychain-key)#  ★ accept-lifetime 00:00:00 Jan 1 2030 infinite ★
R1(config-keychain-key)#  ★ send-lifetime 00:00:00 Jan 1 2030 infinite ★
!                                                   ↑ 未来的时间

R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip authentication mode eigrp 100 md5
R1(config-if)# ip authentication key-chain eigrp 100 EIGRP-KEYS
```

<details><summary>症状与排查</summary>

```cisco
R1# show logging | include DUAL
%DUAL-5-NBRCHANGE: ... is down: ★ Auth failure ★

R1# show key chain
Key-chain EIGRP-KEYS:
    key 1 -- text "Secret1"
        accept lifetime (00:00:00 Jan 1 2030) - (infinite) ★ [not valid now] ★
        send lifetime (00:00:00 Jan 1 2030) - (infinite) ★ [not valid now] ★
                                                          ↑↑↑↑↑↑↑↑↑↑↑↑↑↑
                                              ★ 找到了：密钥还没生效 ★

R1# show clock
*10:23:45.123 CST Sat Aug 30 2026                       ← 当前时间

R1# debug eigrp packets
EIGRP: ignored packet from 10.0.12.2, opcode = 5 (invalid authentication)
```

**★ 排查认证问题时，第一件事看 `show clock` 和 `show key chain`。**

**修复**：
```cisco
R1(config-keychain-key)# accept-lifetime 00:00:00 Jan 1 2026 infinite
R1(config-keychain-key)# send-lifetime 00:00:00 Jan 1 2026 infinite
```

**★ 密钥平滑轮换的正确做法**：
```cisco
key chain EIGRP-KEYS
 key 1
  key-string OldSecret
  accept-lifetime 00:00:00 Jan 1 2026 infinite          ! 一直接受
  send-lifetime 00:00:00 Jan 1 2026 23:59:59 Jun 30 2026  ! 6/30 停止发送
 key 2
  key-string NewSecret
  accept-lifetime 00:00:00 Jun 1 2026 infinite          ! 6/1 开始接受
  send-lifetime 00:00:00 Jul 1 2026 infinite            ! 7/1 开始发送

! ★ 6/1 - 6/30 是重叠期：发旧密钥，同时接受新旧两个 ★
! → 给全网设备留出逐台切换的时间，不中断邻居
```
</details>

### 故障 D：`auto-summary` 开着

```cisco
R2(config-router)# ★ auto-summary ★
```

<details><summary>症状与排查</summary>

```cisco
R1# show ip route eigrp
★ D    10.0.0.0/8 [90/3072] via 10.0.12.2, ... ★
! ★ 所有 10.x 的明细都被汇总成了 A 类主网 ★

R1# show ip protocols | include summar
  ★ Automatic Summarization: enabled ★                  ← 找到了
```

**危害**：
```
   R1 有 10.1.x.x，R2 有 10.2.x.x
        ↓
   双方都汇总成 10.0.0.0/8
        ↓
   ★ 路由环路或黑洞 ★
```

**修复**：
```cisco
R2(config-router)# ★ no auto-summary ★
```

**★ IOS 15.x+ 默认关闭，但老版本默认开启。EIGRP 出现莫名其妙的路由问题，第一件事查这个。**
</details>

### 故障 E：汇总路由的 AD=5 导致黑洞

```cisco
R2(config)# interface GigabitEthernet0/1
R2(config-if)# ip summary-address eigrp 100 10.1.0.0 255.255.0.0
!                                                     ↑ /16，比实际范围大
```

<details><summary>症状与排查</summary>

```cisco
R2# show ip route 10.1.0.0
★ D    10.1.0.0/16 [★5★/2816] is a summary, 00:02:15, Null0 ★
                     ↑ AD = 5！不是 90

R2# ping 10.1.99.99                                    ! 汇总范围内但不存在的地址
Success rate is ★ 0 percent ★ (0/5)

R2# show ip route 10.1.99.99
Routing entry for 10.1.0.0/16
  Known via "eigrp 100", distance ★ 5 ★, metric 2816
  Routing Descriptor Blocks:
  * ★ directly connected, via Null0 ★                  ← 找到黑洞
```

**★ 如果同时有 OSPF 学到 10.1.99.0/24（AD 110）**：
```
   EIGRP 汇总 AD=5  <  OSPF AD=110
        ↓
   ★ EIGRP 汇总赢，流量走 Null0 被丢弃 ★
   ★ 即使有正确的 OSPF 路由也用不上 ★
```

**修复**：
```cisco
R2(config-if)# ip summary-address eigrp 100 10.1.0.0 255.255.0.0 ★ 250 ★
!                                                                  ↑ 调高 AD
```

**★ 排障口诀：看到路由指向 Null0 且不是有意配的，先查汇总。**
</details>

---

## 实验检查清单

```
□ ① 手算 FD/RD，判断 Successor 和 FS，与设备验证一致
□ ② 对比"有 FS"和"没 FS"的收敛速度，记录丢包数
□ ③ ★ 亲手制造一次 SIA，看到 %DUAL-3-SIA 日志
□ ④ 用汇总建立查询边界，验证查询不再扩散
□ ⑤ 配置 Stub，验证 "Suppressing queries"
□ ⑥ 验证 Stub 不能做中转
□ ⑦ 配置 variance 实现不等价负载均衡
□ ⑧ 验证不满足 FC 的路径不能参与负载均衡
□ ⑨ 五个故障都亲手制造并修复
□ ⑩ 写了实验笔记
```

## 关键命令总结

| 命令 | 用途 |
|:--|:--|
| **`show ip eigrp topology`** | ★ 看 Successor 和 FS |
| **`show ip eigrp topology all-links`** | ★ 看所有路径（含非 FS） |
| **`show ip eigrp topology active`** | ★★ 巡检必查，正常应无输出 |
| `show ip eigrp neighbors detail` | 看 Stub 状态和 "Suppressing queries" |
| `show ip protocols` | AS 号、K 值、auto-summary、passive |
| `show key chain` | ★ 认证密钥的有效期 |
| `show logging \| include DUAL` | 邻居变化原因 |

## 核心结论

| 要点 | 说明 |
|:--|:--|
| **可行性条件：邻居 RD < 我的 FD** | 严格小于 |
| **Passive = 正常，Active = 异常** | 反直觉，必须记住 |
| **有 FS → 毫秒级收敛；无 FS → 查询，慢且有 SIA 风险** | 设计时要尽量制造 FS |
| **汇总建立查询边界** | ★ 防 SIA 最有效的手段 |
| **Stub 让上游不发查询** | 分支必配 |
| **只有 FS 能参与不等价负载均衡** | variance 的约束 |
| **汇总路由 AD=5** | 可能压过其他协议造成黑洞 |
| **`no auto-summary` 必配** | 老版本默认开启 |

---

**上一个** ← [Lab 04](lab-04-OSPF多区域.md) ｜ **下一个** → [Lab 06: BGP](lab-06-BGP.md)
