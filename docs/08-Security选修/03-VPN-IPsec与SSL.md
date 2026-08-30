# 03 · VPN：IPsec 与 SSL

## ① 这章解决什么问题

**三个真实需求**：

```
   ★ 需求 1：分公司要访问总部的 ERP ★
      两地都有固定公网 IP，流量固定、持续
      → ★ 站点到站点 IPsec VPN ★
   
   ★ 需求 2：出差员工要访问内网 ★
      IP 不固定，可能在酒店/机场，可能在 NAT 后面
      → ★ 远程接入 VPN（AnyConnect / SSL VPN）★
   
   ★ 需求 3：50 个门店要互访，还要能直连 ★
      全网状 IPsec 需要 50×49/2 = 1225 条隧道 😱
      → ★ DMVPN（见 ENARSI 05）★
```

**这一章讲清楚：IPsec 到底在做什么，为什么配置那么长，以及 SSL VPN 凭什么取代了 IPsec 远程接入。**

---

## ② 原理讲透

### 2.1 IPsec 提供什么

| 能力 | 靠什么实现 |
|:--|:--|
| **机密性** | ★ **加密**（AES/3DES） |
| **完整性** | ★ **HMAC**（SHA-256） |
| **认证性** | ★ **预共享密钥 / 数字证书** |
| **抗重放** | ★ **序列号 + 滑动窗口** |

### 2.2 ★ 两个协议：AH vs ESP ★

| | **AH（协议 51）** | **ESP（协议 50）** |
|:--|:--|:--|
| 完整性 | ✅ | ✅ |
| 认证 | ✅ | ✅ |
| ★ **加密** | ❌ **不加密** | ★ ✅ **加密** |
| 保护 IP 头 | ✅ 保护（除可变字段） | ❌ 不保护外层 |
| ★ **能穿 NAT** | ★ ❌ **不能**（NAT 改 IP → 校验失败） | ★ ⚠️ 需要 **NAT-T** |
| 实际使用 | ★ **几乎不用** | ★ **全部用这个** |

> **★ 考点**：AH 不提供加密。实际部署一律用 ESP。

### 2.3 ★ 两个模式：传输 vs 隧道 ★

```
   ★ 原始包 ★
   ┌─────────┬──────────┐
   │ 原 IP头  │  载荷     │
   └─────────┴──────────┘

   ★ 传输模式（Transport Mode）★
   ┌─────────┬─────┬──────────┬──────┐
   │ 原 IP头  │ ESP │  载荷     │ ESP尾 │
   └─────────┴─────┴──────────┴──────┘
       ↑ 保留原 IP 头，★ 只加密载荷 ★
       ★ 开销小（约 +36 字节）★
       ★ 用途：GRE over IPsec（GRE 已经有新 IP 头了）★
              主机到主机

   ★ 隧道模式（Tunnel Mode）★
   ┌─────────┬─────┬─────────┬──────────┬──────┐
   │ 新 IP头  │ ESP │ 原 IP头  │  载荷     │ ESP尾 │
   └─────────┴─────┴─────────┴──────────┴──────┘
       ↑ ★ 整个原始包（含 IP 头）都被加密 ★
       ★ 开销大（约 +58 字节）★
       ★ 用途：站点到站点 VPN（默认模式）★
```

**★ 为什么 GRE over IPsec 要用传输模式**：
```
   GRE 已经封装了一个新的 IP 头（隧道源/目的）
        ↓
   如果 IPsec 再用【隧道模式】，会再加一个 IP 头
        ↓
   ★ 两层多余的 IP 头 = 浪费 20 字节 ★
        ↓
   ★ 用传输模式，直接加密 GRE 包即可 ★
```

### 2.4 ★★ IKE 协商：为什么配置那么长 ★★

**核心问题：两端怎么在不安全的网络上，安全地商量出一把密钥？**

```
   ★ 阶段 1（IKE SA / ISAKMP SA）★
   目标：★ 建立一条【安全的管理通道】★
   
   ① 协商加密算法、哈希、DH 组、认证方式、生存期
   ② ★ Diffie-Hellman 交换 ★ → 双方各自算出相同的共享密钥
   ③ ★ 互相认证身份 ★（预共享密钥 / 证书）
        ↓
   ★ 此时有了一条加密的"管理隧道"，但还不能传用户数据 ★
   
        ↓
   
   ★ 阶段 2（IPsec SA）★
   目标：★ 在管理通道里，安全地协商【数据加密参数】★
   
   ① 协商用什么算法加密用户数据（transform-set）
   ② 协商保护哪些流量（★ crypto ACL / 感兴趣流 ★）
   ③ 生成实际的数据加密密钥
        ↓
   ★★ 现在才能传用户数据 ★★
```

**★ 为什么要分两个阶段？**
```
   ① ★ 效率 ★：阶段 1 的 DH 计算很贵，
                建一次可以协商多个阶段 2 SA
   ② ★ 安全 ★：阶段 2 可以频繁重新协商（换密钥），
                而不用重做昂贵的 DH
   ③ ★ 灵活 ★：一条管理通道可以保护多组不同的流量
```

**★ 关键：阶段 1 和阶段 2 的生存期不同**
```
   阶段 1 默认 86400 秒（24 小时）
   阶段 2 默认 3600 秒（1 小时）
   
   ★ 阶段 2 到期时，在已有的阶段 1 通道里重新协商，很快
   ★ 阶段 1 到期时，要重做完整的 DH，比较贵
```

### 2.5 IKEv1 vs IKEv2

| | **IKEv1** | **IKEv2** |
|:--|:--|:--|
| 消息数 | 主模式 6 条 + 阶段 2 三条 = **9 条** | ★ **4 条** |
| 野蛮模式 | 3 条（快但不保护身份） | 无此概念 |
| NAT 穿越 | 扩展功能 | ★ **原生支持** |
| ★ **非对称认证** | ❌ 两端必须同方式 | ★ ✅ **一端证书，一端预共享都行** |
| DPD（死对等体检测） | 扩展 | ★ **内置** |
| EAP 支持 | ❌ | ★ ✅（远程接入用） |
| 可靠性 | 无内置重传 | ★ **有序列号和确认** |
| ★ **推荐** | 遗留 | ★ ✅ **新部署一律用 IKEv2** |

**IKEv1 主模式 vs 野蛮模式**：
```
   ★ 主模式（Main Mode）★ —— 6 条消息
      · 前 4 条协商参数和做 DH
      · ★ 后 2 条在【已加密】的通道里交换身份 ★
      · ✅ 保护身份信息
      · ❌ 慢
      · ⚠️ ★ 用预共享密钥时，需要靠源 IP 找密钥
           → 【对端 IP 不固定时无法使用】★
   
   ★ 野蛮模式（Aggressive Mode）★ —— 3 条消息
      · ★ 第一条就明文发送身份 ★
      · ✅ 快
      · ✅ ★ 支持对端 IP 不固定（用 hostname 认证）★
      · ❌ ★ 身份泄露，且预共享密钥哈希可被离线爆破 ★
   
   ★ 结论：需要动态对端就用 IKEv2，不要用野蛮模式 ★
```

### 2.6 ★ NAT-T（NAT 穿越）★

```
   ★ 问题 ★
   ESP 是 IP 协议号 50，★ 没有端口号 ★
        ↓
   PAT 设备需要端口号来做映射
        ↓
   ★★ ESP 包过不了 PAT ★★

   ★ 解决：NAT-T ★
   把 ESP 包再封装进 ★ UDP 4500 ★
        ↓
   ┌──────┬──────────┬─────┬────────┐
   │ IP头  │ UDP 4500 │ ESP │ 原始包  │
   └──────┴──────────┴─────┴────────┘
        ↓
   ★ 有端口号了，PAT 可以正常处理 ★
```

**★ NAT-T 的自动检测**：
```
   IKE 阶段 1 的前两条消息中，双方交换
   ★ 源/目 IP+端口的哈希值 ★
        ↓
   如果收到的哈希和自己算的不一样
        ↓
   ★ 说明中间有 NAT ★
        ↓
   ★ 自动切换到 UDP 4500 封装 ★
```

**★ 必须放行的端口**：
```
   UDP 500   ← IKE 协商
   UDP 4500  ← ★ NAT-T ★
   协议 50   ← ESP（没有 NAT 时）
```

### 2.7 ★ SSL VPN vs IPsec VPN ★

| | **IPsec 远程接入** | **★ SSL VPN（AnyConnect）★** |
|:--|:--|:--|
| 层级 | 网络层（L3） | 传输层（L4，基于 TLS） |
| 端口 | UDP 500/4500 + 协议 50 | ★ **TCP 443 / DTLS UDP 443** |
| ★ **穿透性** | ⚠️ 常被防火墙/酒店 WiFi 挡 | ★ ✅ **443 几乎处处开放** |
| 客户端 | 需要专用客户端 + 复杂配置 | ★ 浏览器或轻量客户端 |
| ★ **部署难度** | 高 | ★ **低** |
| 粒度控制 | 网络级 | ★ **可到应用级** |
| 性能 | ★ 稍好 | 略差（TLS 开销）；**DTLS 弥补** |
| 用途 | ★ **站点到站点** | ★ **远程接入** |

**★ 现状**：
```
   ★ 站点到站点 → IPsec（或 DMVPN / SD-WAN）★
   ★ 远程接入   → SSL VPN（AnyConnect / 各家 SSL VPN）★
   
   IPsec 远程接入（Easy VPN）已基本被淘汰
```

**★ 为什么 AnyConnect 同时用 TLS 和 DTLS**：
```
   ★ TCP over TCP 的问题 ★
   TLS 跑在 TCP 上，用户流量里的 TCP 又跑在里面
        ↓
   外层 TCP 重传 + 内层 TCP 重传
        ↓
   ★★ TCP Meltdown：丢包时性能急剧恶化 ★★
   
        ↓ 解决
   
   ★ DTLS（UDP 443）传实际数据流量 ★
   ★ TLS（TCP 443）只做控制通道和 DTLS 不可用时的回退 ★
```

---

## ③ 配置命令

### 3.1 ★ 站点到站点 IPsec（IKEv2，推荐写法）★

**拓扑**：
```
   总部 192.168.1.0/24 ── R1 (203.0.113.1) ══ Internet ══ (198.51.100.1) R2 ── 分公司 192.168.2.0/24
```

**R1 配置**：
```cisco
!═══ ① IKEv2 提议（阶段 1 算法）═══
crypto ikev2 proposal PROP-AES256
 encryption aes-cbc-256
 integrity  sha256
 ★ group 14 ★

crypto ikev2 policy POL-1
 proposal PROP-AES256

!═══ ② 认证凭据 ═══
crypto ikev2 keyring KR-1
 peer R2
  address 198.51.100.1
  ★ pre-shared-key MyVerySecretKey123! ★

!═══ ③ IKEv2 profile（把身份和凭据关联起来）═══
crypto ikev2 profile PROF-R2
 match identity remote address 198.51.100.1 255.255.255.255
 identity local address 203.0.113.1
 authentication local  pre-share
 authentication remote pre-share
 keyring local KR-1
 ★ dpd 30 5 periodic ★

!═══ ④ transform-set（阶段 2 算法）═══
crypto ipsec transform-set TS-AES256 esp-aes 256 esp-sha256-hmac
 ★ mode tunnel ★

!═══ ⑤ 感兴趣流 ACL ═══
ip access-list extended ACL-VPN-TRAFFIC
 ★ permit ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255 ★

!═══ ⑥ crypto map ═══
crypto map CMAP 10 ipsec-isakmp
 set peer 198.51.100.1
 set transform-set TS-AES256
 set ikev2-profile PROF-R2
 ★ set pfs group14 ★
 ★ match address ACL-VPN-TRAFFIC ★

!═══ ⑦ 应用到出接口 ═══
interface GigabitEthernet0/0
 ip address 203.0.113.1 255.255.255.0
 ★ crypto map CMAP ★
```

**R2 配置**：★ 完全对称，注意感兴趣流 ACL 必须是【镜像】的 ★
```cisco
ip access-list extended ACL-VPN-TRAFFIC
 ★ permit ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255 ★
!  ↑ 源和目的调换
```

> ⚠️ **★ 感兴趣流 ACL 不镜像 = 隧道起不来或单向不通。这是最高频的配置错误之一。**

### 3.2 ★ GRE over IPsec（VTI 写法，更推荐）★

**为什么用 VTI 而不是 crypto map**：
```
   ★ crypto map 的问题 ★
   · 靠 ACL 定义感兴趣流 → 加一个网段就要改 ACL
   · ★ 路由协议跑不了 ★（组播不匹配 ACL）
   · 没有"接口"，QoS/NetFlow/ACL 都不好挂
   
        ↓
   
   ★ VTI（Virtual Tunnel Interface）★
   · ★ 隧道就是一个接口 ★，进这个接口的流量自动加密
   · ★ 路由协议可以直接跑 ★
   · 可以挂 QoS、ACL、NetFlow
   · ★ 加网段只需要改路由，不用改 ACL ★
```

**R1**：
```cisco
crypto ikev2 profile PROF-R2
 match identity remote address 198.51.100.1 255.255.255.255
 authentication local  pre-share
 authentication remote pre-share
 keyring local KR-1

crypto ipsec transform-set TS-AES256 esp-aes 256 esp-sha256-hmac
 ★ mode transport ★         ← ★ GRE over IPsec 用传输模式 ★

crypto ipsec profile IPSEC-PROF
 set transform-set TS-AES256
 set ikev2-profile PROF-R2
 set pfs group14

interface Tunnel0
 ip address 10.0.0.1 255.255.255.252
 ★ ip mtu 1400 ★
 ★ ip tcp adjust-mss 1360 ★
 tunnel source GigabitEthernet0/0
 tunnel destination 198.51.100.1
 ★ tunnel protection ipsec profile IPSEC-PROF ★

! ★ 路由协议直接跑在隧道上 ★
router ospf 1
 network 10.0.0.0 0.0.0.3 area 0
 network 192.168.1.0 0.0.0.255 area 0
```

**★ 纯 IPsec VTI（不要 GRE）**：
```cisco
interface Tunnel0
 ip address 10.0.0.1 255.255.255.252
 ★ tunnel mode ipsec ipv4 ★      ← 不是 GRE
 tunnel source GigabitEthernet0/0
 tunnel destination 198.51.100.1
 tunnel protection ipsec profile IPSEC-PROF
```
⚠️ **纯 IPsec VTI 不支持组播** → **跑不了 OSPF/EIGRP**（只能静态路由或 BGP）。
**要跑 IGP → 用 GRE over IPsec。**

### 3.3 ★ MTU 问题（必须处理）★

| 封装 | 开销 |
|:--|:--|
| GRE | +24 |
| IPsec 传输模式（ESP+AES+SHA） | ≈ +36 |
| IPsec 隧道模式 | ≈ +58 |
| ★ **GRE over IPsec（传输模式）** | ★ **≈ +60** |
| ★ **GRE over IPsec（隧道模式）** | ★ **≈ +82** |
| NAT-T 额外 UDP 头 | +8 |

**★ 万能解法**：
```cisco
interface Tunnel0
 ★ ip mtu 1400 ★
 ★ ip tcp adjust-mss 1360 ★
```

**★ 为什么两条都要**：
```
   ★ ip mtu 1400 ★
      → 让路由器对【超过 1400 的 IP 包】做分片或回 ICMP
      → 依赖 PMTUD，★ 而 PMTUD 常被防火墙挡掉 ICMP 而失效 ★
   
   ★ ip tcp adjust-mss 1360 ★
      → ★ 直接改写 TCP 三次握手中的 MSS 值 ★
      → 让【两端主机】自己就发小包
      → ★ 不依赖 ICMP，最可靠 ★
      → ⚠️ ★ 只对 TCP 有效，UDP 管不了 ★
```

**★ 症状识别**：
```
   ★ "ping 通，SSH 能连，但网页打不开 / 大文件传不动" ★
        ↓
   ★★ 99% 是 MTU 问题 ★★
   
   验证：
   ping 192.168.2.10 source 192.168.1.10 size 1500 df-bit
   → 失败
   ping 192.168.2.10 source 192.168.1.10 size 1300 df-bit
   → 成功
   ★ 小包通、大包不通 = MTU 黑洞 ★
```

### 3.4 三厂商对照

| 功能 | **Cisco IOS** | **H3C** | **华为** |
|:--|:--|:--|:--|
| 阶段 1 提议 | `crypto ikev2 proposal` | `ike proposal 1` | `ike proposal 1` |
| 加密算法 | `encryption aes-cbc-256` | `encryption-algorithm aes-cbc-256` | `encryption-algorithm aes-256` |
| 完整性 | `integrity sha256` | `integrity-algorithm hmac-sha256` | `authentication-algorithm sha2-256` |
| DH 组 | `group 14` | `dh group14` | `dh group14` |
| 预共享密钥 | `crypto ikev2 keyring` | `ike keychain KC` + `pre-shared-key` | `ike peer` + `pre-shared-key` |
| 阶段 2 | `crypto ipsec transform-set` | `ipsec transform-set TS` | `ipsec proposal PROP` |
| 感兴趣流 | `ip access-list extended` | `acl advanced 3000` | `acl 3000` |
| 策略 | `crypto map CMAP 10` | `ipsec policy POL 10 isakmp` | `ipsec policy POL 10 isakmp` |
| 应用到接口 | `crypto map CMAP` | `ipsec apply policy POL` | `ipsec policy POL` |
| 查阶段 1 | `show crypto ikev2 sa` | `display ike sa` | `display ike sa` |
| 查阶段 2 | `show crypto ipsec sa` | `display ipsec sa` | `display ipsec sa` |
| 清除 | `clear crypto sa` | `reset ipsec sa` | `reset ipsec sa` |

**华为配置示例**：
```
ike proposal 1
 encryption-algorithm aes-256
 authentication-algorithm sha2-256
 dh group14

ike peer R2
 ike-proposal 1
 pre-shared-key simple MyVerySecretKey123!
 remote-address 198.51.100.1

ipsec proposal PROP1
 esp encryption-algorithm aes-256
 esp authentication-algorithm sha2-256

acl 3000
 rule 5 permit ip source 192.168.1.0 0.0.0.255 destination 192.168.2.0 0.0.0.255

ipsec policy POL1 10 isakmp
 security acl 3000
 ike-peer R2
 proposal PROP1
 pfs dh-group14

interface GigabitEthernet0/0/0
 ipsec policy POL1
```

---

## ④ 配套实验

### 实验：站点到站点 VPN + 故障注入

**拓扑**（EVE-NG）：
```
   PC1(192.168.1.10) ─ R1(203.0.113.1) ─ ISP ─ R2(198.51.100.1) ─ PC2(192.168.2.10)
```

**步骤**：

**① 先打通底层**（R1 和 R2 的公网 IP 能互 ping）—— ★ 不通就别配 VPN ★

**② 按 §3.1 配置两端**

**③ 触发隧道**（IPsec 是**流量驱动**的）
```
   PC1 ping PC2
   ★ 第一个包会丢（正在协商）★，之后就通了
```

**④ 验证**
```cisco
! ★ 阶段 1 ★
R1# show crypto ikev2 sa
 Tunnel-id  Local            Remote           fvrf/ivrf  ★ Status ★
 1          203.0.113.1/500  198.51.100.1/500 none/none  ★ READY ★

! ★ 阶段 2 ★
R1# show crypto ipsec sa
  ★ #pkts encaps: 45, #pkts encrypt: 45 ★    ← 出方向有加密
  ★ #pkts decaps: 45, #pkts decrypt: 45 ★    ← 入方向有解密
  ★ #recv errors 0 ★
```

**★★ 读这四个计数器是 IPsec 排障的核心技能 ★★**

**⑤ ★ 故障注入 ★**

| # | 注入 | 预期症状 | ★ 诊断关键 ★ |
|:--|:--|:--|:--|
| **F1** | R2 的预共享密钥改错一个字符 | ★ 阶段 1 起不来 | `show crypto ikev2 sa` 无 SA；debug 见 `AUTH failed` |
| **F2** | R2 的 DH 组改成 5 | ★ 阶段 1 协商失败 | debug 见 `no proposal chosen` |
| **F3** | R2 的 transform-set 改成 `esp-3des` | ★ 阶段 1 **成功**，阶段 2 失败 | ★ **IKEv2 SA READY 但 IPsec SA 为空** |
| **F4** | R2 的感兴趣流 ACL **不镜像** | ★ 隧道起来但流量不通 | ★ **encaps 有数，decaps 为 0** |
| **F5** | 中间加一台做 PAT 的设备 | 隧道起不来（无 NAT-T 时） | `show crypto ikev2 sa` 端口变 **4500** = NAT-T 生效 |
| **F6** | 中间 ISP 挡掉 UDP 500 | ★ 完全协商不了 | debug 无任何响应 |
| **F7** | 不配 `ip tcp adjust-mss` 跑 GRE over IPsec | ★ **ping 通，网页打不开** | ★ `ping ... size 1500 df-bit` 失败 |
| **F8** | 两端时间差 2 天（用证书认证时） | ★ 认证失败 | ★ **证书"尚未生效"** → 检查 NTP |

**★ F3 和 F4 是最有价值的两个**：
```
   ★ F3 教会你："阶段 1 成功 ≠ VPN 通" ★
      → 必须分别检查两个阶段
   
   ★ F4 教会你：看 encaps/decaps 计数器判断方向 ★
      · encaps 涨、decaps 不涨 → ★ 我发出去了，但对方没回 ★
        → 对端配置问题，或返回路径被挡
      · 两个都不涨 → ★ 流量根本没进隧道 ★
        → 路由问题，或感兴趣流 ACL 没匹配上
      · encaps 不涨、decaps 涨 → 我这边路由/ACL 有问题
```

---

## ⑤ 排障思路

### 5.1 ★ IPsec 排障的分层顺序 ★

```
   ★ 第 0 层：底层可达性 ★
   两端公网 IP 能不能互 ping？
   ping 198.51.100.1 source 203.0.113.1
   → ★ 不通就别往下查了，先解决路由/ISP 问题 ★
        ↓
   ★ 第 1 层：IKE 阶段 1 ★
   show crypto ikev2 sa   （或 show crypto isakmp sa）
   → 没有 SA 或状态不是 READY/QM_IDLE
     → ★ 检查：预共享密钥、DH 组、加密/哈希算法、
              peer 地址、UDP 500/4500 是否被挡、
              identity 匹配、证书有效期 ★
        ↓
   ★ 第 2 层：IPsec 阶段 2 ★
   show crypto ipsec sa
   → 阶段 1 好但没有阶段 2 SA
     → ★ 检查：transform-set 是否一致、
              感兴趣流 ACL 是否镜像、
              PFS 组是否一致、mode（tunnel/transport）是否一致 ★
        ↓
   ★ 第 3 层：流量 ★
   看 encaps / decaps 计数
   → 都是 0    → ★ 流量没进隧道：路由 / ACL 没匹配 ★
   → encaps 涨、decaps 不涨 → ★ 对端问题或回程被挡 ★
   → 都涨但业务不通 → ★ MTU！★ 或对端 ACL/防火墙
        ↓
   ★ 第 4 层：MTU ★
   ping <对端> size 1400 df-bit  → 通
   ping <对端> size 1500 df-bit  → 不通
   → ★ 配 ip mtu 1400 + ip tcp adjust-mss 1360 ★
```

### 5.2 症状 → 根因速查表

| 症状 | ★ 高频根因 ★ |
|:--|:--|
| **阶段 1 起不来** | ★ 预共享密钥不一致 ★ / DH 组不匹配 / UDP 500 被挡 / peer 地址写错 |
| **阶段 1 好，阶段 2 起不来** | ★ transform-set 不一致 ★ / ★ 感兴趣流 ACL 不镜像 ★ / PFS 组不一致 / mode 不一致 |
| **隧道 UP 但不通** | ★ encaps/decaps 看方向 ★ / 路由没指向隧道 / NAT 抢先处理了流量 |
| ★ **ping 通但业务不通** | ★★ MTU ★★ |
| **隧道频繁重建** | 生存期不一致 / DPD 太激进 / 底层链路丢包 |
| **有 NAT 时起不来** | ★ NAT-T 未生效 / UDP 4500 被挡 ★ |
| **单向通** | ★ 感兴趣流 ACL 不镜像 ★ / 一端有 ACL 挡了返回流量 |
| **用证书时认证失败** | ★ NTP 不同步 → 证书过期/未生效 ★ / CA 链不完整 / CRL 拉不到 |
| **配了 NAT 后 VPN 断了** | ★ NAT 优先于 IPsec 处理，VPN 流量被 NAT 了 ★ |

### 5.3 ★ NAT 和 IPsec 的顺序冲突（超高频坑）★

```
   ★ Cisco IOS 的处理顺序（出方向）★
   路由 → ★ NAT ★ → ★ crypto map（IPsec）★
        ↓
   如果出口做了 PAT（overload），
   192.168.1.0/24 的流量会先被 NAT 成公网 IP
        ↓
   ★ 再到 crypto map 时，源 IP 已经变了 ★
        ↓
   ★★ 不匹配感兴趣流 ACL → 不加密 → 明文发出去 ★★
```

**★ 修复：在 NAT 的 ACL 里排除 VPN 流量 ★**
```cisco
ip access-list extended ACL-NAT
 ★ deny ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255 ★   ← ★ 必须在最前面 ★
 permit ip 192.168.1.0 0.0.0.255 any

ip nat inside source list ACL-NAT interface Gi0/0 overload
```

**★ 用 VTI 就没有这个问题**——因为流量是"进隧道接口"触发加密的，NAT 不会误伤（前提是路由正确指向 Tunnel 接口）。

### 5.4 有用的 debug

```cisco
! ⚠️ ★ 生产环境务必先限定范围 ★
debug crypto condition peer ipv4 198.51.100.1

debug crypto ikev2
debug crypto ikev2 error
debug crypto ipsec

! IKEv1
debug crypto isakmp
debug crypto ipsec

! ★ 看完记得关 ★
undebug all
```

**★ 常见 debug 输出的含义**：

| 输出 | 含义 |
|:--|:--|
| `no proposal chosen` | ★ **算法不匹配**（加密/哈希/DH/transform-set） |
| `AUTHENTICATION_FAILED` | ★ **预共享密钥错** 或证书问题 |
| `invalid ID information` | ★ **identity 不匹配**（peer 地址/hostname） |
| `Phase 1 SA established` 后无下文 | ★ **阶段 2 参数不匹配** |
| `IPSEC(validate_proposal_request)` 失败 | ★ **感兴趣流 ACL 不镜像** |
| `NAT-T detected` | 中间有 NAT，已切 UDP 4500 |
| `Retransmitting` 反复出现 | ★ **对端收不到 / 端口被挡** |

---

## ⑥ 考点提示

```
   ★ ENCOR / SCOR 考点 ★
   · AH 不加密，ESP 加密
   · 传输模式 vs 隧道模式的包结构和用途
   · IKE 两个阶段各自在做什么
   · IKEv1 主模式 vs 野蛮模式
   · IKEv2 的优势
   · NAT-T：UDP 4500
   · GRE over IPsec 用【传输】模式
   
   ★ 必背端口/协议号 ★
   · UDP 500   = IKE
   · ★ UDP 4500 = NAT-T ★
   · ★ 协议 50  = ESP ★
   · 协议 51   = AH
   · 协议 47   = GRE
   · TCP 443   = SSL VPN
   
   ★ 必背开销 ★
   · GRE = +24
   · IPsec 隧道模式 ≈ +58
   · GRE over IPsec ≈ +82（隧道）/ +60（传输）
   · ★ 万能解法：ip mtu 1400 + ip tcp adjust-mss 1360 ★
```

---

## ⑦ 自测题

**1.** IKE 为什么要分两个阶段？两个阶段各自完成什么？如果 `show crypto ikev2 sa` 显示 READY 但业务不通，问题在哪一层？

<details><summary>答案</summary>

**★ 为什么分两阶段**：

```
   ★ 阶段 1（IKE SA）★ —— 建立【管理通道】
   ① 协商阶段 1 参数（加密、哈希、DH 组、认证方式、生存期）
   ② ★ Diffie-Hellman 交换 ★ → 双方各自算出相同的共享密钥
   ③ ★ 互相认证身份 ★（预共享密钥 / 证书）
   → 结果：一条【加密的双向管理隧道】，但★ 不传用户数据 ★

   ★ 阶段 2（IPsec SA）★ —— 在管理通道里协商【数据参数】
   ① 协商 transform-set（用什么加密用户数据）
   ② 协商感兴趣流（保护哪些流量）
   ③ 生成实际的数据加密密钥（配了 PFS 就再做一次 DH）
   → 结果：★ 两条【单向】的 IPsec SA ★（进和出各一条）
```

**★ 分开的三个理由**：
| 理由 | 说明 |
|:--|:--|
| **效率** | DH 计算很贵。★ 一条阶段 1 通道可以协商多个阶段 2 SA ★ |
| **安全** | 阶段 2 可以频繁换密钥（默认 1 小时），★ 不用重做昂贵的 DH ★ |
| **灵活** | 一条管理通道可保护多组不同的流量（多个感兴趣流） |

**★ 生存期差异**：阶段 1 默认 86400s（24h），阶段 2 默认 3600s（1h）。

---

**★ "阶段 1 READY 但业务不通" → 问题在阶段 2 或之后 ★**

**排查顺序**：
```
   ① show crypto ipsec sa
        ↓
   ┌─────────────────────────────────────┐
   │ ★ 情况 A：没有 IPsec SA ★            │
   │ → 阶段 2 协商失败                     │
   │ 检查：                                │
   │  · transform-set 两端是否一致          │
   │  · ★ 感兴趣流 ACL 是否【镜像】★        │
   │  · PFS 组是否一致                     │
   │  · mode（tunnel/transport）是否一致    │
   ├─────────────────────────────────────┤
   │ ★ 情况 B：有 SA，但 encaps=0 ★        │
   │ → ★ 流量根本没进隧道 ★                │
   │ 检查：                                │
   │  · 路由是否指向对端网段                │
   │  · ★ NAT 是否抢先处理了（超高频）★     │
   │  · 感兴趣流 ACL 是否覆盖实际流量        │
   ├─────────────────────────────────────┤
   │ ★ 情况 C：encaps 涨、decaps=0 ★       │
   │ → ★ 我发出去了，对端没回 ★            │
   │ 检查：                                │
   │  · 对端的路由/ACL                      │
   │  · 对端的感兴趣流是否镜像               │
   │  · 返回路径上有无防火墙                 │
   ├─────────────────────────────────────┤
   │ ★ 情况 D：两个都涨，但业务不通 ★       │
   │ → ★★ MTU 问题 ★★                     │
   │ 验证：                                │
   │  ping <对端> size 1500 df-bit → 失败   │
   │  ping <对端> size 1300 df-bit → 成功   │
   │ 修复：ip mtu 1400 +                   │
   │       ip tcp adjust-mss 1360          │
   └─────────────────────────────────────┘
```

**★ 记住这个判断表，IPsec 排障效率能提升一个数量级。**
</details>

**2.** GRE over IPsec 为什么用传输模式？纯 IPsec VTI 有什么局限？

<details><summary>答案</summary>

**★ 为什么用传输模式**：

```
   ★ 原始用户包 ★
   [ 192.168.1.10 → 192.168.2.10 | 数据 ]
        ↓ GRE 封装（+24 字节）
   [ 203.0.113.1 → 198.51.100.1 | GRE | 原包 ]
        ↑ ★ GRE 已经加了一个新的 IP 头 ★
        ↓
   ┌───────────────────────────────────────────┐
   │ 如果 IPsec 用【隧道模式】：                 │
   │ [ 新IP头 | ESP | 203.0.113.1→198.51.100.1 │
   │   | GRE | 原包 ]                           │
   │ ★ 两个几乎一样的外层 IP 头 = 浪费 20 字节 ★ │
   ├───────────────────────────────────────────┤
   │ 用【传输模式】：                            │
   │ [ 203.0.113.1→198.51.100.1 | ESP | GRE    │
   │   | 原包 ]                                 │
   │ ★ 复用 GRE 的 IP 头，节省 20 字节 ★         │
   └───────────────────────────────────────────┘
```

**开销对比**：
| 方案 | 总开销 |
|:--|:--|
| GRE over IPsec **传输模式** | ★ **≈ 60 字节** |
| GRE over IPsec **隧道模式** | ≈ 82 字节 |

**⚠️ 但注意**：**如果隧道两端之间有 NAT，仍然必须用隧道模式**（传输模式不保护也不允许 IP 头被改）—— 实际上 Cisco 会在检测到 NAT 时自动回退到隧道模式。

配置：
```cisco
crypto ipsec transform-set TS esp-aes 256 esp-sha256-hmac
 ★ mode transport ★
```

---

**★ 纯 IPsec VTI 的局限 ★**

```cisco
interface Tunnel0
 ★ tunnel mode ipsec ipv4 ★
```

| 能力 | **纯 IPsec VTI** | **GRE over IPsec** |
|:--|:--|:--|
| ★ **组播/广播** | ★ ❌ **不支持** | ★ ✅ 支持 |
| ★ **IGP（OSPF/EIGRP）** | ★ ❌ **跑不了** | ★ ✅ 可以 |
| **非 IP 协议** | ❌ 只支持 IPv4/IPv6 | ✅（CLNS、IPX 等） |
| 开销 | ★ **小**（≈50 字节） | 大（≈60-82 字节） |
| BGP | ✅ 可以（单播 TCP） | ✅ |
| 静态路由 | ✅ | ✅ |

**★ 选型决策**：
```
   要跑 OSPF/EIGRP？
        ↓ 是
   ★ GRE over IPsec（传输模式）★
        ↓ 否（只用静态路由或 BGP）
   ★ 纯 IPsec VTI（开销更小，配置更简单）★
```

**★ 为什么 VTI 比 crypto map 好（两种 VTI 都适用）**：
```
   ★ crypto map 的痛点 ★
   · 靠 ACL 定义感兴趣流 → ★ 加一个网段就要改两端 ACL ★
   · ★ 跑不了路由协议 ★（组播不匹配 ACL）
   · 不是接口 → QoS / NetFlow / ACL / 计费都不好挂
   · ★ 和 NAT 有处理顺序冲突 ★
   
   ★ VTI 的好处 ★
   · ★ 隧道就是一个接口 ★，路由指过来就加密
   · ★ 加网段只改路由，不改 ACL ★
   · 可以挂 QoS / ACL / NetFlow / 计费
   · show interface Tunnel0 直接看流量统计
   · ★ 是 DMVPN 和 SD-WAN 的基础 ★
   
   ★ 新部署一律用 VTI，crypto map 只在对接老设备时用 ★
```
</details>

**3.** 为什么 SSL VPN 取代了 IPsec 做远程接入？AnyConnect 为什么同时用 TLS 和 DTLS？

<details><summary>答案</summary>

**★ SSL VPN 胜出的核心原因：穿透性 ★**

| | **IPsec 远程接入** | **★ SSL VPN ★** |
|:--|:--|:--|
| 端口/协议 | UDP 500 + UDP 4500 + **协议 50** | ★ **TCP 443 / UDP 443** |
| ★ **在酒店/机场/客户网络** | ⚠️ ★ **经常被挡** ★ | ★ ✅ **443 几乎处处开放** |
| 客户端 | 专用客户端 + 复杂配置（组名、密钥、模式） | ★ 轻量客户端或纯浏览器 |
| ★ **NAT 后面** | 需要 NAT-T，仍可能有问题 | ★ ✅ **就是普通 HTTPS，天然没问题** |
| 用户配置 | 用户要填一堆参数 | ★ **一个 URL** |
| 访问粒度 | 网络级（给一个 IP 就全网可达） | ★ **可到应用级/URL 级** |
| 与准入/身份系统集成 | 弱 | ★ **强**（SAML、MFA、终端合规检查） |
| 性能 | ★ 略好 | 略差（TLS 开销），DTLS 弥补 |

**★ 决定性场景**：
```
   员工在客户公司出差，客户网络只放行 80/443
        ↓
   ★ IPsec VPN：UDP 500 被挡 → 完全连不上 ★
   ★ SSL VPN：走 443 → 正常连上 ★
        ↓
   ★★ 这就是为什么 IPsec 远程接入被淘汰了 ★★
```

---

**★ 为什么 AnyConnect 同时用 TLS 和 DTLS ★**

**问题：TCP over TCP（TCP Meltdown）**
```
   ★ 如果只用 TLS（跑在 TCP 443 上）★
   
   用户的 TCP 流量  →  被 TLS 封装  →  外层 TCP
        ↓ 网络丢一个包
   ★ 外层 TCP 检测到丢包 → 重传 + 降低窗口 ★
        ↓ 同时
   ★ 内层 TCP 也检测到延迟 → 也重传 + 也降窗口 ★
        ↓
   ★★ 两层拥塞控制互相打架 ★★
        ↓
   · 重传风暴
   · 吞吐量断崖式下跌
   · 延迟剧烈抖动
        ↓
   ★★ 这就是 "TCP Meltdown" ★★
```

**★ 解决方案**：
```
   ┌──────────────────────────────────────────┐
   │ ★ TLS 通道（TCP 443）★                    │
   │  · 认证、密钥协商                          │
   │  · 配置下发（IP 池、路由、DNS、Split Tunnel）│
   │  · ★ DTLS 不可用时的【数据回退通道】★       │
   ├──────────────────────────────────────────┤
   │ ★ DTLS 通道（UDP 443）★                   │
   │  · ★ 实际的用户数据流量 ★                  │
   │  · UDP 承载 → ★ 没有外层 TCP 重传 ★        │
   │  · 丢包由【内层】的 TCP 自己处理             │
   │  · ★ 行为和真实网络一致，性能正常 ★         │
   └──────────────────────────────────────────┘
```

**★ 实际效果**：
```
   同一条链路上：
   · 只用 TLS（TCP）：丢包 1% 时吞吐可能跌到 20%
   · 用 DTLS（UDP）：丢包 1% 时吞吐约 90%+
   
   ★ 这就是为什么 AnyConnect 连上后要检查
     "是否用了 DTLS" ★
```

**★ 排障要点**：
```
   用户反馈"VPN 连上了但特别慢"
        ↓
   ★ 检查是不是 DTLS 没建起来，退回了 TLS ★
        ↓
   常见原因：
   · ★ 防火墙挡了 UDP 443 ★
   · MTU 问题导致 DTLS 握手失败
   · 中间设备做深度包检测
        ↓
   AnyConnect 客户端统计里可以看到
   "Protocol: DTLS" 还是 "Protocol: TLS"
```
</details>

---

**下一章** → [04 802.1X 与 NAC](04-802.1X与NAC.md)
