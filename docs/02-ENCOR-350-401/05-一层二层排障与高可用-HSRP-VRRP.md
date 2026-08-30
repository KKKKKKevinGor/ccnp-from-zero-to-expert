# 05 · 一二层排障与高可用：HSRP / VRRP / GLBP

## ① 这章解决什么问题

PC 的默认网关配的是 `192.168.10.1`，这个 IP 在核心交换机 SW1 的 SVI 上。

**SW1 挂了怎么办？**

- PC 的网关配置是静态的，不会自动改成 SW2 的 `192.168.10.2`
- 就算 SW2 活着，PC 也不知道
- 结果：**整个 VLAN 断网**

FHRP（First Hop Redundancy Protocol，第一跳冗余协议）就是解决这个问题的：**让两台（或多台）路由器共享一个虚拟 IP，对 PC 来说永远只有一个网关。**

同时这一章也总结一二层的完整排障方法论——因为**网关高可用出问题时，症状往往和二层故障混在一起**，必须能分清。

---

## ② FHRP 三兄弟

### 2.1 共同原理

```
        PC 的网关配置：192.168.10.1（虚拟 IP）
                          │
        ┌─────────────────┴─────────────────┐
        │                                   │
   ┌────▼─────┐                       ┌─────▼────┐
   │   SW1    │                       │   SW2    │
   │ 真实 IP:  │                       │ 真实 IP:  │
   │ .10.2    │                       │ .10.3    │
   │          │  ←── Hello 报文 ──→    │          │
   │ Active   │                       │ Standby  │
   └──────────┘                       └──────────┘
        │
   持有虚拟 IP 192.168.10.1
   和虚拟 MAC，响应 ARP，转发流量
```

**关键点**：
- PC 的网关永远是**虚拟 IP**，PC 完全不知道背后有几台设备
- **ARP 响应的是虚拟 MAC**，所以主备切换时 PC 的 ARP 缓存不用更新
- 主设备故障 → 备设备接管虚拟 IP 和虚拟 MAC → PC 无感知

### 2.2 三者对比（★ 核心考点）

| | **HSRP** | **VRRP** | **GLBP** |
|:--|:--|:--|:--|
| 标准 | **Cisco 私有** | **IEEE 标准 (RFC 5798)** | **Cisco 私有** |
| 角色名 | **Active / Standby** | **Master / Backup** | **AVG / AVF** |
| 组播地址 | v1: `224.0.0.2`<br>v2: `224.0.0.102` | `224.0.0.18` | `224.0.0.102` |
| 协议/端口 | **UDP 1985** (v1) / **1985** (v2) | **IP 协议号 112** | **UDP 3222** |
| 虚拟 MAC | `0000.0C07.ACxx` (v1)<br>`0000.0C9F.Fxxx` (v2) | `0000.5E00.01xx` | `0007.B400.xxyy` |
| 优先级默认 | **100** | **100** | 100 |
| 优先级范围 | 0–255 | 1–254 | 0–255 |
| 抢占 | **默认关闭**，需 `preempt` | **默认开启** | 默认关闭 |
| 虚拟 IP 可用真实 IP | ❌ 不可以 | ✅ **可以**（此时优先级自动 255） | ❌ |
| Hello / Hold | 3 / 10 秒 | 1 / 3 秒 | 3 / 10 秒 |
| **负载分担** | ❌ 需要多组实现 | ❌ 需要多组实现 | ✅ **原生支持** |
| 组数上限 | v1: 0-255<br>v2: 0-4095 | 1-255 | 0-1023 |

**三个必须记住的差异**：

1. **HSRP 默认不抢占，VRRP 默认抢占。**
   → HSRP 配了高优先级但没生效？八成是忘了 `standby 10 preempt`。

2. **VRRP 的虚拟 IP 可以是某台路由器的真实接口 IP**（这台叫 IP Address Owner，优先级自动变成 255，永远是 Master）。HSRP 不允许。

3. **只有 GLBP 原生支持负载分担**（多台设备同时转发流量）。

### 2.3 HSRP 详解

**状态机**：
```
Initial → Learn → Listen → Speak → Standby → Active
```

| 状态 | 含义 |
|:--|:--|
| Initial | 刚启动 |
| Learn | 还不知道虚拟 IP（等待 Active 通告） |
| Listen | 知道虚拟 IP，但不是 Active 也不是 Standby |
| Speak | 参与选举，发送 Hello |
| **Standby** | 备用，随时准备接管 |
| **Active** | 转发流量，持有虚拟 IP 和 MAC |

**选举规则**：
1. **优先级大的赢**（默认 100）
2. 优先级相同，**接口 IP 大的赢**

**配置**：
```cisco
! ── SW1（期望的 Active）──
SW1(config)# interface Vlan10
SW1(config-if)# ip address 192.168.10.2 255.255.255.0
SW1(config-if)# standby version 2                       ! ★ 建议用 v2
SW1(config-if)# standby 10 ip 192.168.10.1              ! 组号 10，虚拟 IP
SW1(config-if)# standby 10 priority 110                 ! 高于默认 100
SW1(config-if)# standby 10 preempt                      ! ★ 必须配，否则不抢占
SW1(config-if)# standby 10 preempt delay minimum 60     ! 重启后等 60 秒再抢
SW1(config-if)# standby 10 timers 1 3                   ! Hello 1秒，Hold 3秒（加快切换）
SW1(config-if)# standby 10 authentication md5 key-string HsrpKey123
SW1(config-if)# standby 10 name VLAN10-GW

! ── 接口跟踪（★ 关键功能）──
SW1(config-if)# standby 10 track 1 decrement 20         ! 用 track 对象（推荐）
! 或直接跟踪接口
SW1(config-if)# standby 10 track GigabitEthernet0/1 20

! 定义 track 对象
SW1(config)# track 1 interface GigabitEthernet0/1 line-protocol

! ── SW2（Standby）──
SW2(config)# interface Vlan10
SW2(config-if)# ip address 192.168.10.3 255.255.255.0
SW2(config-if)# standby version 2
SW2(config-if)# standby 10 ip 192.168.10.1
SW2(config-if)# standby 10 priority 100
SW2(config-if)# standby 10 preempt
SW2(config-if)# standby 10 authentication md5 key-string HsrpKey123

! ── 查看 ──
SW1# show standby
SW1# show standby brief
SW1# show standby Vlan10 10
SW1# show track
SW1# debug standby events
```

**`show standby brief` 输出**：
```cisco
SW1# show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State   Active          Standby         Virtual IP
Vlan10      10   110 P Active  local           192.168.10.3    192.168.10.1
Vlan20      20   100 P Standby 192.168.20.3    local           192.168.20.1
                  ↑  ↑    ↑
              优先级 抢占  状态
```

### 2.4 接口跟踪（Interface Tracking）—— 最重要的功能

**没有跟踪会发生什么**：

```
              [ Internet ]
                    │
              上行链路断了！
                    │
   ┌────────────────X─────┐     ┌──────────────────┐
   │      SW1 (Active)    │     │   SW2 (Standby)  │
   │   上行 down 但 SVI up │     │   上行正常        │
   └──────────┬───────────┘     └────────┬─────────┘
              │                          │
              └──────────┬───────────────┘
                       [PC]
                       
   PC 的流量还是发给 SW1（因为 SW1 还是 Active）
   → SW1 上行断了，流量进了黑洞
   → 用户断网，但 HSRP 显示"一切正常"
```

**加上跟踪**：
```cisco
SW1(config)# track 1 interface GigabitEthernet0/1 line-protocol

SW1(config)# interface Vlan10
SW1(config-if)# standby 10 priority 110
SW1(config-if)# standby 10 track 1 decrement 20
```

**上行断了** → track 1 变成 Down → SW1 的优先级 **110 − 20 = 90** → 低于 SW2 的 100 → **SW2 抢占成为 Active** → 流量正常。

**decrement 值怎么定（考点）**：

```
必须保证：本端优先级 − decrement < 对端优先级

SW1 优先级 110，SW2 优先级 100
decrement 必须 > 10
→ 设成 20 有足够余量 ✓
→ 设成 5 的话：110-5=105 > 100，不会切换 ✗
```

**跟踪多个对象**：
```cisco
SW1(config)# track 1 interface GigabitEthernet0/1 line-protocol
SW1(config)# track 2 interface GigabitEthernet0/2 line-protocol

! 组合跟踪：任意一个 down 就算 down
SW1(config)# track 10 list boolean or
SW1(config-track)#  object 1
SW1(config-track)#  object 2

! 或：全部 down 才算 down
SW1(config)# track 11 list boolean and
SW1(config-track)#  object 1
SW1(config-track)#  object 2

! 跟踪 IP SLA（探测远端可达性，比跟踪接口更可靠）
SW1(config)# ip sla 1
SW1(config-ip-sla)#  icmp-echo 8.8.8.8 source-interface GigabitEthernet0/1
SW1(config-ip-sla)#  frequency 5
SW1(config)# ip sla schedule 1 life forever start-time now
SW1(config)# track 20 ip sla 1 reachability
SW1(config)# interface Vlan10
SW1(config-if)# standby 10 track 20 decrement 20
```

> **为什么跟踪 IP SLA 比跟踪接口更好**：如果上行经过交换机或光猫，**远端故障时本地接口依然 up**，跟踪接口检测不到。IP SLA 探测真实的端到端可达性，能发现这类故障。
>
> 这和 [基础篇第 4 章](../00-基础篇/04-静态路由与默认网关.md) 讲的浮动静态路由是同一个道理。

### 2.5 VRRP

```cisco
! ── SW1 ──
SW1(config)# interface Vlan10
SW1(config-if)# ip address 192.168.10.2 255.255.255.0
SW1(config-if)# vrrp 10 ip 192.168.10.1
SW1(config-if)# vrrp 10 priority 110
SW1(config-if)# vrrp 10 timers advertise 1              ! 通告间隔 1 秒
SW1(config-if)# vrrp 10 authentication md5 key-string VrrpKey
SW1(config-if)# vrrp 10 track 1 decrement 20
! 注意：VRRP 默认就抢占，不需要配 preempt

! ── 查看 ──
SW1# show vrrp
SW1# show vrrp brief
SW1# show vrrp interface Vlan10
```

**VRRP 的 IP Address Owner 特性**：
```cisco
SW1(config-if)# ip address 192.168.10.1 255.255.255.0    ! 真实 IP
SW1(config-if)# vrrp 10 ip 192.168.10.1                  ! 虚拟 IP = 真实 IP
! → SW1 成为 IP Address Owner，优先级自动变成 255（不可修改）
! → 永远是 Master（只要它活着）
```

> **HSRP 不允许这样做**。这是 HSRP 和 VRRP 一个明显的行为差异，考试会考。

### 2.6 GLBP：唯一原生支持负载分担的

**HSRP/VRRP 的局限**：同一时刻只有一台设备转发流量，**另一台完全闲置**。虽然可以用"多组 + 不同 VLAN 交错"来变相实现负载分担，但配置繁琐。

**GLBP 的解法**：

```
   AVG (Active Virtual Gateway)：只有一个，负责【回应 ARP】
        │
        ├─ PC1 问"192.168.10.1 的 MAC 是多少？"
        │  AVG 答："0007.B400.0101"（AVF-1 的虚拟 MAC）
        │
        ├─ PC2 问同样的问题
        │  AVG 答："0007.B400.0102"（AVF-2 的虚拟 MAC）  ← 轮流分配！
        │
   AVF (Active Virtual Forwarder)：可以有多个（最多 4 个），实际转发流量
```

**关键**：**一个虚拟 IP，多个虚拟 MAC**。AVG 通过给不同的主机返回不同的虚拟 MAC，把流量分散到多台设备上。

```cisco
SW1(config)# interface Vlan10
SW1(config-if)# ip address 192.168.10.2 255.255.255.0
SW1(config-if)# glbp 10 ip 192.168.10.1
SW1(config-if)# glbp 10 priority 110
SW1(config-if)# glbp 10 preempt
SW1(config-if)# glbp 10 load-balancing round-robin        ! 负载分担算法
SW1(config-if)# glbp 10 weighting 100 lower 90 upper 100
SW1(config-if)# glbp 10 weighting track 1 decrement 20

SW1# show glbp
SW1# show glbp brief
```

**三种负载分担算法**：

| 算法 | 说明 | 适用 |
|:--|:--|:--|
| **round-robin**（默认） | 轮流分配虚拟 MAC | 主机数量多、分布均匀 |
| **weighted** | 按权重分配（性能强的设备多分） | 设备性能不对等 |
| **host-dependent** | 同一主机永远得到同一个 MAC | 需要会话保持的场景 |

> **GLBP 的现实处境**：虽然功能最强，但它是 **Cisco 私有**，而且**现代网络更倾向于用 VSS/vPC/StackWise 从根本上消除对 FHRP 的需求**（见 [第 1 章](01-企业网络架构与设计.md)）。所以 GLBP 在实际部署中不如 HSRP 常见，主要是考点。

### 2.7 FHRP 与 VSS/vPC 的关系（重要认知）

```
   有 VSS/StackWise Virtual 时：
   
   两台物理设备 = 一台逻辑设备
        ↓
   只有一个 SVI，只有一个网关 IP
        ↓
   ★ 根本不需要 FHRP ★
   
   切换靠什么？靠 EtherChannel 的成员链路切换（亚秒级）
   比 HSRP 的 3-10 秒快得多
```

> **这是现代园区网设计的一个重要认知转变**：FHRP 是"两台独立设备"时代的产物。当设备可以逻辑合一时（VSS/vPC/Stack），FHRP 就成了多余的复杂度。
>
> **但 FHRP 依然重要**，因为：
> 1. 大量存量网络还在用
> 2. 无法堆叠的场景（不同型号、跨机房、路由器）
> 3. **考试必考**

---

## ③ 一二层排障方法论

### 3.1 分层排查框架

```
   ┌─────────────────────────────────────────────┐
   │ L1 物理层                                    │
   │  show interfaces status                     │
   │  show interfaces counters errors            │
   │  → down/down? CRC? runts? late collision?   │
   └──────────────────┬──────────────────────────┘
                      │ 物理 OK
   ┌──────────────────▼──────────────────────────┐
   │ L2 链路层                                    │
   │  show mac address-table                     │
   │  show interfaces trunk                      │
   │  show spanning-tree                         │
   │  show etherchannel summary                  │
   │  → VLAN 对吗? Trunk 放行了吗? STP 阻塞了吗?  │
   └──────────────────┬──────────────────────────┘
                      │ 二层 OK
   ┌──────────────────▼──────────────────────────┐
   │ L3 网络层（网关层）                           │
   │  show ip interface brief                    │
   │  show standby brief                         │
   │  show ip route                              │
   │  show ip arp                                │
   │  → SVI up? HSRP 正常? 路由存在?              │
   └─────────────────────────────────────────────┘
```

### 3.2 L1 物理层排障

```cisco
! ── 快速扫描所有接口状态 ──
SW1# show interfaces status
Port      Name          Status       Vlan       Duplex  Speed Type
Gi1/0/1   PC-01         connected    10         a-full  a-1000 10/100/1000BaseTX
Gi1/0/2                 notconnect   10           auto   auto 10/100/1000BaseTX
Gi1/0/3   Printer       err-disabled 10           auto   auto 10/100/1000BaseTX
Gi1/0/4   AP-01         disabled     900          auto   auto 10/100/1000BaseTX

! ── 错误计数器 ──
SW1# show interfaces counters errors
Port        Align-Err  FCS-Err  Xmit-Err  Rcv-Err  UnderSize  OutDiscards
Gi1/0/1             0        0         0        0          0            0
Gi1/0/5             0     1245         0     1245          0            0
                              ↑ 物理层有问题

! ── 光模块诊断（DOM）──
SW1# show interfaces GigabitEthernet1/0/49 transceiver detail
                                 Optical   Optical
        Temperature  Voltage  Tx Power  Rx Power
Port    (Celsius)    (Volts)   (dBm)     (dBm)
Gi1/0/49    42.5      3.31      -2.5     -25.8
                                          ↑ 接收光功率太低！正常应在 -3 ~ -15 dBm

! ── err-disable 原因 ──
SW1# show interfaces status err-disabled
Port      Name    Status              Reason
Gi1/0/3   Printer err-disabled        psecure-violation

SW1# show errdisable recovery
```

**状态判读表**：

| Status | Vlan 列显示 | 含义 | 排查方向 |
|:--|:--|:--|:--|
| `connected` | 数字 | ✅ 正常 | — |
| `notconnect` | 数字 | 物理未连接 | 网线、对端接口、光模块 |
| `err-disabled` | `err-disabled` | 被保护机制关闭 | `show int status err-disabled` 看原因 |
| `disabled` | 数字 | 被 `shutdown` | `no shutdown` |
| `inactive` | 数字 | **VLAN 不存在** | 创建该 VLAN |
| `monitoring` | — | SPAN 目的口 | 正常 |

> **`inactive` 状态很容易被忽略**：端口配了 `switchport access vlan 50`，但 VLAN 50 从来没被创建过。端口物理上是通的，但逻辑上不工作。

**关键错误计数器**：

| 计数器 | 含义 | 指向 |
|:--|:--|:--|
| `CRC` / `FCS-Err` | 校验错误 | **物理层**：线缆、接头、光衰、电磁干扰 |
| `Align-Err` | 帧对齐错误 | 物理层，常伴随 CRC |
| `runts` | < 64 字节 | 冲突、双工不匹配 |
| `giants` | > MTU | MTU 不匹配、未预期的 VLAN 标签 |
| **`late collision`** | 64 字节后的冲突 | **双工不匹配**（几乎可直接定性） |
| `input errors` | 输入错误总和 | 看细分项 |
| `OutDiscards` | 输出丢弃 | **拥塞**（缓冲区满） |
| `interface resets` | 接口复位 | 链路不稳定 |

### 3.3 L2 链路层排障

```cisco
! ── MAC 地址表 ──
SW1# show mac address-table
SW1# show mac address-table address 0050.5600.aabb
SW1# show mac address-table interface Gi1/0/1
SW1# show mac address-table count
SW1# show mac address-table dynamic vlan 10

! ── Trunk（★ 最常用）──
SW1# show interfaces trunk
SW1# show interfaces Gi1/0/24 switchport

! ── STP ──
SW1# show spanning-tree summary
SW1# show spanning-tree vlan 10
SW1# show spanning-tree blockedports
SW1# show spanning-tree inconsistentports
SW1# show spanning-tree detail | include ieee|occurr|from|is exec
       ↑ 这条命令能看到最近的拓扑变化次数和来源，排查震荡神器

! ── EtherChannel ──
SW1# show etherchannel summary
SW1# show lacp neighbor

! ── VLAN ──
SW1# show vlan brief
SW1# show vlan id 10
```

**`show spanning-tree detail` 找拓扑震荡源头**：
```cisco
SW1# show spanning-tree vlan 10 detail | include from|occurr
  Number of topology changes 1245 last change occurred 00:00:12 ago
          from GigabitEthernet1/0/7
                ↑ 找到了震荡的源头端口
```

**这条命令是排查"网络莫名其妙卡顿"的利器**——如果拓扑变化次数很高且持续增长，说明有端口在反复 up/down。

### 3.4 L3 网关层排障

```cisco
! ── SVI 状态 ──
SW1# show ip interface brief | include Vlan
Vlan10   192.168.10.2   YES manual  up      up
Vlan20   192.168.20.2   YES manual  up      down       ← 有问题

! SVI 为什么 down？
SW1# show interfaces Vlan20
Vlan20 is up, line protocol is down
! 原因：VLAN 20 里没有任何 up 的端口

! ── HSRP ──
SW1# show standby brief
SW1# show standby Vlan10 10
SW1# show track

! ── ARP ──
SW1# show ip arp
SW1# show ip arp vlan 10
SW1# clear arp-cache
```

**SVI up 的三个条件**（回顾）：
1. VLAN 存在于 VLAN 数据库
2. **该 VLAN 内至少有一个 up 的物理端口**（或配了 `no autostate`）
3. SVI 本身没 shutdown

```cisco
! 强制 SVI 保持 up（用于测试或特殊场景）
SW1(config)# interface Vlan20
SW1(config-if)# no autostate
```

---

## ④ 配套实验：HSRP + 跟踪 + 故障切换

**拓扑**：
```
                    [ Internet / 上游 ]
                     │              │
                  Gi0/1          Gi0/1
              ┌──────┴──────┐  ┌────┴────────┐
              │    SW1      │  │    SW2      │
              │ Vlan10 .10.2│══│ Vlan10 .10.3│
              │ HSRP Pri 110│  │ HSRP Pri 100│
              └──────┬──────┘  └────┬────────┘
                     │              │
                     └──────┬───────┘
                          [SW3 接入]
                            │
                          [PC1]
                     GW: 192.168.10.1
```

### Step 1：基础 HSRP

```cisco
! ── SW1 ──
SW1(config)# interface Vlan10
SW1(config-if)# ip address 192.168.10.2 255.255.255.0
SW1(config-if)# standby version 2
SW1(config-if)# standby 10 ip 192.168.10.1
SW1(config-if)# standby 10 priority 110
SW1(config-if)# standby 10 preempt
SW1(config-if)# standby 10 name VLAN10-GW
SW1(config-if)# no shutdown

! ── SW2 ──
SW2(config)# interface Vlan10
SW2(config-if)# ip address 192.168.10.3 255.255.255.0
SW2(config-if)# standby version 2
SW2(config-if)# standby 10 ip 192.168.10.1
SW2(config-if)# standby 10 priority 100
SW2(config-if)# standby 10 preempt
SW2(config-if)# no shutdown
```

**验证**：
```cisco
SW1# show standby brief
Interface   Grp  Pri P State   Active   Standby         Virtual IP
Vlan10      10   110 P Active  local    192.168.10.3    192.168.10.1
                              ↑ SW1 是 Active ✓

SW2# show standby brief
Vlan10      10   100 P Standby 192.168.10.2  local      192.168.10.1
                              ↑ SW2 是 Standby ✓
```

**在 PC 上验证虚拟 MAC**：
```
PC1> ping 192.168.10.1
PC1> arp -a
  192.168.10.1    00-00-0c-9f-f0-0a    动态
                  ↑ HSRPv2 虚拟 MAC（0000.0C9F.Fxxx，xxx 是组号的十六进制）
```

### Step 2：基础切换测试

```cisco
! PC1 持续 ping 8.8.8.8
! 关掉 SW1 的 SVI
SW1(config)# interface Vlan10
SW1(config-if)# shutdown
```

**观察**：
```cisco
SW2# show standby brief
Vlan10      10   100 P Active  local    unknown        192.168.10.1
                              ↑ SW2 接管了 ✓
```
**ping 丢包**：默认 timers (3/10) 下丢 **3–10 个包**。

**加快切换**：
```cisco
SW1(config-if)# standby 10 timers msec 200 msec 750
SW2(config-if)# standby 10 timers msec 200 msec 750
```
再测：丢 **1 个包以内**。

> ⚠️ **timers 调太激进的风险**：CPU 高负载或链路抖动时可能误判，导致主备频繁切换（震荡），比切换慢更糟。**建议不低于 `msec 200 / msec 750`**。

### Step 3：加接口跟踪（★ 重点）

**先演示不加跟踪的问题**：
```cisco
! 恢复 SW1
SW1(config-if)# no shutdown
! 等 SW1 重新成为 Active

! 断掉 SW1 的上行
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# shutdown
```

**观察**：
```cisco
SW1# show standby brief
Vlan10      10   110 P Active  local    192.168.10.3    192.168.10.1
                              ↑ SW1 还是 Active！

! 但是：
PC1> ping 8.8.8.8
Request timed out.       ← 用户断网了！
```

**这就是"HSRP 显示一切正常，用户却断网"的经典场景。**

**加上跟踪**：
```cisco
SW1(config)# track 1 interface GigabitEthernet0/1 line-protocol

SW1(config)# interface Vlan10
SW1(config-if)# standby 10 track 1 decrement 20
```

**再次断上行**：
```cisco
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# shutdown
```

**观察**：
```cisco
SW1# show track
Track 1
  Interface GigabitEthernet0/1 line-protocol
  Line protocol is Down                        ← 检测到了
    1 change, last change 00:00:05

SW1# show standby Vlan10 10 | include Priority
  Priority 90 (configured 110)                 ← 110 - 20 = 90
    Track object 1 state Down decrement 20

SW1# show standby brief
Vlan10      10    90 P Standby 192.168.10.3   local     192.168.10.1
                               ↑ SW2 接管了 ✓

PC1> ping 8.8.8.8
Reply from 8.8.8.8: bytes=32 time=15ms          ← 用户正常了 ✓
```

### Step 4：跟踪 IP SLA（更可靠）

**场景**：上行经过一台交换机，远端故障时本地接口依然 up。

```cisco
! 定义探测
SW1(config)# ip sla 1
SW1(config-ip-sla)#  icmp-echo 8.8.8.8 source-interface GigabitEthernet0/1
SW1(config-ip-sla)#  frequency 5
SW1(config-ip-sla)#  timeout 2000
SW1(config-ip-sla)#  threshold 1000
SW1(config)# ip sla schedule 1 life forever start-time now

! 绑定 track
SW1(config)# track 20 ip sla 1 reachability
SW1(config-track)#  delay down 10 up 30          ! 防抖

! 应用到 HSRP
SW1(config)# interface Vlan10
SW1(config-if)# standby 10 track 20 decrement 20
```

**验证**：
```cisco
SW1# show ip sla statistics 1
IPSLA operation id: 1
        Latest RTT: 15 milliseconds
Latest operation start time: 10:23:45 CST Sat Aug 30 2026
Latest operation return code: OK               ← 探测正常

SW1# show track 20
Track 20
  IP SLA 1 reachability
  Reachability is Up
```

### Step 5：多组负载分担（HSRP 的变通做法）

**目标**：VLAN 10 走 SW1，VLAN 20 走 SW2，两台设备都在干活。

```cisco
! ── SW1 ──
SW1(config)# interface Vlan10
SW1(config-if)# standby 10 priority 110          ! VLAN 10 主
SW1(config-if)# standby 10 preempt
SW1(config)# interface Vlan20
SW1(config-if)# standby 20 priority 100          ! VLAN 20 备
SW1(config-if)# standby 20 preempt

! ── SW2 ──
SW2(config)# interface Vlan10
SW2(config-if)# standby 10 priority 100          ! VLAN 10 备
SW2(config-if)# standby 10 preempt
SW2(config)# interface Vlan20
SW2(config-if)# standby 20 priority 110          ! VLAN 20 主
SW2(config-if)# standby 20 preempt
```

**★ 关键：STP 根桥必须与 HSRP Active 对齐！**

```cisco
! VLAN 10：SW1 既是 HSRP Active，也是 STP 根桥
SW1(config)# spanning-tree vlan 10 root primary
SW1(config)# spanning-tree vlan 20 root secondary

! VLAN 20：SW2 既是 HSRP Active，也是 STP 根桥
SW2(config)# spanning-tree vlan 20 root primary
SW2(config)# spanning-tree vlan 10 root secondary
```

**为什么必须对齐**（重要）：

```
   如果不对齐：
   VLAN 10 的 HSRP Active 是 SW1，但 STP 根桥是 SW2
        ↓
   接入交换机的根端口指向 SW2
        ↓
   PC 的流量先发到 SW2（因为二层转发跟着 STP 走）
        ↓
   SW2 发现目的 MAC 是 SW1 的虚拟 MAC，再转给 SW1
        ↓
   ★ 流量多走了一跳，横向链路被白白占用
```

**这叫"次优路径"**，虽然能通，但浪费带宽且增加延迟。**HSRP Active 和 STP 根桥必须在同一台设备上。**

### Step 6：验证负载分担

```cisco
SW1# show standby brief
Interface   Grp  Pri P State   Active     Standby       Virtual IP
Vlan10      10   110 P Active  local      192.168.10.3  192.168.10.1
Vlan20      20   100 P Standby 192.168.20.3 local       192.168.20.1

SW1# show spanning-tree root
                                        Root   Hello Max Fwd
Vlan            Root ID                 Cost   Time  Age Dly  Root Port
---------------- -------------------- ------- ----- --- ---  ----------
VLAN0010        4106 0000.0000.1111        0     2   20  15   This bridge is root
VLAN0020        4116 0000.0000.2222        4     2   20  15   Gi0/24
                     ↑ SW1 是 VLAN10 的根，SW2 是 VLAN20 的根 ✓
```

---

## ⑤ 排障速查表

### HSRP/VRRP 排障

| 症状 | 怀疑点 | 验证 |
|:--|:--|:--|
| 两台都是 Active（脑裂） | Hello 收不到 | 检查 VLAN 是否互通、Trunk 是否放行、认证是否一致 |
| 高优先级的没成为 Active | **忘配 `preempt`** | `show standby brief` 看 P 列 |
| 上行断了但不切换 | **没配 track** | `show track`、加接口跟踪 |
| 配了 track 但不切换 | **decrement 值太小** | 确保 `优先级 − decrement < 对端优先级` |
| 主备频繁切换 | timers 太激进 / 链路抖动 | 加大 timers、配 `preempt delay` |
| 切换后流量绕路 | **STP 根桥与 HSRP 不对齐** | `show spanning-tree root` 对比 |
| PC 切换后仍不通 | ARP 缓存 | 正常不应该发生（虚拟 MAC 不变），检查是否用了真实 IP 做网关 |
| HSRP 邻居看不到 | 认证/版本/组号不匹配 | `show standby Vlan10 10`、`debug standby errors` |

### 完整的"用户断网"排查流程

```
用户报"上不了网"
        │
   ① PC 有 IP 吗？
        ├─ 169.254.x.x → DHCP 问题（查 VLAN、helper、地址池）
        └─ 有正常 IP → 继续
        │
   ② PC ping 得通网关吗？
        ├─ 不通 → 二层问题
        │         SW# show interfaces status         （端口 up 吗？VLAN 对吗？）
        │         SW# show mac address-table         （学到 PC 的 MAC 了吗？）
        │         SW# show interfaces trunk          （VLAN 放行了吗？）
        │         SW# show spanning-tree             （被阻塞了吗？）
        │         SW# show ip interface brief | inc Vlan  （SVI up 吗？）
        └─ 通 → 继续
        │
   ③ 网关能 ping 通外网吗？
        ├─ 不通 → 网关的上行有问题
        │         SW# show standby brief             （是 Active 吗？）
        │         SW# show track                     （跟踪对象状态？）
        │         SW# show ip route 0.0.0.0          （有默认路由吗？）
        │         SW# show interfaces Gi0/1          （上行接口正常吗？）
        └─ 通 → 继续
        │
   ④ PC ping 得通外网吗？
        ├─ 不通 → 路由或 NAT 或 ACL
        │         R# show ip route
        │         R# show ip nat translations
        │         R# show access-lists
        └─ 通但业务不通 → L4/L7 问题
                  telnet <ip> <port>
```

---

## ⑥ 三厂商对照 + 考点自测

### 三厂商对照

| 目的 | Cisco | H3C | 华为 |
|:--|:--|:--|:--|
| HSRP/VRRP | `standby 10 ip 192.168.10.1` | **只支持 VRRP**：`vrrp vrid 10 virtual-ip 192.168.10.1` | **只支持 VRRP**：`vrrp vrid 10 virtual-ip 192.168.10.1` |
| 优先级 | `standby 10 priority 110` | `vrrp vrid 10 priority 110` | `vrrp vrid 10 priority 110` |
| 抢占 | `standby 10 preempt` | 默认开启，`vrrp vrid 10 preempt-mode` | 默认开启 |
| 跟踪 | `standby 10 track 1 decrement 20` | `vrrp vrid 10 track interface Gi0/1 reduced 20` | `vrrp vrid 10 track interface Gi0/0/1 reduced 20` |
| 认证 | `standby 10 authentication md5 ...` | `vrrp vrid 10 authentication-mode md5 ...` | `vrrp vrid 10 authentication-mode md5 ...` |
| 查看 | `show standby brief` | `display vrrp` / `display vrrp verbose` | `display vrrp` |

> **重要差异**：**HSRP 和 GLBP 是 Cisco 私有的，H3C/华为只支持 VRRP**。混合厂商组网时必须用 VRRP。
>
> 另外，H3C/华为的跟踪减值关键字是 **`reduced`**，Cisco 是 **`decrement`**。

### 考点

- **HSRP 默认不抢占，VRRP 默认抢占**。
- **HSRP 虚拟 MAC `0000.0C07.ACxx`(v1) / `0000.0C9F.Fxxx`(v2)，VRRP `0000.5E00.01xx`**。
- **VRRP 用 IP 协议号 112**，HSRP 用 **UDP 1985**。
- **VRRP 的虚拟 IP 可以是真实接口 IP**（Owner，优先级 255）。
- **只有 GLBP 原生支持负载分担**。
- **接口跟踪的 decrement 值计算**。
- **HSRP Active 必须与 STP 根桥对齐**。
- **有 VSS/vPC 时不需要 FHRP**。

### 自测题

**1.** HSRP、VRRP、GLBP 最大的三个区别是什么？

<details><summary>答案</summary>

**① 标准 vs 私有**
- **HSRP、GLBP：Cisco 私有** —— 只能在 Cisco 设备之间用
- **VRRP：IEEE 标准（RFC 5798）** —— 跨厂商通用

**混合厂商组网必须用 VRRP**（H3C、华为、Juniper 都只支持 VRRP）。

**② 抢占的默认行为**
- **HSRP：默认不抢占**，必须显式配 `standby 10 preempt`
- **VRRP：默认抢占**
- GLBP：默认不抢占

> **这是实战中最常见的坑**：配了 `standby 10 priority 110` 期望它当 Active，但忘了 `preempt`——设备重启后原来的 Active 会保持不变，高优先级的那台只能当 Standby。
>
> 排查：`show standby brief` 的 **P 列**，有 `P` 才是配了抢占。

**③ 负载分担能力**
- **HSRP / VRRP：同一时刻只有一台转发**，另一台完全闲置
  - 变通做法：配多个组，不同 VLAN 交错（VLAN10 走 SW1，VLAN20 走 SW2）
- **GLBP：原生支持** —— 一个虚拟 IP 对应**多个虚拟 MAC**，AVG 通过给不同主机返回不同的虚拟 MAC，把流量分散到多个 AVF 上

**其他重要差异**：

| | HSRP | VRRP | GLBP |
|:--|:--|:--|:--|
| 角色名 | Active/Standby | Master/Backup | AVG/AVF |
| 传输 | UDP 1985 | **IP 协议号 112** | UDP 3222 |
| 虚拟 IP = 真实 IP | ❌ | ✅ **可以**（Owner，优先级自动 255） | ❌ |
| Hello/Hold | 3/10 秒 | 1/3 秒 | 3/10 秒 |

**现代视角的补充**：

在有 **VSS / StackWise Virtual / vPC** 的网络里，**根本不需要 FHRP**——两台设备逻辑上是一台，只有一个 SVI、一个网关 IP，切换靠 EtherChannel 的成员链路切换（亚秒级，比 HSRP 快得多）。

FHRP 依然重要的场景：存量网络、无法堆叠的设备、跨机房、纯路由器环境，以及**考试**。
</details>

**2.** 配了 `standby 10 priority 110`，但设备还是 Standby。为什么？

<details><summary>答案</summary>

**最可能是忘了配 `standby 10 preempt`（抢占）。**

**HSRP 默认不抢占。** 这意味着：即使你的优先级更高，**只要现有的 Active 还活着，你就不会去抢它**。

**典型场景**：
1. SW1（priority 110）和 SW2（priority 100）同时配好
2. 但 SW2 先启动完成，成为 Active
3. SW1 后启动，虽然优先级 110 更高，但因为没配 preempt，**只能当 Standby**
4. 这个状态会一直保持，直到 SW2 故障

**验证**：
```cisco
SW1# show standby brief
Interface   Grp  Pri P State   Active         Standby   Virtual IP
Vlan10      10   110   Standby 192.168.10.3   local     192.168.10.1
                        ↑ 
                   P 列是空的 = 没配抢占
                   
! 配了抢占后：
Vlan10      10   110 P Active  local          192.168.10.3  192.168.10.1
                        ↑ 有 P
```

**修复**：
```cisco
SW1(config)# interface Vlan10
SW1(config-if)# standby 10 preempt
```
配上后 SW1 会**立即抢占**成为 Active。

**建议同时配抢占延迟**：
```cisco
SW1(config-if)# standby 10 preempt delay minimum 60
```

**为什么需要延迟**：设备重启后，接口很快就 up 了，但**路由协议还没收敛完**（OSPF 邻居建立 + LSDB 同步可能需要 30-60 秒）。如果这时立即抢占成为 Active，流量会涌进来但路由表还不完整 → **黑洞**。

延迟 60 秒让路由先收敛完再接管，避免这个问题。

**其他可能的原因（排除法）**：

| 原因 | 检查 |
|:--|:--|
| **配置的接口不对** | `show run interface Vlan10` 确认命令在正确的接口下 |
| **组号不匹配** | 一边 `standby 10`，另一边 `standby 1` → 会各自成为独立组的 Active |
| **有 track 在减优先级** | `show standby Vlan10 10 \| include Priority` 看实际优先级 |
| **HSRP 版本不一致** | 一边 v1 一边 v2 → 互相看不见 |
| **认证不匹配** | `debug standby errors` |
| **SVI 是 down 的** | `show ip interface brief \| include Vlan10` |

**看实际生效的优先级**：
```cisco
SW1# show standby Vlan10 10
Vlan10 - Group 10 (version 2)
  State is Standby
  Priority 90 (configured 110)              ← 配的是 110，实际是 90
    Track object 1 state Down decrement 20  ← 原来是 track 在减
```
</details>

**3.** HSRP 配了跟踪上行接口，但上行断了却没有切换。检查什么？

<details><summary>答案</summary>

**按可能性排序检查四点**：

**① decrement 值太小（最常见）**

必须保证：**本端优先级 − decrement < 对端优先级**

```
SW1 优先级 110，SW2 优先级 100

decrement = 5  → 110-5 = 105 > 100  ❌ 不会切换
decrement = 10 → 110-10 = 100 = 100 ❌ 平局，不切换（HSRP 不抢占平局）
decrement = 20 → 110-20 = 90 < 100  ✅ 会切换
```

**验证**：
```cisco
SW1# show standby Vlan10 10 | include Priority
  Priority 105 (configured 110)          ← 减完还是比对端高
    Track object 1 state Down decrement 5
```

**② track 对象没有真的 Down**

如果上行经过交换机、光猫、运营商设备，**远端故障时本地接口依然 up**：
```cisco
SW1# show track 1
Track 1
  Interface GigabitEthernet0/1 line-protocol
  Line protocol is Up                    ← 还是 Up！本地接口没 down
```

**解法：改用 IP SLA 跟踪端到端可达性**
```cisco
SW1(config)# ip sla 1
SW1(config-ip-sla)#  icmp-echo 8.8.8.8 source-interface GigabitEthernet0/1
SW1(config-ip-sla)#  frequency 5
SW1(config)# ip sla schedule 1 life forever start-time now
SW1(config)# track 20 ip sla 1 reachability
SW1(config-track)#  delay down 10 up 30
SW1(config)# interface Vlan10
SW1(config-if)# standby 10 track 20 decrement 20
```

**③ 对端没配 `preempt`**

即使 SW1 的优先级降下来了，**如果 SW2 没配抢占，它也不会主动去抢**。

```cisco
SW2# show standby brief
Vlan10      10   100   Standby ...
                    ↑ P 列空的
```

**修复**：
```cisco
SW2(config-if)# standby 10 preempt
```

> **两端都要配 preempt**：SW1 配是为了故障恢复后能抢回来，SW2 配是为了 SW1 降优先级时能接管。

**④ track 命令语法或绑定错误**

```cisco
! 检查 track 对象是否存在
SW1# show track
! 空的？说明没定义

! 检查 HSRP 是否真的绑定了这个 track
SW1# show standby Vlan10 10 | include Track
! 没有输出？说明没绑定

! 检查 track 编号是否一致
SW1# show run | section track
track 1 interface GigabitEthernet0/1 line-protocol
SW1# show run interface Vlan10 | include track
 standby 10 track 2 decrement 20        ← 绑的是 track 2，但只定义了 track 1！
```

**完整的验证流程**：
```cisco
! 1. track 对象存在且状态正确
SW1# show track

! 2. HSRP 绑定了 track，且看到了实际优先级
SW1# show standby Vlan10 10

! 3. 两端都配了 preempt
SW1# show standby brief    ← 看 P 列
SW2# show standby brief

! 4. 实际测试
SW1(config)# interface Gi0/1
SW1(config-if)# shutdown
SW1# show standby brief    ← 应该变成 Standby
SW2# show standby brief    ← 应该变成 Active
```
</details>

**4.** 为什么 HSRP 的 Active 设备必须和 STP 根桥在同一台上？

<details><summary>答案</summary>

**否则会产生"次优路径"，流量白白多走一跳，浪费横向链路带宽。**

**不对齐的情况**：

```
   VLAN 10：
   HSRP Active = SW1
   STP 根桥    = SW2       ← 不对齐！
   
   接入交换机 SW3 的根端口指向 SW2（因为 STP 说根桥在那边）
   
   PC 发包给网关（目的 MAC = HSRP 虚拟 MAC，这个 MAC 在 SW1 上）
        ↓
   SW3 按 STP 转发，把包发给 SW2（根端口方向）
        ↓
   SW2 一看：这个目的 MAC 不是我的，是 SW1 的虚拟 MAC
        ↓
   SW2 通过横向链路转给 SW1
        ↓
   SW1 才做三层转发
   
   ★ 多走了一跳，横向链路承载了本不该有的流量
```

**对齐的情况**：
```
   HSRP Active = SW1
   STP 根桥    = SW1       ← 对齐 ✓
   
   SW3 的根端口指向 SW1
        ↓
   PC 的包直接到达 SW1
        ↓
   SW1 直接三层转发出去
   
   ✅ 最短路径，横向链路只用于故障切换
```

**正确配置（负载分担场景）**：

```cisco
! ── SW1：VLAN 10 的 HSRP Active + STP 根桥 ──
SW1(config)# interface Vlan10
SW1(config-if)# standby 10 priority 110
SW1(config-if)# standby 10 preempt
SW1(config)# spanning-tree vlan 10 root primary       ! ★ 对齐

SW1(config)# interface Vlan20
SW1(config-if)# standby 20 priority 100               ! VLAN 20 是备
SW1(config-if)# standby 20 preempt
SW1(config)# spanning-tree vlan 20 root secondary     ! ★ 对齐

! ── SW2：VLAN 20 的 HSRP Active + STP 根桥 ──
SW2(config)# interface Vlan20
SW2(config-if)# standby 20 priority 110
SW2(config-if)# standby 20 preempt
SW2(config)# spanning-tree vlan 20 root primary       ! ★ 对齐

SW2(config)# interface Vlan10
SW2(config-if)# standby 10 priority 100
SW2(config-if)# standby 10 preempt
SW2(config)# spanning-tree vlan 10 root secondary     ! ★ 对齐
```

**验证对齐**：
```cisco
SW1# show standby brief | include Vlan10
Vlan10      10   110 P Active  local  ...              ← HSRP Active

SW1# show spanning-tree vlan 10 | include root
             This bridge is the root                   ← STP 根桥
                                                       ✅ 对齐
```

**排查次优路径的方法**：
```cisco
! 在 SW2 上看横向链路的流量
SW2# show interfaces Gi0/24 | include rate
  5 minute input rate 450000000 bits/sec        ← 横向链路流量异常高
  
! 说明大量流量在两台核心之间横穿，很可能就是不对齐
```

**这个原则的推广**：**任何"控制平面选路"和"数据平面转发路径"不一致的情况都会产生次优路径。** 除了 HSRP + STP，还有：
- OSPF 的 DR 和 STP 根桥
- vPC 环境下的 peer-link 流量
- FHRP Active 和上行链路 cost 的关系

**设计原则：让所有的"主"角色集中在同一台设备上，"备"角色集中在另一台。**
</details>

**5.** 有了 VSS / vPC / StackWise，还需要 HSRP 吗？为什么？

<details><summary>答案</summary>

**不需要。**

**原因**：VSS/StackWise Virtual/vPC 把**两台物理设备虚拟成一台逻辑设备**：

```
   传统（两台独立设备）：              VSS（逻辑一台）：
   
   SW1: Vlan10 IP = .10.2            逻辑设备: Vlan10 IP = .10.1
   SW2: Vlan10 IP = .10.3                     （只有一个 SVI）
   虚拟 IP = .10.1（HSRP）
        ↓                                      ↓
   需要 HSRP 协调谁是 Active          ★ 只有一个网关，无需协调 ★
```

**逻辑上只有一个 SVI、一个网关 IP，根本不存在"哪台是主"的问题。**

**切换靠什么**：

不是靠 FHRP，而是靠 **EtherChannel 的成员链路切换**：

```
   Access-SW
      ║  ╲          ← 跨设备 EtherChannel（MEC）
      ║   ╲            一条到 VSS 的成员 1
      ║    ╲           一条到 VSS 的成员 2
   ┌──╨─────╨──┐
   │  VSS 逻辑  │
   └───────────┘
   
   成员 1 挂了 → EtherChannel 少一条成员链路 → 流量自动走另一条
                → 亚秒级，甚至无丢包
```

**速度对比**：

| 方案 | 切换时间 |
|:--|:--|
| HSRP 默认 timers (3/10) | **3–10 秒** |
| HSRP 优化 timers (msec 200/750) | **< 1 秒** |
| **EtherChannel 成员切换** | **< 100 毫秒**（甚至无丢包） |
| STP 收敛（Rapid-PVST） | < 1 秒 |
| 路由协议收敛（OSPF + BFD） | < 1 秒 |

**额外的好处**：
- **配置量减半**：只需要配一遍（VSS 是单一控制平面）
- **不会配置不一致**：不存在"两台设备配置不同步"的问题
- **不需要考虑 HSRP 与 STP 根桥对齐**（因为逻辑上只有一台）
- **所有链路 100% 利用**（没有 STP 阻塞，没有 HSRP Standby 闲置）

**什么时候还需要 FHRP**：

| 场景 | 原因 |
|:--|:--|
| **存量网络** | 大量已部署的网络还在用 HSRP，改造成本高 |
| **无法堆叠** | 型号不同、跨机房、距离超限 |
| **纯路由器环境** | 路由器通常不支持 VSS/Stack |
| **需要跨厂商** | 一台 Cisco 一台 H3C → 只能用 VRRP |
| **vPC 环境的特殊配置** | vPC 保持独立控制平面，某些场景仍配 HSRP（但工作方式不同——两台都能转发） |
| **考试** | ENCOR 必考 |

> **vPC 的特殊情况**：Nexus 的 vPC 保持两个独立的控制平面，所以**仍然需要配 HSRP**。但 vPC 有个优化：**两台设备都会响应虚拟 MAC 的流量并转发**（HSRP Active/Standby 只影响控制平面），所以实际上是双活的。这和传统 HSRP 只有一台转发不同。

**设计演进的脉络**：
```
独立双设备 + STP + HSRP     （链路浪费一半，切换 3-10 秒）
        ↓
VSS/Stack + 跨设备 EtherChannel  （链路全用，切换 <100ms，无需 FHRP）
        ↓
Routed Access / SD-Access    （三层到边缘，路由协议收敛，故障域最小）
```

**每一步的方向都是：减少协议数量，缩短故障域，加快收敛。**
</details>

---

**上一章** ← [04 交换进阶：STP 进阶与排障](04-交换进阶-STP进阶与排障.md) ｜ **下一章** → [06 EIGRP](06-EIGRP.md)
