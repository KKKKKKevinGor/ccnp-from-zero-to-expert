# 06 · DMVPN

## ① 这章解决什么问题

公司有 **1 个总部 + 50 个分支**，都要通过互联网建立加密隧道互联。

**方案 A：全互联的点对点 GRE over IPsec**
```
   需要的隧道数 = 51 × 50 / 2 = ★ 1275 条 ★
   
   每条隧道要在两端各配一遍
   加一个新分支 → 要在 51 台设备上加配置
   ★ 完全不可维护 ★
```

**方案 B：星型（Hub-and-Spoke）**
```
   只需 50 条隧道（每个分支到总部一条）
   
   但：分支 A 访问分支 B 的流量必须 ★ 绕道总部 ★
   → 延迟翻倍
   → 总部带宽成为瓶颈
   → VoIP 质量差
```

**DMVPN 是方案 C**：
```
   ★ 平时是星型（配置简单）★
   ★ 分支之间有流量时，★ 自动建立临时的直连隧道 ★ ★
   ★ 流量停了隧道自动拆除 ★
   
   → 配置量 = 星型
   → 转发路径 = 全互联
```

**关键技术：mGRE + NHRP + IPsec。**

---

## ② DMVPN 的三大组件

| 组件 | 全称 | 作用 |
|:--|:--|:--|
| **mGRE** | Multipoint GRE | **一个隧道接口对应多个目的地**（不用为每个对端建一个 Tunnel 接口） |
| **NHRP** | Next Hop Resolution Protocol | **动态解析"隧道内 IP ↔ 公网 IP"的映射**（像隧道世界的 ARP） |
| **IPsec** | — | 加密（可选但生产必配） |

### 2.1 mGRE：一个接口，多个目的地

**普通 GRE**：
```cisco
interface Tunnel0
 tunnel source GigabitEthernet0/1
 tunnel destination 203.2.2.2         ← ★ 固定的单个目的地
```
50 个分支 → 需要 50 个 Tunnel 接口。

**mGRE**：
```cisco
interface Tunnel0
 tunnel source GigabitEthernet0/1
 tunnel mode gre multipoint           ← ★ 没有 destination！
```
**一个接口服务所有对端。目的地在转发时由 NHRP 动态决定。**

### 2.2 NHRP：隧道世界的 ARP

**问题**：mGRE 接口没有固定的 destination，那**转发时怎么知道把包封装给谁**？

```
   Hub 要发包给 Spoke1 的隧道地址 10.0.0.11
        ↓
   ★ Spoke1 的公网 IP 是多少？★
        ↓
   → 查 NHRP 映射表
```

**NHRP 的角色**：

| 角色 | 说明 |
|:--|:--|
| **NHS** (Next Hop Server) | **Hub**，维护映射数据库 |
| **NHC** (Next Hop Client) | **Spoke**，向 NHS 注册自己 |

**注册流程**：
```
   ① Spoke 启动，拿到公网 IP（可能是动态的）
        ↓
   ② Spoke 向 NHS（Hub）发 ★ NHRP Registration ★
      "我的隧道 IP 是 10.0.0.11，公网 IP 是 1.1.1.11"
        ↓
   ③ Hub 记录到 NHRP 映射表
        ↓
   ④ 周期性刷新（默认每 1/3 holdtime）
```

**★ 这就是 DMVPN 支持"分支用动态公网 IP"的原因** —— Hub 不需要预先知道 Spoke 的公网地址，Spoke 自己来注册。

**Spoke-to-Spoke 解析流程（Phase 2/3 的核心）**：
```
   ① Spoke1 要访问 Spoke2 后面的网段
        ↓
   ② 流量先走 Hub（因为路由表指向 Hub）
        ↓
   ③ Hub 发现"这个流量应该 Spoke 之间直接走"
      → 发 ★ NHRP Redirect ★ 给 Spoke1
        ↓
   ④ Spoke1 发 ★ NHRP Resolution Request ★
      "10.0.0.12（Spoke2 的隧道 IP）的公网地址是多少？"
        ↓
   ⑤ Hub 转发这个请求给 Spoke2（或直接回答）
        ↓
   ⑥ Spoke2 回 ★ NHRP Resolution Reply ★："我的公网 IP 是 2.2.2.22"
        ↓
   ⑦ Spoke1 记录映射，★ 直接与 Spoke2 建立动态隧道 ★
        ↓
   ⑧ 后续流量走直连隧道，不再经过 Hub ✓
        ↓
   ⑨ 一段时间没流量 → 隧道自动拆除
```

---

## ③ 三个 Phase（★ 核心考点）

| | **Phase 1** | **Phase 2** | **Phase 3** |
|:--|:--|:--|:--|
| **Hub 的隧道** | mGRE | mGRE | mGRE |
| **Spoke 的隧道** | **P2P GRE** | **mGRE** | **mGRE** |
| **Spoke-to-Spoke 直连** | ❌ **不支持** | ✅ 支持 | ✅ 支持 |
| **Hub 能否汇总路由** | ✅ 可以 | ❌ **不可以** | ✅ **可以** |
| **Spoke 路由表大小** | 小（只需默认路由） | **大**（需要所有 Spoke 的明细） | **小** |
| **关键机制** | — | Spoke 保留原始下一跳 | **NHRP Redirect + Shortcut** |
| **扩展性** | 中 | **差** | ★ **最好** |
| **推荐** | 简单场景 | 已过时 | ★ **首选** |

### 3.1 Phase 1：纯星型

```cisco
! ── Hub ──
interface Tunnel0
 ip address 10.0.0.1 255.255.255.0
 tunnel source GigabitEthernet0/1
 ★ tunnel mode gre multipoint ★
 ip nhrp network-id 100
 ip nhrp map multicast dynamic
 ip nhrp authentication DmvpnKey

! ── Spoke ──
interface Tunnel0
 ip address 10.0.0.11 255.255.255.0
 tunnel source GigabitEthernet0/1
 ★ tunnel destination <Hub公网IP> ★         ← P2P GRE，固定指向 Hub
 ip nhrp network-id 100
 ip nhrp nhs 10.0.0.1
 ip nhrp map 10.0.0.1 <Hub公网IP>
```

**特点**：
- Spoke 只能通过 Hub 通信（没有 Spoke-to-Spoke）
- **Hub 可以向 Spoke 只通告一条默认路由**（因为反正都要走 Hub）
- Spoke 的路由表极小

**适用**：分支之间没有直接通信需求的场景。

### 3.2 Phase 2：支持 Spoke-to-Spoke，但有个大问题

```cisco
! ── Spoke 也改成 mGRE ──
interface Tunnel0
 ip address 10.0.0.11 255.255.255.0
 tunnel source GigabitEthernet0/1
 ★ tunnel mode gre multipoint ★             ← 改成 mGRE
 ip nhrp network-id 100
 ip nhrp nhs 10.0.0.1
 ip nhrp map 10.0.0.1 <Hub公网IP>
 ip nhrp map multicast <Hub公网IP>
```

**★ Phase 2 的致命问题：Hub 不能汇总路由。**

**为什么**：
```
   Spoke-to-Spoke 直连的前提是：
   ★ Spoke1 的路由表里，去 Spoke2 网段的【下一跳】必须是 Spoke2 的隧道 IP ★
        ↓
   如果 Hub 做了汇总，Spoke1 收到的是：
   "10.0.0.0/8 → 下一跳 Hub"
        ↓
   ★ 下一跳是 Hub，不是 Spoke2 ★
        ↓
   ★ 流量永远走 Hub，Spoke-to-Spoke 隧道建不起来 ★
```

**后果**：
```
   50 个分支，每个分支要在路由表里保存
   ★ 其他 49 个分支的所有明细路由 ★
        ↓
   Spoke 的路由表巨大
   路由协议开销大
   扩展性差
```

**Hub 上的配套配置（Phase 2 必需）**：
```cisco
! ── EIGRP 场景 ──
Hub(config)# interface Tunnel0
Hub(config-if)# ★ no ip split-horizon eigrp 100 ★     ! 关闭水平分割
!                （否则从 Spoke 学到的路由不会再发给其他 Spoke）
Hub(config-if)# ★ no ip next-hop-self eigrp 100 ★     ! 保留原始下一跳
!                （否则下一跳会被改成 Hub）

! ── OSPF 场景 ──
Hub(config-if)# ip ospf network broadcast
Hub(config-if)# ip ospf priority 255                  ! Hub 必须是 DR
Spoke(config-if)# ip ospf network broadcast
Spoke(config-if)# ip ospf priority 0                  ! Spoke 永不当 DR
```

> **`no ip next-hop-self eigrp` 是 Phase 2 的关键**。不配的话，Hub 转发路由时会把下一跳改成自己，Spoke-to-Spoke 就失效了。

### 3.3 Phase 3：★ 现代首选

**Phase 3 用 NHRP Redirect + Shortcut 解决了 Phase 2 的问题。**

```cisco
! ── Hub ──
interface Tunnel0
 ip address 10.0.0.1 255.255.255.0
 tunnel source GigabitEthernet0/1
 tunnel mode gre multipoint
 ip nhrp network-id 100
 ip nhrp map multicast dynamic
 ★ ip nhrp redirect ★                      ← Phase 3 的核心（Hub 上）
 ip nhrp authentication DmvpnKey

! ── Spoke ──
interface Tunnel0
 ip address 10.0.0.11 255.255.255.0
 tunnel source GigabitEthernet0/1
 tunnel mode gre multipoint
 ip nhrp network-id 100
 ip nhrp nhs 10.0.0.1
 ip nhrp map 10.0.0.1 <Hub公网IP>
 ip nhrp map multicast <Hub公网IP>
 ★ ip nhrp shortcut ★                      ← Phase 3 的核心（Spoke 上）
```

**工作原理**：
```
   ① Hub 只通告一条汇总路由（甚至默认路由）给 Spoke
      → ★ Spoke 的路由表很小 ★
        ↓
   ② Spoke1 访问 Spoke2 的网段
      → 路由表说走 Hub → 流量先走 Hub
        ↓
   ③ Hub 转发时发现"进接口和出接口是同一个 Tunnel"
      → ★ 发 NHRP Redirect 给 Spoke1 ★
        "这个流量你可以直接找 Spoke2"
        ↓
   ④ Spoke1 收到 Redirect，发 NHRP Resolution Request
        ↓
   ⑤ 得到 Spoke2 的公网 IP，建立直连隧道
        ↓
   ⑥ ★ 在路由表里安装一条【NHRP 快捷路由】★
      指向 Spoke2 的隧道 IP
        ↓
   ⑦ 后续流量直接走 Spoke-to-Spoke 隧道 ✓
```

**★ Phase 3 同时获得了两个好处**：
- **路由表小**（Hub 可以汇总）
- **Spoke-to-Spoke 直连**（NHRP 动态解析）

**验证快捷路由**：
```cisco
Spoke1# show ip route | include %
%    10.2.2.0/24 [250/255] via 10.0.0.12, 00:01:23, Tunnel0
↑                              ↑
NHRP 安装的快捷路由        下一跳是 Spoke2 的隧道 IP
```

**`%` 标记表示这是 NHRP 动态安装的快捷路由。**

---

## ④ 完整配置（Phase 3 + IPsec）

```cisco
! ═══════════════ HUB ═══════════════

! ── ① IKEv2 ──
Hub(config)# crypto ikev2 proposal PROP-1
Hub(config-ikev2-proposal)#  encryption aes-cbc-256
Hub(config-ikev2-proposal)#  integrity sha256
Hub(config-ikev2-proposal)#  group 14

Hub(config)# crypto ikev2 policy POL-1
Hub(config-ikev2-policy)#  proposal PROP-1

Hub(config)# crypto ikev2 keyring KR-1
Hub(config-ikev2-keyring)#  peer ANY
Hub(config-ikev2-keyring-peer)#   address 0.0.0.0 0.0.0.0      ! ★ 接受任意对端
Hub(config-ikev2-keyring-peer)#   pre-shared-key DmvpnSharedKey

Hub(config)# crypto ikev2 profile PROF-1
Hub(config-ikev2-profile)#  match identity remote address 0.0.0.0
Hub(config-ikev2-profile)#  authentication local pre-share
Hub(config-ikev2-profile)#  authentication remote pre-share
Hub(config-ikev2-profile)#  keyring local KR-1

! ── ② IPsec ──
Hub(config)# crypto ipsec transform-set TS-1 esp-aes 256 esp-sha256-hmac
Hub(cfg-crypto-trans)#  mode transport                          ! ★ GRE over IPsec 用传输模式

Hub(config)# crypto ipsec profile IPSEC-PROF
Hub(ipsec-profile)#  set transform-set TS-1
Hub(ipsec-profile)#  set ikev2-profile PROF-1

! ── ③ mGRE 隧道 ──
Hub(config)# interface Tunnel0
Hub(config-if)# ip address 10.0.0.1 255.255.255.0
Hub(config-if)# ip mtu 1400                                     ! ★ 必配
Hub(config-if)# ip tcp adjust-mss 1360                          ! ★ 必配
Hub(config-if)# tunnel source GigabitEthernet0/1
Hub(config-if)# tunnel mode gre multipoint
Hub(config-if)# tunnel key 100
Hub(config-if)# tunnel protection ipsec profile IPSEC-PROF
! ── NHRP ──
Hub(config-if)# ip nhrp network-id 100
Hub(config-if)# ip nhrp authentication DmvpnNhrpKey
Hub(config-if)# ip nhrp map multicast dynamic                   ! ★ 动态学习组播成员
Hub(config-if)# ip nhrp holdtime 600
Hub(config-if)# ip nhrp redirect                                ! ★ Phase 3
! ── 路由协议 ──
Hub(config-if)# ip ospf network point-to-multipoint             ! 或用 EIGRP

! ── ④ 路由（EIGRP 示例）──
Hub(config)# router eigrp 100
Hub(config-router)# network 10.0.0.0 0.0.0.255
Hub(config-router)# network 192.168.0.0 0.0.255.255
Hub(config-router)# no auto-summary

Hub(config)# interface Tunnel0
Hub(config-if)# no ip split-horizon eigrp 100                   ! ★ 必须关
! Phase 3 不需要 no ip next-hop-self（因为靠 NHRP shortcut 而非下一跳保留）
Hub(config-if)# ip summary-address eigrp 100 192.168.0.0 255.255.0.0   ! ★ Phase 3 可以汇总


! ═══════════════ SPOKE ═══════════════

! ── IKEv2 / IPsec 配置与 Hub 相同（略）──

Spoke1(config)# interface Tunnel0
Spoke1(config-if)# ip address 10.0.0.11 255.255.255.0
Spoke1(config-if)# ip mtu 1400
Spoke1(config-if)# ip tcp adjust-mss 1360
Spoke1(config-if)# tunnel source GigabitEthernet0/1
Spoke1(config-if)# tunnel mode gre multipoint
Spoke1(config-if)# tunnel key 100
Spoke1(config-if)# tunnel protection ipsec profile IPSEC-PROF
! ── NHRP ──
Spoke1(config-if)# ip nhrp network-id 100
Spoke1(config-if)# ip nhrp authentication DmvpnNhrpKey
Spoke1(config-if)# ip nhrp nhs 10.0.0.1                         ! Hub 的隧道 IP
Spoke1(config-if)# ip nhrp map 10.0.0.1 <Hub公网IP>             ! 隧道IP ↔ 公网IP
Spoke1(config-if)# ip nhrp map multicast <Hub公网IP>            ! ★ 组播发给 Hub
Spoke1(config-if)# ip nhrp holdtime 600
Spoke1(config-if)# ip nhrp shortcut                             ! ★ Phase 3
Spoke1(config-if)# ip ospf network point-to-multipoint

Spoke1(config)# router eigrp 100
Spoke1(config-router)# network 10.0.0.0 0.0.0.255
Spoke1(config-router)# network 192.168.11.0 0.0.0.255
Spoke1(config-router)# no auto-summary
```

**简化写法（IOS 15.2+）**：
```cisco
! 一条命令代替 nhs + map + map multicast
Spoke1(config-if)# ip nhrp nhs 10.0.0.1 nbma <Hub公网IP> multicast
```

---

## ⑤ 验证与排障

### 5.1 验证命令

```cisco
! ── DMVPN 总览（★ 最常用）──
Hub# show dmvpn
Hub# show dmvpn detail

! ── NHRP ──
Hub# show ip nhrp
Hub# show ip nhrp brief
Hub# show ip nhrp detail
Hub# show ip nhrp traffic
Hub# show ip nhrp nhs detail                      ! Spoke 上看 NHS 状态

! ── 隧道 ──
Hub# show interfaces Tunnel0
Hub# show ip interface brief | include Tunnel

! ── IPsec ──
Hub# show crypto ikev2 sa
Hub# show crypto ipsec sa
Hub# show crypto session
Hub# show crypto session detail

! ── 路由 ──
Hub# show ip route eigrp
Spoke1# show ip route | include %                 ! ★ NHRP 快捷路由

! ── 调试 ──
Hub# debug nhrp
Hub# debug nhrp packet
Hub# debug dmvpn all all
```

### 5.2 `show dmvpn` 输出解读

```cisco
Hub# show dmvpn
Legend: Attrb --> S - Static, D - Dynamic, I - Incomplete
        N - NATed, L - Local, X - No Socket
        T1 - Route Installed, T2 - Nexthop-override
        C - CTS Capable, I2 - Temporary
        # Ent --> Number of NHRP entries with same NBMA peer
        NHS Status: E --> Expecting Replies, R --> Responding, W --> Waiting
        UpDn Time --> Up or Down Time for a Tunnel

Interface: Tunnel0, IPv4 NHRP Details
Type: ★ Hub ★, NHRP Peers: 2,

 # Ent  Peer NBMA Addr  Peer Tunnel Add  State  UpDn Tm  Attrb
 ----- --------------- ---------------- ------ -------- -----
     1     1.1.1.11        10.0.0.11     ★ UP ★ 00:15:23    D
     1     2.2.2.22        10.0.0.12     ★ UP ★ 00:14:56    D
           ↑               ↑              ↑                 ↑
        公网 IP        隧道 IP          状态           D=动态学到
```

**State 的含义**：

| State | 含义 |
|:--|:--|
| **UP** | ✅ 隧道正常 |
| **NHRP** | 正在解析中 |
| **IKE** | IPsec 协商中 |
| **DOWN** | ❌ 不可用 |

**Attrb（属性）**：

| 标志 | 含义 |
|:--|:--|
| **S** | Static（手工配置的映射） |
| **D** | **Dynamic**（NHRP 动态学到的） |
| **I** | Incomplete（解析未完成） |
| **N** | **NATed**（对端在 NAT 后面） |
| **X** | **No Socket**（IPsec 未建立）★ 有问题 |
| **T1** | 路由已安装 |
| **T2** | Nexthop-override |

**Spoke 上看 Spoke-to-Spoke 隧道**：
```cisco
Spoke1# show dmvpn
Interface: Tunnel0, IPv4 NHRP Details
Type: ★ Spoke ★, NHRP Peers: 2,

 # Ent  Peer NBMA Addr  Peer Tunnel Add  State  UpDn Tm  Attrb
 ----- --------------- ---------------- ------ -------- -----
     1     3.3.3.3         10.0.0.1        UP   00:20:15     S    ← 到 Hub（静态）
     1     2.2.2.22        10.0.0.12       UP   00:00:45   ★ D ★  ← 到 Spoke2（动态）
                                                                   ★ Spoke-to-Spoke 建立了 ✓
```

### 5.3 常见故障

| 症状 | 根因 | 验证 |
|:--|:--|:--|
| Spoke 注册不上 Hub | NHS 地址/映射错 | `show ip nhrp nhs detail` |
| | **network-id 不一致** | `show run int Tunnel0` |
| | **NHRP 认证不匹配** | `debug nhrp` |
| | Hub 公网不可达 | `ping <Hub公网IP>` |
| 隧道 UP 但学不到路由 | **没关水平分割**（EIGRP） | `show run int Tu0 \| inc split` |
| | 组播不通 | `show ip nhrp multicast` |
| **Spoke-to-Spoke 建不起来** | **Phase 2 但 Hub 汇总了** | 去掉汇总，或改 Phase 3 |
| | **Phase 3 缺 redirect/shortcut** | `show run int Tu0 \| inc nhrp` |
| | NAT 穿越问题 | `show dmvpn detail` 看 N 标志 |
| **`X` 标志（No Socket）** | **IPsec 没建立** | `show crypto ikev2 sa` |
| 大包不通 | **MTU** | `ping size 1500 df-bit` |
| 隧道频繁 up/down | 递归路由 / 公网不稳 | `show logging \| inc RECURDOWN` |
| NAT 后的 Spoke 连不上 | 缺 NAT-T | 检查 UDP 4500 放行 |

### 5.4 EIGRP over DMVPN 的两个必配项

```cisco
Hub(config)# interface Tunnel0

! ── ① 关闭水平分割（★ 所有 Phase 都需要）──
Hub(config-if)# no ip split-horizon eigrp 100
```
**为什么**：EIGRP 的水平分割规则是"从某接口学到的路由不再从该接口发出"。但 DMVPN 里 **Hub 从 Tunnel0 学到 Spoke1 的路由，又要从 Tunnel0 发给 Spoke2**——被水平分割挡住了。

**症状**：Spoke 之间互相学不到路由（但都能学到 Hub 的）。

```cisco
! ── ② 保留原始下一跳（★ 只有 Phase 2 需要）──
Hub(config-if)# no ip next-hop-self eigrp 100
```
**为什么**：Phase 2 的 Spoke-to-Spoke 依赖"路由的下一跳是对端 Spoke 的隧道 IP"。默认 Hub 会把下一跳改成自己，导致流量永远走 Hub。

**Phase 3 不需要这条**（它靠 NHRP Shortcut 而不是下一跳保留）。

### 5.5 OSPF over DMVPN

```cisco
! ── 推荐：point-to-multipoint ──
Hub(config-if)# ip ospf network point-to-multipoint
Spoke(config-if)# ip ospf network point-to-multipoint
! 优点：不选 DR，适应 Hub-Spoke 拓扑，Spoke 上线下线不影响

! ── 或：broadcast（要控制 DR）──
Hub(config-if)# ip ospf network broadcast
Hub(config-if)# ip ospf priority 255              ! ★ Hub 必须是 DR
Spoke(config-if)# ip ospf network broadcast
Spoke(config-if)# ip ospf priority 0              ! ★ Spoke 永不当 DR
```

> **为什么 Spoke 必须 priority 0**：如果某个 Spoke 当了 DR，其他 Spoke 要和它建立 FULL 邻接——但 **Spoke 之间默认没有隧道**，邻接建不起来，OSPF 就崩了。

> **推荐用 `point-to-multipoint`**：不需要 DR，配置更简单，也更稳定。代价是**每个 Spoke 会产生一条 /32 的主机路由**（P2MP 网络类型的特性），路由表稍大。

---

## ⑥ 配套实验

**拓扑**：
```
                    [ Internet ]
              ┌──────────┼──────────┐
              │          │          │
        1.1.1.11    3.3.3.3     2.2.2.22
         Spoke1       Hub        Spoke2
        10.0.0.11   10.0.0.1    10.0.0.12
             │          │            │
      192.168.11.0/24  192.168.0.0/24  192.168.12.0/24
```

### Step 1：先建纯 GRE 的 Phase 1，验证连通

```cisco
! ── Hub ──
Hub(config)# interface Tunnel0
Hub(config-if)# ip address 10.0.0.1 255.255.255.0
Hub(config-if)# ip mtu 1400
Hub(config-if)# ip tcp adjust-mss 1360
Hub(config-if)# tunnel source GigabitEthernet0/1
Hub(config-if)# tunnel mode gre multipoint
Hub(config-if)# tunnel key 100
Hub(config-if)# ip nhrp network-id 100
Hub(config-if)# ip nhrp authentication DmvpnKey
Hub(config-if)# ip nhrp map multicast dynamic
Hub(config-if)# ip nhrp holdtime 600

! ── Spoke1 ──
Spoke1(config)# interface Tunnel0
Spoke1(config-if)# ip address 10.0.0.11 255.255.255.0
Spoke1(config-if)# ip mtu 1400
Spoke1(config-if)# ip tcp adjust-mss 1360
Spoke1(config-if)# tunnel source GigabitEthernet0/1
Spoke1(config-if)# tunnel mode gre multipoint
Spoke1(config-if)# tunnel key 100
Spoke1(config-if)# ip nhrp network-id 100
Spoke1(config-if)# ip nhrp authentication DmvpnKey
Spoke1(config-if)# ip nhrp nhs 10.0.0.1
Spoke1(config-if)# ip nhrp map 10.0.0.1 3.3.3.3
Spoke1(config-if)# ip nhrp map multicast 3.3.3.3
Spoke1(config-if)# ip nhrp holdtime 600
```

**验证注册**：
```cisco
Hub# show dmvpn
 # Ent  Peer NBMA Addr  Peer Tunnel Add  State  UpDn Tm  Attrb
     1     1.1.1.11        10.0.0.11        UP   00:00:45     D    ← 注册成功 ✓
     1     2.2.2.22        10.0.0.12        UP   00:00:38     D

Hub# show ip nhrp
10.0.0.11/32 via 10.0.0.11
   Tunnel0 created 00:00:45, expire 00:09:15
   Type: dynamic, Flags: unique registered used nhop
   NBMA address: 1.1.1.11
```

```cisco
Spoke1# show ip nhrp nhs detail
Legend: E=Expecting replies, R=Responding, W=Waiting
Tunnel0:
  10.0.0.1  ★ RE ★ priority = 0 cluster = 0  req-sent 12  req-failed 0  repl-recv 12
            ↑ R = Responding（Hub 在响应）✓
```

### Step 2：加路由协议

```cisco
! ── Hub ──
Hub(config)# router eigrp 100
Hub(config-router)# network 10.0.0.0 0.0.0.255
Hub(config-router)# network 192.168.0.0 0.0.0.255
Hub(config-router)# no auto-summary

Hub(config)# interface Tunnel0
Hub(config-if)# ★ no ip split-horizon eigrp 100 ★

! ── Spoke ──
Spoke1(config)# router eigrp 100
Spoke1(config-router)# network 10.0.0.0 0.0.0.255
Spoke1(config-router)# network 192.168.11.0 0.0.0.255
Spoke1(config-router)# no auto-summary
```

**验证**：
```cisco
Spoke1# show ip eigrp neighbors
H   Address      Interface  Hold Uptime   SRTT  RTO   Q  Seq
0   10.0.0.1     Tu0          12  00:02:15   45  270   0  8

Spoke1# show ip route eigrp
D    192.168.0.0/24 [90/26882560] via 10.0.0.1, 00:02:10, Tunnel0
D    192.168.12.0/24 [90/28288000] via ★ 10.0.0.1 ★, 00:02:05, Tunnel0
                                          ↑ 下一跳是 Hub → 流量走 Hub
```

**测试 Spoke-to-Spoke**：
```cisco
Spoke1# traceroute 192.168.12.10
  1 10.0.0.1     20 msec       ← ★ 经过 Hub
  2 10.0.0.12    35 msec
  3 192.168.12.10 40 msec
```

### Step 3：故意演示"关水平分割"的重要性

```cisco
Hub(config)# interface Tunnel0
Hub(config-if)# ip split-horizon eigrp 100          ! 打开水平分割
```

**观察**：
```cisco
Spoke1# show ip route eigrp
D    192.168.0.0/24 [90/26882560] via 10.0.0.1, ...
! ★ 192.168.12.0/24 不见了！★
! Hub 从 Tunnel0 学到 Spoke2 的路由，被水平分割挡住不发给 Spoke1
```

**修复**：
```cisco
Hub(config-if)# no ip split-horizon eigrp 100
```

### Step 4：升级到 Phase 3

```cisco
! ── Hub 加 redirect ──
Hub(config)# interface Tunnel0
Hub(config-if)# ★ ip nhrp redirect ★
Hub(config-if)# ip summary-address eigrp 100 192.168.0.0 255.255.0.0    ! ★ 现在可以汇总了

! ── Spoke 加 shortcut ──
Spoke1(config)# interface Tunnel0
Spoke1(config-if)# ★ ip nhrp shortcut ★
Spoke2(config)# interface Tunnel0
Spoke2(config-if)# ★ ip nhrp shortcut ★
```

**验证路由表变小**：
```cisco
Spoke1# show ip route eigrp
D    192.168.0.0/16 [90/26882560] via 10.0.0.1, 00:01:15, Tunnel0
! ★ 只有一条汇总路由，明细都没了 ✓ ★
```

### Step 5：验证 Spoke-to-Spoke 直连（★ 核心验证）

```cisco
! 从 Spoke1 后面的主机 ping Spoke2 后面的主机
Spoke1# ping 192.168.12.10 source 192.168.11.1 repeat 20
!!!!!!!!!!!!!!!!!!!!
```

**第一次 traceroute（还在走 Hub）**：
```cisco
Spoke1# traceroute 192.168.12.10 source 192.168.11.1
  1 10.0.0.1      20 msec       ← 先走 Hub
  2 10.0.0.12     35 msec
  3 192.168.12.10 40 msec
```

**几秒后再次 traceroute**：
```cisco
Spoke1# traceroute 192.168.12.10 source 192.168.11.1
  1 ★ 10.0.0.12 ★  15 msec      ← ★ 直连了！不再经过 Hub ✓
  2 192.168.12.10  18 msec
```

**看快捷路由**：
```cisco
Spoke1# show ip route | include %
% ★ 192.168.12.0/24 [250/255] via 10.0.0.12, 00:00:35, Tunnel0 ★
  ↑                              ↑
NHRP 安装的快捷路由         下一跳是 Spoke2
```

**看动态隧道**：
```cisco
Spoke1# show dmvpn
 # Ent  Peer NBMA Addr  Peer Tunnel Add  State  UpDn Tm  Attrb
     1     3.3.3.3         10.0.0.1         UP   00:20:15     S    ← 到 Hub
     1  ★ 2.2.2.22 ★    ★ 10.0.0.12 ★     UP   00:00:35   ★ D ★  ← ★ 到 Spoke2！
```

**✅ Spoke-to-Spoke 动态隧道建立成功。**

**观察隧道自动拆除**：停止流量，等几分钟（NHRP holdtime），再看：
```cisco
Spoke1# show dmvpn
 # Ent  Peer NBMA Addr  Peer Tunnel Add  State  UpDn Tm  Attrb
     1     3.3.3.3         10.0.0.1         UP   00:25:15     S
! ★ 到 Spoke2 的动态隧道消失了 ✓ ★
```

### Step 6：加 IPsec

按第 ④ 节的配置在所有设备上加 IKEv2 和 IPsec profile，然后：
```cisco
Hub(config)# interface Tunnel0
Hub(config-if)# tunnel protection ipsec profile IPSEC-PROF shared
!                                                           ↑
!                              ★ mGRE 多对端场景加 shared ★
```

**验证**：
```cisco
Hub# show crypto session
Interface: Tunnel0
Session status: UP-ACTIVE
Peer: 1.1.1.11 port 500
  Session ID: 1
  IKEv2 SA: local 3.3.3.3/500 remote 1.1.1.11/500 Active
  IPSEC FLOW: permit 47 host 3.3.3.3 host 1.1.1.11
        Active SAs: 2, origin: crypto map

Hub# show dmvpn detail
! 确认 Attrb 里没有 X（No Socket）标志
```

### Step 7：故障注入

**故障 A：NHRP network-id 不一致**
```cisco
Spoke1(config-if)# ip nhrp network-id 200          ! Hub 是 100
```
**观察**：
```cisco
Hub# show dmvpn
! Spoke1 消失了

Spoke1# debug nhrp
NHRP: Receive Registration Reply via Tunnel0
NHRP: netid_in = 200, to_us = 0                    ← network-id 不匹配
```

**故障 B：忘了 `ip nhrp map multicast`**
```cisco
Spoke1(config-if)# no ip nhrp map multicast 3.3.3.3
```
**观察**：隧道 UP，NHRP 注册成功，**但 EIGRP 邻居建不起来**（因为 EIGRP Hello 是组播 224.0.0.10，发不出去）。
```cisco
Spoke1# show ip eigrp neighbors
! 空

Spoke1# show ip nhrp multicast
! 空 ← 找到原因
```

**故障 C：Phase 2 下 Hub 做了汇总**
```cisco
! 先去掉 Phase 3 的配置
Hub(config-if)# no ip nhrp redirect
Spoke1(config-if)# no ip nhrp shortcut
Spoke2(config-if)# no ip nhrp shortcut

! Hub 做汇总
Hub(config-if)# ip summary-address eigrp 100 192.168.0.0 255.255.0.0
```
**观察**：
```cisco
Spoke1# show ip route eigrp
D    192.168.0.0/16 [90/26882560] via 10.0.0.1, Tunnel0
                                        ↑ 下一跳是 Hub

Spoke1# traceroute 192.168.12.10
  1 10.0.0.1                             ← ★ 永远走 Hub，Spoke-to-Spoke 建不起来
  2 10.0.0.12
  3 192.168.12.10
```
**这直观演示了"Phase 2 不能汇总"的原因。**

**故障 D：MTU**
```cisco
Spoke1(config-if)# no ip mtu 1400
Spoke1(config-if)# no ip tcp adjust-mss 1360
```
```cisco
Spoke1# ping 192.168.12.10 size 1500 df-bit
M.M.M                                          ← 大包不通
```

---

## ⑦ 自测题

**1.** DMVPN 的三大组件是什么？各自解决什么问题？

<details><summary>答案</summary>

| 组件 | 全称 | 解决的问题 |
|:--|:--|:--|
| **mGRE** | Multipoint GRE | **一个隧道接口服务多个对端**，不用为每个分支建一个 Tunnel 接口 |
| **NHRP** | Next Hop Resolution Protocol | **动态解析"隧道 IP ↔ 公网 IP"的映射**，支持分支用动态公网地址 |
| **IPsec** | — | **加密**（可选但生产必配） |

## mGRE 解决"配置量爆炸"

**普通 GRE**：
```cisco
interface Tunnel0
 tunnel destination 203.2.2.2      ← 固定单个目的地
```
50 个分支 → **50 个 Tunnel 接口**，每加一个分支要在 51 台设备上改配置。

**mGRE**：
```cisco
interface Tunnel0
 tunnel mode gre multipoint        ← ★ 没有 destination
```
**一个接口服务所有对端。** 目的地在转发时动态决定。

## NHRP 解决"我不知道对端的公网 IP"

mGRE 没有固定 destination，那转发时怎么知道封装给谁？

```
   要发包给隧道 IP 10.0.0.11
        ↓
   ★ 它的公网 IP 是多少？★
        ↓
   查 NHRP 映射表
```

**NHRP 就是"隧道世界的 ARP"**：
- ARP 解析 `IP → MAC`
- NHRP 解析 `隧道IP → 公网IP（NBMA地址）`

**角色**：
| 角色 | 是谁 | 职责 |
|:--|:--|:--|
| **NHS** (Next Hop Server) | **Hub** | 维护映射数据库，响应查询 |
| **NHC** (Next Hop Client) | **Spoke** | 向 NHS 注册自己 |

**★ 这就是 DMVPN 支持"分支用动态公网 IP"（比如家庭宽带、4G）的原因**：
```
   Spoke 拿到新的公网 IP
        ↓
   主动向 Hub 发 NHRP Registration
        ↓
   Hub 更新映射表
        ↓
   ★ Hub 不需要预先知道 Spoke 的地址 ★
```

传统的点对点 GRE 做不到这一点（`tunnel destination` 必须是固定的）。

## IPsec 解决"公网传输不安全"

DMVPN 跑在互联网上，必须加密。

```cisco
interface Tunnel0
 tunnel protection ipsec profile IPSEC-PROF ★ shared ★
!                                             ↑
!                          mGRE 多对端场景必须加 shared
```

**为什么用 `mode transport`**：GRE 已经加了外层 IP 头，IPsec 隧道模式会再加一层，浪费 20 字节。传输模式复用 GRE 的外层头。

## 三者的协作

```
   ① mGRE 提供"一对多"的隧道接口
        ↓
   ② NHRP 告诉 mGRE "这个隧道 IP 对应哪个公网 IP"
        ↓
   ③ IPsec 加密封装后的 GRE 包
        ↓
   ★ 结果：配置量像星型，转发路径像全互联，且加密 ★
```

**DMVPN 的完整价值**：
| 维度 | 全互联 GRE | 星型 | **DMVPN** |
|:--|:--|:--|:--|
| 配置量（50 分支） | 1275 条隧道 | 50 条 | ★ **每台一个接口** |
| 加新分支 | 改 51 台设备 | 改 2 台 | ★ **只改 1 台（新分支）** |
| Spoke 间路径 | 直连 | 绕 Hub | ★ **动态直连** |
| 支持动态 IP | ❌ | ❌ | ✅ |
</details>

**2.** DMVPN Phase 1/2/3 的核心区别是什么？

<details><summary>答案</summary>

| | **Phase 1** | **Phase 2** | **Phase 3** |
|:--|:--|:--|:--|
| **Spoke 的隧道类型** | **P2P GRE** | **mGRE** | **mGRE** |
| **Spoke-to-Spoke** | ❌ 不支持 | ✅ 支持 | ✅ 支持 |
| **Hub 能汇总路由** | ✅ | ❌ **不能** | ✅ **能** |
| **Spoke 路由表** | 小 | **大** | **小** |
| **关键机制** | — | 保留原始下一跳 | **NHRP Redirect + Shortcut** |
| **扩展性** | 中 | 差 | ★ **最好** |

## Phase 1：纯星型

```cisco
! Spoke 用 P2P GRE
interface Tunnel0
 tunnel destination <Hub公网IP>     ← 固定指向 Hub
```

- Spoke 只能通过 Hub 通信
- **Hub 可以只通告一条默认路由**给 Spoke
- Spoke 路由表极小

**适用**：分支之间没有直接通信需求（比如所有业务都在总部）。

## Phase 2：支持直连，但不能汇总（★ 关键限制）

```cisco
! Spoke 也改成 mGRE
interface Tunnel0
 tunnel mode gre multipoint
```

**Hub 的必需配置**：
```cisco
Hub(config-if)# no ip split-horizon eigrp 100      ! 关水平分割
Hub(config-if)# ★ no ip next-hop-self eigrp 100 ★ ! 保留原始下一跳
```

**★ 为什么不能汇总**：
```
   Spoke-to-Spoke 直连的前提：
   ★ Spoke1 路由表里，去 Spoke2 网段的【下一跳】必须是 Spoke2 的隧道 IP ★
        ↓
   如果 Hub 汇总了，Spoke1 收到的是：
   "192.168.0.0/16 → 下一跳 10.0.0.1（Hub）"
        ↓
   ★ 下一跳是 Hub，不是 Spoke2 ★
        ↓
   ★ 流量永远走 Hub，直连隧道建不起来 ★
```

**后果**：50 个分支，每个分支的路由表要装其他 49 个分支的**所有明细路由**。

```
   50 分支 × 每分支 5 个网段 = 250 条路由
   × 每个 Spoke 都要装
   ★ 路由协议开销大，收敛慢，扩展性差 ★
```

## Phase 3：两全其美 ★ 现代首选

```cisco
Hub(config-if)#   ★ ip nhrp redirect ★      ! Hub 上
Spoke(config-if)# ★ ip nhrp shortcut ★      ! Spoke 上
```

**工作流程**：
```
   ① Hub 只通告汇总路由（甚至默认路由）
      → ★ Spoke 路由表很小 ★
        ↓
   ② Spoke1 访问 Spoke2，流量先走 Hub
        ↓
   ③ Hub 发现"进接口和出接口都是 Tunnel0"
      → ★ 发 NHRP Redirect 给 Spoke1 ★
        ↓
   ④ Spoke1 发 NHRP Resolution Request 查 Spoke2 的公网 IP
        ↓
   ⑤ 建立直连隧道
        ↓
   ⑥ ★ 在路由表安装一条 NHRP 快捷路由（% 标记）★
        ↓
   ⑦ 后续流量直接走 Spoke-to-Spoke ✓
```

**验证**：
```cisco
Spoke1# show ip route eigrp
D    192.168.0.0/16 [90/26882560] via 10.0.0.1, Tunnel0
! ★ 只有一条汇总路由 ★

Spoke1# show ip route | include %
% ★ 192.168.12.0/24 [250/255] via 10.0.0.12, 00:00:35, Tunnel0 ★
  ↑ NHRP 动态安装的快捷路由，下一跳是 Spoke2
```

**Phase 3 同时获得**：
- ✅ **路由表小**（Hub 可以汇总）
- ✅ **Spoke-to-Spoke 直连**

## 记忆方法

```
   Phase 1：Spoke 是 P2P → 只能走 Hub
   Phase 2：Spoke 是 mGRE + 保留下一跳 → 能直连但不能汇总
   Phase 3：Spoke 是 mGRE + NHRP Redirect/Shortcut → 能直连也能汇总 ★
```

**实践建议**：**新部署一律用 Phase 3。** Phase 2 已经过时（它的限制在规模大时是致命的），Phase 1 只在"确实不需要 Spoke 间通信"的简单场景用。
</details>

**3.** 为什么 DMVPN 上跑 EIGRP 要关闭水平分割？

<details><summary>答案</summary>

**因为 Hub 需要把"从 Tunnel0 学到的路由"再从"同一个 Tunnel0"发出去。**

**水平分割（Split Horizon）的规则**：
> 从某个接口学到的路由，不再从该接口通告出去。

**这条规则在普通网络里是防环的**（避免"你告诉我，我又告诉你"）。

**但在 DMVPN 里会出问题**：

```
   Spoke1 ──┐
            ├── Hub 的 Tunnel0（一个接口，多个对端）
   Spoke2 ──┘
   
   ① Hub 从 Tunnel0 学到 Spoke1 的路由（192.168.11.0/24）
        ↓
   ② Hub 要把这条路由告诉 Spoke2
        ↓
   ③ 但 Spoke2 也在 Tunnel0 上
        ↓
   ★ 水平分割：这条路由是从 Tunnel0 学来的，不能再从 Tunnel0 发出 ★
        ↓
   ★ Spoke2 永远学不到 Spoke1 的路由 ★
```

**症状**：
```cisco
Spoke1# show ip route eigrp
D    192.168.0.0/24 [90/26882560] via 10.0.0.1, Tunnel0    ← 只学到 Hub 的
! ★ 其他 Spoke 的网段一条都没有 ★
```

**修复**：
```cisco
Hub(config)# interface Tunnel0
Hub(config-if)# ★ no ip split-horizon eigrp 100 ★
```

**注意语法差异**：
```cisco
! EIGRP（要指定 AS 号）
no ip split-horizon eigrp 100

! RIP（不需要 AS 号）
no ip split-horizon
```

**为什么关掉不会造成环路**：
- DMVPN 里 Hub 是中心，拓扑是星型的，天然无环
- EIGRP 的 DUAL 算法有自己的防环机制（可行性条件 FC）
- 关闭的只是"接口级"的水平分割，不影响协议本身的防环

**另一个 Phase 2 专有的配置**：
```cisco
Hub(config-if)# ★ no ip next-hop-self eigrp 100 ★
```

**作用**：默认情况下，Hub 转发路由时会把**下一跳改成自己**。这条命令让它**保留原始的下一跳**（Spoke 的隧道 IP）。

**为什么 Phase 2 需要**：
```
   Phase 2 的 Spoke-to-Spoke 依赖：
   ★ 路由表里的下一跳必须是对端 Spoke 的隧道 IP ★
        ↓
   如果下一跳被改成 Hub → 流量永远走 Hub → 直连失效
```

**Phase 3 不需要这条**，因为它靠 **NHRP Shortcut**（在路由表里额外安装快捷路由）而不是"下一跳保留"。

## OSPF over DMVPN 的对应问题

OSPF 没有水平分割的概念，但有**网络类型**的问题：

```cisco
! ── 推荐：point-to-multipoint ──
Hub(config-if)# ip ospf network point-to-multipoint
Spoke(config-if)# ip ospf network point-to-multipoint
! ✅ 不选 DR，适应星型拓扑
! ⚠️ 每个 Spoke 会产生 /32 主机路由，路由表稍大

! ── 或：broadcast（要严格控制 DR）──
Hub(config-if)# ip ospf network broadcast
Hub(config-if)# ★ ip ospf priority 255 ★         ! Hub 必须当 DR
Spoke(config-if)# ip ospf network broadcast
Spoke(config-if)# ★ ip ospf priority 0 ★         ! ★ Spoke 永不当 DR
```

**为什么 Spoke 必须 priority 0**：
```
   如果某个 Spoke 当了 DR
        ↓
   其他所有 Spoke 都要和它建立 FULL 邻接
        ↓
   ★ 但 Spoke 之间默认没有隧道！★
        ↓
   ★ 邻接建不起来，OSPF 崩溃 ★
```

**配置检查清单（EIGRP over DMVPN）**：
```
□ Hub: no ip split-horizon eigrp <AS>        （所有 Phase 都要）
□ Hub: no ip next-hop-self eigrp <AS>        （仅 Phase 2）
□ Hub: ip nhrp map multicast dynamic          （让组播能发给动态注册的 Spoke）
□ Spoke: ip nhrp map multicast <Hub公网IP>    （让组播能发给 Hub）
□ 所有: ip mtu 1400 + ip tcp adjust-mss 1360
```
</details>

**4.** `show dmvpn` 输出里的 `X` 标志是什么意思？

<details><summary>答案</summary>

**`X` = No Socket = IPsec 会话没有建立。**

```cisco
Hub# show dmvpn
 # Ent  Peer NBMA Addr  Peer Tunnel Add  State  UpDn Tm  Attrb
     1     1.1.1.11        10.0.0.11        UP   00:15:23   ★ DX ★
                                                              ↑
                                              D=动态, X=No Socket
```

**含义**：NHRP 层面注册成功了（Hub 知道 Spoke 的公网 IP），**但 IPsec 加密会话没建起来**。

**后果**：
- 隧道显示 UP（GRE 层面通）
- **但流量无法加密传输 → 实际不通**（因为配了 `tunnel protection` 后，未加密的流量会被丢弃）

**排查**：
```cisco
! ① 检查 IKE 阶段一
Hub# show crypto ikev2 sa
! 空 → Phase 1 没建起来

! ② 检查 IPsec 阶段二
Hub# show crypto ipsec sa
! 空或 pkts 为 0

! ③ 详细会话状态
Hub# show crypto session detail
Interface: Tunnel0
Session status: ★ DOWN ★
Peer: 1.1.1.11 port 500
  IKEv2 SA: local 3.3.3.3/500 remote 1.1.1.11/500 ★ Inactive ★

! ④ 调试
Hub# debug crypto ikev2
Hub# debug crypto ipsec
```

**常见原因**：

| 原因 | 检查 |
|:--|:--|
| **IKE 参数不匹配（HAGLE）** | 两端 `crypto ikev2 proposal` 对比 |
| **预共享密钥不同** | `crypto ikev2 keyring` |
| **UDP 500/4500 或 ESP 被拦** | `show access-lists` |
| **NAT 后面但没有 NAT-T** | `show dmvpn detail` 看 `N` 标志 |
| **transform-set 不匹配** | 两端对比 |
| **忘了 `shared` 关键字**（mGRE 多对端） | `show run int Tu0 \| inc protection` |

**★ `shared` 关键字的重要性**：
```cisco
! ❌ mGRE 多对端场景不加 shared 可能出问题
Hub(config-if)# tunnel protection ipsec profile IPSEC-PROF

! ✅ 正确
Hub(config-if)# tunnel protection ipsec profile IPSEC-PROF ★ shared ★
```

**为什么需要 `shared`**：一个 mGRE 接口要和多个对端建立 IPsec SA。`shared` 告诉 IOS "这个 IPsec profile 会被多个 SA 共享"。**不加的话，在某些 IOS 版本上会出现 SA 冲突。**

**ACL 必须放行的三样**：
```cisco
permit udp any any eq 500        ! IKE
permit udp any any eq 4500       ! NAT-T
permit esp any any               ! ESP（协议号 50）
```
**漏放 ESP 是最常见的**（因为它没有端口，容易被忘）。

## `show dmvpn` 的完整标志表

| 标志 | 含义 | 正常？ |
|:--|:--|:--|
| **S** | Static（手工配置的映射） | ✅ |
| **D** | Dynamic（NHRP 动态学到） | ✅ |
| **I** | Incomplete（解析未完成） | ⚠️ 短暂正常 |
| **N** | **NATed**（对端在 NAT 后） | ✅ 正常（需 NAT-T） |
| **L** | Local | ✅ |
| **X** | **No Socket（IPsec 未建立）** | ❌ **有问题** |
| **T1** | Route Installed | ✅ |
| **T2** | Nexthop-override | ✅ |
| **I2** | Temporary | ⚠️ |

**State 字段**：

| State | 含义 |
|:--|:--|
| **UP** | ✅ 正常 |
| **NHRP** | 正在 NHRP 解析 |
| **IKE** | IPsec 协商中 |
| **DOWN** | ❌ 不可用 |

**健康的输出应该是**：
```cisco
Hub# show dmvpn
 # Ent  Peer NBMA Addr  Peer Tunnel Add  State  UpDn Tm  Attrb
     1     1.1.1.11        10.0.0.11      ★ UP ★ 00:15:23   ★ D ★
     1     2.2.2.22        10.0.0.12        UP   00:14:56     D
! State 是 UP，Attrb 只有 D 或 S，★ 没有 X ★
```
</details>

**5.** DMVPN 的 Spoke 在 NAT 后面（比如家庭宽带），能正常工作吗？

<details><summary>答案</summary>

**可以，但需要满足几个条件。**

**DMVPN 天然支持 NAT 后面的 Spoke**，因为：
1. **Spoke 主动向 Hub 注册**（NHRP Registration），不需要 Hub 主动连接 Spoke
2. **NHRP 会记录 NAT 后的公网地址**（Hub 看到的源地址）
3. **IPsec NAT-T** 让 ESP 能穿越 NAT

**验证 NAT 检测**：
```cisco
Hub# show dmvpn detail
 # Ent  Peer NBMA Addr  Peer Tunnel Add  State  UpDn Tm  Attrb
     1  ★ 203.5.5.5 ★    10.0.0.11        UP   00:15:23  ★ DN ★
        ↑ NAT 后的公网 IP                                    ↑
                                                    N = NATed（检测到 NAT）

Hub# show crypto ikev2 sa detail
 Tunnel-id Local            Remote           fvrf/ivrf  Status
 1         3.3.3.3/★4500★   203.5.5.5/★4500★ none/none  READY
                     ↑                 ↑
              ★ 端口是 4500 = NAT-T 生效 ★
```

## 必须满足的条件

**① Hub 必须有【固定的公网 IP】**
```
   Spoke 需要主动连接 Hub
        ↓
   ★ Hub 的地址必须是可预知的 ★
        ↓
   Hub 不能在 NAT 后面（除非做了端口映射且地址固定）
```

**② NAT 设备必须放行**：
```
   · UDP 500   （IKE）
   · UDP 4500  （NAT-T）
   · ESP（协议号 50）—— 如果没有 NAT-T
```

**③ 必须启用 IPsec NAT-T**（现代 IOS 默认启用）
```cisco
Hub(config)# crypto ipsec nat-transparency udp-encapsulation
! 通常是默认的，可以用这条确认
```

**④ NAT 设备的会话超时不能太短**
```
   NHRP 注册默认每 1/3 holdtime 刷新一次
   holdtime 600 秒 → 每 200 秒刷新
        ↓
   如果 NAT 设备的 UDP 会话超时 < 200 秒
        ↓
   ★ 映射会老化，Hub 发不回来 ★
```

**解法：缩短 NHRP holdtime 或启用 keepalive**
```cisco
Spoke(config-if)# ip nhrp holdtime 300           ! 每 100 秒刷新
Spoke(config-if)# ip nhrp registration timeout 60

! IPsec DPD（Dead Peer Detection）也能保活
Spoke(config)# crypto ikev2 dpd 30 5 periodic
```

## Spoke-to-Spoke 的 NAT 限制（★ 重点）

| 场景 | Spoke-to-Spoke 直连 |
|:--|:--|
| **两个 Spoke 都有公网 IP** | ✅ 可以 |
| **一个在 NAT 后，一个有公网 IP** | ✅ **通常可以**（NAT 后的主动发起） |
| **两个都在 NAT 后（对称 NAT）** | ❌ **通常不行** |
| 两个都在 NAT 后（锥形 NAT） | ⚠️ 可能可以（取决于 NAT 类型） |

**为什么两个都在 NAT 后通常不行**：
```
   Spoke1 (NAT-A 后) 想直连 Spoke2 (NAT-B 后)
        ↓
   Spoke1 知道 Spoke2 的公网 IP（NAT-B 的地址）
        ↓
   Spoke1 主动发包给 NAT-B
        ↓
   ★ NAT-B 上没有对应的映射（Spoke2 没有主动发过包给 Spoke1）★
        ↓
   ★ 包被 NAT-B 丢弃 ★
```

这就是 **NAT 穿越（NAT Traversal）的经典难题**，需要"打洞"技术（STUN/TURN）才能解决——而 DMVPN 没有内置这个能力。

**后果**：两个 NAT 后的 Spoke 之间的流量**回退到走 Hub**（Phase 2/3 的直连失效）。

**这在实际部署中很常见**（分支都用家庭宽带/4G）。

## 实践建议

**① Hub 必须有固定公网 IP，且不在 NAT 后**
如果 Hub 必须在 NAT 后，要在 NAT 设备上做端口映射：
```
   UDP 500  → Hub
   UDP 4500 → Hub
   ESP      → Hub（如果需要）
```

**② 部署双 Hub 提高可用性**
```cisco
Spoke(config-if)# ip nhrp nhs 10.0.0.1 nbma <Hub1公网IP> multicast
Spoke(config-if)# ip nhrp nhs 10.0.0.2 nbma <Hub2公网IP> multicast
Spoke(config-if)# ip nhrp nhs cluster 0 max-connections 2
```

**③ 接受"NAT 后的 Spoke 之间可能走 Hub"这个现实**
在规划带宽时要考虑到这部分流量。

**④ 用 DPD 保活，防止 NAT 映射老化**
```cisco
Spoke(config)# crypto ikev2 dpd 30 5 periodic
Spoke(config-if)# ip nhrp holdtime 300
```

**⑤ 验证 NAT 场景**：
```cisco
Hub# show dmvpn detail
! 看 N 标志

Hub# show crypto ikev2 sa detail
! 看端口是不是 4500（NAT-T 生效）

Spoke1# show dmvpn
! 看能否与其他 NAT 后的 Spoke 建立动态隧道
```

> **现代替代方案**：如果 NAT 场景很复杂，**SD-WAN 是更好的选择**——vBond 专门负责 NAT 检测和穿越协商，能处理 DMVPN 处理不了的复杂 NAT 场景。参见 [ENCOR 第 15 章](../02-ENCOR-350-401/15-SD-Access与SD-WAN.md)。
</details>

---

**上一章** ← [05 MPLS 与 L3VPN](05-MPLS与L3VPN.md) ｜ **下一章** → [07 基础设施安全与服务排障](07-基础设施安全与服务排障.md)
