# 05 · TCP/UDP 与常见应用协议

## ① 这章解决什么问题

用户说"网站打不开"。你 ping 服务器，通的。traceroute，也通。路由表没问题，ACL 没拦。

到这一步，网络层已经排除了。**问题在传输层或应用层。**

这一章要教你的是：如何用 `telnet <ip> <port>`、抓包看 RST/SYN、看 TCP 窗口，把"网络问题"和"应用问题"干净地分开。这是运维工作里最能体现专业度的一件事——**准确地把锅甩给（或不甩给）应用团队**。

---

## ② 原理讲透

### 2.1 传输层解决两个问题

1. **进程复用**：一台服务器上跑着 Web(80)、SSH(22)、MySQL(3306)，包到了之后怎么知道给谁？靠**端口号**。
2. **可靠性**：IP 层是"尽力而为"，会丢包、乱序、重复。要可靠传输，得有人负责重传和排序 —— 这就是 TCP。

### 2.2 端口号

16 位，范围 0~65535。

| 范围 | 名称 | 说明 |
|:--|:--|:--|
| 0 – 1023 | 知名端口 Well-known | 由 IANA 分配，Linux 上需 root 才能监听 |
| 1024 – 49151 | 注册端口 Registered | 厂商注册，如 3306 MySQL、8080 HTTP-alt |
| 49152 – 65535 | 动态/私有 Ephemeral | 客户端发起连接时随机选用的源端口 |

**必背端口表**：

| 端口 | 协议 | 服务 | 备注 |
|:--|:--|:--|:--|
| 20/21 | TCP | FTP 数据/控制 | 主动模式数据口是 20 |
| 22 | TCP | SSH / SCP / SFTP | |
| 23 | TCP | Telnet | 明文，生产禁用 |
| 25 | TCP | SMTP | 发邮件 |
| 53 | **TCP+UDP** | DNS | **查询用 UDP，区域传送和大响应用 TCP** |
| 67/68 | UDP | DHCP Server/Client | |
| 69 | UDP | TFTP | 设备升级镜像常用 |
| 80 | TCP | HTTP | |
| 110 | TCP | POP3 | |
| 123 | UDP | NTP | |
| 143 | TCP | IMAP | |
| 161/162 | UDP | SNMP / SNMP Trap | |
| 179 | TCP | **BGP** | 网工必记 |
| 389 | TCP/UDP | LDAP | AD 域 |
| 443 | TCP | HTTPS | |
| 445 | TCP | SMB | Windows 文件共享 |
| 500 | UDP | ISAKMP / IKE | **IPsec 阶段一** |
| 514 | UDP | Syslog | |
| 4500 | UDP | IPsec NAT-T | **穿越 NAT 时用** |
| 1812/1813 | UDP | RADIUS 认证/计费 | 802.1X |
| 49 | TCP | TACACS+ | Cisco AAA |
| 3389 | TCP | RDP | Windows 远程桌面 |
| 5060/5061 | UDP/TCP | SIP | VoIP 信令 |
| 6633/6653 | TCP | OpenFlow | SDN |
| 830 | TCP | NETCONF over SSH | 网络自动化 |

> **网工重点关注**：`179 (BGP)`、`500/4500 (IPsec)`、`1812/1813 (RADIUS)`、`49 (TACACS+)`、`161/162 (SNMP)`、`514 (Syslog)`、`123 (NTP)`、`830 (NETCONF)`。这些在 ENCOR/ENARSI 的 ACL 题和排障题里反复出现。

**没有端口号的协议**：OSPF（IP 协议号 89）、EIGRP（88）、ESP（50）、AH（51）、GRE（47）、ICMP（1）。它们直接跑在 IP 之上，不经过传输层。

> **ACL 写法差异**：放行 OSPF 要写 `permit ospf`，不能写端口。放行 IPsec 要同时放 `udp 500`、`udp 4500` 和 `esp`（协议号 50）。漏放 ESP 是 VPN 建不起来的经典原因。

### 2.3 TCP 头部与标志位

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
┌───────────────────────────────┬───────────────────────────────┐
│          源端口 (16)           │         目的端口 (16)          │
├───────────────────────────────┴───────────────────────────────┤
│                         序列号 Sequence (32)                    │
├───────────────────────────────────────────────────────────────┤
│                      确认号 Acknowledgment (32)                 │
├──────┬───────┬────────────────┬───────────────────────────────┤
│偏移(4)│保留(6)│U A P R S F     │          窗口大小 (16)         │
│      │       │R C S S Y I     │                               │
│      │       │G K H T N N     │                               │
├──────┴───────┴────────────────┼───────────────────────────────┤
│          校验和 (16)           │         紧急指针 (16)          │
├───────────────────────────────┴───────────────────────────────┤
│                    选项 (可变，如 MSS、SACK、窗口缩放)            │
└───────────────────────────────────────────────────────────────┘
```

**六个标志位**（抓包排障核心）：

| 标志 | 全称 | 含义 | 看到它意味着 |
|:--|:--|:--|:--|
| **SYN** | Synchronize | 请求建立连接 | 有人在尝试连接 |
| **ACK** | Acknowledgment | 确认收到 | 正常数据交互 |
| **FIN** | Finish | 正常关闭连接 | 一方数据发完了 |
| **RST** | Reset | **强制重置连接** | **端口没开 / 被防火墙拒绝 / 应用崩了** |
| **PSH** | Push | 立即上交应用层，不缓冲 | 交互式应用（SSH/Telnet） |
| **URG** | Urgent | 紧急数据 | 极少见 |

> **排障金句**：**看到 RST，问题基本在服务端或中间设备，不在网络路径。**
> - 连接刚发 SYN 就收到 RST → 目标端口没有服务监听，或防火墙做了 reject（而非 drop）
> - 连接建立后突然 RST → 应用崩溃、超时、或中间防火墙会话表老化

### 2.4 三次握手

```
   客户端                                        服务器
     │                                             │
     │  ① SYN, Seq=x                               │
     │────────────────────────────────────────────>│   "我要连你，我的序号是 x"
     │                                             │
     │  ② SYN + ACK, Seq=y, Ack=x+1                │
     │<────────────────────────────────────────────│   "同意，我的序号是 y，我收到你的 x 了"
     │                                             │
     │  ③ ACK, Seq=x+1, Ack=y+1                    │
     │────────────────────────────────────────────>│   "我收到你的 y 了"
     │                                             │
     │            ═══ 连接建立 ═══                  │
```

**为什么是三次，不是两次？**

因为**双向都要确认序号**。
- 第 ① 步：客户端告知自己的初始序号 x
- 第 ② 步：服务器确认收到 x，**同时**告知自己的初始序号 y（两件事合并成一个包，这是"三次"而不是四次的原因）
- 第 ③ 步：客户端确认收到 y

如果只有两次，服务器无法确认客户端是否收到了自己的序号 y，也无法排除"客户端发的是一个延迟到达的历史 SYN"这种情况（会导致服务器白白建立一个半开连接，被 SYN Flood 攻击利用）。

### 2.5 四次挥手

```
   客户端                                        服务器
     │  ① FIN, Seq=u                               │
     │────────────────────────────────────────────>│   "我数据发完了"
     │  ② ACK, Ack=u+1                             │
     │<────────────────────────────────────────────│   "知道了"（但我可能还有数据要发）
     │                                             │
     │        ← 服务器可以继续发数据（半关闭）        │
     │                                             │
     │  ③ FIN, Seq=v                               │
     │<────────────────────────────────────────────│   "我也发完了"
     │  ④ ACK, Ack=v+1                             │
     │────────────────────────────────────────────>│   "知道了"
     │                                             │
     │        客户端进入 TIME_WAIT (2MSL)            │
```

**为什么挥手是四次而握手是三次？**

握手时，服务器的 ACK 和 SYN 可以**合并**成一个包——因为服务器一收到连接请求就能立刻决定是否接受。

挥手时不行：客户端说"我不发了"，但**服务器可能还有数据没发完**。所以服务器先回一个 ACK（"知道你不发了"），等自己的数据也发完了，再单独发 FIN。这个中间状态叫**半关闭（Half-Close）**。

**TIME_WAIT 状态（2×MSL，通常 60 秒）的作用**：
1. 确保最后那个 ACK 能到达服务器（如果丢了，服务器会重发 FIN，客户端还在 TIME_WAIT 就能再回一次 ACK）
2. 让本次连接的所有残留报文在网络中消亡，避免污染下一个使用相同四元组的新连接

> **运维现象**：高并发服务器上 `netstat -an | grep TIME_WAIT | wc -l` 数万条是正常的。真正需要担心的是大量 `CLOSE_WAIT`——那说明**你的应用收到 FIN 后没有调用 close()**，是代码 bug。

### 2.6 TCP vs UDP

| | TCP | UDP |
|:--|:--|:--|
| 连接 | 面向连接（握手） | 无连接，发了就不管 |
| 可靠性 | 确认 + 重传 + 排序 | 不保证，丢了就丢了 |
| 流量控制 | 滑动窗口 | 无 |
| 拥塞控制 | 有（慢启动、拥塞避免） | 无 |
| 头部开销 | 20 字节起（含选项可到 60） | **8 字节** |
| 速度 | 较慢 | 快 |
| 一对多 | 不支持 | **支持组播/广播** |
| 典型应用 | HTTP, SSH, BGP, FTP, SMTP | DNS, DHCP, SNMP, TFTP, NTP, VoIP, 视频流 |

**为什么路由协议的选择各不相同**（网工视角，很有意思）：

| 协议 | 传输方式 | 为什么 |
|:--|:--|:--|
| **BGP** | **TCP 179** | 路由表巨大（全网路由 90 万+条），必须可靠、有序、支持大量数据分片重组。而且 BGP 邻居通常不直连，TCP 能跨多跳 |
| **OSPF** | **直接跑 IP（协议号 89）** | 自己实现了确认（LSAck）和重传机制，不需要 TCP。用组播 224.0.0.5/224.0.0.6 高效泛洪 |
| **EIGRP** | **直接跑 IP（协议号 88）** | 自研 RTP（Reliable Transport Protocol），可靠性自己做，组播 224.0.0.10 |
| **RIP** | **UDP 520** | 老协议，设计简单，靠 30 秒周期性全量更新来"自愈"丢包 |

这个对比能帮你理解一个重要设计思想：**可靠性可以在任何一层实现**，选择在哪一层做，取决于对性能和灵活性的权衡。

### 2.7 MSS 与 MTU（排障重点）

```
MTU  = 1500 字节（以太网载荷上限）
       ├── IP 头 20 字节
       ├── TCP 头 20 字节
       └── MSS = 1500 − 20 − 20 = 1460 字节（TCP 能承载的应用数据上限）
```

**MSS 在三次握手时协商**：双方在 SYN 包的 Options 里告知自己能接收的最大段大小，取较小值。

**为什么这是排障重点**：任何隧道封装都会吃掉 MTU。

| 封装 | 额外开销 | 剩余 MTU |
|:--|:--|:--|
| GRE | 24 字节 | 1476 |
| IPsec 传输模式 (ESP) | ~ 38 字节 | ~1462 |
| IPsec 隧道模式 (ESP) | ~ 58 字节 | ~1442 |
| GRE over IPsec | ~ 82 字节 | ~1418 |
| VXLAN | 50 字节 | 1450 |
| MPLS（每个标签） | 4 字节 | 1496 |
| 802.1Q VLAN 标签 | 4 字节 | 1496 |

**经典故障**：建了 GRE 隧道，`ping` 通、`ssh` 能连（小包），但**打开网页卡住、复制大文件失败**（大包）。

原因：应用发了 1500 字节的包，设了 DF 位（现代 OS 都会设，为了做 PMTUD），到了隧道口发现装不下，路由器回一个 ICMP "Fragmentation Needed"。但如果**中间某台设备的 ACL 把 ICMP 全拦了**，源端收不到这个通知，就会一直重发大包，一直失败——这叫 **PMTUD 黑洞**。

**解法**：
```cisco
! 在隧道接口上强制修改 TCP MSS（最常用、最有效）
R1(config)# interface Tunnel0
R1(config-if)# ip tcp adjust-mss 1360
R1(config-if)# ip mtu 1400

! 并且：不要在 ACL 里无脑拦截所有 ICMP
R1(config)# access-list 101 permit icmp any any unreachable
```

`ip tcp adjust-mss` 会让路由器**改写经过的 TCP SYN 包里的 MSS 选项**，强制两端使用更小的段大小，从根本上避免产生超大包。这是网络工程师最常用的救命命令之一。

---

## ③ 配置与验证命令

### Cisco

```cisco
! ── 用 telnet 测试端口连通性（最实用的传输层测试手段）──
R1# telnet 192.168.2.100 80
R1# telnet 192.168.2.100 443 /source-interface GigabitEthernet0/0
! 出现 "Open" = 端口通；"Connection refused" = 收到 RST（服务没起）
! 长时间无响应 = 被防火墙 drop 了

! ── 查看设备自身的连接 ──
R1# show tcp brief
R1# show tcp brief all
R1# show control-plane host open-ports    ! 本机监听了哪些端口

! ── MSS/MTU 调整 ──
R1(config-if)# ip mtu 1400
R1(config-if)# ip tcp adjust-mss 1360
R1(config-if)# mtu 9000                   ! 巨帧（物理 MTU）

! ── 测试 MTU（关键排障手段）──
R1# ping 192.168.2.1 size 1500 df-bit
R1# ping 192.168.2.1 size 1400 df-bit
! 二分法逼近实际可用 MTU

! ── 基于端口的 ACL ──
R1(config)# ip access-list extended WEB-ONLY
R1(config-ext-nacl)# permit tcp any host 192.168.2.100 eq 80
R1(config-ext-nacl)# permit tcp any host 192.168.2.100 eq 443
R1(config-ext-nacl)# permit tcp any host 192.168.2.100 range 8000 8100
R1(config-ext-nacl)# deny ip any any log

! ── 抓包（IOS 内嵌抓包，EPC）──
R1# monitor capture CAP interface Gi0/0 both
R1# monitor capture CAP match ipv4 any any
R1# monitor capture CAP start
R1# monitor capture CAP stop
R1# show monitor capture CAP buffer brief
R1# monitor capture CAP export flash:cap.pcap
```

### Linux 侧常用（排障搭档）

```bash
# 测端口
nc -zv 192.168.2.100 80
curl -v telnet://192.168.2.100:80

# 看连接状态分布
ss -tan | awk '{print $1}' | sort | uniq -c

# 抓包看握手
tcpdump -i eth0 -nn 'tcp port 80 and (tcp[tcpflags] & (tcp-syn|tcp-rst) != 0)'

# 测 MTU
ping -M do -s 1472 192.168.2.1     # 1472 + 8(ICMP) + 20(IP) = 1500
```

### 三厂商对照

| 目的 | Cisco | H3C | 华为 |
|:--|:--|:--|:--|
| 测端口 | `telnet <ip> <port>` | `telnet <ip> <port>` | `telnet <ip> <port>` |
| 调 MSS | `ip tcp adjust-mss 1360` | `tcp mss 1360` | `tcp adjust-mss 1360` |
| 接口 MTU | `ip mtu 1400` | `mtu 1400` | `mtu 1400` |
| 带 DF 位 ping | `ping ... df-bit size 1500` | `ping -f -s 1500 <ip>` | `ping -f -s 1500 <ip>` |
| 查 TCP 连接 | `show tcp brief` | `display tcp status` | `display tcp status` |

---

## ④ 配套实验：用 telnet + 抓包区分网络问题与应用问题

**目标**：建立一套可复用的"分层定位"操作流程。

**环境**：一台路由器 R1，一台 Linux 服务器（192.168.2.100）。

### 场景 A：服务未启动

**在服务器上确保 80 端口没有服务**：`systemctl stop nginx`

```cisco
R1# ping 192.168.2.100
!!!!!                          ← 三层通

R1# telnet 192.168.2.100 80
Trying 192.168.2.100, 80 ... 
% Connection refused by remote host     ← 收到 RST
```

**抓包应该看到**：
```
192.168.1.1 → 192.168.2.100  TCP  [SYN]
192.168.2.100 → 192.168.1.1  TCP  [RST, ACK]     ← 明确拒绝
```

**结论**：网络路径完全没问题，**服务端没监听这个端口**。直接找应用团队。

### 场景 B：防火墙 drop

**在中间路由器加 ACL 静默丢弃**：
```cisco
R2(config)# ip access-list extended BLOCK
R2(config-ext-nacl)# deny tcp any host 192.168.2.100 eq 80
R2(config-ext-nacl)# permit ip any any
R2(config)# interface Gi0/1
R2(config-if)# ip access-group BLOCK in
```

```cisco
R1# ping 192.168.2.100
!!!!!                          ← 依然通（ACL 没拦 ICMP）

R1# telnet 192.168.2.100 80
Trying 192.168.2.100, 80 ...
% Connection timed out; remote host not responding    ← 超时，不是 refused
```

**抓包应该看到**：
```
192.168.1.1 → 192.168.2.100  TCP  [SYN]
192.168.1.1 → 192.168.2.100  TCP  [SYN]  (重传)
192.168.1.1 → 192.168.2.100  TCP  [SYN]  (重传)
（没有任何回应）
```

**结论**：**有中间设备静默丢弃**。用 `show access-lists` 看命中计数确认：
```cisco
R2# show access-lists BLOCK
Extended IP access list BLOCK
    10 deny tcp any host 192.168.2.100 eq www (15 matches)    ← 找到了
    20 permit ip any any (2341 matches)
```

### 场景 C：MTU 黑洞

**在 R1-R2 之间建 GRE 隧道，不调 MSS**：
```cisco
R1(config)# interface Tunnel0
R1(config-if)# ip address 10.10.10.1 255.255.255.252
R1(config-if)# tunnel source Gi0/1
R1(config-if)# tunnel destination <R2的IP>
! 故意不配 ip tcp adjust-mss
```

**测试**：
```cisco
R1# ping 192.168.2.100 size 100
!!!!!                                    ← 小包通

R1# ping 192.168.2.100 size 1500 df-bit
M.M.M                                    ← 大包不通，M 表示需要分片但 DF 置位
```

从 PC 上访问服务器的网页：**页面卡住加载不完**。

**修复**：
```cisco
R1(config-if)# ip mtu 1400
R1(config-if)# ip tcp adjust-mss 1360
```

再测，网页正常。

### 实验结论表（把它贴在你的工位上）

| ping | telnet 端口 | 现象 | 结论 |
|:--|:--|:--|:--|
| ✅ | ✅ Open | 业务仍异常 | **应用层问题**，看日志 |
| ✅ | ❌ Connection refused | 收到 RST | **服务端未监听**，或服务崩了 |
| ✅ | ❌ 超时无响应 | SYN 无回应 | **中间设备 drop**，查 ACL/防火墙 |
| ✅ | ✅ 但大文件失败 | 小包通大包不通 | **MTU 问题**，调 MSS |
| ❌ | ❌ | 都不通 | **三层问题**，回去查路由 |

---

## ⑤ 排障思路

### 症状 → 怀疑点 → 验证 → 根因

| 症状 | 验证 | 根因 |
|:--|:--|:--|
| ping 通，业务不通 | `telnet <ip> <port>` | 端口未开 / 防火墙 / 应用故障 |
| 立即 "Connection refused" | 抓包看 RST | 服务未启动，或被 reject 策略拒绝 |
| 连接超时无响应 | 抓包看有无 SYN 回应 | 中间 drop（ACL/防火墙/安全组） |
| 网页打开一半卡住 | `ping size 1500 df-bit` | **MTU/MSS 问题**（隧道场景高发） |
| 连接建立后随机断开 | 抓包看 RST 来源方向 | 防火墙会话超时 / 应用超时 / keepalive 太长 |
| 只有大文件传输失败 | 同上 | MTU |
| DNS 有时解析有时不解析 | `dig +tcp` 对比 | **DNS 响应超过 512 字节需走 TCP**，防火墙只放了 UDP 53 |
| BGP 邻居起不来 | `telnet <peer> 179` | TCP 179 被拦 / 源地址不对 / TTL 不够（eBGP 多跳） |

### DNS 的 TCP/UDP 陷阱（实战高频）

DNS 查询默认用 **UDP 53**。但：
- 响应超过 512 字节（未启用 EDNS0）→ 自动降级到 **TCP 53** 重试
- **区域传送（AXFR/IXFR）** 一律用 TCP 53
- DNSSEC 签名的响应通常很大 → 更容易走 TCP

**故障表现**：多数域名解析正常，但某几个域名（记录多、有 DNSSEC）解析失败。

**根因**：防火墙 ACL 里只写了 `permit udp any any eq 53`，漏了 TCP。

**正确写法**：
```cisco
permit udp any any eq domain
permit tcp any any eq domain      ← 别漏
```

---

## ⑥ 考点提示 + 自测题

### 考点

- **端口号必背**，尤其网工相关的 179/500/4500/1812/49/161/514/123/830。
- **三次握手/四次挥手**的原因，是理解题不是记忆题。
- **MTU/MSS 与隧道封装的关系**是 ENARSI 排障的重灾区，务必吃透。
- **RST vs 超时** 的区分能力，是区分初级和中级运维的分水岭。
- **DNS 需要同时放行 TCP 和 UDP 53**。

### 自测题

**1.** 为什么 TCP 握手三次就够，挥手却要四次？

<details><summary>答案</summary>

**握手时 ACK 和 SYN 可以合并**：服务器一收到连接请求，就能立即决定接受，所以"确认你的序号"和"告知我的序号"可以打包在同一个报文里（SYN+ACK）。

**挥手时不能合并**：客户端说"我数据发完了"（FIN），服务器必须立刻确认（ACK），但服务器**自己可能还有数据没发完**。所以服务器不能在回 ACK 的同时就发 FIN。等它的数据也传完了，才单独发 FIN。这个中间阶段叫**半关闭（Half-Close）**。

**特例**：如果服务器恰好也没数据要发了，某些实现会把 ACK 和 FIN 合并，变成"三次挥手"。这是合法的优化，抓包时偶尔能见到。
</details>

**2.** 用户反馈"网页能打开首页，但点进去有大图的页面就卡住"。ping 服务器完全正常。最可能是什么问题？

<details><summary>答案</summary>

**MTU/MSS 问题**，路径上大概率存在隧道封装（GRE / IPsec / VPN / VXLAN），并且 PMTUD 被破坏（中间设备拦了 ICMP unreachable）。

**为什么首页能开、大图不行**：首页 HTML 很小，几个包就传完，每个包都在 MSS 以内；大图会触发大量满载的 1460 字节段，超过隧道的实际承载能力就被丢了。

**验证**：
```cisco
R1# ping <服务器IP> size 1500 df-bit      ← 不通
R1# ping <服务器IP> size 1300 df-bit      ← 通
```
二分逼近，找到实际可用 MTU。

**修复**：在隧道接口配 `ip tcp adjust-mss`（通常设为 实际MTU − 40）。同时检查是否有 ACL 拦了 `icmp unreachable`，把它放行。
</details>

**3.** `telnet 192.168.1.100 3306` 立即返回 "Connection refused"，和一直卡住超时，分别说明什么？

<details><summary>答案</summary>

**Connection refused（立即）**：
- 客户端发了 SYN，**收到了 RST**
- 说明**包成功到达了目标主机**，网络路径完全通畅
- 目标主机的 3306 端口**没有进程监听**（服务没起 / 服务崩了 / 监听在 127.0.0.1 而非 0.0.0.0）
- 也可能是中间防火墙配置为 `reject` 而非 `drop`（会主动回 RST），但这种配置较少见
- **结论：网络没问题，去查应用**

**超时无响应（卡住十几秒）**：
- 客户端发了 SYN，**什么都没收到**（会重传 3~5 次）
- 说明包在路上被**静默丢弃**了
- 常见于：ACL deny、防火墙 drop 策略、云安全组、目标主机 iptables DROP、路由黑洞
- **结论：网络路径有阻断，去查 ACL / 防火墙 / 安全组**

**这是运维排障最快的分水岭判断，五秒钟就能决定该找谁。**
</details>

**4.** 一台设备的 ACL 只写了 `permit udp any any eq 53`，用户报告"大部分网站能上，但公司内部某个系统的域名解析不了"。为什么？

<details><summary>答案</summary>

**DNS 响应超过 512 字节时会切换到 TCP 53**，而 ACL 没放行 TCP 53。

触发条件：
- 该域名有很多条记录（比如一个域名配了 10+ 个 A 记录做负载均衡）
- 启用了 DNSSEC（签名数据很大）
- 有很长的 TXT 记录（如 SPF/DKIM 记录）
- 客户端发起了区域传送（AXFR）

DNS 服务器会先用 UDP 回一个带 **TC（Truncated）标志**的响应，客户端看到这个标志后**改用 TCP 53 重新查询**。TCP 被 ACL 拦掉 → 解析失败。

**修复**：
```cisco
permit udp any any eq domain
permit tcp any any eq domain
```

**这是实战中非常常见的坑**，因为它表现为"随机的、只影响个别域名"的诡异故障，很难联想到 ACL。
</details>

**5.** 为什么 BGP 用 TCP 179，而 OSPF 直接跑在 IP 之上不用传输层？

<details><summary>答案</summary>

**BGP 用 TCP 的理由**：
1. **数据量巨大**：互联网全表 90 万+ 条路由，一次全量更新可能几十 MB，必须有可靠的分片、排序、重传、流控。自己实现这套机制不划算。
2. **邻居可以不直连**：eBGP 常跨多跳（比如通过环回口建邻居），TCP 天然支持跨网络的端到端连接；而 OSPF/EIGRP 的邻居必须在同一链路上。
3. **需要长连接维持**：BGP 会话一建立就长期保持，TCP 的连接语义正好匹配。

**OSPF 不用 TCP 的理由**：
1. **自己实现了可靠性**：OSPF 有显式的 LSAck 确认机制和重传定时器（RxmtInterval，默认 5 秒），针对 LSA 泛洪场景做了专门优化，比通用 TCP 更高效。
2. **需要组播**：OSPF 用 `224.0.0.5`（所有 OSPF 路由器）和 `224.0.0.6`（DR/BDR）**一次发给链路上所有邻居**。TCP 是点对点的，做不到组播——如果用 TCP，一个链路上有 10 个邻居就要维护 10 条连接，开销大得多。
3. **邻居必然直连**：不需要跨网络传输能力。

**设计启示**：可靠性可以在协议栈的任何一层实现。选择在哪一层做，取决于你对**性能、灵活性、实现复杂度**的权衡。BGP 选择"复用现成的 TCP"，OSPF 选择"自己做以换取组播和效率"——两个都是正确答案，因为它们面对的场景不同。
</details>

---

**上一章** ← [04 静态路由与默认网关](04-静态路由与默认网关.md) ｜ **下一章** → [06 Cisco IOS 操作入门](06-Cisco-IOS操作入门.md)
