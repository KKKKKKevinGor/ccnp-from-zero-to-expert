# 05 · MPLS 与 L3VPN

## ① 这章解决什么问题

**运营商的困境**：一台 PE（运营商边缘路由器）要同时服务 100 个企业客户。

- 客户 A 用 `192.168.1.0/24`
- 客户 B 也用 `192.168.1.0/24`
- 客户 C 还是 `192.168.1.0/24`

**三个问题**：
1. **地址重叠** —— 传统路由表里同一个前缀只能有一条
2. **必须隔离** —— 客户之间绝不能互通
3. **规模** —— 100 个客户 × 每个几百条路由 = 几万条，还要在整个骨干网传播

**MPLS L3VPN 就是这三个问题的答案**：
- **VRF** 解决隔离（每个客户一张独立路由表）
- **RD** 解决地址重叠（给前缀加前缀，变成全局唯一）
- **MP-BGP** 解决传播（只在 PE 之间传，P 路由器完全不知道客户路由）
- **MPLS 标签** 解决转发（骨干网按标签转发，不查 IP）

> **ENARSI 层面要求理解概念和基本配置，不要求达到 CCIE SP 的深度。** 但**标签转发流程**和 **RD/RT 的区别**是必考点。

---

## ② MPLS 基础

### 2.1 为什么需要 MPLS

**传统 IP 转发**：每台路由器都要**查完整的路由表**做最长匹配。路由表越大越慢。

**MPLS 转发**：中间路由器**只看一个 20 位的标签**，做精确匹配（不是最长匹配），查表极快。

```
   传统 IP：每跳都查路由表（最长匹配，慢）
        ↓
   MPLS：  入口打标签 → 中间只换标签 → 出口剥标签
           ★ 中间路由器根本不需要知道目的 IP ★
```

> **注意**：现代硬件的 IP 查表已经很快了（TCAM/ASIC），"MPLS 转发更快"这个理由已经不成立。
>
> **MPLS 今天的真正价值是**：
> 1. **VPN**（L3VPN、L2VPN/VPLS）—— 最主要的用途
> 2. **流量工程（MPLS-TE）** —— 强制流量走指定路径
> 3. **快速重路由（FRR）** —— 50ms 级别的保护倒换

### 2.2 标签结构

```
 ┌──────────────────┬─────┬───┬─────────┐
 │   Label (20 bit) │ EXP │ S │  TTL    │
 │                  │(3)  │(1)│  (8)    │
 └──────────────────┴─────┴───┴─────────┘
              共 32 位 = 4 字节
```

| 字段 | 位数 | 作用 |
|:--|:--|:--|
| **Label** | 20 | 标签值（2^20 ≈ 100 万个） |
| **EXP** | 3 | 实验位，实际用作 **QoS 优先级**（相当于 DSCP 的前 3 位） |
| **S** | 1 | **栈底标志**（Bottom of Stack）：1 = 这是最后一层标签 |
| **TTL** | 8 | 生存时间，**防环** |

**标签的位置**：
```
┌──────────┬───────────┬───────────┬────────────┐
│ 二层头    │ MPLS 标签 │  IP 头     │   数据      │
│(以太网)   │ (可多层)  │           │            │
└──────────┴───────────┴───────────┴────────────┘
            ↑ 在二层和三层之间 → 所以叫 "2.5 层协议"
```

**保留标签值**：

| 值 | 名称 | 含义 |
|:--|:--|:--|
| **0** | IPv4 Explicit NULL | 显式空标签，弹出后按 IPv4 转发 |
| 1 | Router Alert | 类似 IP 的 Router Alert 选项 |
| 2 | IPv6 Explicit NULL | |
| **3** | **Implicit NULL** | ★ **隐式空标签，用于 PHP** |
| 4-15 | 保留 | |
| 16+ | 可用 | 动态分配 |

### 2.3 三个角色

```
   CE ──── PE ──── P ──── P ──── PE ──── CE
   ↑        ↑      ↑             ↑        ↑
  客户     运营商  运营商核心    运营商    客户
  设备     边缘    (只转标签)    边缘      设备
```

| 角色 | 全称 | 职责 | 知道客户路由吗 |
|:--|:--|:--|:--|
| **CE** | Customer Edge | 客户侧路由器，**跑普通 IP** | ✅ 只知道自己的 |
| **PE** | Provider Edge | 运营商边缘，**维护 VRF，打/剥标签，跑 MP-BGP** | ✅ 知道它服务的客户的 |
| **P** | Provider | 运营商核心，**只做标签交换** | ❌ **完全不知道** |

> **★ P 路由器不知道任何客户路由** —— 这是 MPLS L3VPN 可扩展性的关键。骨干网的路由表只包含运营商自己的基础设施地址（PE 的 Loopback），不会因为客户增加而膨胀。

### 2.4 标签操作：压入、交换、弹出

```
   CE1 ──── PE1 ──── P1 ──── P2 ──── PE2 ──── CE2
             ↑        ↑       ↑       ↑
           PUSH     SWAP    SWAP     POP
          (压入)   (交换)  (交换)   (弹出)
```

**完整的转发流程**：

```
① CE1 发一个普通 IP 包给 PE1
   [IP: 192.168.2.10]

② PE1（入口 LSR / Ingress）：
   · 查 VRF 路由表，确定出口 PE 是 PE2
   · ★ PUSH 两层标签 ★
   [外层标签 L1 | 内层标签 VPN-Label | IP]
     ↑ 传输标签(LDP)    ↑ VPN 标签(MP-BGP)
     用于送到 PE2       用于 PE2 识别是哪个 VRF

③ P1（中间 LSR / Transit）：
   · 只看外层标签 L1
   · ★ SWAP：L1 → L2 ★
   [L2 | VPN-Label | IP]
   ★ P1 完全不知道内层标签和 IP 是什么 ★

④ P2（倒数第二跳 / Penultimate）：
   · ★ PHP：POP 外层标签 ★
   [VPN-Label | IP]

⑤ PE2（出口 LSR / Egress）：
   · 看 VPN 标签，确定是哪个 VRF
   · ★ POP VPN 标签 ★
   [IP: 192.168.2.10]
   · 在对应 VRF 的路由表里查，发给 CE2
```

### 2.5 PHP（倒数第二跳弹出）★ 考点

**PHP = Penultimate Hop Popping**

**问题**：如果 P2 不弹标签，PE2 收到的包有两层标签：
```
   PE2 要做两次查表：
   ① 查外层标签 → 发现是给自己的 → 弹出
   ② 查 VPN 标签 → 确定 VRF → 弹出 → 查 IP
   ★ 两次查表，PE2 负担重 ★
```

**PHP 的解法**：让**倒数第二跳（P2）**提前把外层标签弹掉。

```
   PE2 向 P2 通告标签时，通告的是 ★ 标签 3（Implicit NULL）★
        ↓
   P2 收到"标签 3"的含义是："发给我的时候不要打标签，直接弹掉"
        ↓
   P2 弹出外层标签，只发 [VPN-Label | IP] 给 PE2
        ↓
   ★ PE2 只需要一次查表 ★
```

**验证**：
```cisco
P2# show mpls forwarding-table
Local  Outgoing  Prefix           Bytes Label  Outgoing   Next Hop
Label  Label     or Tunnel Id     Switched     interface
16     ★ Pop Label ★ 2.2.2.2/32   1234567      Gi0/1      10.0.0.2
       ↑ 显示 "Pop Label" 就是 PHP 生效
```

```cisco
PE2# show mpls ldp bindings 2.2.2.2 32
  lib entry: 2.2.2.2/32, rev 8
        local binding:  label: ★ imp-null ★             ← 通告给邻居的是 3
        remote binding: lsr: 3.3.3.3:0, label: 16
```

**关闭 PHP（某些场景需要，比如要保留 EXP 位做 QoS）**：
```cisco
PE2(config)# mpls ldp explicit-null
! 改为通告标签 0（Explicit NULL），标签还在但值是 0
```

### 2.6 LDP（标签分发协议）

**PE 和 P 之间用 LDP 交换标签映射。**

```cisco
! ── 全局启用 ──
R1(config)# mpls label protocol ldp
R1(config)# mpls ldp router-id Loopback0 force

! ── 接口启用 ──
R1(config)# interface GigabitEthernet0/1
R1(config-if)# mpls ip
R1(config-if)# mtu 1600                            ! ★ 考虑标签开销

! ── 或用 OSPF 自动在所有接口启用（IOS-XE）──
R1(config)# router ospf 1
R1(config-router)# mpls ldp autoconfig

! ── 认证 ──
R1(config)# mpls ldp neighbor 2.2.2.2 password MyLdpSecret

! ── 查看 ──
R1# show mpls ldp neighbor
R1# show mpls ldp bindings
R1# show mpls forwarding-table
R1# show mpls interfaces
R1# show mpls ldp discovery
```

**LDP 的端口**：
- **UDP 646** —— Hello 发现（组播 224.0.0.2）
- **TCP 646** —— 会话建立和标签交换

**LDP 邻居建立的条件**：
1. **能互相收到 Hello**（UDP 646）
2. **LDP Router ID 必须可达**（★ 常见问题：Router ID 用了 Loopback 但没通告进 IGP）
3. 认证匹配（如果配了）

```cisco
! ★ 最常见的 LDP 故障：Router ID 不可达
R1# show mpls ldp discovery
 Local LDP Identifier:
    1.1.1.1:0
    Discovery Sources:
    Interfaces:
        GigabitEthernet0/1 (ldp): xmit/recv
            LDP Id: 2.2.2.2:0                      ← 发现了邻居
            
R1# show mpls ldp neighbor
! 空！★ 发现了但会话建不起来 ★

R1# ping 2.2.2.2
% Network not in table                             ← ★ 找到原因：Router ID 不可达
```

**修复**：确保所有 PE/P 的 Loopback 都通告进 IGP。

---

## ③ MPLS L3VPN

### 3.1 核心概念

| 概念 | 全称 | 作用 |
|:--|:--|:--|
| **VRF** | Virtual Routing and Forwarding | **独立的路由表**，实现隔离 |
| **RD** | **Route Distinguisher** | **区分路由**，解决地址重叠 |
| **RT** | **Route Target** | **控制路由的导入导出**，决定谁能看到谁 |
| **VPNv4** | — | RD + IPv4 前缀 = 96 位地址族 |
| **MP-BGP** | Multiprotocol BGP | 在 PE 之间传播 VPNv4 路由 |

### 3.2 RD vs RT（★★★ 最重要的考点）

| | **RD (Route Distinguisher)** | **RT (Route Target)** |
|:--|:--|:--|
| **作用** | **让重叠的地址变得唯一** | **控制路由的导入/导出** |
| 本质 | **前缀的一部分**（拼在 IPv4 前缀前面） | **BGP 扩展团体属性**（Extended Community） |
| 数量 | **每个 VRF 一个** | **每个 VRF 可以有多个**（import/export 各自可多个） |
| 影响 | 只影响"唯一性" | **决定 VPN 的拓扑**（全互联/星型/混合） |
| 类比 | **身份证号** | **通行证 / 门禁卡** |

**RD 的工作方式**：
```
   客户 A 的 192.168.1.0/24
        ↓ PE 加上 RD 65000:100
   ★ VPNv4 前缀：65000:100:192.168.1.0/24 ★
   
   客户 B 的 192.168.1.0/24
        ↓ PE 加上 RD 65000:200
   ★ VPNv4 前缀：65000:200:192.168.1.0/24 ★
   
   ★ 两条 VPNv4 路由完全不同，可以共存于同一个 MP-BGP 表 ★
```

**RD 格式**（96 位 = RD 64 位 + IPv4 32 位）：
```
   Type 0: ASN:NN        （如 65000:100）      ← 最常用
   Type 1: IP:NN         （如 1.1.1.1:100）
   Type 2: 4byteASN:NN
```

**RT 的工作方式**：
```
   PE1 上的 VRF CUSTOMER-A：
     export route-target 65000:100     ← 我导出的路由打上这个标签
     import route-target 65000:100     ← 我导入带这个标签的路由
   
   PE2 上的 VRF CUSTOMER-A：
     export route-target 65000:100
     import route-target 65000:100
        ↓
   ★ 两端 RT 匹配 → 路由互通 ✓ ★
```

**★ 关键理解：RT 决定 VPN 的拓扑。**

**全互联（Full Mesh）**：所有站点 import/export 同一个 RT
```cisco
vrf definition CUSTOMER-A
 rd 65000:100
 address-family ipv4
  route-target export 65000:100
  route-target import 65000:100
```

**中心辐射（Hub-and-Spoke）★ 经典设计**：
```cisco
! ── Hub（总部）──
vrf definition HUB
 rd 65000:100
 address-family ipv4
  route-target export 65000:1000      ! 导出"来自 Hub"
  route-target import 65000:2000      ! 导入"来自 Spoke"

! ── Spoke（分支）──
vrf definition SPOKE
 rd 65000:101                          ! ★ 每个 Spoke 用不同的 RD
 address-family ipv4
  route-target export 65000:2000      ! 导出"来自 Spoke"
  route-target import 65000:1000      ! ★ 只导入 Hub 的，不导入其他 Spoke 的
```

**效果**：
```
   Spoke-A ──> Hub ──> Spoke-B    ✅ 可以（都经过 Hub）
   Spoke-A ──X──> Spoke-B         ❌ 不可以（RT 不匹配）
   
   ★ 所有分支之间的流量必须经过总部 ★
   → 总部可以做集中的安全检查、审计、流量监控
```

**这就是"RT 决定拓扑"的含义。** 只改 RT 配置，就能把全互联变成星型，不需要动任何路由或 ACL。

> **★ 考点：RD 不能用来控制路由的导入导出。** 有些人误以为"RD 不同就不能互通"——错。RD 只是让前缀唯一，**是否互通完全由 RT 决定**。
>
> 两个 VRF 可以有不同的 RD 但相同的 RT → 互通；相同的 RD 但不同的 RT → 不互通。

### 3.3 完整配置

```cisco
! ═══════════ PE1 ═══════════

! ── ① IGP（PE 和 P 之间）──
PE1(config)# interface Loopback0
PE1(config-if)# ip address 1.1.1.1 255.255.255.255
PE1(config)# router ospf 1
PE1(config-router)# router-id 1.1.1.1
PE1(config-router)# network 1.1.1.1 0.0.0.0 area 0
PE1(config-router)# network 10.0.0.0 0.0.255.255 area 0
PE1(config-router)# mpls ldp autoconfig                   ! 自动在 OSPF 接口启用 LDP

! ── ② MPLS/LDP ──
PE1(config)# mpls label protocol ldp
PE1(config)# mpls ldp router-id Loopback0 force
PE1(config)# interface GigabitEthernet0/1
PE1(config-if)# mpls ip
PE1(config-if)# mtu 1600                                  ! ★ 考虑标签开销

! ── ③ 定义 VRF ──
PE1(config)# vrf definition CUSTOMER-A
PE1(config-vrf)#  rd 65000:100
PE1(config-vrf)#  address-family ipv4
PE1(config-vrf-af)#   route-target export 65000:100
PE1(config-vrf-af)#   route-target import 65000:100
PE1(config-vrf-af)#   exit-address-family

PE1(config)# vrf definition CUSTOMER-B
PE1(config-vrf)#  rd 65000:200
PE1(config-vrf)#  address-family ipv4
PE1(config-vrf-af)#   route-target export 65000:200
PE1(config-vrf-af)#   route-target import 65000:200

! ── ④ PE-CE 接口划入 VRF ──
PE1(config)# interface GigabitEthernet0/2
PE1(config-if)# vrf forwarding CUSTOMER-A                 ! ★ 会清除 IP，先划 VRF 再配 IP
PE1(config-if)# ip address 192.168.1.1 255.255.255.0

! ── ⑤ MP-BGP（PE 之间）──
PE1(config)# router bgp 65000
PE1(config-router)# bgp router-id 1.1.1.1
PE1(config-router)# neighbor 2.2.2.2 remote-as 65000
PE1(config-router)# neighbor 2.2.2.2 update-source Loopback0

PE1(config-router)# address-family vpnv4                  ! ★ VPNv4 地址族
PE1(config-router-af)#  neighbor 2.2.2.2 activate
PE1(config-router-af)#  neighbor 2.2.2.2 send-community both    ! ★ 必须（RT 是扩展团体）
PE1(config-router-af)#  exit-address-family

! ── ⑥ PE-CE 路由（三选一）──
! 方式 A：静态路由
PE1(config)# ip route vrf CUSTOMER-A 192.168.10.0 255.255.255.0 192.168.1.2

! 方式 B：OSPF
PE1(config)# router ospf 10 vrf CUSTOMER-A
PE1(config-router)# router-id 1.1.1.1
PE1(config-router)# network 192.168.1.0 0.0.0.255 area 0
PE1(config-router)# domain-id 0.0.0.100                   ! 同一客户的所有 PE 要一致

! 方式 C：BGP（推荐，最简洁）
PE1(config)# router bgp 65000
PE1(config-router)# address-family ipv4 vrf CUSTOMER-A
PE1(config-router-af)#  neighbor 192.168.1.2 remote-as 65100
PE1(config-router-af)#  neighbor 192.168.1.2 activate

! ── ⑦ 把 PE-CE 的 IGP 重分发进 MP-BGP（用 OSPF/静态时需要）──
PE1(config)# router bgp 65000
PE1(config-router)# address-family ipv4 vrf CUSTOMER-A
PE1(config-router-af)#  redistribute ospf 10 vrf CUSTOMER-A
PE1(config-router-af)#  redistribute static
PE1(config-router-af)#  redistribute connected

! 反向：把 BGP 重分发进 PE-CE 的 OSPF
PE1(config)# router ospf 10 vrf CUSTOMER-A
PE1(config-router)# redistribute bgp 65000 subnets
```

### 3.4 验证命令

```cisco
! ── MPLS 基础 ──
PE1# show mpls ldp neighbor
PE1# show mpls forwarding-table
PE1# show mpls interfaces
PE1# show mpls ldp bindings

! ── VRF ──
PE1# show vrf
PE1# show vrf detail CUSTOMER-A
PE1# show ip route vrf CUSTOMER-A
PE1# show ip vrf interfaces

! ── MP-BGP ──
PE1# show bgp vpnv4 unicast all
PE1# show bgp vpnv4 unicast all summary
PE1# show bgp vpnv4 unicast vrf CUSTOMER-A
PE1# show bgp vpnv4 unicast rd 65000:100

! ── 端到端测试 ──
PE1# ping vrf CUSTOMER-A 192.168.2.10
PE1# traceroute vrf CUSTOMER-A 192.168.2.10
PE1# trace mpls ip 2.2.2.2/32                     ! MPLS 路径追踪
```

**`show mpls forwarding-table` 解读**：
```cisco
PE1# show mpls forwarding-table
Local  Outgoing   Prefix            Bytes Label  Outgoing   Next Hop
Label  Label      or Tunnel Id      Switched     interface
16     17         2.2.2.2/32        1234567      Gi0/1      10.0.12.2
       ↑          ↑                                          ↑
    出标签      目的（PE2 的 Loopback）                    下一跳
       
17     Pop Label  3.3.3.3/32        987654       Gi0/1      10.0.12.2
       ↑ PHP 生效

20     No Label   192.168.1.0/24[V] 456789       Gi0/2      192.168.1.2
       ↑                        ↑
    本地 VPN 标签           [V] = VRF 路由
```

**`show bgp vpnv4 unicast all` 解读**：
```cisco
PE1# show bgp vpnv4 unicast all
   Network          Next Hop     Metric LocPrf Weight Path
Route Distinguisher: 65000:100 (default for vrf CUSTOMER-A)
 *>  192.168.1.0/24   0.0.0.0        0         32768 ?        ← 本地的
 *>i 192.168.2.0/24   2.2.2.2        0    100      0 ?        ← 从 PE2 学到的
                       ↑ 下一跳是 PE2 的 Loopback

Route Distinguisher: 65000:200 (default for vrf CUSTOMER-B)
 *>  192.168.1.0/24   0.0.0.0        0         32768 ?        ← ★ 相同前缀，不同 RD
```

**✅ 这就是 RD 解决地址重叠的直观体现**——两个客户都用 `192.168.1.0/24`，在 VPNv4 表里通过 RD 区分开。

---

## ④ 排障

### 4.1 分层排障框架

```
   ① Underlay（IGP）通吗？
      PE1# ping 2.2.2.2 source Loopback0
      ★ PE 的 Loopback 必须互通 ★
        ↓
   ② LDP 邻居起来了吗？
      PE1# show mpls ldp neighbor
      → 空？检查 Router ID 可达性、接口 mpls ip
        ↓
   ③ 标签转发表有吗？
      PE1# show mpls forwarding-table | include 2.2.2.2
      → 没有？LDP 标签分发有问题
        ↓
   ④ MP-BGP 邻居起来了吗？
      PE1# show bgp vpnv4 unicast all summary
      → 检查 activate 和 send-community
        ↓
   ⑤ VPNv4 路由收到了吗？
      PE1# show bgp vpnv4 unicast all
      → 有 RD 但没有对方的路由 → RT 不匹配
        ↓
   ⑥ 路由进 VRF 表了吗？
      PE1# show ip route vrf CUSTOMER-A
      → BGP 表有但 VRF 表没有 → RT import 不匹配
        ↓
   ⑦ PE-CE 路由正常吗？
      PE1# show ip route vrf CUSTOMER-A
      PE1# ping vrf CUSTOMER-A <CE的IP>
```

### 4.2 常见故障

| 症状 | 根因 | 验证 |
|:--|:--|:--|
| LDP 邻居建不起来 | **Router ID 不可达** | `ping <对端 LDP Router ID>` |
| | 接口没配 `mpls ip` | `show mpls interfaces` |
| | 认证不匹配 | `debug mpls ldp session` |
| MP-BGP 邻居 Established 但没路由 | **没 `activate`** | `show run \| sec address-family vpnv4` |
| **RT 携带不过来** | **缺 `send-community both`** | 同上 |
| **VPNv4 有路由但 VRF 表没有** | **RT import 不匹配** | `show vrf detail`、`show bgp vpnv4 all` |
| 客户站点互相能通（不该通） | **RT 配错了** | `show vrf detail` 对比 |
| **大包不通小包通** | **MTU 不足**（标签开销） | `ping size 1500 df-bit` |
| PE-CE OSPF 路由变成 E2 | domain-id 不一致 | `show ip ospf \| inc Domain` |
| traceroute 看不到 P 路由器 | 正常（P 不解 IP） | `no mpls ip propagate-ttl` 的效果 |

### 4.3 MTU 问题（★ 必须处理）

**MPLS 标签开销**：
```
   每层标签 4 字节
   L3VPN 通常两层 = 8 字节
   
   原始包 1500 + 8 = 1508 字节
        ↓
   如果骨干网接口 MTU 是 1500 → ★ 被丢弃 ★
```

**症状**：ping 通、SSH 能连（小包），但**网页打不开、大文件传不动**（和 GRE/VXLAN 的 MTU 问题一模一样）。

**解法**：
```cisco
! ── 骨干网所有接口都要加大 MTU ──
P1(config)# interface GigabitEthernet0/1
P1(config-if)# mtu 1600                       ! 至少 1508，建议 1600+
P1(config-if)# mpls mtu 1600

! ── 或直接开巨帧 ──
P1(config-if)# mtu 9000

! ── 验证 ──
PE1# ping 2.2.2.2 size 1500 df-bit source Loopback0
PE1# show mpls interfaces detail
```

> **★ MPLS 部署的第一个检查项永远是 MTU**（和 VXLAN 一样）。骨干网路径上**每一台设备的每一个接口**都要配，漏一台就有问题。

### 4.4 TTL 传播

```cisco
! 默认：MPLS 会把 IP 的 TTL 复制到标签，客户能看到运营商的内部拓扑
PE1(config)# mpls ip propagate-ttl

! ★ 运营商通常关闭，隐藏内部拓扑 ★
PE1(config)# no mpls ip propagate-ttl
```

**效果对比**：
```
   开启 propagate-ttl（默认）：
   CE1# traceroute 192.168.2.10
     1 192.168.1.1     (PE1)
     2 10.0.12.2       (P1)      ← ★ 客户能看到运营商内部
     3 10.0.23.3       (P2)      ← ★
     4 192.168.2.1     (PE2)
     5 192.168.2.10

   关闭 propagate-ttl：
   CE1# traceroute 192.168.2.10
     1 192.168.1.1     (PE1)
     2 192.168.2.1     (PE2)     ← ★ 骨干网像一跳
     3 192.168.2.10
```

---

## ⑤ 配套实验

**拓扑**：
```
   CE1 ──── PE1 ──── P ──── PE2 ──── CE2
   192.168.1.0/24              192.168.2.0/24
        │                            │
   VRF: CUSTOMER-A            VRF: CUSTOMER-A
   
   CE3 ──── PE1 ──── P ──── PE2 ──── CE4
   192.168.1.0/24              192.168.2.0/24
        ↑ ★ 地址故意重叠 ★
   VRF: CUSTOMER-B            VRF: CUSTOMER-B
```

### Step 1：Underlay + LDP

```cisco
! ── 所有 PE 和 P ──
PE1(config)# interface Loopback0
PE1(config-if)# ip address 1.1.1.1 255.255.255.255

PE1(config)# router ospf 1
PE1(config-router)# router-id 1.1.1.1
PE1(config-router)# network 1.1.1.1 0.0.0.0 area 0
PE1(config-router)# network 10.0.0.0 0.0.255.255 area 0
PE1(config-router)# mpls ldp autoconfig

PE1(config)# mpls label protocol ldp
PE1(config)# mpls ldp router-id Loopback0 force

PE1(config)# interface GigabitEthernet0/1
PE1(config-if)# mtu 1600                          ! ★ 别忘了
```

**验证**：
```cisco
PE1# ping 2.2.2.2 source Loopback0
!!!!!                                             ← Underlay 通 ✓

PE1# show mpls ldp neighbor
    Peer LDP Ident: 3.3.3.3:0; Local LDP Ident 1.1.1.1:0
        TCP connection: 3.3.3.3.646 - 1.1.1.1.31234
        State: ★ Oper ★; Msgs sent/rcvd: 45/43     ← LDP 正常 ✓

PE1# show mpls forwarding-table
Local  Outgoing  Prefix         Bytes Label  Outgoing  Next Hop
Label  Label     or Tunnel Id   Switched     interface
16     17        2.2.2.2/32     0            Gi0/1     10.0.13.3
                                                       ↑ 有标签转发表项 ✓
```

### Step 2：VRF + MP-BGP

```cisco
PE1(config)# vrf definition CUSTOMER-A
PE1(config-vrf)#  rd 65000:100
PE1(config-vrf)#  address-family ipv4
PE1(config-vrf-af)#   route-target both 65000:100     ! both = import + export

PE1(config)# vrf definition CUSTOMER-B
PE1(config-vrf)#  rd 65000:200
PE1(config-vrf)#  address-family ipv4
PE1(config-vrf-af)#   route-target both 65000:200

PE1(config)# interface GigabitEthernet0/2
PE1(config-if)# vrf forwarding CUSTOMER-A
PE1(config-if)# ip address 192.168.1.1 255.255.255.0

PE1(config)# interface GigabitEthernet0/3
PE1(config-if)# vrf forwarding CUSTOMER-B
PE1(config-if)# ip address 192.168.1.1 255.255.255.0   ! ★ 相同 IP，不同 VRF

PE1(config)# router bgp 65000
PE1(config-router)# neighbor 2.2.2.2 remote-as 65000
PE1(config-router)# neighbor 2.2.2.2 update-source Loopback0
PE1(config-router)# address-family vpnv4
PE1(config-router-af)#  neighbor 2.2.2.2 activate
PE1(config-router-af)#  neighbor 2.2.2.2 send-community both    ! ★ 必须
```

### Step 3：验证地址重叠（★ 核心验证）

```cisco
PE1# show ip route vrf CUSTOMER-A | include 192.168.1.0
C    192.168.1.0/24 is directly connected, GigabitEthernet0/2

PE1# show ip route vrf CUSTOMER-B | include 192.168.1.0
C    192.168.1.0/24 is directly connected, GigabitEthernet0/3

! ★ 同一个前缀在两个 VRF 里共存 ✓ ★
```

```cisco
PE1# show bgp vpnv4 unicast all
   Network          Next Hop     Metric LocPrf Weight Path
Route Distinguisher: 65000:100 (default for vrf CUSTOMER-A)
 *>  192.168.1.0/24   0.0.0.0        0         32768 ?
 *>i 192.168.2.0/24   2.2.2.2        0    100      0 ?

Route Distinguisher: 65000:200 (default for vrf CUSTOMER-B)
 *>  192.168.1.0/24   0.0.0.0        0         32768 ?     ← ★ 相同前缀
 *>i 192.168.2.0/24   2.2.2.2        0    100      0 ?     ← 不同 RD，共存 ✓
```

**✅ RD 解决地址重叠的直观证明。**

### Step 4：验证隔离

```cisco
PE1# ping vrf CUSTOMER-A 192.168.2.10
!!!!!                                             ← 客户 A 内部通 ✓

PE1# ping vrf CUSTOMER-B 192.168.2.10
!!!!!                                             ← 客户 B 内部通 ✓

! 试图跨 VRF（应该不通）
CE1(客户A)# ping 192.168.2.10                     ← 到达的是客户 A 的站点
CE3(客户B)# ping 192.168.2.10                     ← 到达的是客户 B 的站点
! ★ 同一个 IP，去往不同的地方，完全隔离 ✓ ★
```

### Step 5：故障注入

**故障 A：RT 不匹配**
```cisco
PE2(config)# vrf definition CUSTOMER-A
PE2(config-vrf)#  address-family ipv4
PE2(config-vrf-af)#   no route-target import 65000:100
PE2(config-vrf-af)#   route-target import 65000:999      ! 改错了
```

**观察**：
```cisco
PE2# show bgp vpnv4 unicast all | include 192.168.1.0
Route Distinguisher: 65000:100
 *>i 192.168.1.0/24   1.1.1.1  ...
! ★ BGP 表里有 ★

PE2# show ip route vrf CUSTOMER-A | include 192.168.1.0
! ★ 但 VRF 路由表里没有 ★
```

**✅ 这个对比清楚地展示了 RT 的作用**：RT 不匹配时，路由能收到（在 VPNv4 表里），但**不会被导入 VRF 路由表**。

**排查**：
```cisco
PE2# show vrf detail CUSTOMER-A
VRF CUSTOMER-A (VRF Id = 1); default RD 65000:100
  Interfaces:
    Gi0/2
Address family ipv4 unicast (Table ID = 0x1):
  Export VPN route-target communities
    RT:65000:100
  Import VPN route-target communities
    RT:65000:999                                  ← ★ 找到了
```

**故障 B：缺 `send-community`**
```cisco
PE1(config-router-af)# no neighbor 2.2.2.2 send-community both
PE1# clear ip bgp 2.2.2.2 soft out
```

**观察**：
```cisco
PE2# show bgp vpnv4 unicast all | include 192.168.1.0
! 空！★ 路由收不到，因为 RT（扩展团体）没被发送 ★
```

**修复**：
```cisco
PE1(config-router-af)# neighbor 2.2.2.2 send-community both
```

**故障 C：MTU 不足**
```cisco
P(config)# interface GigabitEthernet0/1
P(config-if)# mtu 1500
```

**观察**：
```cisco
CE1# ping 192.168.2.10 size 100
!!!!!                                             ← 小包通

CE1# ping 192.168.2.10 size 1500 df-bit
.....                                             ← ★ 大包不通
```

**修复**：`mtu 1600`

**故障 D：LDP Router ID 不可达**
```cisco
P(config)# router ospf 1
P(config-router)# no network 3.3.3.3 0.0.0.0 area 0    ! Loopback 不通告了
```

**观察**：
```cisco
PE1# show mpls ldp discovery
        GigabitEthernet0/1 (ldp): xmit/recv
            LDP Id: 3.3.3.3:0                     ← 发现了

PE1# show mpls ldp neighbor
! 空 ★ 但会话建不起来 ★

PE1# ping 3.3.3.3
% Network not in table                            ← 找到原因
```

---

## ⑥ 自测题

**1.** MPLS 标签的压入、交换、弹出分别发生在哪里？什么是 PHP？

<details><summary>答案</summary>

```
   CE1 ──── PE1 ──── P1 ──── P2 ──── PE2 ──── CE2
             ↑        ↑       ↑        ↑
           PUSH      SWAP    POP     (只剩VPN标签)
          (压入)    (交换)   (PHP)
```

| 操作 | 在哪 | 说明 |
|:--|:--|:--|
| **PUSH（压入）** | **入口 PE（Ingress LSR）** | 给 IP 包打上标签（L3VPN 打**两层**） |
| **SWAP（交换）** | **中间 P 路由器（Transit LSR）** | 只看外层标签，换成新标签 |
| **POP（弹出）** | **倒数第二跳（PHP）或出口 PE** | 去掉标签 |

**L3VPN 的两层标签**：
```
   [外层标签 | 内层标签 | IP 包]
      ↑          ↑
   传输标签    VPN 标签
   (LDP分发)   (MP-BGP分发)
   
   外层：用于把包从 PE1 送到 PE2（P 路由器只看这个）
   内层：PE2 用来识别"这个包属于哪个 VRF"
```

## PHP（Penultimate Hop Popping，倒数第二跳弹出）

**Penultimate = 倒数第二个**

**问题**：如果 P2（倒数第二跳）不弹标签，PE2 要做两次查表：
```
   PE2 收到 [外层标签 | VPN标签 | IP]
        ↓
   ① 查外层标签 → 发现是给自己的 → 弹出
   ② 查 VPN 标签 → 确定 VRF → 弹出 → 查 IP
   ★ 两次查表，PE2 负担重 ★
```

**PHP 的解法**：
```
   PE2 向 P2 通告标签时，★ 通告标签 3（Implicit NULL）★
        ↓
   标签 3 的含义："发给我时不要打外层标签，直接弹掉"
        ↓
   P2 弹出外层标签，只发 [VPN标签 | IP] 给 PE2
        ↓
   ★ PE2 只需要一次查表 ★
```

**验证 PHP 生效**：
```cisco
P2# show mpls forwarding-table
Local  Outgoing      Prefix        Bytes Label  Outgoing  Next Hop
Label  Label         or Tunnel Id  Switched     interface
16     ★ Pop Label ★ 2.2.2.2/32    1234567      Gi0/1     10.0.0.2
       ↑ 显示 "Pop Label" = PHP
```

```cisco
PE2# show mpls ldp bindings 2.2.2.2 32
  lib entry: 2.2.2.2/32, rev 8
        local binding:  label: ★ imp-null ★      ← 通告给上游的是标签 3
```

**保留标签值**：

| 值 | 名称 | 用途 |
|:--|:--|:--|
| **0** | IPv4 Explicit NULL | 显式空标签，**标签还在但值是 0** |
| 1 | Router Alert | |
| 2 | IPv6 Explicit NULL | |
| **3** | **Implicit NULL** | ★ **PHP：请不要打标签** |

**Explicit NULL vs Implicit NULL**：

| | Implicit NULL (3) | Explicit NULL (0) |
|:--|:--|:--|
| 行为 | 倒数第二跳**完全弹掉**标签 | 倒数第二跳把标签**换成 0** |
| 出口收到 | 没有外层标签 | **有一个值为 0 的标签** |
| **EXP 位（QoS）** | ❌ **丢失** | ✅ **保留** |
| 默认 | ✅ | 需手工配置 |

**什么时候用 Explicit NULL**：需要在出口 PE 上**保留 QoS 信息（EXP 位）**时。
```cisco
PE2(config)# mpls ldp explicit-null
```

**完整的转发流程回顾**：
```
① CE1 → PE1：  [IP]
② PE1 → P1：   [L16 | VPN-L100 | IP]      PUSH 两层
③ P1 → P2：    [L17 | VPN-L100 | IP]      SWAP 外层
④ P2 → PE2：   [VPN-L100 | IP]            POP 外层（PHP）
⑤ PE2 → CE2：  [IP]                       POP VPN 标签
```
</details>

**2.** RD 和 RT 有什么区别？为什么两个都需要？

<details><summary>答案</summary>

| | **RD (Route Distinguisher)** | **RT (Route Target)** |
|:--|:--|:--|
| **作用** | **让重叠的地址变得唯一** | **控制路由的导入/导出** |
| **本质** | **前缀的一部分**（拼在 IPv4 前缀前） | **BGP 扩展团体属性** |
| **数量** | **每个 VRF 一个** | **每个 VRF 可多个**（import/export 各自可多个） |
| **决定什么** | 唯一性 | ★ **VPN 的拓扑结构** |
| **类比** | **身份证号**（保证不重名） | **门禁卡**（决定能进哪些门） |

## RD：解决地址重叠

```
   客户 A：192.168.1.0/24
        ↓ 加上 RD 65000:100
   ★ VPNv4 前缀：65000:100:192.168.1.0/24 ★

   客户 B：192.168.1.0/24
        ↓ 加上 RD 65000:200
   ★ VPNv4 前缀：65000:200:192.168.1.0/24 ★
   
   → 两条完全不同的 VPNv4 路由，可以共存于同一个 MP-BGP 表
```

**没有 RD 会怎样**：MP-BGP 表里只能有一条 `192.168.1.0/24`，第二个客户的路由会被覆盖。

**VPNv4 前缀 = RD (64位) + IPv4 (32位) = 96 位**

## RT：决定谁能看到谁

```
   PE1 的 VRF CUSTOMER-A：
     export RT 65000:100      ← 我导出的路由打上这个"标签"
     import RT 65000:100      ← 我导入带这个"标签"的路由
   
   PE2 的 VRF CUSTOMER-A：
     export RT 65000:100
     import RT 65000:100
        ↓
   ★ RT 匹配 → 路由互通 ✓ ★
```

**★ RT 决定 VPN 拓扑（关键理解）**：

**全互联（Full Mesh）**：
```cisco
! 所有站点都是
route-target both 65000:100
! 结果：任意两个站点直接互通
```

**中心辐射（Hub-and-Spoke）**：
```cisco
! Hub（总部）
route-target export 65000:1000
route-target import 65000:2000

! Spoke（分支）
route-target export 65000:2000
route-target import 65000:1000        ← ★ 只导入 Hub 的
```
**效果**：
```
   Spoke-A → Hub → Spoke-B    ✅ 可以
   Spoke-A ─X→ Spoke-B        ❌ 不可以（RT 不匹配）
   
   ★ 所有分支间流量必须经过总部 ★
   → 总部可以做集中安全检查、审计、流量控制
```

**Extranet（跨客户共享服务）**：
```cisco
! 共享服务 VRF（比如公共的 DNS/NTP 服务器）
vrf definition SHARED-SERVICES
 rd 65000:999
 address-family ipv4
  route-target export 65000:999

! 客户 A 也导入共享服务的 RT
vrf definition CUSTOMER-A
 rd 65000:100
 address-family ipv4
  route-target both 65000:100
  route-target import 65000:999       ← ★ 额外导入共享服务
```

## 为什么两个都需要

**只有 RD 不行**：
```
   RD 让前缀唯一了，但【谁能看到谁】没有定义
   → 要么全部互通（失去隔离）
   → 要么全部不通（失去意义）
```

**只有 RT 不行**：
```
   两个客户都用 192.168.1.0/24
   → MP-BGP 表里只能存一条
   → ★ 地址冲突，无法共存 ★
```

**★ 最重要的考点：RD 不能用来控制路由的导入导出。**

常见误解："RD 不同就不能互通" —— **错**。

```
   VRF-A: rd 65000:100, rt both 65000:999
   VRF-B: rd 65000:200, rt both 65000:999
        ↓
   ★ RD 不同，但 RT 相同 → 互通 ★
   
   VRF-C: rd 65000:100, rt both 65000:111
   VRF-D: rd 65000:100, rt both 65000:222
        ↓
   ★ RD 相同，但 RT 不同 → 不互通 ★
```

**是否互通完全由 RT 决定，与 RD 无关。**

**实践中的 RD 规划建议**：

| 策略 | 说明 |
|:--|:--|
| **每个 VRF 每台 PE 用不同的 RD** | 便于 BGP 做多路径（同一前缀不同 RD 是不同的 VPNv4 路由，可以同时存在） |
| 每个 VRF 全网用同一个 RD | 配置简单，但失去多路径能力 |

**推荐格式**：`<AS号>:<客户ID><PE编号>`，比如 `65000:10001`（客户 100，PE 01）。

**验证**：
```cisco
PE1# show vrf detail CUSTOMER-A
VRF CUSTOMER-A (VRF Id = 1); default RD ★ 65000:100 ★
Address family ipv4 unicast (Table ID = 0x1):
  Export VPN route-target communities
    ★ RT:65000:100 ★
  Import VPN route-target communities
    ★ RT:65000:100 ★
```
</details>

**3.** MP-BGP 邻居 Established 了，但 VPNv4 路由收不到。检查什么？

<details><summary>答案</summary>

**三个最常见的原因，按顺序检查**：

## ① 缺 `neighbor X activate`（在 vpnv4 地址族下）

```cisco
! ❌ 只在全局配了邻居，没在 vpnv4 地址族下激活
PE1(config)# router bgp 65000
PE1(config-router)# neighbor 2.2.2.2 remote-as 65000
PE1(config-router)# neighbor 2.2.2.2 update-source Loopback0
! 邻居能建立（IPv4 地址族），但不交换 VPNv4 路由

! ✅ 正确
PE1(config-router)# address-family vpnv4
PE1(config-router-af)#  ★ neighbor 2.2.2.2 activate ★
```

**验证**：
```cisco
PE1# show bgp vpnv4 unicast all summary
Neighbor    V   AS  MsgRcvd MsgSent TblVer InQ OutQ Up/Down State/PfxRcd
2.2.2.2     4 65000    125     128      8   0    0 00:15:12       0
                                                                  ↑
                                                    收到 0 条 → 有问题

! 如果邻居根本不出现在这个列表里 → 没 activate
```

## ② 缺 `send-community both`（★ 最隐蔽）

**RT 是 BGP 的【扩展团体属性】。Cisco 默认不发送团体属性！**

```cisco
! ❌ 缺失
PE1(config-router-af)# neighbor 2.2.2.2 activate
! → 路由发出去了，但 RT 属性没带上
! → 对端收到路由但不知道该导入哪个 VRF → 丢弃

! ✅ 正确
PE1(config-router-af)# neighbor 2.2.2.2 send-community both
!                                                     ↑
!                              both = standard + extended
!                              （RT 是 extended community）
```

**验证**：
```cisco
PE1# show run | section address-family vpnv4
 address-family vpnv4
  neighbor 2.2.2.2 activate
  neighbor 2.2.2.2 send-community both        ← 必须有

PE1# show bgp vpnv4 unicast all neighbors 2.2.2.2 | include Community
  Community attribute sent to this neighbor (both)
```

## ③ RT 不匹配

**这种情况的特征：VPNv4 表里【有】路由，但 VRF 路由表里【没有】。**

```cisco
PE2# show bgp vpnv4 unicast all | include 192.168.1.0
Route Distinguisher: 65000:100
 *>i 192.168.1.0/24   1.1.1.1  ...
! ★ BGP 表里有 ★

PE2# show ip route vrf CUSTOMER-A | include 192.168.1.0
! ★ VRF 路由表里没有 ★
```

**这个对比是判断 RT 问题的黄金标志。**

**排查**：
```cisco
PE1# show vrf detail CUSTOMER-A | include RT
  Export VPN route-target communities
    RT:65000:100
  Import VPN route-target communities
    RT:65000:100

PE2# show vrf detail CUSTOMER-A | include RT
  Export VPN route-target communities
    RT:65000:100
  Import VPN route-target communities
    ★ RT:65000:999 ★                         ← 找到了，写错了
```

**修复**：
```cisco
PE2(config)# vrf definition CUSTOMER-A
PE2(config-vrf)#  address-family ipv4
PE2(config-vrf-af)#   no route-target import 65000:999
PE2(config-vrf-af)#   route-target import 65000:100
```

## 完整排查流程

```cisco
! ① MP-BGP 邻居状态
PE1# show bgp vpnv4 unicast all summary
! → 邻居不在列表里 = 没 activate
! → PfxRcd 是 0 = 收不到路由

! ② 检查配置
PE1# show run | section address-family vpnv4
! → 有 activate 吗？有 send-community both 吗？

! ③ 我发出去了吗
PE1# show bgp vpnv4 unicast all neighbors 2.2.2.2 advertised-routes
! → 空 = 本地没有路由可发（检查 VRF 里有没有路由）

! ④ 对方收到了吗
PE2# show bgp vpnv4 unicast all
! → 有路由 = 传输没问题，问题在 RT
! → 没路由 = 传输有问题（activate / send-community）

! ⑤ RT 对比
PE1# show vrf detail CUSTOMER-A | include RT
PE2# show vrf detail CUSTOMER-A | include RT

! ⑥ 检查是否被 route-map 过滤
PE1# show run | include neighbor 2.2.2.2 route-map
```

## 其他可能原因

| 原因 | 检查 |
|:--|:--|
| VRF 里本来就没有路由 | `show ip route vrf CUSTOMER-A` |
| PE-CE 路由没重分发进 BGP | `show run \| sec address-family ipv4 vrf` |
| 被 route-map 过滤 | `show route-map` |
| `maximum-prefix` 触发 | `show logging \| inc PFX` |
| iBGP 水平分割（没用 RR） | 检查 PE 之间是否全互联 |

## 大规模部署的建议

PE 数量多时（>10 台），**用 Route Reflector**：
```cisco
! RR 上
RR(config-router)# address-family vpnv4
RR(config-router-af)#  neighbor <PE1> activate
RR(config-router-af)#  neighbor <PE1> route-reflector-client
RR(config-router-af)#  neighbor <PE1> send-community both

! ★ RR 需要保留所有 RT（不做 RT 过滤）
RR(config-router-af)#  no bgp default route-target filter
```

**`no bgp default route-target filter` 很重要**：默认情况下，PE 会丢弃"RT 不匹配任何本地 VRF"的 VPNv4 路由（节省内存）。但 RR 本身不配置 VRF，如果不关掉这个过滤，**RR 会把所有路由都丢掉**。
</details>

**4.** 为什么 MPLS 骨干网的 MTU 必须加大？

<details><summary>答案</summary>

**因为 MPLS 标签是【额外插入】的，会让包变长。**

**开销计算**：
```
   每层 MPLS 标签 = 4 字节
   
   L3VPN 通常两层标签：
   · 外层：传输标签（LDP 分发）
   · 内层：VPN 标签（MP-BGP 分发）
        ↓
   ★ 8 字节开销 ★
   
   原始包 1500 字节 + 8 = 1508 字节
        ↓
   如果骨干网接口 MTU 是 1500 → ★ 超了，被丢弃 ★
```

**更多层的场景**：

| 场景 | 标签层数 | 开销 |
|:--|:--|:--|
| 纯 MPLS 转发 | 1 层 | 4 字节 |
| **L3VPN** | **2 层** | **8 字节** |
| L3VPN + TE | 3 层 | 12 字节 |
| L3VPN + TE + FRR | 4 层 | 16 字节 |
| L2VPN/VPLS | 2-3 层 | 8-12 字节 |

**症状（和 GRE/VXLAN 的 MTU 问题完全一样）**：
- ✅ ping 通（小包）
- ✅ SSH 能连（小包）
- ❌ **网页打不开、大文件传不动、数据库同步失败**（大包）
- ❌ 症状随机，难以复现

**解法**：
```cisco
! ── 骨干网【所有】接口都要加大 MTU ──
P1(config)# interface GigabitEthernet0/1
P1(config-if)# mtu 1600                       ! 至少 1508，建议 1600+
P1(config-if)# mpls mtu 1600

! ── 或直接开巨帧（推荐）──
P1(config-if)# mtu 9000
P1(config-if)# mpls mtu 9000

! ── 验证 ──
PE1# ping 2.2.2.2 size 1500 df-bit source Loopback0
!!!!!                                          ← 通了 ✓

PE1# show mpls interfaces detail
Interface GigabitEthernet0/1:
        IP labeling enabled (ldp)
        LSP Tunnel labeling not enabled
        ★ MPLS operational ★
        MTU = 1600
```

**⚠️ 必须全路径一致**：

```
   CE1 ── PE1 ── P1 ── P2 ── P3 ── PE2 ── CE2
                  ↑
            如果只有这一台的 MTU 是 1500
                  ↓
        ★ 整条路径都受影响 ★
```

**每一台 P 和 PE 的每一个骨干接口都要配。**

**为什么推荐直接开巨帧（9000）**：
1. 一劳永逸，未来叠加更多标签（TE、FRR）也不怕
2. 巨帧本身提升大流量场景的性能
3. 骨干网都是自己的设备，没有兼容性问题

**检查工具**：
```cisco
! 逐跳测试 MTU
PE1# ping 2.2.2.2 size 1500 df-bit source Loopback0
PE1# ping 10.0.12.2 size 1600 df-bit

! MPLS 专用的路径测试
PE1# trace mpls ip 2.2.2.2/32
Tracing MPLS Label Switched Path to 2.2.2.2/32, timeout is 2 seconds
  0 10.0.12.1 MRU 1600 [Labels: 16 Exp: 0]
R 1 10.0.12.2 MRU 1600 [Labels: 17 Exp: 0] 4 ms
                   ↑ MRU = Maximum Receive Unit，显示每跳的 MTU
R 2 10.0.23.3 MRU 1500 [Labels: implicit-null] 8 ms
                   ↑ ★ 这一跳 MTU 小了，找到问题
```

**`trace mpls ip` 的 MRU 字段是排查 MPLS MTU 问题最直接的工具。**

**如果实在无法加大骨干 MTU（比如运营商链路限制）**：
```cisco
! 在 PE-CE 接口上调整 MSS
PE1(config)# interface GigabitEthernet0/2
PE1(config-if)# ip tcp adjust-mss 1400
```
这只能解决 TCP 流量，UDP（VoIP、视频）依然会有问题。**治标不治本。**

> **★ MPLS 部署检查清单第一条永远是 MTU**（和 VXLAN 一样）。我见过太多"L3VPN 上线后业务时好时坏"的案例，最后都是某台 P 路由器的某个接口忘了配 MTU。
</details>

**5.** 客户在 CE 上 traceroute，能看到运营商内部的 P 路由器吗？怎么隐藏？

<details><summary>答案</summary>

**默认能看到**，因为 MPLS 默认开启 **TTL 传播（propagate-ttl）**。

**默认行为（`mpls ip propagate-ttl` 开启）**：
```
   PE1 打标签时，★ 把 IP 头的 TTL 复制到 MPLS 标签的 TTL 字段 ★
        ↓
   每经过一台 P 路由器，标签 TTL 减 1
        ↓
   TTL 到 0 时，P 路由器回 ICMP Time Exceeded
        ↓
   ★ 客户的 traceroute 看到了运营商的内部 IP ★
```

```cisco
CE1# traceroute 192.168.2.10
  1 192.168.1.1  4 msec    (PE1)
  2 10.0.12.2    8 msec    ★ P1 —— 运营商内部 IP 暴露
  3 10.0.23.3    12 msec   ★ P2 —— 暴露
  4 192.168.2.1  16 msec   (PE2)
  5 192.168.2.10 20 msec
```

**这是运营商不希望的**：
- 暴露内部拓扑结构（有多少跳、什么架构）
- 暴露内部 IP 编址方案
- **给攻击者提供侦察信息**
- 客户可能据此质疑"为什么绕这么多跳"

**隐藏方法**：
```cisco
! 在【所有 PE】上配置
PE1(config)# no mpls ip propagate-ttl
PE2(config)# no mpls ip propagate-ttl
```

**效果**：
```
   PE1 打标签时，★ 标签的 TTL 设为固定值 255 ★（不复制 IP 的 TTL）
        ↓
   经过 P 路由器时标签 TTL 递减，但因为初始值是 255，不会到 0
        ↓
   到达 PE2 时，把【原始的 IP TTL 减 1】（整个 MPLS 域算一跳）
        ↓
   ★ 客户看到的是：骨干网只有一跳 ★
```

```cisco
CE1# traceroute 192.168.2.10
  1 192.168.1.1  4 msec    (PE1)
  2 192.168.2.1  16 msec   ★ 直接到 PE2，骨干网被隐藏
  3 192.168.2.10 20 msec
```

**更精细的控制**：
```cisco
! 只对转发的流量隐藏，本地产生的流量（运营商自己排障用）保留 TTL 传播
PE1(config)# no mpls ip propagate-ttl forwarded
!                                     ↑
!                        只影响【转发】的流量

! 只对本地流量隐藏
PE1(config)# no mpls ip propagate-ttl local
```

**这个配置让运营商自己排障时依然能看到完整路径，而客户看不到。**

**运营商内部的排障工具**（即使关闭了 TTL 传播）：
```cisco
! MPLS 专用的路径追踪
PE1# trace mpls ip 2.2.2.2/32
Tracing MPLS Label Switched Path to 2.2.2.2/32, timeout is 2 seconds

  0 10.0.12.1 MRU 1600 [Labels: 16 Exp: 0]
R 1 10.0.12.2 MRU 1600 [Labels: 17 Exp: 0] 4 ms
R 2 10.0.23.3 MRU 1600 [Labels: implicit-null] 8 ms
! 依然能看到完整路径 ✓

! MPLS Ping（验证 LSP 完整性）
PE1# ping mpls ipv4 2.2.2.2/32
Sending 5, 100-byte MPLS Echos to 2.2.2.2/32,
     timeout is 2 seconds, send interval is 0 msec:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 4/6/8 ms
```

**`ping mpls` 和 `trace mpls` 是 MPLS 专用的 OAM 工具**，它们验证的是**标签转发路径（LSP）**是否完整，而不是 IP 可达性。

**为什么需要专门的 MPLS OAM**：
```
   可能出现的情况：
   · IP 层通（IGP 路由正常）
   · 但 LSP 断了（LDP 标签没分发下来）
        ↓
   ★ 普通 ping 通，但 MPLS 流量不通 ★
   
   → 必须用 ping mpls / trace mpls 才能发现
```

**部署建议**：
```cisco
! 所有 PE 上标准配置
no mpls ip propagate-ttl forwarded

! 保留运营商自己的排障能力
! （local 流量依然传播 TTL）
```
</details>

---

**上一章** ← [04 路由重分发与路由策略](04-路由重分发与路由策略.md) ｜ **下一章** → [06 DMVPN](06-DMVPN.md)
