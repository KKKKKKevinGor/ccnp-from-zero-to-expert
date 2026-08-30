# Lab 09 · DMVPN Phase 3

**对应章节**：[ENARSI 06](../03-ENARSI-300-410/06-DMVPN.md)
**难度**：★★★★　**时长**：3 小时

## 目标

1. 搭建 DMVPN Phase 1，理解 mGRE + NHRP
2. 升级到 Phase 2，**验证"Hub 汇总后 Spoke-to-Spoke 失效"**
3. 升级到 Phase 3，**亲眼看到动态 Spoke-to-Spoke 隧道的建立与拆除**
4. 加 IPsec，用 `show dmvpn` 的标志位排障
5. 制造并排查 5 个典型故障

---

## 拓扑

```
                    [ Internet ]
              ┌──────────┼──────────┐
              │          │          │
         1.1.1.11    3.3.3.3    2.2.2.22
        ┌───▼───┐  ┌────▼───┐  ┌────▼───┐
        │Spoke1 │  │  Hub   │  │Spoke2  │
        │10.0.0.11  │10.0.0.1│  │10.0.0.12
        └───┬───┘  └────┬───┘  └────┬───┘
            │           │           │
     192.168.11.0/24  192.168.0.0/24  192.168.12.0/24
```

| 设备 | 公网 IP | 隧道 IP | 内网 |
|:--|:--|:--|:--|
| Hub | 3.3.3.3 | 10.0.0.1 | 192.168.0.0/24 |
| Spoke1 | 1.1.1.11 | 10.0.0.11 | 192.168.11.0/24 |
| Spoke2 | 2.2.2.22 | 10.0.0.12 | 192.168.12.0/24 |

---

## Part 1：Phase 1（纯星型）

### Step 1：Hub 配置

```cisco
Hub(config)# interface Tunnel0
Hub(config-if)# description ### DMVPN Hub ###
Hub(config-if)# ip address 10.0.0.1 255.255.255.0
Hub(config-if)# ★ ip mtu 1400 ★
Hub(config-if)# ★ ip tcp adjust-mss 1360 ★
Hub(config-if)# tunnel source GigabitEthernet0/1
Hub(config-if)# ★ tunnel mode gre multipoint ★           ! ★ mGRE，没有 destination
Hub(config-if)# tunnel key 100
! ── NHRP ──
Hub(config-if)# ★ ip nhrp network-id 100 ★
Hub(config-if)# ip nhrp authentication DmvpnNhrpKey
Hub(config-if)# ★ ip nhrp map multicast dynamic ★        ! ★ 动态学习组播成员
Hub(config-if)# ip nhrp holdtime 600
Hub(config-if)# no shutdown
```

### Step 2：Spoke 配置（Phase 1 用 P2P GRE）

```cisco
Spoke1(config)# interface Tunnel0
Spoke1(config-if)# ip address 10.0.0.11 255.255.255.0
Spoke1(config-if)# ip mtu 1400
Spoke1(config-if)# ip tcp adjust-mss 1360
Spoke1(config-if)# tunnel source GigabitEthernet0/1
Spoke1(config-if)# ★ tunnel destination 3.3.3.3 ★        ! ★ Phase 1：P2P GRE
Spoke1(config-if)# tunnel key 100
! ── NHRP ──
Spoke1(config-if)# ip nhrp network-id 100
Spoke1(config-if)# ip nhrp authentication DmvpnNhrpKey
Spoke1(config-if)# ★ ip nhrp nhs 10.0.0.1 ★              ! Hub 的隧道 IP
Spoke1(config-if)# ★ ip nhrp map 10.0.0.1 3.3.3.3 ★      ! 隧道IP ↔ 公网IP
Spoke1(config-if)# ★ ip nhrp map multicast 3.3.3.3 ★     ! ★ 组播发给 Hub
Spoke1(config-if)# ip nhrp holdtime 600

! Spoke2 类似（隧道 IP 改成 10.0.0.12）
```

**IOS 15.2+ 的简化写法**：
```cisco
Spoke1(config-if)# ★ ip nhrp nhs 10.0.0.1 nbma 3.3.3.3 multicast ★
! 一条命令代替 nhs + map + map multicast
```

### Step 3：★ 验证 NHRP 注册

```cisco
Hub# ★ show dmvpn ★
Legend: Attrb --> S - Static, D - Dynamic, I - Incomplete
        N - NATed, L - Local, X - No Socket
        T1 - Route Installed, T2 - Nexthop-override

Interface: Tunnel0, IPv4 NHRP Details
Type: ★ Hub ★, NHRP Peers: 2,

 # Ent  Peer NBMA Addr  Peer Tunnel Add  State  UpDn Tm  Attrb
 ----- --------------- ---------------- ------ -------- -----
     1    ★1.1.1.11★      ★10.0.0.11★     ★UP★  00:00:45   ★D★
     1     2.2.2.22        10.0.0.12       UP   00:00:38    D
           ↑ 公网 IP        ↑ 隧道 IP        ↑         ↑
                                        状态正常    D=动态学到
```

```cisco
Hub# show ip nhrp
10.0.0.11/32 via 10.0.0.11
   Tunnel0 created 00:00:45, expire 00:09:15
   Type: ★ dynamic ★, Flags: unique registered used nhop
   ★ NBMA address: 1.1.1.11 ★
```

```cisco
Spoke1# ★ show ip nhrp nhs detail ★
Legend: E=Expecting replies, R=Responding, W=Waiting
Tunnel0:
  10.0.0.1  ★ RE ★  priority = 0 cluster = 0  req-sent 12  req-failed 0  repl-recv 12
            ↑↑
      R = Responding（Hub 在响应）✓
```

**★ 理解 NHRP 注册**：
```
   ① Spoke 启动，拿到公网 IP（★ 可以是动态的 ★）
   ② Spoke 向 NHS（Hub）发 NHRP Registration
      "我的隧道 IP 是 10.0.0.11，公网 IP 是 1.1.1.11"
   ③ Hub 记录到 NHRP 映射表
   ④ 周期性刷新（默认每 1/3 holdtime）
   
   ★ 这就是 DMVPN 支持"分支用动态公网 IP"的原因 ★
```

### Step 4：加路由协议

```cisco
! ── Hub ──
Hub(config)# router eigrp 100
Hub(config-router)# network 10.0.0.0 0.0.0.255
Hub(config-router)# network 192.168.0.0 0.0.0.255
Hub(config-router)# no auto-summary

Hub(config)# interface Tunnel0
Hub(config-if)# ★ no ip split-horizon eigrp 100 ★        ! ★★★ 必须

! ── Spoke ──
Spoke1(config)# router eigrp 100
Spoke1(config-router)# network 10.0.0.0 0.0.0.255
Spoke1(config-router)# network 192.168.11.0 0.0.0.255
Spoke1(config-router)# no auto-summary
```

**验证**：
```cisco
Spoke1# show ip eigrp neighbors
H   Address     Interface  Hold Uptime   SRTT  RTO  Q  Seq
0   10.0.0.1    ★ Tu0 ★     12  00:02:15   45  270  0  8

Spoke1# show ip route eigrp
D    192.168.0.0/24 [90/26882560] via 10.0.0.1, 00:02:10, Tunnel0
D    192.168.12.0/24 [90/28288000] via ★ 10.0.0.1 ★, 00:02:05, Tunnel0
                                          ↑ 下一跳是 Hub
```

### Step 5：★ 验证"关水平分割"的必要性

```cisco
Hub(config)# interface Tunnel0
Hub(config-if)# ★ ip split-horizon eigrp 100 ★           ! 打开水平分割
```

```cisco
Spoke1# show ip route eigrp
D    192.168.0.0/24 [90/26882560] via 10.0.0.1, Tunnel0
! ★ 192.168.12.0/24 不见了！★
```

**为什么**：
```
   Hub 从 Tunnel0 学到 Spoke2 的路由（192.168.12.0/24）
        ↓
   Hub 要把它告诉 Spoke1
        ↓
   ★ 但 Spoke1 也在 Tunnel0 上 ★
        ↓
   ★ 水平分割：从 Tunnel0 学来的不能再从 Tunnel0 发出 ★
        ↓
   ★ Spoke1 永远学不到 Spoke2 的路由 ★
```

**修复**：`no ip split-horizon eigrp 100`

### Step 6：测试流量路径（Phase 1）

```cisco
Spoke1# traceroute 192.168.12.1 source 192.168.11.1
  1 ★ 10.0.0.1 ★    20 msec       ← 经过 Hub
  2 10.0.0.12      35 msec
  3 192.168.12.1   40 msec
```

**★ Phase 1 的流量必须绕道 Hub**，因为 Spoke 的隧道是 P2P GRE，只能连 Hub。

---

## Part 2：Phase 2（支持 Spoke-to-Spoke，但不能汇总）

### Step 1：Spoke 改成 mGRE

```cisco
Spoke1(config)# interface Tunnel0
Spoke1(config-if)# ★ no tunnel destination 3.3.3.3 ★
Spoke1(config-if)# ★ tunnel mode gre multipoint ★        ! 改成 mGRE

Spoke2(config)# interface Tunnel0
Spoke2(config-if)# no tunnel destination 3.3.3.3
Spoke2(config-if)# tunnel mode gre multipoint
```

### Step 2：Hub 保留原始下一跳（Phase 2 关键）

```cisco
Hub(config)# interface Tunnel0
Hub(config-if)# ★ no ip next-hop-self eigrp 100 ★
```

### Step 3：验证下一跳变化

```cisco
Spoke1# show ip route eigrp
D    192.168.12.0/24 [90/28288000] via ★ 10.0.0.12 ★, Tunnel0
                                          ↑↑↑↑↑↑↑↑↑
              ★ 下一跳变成了 Spoke2 的隧道 IP（不是 Hub）★
```

### Step 4：验证 Spoke-to-Spoke 直连

```cisco
Spoke1# ping 192.168.12.1 source 192.168.11.1 repeat 20
!!!!!!!!!!!!!!!!!!!!

! 几秒后
Spoke1# traceroute 192.168.12.1 source 192.168.11.1
  1 ★ 10.0.0.12 ★  15 msec       ← ★ 直连了！不经过 Hub ✓
  2 192.168.12.1  18 msec
```

```cisco
Spoke1# show dmvpn
 # Ent  Peer NBMA Addr  Peer Tunnel Add  State  UpDn Tm  Attrb
     1     3.3.3.3         10.0.0.1        UP   00:20:15   ★S★    ← 到 Hub（静态）
     1  ★2.2.2.22★      ★10.0.0.12★      UP   00:00:35   ★D★    ← ★ 到 Spoke2！动态
```

**✅ Spoke-to-Spoke 动态隧道建立成功。**

### Step 5：★★ 核心实验 —— 验证 "Phase 2 不能汇总"

```cisco
Hub(config)# interface Tunnel0
Hub(config-if)# ★ ip summary-address eigrp 100 192.168.0.0 255.255.0.0 ★
```

**观察 Spoke 的路由表**：
```cisco
Spoke1# show ip route eigrp
D    192.168.0.0/16 [90/26882560] via ★ 10.0.0.1 ★, Tunnel0
                                          ↑↑↑↑↑↑↑↑
                          ★ 下一跳变回 Hub 了！★
! ★ 192.168.12.0/24 的明细不见了 ★
```

**测试流量路径**：
```cisco
! 先清掉已有的动态隧道
Spoke1# clear dmvpn session

Spoke1# traceroute 192.168.12.1 source 192.168.11.1
  1 ★ 10.0.0.1 ★    20 msec       ← ★ 又走 Hub 了
  2 10.0.0.12      35 msec
  3 192.168.12.1   40 msec

Spoke1# show dmvpn
 # Ent  Peer NBMA Addr  Peer Tunnel Add  State  UpDn Tm  Attrb
     1     3.3.3.3         10.0.0.1        UP   00:25:15    S
! ★ 没有到 Spoke2 的动态隧道 ★
```

**★★ 为什么**：
```
   Phase 2 的 Spoke-to-Spoke 依赖：
   ★ 路由表里去 Spoke2 网段的【下一跳】必须是 Spoke2 的隧道 IP ★
        ↓
   Hub 汇总后，Spoke1 收到的是：
   "192.168.0.0/16 → 下一跳 10.0.0.1（Hub）"
        ↓
   ★ 下一跳是 Hub，不是 Spoke2 ★
        ↓
   ★ 流量永远走 Hub，直连隧道建不起来 ★
```

**后果**：50 个分支时，每个分支要在路由表里装**其他 49 个分支的所有明细路由**——扩展性极差。

**撤销汇总**：
```cisco
Hub(config-if)# no ip summary-address eigrp 100 192.168.0.0 255.255.0.0
```

---

## Part 3：★ Phase 3（两全其美）

### Step 1：加 redirect 和 shortcut

```cisco
! ── Hub ──
Hub(config)# interface Tunnel0
Hub(config-if)# ★ ip nhrp redirect ★
Hub(config-if)# ★ ip summary-address eigrp 100 192.168.0.0 255.255.0.0 ★   ! ★ 现在可以汇总了

! ── 所有 Spoke ──
Spoke1(config)# interface Tunnel0
Spoke1(config-if)# ★ ip nhrp shortcut ★
Spoke2(config)# interface Tunnel0
Spoke2(config-if)# ip nhrp shortcut
```

### Step 2：验证路由表变小

```cisco
Spoke1# show ip route eigrp
D    192.168.0.0/16 [90/26882560] via 10.0.0.1, 00:01:15, Tunnel0
! ★ 只有一条汇总路由，明细全没了 ✓ ★
```

### Step 3：★★ 验证 Spoke-to-Spoke 依然能直连

```cisco
! 清空缓存
Spoke1# clear dmvpn session
Spoke1# clear ip nhrp

! 第一次 traceroute（还在走 Hub）
Spoke1# traceroute 192.168.12.1 source 192.168.11.1
  1 10.0.0.1      20 msec       ← 先走 Hub
  2 10.0.0.12     35 msec
  3 192.168.12.1  40 msec

! ★ 几秒后再次 traceroute ★
Spoke1# traceroute 192.168.12.1 source 192.168.11.1
  1 ★ 10.0.0.12 ★  15 msec       ← ★★ 直连了！★★
  2 192.168.12.1  18 msec
```

### Step 4：★ 查看 NHRP 快捷路由

```cisco
Spoke1# ★ show ip route | include % ★
★ %    192.168.12.0/24 [250/255] via 10.0.0.12, 00:00:35, Tunnel0 ★
  ↑                                    ↑
  NHRP 动态安装的快捷路由        下一跳是 Spoke2
```

**★ `%` 标记表示这是 NHRP 安装的快捷路由**（不是路由协议学到的）。

```cisco
Spoke1# show ip nhrp
192.168.12.0/24 via 10.0.0.12
   Tunnel0 created 00:00:35, expire 00:04:25
   Type: ★ dynamic ★, Flags: router ★ rib nho ★
   NBMA address: 2.2.2.22

10.0.0.12/32 via 10.0.0.12
   Tunnel0 created 00:00:35, expire 00:04:25
   Type: dynamic, Flags: router used nhop
   NBMA address: 2.2.2.22
```

```cisco
Spoke1# show dmvpn
 # Ent  Peer NBMA Addr  Peer Tunnel Add  State  UpDn Tm  Attrb
     1     3.3.3.3         10.0.0.1        UP   00:30:15   S
     1  ★2.2.2.22★      ★10.0.0.12★      UP   00:00:35  ★DT1★
                                                           ↑↑↑
                                          D=动态, T1=路由已安装
```

**★★ Phase 3 同时获得了两个好处**：
- ✅ **路由表小**（Hub 可以汇总，只有一条 /16）
- ✅ **Spoke-to-Spoke 直连**（NHRP Redirect + Shortcut）

### Step 5：观察隧道自动拆除

```cisco
! 停止流量，等 NHRP holdtime（默认 600 秒，可以先调小便于观察）
Spoke1(config)# interface Tunnel0
Spoke1(config-if)# ip nhrp holdtime 60

! 等 60 秒后
Spoke1# show dmvpn
 # Ent  Peer NBMA Addr  Peer Tunnel Add  State  UpDn Tm  Attrb
     1     3.3.3.3         10.0.0.1        UP   00:35:15   S
! ★ 到 Spoke2 的动态隧道消失了 ✓ ★

Spoke1# show ip route | include %
! ★ 快捷路由也消失了 ★
```

**★ 这就是 DMVPN 的精妙之处：按需建立，用完拆除。**

---

## Part 4：加 IPsec

### Step 1：配置（所有设备相同）

```cisco
! ── IKEv2 ──
crypto ikev2 proposal PROP-1
 encryption aes-cbc-256
 integrity sha256
 group 14

crypto ikev2 policy POL-1
 proposal PROP-1

crypto ikev2 keyring KR-1
 peer ANY
  ★ address 0.0.0.0 0.0.0.0 ★                    ! ★ 接受任意对端（Spoke 可能是动态 IP）
  pre-shared-key DmvpnSharedKey

crypto ikev2 profile PROF-1
 ★ match identity remote address 0.0.0.0 ★
 authentication local pre-share
 authentication remote pre-share
 keyring local KR-1

! ── IPsec ──
crypto ipsec transform-set TS-1 esp-aes 256 esp-sha256-hmac
 ★ mode transport ★

crypto ipsec profile IPSEC-PROF
 set transform-set TS-1
 set ikev2-profile PROF-1

! ── 应用（★ 注意 shared 关键字）──
interface Tunnel0
 ★ tunnel protection ipsec profile IPSEC-PROF shared ★
```

**★ `shared` 关键字的重要性**：一个 mGRE 接口要和多个对端建立 IPsec SA。`shared` 告诉 IOS "这个 profile 会被多个 SA 共享"。**不加的话某些 IOS 版本会出现 SA 冲突。**

### Step 2：验证

```cisco
Hub# show crypto session
Interface: Tunnel0
Session status: ★ UP-ACTIVE ★
Peer: 1.1.1.11 port 500
  IKEv2 SA: local 3.3.3.3/500 remote 1.1.1.11/500 Active
  IPSEC FLOW: permit 47 host 3.3.3.3 host 1.1.1.11
        Active SAs: 2

Hub# ★ show dmvpn detail ★
 # Ent  Peer NBMA Addr  Peer Tunnel Add  State  UpDn Tm  Attrb
     1     1.1.1.11        10.0.0.11        UP   00:15:23   ★D★
! ★ Attrb 里没有 X（No Socket）= IPsec 正常 ✓ ★
```

### Step 3：★ 理解 `X` 标志

**故意制造 IPsec 失败**：
```cisco
Spoke1(config)# crypto ikev2 keyring KR-1
Spoke1(config-ikev2-keyring)# peer ANY
Spoke1(config-ikev2-keyring-peer)# ★ pre-shared-key WrongKey ★
Spoke1# clear crypto ikev2 sa
```

```cisco
Hub# show dmvpn
 # Ent  Peer NBMA Addr  Peer Tunnel Add  State  UpDn Tm  Attrb
     1     1.1.1.11        10.0.0.11        UP   00:15:23  ★DX★
                                                             ↑
                                        ★ X = No Socket（IPsec 未建立）★
```

**含义**：NHRP 注册成功了（Hub 知道 Spoke 的公网 IP），**但 IPsec 会话没建起来 → 流量实际不通**。

**排查**：
```cisco
Hub# show crypto ikev2 sa
! 空或状态异常

Hub# debug crypto ikev2
★ IKEv2:(SESSION ID = 1,SA ID = 1):Verification of peer's authentication data FAILED ★
```

**★ `show dmvpn` 标志速查**：

| 标志 | 含义 | 正常？ |
|:--|:--|:--|
| **S** | Static（手工映射） | ✅ |
| **D** | Dynamic（NHRP 动态学到） | ✅ |
| **I** | Incomplete（解析未完成） | ⚠️ 短暂 |
| **N** | **NATed**（对端在 NAT 后） | ✅ |
| **X** | **No Socket（IPsec 未建立）** | ❌ **有问题** |
| **T1** | Route Installed | ✅ |
| **T2** | Nexthop-override | ✅ |

---

## Part 5：故障注入

### 故障 A：NHRP network-id 不一致

```cisco
Spoke1(config)# interface Tunnel0
Spoke1(config-if)# ★ ip nhrp network-id 200 ★           ! Hub 是 100
```

<details><summary>症状与排查</summary>

```cisco
Hub# show dmvpn
! ★ Spoke1 消失了 ★

Spoke1# debug nhrp
NHRP: Receive Registration Reply via Tunnel0
★ NHRP: netid_in = 200, to_us = 0 ★                     ← network-id 不匹配

Spoke1# show ip nhrp nhs detail
Tunnel0:
  10.0.0.1  ★ E ★  priority = 0  req-sent 12  ★ req-failed 12 ★  repl-recv 0
            ↑                                    ↑
      E = Expecting（一直在等）           全部失败
```

**修复**：`ip nhrp network-id 100`
</details>

### 故障 B：忘配 `ip nhrp map multicast`

```cisco
Spoke1(config-if)# ★ no ip nhrp map multicast 3.3.3.3 ★
```

<details><summary>症状与排查</summary>

**症状**：
- ✅ 隧道 UP
- ✅ NHRP 注册成功
- ❌ **EIGRP 邻居建不起来**

```cisco
Spoke1# show dmvpn
 # Ent  Peer NBMA Addr  Peer Tunnel Add  State  UpDn Tm  Attrb
     1     3.3.3.3         10.0.0.1       ★UP★  00:05:23   S
! 隧道正常

Spoke1# show ip eigrp neighbors
! ★ 空！★

Spoke1# ★ show ip nhrp multicast ★
! ★ 空 —— 找到原因 ★
```

**为什么**：EIGRP 的 Hello 是**组播 224.0.0.10**。mGRE 接口不知道该把组播发给谁，**必须用 `ip nhrp map multicast` 明确指定**。

**修复**：
```cisco
Spoke1(config-if)# ip nhrp map multicast 3.3.3.3
```

**Hub 侧用 `dynamic`**：
```cisco
Hub(config-if)# ★ ip nhrp map multicast dynamic ★
! 自动把组播发给所有已注册的 Spoke
```
</details>

### 故障 C：忘了 `no ip split-horizon eigrp`

```cisco
Hub(config-if)# ★ ip split-horizon eigrp 100 ★
```

<details><summary>症状与排查</summary>

```cisco
Spoke1# show ip route eigrp
D    192.168.0.0/24 [90/26882560] via 10.0.0.1, Tunnel0
! ★ 只学到 Hub 的网段，学不到其他 Spoke 的 ★

Hub# show run interface Tunnel0 | include split
 ip split-horizon eigrp 100                    ← 找到了（应该是 no）
```

**修复**：`no ip split-horizon eigrp 100`

**★ 这是 DMVPN + EIGRP 的必配项**（所有 Phase 都需要）。
</details>

### 故障 D：MTU

```cisco
Spoke1(config-if)# ★ no ip mtu 1400 ★
Spoke1(config-if)# ★ no ip tcp adjust-mss 1360 ★
```

<details><summary>症状与排查</summary>

```cisco
Spoke1# ping 192.168.12.1 source 192.168.11.1 size 100
!!!!!                                              ← 小包通

Spoke1# ping 192.168.12.1 source 192.168.11.1 ★ size 1500 df-bit ★
★ M.M.M ★                                          ← 大包不通
```

**从 PC 访问对端 Web 服务**：页面卡住。

**开销计算**：
```
   GRE:        24 字节
   IPsec ESP:  约 58 字节（传输模式）
   ─────────────────────
   合计:       约 82 字节
   
   1500 - 82 = 1418  → ★ 保守设 ip mtu 1400 ★
   1400 - 40 = 1360  → ★ ip tcp adjust-mss 1360 ★
```

**修复**：加回两条命令。
</details>

### 故障 E：Spoke 在 NAT 后面

```cisco
! 在 Spoke1 前面加一台 NAT 设备
NAT-DEV(config)# interface GigabitEthernet0/0
NAT-DEV(config-if)# ip nat inside
NAT-DEV(config)# interface GigabitEthernet0/1
NAT-DEV(config-if)# ip nat outside
NAT-DEV(config)# ip nat inside source list 1 interface Gi0/1 overload
NAT-DEV(config)# access-list 1 permit 1.1.1.0 0.0.0.255
```

<details><summary>验证 NAT 穿越</summary>

```cisco
Hub# ★ show dmvpn detail ★
 # Ent  Peer NBMA Addr  Peer Tunnel Add  State  UpDn Tm  Attrb
     1  ★203.5.5.5★      10.0.0.11        UP   00:05:23  ★DN★
        ↑ NAT 后的公网 IP                                  ↑
                                              ★ N = NATed（检测到 NAT）★

Hub# show crypto ikev2 sa detail
 Tunnel-id Local            Remote           fvrf/ivrf  Status
 1         3.3.3.3/★4500★   203.5.5.5/★4500★ none/none  READY
                     ↑                 ↑
              ★ 端口是 4500 = NAT-T 生效 ✓ ★
```

**★ DMVPN 天然支持 NAT 后的 Spoke**，因为：
1. Spoke **主动向 Hub 注册**（不需要 Hub 主动连接 Spoke）
2. NHRP 记录 NAT 后的公网地址
3. IPsec NAT-T 让 ESP 穿越 NAT

**必须满足的条件**：
- ★ **Hub 必须有固定的公网 IP，且不在 NAT 后** ★
- NAT 设备放行 UDP 500、UDP 4500
- NAT 会话超时不能太短（否则映射老化）

**★ Spoke-to-Spoke 的 NAT 限制**：

| 场景 | 直连？ |
|:--|:--|
| 两个 Spoke 都有公网 IP | ✅ |
| 一个 NAT 后，一个公网 | ✅ 通常可以 |
| **两个都在 NAT 后（对称 NAT）** | ❌ **通常不行** |

**为什么**：Spoke1 主动发包给 NAT-B，但 **NAT-B 上没有对应的映射**（Spoke2 没有主动发过包给 Spoke1）→ 被丢弃。

**这就是 NAT 穿越的经典难题**，DMVPN 没有"打洞"能力。后果是这两个 Spoke 之间的流量**回退到走 Hub**。

**缓解措施**：
```cisco
! 缩短 NHRP holdtime，保持 NAT 映射活跃
Spoke1(config-if)# ip nhrp holdtime 300
Spoke1(config-if)# ip nhrp registration timeout 60

! IPsec DPD 保活
Spoke1(config)# crypto ikev2 dpd 30 5 periodic
```
</details>

---

## Part 6：双 Hub 冗余（进阶）

```cisco
! ── Spoke 配置两个 NHS ──
Spoke1(config)# interface Tunnel0
Spoke1(config-if)# ip nhrp nhs 10.0.0.1 nbma 3.3.3.3 multicast
Spoke1(config-if)# ip nhrp nhs 10.0.0.2 nbma 4.4.4.4 multicast
Spoke1(config-if)# ★ ip nhrp nhs cluster 0 max-connections 2 ★
```

```cisco
Spoke1# show ip nhrp nhs detail
Tunnel0:
  10.0.0.1  RE  priority = 0 cluster = 0
  10.0.0.2  RE  priority = 0 cluster = 0
! ★ 两个 Hub 都在响应 ✓ ★
```

**主备模式（优先用 Hub1）**：
```cisco
Spoke1(config-if)# ip nhrp nhs 10.0.0.1 nbma 3.3.3.3 multicast ★ priority 1 ★
Spoke1(config-if)# ip nhrp nhs 10.0.0.2 nbma 4.4.4.4 multicast ★ priority 2 ★
Spoke1(config-if)# ip nhrp nhs cluster 0 max-connections 1
```

---

## 实验检查清单

```
□ ① Phase 1 建立，NHRP 注册成功
□ ② ★ 验证"关水平分割"的必要性
□ ③ Phase 2：Spoke 改 mGRE + no ip next-hop-self
□ ④ ★★ 验证 Spoke-to-Spoke 直连（traceroute 只有 2 跳）
□ ⑤ ★★ 验证"Phase 2 汇总后 Spoke-to-Spoke 失效"
□ ⑥ Phase 3：加 redirect + shortcut
□ ⑦ ★★ 验证 Phase 3 同时实现"汇总 + 直连"
□ ⑧ ★ 看到 NHRP 快捷路由（% 标记）
□ ⑨ 观察动态隧道的自动拆除
□ ⑩ 加 IPsec，验证 X 标志的含义
□ ⑪ 五个故障都亲手制造并修复
□ ⑫ 写了实验笔记
```

## 关键命令总结

| 命令 | 用途 |
|:--|:--|
| **`show dmvpn`** | ★★ 第一命令：State + Attrb 标志 |
| `show dmvpn detail` | 含 NAT 和 IPsec 详情 |
| `show ip nhrp` | NHRP 映射表 |
| **`show ip nhrp nhs detail`** | ★ Spoke 上看 NHS 状态（R = Responding） |
| `show ip nhrp multicast` | ★ 组播映射（EIGRP 邻居建不起来时查） |
| **`show ip route \| include %`** | ★ NHRP 快捷路由（Phase 3） |
| `show crypto session` | IPsec 会话 |
| `clear dmvpn session` | 清除动态隧道（测试用） |
| `debug nhrp` | NHRP 协商过程 |

## 三个 Phase 对比（必背）

| | **Phase 1** | **Phase 2** | **Phase 3** |
|:--|:--|:--|:--|
| Spoke 隧道 | **P2P GRE** | **mGRE** | **mGRE** |
| Spoke-to-Spoke | ❌ | ✅ | ✅ |
| **Hub 能汇总** | ✅ | ❌ | ✅ |
| Spoke 路由表 | 小 | **大** | **小** |
| 关键机制 | — | 保留原始下一跳 | **NHRP Redirect + Shortcut** |
| 推荐 | 简单场景 | 已过时 | ★ **首选** |

## EIGRP over DMVPN 必配项

```
□ Hub: ★ no ip split-horizon eigrp <AS> ★     （所有 Phase）
□ Hub: no ip next-hop-self eigrp <AS>          （仅 Phase 2）
□ Hub: ip nhrp map multicast dynamic
□ Spoke: ip nhrp map multicast <Hub公网IP>
□ 所有: ip mtu 1400 + ip tcp adjust-mss 1360
□ 所有: tunnel protection ipsec profile X ★ shared ★
```

---

**上一个** ← [Lab 08](lab-08-GRE-over-IPsec.md) ｜ **下一个** → [Lab 10: 综合排障](lab-10-综合排障.md)
