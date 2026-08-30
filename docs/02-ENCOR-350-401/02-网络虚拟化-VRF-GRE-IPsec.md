# 02 · 网络虚拟化：VRF / GRE / IPsec

## ① 这章解决什么问题

三个真实需求：

1. **"财务网和办公网必须完全隔离，但要共用同一批交换机和路由器。"**
   → VLAN 能隔离二层，但路由器上只有一张路由表，三层还是通的。怎么办？

2. **"分公司和总部之间要跑 OSPF，但中间隔着互联网。"**
   → OSPF 是链路本地协议，用组播，跨不了互联网。怎么办？

3. **"上面那条链路走的是公网，数据不能被人看到。"**
   → 怎么加密？

答案分别是 **VRF、GRE、IPsec**。而它们组合起来（GRE over IPsec + VRF），就是企业 WAN 互联的经典方案，也是 DMVPN 和 SD-WAN 的基础。

**这一章是"抽象层层叠加"主线的第二站**：VLAN 抽象了二层，VRF 抽象了三层，隧道抽象了链路。

---

## ② VRF：三层的隔离

### 2.1 VRF 是什么

**VRF (Virtual Routing and Forwarding) = 一台路由器上的多张独立路由表。**

```
   没有 VRF：                        有 VRF：
   
   ┌──────────────────┐            ┌──────────────────────────┐
   │   一张路由表      │            │  VRF: FINANCE            │
   │                  │            │  ┌────────────────────┐  │
   │ 10.1.0.0/24 → Gi0│            │  │ 10.1.0.0/24 → Gi0  │  │
   │ 10.2.0.0/24 → Gi1│            │  └────────────────────┘  │
   │ 10.3.0.0/24 → Gi2│            │                          │
   │                  │            │  VRF: OFFICE             │
   │  ↑ 全部互通       │            │  ┌────────────────────┐  │
   └──────────────────┘            │  │ 10.2.0.0/24 → Gi1  │  │
                                   │  │ 10.3.0.0/24 → Gi2  │  │
                                   │  └────────────────────┘  │
                                   │   ↑ 两张表完全独立        │
                                   └──────────────────────────┘
```

**核心价值**：
- **完全的路由隔离**。两个 VRF 之间**默认不可能通信**——不是 ACL 拦着，而是**路由表里根本没有对方的条目**。
- **地址空间可以重叠**。VRF-A 和 VRF-B 都可以有 `192.168.1.0/24`，互不冲突（这在多租户和企业并购场景中极其有用）。

**类比**：
- **VLAN** = 二层的虚拟化（一台交换机变成多台虚拟交换机）
- **VRF** = 三层的虚拟化（一台路由器变成多台虚拟路由器）

### 2.2 VRF-Lite vs MPLS L3VPN

| | **VRF-Lite** | **MPLS L3VPN** |
|:--|:--|:--|
| 范围 | **单台设备本地** | **跨整个 MPLS 骨干网** |
| 跨设备传播 | 每台设备都要手工配 VRF，链路要用子接口/独立链路 | **MP-BGP 自动传播 VRF 路由** |
| 标识 | 只有 VRF 名字 | **RD（区分路由）+ RT（控制导入导出）** |
| 复杂度 | 简单 | 复杂 |
| 适用 | 企业内部隔离（财务/访客/生产） | **运营商、大型企业骨干** |

**本章讲 VRF-Lite**（ENCOR 范围），MPLS L3VPN 在 [ENARSI 第 5 章](../03-ENARSI-300-410/05-MPLS与L3VPN.md)。

### 2.3 VRF-Lite 配置

```cisco
! ── 定义 VRF（现代语法，IOS 15.x+）──
R1(config)# vrf definition FINANCE
R1(config-vrf)#  description ### 财务专网 ###
R1(config-vrf)#  address-family ipv4
R1(config-vrf-af)#  exit-address-family

R1(config)# vrf definition OFFICE
R1(config-vrf)#  address-family ipv4

! ── 把接口划入 VRF ──
R1(config)# interface GigabitEthernet0/0
R1(config-if)# vrf forwarding FINANCE          ! ⚠️ 这条命令会清除接口上已有的 IP！
R1(config-if)# ip address 10.1.31.1 255.255.255.0   ! 所以要重新配

R1(config)# interface GigabitEthernet0/1
R1(config-if)# vrf forwarding OFFICE
R1(config-if)# ip address 10.1.10.1 255.255.255.0

! ── 子接口方式（一个物理口承载多个 VRF）──
R1(config)# interface GigabitEthernet0/2.31
R1(config-subif)# encapsulation dot1Q 31
R1(config-subif)# vrf forwarding FINANCE
R1(config-subif)# ip address 10.1.131.1 255.255.255.0

R1(config)# interface GigabitEthernet0/2.10
R1(config-subif)# encapsulation dot1Q 10
R1(config-subif)# vrf forwarding OFFICE
R1(config-subif)# ip address 10.1.110.1 255.255.255.0
```

> ⚠️ **最容易踩的坑**：`vrf forwarding` 命令会**立即清除该接口上已配置的 IP 地址**。IOS 会提示：
> ```
> % Interface GigabitEthernet0/0 IP address 10.1.31.1 removed due to enabling VRF FINANCE
> ```
> **正确顺序：先划 VRF，再配 IP。** 在生产环境远程操作时忘了这点 = 立刻断连。

### 2.4 VRF 内的路由与命令

**所有涉及路由和转发的命令，都必须加 `vrf` 关键字**：

```cisco
! ── 静态路由 ──
R1(config)# ip route vrf FINANCE 10.1.32.0 255.255.255.0 10.1.31.2

! ── 动态路由（OSPF）──
R1(config)# router ospf 10 vrf FINANCE
R1(config-router)# router-id 10.1.255.1
R1(config-router)# network 10.1.31.0 0.0.0.255 area 0

R1(config)# router ospf 20 vrf OFFICE          ! 不同 VRF 用不同进程
R1(config-router)# router-id 10.1.255.1        ! Router ID 可以相同（不同进程互不干扰）
R1(config-router)# network 10.1.10.0 0.0.0.255 area 0

! ── EIGRP（用地址族语法）──
R1(config)# router eigrp CORP
R1(config-router)# address-family ipv4 vrf FINANCE autonomous-system 100
R1(config-router-af)#  network 10.1.31.0 0.0.0.255

! ── BGP ──
R1(config)# router bgp 65001
R1(config-router)# address-family ipv4 vrf FINANCE
R1(config-router-af)#  neighbor 10.1.31.2 remote-as 65002

! ── 查看与测试（★ 都要加 vrf）──
R1# show ip route vrf FINANCE
R1# show ip route vrf OFFICE
R1# show ip interface brief vrf FINANCE
R1# show ip arp vrf FINANCE
R1# ping vrf FINANCE 10.1.31.2
R1# traceroute vrf FINANCE 10.1.32.10
R1# telnet 10.1.31.2 /vrf FINANCE
R1# show vrf
R1# show vrf detail
```

> **排障第一课**：在有 VRF 的设备上，`show ip route` 只显示 **global 路由表**（不属于任何 VRF 的）。你的 VRF 路由**不会出现在这里**。忘记加 `vrf FINANCE` 会让你以为"路由丢了"，白白排查半天。
>
> **口诀：有 VRF 的设备，任何 show/ping/traceroute 都先问一句"哪个 VRF？"**

### 2.5 VRF 之间的受控互访（路由泄露）

两个 VRF 默认不通，但实际业务往往需要"大部分隔离，少量互访"。比如财务 VRF 需要访问共享的 DNS 服务器。

**方法 1：静态路由泄露（VRF-Lite 场景最常用）**

```cisco
! 从 FINANCE 泄露一条路由到 global 表
R1(config)# ip route vrf FINANCE 10.1.30.53 255.255.255.255 GigabitEthernet0/3 10.1.30.1 global
                                                                                        ↑↑↑↑↑↑
                                                              下一跳在 global 表中查找

! 反向：从 global 泄露到 FINANCE
R1(config)# ip route 10.1.31.0 255.255.255.0 GigabitEthernet0/0 10.1.31.2
```

**方法 2：用 BGP + Route-Target（需要 MP-BGP，更灵活）**
```cisco
R1(config)# vrf definition FINANCE
R1(config-vrf)#  rd 65001:31
R1(config-vrf)#  address-family ipv4
R1(config-vrf-af)#   route-target export 65001:31
R1(config-vrf-af)#   route-target import 65001:31
R1(config-vrf-af)#   route-target import 65001:99      ! 导入共享服务 VRF 的路由
```

**方法 3：物理回环（Hairpin，最笨但最可控）**

用两个物理接口，一个在 VRF-A，一个在 VRF-B，用网线直连（或用两个逻辑接口 loopback pair）。所有 VRF 间流量强制走这条链路，**中间可以串防火墙**。

> **推荐做法**：**VRF 隔离 + 防火墙做互访控制**。VRF 保证"默认不通"，需要互通的流量必须经过防火墙，有完整的策略和日志。这是零信任分段的基础形态。

---

## ③ GRE：把三层网络"隧道化"

### 3.1 GRE 是什么

**GRE (Generic Routing Encapsulation) = 把一个 IP 包整个塞进另一个 IP 包里。**

```
原始包：
┌─────────┬──────────────┐
│ IP 头    │    载荷       │   源 10.1.1.10 → 目的 10.2.1.10（私网）
└─────────┴──────────────┘

GRE 封装后：
┌──────────┬─────────┬─────────┬──────────────┐
│ 新 IP 头  │ GRE 头  │ 原 IP 头 │    载荷       │
│ 20 字节   │ 4 字节  │         │              │
└──────────┴─────────┴─────────┴──────────────┘
     ↑
  源 202.1.1.1 → 目的 203.2.2.2（公网，可路由）

总开销：24 字节 → MTU 从 1500 降到 1476
```

**GRE 解决的三个问题**：

| 问题 | GRE 怎么解决 |
|:--|:--|
| **私网地址跨公网** | 把私网包封装进公网包 |
| **动态路由协议跨公网** | OSPF/EIGRP 的组播被封装成单播，能跨互联网 |
| **非 IP 协议传输** | GRE 可以封装任意三层协议（IPv6、IPX、MPLS） |

> **最重要的一点**：**GRE 让两个远隔千里的路由器"看起来像直连"**。有了这个虚拟的直连链路，动态路由协议就能正常工作了。

### 3.2 GRE 配置

```cisco
! ── R1（总部）──
R1(config)# interface Tunnel0
R1(config-if)# description ### GRE to Branch ###
R1(config-if)# ip address 172.16.0.1 255.255.255.252    ! 隧道内部地址（私有）
R1(config-if)# tunnel source GigabitEthernet0/1         ! 或直接写公网 IP
R1(config-if)# tunnel destination 203.2.2.2             ! 对端的公网 IP
R1(config-if)# tunnel mode gre ip                       ! 默认就是这个，可省略
R1(config-if)# ip mtu 1400                              ! ★ 重要
R1(config-if)# ip tcp adjust-mss 1360                   ! ★ 更重要
R1(config-if)# keepalive 10 3                           ! 隧道保活，10秒一次，3次失败则 down

! ── R2（分支）──
R2(config)# interface Tunnel0
R2(config-if)# ip address 172.16.0.2 255.255.255.252
R2(config-if)# tunnel source GigabitEthernet0/1
R2(config-if)# tunnel destination 202.1.1.1
R2(config-if)# ip mtu 1400
R2(config-if)# ip tcp adjust-mss 1360
R2(config-if)# keepalive 10 3

! ── 在隧道上跑动态路由 ──
R1(config)# router ospf 1
R1(config-router)# network 172.16.0.0 0.0.0.3 area 0    ! 隧道地址进 OSPF
R1(config-router)# network 10.1.0.0 0.0.255.255 area 0  ! 内网网段

! ── 查看 ──
R1# show interfaces Tunnel0
R1# show ip interface brief | include Tunnel
```

### 3.3 GRE 的两个经典陷阱

#### 陷阱 1：递归路由（Recursive Routing）

**症状**：隧道反复 up/down，日志刷屏：
```
%TUN-5-RECURDOWN: Tunnel0 temporarily disabled due to recursive routing
```

**原因**：**去往隧道目的地址的路由，被学到了隧道内部**。

```
1. Tunnel0 的 destination 是 203.2.2.2
2. R1 通过 Tunnel0 跑 OSPF，学到了一条 "203.2.2.0/24 via Tunnel0"
3. 于是 R1 要访问 203.2.2.2，得先走 Tunnel0
4. 但 Tunnel0 要建立，又得先能访问 203.2.2.2
5. → 死循环，隧道 down
6. 隧道 down 后路由消失，隧道又 up
7. → 无限震荡
```

**解决方案**：

```cisco
! 方案 1（最直接）：不要把隧道目的地址所在网段通告进跨隧道的路由协议
! 检查一下 network 语句有没有把公网网段带进去

! 方案 2：加一条更精确的静态路由到隧道目的地址
R1(config)# ip route 203.2.2.2 255.255.255.255 202.1.1.254
!            用 /32 主机路由，最长匹配，永远优先于隧道学到的路由

! 方案 3（最优雅）：把 Underlay（公网路由）和 Overlay（隧道内路由）放在不同 VRF
R1(config)# interface Tunnel0
R1(config-if)# tunnel vrf UNDERLAY          ! 隧道的外层封装在 UNDERLAY VRF 中查路由
R1(config-if)# vrf forwarding OVERLAY       ! 隧道内部流量属于 OVERLAY VRF
```

> **方案 3 是 DMVPN 和 SD-WAN 的标准做法**：用 VRF 把"底层传输网"和"业务网"彻底分开，从架构上杜绝递归路由。这也是 VRF 和隧道结合的典型场景。

#### 陷阱 2：MTU 问题

**症状**：隧道 up，ping 通，SSH 能连，但**网页打不开、大文件传不动**。

**原因**：GRE 加了 24 字节头，实际可用 MTU 变成 1476。应用发的 1500 字节包（带 DF 位）到隧道口装不下，被丢弃。如果路径上 ICMP 被拦，源端收不到 "Fragmentation Needed" 通知，就形成 **PMTUD 黑洞**。

**解决**：
```cisco
R1(config-if)# ip mtu 1400                  ! 隧道接口的 IP MTU
R1(config-if)# ip tcp adjust-mss 1360       ! ★ 关键：改写经过的 TCP SYN 的 MSS
```

**两者的区别（考点）**：

| 命令 | 作用对象 | 机制 |
|:--|:--|:--|
| **`ip mtu 1400`** | 所有 IP 流量 | 告诉路由器"从这个接口出去的包最大 1400"，超过就分片或丢弃 |
| **`ip tcp adjust-mss 1360`** | **只对 TCP** | **改写经过的 TCP SYN 包里的 MSS 选项**，让两端主动使用更小的段 |

**为什么两个都要配**：
- `ip mtu` 是被动的——包已经太大了才处理（分片，性能差；或丢弃 + 发 ICMP，可能被拦）
- `ip tcp adjust-mss` 是主动的——**从源头就让两端不产生大包**，最高效，且不依赖 ICMP

**数值怎么定**：
```
ip mtu = 物理MTU − 封装开销
       = 1500 − 24 (GRE) = 1476    → 保守起见设 1400

ip tcp adjust-mss = ip mtu − 40 (IP头20 + TCP头20)
                  = 1400 − 40 = 1360
```

**如果还有 IPsec**：
```
GRE over IPsec 开销约 82 字节 → ip mtu 1400, adjust-mss 1360（这组值有足够余量）
```

> **实战建议**：**直接用 `ip mtu 1400` + `ip tcp adjust-mss 1360` 这组值**。它对 GRE、IPsec、GRE over IPsec 都留了足够余量，不用每次精确计算。宁可损失一点点效率，也不要留 MTU 黑洞。

---

## ④ IPsec：加密

### 4.1 IPsec 的两个协议 + 两种模式

**两个协议**：

| 协议 | 协议号 | 提供 | 能穿 NAT？ |
|:--|:--|:--|:--|
| **AH** (Authentication Header) | **51** | 完整性 + 认证，**不加密** | ❌ **不能**（它校验整个 IP 头，NAT 改了头就校验失败） |
| **ESP** (Encapsulating Security Payload) | **50** | **加密** + 完整性 + 认证 | ⚠️ 需要 NAT-T |

> **实战中只用 ESP**。AH 不加密，且无法穿越 NAT，基本已被淘汰。考试里 AH 只作为对比出现。

**两种模式**：

```
传输模式 (Transport Mode)：
┌────────┬─────────┬──────────┐
│ 原IP头  │ ESP 头  │  载荷     │   原 IP 头保留，只加密载荷
└────────┴─────────┴──────────┘   用于：主机到主机、GRE over IPsec

隧道模式 (Tunnel Mode)：
┌────────┬─────────┬────────┬──────────┐
│ 新IP头  │ ESP 头  │ 原IP头  │  载荷     │   整个原包被加密
└────────┴─────────┴────────┴──────────┘   用于：站点到站点 VPN
```

### 4.2 IKE：协商密钥的过程

IPsec 需要双方协商加密算法和密钥，这由 **IKE (Internet Key Exchange)** 完成。

**IKEv1 两个阶段**：

| 阶段 | 目的 | 协商内容 | 端口 |
|:--|:--|:--|:--|
| **Phase 1 (ISAKMP SA)** | 建立一条**安全的管理通道** | 加密算法、哈希、认证方式、DH 组、生存期 | **UDP 500** |
| **Phase 2 (IPsec SA)** | 在管理通道里协商**实际的数据加密参数** | 转换集（transform-set）、感兴趣流、PFS | 同上 |

**Phase 1 的两种模式**：
- **Main Mode（主模式）**：6 个消息，隐藏身份，更安全，用于固定 IP
- **Aggressive Mode（野蛮模式）**：3 个消息，更快，但身份信息明文，用于动态 IP

**IKEv2**：把两个阶段简化成 4 个消息，更快、更安全、原生支持 NAT-T 和 MOBIKE。**新部署优先用 IKEv2。**

**必须完全匹配的参数（Phase 1）**：
1. 加密算法（AES-256 等）
2. 哈希算法（SHA-256 等）
3. 认证方式（预共享密钥 / 证书）
4. **DH 组**（Diffie-Hellman，14/19/20/21）
5. 生存期（可以不同，取较小值）

> **考点口诀：HAGLE**
> **H**ash、**A**uthentication、**G**roup (DH)、**L**ifetime、**E**ncryption

### 4.3 NAT-T（NAT 穿越）

**问题**：ESP 是协议号 50，**没有端口号**。PAT 靠端口区分会话，遇到 ESP 就无能为力。

**解法**：**NAT-T** 把 ESP 封装进 **UDP 4500**：

```
无 NAT：  IP | ESP | 数据
有 NAT-T：IP | UDP:4500 | ESP | 数据
                 ↑ 有端口了，PAT 可以正常工作
```

**自动检测**：IKE 协商时双方会互发 NAT-Discovery 载荷，如果检测到中间有 NAT，自动切换到 UDP 4500。

**ACL 必须放行三样**：
```cisco
permit udp any any eq 500        ! IKE
permit udp any any eq 4500       ! NAT-T
permit esp any any               ! ESP（无 NAT 时用）
```

> **漏放 ESP（协议号 50）是 VPN 建不起来的经典原因**。很多人只放了 UDP 500/4500，因为"ESP 没有端口所以想不起来"。

### 4.4 GRE over IPsec（企业 WAN 标准方案）

**为什么要组合**：

| 技术 | 能做什么 | 不能做什么 |
|:--|:--|:--|
| **GRE** | 封装任意协议、**支持动态路由协议和组播** | ❌ 不加密 |
| **IPsec** | 加密 | ❌ **不支持组播**，所以路由协议跑不了 |
| **GRE over IPsec** | ✅ 两者兼得 | — |

**顺序很重要**：

```
GRE over IPsec（★ 正确做法）：
   先 GRE 封装，再 IPsec 加密
   [新IP | ESP | GRE | 原IP | 数据]
   → 加密的是整个 GRE 包，包括路由协议报文 ✓

IPsec over GRE（不常用）：
   先 IPsec 加密，再 GRE 封装
   [新IP | GRE | ESP | 原IP | 数据]
   → GRE 头在外面，不受保护 ✗
```

**配置（IKEv2 版本）**：

```cisco
! ══════ R1（总部）══════

! ── ① IKEv2 提案与策略 ──
R1(config)# crypto ikev2 proposal PROP-1
R1(config-ikev2-proposal)#  encryption aes-cbc-256
R1(config-ikev2-proposal)#  integrity sha256
R1(config-ikev2-proposal)#  group 14

R1(config)# crypto ikev2 policy POL-1
R1(config-ikev2-policy)#  proposal PROP-1

! ── ② 密钥环（预共享密钥）──
R1(config)# crypto ikev2 keyring KR-1
R1(config-ikev2-keyring)#  peer BRANCH
R1(config-ikev2-keyring-peer)#   address 203.2.2.2
R1(config-ikev2-keyring-peer)#   pre-shared-key MyVerySecretKey123!

! ── ③ IKEv2 profile ──
R1(config)# crypto ikev2 profile PROF-1
R1(config-ikev2-profile)#  match identity remote address 203.2.2.2 255.255.255.255
R1(config-ikev2-profile)#  authentication local pre-share
R1(config-ikev2-profile)#  authentication remote pre-share
R1(config-ikev2-profile)#  keyring local KR-1

! ── ④ IPsec 转换集（Phase 2）──
R1(config)# crypto ipsec transform-set TS-1 esp-aes 256 esp-sha256-hmac
R1(cfg-crypto-trans)#  mode transport                ! ★ GRE over IPsec 用传输模式

! ── ⑤ IPsec profile ──
R1(config)# crypto ipsec profile IPSEC-PROF
R1(ipsec-profile)#  set transform-set TS-1
R1(ipsec-profile)#  set ikev2-profile PROF-1
R1(ipsec-profile)#  set pfs group14

! ── ⑥ 应用到隧道接口 ──
R1(config)# interface Tunnel0
R1(config-if)# ip address 172.16.0.1 255.255.255.252
R1(config-if)# tunnel source GigabitEthernet0/1
R1(config-if)# tunnel destination 203.2.2.2
R1(config-if)# tunnel protection ipsec profile IPSEC-PROF   ! ★ 一条命令搞定
R1(config-if)# ip mtu 1400
R1(config-if)# ip tcp adjust-mss 1360
```

> **`tunnel protection ipsec profile` 的优雅之处**：不需要写 crypto map，不需要定义"感兴趣流" ACL——**所有经过这个隧道的流量自动被加密**。这比传统的 crypto map 方式简洁太多，也不容易出错。
>
> **为什么用 `mode transport`**：GRE 已经加了一层新 IP 头，IPsec 再用隧道模式会**再加一层 IP 头**，白白浪费 20 字节。传输模式复用 GRE 的外层 IP 头，更高效。

### 4.5 IPsec 排障

```cisco
! ── Phase 1 状态 ──
R1# show crypto ikev2 sa
R1# show crypto ikev2 sa detail
R1# show crypto isakmp sa                    ! IKEv1

! ── Phase 2 状态 ──
R1# show crypto ipsec sa
R1# show crypto session
R1# show crypto session detail

! ── 隧道保护状态 ──
R1# show crypto ipsec sa peer 203.2.2.2

! ── 调试（慎用，会刷屏）──
R1# debug crypto ikev2
R1# debug crypto ipsec
R1# debug crypto condition peer ipv4 203.2.2.2     ! 只调试特定对端

! ── 清除重建 ──
R1# clear crypto sa
R1# clear crypto ikev2 sa
```

**`show crypto ipsec sa` 的关键计数器**：

```cisco
R1# show crypto ipsec sa
  ...
  #pkts encaps: 1245, #pkts encrypt: 1245, #pkts digest: 1245
  #pkts decaps: 1198, #pkts decrypt: 1198, #pkts verify: 1198
        ↑                    ↑
   出方向计数            入方向计数
```

**诊断逻辑（非常实用）**：

| 现象 | 含义 |
|:--|:--|
| `encaps` 和 `decaps` **都在涨** | ✅ 双向正常 |
| `encaps` 涨，`decaps` **不涨** | ❌ **我发出去了，对方没回**——对端配置问题、对端 ACL 拦了、回程路由不通 |
| `encaps` **不涨** | ❌ **流量根本没进隧道**——路由问题，或 NAT 把源地址改了（见 [CCNA 篇第 6 章](../01-CCNA补齐篇/06-NAT地址转换.md)） |
| 都不涨，但 IKE SA 是 UP | ❌ Phase 1 成功但没有数据触发 Phase 2，或感兴趣流不匹配 |

---

## ⑤ 配套实验：GRE over IPsec + VRF

**拓扑**：
```
   总部 10.1.0.0/16                            分支 10.2.0.0/16
        │                                            │
     [R1] 202.1.1.1 ────── [ Internet ] ────── 203.2.2.2 [R2]
        │                                            │
        └────── Tunnel0: 172.16.0.0/30 ─────────────┘
                （GRE over IPsec，跑 OSPF）
```

### Step 1：先建纯 GRE，验证连通

```cisco
! R1
R1(config)# interface Tunnel0
R1(config-if)# ip address 172.16.0.1 255.255.255.252
R1(config-if)# tunnel source GigabitEthernet0/1
R1(config-if)# tunnel destination 203.2.2.2
R1(config-if)# ip mtu 1400
R1(config-if)# ip tcp adjust-mss 1360
R1(config-if)# keepalive 10 3

! R2 对称配置
```

**验证**：
```cisco
R1# show ip interface brief | include Tunnel
Tunnel0    172.16.0.1    YES manual  up      up          ← 必须 up/up

R1# ping 172.16.0.2
!!!!!
```

### Step 2：在隧道上跑 OSPF

```cisco
R1(config)# router ospf 1
R1(config-router)# router-id 1.1.1.1
R1(config-router)# network 172.16.0.0 0.0.0.3 area 0
R1(config-router)# network 10.1.0.0 0.0.255.255 area 0

! ★ 建议把隧道改成点对点，省掉 DR 选举
R1(config)# interface Tunnel0
R1(config-if)# ip ospf network point-to-point
```

**验证**：
```cisco
R1# show ip ospf neighbor
Neighbor ID   Pri  State     Dead Time   Address       Interface
2.2.2.2         0  FULL/  -  00:00:35    172.16.0.2    Tunnel0

R1# show ip route ospf
O   10.2.0.0/16 [110/1001] via 172.16.0.2, 00:02:15, Tunnel0     ← 学到分支路由 ✓
```

**这就是 GRE 的核心价值**：OSPF 在互联网上跑起来了。

### Step 3：加 IPsec 加密

按 4.4 节的配置在两端都配好，然后：

```cisco
R1(config)# interface Tunnel0
R1(config-if)# tunnel protection ipsec profile IPSEC-PROF
```

**验证**：
```cisco
R1# show crypto ikev2 sa
 Tunnel-id Local            Remote           fvrf/ivrf   Status
 1         202.1.1.1/500    203.2.2.2/500    none/none   READY
                                                          ↑ Phase 1 成功

R1# show crypto ipsec sa | include pkts
    #pkts encaps: 234, #pkts encrypt: 234, #pkts digest: 234
    #pkts decaps: 231, #pkts decrypt: 231, #pkts verify: 231
     ↑ 双向都在涨 = 加密正常工作
```

**抓包验证**（在公网侧）：加密前能看到 GRE 和内层 IP；加密后只能看到 ESP，内容不可读。

### Step 4：故障注入

**故障 A：递归路由**
```cisco
! 故意把公网网段通告进 OSPF
R1(config)# router ospf 1
R1(config-router)# network 202.1.1.0 0.0.0.255 area 0
R2(config)# router ospf 1
R2(config-router)# network 203.2.2.0 0.0.0.255 area 0
```
**观察**：
```
%TUN-5-RECURDOWN: Tunnel0 temporarily disabled due to recursive routing
%LINEPROTO-5-UPDOWN: Line protocol on Interface Tunnel0, changed state to down
%LINEPROTO-5-UPDOWN: Line protocol on Interface Tunnel0, changed state to up
（反复循环）
```
**修复**：
```cisco
R1(config)# ip route 203.2.2.2 255.255.255.255 202.1.1.254   ! /32 静态路由，永远优先
! 或者去掉那条 network 语句
```

**故障 B：MTU 黑洞**
```cisco
R1(config)# interface Tunnel0
R1(config-if)# no ip tcp adjust-mss 1360
R1(config-if)# no ip mtu 1400
```
**观察**：
```cisco
R1# ping 10.2.1.10 size 100
!!!!!                            ← 小包通

R1# ping 10.2.1.10 size 1500 df-bit
M.M.M                            ← 大包不通（M = 需要分片但 DF 置位）
```
从 PC 访问对端 Web 服务，页面卡住加载不完。
**修复**：加回那两条命令。

**故障 C：Phase 1 参数不匹配**
```cisco
R2(config)# crypto ikev2 proposal PROP-1
R2(config-ikev2-proposal)# group 5          ! R1 是 group 14
```
**观察**：
```cisco
R1# show crypto ikev2 sa
! 空输出，或状态一直在 IN-NEG

R1# debug crypto ikev2
IKEv2:(SESSION ID = 1,SA ID = 1):Failed to find a matching policy
```
**修复**：两端 HAGLE 五要素必须一致。

### Step 5：加上 VRF（进阶）

把业务流量放进 VRF，把公网传输放在 global 表——**从架构上根除递归路由**：

```cisco
R1(config)# vrf definition CORP
R1(config-vrf)#  address-family ipv4

R1(config)# interface Tunnel0
R1(config-if)# vrf forwarding CORP              ! 隧道内部属于 CORP VRF
R1(config-if)# ip address 172.16.0.1 255.255.255.252
R1(config-if)# tunnel source GigabitEthernet0/1
R1(config-if)# tunnel destination 203.2.2.2
R1(config-if)# tunnel vrf default                ! 外层封装在 global 表中查路由

R1(config)# interface GigabitEthernet0/0         ! 内网口
R1(config-if)# vrf forwarding CORP
R1(config-if)# ip address 10.1.1.1 255.255.255.0

R1(config)# router ospf 1 vrf CORP
R1(config-router)# network 172.16.0.0 0.0.0.3 area 0
R1(config-router)# network 10.1.0.0 0.0.255.255 area 0
```

**现在无论 OSPF 学到什么，都只在 CORP VRF 里；隧道目的地址在 global 表中查找，永远不会递归。**

**验证**：
```cisco
R1# show ip route                    ! global 表：只有公网路由
R1# show ip route vrf CORP           ! CORP 表：只有业务路由
R1# ping vrf CORP 10.2.1.10
```

---

## ⑥ 排障速查 + 三厂商对照

### 排障速查表

| 症状 | 怀疑点 | 验证命令 |
|:--|:--|:--|
| VRF 里 `show ip route` 看不到路由 | 忘加 vrf 关键字 | `show ip route vrf NAME` |
| 划 VRF 后接口 IP 没了 | `vrf forwarding` 清了 IP | 重新配 IP |
| Tunnel 一直 down | source/destination 不可达 | `ping <tunnel destination>` |
| Tunnel 反复 up/down | **递归路由** | `show logging \| inc RECURDOWN` |
| Tunnel up 但 ping 不通 | 隧道内地址/路由 | `show ip route \| inc Tunnel` |
| ping 通但网页打不开 | **MTU** | `ping size 1500 df-bit` |
| IKE SA 建不起来 | Phase 1 参数（HAGLE） | `show crypto ikev2 sa`、`debug crypto ikev2` |
| IKE 起来但 IPsec SA 没有 | Phase 2 参数 / 感兴趣流 | `show crypto ipsec sa` |
| `encaps` 涨 `decaps` 不涨 | 对端问题 / 回程 | 在对端查 `show crypto ipsec sa` |
| `encaps` 不涨 | **流量没进隧道** | 检查路由、**检查 NAT 是否 deny 了 VPN 流量** |
| 有 NAT 时 VPN 建不起来 | 缺 NAT-T / 缺 ESP 放行 | 放行 UDP 4500、协议 50 |

### 三厂商对照

| 目的 | Cisco | H3C | 华为 |
|:--|:--|:--|:--|
| 定义 VRF | `vrf definition NAME` | `ip vpn-instance NAME` | `ip vpn-instance NAME` |
| 接口划入 VRF | `vrf forwarding NAME` | `ip binding vpn-instance NAME` | `ip binding vpn-instance NAME` |
| VRF 路由表 | `show ip route vrf NAME` | `display ip routing-table vpn-instance NAME` | `display ip routing-table vpn-instance NAME` |
| VRF ping | `ping vrf NAME <ip>` | `ping -vpn-instance NAME <ip>` | `ping -vpn-instance NAME <ip>` |
| GRE 隧道 | `interface Tunnel0` + `tunnel mode gre ip` | `interface Tunnel0 mode gre` | `interface Tunnel0` + `tunnel-protocol gre` |
| 隧道源 | `tunnel source Gi0/1` | `source GigabitEthernet0/1` | `source GigabitEthernet0/0/1` |
| 隧道目的 | `tunnel destination 203.2.2.2` | `destination 203.2.2.2` | `destination 203.2.2.2` |
| MSS 调整 | `ip tcp adjust-mss 1360` | `tcp mss 1360` | `tcp adjust-mss 1360` |
| IKE 提案 | `crypto ikev2 proposal` | `ike proposal` | `ike proposal` |
| IPsec 转换集 | `crypto ipsec transform-set` | `ipsec transform-set` | `ipsec proposal` |
| 查 IKE SA | `show crypto ikev2 sa` | `display ike sa` | `display ike sa` |
| 查 IPsec SA | `show crypto ipsec sa` | `display ipsec sa` | `display ipsec sa` |

> **术语对照**：Cisco 叫 **VRF**，H3C/华为叫 **VPN-instance**。概念完全一样。

---

## ⑦ 考点提示 + 自测题

### 考点

- **VRF 实现三层隔离**，与 VLAN（二层隔离）的对比。
- **`vrf forwarding` 会清除接口 IP**。
- **所有 VRF 相关命令都要加 `vrf` 关键字**。
- **GRE 开销 24 字节**，MTU/MSS 计算。
- **递归路由**的成因和三种解法。
- **`ip mtu` vs `ip tcp adjust-mss`** 的区别。
- **GRE over IPsec 用传输模式**，为什么。
- **AH 不能穿 NAT，ESP 需要 NAT-T**。
- **HAGLE 五要素**必须匹配。
- **`encaps`/`decaps` 计数器的诊断逻辑**。

### 自测题

**1.** VRF 和 VLAN 有什么本质区别？

<details><summary>答案</summary>

| | **VLAN** | **VRF** |
|:--|:--|:--|
| 工作层次 | **二层** | **三层** |
| 隔离什么 | **广播域** | **路由表** |
| 虚拟化对象 | 一台交换机 → 多台虚拟交换机 | 一台路由器 → 多台虚拟路由器 |
| 地址重叠 | 不适用 | ✅ **支持**（不同 VRF 可用相同网段） |
| 跨设备 | 靠 Trunk（802.1Q 标签） | VRF-Lite 靠子接口；跨骨干靠 MPLS L3VPN |

**关键点：只有 VLAN 是不够的。**

如果你在三层交换机上划了 VLAN 10（财务）和 VLAN 20（办公），配了两个 SVI，然后开了 `ip routing`——**这两个 VLAN 就通了**。因为它们的网段都在同一张路由表里，路由器天然会转发。

要真正隔离，必须把 SVI 划进不同的 VRF：
```cisco
SW(config)# vrf definition FINANCE
SW(config-vrf)#  address-family ipv4

SW(config)# interface Vlan10
SW(config-if)# vrf forwarding FINANCE
SW(config-if)# ip address 10.1.10.1 255.255.255.0
```

此时 VLAN 10 的路由在 FINANCE 表里，VLAN 20 的路由在 global 表里，**两张表互不可见**，物理上就不可能路由过去。

**这比 ACL 隔离强得多**：
- ACL 是"允许转发，但拦下来" → 规则写错、顺序错、被绕过都会失效
- VRF 是"根本没有路由，转发不了" → **从机制上杜绝**

**一句话总结**：VLAN 隔离"谁能听到谁的广播"，VRF 隔离"谁能路由到谁"。
</details>

**2.** GRE 隧道反复 up/down，日志显示 `%TUN-5-RECURDOWN`。什么原因？三种解法是什么？

<details><summary>答案</summary>

**递归路由（Recursive Routing）**：去往隧道目的地址的路由，被学到了隧道内部。

**死循环过程**：
```
1. Tunnel0 destination = 203.2.2.2
2. 隧道 up 后，OSPF 通过隧道学到 "203.2.2.0/24 via Tunnel0"
3. 这条路由比原来的默认路由更具体，进了路由表
4. 现在要访问 203.2.2.2，路由表说"走 Tunnel0"
5. 但 Tunnel0 的外层封装需要先能到 203.2.2.2
6. → 无解 → 隧道 down
7. 隧道 down 后，OSPF 邻居断，那条路由消失
8. → 隧道又能 up 了 → 回到第 2 步，无限循环
```

**三种解法**：

**① 不要把公网网段通告进跨隧道的路由协议**（最直接）
```cisco
R1(config-router)# no network 202.1.1.0 0.0.0.255 area 0
```
检查 `network` 语句，尤其小心 `network 0.0.0.0 255.255.255.255 area 0` 这种偷懒写法。

**② 加一条 /32 静态路由到隧道目的地址**
```cisco
R1(config)# ip route 203.2.2.2 255.255.255.255 202.1.1.254
```
利用**最长匹配原则**：`/32` 永远比隧道学到的 `/24` 更具体，所以外层封装永远走真实的公网路径。

**③ 用 VRF 分离 Underlay 和 Overlay**（最优雅，SD-WAN 的做法）
```cisco
R1(config)# interface Tunnel0
R1(config-if)# vrf forwarding OVERLAY      ! 隧道内部流量在 OVERLAY VRF
R1(config-if)# tunnel vrf default          ! 外层封装在 global 表查路由
```
两张路由表物理隔离，**OVERLAY 学到什么都不会影响 global 表**，从架构上根除这个问题。

**检测方法**：
```cisco
R1# show logging | include RECURDOWN
R1# show ip route 203.2.2.2               ! 看它的下一跳是不是 Tunnel0
Routing entry for 203.2.2.0/24
  Known via "ospf 1", ...
  * 172.16.0.2, from 2.2.2.2, via Tunnel0     ← 找到了，就是它
```

**推荐做法**：**方案 ② 作为快速修复，方案 ③ 作为长期架构。** 方案 ① 在网络复杂后容易被人不小心破坏（有人加了个 network 语句就复发了）。
</details>

**3.** `ip mtu 1400` 和 `ip tcp adjust-mss 1360` 有什么区别？为什么两个都要配？

<details><summary>答案</summary>

| 命令 | 作用范围 | 机制 | 时机 |
|:--|:--|:--|:--|
| **`ip mtu 1400`** | **所有 IP 流量**（TCP/UDP/ICMP） | 限制从该接口出去的 IP 包最大长度，超过就分片（或 DF 置位时丢弃 + 回 ICMP） | **被动**，包已经产生了才处理 |
| **`ip tcp adjust-mss 1360`** | **只对 TCP** | **改写经过的 TCP SYN 包里的 MSS 选项字段**，让两端协商出更小的段大小 | **主动**，从源头避免产生大包 |

**为什么 `ip mtu` 不够**：

它是被动的补救措施。当一个 1500 字节的包到达隧道口：
- 如果 **DF 位没置**：路由器分片。分片会显著降低性能（接收端要重组，CPU 开销大），而且如果中间任何一片丢失，整个包都要重传。
- 如果 **DF 位置了**（现代 OS 默认都置，为了做 PMTUD）：路由器**丢弃**该包，并回一个 ICMP "Fragmentation Needed"（类型 3 代码 4）。

**问题就出在这个 ICMP 上**：如果路径上有任何设备（防火墙、云安全组、ACL）拦了 ICMP，源端**永远收不到这个通知**，会一直重发大包，一直失败——这就是 **PMTUD 黑洞**。

**症状**：ping 通、SSH 能连（小包），但网页打不开、大文件传不动（大包）。**非常隐蔽**。

**`ip tcp adjust-mss` 的高明之处**：

它在 TCP 三次握手时就介入了：
```
客户端 → SYN, MSS=1460
              ↓ 路由器改写
        SYN, MSS=1360        ← 到达服务器时已经被改小
服务器 → SYN+ACK, MSS=1460
              ↓ 路由器改写
        SYN+ACK, MSS=1360
              ↓
   两端协商结果：MSS = 1360，永远不会产生超过 1400 字节的 IP 包
```

**根本不需要 PMTUD，不依赖 ICMP，不产生分片。**

**为什么两个都要配**：
- `ip tcp adjust-mss` **只管 TCP**。UDP 流量（VoIP、视频流、DNS 大响应、某些 VPN 内嵌协议）不受影响，还是需要 `ip mtu` 来兜底。
- `ip mtu` 也决定了路由协议报文（OSPF LSU 等）的大小，配错会导致 OSPF 卡在 ExStart。

**数值计算**：
```
物理 MTU:              1500
减 GRE 开销:           -24    → 1476
减 IPsec 开销（约）:    -58    → 1418
保守取值 ip mtu:              1400
再减 IP头20 + TCP头20: -40    → 1360  = adjust-mss
```

**实战建议**：直接用 **`ip mtu 1400` + `ip tcp adjust-mss 1360`**。这组值对 GRE、IPsec、GRE over IPsec、甚至再套一层 VXLAN 都留了余量。宁可损失 6% 的效率，也不要留一个随机发作的 MTU 黑洞。

**同时**：**不要在 ACL 里无脑拦掉所有 ICMP**：
```cisco
permit icmp any any unreachable        ! PMTUD 需要
permit icmp any any time-exceeded      ! traceroute 需要
permit icmp any any echo-reply
```
</details>

**4.** 为什么企业 WAN 用 GRE over IPsec，而不是单纯用 IPsec？

<details><summary>答案</summary>

**因为纯 IPsec 不支持组播和广播，导致动态路由协议无法运行。**

| 技术 | 加密 | 支持组播/广播 | 支持非 IP 协议 | 支持动态路由 |
|:--|:--|:--|:--|:--|
| **纯 GRE** | ❌ | ✅ | ✅ | ✅ |
| **纯 IPsec** | ✅ | ❌ | ❌ | ❌ |
| **GRE over IPsec** | ✅ | ✅ | ✅ | ✅ |

**纯 IPsec 的具体问题**：

1. **OSPF 用组播 224.0.0.5**，EIGRP 用 224.0.0.10 —— IPsec 的感兴趣流 ACL 只能匹配单播 IP，组播包不会被加密传输，**邻居永远建不起来**。

2. **只能靠静态路由**。有 10 个分支就要写 10×N 条静态路由，加一个网段要去所有站点改配置，链路故障不会自动切换。运维噩梦。

3. **不支持非 IP 协议**（虽然现在很少见）。

**GRE over IPsec 的价值链**：
```
GRE 让远端路由器"看起来像直连"
     ↓
动态路由协议可以正常运行（组播被 GRE 封装成单播）
     ↓
IPsec 加密整个 GRE 包
     ↓
既有动态路由的灵活性，又有加密的安全性
```

**顺序为什么是 "GRE over IPsec" 而不是 "IPsec over GRE"**：

```
GRE over IPsec（先 GRE 后 IPsec）：
   [新IP | ESP | GRE | 原IP | 数据]
                 ↑ GRE 头在 ESP 里面，受保护 ✓
   
IPsec over GRE（先 IPsec 后 GRE）：
   [新IP | GRE | ESP | 原IP | 数据]
           ↑ GRE 头暴露在外，不受保护 ✗
   而且开销更大
```

**配置上的体现**：
```cisco
R1(config)# interface Tunnel0
R1(config-if)# tunnel protection ipsec profile IPSEC-PROF
```
一条 `tunnel protection` 就是 GRE over IPsec——先建 GRE 隧道，再用 IPsec 保护它。

**为什么用 `mode transport` 而不是 `mode tunnel`**：
GRE 已经加了一层新的公网 IP 头。IPsec 隧道模式会**再加一层 IP 头**，纯属浪费 20 字节。传输模式复用 GRE 的外层 IP 头，开销更小。

**现代演进**：
- **DMVPN** = GRE over IPsec + NHRP，让分支之间可以动态建立直连隧道（不用全 mesh 手工配置）。见 [ENARSI 第 6 章](../03-ENARSI-300-410/06-DMVPN.md)
- **SD-WAN** = 上述能力 + 集中化控制器 + 应用级智能选路。见 [第 15 章](15-SD-Access与SD-WAN.md)

两者的底层都是 GRE over IPsec 的思想。
</details>

**5.** IPsec 隧道的 `show crypto ipsec sa` 显示 `#pkts encaps` 在增长，但 `#pkts decaps` 始终是 0。说明什么？

<details><summary>答案</summary>

**说明本端在正常发送加密流量，但完全收不到对端的回程加密流量。**

问题**不在本端的 IPsec 配置**（因为加密在正常工作），而在以下几个方向：

**① 对端的 IPsec 配置有问题**
```cisco
! 登到对端检查
R2# show crypto ipsec sa | include pkts
    #pkts encaps: 0, #pkts encrypt: 0        ← 对端也没在发
    #pkts decaps: 1245, #pkts decrypt: 1245  ← 但对端收到了我们的包
```
如果对端 `decaps` 在涨但 `encaps` 是 0，说明**对端收到了但没有回程流量**——查对端的路由或感兴趣流。

**② 对端没有回程路由**
对端收到包、解密了、交给上层，但**不知道怎么把响应送回来**。
```cisco
R2# show ip route 10.1.0.0
% Network not in table                       ← 找到问题
```

**③ 中间设备拦了回程的 ESP/UDP 4500**
防火墙或 ACL 是单向配置的，只放行了出方向。
```cisco
R1# show access-lists FROM-INTERNET
    10 permit udp any any eq 4500 (0 matches)     ← 一次都没命中，可疑
```

**④ 对端的 NAT 把回程流量改写了**
对端配了 NAT 但没有 deny 掉 VPN 流量，导致回程包的源地址被改成公网地址，不匹配隧道。

**⑤ 非对称路由**
回程流量走了另一条路径（比如另一条互联网出口），没有经过 IPsec 隧道。

**排查顺序**：
```cisco
! 1. 先确认 Phase 1 是双向 UP 的
R1# show crypto ikev2 sa
 Tunnel-id  Local          Remote         Status
 1          202.1.1.1/500  203.2.2.2/500  READY      ← 是 READY 说明控制面正常

! 2. 到对端看计数器（这是最关键的一步）
R2# show crypto ipsec sa | include pkts

! 3. 检查对端路由
R2# show ip route <本端内网网段>

! 4. 检查对端 NAT
R2# show access-lists NAT-LIST
! 确认有 deny ip <本端网段> <对端网段>

! 5. 检查中间 ACL
R1# show access-lists | include 4500|esp
```

**记住这个对照表**：

| encaps | decaps | 诊断 |
|:--|:--|:--|
| 涨 | 涨 | ✅ 正常 |
| 涨 | **0** | ❌ **对端问题**：对端配置/路由/NAT/ACL |
| **0** | 涨 | ❌ **本端问题**：本端路由没把流量送进隧道，或 NAT 改了源地址 |
| 0 | 0 | ❌ 隧道完全没流量：路由、感兴趣流、或根本没有业务在跑 |

> **这个计数器对照表是 IPsec 排障最高效的工具**，30 秒就能判断问题在哪一端，避免两边瞎查。
</details>

---

**上一章** ← [01 企业网络架构与设计](01-企业网络架构与设计.md) ｜ **下一章** → [03 Overlay：VXLAN 与 LISP](03-Overlay-VXLAN与LISP.md)
