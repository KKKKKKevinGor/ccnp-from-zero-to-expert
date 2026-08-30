# 06 · NAT 地址转换

## ① 这章解决什么问题

公司有 300 台电脑，用的是 `192.168.1.0/24` 私网地址。运营商只给了 **1 个公网 IP**。

300 台机器怎么共用 1 个公网 IP 上网？

**NAT（Network Address Translation）** 就是答案：在出口路由器上，把私网源地址改写成公网地址；回包时再改回来。

理解 NAT 还有更深的意义——**它是"IP 地址端到端不变"这条规则的唯一例外**。而正是这个例外，造成了 VoIP、FTP、IPsec 的一系列麻烦，也是很多"莫名其妙不通"的根源。

---

## ② 原理讲透

### 2.1 为什么需要 NAT

1. **IPv4 地址枯竭**（根本原因）。全球 43 亿个 IPv4 地址早已分完，不可能给每台设备一个公网 IP。
2. **私网地址不可路由**。RFC 1918 定义的 `10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16` 在互联网上会被直接丢弃。
3. **附带的安全性**。外部无法直接主动连接内网主机（但这是副作用，**不是安全机制**，别把 NAT 当防火墙）。

### 2.2 四个地址术语（考试必考，实战容易绕晕）

这是 NAT 最让人头疼的部分。用一句话理清：

> **inside/outside 说的是"这个设备在哪一侧"，local/global 说的是"在哪一侧看到的地址"。**

```
     内部网络                    NAT 路由器                  外部网络
                                     │
   PC (192.168.1.10) ────────────────┤──────────────── Server (200.1.1.1)
                                     │
                            inside ← │ → outside
```

| 术语 | 含义 | 示例 |
|:--|:--|:--|
| **Inside Local** | 内部设备的**真实私网地址**（在内部看内部） | `192.168.1.10` |
| **Inside Global** | 内部设备转换后的**公网地址**（在外部看内部） | `202.100.1.1` |
| **Outside Global** | 外部设备的**真实公网地址**（在外部看外部） | `200.1.1.1` |
| **Outside Local** | 外部设备在内部**被看到的地址**（在内部看外部） | 通常 = Outside Global |

**记忆技巧**：
- **Inside / Outside** = 这台设备物理上属于内网还是外网
- **Local / Global** = 从内网视角看到的地址 / 从外网视角看到的地址

**数据包流转示例**：

```
① PC 发出（NAT 之前）
   源: 192.168.1.10 (Inside Local)  →  目的: 200.1.1.1 (Outside Global)

② 经过 NAT 路由器（源地址被改写）
   源: 202.100.1.1 (Inside Global)  →  目的: 200.1.1.1 (Outside Global)

③ 服务器回包
   源: 200.1.1.1                    →  目的: 202.100.1.1 (Inside Global)

④ NAT 路由器查表还原
   源: 200.1.1.1                    →  目的: 192.168.1.10 (Inside Local)
```

`Outside Local` 只在做**目的地址 NAT**（比如内网访问某公网服务时把目的地址也改写）时才会和 Outside Global 不同，实战中很少用到。

### 2.3 三种 NAT 类型

#### ① 静态 NAT（Static NAT）—— 一对一固定映射

```cisco
R1(config)# ip nat inside source static 192.168.1.100 202.100.1.10
```

**用途**：内网服务器需要被外网访问（Web 服务器、邮件服务器）。
**特点**：一个私网 IP 固定对应一个公网 IP，**双向都可以主动发起连接**。
**缺点**：**极其浪费公网地址**，一台服务器占一个。

#### ② 动态 NAT（Dynamic NAT）—— 从地址池里动态分配

```cisco
R1(config)# ip nat pool PUBLIC-POOL 202.100.1.10 202.100.1.20 netmask 255.255.255.0
R1(config)# access-list 1 permit 192.168.1.0 0.0.0.255
R1(config)# ip nat inside source list 1 pool PUBLIC-POOL
```

**用途**：内网机器数量 > 公网地址数，但同时上网的不多。
**特点**：先到先得，用完释放。
**缺点**：**地址池用完后，后来的机器就上不了网**。实战中几乎不用。

#### ③ PAT / NAT Overload（★ 最常用）—— 多对一，靠端口区分

```cisco
R1(config)# access-list 1 permit 192.168.1.0 0.0.0.255
R1(config)# ip nat inside source list 1 interface GigabitEthernet0/1 overload
                                                                      ↑↑↑↑↑↑↑↑
```

**PAT = Port Address Translation**，也叫 **NAT Overload**。

**原理**：不仅改写 IP，**还改写源端口**，用端口号来区分不同的内网会话。

```
NAT 转换表：
┌─────────────────────┬─────────────────────┬──────────────────┐
│ Inside Local        │ Inside Global       │ Outside          │
├─────────────────────┼─────────────────────┼──────────────────┤
│ 192.168.1.10:1234   │ 202.100.1.1:50001   │ 200.1.1.1:80     │
│ 192.168.1.11:1234   │ 202.100.1.1:50002   │ 200.1.1.1:80     │
│ 192.168.1.12:5678   │ 202.100.1.1:50003   │ 200.1.1.1:443    │
└─────────────────────┴─────────────────────┴──────────────────┘
                                    ↑
                        同一个公网 IP，靠端口区分
```

**理论容量**：一个公网 IP 有 65535 个端口，实际可用约 **4000~64000 个并发会话**（取决于平台和保留端口）。

> **实战容量估算**：现代浏览器打开一个网页可能建立 10–50 个并发连接。所以**一个公网 IP 实际能支撑几百到一千个用户**。企业规模超过这个数就要考虑多个公网 IP 或多出口。

### 2.4 端口转发（Port Forwarding / 静态 PAT）

只有一个公网 IP，但内网有多个服务要对外发布：

```cisco
! 把公网 IP 的 80 端口映射到内网 Web 服务器
R1(config)# ip nat inside source static tcp 192.168.1.100 80 202.100.1.1 80

! 把公网 IP 的 2222 端口映射到内网 SSH 服务器的 22
R1(config)# ip nat inside source static tcp 192.168.1.200 22 202.100.1.1 2222

! UDP 服务
R1(config)# ip nat inside source static udp 192.168.1.150 53 202.100.1.1 53
```

这样一个公网 IP 就能对外发布多个服务，是中小企业最常用的做法。

### 2.5 inside / outside 接口标记（最容易忘的一步）

**NAT 必须知道哪个接口是内网、哪个是外网**，否则不工作：

```cisco
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip nat inside                 ! 内网侧

R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip nat outside                ! 外网侧
```

> ⚠️ **忘配 `ip nat inside/outside` 是 NAT 不生效的头号原因。** 配置看起来完全正确，`show ip nat translations` 是空的，`show ip nat statistics` 显示 0 次转换。
>
> **一分钟自查**：
> ```cisco
> R1# show ip nat statistics | include interfaces
> Outside interfaces:
>   GigabitEthernet0/1
> Inside interfaces:
>   GigabitEthernet0/0
> ```
> 两行都有内容才算对。

### 2.6 NAT 的处理顺序（进阶但重要）

**这决定了 ACL 和 NAT 的相互影响，是 ENARSI 的考点。**

```
Inside → Outside 方向（内网访问外网）：
  ① 入接口 ACL 检查
  ② 路由查找（决定出接口）
  ③ ★ NAT（源地址转换）
  ④ 出接口 ACL 检查            ← 此时源地址已经是公网地址！

Outside → Inside 方向（外网访问内网）：
  ① 入接口 ACL 检查            ← 此时目的地址还是公网地址！
  ② ★ NAT（目的地址还原）
  ③ 路由查找
  ④ 出接口 ACL 检查            ← 此时目的地址已经是私网地址
```

**实战含义（非常重要）**：

在**外网接口的 in 方向**写 ACL 放行外部访问内部服务器时，**必须写公网地址，不能写私网地址**：

```cisco
! ✅ 正确：ACL 在 NAT 之前执行，看到的是公网地址
R1(config)# ip access-list extended FROM-INTERNET
R1(config-ext-nacl)# permit tcp any host 202.100.1.1 eq 80
R1(config)# interface Gi0/1
R1(config-if)# ip access-group FROM-INTERNET in

! ❌ 错误：写私网地址，永远不会匹配
R1(config-ext-nacl)# permit tcp any host 192.168.1.100 eq 80
```

**这是 NAT + ACL 组合最经典的坑。**

### 2.7 NAT 带来的问题（理解这些才算真懂）

#### 问题 1：破坏了端到端原则

某些协议**在应用层载荷里携带 IP 地址**，NAT 只改 IP 头，不改载荷，导致协议失效：

| 协议 | 问题 | 解决 |
|:--|:--|:--|
| **FTP（主动模式）** | PORT 命令在载荷里传 IP 和端口 | NAT ALG（应用层网关）/ 改用被动模式 |
| **SIP / VoIP** | SDP 里传媒体流的 IP 端口 | SIP ALG / STUN / TURN |
| **IPsec AH** | AH 对整个 IP 头做完整性校验，NAT 改了头 → 校验失败 | **AH 根本无法穿越 NAT**，只能用 ESP |
| **IPsec ESP** | ESP 没有端口，PAT 无法区分多个会话 | **NAT-T**（封装进 UDP 4500） |
| **ICMP** | 没有端口 | PAT 用 ICMP Identifier 字段替代 |

**Cisco 的 ALG（Application Layer Gateway）会自动处理部分协议**：
```cisco
R1# show ip nat statistics | include ALG
! 查看已启用的 ALG

! 关闭某个 ALG（有时 ALG 反而会捣乱）
R1(config)# no ip nat service sip tcp port 5060
R1(config)# no ip nat service sip udp port 5060
```

> **实战经验**：VoIP 电话注册不上、通话没声音（单通），**第一件事就是关掉 SIP ALG**。很多路由器的 SIP ALG 实现有 bug，改写载荷时改错，反而导致问题。

#### 问题 2：IPsec 与 NAT-T

```
IPsec ESP 直接跑在 IP 之上（协议号 50），没有端口号
        ↓
PAT 无法用端口区分多个内网主机的 VPN 会话
        ↓
NAT-T（NAT Traversal）：把 ESP 封装进 UDP 4500
        ↓
现在有端口了，PAT 可以正常工作 ✓
```

**ACL 必须放行**：
```cisco
permit udp any any eq 500        ! IKE 阶段一
permit udp any any eq 4500       ! NAT-T
permit esp any any               ! ESP（如果没有 NAT 的话）
```

详见 [ENCOR 第 2 章](../02-ENCOR-350-401/02-网络虚拟化-VRF-GRE-IPsec.md)。

#### 问题 3：NAT 不是防火墙

**常见误解**："我们用了 NAT，外面进不来，很安全。"

**真相**：
- NAT 只是**默认没有入方向的映射**，不是主动阻止
- 一旦内网主机主动发起连接，NAT 表就有了条目，**攻击者可以利用这个窗口**（NAT 打洞技术，P2P 软件就是这么工作的）
- NAT 完全不检查流量内容，恶意软件的 C2 通信、数据外泄一路畅通
- 配了端口转发后，那个服务就完全暴露了

**结论**：NAT 提供的是"地址隐藏"，不是"访问控制"。**该配防火墙还是要配防火墙。**

---

## ③ 配置命令

### Cisco

```cisco
! ═══ ① 标记接口（★ 必做第一步）═══
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip nat inside
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip nat outside

! ═══ ② PAT（最常用）═══
R1(config)# ip access-list standard NAT-LIST
R1(config-std-nacl)# permit 192.168.1.0 0.0.0.255
R1(config-std-nacl)# permit 192.168.2.0 0.0.0.255
R1(config)# ip nat inside source list NAT-LIST interface GigabitEthernet0/1 overload

! 用地址池做 PAT（有多个公网 IP 时）
R1(config)# ip nat pool PUBLIC 202.100.1.10 202.100.1.12 netmask 255.255.255.248
R1(config)# ip nat inside source list NAT-LIST pool PUBLIC overload

! ═══ ③ 静态 NAT（服务器对外）═══
R1(config)# ip nat inside source static 192.168.1.100 202.100.1.10

! ═══ ④ 端口转发（静态 PAT）═══
R1(config)# ip nat inside source static tcp 192.168.1.100 80 202.100.1.1 80
R1(config)# ip nat inside source static tcp 192.168.1.200 22 202.100.1.1 2222
R1(config)# ip nat inside source static udp 192.168.1.150 161 202.100.1.1 161

! ═══ ⑤ 动态 NAT（不带 overload）═══
R1(config)# ip nat pool DYN-POOL 202.100.1.20 202.100.1.30 netmask 255.255.255.0
R1(config)# ip nat inside source list NAT-LIST pool DYN-POOL

! ═══ ⑥ 超时时间调整 ═══
R1(config)# ip nat translation timeout 3600           ! 通用，默认 24 小时
R1(config)# ip nat translation tcp-timeout 3600       ! TCP，默认 24 小时
R1(config)# ip nat translation udp-timeout 300        ! UDP，默认 5 分钟
R1(config)# ip nat translation icmp-timeout 60        ! ICMP，默认 60 秒
R1(config)# ip nat translation finrst-timeout 60      ! FIN/RST 后保留时间
R1(config)# ip nat translation max-entries 10000      ! 最大条目数，防内存耗尽

! ═══ ⑦ 排除某些流量不做 NAT（VPN 场景必用）═══
R1(config)# ip access-list extended NAT-LIST
R1(config-ext-nacl)# deny ip 192.168.1.0 0.0.0.255 192.168.99.0 0.0.0.255   ! 去总部的走 VPN，不 NAT
R1(config-ext-nacl)# permit ip 192.168.1.0 0.0.0.255 any                    ! 其余上网做 NAT

! ═══ 查看与排障 ═══
R1# show ip nat translations                   ! ★ 看转换表
R1# show ip nat translations verbose
R1# show ip nat statistics                     ! ★ 看接口标记和命中次数
R1# clear ip nat translation *                 ! 清空转换表
R1# clear ip nat translation inside 192.168.1.10 202.100.1.1
R1# debug ip nat                               ! ⚠️ 高流量下会刷屏，慎用
R1# debug ip nat detailed
```

### 输出解读

```cisco
R1# show ip nat translations
Pro  Inside global       Inside local       Outside local      Outside global
tcp  202.100.1.1:50001   192.168.1.10:1234  200.1.1.1:80       200.1.1.1:80
tcp  202.100.1.1:50002   192.168.1.11:1234  200.1.1.1:80       200.1.1.1:80
icmp 202.100.1.1:1       192.168.1.10:1      8.8.8.8:1          8.8.8.8:1
---  202.100.1.10        192.168.1.100      ---                ---
 ↑                                                              
静态 NAT 条目（协议是 ---，永久存在）
```

```cisco
R1# show ip nat statistics
Total active translations: 47 (1 static, 46 dynamic; 46 extended)
Peak translations: 312, occurred 02:15:33 ago
Outside interfaces:
  GigabitEthernet0/1                          ← ★ 必须有
Inside interfaces:
  GigabitEthernet0/0                          ← ★ 必须有
Hits: 128456  Misses: 0                       ← Hits 增长说明 NAT 在工作
CEF Translated packets: 128456, CEF Punted packets: 0
Expired translations: 8934
Dynamic mappings:
-- Inside Source
[Id: 1] access-list NAT-LIST interface GigabitEthernet0/1 refcount 46
```

**排障要点**：
- `Inside/Outside interfaces` 为空 → **忘了标记接口**
- `Hits` 一直是 0 → 流量没匹配到 ACL，或者没走这条路径
- `Misses` 很高 → 有流量匹配了但转换失败（地址池耗尽？）
- `Total active translations` 接近 `max-entries` → 会话数快满了

### 三厂商对照

| 目的 | Cisco | H3C | 华为 |
|:--|:--|:--|:--|
| 标记内网口 | `ip nat inside` | 不需要（用 zone 或直接在出接口配） | 不需要 |
| 标记外网口 | `ip nat outside` | — | — |
| PAT（出接口） | `ip nat inside source list 1 interface Gi0/1 overload` | `nat outbound 2000`（接口下） | `nat outbound 2000`（接口下） |
| PAT（地址池） | `ip nat pool P ... ` + `... pool P overload` | `nat address-group 1` + `nat outbound 2000 address-group 1` | `nat address-group 1` + `nat outbound 2000 address-group 1` |
| 静态 NAT | `ip nat inside source static <私> <公>` | `nat static outbound <私> <公>` | `nat static global <公> inside <私>` |
| 端口转发 | `ip nat inside source static tcp <私> 80 <公> 80` | `nat server protocol tcp global <公> 80 inside <私> 80` | `nat server protocol tcp global <公> 80 inside <私> 80` |
| 查转换表 | `show ip nat translations` | `display nat session` | `display nat session all` |
| 查统计 | `show ip nat statistics` | `display nat all` | `display nat outbound` |

> **架构差异**：**Cisco 需要显式标记 `ip nat inside/outside`，H3C/华为不需要**——它们把 NAT 配置直接绑在出接口上（`nat outbound`），方向是隐含的。从 Cisco 转过来的人会觉得国产设备更简洁；反过来则容易忘掉标记接口这一步。
>
> **术语差异**：Cisco 的"端口转发"在 H3C/华为叫 **`nat server`**（NAT 服务器映射），这个名字其实更直观。

---

## ④ 配套实验：PAT + 端口转发 + VPN 流量排除

**拓扑**：
```
   内网 192.168.1.0/24
        │
   PC1  192.168.1.10
   Web  192.168.1.100 (80端口)
   SSH  192.168.1.200 (22端口)
        │
     Gi0/0 (192.168.1.1)  [ip nat inside]
   ┌────┴─────┐
   │    R1    │
   └────┬─────┘
     Gi0/1 (202.100.1.1)  [ip nat outside]
        │
   [ Internet ]  ── Server 200.1.1.1
```

### Step 1：基础 PAT

```cisco
! ① 标记接口
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# ip nat inside
R1(config-if)# no shutdown

R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip address 202.100.1.1 255.255.255.0
R1(config-if)# ip nat outside
R1(config-if)# no shutdown

! ② 定义要做 NAT 的流量
R1(config)# ip access-list standard NAT-LIST
R1(config-std-nacl)# permit 192.168.1.0 0.0.0.255

! ③ 配置 PAT
R1(config)# ip nat inside source list NAT-LIST interface GigabitEthernet0/1 overload

! ④ 默认路由指向运营商
R1(config)# ip route 0.0.0.0 0.0.0.0 202.100.1.254
```

### Step 2：验证 PAT

```cisco
! 在 PC1 上访问外网
PC1> ping 200.1.1.1
PC1> telnet 200.1.1.1 80
```

```cisco
R1# show ip nat translations
Pro  Inside global       Inside local        Outside local    Outside global
icmp 202.100.1.1:1       192.168.1.10:1      200.1.1.1:1      200.1.1.1:1
tcp  202.100.1.1:50123   192.168.1.10:52341  200.1.1.1:80     200.1.1.1:80
     ↑ 转换后的公网地址+端口  ↑ 原始私网地址+端口
```

**在外网服务器上抓包**，会看到源地址是 `202.100.1.1`，而不是 `192.168.1.10`。这就验证了 NAT 生效。

### Step 3：多台内网机器共用一个公网 IP

```cisco
! PC1、PC2、PC3 同时访问同一个网站
R1# show ip nat translations
Pro  Inside global       Inside local        Outside local    Outside global
tcp  202.100.1.1:50123   192.168.1.10:52341  200.1.1.1:80     200.1.1.1:80
tcp  202.100.1.1:50124   192.168.1.11:49872  200.1.1.1:80     200.1.1.1:80
tcp  202.100.1.1:50125   192.168.1.12:61234  200.1.1.1:80     200.1.1.1:80
                    ↑↑↑↑↑
        同一个 IP，不同端口 —— 这就是 PAT 的核心
```

### Step 4：端口转发（对外发布服务）

```cisco
R1(config)# ip nat inside source static tcp 192.168.1.100 80 202.100.1.1 80
R1(config)# ip nat inside source static tcp 192.168.1.200 22 202.100.1.1 2222
```

**验证**：
```cisco
R1# show ip nat translations
Pro  Inside global       Inside local        Outside local    Outside global
tcp  202.100.1.1:80      192.168.1.100:80    ---              ---
tcp  202.100.1.1:2222    192.168.1.200:22    ---              ---
     ↑ 静态映射，永久存在，不会老化
```

**从外网测试**：
```bash
curl http://202.100.1.1          # 访问内网 Web 服务器
ssh -p 2222 user@202.100.1.1     # 访问内网 SSH
```

### Step 5：加 ACL（★ 踩坑重点）

给外网接口加入方向 ACL，只允许访问发布的服务：

```cisco
R1(config)# ip access-list extended FROM-INTERNET
 ! ★ 必须写公网地址！因为 ACL 在 NAT 之前执行
R1(config-ext-nacl)# permit tcp any host 202.100.1.1 eq 80
R1(config-ext-nacl)# permit tcp any host 202.100.1.1 eq 2222
 ! 允许回程流量
R1(config-ext-nacl)# permit tcp any any established
R1(config-ext-nacl)# permit icmp any any echo-reply
R1(config-ext-nacl)# permit icmp any any unreachable       ! ★ PMTUD
R1(config-ext-nacl)# permit icmp any any time-exceeded
R1(config-ext-nacl)# permit udp any eq domain any          ! DNS 回应
R1(config-ext-nacl)# deny ip any any log

R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip access-group FROM-INTERNET in
```

**故意写错试试**：
```cisco
R1(config-ext-nacl)# 5 permit tcp any host 192.168.1.100 eq 80    ! 写私网地址
```
```cisco
R1# show access-lists FROM-INTERNET
    5 permit tcp any host 192.168.1.100 eq www (0 matches)    ← 永远是 0！
    10 permit tcp any host 202.100.1.1 eq www (245 matches)
```

**这就证明了 ACL 在 NAT 之前执行。** 记住这个顺序，能省你很多调试时间。

### Step 6：排除 VPN 流量（实战高频场景）

**场景**：分支机构有一条 IPsec VPN 到总部（`192.168.99.0/24`）。去总部的流量**不能做 NAT**（要保留原始私网地址，否则总部收到的是公网地址，回不来）。

```cisco
! 改成扩展 ACL，先 deny 掉去总部的流量
R1(config)# no ip access-list standard NAT-LIST
R1(config)# ip access-list extended NAT-LIST
R1(config-ext-nacl)# deny ip 192.168.1.0 0.0.0.255 192.168.99.0 0.0.0.255
R1(config-ext-nacl)# permit ip 192.168.1.0 0.0.0.255 any

R1(config)# ip nat inside source list NAT-LIST interface GigabitEthernet0/1 overload
```

**验证**：
```cisco
! ping 总部，不应该出现在 NAT 表里
PC1> ping 192.168.99.10

R1# show ip nat translations | include 192.168.99
! 应该没有输出 ✓

! ping 互联网，应该出现在 NAT 表里
PC1> ping 8.8.8.8
R1# show ip nat translations | include 8.8.8.8
icmp 202.100.1.1:5  192.168.1.10:5  8.8.8.8:5  8.8.8.8:5    ✓
```

> **这是站点到站点 VPN 部署中最常见的故障**：VPN 隧道建起来了，但两边内网就是不通。**根因 90% 是"忘了在 NAT 的 ACL 里 deny 掉去对端的流量"**——流量被 NAT 成了公网地址，就不再匹配 VPN 的感兴趣流了。

---

## ⑤ 排障思路

| 症状 | 怀疑点 | 验证 | 根因 |
|:--|:--|:--|:--|
| NAT 完全不工作 | 接口标记 | `show ip nat statistics \| inc interfaces` | **忘了 `ip nat inside/outside`** |
| 转换表是空的 | ACL 不匹配 | `show access-lists NAT-LIST` | ACL 网段写错 / 没有命中 |
| 部分机器能上网、部分不能 | 地址池耗尽 | `show ip nat statistics` 看 Misses | 动态 NAT 池用完，改用 overload |
| 静态 NAT 配了但外面进不来 | ACL 方向/地址 | `show access-lists` 看命中 | **ACL 写了私网地址**（应写公网） |
| VPN 隧道通但内网不通 | NAT 顺序 | `show ip nat translations \| inc <对端网段>` | **NAT ACL 没 deny 掉 VPN 流量** |
| VoIP 单通/注册失败 | ALG | `show ip nat statistics` | SIP ALG 改写载荷出错，尝试关闭 |
| FTP 能登录不能传文件 | 主动/被动模式 | 抓包看 PORT/PASV | 主动模式穿不过 NAT，改被动 |
| 会话数满，新连接失败 | 表项耗尽 | `show ip nat statistics` | 超时太长 / 有主机在扫描 |
| 大文件传不动 | MTU | `ping size 1500 df-bit` | NAT + 隧道叠加导致 MTU 不足 |
| CPU 高 | NAT 表巨大 | `show ip nat statistics` | 表项过多，考虑硬件 NAT 或缩短超时 |

### 排障三板斧

```cisco
! ① 接口标记对了吗？
R1# show ip nat statistics | include interfaces -A 4

! ② ACL 匹配了吗？
R1# show access-lists NAT-LIST
! 看 matches 计数，为 0 说明流量没匹配上

! ③ 转换表建立了吗？
R1# clear ip nat translation *
! 让用户重现问题
R1# show ip nat translations
```

### NAT 表项耗尽的处理

```cisco
R1# show ip nat statistics
Total active translations: 63500 (2 static, 63498 dynamic; 63498 extended)
Peak translations: 64000                       ← 快满了

! 查是谁占用最多
R1# show ip nat translations | include 192.168.1.66 | count
! 某台机器占了几万条 → 可能中毒在扫描

! 应急：缩短超时
R1(config)# ip nat translation timeout 300
R1(config)# ip nat translation udp-timeout 60
R1(config)# ip nat translation icmp-timeout 10

! 限制单主机的最大条目数
R1(config)# ip nat translation max-entries host 192.168.1.66 100

! 全局上限（防止内存耗尽导致设备宕机）
R1(config)# ip nat translation max-entries 60000
```

> **实战经验**：出口路由器 CPU 突然飙高、NAT 表暴涨，**十有八九是内网有机器中了病毒在做端口扫描**。用 `show ip nat translations | include <IP>` 逐个排查，找到源头后隔离处理。
>
> 长期方案：配 `max-entries host` 限制单机会话数，让中毒机器不至于拖垮整个出口。

---

## ⑥ 考点提示 + 自测题

### 考点

- **四个地址术语**（Inside Local/Global、Outside Local/Global）。
- **PAT 用端口区分会话**的原理。
- **`ip nat inside/outside` 必须标记**。
- **NAT 与 ACL 的处理顺序**（外网入方向 ACL 看到的是公网地址）。
- **VPN 场景要 deny 掉不做 NAT 的流量**。
- **NAT 不是防火墙**。
- **IPsec ESP 需要 NAT-T 才能穿越 PAT**。

### 自测题

**1.** `ip nat inside source list 1 interface Gi0/1 overload` 中的 `overload` 是什么意思？不加会怎样？

<details><summary>答案</summary>

**`overload` 启用 PAT（Port Address Translation）** —— 除了改写 IP 地址，**还改写源端口**，让多个内网主机共用一个公网 IP，靠端口号区分不同会话。

**不加 `overload` 的后果**：变成**动态 NAT**，是**一对一**映射。

```cisco
! 不加 overload = 动态 NAT
ip nat pool P 202.100.1.10 202.100.1.12 netmask 255.255.255.248
ip nat inside source list 1 pool P
! 只有 3 个公网地址 → 同时只能有 3 台内网机器上网
! 第 4 台机器会失败：%NAT: translation failed (A), dropping packet
```

**用 `interface Gi0/1 overload` 时**，即使只有 1 个公网 IP，也能支撑几百上千个并发会话。

**验证差异**：
```cisco
! PAT（有 overload）—— 注意端口号
R1# show ip nat translations
tcp  202.100.1.1:50001   192.168.1.10:1234   ...
tcp  202.100.1.1:50002   192.168.1.11:1234   ...
                    ↑ 同一 IP 不同端口

! 动态 NAT（无 overload）—— 没有端口
R1# show ip nat translations
---  202.100.1.10  192.168.1.10  ---  ---
---  202.100.1.11  192.168.1.11  ---  ---
     ↑ 不同 IP，一对一
```

**实战中几乎永远用 `overload`**，纯动态 NAT 只在极少数需要保持一对一映射的场景（如某些合规要求）才用。
</details>

**2.** 配好了 NAT，但 `show ip nat translations` 是空的，内网上不了网。最可能忘了什么？

<details><summary>答案</summary>

**忘了在接口上标记 `ip nat inside` 和 `ip nat outside`。**

这是 NAT 不生效的**头号原因**。因为 NAT 配置本身（ACL、pool、inside source 语句）看起来完全正确，很难联想到接口标记。

**一分钟自查**：
```cisco
R1# show ip nat statistics
Total active translations: 0 (0 static, 0 dynamic; 0 extended)
Outside interfaces:
                                    ← 空的！
Inside interfaces:
                                    ← 空的！
Hits: 0  Misses: 0
```

**两行都必须有内容**：
```cisco
Outside interfaces:
  GigabitEthernet0/1                ✓
Inside interfaces:
  GigabitEthernet0/0                ✓
```

**修复**：
```cisco
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip nat inside
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip nat outside
```

**其他可能的原因（按概率排序）**：
1. ACL 网段写错（`show access-lists` 看 matches 为 0）
2. 没有默认路由指向运营商（流量根本没到外网接口）
3. 出接口 down
4. 多出口场景下流量走了另一条没配 NAT 的路径

**H3C/华为用户注意**：国产设备**不需要标记 inside/outside**，NAT 直接配在出接口上。所以从国产设备转 Cisco 的人特别容易踩这个坑。
</details>

**3.** 内网 Web 服务器 `192.168.1.100` 通过静态 NAT 映射到公网 `202.100.1.10`。你在外网接口的 in 方向写了 ACL：`permit tcp any host 192.168.1.100 eq 80`。为什么外网还是访问不了？

<details><summary>答案</summary>

**因为在外网接口的 in 方向，ACL 在 NAT 之前执行，此时数据包的目的地址还是公网地址 `202.100.1.10`，不是私网地址。**

**Outside → Inside 的处理顺序**：
```
① 入接口 ACL 检查      ← 目的地址是 202.100.1.10（公网）
② NAT（目的地址还原）   ← 这里才变成 192.168.1.100
③ 路由查找
④ 出接口 ACL 检查      ← 此时才是私网地址
```

所以 ACL 写 `host 192.168.1.100` **永远不会匹配**。

**验证**：
```cisco
R1# show access-lists FROM-INTERNET
    10 permit tcp any host 192.168.1.100 eq www (0 matches)    ← 永远是 0
    20 deny ip any any log (523 matches)                        ← 全被这条拦了
```

**正确写法**：
```cisco
R1(config)# ip access-list extended FROM-INTERNET
R1(config-ext-nacl)# permit tcp any host 202.100.1.10 eq 80    ← 写公网地址
```

**记忆方法**：**"ACL 看到的永远是那一侧接口上真实传输的地址。"**
- 外网接口上传输的是公网地址 → ACL 写公网地址
- 内网接口上传输的是私网地址 → ACL 写私网地址

**反向验证（Inside → Outside）**：
```
① 入接口 ACL（内网口 in）  ← 源地址是 192.168.1.10（私网）
② 路由查找
③ NAT（源地址转换）        ← 变成 202.100.1.1
④ 出接口 ACL（外网口 out） ← 源地址是 202.100.1.1（公网）
```
所以在**内网接口 in 方向**的 ACL 要写私网地址，在**外网接口 out 方向**的 ACL 要写公网地址。

**这个顺序是 ENARSI 的高频考点**，也是实战中调试 NAT+ACL 最耗时的地方。
</details>

**4.** 分支和总部建了 IPsec VPN，隧道状态是 UP，但两边内网互相 ping 不通。最可能的原因？

<details><summary>答案</summary>

**NAT 的 ACL 没有排除去往对端内网的流量**，导致去总部的包被 NAT 成了公网地址，不再匹配 VPN 的"感兴趣流"，于是走了普通互联网路径出去，对端收到后回不来。

**故障过程**：
```
1. PC (192.168.1.10) → 总部 (192.168.99.10)
2. R1 上 NAT 先执行：源地址被改成 202.100.1.1
3. 再匹配 IPsec 感兴趣流 ACL：permit ip 192.168.1.0 0.0.0.255 192.168.99.0 0.0.0.255
   → 源地址已经是 202.100.1.1，不匹配了！
4. 包不进隧道，直接从互联网发出
5. 总部收到源地址是公网的包，或者根本收不到 → 不通
```

**正确配置**：在 NAT 的 ACL 里**先 deny 掉去往 VPN 对端的流量**：

```cisco
R1(config)# ip access-list extended NAT-LIST
R1(config-ext-nacl)# deny ip 192.168.1.0 0.0.0.255 192.168.99.0 0.0.0.255   ! ★ 必须在前面
R1(config-ext-nacl)# permit ip 192.168.1.0 0.0.0.255 any

R1(config)# ip nat inside source list NAT-LIST interface Gi0/1 overload
```

**顺序至关重要**：`deny` 必须在 `permit ip ... any` **前面**，否则永远不会生效（参见 [第 5 章 ACL](05-ACL访问控制列表.md) 的顺序规则）。

**验证**：
```cisco
! 去总部的流量不应出现在 NAT 表里
R1# show ip nat translations | include 192.168.99
! 空输出 = 正确 ✓

! 检查 ACL 命中
R1# show access-lists NAT-LIST
    10 deny ip 192.168.1.0 0.0.0.255 192.168.99.0 0.0.0.255 (456 matches)   ← 有命中 ✓
    20 permit ip 192.168.1.0 0.0.0.255 any (23451 matches)

! 检查 IPsec 加密计数
R1# show crypto ipsec sa | include pkts
    #pkts encaps: 456, #pkts encrypt: 456                    ← 有加密包 ✓
    #pkts decaps: 452, #pkts decrypt: 452                    ← 有解密包 ✓
```

**如果 `#pkts encaps` 是 0**，说明流量根本没进隧道，就是这个 NAT 问题。

> **这是站点到站点 VPN 部署的头号故障**，几乎每个第一次配 VPN 的人都会踩。记住这条口诀：**"配完 VPN，先看 NAT。"**
</details>

**5.** 有人说"我们用了 NAT，外面进不来，所以很安全，不用装防火墙"。这个说法对吗？

<details><summary>答案</summary>

**不对。NAT 提供的是"地址隐藏"，不是"访问控制"。**

**NAT 的所谓"安全性"只是副作用**：因为没有入方向的映射条目，外部无法主动定位到内网主机。但这有大量绕过方式：

**① NAT 打洞（NAT Traversal / Hole Punching）**
内网主机主动发起连接后，NAT 表上就有了条目。攻击者（或 P2P 软件）可以利用这个窗口从外部发包进来。这正是 BitTorrent、Skype、WebRTC 能穿透 NAT 的原理——**能被善用，就能被恶用**。

**② NAT 完全不检查流量内容**
- 内网机器中了木马，主动连接 C2 服务器 → NAT 一路放行
- 员工往外传公司机密文件 → NAT 一路放行
- 恶意软件通过 DNS 隧道外泄数据 → NAT 一路放行

NAT 只改地址，**对流量做什么、传什么内容毫无概念**。

**③ 端口转发一旦配置就完全暴露**
配了 `ip nat inside source static tcp 192.168.1.100 80 202.100.1.1 80`，这台 Web 服务器就和放在公网上没有区别，NAT 提供不了任何保护。

**④ IPv6 环境下 NAT 会消失**
IPv6 地址充足，设计上就不需要 NAT。如果一个组织的安全完全依赖 NAT，向 IPv6 迁移时会瞬间裸奔。

**⑤ 内部横向移动不受影响**
NAT 只在出口生效。内网机器之间的攻击（勒索软件横向传播、ARP 欺骗、内网扫描）NAT 完全管不着——而这恰恰是现代攻击的主要路径。

**正确的做法（纵深防御）**：

| 层次 | 措施 |
|:--|:--|
| 边界 | 有状态防火墙（ZBF / ASA / FTD），明确的入站/出站策略 |
| 边界 | IPS/IDS，检测已知攻击特征 |
| 出站 | **出站流量也要管控**——限制内网主机能访问哪些外部服务 |
| 内部 | VLAN 隔离 + ACL，限制横向移动 |
| 接入 | 802.1X 认证，未授权设备接不进来 |
| 终端 | EDR、补丁管理 |
| 架构 | 零信任分段（见 [Security 选修第 5 章](../08-Security选修/05-零信任与网络分段.md)） |

**一句话总结**：**NAT 是地址复用技术，不是安全技术。** 把它当防火墙用，就像把窗帘当防盗门——能挡住随意的窥视，挡不住有目的的入侵。
</details>

---

**上一章** ← [05 ACL 访问控制列表](05-ACL访问控制列表.md) ｜ **下一章** → [07 DHCP/DNS/NTP 基础服务](07-DHCP-DNS-NTP基础服务.md)
