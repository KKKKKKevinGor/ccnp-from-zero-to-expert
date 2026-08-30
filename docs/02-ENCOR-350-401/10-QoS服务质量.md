# 10 · QoS 服务质量

## ① 这章解决什么问题

分公司到总部的专线只有 20Mbps。上午 10 点，几件事同时发生：

- 财务部在开视频会议（需要低延迟、低抖动）
- 有人在下载 5GB 的安装包（能等，但会占满带宽）
- ERP 系统在同步数据（重要，但不那么急）
- 有人在刷抖音（无所谓）

**结果：视频会议卡成幻灯片。**

因为**网络默认是"先来先服务"（FIFO）**——它不知道哪个包更重要。下载的包和语音的包排在同一个队列里，语音包要等前面几百个下载包发完才能出去。

**QoS 就是给流量分优先级**：让语音包插队，让下载包排后面。

**核心认知**：**QoS 不能创造带宽，它只能决定"带宽不够时谁先死"。** 如果链路长期 100% 满载，QoS 只能保证关键业务活着，代价是其他业务更慢。

---

## ② 原理讲透

### 2.1 QoS 要解决的四个问题

| 问题 | 英文 | 影响 | 敏感业务 |
|:--|:--|:--|:--|
| **带宽** | Bandwidth | 传得慢 | 文件传输、备份 |
| **延迟** | Delay / Latency | 响应慢 | **语音（< 150ms）**、交易系统 |
| **抖动** | Jitter | **语音断续、视频卡顿** | **语音（< 30ms）**、视频 |
| **丢包** | Packet Loss | 重传、质量下降 | **语音（< 1%）**、TCP 应用 |

**语音的严格要求（必记）**：

| 指标 | 要求 |
|:--|:--|
| 单向延迟 | **≤ 150 ms**（ITU-T G.114） |
| 抖动 | **≤ 30 ms** |
| 丢包 | **≤ 1%** |
| 单路带宽 | G.711 约 **80 kbps**（含开销），G.729 约 **26 kbps** |

> **为什么语音对抖动特别敏感**：语音是恒定码率的实时流，接收端要按固定间隔播放。抖动大意味着包到达时间不规律，抖动缓冲区（jitter buffer）要么放大延迟，要么直接丢包 → 听起来断断续续。

### 2.2 三种 QoS 模型

| 模型 | 说明 | 现状 |
|:--|:--|:--|
| **Best Effort** | 尽力而为，不做任何区分（FIFO） | 默认行为 |
| **IntServ**（集成服务） | 用 **RSVP** 为每条流预留带宽 | ❌ 扩展性差，几乎不用 |
| **DiffServ**（区分服务） | **按类别标记，逐跳处理** | ✅ **主流** |

**DiffServ 的思想**：
- 在**网络边缘**给数据包**打标记**（分类）
- 网络**核心**只看标记，按预定义的策略处理（**Per-Hop Behavior, PHB**）
- 核心不需要维护每条流的状态 → **扩展性好**

**这个"边缘分类、核心按标记转发"的思想**和 MPLS、SD-Access 的 SGT 是一脉相承的。

### 2.3 标记（Marking）—— QoS 的语言

#### 二层：CoS (Class of Service)

在 **802.1Q 标签的 PCP 字段**（3 位），值 **0–7**。

```
┌──────────┬────────┬────────┬─────────┐
│ TPID     │ PCP(3) │ DEI(1) │ VID(12) │
│ 0x8100   │ ↑ CoS  │        │         │
└──────────┴────────┴────────┴─────────┘
```

> **限制**：CoS **只存在于带 802.1Q 标签的帧里**。Access 口（无标签）和三层链路上没有 CoS。所以**跨路由器传递必须用三层标记（DSCP）**。

#### 三层：DSCP (Differentiated Services Code Point)

在 **IP 头的 ToS 字段**，占 **6 位**，值 **0–63**。

```
   IP 头的 ToS 字节（8 位）：
   ┌───────────────────────┬─────────┐
   │      DSCP (6 位)       │ ECN(2)  │
   └───────────────────────┴─────────┘
   
   旧的 IP Precedence 只用了高 3 位（0-7），
   DSCP 扩展到 6 位（0-63），向后兼容
```

**常用 DSCP 值（★ 必背）**：

| PHB 名称 | DSCP 值 | 二进制 | 用途 |
|:--|:--|:--|:--|
| **EF** (Expedited Forwarding) | **46** | 101110 | **★ 语音（VoIP RTP）** |
| **CS5** | 40 | 101000 | 广播视频 |
| **AF41** | **34** | 100010 | **★ 交互式视频（会议）** |
| AF42 | 36 | 100100 | |
| AF43 | 38 | 100110 | |
| **CS4** | 32 | 100000 | 实时交互 |
| **AF31** | **26** | 011010 | **★ 语音信令（SIP/SCCP）** |
| **CS3** | 24 | 011000 | 呼叫信令 |
| **AF21** | **18** | 010010 | **★ 关键数据（ERP、数据库）** |
| CS2 | 16 | 010000 | 网管流量（SNMP/SSH） |
| **AF11** | 10 | 001010 | 批量数据（备份、文件传输） |
| CS1 | 8 | 001000 | **Scavenger（劣质流量，P2P、娱乐）** |
| **BE / Default** | **0** | 000000 | **★ 默认，尽力而为** |

**AF（Assured Forwarding）的编码规则**：

```
   AFxy
    ↑↑
    ││
    │└─ y = 丢弃优先级（1=低，2=中，3=高）
    └── x = 类别（1-4，数字越大越优先）

   DSCP 值 = 8x + 2y

   AF41 = 8×4 + 2×1 = 34 ✓
   AF21 = 8×2 + 2×1 = 18 ✓
   AF11 = 8×1 + 2×1 = 10 ✓
```

**记忆技巧**：
- **EF = 46** —— 语音专用，记死它
- **AF41 = 34** —— 视频
- **AF21 = 18** —— 关键数据
- **CS x = 8x** —— CS1=8, CS3=24, CS5=40

**CoS ↔ DSCP 的默认映射**（Cisco）：

| CoS | DSCP | 说明 |
|:--|:--|:--|
| 0 | 0 (BE) | 默认 |
| 1 | 8 (CS1) | Scavenger |
| 2 | 16 (CS2) | |
| 3 | 24 (CS3) | 信令 |
| 4 | 32 (CS4) | |
| **5** | **40 (CS5)** | ⚠️ 注意：**不是 EF(46)**！ |
| 6 | 48 (CS6) | 网络控制 |
| 7 | 56 (CS7) | |

> ⚠️ **常见误解**：很多人以为 CoS 5 自动映射到 EF(46)，其实默认映射到 **CS5(40)**。语音的 EF(46) 需要**显式配置映射**：
> ```cisco
> SW1(config)# mls qos map cos-dscp 0 8 16 24 32 46 48 56
> !                                          ↑ CoS 5 → DSCP 46
> ```

### 2.4 QoS 的四大工具

```
   ① 分类与标记 (Classification & Marking)
      → 识别流量，打上标记
           ↓
   ② 队列与调度 (Queuing & Scheduling)
      → 按标记分到不同队列，决定谁先发
           ↓
   ③ 拥塞避免 (Congestion Avoidance)
      → 队列快满时主动丢一些，避免全局同步
           ↓
   ④ 整形与监管 (Shaping & Policing)
      → 限制流量速率
```

#### ① 分类与标记

**在哪里做**：**网络边缘**（接入交换机、路由器的入接口）。

**信任边界（Trust Boundary）**：

```
   [IP 电话] ─── 信任 ─── [接入交换机] ─── 信任 ─── [核心] ...
   [PC] ────── 不信任 ──┘
   
   IP 电话自己打的标记可以信任
   PC 打的标记不能信任（用户可以随便改，把自己的下载标成 EF）
```

```cisco
! 信任 IP 电话的 CoS（但不信任它后面接的 PC）
SW1(config-if)# mls qos trust device cisco-phone
SW1(config-if)# mls qos trust cos

! 信任 DSCP（上联到核心的口）
SW1(config-if)# mls qos trust dscp

! 不信任，全部重标记为 0（接普通 PC 的口）
SW1(config-if)# mls qos cos 0
SW1(config-if)# mls qos cos override
```

> **信任边界原则**：**越靠近边缘越好**。在接入层就把不可信的标记清零或重标记，这样核心层可以放心地信任所有标记。
>
> 如果信任边界设在核心，那么任何一个用户都可以把自己的 P2P 流量标成 EF，占满语音队列——**QoS 体系完全失效**。

#### ② 队列与调度（★ 核心）

| 调度算法 | 说明 | 问题 |
|:--|:--|:--|
| **FIFO** | 先进先出 | 无区分 |
| **PQ**（Priority Queuing） | 严格优先级，高优先级永远先发 | **饿死低优先级** |
| **WFQ**（Weighted Fair） | 按流公平分配 | 无法指定业务 |
| **CBWFQ**（Class-Based WFQ） | **按类别分配带宽百分比** | 语音延迟仍不够低 |
| **LLQ**（Low Latency Queuing） | **CBWFQ + 一个严格优先队列** | ★ **语音的标准方案** |

**LLQ = CBWFQ + PQ**：

```
   ┌─────────────────────────────────────┐
   │  ★ 优先队列 (Priority Queue)         │  ← 语音 EF
   │     严格优先，但【有带宽上限】         │     priority percent 10
   └─────────────────────────────────────┘
   ┌─────────────────────────────────────┐
   │  类别队列 1（视频 AF41）30%           │  ← CBWFQ
   ├─────────────────────────────────────┤
   │  类别队列 2（关键数据 AF21）25%       │
   ├─────────────────────────────────────┤
   │  类别队列 3（默认 BE）剩余            │
   └─────────────────────────────────────┘
```

**LLQ 的关键设计：优先队列有带宽上限（Policer）。**

- 语音包**永远优先发送**（低延迟 ✓）
- 但**超过设定带宽的部分会被丢弃**（防止语音把其他业务饿死 ✓）

**这个"优先但有上限"的设计是 LLQ 的精髓。** 纯 PQ 会导致低优先级饿死，纯 CBWFQ 无法给语音足够低的延迟，LLQ 两者兼得。

**带宽分配的黄金法则**：

| 类别 | 建议占比 |
|:--|:--|
| **语音（LLQ 优先队列）** | **≤ 33%** |
| 视频 | 视需求 |
| 关键数据 | 视需求 |
| **默认类（BE）** | **≥ 25%** |
| Scavenger | 1-5% |

> **语音不超过 33%** 是 Cisco 的强烈建议。超过的话，即使有 policer，突发时也会严重挤压其他业务，而且优先队列本身的排队延迟会上升。

#### ③ 拥塞避免：WRED

**问题：TCP 全局同步（Global Synchronization）**

```
   队列满了 → 尾丢弃（Tail Drop）→ 所有 TCP 流同时丢包
        ↓
   所有 TCP 同时降速（拥塞窗口减半）→ 链路突然空闲
        ↓
   所有 TCP 同时加速 → 队列又满 → 又同时丢包
        ↓
   ★ 链路利用率呈锯齿状波动，平均利用率很低 ★
```

**WRED (Weighted Random Early Detection) 的解法**：

**队列还没满的时候，就开始随机丢一部分包**（丢包概率随队列长度增加）。

- **随机**丢 → 只有部分 TCP 流降速，不会全局同步
- **提前**丢 → 给 TCP 提前减速的信号，避免队列真正打满
- **加权**（Weighted）→ 优先级低的先丢（比如 AF13 比 AF11 先丢）

```cisco
R1(config-pmap-c)# random-detect dscp-based
R1(config-pmap-c)# random-detect dscp af11 32 40 10
!                                     ↑  ↑  ↑
!                             最小阈值 最大阈值 丢弃概率分母(1/10)
```

> ⚠️ **WRED 只对 TCP 有效**。UDP（语音、视频）不会因为丢包而降速，随机丢包只会直接损害质量。**所以绝不要在语音队列上启用 WRED。**

#### ④ 整形 vs 监管（★ 高频考点）

| | **整形 Shaping** | **监管 Policing** |
|:--|:--|:--|
| 超速时的动作 | **缓存排队，延迟发送** | **直接丢弃**（或重标记后放行） |
| 是否引入延迟 | ✅ **是** | ❌ 否 |
| 是否丢包 | ❌ 一般不（除非缓冲区满） | ✅ **是** |
| 对 TCP 的影响 | 友好（不触发重传） | 不友好（触发重传） |
| 应用方向 | **通常出方向（out）** | **入/出都可以** |
| 典型场景 | **出口限速匹配运营商带宽** | **限制某类流量的上限** |

```
   整形（Shaping）：
   突发流量 ──> [缓冲队列] ──> 平滑的输出
                  ↑ 排队等待，延迟增加
                  
   监管（Policing）：
   突发流量 ──> [令牌桶] ──> 符合速率的通过
                  └────────> ★ 超出的直接丢弃 ★
```

**什么时候用哪个**：

| 场景 | 选择 | 原因 |
|:--|:--|:--|
| **出口带宽匹配运营商** | **Shaping** | 运营商那边会丢弃超速流量，不如自己先排队 |
| 限制 P2P/下载的上限 | **Policing** | 就是要惩罚它 |
| 分公司出口（子速率接口） | **Shaping** | 物理口 1G，但只买了 100M |
| 入方向限速 | **Policing** | 包已经收到了，缓存没意义 |
| **对 TCP 业务限速** | **Shaping** | 丢包会触发重传，反而更浪费带宽 |

> **经典场景**：路由器的物理接口是 1Gbps，但运营商只给了 100Mbps。如果不做整形，路由器会以 1Gbps 的速率突发，运营商侧会**大量丢包**（因为超了合同速率）。
>
> **解法**：在出接口做 **Shaping 到 100Mbps**（实际配 95Mbps 留余量），让路由器自己排队，而不是让运营商随机丢弃。**这样你还能控制"丢谁"（QoS 生效），而运营商是无差别丢弃。**

---

## ③ 配置：MQC（模块化 QoS 命令行）

**Cisco QoS 的标准配置框架，三步走**：

```
   ① class-map    → 定义"什么是这类流量"（分类）
   ② policy-map   → 定义"对这类流量做什么"（动作）
   ③ service-policy → 应用到接口
```

### 完整配置示例

```cisco
! ═══ ① 分类：class-map ═══
R1(config)# class-map match-all VOICE
R1(config-cmap)#  match dscp ef
R1(config-cmap)#  match protocol rtp                 ! NBAR 深度识别

R1(config)# class-map match-any VIDEO
R1(config-cmap)#  match dscp af41 af42 af43
R1(config-cmap)#  match dscp cs4

R1(config)# class-map match-any SIGNALING
R1(config-cmap)#  match dscp cs3 af31
R1(config-cmap)#  match protocol sip
R1(config-cmap)#  match protocol skinny

R1(config)# class-map match-any CRITICAL-DATA
R1(config-cmap)#  match dscp af21 af22 af23
R1(config-cmap)#  match access-group name ERP-TRAFFIC

R1(config)# class-map match-any SCAVENGER
R1(config-cmap)#  match dscp cs1
R1(config-cmap)#  match protocol bittorrent
R1(config-cmap)#  match protocol edonkey

! match-all vs match-any：
!   match-all = 所有条件都要满足（AND）
!   match-any = 满足任一条件即可（OR）

! ═══ ② 策略：policy-map ═══
R1(config)# policy-map WAN-EDGE-OUT

R1(config-pmap)# class VOICE
R1(config-pmap-c)#  priority percent 10             ! ★ LLQ 优先队列，上限 10%
R1(config-pmap-c)#  set dscp ef

R1(config-pmap)# class VIDEO
R1(config-pmap-c)#  bandwidth percent 25            ! CBWFQ，保证 25%
R1(config-pmap-c)#  set dscp af41
R1(config-pmap-c)#  random-detect dscp-based        ! WRED

R1(config-pmap)# class SIGNALING
R1(config-pmap-c)#  bandwidth percent 5
R1(config-pmap-c)#  set dscp cs3

R1(config-pmap)# class CRITICAL-DATA
R1(config-pmap-c)#  bandwidth percent 25
R1(config-pmap-c)#  set dscp af21
R1(config-pmap-c)#  random-detect dscp-based

R1(config-pmap)# class SCAVENGER
R1(config-pmap-c)#  bandwidth percent 1             ! 只给 1%
R1(config-pmap-c)#  set dscp cs1

R1(config-pmap)# class class-default                ! ★ 默认类，必须留够
R1(config-pmap-c)#  bandwidth percent 25
R1(config-pmap-c)#  fair-queue
R1(config-pmap-c)#  random-detect

! ═══ ③ 应用到接口 ═══
R1(config)# interface GigabitEthernet0/1
R1(config-if)# service-policy output WAN-EDGE-OUT

! ═══ 整形（子速率接口场景）═══
R1(config)# policy-map SHAPE-100M
R1(config-pmap)# class class-default
R1(config-pmap-c)#  shape average 95000000          ! 整形到 95Mbps
R1(config-pmap-c)#  service-policy WAN-EDGE-OUT     ! ★ 嵌套：先整形再做队列

R1(config)# interface GigabitEthernet0/1
R1(config-if)# service-policy output SHAPE-100M

! ═══ 监管（限制某类流量）═══
R1(config)# policy-map POLICE-P2P
R1(config-pmap)# class SCAVENGER
R1(config-pmap-c)#  police cir 1000000 bc 31250 be 31250
R1(config-pmap-c-police)#   conform-action transmit
R1(config-pmap-c-police)#   exceed-action set-dscp-transmit cs1    ! 重标记后放行
R1(config-pmap-c-police)#   violate-action drop                    ! 严重超速才丢

! ═══ 查看 ═══
R1# show policy-map
R1# show policy-map interface GigabitEthernet0/1     ! ★ 最重要，看实际效果
R1# show class-map
R1# show mls qos                                     ! 交换机上
R1# show mls qos interface Gi1/0/1
R1# show mls qos maps
```

### `show policy-map interface` 输出解读（★ 排障核心）

```cisco
R1# show policy-map interface GigabitEthernet0/1

 GigabitEthernet0/1
  Service-policy output: WAN-EDGE-OUT

    Class-map: VOICE (match-all)
      1245678 packets, 98765432 bytes           ← 匹配到的流量
      5 minute offered rate 1850000 bps, drop rate 0 bps
                              ↑                        ↑
                        实际速率 1.85Mbps          没有丢包 ✓
      Match: dscp ef (46)
      Priority: 10% (2000 kbps), burst bytes 50000, b/w exceed drops: 0
                                                          ↑
                                            超过 10% 被丢的包数（应该是 0）

    Class-map: VIDEO (match-any)
      456789 packets, 234567890 bytes
      5 minute offered rate 4500000 bps, drop rate 12000 bps
                                                    ↑ 有丢包！
      Match: dscp af41 (34)
      Queueing
        queue limit 64 packets
        (queue depth/total drops/no-buffer drops) 45/1234/0
                        ↑           ↑
                   队列深度      总丢包数
        bandwidth 25% (5000 kbps)

    Class-map: class-default (match-any)
      9876543 packets, 8765432100 bytes
      5 minute offered rate 12000000 bps, drop rate 3400000 bps
                                                     ↑ 大量丢包（正常，默认类就是要被牺牲的）
```

**排障要点**：
| 观察 | 含义 |
|:--|:--|
| `packets` 为 0 | **class-map 没匹配到流量** —— 分类条件写错了，或流量没有被正确标记 |
| **`b/w exceed drops` > 0** | **语音超过了优先队列的带宽上限** —— 需要增加 `priority percent` 或减少语音路数 |
| 某类 `drop rate` 很高 | 该类带宽不足，需要调整分配 |
| `queue depth` 持续很高 | 队列积压，延迟大 |

---

## ④ 配套实验：WAN 出口 QoS

**场景**：分公司 → 总部，20Mbps 专线，物理口是 1Gbps。

**业务需求**：

| 业务 | 标记 | 带宽需求 | 要求 |
|:--|:--|:--|:--|
| VoIP（10 路 G.711） | EF (46) | 10 × 80k = **800kbps** | 低延迟、低抖动 |
| 视频会议 | AF41 (34) | **5 Mbps** | 保证带宽 |
| ERP 系统 | AF21 (18) | **5 Mbps** | 保证带宽 |
| 网页/邮件 | BE (0) | 剩余 | 尽力而为 |
| P2P/下载 | CS1 (8) | **限制** | 不能影响别人 |

### Step 1：接入层标记（信任边界）

```cisco
! ── 接 IP 电话的端口 ──
SW1(config)# interface range GigabitEthernet1/0/1-20
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 10          ! 数据 VLAN
SW1(config-if-range)# switchport voice vlan 100          ! 语音 VLAN
SW1(config-if-range)# mls qos trust device cisco-phone   ! 只信任 Cisco 电话
SW1(config-if-range)# mls qos trust cos                  ! 信任它的 CoS
SW1(config-if-range)# spanning-tree portfast

! ── 接普通 PC 的端口（不信任）──
SW1(config)# interface range GigabitEthernet1/0/21-40
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 10
SW1(config-if-range)# mls qos cos 0
SW1(config-if-range)# mls qos cos override               ! ★ 强制清零，不管用户标什么

! ── 上联到路由器的口（信任 DSCP）──
SW1(config)# interface GigabitEthernet1/0/48
SW1(config-if)# mls qos trust dscp

! ── 全局开启 QoS ──
SW1(config)# mls qos
SW1(config)# mls qos map cos-dscp 0 8 16 24 32 46 48 56  ! ★ CoS5 → DSCP 46(EF)
```

### Step 2：路由器上分类与标记

```cisco
! ── 用 ACL 识别应用 ──
R1(config)# ip access-list extended ERP-TRAFFIC
R1(config-ext-nacl)# permit tcp any host 10.1.30.50 eq 8080
R1(config-ext-nacl)# permit tcp any host 10.1.30.51 eq 1521

! ── 分类 ──
R1(config)# class-map match-any VOICE
R1(config-cmap)#  match dscp ef

R1(config)# class-map match-any VIDEO
R1(config-cmap)#  match dscp af41

R1(config)# class-map match-any SIGNALING
R1(config-cmap)#  match dscp cs3 af31

R1(config)# class-map match-any ERP
R1(config-cmap)#  match access-group name ERP-TRAFFIC
R1(config-cmap)#  match dscp af21

R1(config)# class-map match-any SCAVENGER
R1(config-cmap)#  match protocol bittorrent
R1(config-cmap)#  match protocol edonkey
R1(config-cmap)#  match dscp cs1

! ── 入方向重标记（把 ERP 打上 AF21）──
R1(config)# policy-map LAN-IN-MARK
R1(config-pmap)# class ERP
R1(config-pmap-c)#  set dscp af21
R1(config-pmap)# class SCAVENGER
R1(config-pmap-c)#  set dscp cs1

R1(config)# interface GigabitEthernet0/0
R1(config-if)# service-policy input LAN-IN-MARK
```

### Step 3：出方向队列策略

**带宽计算**（总共 20Mbps）：

| 类别 | 带宽 | 百分比 | 说明 |
|:--|:--|:--|:--|
| 语音 | 800 kbps → 留 2 Mbps | **10%** | LLQ，留 2.5 倍余量 |
| 视频 | 5 Mbps | **25%** | CBWFQ |
| 信令 | 1 Mbps | **5%** | CBWFQ |
| ERP | 5 Mbps | **25%** | CBWFQ |
| Scavenger | 200 kbps | **1%** | 严格限制 |
| 默认 | 6.8 Mbps | **34%** | 剩余 |

```cisco
R1(config)# policy-map WAN-OUT
R1(config-pmap)# class VOICE
R1(config-pmap-c)#  priority percent 10                  ! ★ LLQ
R1(config-pmap-c)#  set dscp ef

R1(config-pmap)# class VIDEO
R1(config-pmap-c)#  bandwidth percent 25
R1(config-pmap-c)#  random-detect dscp-based

R1(config-pmap)# class SIGNALING
R1(config-pmap-c)#  bandwidth percent 5

R1(config-pmap)# class ERP
R1(config-pmap-c)#  bandwidth percent 25
R1(config-pmap-c)#  random-detect dscp-based

R1(config-pmap)# class SCAVENGER
R1(config-pmap-c)#  bandwidth percent 1

R1(config-pmap)# class class-default
R1(config-pmap-c)#  bandwidth percent 34
R1(config-pmap-c)#  fair-queue
R1(config-pmap-c)#  random-detect
```

### Step 4：整形到实际带宽（★ 关键步骤）

**物理口是 1Gbps，但实际只有 20Mbps。** 不整形的话，路由器会以 1Gbps 突发，运营商侧无差别丢包，QoS 完全失效。

```cisco
R1(config)# policy-map SHAPE-WAN
R1(config-pmap)# class class-default
R1(config-pmap-c)#  shape average 19000000               ! 整形到 19Mbps（留 5% 余量）
R1(config-pmap-c)#  service-policy WAN-OUT               ! ★ 嵌套子策略

R1(config)# interface GigabitEthernet0/1
R1(config-if)# service-policy output SHAPE-WAN
```

> **为什么必须整形**：
> ```
> 不整形：路由器 1Gbps 突发 → 运营商侧 20Mbps 处丢包
>         ★ 运营商是【无差别丢弃】，语音包和下载包一样丢 ★
>         → 你的 QoS 队列在 1Gbps 接口上根本没排队（没拥塞就不排队）
>         → QoS 完全没生效
>
> 整形后：路由器自己限速到 19Mbps → 队列在【路由器上】形成
>         ★ QoS 在这里生效，你能控制"丢谁" ★
> ```
>
> **这是 QoS 部署最容易被忽略、也最关键的一步。** 很多人配了完整的 policy-map 却发现没效果，根因就是接口速率远大于实际带宽，队列从来不拥塞。

### Step 5：验证

```cisco
! ── 检查标记是否正确 ──
R1# show policy-map interface GigabitEthernet0/0 input
    Class-map: ERP (match-any)
      45678 packets                          ← 有匹配 ✓
      Match: access-group name ERP-TRAFFIC
      QoS Set
        dscp af21
          Packets marked 45678                ← 标记成功 ✓

! ── 检查队列效果 ──
R1# show policy-map interface GigabitEthernet0/1
    Class-map: VOICE (match-all)
      12456 packets, 1987654 bytes
      5 minute offered rate 780000 bps, drop rate 0 bps
                              ↑ 780kbps       ↑ 0 丢包 ✓
      Priority: 10% (1900 kbps), b/w exceed drops: 0
                                                    ↑ 没超限 ✓

! ── 检查整形 ──
R1# show policy-map interface Gi0/1 | include shape|Queueing -A 5
      shape (average) cir 19000000, bc 76000, be 76000
      target shape rate 19000000
```

### Step 6：压力测试

```bash
# 用 iperf 灌满链路
$ iperf -c <总部IP> -t 300 -P 10          # 10 个 TCP 流打满

# 同时打语音电话，测试质量
# 或用 iperf 模拟语音流：
$ iperf -c <总部IP> -u -b 80k -S 0xB8 -t 300
#                            ↑ 0xB8 = DSCP 46 (EF) 的 ToS 值
```

**预期结果**：
- 链路被打满（`show interfaces` 显示 19Mbps）
- **语音流 0 丢包，延迟 < 20ms，抖动 < 5ms** ✓
- 默认类大量丢包（正常，它就是被牺牲的）

```cisco
R1# show policy-map interface Gi0/1
    Class-map: VOICE
      5 minute offered rate 800000 bps, drop rate 0 bps       ← ✓
    Class-map: class-default
      5 minute offered rate 15000000 bps, drop rate 8000000 bps  ← 被牺牲
```

**对比实验：关掉 QoS 再测**
```cisco
R1(config)# interface Gi0/1
R1(config-if)# no service-policy output SHAPE-WAN
```
**预期**：语音开始丢包、抖动飙升、通话质量明显下降。

---

## ⑤ 排障速查表

| 症状 | 怀疑点 | 验证 |
|:--|:--|:--|
| **QoS 配了没效果** | **接口没拥塞**（物理速率 >> 实际带宽） | **加 shaping** |
| class-map 匹配数为 0 | 分类条件写错 / 流量没被标记 | `show policy-map interface` |
| 语音仍然卡 | 优先队列带宽不足 | 看 `b/w exceed drops` |
| | 信任边界配错 | `show mls qos interface` |
| | 上游没有 QoS（端到端不一致） | 逐跳检查 |
| 标记在中途丢失 | 某台设备重标记了 | 逐跳 `show policy-map interface` |
| CoS 到 DSCP 映射不对 | 默认 CoS5→CS5(40) 而非 EF(46) | `show mls qos maps cos-dscp` |
| 带宽百分比配置报错 | 总和超过 100% | 检查所有 class 的 bandwidth |
| WRED 让语音更卡 | **WRED 用在了 UDP 队列上** | 语音队列不要配 WRED |
| 整形后带宽利用率低 | Bc/Be 参数不当 | 调整突发参数 |

### QoS 部署检查清单

```
□ ① 信任边界设在接入层，PC 端口不信任
□ ② CoS→DSCP 映射正确（CoS5 → EF 46）
□ ③ 端到端所有设备的 QoS 策略一致（标记不能中途丢失）
□ ④ 出口做了 shaping（物理速率 ≠ 实际带宽时必须）
□ ⑤ 语音用 LLQ（priority），不超过 33%
□ ⑥ class-default 至少留 25%
□ ⑦ 所有 bandwidth percent 总和 ≤ 100%（含 priority）
□ ⑧ 语音队列不配 WRED
□ ⑨ 压力测试验证过（打满链路，语音仍正常）
□ ⑩ 监控到位（丢包率、队列深度）
```

---

## ⑥ 三厂商对照 + 考点自测

### 三厂商对照

| 目的 | Cisco | H3C | 华为 |
|:--|:--|:--|:--|
| 分类 | `class-map NAME` | `traffic classifier NAME` | `traffic classifier NAME` |
| 行为 | `policy-map NAME` | `traffic behavior NAME` | `traffic behavior NAME` |
| 关联 | policy-map 里 `class NAME` | `qos policy NAME` + `classifier X behavior Y` | `traffic policy NAME` |
| 应用 | `service-policy output NAME` | `qos apply policy NAME outbound` | `traffic-policy NAME outbound` |
| 设 DSCP | `set dscp ef` | `remark dscp ef` | `remark dscp ef` |
| LLQ | `priority percent 10` | `queue ef bandwidth 2000` | `queue ef bandwidth 2000` |
| CBWFQ | `bandwidth percent 25` | `queue af bandwidth 5000` | `queue af bandwidth 5000` |
| 整形 | `shape average 19000000` | `gts cir 19000000` | `qos gts cir 19000000` |
| 监管 | `police cir 1000000` | `car cir 1000000` | `qos car cir 1000000` |
| 信任 | `mls qos trust dscp` | `qos trust dscp` | `trust dscp` |
| 查看 | `show policy-map interface X` | `display qos policy interface X` | `display traffic policy statistics interface X` |

### 考点

- **DSCP 值必背**：EF=46（语音）、AF41=34（视频）、AF21=18（关键数据）、CS3=24（信令）、BE=0
- **AF 编码公式：DSCP = 8x + 2y**
- **CoS 只有 3 位（0-7），只存在于 802.1Q 标签里**
- **LLQ = CBWFQ + 有上限的优先队列**
- **整形 vs 监管**：整形缓存延迟，监管直接丢弃
- **信任边界应设在网络边缘**
- **WRED 只对 TCP 有效**
- **接口速率 >> 实际带宽时必须做 shaping**

### 自测题

**1.** 语音应该标记成什么 DSCP？为什么？视频呢？

<details><summary>答案</summary>

**语音（RTP 媒体流）→ DSCP EF (46)**

**EF = Expedited Forwarding（加速转发）**，专门为**低延迟、低抖动、低丢包**的实时流量设计。

**为什么是 EF**：
- 它对应 **LLQ 的严格优先队列**——语音包永远先于其他流量发送
- 语音的三个硬指标：延迟 ≤ 150ms、抖动 ≤ 30ms、丢包 ≤ 1%，只有严格优先才能保证

**语音信令（SIP/SCCP/H.323）→ CS3 (24) 或 AF31 (26)**

**为什么信令和媒体流分开**：
- 信令流量很小（几 kbps），但**必须可靠**（丢了就打不通电话）
- 信令**不需要严格的低延迟**（100ms 的建立延迟用户感知不到）
- 如果把信令也放进 EF 队列，会占用宝贵的优先队列带宽

**视频会议 → AF41 (34)**

**为什么不是 EF**：
- 视频**码率高且突发性强**（关键帧比普通帧大很多倍）
- 如果放进优先队列，一个关键帧的突发就可能挤压语音
- **AF41 用 CBWFQ 保证带宽即可**，视频对抖动的容忍度比语音高（有缓冲区）

**完整的企业 QoS 标记方案**（Cisco 推荐的 12 类模型简化版）：

| 业务 | DSCP | 队列方式 | 建议带宽 |
|:--|:--|:--|:--|
| **语音 RTP** | **EF (46)** | **LLQ 优先队列** | ≤ 33%，通常 10% |
| **广播视频** | CS5 (40) | LLQ 或 CBWFQ | 视需求 |
| **交互视频（会议）** | **AF41 (34)** | CBWFQ | 25% |
| **呼叫信令** | **CS3 (24)** | CBWFQ | 5% |
| 网管流量 | CS2 (16) | CBWFQ | 2% |
| **关键业务数据** | **AF21 (18)** | CBWFQ + WRED | 25% |
| 事务型数据 | AF31 (26) | CBWFQ + WRED | 视需求 |
| 批量数据（备份） | AF11 (10) | CBWFQ + WRED | 4% |
| **Scavenger（P2P/娱乐）** | **CS1 (8)** | CBWFQ | **1%** |
| **默认（网页/邮件）** | **BE (0)** | CBWFQ + WRED | **≥ 25%** |

**记忆技巧**：
```
EF = 46        ← 语音，记死它
AF41 = 34      ← 视频
AF21 = 18      ← 关键数据
CS3 = 24       ← 信令
CS1 = 8        ← 垃圾流量
BE = 0         ← 默认

AF 公式：DSCP = 8x + 2y
CS 公式：DSCP = 8x
```

**⚠️ 特别注意的坑**：Cisco 交换机上 **CoS 5 默认映射到 DSCP 40 (CS5)，不是 46 (EF)**。必须显式配置：
```cisco
SW1(config)# mls qos map cos-dscp 0 8 16 24 32 46 48 56
!                                          ↑ 第 6 个值对应 CoS 5
```
</details>

**2.** 整形（Shaping）和监管（Policing）有什么区别？分别用在什么场景？

<details><summary>答案</summary>

| | **整形 Shaping** | **监管 Policing** |
|:--|:--|:--|
| 超速时的动作 | **缓存排队，延迟发送** | **直接丢弃**（或重标记后放行） |
| 引入延迟 | ✅ **是**（排队等待） | ❌ 否 |
| 引入丢包 | ❌ 一般不（除非缓冲区满） | ✅ **是** |
| 输出速率曲线 | **平滑** | **锯齿状**（允许突发后被削） |
| 对 TCP | **友好**（不触发重传和降速） | **不友好**（触发重传） |
| 需要缓冲区 | ✅ 是 | ❌ 否 |
| 应用方向 | 通常**出方向** | **入/出都可以** |

```
   整形：
   突发流量 ──> [缓冲队列] ──> 平滑输出（速率恒定）
                  ↑ 排队，延迟增加但不丢包
   
   监管：
   突发流量 ──> [令牌桶] ──┬──> 符合速率的通过
                          └──> ★ 超出的直接丢弃 ★
```

**场景选择**：

| 场景 | 选 | 原因 |
|:--|:--|:--|
| **出口匹配运营商带宽**（物理 1G，买了 100M） | **Shaping** | ★ 运营商会无差别丢弃超速流量。自己先排队，能控制"丢谁" |
| 限制 P2P/下载的上限 | **Policing** | 就是要惩罚它，不值得为它缓存 |
| 分公司子速率接口 | **Shaping** | 同第一条 |
| 入方向限速 | **Policing** | 包已经收进来了，缓存没有意义 |
| **给 TCP 业务限速** | **Shaping** | 丢包触发 TCP 重传，反而浪费更多带宽 |
| 给 UDP 视频限速 | Policing | UDP 不重传，丢就丢了 |
| 运营商侧对客户限速 | Policing | 严格执行合同速率 |

**★ 最重要的场景：出口带宽不匹配**

```
   问题：路由器物理接口 1Gbps，运营商专线只有 100Mbps
   
   不做 shaping：
     路由器以 1Gbps 突发 → 运营商设备在 100M 处丢包
     ★ 运营商是【无差别丢弃】，语音包和 P2P 包一视同仁 ★
     而且路由器的接口从来不拥塞（1G 口跑 100M 流量），
     ★ QoS 队列根本不排队，policy-map 完全没生效 ★
   
   做了 shaping 到 95Mbps：
     队列在【路由器上】形成
     ★ 你的 QoS 策略在这里生效，能保证语音优先 ★
     发出去的流量已经符合 100M，运营商不会再丢
```

**这是 QoS 部署最容易被忽略的一步。** 很多人配了完美的 policy-map 却发现"QoS 没效果"，根因就在这里。

**配置对比**：
```cisco
! ── 整形（嵌套子策略）──
policy-map SHAPE-100M
 class class-default
  shape average 95000000                   ! 整形到 95Mbps
  service-policy QUEUE-POLICY              ! ★ 嵌套：在整形后的虚拟接口上做队列

! ── 监管（三色标记）──
policy-map POLICE-P2P
 class SCAVENGER
  police cir 1000000 bc 31250 be 31250
   conform-action transmit                 ! 符合 → 通过
   exceed-action set-dscp-transmit cs1     ! 超出 → 降级后通过
   violate-action drop                     ! 严重超出 → 丢弃
```

**Policing 的三色标记（考点）**：

| 颜色 | 条件 | 典型动作 |
|:--|:--|:--|
| **绿（conform）** | 在 CIR 内 | transmit |
| **黄（exceed）** | 超过 CIR 但在 Bc+Be 内 | 重标记后 transmit |
| **红（violate）** | 超过所有桶 | drop |

**"重标记而不丢弃"是个好实践**：超速的流量降级成 CS1（Scavenger），链路空闲时它还能跑，链路拥塞时它最先被丢。既限制了它，又不浪费空闲带宽。
</details>

**3.** 什么是 LLQ？为什么优先队列要有带宽上限？

<details><summary>答案</summary>

**LLQ (Low Latency Queuing) = CBWFQ + 一个带上限的严格优先队列。**

```
   ┌────────────────────────────────────────┐
   │ ★ 优先队列（Priority Queue）            │  ← 语音 EF
   │   严格优先：有包就先发                    │     priority percent 10
   │   ★ 但有带宽上限（内置 policer）★        │
   └────────────────────────────────────────┘
   ┌────────────────────────────────────────┐
   │ 类别队列 1：视频 AF41    bandwidth 25%  │
   ├────────────────────────────────────────┤
   │ 类别队列 2：关键数据 AF21 bandwidth 25% │  ← CBWFQ
   ├────────────────────────────────────────┤
   │ 类别队列 3：默认 BE      bandwidth 25%  │
   └────────────────────────────────────────┘
```

**为什么优先队列必须有上限（★ 核心考点）**：

**因为纯优先队列（PQ）会"饿死"低优先级流量。**

```
   没有上限的 PQ：
   
   只要优先队列里还有包，就永远先发它
        ↓
   如果语音流量突然暴涨（比如被攻击、或配置错误
   导致大量流量被标成 EF）
        ↓
   ★ 优先队列吃掉 100% 带宽 ★
        ↓
   ★ 所有其他业务完全饿死，一个包都发不出去 ★
        ↓
   路由协议 Hello 包发不出去 → 邻居断开 → 网络崩溃
```

**LLQ 的解法：给优先队列内置一个 policer。**

```cisco
R1(config-pmap-c)# priority percent 10
!                  ↑ 严格优先，但最多用 10% 带宽
!                    超过 10% 的部分【被丢弃】
```

**行为**：
- **链路不拥塞时**：优先队列可以超过 10%（policer 只在拥塞时才严格执行）
- **链路拥塞时**：优先队列**最多用 10%**，超出部分丢弃

**这个设计同时满足了两个矛盾的需求**：
1. 语音需要**严格优先**（低延迟） ✓
2. 其他业务需要**不被饿死**（公平性） ✓

**带宽规划的黄金法则**：

| 类别 | 建议 |
|:--|:--|
| **优先队列（语音）** | **≤ 33%** |
| **class-default** | **≥ 25%** |

**为什么语音不超过 33%**：
1. 超过后，即使有 policer，突发时也会严重挤压其他业务
2. **优先队列内部也会排队**——如果语音本身就占了 50% 带宽，语音包之间也要互相等待，延迟反而上升
3. Cisco 的实测建议值

**为什么 class-default 要留 25%**：
默认类承载着大量"没被分类的流量"，包括：
- 路由协议报文（OSPF Hello、BGP Keepalive）
- 管理流量（SSH、SNMP）
- 各种未识别的业务

饿死它们会导致网络本身不稳定。

**验证优先队列是否超限**：
```cisco
R1# show policy-map interface Gi0/1
    Class-map: VOICE (match-all)
      Priority: 10% (2000 kbps), burst bytes 50000, b/w exceed drops: 1245
                                                          ↑↑↑↑
                                              ★ 有丢包！说明语音超过了 10%
```

**看到 `b/w exceed drops` > 0 的处理**：
1. **增加 `priority percent`**（但不超过 33%）
2. **减少并发语音路数**
3. 检查是否有非语音流量被误标记成 EF（**常见！**）
   ```cisco
   R1# show policy-map interface Gi0/1 | include VOICE -A 3
   ! 看 offered rate 是否远超预期
   ! 10 路 G.711 应该是 800kbps，如果是 5Mbps 说明有别的流量混进来了
   ```

**多个优先队列（LLQ 可以有多个 priority 类）**：
```cisco
policy-map WAN-OUT
 class VOICE
  priority percent 10
 class VIDEO-REALTIME
  priority percent 20            ! 第二个优先队列
```
但**所有 priority 类共享同一个物理优先队列**，它们之间是 FIFO 的。所以通常只用一个。
</details>

**4.** 什么是信任边界？为什么它应该设在网络边缘？

<details><summary>答案</summary>

**信任边界（Trust Boundary）= 网络中"开始信任数据包 QoS 标记"的那个点。**

```
   [IP 电话] ──★信任★─┐
                      ├── [接入交换机] ──信任── [汇聚] ── [核心]
   [PC] ────不信任────┘
        ↑
   信任边界在这里
```

**为什么必须设在边缘**：

**因为终端用户可以随意设置自己数据包的 QoS 标记。**

如果信任边界设在核心：
```
   用户在自己 PC 上把 BT 下载的流量标成 DSCP EF (46)
        ↓
   接入交换机信任了这个标记，原样传给核心
        ↓
   核心把它放进【语音优先队列】
        ↓
   ★ BT 流量占满优先队列的带宽上限 ★
        ↓
   ★ 真正的语音被挤掉，QoS 体系完全失效 ★
```

**在 Windows/Linux 上改 DSCP 标记非常容易**（组策略、注册表、iptables、socket 选项），任何稍懂技术的用户都能做到。

**正确的信任边界配置**：

```cisco
! ── ① 接普通 PC 的端口：不信任，强制清零 ──
SW1(config)# interface range GigabitEthernet1/0/21-40
SW1(config-if-range)# mls qos cos 0
SW1(config-if-range)# mls qos cos override          ! ★ 强制覆盖，不管用户标什么

! ── ② 接 IP 电话的端口：条件信任 ──
SW1(config)# interface range GigabitEthernet1/0/1-20
SW1(config-if-range)# mls qos trust device cisco-phone   ! ★ 只有检测到是 Cisco 电话才信任
SW1(config-if-range)# mls qos trust cos
! CDP 检测不到电话时，自动降级为不信任

! ── ③ 上联到汇聚/核心的端口：信任 DSCP ──
SW1(config)# interface GigabitEthernet1/0/48
SW1(config-if)# mls qos trust dscp

! ── ④ 接服务器的端口：按需分类和标记 ──
SW1(config)# interface GigabitEthernet1/0/45
SW1(config-if)# service-policy input MARK-SERVER-TRAFFIC
! 用 ACL 识别应用，主动打标记，而不是信任服务器自己标的
```

**`mls qos trust device cisco-phone` 的巧妙之处**：

```
   端口下接的是 IP 电话（电话再串接 PC）
        ↓
   交换机通过 ★ CDP ★ 检测对端是不是 Cisco 电话
        ├─ 是 → 信任电话的 CoS（语音 VLAN 的流量标记为 CoS 5）
        │       同时通过 CDP 告诉电话："把 PC 的流量标成 CoS 0"
        └─ 否（有人拔了电话直接接 PC）→ ★ 自动降级为不信任 ★
```

**这解决了"电话和 PC 共用一个端口"的难题**：既信任电话，又不信任它后面的 PC。

**信任边界的三种典型位置**：

| 位置 | 适用场景 | 说明 |
|:--|:--|:--|
| **接入交换机端口**（推荐） | 大多数企业网 | 最靠近边缘，控制力最强 |
| IP 电话 | 有 Cisco IP 电话 | 电话自己标记，交换机条件信任 |
| 汇聚层 | 接入层设备不支持 QoS | 次优选择 |

**验证**：
```cisco
SW1# show mls qos interface GigabitEthernet1/0/25
GigabitEthernet1/0/25
trust state: not trusted                    ← PC 口，不信任 ✓
trust mode: not trusted
COS override: enable                        ← 强制清零 ✓
default COS: 0

SW1# show mls qos interface GigabitEthernet1/0/1
GigabitEthernet1/0/1
trust state: trust cos                      ← 电话口，信任 CoS ✓
trust device: cisco-phone                   ← 条件信任 ✓
```

**原则总结**：
> **信任边界越靠近网络边缘越好。** 在边缘做一次分类和标记，核心层就可以简单地信任所有标记，专注于高速转发（这也呼应了 [第 1 章](01-企业网络架构与设计.md) "核心层不做策略"的原则）。
</details>

**5.** 你配了完整的 QoS 策略，但发现完全没有效果。最可能的原因是什么？

<details><summary>答案</summary>

**最可能是：接口的物理速率远大于实际可用带宽，导致接口从来不拥塞，QoS 队列根本不排队。**

**QoS 生效的前提：必须有拥塞。**

```
   路由器物理接口：1 Gbps
   运营商实际带宽：20 Mbps
        ↓
   业务流量 20 Mbps 涌向 1Gbps 接口
        ↓
   ★ 接口一点都不拥塞（利用率只有 2%）★
        ↓
   ★ 队列永远是空的，不需要调度 ★
        ↓
   ★ policy-map 配了也白配，队列策略完全不生效 ★
        ↓
   流量以 1Gbps 突发出去，在【运营商侧】被丢弃
   而运营商是【无差别丢弃】，语音和下载一样丢
```

**解法：加整形（Shaping）**

```cisco
R1(config)# policy-map SHAPE-WAN
R1(config-pmap)# class class-default
R1(config-pmap-c)#  shape average 19000000        ! 整形到 19Mbps（留 5% 余量）
R1(config-pmap-c)#  service-policy WAN-OUT        ! ★ 嵌套原有的队列策略

R1(config)# interface GigabitEthernet0/1
R1(config-if)# service-policy output SHAPE-WAN    ! 应用父策略
```

**整形后的效果**：
```
   整形器把发送速率限制在 19Mbps
        ↓
   ★ 队列在【路由器上】形成 ★
        ↓
   ★ 队列策略生效，语音优先发送 ★
        ↓
   发出去的流量已经符合 20Mbps，运营商不再丢包
```

**验证是否真的拥塞了**：
```cisco
R1# show policy-map interface GigabitEthernet0/1
    Class-map: class-default
      Queueing
        queue limit 64 packets
        (queue depth/total drops/no-buffer drops) 0/0/0
                        ↑    ↑
                   队列深度 0，丢包 0
                   ★ 说明从来没排过队，QoS 没生效 ★
```

**如果配置正确，应该看到**：
```cisco
    Class-map: class-default
        (queue depth/total drops/no-buffer drops) 45/12345/0
                        ↑         ↑
                    有队列深度  有丢包 → QoS 在工作 ✓
```

**其他可能的原因（按概率排序）**：

**② class-map 匹配数为 0（分类没生效）**
```cisco
R1# show policy-map interface Gi0/1
    Class-map: VOICE (match-all)
      0 packets, 0 bytes            ← ★ 一个包都没匹配到
```
**原因**：
- 流量根本没被标记（上游没做 marking）
- 标记在中途被清除了（某台设备重标记为 0）
- class-map 的 match 条件写错

**排查**：
```cisco
! 逐跳检查标记是否保留
R1# show policy-map interface <入接口> input
! 或抓包看 DSCP 值
```

**③ 应用方向错了**
```cisco
! ❌ 错误：队列策略应用在入方向
R1(config-if)# service-policy input WAN-OUT
! 入方向不能做队列（包已经收到了，排队没意义）
! 只能做分类、标记、监管

! ✅ 正确
R1(config-if)# service-policy output WAN-OUT
```

**④ 应用在错误的接口**
```cisco
! 应该应用在【拥塞的那个接口】——通常是 WAN 出口
! 应用在 LAN 口没有意义（LAN 口 1G，不会拥塞）
```

**⑤ 全局 QoS 没开（交换机上）**
```cisco
SW1(config)# mls qos                   ! ★ Catalyst 交换机必须先全局开启
SW1# show mls qos
QoS is enabled                         ← 必须是 enabled
```

**⑥ 端到端不一致**
QoS 必须**逐跳生效**。如果中间有一台设备没配 QoS 或重标记了，整条链路的效果就断了。

```
   [接入交换机 ✓] → [汇聚 ✓] → [核心 ✗ 没配] → [WAN 路由器 ✓]
                                    ↑
                            这一跳就把优势抹平了
```

**排障流程**：
```
① show policy-map interface <出接口>
   → 有 queue depth 和 drops 吗？没有 = 没拥塞 = 需要 shaping
   
② 各 class-map 的 packets 计数
   → 是 0 吗？= 分类没匹配上
   
③ 逐跳检查 DSCP 标记是否保留
   → 抓包或 show policy-map interface
   
④ 检查应用方向和接口
   → 队列必须在 output，且在拥塞的接口上
   
⑤ 压力测试验证
   → 打满链路，看语音是否还正常
```

> **实战教训**：**没有经过压力测试的 QoS 配置等于没配。** 平时链路不满，任何配置看起来都"正常"；只有在真正拥塞时才知道策略对不对。**部署 QoS 后必须做一次打满链路的验证测试。**
</details>

---

**上一章** ← [09 组播 Multicast](09-组播Multicast.md) ｜ **下一章** → [11 无线架构与漫游](11-无线架构与漫游.md)
