# 07 · DHCP / DNS / NTP 基础服务

## ① 这章解决什么问题

网络通了，但用户还是用不了：

- 新员工插上网线，电脑显示"无法连接网络"——IP 是 `169.254.x.x`
- 能 ping 通 IP，但输入网址打不开
- 设备日志时间全乱，出了事故根本没法对时间线做关联分析
- 802.1X 认证莫名失败，证书报"尚未生效"

这四个问题分别对应 **DHCP、DNS、NTP**。它们不属于"路由交换"，但**它们坏了，网络就等于坏了**。而且在排障时，它们是最容易被忽略的一层——工程师往往在路由表里找了半天，最后发现是 DHCP 中继没配。

---

## ② 原理讲透

### 2.1 DHCP：四步握手（DORA）

```
   客户端                                        DHCP 服务器
     │                                                │
     │ ① DHCP DISCOVER  （广播 255.255.255.255）       │
     │───────────────────────────────────────────────>│  "有人能给我个 IP 吗？"
     │   源: 0.0.0.0:68  目的: 255.255.255.255:67      │
     │                                                │
     │ ② DHCP OFFER     （广播或单播）                  │
     │<───────────────────────────────────────────────│  "给你 192.168.1.100"
     │                                                │
     │ ③ DHCP REQUEST   （广播）                       │
     │───────────────────────────────────────────────>│  "我要这个地址"
     │   仍然广播，是为了告诉其他 DHCP 服务器"我不用你的" │
     │                                                │
     │ ④ DHCP ACK                                     │
     │<───────────────────────────────────────────────│  "确认，租期 8 小时"
     │                                                │
     │           客户端发 ARP 检测地址是否冲突            │
```

**记忆：D-O-R-A（Discover, Offer, Request, Acknowledge）**

**关键细节**：
- **DISCOVER 和 REQUEST 都是广播**。REQUEST 也广播是为了通知其他 DHCP 服务器"我选了别人的，你把预留的地址释放吧"。
- 客户端此时**还没有 IP**，所以源地址是 `0.0.0.0`。
- 使用 **UDP 67（服务器）/ 68（客户端）**。

**租期续订（Renew）**：
- **T1 = 50% 租期**：客户端**单播**向原服务器请求续租
- **T2 = 87.5% 租期**：如果 T1 失败，客户端**广播**请求任意服务器续租
- **100%**：租期到期，放弃地址，重新走 DORA

### 2.2 DHCP 中继（Relay Agent）—— 跨网段的关键

**问题**：DHCP Discover 是广播，**路由器默认不转发广播**。所以客户端和 DHCP 服务器不在同一网段时，请求根本到不了服务器。

**解法**：在客户端所在网段的**网关接口**上配置 DHCP 中继：

```cisco
R1(config)# interface Vlan10
R1(config-if)# ip helper-address 10.0.0.53          ! DHCP 服务器地址
```

**中继的工作过程**：

```
客户端 ──广播──> 网关(中继) ──单播──> DHCP 服务器
                    │
                    └─ 把广播包改成单播
                    └─ ★ 在 giaddr 字段填入自己接收该请求的接口 IP
                                        ↓
                              服务器靠 giaddr 判断该从哪个地址池分配
```

**`giaddr`（Gateway IP Address）字段是整个机制的核心**：

| 作用 | 说明 |
|:--|:--|
| **告诉服务器从哪个池分配** | 服务器看 giaddr=192.168.10.1，就知道该分 `192.168.10.0/24` 的地址 |
| **告诉服务器往哪回包** | 服务器把 OFFER 单播回 giaddr，中继再广播给客户端 |

> **考点**：如果一个 SVI 上有多个 IP（主地址 + secondary），`giaddr` 填的是**主地址**。所以 DHCP 服务器上的地址池必须匹配主地址所在网段。

**`ip helper-address` 的副作用（重要）**：

它不只转发 DHCP，**默认还转发这 8 种 UDP 广播**：

| 端口 | 服务 |
|:--|:--|
| 37 | Time |
| 49 | TACACS |
| **53** | **DNS** |
| **67/68** | **DHCP/BOOTP** |
| 69 | TFTP |
| 137 | NetBIOS Name Service |
| 138 | NetBIOS Datagram |
| 49 | TACACS |

**这可能造成不必要的流量泛洪**（尤其 NetBIOS）。精确控制：

```cisco
! 关闭默认转发的服务
R1(config)# no ip forward-protocol udp 137
R1(config)# no ip forward-protocol udp 138
R1(config)# no ip forward-protocol udp 69

! 只保留 DHCP
R1(config)# ip forward-protocol udp 67
R1(config)# ip forward-protocol udp 68
```

### 2.3 DHCP 常见故障：`169.254.x.x`

**看到 `169.254.x.x` 就是 APIPA（自动私有 IP 编址）**，说明客户端**完全没收到 DHCP 响应**。

**排查顺序**：

```
1. 端口在正确的 VLAN 吗？
   SW# show interfaces Gi1/0/5 switchport
   
2. 端口配了 PortFast 吗？
   → 没配的话，STP 前 30 秒不转发，DHCP Discover 被丢弃
   SW# show spanning-tree interface Gi1/0/5 portfast
   
3. 网关配了 ip helper-address 吗？（跨网段场景）
   R1# show ip interface Vlan10 | include Helper
   
4. DHCP 服务器地址池有空闲吗？
   R1# show ip dhcp pool
   R1# show ip dhcp binding
   
5. 中间有 ACL 拦了 UDP 67/68 吗？
   R1# show access-lists
   
6. 有没有 DHCP Snooping 把端口当成不可信口？
   SW# show ip dhcp snooping
```

> **实战口诀**：看到 `169.254`，**先查 VLAN，再查 helper，最后查地址池**。这三个覆盖了 90% 的情况。

### 2.4 DHCP Snooping（安全，ENCOR 考点）

**要防的攻击**：

**① 恶意 DHCP 服务器（DHCP Spoofing）**
攻击者在内网架一台 DHCP 服务器，抢先响应 Discover，把网关指向自己 → **中间人攻击**，所有流量经过攻击者。

**② DHCP 耗尽攻击（Starvation）**
攻击者伪造大量 MAC 疯狂请求地址，把地址池耗光 → 正常用户拿不到 IP → **拒绝服务**。

**防御原理**：把端口分为**信任口（trust）**和**非信任口（untrust）**。

```
                  [DHCP 服务器]
                       │
                  trust 口 ← 只有这个口能发 DHCP OFFER/ACK
                  ┌────┴────┐
                  │  交换机  │
                  └─┬──┬──┬─┘
              untrust 口们 ← 只能发 DISCOVER/REQUEST
                │  │  │
               PC PC 攻击者 ← 它发的 OFFER 会被直接丢弃
```

```cisco
! 全局开启
SW1(config)# ip dhcp snooping
SW1(config)# ip dhcp snooping vlan 10,20,30

! 标记信任口（上联交换机/DHCP服务器方向）
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# ip dhcp snooping trust

! 限速（防耗尽攻击）
SW1(config)# interface range GigabitEthernet1/0/1-20
SW1(config-if-range)# ip dhcp snooping limit rate 10          ! 每秒最多 10 个 DHCP 包

! Option 82（在中继场景中携带端口信息）
SW1(config)# no ip dhcp snooping information option           ! 有些环境需要关掉

! 查看
SW1# show ip dhcp snooping
SW1# show ip dhcp snooping binding
```

**DHCP Snooping 绑定表的价值**：它记录了 `MAC + IP + VLAN + 端口 + 租期` 的对应关系，是 **DAI（动态 ARP 检测）** 和 **IPSG（IP 源防护）** 的数据基础：

```cisco
! 基于 snooping 表防 ARP 欺骗
SW1(config)# ip arp inspection vlan 10,20,30
SW1(config-if)# ip arp inspection trust        ! 上联口设为信任

! 基于 snooping 表防 IP 地址盗用
SW1(config-if)# ip verify source
SW1(config-if)# ip verify source port-security   ! 同时校验 MAC
```

> **这三件套（DHCP Snooping → DAI → IPSG）是接入层安全的标准组合**，ENCOR 安全章节会详细展开。

### 2.5 DNS

**解析流程**：

```
   客户端
     │ 查 www.example.com
     ▼
  ① 本地 hosts 文件
     │ 没有
     ▼
  ② 本地 DNS 缓存
     │ 没有
     ▼
  ③ 本地 DNS 服务器（递归解析器）
     │ 没有缓存 → 开始递归查询
     ▼
  ④ 根域名服务器 (.)      → "去问 .com 服务器"
     ▼
  ⑤ 顶级域服务器 (.com)   → "去问 example.com 的服务器"
     ▼
  ⑥ 权威服务器 (example.com) → "www.example.com = 93.184.216.34"
     ▼
   返回给客户端并缓存
```

**★ TTL：缓存能存多久 ★**

上面流程的最后一步是"缓存"，但**能缓存多久**由权威服务器说了算——这个值就是 **DNS TTL**。

> ⚠️ **`TTL` 这个缩写在网络里指两个完全不同的东西，考试和面试都爱在这里设陷阱。**
> 看到 TTL 先问自己一句：**几跳，还是几秒？**

| | **IP 头 TTL** | **DNS TTL** |
|:--|:--|:--|
| 位置 | IP 报文头字段 | DNS 应答记录里的字段 |
| 单位 | **跳数**（每过一台路由器减 1） | **秒** |
| 作用 | 防环路、`traceroute` 的原理 | 解析结果的缓存有效期 |
| 归零后 | 丢包 + 回 ICMP Time Exceeded | 缓存失效，必须重新查询 |
| 详见 | [基础篇第 1 章](../00-基础篇/01-网络分层与数据封装.md) | 本节 |

DNS 应答的每条记录都自带 TTL：

```bash
$ dig www.example.com

;; ANSWER SECTION:
www.example.com.    3600    IN    A    93.184.216.34
                    ↑
                    TTL=3600 秒，解析器可以缓存 1 小时
```

**TTL 长短是一个明确的权衡**：

| | TTL 长（如 86400） | TTL 短（如 1~60） |
|:--|:--|:--|
| 上游查询量 | 少，权威服务器压力小 | 大 |
| 解析速度 | 快（多数命中缓存） | 略慢 |
| **改地址后的生效速度** | **慢，最长要等满一个 TTL** | **快** |
| 典型用途 | 稳定的 `NS`、`MX` 记录 | **CDN、故障切换、GSLB** |

**为什么 CDN 的 TTL 只有几秒**：CDN 靠 DNS 做**节点调度**——它要按你的位置、运营商、节点负载和健康状态，每次给出可能不同的一组 IP。**TTL 必须短，调度才有意义。**

**★ 坑：min-TTL 覆盖（缓存下限）★**

很多 DNS 缓存/转发设备——dnsmasq、AdGuard，以及路由器和防火墙上的 DNS proxy——都提供一个"最短缓存时间"参数，用途是减少上游查询、加快解析：

```
dnsmasq   : --min-cache-ttl=3600
AdGuard   : cache_min_ttl
部分设备  : cache-min-ttl / TTL override
```

出发点是好的，但它会**直接废掉 CDN 的秒级调度机制**：

```
   权威说：这组 IP 你只能记 1 秒
   设备做：我记 3600 秒
        │
        ▼
   这一小时内，无论 CDN 怎么调度、节点是否已下线，
   全网用户拿到的都是同一组被冻住的 IP
        │
        ▼
   ★ 一旦冻住的那组节点下线 → 全公司对该站点白屏，
     直到缓存到期才"自己好了" ★
```

**症状特征（最快的识别点）**：

- **主站能打开，但页面白屏或卡死**——因为 CSS / JS / 字体通常托管在**独立的 CDN 域名**上，主站域名正常不代表资源域名正常
- 现象**时好时坏、找不到规律**，间歇周期恰好等于设备的缓存时长
- 换用公共 DNS 立刻正常，切回内网 DNS 立刻复现

**一招定性——比 TTL 量级**：

```bash
# 内网 DNS 与公共 DNS 问同一个域名，只看 TTL
$ dig @<内网DNS>  cdn.example.com +noall +answer
cdn.example.com.   16722   IN   A   ...      ← 上万秒
$ dig @223.5.5.5   cdn.example.com +noall +answer
cdn.example.com.      54   IN   A   ...      ← 权威只给 54 秒

# 放大 310 倍 → 命中 min-TTL 覆盖
```

> 完整的真实案例（含 A/B 验证、TTL 倒数过程、自愈与复发）见
> [排障方法论 · 真实故障案例集](../05-排障方法论/04-真实故障案例集.md) **案例 13**。

**⚠️ 不要用 hosts 写死 CDN 的 IP**——CDN 节点本就轮换，写死等于把当前这组**永久**冻住。节点一下线，故障就从"间歇"变成"永久"，而且更难定位。

**常见记录类型**：

| 类型 | 作用 | 示例 |
|:--|:--|:--|
| **A** | 域名 → IPv4 | `www.example.com → 93.184.216.34` |
| **AAAA** | 域名 → IPv6 | `www.example.com → 2606:2800:220:1::` |
| **CNAME** | 域名 → 别名 | `www → example.com` |
| **MX** | 邮件服务器 | `example.com → mail.example.com (优先级 10)` |
| **NS** | 域名服务器 | `example.com → ns1.example.com` |
| **PTR** | IP → 域名（反向解析） | `34.216.184.93.in-addr.arpa → www.example.com` |
| **TXT** | 任意文本 | SPF、DKIM、域名验证 |
| **SRV** | 服务定位 | AD 域控发现、SIP |

> **网工特别关注 PTR 和 SRV**：
> - **PTR**：很多邮件服务器会做反向解析校验，没有 PTR 记录的 IP 发的邮件容易被判为垃圾邮件。
> - **SRV**：Windows AD 域完全依赖 SRV 记录来定位域控。DNS 配错，整个域就瘫了。

**TCP vs UDP（回顾 [基础篇第 5 章](../00-基础篇/05-TCP-UDP与常见应用协议.md)）**：
- 普通查询：**UDP 53**
- 响应 > 512 字节、区域传送（AXFR）：**TCP 53**
- **ACL 必须两个都放行**

**AAAA 记录导致的超时问题（实战高频）**：

现代系统查域名时会**同时发 A 和 AAAA 查询**。如果网络不支持 IPv6 或 DNS 服务器不响应 AAAA，客户端会等待超时（通常 5 秒）才降级用 IPv4。

**症状**：所有网络访问都慢 5 秒左右，但一旦连上就正常。

**排查**：
```bash
dig A www.example.com      # 快
dig AAAA www.example.com   # 超时？
```

**解决**：在 DNS 服务器或容器/主机层面禁用 AAAA 查询，或修复 IPv6 解析。

> 这个坑我在 n8n 容器里踩过——容器内 DNS 查 AAAA 超时导致 IMAP 连接每次都要多等 5 秒，最后靠给容器加 `dns` 配置解决。**Docker 环境尤其容易中招。**

### 2.6 NTP

**为什么网络设备的时间同步至关重要**：

| 场景 | 时间不准的后果 |
|:--|:--|
| **日志分析** | 多台设备日志时间戳对不上，故障根本无法关联分析 |
| **证书验证** | 时间偏差过大 → 证书"尚未生效"或"已过期" → HTTPS/802.1X 全挂 |
| **Kerberos 认证** | 默认容忍 5 分钟偏差，超了直接认证失败 → **AD 域全线登录失败** |
| **计费/审计** | 时间戳不可信，审计报告无效 |
| **路由协议** | 某些认证机制依赖时间 |

**NTP 层级（Stratum）**：

```
Stratum 0 —— 原子钟 / GPS（参考时钟本身，不上网）
    ↓
Stratum 1 —— 直连 Stratum 0 的服务器（最权威的网络时间源）
    ↓
Stratum 2 —— 从 Stratum 1 同步
    ↓
Stratum 3 —— 从 Stratum 2 同步
    ↓
   ...   最多到 Stratum 15，16 = 未同步
```

**数字越小越权威。企业网络通常在 Stratum 3–4 就足够了。**

**架构建议**：

```
   [公网 NTP 池 / GPS 时钟]
            │
    ┌───────▼────────┐
    │  核心路由器/服务器 │ ← 作为内网 NTP 主服务器（Stratum 2-3）
    └───────┬────────┘
            │
    ┌───────▼────────────────────────┐
    │  所有交换机、路由器、服务器、终端   │ ← 都指向内网 NTP 服务器
    └────────────────────────────────┘
```

**不要让每台设备都去连公网 NTP**：既浪费带宽，又可能因为出口故障导致全网时间失步，还有安全风险。

**推荐的 NTP 源（国内）**：
```
ntp.aliyun.com
ntp.tencent.com
cn.pool.ntp.org
203.107.6.88      (阿里云)
```

---

## ③ 配置命令

### DHCP 服务器（路由器充当）

```cisco
! 排除不分配的地址（网关、服务器、打印机等静态地址）
R1(config)# ip dhcp excluded-address 192.168.10.1 192.168.10.20
R1(config)# ip dhcp excluded-address 192.168.10.250 192.168.10.254

! 定义地址池
R1(config)# ip dhcp pool VLAN10-OFFICE
R1(dhcp-config)#  network 192.168.10.0 255.255.255.0
R1(dhcp-config)#  default-router 192.168.10.1
R1(dhcp-config)#  dns-server 10.0.0.53 10.0.0.54
R1(dhcp-config)#  domain-name corp.example.com
R1(dhcp-config)#  lease 0 8 0                          ! 天 时 分 → 8 小时
R1(dhcp-config)#  option 150 ip 10.0.0.60              ! TFTP 服务器（Cisco IP 电话用）
R1(dhcp-config)#  option 42 ip 10.0.0.61               ! NTP 服务器
R1(dhcp-config)#  option 43 hex ...                    ! 无线 AP 找 WLC 用

! 静态绑定（给特定 MAC 固定 IP）
R1(config)# ip dhcp pool PRINTER-01
R1(dhcp-config)#  host 192.168.10.200 255.255.255.0
R1(dhcp-config)#  hardware-address 001a.2b3c.4d5e
R1(dhcp-config)#  default-router 192.168.10.1

! 查看
R1# show ip dhcp pool
R1# show ip dhcp binding
R1# show ip dhcp conflict                     ! 地址冲突记录
R1# show ip dhcp server statistics
R1# clear ip dhcp binding *
R1# debug ip dhcp server events
```

### DHCP 中继

```cisco
R1(config)# interface Vlan10
R1(config-if)# ip helper-address 10.0.0.53
R1(config-if)# ip helper-address 10.0.0.54     ! 可配多个，冗余

! 精简转发的协议
R1(config)# no ip forward-protocol udp 137
R1(config)# no ip forward-protocol udp 138

! 验证
R1# show ip interface Vlan10 | include Helper
  Helper address is 10.0.0.53
```

### DHCP 客户端（路由器从上游拿地址）

```cisco
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip address dhcp
R1# show dhcp lease
```

### DNS

```cisco
! 路由器作为 DNS 客户端
R1(config)# ip domain-lookup
R1(config)# ip name-server 10.0.0.53 8.8.8.8
R1(config)# ip domain-name corp.example.com

! ⚠️ 实验环境建议关闭，避免敲错命令时卡 30 秒
R1(config)# no ip domain-lookup

! 静态解析（相当于 hosts 文件）
R1(config)# ip host CORE-SW1 10.0.0.1
R1(config)# ip host WEB-SERVER 10.0.0.100

! 路由器作为简易 DNS 服务器（小型网络）
R1(config)# ip dns server

! 验证
R1# ping CORE-SW1
R1# show hosts
```

### NTP

```cisco
! ── 客户端（大多数设备）──
R1(config)# ntp server 10.0.0.61
R1(config)# ntp server 10.0.0.62 prefer          ! 优先使用
R1(config)# clock timezone CST 8                 ! 中国标准时间 UTC+8
R1(config)# service timestamps log datetime msec localtime show-timezone
R1(config)# service timestamps debug datetime msec localtime show-timezone

! ── 服务器（内网 NTP 主机）──
R1(config)# ntp master 3                         ! 声称自己是 Stratum 3
R1(config)# ntp server ntp.aliyun.com            ! 同时从上游同步

! ── 认证（生产环境推荐）──
R1(config)# ntp authenticate
R1(config)# ntp authentication-key 1 md5 NtpSecretKey
R1(config)# ntp trusted-key 1
R1(config)# ntp server 10.0.0.61 key 1

! ── 限制谁能查询（安全）──
R1(config)# access-list 20 permit 10.0.0.0 0.255.255.255
R1(config)# ntp access-group peer 20

! ── 验证 ──
R1# show ntp status
R1# show ntp associations
R1# show ntp associations detail
R1# show clock detail
```

**`show ntp status` 输出解读**：
```cisco
R1# show ntp status
Clock is synchronized, stratum 4, reference is 10.0.0.61
      ↑ 关键！必须是 synchronized
nominal freq is 250.0000 Hz, actual freq is 249.9990 Hz, precision is 2**18
reference time is E5A2B3C4.12345678 (10:23:45.071 CST Sat Aug 30 2026)
clock offset is 1.2345 msec, root delay is 15.23 msec
                    ↑ 与参考源的偏差，应该很小
```

**`show ntp associations` 输出解读**：
```cisco
R1# show ntp associations
  address         ref clock       st   when   poll reach  delay  offset   disp
*~10.0.0.61       203.107.6.88     3     45     64   377  1.234   0.567  0.123
+~10.0.0.62       203.107.6.88     3     52     64   377  2.345   0.789  0.234
 ~10.0.0.63       .INIT.          16      -   1024     0  0.000   0.000 16000
 ↑                                 ↑                  ↑
标志                             stratum            reach
```

| 标志 | 含义 |
|:--|:--|
| **`*`** | **当前同步的主时钟源** ✅ |
| `+` | 候选源（备选） |
| `-` | 被算法排除的源 |
| `x` | 被判定为"假时钟"（falseticker） |
| （空） | 不可用 |

**`reach` 字段（八进制）**：记录最近 8 次轮询的成功情况。
- **`377`（八进制）= 11111111（二进制）= 最近 8 次全部成功** ✅
- `0` = 一次都没成功 ❌
- `1` = 只有最近一次成功（刚开始同步）

> **排障要点**：`stratum 16` + `.INIT.` + `reach 0` 三者同时出现 = **完全联系不上这个 NTP 服务器**。查网络可达性和 UDP 123 是否被拦。

### 三厂商对照

| 目的 | Cisco | H3C | 华为 |
|:--|:--|:--|:--|
| DHCP 地址池 | `ip dhcp pool NAME` | `dhcp server ip-pool NAME` | `ip pool NAME` |
| 网段 | `network 192.168.10.0 255.255.255.0` | `network 192.168.10.0 mask 255.255.255.0` | `network 192.168.10.0 mask 24` |
| 网关 | `default-router 192.168.10.1` | `gateway-list 192.168.10.1` | `gateway-list 192.168.10.1` |
| DNS | `dns-server 10.0.0.53` | `dns-list 10.0.0.53` | `dns-list 10.0.0.53` |
| 排除地址 | `ip dhcp excluded-address ...` | `forbidden-ip ...` | `excluded-ip-address ...` |
| 开启 DHCP | 默认开启 | `dhcp enable` | `dhcp enable` |
| DHCP 中继 | `ip helper-address 10.0.0.53` | `dhcp relay server-address 10.0.0.53` | `dhcp relay server-ip 10.0.0.53` |
| DHCP Snooping | `ip dhcp snooping` | `dhcp snooping enable` | `dhcp snooping enable` |
| 信任口 | `ip dhcp snooping trust` | `dhcp snooping trust` | `dhcp snooping trusted` |
| 启用域名解析 | `ip domain-lookup` | `dns resolve` | `dns resolve` |
| 指定 DNS 服务器 | `ip name-server 10.0.0.53` | `dns server 10.0.0.53` | `dns server 10.0.0.53` |
| DNS 代理 / 缓存 | `ip dns server` | `dns proxy enable` | `dns proxy enable` |
| **查看解析缓存** | `show hosts` | `display dns host` | `display dns dynamic-host` |
| **清除解析缓存** | `clear host *` | `reset dns host` | `reset dns dynamic-host` |
| NTP 客户端 | `ntp server 10.0.0.61` | `ntp-service unicast-server 10.0.0.61` | `ntp-service unicast-server 10.0.0.61` |
| NTP 服务端 | `ntp master 3` | `ntp-service refclock-master 3` | `ntp-service refclock-master 3` |
| 时区 | `clock timezone CST 8` | `clock timezone CST add 8` | `clock timezone CST add 08:00:00` |
| 查 NTP | `show ntp status` | `display ntp-service status` | `display ntp-service status` |

---

## ④ 配套实验：跨网段 DHCP + Snooping

**拓扑**：
```
   VLAN10 (192.168.10.0/24)      VLAN20 (192.168.20.0/24)
        PC1                            PC2
         │                              │
      Gi1/0/1                        Gi1/0/2
         └──────────┬───────────────────┘
              ┌─────┴──────┐
              │    SW1     │ (三层交换机)
              │  SVI 网关   │
              └─────┬──────┘
                 Gi1/0/24
                    │
              [DHCP 服务器 10.0.0.53]  (VLAN 100)
```

### Step 1：SW1 配置 SVI 和中继

```cisco
SW1(config)# ip routing

SW1(config)# vlan 10,20,100

SW1(config)# interface Vlan10
SW1(config-if)# ip address 192.168.10.1 255.255.255.0
SW1(config-if)# ip helper-address 10.0.0.53           ! ★ 关键
SW1(config-if)# no shutdown

SW1(config)# interface Vlan20
SW1(config-if)# ip address 192.168.20.1 255.255.255.0
SW1(config-if)# ip helper-address 10.0.0.53           ! ★ 每个 SVI 都要配
SW1(config-if)# no shutdown

SW1(config)# interface Vlan100
SW1(config-if)# ip address 10.0.0.1 255.255.255.0
SW1(config-if)# no shutdown
```

### Step 2：DHCP 服务器配置（这里用路由器模拟）

```cisco
DHCP-SRV(config)# ip dhcp excluded-address 192.168.10.1 192.168.10.20
DHCP-SRV(config)# ip dhcp excluded-address 192.168.20.1 192.168.20.20

DHCP-SRV(config)# ip dhcp pool VLAN10
DHCP-SRV(dhcp-config)#  network 192.168.10.0 255.255.255.0
DHCP-SRV(dhcp-config)#  default-router 192.168.10.1
DHCP-SRV(dhcp-config)#  dns-server 10.0.0.53
DHCP-SRV(dhcp-config)#  lease 0 8 0

DHCP-SRV(config)# ip dhcp pool VLAN20
DHCP-SRV(dhcp-config)#  network 192.168.20.0 255.255.255.0
DHCP-SRV(dhcp-config)#  default-router 192.168.20.1
DHCP-SRV(dhcp-config)#  dns-server 10.0.0.53
DHCP-SRV(dhcp-config)#  lease 0 8 0

! ★ 关键：DHCP 服务器必须有回程路由，否则 OFFER 发不回去
DHCP-SRV(config)# ip route 192.168.0.0 255.255.0.0 10.0.0.1
```

### Step 3：验证

```cisco
! PC1 上执行 ipconfig /renew（或 dhclient）

DHCP-SRV# show ip dhcp binding
IP address       Client-ID/MAC        Lease expiration        Type
192.168.10.21    0100.5056.aabb.cc    Aug 31 2026 06:23 PM    Automatic
192.168.20.21    0100.5056.ddee.ff    Aug 31 2026 06:24 PM    Automatic
```

**在中继上抓包**，你会看到：
```
方向：SW1 → DHCP-SRV
源 IP: 192.168.10.1       ← 中继接口地址
目的 IP: 10.0.0.53
giaddr: 192.168.10.1      ← ★ 这就是服务器判断该分哪个池的依据
```

### Step 4：故障注入

**故障 A：忘配 helper-address**
```cisco
SW1(config)# interface Vlan10
SW1(config-if)# no ip helper-address 10.0.0.53
```
**症状**：PC1 拿到 `169.254.x.x`。VLAN20 正常。
**排查**：
```cisco
SW1# show ip interface Vlan10 | include Helper
! 无输出 → 找到问题
SW1# show ip interface Vlan20 | include Helper
  Helper address is 10.0.0.53          ← 对比可见差异
```

**故障 B：DHCP 服务器缺回程路由**
```cisco
DHCP-SRV(config)# no ip route 192.168.0.0 255.255.0.0 10.0.0.1
```
**症状**：所有 VLAN 都拿不到 IP。
**排查**：在 DHCP 服务器上抓包，能看到 DISCOVER 进来，但 OFFER 发不出去。
```cisco
DHCP-SRV# debug ip dhcp server events
DHCPD: DHCPDISCOVER received from client on interface Gi0/0.
DHCPD: Sending DHCPOFFER to client (192.168.10.1).
DHCPD: no route to 192.168.10.1                      ← 明确的错误信息
```

> **这个故障很有教育意义**：DHCP 是双向通信，服务器必须**能路由回中继的地址**。很多人只配了去程路由，忘了 DHCP 服务器侧也要有回程。

**故障 C：地址池耗尽**
```cisco
DHCP-SRV(config)# ip dhcp excluded-address 192.168.10.1 192.168.10.253
! 只剩 254 一个地址
```
**症状**：第二台机器拿不到 IP。
**排查**：
```cisco
DHCP-SRV# show ip dhcp pool
Pool VLAN10 :
 Utilization mark (high/low)    : 100 / 0
 Subnet size (first/next)       : 0 / 0
 Total addresses                : 254
 Leased addresses               : 1
 Excluded addresses             : 253
 Pending event                  : none
                                       ↑ 可用地址几乎为 0
```

### Step 5：DHCP Snooping 实验

```cisco
SW1(config)# ip dhcp snooping
SW1(config)# ip dhcp snooping vlan 10,20

! 上联口（通往真正的 DHCP 服务器）设为信任
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# ip dhcp snooping trust

! 接终端的口限速
SW1(config)# interface range GigabitEthernet1/0/1-20
SW1(config-if-range)# ip dhcp snooping limit rate 10
```

**模拟攻击**：在 Gi1/0/3 接一台配置了 DHCP 服务的路由器（假冒 DHCP 服务器）。

**预期**：它发的 OFFER 会被交换机丢弃，日志报：
```
%DHCP_SNOOPING-5-DHCP_SNOOPING_UNTRUSTED_PORT: DHCP_SNOOPING drop message on 
untrusted port, message type: DHCPOFFER, MAC sa: 001a.2b3c.4d5e
```

**查看绑定表**：
```cisco
SW1# show ip dhcp snooping binding
MacAddress          IpAddress        Lease(sec)  Type           VLAN  Interface
------------------  ---------------  ----------  -------------  ----  --------------
00:50:56:AA:BB:CC   192.168.10.21    28234       dhcp-snooping  10    Gi1/0/1
00:50:56:DD:EE:FF   192.168.20.21    28245       dhcp-snooping  20    Gi1/0/2
```

**这张表就是后续 DAI 和 IPSG 的数据基础。**

### Step 6：NTP 实验

```cisco
! SW1 作为内网 NTP 服务器
SW1(config)# clock set 10:00:00 30 Aug 2026
SW1(config)# clock timezone CST 8
SW1(config)# ntp master 3

! 其他设备作为客户端
R2(config)# clock timezone CST 8
R2(config)# ntp server 10.0.0.1
R2(config)# service timestamps log datetime msec localtime show-timezone
```

**验证（注意 NTP 同步需要几分钟）**：
```cisco
R2# show ntp status
Clock is synchronized, stratum 4, reference is 10.0.0.1     ← 等到出现 synchronized

R2# show ntp associations
  address         ref clock       st   when   poll reach  delay  offset   disp
*~10.0.0.1        127.127.1.1      3     23     64   377  1.234   0.567  0.123
 ↑ 星号表示已同步                                        ↑ 377 = 8次全成功
```

> **耐心提示**：NTP 首次同步通常需要 **5–15 分钟**（要多次采样才能确认时钟稳定）。刚配完就 `show ntp status` 看到 `unsynchronized` 是正常的，别急着排障。

---

## ⑤ 排障思路

### DHCP

| 症状 | 检查 | 命令 |
|:--|:--|:--|
| 拿到 `169.254.x.x` | 端口 VLAN | `show interfaces Gi1/0/5 switchport` |
| | PortFast | `show spanning-tree interface Gi1/0/5 portfast` |
| | helper-address | `show ip interface Vlan10 \| inc Helper` |
| | 地址池 | `show ip dhcp pool` |
| | ACL | `show access-lists` |
| | Snooping | `show ip dhcp snooping` |
| 拿到了但是错误网段的 IP | giaddr / 地址池匹配 | 抓包看 giaddr，对比服务器池配置 |
| 部分用户拿不到 | 地址池耗尽 | `show ip dhcp pool` 看 Leased/Total |
| 拿到 IP 但上不了网 | 网关/DNS 选项 | `show ip dhcp pool` 看 default-router |
| 地址冲突 | 有人静态配了池内地址 | `show ip dhcp conflict` |
| DHCP 时通时不通 | 有恶意 DHCP 服务器 | 开 DHCP Snooping，看日志 |

### DNS

| 症状 | 检查 | 命令 |
|:--|:--|:--|
| ping IP 通、ping 域名不通 | DNS 配置 | `nslookup`、`dig` |
| 部分域名解析失败 | **TCP 53 被拦** | `dig +tcp <域名>` |
| 所有访问慢 5 秒 | **AAAA 查询超时** | `dig AAAA <域名>` |
| **主站能开、页面白屏** | **min-TTL 覆盖冻住了 CDN 节点** | 内网 / 公共 DNS **比 TTL 量级** |
| **时好时坏、找不到规律** | 缓存周期 = 间歇周期 | 连查同一记录看 **TTL 是否在倒数** |
| 内网域名解析不了 | 搜索域 / 内网 DNS | `show hosts`、检查 domain-name |
| 设备敲错命令卡 30 秒 | `ip domain-lookup` 开着 | `no ip domain-lookup` |

### NTP

| 症状 | 检查 | 命令 |
|:--|:--|:--|
| `unsynchronized` | 可达性、UDP 123 | `ping <ntp服务器>`、`show ntp associations` |
| `stratum 16` + `.INIT.` | 完全联系不上 | 查路由、ACL |
| `reach 0` | 报文没到达 | 抓包 UDP 123 |
| 同步了但时间还是错 | 时区 | `show clock detail`、`clock timezone` |
| 日志时间戳没有时区 | timestamps 配置 | `service timestamps log datetime localtime show-timezone` |
| 证书报"尚未生效" | 系统时间偏差大 | `show clock`，先手工 `clock set` 再让 NTP 慢慢校准 |

> **NTP 的一个特性**：如果本地时间和 NTP 源偏差**超过 1000 秒**，NTP 会**拒绝同步**（认为可能是攻击或故障）。这时必须先手工 `clock set` 把时间调到大致正确，NTP 才能接管。
>
> ```cisco
> R1# clock set 14:30:00 30 Aug 2026
> R1(config)# ntp server 10.0.0.61
> ```

---

## ⑥ 考点提示 + 自测题

### 考点

- **DORA 四步**及各步是广播还是单播。
- **`ip helper-address` 和 `giaddr` 字段的作用**。
- **`169.254.x.x` = DHCP 失败**。
- **DHCP Snooping 的 trust/untrust 概念**，以及它与 DAI、IPSG 的关系。
- **DNS 需要 TCP+UDP 53 都放行**。
- **`TTL` 有两个含义**：IP 头 TTL 是**跳数**，DNS TTL 是**秒**。别混。
- **DNS TTL 的权衡**：长 = 查询少但改地址生效慢；短 = 调度灵活，CDN 必须用短 TTL。
- **min-TTL 覆盖会废掉 CDN 调度**，典型症状是"主站正常但页面白屏、时好时坏"。
- **NTP Stratum 层级、`reach 377` 的含义**。
- **NTP 偏差 > 1000 秒会拒绝同步**。

### 自测题

**1.** DHCP 客户端和服务器不在同一网段，需要配什么？`giaddr` 字段起什么作用？

<details><summary>答案</summary>

**需要在客户端所在网段的网关接口上配置 DHCP 中继**：
```cisco
R1(config)# interface Vlan10
R1(config-if)# ip helper-address 10.0.0.53
```

**为什么需要**：DHCP DISCOVER 是**广播**（目的 `255.255.255.255`），路由器默认**不转发广播**，所以请求到不了跨网段的服务器。

**`giaddr`（Gateway IP Address）字段的两个作用**：

1. **告诉服务器该从哪个地址池分配**
   中继在转发时，把**接收该请求的接口 IP** 填进 giaddr。服务器看到 `giaddr = 192.168.10.1`，就知道这个客户端在 `192.168.10.0/24` 网段，应该从对应的池里分地址。
   
   → **没有 giaddr，服务器根本不知道该分哪个网段的地址。**

2. **告诉服务器往哪里回包**
   服务器把 OFFER **单播**发给 giaddr（也就是中继），中继再把它广播给客户端。

**完整流程**：
```
客户端 ──广播 DISCOVER──> 中继(网关)
                            │ 改成单播，填 giaddr=192.168.10.1
                            ▼
                        DHCP 服务器
                            │ 根据 giaddr 选池，OFFER 单播回 192.168.10.1
                            ▼
                         中继 ──广播 OFFER──> 客户端
```

**两个容易忽略的配套要求**：

1. **DHCP 服务器必须有回到 giaddr 的路由**。否则 OFFER 发不回去。这是跨网段 DHCP 的第二大故障原因。

2. **如果 SVI 有多个 IP（主 + secondary），giaddr 用的是主地址**。所以服务器上的地址池必须匹配**主地址**所在网段。
</details>

**2.** 用户电脑显示 IP 是 `169.254.100.50`。这说明什么？按什么顺序排查？

<details><summary>答案</summary>

**`169.254.x.x` 是 APIPA（Automatic Private IP Addressing，RFC 3927）**，操作系统在**完全没有收到任何 DHCP 响应**时自动生成的链路本地地址。

**这个信号非常有价值**：它直接告诉你"网络层以上的东西不用查了，问题在 DHCP 链路上"，能省掉大量无用的排查。

**排查顺序（从近到远）**：

**① 端口在正确的 VLAN 吗？**
```cisco
SW1# show interfaces GigabitEthernet1/0/5 switchport
Administrative Mode: static access
Access Mode VLAN: 1 (default)          ← 应该是 10，配错了！
```
这是最常见的原因，尤其是新员工工位、办公室搬迁后。

**② 端口配了 PortFast 吗？**
```cisco
SW1# show spanning-tree interface Gi1/0/5 portfast
```
没配 PortFast 的话，端口插线后要经过 **Listening (15s) + Learning (15s) = 30 秒**才转发。Windows 的 DHCP 客户端在这期间已经放弃了，直接用 APIPA。

**③ 网关配了 helper-address 吗？**（跨网段场景）
```cisco
SW1# show ip interface Vlan10 | include Helper
```

**④ DHCP 服务器地址池还有空闲吗？**
```cisco
DHCP-SRV# show ip dhcp pool
Pool VLAN10 :
 Total addresses     : 254
 Leased addresses    : 254          ← 满了
```

**⑤ 路径上有 ACL 拦了 UDP 67/68 吗？**
```cisco
R1# show access-lists | include 67|68|bootp
```

**⑥ DHCP Snooping 把端口当成不可信了吗？**
```cisco
SW1# show ip dhcp snooping
SW1# show logging | include DHCP_SNOOPING
```

**⑦ 有恶意 DHCP 服务器抢答了吗？**
如果拿到的是奇怪网段的 IP（而不是 169.254），要怀疑这个。

**实战口诀：看到 `169.254`，先查 VLAN，再查 helper，最后查地址池。** 这三项覆盖 90% 的情况。
</details>

**3.** DHCP Snooping 的 trust 口应该配在哪里？如果配反了会怎样？

<details><summary>答案</summary>

**trust 口应该配在"通往合法 DHCP 服务器方向"的端口上**——通常是：
- 直连 DHCP 服务器的端口
- 上联到核心/汇聚交换机的 Trunk 口（因为 DHCP 服务器在上游）

**所有接终端的接入口保持默认的 untrust。**

```cisco
SW1(config)# ip dhcp snooping
SW1(config)# ip dhcp snooping vlan 10,20,30

! 上联口设为信任
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# ip dhcp snooping trust

! 接入口保持 untrust（默认），加限速
SW1(config)# interface range GigabitEthernet1/0/1-20
SW1(config-if-range)# ip dhcp snooping limit rate 10
```

**trust 与 untrust 的行为差异**：

| 报文类型 | trust 口 | untrust 口 |
|:--|:--|:--|
| DISCOVER / REQUEST（客户端发的） | ✅ 允许 | ✅ 允许 |
| **OFFER / ACK / NAK（服务器发的）** | ✅ 允许 | ❌ **丢弃** |

**配反的后果（两种）**：

**情况 A：把上联口配成 untrust（或者忘了配 trust）**
→ 合法 DHCP 服务器的 OFFER/ACK 被丢弃
→ **全网所有客户端都拿不到 IP**
→ 这是配置 DHCP Snooping 时最常见的翻车方式，**会造成大面积断网**

**情况 B：把接入口配成 trust**
→ 该端口下的恶意 DHCP 服务器可以正常工作
→ **防护形同虚设**，攻击者可以做中间人

**部署 DHCP Snooping 的安全流程**：
1. **先在测试 VLAN 上开启**，验证无误
2. 确认所有上联口和 DHCP 服务器直连口都配了 `trust`
3. 用 `show ip dhcp snooping` 核对 trust 口列表
4. 再逐个 VLAN 推广
5. **千万不要一上来就 `ip dhcp snooping vlan 1-4094`**

```cisco
SW1# show ip dhcp snooping
Switch DHCP snooping is enabled
DHCP snooping is configured on following VLANs: 10,20,30
Insertion of option 82 is enabled
Interface                  Trusted    Rate limit (pps)
------------------------   -------    ----------------
GigabitEthernet1/0/24      yes        unlimited          ← 确认这里
GigabitEthernet1/0/1       no         10
```
</details>

**4.** 为什么网络设备的 NTP 同步对故障排查至关重要？

<details><summary>答案</summary>

**核心理由：没有统一的时间基准，多设备的日志就无法关联，故障根因分析基本不可能。**

**场景举例**：某天上午业务中断 3 分钟。你要还原发生了什么：

```
核心交换机日志：  09:15:23  Interface Gi1/0/1 down
汇聚交换机日志：  09:12:47  STP topology change              ← 时间对不上
出口路由器日志：  09:20:11  BGP neighbor down                ← 时间也对不上
防火墙日志：      09:14:02  High CPU utilization
```

**时间不同步的话，你根本无法判断因果关系**——到底是接口先 down 导致 STP 收敛，还是 STP 震荡导致接口 flap？没有可信的时间线，只能靠猜。

**其他严重后果**：

| 场景 | 后果 |
|:--|:--|
| **证书验证** | 时间偏差 → HTTPS 报"证书尚未生效"、802.1X 认证失败、VPN 建不起来 |
| **Kerberos / AD 域** | 默认只容忍 **5 分钟**偏差，超了直接认证失败 → **全公司登录不了** |
| **日志审计合规** | 时间戳不可信，审计报告和取证材料无效 |
| **SIEM 关联分析** | 安全事件无法按时间线关联，攻击链还原不出来 |
| **计费与 SLA** | 流量统计、可用性计算全部失真 |
| **备份与快照** | 恢复到错误的时间点 |

**最佳实践**：

```cisco
! 1. 所有设备统一时区
R1(config)# clock timezone CST 8

! 2. 日志时间戳带毫秒和时区
R1(config)# service timestamps log datetime msec localtime show-timezone
R1(config)# service timestamps debug datetime msec localtime show-timezone

! 3. 指向内网 NTP 服务器（至少两个，冗余）
R1(config)# ntp server 10.0.0.61 prefer
R1(config)# ntp server 10.0.0.62

! 4. 启用认证，防 NTP 欺骗
R1(config)# ntp authenticate
R1(config)# ntp authentication-key 1 md5 NtpSecretKey
R1(config)# ntp trusted-key 1
R1(config)# ntp server 10.0.0.61 key 1
```

**架构建议**：
- 内网建 **2 台 NTP 服务器**，它们从公网 NTP 池或 GPS 时钟同步
- 所有其他设备只指向这两台内网服务器
- **不要让每台设备都去连公网**：浪费带宽、出口故障会拖垮全网时间、有安全风险

**验证清单**：
```cisco
R1# show ntp status                    ! 必须是 "Clock is synchronized"
R1# show ntp associations              ! 必须有 * 标记的源，reach 应为 377
R1# show clock detail                  ! 确认时区和时间正确
```
</details>

**5.** 用户反馈"网络访问都要卡 5 秒才响应，连上之后就正常"。ping 网关和 DNS 服务器都很快。可能是什么问题？

<details><summary>答案</summary>

**最可能是 AAAA（IPv6）DNS 查询超时。**

**原理**：现代操作系统和应用（浏览器、curl、各种客户端库）在解析域名时会**同时发起 A 和 AAAA 两个查询**（这叫 Happy Eyeballs 机制的前置步骤）。

如果：
- DNS 服务器不响应 AAAA 查询（而不是明确返回"无记录"）
- 或者路径上有设备把 AAAA 响应丢弃了

客户端会**等待超时（通常 5 秒）**，然后才降级使用 IPv4 的 A 记录结果。

**症状特征（很好识别）**：
- 延迟固定在 5 秒左右（而不是随机的）
- 连接建立后速度完全正常
- ping IP 地址很快，ping 域名慢
- 所有应用都受影响

**验证**：
```bash
# 分别测试 A 和 AAAA
dig A www.example.com        # 几毫秒返回
dig AAAA www.example.com     # 卡住 5 秒 → 找到问题

# 或者对比
time nslookup -type=A www.example.com
time nslookup -type=AAAA www.example.com
```

**解决方案（按场景）**：

| 场景 | 方案 |
|:--|:--|
| DNS 服务器不支持 IPv6 | 让它对 AAAA 查询**明确返回 NOERROR + 空答案**，而不是不响应 |
| 防火墙拦了 AAAA 响应 | 放行，或让 DNS 服务器返回明确的否定响应 |
| Docker 容器内 | 给容器配置 `dns` 和 `dns_opt: ["single-request-reopen"]` |
| Linux 主机 | `/etc/resolv.conf` 加 `options single-request-reopen` |
| 彻底禁用 IPv6 解析 | Linux: `sysctl -w net.ipv6.conf.all.disable_ipv6=1`（治标） |

**其他可能的 5 秒延迟原因**（供对照排查）：

| 原因 | 特征 |
|:--|:--|
| 主 DNS 不响应，等超时后用备用 DNS | `dig @主DNS` 直接测试 |
| 反向 DNS 查询超时（SSH 登录慢） | SSH 服务端配 `UseDNS no` |
| LDAP/AD 认证超时 | 检查域控可达性 |
| 代理自动配置（PAC）文件加载慢 | 浏览器代理设置 |
| IPv6 路由存在但不通（Happy Eyeballs 失效） | `ping6` 测试 |

> **实战经验**：这类"固定延迟"问题的排查关键是**注意延迟的规律性**。随机延迟通常是网络拥塞或丢包；**固定的 5 秒、10 秒、30 秒延迟几乎总是某个超时定时器**——顺着"哪个协议的默认超时是这个数"去查，往往一击即中。
</details>

**6.** 面试官问："`TTL` 是什么？"你会怎么答？另外：某网站主站能打开但页面一直白屏，换成 `223.5.5.5` 就正常，切回公司 DNS 又坏，过几小时它自己好了。根因最可能是什么？

<details><summary>答案</summary>

**第一问——先反问"哪个 TTL"，这就是考点。**

| | **IP 头 TTL** | **DNS TTL** |
|:--|:--|:--|
| 单位 | **跳数** | **秒** |
| 谁减它 | 每台路由器减 1（三层交换机做路由时也减） | 不递减，是缓存有效期 |
| 归零 | 丢包 + 回 ICMP Time Exceeded | 缓存失效，重新查询 |
| 用途 | 防环、`traceroute` | 控制解析结果能缓存多久 |

一句话答法："TTL 在 IP 头里是**跳数限制**，用来防环，也是 traceroute 的原理；在 DNS 里是**缓存秒数**，决定解析结果能被缓存多久。两者只是重名。"

**第二问——根因：内网 DNS 开了 min-TTL 覆盖，冻住了一组已失效的 CDN 节点。**

推导链：

```
   ① 主站能开、页面白屏
      → 主站域名正常，挂的是承载 CSS/JS/字体的独立 CDN 域名
      → 这是"资源域"故障，不是"带宽慢"

   ② 换公共 DNS 就好、切回就坏
      → 唯一变量是"谁给的地址"
      → 排除防火墙、链路、带宽、网站本身

   ③ 过几小时自己好
      → 缓存到期后重新查询，抽到了可用节点
      → ★ 不是修好了，是进入下一轮抽签 ★
```

**定性只需一条命令——比 TTL 量级**：

```bash
$ dig @<内网DNS> cdn.example.com +noall +answer
cdn.example.com.   16722   IN   A   ...     ← 上万秒
$ dig @223.5.5.5  cdn.example.com +noall +answer
cdn.example.com.      54   IN   A   ...     ← 权威只给 54 秒
```

权威给几十秒、内网给上万秒 = **命中 min-TTL 覆盖**。

**两个容易答错的点**：

1. **别答"DNS 污染/劫持"**。污染是返回**错误**的地址，这里返回的是**曾经正确、现在已下线**的地址——性质不同，且换公共 DNS 能好、过期能自愈，都不符合污染特征。

2. **别建议用 hosts 写死 IP**。CDN 节点本就轮换，写死等于把当前这组**永久**冻住，会把间歇故障变成永久故障。正确做法是**关掉设备上的 min-TTL 覆盖**。

> 完整案例见 [排障方法论 · 真实故障案例集](../05-排障方法论/04-真实故障案例集.md) **案例 13**。
</details>

---

**上一章** ← [06 NAT 地址转换](06-NAT地址转换.md) ｜ **下一章** → [08 无线与安全基础](08-无线与安全基础.md)
