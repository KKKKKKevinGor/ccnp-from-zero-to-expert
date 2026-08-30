# Lab 10 · 综合排障（毕业实验）

**对应章节**：全部　**难度**：★★★★★　**时长**：4 小时

> **这个实验的玩法不一样**：
> 不给你配置步骤，只给你**故障现象**。你要自己定位根因并修复。
>
> **这就是 ENARSI 考试和真实工作的形态。**

---

## 玩法说明

### 单人模式
1. 先按 Part 1 搭好完整拓扑（这是"正常状态"）
2. **导出配置备份**（`copy running-config flash:good.cfg`）
3. 从 Part 3 的故障库里**随机挑一个**，照着"故障注入脚本"敲进去
4. **不看答案**，自己排查
5. 修复后对比 `good.cfg` 验证

### 双人模式（★ 更接近真实）
1. 甲搭好拓扑，乙不看
2. **甲从故障库注入 1-3 个故障**（不告诉乙是什么）
3. 乙只知道用户报障描述（比如"财务部访问 ERP 慢"）
4. 乙排查，甲计时
5. 交换角色

### 计时标准

| 时间 | 评级 |
|:--|:--|
| < 10 分钟 | ★★★★★ 专家 |
| 10-20 分钟 | ★★★★ 熟练 |
| 20-40 分钟 | ★★★ 合格 |
| > 40 分钟 | ★★ 需要加强分层排障的习惯 |

---

## Part 1：搭建完整拓扑

```
                          [ ISP-1 ]  AS 100        [ ISP-2 ]  AS 200
                          202.1.1.254                203.2.2.254
                               │                          │
                          202.1.1.1                  203.2.2.2
                     ┌─────────▼────┐            ┌────────▼─────┐
                     │      R1      │═══iBGP═════│      R2      │
                     │  AS 65001    │            │   AS 65001   │
                     │  Lo0:1.1.1.1 │            │  Lo0:2.2.2.2 │
                     └──────┬───────┘            └───────┬──────┘
                            │      OSPF Area 0           │
                            └────────────┬───────────────┘
                                    ┌────▼─────┐
                                    │    R3    │  核心
                                    │Lo0:3.3.3.3
                                    └────┬─────┘
                                         │
                            ┌────────────┴────────────┐
                       ┌────▼────┐              ┌─────▼───┐
                       │  SW1    │══ Po1 (LACP) │  SW2    │
                       │ (L3/HSRP)              │(L3/HSRP)│
                       └────┬────┘              └─────┬───┘
                            └───────────┬─────────────┘
                                   ┌────▼────┐
                                   │  SW3    │ (L2 接入)
                                   └────┬────┘
                                        │
                                 [PC1]      [PC2]
                                VLAN10     VLAN20
                            192.168.10.10  192.168.20.10
```

### 涉及的技术栈

| 层 | 技术 |
|:--|:--|
| L2 | VLAN、Trunk、MST、LACP、PortFast/BPDU Guard |
| L3 网关 | SVI、HSRP + IP SLA Track |
| IGP | OSPF 多区域 |
| EGP | BGP 双出口、LOCAL_PREF、AS-Path Prepend |
| 服务 | DHCP 中继、NAT、NTP、Syslog |
| 安全 | ACL、DHCP Snooping、Port Security |

### 基础配置要点（自己补全）

```cisco
! ── SW3（接入）──
vlan 10,20,999
interface range GigabitEthernet1/0/1-2
 switchport mode access
 switchport access vlan 10          ! Gi1/0/2 用 vlan 20
 switchport nonegotiate
 spanning-tree portfast
 spanning-tree bpduguard enable
interface range GigabitEthernet1/0/23-24
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20

! ── SW1/SW2（三层 + HSRP）──
ip routing
vlan 10,20,999
spanning-tree mode mst
spanning-tree mst configuration
 name CAMPUS
 revision 1
 instance 1 vlan 10
 instance 2 vlan 20
!
interface Vlan10
 ip address 192.168.10.2 255.255.255.0      ! SW2 用 .3
 ip helper-address 10.1.30.53
 standby version 2
 standby 10 ip 192.168.10.1
 standby 10 priority 110                     ! SW2 用 100
 standby 10 preempt
 standby 10 track 1 decrement 20
!
track 1 interface GigabitEthernet1/0/48 line-protocol
 delay down 10 up 30

! ── R1/R2（BGP 出口）──
router bgp 65001
 bgp router-id 1.1.1.1
 neighbor 202.1.1.254 remote-as 100
 neighbor 202.1.1.254 soft-reconfiguration inbound
 neighbor 2.2.2.2 remote-as 65001
 neighbor 2.2.2.2 update-source Loopback0
 neighbor 2.2.2.2 next-hop-self
 network 192.168.0.0 mask 255.255.240.0
!
ip route 192.168.0.0 255.255.240.0 Null0
!
ip nat inside source list NAT-LIST interface GigabitEthernet0/1 overload
ip access-list extended NAT-LIST
 permit ip 192.168.0.0 0.0.15.255 any

! ── R3（核心 OSPF）──
router ospf 1
 router-id 3.3.3.3
 auto-cost reference-bandwidth 100000
 network 10.0.0.0 0.0.255.255 area 0
 network 192.168.0.0 0.0.15.255 area 0
```

### ★ 保存"正常状态"

```cisco
! 每台设备都执行
Rx# copy running-config flash:good.cfg
Rx# write memory
```

**恢复方式**：
```cisco
Rx# configure replace flash:good.cfg
```

### 验证正常状态（基线）

```
□ PC1 能 ping 通网关 192.168.10.1
□ PC1 能 ping 通 PC2（跨 VLAN）
□ PC1 能 ping 通 8.8.8.8（出网）
□ PC1 能通过 DHCP 拿到 IP
□ show standby brief 显示 SW1 是 VLAN10 的 Active
□ show ip bgp summary 显示两个邻居都 Established
□ show ip ospf neighbor 所有邻居 FULL
□ show etherchannel summary 显示 Po1(SU) 两个成员 (P)
□ show spanning-tree mst 显示负载分担正常
□ traceroute 8.8.8.8 走电信（R1）
```

**★ 把这些命令的正常输出截图/保存下来**——排障时对比用。

---

## Part 2：排障方法论回顾

### 万能框架

```
   ① ★ 精确描述现象 ★
      "A 能 ping B，B 不能 ping A" ≠ "网络不通"
      精确的描述本身就包含 50% 的答案
        ↓
   ② ★ 确定影响范围 ★
      一台？一个 VLAN？一栋楼？全网？
      → 范围决定问题在哪一层
        ↓
   ③ ★ 查变更 ★
      show archive log config all
      show logging | include CONFIG_I
      80% 的故障是变更引起的
        ↓
   ④ ★ 分层定位 ★（每层一个明确的命令）
      L1 → show interfaces status
      L2 → show mac address-table / show interfaces trunk / show spanning-tree
      L3 → show ip route / ping / traceroute
      L4 → telnet <ip> <port>
      L7 → 应用日志
        ↓
   ⑤ ★ 二分法 ★
      中间点测试，判断问题在前半段还是后半段
        ↓
   ⑥ ★ 对比法 ★
      能通的 vs 不能通的，配置差在哪？
      show run interface X | show run interface Y
        ↓
   ⑦ ★★ 验证假设 ★★
      不要凭猜测就改配置
      先用命令证明你的假设成立
```

### 快速判断表

| 现象 | 最可能的层 | 第一个命令 |
|:--|:--|:--|
| 接口 down/down | L1 | `show interfaces status` |
| 接口 up/down | L2 | `show interfaces` 看封装 |
| 同 VLAN 不通 | L2 | `show mac address-table` |
| 同 VLAN 跨交换机不通 | L2 | **`show interfaces trunk`** |
| 跨 VLAN 全不通，SVI 正常 | L3 | **`show run \| inc ^ip routing`** |
| 跨网段不通 | L3 | `show ip route` |
| **A→B 通，B→A 不通** | **L3 回程** | 在 B 上 `show ip route <A>` |
| ping 通但业务不通 | L4 | `telnet <ip> <port>` |
| **小包通大包不通** | **MTU** | `ping size 1500 df-bit` |
| **拿到 169.254.x.x** | **DHCP** | `show ip int Vlan10 \| inc Helper` |
| **证书报"尚未生效"** | **NTP** | `show ntp status` |
| **固定延迟 5 秒** | **DNS AAAA** | `dig AAAA <域名>` |
| traceroute 看到 IP 往复 | **路由环路** | 查重分发 |

---

## Part 3：故障库

**★ 使用方法**：随机挑一个，把"注入脚本"敲进去，然后自己排查。

---

### 故障 01 · 用户报："我的电脑连不上网"

**难度**：★

<details><summary>注入脚本（排障者不要看）</summary>

```cisco
SW3(config)# interface GigabitEthernet1/0/1
SW3(config-if)# switchport access vlan 30
! VLAN 30 不存在
```
</details>

<details><summary>参考排查路径</summary>

```cisco
! ① PC 的 IP 是什么？
PC1> ipconfig
IPv4 Address. . . . : ★ 169.254.15.230 ★
! ★ APIPA → DHCP 完全失败 ★

! ② 端口状态
SW3# show interfaces GigabitEthernet1/0/1 status
Port    Name   Status         Vlan    Duplex  Speed Type
Gi1/0/1        ★ inactive ★   ★ 30 ★   a-full a-1000 10/100/1000BaseTX
                 ↑             ↑
        ★ inactive = VLAN 不存在 ★

! ③ 确认
SW3# show vlan brief | include ^30
! ★ 空 —— VLAN 30 没创建 ★

SW3# show vlan brief
VLAN Name              Status    Ports
10   VLAN0010          active    ...
20   VLAN0020          active    ...
999  NATIVE-UNUSED     active
! ★ 没有 VLAN 30 ★
```

**根因**：端口被划到了不存在的 VLAN 30。

**修复**：
```cisco
SW3(config)# interface GigabitEthernet1/0/1
SW3(config-if)# switchport access vlan 10
```

**★ 学到的**：`show interfaces status` 里的 **`inactive`** 状态直接指向"VLAN 不存在"。
</details>

---

### 故障 02 · 用户报："财务部（VLAN10）的人上不了网，销售部（VLAN20）正常"

**难度**：★★

<details><summary>注入脚本</summary>

```cisco
SW1(config)# interface Port-channel1
SW1(config-if)# switchport trunk allowed vlan 20,999
! 移除了 VLAN 10
```
</details>

<details><summary>参考排查路径</summary>

```cisco
! ① 范围：只影响 VLAN 10 → 二层问题
! ② PC1 能 ping 通网关吗？
PC1> ping 192.168.10.1
Request timed out.
! ★ 连网关都不通 → L2 问题 ★

! ③ 同 VLAN 内呢？
PC1> ping <同VLAN的另一台PC>
Reply from ...
! ★ 同 VLAN 通，说明本地交换机没问题 ★
! ★ 问题在上联 ★

! ④ ★ 查 Trunk（四段输出法）★
SW1# show interfaces trunk

Port        Mode  Encapsulation  Status    Native vlan
Po1         on    802.1q         trunking  999

Port        ★ Vlans allowed on trunk ★
Po1         ★ 20,999 ★                    ← ★ 第2段：VLAN 10 不见了！★

Port        Vlans allowed and active in management domain
Po1         20

Port        Vlans in spanning tree forwarding state and not pruned
Po1         20
```

**★ 四段输出法的用法**：从第 4 段往上看
- 第 4 段没有 VLAN 10 → 往上
- 第 3 段也没有 → 往上
- **第 2 段也没有 → 配置里就没放行**

（如果第 2 段有但第 3 段没有 → VLAN 没创建；第 3 段有但第 4 段没有 → 被 STP 阻塞）

**修复**：
```cisco
SW1(config)# interface Port-channel1
SW1(config-if)# ★ switchport trunk allowed vlan add 10 ★
!                                             ↑↑↑ 用 add，不是覆盖！
```

**★ 学到的**：`show interfaces trunk` 的四段输出法能在 30 秒内定位 90% 的 Trunk 问题。
</details>

---

### 故障 03 · 用户报："所有 VLAN 内部能通，但跨 VLAN 完全不通"

**难度**：★★

<details><summary>注入脚本</summary>

```cisco
SW1(config)# no ip routing
SW2(config)# no ip routing
```
</details>

<details><summary>参考排查路径</summary>

```cisco
! ① 现象确认
PC1(VLAN10)> ping 192.168.10.1        ← 网关
Reply from 192.168.10.1               ✓ 通
PC1> ping 192.168.20.10               ← 跨 VLAN
Request timed out.                    ✗ 不通

! ② SVI 状态
SW1# show ip interface brief | include Vlan
Vlan10   192.168.10.2   YES manual  ★ up      up ★
Vlan20   192.168.20.2   YES manual  up        up
! ★ 都正常！★

! ③ 路由表
SW1# show ip route connected
C    192.168.10.0/24 is directly connected, Vlan10
C    192.168.20.0/24 is directly connected, Vlan20
! ★ 也正常！★

! ★★ 所有表面证据都正常，但就是不转发 ★★

! ④ ★ 唯一的线索 ★
SW1# ★ show running-config | include ^ip routing ★
! ★ 空输出 = 没开启 ★
```

**根因**：三层交换机没开启 `ip routing`。

**修复**：
```cisco
SW1(config)# ip routing
SW2(config)# ip routing
```

**★ 学到的**：
- **症状特征**：同 VLAN 通、跨 VLAN 全不通、**SVI 和路由表都正常**
- **这是最迷惑的故障之一**，因为所有常规检查都显示正常
- **厂商差异**：H3C/华为的三层交换机默认开启路由转发，没有这个开关。从国产设备转 Cisco 的人特别容易踩
</details>

---

### 故障 04 · 用户报："能上网，但访问总部 ERP 系统的大页面会卡住"

**难度**：★★★

<details><summary>注入脚本</summary>

```cisco
R1(config)# interface Tunnel0
R1(config-if)# no ip mtu 1400
R1(config-if)# no ip tcp adjust-mss 1360
! 或者在中间某台设备上
R3(config)# interface GigabitEthernet0/1
R3(config-if)# ip mtu 1300
```
</details>

<details><summary>参考排查路径</summary>

```cisco
! ① ★ 现象的关键：小的能通，大的不能 ★
PC1> ping 10.1.30.50
Reply from 10.1.30.50: bytes=32 time=15ms      ✓ 通
PC1> curl http://10.1.30.50/small.html
<html>...                                       ✓ 小页面正常
PC1> curl http://10.1.30.50/large.html
（卡住）                                        ✗ 大页面卡死

! ② ★ MTU 测试（第一反应就该是这个）★
R1# ping 10.1.30.50 size 100
!!!!!                                           ✓

R1# ★ ping 10.1.30.50 size 1500 df-bit ★
★ M.M.M ★                                      ✗ 大包不通
! ★ M = 需要分片但 DF 位置位 → MTU 问题确认 ★

! ③ 二分法找实际 MTU
R1# ping 10.1.30.50 size 1400 df-bit
!!!!!                                           ✓
R1# ping 10.1.30.50 size 1450 df-bit
M.M.M                                           ✗
R1# ping 10.1.30.50 size 1420 df-bit
!!!!!                                           ✓
! ★ 实际可用 MTU 约 1420 ★

! ④ 逐跳查 MTU
R1# show ip interface Tunnel0 | include MTU
R3# show ip interface GigabitEthernet0/1 | include MTU
  MTU is ★ 1300 ★ bytes                         ← 找到了

! ⑤ 检查是否有 ICMP 被拦（PMTUD 黑洞的成因）
R3# show access-lists | include icmp
```

**根因**：路径上某处 MTU 不足，且 ICMP unreachable 可能被拦，导致 PMTUD 失效。

**修复**：
```cisco
! 方案 1：修复 MTU
R3(config-if)# ip mtu 1500

! 方案 2（隧道场景）：配 MSS 调整
R1(config)# interface Tunnel0
R1(config-if)# ★ ip mtu 1400 ★
R1(config-if)# ★ ip tcp adjust-mss 1360 ★

! 方案 3：确保 ICMP unreachable 放行
R3(config)# ip access-list extended XXX
R3(config-ext-nacl)#  permit icmp any any ★ unreachable ★
```

**★ 学到的**：
- **"小包通大包不通" = MTU 问题**，这是最典型的特征
- `ip mtu` 管所有流量（被动），`ip tcp adjust-mss` 主动改写 TCP MSS
- **不要在 ACL 里无脑拦所有 ICMP**，`unreachable` 是 PMTUD 必需的
</details>

---

### 故障 05 · 用户报："上网时快时慢，有时候完全断几秒"

**难度**：★★★★

<details><summary>注入脚本</summary>

```cisco
! 在 SW3 上关掉 BPDU Guard，然后接一台交换机形成环路
SW3(config)# interface GigabitEthernet1/0/10
SW3(config-if)# no spanning-tree bpduguard enable
SW3(config-if)# no spanning-tree portfast
! 然后把 Gi1/0/10 和 Gi1/0/11 用网线直连（自环）
! 或者接一台关了 STP 的交换机
```
</details>

<details><summary>参考排查路径</summary>

```cisco
! ① CPU 和接口状态
SW3# show processes cpu sorted | exclude 0.00
CPU utilization for five seconds: ★ 95% ★/78%
 PID Runtime(ms) Invoked  uSecs  5Sec  1Min  5Min TTY Process
  85   12345678  9876543   1250 ★45.2%★ 43.1% 41.5%  0 ★ Spanning Tree ★
                                                          ↑ STP 进程占用高

! ② 接口流量
SW3# show interfaces | include rate|is up
GigabitEthernet1/0/10 is up, line protocol is up
  5 minute input rate ★ 890000000 ★ bits/sec
  5 minute output rate ★ 890000000 ★ bits/sec
! ★ 接近千兆满载！★

! ③ ★ MAC 表震荡（环路的确定证据）★
SW3# show mac address-table address 0050.5600.aabb
          Mac Address Table
Vlan    Mac Address       Type        Ports
  10    0050.5600.aabb    DYNAMIC     ★ Gi1/0/1 ★
! 立刻再执行一次
SW3# show mac address-table address 0050.5600.aabb
  10    0050.5600.aabb    DYNAMIC     ★ Gi1/0/10 ★
! ★★ 同一个 MAC 在两个端口间跳变 = 环路 ★★

! ④ 拓扑变化次数
SW3# show spanning-tree vlan 10 detail | include occurr|from
  Number of topology changes ★ 12453 ★ last change occurred 00:00:03 ago
          ★ from GigabitEthernet1/0/10 ★
                                    ↑ ★ 找到源头端口 ★

! ⑤ 看那个端口对面是什么
SW3# show cdp neighbors GigabitEthernet1/0/10
! 如果显示是一台交换机 → 私接设备

! ⑥ 检查保护为什么没生效
SW3# show run interface GigabitEthernet1/0/10
interface GigabitEthernet1/0/10
 switchport mode access
 ! ★ 没有 spanning-tree bpduguard enable ★
```

**根因**：某个端口私接了交换机形成环路，且该端口没配 BPDU Guard。

**应急处理（★ 先止血）**：
```cisco
SW3(config)# interface GigabitEthernet1/0/10
SW3(config-if)# ★ shutdown ★
```

**长期加固**：
```cisco
SW3(config)# ★ spanning-tree portfast default ★
SW3(config)# ★ spanning-tree portfast bpduguard default ★
SW3(config)# ★ spanning-tree loopguard default ★
SW3(config)# errdisable recovery cause bpduguard
SW3(config)# errdisable recovery interval 300

SW3(config)# interface range GigabitEthernet1/0/1-20
SW3(config-if-range)# ★ storm-control broadcast level 5.00 ★
SW3(config-if-range)# storm-control action trap
```

**★ 学到的**：
- **环路的三个证据**：CPU 高（STP 进程）、流量暴涨、**MAC 表震荡**
- **`show spanning-tree detail | include occurr|from`** 能直接找到震荡源头
- **BPDU Guard 是防私接交换机最有效的手段**，所有接入端口必配
</details>

---

### 故障 06 · 用户报："出口切换到备用线路了，但没人操作过"

**难度**：★★★

<details><summary>注入脚本</summary>

```cisco
R1(config)# route-map TELECOM-IN permit 10
R1(config-route-map)#  match ip address prefix-list NONEXISTENT
R1(config-route-map)#  set local-preference 200
! ★ 引用了不存在的 prefix-list，且没有兜底 permit ★
R1(config)# router bgp 65001
R1(config-router)# neighbor 202.1.1.254 route-map TELECOM-IN in
R1# clear ip bgp 202.1.1.254 soft in
```
</details>

<details><summary>参考排查路径</summary>

```cisco
! ① 确认流量走了哪条
R3# traceroute 8.8.8.8 source Loopback0
  1 10.0.23.2       ← ★ 走 R2（联通），不是 R1（电信）★
  2 203.2.2.254
  ...

! ② 查 BGP 邻居状态
R1# show ip bgp summary
Neighbor       V   AS  MsgRcvd MsgSent TblVer InQ OutQ Up/Down State/PfxRcd
202.1.1.254    4  100     2451    2438      8   0    0 02:15:33   ★ 0 ★
                                                                    ↑
                                              ★ 前缀数是 0！本来应该有 1245 条 ★
2.2.2.2        4 65001     125     128      8   0    0 00:15:12    1245

! ③ ★ 关键对比 ★
R1# show ip bgp neighbors 202.1.1.254 ★ received-routes ★ | count
Number of lines which match regexp = ★ 1248 ★      ← 原始收到 1245 条

R1# show ip bgp neighbors 202.1.1.254 ★ routes ★ | count
Number of lines which match regexp = ★ 0 ★         ← ★ 过滤后一条不剩 ★

! ★★ received 有但 routes 没有 = 被 in 方向的策略过滤了 ★★

! ④ 查策略
R1# show run | include neighbor 202.1.1.254
 neighbor 202.1.1.254 route-map ★ TELECOM-IN ★ in

R1# show route-map TELECOM-IN
route-map TELECOM-IN, permit, sequence 10
  Match clauses:
    ★ ip address prefix-lists: NONEXISTENT ★
  Set clauses:
    local-preference 200
  Policy routing matches: 0 packets, 0 bytes
! ★ 只有 seq 10，没有兜底！★

! ⑤ 确认 prefix-list 不存在
R1# show ip prefix-list NONEXISTENT
! ★ 空 —— 不存在的 prefix-list 匹配不到任何东西 ★
```

**根因**：route-map 引用了不存在的 prefix-list，且**没有兜底的 permit 语句**，导致**隐含 deny 拦掉了所有路由**。

**修复**：
```cisco
! 方案 1：加兜底
R1(config)# ★ route-map TELECOM-IN permit 20 ★

! 方案 2：修正 prefix-list
R1(config)# ip prefix-list ALL seq 5 permit 0.0.0.0/0 ★ le 32 ★
R1(config)# route-map TELECOM-IN permit 10
R1(config-route-map)#  match ip address prefix-list ALL
R1(config-route-map)#  set local-preference 200
R1(config)# route-map TELECOM-IN permit 20

R1# clear ip bgp 202.1.1.254 soft in
```

**验证**：
```cisco
R1# show ip bgp summary | include 202.1.1.254
202.1.1.254  4  100  2451  2438  8  0  0 02:15:33  ★ 1245 ★     ← 恢复 ✓
```

**★ 学到的**：
- **`received-routes` vs `routes` 的对比**是判断"是否被策略过滤"的黄金方法
- **route-map、prefix-list、ACL 末尾都有隐含 deny**
- **写 route-map 永远加兜底的 permit**
- **改完立即对比前缀数**
</details>

---

### 故障 07 · 用户报："主备切换后网络更慢了"

**难度**：★★★★

<details><summary>注入脚本</summary>

```cisco
! HSRP Active 和 STP 根桥不对齐
SW1(config)# spanning-tree mst 1 root primary
SW1(config)# spanning-tree mst 2 root primary          ! 两个实例都在 SW1
SW2(config)# spanning-tree mst 1 root secondary
SW2(config)# spanning-tree mst 2 root secondary
! 但 HSRP：VLAN10 Active 在 SW1，VLAN20 Active 在 SW2
```
</details>

<details><summary>参考排查路径</summary>

```cisco
! ① 现象：能通但慢，且横向链路流量高
SW1# show interfaces Port-channel1 | include rate
  5 minute input rate ★ 450000000 ★ bits/sec
  5 minute output rate ★ 450000000 ★ bits/sec
! ★ 两台核心之间的横向链路流量异常高 ★
! ★ 正常情况下这条链路只应该有少量控制流量 ★

! ② ★ 检查 HSRP 和 STP 是否对齐 ★
SW1# show standby brief
Interface   Grp  Pri P State    Active         Standby        Virtual IP
Vlan10      10   110 P ★Active★  local        192.168.10.3   192.168.10.1
Vlan20      20   100 P Standby  192.168.20.3   local          192.168.20.1

SW2# show standby brief
Vlan10      10   100 P Standby  192.168.10.2   local          192.168.10.1
Vlan20      20   110 P ★Active★  local        192.168.20.2   192.168.20.1

! ★ HSRP：VLAN10 → SW1，VLAN20 → SW2 ★

SW1# show spanning-tree mst 1 | include root
             ★ This bridge is the root ★           ← VLAN10 的根在 SW1 ✓ 对齐
SW1# show spanning-tree mst 2 | include root
             ★ This bridge is the root ★           ← ★ VLAN20 的根也在 SW1！★

! ★★ 不对齐：VLAN20 的 HSRP Active 在 SW2，但 STP 根桥在 SW1 ★★

! ③ 验证次优路径
SW3# show spanning-tree mst 2
Interface        Role Sts Cost      Prio.Nbr Type
Gi1/0/23         Root FWD 20000     128.23   P2p       ← 朝 SW1 转发
Gi1/0/24         Altn ★BLK★ 20000   128.24   P2p       ← 朝 SW2 阻塞
! ★ VLAN20 的流量只能先发给 SW1，再横向转给 SW2 ★
```

**根因**：HSRP Active 与 STP 根桥不对齐，导致 VLAN20 的流量走了次优路径（多一跳，占用横向链路）。

**修复**：
```cisco
SW1(config)# spanning-tree mst 1 root primary
SW1(config)# ★ spanning-tree mst 2 root secondary ★
SW2(config)# ★ spanning-tree mst 2 root primary ★
SW2(config)# spanning-tree mst 1 root secondary
```

**验证**：
```cisco
SW2# show spanning-tree mst 2 | include root
             ★ This bridge is the root ★           ← 现在对齐了 ✓

SW1# show interfaces Port-channel1 | include rate
  5 minute input rate ★ 12000000 ★ bits/sec       ← 横向流量大幅下降 ✓
```

**★ 学到的**：
- **HSRP Active 必须与 STP 根桥在同一台设备上**
- **症状特征**：能通但慢，**横向链路流量异常高**
- **原则：让所有"主"角色集中在同一台设备**
</details>

---

### 故障 08 · 用户报："新装的电脑拿不到 IP，老电脑正常"

**难度**：★★★

<details><summary>注入脚本</summary>

```cisco
SW3(config)# ip dhcp snooping
SW3(config)# ip dhcp snooping vlan 10,20
! ★ 但没有把上联口设为 trust ★
```
</details>

<details><summary>参考排查路径</summary>

```cisco
! ① 新机器的 IP
PC-NEW> ipconfig
IPv4 Address. . . . : ★ 169.254.x.x ★

! ② 老机器为什么正常？
! → 因为它们的租约还没到期，一直在用旧 IP

! ③ 端口和 VLAN 检查
SW3# show interfaces GigabitEthernet1/0/5 switchport | include Access Mode
Access Mode VLAN: 10 (VLAN0010)                 ✓ 正常

! ④ helper-address
SW1# show ip interface Vlan10 | include Helper
  Helper address is 10.1.30.53                  ✓ 正常

! ⑤ ★ 关键：查日志 ★
SW3# show logging | include DHCP
★ %DHCP_SNOOPING-5-DHCP_SNOOPING_UNTRUSTED_PORT: DHCP_SNOOPING drop message 
   on untrusted port, message type: ★ DHCPOFFER ★, MAC sa: 0011.2233.4455 ★
                                        ↑↑↑↑↑↑↑↑↑
                        ★ 服务器的 OFFER 被丢弃了！★

! ⑥ 检查 Snooping 配置
SW3# ★ show ip dhcp snooping ★
Switch DHCP snooping is enabled
DHCP snooping is configured on following VLANs: 10,20
Interface                  ★ Trusted ★    Rate limit (pps)
------------------------   -------       ----------------
GigabitEthernet1/0/23      ★ no ★        unlimited
GigabitEthernet1/0/24      ★ no ★        unlimited
! ★★ 上联口是 untrust！服务器的 OFFER 被丢了 ★★
```

**根因**：开了 DHCP Snooping，但**上联口没设为 trust**，导致 DHCP 服务器的 OFFER/ACK 被丢弃。

**修复**：
```cisco
SW3(config)# interface range GigabitEthernet1/0/23-24
SW3(config-if-range)# ★ ip dhcp snooping trust ★
```

**验证**：
```cisco
SW3# show ip dhcp snooping
Interface                  Trusted    Rate limit (pps)
GigabitEthernet1/0/23      ★ yes ★    unlimited          ✓
GigabitEthernet1/0/24      yes        unlimited

SW3# show ip dhcp snooping binding
MacAddress          IpAddress        Lease(sec)  Type           VLAN  Interface
00:11:22:33:44:55   192.168.10.55    28234       dhcp-snooping  10    Gi1/0/5
! ★ 绑定表开始填充 ✓ ★
```

**★ 学到的**：
- **DHCP Snooping 的 trust 口必须配在"通往合法 DHCP 服务器方向"**
- **忘配 trust = 全网拿不到 IP**（这是部署 Snooping 时最常见的翻车方式）
- **"老机器正常、新机器不行"** 是租约还没到期的表现，说明故障是**最近才引入的**
- **部署检查清单**：先在测试 VLAN 验证 → 确认所有上联口都 trust → 再逐 VLAN 推广
</details>

---

### 故障 09 · 用户报："能 ping 通服务器，但业务系统打不开"

**难度**：★★

<details><summary>注入脚本</summary>

```cisco
R3(config)# ip access-list extended SERVER-ACL
R3(config-ext-nacl)#  permit icmp any any
R3(config-ext-nacl)#  permit tcp any host 10.1.30.50 eq 80
R3(config-ext-nacl)#  deny ip any any log
! ★ 只放行了 80，业务实际用 8080 ★
R3(config)# interface GigabitEthernet0/2
R3(config-if)# ip access-group SERVER-ACL out
```
</details>

<details><summary>参考排查路径</summary>

```cisco
! ① ★ 分层判断：ping 通 = L3 没问题 ★
PC1> ping 10.1.30.50
Reply from 10.1.30.50: bytes=32 time=8ms        ✓ L3 通

! ② ★ L4 测试（关键的一步）★
PC1> telnet 10.1.30.50 8080
Connecting To 10.1.30.50...★ Could not open connection: 连接超时 ★
! ★ 超时（不是 refused）= 被静默丢弃 = 中间有 ACL/防火墙 ★

! ③ 对比测试
PC1> telnet 10.1.30.50 80
（连上了）                                       ← 80 通，8080 不通

! ★★ 结论：不是服务器问题，是中间设备只放行了 80 ★★

! ④ 查 ACL
R3# ★ show access-lists SERVER-ACL ★
Extended IP access list SERVER-ACL
    10 permit icmp any any ★ (2456 matches) ★
    20 permit tcp any host 10.1.30.50 eq www ★ (12 matches) ★
    30 ★ deny ip any any log (1245 matches) ★
                                ↑ ★ 大量被拒绝 ★

! ⑤ 看被拒绝的详情
R3# show logging | include IPACCESSLOG
★ %SEC-6-IPACCESSLOGP: list SERVER-ACL denied tcp 192.168.10.10(51234) 
   -> 10.1.30.50(★8080★), 24 packets ★
                            ↑ ★ 找到了：8080 端口被拒绝 ★
```

**根因**：ACL 只放行了 80 端口，业务系统实际用 8080。

**修复**：
```cisco
R3(config)# ip access-list extended SERVER-ACL
R3(config-ext-nacl)# ★ 25 permit tcp any host 10.1.30.50 eq 8080 ★
!                     ↑ 插入到序号 25（在 deny 之前）
```

**验证**：
```cisco
R3# show access-lists SERVER-ACL
    10 permit icmp any any (2456 matches)
    20 permit tcp any host 10.1.30.50 eq www (12 matches)
    ★ 25 permit tcp any host 10.1.30.50 eq 8080 (45 matches) ★  ← 有命中 ✓
    30 deny ip any any log (1245 matches)
```

**★ 学到的**：
- **"ping 通但业务不通" = L4 或 L7 问题**，不要再纠结路由
- **`telnet <ip> <port>` 是分层定位的关键工具**：
  | 结果 | 含义 |
  |:--|:--|
  | **Open** | 端口通，问题在应用层 |
  | **Connection refused** | 收到 RST → **服务端没监听** |
  | **超时** | ★ **被中间设备静默丢弃**（ACL/防火墙） |
- **`show access-lists` 的命中计数 + `deny ... log` 的日志** 能直接告诉你哪个端口被拦
</details>

---

### 故障 10 · 用户报："今天所有设备的日志时间都不对，安全审计报警了"

**难度**：★★

<details><summary>注入脚本</summary>

```cisco
R3(config)# ip access-list extended BLOCK-NTP
R3(config-ext-nacl)#  deny udp any any eq ntp
R3(config-ext-nacl)#  permit ip any any
R3(config)# interface GigabitEthernet0/1
R3(config-if)# ip access-group BLOCK-NTP out

! 同时手工改错某台设备的时间
SW1# clock set 10:00:00 1 Jan 2020
```
</details>

<details><summary>参考排查路径</summary>

```cisco
! ① 确认时间
SW1# ★ show clock ★
★ 10:23:45.123 CST Wed Jan 1 2020 ★             ← ★ 时间完全不对 ★

R3# show clock
*14:35:22.456 CST Sat Aug 31 2026               ← 这台是对的

! ② NTP 同步状态
SW1# ★ show ntp status ★
★ Clock is unsynchronized ★, stratum 16, no reference clock
   ↑↑↑↑↑↑↑↑↑↑↑↑↑↑                      ↑
   ★ 未同步 ★                    ★ stratum 16 = 无效 ★

SW1# ★ show ntp associations ★
  address         ref clock   st  when  poll reach  delay  offset  disp
  ~10.1.30.61     ★.INIT.★   ★16★   -   1024  ★0★   0.000   0.000  16000
                     ↑        ↑            ↑
              未初始化    无效层级    ★ reach 0 = 一次都没成功 ★

! ③ 网络可达性
SW1# ping 10.1.30.61
!!!!!                                            ✓ IP 通

! ④ ★ UDP 123 通吗？★
! 用扩展 ping 或抓包
SW1# debug ntp packets
! ★ 只有发出去的，没有收到的 ★

! ⑤ 查 ACL
R3# ★ show access-lists BLOCK-NTP ★
Extended IP access list BLOCK-NTP
    10 ★ deny udp any any eq ntp (1245 matches) ★  ← 找到了
    20 permit ip any any
```

**根因**：ACL 拦截了 UDP 123（NTP）。

**修复**：
```cisco
R3(config)# ip access-list extended BLOCK-NTP
R3(config-ext-nacl)# ★ no deny udp any any eq ntp ★
R3(config-ext-nacl)# ★ permit udp any any eq ntp ★

! ★ 如果时间偏差 > 1000 秒，NTP 会拒绝同步，需要先手工校准 ★
SW1# ★ clock set 14:35:00 31 Aug 2026 ★
```

**验证（NTP 同步需要 5-15 分钟）**：
```cisco
SW1# show ntp status
★ Clock is synchronized ★, stratum 4, reference is 10.1.30.61

SW1# show ntp associations
  address         ref clock      st  when  poll reach  delay  offset  disp
★*★~10.1.30.61    203.107.6.88   3    45    64  ★377★  1.234   0.567  0.123
 ↑                                              ↑
★ * = 当前同步源 ★                    ★ 377（八进制）= 最近8次全成功 ★
```

**★ 学到的**：
- **NTP 是很多功能的隐藏前提**：
  | 依赖 NTP 的功能 | 时间不准的后果 |
  |:--|:--|
  | **日志关联分析** | 多设备日志对不上，无法还原故障时间线 |
  | **证书验证** | HTTPS/802.1X/DTLS 报"证书尚未生效" |
  | **Kerberos/AD** | 容忍 5 分钟，超了**全公司登录失败** |
  | **时间 ACL** | 行为不可预测 |
  | **key-chain 的 lifetime** | 路由协议认证失败 |
  | **审计合规** | 时间戳不可信，报告无效 |
- **`reach 377`**（八进制）= 最近 8 次轮询全部成功
- **★ NTP 偏差 > 1000 秒时会拒绝同步**，必须先手工 `clock set`
</details>

---

### 故障 11 · 综合故障（★ 三个故障同时存在）

**难度**：★★★★★

<details><summary>注入脚本</summary>

```cisco
! 故障 1
SW1(config)# interface Vlan10
SW1(config-if)# no standby 10 preempt

! 故障 2
R1(config)# interface GigabitEthernet0/1
R1(config-if)# shutdown
R1(config-if)# no shutdown
! （模拟一次链路抖动，让 HSRP 切换后不切回）

! 故障 3
R2(config)# ip access-list extended NAT-LIST
R2(config-ext-nacl)#  no permit ip 192.168.0.0 0.0.15.255 any
R2(config-ext-nacl)#  permit ip 192.168.0.0 0.0.7.255 any
! （NAT 的范围缩小了，VLAN20 的 192.168.20.x 不再被 NAT）
```

**用户报障**："VLAN10 的人上网正常但速度慢，VLAN20 的人完全上不了网。"
</details>

<details><summary>参考排查路径</summary>

**★ 关键：不要试图一次解决所有问题。分而治之。**

```
   现象分解：
   ① VLAN10：能上网但慢       → 可能是路径问题
   ② VLAN20：完全不通         → 独立的问题
   
   ★ 两个不同的现象，很可能是两个不同的根因 ★
```

**先解决 VLAN20（完全不通，更严重）**：
```cisco
! ① 分层定位
PC2(VLAN20)> ping 192.168.20.1
Reply ✓                                          ← 网关通，L2/L3 本地正常

PC2> ping 10.1.30.50                             ← 内网服务器
Reply ✓                                          ← 内网路由正常

PC2> ★ ping 8.8.8.8 ★
Request timed out.                               ← ★ 出网不通

! ② ★ 范围判断：内网通、外网不通 → 问题在出口 ★

! ③ 在出口路由器上测
R2# ping 8.8.8.8 source Loopback0
!!!!!                                            ← 路由器自己能出去

! ④ ★ 查 NAT ★
R2# show ip nat translations | include 192.168.20
! ★ 空！VLAN20 的流量没有被 NAT ★

R2# show ip nat translations | include 192.168.10
tcp 203.2.2.2:50001  192.168.10.10:51234 ...     ← VLAN10 有

! ⑤ 查 NAT 的 ACL
R2# ★ show access-lists NAT-LIST ★
Extended IP access list NAT-LIST
    10 permit ip 192.168.0.0 ★ 0.0.7.255 ★ any (12456 matches)
                              ↑↑↑↑↑↑↑↑
        ★ 通配符只覆盖 192.168.0.0 - 192.168.7.255 ★
        ★ 192.168.20.x 不在范围内！★
```

**修复 1**：
```cisco
R2(config)# ip access-list extended NAT-LIST
R2(config-ext-nacl)# ★ no permit ip 192.168.0.0 0.0.7.255 any ★
R2(config-ext-nacl)# ★ permit ip 192.168.0.0 0.0.15.255 any ★
!                                    ↑ /20 覆盖 192.168.0.0 - 192.168.15.255
```

**再解决 VLAN10 的"慢"**：
```cisco
! ⑥ 看流量走哪条出口
R3# traceroute 8.8.8.8 source 192.168.10.1
  1 10.0.23.2       ← ★ 走 R2（联通备线）★
  2 203.2.2.254
! 应该走 R1（电信主线）

! ⑦ ★ 查 HSRP ★
SW1# show standby brief
Interface   Grp  Pri  P  State    Active         Standby   Virtual IP
Vlan10      10   110 ★ ★ ★Standby★ 192.168.10.3  local     192.168.10.1
                       ↑    ↑↑↑↑↑↑↑
              ★ P 列空 ★  ★ 优先级 110 更高却是 Standby ★

! ★★ 找到了：优先级更高但没配 preempt，所以链路恢复后不切回 ★★
```

**修复 2**：
```cisco
SW1(config)# interface Vlan10
SW1(config-if)# ★ standby 10 preempt ★
SW1(config-if)# ★ standby 10 preempt delay minimum 60 ★
```

**验证全部修复**：
```cisco
SW1# show standby brief
Vlan10      10   110 ★P★ ★Active★  local  ...            ✓

PC2> ping 8.8.8.8
Reply from 8.8.8.8: bytes=32 time=25ms                   ✓

R3# traceroute 8.8.8.8 source 192.168.10.1
  1 10.0.13.1       ← 走 R1（电信）✓
```

**★ 学到的**：
- **多个故障同时存在时，先分解现象**："完全不通"和"能通但慢"通常是不同的根因
- **优先处理最严重的**（完全不通 > 性能问题）
- **通配符掩码算错是经典错误**：`0.0.7.255` = /21，`0.0.15.255` = /20
- **HSRP 不配 preempt 的后果**：链路抖动一次后**永久性地停留在备用路径上**，而且没有任何告警
</details>

---

## Part 4：排障能力自评

完成所有故障后，给自己打分：

| 能力项 | 自评（1-5） |
|:--|:--|
| 能精确描述现象（而不是"网络不通"） | |
| 能快速判断影响范围 | |
| 会用分层法（L1→L7）逐层排除 | |
| 会用二分法缩小范围 | |
| 会用对比法（能通 vs 不能通） | |
| **验证假设再动手改配置**（而不是瞎试） | |
| 熟悉每一层的关键命令 | |
| 能看懂日志并从中找线索 | |
| 修复后会验证并记录 | |

**总分 < 30**：回去把前面的实验重做一遍，重点做故障注入部分。

---

## Part 5：常用排障命令速查卡

**★ 打印出来贴在工位上。**

### L1 物理层
```cisco
show interfaces status
show interfaces status err-disabled
show interfaces <接口> | include error|drop|rate|collision
show interfaces counters errors
show interfaces <接口> transceiver detail      ! 光模块诊断
```

### L2 链路层
```cisco
show mac address-table [address X] [interface X] [vlan X]
★ show interfaces trunk ★                      ! 四段输出法
show vlan brief
★ show spanning-tree vlan X detail | include occurr|from ★   ! 找震荡源
show spanning-tree inconsistentports
show etherchannel summary
show cdp neighbors detail / show lldp neighbors detail
```

### L3 网络层
```cisco
show ip interface brief
show ip route [目标IP]
★ ping <目标> source <源接口> ★                 ! ★ 永远指定 source
★ ping <目标> size 1500 df-bit ★                ! ★ MTU 测试
traceroute <目标> source <源接口>
show ip arp
★ show standby brief ★                          ! HSRP（看 P 列）
show track [brief]
```

### 路由协议
```cisco
show ip ospf neighbor
★ show ip ospf database ★                       ! ★ 排障分界线
show ip ospf interface <接口>
show ip eigrp neighbors [detail]
★ show ip eigrp topology [active] [all-links] ★
★ show ip bgp summary ★
★ show ip bgp <前缀> ★                          ! 看选路原因
show ip bgp neighbors X received-routes|routes|advertised-routes
★ show ip bgp rib-failure ★
show ip protocols
```

### L4 传输层
```cisco
★ telnet <ip> <port> ★                          ! ★ 分层定位关键工具
show tcp brief
show control-plane host open-ports
```

### 服务
```cisco
show ip dhcp binding / pool / conflict
show ip dhcp snooping [binding]
★ show ntp status / show ntp associations ★
show ip nat translations [| include X | count]
show ip nat statistics
★ show access-lists ★                           ! 看命中计数
show ip sla statistics
```

### 系统
```cisco
★ show processes cpu sorted | exclude 0.00 ★
show processes memory sorted
★ show logging [| include XXX] ★
★ show archive log config all ★                 ! 谁改了什么
show version
show environment all
```

### 抓包
```cisco
monitor capture CAP interface X both
monitor capture CAP match ipv4 any any
monitor capture CAP start / stop
show monitor capture CAP buffer brief
monitor capture CAP export flash:cap.pcap
```

---

## 🎓 毕业

完成这个实验，你已经具备：

✅ 从零搭建一个完整的企业网络（L2 + L3 + 路由 + 出口 + 安全）
✅ 系统化的分层排障能力
✅ 对 11 类典型故障的"症状 → 根因"直觉
✅ 用命令验证假设而不是瞎试的习惯

**下一步**：
1. 精读 [Stage 5 · 排障方法论](../05-排障方法论/README.md)
2. 刷 [Stage 6 · 备考与考试](../06-备考与考试/README.md) 的易错点清单
3. **约考**

---

**上一个** ← [Lab 09](lab-09-DMVPN.md) ｜ **下一阶段** → [Stage 5 · 排障方法论](../05-排障方法论/README.md)
