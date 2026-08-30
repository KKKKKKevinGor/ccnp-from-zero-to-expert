# Lab 08 · GRE over IPsec + VRF

**对应章节**：[ENCOR 02](../02-ENCOR-350-401/02-网络虚拟化-VRF-GRE-IPsec.md)
**难度**：★★★　**时长**：2 小时

## 目标

1. 建 GRE 隧道，让 OSPF 跨"互联网"运行
2. **亲手制造递归路由，看到隧道反复 up/down**
3. **亲手制造 MTU 黑洞，理解 `ip mtu` 和 `ip tcp adjust-mss` 的区别**
4. 加 IPsec 加密，用 `encaps/decaps` 计数器排障
5. 用 VRF 从架构上根除递归路由

---

## 拓扑

```
   总部 10.1.0.0/16                        分支 10.2.0.0/16
        │                                        │
     [HQ-LAN]                                [BR-LAN]
        │                                        │
   ┌────▼─────┐                            ┌─────▼────┐
   │    R1    │ 202.1.1.1                  │    R2    │ 203.2.2.2
   │  (总部)   ├──────[ ISP 模拟互联网 ]─────┤  (分支)   │
   └──────────┘                            └──────────┘
        └───────── Tunnel0: 172.16.0.0/30 ─────────┘
                （GRE over IPsec，跑 OSPF）
```

| 设备 | 内网 | 公网 IP | 隧道 IP |
|:--|:--|:--|:--|
| R1（总部） | 10.1.1.0/24 | 202.1.1.1 | 172.16.0.1 |
| R2（分支） | 10.2.1.0/24 | 203.2.2.2 | 172.16.0.2 |
| ISP | — | 202.1.1.254 / 203.2.2.254 | — |

---

## Part 1：Underlay（模拟互联网）

```cisco
! ── R1 ──
R1(config)# interface GigabitEthernet0/1
R1(config-if)# description ### To ISP ###
R1(config-if)# ip address 202.1.1.1 255.255.255.0
R1(config-if)# no shutdown

R1(config)# interface GigabitEthernet0/0
R1(config-if)# description ### HQ LAN ###
R1(config-if)# ip address 10.1.1.1 255.255.255.0
R1(config-if)# no shutdown

! ★ 只有一条默认路由指向 ISP（模拟真实的互联网接入）★
R1(config)# ip route 0.0.0.0 0.0.0.0 202.1.1.254

! ── R2 类似 ──
R2(config)# ip route 0.0.0.0 0.0.0.0 203.2.2.254

! ── ISP（只做转发，不知道任何私网）──
ISP(config)# interface GigabitEthernet0/1
ISP(config-if)# ip address 202.1.1.254 255.255.255.0
ISP(config)# interface GigabitEthernet0/2
ISP(config-if)# ip address 203.2.2.254 255.255.255.0
```

**验证公网可达**：
```cisco
R1# ping 203.2.2.2
!!!!!                                              ← 公网通 ✓

R1# ping 10.2.1.1
! ★ 不通（ISP 不知道私网）★
```

---

## Part 2：建 GRE 隧道

### Step 1：配置

```cisco
! ── R1 ──
R1(config)# interface Tunnel0
R1(config-if)# description ### GRE to Branch ###
R1(config-if)# ip address 172.16.0.1 255.255.255.252
R1(config-if)# ★ tunnel source GigabitEthernet0/1 ★
R1(config-if)# ★ tunnel destination 203.2.2.2 ★
R1(config-if)# tunnel mode gre ip                  ! 默认值，可省略
R1(config-if)# ★ ip mtu 1400 ★
R1(config-if)# ★ ip tcp adjust-mss 1360 ★
R1(config-if)# keepalive 10 3                      ! 隧道保活
R1(config-if)# no shutdown

! ── R2 ──
R2(config)# interface Tunnel0
R2(config-if)# ip address 172.16.0.2 255.255.255.252
R2(config-if)# tunnel source GigabitEthernet0/1
R2(config-if)# tunnel destination 202.1.1.1
R2(config-if)# ip mtu 1400
R2(config-if)# ip tcp adjust-mss 1360
R2(config-if)# keepalive 10 3
```

### Step 2：验证隧道

```cisco
R1# show ip interface brief | include Tunnel
Tunnel0    172.16.0.1    YES manual  ★ up      up ★

R1# ping 172.16.0.2
!!!!!                                              ← 隧道内通 ✓

R1# show interfaces Tunnel0
Tunnel0 is up, line protocol is up
  Hardware is Tunnel
  Internet address is 172.16.0.1/30
  ★ MTU 17916 bytes ★, BW 100 Kbit/sec, DLY 50000 usec
  Tunnel source 202.1.1.1 (GigabitEthernet0/1), destination 203.2.2.2
  ★ Tunnel protocol/transport GRE/IP ★
  Key disabled, sequencing disabled
  Checksumming of packets disabled
  Tunnel TTL 255, Fast tunneling enabled
  ★ Tunnel transport MTU 1476 bytes ★
                       ↑ 1500 - 24（GRE 开销）
```

### Step 3：★ 隧道的价值 —— 在上面跑 OSPF

```cisco
R1(config)# router ospf 1
R1(config-router)# router-id 1.1.1.1
R1(config-router)# ★ network 172.16.0.0 0.0.0.3 area 0 ★       ! 隧道地址
R1(config-router)# network 10.1.0.0 0.0.255.255 area 0         ! 内网

R1(config)# interface Tunnel0
R1(config-if)# ★ ip ospf network point-to-point ★              ! 省掉 DR 选举

! ── R2 ──
R2(config)# router ospf 1
R2(config-router)# router-id 2.2.2.2
R2(config-router)# network 172.16.0.0 0.0.0.3 area 0
R2(config-router)# network 10.2.0.0 0.0.255.255 area 0
R2(config)# interface Tunnel0
R2(config-if)# ip ospf network point-to-point
```

**验证**：
```cisco
R1# show ip ospf neighbor
Neighbor ID  Pri  State     Dead Time  Address      Interface
2.2.2.2        0  ★FULL/  -★  00:00:35  172.16.0.2  ★ Tunnel0 ★

R1# show ip route ospf
★ O    10.2.1.0/24 [110/1001] via 172.16.0.2, 00:02:15, Tunnel0 ★

R1# ping 10.2.1.1 source 10.1.1.1
!!!!!                                              ← 内网互通 ✓
```

**★★ 这就是 GRE 的核心价值：OSPF 在互联网上跑起来了。**

**为什么纯 IPsec 做不到**：IPsec 不支持组播，OSPF 的 Hello（224.0.0.5）传不过去。

---

## Part 3：★ 故障实验 1 —— 递归路由

### Step 1：制造递归路由

```cisco
! ★ 故意把公网网段通告进 OSPF ★
R1(config)# router ospf 1
R1(config-router)# ★ network 202.1.1.0 0.0.0.255 area 0 ★

R2(config)# router ospf 1
R2(config-router)# ★ network 203.2.2.0 0.0.0.255 area 0 ★
```

### Step 2：观察隧道震荡

```cisco
R1# show logging | include TUN|LINEPROTO
★ %TUN-5-RECURDOWN: Tunnel0 temporarily disabled due to recursive routing ★
%LINEPROTO-5-UPDOWN: Line protocol on Interface Tunnel0, changed state to down
%LINEPROTO-5-UPDOWN: Line protocol on Interface Tunnel0, changed state to up
★ %TUN-5-RECURDOWN: Tunnel0 temporarily disabled due to recursive routing ★
%LINEPROTO-5-UPDOWN: Line protocol on Interface Tunnel0, changed state to down
! ★ 反复循环 ★
```

```cisco
R1# show ip interface brief | include Tunnel
Tunnel0    172.16.0.1    YES manual  up    ★ down ★
! （一会儿 up 一会儿 down）
```

### Step 3：★ 理解递归路由的死循环

```cisco
R1# show ip route 203.2.2.2
Routing entry for 203.2.2.0/24
  Known via "★ ospf 1 ★", distance 110, metric 1001
  Routing Descriptor Blocks:
  * 172.16.0.2, from 2.2.2.2, via ★ Tunnel0 ★
                                    ↑↑↑↑↑↑↑
    ★ 去 203.2.2.2（隧道目的地址）的路由，指向了 Tunnel0 自己！★
```

**死循环过程**：
```
   ① Tunnel0 的 destination = 203.2.2.2
   ② OSPF 通过隧道学到 "203.2.2.0/24 via Tunnel0"
      （这条比原来的默认路由更具体，进了路由表）
   ③ 现在要访问 203.2.2.2，路由表说"走 Tunnel0"
   ④ 但 Tunnel0 的外层封装需要先能到 203.2.2.2
   ⑤ ★ 无解 → 隧道 down ★
   ⑥ 隧道 down 后，OSPF 邻居断，那条路由消失
   ⑦ ★ 隧道又能 up 了 → 回到第 ② 步，无限循环 ★
```

### Step 4：三种修复方法

**方法 1：不要把公网网段通告进 OSPF（最直接）**
```cisco
R1(config-router)# ★ no network 202.1.1.0 0.0.0.255 area 0 ★
R2(config-router)# no network 203.2.2.0 0.0.0.255 area 0
```

**方法 2：加 /32 静态路由（利用最长匹配）**
```cisco
R1(config)# ★ ip route 203.2.2.2 255.255.255.255 202.1.1.254 ★
!               ↑ /32 永远比 OSPF 学到的 /24 更具体
```
```cisco
R1# show ip route 203.2.2.2
Routing entry for ★ 203.2.2.2/32 ★
  Known via "★ static ★", distance 1, metric 0
  Routing Descriptor Blocks:
  * ★ 202.1.1.254 ★                             ← 走真实的公网路径 ✓
```

**方法 3：用 VRF 分离（★ 最优雅，见 Part 6）**

### Step 5：验证修复

```cisco
R1# show logging | include RECURDOWN
! ★ 不再有新的记录 ★

R1# show ip interface brief | include Tunnel
Tunnel0    172.16.0.1    YES manual  ★ up      up ★         ← 稳定了 ✓
```

---

## Part 4：★ 故障实验 2 —— MTU 黑洞

### Step 1：去掉 MTU 配置

```cisco
R1(config)# interface Tunnel0
R1(config-if)# ★ no ip mtu 1400 ★
R1(config-if)# ★ no ip tcp adjust-mss 1360 ★

R2(config)# interface Tunnel0
R2(config-if)# no ip mtu 1400
R2(config-if)# no ip tcp adjust-mss 1360
```

### Step 2：观察症状

```cisco
! ── 小包正常 ──
R1# ping 10.2.1.1 source 10.1.1.1 size 100
!!!!!                                              ← ✅ 通

! ── 大包不通 ──
R1# ping 10.2.1.1 source 10.1.1.1 ★ size 1500 df-bit ★
Type escape sequence to abort.
Sending 5, 1500-byte ICMP Echos to 10.2.1.1, timeout is 2 seconds:
Packet sent with the DF bit set
★ M.M.M ★
Success rate is 0 percent (0/5)
! ★ M = 需要分片但 DF 位置位 ★
```

**从 PC 访问对端 Web 服务**：**页面卡住加载不完**（这是最典型的用户报障）。

### Step 3：二分法找出实际可用 MTU

```cisco
R1# ping 10.2.1.1 size 1476 df-bit
!!!!!                                              ← 通

R1# ping 10.2.1.1 size 1477 df-bit
M.M.M                                              ← 不通

! ★ 实际可用 MTU = 1476 = 1500 - 24（GRE 开销）★
```

### Step 4：★ 理解 `ip mtu` 和 `ip tcp adjust-mss` 的区别

**只配 `ip mtu`**：
```cisco
R1(config-if)# ip mtu 1400
R2(config-if)# ip mtu 1400
```
```cisco
R1# ping 10.2.1.1 size 1500 df-bit
M.M.M                                              ← ★ 依然不通！
```

**为什么**：`ip mtu` 只是**限制从这个接口出去的包最大长度**。当 1500 字节的包（带 DF 位）到达时，路由器**丢弃它并回一个 ICMP "Fragmentation Needed"**。

**如果路径上 ICMP 被拦，源端收不到通知 → 一直重发大包 → ★ PMTUD 黑洞 ★**

```cisco
! 模拟 ICMP 被拦
ISP(config)# ip access-list extended BLOCK-ICMP
ISP(config-ext-nacl)#  deny icmp any any unreachable
ISP(config-ext-nacl)#  permit ip any any
ISP(config)# interface GigabitEthernet0/1
ISP(config-if)# ip access-group BLOCK-ICMP out
```

**加上 `ip tcp adjust-mss`**：
```cisco
R1(config-if)# ★ ip tcp adjust-mss 1360 ★
R2(config-if)# ip tcp adjust-mss 1360
```

**验证 TCP 流量正常了**：
```cisco
! 从 PC 访问对端的 Web 服务
PC-HQ> curl http://10.2.1.100
! ★ 正常 ✓ ★
```

**★ 抓包看 MSS 被改写**：
```
   客户端 → SYN, MSS=1460
              ↓ 路由器改写
         SYN, MSS=★1360★                ← 到达服务器时已被改小
   服务器 → SYN+ACK, MSS=1460
              ↓ 路由器改写
         SYN+ACK, MSS=★1360★
              ↓
   ★ 两端协商结果：MSS=1360，永远不产生超过 1400 字节的 IP 包 ★
   ★ 根本不需要 PMTUD，不依赖 ICMP ★
```

### Step 5：★ 对比总结

| 命令 | 作用范围 | 机制 | 时机 |
|:--|:--|:--|:--|
| **`ip mtu 1400`** | **所有 IP 流量** | 限制包长，超过就分片或丢弃+回 ICMP | **被动** |
| **`ip tcp adjust-mss 1360`** | **只对 TCP** | **改写 SYN 包里的 MSS 选项** | **主动，从源头避免** |

**为什么两个都要配**：
- `adjust-mss` **只管 TCP**。UDP 流量（VoIP、视频、DNS 大响应）不受影响，需要 `ip mtu` 兜底
- `ip mtu` 也决定路由协议报文的大小

**数值计算**：
```
   物理 MTU              1500
   减 GRE 开销            -24  → 1476
   减 IPsec 开销（约）     -58  → 1418
   保守取值 ip mtu:              ★ 1400 ★
   再减 IP头20+TCP头20:   -40  → ★ 1360 ★ = adjust-mss
```

**★ 实战建议：直接用 `ip mtu 1400` + `ip tcp adjust-mss 1360`。** 这组值对 GRE、IPsec、GRE over IPsec 都留了足够余量。

**★ 同时：不要在 ACL 里无脑拦所有 ICMP**：
```cisco
permit icmp any any ★ unreachable ★           ! PMTUD 需要
permit icmp any any time-exceeded             ! traceroute 需要
permit icmp any any echo-reply
```

---

## Part 5：加 IPsec 加密

### Step 1：IKEv2 配置

```cisco
! ══════ R1 和 R2 都配（参数必须完全一致）══════

! ── ① IKEv2 提案（HAGLE 五要素）──
crypto ikev2 proposal PROP-1
 ★ encryption aes-cbc-256 ★
 ★ integrity sha256 ★
 ★ group 14 ★

crypto ikev2 policy POL-1
 proposal PROP-1

! ── ② 密钥环 ──
crypto ikev2 keyring KR-1
 peer REMOTE
  ! R1 上写 R2 的公网 IP，R2 上写 R1 的
  address 203.2.2.2
  ★ pre-shared-key MyVerySecretKey123! ★

! ── ③ IKEv2 profile ──
crypto ikev2 profile PROF-1
 match identity remote address 203.2.2.2 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local KR-1

! ── ④ IPsec 转换集 ──
crypto ipsec transform-set TS-1 esp-aes 256 esp-sha256-hmac
 ★ mode transport ★                      ! ★ GRE over IPsec 用传输模式

! ── ⑤ IPsec profile ──
crypto ipsec profile IPSEC-PROF
 set transform-set TS-1
 set ikev2-profile PROF-1
 set pfs group14

! ── ⑥ 应用到隧道（★ 一条命令搞定）──
interface Tunnel0
 ★ tunnel protection ipsec profile IPSEC-PROF ★
```

**★ `tunnel protection` 的优雅之处**：
- 不需要写 crypto map
- 不需要定义"感兴趣流" ACL
- **所有经过这个隧道的流量自动被加密**

**★ 为什么用 `mode transport`**：GRE 已经加了一层新 IP 头，IPsec 隧道模式会**再加一层**，白白浪费 20 字节。传输模式复用 GRE 的外层头。

### Step 2：验证

```cisco
R1# show crypto ikev2 sa
 IPv4 Crypto IKEv2  SA

 Tunnel-id Local            Remote           fvrf/ivrf  Status
 1         202.1.1.1/500    203.2.2.2/500    none/none  ★ READY ★
                                                          ↑ Phase 1 成功

R1# show crypto session
Interface: Tunnel0
Session status: ★ UP-ACTIVE ★
Peer: 203.2.2.2 port 500
  Session ID: 1
  IKEv2 SA: local 202.1.1.1/500 remote 203.2.2.2/500 Active
  IPSEC FLOW: permit 47 host 202.1.1.1 host 203.2.2.2
                     ↑ 协议 47 = GRE
        Active SAs: 2, origin: crypto map
```

### Step 3：★ 核心排障工具 —— encaps/decaps 计数器

```cisco
R1# show crypto ipsec sa | include pkts
    ★ #pkts encaps: 1245 ★, #pkts encrypt: 1245, #pkts digest: 1245
    ★ #pkts decaps: 1198 ★, #pkts decrypt: 1198, #pkts verify: 1198
         ↑                       ↑
      出方向计数             入方向计数
```

**★ 诊断对照表（IPsec 排障最高效的工具）**：

| encaps | decaps | 诊断 |
|:--|:--|:--|
| **涨** | **涨** | ✅ **双向正常** |
| **涨** | **0** | ❌ **对端问题**：对端配置/路由/NAT/ACL |
| **0** | 涨 | ❌ **本端问题**：路由没把流量送进隧道，或 NAT 改了源地址 |
| 0 | 0 | ❌ 隧道完全没流量：路由、感兴趣流、或没业务在跑 |

**30 秒就能判断问题在哪一端。**

### Step 4：抓包验证加密

```cisco
! 在 R1-ISP 之间抓包
R1# monitor capture CAP interface GigabitEthernet0/1 both
R1# monitor capture CAP match ipv4 any any
R1# monitor capture CAP start
R1# ping 10.2.1.1 source 10.1.1.1 repeat 10
R1# monitor capture CAP stop
R1# show monitor capture CAP buffer brief
```

**加密前**（只有 GRE）：
```
IP 202.1.1.1 > 203.2.2.2: ★ GRE ★, length 104: IP 10.1.1.1 > 10.2.1.1: ICMP echo request
                            ↑ 能看到内层 IP
```

**加密后**（ESP）：
```
IP 202.1.1.1 > 203.2.2.2: ★ ESP(spi=0x12345678,seq=0x1) ★, length 132
                            ↑ ★ 内容不可读 ✓ ★
```

---

## Part 6：★ 故障实验 3 —— IPsec 参数不匹配

### Step 1：制造 Phase 1 参数不匹配

```cisco
R2(config)# crypto ikev2 proposal PROP-1
R2(config-ikev2-proposal)# ★ group 5 ★                 ! R1 是 group 14
```

```cisco
R1# clear crypto ikev2 sa
R1# show crypto ikev2 sa
! ★ 空输出，或状态一直在 IN-NEG ★

R1# debug crypto ikev2
★ IKEv2:(SESSION ID = 1,SA ID = 1):Failed to find a matching policy ★
```

**★ Phase 1 必须完全匹配的五要素（HAGLE）**：
```
   ★ H ★ash          （sha256）
   ★ A ★uthentication（pre-share）
   ★ G ★roup (DH)    （group 14）
   ★ L ★ifetime      （可以不同，取较小值）
   ★ E ★ncryption    （aes-cbc-256）
```

**修复**：`R2(config-ikev2-proposal)# group 14`

### Step 2：制造预共享密钥不匹配

```cisco
R2(config)# crypto ikev2 keyring KR-1
R2(config-ikev2-keyring)# peer REMOTE
R2(config-ikev2-keyring-peer)# ★ pre-shared-key WrongKey ★
```

```cisco
R1# debug crypto ikev2
★ IKEv2:(SESSION ID = 1,SA ID = 1):Verification of peer's authentication data FAILED ★
```

### Step 3：制造 ACL 拦截（缺 ESP 放行）

```cisco
ISP(config)# ip access-list extended BLOCK-VPN
ISP(config-ext-nacl)#  permit udp any any eq 500
ISP(config-ext-nacl)#  permit udp any any eq 4500
ISP(config-ext-nacl)#  ★ deny esp any any ★              ! 故意不放行 ESP
ISP(config-ext-nacl)#  permit ip any any
ISP(config)# interface GigabitEthernet0/1
ISP(config-if)# ip access-group BLOCK-VPN in
```

```cisco
R1# show crypto ikev2 sa
 1  202.1.1.1/500  203.2.2.2/500  none/none  ★ READY ★
! ★ Phase 1 成功（UDP 500 通）★

R1# show crypto ipsec sa | include pkts
    #pkts encaps: 456, #pkts encrypt: 456
    ★ #pkts decaps: 0 ★, #pkts decrypt: 0
                     ↑ ★ 收不到对端的加密流量 ★

R1# ping 10.2.1.1 source 10.1.1.1
.....                                              ← 不通
```

**★ 这是"IKE 起来了但数据不通"的典型场景。**

**排查**：
```cisco
ISP# show access-lists BLOCK-VPN
Extended IP access list BLOCK-VPN
    10 permit udp any any eq isakmp (24 matches)
    20 permit udp any any eq 4500 (0 matches)
    30 ★ deny esp any any (1245 matches) ★         ← 找到了
    40 permit ip any any
```

**★ ACL 必须放行三样**：
```cisco
permit udp any any eq 500        ! IKE 阶段一
permit udp any any eq 4500       ! NAT-T
★ permit esp any any ★           ! ESP（协议号 50）★ 最容易漏
```

**为什么容易漏**：ESP 没有端口号，写 ACL 时容易只想到 UDP 500/4500。

---

## Part 7：★ 用 VRF 根除递归路由

### 思路

```
   把"公网传输"和"业务网络"放在不同的路由表里
        ↓
   ★ 业务路由（OSPF 学到的）不会影响公网路由表 ★
        ↓
   ★ 从架构上杜绝递归路由 ★
```

**这也是 DMVPN 和 SD-WAN 的标准做法。**

### 配置

```cisco
! ── ① 定义 VRF ──
R1(config)# vrf definition CORP
R1(config-vrf)#  description ### 业务网络 ###
R1(config-vrf)#  address-family ipv4

! ── ② 内网接口划入 VRF ──
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ★ vrf forwarding CORP ★             ! ★ 会清除 IP，要重新配
R1(config-if)# ip address 10.1.1.1 255.255.255.0

! ── ③ 隧道：内部属于 VRF，外部封装在 global 表 ──
R1(config)# interface Tunnel0
R1(config-if)# ★ vrf forwarding CORP ★             ! 隧道内部流量属于 CORP
R1(config-if)# ip address 172.16.0.1 255.255.255.252
R1(config-if)# tunnel source GigabitEthernet0/1
R1(config-if)# tunnel destination 203.2.2.2
R1(config-if)# ★ tunnel vrf default ★              ! ★★ 外层封装在 global 表查路由
R1(config-if)# ip mtu 1400
R1(config-if)# ip tcp adjust-mss 1360
R1(config-if)# tunnel protection ipsec profile IPSEC-PROF

! ── ④ 公网接口保持在 global 表 ──
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip address 202.1.1.1 255.255.255.0
! （不划入任何 VRF）

! ── ⑤ 公网默认路由在 global 表 ──
R1(config)# ip route 0.0.0.0 0.0.0.0 202.1.1.254

! ── ⑥ OSPF 在 VRF 里跑 ──
R1(config)# ★ router ospf 1 vrf CORP ★
R1(config-router)# router-id 1.1.1.1
R1(config-router)# network 172.16.0.0 0.0.0.3 area 0
R1(config-router)# network 10.1.0.0 0.0.255.255 area 0
```

### 验证

```cisco
! ── global 表：只有公网路由 ──
R1# show ip route
Gateway of last resort is 202.1.1.254 to network 0.0.0.0
S*    0.0.0.0/0 [1/0] via 202.1.1.254
      202.1.1.0/24 is directly connected, GigabitEthernet0/1
! ★ 没有任何私网路由 ★

! ── CORP VRF：只有业务路由 ──
R1# ★ show ip route vrf CORP ★
      10.1.1.0/24 is directly connected, GigabitEthernet0/0
★ O    10.2.1.0/24 [110/1001] via 172.16.0.2, Tunnel0 ★
      172.16.0.0/30 is directly connected, Tunnel0
! ★ 没有任何公网路由 ★

! ── 测试 ──
R1# ★ ping vrf CORP 10.2.1.1 source 10.1.1.1 ★
!!!!!                                              ✓
```

### ★ 验证递归路由被根除

```cisco
! 故意在 VRF 的 OSPF 里通告公网网段（模拟误配）
R1(config)# router ospf 1 vrf CORP
R1(config-router)# network 202.1.1.0 0.0.0.255 area 0

! ★ 但这个接口在 global 表，不在 CORP VRF 里，所以这条 network 语句不生效 ★
```

**即使真的有一条 `203.2.2.0/24` 出现在 CORP VRF 里**：
```cisco
R1# show ip route vrf CORP 203.2.2.0
! 假设有一条指向 Tunnel0 的路由

! ★ 但隧道的外层封装是在 global 表查路由的（tunnel vrf default）★
R1# show ip route 203.2.2.2                        ! global 表
S*    0.0.0.0/0 [1/0] via 202.1.1.254              ← 走真实公网路径 ✓
```

**★★ 两张路由表物理隔离，CORP VRF 学到什么都不会影响 global 表 → 从架构上根除递归路由 ★★**

### 常用 VRF 命令

```cisco
R1# show vrf
R1# show vrf detail CORP
R1# show ip route vrf CORP
R1# show ip interface brief vrf CORP
R1# show ip arp vrf CORP
R1# ★ ping vrf CORP <目标> ★
R1# ★ traceroute vrf CORP <目标> ★
R1# telnet <目标> /vrf CORP
```

**★ 排障第一课**：在有 VRF 的设备上，**`show ip route` 只显示 global 表**。忘记加 `vrf CORP` 会让你以为"路由丢了"。

---

## 实验检查清单

```
□ ① GRE 隧道建立，OSPF 跨"互联网"运行
□ ② ★ 亲手制造递归路由，看到 %TUN-5-RECURDOWN 反复刷屏
□ ③ 用三种方法修复递归路由
□ ④ ★ 亲手制造 MTU 黑洞，二分法找出实际 MTU
□ ⑤ ★ 验证 ip mtu 和 ip tcp adjust-mss 的区别
□ ⑥ 加 IPsec，验证 IKE SA 和 IPsec SA
□ ⑦ ★ 用 encaps/decaps 计数器判断问题在哪一端
□ ⑧ 抓包对比加密前后
□ ⑨ 制造三种 IPsec 故障（Phase1参数/密钥/ESP被拦）
□ ⑩ ★ 用 VRF 从架构上根除递归路由
□ ⑪ 写了实验笔记
```

## 关键命令总结

| 命令 | 用途 |
|:--|:--|
| `show interfaces Tunnel0` | 隧道状态、transport MTU |
| **`show logging \| include RECURDOWN`** | ★ 递归路由 |
| **`ping <目标> size 1500 df-bit`** | ★ MTU 测试 |
| `show crypto ikev2 sa` | Phase 1 状态 |
| **`show crypto ipsec sa \| include pkts`** | ★★ encaps/decaps 诊断 |
| `show crypto session` | 会话总览 |
| `debug crypto ikev2` | Phase 1 协商失败原因 |
| **`show ip route vrf CORP`** | ★ VRF 路由表（别忘了加 vrf） |
| `ping vrf CORP <目标>` | VRF 内测试 |

## 核心结论

| 要点 | 说明 |
|:--|:--|
| **GRE 让动态路由跨公网运行** | 纯 IPsec 不支持组播 |
| **GRE over IPsec 用 transport 模式** | 复用 GRE 的外层 IP 头 |
| **递归路由**：隧道目的地址的路由指向隧道自己 | 三种解法，VRF 最优 |
| **MTU 必配 `ip mtu 1400` + `ip tcp adjust-mss 1360`** | 前者管所有，后者主动改 TCP MSS |
| **不要拦 `icmp unreachable`** | PMTUD 需要 |
| **ACL 必须放行 UDP 500/4500 + ★ESP★** | ESP 最容易漏 |
| **encaps 涨 decaps 不涨 = 对端问题** | 30 秒定位 |
| **VRF 分离 Underlay 和 Overlay** | SD-WAN 的标准做法 |

---

**上一个** ← [Lab 07](lab-07-重分发与路由策略.md) ｜ **下一个** → [Lab 09: DMVPN](lab-09-DMVPN.md)
