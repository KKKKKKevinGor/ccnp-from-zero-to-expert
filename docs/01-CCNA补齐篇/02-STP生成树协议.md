# 02 · STP 生成树协议

## ① 这章解决什么问题

上一章我们知道了：**二层环路会导致广播风暴，因为以太网帧没有 TTL**。

但网络又必须有冗余——两台核心交换机之间只接一根线，线断了整栋楼瘫痪，这是不可接受的。

所以我们陷入了一个矛盾：
- **要可靠性 → 必须有冗余物理链路**
- **有冗余物理链路 → 就会成环 → 就会广播风暴**

**STP 就是这个矛盾的解**：保留物理冗余，用协议在逻辑上**阻塞**掉多余的链路，形成一棵无环的树。主链路断了，被阻塞的链路自动放开顶上。

学 STP 的方式不是背结论，而是**掌握选举的推导过程**。因为 RSTP、MST、PVST+，甚至 H3C/华为的实现，用的都是同一套选举逻辑。推导会了，一通百通。

---

## ② 原理讲透

### 2.1 STP 的三步走

STP 的全部工作可以概括成三步，**顺序不能乱**：

```
第 1 步：选出一个【根桥】(Root Bridge)              —— 整个网络选 1 个
          ↓
第 2 步：每台非根交换机选出一个【根端口】(Root Port) —— 每台设备选 1 个
          ↓
第 3 步：每条链路段选出一个【指定端口】(Designated Port) —— 每个网段选 1 个
          ↓
剩下的端口 → 全部【阻塞】(Blocking / Non-Designated)
```

**记忆口诀**：**一网一根桥，一机一根口，一段一指定，其余全阻塞。**

### 2.2 BPDU：交换机之间的选举选票

交换机通过发送 **BPDU（Bridge Protocol Data Unit）** 来交换信息，目的 MAC 是组播地址 `01:80:C2:00:00:00`。

BPDU 里最关键的四个字段（**这四个就是选举的比较顺序**）：

```
┌───────────────────────────────────────┐
│ 1. Root Bridge ID   （根桥ID）         │  ← 我认为谁是根
│ 2. Root Path Cost   （到根的开销）      │  ← 我到根有多远
│ 3. Sender Bridge ID （发送者桥ID）      │  ← 我是谁
│ 4. Sender Port ID   （发送者端口ID）    │  ← 我从哪个口发的
└───────────────────────────────────────┘
```

**Bridge ID（桥 ID）结构 —— 8 字节**：

```
┌──────────────────┬──────────────────────┬─────────────────────┐
│  桥优先级 (4 bit)  │  扩展系统 ID (12 bit) │   MAC 地址 (48 bit)  │
│  0 ~ 61440        │   = VLAN ID          │                     │
│  步长 4096        │                      │                     │
└──────────────────┴──────────────────────┴─────────────────────┘
        └──────── 合起来 16 bit = 优先级字段 ────────┘
```

**为什么优先级必须是 4096 的倍数**：因为低 12 位被"扩展系统 ID"占用了（用来放 VLAN ID，这是 PVST+ 的做法，让每个 VLAN 能有不同的根桥）。所以实际能调的只有高 4 位，步长就是 2^12 = 4096。

**默认优先级 32768**。配置时实际写入的是 `优先级 + VLAN ID`：
```
priority 32768 + VLAN 10 = 32778   ← show 命令里看到的就是这个数
```

### 2.3 第 1 步：选根桥

**规则：Bridge ID 最小的当根桥。**

比较顺序：
1. **先比优先级**（数字小的赢）
2. 优先级相同，**再比 MAC 地址**（小的赢）

```
SW1: 优先级 32768, MAC aaaa.aaaa.aaaa
SW2: 优先级 32768, MAC bbbb.bbbb.bbbb
SW3: 优先级 4096,  MAC cccc.cccc.cccc    ← 优先级最小，SW3 是根桥
SW4: 优先级 32768, MAC dddd.dddd.dddd
```

如果全都是默认优先级 32768，那就**是 MAC 地址最小的当根桥**——通常是网络里最老、最破的那台交换机（老设备 MAC 地址往往更小）。

> ⚠️ **这是 STP 最大的实践陷阱**：不手工指定根桥，网络会自己选出一个位置极差的根桥（比如某个接入层的老交换机），导致所有流量绕远路，性能极差且难以排查。
>
> **铁律：生产网络必须手工指定根桥。** 通常指定核心交换机为主根，另一台核心为备根。

**根桥的特点**：**根桥上的所有端口都是指定端口，全部转发，没有阻塞口。**

### 2.4 第 2 步：选根端口（每台非根交换机选一个）

**根端口 = 这台交换机上"去往根桥最近"的那个端口。**

比较顺序（从上往下，分出胜负就停）：

| 顺序 | 比较项 | 规则 |
|:--|:--|:--|
| ① | **到根桥的累计路径开销** (Root Path Cost) | 小的赢 |
| ② | **上游邻居的 Bridge ID** | 小的赢 |
| ③ | **上游邻居的 Port ID**（优先级 + 端口号） | 小的赢 |
| ④ | 本地 Port ID | 小的赢（极少用到，只在 EtherChannel 或自环时） |

**路径开销表（必背）**：

| 带宽 | STP (802.1D) Cost | RSTP/MST 长格式 Cost |
|:--|:--|:--|
| 10 Mbps | 100 | 2,000,000 |
| 100 Mbps | **19** | 200,000 |
| 1 Gbps | **4** | 20,000 |
| 10 Gbps | **2** | 2,000 |
| 100 Gbps | 1 | 200 |

> **关键**：**开销是累加"入方向"的**，也就是接收 BPDU 的那个端口的开销。计算时从自己往根桥方向逐段累加。

**开销计算示例**：

```
        [SW1 = 根桥]
        /            \
   1Gbps(4)        100Mbps(19)
      /                  \
  [SW2]                [SW3]
      \                  /
      1Gbps(4)      1Gbps(4)
         \            /
          \          /
           [  SW4  ]
```

**SW4 的根端口是哪个？**

- 经 SW2 的路径：SW4→SW2 (4) + SW2→SW1 (4) = **8**
- 经 SW3 的路径：SW4→SW3 (4) + SW3→SW1 (19) = **23**

**8 < 23 → SW4 朝向 SW2 的端口是根端口。**

### 2.5 第 3 步：选指定端口（每个网段选一个）

**网段（Segment）= 一条链路，两端各有一个端口。这两个端口里必须选出一个当指定端口，另一个被阻塞。**

比较顺序：

| 顺序 | 比较项 | 规则 |
|:--|:--|:--|
| ① | 该端口所在交换机**到根桥的路径开销** | 小的赢 |
| ② | 该交换机的 **Bridge ID** | 小的赢 |
| ③ | 该端口的 **Port ID** | 小的赢 |

**三条捷径规则**（能省 90% 的计算量）：
1. **根桥的所有端口都是指定端口**（因为它到根的开销是 0，无人能敌）
2. **根端口一定不是指定端口**（一个端口不能既是根口又是指定口）
3. **每个网段有且仅有一个指定端口**，另一端要么是根端口，要么被阻塞

### 2.6 完整推导实战

```
                    [SW1] 优先级 32768, MAC 0000.0000.1111
                    /              \
          Gi0/1 ┌──┘ 1Gbps          └──┐ Gi0/2  1Gbps
                │                       │
    Gi0/1 ┌─────┴──┐             ┌──────┴─────┐ Gi0/1
          │  SW2   │             │    SW3     │
          │ 32768  │             │   32768    │
          │ ..2222 │             │   ..3333   │
          └────┬───┘             └────┬───────┘
          Gi0/2│      1Gbps           │Gi0/2
               └──────────────────────┘
```

**第 1 步：选根桥**
三台优先级都是 32768，比 MAC：`0000.0000.1111` 最小 → **SW1 是根桥**。
→ SW1 的 Gi0/1、Gi0/2 都是**指定端口**（规则：根桥所有口都是 DP）。

**第 2 步：选根端口**
- **SW2**：
  - 经 Gi0/1 直连 SW1：开销 = 4
  - 经 Gi0/2 → SW3 → SW1：开销 = 4 + 4 = 8
  - 4 < 8 → **SW2 的 Gi0/1 是根端口 (RP)**
- **SW3**：同理 → **SW3 的 Gi0/1 是根端口 (RP)**

**第 3 步：选指定端口**

链路 SW1–SW2：SW1 的 Gi0/1 已是 DP，SW2 的 Gi0/1 是 RP。✓ 完成
链路 SW1–SW3：SW1 的 Gi0/2 已是 DP，SW3 的 Gi0/1 是 RP。✓ 完成

**链路 SW2–SW3（关键）**：两端是 SW2-Gi0/2 和 SW3-Gi0/2，都不是 RP，必须比一比。
- ① 到根开销：SW2 = 4，SW3 = 4 → **平**
- ② Bridge ID：SW2 = `32768.0000.0000.2222`，SW3 = `32768.0000.0000.3333` → **SW2 小，SW2 赢**

→ **SW2 的 Gi0/2 是指定端口 (DP)**
→ **SW3 的 Gi0/2 被阻塞 (Blocking)** ← 就是这个端口断开了环路

**最终拓扑**：
```
                    [SW1] 根桥
              DP  /          \  DP
                 /            \
            RP  /              \  RP
          [SW2] ──── DP ──✕── BLK ──── [SW3]
                         ↑
                  逻辑上断开，形成一棵树
```

**验证一下有没有环**：从任一点出发，沿转发路径走，不可能回到起点。✓

### 2.7 端口状态与收敛时间

**经典 STP (802.1D) 的五个状态**：

| 状态 | 时长 | 学 MAC？ | 转发数据？ | 收发 BPDU？ | 说明 |
|:--|:--|:--|:--|:--|:--|
| **Disabled** | — | ❌ | ❌ | ❌ | 端口被 shutdown |
| **Blocking** | 20s (Max Age) | ❌ | ❌ | **只收** | 监听 BPDU，防环 |
| **Listening** | 15s (Forward Delay) | ❌ | ❌ | 收+发 | 参与选举 |
| **Learning** | 15s (Forward Delay) | ✅ | ❌ | 收+发 | 建 MAC 表但不转发 |
| **Forwarding** | — | ✅ | ✅ | 收+发 | 正常工作 |

**收敛时间**：
- 端口从 Blocking 到 Forwarding：**20 + 15 + 15 = 50 秒**
- 端口直接从 down 到 Forwarding（比如插上新线）：**15 + 15 = 30 秒**

> **这 30–50 秒就是 STP 最大的痛点**。用户拔了网线重插，要等 30 秒才通；主链路断了，要 50 秒才切换——期间业务全断。**这直接催生了 RSTP（毫秒级收敛）和 PortFast。**

**默认定时器**（可调但不建议手工改）：

| 定时器 | 默认值 | 作用 |
|:--|:--|:--|
| Hello Time | 2 秒 | 根桥发 BPDU 的间隔 |
| Max Age | 20 秒 | 没收到 BPDU 多久后认为链路失效（= 10 个 Hello） |
| Forward Delay | 15 秒 | Listening 和 Learning 各停留的时长 |

### 2.8 PortFast、BPDU Guard、Root Guard（实战三件套）

#### PortFast

接终端的端口，让它**跳过 Listening/Learning，直接进入 Forwarding**。

```cisco
SW1(config-if)# spanning-tree portfast
! 或全局给所有 access 口开启
SW1(config)# spanning-tree portfast default
```

**解决的问题**：PC 开机时 DHCP 请求发得很早，如果端口还在 Listening（前 30 秒），DHCP Discover 会被丢弃，导致拿不到 IP（拿到 `169.254.x.x`）。

> ⚠️ **PortFast 只能用在接终端的端口**。如果误配在连交换机的口上，插上线的瞬间就直接转发，**会造成瞬时环路**。所以必须配套 BPDU Guard。

#### BPDU Guard

PortFast 端口一旦**收到 BPDU**（说明有人接了交换机），立即 **err-disable** 关闭该端口。

```cisco
SW1(config-if)# spanning-tree bpduguard enable
! 或全局：所有 portfast 口自动启用
SW1(config)# spanning-tree portfast bpduguard default
```

**恢复方式**：
```cisco
! 手工恢复
SW1(config-if)# shutdown
SW1(config-if)# no shutdown

! 自动恢复（推荐配上）
SW1(config)# errdisable recovery cause bpduguard
SW1(config)# errdisable recovery interval 300
```

**这是防止"员工私接小交换机造成全楼环路"最有效的一招。**

#### Root Guard

防止某个方向的设备成为根桥。配在**朝向下游（接入层）的端口**上：

```cisco
SW1(config-if)# spanning-tree guard root
```

如果该端口收到了"优于当前根桥"的 BPDU，端口进入 **root-inconsistent** 状态（相当于阻塞），直到不再收到为止。

**用途**：防止别人接一台优先级极低的交换机进来，把根桥抢走，导致全网流量绕路。

#### Loop Guard

防止**单向链路故障**（比如光纤只有一个方向断了）导致阻塞端口错误地转为转发。

```cisco
SW1(config-if)# spanning-tree guard loop
! 或全局
SW1(config)# spanning-tree loopguard default
```

**原理**：如果一个非指定端口**突然收不到 BPDU 了**，正常 STP 会认为"上游没了"，把它转为转发（可能成环）。Loop Guard 会把它置为 `loop-inconsistent` 阻塞状态，更安全。

**三件套对比**：

| 特性 | 配在哪 | 防什么 | 触发后状态 |
|:--|:--|:--|:--|
| **PortFast** | 接终端口 | 收敛慢 | — |
| **BPDU Guard** | 接终端口 | 私接交换机 | err-disable |
| **Root Guard** | 朝下游口 | 根桥被抢 | root-inconsistent |
| **Loop Guard** | 根端口/阻塞口 | 单向链路成环 | loop-inconsistent |
| **BPDU Filter** | 接终端口 | 不发 BPDU（慎用） | — |

> ⚠️ **BPDU Filter 慎用**：全局配置和接口配置的行为不同，接口下配置会**完全不收不发 BPDU**，等于把这个口从 STP 里摘掉，极易造成环路。除非你非常清楚在做什么，否则不要用。

### 2.9 STP 的几个版本

| 版本 | 标准 | 特点 | 收敛 |
|:--|:--|:--|:--|
| **STP** | 802.1D | 全网一棵树，所有 VLAN 共用 | 30–50 秒 |
| **PVST+** | Cisco 私有 | **每个 VLAN 一棵树**，可做负载分担 | 30–50 秒 |
| **RSTP** | 802.1w | 快速收敛，端口角色重新设计 | **< 1 秒** |
| **Rapid-PVST+** | Cisco 私有 | RSTP + 每 VLAN 一棵树（**Cisco 默认**） | < 1 秒 |
| **MST/MSTP** | 802.1s | **多个 VLAN 映射到一个实例**，省资源 | < 1 秒 |

**为什么需要 PVST+（每 VLAN 一棵树）**：

```
单棵树（802.1D）：            PVST+（每 VLAN 一棵树）：
                             
  [SW1]                        VLAN10: SW1 为根 → 左边链路转发
  /    \                       VLAN20: SW2 为根 → 右边链路转发
 ✓      ✕(阻塞)                
[SW2]  [SW3]                   两条链路都在用，负载分担！
                             
右边链路完全闲置，浪费 50% 带宽
```

**为什么需要 MST**：PVST+ 每个 VLAN 跑一个 STP 实例，1000 个 VLAN 就是 1000 个实例，交换机 CPU 扛不住。MST 把多个 VLAN 映射到少数几个实例（比如 VLAN 1-500 → 实例 1，VLAN 501-1000 → 实例 2），既保留负载分担能力，又大幅节省资源。

**这三个版本的演进逻辑是 ENCOR 的重点，详见 [ENCOR 第 4 章](../02-ENCOR-350-401/04-交换进阶-STP进阶与排障.md)。**

---

## ③ 配置命令

### Cisco

```cisco
! ═══ 选择 STP 模式 ═══
SW1(config)# spanning-tree mode pvst              ! PVST+
SW1(config)# spanning-tree mode rapid-pvst        ! Rapid-PVST+（推荐/默认）
SW1(config)# spanning-tree mode mst               ! MST

! ═══ 手工指定根桥（★ 生产必配）═══
! 方法 1：直接设优先级（必须是 4096 的倍数）
SW1(config)# spanning-tree vlan 10 priority 4096      ! 主根
SW2(config)# spanning-tree vlan 10 priority 8192      ! 备根

! 方法 2：用宏命令（更省事，IOS 自动算合适的值）
SW1(config)# spanning-tree vlan 10 root primary
SW2(config)# spanning-tree vlan 10 root secondary

! 负载分担：不同 VLAN 用不同根桥
SW1(config)# spanning-tree vlan 10,30 root primary
SW1(config)# spanning-tree vlan 20,40 root secondary
SW2(config)# spanning-tree vlan 20,40 root primary
SW2(config)# spanning-tree vlan 10,30 root secondary

! ═══ 调整开销和端口优先级（影响选路）═══
SW1(config-if)# spanning-tree cost 10                 ! 改路径开销
SW1(config-if)# spanning-tree port-priority 64        ! 改端口优先级（默认128，步长16）
SW1(config-if)# spanning-tree vlan 10 cost 10         ! 只改某个 VLAN 的

! ═══ 实战三件套 ═══
! 接终端端口
SW1(config-if)# spanning-tree portfast
SW1(config-if)# spanning-tree bpduguard enable

! 全局默认（推荐）
SW1(config)# spanning-tree portfast default
SW1(config)# spanning-tree portfast bpduguard default

! 朝向下游的端口
SW1(config-if)# spanning-tree guard root

! 阻塞端口/根端口防单向链路
SW1(config)# spanning-tree loopguard default

! err-disable 自动恢复
SW1(config)# errdisable recovery cause bpduguard
SW1(config)# errdisable recovery cause link-flap
SW1(config)# errdisable recovery interval 300

! ═══ 查看 ═══
SW1# show spanning-tree                          ! 全部 VLAN
SW1# show spanning-tree vlan 10                  ! 指定 VLAN（最常用）
SW1# show spanning-tree summary                  ! 摘要，看有几个口阻塞
SW1# show spanning-tree root                     ! 各 VLAN 的根桥
SW1# show spanning-tree blockedports             ! 哪些口被阻塞
SW1# show spanning-tree interface Gi0/1 detail
SW1# show spanning-tree inconsistentports        ! guard 触发的异常口
SW1# show errdisable recovery
```

### 输出解读

```cisco
SW1# show spanning-tree vlan 10

VLAN0010
  Spanning tree enabled protocol rstp
  Root ID    Priority    4106                      ← 4096 + VLAN 10
             Address     0000.0000.1111
             This bridge is the root                ← 我就是根桥
             Hello Time 2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    4106  (priority 4096 sys-id-ext 10)
             Address     0000.0000.1111

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------
Gi0/1            Desg FWD 4         128.1    P2p
Gi0/2            Desg FWD 4         128.2    P2p
Gi0/3            Desg FWD 19        128.3    P2p
                  ↑    ↑
                 角色  状态
```

**Role（角色）**：`Root`=根端口 / `Desg`=指定端口 / `Altn`=替代端口(阻塞) / `Back`=备份端口
**Sts（状态）**：`FWD`=转发 / `BLK`=阻塞 / `LRN`=学习 / `LIS`=侦听

### 三厂商对照

| 目的 | Cisco | H3C | 华为 |
|:--|:--|:--|:--|
| 开启 STP | 默认开启 | `stp global enable` | `stp enable` |
| 模式 | `spanning-tree mode rapid-pvst` | `stp mode rstp` / `stp mode mstp` | `stp mode rstp` / `stp mode mstp` |
| 设根桥 | `spanning-tree vlan 10 root primary` | `stp instance 0 root primary` | `stp instance 0 root primary` |
| 设优先级 | `spanning-tree vlan 10 priority 4096` | `stp instance 0 priority 4096` | `stp instance 0 priority 4096` |
| 边缘端口 | `spanning-tree portfast` | `stp edged-port` | `stp edged-port enable` |
| BPDU 保护 | `spanning-tree bpduguard enable` | `stp bpdu-protection` | `stp bpdu-protection` |
| 根保护 | `spanning-tree guard root` | `stp root-protection` | `stp root-protection` |
| 环路保护 | `spanning-tree guard loop` | `stp loop-protection` | `stp loop-protection` |
| 改开销 | `spanning-tree cost 10` | `stp cost 10` | `stp cost 10` |
| 查看 | `show spanning-tree` | `display stp brief` | `display stp brief` |

> **重要差异**：**Cisco 默认是每 VLAN 一棵树（Rapid-PVST+），H3C/华为默认是 MSTP（所有 VLAN 在实例 0）**。混合组网时，Cisco 的 PVST+ 和 H3C/华为的 MSTP **无法直接互通**——需要把 Cisco 也改成 MST 模式，并保证 MST 的 region name、revision、VLAN-实例映射三者完全一致。这是国内混合组网最经典的坑。

---

## ④ 配套实验：手工推导 + 设备验证

**拓扑**（三角环）：
```
                 ┌────────┐
                 │  SW1   │
                 └─┬────┬─┘
              Gi0/1│    │Gi0/2
                   │    │
        ┌──────────┘    └──────────┐
        │Gi0/1                Gi0/1│
   ┌────┴───┐   Gi0/2  Gi0/2  ┌────┴───┐
   │  SW2   ├─────────────────┤  SW3   │
   └────────┘                 └────────┘
```
所有链路 1Gbps。

### Step 1：先在纸上推导（不许看设备）

给定：
- SW1: MAC `0000.0000.1111`
- SW2: MAC `0000.0000.2222`
- SW3: MAC `0000.0000.3333`
- 优先级全部默认 32768

**填表**：

| 交换机 | 端口 | 角色 (Root/Desg/Altn) | 状态 (FWD/BLK) |
|:--|:--|:--|:--|
| SW1 | Gi0/1 | ? | ? |
| SW1 | Gi0/2 | ? | ? |
| SW2 | Gi0/1 | ? | ? |
| SW2 | Gi0/2 | ? | ? |
| SW3 | Gi0/1 | ? | ? |
| SW3 | Gi0/2 | ? | ? |

<details><summary>推导答案</summary>

**第 1 步 选根桥**：优先级都是 32768，比 MAC → `...1111` 最小 → **SW1 是根桥**
→ SW1 的 Gi0/1、Gi0/2 **都是指定端口 (Desg/FWD)**

**第 2 步 选根端口**：
- SW2：Gi0/1 直连根桥开销 4；Gi0/2 经 SW3 开销 4+4=8 → **Gi0/1 是 Root Port**
- SW3：同理 → **Gi0/1 是 Root Port**

**第 3 步 选指定端口（SW2–SW3 链路）**：
- 到根开销：SW2=4，SW3=4 → 平
- Bridge ID：SW2 (`...2222`) < SW3 (`...3333`) → **SW2 赢**
- → SW2-Gi0/2 = **Desg/FWD**，SW3-Gi0/2 = **Altn/BLK**

| 交换机 | 端口 | 角色 | 状态 |
|:--|:--|:--|:--|
| SW1 | Gi0/1 | Desg | FWD |
| SW1 | Gi0/2 | Desg | FWD |
| SW2 | Gi0/1 | **Root** | FWD |
| SW2 | Gi0/2 | Desg | FWD |
| SW3 | Gi0/1 | **Root** | FWD |
| SW3 | Gi0/2 | **Altn** | **BLK** ← 环路在这里被断开 |
</details>

### Step 2：在设备上验证

```cisco
! 三台都配好 VLAN 10 和 Trunk 后
SW3# show spanning-tree vlan 10

Interface   Role Sts Cost   Prio.Nbr Type
----------- ---- --- ------ -------- ------
Gi0/1       Root FWD 4      128.1    P2p
Gi0/2       Altn BLK 4      128.2    P2p      ← 与推导一致 ✓
```

```cisco
SW1# show spanning-tree vlan 10 | include root
             This bridge is the root                ← SW1 是根 ✓
```

### Step 3：手工指定根桥（生产必做）

假设我们希望 SW2 做主根（它是核心交换机）：

```cisco
SW2(config)# spanning-tree vlan 10 root primary
SW3(config)# spanning-tree vlan 10 root secondary
```

**验证变化**：
```cisco
SW2# show spanning-tree vlan 10 | include priority
Bridge ID  Priority  24586  (priority 24576 sys-id-ext 10)
                              ↑ root primary 自动设成了 24576

SW1# show spanning-tree vlan 10
! SW1 现在不再是根，它会出现 Root Port
```

**重新推导**：根桥换成 SW2 后，阻塞口会移到哪里？自己推一遍，再用设备验证。

<details><summary>提示</summary>

SW2 成为根桥后：
- SW2 的两个口都是 Desg/FWD
- SW1 的 Gi0/1（直连 SW2）是 Root Port
- SW3 的 Gi0/2（直连 SW2）是 Root Port
- SW1–SW3 链路：两边到根开销都是 4，比 Bridge ID：SW1(`...1111`) < SW3(`...3333`) → SW1 赢
- → **SW3 的 Gi0/1 被阻塞**

阻塞点从 SW3-Gi0/2 移到了 SW3-Gi0/1。
</details>

### Step 4：负载分担实验

```cisco
! VLAN 10 走 SW2 这边，VLAN 20 走 SW3 这边
SW2(config)# spanning-tree vlan 10 root primary
SW2(config)# spanning-tree vlan 20 root secondary
SW3(config)# spanning-tree vlan 20 root primary
SW3(config)# spanning-tree vlan 10 root secondary
```

**验证**：
```cisco
SW1# show spanning-tree vlan 10 | include Root|Altn
SW1# show spanning-tree vlan 20 | include Root|Altn
! 两个 VLAN 的阻塞端口应该不同 → 两条链路都在被使用
```

这就是 **PVST+ 相比 802.1D 的核心价值**：同样的物理拓扑，带宽利用率翻倍。

### Step 5：测收敛时间

```cisco
! 在 PC 上持续 ping 对端，然后断开主链路
SW2(config)# interface Gi0/1
SW2(config-if)# shutdown

! 观察 ping 丢包数量
! Rapid-PVST+：丢 1-3 个包（< 1 秒）
! 改成 pvst 模式再测：丢 30-50 个包（30-50 秒）
```

```cisco
! 切换模式对比
SW1(config)# spanning-tree mode pvst        ! 慢速
SW1(config)# spanning-tree mode rapid-pvst  ! 快速
```

**这个对比会让你切身体会 RSTP 的价值。**

---

## ⑤ 排障思路

| 症状 | 怀疑点 | 验证命令 | 根因 |
|:--|:--|:--|:--|
| CPU 100%、端口灯狂闪 | 环路 | `show processes cpu sorted`、`show spanning-tree` | STP 被关 / BPDU 被过滤 / 单向链路 |
| MAC 表项在端口间跳变 | 环路 | 反复 `show mac address-table address X` | 同上 |
| 流量走了很绕的路径 | 根桥位置不对 | `show spanning-tree root` | 没手工指定根桥，选出了接入层设备 |
| 某端口一直 `BKN*` / err-disable | BPDU Guard | `show interfaces status err-disabled` | PortFast 口收到了 BPDU（私接交换机） |
| 端口 `root-inconsistent` | Root Guard | `show spanning-tree inconsistentports` | 下游有优先级更高的交换机 |
| 端口 `loop-inconsistent` | Loop Guard | 同上 | 单向链路，收不到 BPDU |
| PC 开机拿不到 DHCP | 收敛慢 | `show spanning-tree interface X` | 没配 PortFast，端口 30 秒才转发 |
| 混合厂商组网 STP 不收敛 | 协议不兼容 | 对比两边 STP 模式 | Cisco PVST+ 与 H3C/华为 MSTP 不互通 |
| 切换耗时 30-50 秒 | 用的是老 STP | `show spanning-tree \| include protocol` | 应改成 rapid-pvst 或 mst |

### 环路应急处理流程

```
1. 止血：找到疑似环路的端口，先 shutdown
   SW1(config-if)# shutdown
   
2. 确认：
   SW1# show spanning-tree summary        ← 看有没有端口阻塞
   SW1# show mac address-table | include <震荡的MAC>
   SW1# show interfaces | include (rate|drops)
   
3. 定位：
   SW1# show spanning-tree inconsistentports
   SW1# show interfaces status err-disabled
   SW1# show cdp neighbors                ← 看端口对面是什么设备
   
4. 根因：
   - 有人私接了小交换机 → BPDU Guard 应该拦住的，检查是否配了
   - 光纤单向故障 → Loop Guard
   - STP 被误关 → show run | include spanning-tree
   - 混合厂商模式不匹配 → 统一改 MST
   
5. 加固：
   - 所有接入口：portfast + bpduguard
   - 全局：loopguard default
   - 手工指定根桥和备根
   - storm-control
```

### 定位根桥的方法

不知道网络里根桥在哪？从任意一台交换机开始：

```cisco
SW-X# show spanning-tree vlan 10 | include Root ID -A 3
Root ID    Priority    32778
           Address     0000.0000.1111       ← 根桥的 MAC
           Cost        8
           Port        1 (GigabitEthernet0/1)  ← 根端口，往这个方向走
```

沿着根端口的方向，用 `show cdp neighbors` 找到下一跳，登上去再看，直到某台显示 `This bridge is the root`。

---

## ⑥ 考点提示 + 自测题

### 考点

- **三步选举的推导**是核心，必须能手算。
- **开销值 4 / 19 / 2** 必背。
- **默认收敛时间 30/50 秒**及其构成（20 Max Age + 15 + 15）。
- **PortFast / BPDU Guard / Root Guard / Loop Guard** 的用途和配置位置。
- **PVST+ 负载分担**的实现方法。
- **优先级必须是 4096 的倍数**及其原因（扩展系统 ID 占了 12 位）。

### 自测题

**1.** 网络里四台交换机优先级都是默认 32768，MAC 分别是 `aaaa.1111`、`aaaa.2222`、`bbbb.1111`、`0000.9999`。谁是根桥？

<details><summary>答案</summary>

**MAC 为 `0000.9999` 的那台。**

优先级都相同（32768），比较 MAC 地址，**数值最小的赢**。MAC 是十六进制，逐字节比较：
- `0000.9999` 的第一个字节是 `00`
- 其余三台的第一个字节是 `aa` 或 `bb`
- `00` < `aa` < `bb`

所以 `0000.9999` 最小，它是根桥。

**实践警示**：MAC 地址通常和设备生产批次相关，**越老的设备 MAC 往往越小**。所以默认情况下，网络会选出一台老旧的、可能位于接入层的交换机做根桥，导致所有跨交换机流量都要绕到那台老设备上，性能极差。

**这就是"必须手工指定根桥"的原因**：
```cisco
CoreSW1(config)# spanning-tree vlan 1-4094 root primary
CoreSW2(config)# spanning-tree vlan 1-4094 root secondary
```
</details>

**2.** 为什么 STP 的桥优先级必须是 4096 的倍数？

<details><summary>答案</summary>

因为 Bridge ID 的 16 位"优先级字段"被拆成了两部分：

```
┌────────────────┬──────────────────────┐
│ 桥优先级 (4 bit) │  扩展系统 ID (12 bit)  │
└────────────────┴──────────────────────┘
```

**低 12 位被"扩展系统 ID"占用**，用来存放 **VLAN ID**（这是 PVST+ 为了让每个 VLAN 能有独立的 STP 实例而做的设计，见 IEEE 802.1t 修订）。

管理员实际能配置的只有**高 4 位**，所以最小步长是 2^12 = **4096**。可选值只有：
`0, 4096, 8192, 12288, 16384, 20480, 24576, 28672, 32768（默认）, ..., 61440`

**这也解释了 `show spanning-tree` 里为什么优先级显示成奇怪的数字**：
```
Bridge ID  Priority  32778  (priority 32768 sys-id-ext 10)
                              ↑ 实际优先级      ↑ VLAN ID
32768 + 10 = 32778
```

**顺带一提**：`root primary` 宏命令会自动把优先级设为 24576（如果当前根桥优先级 ≥ 24576），`root secondary` 设为 28672。
</details>

**3.** 下图中 SW3 的根端口是哪个？（数字是链路带宽）

```
        [SW1 根桥]
       /          \
   1Gbps        100Mbps
     /              \
  [SW2]            [SW3]
     \              /
      1Gbps    1Gbps
        \        /
         \      /
        （SW2-SW3 直连）
```

<details><summary>答案</summary>

**SW3 朝向 SW2 的端口是根端口。**

计算两条路径的累计开销：
- **SW3 → SW1 直连**（100Mbps）：开销 = **19**
- **SW3 → SW2 → SW1**（1Gbps + 1Gbps）：开销 = 4 + 4 = **8**

**8 < 19**，所以 SW3 认为"经过 SW2 到根桥更近"，选朝向 SW2 的口作为根端口。

**这道题的意义**：它说明 **STP 的最优路径是按开销算的，不是按跳数**。绕一跳走千兆，比直连走百兆更优。这跟 OSPF 按带宽算 cost 是同一个思路。

**实践提醒**：如果你希望流量走直连的那条百兆链路（比如那是专线），可以手工调整开销：
```cisco
SW3(config-if)# spanning-tree cost 3      ! 改成比 8 小
```
但更好的做法是**升级链路带宽**，而不是骗 STP。
</details>

**4.** 一个 PortFast 端口收到了 BPDU，会发生什么？为什么这个机制很重要？

<details><summary>答案</summary>

**如果配了 BPDU Guard**：端口立即进入 **err-disable** 状态（被关闭），并打印日志：
```
%SPANTREE-2-BLOCK_BPDUGUARD: Received BPDU on port Gi1/0/5 with BPDU Guard enabled. Disabling port.
%PM-4-ERR_DISABLE: bpduguard error detected on Gi1/0/5, putting Gi1/0/5 in err-disable state
```

**如果没配 BPDU Guard**：端口会**退出 PortFast 状态**，转为正常的 STP 端口，重新走 Listening → Learning → Forwarding 流程（30 秒）。

**为什么重要**：

PortFast 端口本应接终端（PC、打印机），终端不会发 BPDU。**收到 BPDU 说明有人接了一台交换机**——可能是：
1. 员工私自接了小交换机/傻瓜 HUB 来扩展端口
2. 有人误把两个墙上网口用网线连了起来
3. 恶意的 STP 攻击（发送优先级极高的 BPDU 抢根桥）

前两种情况**极易造成二层环路**。而 PortFast 端口是**直接进入转发状态的**，一旦成环，广播风暴会在毫秒内爆发。BPDU Guard 就是这道防线。

**配置模板（接入端口标准配置）**：
```cisco
SW1(config)# spanning-tree portfast default
SW1(config)# spanning-tree portfast bpduguard default
SW1(config)# errdisable recovery cause bpduguard
SW1(config)# errdisable recovery interval 300
```
最后两条让端口在 5 分钟后自动尝试恢复，避免每次都要人工去 `shut/no shut`。

> **实战价值**：在我见过的企业网络事故里，"员工私接交换机导致全楼断网"是最常见的一类。BPDU Guard 是成本最低、效果最好的防护，**所有接入端口都应该配**。
</details>

**5.** Cisco 交换机和 H3C 交换机对接，STP 一直不收敛、端口反复震荡。可能是什么原因？

<details><summary>答案</summary>

**最可能是 STP 模式不兼容**。

- **Cisco 默认是 Rapid-PVST+**（每个 VLAN 一棵独立的树，Cisco 私有协议）
- **H3C/华为默认是 MSTP**（802.1s，多个 VLAN 映射到少数实例）

**PVST+ 是 Cisco 私有的**，它给每个 VLAN 发独立的 BPDU（用 Cisco 私有的 SSTP 格式，组播地址 `01:00:0C:CC:CC:CD`）。H3C 设备不认识这种 BPDU，双方无法正确协商，导致拓扑计算混乱。

**解决方案：统一改成 MST（802.1s 是标准协议，各家都支持）**

```cisco
! Cisco 侧
SW-Cisco(config)# spanning-tree mode mst
SW-Cisco(config)# spanning-tree mst configuration
SW-Cisco(config-mst)#  name REGION-A
SW-Cisco(config-mst)#  revision 1
SW-Cisco(config-mst)#  instance 1 vlan 10,20
SW-Cisco(config-mst)#  instance 2 vlan 30,40
```

```
# H3C 侧
[SW-H3C] stp mode mstp
[SW-H3C] stp region-configuration
[SW-H3C-mst-region] region-name REGION-A
[SW-H3C-mst-region] revision-level 1
[SW-H3C-mst-region] instance 1 vlan 10 20
[SW-H3C-mst-region] instance 2 vlan 30 40
[SW-H3C-mst-region] active region-configuration     ← ★ H3C 必须敲这句才生效
```

**三个必须完全一致的参数**（否则会被认为是不同的 MST 域，退化成外部 STP）：
1. **Region Name**（域名）
2. **Revision Level**（修订号）
3. **VLAN → Instance 的映射关系**

**验证**：
```cisco
SW-Cisco# show spanning-tree mst configuration      ! 看配置摘要（digest）
SW-Cisco# show spanning-tree mst configuration digest
```
```
[SW-H3C] display stp region-configuration
```
**两边的 digest 值必须相同**，不同说明映射关系有差异。

> **H3C 的坑**：修改 region-configuration 后必须执行 `active region-configuration` 才生效。忘了这句是国内混合组网最常见的失败原因。
</details>

---

**上一章** ← [01 VLAN 与 Trunk](01-VLAN与Trunk.md) ｜ **下一章** → [03 EtherChannel 链路聚合](03-EtherChannel链路聚合.md)
