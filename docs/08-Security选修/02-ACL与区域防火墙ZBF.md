# 02 · ACL 与区域防火墙 ZBF

## ① 这章解决什么问题

**场景**：公司有一台路由器做出口，你在上面配了 ACL：

```cisco
ip access-list extended OUTSIDE-IN
 deny ip any any
interface Gi0/0
 ip access-group OUTSIDE-IN in
```

**结果：内网用户上不了网了。**

你想："我只是禁止外面进来啊，为什么内网出去也不行了？"

```
   ★ 因为 ACL 是【无状态】的 ★
   
   内网 PC → 外网服务器（请求包）
        ↓ 出去了，没问题
   外网服务器 → 内网 PC（响应包）
        ↓ 从 Gi0/0 【入方向】进来
        ↓ 撞上 deny ip any any
   ★★ 响应包被丢了 ★★
   
   ACL 不知道"这是我刚才发出去那个请求的响应"
   它只是逐包匹配规则，★ 没有记忆 ★
```

**这一章讲的就是：怎么从"无记忆的包过滤"进化到"有记忆的状态检测"。**

---

## ② 原理讲透

### 2.1 三代防火墙技术

| 代 | 技术 | 判断依据 | 局限 |
|:--|:--|:--|:--|
| **一代** | **包过滤（ACL）** | 五元组：源/目 IP、源/目端口、协议 | ★ **无状态**，响应包必须手工放行 |
| **二代** | **状态检测（Stateful）** | 五元组 + **连接状态表** | 不看应用层内容 |
| **三代** | **应用层网关 / NGFW** | + 应用识别、用户身份、内容检测 | 性能开销大 |

### 2.2 ★ 状态检测的核心：会话表 ★

```
   ★ 内网 PC 主动发起连接 ★
   192.168.1.10:54321 → 8.8.8.8:443 (TCP SYN)
        ↓
   防火墙检查策略：inside → outside 允许
        ↓
   ★★ 在会话表中创建一条记录 ★★
   ┌──────────────────────────────────────────────┐
   │ 192.168.1.10:54321 ↔ 8.8.8.8:443  TCP  ESTAB │
   │ 空闲超时：3600s                                │
   └──────────────────────────────────────────────┘
        ↓
   ★ 响应包回来时 ★
   8.8.8.8:443 → 192.168.1.10:54321 (SYN-ACK)
        ↓
   防火墙查会话表 → ★ 命中！★
        ↓
   ★ 直接放行，不需要 outside → inside 的策略 ★
```

**★ 这就是状态检测和 ACL 的根本区别**：
```
   ACL：     每个包独立判断，★ 双向都要写规则 ★
   状态检测：★ 只需要为【发起方向】写策略 ★，
             响应流量自动放行
```

### 2.3 无状态 ACL 的传统"补丁"

在没有状态检测的年代，工程师用两种办法：

**① `established` 关键字（只能用于 TCP）**
```cisco
ip access-list extended OUTSIDE-IN
 ★ permit tcp any 192.168.1.0 0.0.0.255 established ★
 deny ip any any log
```
**原理**：只放行 **ACK 或 RST 标志位置位**的 TCP 包（即"不是新连接的第一个包"）。

**⚠️ 缺陷**：
- **只对 TCP 有效**，UDP/ICMP 没辙
- **可以被伪造**：攻击者随便构造一个 ACK 包就能穿过

**② 反射 ACL（Reflexive ACL）—— 真正的"穷人版状态检测"**
```cisco
! 出方向：为每条出去的流量【动态创建】一条反向条目
ip access-list extended OUTBOUND
 permit ip 192.168.1.0 0.0.0.255 any ★ reflect MIRROR-ACL timeout 300 ★

! 入方向：先查动态生成的条目
ip access-list extended INBOUND
 ★ evaluate MIRROR-ACL ★
 permit icmp any any echo-reply
 deny ip any any log

interface Gi0/0
 ip access-group OUTBOUND out
 ip access-group INBOUND in
```

**★ 效果**：内网出去时自动生成一条精确匹配的反向放行条目，超时后自动删除。

**⚠️ 缺陷**：
- 不理解应用层协议（**FTP 主动模式过不去**）
- 配置繁琐
- 已被 ZBF 取代

---

## ③ ★ ZBF（Zone-Based Firewall）★

**Cisco IOS 上的状态检测防火墙**，从 IOS 12.4(6)T 开始。

### 3.1 核心概念：区域（Zone）

```
   ★ 传统 ACL：策略绑在【接口】上 ★
      接口多了就是灾难：10 个接口 → 90 个方向组合
   
        ↓
   
   ★ ZBF：策略绑在【区域对】上 ★
      把接口划分到区域，策略定义在 "区域A → 区域B"
      
   ┌─────────┐        ┌─────────┐        ┌─────────┐
   │ INSIDE  │        │   自    │        │ OUTSIDE │
   │ Gi0/1   │        │ (self)  │        │ Gi0/0   │
   │ Gi0/2   │        │ 路由器   │        │         │
   └────┬────┘        │  自身    │        └────┬────┘
        │             └─────────┘             │
        │                                     │
        └──── zone-pair IN→OUT ───────────────┘
              （策略写在这里）
```

### 3.2 ★★ ZBF 的三条铁律 ★★

```
   ★ 铁律 1：同一区域内的接口之间，流量【默认放行】★
      （不需要配策略）
   
   ★ 铁律 2：不同区域之间，如果【没有 zone-pair】，
              流量【默认全部丢弃】★
      （这是最容易踩的坑）
   
   ★ 铁律 3：区域接口 ↔ 未划分区域的接口，
              流量【默认全部丢弃】★
      （另一个大坑）
```

> ⚠️ **★ 这三条铁律是 ZBF 排障的起点。**
> **"配完 ZBF 就断网了"** 99% 是因为**某个接口没划分区域**，或者**漏配了某个方向的 zone-pair**。

### 3.3 特殊区域：`self`

**`self` 代表路由器自身**（管理流量：SSH、SNMP、路由协议、ping 路由器接口）。

```
   ★ self 区域的默认行为和别的区域【相反】★
   
   · 普通区域间：没有 zone-pair = 全部拒绝
   · ★ 涉及 self 的：没有 zone-pair = 【全部允许】★
   
   ★ 一旦你为 self 配了 zone-pair，
     那个方向就变成"策略说了算"，
     没匹配上的就被丢弃 ★
```

**⚠️ 这是"配完 ZBF 后 SSH 登不上"的最常见原因。**

### 3.4 ZBF 的配置结构：C3PL

**Cisco Common Classification Policy Language** —— 和 QoS 的 MQC 是同一套语法。

```
   ★ 三步走 ★
   
   ① class-map    ：★ 匹配什么流量 ★
        ↓
   ② policy-map   ：★ 对匹配的流量做什么 ★
                     (inspect / pass / drop)
        ↓
   ③ zone-pair    ：★ 把策略应用到哪个方向 ★
```

### 3.5 三个动作的区别

| 动作 | 含义 | 会话表 | 用途 |
|:--|:--|:--|:--|
| **`inspect`** | ★ **状态检测** | ★ **创建会话，响应自动放行** | ★ **绝大多数场景用这个** |
| **`pass`** | 单向放行 | ❌ **不创建会话** | ★ IPsec/GRE 等**本身无状态**的协议 |
| **`drop`** | 丢弃 | — | 显式拒绝（加 `log` 可记日志） |

**★ `pass` 和 `inspect` 的关键区别**：
```
   ★ pass ★：只放行【这个方向】，
             反向流量需要【另外配一个 zone-pair】
             
   ★ inspect ★：放行这个方向，
                ★ 反向的响应流量自动放行 ★
   
   ★ 什么时候必须用 pass？★
   · IPsec ESP（协议 50）—— 不是 TCP/UDP，没有"会话"概念
   · GRE 隧道
   · 路由协议（OSPF/EIGRP 用组播，不是会话）
   · ★ 用 pass 时记得两个方向都配 ★
```

---

## ④ 配置命令

### 4.1 完整的 ZBF 配置示例

**拓扑**：
```
   内网 192.168.1.0/24 ──[Gi0/1]── R1 ──[Gi0/0]── 互联网
                                    │
                                 [Gi0/2]
                                    │
                              DMZ 172.16.1.0/24
                              （Web 服务器 172.16.1.10）
```

**需求**：
1. 内网可以访问互联网和 DMZ
2. 互联网只能访问 DMZ 的 Web 服务（80/443）
3. DMZ 不能主动访问内网
4. 只允许内网 SSH 管理路由器

```cisco
!═══════════ 步骤 1：定义区域 ═══════════
zone security INSIDE
 description 内网
zone security OUTSIDE
 description 互联网
zone security DMZ
 description 服务器区

!═══════════ 步骤 2：接口划入区域 ═══════════
interface GigabitEthernet0/1
 zone-member security INSIDE
interface GigabitEthernet0/0
 zone-member security OUTSIDE
interface GigabitEthernet0/2
 zone-member security DMZ

!═══════════ 步骤 3：class-map 定义流量 ═══════════
! 内网出去：允许常见应用
class-map type inspect match-any CM-INSIDE-TO-OUT
 match protocol http
 match protocol https
 match protocol dns
 match protocol icmp
 match protocol ftp
 match protocol ssh
 match protocol ntp

! 互联网访问 DMZ Web：用 ACL 精确限定目的
ip access-list extended ACL-WEB-SERVER
 permit tcp any host 172.16.1.10 eq 80
 permit tcp any host 172.16.1.10 eq 443

class-map type inspect match-all CM-OUT-TO-DMZ
 ★ match access-group name ACL-WEB-SERVER ★
 match protocol http

! 内网访问 DMZ：全放
class-map type inspect match-any CM-INSIDE-TO-DMZ
 match protocol http
 match protocol https
 match protocol ssh
 match protocol icmp

! 管理流量（去 self）
ip access-list extended ACL-MGMT
 permit ip 192.168.1.0 0.0.0.255 any

class-map type inspect match-all CM-MGMT
 match access-group name ACL-MGMT
 match protocol ssh

!═══════════ 步骤 4：policy-map 定义动作 ═══════════
policy-map type inspect PM-INSIDE-TO-OUT
 class type inspect CM-INSIDE-TO-OUT
  ★ inspect ★
 class class-default
  ★ drop log ★

policy-map type inspect PM-OUT-TO-DMZ
 class type inspect CM-OUT-TO-DMZ
  inspect
 class class-default
  drop log

policy-map type inspect PM-INSIDE-TO-DMZ
 class type inspect CM-INSIDE-TO-DMZ
  inspect
 class class-default
  drop

policy-map type inspect PM-TO-SELF
 class type inspect CM-MGMT
  inspect
 class class-default
  drop log

!═══════════ 步骤 5：zone-pair 应用策略 ═══════════
zone-pair security ZP-IN-OUT source INSIDE destination OUTSIDE
 service-policy type inspect PM-INSIDE-TO-OUT

zone-pair security ZP-OUT-DMZ source OUTSIDE destination DMZ
 service-policy type inspect PM-OUT-TO-DMZ

zone-pair security ZP-IN-DMZ source INSIDE destination DMZ
 service-policy type inspect PM-INSIDE-TO-DMZ

! ★ 管理流量：内网 → 路由器自身 ★
zone-pair security ZP-IN-SELF source INSIDE destination self
 service-policy type inspect PM-TO-SELF

!═══════════ 没有配的 zone-pair = 默认拒绝 ═══════════
! DMZ → INSIDE     ：没配 → ★ 拒绝 ★（符合需求）
! OUTSIDE → INSIDE ：没配 → ★ 拒绝 ★（符合需求）
! DMZ → OUTSIDE    ：没配 → 拒绝（如果服务器要更新补丁，需要加）
```

### 4.2 验证命令

```cisco
! ★ 看区域和区域对的整体结构 ★
show zone security
show zone-pair security

! ★★ 最重要：看策略命中情况 ★★
show policy-map type inspect zone-pair
show policy-map type inspect zone-pair ZP-IN-OUT

! ★★ 看会话表（状态检测的核心）★★
show policy-firewall sessions
show policy-firewall sessions detail

! 统计
show policy-firewall stats zone-pair
show policy-firewall config
```

**`show policy-map type inspect zone-pair` 的读法**：
```
 Zone-pair: ZP-IN-OUT
  Service-policy inspect : PM-INSIDE-TO-OUT
   Class-map: CM-INSIDE-TO-OUT (match-any)
     Match: protocol http
       ★ 1523 packets, 98452 bytes ★    ← 有命中，说明匹配上了
     Inspect
       Session creations since subsystem startup: ★ 342 ★
       Current session counts (estab/half-open/terminating): ★ [12:0:1] ★
   Class-map: class-default
     ★ 87 packets ★                      ← ★ 这里有计数 = 有流量被丢了 ★
     Drop                                   ★ 排障时先看这个 ★
```

### 4.3 三厂商对照

| 功能 | **Cisco IOS (ZBF)** | **H3C (Comware 安全域)** | **华为 (USG/AR)** |
|:--|:--|:--|:--|
| 创建区域 | `zone security INSIDE` | `security-zone name Trust` | `firewall zone trust` |
| 接口入域 | `zone-member security INSIDE`（接口下） | `import interface Gi1/0/1`（域视图下） | `add interface Gi0/0/1`（域视图下） |
| 区域优先级 | 无（靠 zone-pair） | 无 | ★ `set priority 85`（数值大=可信） |
| 策略 | `zone-pair` + `policy-map type inspect` | `security-policy ip` + `rule` | `security-policy` + `rule name` |
| 状态检测 | `inspect` | 默认开启（`session` 表） | 默认开启 |
| 查会话 | `show policy-firewall sessions` | `display session table` | `display firewall session table` |
| 默认策略 | ★ 无 zone-pair = 拒绝 | ★ 默认拒绝 | ★ 默认拒绝（`default action deny`） |

**★ 华为的区域优先级逻辑（和 Cisco 不同）**：
```
   华为预置四个区域：
   · local    优先级 100  （设备自身，相当于 Cisco 的 self）
   · trust    优先级 85   （内网）
   · dmz      优先级 50   （服务器区）
   · untrust  优先级 5    （外网）
   
   ★ 高优先级 → 低优先级 = Outbound 方向
   ★ 低优先级 → 高优先级 = Inbound 方向
   
   ★ 但现在（V500 以后）也是【默认全部拒绝】，
     必须显式写 security-policy ★
```

**华为配置示例**：
```
firewall zone trust
 set priority 85
 add interface GigabitEthernet0/0/1

firewall zone untrust
 set priority 5
 add interface GigabitEthernet0/0/0

security-policy
 rule name trust_to_untrust
  source-zone trust
  destination-zone untrust
  source-address 192.168.1.0 mask 255.255.255.0
  action permit
```

**H3C 配置示例**：
```
security-zone name Trust
 import interface GigabitEthernet1/0/1

security-zone name Untrust
 import interface GigabitEthernet1/0/0

object-policy ip Trust-Untrust
 rule 0 pass

zone-pair security source Trust destination Untrust
 object-policy apply ip Trust-Untrust
```

---

## ⑤ 配套实验

### 实验目标
在 EVE-NG 中搭建三区域 ZBF，验证状态检测行为，并注入故障。

### 拓扑
```
   PC1 (192.168.1.10)
       │
   [Gi0/1] INSIDE
       R1  ──[Gi0/2] DMZ── SRV (172.16.1.10, 起 HTTP)
   [Gi0/0] OUTSIDE
       │
   PC2 (203.0.113.10)  ← 模拟互联网
```

### 步骤

**① 基础连通**（不配 ZBF）
```
   PC1 ping SRV      → ✅ 通
   PC1 ping PC2      → ✅ 通
   PC2 ping PC1      → ✅ 通  ← ★ 这就是问题：外网能主动进内网 ★
```

**② 应用上面 §4.1 的完整 ZBF 配置**

**③ 验证状态检测**
```
   PC1 → PC2 telnet 80  → ✅ 通（策略允许 + 状态检测放行响应）
   ★ PC2 → PC1 ping     → ❌ 不通（没有 OUTSIDE→INSIDE 的 zone-pair）★
   PC2 → SRV curl 80    → ✅ 通（有 OUTSIDE→DMZ 策略）
   ★ SRV → PC1 ping     → ❌ 不通（没有 DMZ→INSIDE）★
```

**看会话表**：
```cisco
R1# show policy-firewall sessions
Session ID 0x0000000A (192.168.1.10:54321)=>(203.0.113.10:80) tcp SIS_OPEN
  Created 00:00:15, Last heard 00:00:02
  Bytes sent (initiator:responder) [520:1840]
```
★ **这一行就是"防火墙的记忆"**——它记住了这条连接，所以响应包能回来。★

**④ ★★ 故障注入（本实验最有价值的部分）★★**

| # | 注入操作 | 预期症状 | 训练什么 |
|:--|:--|:--|:--|
| **F1** | 在 Gi0/2 上 `no zone-member security DMZ` | ★ **DMZ 完全不通**（连内网都访问不了） | **铁律 3**：区域接口 ↔ 无区域接口 = 全丢 |
| **F2** | 删掉 `ZP-IN-SELF` | ★ **SSH 还能登**（self 无 zone-pair = 允许） | 理解 self 的**反向默认** |
| **F3** | 把 `PM-TO-SELF` 里的 `inspect` 改成 `drop` | ★ **SSH 立刻断，且救不回来**（除非 Console） | ★ **self 策略的杀伤力** |
| **F4** | 在 `CM-INSIDE-TO-OUT` 里删掉 `match protocol dns` | ★ **能 ping 通 IP，但域名解析全失败** | **"网通但业务不通"**的经典形态 |
| **F5** | 在 R1 和 R2 间跑 GRE，zone-pair 用 `inspect` | ★ **GRE 隧道起不来** | ★ **无状态协议必须用 `pass`** |
| **F6** | 把 `class-default` 的 `drop log` 改成 `drop`（去掉 log） | 症状不变，但**排障时看不到日志** | 日志的价值 |

**★ F4 是最贴近真实工单的**：
```
   用户报："网断了"
   你 ping 8.8.8.8 → ✅ 通
   你说："网没断啊"
   用户："可是网页打不开！"
        ↓
   ★ nslookup 一试 → DNS 超时 ★
        ↓
   ★ show policy-map type inspect zone-pair
     → class-default 的 drop 计数在飞涨 ★
        ↓
   ★ 根因：DNS 没在放行列表里 ★
```

---

## ⑥ 排障思路

### 6.1 ★ ZBF 排障的固定套路 ★

```
   ① ★ 先看 class-default 的 drop 计数 ★
      show policy-map type inspect zone-pair
      → 计数在涨 = 有流量被策略丢了 → 去看是什么流量
      → 计数不涨 = ★ 流量根本没到这个 zone-pair ★
                    → 可能是路由问题，或走了别的 zone-pair，
                      或者根本没有对应的 zone-pair（默认丢弃，不计数）
        ↓
   ② ★ 确认接口的区域归属 ★
      show zone security
      → ★ 有没有接口忘了划区域？★（最高频的坑）
        ↓
   ③ ★ 确认 zone-pair 是否存在 ★
      show zone-pair security
      → ★ 缺哪个方向？★
      → ⚠️ zone-pair 是【单向】的，
           A→B 和 B→A 是两条独立的 zone-pair
        ↓
   ④ ★ 看会话表 ★
      show policy-firewall sessions
      → 有会话 = 连接建起来了，问题在别处（路由/应用）
      → 无会话 = 连初始包都没放行
        ↓
   ⑤ 用 debug 精确定位
      ip access-list extended ACL-DEBUG
       permit ip host 192.168.1.10 host 203.0.113.10
      debug policy-firewall detail
      （⚠️ 生产环境务必先配 ACL 限定范围）
```

### 6.2 症状 → 根因速查表

| 症状 | 高频根因 | 验证命令 |
|:--|:--|:--|
| ★ **配完 ZBF 整个网断了** | ★ **有接口没划区域**（铁律 3） | `show zone security` |
| ★ **SSH 登不上路由器** | ★ **配了 self 的 zone-pair 但没放行 SSH** | `show zone-pair security \| i self` |
| **能 ping 通但业务不通** | ★ **class-map 里缺对应的 `match protocol`** | `show policy-map type inspect zone-pair` |
| **能上网但打不开网页** | ★ **DNS 没放行** | 同上 + `nslookup` 测试 |
| **GRE / IPsec 隧道起不来** | ★ **用了 `inspect` 而不是 `pass`** | `show policy-firewall sessions` 无会话 |
| **路由邻居断了** | ★ **组播的路由协议报文被丢**（需要 `pass` 或放行 self） | `show ip ospf neighbor` |
| **FTP 传输失败但能登录** | **ALG/inspect 没识别 FTP 数据通道** | `match protocol ftp`（必须用协议名，不能只写端口） |
| **单向通，反向不通** | ★ **用了 `pass` 但只配了一个方向** | `show zone-pair security` |
| **会话数暴涨、内存告警** | **超时时间太长 / 有扫描攻击** | `show policy-firewall sessions` 计数 |

### 6.3 ★ 一个真实案例 ★

**症状**：新上线的 ZBF，白天正常，**晚上 22:00 后备份任务全部失败**。

**排查**：
```
   ① 备份服务器在 DMZ，要往内网 NAS 写数据
      → DMZ → INSIDE 方向
        ↓
   ② show zone-pair security
      → ★ 根本没有 DMZ → INSIDE 的 zone-pair ★
      → 默认拒绝
        ↓
   ③ 为什么白天没人发现？
      → ★ 白天是内网【主动】访问 DMZ（INSIDE→DMZ 有策略）
         状态检测让响应回得来
         而备份是 DMZ【主动】发起的 ★
        ↓
   ★★ 根因：混淆了"能通"和"能主动发起" ★★
```

**修复**：
```cisco
ip access-list extended ACL-BACKUP
 permit tcp host 172.16.1.20 host 192.168.1.100 eq 445
 permit tcp host 172.16.1.20 host 192.168.1.100 eq 2049

class-map type inspect match-all CM-DMZ-TO-IN
 match access-group name ACL-BACKUP

policy-map type inspect PM-DMZ-TO-IN
 class type inspect CM-DMZ-TO-IN
  inspect
 class class-default
  drop log

zone-pair security ZP-DMZ-IN source DMZ destination INSIDE
 service-policy type inspect PM-DMZ-TO-IN
```

**★ 教训**：
```
   ★ 部署 ZBF 之前，必须先画出【所有业务的发起方向】★
   
   不要只想"谁能访问谁"，
   要想"★ 谁【主动】访问谁 ★"
   
   容易漏掉的反向主动流量：
   · 备份任务（服务器 → 存储）
   · 监控回传（Agent → 服务器）
   · 日志上报（设备 → Syslog）
   · 补丁更新（服务器 → 互联网）
   · 数据库同步
   · ★ 打印机回连（打印服务器 → 打印机）★
```

---

## ⑦ 考点提示

```
   ★ ENCOR 考点 ★
   · ACL 是无状态的，ZBF 是有状态的
   · ZBF 的三条默认规则（尤其是"无 zone-pair = 拒绝"）
   · self 区域的反向默认行为
   · inspect / pass / drop 的区别和适用场景
   
   ★ 高频陷阱 ★
   · zone-pair 是【单向】的
   · 同区域内接口默认互通
   · 区域接口 ↔ 无区域接口 = 全丢
   · IPsec/GRE 必须用 pass
   · self 的 zone-pair 一旦配了，没匹配的就丢
```

**★ 必背的对比**：

| | ACL | 反射 ACL | ZBF |
|:--|:--|:--|:--|
| 状态 | ❌ | ⚠️ 伪状态 | ★ ✅ |
| 应用层识别 | ❌ | ❌ | ★ ✅（`match protocol`） |
| 配置粒度 | 接口方向 | 接口方向 | ★ **区域对** |
| FTP 主动模式 | 需手工放高端口 | ❌ | ★ ✅ 自动 |
| 默认行为 | 隐含 deny any | — | ★ 无 zone-pair = deny |

---

## ⑧ 自测题

**1.** 为什么下面这条 ACL 会导致内网上不了网？正确的做法有哪几种？

```cisco
ip access-list extended OUTSIDE-IN
 deny ip any any
interface Gi0/0
 ip access-group OUTSIDE-IN in
```

<details><summary>答案</summary>

**★ 根因：ACL 是无状态的。**

```
   内网 PC → 外网（请求包）：从 Gi0/0 【出方向】走，ACL 没管
        ↓
   外网 → 内网 PC（响应包）：从 Gi0/0 【入方向】进
        ↓
   ★ 撞上 deny ip any any → 被丢弃 ★
   
   ACL 不记得"这是我刚发出去那个请求的响应"，
   它只是【逐包】匹配规则。
```

**三种解决办法（由差到好）**：

**① `established` 关键字**
```cisco
ip access-list extended OUTSIDE-IN
 ★ permit tcp any 192.168.1.0 0.0.0.255 established ★
 permit icmp any any echo-reply
 permit udp any eq 53 192.168.1.0 0.0.0.255
 deny ip any any log
```
⚠️ **缺陷**：只对 TCP 有效；**能被伪造**（构造带 ACK 位的包就能穿过）；UDP/ICMP 要手工逐个放行。

**② 反射 ACL（Reflexive ACL）**
```cisco
ip access-list extended OUTBOUND
 permit ip 192.168.1.0 0.0.0.255 any ★ reflect MIRROR timeout 300 ★
ip access-list extended INBOUND
 ★ evaluate MIRROR ★
 deny ip any any log
interface Gi0/0
 ip access-group OUTBOUND out
 ip access-group INBOUND in
```
✅ 对 TCP/UDP/ICMP 都有效，动态精确匹配。
⚠️ **缺陷**：不理解应用层（**FTP 主动模式过不去**），配置繁琐。

**③ ★ ZBF（推荐）★**
```cisco
zone security INSIDE
zone security OUTSIDE
interface Gi0/1
 zone-member security INSIDE
interface Gi0/0
 zone-member security OUTSIDE

class-map type inspect match-any CM-OUT
 match protocol http
 match protocol https
 match protocol dns
 match protocol icmp

policy-map type inspect PM-OUT
 class type inspect CM-OUT
  ★ inspect ★
 class class-default
  drop log

zone-pair security ZP-IN-OUT source INSIDE destination OUTSIDE
 service-policy type inspect PM-OUT
```
✅ 真正的状态检测；✅ 应用层识别（FTP/SIP 的数据通道自动处理）；✅ **只需要为发起方向写策略**。

> ⚠️ **注意**：配 ZBF 时**别忘了 self**——不配 self 的 zone-pair 反而是安全的（默认允许），一旦配了就必须显式放行 SSH。
</details>

**2.** ZBF 的 `inspect` 和 `pass` 有什么区别？什么情况下必须用 `pass`？

<details><summary>答案</summary>

| | **`inspect`** | **`pass`** |
|:--|:--|:--|
| 状态检测 | ★ **是** | ❌ 否 |
| 会话表 | ★ **创建条目** | ❌ 不创建 |
| 反向流量 | ★ **自动放行** | ❌ **需要单独配反向 zone-pair** |
| 应用层识别 | ✅（FTP/SIP 数据通道） | ❌ |
| 适用 | ★ **TCP/UDP/ICMP 等有会话概念的** | ★ **无会话概念的协议** |

**★ 必须用 `pass` 的场景**：

**① IPsec ESP / AH**
```
   ESP 是 IP 协议号 50，★ 不是 TCP/UDP，没有端口，没有"会话" ★
   inspect 无法为它建立会话表条目 → 隧道起不来
```
```cisco
ip access-list extended ACL-VPN
 permit esp any any
 permit udp any any eq 500      ! IKE
 permit udp any any eq 4500     ! NAT-T

class-map type inspect match-all CM-VPN
 match access-group name ACL-VPN

policy-map type inspect PM-VPN
 class type inspect CM-VPN
  ★ pass ★
 class class-default
  drop

! ★ 两个方向都要配 ★
zone-pair security ZP-OUT-SELF source OUTSIDE destination self
 service-policy type inspect PM-VPN
zone-pair security ZP-SELF-OUT source self destination OUTSIDE
 service-policy type inspect PM-VPN
```

**② GRE 隧道**（IP 协议号 47）—— 同理。

**③ 路由协议**
```
   OSPF（协议 89，组播 224.0.0.5）
   EIGRP（协议 88，组播 224.0.0.10）
        ↓
   ★ 组播、无连接、双向独立发送 ★
   → inspect 建不了会话
   → 必须 pass，且两个方向都配（或者干脆不为 self 配 zone-pair）
```

**★ 记忆口诀**：
```
   ★ 有"请求-响应"配对关系的 → inspect ★
   ★ 单向发、各发各的、非 TCP/UDP 的 → pass（双向都配）★
```

**⚠️ 常见错误**：
```
   工程师配了：
     zone-pair OUTSIDE → self  用 pass 放行 ESP
   然后 VPN 还是不通
        ↓
   ★ 因为只配了一个方向 ★
   路由器【自己发出去】的 ESP（self → OUTSIDE）没有策略
        ↓
   ★ pass 不建会话，所以两个方向必须【各配一次】★
```
</details>

**3.** ZBF 的三条默认规则是什么？"配完 ZBF 后整个网络断了"最可能是哪一条导致的？

<details><summary>答案</summary>

**★ 三条铁律**：

```
   ★ 铁律 1 ★
   同一区域内的接口之间 → 【默认放行】
   （不需要 zone-pair）
   
   ★ 铁律 2 ★
   不同区域之间，没有 zone-pair → 【默认全部丢弃】
   
   ★ 铁律 3 ★
   区域接口 ↔ 未划分区域的接口 → 【默认全部丢弃】
```

**★ 补充：`self` 区域是例外**
```
   涉及 self 的流量，如果没有 zone-pair → ★ 默认【允许】★
   （和铁律 2 相反）
   
   但一旦为 self 配了 zone-pair，
   那个方向就变成"策略说了算"，没匹配的就丢。
```

**★ "配完 ZBF 整个网断了" → 99% 是【铁律 3】★**

**典型场景**：
```
   路由器有 5 个接口：
   · Gi0/0  → OUTSIDE  ✅ 划了
   · Gi0/1  → INSIDE   ✅ 划了
   · Gi0/2  → DMZ      ✅ 划了
   · Gi0/3  → ★ 忘了划 ★
   · Lo0    → ★ 忘了划 ★（管理环回口）
        ↓
   ★ Gi0/3 和 Lo0 上的所有流量全部被丢弃 ★
   （因为它们是"无区域接口"，和任何区域接口之间都不通）
        ↓
   症状：
   · Gi0/3 后面的分支网络完全失联
   · 用 Lo0 做源的 Syslog/NetFlow/SNMP 全部发不出去
   · 路由协议如果用 Lo0 做 router-id 或 source 也会异常
```

**排查命令**：
```cisco
R1# show zone security
zone self
  Description: System defined zone
zone INSIDE
  Member Interfaces:
    GigabitEthernet0/1
zone OUTSIDE
  Member Interfaces:
    GigabitEthernet0/0
zone DMZ
  Member Interfaces:
    GigabitEthernet0/2

! ★ 对比接口列表，找出没出现在上面的接口 ★
R1# show ip interface brief
```

**修复的两种思路**：
```
   ① 给遗漏的接口划区域
      interface Gi0/3
       zone-member security BRANCH
      （然后配相应的 zone-pair）
   
   ② ★ 如果某些接口不需要防火墙，
        把它们放进【同一个区域】★
        （铁律 1：同区域内默认互通）
      interface Gi0/3
       zone-member security INSIDE
      interface Lo0
       zone-member security INSIDE
```

**★ 部署 ZBF 的标准流程（防止踩这个坑）**：
```
   ① show ip interface brief → ★ 列出所有 up 的接口 ★
   ② 为【每一个】接口决定归属区域（一个都不能漏）
   ③ 画出所有业务的【发起方向】矩阵
   ④ 为每个需要通的方向配 zone-pair
   ⑤ ★ configure terminal revert timer 15 ★（保命）
   ⑥ 下发配置
   ⑦ 全面验证（含 SSH、Syslog、监控、备份）
   ⑧ configure confirm
```
</details>

---

**下一章** → [03 VPN：IPsec 与 SSL](03-VPN-IPsec与SSL.md)
