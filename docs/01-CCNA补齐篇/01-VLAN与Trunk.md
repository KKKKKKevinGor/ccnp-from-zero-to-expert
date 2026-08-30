# 01 · VLAN 与 Trunk

## ① 这章解决什么问题

一栋楼三层办公，300 台电脑，全接在交换机上。现在有几个麻烦：

1. **广播泛滥**：任何一台机器发广播（ARP、DHCP Discover、NetBIOS），300 台机器全都要处理。
2. **没有隔离**：财务部的机器和访客的机器在同一个网里，谁都能扫到谁。
3. **物理位置绑死**：财务部要分散在三个楼层办公，但他们要在同一个网段——难道重新布线？

**VLAN 就是这三个问题的答案**：它在一台（或一组）交换机上，用软件划分出多个互相隔离的逻辑广播域，跟物理位置无关。

理解 VLAN 还有一层更深的意义：**"用逻辑标识把共享的物理设施切分成互相隔离的多份"** 这个思想，后面会以 VRF（三层版本）、VXLAN（跨三层的二层）、SD-Access（策略化版本）的形式反复出现。VLAN 是这条线的起点。

---

## ② 原理讲透

### 2.1 VLAN 做了什么

**一句话：VLAN = 一个广播域。**

```
没有 VLAN：                          有 VLAN：
┌──────────────────────┐            ┌──────────────────────┐
│      交换机           │            │      交换机           │
│  P1 P2 P3 P4 P5 P6   │            │  P1 P2 │ P3 P4 │ P5 P6│
│  └──────┬───────┘    │            │ VLAN10 │VLAN20 │VLAN30│
│    一个广播域         │            │  ↑独立   ↑独立   ↑独立 │
└──────────────────────┘            └──────────────────────┘
 广播到所有口                          广播只在本 VLAN 内
```

**关键规则**：
- **不同 VLAN 之间二层不通**，必须经过三层设备（路由器 / 三层交换机）才能互访
- 交换机的 MAC 地址表是**按 VLAN 分开维护**的（同一个 MAC 可以在不同 VLAN 里出现）
- 每个 VLAN 通常对应一个 IP 网段（不是强制，但强烈建议 1:1 对应，否则排障噩梦）

### 2.2 VLAN 编号范围

| 范围 | 名称 | 说明 |
|:--|:--|:--|
| 0, 4095 | 保留 | 不可用 |
| **1** | 默认 VLAN | 所有端口出厂默认在这里。**生产环境不要用** |
| 2 – 1001 | 正常范围 (Normal) | 存在 VLAN 数据库，可通过 VTP 传播 |
| 1002 – 1005 | 保留 | FDDI/Token Ring 遗留，不能删 |
| 1006 – 4094 | 扩展范围 (Extended) | 只存在 running-config，早期需 VTP transparent 模式 |

> **安全最佳实践**：VLAN 1 不用于承载任何业务流量，也不做 Native VLAN。原因见 2.6 节的 VLAN 跳跃攻击。

### 2.3 端口类型

| 类型 | 承载 | 用途 | 帧是否带标签 |
|:--|:--|:--|:--|
| **Access** | 单个 VLAN | 接终端（PC、打印机、摄像头） | **不带标签**（出接口时剥掉） |
| **Trunk** | 多个 VLAN | 交换机互联、接 AP、接虚拟化宿主机 | **带 802.1Q 标签**（Native VLAN 除外） |
| Voice VLAN | 数据 VLAN + 语音 VLAN | 接 IP 电话（电话再串接 PC） | 语音带标签，数据不带 |

### 2.4 802.1Q 标签（Trunk 的核心）

```
普通以太网帧：
┌────────┬───────┬──────┬──────────┬─────┐
│ 目的MAC │ 源MAC │ 类型 │   数据    │ FCS │
└────────┴───────┴──────┴──────────┴─────┘

加了 802.1Q 标签后（在源 MAC 之后插入 4 字节）：
┌────────┬───────┬══════════════┬──────┬──────────┬─────┐
│ 目的MAC │ 源MAC │ 802.1Q (4B)  │ 类型 │   数据    │ FCS │
└────────┴───────┴══════════════┴──────┴──────────┴─────┘
                        │
        ┌───────────────┴───────────────┐
        │ TPID(16) │ PCP(3) │ DEI(1) │ VID(12) │
        │  0x8100  │ 优先级  │  丢弃  │ VLAN ID │
        └──────────┴────────┴────────┴─────────┘
```

| 字段 | 位数 | 说明 |
|:--|:--|:--|
| **TPID** | 16 | 固定 `0x8100`，标识"这是个 VLAN 标签" |
| **PCP** | 3 | 优先级 0–7，**这就是 CoS（Class of Service）**，QoS 二层标记 |
| **DEI/CFI** | 1 | 丢弃资格标识 |
| **VID** | **12** | VLAN ID，2^12 = **4096**，这就是 VLAN 最多 4094 个的原因 |

> **两个重要推论**：
> 1. **加标签会让帧变长 4 字节**（1518 → 1522），所以交换机要支持 "baby giant" 帧。这也是 MTU 计算时要考虑的因素之一。
> 2. **VLAN 只有 12 位 = 最多 4094 个**，这在大型数据中心（多租户，每个租户要独立网段）根本不够用。**这正是 VXLAN 出现的直接原因**——VXLAN 的 VNI 是 24 位，1600 万个。这条线索请记住，ENCOR 讲 VXLAN 时会呼应。

### 2.5 Native VLAN（最容易出问题的地方）

Trunk 上有一个特殊的 VLAN，它的帧**不打标签**直接发送，这就是 **Native VLAN**（默认是 VLAN 1）。

**为什么要有它**：兼容不认识 802.1Q 的老设备（比如集线器、老网卡），让它们至少能收发一个 VLAN 的流量。

**不一致会怎样**：

```
SW1 (Native VLAN 1) ═══════ Trunk ═══════ SW2 (Native VLAN 99)
```

SW1 的 VLAN 1 流量不打标签发出 → SW2 收到无标签帧，按自己的 Native VLAN 处理 → **归入 VLAN 99**。

结果：**VLAN 1 和 VLAN 99 被无声地"焊"在了一起**，两个本该隔离的网段互通了。

**症状**：
- CDP/LLDP 会报错：`Native VLAN mismatch discovered on GigabitEthernet0/1 (1), with SW2 GigabitEthernet0/1 (99)`
- 两个 VLAN 的主机能互相 ping 通（本不该）
- STP 可能出现异常（不同 VLAN 的 BPDU 混在一起）

**排查**：
```cisco
SW1# show interfaces trunk
Port        Mode  Encapsulation  Status    Native vlan
Gi0/1       on    802.1q         trunking  1              ← 对比两端这一列
```

### 2.6 VLAN 跳跃攻击（安全考点）

**攻击方式一：Switch Spoofing（交换机欺骗）**

攻击者的网卡伪装成一台交换机，发送 DTP 协商帧。如果交换机端口是 `dynamic auto` 或 `dynamic desirable`（默认往往是 `dynamic auto`），就会**自动协商成 Trunk**，攻击者就能收到所有 VLAN 的流量。

**防御**：
```cisco
SW1(config-if)# switchport mode access      ! 明确指定为 access
SW1(config-if)# switchport nonegotiate      ! 关闭 DTP 协商
```

**攻击方式二：Double Tagging（双标签）**

```
攻击者发出：[外层标签 VLAN 1 (=Native)] [内层标签 VLAN 20] [数据]
                    ↓
第一台交换机：入口是 access VLAN 1，Trunk 的 Native 也是 VLAN 1
              → 发到 Trunk 时，Native VLAN 不打标签 → 剥掉外层标签
                    ↓
第二台交换机：收到的帧只剩 [VLAN 20 标签] [数据]
              → 转发进 VLAN 20！  攻击成功（单向）
```

**防御**（三管齐下）：
```cisco
! 1. Native VLAN 改成一个没人用的 VLAN
SW1(config-if)# switchport trunk native vlan 999

! 2. Native VLAN 也强制打标签
SW1(config)# vlan dot1q tag native

! 3. Trunk 上只放行必要的 VLAN
SW1(config-if)# switchport trunk allowed vlan 10,20,30
```

> **考点**：Double Tagging 是**单向攻击**（能发过去，回不来），因为回程流量只有一层标签。但对于 DoS 或注入攻击，单向就够了。

### 2.7 VLAN 间路由（Inter-VLAN Routing）

不同 VLAN 要互通，必须走三层。三种做法：

#### 方案 1：Router-on-a-Stick（单臂路由）

```
        Trunk (Gi0/0)
SW1 ═══════════════════ R1
                        ├─ Gi0/0.10  (VLAN 10 网关)
                        ├─ Gi0/0.20  (VLAN 20 网关)
                        └─ Gi0/0.30  (VLAN 30 网关)
```

```cisco
R1(config)# interface GigabitEthernet0/0
R1(config-if)# no shutdown                     ! 物理口必须 up，但不配 IP

R1(config)# interface GigabitEthernet0/0.10
R1(config-subif)# encapsulation dot1Q 10       ! 关键：指定 VLAN
R1(config-subif)# ip address 192.168.10.1 255.255.255.0

R1(config)# interface GigabitEthernet0/0.20
R1(config-subif)# encapsulation dot1Q 20
R1(config-subif)# ip address 192.168.20.1 255.255.255.0

! Native VLAN 的子接口需要特殊写法
R1(config)# interface GigabitEthernet0/0.99
R1(config-subif)# encapsulation dot1Q 99 native
R1(config-subif)# ip address 192.168.99.1 255.255.255.0
```

**优点**：便宜，一个物理口搞定。
**缺点**：**所有跨 VLAN 流量都要在这一根线上走两遍**（进来一遍出去一遍），带宽减半，是明显的瓶颈。只适合小规模。

#### 方案 2：三层交换机 + SVI（推荐）

```cisco
SW1(config)# ip routing                        ! 关键！默认关闭

SW1(config)# interface Vlan10
SW1(config-if)# ip address 192.168.10.1 255.255.255.0
SW1(config-if)# no shutdown

SW1(config)# interface Vlan20
SW1(config-if)# ip address 192.168.20.1 255.255.255.0
SW1(config-if)# no shutdown
```

**优点**：**硬件转发（ASIC）**，线速，没有瓶颈。现代企业网的标准做法。
**注意**：`ip routing` 不开的话，SVI 配了 IP 也不会路由——这是新手最常见的坑。

#### 方案 3：三层交换机的路由口（Routed Port）

```cisco
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# no switchport                  ! 关键：变成路由口
SW1(config-if)# ip address 10.0.0.1 255.255.255.252
```

用于交换机之间的三层互联（比如接入层到核心层），不需要 VLAN 和 STP。

**SVI 状态的三个条件**（考点）：SVI 要 up，必须同时满足：
1. VLAN 存在于 VLAN 数据库（`vlan 10` 已创建）
2. **该 VLAN 内至少有一个物理端口是 up 的**（或者配了 `no autostate`）
3. SVI 本身没有 shutdown

> 第 2 条最容易忽略。你配好了 `interface Vlan10` 和 IP，但 VLAN 10 里所有端口都没接线 → SVI 是 down 的，路由不生效。

---

## ③ 配置命令

### Cisco

```cisco
! ═══ 创建 VLAN ═══
SW1(config)# vlan 10
SW1(config-vlan)# name FINANCE
SW1(config-vlan)# exit
SW1(config)# vlan 20
SW1(config-vlan)# name SALES

! 批量创建
SW1(config)# vlan 10,20,30,40

! ═══ Access 口 ═══
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# switchport nonegotiate          ! 关 DTP，安全
SW1(config-if)# spanning-tree portfast          ! 终端口快速转发
SW1(config-if)# spanning-tree bpduguard enable  ! 收到 BPDU 就关口

! 批量配置
SW1(config)# interface range GigabitEthernet1/0/1-12
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 10
SW1(config-if-range)# spanning-tree portfast

! ═══ Trunk 口 ═══
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# switchport trunk encapsulation dot1q   ! 老平台需要，新平台默认
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk native vlan 999       ! 改掉默认的 VLAN 1
SW1(config-if)# switchport trunk allowed vlan 10,20,30 ! 只放行必要 VLAN
SW1(config-if)# switchport nonegotiate

! 增删允许的 VLAN（⚠️ 注意 add，不加 add 是覆盖！）
SW1(config-if)# switchport trunk allowed vlan add 40
SW1(config-if)# switchport trunk allowed vlan remove 30

! ═══ Voice VLAN ═══
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10       ! PC 的数据 VLAN
SW1(config-if)# switchport voice vlan 100       ! IP 电话的语音 VLAN

! ═══ 查看 ═══
SW1# show vlan brief
SW1# show vlan id 10
SW1# show interfaces trunk                      ! 最常用，看 Native/允许/活动 VLAN
SW1# show interfaces GigabitEthernet1/0/1 switchport
SW1# show interfaces status
SW1# show mac address-table vlan 10
```

> ⚠️ **`switchport trunk allowed vlan 40` vs `... allowed vlan add 40`**
> 前者是**覆盖**（只剩 VLAN 40，其他全断），后者是**追加**。在生产网络上敲错这个，能瞬间切断整栋楼的网络。**这是运维事故 Top 3 的命令。**

### 三厂商对照

| 目的 | Cisco | H3C | 华为 |
|:--|:--|:--|:--|
| 创建 VLAN | `vlan 10` | `vlan 10` | `vlan 10` |
| 命名 | `name FINANCE` | `name FINANCE` | `name FINANCE` |
| 设为 Access | `switchport mode access` | `port link-type access` | `port link-type access` |
| Access 划 VLAN | `switchport access vlan 10` | `port access vlan 10` | `port default vlan 10` |
| 设为 Trunk | `switchport mode trunk` | `port link-type trunk` | `port link-type trunk` |
| Trunk 放行 | `switchport trunk allowed vlan 10,20` | `port trunk permit vlan 10 20` | `port trunk allow-pass vlan 10 20` |
| Native VLAN | `switchport trunk native vlan 999` | `port trunk pvid vlan 999` | `port trunk pvid vlan 999` |
| 创建 SVI | `interface Vlan10` | `interface Vlan-interface 10` | `interface Vlanif 10` |
| 开启三层转发 | `ip routing` | 默认开启 | 默认开启 |
| 查 VLAN | `show vlan brief` | `display vlan brief` | `display vlan` |
| 查 Trunk | `show interfaces trunk` | `display port trunk` | `display port vlan` |

> **术语对照**：Cisco 叫 **Native VLAN**，H3C/华为叫 **PVID**（Port VLAN ID）。概念完全一样：这个 VLAN 的帧在该端口上不打标签。
>
> **重大差异**：**H3C/华为的三层交换机默认就开启路由转发**，不需要 `ip routing`。从 Cisco 转过来的人容易忘配这条；从国产设备转 Cisco 的人则容易踩这个坑（SVI 配好了但不通）。

---

## ④ 配套实验：三 VLAN 互通

**拓扑**：
```
   PC1(VLAN10)  PC2(VLAN20)          PC3(VLAN10)  PC4(VLAN30)
       │            │                     │            │
    Gi1/0/1      Gi1/0/2               Gi1/0/1      Gi1/0/2
       └────┬───────┘                     └────┬───────┘
         ┌──┴───┐        Trunk           ┌─────┴──┐
         │ SW1  │Gi1/0/24 ══════ Gi1/0/24│  SW2   │
         │(L3)  │                        │  (L2)  │
         └──────┘                        └────────┘
```
SW1 是三层交换机（做网关），SW2 是二层接入交换机。

| VLAN | 网段 | 网关 (SVI on SW1) |
|:--|:--|:--|
| 10 FINANCE | 192.168.10.0/24 | 192.168.10.1 |
| 20 SALES | 192.168.20.0/24 | 192.168.20.1 |
| 30 IT | 192.168.30.0/24 | 192.168.30.1 |
| 999 NATIVE-UNUSED | — | 不用，仅作 Native |

### Step 1：SW2（接入层）配置

```cisco
SW2(config)# vlan 10
SW2(config-vlan)# name FINANCE
SW2(config)# vlan 20
SW2(config-vlan)# name SALES
SW2(config)# vlan 30
SW2(config-vlan)# name IT
SW2(config)# vlan 999
SW2(config-vlan)# name NATIVE-UNUSED

! 接入端口
SW2(config)# interface GigabitEthernet1/0/1
SW2(config-if)# switchport mode access
SW2(config-if)# switchport access vlan 10
SW2(config-if)# switchport nonegotiate
SW2(config-if)# spanning-tree portfast
SW2(config-if)# spanning-tree bpduguard enable

SW2(config)# interface GigabitEthernet1/0/2
SW2(config-if)# switchport mode access
SW2(config-if)# switchport access vlan 30
SW2(config-if)# switchport nonegotiate
SW2(config-if)# spanning-tree portfast
SW2(config-if)# spanning-tree bpduguard enable

! 上联 Trunk
SW2(config)# interface GigabitEthernet1/0/24
SW2(config-if)# switchport trunk encapsulation dot1q
SW2(config-if)# switchport mode trunk
SW2(config-if)# switchport trunk native vlan 999
SW2(config-if)# switchport trunk allowed vlan 10,20,30
SW2(config-if)# switchport nonegotiate
```

### Step 2：SW1（三层核心）配置

```cisco
SW1(config)# ip routing                          ! ★ 关键，别忘了

SW1(config)# vlan 10,20,30,999

! 接入端口（同 SW2 写法，略）

! 下联 Trunk —— 参数必须和 SW2 完全一致
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# switchport trunk encapsulation dot1q
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk native vlan 999
SW1(config-if)# switchport trunk allowed vlan 10,20,30
SW1(config-if)# switchport nonegotiate

! 网关 SVI
SW1(config)# interface Vlan10
SW1(config-if)# ip address 192.168.10.1 255.255.255.0
SW1(config-if)# no shutdown

SW1(config)# interface Vlan20
SW1(config-if)# ip address 192.168.20.1 255.255.255.0
SW1(config-if)# no shutdown

SW1(config)# interface Vlan30
SW1(config-if)# ip address 192.168.30.1 255.255.255.0
SW1(config-if)# no shutdown
```

### Step 3：验证

```cisco
! ── 检查 VLAN 与端口归属 ──
SW1# show vlan brief
VLAN Name          Status    Ports
---- ------------- --------- -------------------------------
10   FINANCE       active    Gi1/0/1
20   SALES         active    Gi1/0/2
30   IT            active
999  NATIVE-UNUSED active

! ── 检查 Trunk（最重要的一条命令）──
SW1# show interfaces trunk
Port        Mode  Encapsulation  Status    Native vlan
Gi1/0/24    on    802.1q         trunking  999          ← 两端必须一致

Port        Vlans allowed on trunk
Gi1/0/24    10,20,30                                    ← 放行列表

Port        Vlans allowed and active in management domain
Gi1/0/24    10,20,30

Port        Vlans in spanning tree forwarding state and not pruned
Gi1/0/24    10,20,30                                    ← 实际转发的

! ── 检查 SVI ──
SW1# show ip interface brief | include Vlan
Vlan10   192.168.10.1   YES manual  up    up            ← 必须 up/up
Vlan20   192.168.20.1   YES manual  up    up
Vlan30   192.168.30.1   YES manual  up    up

! ── 检查路由表 ──
SW1# show ip route connected
C    192.168.10.0/24 is directly connected, Vlan10
C    192.168.20.0/24 is directly connected, Vlan20
C    192.168.30.0/24 is directly connected, Vlan30
```

**连通性测试矩阵**：

| 测试 | 预期 | 说明 |
|:--|:--|:--|
| PC1 ping 192.168.10.1 | ✅ | 通网关 |
| PC1 ping PC3（同 VLAN，跨交换机） | ✅ | Trunk 正常 |
| PC1 ping PC2（跨 VLAN） | ✅ | SVI 路由正常 |
| PC1 ping PC4（跨 VLAN 跨交换机） | ✅ | 全链路正常 |

### Step 4：故意制造故障并排查（重点）

**故障 1：Native VLAN 不匹配**
```cisco
SW2(config)# interface Gi1/0/24
SW2(config-if)# switchport trunk native vlan 1
```
**观察**：Console 会打印 CDP 告警：
```
%CDP-4-NATIVE_VLAN_MISMATCH: Native VLAN mismatch discovered on 
GigabitEthernet1/0/24 (999), with SW2 GigabitEthernet1/0/24 (1).
```
**修复**：改回 999。

**故障 2：Trunk 未放行某 VLAN**
```cisco
SW1(config-if)# switchport trunk allowed vlan 10,20      ! 去掉了 30
```
**观察**：PC4（VLAN 30）无法 ping 通网关。
**排查**：
```cisco
SW1# show interfaces trunk
Port        Vlans allowed on trunk
Gi1/0/24    10,20                        ← 30 不见了，找到问题
```
**修复**：
```cisco
SW1(config-if)# switchport trunk allowed vlan add 30      ! 注意用 add
```

**故障 3：忘了 `ip routing`**
```cisco
SW1(config)# no ip routing
```
**观察**：同 VLAN 内通，**跨 VLAN 全部不通**。SVI 依然显示 up/up，路由表里也有直连路由，非常迷惑。
**排查**：
```cisco
SW1# show ip route
! 路由表看起来正常，但转发不工作
SW1# show running-config | include ip routing
! 没有输出 → 说明没开
```
**修复**：`ip routing`

> 这个故障之所以典型，是因为**所有表面证据都正常**（SVI up、路由表有条目），只有一个隐藏的全局开关关着。记住这个症状：**同 VLAN 通、跨 VLAN 全不通、且 SVI 状态正常 → 检查 `ip routing`**。

---

## ⑤ 排障思路

| 症状 | 怀疑点 | 验证命令 | 根因 |
|:--|:--|:--|:--|
| 同 VLAN 跨交换机不通 | Trunk | `show interfaces trunk` | VLAN 未放行 / Trunk 没起来 |
| 两个本该隔离的 VLAN 互通了 | Native VLAN | `show interfaces trunk` 对比两端 | Native VLAN 不一致 |
| 跨 VLAN 全部不通，SVI 正常 | 三层转发 | `show run \| inc ip routing` | **忘了 `ip routing`** |
| 某个 SVI 一直是 down | SVI up 条件 | `show vlan brief`、`show int status` | VLAN 里没有 up 的端口 |
| 端口配了 VLAN 但不生效 | 端口模式 | `show int Gi1/0/1 switchport` | 端口还在 dynamic 模式 |
| 接了 AP/服务器后网络异常 | 端口类型 | `show int trunk` | 该配 Trunk 却配了 Access |
| Trunk 时通时不通 | DTP 协商 | `show int Gi1/0/24 switchport` | 两端模式组合不稳定 |
| 敲完命令整栋楼断网 | allowed vlan 覆盖 | `show interfaces trunk` | 用了 `allowed vlan X` 而非 `add X` |

### `show interfaces trunk` 四段输出的读法

```cisco
SW1# show interfaces trunk

! 第 1 段：模式、封装、状态、Native VLAN
Port        Mode         Encapsulation  Status        Native vlan
Gi1/0/24    on           802.1q         trunking      999
            └─配置的模式                 └─实际状态     └─对比两端！

! 第 2 段：配置上允许哪些 VLAN
Port        Vlans allowed on trunk
Gi1/0/24    10,20,30

! 第 3 段：允许的 VLAN 里，哪些实际存在于 VLAN 数据库
Port        Vlans allowed and active in management domain
Gi1/0/24    10,20,30
            ↑ 如果这里比第 2 段少，说明某个 VLAN 没创建

! 第 4 段：最终真正转发的（还要过 STP 这一关）
Port        Vlans in spanning tree forwarding state and not pruned
Gi1/0/24    10,20
            ↑ 如果这里比第 3 段少，说明该 VLAN 被 STP 阻塞了
```

> **排障时从下往上看**：先看第 4 段有没有你要的 VLAN，没有的话往上找是在哪一步被卡掉的。这个方法能在 30 秒内定位 90% 的 Trunk 问题。

### DTP 模式组合表（考点）

| SW1 \ SW2 | access | trunk | dynamic auto | dynamic desirable |
|:--|:--|:--|:--|:--|
| **access** | access | ⚠️ 不一致 | access | access |
| **trunk** | ⚠️ 不一致 | **trunk** | **trunk** | **trunk** |
| **dynamic auto** | access | **trunk** | ❌ **access** | **trunk** |
| **dynamic desirable** | access | **trunk** | **trunk** | **trunk** |

**要记的两条**：
1. **`auto` + `auto` = access**（双方都在等对方先开口，谁也不主动，最终退化成 access）—— 这是最容易出问题的组合。
2. **生产环境永远手工写死**：`switchport mode access` 或 `switchport mode trunk` + `switchport nonegotiate`。不要依赖 DTP，既不安全（VLAN 跳跃）又不可预测。

---

## ⑥ 考点提示 + 自测题

### 考点

- **802.1Q 标签 4 字节、VID 12 位 = 4094 个 VLAN** → 直接引出 VXLAN 的必要性。
- **Native VLAN 不一致**的后果和排查是必考。
- **VLAN 跳跃攻击**（Switch Spoofing + Double Tagging）及防御是安全部分的常客。
- **`ip routing` 忘配**是三层交换机最经典的坑。
- **`allowed vlan` 覆盖 vs `add`** 是实战安全意识题。
- **SVI up 的三个条件**。

### 自测题

**1.** Trunk 一端 Native VLAN 是 1，另一端是 99，会发生什么？为什么这是安全隐患？

<details><summary>答案</summary>

**后果**：VLAN 1 和 VLAN 99 的流量被"焊接"在一起。

具体过程：SW1 把 VLAN 1 的帧**不打标签**发到 Trunk（因为 VLAN 1 是它的 Native）；SW2 收到无标签帧，按自己的 Native VLAN 处理，**归入 VLAN 99**。反向同理。结果两个本该隔离的广播域互通了。

**安全隐患**：
1. **意外的 VLAN 泄露**：本该隔离的部门（比如访客网和内网）可以互访，绕过了所有基于 VLAN 的隔离策略。
2. **STP 异常**：不同 VLAN 的 BPDU 混在一起，可能导致 STP 拓扑计算错误。
3. **为 Double Tagging 攻击创造条件**。

**检测**：CDP/LLDP 会主动报 `%CDP-4-NATIVE_VLAN_MISMATCH`；或用 `show interfaces trunk` 对比两端的 `Native vlan` 列。

**最佳实践**：
```cisco
switchport trunk native vlan 999      ! 用一个不承载任何业务的 VLAN
vlan dot1q tag native                 ! 全局强制给 Native VLAN 也打标签
```
</details>

**2.** 为什么 VLAN 最多只能有 4094 个？这个限制导致了什么后果？

<details><summary>答案</summary>

**原因**：802.1Q 标签里的 **VID 字段只有 12 位**，2^12 = 4096。其中 VLAN 0 和 4095 保留，所以可用的是 1–4094。

**后果**：在大型数据中心或云环境里，**多租户场景下 4094 个远远不够**。一个中等规模的公有云可能有上万个租户，每个租户至少要一个独立的二层网络。

**这直接催生了 VXLAN**：
- VXLAN 的 **VNI（VXLAN Network Identifier）是 24 位** → 2^24 = **16,777,216** 个，彻底解决了标识符耗尽问题
- 而且 VXLAN 把二层帧封装在 UDP 里（目的端口 4789），**可以跨三层网络传输**，突破了 VLAN 必须在同一个二层域内的物理限制

这条演进线索（VLAN 4094 不够 → VXLAN 1600 万）是 ENCOR 讲 Overlay 网络时的核心动机。详见 [ENCOR 第 3 章](../02-ENCOR-350-401/03-Overlay-VXLAN与LISP.md)。
</details>

**3.** 三层交换机上配好了 SVI 和 IP，同 VLAN 内能通，但跨 VLAN 完全不通。SVI 显示 up/up，路由表里也有直连路由。问题在哪？

<details><summary>答案</summary>

**没有开启 `ip routing`。**

Cisco 的三层交换机（Catalyst 系列）默认**不开启**三层转发功能，即使配了 SVI 和 IP 也只是把它当作管理接口用。必须显式开启：

```cisco
SW1(config)# ip routing
```

**这个故障之所以难查**，是因为所有表面证据都正常：
- `show ip interface brief` → SVI 是 up/up ✓
- `show ip route` → 有直连路由 ✓
- 同 VLAN 内通信正常 ✓

只有一个隐藏的全局开关关着。

**验证方法**：
```cisco
SW1# show running-config | include ^ip routing
! 没有输出 = 没开启

SW1# show ip route
! 如果没开 ip routing，路由表顶部通常不会显示 "Gateway of last resort"，
! 且只有 Connected 路由，学不到任何动态路由
```

**厂商差异提醒**：**H3C 和华为的三层交换机默认就开启路由转发**，没有这个开关。从国产设备转到 Cisco 的工程师踩这个坑的概率极高。
</details>

**4.** 生产网络上，你想给某个 Trunk 口增加 VLAN 40 的放行。敲 `switchport trunk allowed vlan 40` 会发生什么？

<details><summary>答案</summary>

**整个 Trunk 上除 VLAN 40 外的所有 VLAN 立刻中断。**

`switchport trunk allowed vlan 40` 是**覆盖**操作——它把允许列表**替换**成"只有 VLAN 40"。原来放行的 10、20、30 全部被移除，对应的所有业务瞬间断网。

**正确写法**：
```cisco
SW1(config-if)# switchport trunk allowed vlan add 40
```

**相关命令**：
| 命令 | 作用 |
|:--|:--|
| `switchport trunk allowed vlan 10,20,30` | **覆盖**（危险） |
| `switchport trunk allowed vlan add 40` | 追加 |
| `switchport trunk allowed vlan remove 30` | 移除 |
| `switchport trunk allowed vlan all` | 放行全部 1-4094 |
| `switchport trunk allowed vlan except 999` | 除了 999 全放行 |

**安全操作习惯**：
1. 改之前先 `show interfaces trunk` 记录当前列表
2. 用 `add` / `remove`，永远不要用裸的赋值形式
3. 远程操作前用 `configure terminal revert timer 5`（见 [基础篇第 6 章](../00-基础篇/06-Cisco-IOS操作入门.md)）

**这条命令在运维事故排行榜上稳居前三**，和"敲错 ACL"、"reload 错设备"并列。
</details>

**5.** 一个 Access 口的默认 DTP 模式是 `dynamic auto`，攻击者接上笔记本发送 DTP 帧会怎样？怎么防？

<details><summary>答案</summary>

**攻击者的笔记本可以伪装成交换机**，与端口协商成 **Trunk**（因为 `dynamic auto` 会响应对方的 `desirable` 请求）。一旦成为 Trunk，攻击者就能**收到所有 VLAN 的流量**，也能向任意 VLAN 注入流量——这就是 **Switch Spoofing 攻击**。

**防御（接终端的端口必配）**：
```cisco
SW1(config-if)# switchport mode access        ! 强制 access，不参与协商
SW1(config-if)# switchport access vlan 10
SW1(config-if)# switchport nonegotiate        ! 彻底关闭 DTP 帧的收发
SW1(config-if)# spanning-tree portfast
SW1(config-if)# spanning-tree bpduguard enable ! 收到 BPDU 就 err-disable
```

**加固清单（接入端口标准模板）**：
1. `switchport mode access` —— 不给协商机会
2. `switchport nonegotiate` —— 不发不收 DTP
3. `spanning-tree bpduguard enable` —— 防止私接交换机造成环路
4. `switchport port-security` —— 限制 MAC 数量，防 MAC 泛洪
5. 未使用的端口：`shutdown` + 划入一个隔离的"黑洞 VLAN"

> **实战提醒**：不同 Cisco 平台的默认 DTP 模式不一致（有的是 `dynamic auto`，有的是 `dynamic desirable`），**不要依赖默认值**。所有端口都显式配置。
</details>

---

**下一章** → [02 STP 生成树协议](02-STP生成树协议.md)
