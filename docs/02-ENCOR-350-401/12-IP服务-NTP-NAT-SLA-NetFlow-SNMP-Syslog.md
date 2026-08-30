# 12 · IP 服务：NTP / NAT / IP SLA / NetFlow / SNMP / Syslog

## ① 这章解决什么问题

这一章覆盖 ENCOR 考纲里的 **"网络保障（Network Assurance）"** 域（占 10%），核心是回答一个问题：

**"网络到底出了什么事，我怎么知道？"**

- 出故障了，你是**用户打电话告诉你**，还是**监控先告警**？
- 出口带宽被占满了，你知道**是谁在占**吗？
- 主备链路切换失败，你能**提前发现**吗？
- 事后复盘，你有**完整的时间线**吗？

这些工具就是网络的"眼睛"：
- **IP SLA** —— 主动探测，故障前发现
- **NetFlow** —— 看清谁在用带宽
- **SNMP** —— 性能监控与告警
- **Syslog** —— 事件记录与审计
- **NTP** —— 让上面所有数据的时间戳可信

---

## ② IP SLA：主动探测

### 2.1 为什么需要

**被动监控的局限**：`show interfaces` 只能告诉你"这个接口有没有 down"，但：
- 接口 up 但对端设备挂了？
- 链路通但延迟从 10ms 涨到 500ms？
- 丢包率从 0 涨到 5%？

**IP SLA 主动发探测包**，测量真实的端到端质量。

### 2.2 常用探测类型

| 类型 | 测什么 | 用途 |
|:--|:--|:--|
| **icmp-echo** | 可达性、RTT | ★ 最常用，配合 Track 做路由切换 |
| **udp-jitter** | **延迟、抖动、丢包** | ★ 语音/视频质量评估 |
| **udp-jitter codec** | **MOS 值**（语音质量评分） | VoIP 部署前评估 |
| **tcp-connect** | TCP 端口可达性 | 服务可用性监控 |
| **http** | HTTP 响应时间 | Web 服务监控 |
| **dns** | DNS 解析时间 | DNS 服务监控 |
| **path-echo** | 逐跳 RTT | 定位延迟在哪一跳 |

### 2.3 配置

```cisco
! ── ① ICMP 探测（最常用）──
R1(config)# ip sla 1
R1(config-ip-sla)#  icmp-echo 8.8.8.8 source-interface GigabitEthernet0/1
R1(config-ip-sla-echo)#   frequency 5                  ! 每 5 秒探一次
R1(config-ip-sla-echo)#   timeout 2000                 ! 2 秒超时
R1(config-ip-sla-echo)#   threshold 1000               ! 超过 1 秒算异常
R1(config-ip-sla-echo)#   request-data-size 64
R1(config-ip-sla-echo)#   tag "TO-INTERNET"
R1(config)# ip sla schedule 1 life forever start-time now

! ── ② UDP Jitter（语音质量评估）──
! 目的端必须开启 responder
R2(config)# ip sla responder

R1(config)# ip sla 2
R1(config-ip-sla)#  udp-jitter 10.2.1.1 16384 codec g711alaw
R1(config-ip-sla-jitter)#   frequency 30
R1(config-ip-sla-jitter)#   num-packets 1000
R1(config)# ip sla schedule 2 life forever start-time now

! ── ③ HTTP 探测 ──
R1(config)# ip sla 3
R1(config-ip-sla)#  http get http://10.1.30.100/health
R1(config-ip-sla-http)#   frequency 60
R1(config)# ip sla schedule 3 life forever start-time now

! ── ④ 绑定 Track 对象 ──
R1(config)# track 1 ip sla 1 reachability
R1(config-track)#  delay down 10 up 30                 ! ★ 防抖

R1(config)# track 2 ip sla 1 state                     ! 跟踪状态（含阈值）

! ── ⑤ 用 Track 驱动动作 ──
! 浮动静态路由
R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.11.1 track 1
R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.22.1 10

! HSRP
R1(config)# interface Vlan10
R1(config-if)# standby 10 track 1 decrement 20

! 策略路由
R1(config)# route-map PBR permit 10
R1(config-route-map)#  set ip next-hop verify-availability 10.0.11.1 1 track 1

! ── 查看 ──
R1# show ip sla configuration
R1# show ip sla statistics
R1# show ip sla statistics 1
R1# show ip sla statistics aggregated 1
R1# show track
R1# show track brief
```

**`show ip sla statistics` 输出**：
```cisco
R1# show ip sla statistics 1
IPSLAs Latest Operation Statistics

IPSLA operation id: 1
        Latest RTT: 15 milliseconds
Latest operation start time: 10:23:45 CST Sat Aug 30 2026
Latest operation return code: OK                    ← ★ OK / Timeout / NoConnection
Number of successes: 1245
Number of failures: 3
Operation time to live: Forever
```

**`return code` 的含义**：

| 值 | 含义 |
|:--|:--|
| **OK** | 探测成功 |
| **Timeout** | 超时（目标不可达或太慢） |
| No Connection | 无法建立连接（TCP/HTTP 类型） |
| Down | 探测被禁用 |

### 2.4 组合跟踪（Track List）

```cisco
! 定义多个探测
R1(config)# track 1 ip sla 1 reachability          ! 探测 8.8.8.8
R1(config)# track 2 ip sla 2 reachability          ! 探测 114.114.114.114
R1(config)# track 3 interface Gi0/1 line-protocol  ! 接口状态

! 布尔或：任一 up 就算 up（更宽容）
R1(config)# track 10 list boolean or
R1(config-track)#  object 1
R1(config-track)#  object 2

! 布尔与：全部 up 才算 up（更严格）
R1(config)# track 11 list boolean and
R1(config-track)#  object 1
R1(config-track)#  object 3

! 阈值百分比：至少 60% 的对象 up
R1(config)# track 12 list threshold percentage
R1(config-track)#  object 1
R1(config-track)#  object 2
R1(config-track)#  object 3
R1(config-track)#  threshold percentage up 60 down 40

! 权重
R1(config)# track 13 list threshold weight
R1(config-track)#  object 1 weight 30
R1(config-track)#  object 2 weight 20
R1(config-track)#  threshold weight up 40 down 20
```

> **为什么要探测多个目标**：单一目标（比如只探 8.8.8.8）不可靠——那台服务器可能自己在维护，或者被墙了。**探测 2-3 个不同运营商的目标，用布尔或组合**，才能准确反映"我的链路到底通不通"。

### 2.5 `delay down/up`（防抖，★ 实战必配）

```cisco
R1(config)# track 1 ip sla 1 reachability
R1(config-track)#  delay down 10 up 30
!                        ↑       ↑
!                  down 确认10秒  up 确认30秒
```

**为什么需要**：
- **不加防抖**：链路轻微抖动（丢一个探测包）就触发路由切换，切换本身有代价（收敛、会话中断）。**频繁切换比不切换更糟**。
- **`down 10`**：连续 10 秒不通才认为真的断了
- **`up 30`**：恢复后要稳定 30 秒才切回来（避免链路刚恢复还不稳定就切回去）

> **`up` 的延迟应该大于 `down`**。故障要快速响应，恢复要谨慎确认。

---

## ③ NetFlow：看清谁在用带宽

### 3.1 为什么需要

**场景**：出口 100Mbps 被占满了。`show interfaces` 只能告诉你"满了"，但：
- **是谁在占？**（哪台机器）
- **在干什么？**（什么应用）
- **访问哪里？**（什么目的地）

**NetFlow 记录每一条"流"的元数据**，回答上述问题。

### 3.2 流（Flow）的定义

**传统 NetFlow v5 的七元组**：
```
① 源 IP
② 目的 IP
③ 源端口
④ 目的端口
⑤ 三层协议号
⑥ ToS 字节（DSCP）
⑦ 输入接口
```
**这七个字段完全相同的包 = 同一条流。**

**Flexible NetFlow（FNF）**：可以**自定义**用哪些字段作为 key，以及采集哪些数据。

### 3.3 配置（Flexible NetFlow）

```cisco
! ── ① 定义流记录（采集什么）──
R1(config)# flow record FLOW-RECORD-V4
R1(config-flow-record)#  match ipv4 source address           ! key
R1(config-flow-record)#  match ipv4 destination address      ! key
R1(config-flow-record)#  match ipv4 protocol                 ! key
R1(config-flow-record)#  match transport source-port         ! key
R1(config-flow-record)#  match transport destination-port    ! key
R1(config-flow-record)#  match ipv4 tos                      ! key
R1(config-flow-record)#  match interface input               ! key
R1(config-flow-record)#  collect counter bytes               ! 采集
R1(config-flow-record)#  collect counter packets
R1(config-flow-record)#  collect timestamp sys-uptime first
R1(config-flow-record)#  collect timestamp sys-uptime last
R1(config-flow-record)#  collect interface output
R1(config-flow-record)#  collect application name            ! NBAR2 应用识别

! ── ② 定义导出器（发给谁）──
R1(config)# flow exporter FLOW-EXPORTER
R1(config-flow-exporter)#  destination 10.1.30.200
R1(config-flow-exporter)#  source Loopback0
R1(config-flow-exporter)#  transport udp 2055
R1(config-flow-exporter)#  export-protocol netflow-v9        ! 或 ipfix
R1(config-flow-exporter)#  template data timeout 60

! ── ③ 定义监视器（组合）──
R1(config)# flow monitor FLOW-MONITOR
R1(config-flow-monitor)#  record FLOW-RECORD-V4
R1(config-flow-monitor)#  exporter FLOW-EXPORTER
R1(config-flow-monitor)#  cache timeout active 60            ! 活跃流 60 秒导出一次
R1(config-flow-monitor)#  cache timeout inactive 15          ! 非活跃流 15 秒后导出

! ── ④ 应用到接口 ──
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip flow monitor FLOW-MONITOR input
R1(config-if)# ip flow monitor FLOW-MONITOR output

! ── ⑤ 采样（高流量场景，减轻 CPU）──
R1(config)# sampler SAMPLER-1
R1(config-sampler)#  mode random 1 out-of 100                ! 1/100 采样
R1(config-if)# ip flow monitor FLOW-MONITOR sampler SAMPLER-1 input

! ── 查看 ──
R1# show flow monitor FLOW-MONITOR cache
R1# show flow monitor FLOW-MONITOR cache sort highest counter bytes top 10   ! ★ 找出流量大户
R1# show flow exporter statistics
R1# show flow interface
R1# show flow record
```

**排查"谁在占带宽"**：
```cisco
R1# show flow monitor FLOW-MONITOR cache sort highest counter bytes top 10

IPV4 SRC ADDR   IPV4 DST ADDR   TRNS SRC PORT  TRNS DST PORT  bytes
===============  ==============  =============  =============  ==========
192.168.1.66     203.119.x.x            51234           443    8234567890   ← 找到了
192.168.1.66     203.119.x.x            51235           443    6543210987
192.168.1.102    140.82.x.x             49821           443     234567890
```

**一眼看出 `192.168.1.66` 在大量上传数据。**

### 3.4 传统 NetFlow（简化配置）

```cisco
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip flow ingress
R1(config-if)# ip flow egress

R1(config)# ip flow-export version 9
R1(config)# ip flow-export destination 10.1.30.200 2055
R1(config)# ip flow-export source Loopback0

R1# show ip cache flow
R1# show ip flow export
```

### 3.5 NetFlow vs SPAN vs sFlow

| | **NetFlow** | **SPAN/RSPAN** | **sFlow** |
|:--|:--|:--|:--|
| 采集什么 | **流的元数据**（谁到谁、多少字节） | **完整的数据包** | 采样的包头 + 计数器 |
| 开销 | 低-中 | **高**（复制全部流量） | **低** |
| 用途 | **流量分析、计费、异常检测** | **深度包分析、抓包排障** | 大规模流量趋势 |
| 存储 | 小 | **巨大** | 小 |
| 厂商 | Cisco 主导（IPFIX 是标准化版本） | 通用 | 标准（多厂商） |

**选择**：
- 想知道"**谁在用带宽**" → **NetFlow**
- 想知道"**包里面是什么内容**" → **SPAN + Wireshark**
- 大规模网络的流量趋势 → sFlow

---

## ④ SNMP：性能监控与告警

### 4.1 版本对比（★ 考点）

| 版本 | 认证 | 加密 | 说明 |
|:--|:--|:--|:--|
| **v1** | Community String（**明文**） | ❌ | 已淘汰 |
| **v2c** | Community String（**明文**） | ❌ | 仍广泛使用，但不安全 |
| **v3** | **用户名 + 认证（MD5/SHA）** | ✅ **DES/AES** | ★ **生产环境应使用** |

**SNMPv3 的三种安全级别**：

| 级别 | 认证 | 加密 |
|:--|:--|:--|
| **noAuthNoPriv** | ❌ | ❌ |
| **authNoPriv** | ✅ | ❌ |
| **authPriv** | ✅ | ✅ ★ 推荐 |

### 4.2 配置

```cisco
! ═══ SNMPv2c（简单但不安全）═══
R1(config)# snmp-server community MyReadOnly RO 20        ! 只读 + ACL 限制
R1(config)# snmp-server community MyReadWrite RW 21       ! 读写（★ 慎用）
R1(config)# access-list 20 permit 10.1.30.200
R1(config)# access-list 20 permit 10.1.30.201

! ═══ SNMPv3（★ 推荐）═══
R1(config)# snmp-server group MONITORING v3 priv read SNMPVIEW access 20
R1(config)# snmp-server view SNMPVIEW iso included
R1(config)# snmp-server user nms-user MONITORING v3 auth sha MyAuthPass priv aes 128 MyPrivPass

! ═══ Trap 配置 ═══
R1(config)# snmp-server host 10.1.30.200 version 3 priv nms-user
R1(config)# snmp-server enable traps snmp linkdown linkup coldstart warmstart
R1(config)# snmp-server enable traps config
R1(config)# snmp-server enable traps envmon
R1(config)# snmp-server enable traps cpu threshold
R1(config)# snmp-server enable traps bgp
R1(config)# snmp-server enable traps ospf
R1(config)# snmp-server enable traps hsrp
R1(config)# snmp-server enable traps syslog

! 系统信息
R1(config)# snmp-server location "SZ-Office-8F-RackA-U12"
R1(config)# snmp-server contact "netadmin@example.com"
R1(config)# snmp-server chassis-id "R1-CORE-01"

! ═══ CPU / 内存阈值告警 ═══
R1(config)# process cpu threshold type total rising 80 interval 30 falling 40 interval 30
R1(config)# snmp-server enable traps cpu threshold

! ═══ 查看 ═══
R1# show snmp
R1# show snmp community
R1# show snmp user
R1# show snmp group
R1# show snmp host
```

### 4.3 Trap vs Inform

| | **Trap** | **Inform** |
|:--|:--|:--|
| 可靠性 | ❌ **发完就不管**（UDP，可能丢） | ✅ **需要 NMS 确认，未确认会重传** |
| 开销 | 低 | 高（要保存直到确认） |
| 用途 | 一般告警 | **关键告警** |

```cisco
R1(config)# snmp-server host 10.1.30.200 informs version 2c MyCommunity
R1(config)# snmp-server enable traps
```

### 4.4 常用 MIB / OID

| 监控项 | OID |
|:--|:--|
| 系统描述 | `1.3.6.1.2.1.1.1` (sysDescr) |
| 运行时间 | `1.3.6.1.2.1.1.3` (sysUpTime) |
| 接口列表 | `1.3.6.1.2.1.2.2.1.2` (ifDescr) |
| **接口入流量** | `1.3.6.1.2.1.2.2.1.10` (ifInOctets) |
| **接口出流量** | `1.3.6.1.2.1.2.2.1.16` (ifOutOctets) |
| **64 位入流量** | `1.3.6.1.2.1.31.1.1.1.6` (ifHCInOctets) ★ 千兆以上必须用 |
| 接口状态 | `1.3.6.1.2.1.2.2.1.8` (ifOperStatus) |
| **CPU 5 分钟** | `1.3.6.1.4.1.9.2.1.58` (Cisco) |
| **内存空闲** | `1.3.6.1.4.1.9.9.48.1.1.1.6` (Cisco) |

> ⚠️ **千兆以上接口必须用 64 位计数器（ifHCInOctets）**。32 位计数器（ifInOctets）在千兆满载时**约 34 秒就会翻转一圈**，导致监控数据完全错误（出现负值或巨大的尖峰）。
>
> **这是流量监控最常见的坑**——图表上出现莫名其妙的尖峰，八成是用了 32 位计数器。

---

## ⑤ Syslog：事件记录

### 5.1 严重级别（★ 必背）

| Level | 名称 | 含义 | 示例 |
|:--|:--|:--|:--|
| **0** | **Emergency** | 系统不可用 | 系统崩溃 |
| **1** | **Alert** | 需立即处理 | 温度过高 |
| **2** | **Critical** | 严重 | 硬件故障、内存耗尽 |
| **3** | **Error** | 错误 | 接口错误、认证失败 |
| **4** | **Warning** | 警告 | 配置警告、阈值接近 |
| **5** | **Notification** | 正常但重要 | **接口 up/down、配置变更** |
| **6** | **Informational** | 信息 | ACL 匹配日志 |
| **7** | **Debugging** | 调试 | debug 输出 |

**记忆口诀**：**E**very **A**wesome **C**isco **E**ngineer **W**ill **N**eed **I**ce cream **D**aily
（Emergency, Alert, Critical, Error, Warning, Notification, Informational, Debugging）

> **数字越小越严重。** 设置日志级别为 N 时，会记录 **0 到 N** 的所有级别。
>
> **生产环境建议：`informational` (6)**。设成 `debugging` (7) 会产生海量日志。

### 5.2 日志消息格式

```
*Aug 30 10:23:45.123 CST: %LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/1, changed state to down
 └────────┬─────────┘  └─────┬────┘ ↑ └──┬──┘ └──────────────────┬──────────────────────┘
      时间戳            设施(Facility) │  助记符                  描述
                                   严重级别
```

### 5.3 配置

```cisco
! ── 时间戳（★ 必配）──
R1(config)# service timestamps log datetime msec localtime show-timezone
R1(config)# service timestamps debug datetime msec localtime show-timezone
R1(config)# clock timezone CST 8

! ── 序列号（防止日志被篡改/遗漏）──
R1(config)# service sequence-numbers

! ── 本地缓冲区 ──
R1(config)# logging buffered 65536 informational

! ── 远程 Syslog 服务器 ──
R1(config)# logging host 10.1.30.210
R1(config)# logging host 10.1.30.211 transport tcp port 1514    ! TCP 更可靠
R1(config)# logging trap informational                          ! 发送级别
R1(config)# logging source-interface Loopback0                  ! ★ 固定源地址
R1(config)# logging facility local6

! ── Console / VTY ──
R1(config)# logging console warnings          ! Console 只显示 warning 以上（防刷屏）
R1(config)# logging monitor informational
R1(config)# line vty 0 15
R1(config-line)# logging synchronous

! ── 速率限制（防止日志风暴打爆 CPU）──
R1(config)# logging rate-limit 100 except errors

! ── 查看 ──
R1# show logging
R1# show logging | include %LINK
R1# show logging | include GigabitEthernet0/1
R1# clear logging
```

> **`logging source-interface Loopback0` 很重要**：不配的话，日志的源 IP 会是出接口地址，多路径时可能变化，导致 Syslog 服务器上同一台设备的日志被归到不同来源。**用 Loopback 保证源地址恒定。**

### 5.4 EEM：自动化响应

**EEM (Embedded Event Manager)** 可以在特定事件发生时**自动执行动作**。

```cisco
! 示例 1：接口 down 时自动收集诊断信息
R1(config)# event manager applet INTERFACE-DOWN
R1(config-applet)#  event syslog pattern "%LINEPROTO-5-UPDOWN.*GigabitEthernet0/1.*down"
R1(config-applet)#  action 1.0 cli command "enable"
R1(config-applet)#  action 2.0 cli command "show interfaces GigabitEthernet0/1"
R1(config-applet)#  action 3.0 cli command "show logging | last 50"
R1(config-applet)#  action 4.0 syslog msg "ALERT: Gi0/1 went down, diagnostics collected"

! 示例 2：CPU 超过 90% 时记录进程信息
R1(config)# event manager applet HIGH-CPU
R1(config-applet)#  event snmp oid 1.3.6.1.4.1.9.2.1.58.0 get-type exact entry-op ge entry-val 90 poll-interval 30
R1(config-applet)#  action 1.0 cli command "enable"
R1(config-applet)#  action 2.0 cli command "show processes cpu sorted | exclude 0.00"
R1(config-applet)#  action 3.0 syslog msg "ALERT: CPU exceeded 90%"

! 示例 3：配置变更时自动备份
R1(config)# event manager applet CONFIG-BACKUP
R1(config-applet)#  event syslog pattern "%SYS-5-CONFIG_I"
R1(config-applet)#  action 1.0 cli command "enable"
R1(config-applet)#  action 2.0 cli command "copy running-config tftp://10.1.30.220/$_event_pub_time-R1.cfg"

R1# show event manager policy registered
R1# show event manager history events
```

---

## ⑥ NTP（回顾 + 进阶）

**详见 [Stage 1 第 7 章](../01-CCNA补齐篇/07-DHCP-DNS-NTP基础服务.md)。这里补充企业级配置。**

```cisco
! ── 内网 NTP 服务器 ──
NTP-SRV(config)# ntp master 3
NTP-SRV(config)# ntp server ntp.aliyun.com
NTP-SRV(config)# ntp authenticate
NTP-SRV(config)# ntp authentication-key 1 md5 NtpSecretKey
NTP-SRV(config)# ntp trusted-key 1

! ── 客户端 ──
R1(config)# ntp server 10.1.30.61 key 1 prefer
R1(config)# ntp server 10.1.30.62 key 1
R1(config)# ntp authenticate
R1(config)# ntp authentication-key 1 md5 NtpSecretKey
R1(config)# ntp trusted-key 1
R1(config)# ntp source Loopback0
R1(config)# clock timezone CST 8

! ── 限制访问（安全）──
R1(config)# access-list 30 permit 10.0.0.0 0.255.255.255
R1(config)# ntp access-group peer 30
R1(config)# ntp access-group serve-only 30

! ── 查看 ──
R1# show ntp status
R1# show ntp associations
R1# show clock detail
```

**关键验证**：
```cisco
R1# show ntp status
Clock is synchronized, stratum 4, reference is 10.1.30.61
      ↑ ★ 必须是 synchronized

R1# show ntp associations
  address         ref clock    st  when  poll reach  delay  offset  disp
*~10.1.30.61      203.107.6.88  3    45    64   377  1.234   0.567 0.123
 ↑                                              ↑
星号=当前同步源                              377=最近8次全成功
```

---

## ⑦ 配套实验：出口链路质量监控与自动切换

**场景**：双运营商出口，需要在电信链路质量下降时自动切到联通。

### Step 1：配置多目标探测

```cisco
! 探测电信方向（通过电信出口）
R1(config)# ip sla 1
R1(config-ip-sla)#  icmp-echo 114.114.114.114 source-interface GigabitEthernet0/1
R1(config-ip-sla-echo)#   frequency 5
R1(config-ip-sla-echo)#   timeout 2000
R1(config-ip-sla-echo)#   threshold 500
R1(config-ip-sla-echo)#   tag "TELECOM-DNS"

R1(config)# ip sla 2
R1(config-ip-sla)#  icmp-echo 223.5.5.5 source-interface GigabitEthernet0/1
R1(config-ip-sla-echo)#   frequency 5
R1(config-ip-sla-echo)#   timeout 2000
R1(config-ip-sla-echo)#   tag "ALIYUN-DNS"

R1(config)# ip sla schedule 1 life forever start-time now
R1(config)# ip sla schedule 2 life forever start-time now
```

### Step 2：组合跟踪

```cisco
R1(config)# track 1 ip sla 1 reachability
R1(config-track)#  delay down 10 up 30

R1(config)# track 2 ip sla 2 reachability
R1(config-track)#  delay down 10 up 30

! 两个目标都不通才认为链路故障（避免单个目标故障导致误切换）
R1(config)# track 10 list boolean or
R1(config-track)#  object 1
R1(config-track)#  object 2
R1(config-track)#  delay down 15 up 60
```

### Step 3：驱动路由切换

```cisco
! 主路由：走电信，绑定 track 10
R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.11.1 track 10

! 备份路由：走联通，AD 更高
R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.22.1 10
```

### Step 4：验证

```cisco
R1# show ip sla statistics
IPSLA operation id: 1
        Latest RTT: 12 milliseconds
Latest operation return code: OK                 ← ✓

R1# show track
Track 1
  IP SLA 1 reachability
  Reachability is Up                             ← ✓
    2 changes, last change 00:15:23
  Delay up 30 secs, down 10 secs
  Latest operation return code: OK

Track 10
  List boolean or
  Boolean OR is Up
    2 changes, last change 00:15:23
    object 1 Up
    object 2 Up
  Tracked by:
    STATIC-IP-ROUTING 0                          ← ★ 被静态路由使用

R1# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
  Known via "static", distance 1, metric 0
  * 10.0.11.1                                    ← 走电信 ✓
```

### Step 5：模拟故障

```cisco
! 用 ACL 阻断探测目标，模拟链路质量下降
R1(config)# ip access-list extended BLOCK-PROBE
R1(config-ext-nacl)#  deny icmp any host 114.114.114.114
R1(config-ext-nacl)#  deny icmp any host 223.5.5.5
R1(config-ext-nacl)#  permit ip any any
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip access-group BLOCK-PROBE out
```

**观察（约 15 秒后，因为配了 `delay down 15`）**：
```cisco
R1# show track 10
Track 10
  List boolean or
  Boolean OR is Down                             ← 检测到故障 ✓
    2 changes, last change 00:00:18

R1# show ip route 0.0.0.0
  * 10.0.22.1                                    ← ★ 自动切到联通 ✓
```

**日志**：
```
%TRACK-6-STATE: 1 ip sla 1 reachability Up -> Down
%TRACK-6-STATE: 10 list boolean or Up -> Down
```

### Step 6：恢复验证

```cisco
R1(config)# interface GigabitEthernet0/1
R1(config-if)# no ip access-group BLOCK-PROBE out
```

**观察**：60 秒后（`delay up 60`）才切回电信——**这个延迟是有意的**，避免链路刚恢复还不稳定就切回去。

### Step 7：加上 NetFlow 观察流量变化

```cisco
R1(config)# flow monitor FLOW-MON
R1(config-flow-monitor)#  record netflow ipv4 original-input
R1(config-flow-monitor)#  cache timeout active 60

R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip flow monitor FLOW-MON input
R1(config)# interface GigabitEthernet0/2
R1(config-if)# ip flow monitor FLOW-MON input

R1# show flow monitor FLOW-MON cache sort highest counter bytes top 10
```

**切换前后对比流量分布**，验证流量确实转移到了备份链路。

---

## ⑧ 排障速查表

| 症状 | 怀疑点 | 验证 |
|:--|:--|:--|
| IP SLA 一直 Timeout | 目标不可达 / 源接口错 | `show ip sla statistics`、手工 ping |
| Track 状态不变 | 未绑定 / delay 太长 | `show track`、`show track brief` |
| 配了 track 但路由不切 | decrement/AD 值不当 | 检查主备路由的 AD |
| 频繁切换（震荡） | **没配 delay** | 加 `delay down X up Y` |
| NetFlow 没数据 | 未应用到接口 | `show flow interface` |
| NetFlow 导出失败 | 目标不可达 / 端口错 | `show flow exporter statistics` |
| SNMP 取不到数据 | Community/ACL/版本 | `show snmp`、NMS 侧测试 |
| **流量图有异常尖峰** | **32 位计数器翻转** | 改用 `ifHCInOctets` (64位) |
| Syslog 收不到 | 级别/源地址/网络 | `show logging`、抓 UDP 514 |
| **日志时间对不上** | **NTP 没同步** | `show ntp status` |
| 日志刷屏影响操作 | Console 级别太低 | `logging console warnings` |

---

## ⑨ 三厂商对照 + 考点自测

### 三厂商对照

| 目的 | Cisco | H3C | 华为 |
|:--|:--|:--|:--|
| IP SLA | `ip sla 1` + `icmp-echo` | `nqa entry admin test` + `type icmp-echo` | `nqa test-instance admin test` + `test-type icmp` |
| Track | `track 1 ip sla 1 reachability` | `track 1 nqa entry admin test reaction 1` | `track 1 nqa entry admin test reaction 1` |
| NetFlow | `flow monitor` / `ip flow ingress` | `ip netstream inbound` | `ip netstream inbound` |
| NetFlow 导出 | `flow exporter` | `ip netstream export host X 2055` | `netstream export ip host X 2055` |
| SNMP 团体 | `snmp-server community X RO` | `snmp-agent community read X` | `snmp-agent community read X` |
| SNMP v3 | `snmp-server user ...` | `snmp-agent usm-user v3 ...` | `snmp-agent usm-user v3 ...` |
| Syslog 服务器 | `logging host 10.1.30.210` | `info-center loghost 10.1.30.210` | `info-center loghost 10.1.30.210` |
| 日志级别 | `logging trap informational` | `info-center source default loghost level informational` | `info-center loghost X level informational` |
| NTP 客户端 | `ntp server X` | `ntp-service unicast-server X` | `ntp-service unicast-server X` |
| 查看日志 | `show logging` | `display logbuffer` | `display logbuffer` |

> **术语差异**：Cisco 的 **IP SLA** 在 H3C/华为叫 **NQA**（Network Quality Analyzer）；**NetFlow** 在 H3C/华为叫 **NetStream**。

### 考点

- **IP SLA + Track** 驱动路由/HSRP 切换
- **`delay down/up` 防抖的必要性**
- **NetFlow 的流定义（七元组）**
- **NetFlow vs SPAN 的区别**
- **SNMP v2c 明文 vs v3 加密**
- **Trap vs Inform**
- **千兆以上必须用 64 位计数器**
- **Syslog 8 个严重级别**（0 最严重）
- **NTP 对日志关联分析的重要性**

### 自测题

**1.** 为什么浮动静态路由需要配合 IP SLA + Track，而不能只靠接口状态？

<details><summary>答案</summary>

**因为浮动静态路由的撤销条件是"本地出接口 down 或下一跳不可达"。如果故障发生在远端，本地接口依然 up，路由不会被撤销。**

**典型场景**：
```
   R1 ──── [交换机] ──── [光猫] ──── [运营商] ──── Internet
    ↑
   Gi0/1 一直是 up（它连的是交换机，链路层没断）
   
   运营商侧故障 → 实际不通了
   但 R1 的 Gi0/1 依然 up → 主路由依然在路由表里
   → ★ 流量继续送进黑洞 ★
```

**解法：IP SLA 探测端到端可达性**
```cisco
R1(config)# ip sla 1
R1(config-ip-sla)#  icmp-echo 114.114.114.114 source-interface GigabitEthernet0/1
R1(config-ip-sla-echo)#   frequency 5
R1(config-ip-sla-echo)#   timeout 2000
R1(config)# ip sla schedule 1 life forever start-time now

R1(config)# track 1 ip sla 1 reachability
R1(config-track)#  delay down 10 up 30

R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.11.1 track 1     ! ★ 绑定 track
R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.22.1 10          ! 备份
```

**工作原理**：
```
   IP SLA 每 5 秒探测一次 114.114.114.114
        ↓
   连续失败 10 秒 → track 1 变成 Down
        ↓
   ★ 绑定了 track 1 的主路由被从路由表撤销 ★
        ↓
   AD=10 的备份路由自动生效
```

**★ `delay down/up` 的重要性**：
```cisco
R1(config-track)#  delay down 10 up 30
!                        ↑      ↑
!                 down确认10秒  up确认30秒
```

**不配的后果**：偶尔丢一个探测包（网络正常波动）就触发路由切换。**频繁切换（震荡）比不切换更糟**——每次切换都会中断 TCP 会话、触发路由收敛、可能引发上游的连锁反应。

**为什么 `up` 的延迟要大于 `down`**：
- 故障要**快速响应**（down 10 秒）
- 恢复要**谨慎确认**（up 30 秒），因为链路刚恢复时往往还不稳定

**探测目标的选择（重要）**：

| 目标 | 优点 | 缺点 |
|:--|:--|:--|
| 直连的运营商网关 | 快速反映本段链路 | 只能发现第一跳的问题 |
| **公网 DNS（114/223.5.5.5/8.8.8.8）** | 反映端到端可达性 | 目标本身可能不稳定 |
| 自己的另一个站点 | 最贴近实际业务 | 需要对端配合 |

**最佳实践：探测 2-3 个不同的目标，用布尔或组合**
```cisco
R1(config)# track 1 ip sla 1 reachability      ! 探 114.114.114.114
R1(config)# track 2 ip sla 2 reachability      ! 探 223.5.5.5
R1(config)# track 10 list boolean or           ! 任一通就算通
R1(config-track)#  object 1
R1(config-track)#  object 2
R1(config-track)#  delay down 15 up 60
R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.11.1 track 10
```

这样避免"某个 DNS 服务器自己维护"导致的误切换。
</details>

**2.** NetFlow 和 SPAN 有什么区别？分别用在什么场景？

<details><summary>答案</summary>

| | **NetFlow** | **SPAN / RSPAN / ERSPAN** |
|:--|:--|:--|
| 采集什么 | **流的元数据**（源目 IP/端口、字节数、包数、时间） | **完整的原始数据包** |
| 数据量 | **小**（一条流一条记录） | **巨大**（复制全部流量） |
| 设备开销 | 低-中 | **高**（占用背板带宽和目的口带宽） |
| 能回答 | "**谁在用带宽**"、"访问了哪里"、"用什么协议" | "**包里到底是什么**"、协议交互细节 |
| 不能回答 | 包的具体内容 | 长期趋势（数据量太大存不下） |
| 存储 | 可长期保存（几个月） | 只能短期（几分钟到几小时） |
| 典型工具 | ntopng、SolarWinds、ELK | Wireshark、tcpdump |

**场景选择**：

**用 NetFlow**：
- ✅ "出口带宽被谁占满了？"
- ✅ "上个月哪个部门流量最大？"（计费）
- ✅ "有没有异常的外联行为？"（安全，比如某台机器突然大量连接境外 IP）
- ✅ "DDoS 攻击的流量特征是什么？"
- ✅ 长期趋势分析和容量规划

```cisco
R1# show flow monitor FLOW-MON cache sort highest counter bytes top 10
IPV4 SRC ADDR   IPV4 DST ADDR   TRNS DST PORT   bytes
==============  ==============  =============   ==========
192.168.1.66    203.119.x.x              443    8234567890    ← 一眼看出是谁
```

**用 SPAN**：
- ✅ "TCP 握手到底卡在哪一步？"
- ✅ "这个应用的报文格式对不对？"
- ✅ "为什么服务器回了 RST？"
- ✅ 协议级排障、安全取证

```cisco
! 本地 SPAN
SW1(config)# monitor session 1 source interface GigabitEthernet1/0/5 both
SW1(config)# monitor session 1 destination interface GigabitEthernet1/0/48

! 远程 SPAN（RSPAN）
SW1(config)# vlan 999
SW1(config-vlan)#  remote-span
SW1(config)# monitor session 1 source interface Gi1/0/5
SW1(config)# monitor session 1 destination remote vlan 999

! ERSPAN（跨三层，封装成 GRE）
SW1(config)# monitor session 1 type erspan-source
SW1(config-mon-erspan-src)#  source interface Gi1/0/5
SW1(config-mon-erspan-src)#  destination
SW1(config-mon-erspan-src-dst)#   erspan-id 100
SW1(config-mon-erspan-src-dst)#   ip address 10.1.30.230
SW1(config-mon-erspan-src-dst)#   origin ip address 10.1.1.1
```

**⚠️ SPAN 的注意事项**：
1. **目的口的带宽必须足够**。镜像一个千兆口的双向流量，目的口需要 2Gbps 才不丢——所以通常要用万兆口做目的口。
2. **占用设备资源**。多数平台只支持 2-4 个 SPAN 会话。
3. **目的口不能同时做正常转发**（会被独占）。
4. 高流量场景下 **SPAN 本身会丢包**，抓到的不是完整的流量。

**实战组合用法（推荐）**：
```
   ① 平时开着 NetFlow  → 持续监控，发现异常
        ↓
   ② 发现异常后（比如某个 IP 流量暴增）
        ↓
   ③ 针对性地开 SPAN → 抓那台机器的包，深入分析
        ↓
   ④ 分析完关掉 SPAN
```

**先用 NetFlow 定位"是谁"，再用 SPAN 分析"在干什么"。** 这样既有长期可见性，又不会常态化地消耗设备资源。
</details>

**3.** SNMP v2c 和 v3 有什么区别？为什么生产环境应该用 v3？

<details><summary>答案</summary>

| | **SNMPv2c** | **SNMPv3** |
|:--|:--|:--|
| 认证 | **Community String（明文）** | **用户名 + 密码（MD5/SHA 哈希）** |
| 加密 | ❌ **完全不加密** | ✅ **DES / 3DES / AES** |
| 完整性 | ❌ | ✅ 防篡改 |
| 防重放 | ❌ | ✅ |
| 访问控制 | 粗粒度（RO/RW） | **细粒度（View + Group）** |

**v2c 的三个致命问题**：

**① Community String 明文传输**
抓一个包就能看到 community。而 community 相当于密码——拿到 RO 的 community 就能读取整个设备的配置和状态，拿到 RW 的就能**改配置**。

```bash
# 攻击者抓包
tcpdump -i eth0 -A udp port 161
# 直接看到：community: "public" 或 "MyReadOnly"
```

**② 大量设备用默认 community**
`public`（只读）和 `private`（读写）是出厂默认值。**扫描工具几秒钟就能把整个网段的设备扫一遍**：
```bash
onesixtyone -c community.txt 10.1.0.0/16
```

**③ RW community 泄露 = 设备被完全控制**
```bash
# 用 RW community 修改配置
snmpset -v2c -c private 10.1.1.1 1.3.6.1.4.1.9.2.1.53.10.1.30.99 s "startup-config"
# 甚至可以让设备把配置文件 TFTP 到攻击者的服务器
```

**SNMPv3 的配置**：
```cisco
! ① 定义视图（能看哪些 OID）
R1(config)# snmp-server view MONITOR-VIEW iso included
R1(config)# snmp-server view MONITOR-VIEW 1.3.6.1.6.3.15 excluded     ! 排除敏感的 USM MIB

! ② 定义组（安全级别 + 权限）
R1(config)# snmp-server group MONITORING v3 priv read MONITOR-VIEW access 20
!                                          ↑
!                                    priv = 认证 + 加密

! ③ 定义用户
R1(config)# snmp-server user nms-user MONITORING v3 auth sha MyAuthPassword priv aes 128 MyPrivPassword
!                                                       ↑                    ↑
!                                                   认证算法              加密算法

! ④ ACL 限制来源
R1(config)# access-list 20 permit 10.1.30.200
R1(config)# access-list 20 permit 10.1.30.201

! ⑤ Trap
R1(config)# snmp-server host 10.1.30.200 version 3 priv nms-user
```

**三种安全级别**：

| 级别 | 认证 | 加密 | 说明 |
|:--|:--|:--|:--|
| `noAuthNoPriv` | ❌ | ❌ | 只有用户名，形同虚设 |
| `authNoPriv` | ✅ | ❌ | 能验证身份，但数据明文 |
| **`priv`** | ✅ | ✅ | ★ **生产环境用这个** |

**如果暂时无法迁移到 v3，v2c 的最低加固要求**：
```cisco
! ① 绝不使用默认 community
R1(config)# no snmp-server community public
R1(config)# no snmp-server community private

! ② 用复杂的 community（当密码看待）
R1(config)# snmp-server community Xk9#mP2$vL8qR RO 20

! ③ ★ 必须配 ACL 限制来源
R1(config)# access-list 20 permit host 10.1.30.200
R1(config)# access-list 20 deny any log

! ④ ★★ 绝不配置 RW community
!    监控只需要只读权限。需要改配置用 NETCONF/RESTCONF + AAA

! ⑤ 只在管理 VLAN 上开放 SNMP
```

**审计现有环境**：
```cisco
R1# show snmp community
R1# show snmp user
R1# show snmp group
R1# show snmp host
```

**如果发现有 RW community 或没有 ACL 限制，这是高危漏洞，应立即处理。**
</details>

**4.** Syslog 的 8 个严重级别是什么？生产环境应该设置成哪个级别？

<details><summary>答案</summary>

| Level | 名称 | 含义 | 典型消息 |
|:--|:--|:--|:--|
| **0** | **Emergency** | 系统不可用 | 系统崩溃 |
| **1** | **Alert** | 需立即处理 | 温度过高、电源故障 |
| **2** | **Critical** | 严重情况 | 硬件故障、内存耗尽 |
| **3** | **Error** | 错误 | 接口错误、认证失败、OSPF 邻居异常 |
| **4** | **Warning** | 警告 | 配置警告、阈值接近 |
| **5** | **Notification** | 正常但重要 | **接口 up/down、配置变更、BGP 邻居变化** |
| **6** | **Informational** | 一般信息 | ACL 匹配日志、DHCP 分配 |
| **7** | **Debugging** | 调试信息 | debug 命令的输出 |

**记忆口诀**：
```
Every Awesome Cisco Engineer Will Need Ice cream Daily
  E      A      C      E      W     N      I         D
  0      1      2      3      4     5      6         7
```

**★ 数字越小越严重。** 配置级别为 N 时，会记录 **0 到 N** 的所有级别。

**生产环境建议**：

| 目标 | 建议级别 | 理由 |
|:--|:--|:--|
| **远程 Syslog 服务器** | **`informational` (6)** | 保留足够的排障信息，同时避免 debug 级别的海量日志 |
| **本地缓冲区** | **`informational` (6)** | 同上 |
| **Console** | **`warnings` (4)** | ★ 防止日志刷屏影响命令行操作 |
| **VTY (monitor)** | `informational` (6) | 按需 |

```cisco
R1(config)# logging trap informational           ! 发给 Syslog 服务器
R1(config)# logging buffered 65536 informational ! 本地缓冲
R1(config)# logging console warnings             ! ★ Console 只显示重要的
R1(config)# logging monitor informational        ! VTY
```

**为什么 Console 要设成 warnings**：

在故障排查时，Console 是你最后的救命通道。如果这时候大量 `informational` 级别的日志在刷屏，**你根本没法敲命令**。

```cisco
! 同时配上这条，让日志不打断你正在输入的命令
R1(config)# line console 0
R1(config-line)# logging synchronous
```

**为什么不用 debugging (7)**：
- `debug` 输出量极大，**可能把 CPU 打满**
- 会淹没真正重要的日志
- 磁盘/存储会被快速撑满

**如果确实需要 debug，用条件调试**：
```cisco
R1# debug ip packet 101                          ! 只调试匹配 ACL 101 的
R1# debug condition interface GigabitEthernet0/1  ! 只调试这个接口
R1# debug crypto condition peer ipv4 203.2.2.2    ! 只调试这个 IPsec 对端
```

**完整的 Syslog 配置模板**：
```cisco
! 时间戳（★ 必配，否则日志没有时间意义）
service timestamps log datetime msec localtime show-timezone
service timestamps debug datetime msec localtime show-timezone
service sequence-numbers                         ! 序列号，防遗漏
clock timezone CST 8

! 本地
logging buffered 65536 informational

! 远程（配两个，冗余）
logging host 10.1.30.210
logging host 10.1.30.211 transport tcp port 1514
logging trap informational
logging source-interface Loopback0               ! ★ 固定源地址
logging facility local6

! Console / VTY
logging console warnings
logging monitor informational

! 速率限制（防日志风暴打爆 CPU）
logging rate-limit 100 except errors

! NTP（★ 前提条件）
ntp server 10.1.30.61 prefer
ntp server 10.1.30.62
```

**★ 最重要的前提：NTP 必须同步。**

没有准确的时间，多台设备的日志**根本无法关联分析**。故障复盘时，你需要知道"接口 down 和 BGP 邻居断开哪个先发生"——时间不准就完全无从判断。

参见 [Stage 1 第 7 章](../01-CCNA补齐篇/07-DHCP-DNS-NTP基础服务.md) 关于 NTP 的详细说明。
</details>

**5.** 监控图表上出现莫名其妙的流量尖峰或负值，最可能是什么原因？

<details><summary>答案</summary>

**最可能是使用了 32 位计数器，在高速接口上发生了计数器翻转（wrap）。**

**原理**：

SNMP 的传统接口流量计数器 **`ifInOctets` / `ifOutOctets` 是 32 位的**：
```
最大值 = 2^32 - 1 = 4,294,967,295 字节 ≈ 4 GB
```

**翻转时间计算**：
```
千兆接口满载：1 Gbps = 125 MB/s
4,294,967,295 ÷ 125,000,000 ≈ ★ 34 秒 ★

万兆接口满载：约 3.4 秒
```

**如果 SNMP 轮询间隔是 5 分钟（常见默认值）**，那么千兆接口在这 5 分钟里可能**翻转了 8 次以上**，采集到的数据完全没有意义。

**表现**：
- 流量图出现**巨大的尖峰**（因为计算的差值是错的）
- 出现**负值**（当前值小于上次值）
- 数据看起来"随机跳动"

**解决方案：使用 64 位计数器（High Capacity Counters）**

| 计数器 | OID | 位数 | 适用 |
|:--|:--|:--|:--|
| `ifInOctets` | `1.3.6.1.2.1.2.2.1.10` | 32 位 | ❌ 只适合 ≤ 100Mbps |
| **`ifHCInOctets`** | **`1.3.6.1.2.1.31.1.1.1.6`** | **64 位** | ✅ **千兆及以上必用** |
| `ifOutOctets` | `1.3.6.1.2.1.2.2.1.16` | 32 位 | ❌ |
| **`ifHCOutOctets`** | **`1.3.6.1.2.1.31.1.1.1.10`** | **64 位** | ✅ |

**64 位计数器的翻转时间**：
```
2^64 ≈ 1.8 × 10^19 字节
万兆接口满载翻转需要 ★ 约 468 年 ★
```

**配置要点**：

**① 64 位计数器需要 SNMPv2c 或 v3**（SNMPv1 不支持 64 位）
```cisco
R1(config)# snmp-server community MyCommunity RO 20
! v2c 及以上才支持 Counter64
```

**② 监控系统要正确配置**
- **Zabbix**：使用 `ifHCInOctets` 而不是 `ifInOctets`；模板选 "Network Generic by SNMP"（默认用 64 位）
- **PRTG**：传感器选 "SNMP Traffic (64-bit)"
- **Cacti**：模板里选 64 位 OID
- **LibreNMS/Observium**：默认自动检测并使用 64 位

**验证**：
```bash
# 用 snmpwalk 确认设备支持
snmpwalk -v2c -c MyCommunity 10.1.1.1 1.3.6.1.2.1.31.1.1.1.6
IF-MIB::ifHCInOctets.1 = Counter64: 12345678901234
                                    ↑ Counter64 = 支持 ✓
```

**其他可能导致图表异常的原因**：

| 原因 | 表现 | 排查 |
|:--|:--|:--|
| **32 位计数器翻转** | 尖峰、负值 | ★ 改用 64 位 |
| **设备重启** | 计数器归零，产生巨大负值 | 看 `sysUpTime` |
| SNMP 轮询超时/丢包 | 数据点缺失，插值产生异常 | 检查 SNMP 响应时间 |
| 轮询间隔太长 | 峰值被平均掉，看不出真实拥塞 | 缩短到 1-5 分钟 |
| 接口索引变化 | 数据突然归属到错误的接口 | 配 `snmp-server ifindex persist` |
| 真实的流量突发 | — | 用 NetFlow 确认 |

**接口索引持久化（另一个常见坑）**：
```cisco
R1(config)# snmp ifmib ifindex persist
```
**不配的话**，设备重启或插拔模块后，接口的 SNMP 索引（ifIndex）可能变化——原来 index 5 是 Gi0/1，重启后变成了 Gi0/5。**监控系统会把历史数据归属到错误的接口上**，图表完全错乱。

**监控部署检查清单**：
```
□ ① 使用 64 位计数器（ifHCInOctets / ifHCOutOctets）
□ ② SNMP 版本 ≥ v2c
□ ③ 配置 snmp ifmib ifindex persist
□ ④ 轮询间隔 1-5 分钟
□ ⑤ 用 SNMPv3 + ACL 限制来源
□ ⑥ NTP 同步（时间戳准确）
□ ⑦ 配合 NetFlow 做流量成分分析
```
</details>

---

**上一章** ← [11 无线架构与漫游](11-无线架构与漫游.md) ｜ **下一章** → [13 网络安全](13-网络安全-AAA-802.1X-控制平面保护.md)
