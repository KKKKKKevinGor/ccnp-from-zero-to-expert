# 01 · SCOR 概览与安全模型

## ① SCOR 350-701 是什么

**Implementing and Operating Cisco Security Core Technologies** —— CCNP Security 的核心考试。

**考纲权重**：

| 域 | 权重 | 内容 |
|:--|:--|:--|
| 1.0 安全概念 | **25%** | 威胁、加密、安全模型、API 安全 |
| 2.0 网络安全 | **20%** | 防火墙、IPS、ACL、控制平面保护 |
| 3.0 云安全 | **15%** | 云部署模型、云安全责任、SaaS 安全 |
| 4.0 内容安全 | **15%** | 邮件/Web 安全、Umbrella |
| 5.0 终端保护 | **10%** | AMP、EDR、终端合规 |
| 6.0 安全网络访问 | **15%** | **802.1X、ISE、TrustSec、MFA** |

> **★ 域 2 和域 6 与 CCNP Enterprise 高度重叠**——这也是为什么 Enterprise 工程师学安全的成本很低。

---

## ② 安全的三个基本目标：CIA

| 目标 | 英文 | 含义 | 网络中的体现 |
|:--|:--|:--|:--|
| **机密性** | **Confidentiality** | 数据不被未授权者读取 | **加密**（IPsec、TLS、WPA3、MACsec） |
| **完整性** | **Integrity** | 数据不被篡改 | **哈希/HMAC**（SHA-256）、路由协议认证 |
| **可用性** | **Availability** | 服务持续可用 | **冗余、DDoS 防护、CoPP、风暴抑制** |

**★ 三者常有冲突**：
```
   加密强度 ↑  →  性能 ↓（可用性受影响）
   访问控制 ↑  →  易用性 ↓
   审计粒度 ↑  →  存储成本 ↑
   
   ★ 安全设计的本质是【权衡】，不是"越严越好" ★
```

**补充的两个目标**（常与 CIA 并列）：
- **认证性（Authenticity）**：确认对方身份 → 证书、802.1X
- **不可否认性（Non-repudiation）**：无法抵赖 → 数字签名、审计日志

---

## ③ 常见威胁分类

### 3.1 按攻击目标

| 类型 | 说明 | 网络侧的防护 |
|:--|:--|:--|
| **侦察（Reconnaissance）** | 扫描、探测、信息收集 | ACL、IPS、蜜罐、NetFlow 异常检测 |
| **访问（Access）** | 未授权访问、提权 | **802.1X、AAA、强密码、MFA** |
| **拒绝服务（DoS/DDoS）** | 耗尽资源 | **CoPP、风暴抑制、限速、云清洗** |
| **中间人（MITM）** | 窃听、篡改 | **加密、DAI、DHCP Snooping** |
| **恶意软件（Malware）** | 病毒、勒索、蠕虫 | **微分段**、终端防护、出站过滤 |

### 3.2 ★ 二层攻击（网工必知）★

| 攻击 | 原理 | ★ 防护 |
|:--|:--|:--|
| **MAC 泛洪** | 灌满 MAC 表，逼交换机泛洪，从而窃听 | ★ **Port Security** |
| **VLAN 跳跃（Switch Spoofing）** | 伪装成交换机协商 Trunk | ★ **`switchport mode access` + `nonegotiate`** |
| **VLAN 跳跃（Double Tagging）** | 双层标签，利用 Native VLAN | ★ **Native VLAN 改成未用的 + `vlan dot1q tag native`** |
| **DHCP Spoofing** | 假 DHCP 服务器，把网关指向自己 | ★ **DHCP Snooping** |
| **DHCP 耗尽** | 伪造大量 MAC 请求地址 | ★ **DHCP Snooping rate limit** |
| **ARP 欺骗** | 伪造 ARP 响应做中间人 | ★ **DAI（依赖 DHCP Snooping）** |
| **IP 盗用** | 手工改 IP 绕过策略 | ★ **IPSG（依赖 DHCP Snooping）** |
| **STP 攻击** | 发送优先级极高的 BPDU 抢根桥 | ★ **BPDU Guard + Root Guard** |
| **CDP/LLDP 侦察** | 收集设备信息 | 面向不可信网络的接口关闭 CDP |

**★ 接入层安全"三件套"的依赖链**：
```
   ★ DHCP Snooping ★（建立 MAC↔IP↔端口 绑定表）
          ↓ 提供数据基础
     ┌────┴────┐
   ★ DAI ★   ★ IPSG ★
   防ARP欺骗  防IP盗用
```
**必须先开 DHCP Snooping，DAI 和 IPSG 才有数据可用。**

### 3.3 三层及以上攻击

| 攻击 | 防护 |
|:--|:--|
| **IP 源地址欺骗** | ★ **uRPF** |
| **路由协议注入** | ★ **路由协议认证 + passive-interface** |
| **BGP 前缀劫持** | ★ **prefix-list 过滤 + maximum-prefix + RPKI** |
| **TCP SYN Flood** | 限速、SYN Cookie、清洗设备 |
| **DNS 投毒/放大** | DNSSEC、限制递归、响应速率限制 |
| **应用层攻击** | WAF、下一代防火墙、IPS |

---

## ④ ★ 纵深防御（Defense in Depth）★

**核心思想：不依赖单一防护点，多层叠加。**

```
   ┌────────────────────────────────────────────┐
   │ ★ 策略与流程 ★                              │
   │  安全策略、变更管理、应急响应、培训            │
   ├────────────────────────────────────────────┤
   │ ★ 物理安全 ★                                │
   │  机房门禁、设备上锁、Console 口保护            │
   ├────────────────────────────────────────────┤
   │ ★ 边界 ★                                    │
   │  防火墙、IPS、DDoS 防护、出站过滤              │
   ├────────────────────────────────────────────┤
   │ ★ 网络内部 ★                                │
   │  VLAN/VRF 分段、微分段(SGT)、ACL、uRPF        │
   ├────────────────────────────────────────────┤
   │ ★ 接入 ★                                    │
   │  802.1X、Port Security、DHCP Snooping/DAI/IPSG│
   ├────────────────────────────────────────────┤
   │ ★ 主机与终端 ★                              │
   │  补丁、EDR、主机防火墙、磁盘加密                │
   ├────────────────────────────────────────────┤
   │ ★ 应用与数据 ★                              │
   │  WAF、加密存储、访问控制、DLP                  │
   ├────────────────────────────────────────────┤
   │ ★ 监控与响应 ★（贯穿所有层）                  │
   │  SIEM、NetFlow、日志、威胁情报                 │
   └────────────────────────────────────────────┘
```

**★ 网络工程师的主战场是"边界 / 网络内部 / 接入"三层。**

---

## ⑤ 网络设备的三个平面

| 平面 | 处理什么 | 攻击面 | 防护 |
|:--|:--|:--|:--|
| **数据平面 Data Plane** | 用户流量转发 | 恶意流量穿透 | ACL、uRPF、QoS、风暴抑制 |
| **★ 控制平面 Control Plane ★** | 路由协议、ARP、STP | ★ **CPU 耗尽** | ★ **CoPP/CPPr、路由协议认证** |
| **管理平面 Management Plane** | SSH、SNMP、NetFlow | 未授权访问 | ★ **AAA、ACL 限源、SSHv2、关闭不用的服务** |

**★ 控制平面攻击的可怕之处**：
```
   攻击者不需要"攻破"设备，只需要发送大量
   【必须由 CPU 处理】的包（ARP、TTL=1、SSH 请求）
        ↓
   ★ CPU 100% ★
        ↓
   · 路由协议 Hello 处理不过来 → 邻居断开
   · SSH 登不上去 → 失去管理能力
   · 整台设备失去响应
   
   ★ 设备"没被攻破"，但已经瘫痪了 ★
```

**详见 [ENCOR 13 §4](../02-ENCOR-350-401/13-网络安全-AAA-802.1X-控制平面保护.md)**。

---

## ⑥ 加密基础（考试常考概念）

### 6.1 对称 vs 非对称

| | **对称加密** | **非对称加密** |
|:--|:--|:--|
| 密钥 | **同一个密钥**加解密 | **公钥加密，私钥解密** |
| 速度 | ★ **快** | 慢（约 1000 倍） |
| 密钥分发 | ★ **难**（怎么安全地把密钥给对方） | ★ **容易**（公钥可公开） |
| 算法 | **AES**、3DES、DES | **RSA**、**ECC**、DSA |
| 用途 | ★ **加密大量数据** | ★ **密钥交换、数字签名** |

**★ 实际中两者结合**（TLS/IPsec 都是这样）：
```
   ① 用【非对称】安全地协商出一个【对称密钥】
        （Diffie-Hellman 密钥交换）
   ② 用【对称】加密实际的数据
        ↓
   ★ 既解决了密钥分发，又保证了性能 ★
```

### 6.2 哈希与 HMAC

| | 说明 |
|:--|:--|
| **哈希（Hash）** | 单向函数，任意长度 → 固定长度。**只保证完整性** |
| 常用算法 | ★ **SHA-256**、SHA-1（已不安全）、MD5（已不安全） |
| **HMAC** | **哈希 + 密钥**。同时保证**完整性 + 认证性** |
| 用途 | 路由协议认证、IPsec 完整性校验、密码存储 |

**★ 为什么单纯的哈希不够**：
```
   攻击者篡改数据 → 重新计算哈希 → 一起发过去
        ↓
   ★ 接收方验证通过（哈希是对的）★
        ↓
   → 必须加密钥（HMAC），攻击者没有密钥就算不出正确的 HMAC
```

### 6.3 Diffie-Hellman（DH）

**在不安全的信道上协商出共享密钥。**

| DH Group | 强度 | 说明 |
|:--|:--|:--|
| Group 1/2/5 | ❌ 已不安全 | 768/1024/1536 位 |
| **Group 14** | ✅ 2048 位 | ★ **最低推荐** |
| Group 15/16 | ✅ 3072/4096 位 | 更强 |
| **Group 19/20/21** | ✅ **ECC** | ★ **推荐**，性能更好 |

**★ IPsec 配置中的 `group 14` 就是这个。**

### 6.4 PKI（公钥基础设施）

```
   ★ CA（证书颁发机构）★
        │ 签发
   ┌────┴────┐
   证书       证书
   （包含公钥 + 身份信息 + CA 签名）
        ↓
   验证方用 CA 的公钥验证签名
        ↓
   ★ 确认"这个公钥确实属于这个身份" ★
```

**证书的关键字段**：
- **Subject**：证书主体（谁的）
- **Issuer**：签发者（哪个 CA）
- **Validity**：★ **有效期**（这就是为什么 NTP 重要）
- **Public Key**：公钥
- **Signature**：CA 的签名

**★ 在网络中的应用**：
```
   · HTTPS / TLS
   · IPsec 证书认证（比预共享密钥更安全，适合大规模）
   · 802.1X EAP-TLS
   · CAPWAP 的 DTLS（AP 和 WLC 互认）
   · SSH 主机密钥
```

> ⚠️ **★ 证书验证依赖系统时间。** NTP 不同步 → 证书"尚未生效"或"已过期" → 认证全部失败。
> **这是很多"莫名其妙认证失败"的根因**（见 [排障案例 10](../05-排障方法论/04-真实故障案例集.md)）。

---

## ⑦ Cisco 安全产品线（了解即可）

| 产品 | 用途 |
|:--|:--|
| **ASA / Firepower (FTD)** | 防火墙 / 下一代防火墙 |
| **ISE** | ★ 身份服务引擎（802.1X、TrustSec、准入） |
| **Umbrella** | 云 DNS 安全（原 OpenDNS） |
| **AMP for Endpoints (Secure Endpoint)** | 终端防护 / EDR |
| **ESA / WSA** | 邮件 / Web 安全网关 |
| **Stealthwatch (Secure Network Analytics)** | ★ 基于 NetFlow 的异常检测 |
| **Duo** | 多因素认证（MFA） |
| **SecureX** | 安全产品的统一编排平台 |

---

## ⑧ ★ 实用的加固检查清单 ★

**这是一份可以直接拿去用的清单。**

### 管理平面
```
□ 关闭 Telnet，只用 SSHv2
□ 配置 AAA（TACACS+），★ 带 local 保底 ★
□ VTY 配 access-class 限制来源 IP
□ 配置 login block-for 防暴力破解
□ 关闭不用的服务（http server、small-servers、bootp、source-route）
□ 配置 banner（法律意义）
□ ★ 配置 archive log config（审计谁改了什么）★
□ exec-timeout 不超过 15 分钟
□ ★ 独立的带外管理网络 ★
```

### 控制平面
```
□ ★ 路由协议认证 ★（OSPF/EIGRP/BGP 全部）
□ ★ passive-interface default ★，只放开必要的
□ ★ CoPP ★（先监控收集基线，再限速）
□ BGP：maximum-prefix + prefix-list 过滤 + TTL security
□ 关闭 IP redirects / unreachables（面向不可信网络的接口）
□ ★ NTP 认证 ★
```

### 数据平面
```
□ ★ uRPF ★（单归属边界用 strict，多归属用 loose）
□ 出口 ACL：入向过滤 Bogon、私网、碎片前缀
□ ★ 出站过滤 ★（限制内网主动访问的高危端口：445/135/139）
□ NAT：max-entries all-host 限制
□ QoS 保护关键业务
```

### 接入层（★ 最容易被忽略但最重要）
```
□ ★ 802.1X + MAB ★（配 Critical Auth 降级）
□ Port Security（violation restrict）
□ ★ DHCP Snooping ★（★ 上联口 trust ★）
□ ★ DAI ★（静态设备手工绑定）
□ ★ IPSG ★
□ ★ BPDU Guard + PortFast ★
□ storm-control
□ switchport mode access + nonegotiate
□ ★ 未使用的端口：shutdown + 划入黑洞 VLAN ★
□ Native VLAN 改成未使用的 VLAN
```

### 监控与响应
```
□ ★ NTP 同步（所有设备）★
□ Syslog 集中收集（★ 带时间戳和源接口 ★）
□ SNMPv3（不用 v2c；必须用 v2c 时配 ACL + 复杂 community + 只读）
□ ★ NetFlow ★（流量可视化 + 异常检测）
□ ★ 配置自动备份 ★（Oxidized/RANCID）
□ 关键告警配置（CPU、内存、接口错误、BGP 前缀数突变、NAT 表项）
```

---

## ⑨ ★ 安全设计的三个原则 ★

```
   ★ 1. 最小权限（Least Privilege）★
      · 只给必要的访问权限
      · ACL 用"默认拒绝 + 显式允许"
      · 账号只给需要的权限级别
      · 服务只开必要的端口
   
   ★ 2. 默认拒绝（Default Deny）★
      · ACL 末尾 deny any（隐含的，但建议显式写 + log）
      · VRF 之间默认不通，需要互访才配
      · 未使用的端口默认 shutdown
      · 新设备默认不可信，认证通过才放行
   
   ★ 3. 纵深防御（Defense in Depth）★
      · 不依赖单点
      · 边界 + 内部 + 接入 + 终端多层
      · 一层被突破，还有下一层
```

**★ 一个反例（真实场景）**：
```
   "我们有防火墙，内网是安全的"
        ↓
   · 员工点了钓鱼邮件 → 恶意软件在内网运行
   · 内网没有分段 → ★ 横向移动畅通无阻 ★
   · 445 端口全网开放 → 勒索软件快速传播
   · 没有出站过滤 → C2 通信一路畅通
        ↓
   ★★ 防火墙一点用都没有 ★★
   
   ★ 这就是为什么需要"微分段"和"零信任"★
   → 见 [05 零信任与网络分段](05-零信任与网络分段.md)
```

---

## ⑩ 自测题

**1.** 二层攻击"三件套"防护（DHCP Snooping / DAI / IPSG）的依赖关系是什么？部署顺序？

<details><summary>答案</summary>

**依赖关系**：
```
   ★ DHCP Snooping ★（基础）
      建立绑定表：MAC ↔ IP ↔ VLAN ↔ 端口 ↔ 租期
          ↓ 提供数据基础
     ┌────┴────┐
   ★ DAI ★   ★ IPSG ★
   （动态ARP检测） （IP源防护）
   防 ARP 欺骗    防 IP 盗用
```

**DAI 和 IPSG 都依赖 DHCP Snooping 的绑定表**，必须先开 Snooping。

**★ 部署顺序（必须遵守）**：
```
   ① 开 DHCP Snooping
      ★ 上联口设为 trust ★（忘了 = 全网拿不到 IP）
        ↓
   ② 跑一个完整的 DHCP 租期周期
      让绑定表填充完整
        ↓
   ③ 检查绑定表覆盖率
      show ip dhcp snooping binding
        ↓
   ④ ★★ 为所有静态 IP 设备添加手工绑定 ★★
      （服务器、打印机、摄像头、门禁）
        ↓
   ⑤ 开 DAI
        ↓
   ⑥ 开 IPSG
```

**★ 跳过第 ④ 步 = 所有静态 IP 设备断网**（它们不用 DHCP，不在绑定表里，会被 DAI/IPSG 判定为伪造）。

**静态绑定配置**：
```cisco
! IPSG 的静态绑定
SW1(config)# ip source binding 0011.2233.4455 vlan 10 192.168.10.200 interface Gi1/0/30

! DAI 的静态绑定（用 ARP ACL）
SW1(config)# arp access-list STATIC-DEVICES
SW1(config-arp-nacl)#  permit ip host 192.168.10.200 mac host 0011.2233.4455
SW1(config)# ip arp inspection filter STATIC-DEVICES vlan 10
```

**完整配置**：
```cisco
! 全局
SW1(config)# ip dhcp snooping
SW1(config)# ip dhcp snooping vlan 10,20,100
SW1(config)# ip arp inspection vlan 10,20,100
SW1(config)# ip arp inspection validate src-mac dst-mac ip

! 上联口（信任）
SW1(config)# interface Gi1/0/48
SW1(config-if)# ★ ip dhcp snooping trust ★
SW1(config-if)# ★ ip arp inspection trust ★

! 接入口
SW1(config)# interface range Gi1/0/1-20
SW1(config-if-range)# ip dhcp snooping limit rate 10
SW1(config-if-range)# ip arp inspection limit rate 15
SW1(config-if-range)# ip verify source

! err-disable 自动恢复
SW1(config)# errdisable recovery cause arp-inspection
SW1(config)# errdisable recovery cause dhcp-rate-limit
SW1(config)# errdisable recovery interval 300
```
</details>

**2.** 对称加密和非对称加密的区别？为什么实际中要结合使用？

<details><summary>答案</summary>

| | **对称加密** | **非对称加密** |
|:--|:--|:--|
| 密钥 | **同一个密钥**加解密 | 公钥加密，私钥解密 |
| 速度 | ★ **快**（硬件加速） | 慢（约 1000 倍） |
| 密钥分发 | ★ **难**（怎么安全地传密钥？） | 容易（公钥可公开） |
| 算法 | AES、3DES | RSA、ECC、DH |
| 适合 | ★ **加密大量数据** | ★ **密钥交换、数字签名** |

**★ 为什么要结合**：

**对称加密的困境**：
```
   A 和 B 要用 AES 通信，需要共享一个密钥
        ↓
   ★ 但怎么把这个密钥安全地传给对方？★
   （如果信道是安全的，就不需要加密了；
     如果信道不安全，密钥会被截获）
        ↓
   ★ 这就是"密钥分发问题" ★
```

**非对称加密的困境**：
```
   RSA 加密 1GB 的文件？
        ↓
   ★ 慢到不可用（比 AES 慢 1000 倍）★
```

**★ 结合方案（TLS/IPsec 都是这样）**：
```
   ① 用【非对称】/【DH】安全地协商出一个【对称密钥】
      · Diffie-Hellman：双方各自生成一部分，通过公开信道交换，
        各自算出【相同的】共享密钥，★ 而窃听者算不出 ★
      · 或用 RSA：A 生成对称密钥，用 B 的公钥加密后发给 B
        ↓
   ② 用这个【对称密钥】+ AES 加密实际数据
        ↓
   ★ 既解决了密钥分发，又保证了性能 ★
```

**在 IPsec 中的体现**：
```cisco
crypto ikev2 proposal PROP-1
 ★ encryption aes-cbc-256 ★    ← 对称加密（加密数据）
 ★ integrity sha256 ★          ← HMAC（完整性）
 ★ group 14 ★                  ← DH（协商对称密钥）
```

**★ PFS（Perfect Forward Secrecy）**：
```cisco
crypto ipsec profile IPSEC-PROF
 ★ set pfs group14 ★
```
每次重新协商 SA 时**重新做一次 DH**，生成全新的密钥。
**效果**：即使某个密钥泄露，**之前的通信记录也解不开**。
</details>

**3.** 控制平面攻击为什么可怕？怎么防护？

<details><summary>答案</summary>

**★ 可怕之处：攻击者不需要"攻破"设备，只需要让它"忙不过来"。**

**原理**：
```
   网络设备的转发分两条路径：
   
   ★ 数据平面 ★：用户流量 → ASIC 硬件转发（线速，CPU 不参与）
   ★ 控制平面 ★：某些包必须由 CPU 处理
                  · 路由协议报文（OSPF Hello、BGP Update）
                  · ARP 请求/响应
                  · 目的是设备自身的包（SSH、SNMP、ping 路由器）
                  · TTL=1 的包（要回 ICMP Time Exceeded）
                  · 带 IP Options 的包
                  · 无法 CEF 转发的包
        ↓
   攻击者发送大量【必须由 CPU 处理】的包
        ↓
   ★★ CPU 100% ★★
        ↓
   · 路由协议 Hello 处理不过来 → ★ 邻居超时断开 → 路由震荡 ★
   · SSH 登不上去 → ★ 失去管理能力 ★
   · 整台设备失去响应
   
   ★ 设备"没被入侵"，但已经完全瘫痪 ★
   ★ 而且门槛极低——一个脚本就能做到 ★
```

**防护手段**：

**① ★ CoPP（Control Plane Policing）★ —— 核心手段**
```cisco
! 给送往 CPU 的流量做 QoS 限速
control-plane
 service-policy input COPP-POLICY
```

**★ 分类原则**：
| 类别 | 策略 | 理由 |
|:--|:--|:--|
| **路由协议** | ★ **不限速（只监控）** | 断了就是灾难 |
| 管理流量（SSH/SNMP） | 中等限速 + **源 ACL 限制** | 保证管理通道 |
| ICMP | 严格限速 | 常被用于攻击 |
| 明确不需要的（Telnet/TFTP） | **直接丢弃** | 减少攻击面 |
| class-default | 限速 | 兜底 |

**★ 部署流程（关键）**：
```
   ① 先用【只监控不丢弃】的策略跑 1-2 周
      （所有 exceed-action 都设为 transmit）
        ↓
   ② 看实际流量，确定基线
      show policy-map control-plane
        ↓
   ③ 用实测值 × 2-3 倍作为正式阈值
        ↓
   ④ ★ 用 configure revert timer 做保险 ★
        ↓
   ⑤ 逐台部署，不要批量下发
```

**⚠️ 最大的风险：把自己锁死。** 限得太狠 → SSH 登不上，路由邻居全断。

**② ★ 路由协议认证 ★**
```cisco
! 防止伪造邻居注入恶意路由
key chain ROUTING-KEYS
 key 1
  key-string MySecret
  cryptographic-algorithm hmac-sha-256

interface Gi0/1
 ip ospf authentication key-chain ROUTING-KEYS
```

**③ ★ passive-interface default ★**
```cisco
router ospf 1
 ★ passive-interface default ★
 no passive-interface GigabitEthernet0/1
```
**只在真正需要建邻居的接口上发 Hello**，其他接口不发 → 攻击者无法在接入端口伪造邻居。

**④ 管理平面加固**
```cisco
line vty 0 15
 ★ access-class MGMT-HOSTS in ★
 transport input ssh
 exec-timeout 10 0

login block-for 300 attempts 5 within 60
no ip http server
no service tcp-small-servers
```

**⑤ 关闭不必要的响应**
```cisco
interface Gi0/1
 no ip redirects
 no ip unreachables      ! ⚠️ 会影响 PMTUD 和 traceroute
 no ip proxy-arp
 no ip directed-broadcast
```

> ⚠️ **`no ip unreachables` 的副作用**：会禁止 ICMP Unreachable，包括 **PMTUD 需要的 "Fragmentation Needed"**。
> **在有隧道的环境关掉它会导致 MTU 黑洞。**
> **建议**：只在面向不可信网络的接口关，或用 CoPP 限速而不是完全禁用。

**监控**：
```cisco
show processes cpu sorted | exclude 0.00
show policy-map control-plane          ! 看 exceeded 计数是否突增
```
</details>

---

**下一章** → [02 ACL 与区域防火墙 ZBF](02-ACL与区域防火墙ZBF.md)
