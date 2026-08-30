# 05 · ACL 访问控制列表

## ① 这章解决什么问题

财务部的网段配好了，能上网了，能访问服务器了。然后老板问：**"能不能让销售部访问不了财务的共享盘？"**

路由让"能通"，ACL 决定"该不该通"。

但 ACL 的价值远不止访问控制。它是 IOS 里一套**通用的"匹配流量"语言**：
- **访问控制** —— `ip access-group`
- **NAT 匹配哪些流量做转换** —— `ip nat inside source list`
- **路由过滤** —— `distribute-list`
- **QoS 分类** —— `class-map match access-group`
- **VPN 感兴趣流** —— `crypto map match address`
- **debug 过滤** —— `debug ip packet 101`

**学会 ACL 就是学会了 IOS 的"流量描述语言"。** 后面 route-map、prefix-list、class-map 全都建立在这个基础上。

---

## ② 原理讲透

### 2.1 ACL 的工作方式

ACL 是一张**自上而下、顺序匹配**的规则表：

```
数据包进来
    │
    ▼
 规则 1 匹配吗？ ── 是 ──> 执行动作 (permit/deny) ──> 【停止，不再往下看】
    │ 否
    ▼
 规则 2 匹配吗？ ── 是 ──> 执行动作 ──> 【停止】
    │ 否
    ▼
   ......
    │ 都不匹配
    ▼
 【隐含的 deny any】 ──> 丢弃
```

**三条铁律**：

1. **自上而下，第一条匹配即执行，后面的规则不再检查。**
   → 所以 **顺序至关重要**。具体的规则必须放在宽泛的规则前面。

2. **末尾有一条看不见的 `deny any`。**
   → 所以 ACL **至少要有一条 permit**，否则等于全部拒绝。这是新手最常见的翻车点。

3. **ACL 应用在接口上时要指定方向（in / out）。**
   → 方向是**相对于路由器**的：`in` = 进入路由器，`out` = 离开路由器。

### 2.2 ACL 类型

| 类型 | 编号范围 | 能匹配什么 | 应该放哪 |
|:--|:--|:--|:--|
| **标准 ACL** | 1–99, 1300–1999 | **只能匹配源 IP** | **靠近目的地** |
| **扩展 ACL** | 100–199, 2000–2699 | 源IP + 目的IP + 协议 + 端口 + 更多 | **靠近源** |
| **命名 ACL** | 用名字 | 同上，但可编辑 | 同上 |

**为什么标准 ACL 要放靠近目的地？**

标准 ACL 只能看源 IP，无法区分"这个包要去哪儿"。如果放在靠近源的位置，会把该源发往**所有目的地**的流量全部拦掉——包括本来允许访问的。

```
   PC-A ──── R1 ──── R2 ──── [财务服务器]
                       └──── [公共文件服务器]

需求：禁止 PC-A 访问财务服务器，但允许访问公共文件服务器

如果标准 ACL "deny PC-A" 放在 R1 入口：
    → PC-A 去哪儿都被拦，公共文件服务器也访问不了 ❌
    
放在 R2 靠近财务服务器的出接口：
    → 只拦住去财务服务器的方向 ✅
```

**为什么扩展 ACL 要放靠近源？**

扩展 ACL 能精确指定目的地和端口，不会误伤。放在靠近源的位置可以**尽早丢弃无用流量**，避免它白白占用整条链路的带宽和中间设备的转发资源。

> **口诀：标准靠目的，扩展靠源头。**
> 底层逻辑：**能精确匹配的就尽早拦，不能精确匹配的就尽晚拦。**

### 2.3 通配符掩码（回顾）

```
通配符掩码：0 = 必须匹配，1 = 不关心
（和子网掩码正好相反）
```

| 需求 | 写法 | 等价简写 |
|:--|:--|:--|
| 精确一个主机 | `192.168.1.10 0.0.0.0` | `host 192.168.1.10` |
| 整个 /24 | `192.168.1.0 0.0.0.255` | — |
| 整个 /26 | `192.168.1.0 0.0.0.63` | — |
| 整个 /22 | `10.1.4.0 0.0.3.255` | — |
| 任意地址 | `0.0.0.0 255.255.255.255` | `any` |

**快速换算**：通配符掩码 = `255.255.255.255 − 子网掩码`

| CIDR | 子网掩码 | 通配符 |
|:--|:--|:--|
| /24 | 255.255.255.0 | 0.0.0.255 |
| /25 | 255.255.255.128 | 0.0.0.127 |
| /26 | 255.255.255.192 | 0.0.0.63 |
| /27 | 255.255.255.224 | 0.0.0.31 |
| /28 | 255.255.255.240 | 0.0.0.15 |
| /30 | 255.255.255.252 | 0.0.0.3 |
| /23 | 255.255.254.0 | 0.0.1.255 |
| /22 | 255.255.252.0 | 0.0.3.255 |
| /16 | 255.255.0.0 | 0.0.255.255 |

### 2.4 标准 ACL

```cisco
! 编号式
R1(config)# access-list 10 permit 192.168.1.0 0.0.0.255
R1(config)# access-list 10 deny host 192.168.2.50
R1(config)# access-list 10 permit any

! 命名式（推荐）
R1(config)# ip access-list standard ALLOW-LAN
R1(config-std-nacl)# permit 192.168.1.0 0.0.0.255
R1(config-std-nacl)# deny host 192.168.2.50
R1(config-std-nacl)# permit any

! 应用到接口
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip access-group ALLOW-LAN out
```

### 2.5 扩展 ACL

**完整语法**：
```
access-list <100-199> {permit|deny} <协议> <源> <源通配符> [源端口]
                                     <目的> <目的通配符> [目的端口] [选项]
```

```cisco
R1(config)# ip access-list extended WEB-POLICY

! 允许访问 Web 服务器的 80 和 443
R1(config-ext-nacl)# permit tcp 192.168.1.0 0.0.0.255 host 10.0.0.100 eq 80
R1(config-ext-nacl)# permit tcp 192.168.1.0 0.0.0.255 host 10.0.0.100 eq 443

! 端口范围
R1(config-ext-nacl)# permit tcp any host 10.0.0.100 range 8000 8100

! 大于/小于/不等于
R1(config-ext-nacl)# permit tcp any any gt 1023
R1(config-ext-nacl)# permit udp any any lt 1024
R1(config-ext-nacl)# deny tcp any any neq 80

! 允许 DNS（★ TCP 和 UDP 都要，见基础篇第5章）
R1(config-ext-nacl)# permit udp any any eq domain
R1(config-ext-nacl)# permit tcp any any eq domain

! 允许 ICMP（细粒度）
R1(config-ext-nacl)# permit icmp any any echo             ! ping 请求
R1(config-ext-nacl)# permit icmp any any echo-reply       ! ping 响应
R1(config-ext-nacl)# permit icmp any any unreachable      ! ★ PMTUD 必须放行
R1(config-ext-nacl)# permit icmp any any time-exceeded    ! traceroute 需要

! 允许路由协议（协议号，不是端口）
R1(config-ext-nacl)# permit ospf any any
R1(config-ext-nacl)# permit eigrp any any
R1(config-ext-nacl)# permit tcp any any eq 179            ! BGP 用 TCP 179

! 允许 IPsec（三样都要，缺一不可）
R1(config-ext-nacl)# permit udp any any eq isakmp         ! UDP 500
R1(config-ext-nacl)# permit udp any any eq 4500           ! NAT-T
R1(config-ext-nacl)# permit esp any any                   ! 协议号 50

! 已建立的 TCP 连接（允许回程流量）
R1(config-ext-nacl)# permit tcp any any established

! 显式拒绝并记录（推荐加在最后，便于排障）
R1(config-ext-nacl)# deny ip any any log

! 应用
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip access-group WEB-POLICY in
```

### 2.6 `established` 关键字（重要）

```cisco
permit tcp any any established
```

**含义**：只允许 **ACK 或 RST 标志位被置位**的 TCP 报文通过。

**作用**：允许内网主动发起的连接的**回程流量**，但阻止外网主动发起的连接。

```
内网 → 外网：SYN                    （ACL 允许出方向）
外网 → 内网：SYN+ACK （有 ACK）     ✅ established 放行
外网 → 内网：SYN     （只有 SYN）    ❌ established 拒绝  ← 阻止外部主动连接
```

**局限**：
- 只对 TCP 有效，**UDP 和 ICMP 无能为力**（它们没有状态位）
- 是"伪状态检测"，攻击者可以构造带 ACK 位的包绕过

**更好的方案：`reflexive ACL` 或 `CBAC/ZBF`（真正的有状态检测）**
```cisco
! 自反 ACL（真状态跟踪）
R1(config)# ip access-list extended OUTBOUND
R1(config-ext-nacl)# permit tcp any any reflect TCP-TRAFFIC
R1(config-ext-nacl)# permit udp any any reflect UDP-TRAFFIC

R1(config)# ip access-list extended INBOUND
R1(config-ext-nacl)# evaluate TCP-TRAFFIC
R1(config-ext-nacl)# evaluate UDP-TRAFFIC
R1(config-ext-nacl)# deny ip any any
```

ZBF（Zone-Based Firewall）在 [Security 选修模块](../08-Security选修/02-ACL与区域防火墙ZBF.md) 详细讲。

### 2.7 编辑 ACL（命名 ACL 的核心优势）

**编号 ACL 的痛点**：不能删除单条规则，只能整个删掉重建。而且新增的规则永远加在末尾。

**命名 ACL 支持按序号编辑**：
```cisco
R1(config)# ip access-list extended WEB-POLICY
R1(config-ext-nacl)# 15 permit tcp any host 10.0.0.100 eq 8080    ! 插入到序号 15 的位置
R1(config-ext-nacl)# no 20                                        ! 删除序号 20 的规则

! 查看序号
R1# show access-lists WEB-POLICY
Extended IP access list WEB-POLICY
    10 permit tcp 192.168.1.0 0.0.0.255 host 10.0.0.100 eq www (245 matches)
    15 permit tcp any host 10.0.0.100 eq 8080
    20 permit tcp 192.168.1.0 0.0.0.255 host 10.0.0.100 eq 443 (89 matches)
    30 deny ip any any log (12 matches)

! 重新编号（整理序号，留出插入空间）
R1(config)# ip access-list resequence WEB-POLICY 10 10
                                              ↑起始 ↑步长
```

> **实践建议**：**永远用命名 ACL**。编号 ACL 除了在极老的设备上没有任何优势。命名 ACL 可读性好（`ALLOW-DMZ-WEB` 比 `101` 清楚多了）、可以插入删除单条规则、方便交接。

---

## ③ 配置命令

```cisco
! ═══ 应用到接口 ═══
R1(config-if)# ip access-group WEB-POLICY in
R1(config-if)# ip access-group WEB-POLICY out
R1(config-if)# no ip access-group WEB-POLICY in       ! 解除

! ═══ 保护 VTY（限制谁能 SSH 到设备）═══
R1(config)# ip access-list standard MGMT-HOSTS
R1(config-std-nacl)# permit 10.0.0.0 0.0.0.255
R1(config-std-nacl)# permit host 192.168.100.50
R1(config)# line vty 0 15
R1(config-line)# access-class MGMT-HOSTS in           ! 注意是 access-class 不是 access-group

! ═══ 用于其他功能 ═══
! NAT
R1(config)# ip nat inside source list NAT-LIST interface Gi0/1 overload

! 路由过滤
R1(config-router)# distribute-list 10 in

! QoS 分类
R1(config)# class-map VOICE
R1(config-cmap)# match access-group name VOICE-TRAFFIC

! debug 过滤（避免刷屏）
R1# debug ip packet 101

! ═══ 查看与排障 ═══
R1# show access-lists                                  ! ★ 看命中计数
R1# show access-lists WEB-POLICY
R1# show ip interface GigabitEthernet0/0 | include access list
R1# show ip access-lists interface Gi0/0
R1# clear access-list counters                         ! 清零计数，方便观察
R1# clear access-list counters WEB-POLICY

! 查看被拒绝的日志
R1# show logging | include %SEC-6-IPACCESSLOG
```

### `show access-lists` 的命中计数是排障利器

```cisco
R1# show access-lists WEB-POLICY
Extended IP access list WEB-POLICY
    10 permit tcp 192.168.1.0 0.0.0.255 host 10.0.0.100 eq www (1245 matches)
    20 permit tcp 192.168.1.0 0.0.0.255 host 10.0.0.100 eq 443 (0 matches)
                                                                    ↑
                                          没有命中！可能规则写错了，或流量根本没到这里
    30 deny ip any any log (89 matches)
                              ↑
                    有 89 个包被拒绝了，看日志能知道是谁
```

**用法**：
```cisco
R1# clear access-list counters WEB-POLICY     ! 先清零
! 让用户重现问题
R1# show access-lists WEB-POLICY              ! 看哪条规则命中了
```

这比抓包快得多，是排 ACL 问题的第一手段。

### 三厂商对照

| 目的 | Cisco | H3C | 华为 |
|:--|:--|:--|:--|
| 基本 ACL | `access-list 10 permit ...` | `acl basic 2000` → `rule permit source ...` | `acl 2000` → `rule permit source ...` |
| 高级 ACL | `access-list 100 permit tcp ...` | `acl advanced 3000` → `rule permit tcp ...` | `acl 3000` → `rule permit tcp ...` |
| 命名 ACL | `ip access-list extended NAME` | `acl advanced name NAME` | `acl name NAME advance` |
| 应用到接口 | `ip access-group 100 in` | `packet-filter 3000 inbound` | `traffic-filter inbound acl 3000` |
| 保护 VTY | `access-class 10 in` | `user-interface vty 0 4` → `acl 2000 inbound` | `user-interface vty 0 4` → `acl 2000 inbound` |
| 查看 | `show access-lists` | `display acl all` | `display acl all` |

> **编号范围差异**：
> - Cisco：标准 1–99/1300–1999，扩展 100–199/2000–2699
> - H3C/华为：**基本 ACL 2000–2999**（相当于标准），**高级 ACL 3000–3999**（相当于扩展），二层 ACL 4000–4999
>
> **默认动作差异（重要）**：Cisco ACL 末尾是隐含的 **`deny any`**；**H3C/华为的 ACL 本身末尾也是隐含拒绝，但取决于应用方式**——用 `packet-filter` 时默认拒绝，用在 QoS 分类时行为不同。跨厂商迁移策略时务必逐条验证。

---

## ④ 配套实验：分部门访问控制

**场景**：
```
                       ┌─────────┐
   VLAN10 财务 ────────┤         │
   192.168.10.0/24     │         │      ┌──────────────────┐
                       │   R1    ├──────┤ 服务器区          │
   VLAN20 销售 ────────┤         │      │ Web:  10.0.0.100 │
   192.168.20.0/24     │         │      │ 财务: 10.0.0.200 │
                       │         │      │ DB:   10.0.0.300 │
   VLAN30 访客 ────────┤         │      └──────────────────┘
   192.168.30.0/24     └────┬────┘
                            │ Gi0/3
                        [ Internet ]
```

**策略需求**：

| 来源 | Web(80/443) | 财务系统(8080) | 数据库(3306) | 互联网 | 互相访问 |
|:--|:--|:--|:--|:--|:--|
| 财务 VLAN10 | ✅ | ✅ | ❌ | ✅ | — |
| 销售 VLAN20 | ✅ | ❌ | ❌ | ✅ | ❌ 不能访问财务 |
| 访客 VLAN30 | ❌ | ❌ | ❌ | ✅ 仅互联网 | ❌ 不能访问任何内网 |

### Step 1：访客隔离（最简单也最重要）

```cisco
R1(config)# ip access-list extended GUEST-POLICY
 ! 允许 DHCP（访客要拿 IP）
R1(config-ext-nacl)# permit udp any any eq bootps
R1(config-ext-nacl)# permit udp any any eq bootpc
 ! 允许 DNS
R1(config-ext-nacl)# permit udp any host 10.0.0.53 eq domain
R1(config-ext-nacl)# permit tcp any host 10.0.0.53 eq domain
 ! ★ 拒绝访问所有内网（放在允许上网之前）
R1(config-ext-nacl)# deny ip any 10.0.0.0 0.255.255.255
R1(config-ext-nacl)# deny ip any 172.16.0.0 0.15.255.255
R1(config-ext-nacl)# deny ip any 192.168.0.0 0.0.255.255
 ! 其余（互联网）放行
R1(config-ext-nacl)# permit ip any any

R1(config)# interface Vlan30
R1(config-if)# ip access-group GUEST-POLICY in
```

> **顺序的重要性**：三条 `deny` 私网的规则**必须放在 `permit ip any any` 前面**。如果反过来，`permit ip any any` 会先匹配，后面的 deny 永远不会执行——访客就能访问整个内网了。
>
> **这是 ACL 最经典的错误。** 记住：**具体规则在前，宽泛规则在后。**

### Step 2：销售部策略

```cisco
R1(config)# ip access-list extended SALES-POLICY
 ! 允许访问 Web 服务器
R1(config-ext-nacl)# permit tcp 192.168.20.0 0.0.0.255 host 10.0.0.100 eq 80
R1(config-ext-nacl)# permit tcp 192.168.20.0 0.0.0.255 host 10.0.0.100 eq 443
 ! 允许 DNS
R1(config-ext-nacl)# permit udp 192.168.20.0 0.0.0.255 host 10.0.0.53 eq domain
R1(config-ext-nacl)# permit tcp 192.168.20.0 0.0.0.255 host 10.0.0.53 eq domain
 ! 拒绝访问财务 VLAN 和财务服务器
R1(config-ext-nacl)# deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255 log
R1(config-ext-nacl)# deny ip 192.168.20.0 0.0.0.255 host 10.0.0.200 log
R1(config-ext-nacl)# deny ip 192.168.20.0 0.0.0.255 host 10.0.0.300 log
 ! 其余上网放行
R1(config-ext-nacl)# permit ip 192.168.20.0 0.0.0.255 any

R1(config)# interface Vlan20
R1(config-if)# ip access-group SALES-POLICY in
```

### Step 3：财务部策略

```cisco
R1(config)# ip access-list extended FINANCE-POLICY
R1(config-ext-nacl)# permit tcp 192.168.10.0 0.0.0.255 host 10.0.0.100 eq 80
R1(config-ext-nacl)# permit tcp 192.168.10.0 0.0.0.255 host 10.0.0.100 eq 443
R1(config-ext-nacl)# permit tcp 192.168.10.0 0.0.0.255 host 10.0.0.200 eq 8080
R1(config-ext-nacl)# permit udp 192.168.10.0 0.0.0.255 host 10.0.0.53 eq domain
R1(config-ext-nacl)# permit tcp 192.168.10.0 0.0.0.255 host 10.0.0.53 eq domain
 ! 明确拒绝直连数据库（财务系统才能连 DB）
R1(config-ext-nacl)# deny tcp 192.168.10.0 0.0.0.255 host 10.0.0.300 eq 3306 log
R1(config-ext-nacl)# permit ip 192.168.10.0 0.0.0.255 any

R1(config)# interface Vlan10
R1(config-if)# ip access-group FINANCE-POLICY in
```

### Step 4：验证测试矩阵

| 测试 | 命令 | 预期 |
|:--|:--|:--|
| 财务 → 财务系统 | `telnet 10.0.0.200 8080` | ✅ Open |
| 财务 → 数据库 | `telnet 10.0.0.300 3306` | ❌ 超时 |
| 销售 → Web | `telnet 10.0.0.100 80` | ✅ Open |
| 销售 → 财务系统 | `telnet 10.0.0.200 8080` | ❌ 超时 |
| 销售 → 财务 PC | `ping 192.168.10.10` | ❌ 不通 |
| 访客 → 内网任意 | `ping 10.0.0.100` | ❌ 不通 |
| 访客 → 互联网 | `ping 8.8.8.8` | ✅ 通 |

### Step 5：用命中计数验证

```cisco
R1# clear access-list counters
! 让各部门执行测试
R1# show access-lists SALES-POLICY

Extended IP access list SALES-POLICY
    10 permit tcp 192.168.20.0 0.0.0.255 host 10.0.0.100 eq www (45 matches)
    20 permit tcp 192.168.20.0 0.0.0.255 host 10.0.0.100 eq 443 (128 matches)
    30 permit udp 192.168.20.0 0.0.0.255 host 10.0.0.53 eq domain (67 matches)
    40 permit tcp 192.168.20.0 0.0.0.255 host 10.0.0.53 eq domain (2 matches)
    50 deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255 log (8 matches)  ← 有人试图访问财务
    60 deny ip 192.168.20.0 0.0.0.255 host 10.0.0.200 log (3 matches)
    70 deny ip 192.168.20.0 0.0.0.255 host 10.0.0.300 log (0 matches)
    80 permit ip 192.168.20.0 0.0.0.255 any (2341 matches)
```

```cisco
R1# show logging | include IPACCESSLOG
%SEC-6-IPACCESSLOGDP: list SALES-POLICY denied icmp 192.168.20.50 -> 192.168.10.10 (8 packets)
                                                        ↑ 具体是谁在尝试
```

> ⚠️ **`log` 关键字的性能代价**：带 `log` 的 ACL 规则会**把匹配的包送到 CPU 处理**（process switching），而不是走硬件快速转发。在高流量场景下会导致 CPU 飙升。
>
> **生产环境用法**：只在排障期间临时加 `log`，问题解决后去掉。或者用 `log-input`（会额外记录入接口和源 MAC，更有诊断价值，但代价更高）。
>
> **替代方案**：用 NetFlow 或 Flexible NetFlow 做流量可视化，比 ACL log 高效得多。

---

## ⑤ 排障思路

| 症状 | 怀疑点 | 验证 | 根因 |
|:--|:--|:--|:--|
| ACL 配了但完全没生效 | 未应用到接口 | `show ip int Gi0/0 \| inc access` | 忘了 `ip access-group` |
| 全部流量被拦 | 缺 permit / 顺序错 | `show access-lists` 看哪条命中 | 隐含 `deny any` 生效了 |
| 该拦的没拦住 | 顺序错 | 同上 | 宽泛的 permit 排在具体的 deny 前面 |
| 部分应用不通 | 端口遗漏 | `show access-lists` 看 deny 计数 | 漏放某个端口（DNS 的 TCP、被动 FTP） |
| ping 通但业务不通 | 只放了 ICMP | 同上 | 忘记放行业务端口 |
| 大文件传不动 | ICMP 被全拦 | 检查是否有 `deny icmp` | **PMTUD 黑洞**，需放行 `icmp unreachable` |
| VPN 建不起来 | IPsec 三件套 | 检查 udp500/4500/esp | 漏放 ESP（协议号 50） |
| OSPF 邻居起不来 | 未放行协议 89 | `show access-lists` | 只写了 TCP/UDP，漏了 `permit ospf` |
| CPU 飙升 | ACL log | `show processes cpu sorted` | `log` 关键字导致 process switching |
| 改 ACL 后自己被踢出 | 未保护管理流量 | — | 应在最前面加一条放行管理 IP |

### 方向判断（最容易搞错的地方）

**方向是相对于路由器的**：

```
              ┌─────────────────┐
   PC ────────┤ Gi0/0     Gi0/1 ├──────── Server
              │   ↓in     out↑  │
              │   ↑out     in↓  │
              └─────────────────┘
              
PC → Server 的流量：从 Gi0/0 进（in），从 Gi0/1 出（out）
Server → PC 的流量：从 Gi0/1 进（in），从 Gi0/0 出（out）
```

**想拦住 "PC 访问 Server"**，可以放在：
- `Gi0/0 in`（推荐，尽早拦截）
- `Gi0/1 out`

**⚠️ 常见错误**：写好了 ACL 匹配 "源=PC，目的=Server"，却应用在 `Gi0/0 out`。此时该接口的 out 方向流量是 Server→PC，源目正好相反，永远不会匹配。

**验证方法**：
```cisco
R1# show access-lists MY-ACL
! 如果所有规则的 matches 都是 0，很可能是方向搞反了
```

### ACL 修改的安全操作

**在生产环境修改 ACL，必须防止把自己锁在门外**：

```cisco
! 方法 1：在最前面加一条放行自己
R1(config)# ip access-list extended PROD-ACL
R1(config-ext-nacl)# 1 permit ip host <你的管理IP> any

! 方法 2：用 configure revert（推荐）
R1# configure terminal revert timer 5
R1(config)# ... 改 ACL ...
R1(config)# end
! 验证连接正常
R1# configure confirm

! 方法 3：先做副本，验证后切换
R1(config)# ip access-list extended PROD-ACL-V2
R1(config-ext-nacl)# ... 新规则 ...
R1(config)# interface Gi0/0
R1(config-if)# ip access-group PROD-ACL-V2 in       ! 一条命令切换
! 有问题立刻切回
R1(config-if)# ip access-group PROD-ACL in
```

> **方法 3 是最专业的做法**：新旧 ACL 并存，切换是原子操作，回滚只需一条命令。核心设备强烈推荐。

---

## ⑥ 考点提示 + 自测题

### 考点

- **标准靠目的、扩展靠源**及其原因。
- **末尾隐含 `deny any`**。
- **顺序：具体在前，宽泛在后**。
- **通配符掩码计算**。
- **`established` 关键字**的作用和局限。
- **协议号 vs 端口号**：OSPF(89)、EIGRP(88)、ESP(50)、GRE(47) 不能写端口。
- **DNS 需要同时放行 TCP/UDP 53**。
- **放行 `icmp unreachable` 以避免 PMTUD 黑洞**。

### 自测题

**1.** 下面这个 ACL 有什么问题？
```cisco
ip access-list extended MY-ACL
 permit ip any any
 deny tcp 192.168.20.0 0.0.0.255 host 10.0.0.200 eq 8080
```

<details><summary>答案</summary>

**`deny` 规则永远不会生效。**

ACL 是**自上而下顺序匹配，第一条匹配就执行并停止**。`permit ip any any` 匹配所有流量，所以任何包在第一条就被放行了，后面的 `deny` 规则**永远不会被检查到**。

`show access-lists` 会明显暴露这个问题：
```cisco
R1# show access-lists MY-ACL
Extended IP access list MY-ACL
    10 permit ip any any (98234 matches)
    20 deny tcp 192.168.20.0 0.0.0.255 host 10.0.0.200 eq 8080 (0 matches)
                                                                     ↑ 永远是 0
```

**正确写法**：
```cisco
ip access-list extended MY-ACL
 deny tcp 192.168.20.0 0.0.0.255 host 10.0.0.200 eq 8080 log
 permit ip any any
```

**通用原则：具体的规则在前，宽泛的规则在后。**

这个错误在实际工作中极其常见，尤其是"临时加一条 deny"的时候顺手加到了末尾。**修改 ACL 后一定要用 `show access-lists` 检查命中计数**——`0 matches` 的规则要么是写错了，要么是位置不对。
</details>

**2.** 为什么标准 ACL 要放在靠近目的地的位置，而扩展 ACL 要放在靠近源的位置？

<details><summary>答案</summary>

**标准 ACL 只能匹配源 IP，无法区分目的地。**

如果放在靠近源的位置，它会拦掉这个源发往**所有目的地**的流量。

```
   PC-A ──── R1 ──── R2 ──┬── [财务服务器]  ← 要拦
                          └── [文件服务器]  ← 要放行

需求：只禁止 PC-A 访问财务服务器

标准 ACL "deny host PC-A" 放在 R1 入口：
   → PC-A 去任何地方都被拦，文件服务器也访问不了 ❌ 误伤

放在 R2 连接财务服务器的出接口：
   → 只影响去财务服务器的流量 ✅
```

**扩展 ACL 能精确指定源、目的、协议、端口**，不存在误伤风险。放在靠近源的位置可以**尽早丢弃无用流量**，好处是：
1. 不占用中间链路带宽
2. 不消耗中间路由器的转发资源
3. 攻击流量在入口就被挡住，不进入网络核心

**底层逻辑一句话**：**能精确匹配的就尽早拦，不能精确匹配的就尽晚拦。**

**实践中的现实**：现在几乎都用扩展 ACL（或命名 ACL），标准 ACL 主要用于：
- `access-class` 保护 VTY（只需匹配源）
- 路由过滤 `distribute-list`（只需匹配前缀）
- NAT 匹配（只需匹配源网段）

所以"标准 ACL 靠目的"更多是个考点，实战中遇到的机会不多。
</details>

**3.** 一条 ACL 里写了 `permit udp any any eq 53`，用户报告"大部分网站正常，但公司某内部系统解析不了域名"。为什么？

<details><summary>答案</summary>

**DNS 响应超过 512 字节时会切换到 TCP 53，而 ACL 没放行 TCP。**

触发场景：
- 域名配了很多条 A 记录（负载均衡）
- 启用了 DNSSEC（签名数据大）
- 有很长的 TXT 记录（SPF/DKIM）

DNS 服务器会先用 UDP 返回一个带 **TC（Truncated）标志**的响应，客户端看到后**改用 TCP 53 重新查询**。TCP 被拦 → 解析失败。

**修复**：
```cisco
permit udp any any eq domain
permit tcp any any eq domain          ← 补上
```

**这个故障很难查**，因为它表现为"随机的、只影响个别域名"的诡异问题，很少有人第一时间联想到 ACL。

**同类陷阱清单**（都是"漏放了配套协议"）：

| 服务 | 容易漏放的部分 |
|:--|:--|
| DNS | TCP 53 |
| **IPsec VPN** | **ESP（协议号 50）**，只放了 UDP 500/4500 |
| FTP 被动模式 | 高位随机端口（21 只是控制通道） |
| PMTUD | `icmp unreachable`（导致大包黑洞） |
| traceroute | `icmp time-exceeded`、UDP 33434+ |
| OSPF | **协议号 89**（不是端口！） |
| EIGRP | **协议号 88** |
| GRE | **协议号 47** |
| SIP/VoIP | RTP 的高位 UDP 端口范围 |
</details>

**4.** `permit tcp any any established` 是什么意思？它能替代真正的防火墙吗？

<details><summary>答案</summary>

**含义**：只允许 **ACK 或 RST 标志位被置位**的 TCP 报文通过。

**用途**：允许内网主动发起的连接的回程流量，同时阻止外网主动发起的连接。

```
内网 → 外网：SYN                    （出方向 ACL 允许）
外网 → 内网：SYN+ACK （带 ACK 位）    ✅ established 放行（这是回程）
外网 → 内网：SYN     （只有 SYN）     ❌ established 拒绝（这是主动连接）
```

**不能替代真正的防火墙，有三个致命局限**：

1. **它不是真正的状态检测**。它只看单个包的标志位，**不维护连接状态表**。攻击者可以构造一个带 ACK 标志的伪造包，直接穿过这条规则（虽然建立不了完整连接，但可以用于扫描、DoS 或某些注入攻击）。

2. **对 UDP 和 ICMP 完全无效**。UDP 没有标志位，`established` 关键字对它不适用。所以 DNS、NTP、VoIP、视频流的回程流量都无法用这个方法保护。

3. **不做深度检测**。不理解应用层协议，无法处理 FTP 被动模式的动态端口、SIP 的媒体流协商等场景。

**真正的有状态方案（由弱到强）**：

| 方案 | 状态跟踪 | 支持 UDP | 应用层感知 |
|:--|:--|:--|:--|
| `established` | ❌ 伪状态 | ❌ | ❌ |
| **Reflexive ACL** | ✅ 真状态 | ✅ | ❌ |
| **CBAC**（旧） | ✅ | ✅ | ✅ 部分 |
| **ZBF**（区域防火墙，推荐） | ✅ | ✅ | ✅ |
| **专业防火墙**（ASA/FTD/Palo Alto） | ✅ | ✅ | ✅ 深度 |

**Reflexive ACL 示例**（路由器上的轻量方案）：
```cisco
R1(config)# ip access-list extended OUTBOUND
R1(config-ext-nacl)# permit tcp any any reflect MIRROR
R1(config-ext-nacl)# permit udp any any reflect MIRROR
R1(config-ext-nacl)# permit icmp any any reflect MIRROR

R1(config)# ip access-list extended INBOUND
R1(config-ext-nacl)# evaluate MIRROR              ! 动态放行回程
R1(config-ext-nacl)# deny ip any any log

R1(config)# interface Gi0/1
R1(config-if)# ip access-group OUTBOUND out
R1(config-if)# ip access-group INBOUND in
```

ZBF 详见 [Security 选修模块第 2 章](../08-Security选修/02-ACL与区域防火墙ZBF.md)。
</details>

**5.** 你在生产路由器上修改 ACL，改完后 SSH 会话断了，再也连不上。怎么预防这种情况？

<details><summary>答案</summary>

**三种预防方法，按推荐顺序**：

**方法 1：`configure revert`（最优雅，不重启）**
```cisco
R1# write memory                          ! 先保存已知正确的配置
R1# configure terminal revert timer 5     ! 5 分钟内不确认就自动回滚
R1(config)# ip access-list extended PROD-ACL
R1(config-ext-nacl)# ... 修改 ...
R1(config-ext-nacl)# end
! 此时验证 SSH 还通吗？业务正常吗？
R1# configure confirm                     ! 确认，取消自动回滚
R1# write memory
```
断连了什么都不用做，5 分钟后配置自动回滚，**设备不重启，业务不中断**。

**方法 2：ACL 副本切换（最专业，零风险）**
```cisco
! 建一个新版本，不动现有的
R1(config)# ip access-list extended PROD-ACL-V2
R1(config-ext-nacl)# ... 完整的新规则 ...

! 一条命令原子切换
R1(config)# interface Gi0/0
R1(config-if)# ip access-group PROD-ACL-V2 in

! 有问题一条命令切回
R1(config-if)# ip access-group PROD-ACL in
```
好处：新旧 ACL 并存，切换和回滚都是单条命令，**永远不会出现"ACL 只改了一半"的中间态**。

**方法 3：ACL 第一条永远放行管理流量**
```cisco
R1(config)# ip access-list extended PROD-ACL
R1(config-ext-nacl)# 1 permit ip host 10.0.0.50 any        ! 你的管理机
R1(config-ext-nacl)# 2 permit ip 10.0.99.0 0.0.0.255 any   ! 管理网段
```
序号 1、2 排在最前，无论后面怎么改都不会切断管理连接。

**方法 4：`reload in`（老办法，代价大）**
```cisco
R1# write memory
R1# reload in 10
R1(config)# ... 改 ...
R1# reload cancel                          ! 验证 OK 后取消
```
断连的话 10 分钟后设备重启回到旧配置。但**重启会中断所有业务**，核心设备上代价太高。

**最佳组合：方法 3（永久防护）+ 方法 1 或 2（本次操作防护）。**

**额外提醒**：还应该配好**带外管理**（Console Server / 独立管理网口 / 4G 带外卡），这才是最后的保险。所有软件层面的保险绳都可能失效，物理带外通道不会。
</details>

---

**上一章** ← [04 OSPF 单区域](04-OSPF单区域.md) ｜ **下一章** → [06 NAT 地址转换](06-NAT地址转换.md)
