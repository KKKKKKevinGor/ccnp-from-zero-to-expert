# 04 · 交换进阶：STP 进阶与排障

## ① 这章解决什么问题

[Stage 1 第 2 章](../01-CCNA补齐篇/02-STP生成树协议.md) 讲了经典 STP，但它有两个致命问题：

1. **慢**。收敛要 30–50 秒。现代业务（VoIP、视频会议、金融交易）根本忍受不了。
2. **要么浪费带宽，要么消耗 CPU**。单棵树（802.1D）浪费一半链路；每 VLAN 一棵树（PVST+）在 1000 个 VLAN 时把 CPU 跑满。

这一章讲这两个问题的解法：**RSTP（快）+ MST（省）**，以及一套完整的二层排障方法论。

---

## ② RSTP：为什么能做到亚秒级收敛

### 2.1 端口角色的重新设计

**经典 STP 只有三种角色**：Root、Designated、Blocking。

**RSTP (802.1w) 增加了两种"备份"角色**，这是快速收敛的关键：

| 角色 | 说明 | 作用 |
|:--|:--|:--|
| **Root Port** | 去往根桥的最优端口 | 转发 |
| **Designated Port** | 该网段的指定端口 | 转发 |
| **Alternate Port** | **备用的根端口**（收到来自其他交换机的更优 BPDU） | **根端口失效时立即接替，无需等待** |
| **Backup Port** | **备用的指定端口**（收到来自自己的 BPDU，同一网段） | 罕见，只在共享介质（HUB）上出现 |
| **Disabled** | 未启用 | — |

**关键：Alternate Port 是"预计算好的备份路径"。**

```
经典 STP：根端口失效 → 重新计算 → Listening(15s) → Learning(15s) → Forwarding
                                    总共 30 秒

RSTP：   根端口失效 → Alternate Port 立即变成 Root Port 并转发
                      < 1 秒（甚至毫秒级）
```

这就像**热备份**和**冷启动**的区别。

### 2.2 端口状态简化

| 经典 STP (802.1D) | RSTP (802.1w) | 说明 |
|:--|:--|:--|
| Disabled | **Discarding** | 三个不转发的状态合并成一个 |
| Blocking | **Discarding** | |
| Listening | **Discarding** | |
| Learning | **Learning** | 学 MAC 但不转发 |
| Forwarding | **Forwarding** | 正常转发 |

**从 5 个状态简化到 3 个**，因为 Listening 状态在 RSTP 里没有存在的必要了（有了 Proposal/Agreement 握手机制）。

### 2.3 三个加速机制

#### ① 边缘端口（Edge Port）

相当于 PVST 的 PortFast，接终端的端口**直接进入 Forwarding**。

```cisco
SW1(config-if)# spanning-tree portfast          ! 在 RSTP 模式下就是 Edge Port
```

#### ② 链路类型（Link Type）

| 类型 | 判断依据 | 能否快速收敛 |
|:--|:--|:--|
| **Point-to-point (P2P)** | **全双工** | ✅ 可以用 Proposal/Agreement |
| **Shared** | 半双工 | ❌ 必须走传统的定时器 |

**这就是为什么"全双工"对 RSTP 至关重要**——只有 P2P 链路才能用快速握手。

```cisco
! 手工指定（通常自动检测就够了）
SW1(config-if)# spanning-tree link-type point-to-point
```

> **实战陷阱**：如果某条链路因为双工协商问题变成了半双工，RSTP 会把它当成 Shared 链路，**收敛速度退回到 30 秒**。而且这个问题很隐蔽——链路是通的，只是慢。
>
> 排查：`show spanning-tree interface Gi0/1 detail | include Link type`

#### ③ Proposal / Agreement 握手（RSTP 的核心）

这是 RSTP 最精妙的设计：**用一次握手代替定时器等待**。

```
   [SW1 上游]                          [SW2 下游]
        │                                   │
        │  ① Proposal（我想让这个口转发）      │
        │──────────────────────────────────>│
        │                                   │
        │                          ② SW2 执行 "Sync"：
        │                             把自己所有非边缘的
        │                             指定端口置为 Discarding
        │                             （防止环路）
        │                                   │
        │  ③ Agreement（同意，你可以转发了）   │
        │<──────────────────────────────────│
        │                                   │
        │  ④ SW1 的端口立即 Forwarding        │
        │                                   │
        │                          ⑤ SW2 对它的下游
        │                             重复这个过程
        │                             （逐级向下传播）
```

**为什么这样是安全的**：SW2 在同意之前，**先把自己所有可能形成环路的端口阻塞掉**（Sync 操作）。所以即使 SW1 立即转发，也不会成环。然后再逐级向下重复，像多米诺骨牌一样快速传播。

**这就是 RSTP 能亚秒级收敛的原理**——它不是"等待足够长的时间确保安全"，而是"用显式握手确认安全"。

### 2.4 拓扑变化处理的改进

| | 经典 STP | RSTP |
|:--|:--|:--|
| 谁能发 TCN | 只有非根桥，且要逐级上报到根桥 | **任何交换机检测到变化都能直接泛洪 TC** |
| 传播方式 | 上报到根桥 → 根桥再往下通知 | **直接向全网泛洪** |
| MAC 表处理 | 老化时间从 300 秒缩短到 15 秒（**渐进清除**） | **立即清除**相关端口的 MAC 表项 |
| 触发条件 | 任何端口状态变化（包括终端插拔） | **只有非边缘端口转为 Forwarding** 才触发 |

**最后一条很重要**：经典 STP 里，用户插拔网线会触发 TCN，导致全网 MAC 表老化加速——这是很多网络"莫名其妙泛洪"的原因。RSTP 中边缘端口的变化**不触发 TC**，大幅减少了不必要的震荡。

---

## ③ MST：为什么需要，怎么配

### 3.1 三种 STP 模式的取舍

| | 802.1D (STP) | **PVST+ / Rapid-PVST+** | **MST (802.1s)** |
|:--|:--|:--|:--|
| 实例数 | **1 棵树** | **每 VLAN 一棵树** | **可配置（通常 2-4 棵）** |
| 负载分担 | ❌ 无 | ✅ 有 | ✅ 有 |
| CPU/内存消耗 | 最低 | **最高**（1000 VLAN = 1000 实例） | **低** |
| 标准 | IEEE | **Cisco 私有** | **IEEE 标准** |
| 跨厂商 | ✅ | ❌ | ✅ |
| Cisco 默认 | — | ✅ Rapid-PVST+ | — |
| H3C/华为默认 | — | — | ✅ MSTP |

**MST 的核心思想：把多个 VLAN 映射到少数几个实例。**

```
PVST+（1000 个 VLAN）：            MST（1000 个 VLAN）：
                                   
VLAN 1   → 实例 1                  VLAN 1-500   ─┐
VLAN 2   → 实例 2                                ├→ 实例 1（根桥在 SW1）
VLAN 3   → 实例 3                  VLAN 501-1000─┐
...                                              ├→ 实例 2（根桥在 SW2）
VLAN 1000→ 实例 1000                            ┘
                                   
1000 个 STP 实例                    只有 2 个 STP 实例
CPU 爆炸                            CPU 轻松
                                   仍然实现了负载分担 ✓
```

### 3.2 MST 的核心概念

#### Region（域）

**同一个 MST Region 内的交换机，必须有完全相同的三个参数**：

| 参数 | 说明 |
|:--|:--|
| **① Region Name** | 域名，字符串 |
| **② Revision Number** | 修订号，整数 |
| **③ VLAN → Instance 映射表** | 哪些 VLAN 属于哪个实例 |

**如果三者不完全一致，交换机会认为对方在不同的 Region**，此时它们之间只运行 **CST（Common Spanning Tree）**，整个远端 Region 被当作**一个虚拟的网桥**——负载分担失效，行为退化。

> **这是 MST 部署最常见的失败原因**。而且症状很迷惑：STP 能收敛，网络也通，但负载分担没生效，或者某些 VLAN 走了奇怪的路径。

**验证方法（关键）**：
```cisco
SW1# show spanning-tree mst configuration digest
Name      [REGION-A]
Revision  1     Instances configured 3
Digest    0x8B4CDF1A7D1C4E9F2A3B5C6D7E8F9A0B
Pre-std Digest 0x...
          ↑ ★ 同一 Region 内所有设备的 Digest 必须完全相同
```

**Digest 是三个参数的哈希值**。**对比两台设备的 Digest，一眼就知道配置是否一致**——比逐条对比 VLAN 映射快得多。

#### IST（Instance 0）

**MST 实例 0 是特殊的**，叫 **IST（Internal Spanning Tree）**：
- 它**自动包含所有没有被显式映射到其他实例的 VLAN**
- 它负责与 Region 外部（其他 Region 或 PVST 网络）交互
- **不能删除，不能改映射**（只能通过把 VLAN 映射到其他实例来"移出"）

> **考点**：如果你只配了 `instance 1 vlan 10,20`，那么**其余所有 VLAN（1-9, 11-19, 21-4094）都在实例 0** 里。

#### CST / CIST

| 术语 | 全称 | 含义 |
|:--|:--|:--|
| **IST** | Internal Spanning Tree | Region **内部**的实例 0 |
| **CST** | Common Spanning Tree | Region **之间**的生成树（把每个 Region 看成一个网桥） |
| **CIST** | Common and Internal Spanning Tree | IST + CST 的总和，整个网络的完整视图 |

### 3.3 MST 配置

```cisco
! ── ① 切换到 MST 模式 ──
SW1(config)# spanning-tree mode mst

! ── ② 配置 Region（三个参数必须全网一致）──
SW1(config)# spanning-tree mst configuration
SW1(config-mst)#  name REGION-A                    ! ① 域名
SW1(config-mst)#  revision 1                       ! ② 修订号
SW1(config-mst)#  instance 1 vlan 10,20,30         ! ③ 映射表
SW1(config-mst)#  instance 2 vlan 40,50,60
SW1(config-mst)#  show pending                     ! ★ 提交前预览
SW1(config-mst)#  exit                             ! 退出即生效

! ── ③ 指定各实例的根桥（实现负载分担）──
SW1(config)# spanning-tree mst 1 root primary      ! SW1 是实例1的根
SW1(config)# spanning-tree mst 2 root secondary
SW2(config)# spanning-tree mst 2 root primary      ! SW2 是实例2的根
SW2(config)# spanning-tree mst 1 root secondary

! 或手工设优先级
SW1(config)# spanning-tree mst 1 priority 4096
SW1(config)# spanning-tree mst 2 priority 8192

! ── ④ 调整开销和端口优先级 ──
SW1(config-if)# spanning-tree mst 1 cost 100
SW1(config-if)# spanning-tree mst 1 port-priority 64

! ── ⑤ 保护机制（同 PVST）──
SW1(config)# spanning-tree portfast default
SW1(config)# spanning-tree portfast bpduguard default
SW1(config)# spanning-tree loopguard default
SW1(config-if)# spanning-tree guard root

! ── 查看 ──
SW1# show spanning-tree mst
SW1# show spanning-tree mst 1
SW1# show spanning-tree mst configuration
SW1# show spanning-tree mst configuration digest     ! ★ 对比一致性
SW1# show spanning-tree mst interface Gi0/1
SW1# show spanning-tree summary
```

> ⚠️ **`spanning-tree mst configuration` 的配置是"暂存"的**，退出配置子模式（`exit`）时才一次性生效。所以修改映射表时**不会产生中间态**——这是个很好的设计。
>
> 用 `show pending` 可以在生效前预览将要提交的配置。

### 3.4 混合厂商 MST 对接

**Cisco 和 H3C/华为对接必须用 MST**（因为 PVST+ 是 Cisco 私有的）。

```cisco
! ── Cisco ──
SW-Cisco(config)# spanning-tree mode mst
SW-Cisco(config)# spanning-tree mst configuration
SW-Cisco(config-mst)#  name REGION-A
SW-Cisco(config-mst)#  revision 1
SW-Cisco(config-mst)#  instance 1 vlan 10,20
SW-Cisco(config-mst)#  instance 2 vlan 30,40
SW-Cisco(config-mst)#  exit
```

```
# ── H3C ──
[SW-H3C] stp mode mstp
[SW-H3C] stp region-configuration
[SW-H3C-mst-region] region-name REGION-A
[SW-H3C-mst-region] revision-level 1
[SW-H3C-mst-region] instance 1 vlan 10 20
[SW-H3C-mst-region] instance 2 vlan 30 40
[SW-H3C-mst-region] active region-configuration        ← ★★★ 必须敲这句！
```

```
# ── 华为 ──
[SW-HW] stp mode mstp
[SW-HW] stp region-configuration
[SW-HW-mst-region] region-name REGION-A
[SW-HW-mst-region] revision-level 1
[SW-HW-mst-region] instance 1 vlan 10 20
[SW-HW-mst-region] instance 2 vlan 30 40
[SW-HW-mst-region] active region-configuration         ← ★★★ 同样必须
```

> ⚠️ **H3C/华为的 `active region-configuration` 是国内混合组网的头号坑。**
>
> 不敲这句，配置写进去了但**不生效**，`display stp region-configuration` 看到的还是旧的。表现为"两边配置看起来一模一样，但 Region 就是不匹配"。
>
> **验证**：
> ```
> [SW-H3C] display stp region-configuration
> Oper Configuration                         ← 看 "Oper"（生效的），不是配置的
>    Format selector      :0
>    Region name          :REGION-A
>    Revision level       :1
>    Instance   VLANs Mapped
>    0          1 to 9, 21 to 29, 41 to 4094
>    1          10, 20
>    2          30, 40
> ```

---

## ④ STP 保护机制全家桶

| 机制 | 配在哪 | 防什么 | 触发后状态 | 恢复方式 |
|:--|:--|:--|:--|:--|
| **PortFast / Edge** | 接终端口 | 收敛慢 | — | — |
| **BPDU Guard** | 接终端口 | **私接交换机** | err-disable | shut/no shut 或自动 |
| **BPDU Filter** | 接终端口 | 不发 BPDU | ⚠️ 慎用 | — |
| **Root Guard** | **朝向下游**的口 | **根桥被抢** | root-inconsistent | 停止收到更优 BPDU 后自动恢复 |
| **Loop Guard** | **根端口 / 阻塞口** | **单向链路成环** | loop-inconsistent | 恢复收到 BPDU 后自动 |
| **UDLD** | 光纤链路两端 | **单向链路** | err-disable (aggressive) | 手工或自动 |
| **BPDU Skew Detection** | 全局 | BPDU 延迟 | 仅告警 | — |

### 4.1 Root Guard vs Loop Guard（最容易混）

| | **Root Guard** | **Loop Guard** |
|:--|:--|:--|
| 配在哪 | **朝向下游/接入层**的端口 | **根端口和阻塞端口** |
| 检测什么 | 收到**更优的 BPDU** | **突然收不到 BPDU** |
| 防什么 | 有人接了优先级更高的交换机，**抢走根桥** | **单向链路**导致阻塞口误转为转发 |
| 触发状态 | `root-inconsistent`（阻塞） | `loop-inconsistent`（阻塞） |
| 恢复 | 不再收到更优 BPDU 后**自动恢复** | 重新收到 BPDU 后**自动恢复** |

**记忆**：
- **Root Guard 防"外来的更好"**（不许你当根桥）
- **Loop Guard 防"该来的没来"**（收不到 BPDU 时宁可阻塞也不冒险转发）

```cisco
! Root Guard：配在汇聚交换机朝向接入交换机的端口
Dist-SW(config)# interface range GigabitEthernet1/0/1-24
Dist-SW(config-if-range)# spanning-tree guard root

! Loop Guard：全局启用（会自动作用于根端口和阻塞端口）
SW1(config)# spanning-tree loopguard default

! 查看触发情况
SW1# show spanning-tree inconsistentports
Name                 Interface              Inconsistency
-------------------- ---------------------- ------------------
VLAN0010             GigabitEthernet1/0/5   Root Inconsistent
VLAN0020             GigabitEthernet1/0/23  Loop Inconsistent
```

### 4.2 UDLD（单向链路检测）

**问题场景**：光纤链路的 TX 或 RX 单方向断了（光模块半坏、光纤跳线插错、光衰过大）。

**后果**：
- 一端能发不能收，另一端能收不能发
- **物理接口依然显示 up**（因为收到光信号）
- STP 的阻塞端口收不到 BPDU → 认为上游没了 → **转为转发 → 成环**

**UDLD 通过双向的 hello 报文检测这种情况**：

```cisco
! 全局启用（只对光纤口生效）
SW1(config)# udld enable                  ! Normal 模式：只告警
SW1(config)# udld aggressive              ! Aggressive 模式：err-disable 端口

! 接口级
SW1(config-if)# udld port aggressive

! 查看
SW1# show udld
SW1# show udld GigabitEthernet1/0/25
SW1# udld reset                           ! 重置被 UDLD 关闭的端口
```

| 模式 | 检测到单向链路后 |
|:--|:--|
| **Normal** | 只标记为 undetermined，**发 Syslog 告警，不关端口** |
| **Aggressive** | 尝试 8 次重建失败后，**err-disable 该端口** |

> **推荐 Aggressive 模式 + Loop Guard 组合**：
> - UDLD Aggressive 在链路层面快速关掉有问题的端口
> - Loop Guard 在 STP 层面兜底
>
> 两者互补，不是二选一。UDLD 检测的是"链路本身单向"，Loop Guard 检测的是"BPDU 没收到"（可能是链路问题，也可能是对端 CPU 忙、软件 bug）。

### 4.3 BPDU Filter 的危险（必须理解）

```cisco
! 全局配置（相对安全）
SW1(config)# spanning-tree portfast bpdufilter default
! 行为：PortFast 口不主动发 BPDU，但如果收到 BPDU，会退出 PortFast 转为普通端口

! 接口配置（★ 危险）
SW1(config-if)# spanning-tree bpdufilter enable
! 行为：完全不收不发 BPDU —— 相当于把这个口从 STP 里彻底摘除！
```

**接口级 BPDU Filter 等于关闭了这个端口的环路保护。** 如果有人在这个端口后面接了交换机形成环路，**没有任何机制能发现**。

> **实践建议：不要在接口下配 BPDU Filter。** 需要抑制 BPDU 时用全局配置（它保留了"收到 BPDU 就退出 PortFast"的保护）。

---

## ⑤ 配套实验：MST 负载分担 + 保护机制

**拓扑**：
```
              [SW1]              [SW2]
                ║  ╲            ╱  ║
                ║   ╲          ╱   ║
                ║    ╲        ╱    ║
                ║     ╲      ╱     ║
                ║      ╲    ╱      ║
              ┌─╨───────╲──╱───────╨─┐
              │          ╳           │
              └─┬────────╱──╲────────┬┘
                │       ╱    ╲       │
              [SW3]                [SW4]
   （接入层）
```
VLAN 10,20 → 实例 1；VLAN 30,40 → 实例 2

### Step 1：全部切到 MST

```cisco
! 四台都执行
SWx(config)# spanning-tree mode mst
SWx(config)# spanning-tree mst configuration
SWx(config-mst)#  name CAMPUS
SWx(config-mst)#  revision 1
SWx(config-mst)#  instance 1 vlan 10,20
SWx(config-mst)#  instance 2 vlan 30,40
SWx(config-mst)#  exit
```

### Step 2：验证 Region 一致性（★ 关键步骤）

```cisco
SW1# show spanning-tree mst configuration digest
Name      [CAMPUS]
Revision  1     Instances configured 3
Digest    0x5B4D2E1F8A9C3D7E6F5A4B3C2D1E0F9A

SW2# show spanning-tree mst configuration digest
Digest    0x5B4D2E1F8A9C3D7E6F5A4B3C2D1E0F9A     ← 必须完全相同
```

**如果不同**，说明三个参数有差异。逐个对比：
```cisco
SW1# show spanning-tree mst configuration
Name      [CAMPUS]
Revision  1     Instances configured 3

Instance  Vlans mapped
--------  ---------------------------------------------------------
0         1-9,11-19,21-29,31-39,41-4094        ← 注意实例 0 包含了所有未映射的
1         10,20
2         30,40
```

### Step 3：配置负载分担

```cisco
! SW1 是实例 1 的根，实例 2 的备根
SW1(config)# spanning-tree mst 1 root primary
SW1(config)# spanning-tree mst 2 root secondary

! SW2 相反
SW2(config)# spanning-tree mst 2 root primary
SW2(config)# spanning-tree mst 1 root secondary
```

### Step 4：验证负载分担生效

```cisco
SW3# show spanning-tree mst 1

##### MST1    vlans mapped:   10,20
Bridge        address 0000.0000.3333  priority  32769 (32768 sysid 1)
Root          address 0000.0000.1111  priority  4097  (4096 sysid 1)
              port    Gi0/1            cost      20000

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------
Gi0/1            Root FWD 20000     128.1    P2p        ← 朝 SW1 转发
Gi0/2            Altn BLK 20000     128.2    P2p        ← 朝 SW2 阻塞
```

```cisco
SW3# show spanning-tree mst 2

##### MST2    vlans mapped:   30,40
Root          address 0000.0000.2222  priority  4098

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------
Gi0/1            Altn BLK 20000     128.1    P2p        ← 朝 SW1 阻塞
Gi0/2            Root FWD 20000     128.2    P2p        ← 朝 SW2 转发
```

**✅ 负载分担成功**：
- VLAN 10,20 的流量走 SW1
- VLAN 30,40 的流量走 SW2
- **两条上行链路都在使用**

### Step 5：测收敛速度

```cisco
! 在 PC 上持续 ping：ping <对端> -t
! 断开主链路
SW3(config)# interface GigabitEthernet0/1
SW3(config-if)# shutdown
```

**预期**：丢 **0–2 个包**（< 1 秒）。

**对比实验**：切回经典 PVST 模式再测：
```cisco
SWx(config)# spanning-tree mode pvst
```
**预期**：丢 **30–50 个包**（30–50 秒）。

**这个对比会让你切身体会 RSTP/MST 的价值。**

### Step 6：故障注入

**故障 A：Region 不匹配**
```cisco
SW4(config)# spanning-tree mst configuration
SW4(config-mst)#  revision 2                     ! 改成 2，其他是 1
SW4(config-mst)#  exit
```

**观察**：
```cisco
SW3# show spanning-tree mst 1
Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------
Gi0/3            Desg FWD 20000     128.3    P2p Bound(RSTP)
                                                  ↑↑↑↑↑↑↑↑↑↑↑
                                    "Bound" = Region 边界！
```

**`Bound(RSTP)` 或 `Boundary` 说明这个端口连接的是不同 Region**（或非 MST 设备），此时该端口只跑 CST，**负载分担在这个方向失效**。

**排查**：
```cisco
SW3# show spanning-tree mst configuration digest
SW4# show spanning-tree mst configuration digest
! 对比 Digest，不同就是配置有差异
```

**故障 B：BPDU Guard 触发**
```cisco
SW3(config)# interface GigabitEthernet0/10
SW3(config-if)# spanning-tree portfast
SW3(config-if)# spanning-tree bpduguard enable
! 在这个口接一台交换机
```

**观察**：
```
%SPANTREE-2-BLOCK_BPDUGUARD: Received BPDU on port Gi0/10 with BPDU Guard enabled. Disabling port.
%PM-4-ERR_DISABLE: bpduguard error detected on Gi0/10, putting Gi0/10 in err-disable state
```
```cisco
SW3# show interfaces status err-disabled
Port      Name    Status              Reason               Err-disabled Vlans
Gi0/10            err-disabled        bpduguard
```

**恢复**：
```cisco
SW3(config)# errdisable recovery cause bpduguard
SW3(config)# errdisable recovery interval 300
SW3# show errdisable recovery
```

**故障 C：Root Guard 触发**
```cisco
SW1(config)# interface GigabitEthernet0/3      ! 朝向 SW3
SW1(config-if)# spanning-tree guard root

! 在 SW3 上把优先级调到最低（想抢根桥）
SW3(config)# spanning-tree mst 1 priority 0
```

**观察**：
```
%SPANTREE-2-ROOTGUARD_BLOCK: Root guard blocking port GigabitEthernet0/3 on MST1.
```
```cisco
SW1# show spanning-tree inconsistentports
Name                 Interface              Inconsistency
-------------------- ---------------------- ------------------
MST1                 GigabitEthernet0/3     Root Inconsistent
```

**恢复**：把 SW3 的优先级改回去，端口会**自动恢复**（不需要人工干预）。

---

## ⑥ 二层排障方法论

### 6.1 环路的识别与应急

**症状识别**：

| 现象 | 命令 |
|:--|:--|
| CPU 接近 100% | `show processes cpu sorted` |
| 端口流量异常高 | `show interfaces \| include rate` |
| MAC 表项在端口间跳变 | 反复 `show mac address-table address <MAC>` |
| 大量广播/组播 | `show interfaces counters` |
| Console 都登不上 | 说明 CPU 已经被打爆 |

**应急处理流程**：

```
① 止血（优先级最高）
   - 找到可疑端口，立即 shutdown
   - 判断依据：show interfaces 里流量异常高的端口
   - 或者：show mac address-table 里 MAC 震荡涉及的端口
   
② 保命
   - 如果 CPU 高到无法操作，从 Console 或带外管理接入
   - 必要时先关掉整个可疑区域的上联口，保住核心
   
③ 定位
   SW# show spanning-tree summary          ← 应该有阻塞端口，没有就异常
   SW# show spanning-tree inconsistentports
   SW# show interfaces status err-disabled
   SW# show cdp neighbors                  ← 看可疑端口对面是什么
   SW# show logging | include SPANTREE
   
④ 根因
   - 私接交换机 → BPDU Guard 没配或没生效
   - 光纤单向故障 → UDLD/Loop Guard
   - STP 被误关 → show run | include no spanning-tree
   - 混合厂商模式不兼容 → 统一 MST
   - 端口被配了 BPDU Filter → show run interface
   
⑤ 加固
   - 所有接入口：portfast + bpduguard
   - 全局：loopguard default、udld aggressive
   - 汇聚朝下游：root guard
   - 手工指定根桥和备根
   - storm-control 兜底
```

### 6.2 快速定位根桥

```cisco
! 从任意一台开始
SW-X# show spanning-tree mst 1 | include Root -A 3
Root          address 0000.0000.1111  priority  4097
              port    Gi0/1            cost      20000
                      ↑ 根端口，往这个方向走

SW-X# show cdp neighbors GigabitEthernet0/1
! 找到对端设备，登上去重复，直到看到 "This bridge is the root"
```

### 6.3 storm-control 兜底

即使 STP 配置正确，也应该配风暴抑制作为最后一道防线：

```cisco
SW1(config)# interface range GigabitEthernet1/0/1-24
SW1(config-if-range)# storm-control broadcast level 5.00
SW1(config-if-range)# storm-control multicast level 5.00
SW1(config-if-range)# storm-control unicast level pps 10k      ! 未知单播
SW1(config-if-range)# storm-control action trap                ! 告警（不关端口）
! 或
SW1(config-if-range)# storm-control action shutdown            ! 更激进

SW1# show storm-control
SW1# show storm-control broadcast
```

> **level 值怎么定**：先用 `show interfaces counters` 观察正常业务的广播比例，通常 < 1%。设成 5% 留足余量。设得太低会误伤（比如 DHCP 高峰期、ARP 广播）。

---

## ⑦ 三厂商对照 + 考点自测

### 三厂商对照

| 目的 | Cisco | H3C | 华为 |
|:--|:--|:--|:--|
| MST 模式 | `spanning-tree mode mst` | `stp mode mstp` | `stp mode mstp` |
| 进 Region 配置 | `spanning-tree mst configuration` | `stp region-configuration` | `stp region-configuration` |
| 域名 | `name REGION-A` | `region-name REGION-A` | `region-name REGION-A` |
| 修订号 | `revision 1` | `revision-level 1` | `revision-level 1` |
| 实例映射 | `instance 1 vlan 10,20` | `instance 1 vlan 10 20` | `instance 1 vlan 10 20` |
| **激活配置** | 退出即生效 | **`active region-configuration`** ★ | **`active region-configuration`** ★ |
| 设根桥 | `spanning-tree mst 1 root primary` | `stp instance 1 root primary` | `stp instance 1 root primary` |
| 边缘端口 | `spanning-tree portfast` | `stp edged-port` | `stp edged-port enable` |
| BPDU 保护 | `spanning-tree bpduguard enable` | `stp bpdu-protection` | `stp bpdu-protection` |
| 根保护 | `spanning-tree guard root` | `stp root-protection` | `stp root-protection` |
| 环路保护 | `spanning-tree guard loop` | `stp loop-protection` | `stp loop-protection` |
| 查看 | `show spanning-tree mst` | `display stp instance 1 brief` | `display stp instance 1 brief` |
| 查 Region | `show spanning-tree mst configuration` | `display stp region-configuration` | `display stp region-configuration` |

### 考点

- **RSTP 的 Alternate/Backup 端口**，以及为什么能快速收敛。
- **Proposal/Agreement 握手机制**。
- **RSTP 需要全双工（P2P 链路）才能快速收敛**。
- **MST Region 的三要素**：name、revision、VLAN 映射。
- **Digest 用于验证 Region 一致性**。
- **实例 0（IST）包含所有未映射的 VLAN**。
- **Root Guard vs Loop Guard** 的配置位置和作用。
- **UDLD 的两种模式**。
- **接口级 BPDU Filter 的危险性**。

### 自测题

**1.** RSTP 为什么能做到亚秒级收敛，而经典 STP 需要 30-50 秒？

<details><summary>答案</summary>

**三个原因**：

**① Alternate Port 是"预计算好的热备份"**

经典 STP 只有 Blocking 状态，没有"备份根端口"的概念。根端口失效时必须**重新计算整棵树**，然后走 Listening(15s) → Learning(15s)。

RSTP 的 **Alternate Port 已经知道自己是"次优的通往根桥的路径"**。根端口一失效，它**立即**转为 Root Port 并开始转发——不需要重新计算，不需要等待定时器。

这是"热备份"和"冷启动"的区别。

**② Proposal/Agreement 握手代替定时器**

经典 STP 的 15 秒 Forward Delay 本质上是**"等待足够长的时间，确保拓扑信息已经传遍全网"**——这是一种保守的、基于时间的安全保证。

RSTP 用**显式握手**：
```
上游发 Proposal → 下游先把自己所有非边缘的指定端口阻塞（Sync）→ 回 Agreement → 上游立即转发
```
下游在同意之前**主动阻塞了所有可能成环的端口**，所以立即转发是安全的。然后这个过程**逐级向下传播**，像多米诺骨牌一样快速。

**用确认代替等待**，这是 RSTP 最核心的设计思想。

**③ 拓扑变化处理更快**

| | 经典 STP | RSTP |
|:--|:--|:--|
| TCN 传播 | 逐级上报到根桥，根桥再向下通知 | **任何交换机直接向全网泛洪 TC** |
| MAC 表 | 老化时间缩短到 15 秒，渐进清除 | **立即清除**相关表项 |
| 触发条件 | 任何端口变化（包括终端插拔） | **只有非边缘端口转 Forwarding** |

**前提条件（容易被忽略）**：

RSTP 的快速收敛依赖 **P2P 链路（全双工）**。如果链路因为双工协商问题变成半双工，RSTP 会把它当作 **Shared 链路**，**退回到传统的定时器机制，收敛速度回到 30 秒**。

```cisco
SW1# show spanning-tree interface Gi0/1 detail | include Link type
   Link type is point-to-point by default        ← 应该是这个
   Link type is shared by default                ← 有问题！查双工
```

**这是个很隐蔽的性能问题**：链路是通的，STP 也在跑，只是收敛慢了 30 倍。
</details>

**2.** MST 中，哪三个参数必须在同一个 Region 内完全一致？不一致会怎样？

<details><summary>答案</summary>

**三个参数**：
1. **Region Name**（域名）
2. **Revision Number**（修订号）
3. **VLAN → Instance 映射表**

**不一致的后果**：

交换机会认为对方**属于不同的 MST Region**。此时：
- 边界端口只运行 **CST（Common Spanning Tree）**
- **整个远端 Region 被当作一个单一的虚拟网桥**
- **负载分担在跨 Region 方向完全失效**
- 可能出现次优路径

**症状很迷惑**：网络是通的，STP 也收敛了，但：
- 某些 VLAN 走了奇怪的路径
- 明明配了负载分担，但流量都挤在一条链路上
- `show spanning-tree mst` 里看到 `Bound(RSTP)` 或 `Boundary` 标记

**验证方法（最快）**：

```cisco
SW1# show spanning-tree mst configuration digest
Name      [CAMPUS]
Revision  1     Instances configured 3
Digest    0x5B4D2E1F8A9C3D7E6F5A4B3C2D1E0F9A
          ↑ 这是三个参数的哈希值
```

**对比所有设备的 Digest**，相同就是同一个 Region，不同就有差异。**这比逐条对比 VLAN 映射表快得多**（尤其是映射表有几十行的时候）。

**发现边界端口**：
```cisco
SW3# show spanning-tree mst 1
Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- ---------------
Gi0/3            Desg FWD 20000     128.3    P2p Bound(RSTP)
                                                 ↑↑↑↑↑↑↑↑↑↑↑
                                    Boundary = 对面是不同 Region
```

**常见的不一致原因**：

| 原因 | 说明 |
|:--|:--|
| **大小写不同** | `REGION-A` vs `region-a` —— **Region Name 区分大小写** |
| 修订号忘了改 | 新加的设备默认 revision 0 |
| VLAN 映射有一个数字不同 | `instance 1 vlan 10,20` vs `instance 1 vlan 10,21` |
| **H3C/华为忘了 `active region-configuration`** | ★ 国内混合组网头号坑 |
| 新增 VLAN 后只在部分设备上更新了映射 | 运维流程问题 |

**运维建议**：把 MST Region 配置做成**标准模板**，任何设备上线都用同一份配置。新增 VLAN 时，**必须同时更新所有设备的映射表**（并且 revision 号 +1，作为版本标记）。
</details>

**3.** Root Guard 和 Loop Guard 分别配在哪些端口上？它们防的是什么？

<details><summary>答案</summary>

| | **Root Guard** | **Loop Guard** |
|:--|:--|:--|
| **配在哪** | **朝向下游/接入层的端口**（汇聚交换机上朝接入交换机的口） | **根端口和阻塞端口**（通常全局启用，自动作用于这些端口） |
| **检测什么** | 该端口**收到了更优的 BPDU** | 该端口**突然收不到 BPDU 了** |
| **防什么** | 有人接入优先级更高的交换机，**抢走根桥**，导致全网流量绕路 | **单向链路**故障导致阻塞端口误认为"上游没了"而转为转发，**形成环路** |
| **触发状态** | `root-inconsistent`（阻塞该端口） | `loop-inconsistent`（保持阻塞） |
| **恢复** | 不再收到更优 BPDU 后**自动恢复** | 重新收到 BPDU 后**自动恢复** |

**记忆口诀**：
- **Root Guard 防"外来的更好"** —— 你不许当根桥
- **Loop Guard 防"该来的没来"** —— 收不到 BPDU 时宁可继续阻塞，也不冒险转发

**配置**：
```cisco
! Root Guard：汇聚交换机朝向接入层的所有端口
Dist-SW(config)# interface range GigabitEthernet1/0/1-24
Dist-SW(config-if-range)# spanning-tree guard root

! Loop Guard：全局启用最省事
SW1(config)# spanning-tree loopguard default
! 或接口级
SW1(config-if)# spanning-tree guard loop
```

**为什么 Root Guard 要配在朝下游的端口**：

根桥应该在核心/汇聚层。如果有人在接入层接了一台优先级极低（数值小）的交换机，它会赢得根桥选举，导致**全网流量都要绕到那台接入交换机**——性能灾难，而且难以察觉（网络是通的，只是慢）。

Root Guard 让这个端口在收到"更优 BPDU"时进入 `root-inconsistent` 状态，**拒绝接受这个新根桥**。

**为什么 Loop Guard 要配在根端口和阻塞端口**：

正常情况下，阻塞端口会持续收到上游的 BPDU（这是它保持阻塞的依据）。

如果发生**单向链路故障**（光纤 TX/RX 有一个方向坏了）：
- 阻塞端口**收不到 BPDU 了**
- 标准 STP 行为：等 Max Age(20s) 超时 → 认为上游没了 → **转为指定端口并开始转发**
- 但实际上上游还在，链路只是单向 → **形成环路**

Loop Guard 的行为：收不到 BPDU 时，**不转为转发，而是置为 `loop-inconsistent`（保持阻塞）**。宁可损失这条链路，也不冒环路的风险。

**互补关系**：
```cisco
SW1(config)# spanning-tree loopguard default      ! STP 层面兜底
SW1(config)# udld aggressive                       ! 链路层面主动检测
```
- **UDLD** 检测的是"物理链路确实是单向的"，检测到就 err-disable 端口
- **Loop Guard** 检测的是"BPDU 没收到"（可能是链路单向，也可能是对端 CPU 忙、软件 bug、MTU 问题）

**两者应该同时配**，覆盖不同的失效模式。

**验证**：
```cisco
SW1# show spanning-tree inconsistentports
Name                 Interface              Inconsistency
-------------------- ---------------------- ------------------
MST1                 GigabitEthernet0/3     Root Inconsistent
MST1                 GigabitEthernet0/24    Loop Inconsistent
```
</details>

**4.** 为什么不应该在接口下配 `spanning-tree bpdufilter enable`？

<details><summary>答案</summary>

**因为接口级 BPDU Filter 会让该端口完全不收不发 BPDU，等于把它从 STP 中彻底摘除，失去所有环路保护。**

**两种配置方式的行为完全不同**（这是个经典陷阱）：

| 配置方式 | 行为 |
|:--|:--|
| **全局**：`spanning-tree portfast bpdufilter default` | PortFast 端口不主动发 BPDU；**但如果收到 BPDU，会退出 PortFast，转为普通 STP 端口** ✅ 保留了保护 |
| **接口**：`spanning-tree bpdufilter enable` | **完全不收不发 BPDU**，无论收到什么都不响应 ❌ **无任何保护** |

**接口级配置的危险场景**：

```
   SW1 Gi0/5 (bpdufilter enable，完全不参与 STP)
        │
    [有人接了一台交换机]
        │
   SW2 ────────────┐
        │          │  ← 另一条路径也接回 SW1
        └──────────┘
        
   → 形成环路
   → SW1 的 Gi0/5 因为 BPDU Filter，收不到也发不出 BPDU
   → STP 完全不知道这个环路的存在
   → 广播风暴，且没有任何机制能自动发现和阻止
```

**正确的做法**：

```cisco
! 接终端的端口：PortFast + BPDU Guard
SW1(config)# spanning-tree portfast default
SW1(config)# spanning-tree portfast bpduguard default
```

BPDU Guard 的行为恰恰相反——**收到 BPDU 就立即关闭端口**，这才是接终端端口应有的保护。

**什么情况下真的需要 BPDU Filter**：

极少数场景，比如：
- 连接第三方设备（某些防火墙、负载均衡器），它们不理解 BPDU 但会把 BPDU 转发出去，可能造成问题
- 服务提供商的边界，不希望自己的 STP 域和客户的合并

即便如此，**也应该优先用全局配置**，并且：
1. 明确文档记录为什么用它
2. 配合 `storm-control` 作为兜底
3. 定期审计这些端口

**判断口诀**：
- 你想说"这个口后面不该有交换机" → 用 **BPDU Guard**（收到就关）
- 你想说"这个口不参与 STP" → 先问自己**为什么**，99% 的情况你其实想要的是 BPDU Guard
</details>

**5.** Cisco 交换机和 H3C 交换机做 MST 对接，两边配置看起来完全一样，但 Region 就是不匹配。最可能漏了什么？

<details><summary>答案</summary>

**H3C（和华为）侧漏了 `active region-configuration` 命令。**

**H3C/华为的 MST Region 配置是"两阶段提交"的**：
1. 在 `stp region-configuration` 视图下写配置 → 只是写进了"待生效"缓冲区
2. **必须执行 `active region-configuration` 才会真正生效**

不敲这句，`display current-configuration` 里能看到配置，但**实际运行的还是旧的（或默认的）Region**。

```
# ❌ 错误（配了但没生效）
[SW-H3C] stp region-configuration
[SW-H3C-mst-region] region-name REGION-A
[SW-H3C-mst-region] revision-level 1
[SW-H3C-mst-region] instance 1 vlan 10 20
[SW-H3C-mst-region] quit                        ← 直接退出，配置没生效！

# ✅ 正确
[SW-H3C] stp region-configuration
[SW-H3C-mst-region] region-name REGION-A
[SW-H3C-mst-region] revision-level 1
[SW-H3C-mst-region] instance 1 vlan 10 20
[SW-H3C-mst-region] active region-configuration  ← ★ 必须
[SW-H3C-mst-region] quit
```

**验证方法（关键）**：

```
[SW-H3C] display stp region-configuration

 Oper Configuration                          ← 看 "Oper"（正在生效的）
   Format selector      :0
   Region name          :REGION-A
   Revision level       :1

   Instance   VLANs Mapped
   0          1 to 9, 11 to 19, 21 to 4094
   1          10, 20
```

**如果 Oper 部分显示的还是默认值**（Region name 通常是设备的 MAC 地址），说明没激活。

**Cisco 侧的对比**：
```cisco
SW-Cisco# show spanning-tree mst configuration
Name      [REGION-A]
Revision  1     Instances configured 2

SW-Cisco# show spanning-tree mst configuration digest
Digest    0x5B4D2E1F8A9C3D7E6F5A4B3C2D1E0F9A
```

**Cisco 是退出 `spanning-tree mst configuration` 子模式时自动生效**，没有单独的激活命令。这个差异导致从 Cisco 转到国产设备的工程师经常踩坑。

**其他可能的原因（排除法）**：

| 原因 | 检查 |
|:--|:--|
| **Region Name 大小写不同** | `REGION-A` ≠ `region-a`，**区分大小写** |
| Revision 号不同 | 逐个核对 |
| VLAN 映射有细微差异 | 比如一边 `vlan 10 20`，另一边 `vlan 10 to 20`（后者是 10-20 共 11 个！） |
| **一边是 MSTP，一边还是 PVST+** | `show spanning-tree summary` / `display stp brief` 确认模式 |
| 实例 0 的隐含映射不同 | 因为其他实例映射不同，导致实例 0 的剩余 VLAN 也不同 |

> **注意 H3C/华为的 `vlan 10 20` 和 `vlan 10 to 20` 的区别**：
> - `vlan 10 20` = VLAN 10 和 VLAN 20（两个）
> - `vlan 10 to 20` = VLAN 10 到 20（十一个）
>
> 这个语法差异也会导致映射表不一致。Cisco 用逗号和连字符：`vlan 10,20` vs `vlan 10-20`。

**混合组网的标准检查流程**：
1. 两边都确认模式是 MST/MSTP
2. H3C/华为侧确认执行了 `active region-configuration`
3. 对比 Cisco 的 Digest 和 H3C 的 Oper 配置
4. 看边界端口是否还显示 `Bound`
</details>

---

**上一章** ← [03 Overlay：VXLAN 与 LISP](03-Overlay-VXLAN与LISP.md) ｜ **下一章** → [05 一二层排障与高可用：HSRP/VRRP/GLBP](05-一层二层排障与高可用-HSRP-VRRP.md)
