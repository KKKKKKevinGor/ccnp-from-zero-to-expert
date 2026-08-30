# 15 · SD-Access 与 SD-WAN

> **本章是前面所有技术的集大成。** VXLAN、LISP、VRF、IPsec、BGP、802.1X、API —— 在这里组装成完整的解决方案。

## ① 这章解决什么问题

**传统网络的三个根本痛点**：

1. **配置分散**。改一个策略要登录几十台设备逐台配，容易漏、容易不一致、无法回滚。
2. **策略绑定 IP**。ACL 写的是 IP 地址，人一换工位、设备一换网段，策略就失效。
3. **手工运维**。故障靠人查，容量靠人估，变更靠人敲。

**SDN（软件定义网络）的解法**：

```
   传统网络：                       SDN：
   
   每台设备自己算路由、自己做策略      ┌──────────────────┐
   （控制平面分布在每台设备上）        │   ★ 控制器 ★      │  集中的控制平面
                                     │  统一计算、统一下发 │
   [SW1] [SW2] [SW3] [SW4]           └────────┬─────────┘
    ↑     ↑     ↑     ↑                       │ API 下发
   各自为政                          [SW1] [SW2] [SW3] [SW4]
                                      ↑ 只负责转发（数据平面）
```

**核心思想：控制平面与数据平面分离，控制平面集中化。**

Cisco 的两个落地方案：
- **SD-Access** → 园区网（Campus）
- **SD-WAN** → 广域网（WAN）

---

## ② SD-Access

### 2.1 技术组成（★ 核心考点）

**SD-Access = LISP（控制面）+ VXLAN（数据面）+ TrustSec/SGT（策略面）**

| 平面 | 技术 | 作用 |
|:--|:--|:--|
| **控制平面** | **LISP** | 维护"哪个终端（EID）在哪台交换机（RLOC）后面"的映射 |
| **数据平面** | **VXLAN** | 封装转发，VNI 区分不同的虚拟网络 |
| **策略平面** | **Cisco TrustSec (SGT)** | 用**安全组标签**做策略，与 IP 完全解耦 |
| **底层网络** | **IS-IS**（默认） | Underlay 路由，保证各节点 Loopback 互通 |
| **管理编排** | **DNA Center** | 图形化配置、自动化下发、遥测分析 |

> **这三个技术分别在 [第 3 章](03-Overlay-VXLAN与LISP.md)（LISP/VXLAN）和 [第 13 章](13-网络安全-AAA-802.1X-控制平面保护.md)（TrustSec）讲过。** SD-Access 是把它们组装起来。

### 2.2 Fabric 角色

```
        ┌──────────────────────────────────────────┐
        │       DNA Center（管理与编排）             │
        │   · 图形化配置  · 自动化下发                │
        │   · 策略定义    · 遥测分析                  │
        └───────────────────┬──────────────────────┘
                            │
        ┌───────────────────▼──────────────────────┐
        │   Control Plane Node（LISP Map-Server）   │
        │   维护 EID → RLOC 映射数据库               │
        └───────────────────┬──────────────────────┘
                            │
   ┌────────────────────────┼────────────────────────┐
   │                        │                        │
┌──▼──────────┐   ┌─────────▼────────┐   ┌───────────▼────┐
│  Edge Node  │   │   Edge Node      │   │  Border Node   │
│ （接入交换机）│   │                  │   │ （对接外部网络）│
│ VTEP + xTR  │   │                  │   │                │
│ 终端的默认网关│   │                  │   │                │
└──┬──────────┘   └──────────────────┘   └────────────────┘
   │
[终端] [AP]
```

| 角色 | 职责 | 通常是什么设备 |
|:--|:--|:--|
| **Control Plane Node** | LISP 的 **Map-Server / Map-Resolver**，维护终端映射数据库 | 核心交换机 / 独立设备 |
| **Edge Node** | **接入交换机**。是 **VTEP**（VXLAN 封装）+ **xTR**（LISP），**终端的默认网关** | Catalyst 9300 |
| **Border Node** | Fabric 与**外部网络**的边界，做路由泄露和 SGT 转换 | Catalyst 9500 |
| **Intermediate Node** | 纯转发（Underlay 路由），不参与 Fabric | Catalyst 9500 |
| **Fabric WLC** | 无线控制器，AP 也成为 Fabric 的一部分 | C9800 |
| **Fabric AP** | AP 直接做 VXLAN 封装，无线和有线策略统一 | C9100 系列 |

**Border Node 的三种类型**：

| 类型 | 用途 |
|:--|:--|
| **Internal Border** | 连接内部的已知网络（数据中心、其他园区） |
| **External Border** | 连接未知网络（互联网），是默认路由的出口 |
| **Anywhere Border** | 两者兼具 |

### 2.3 两个关键抽象

#### Virtual Network (VN) —— 宏分段

**VN 本质上就是 VRF**，提供**完全的路由隔离**。

```
   VN: EMPLOYEE   → VRF EMPLOYEE   → VNI 4097
   VN: GUEST      → VRF GUEST      → VNI 4098
   VN: IOT        → VRF IOT        → VNI 4099
   
   ★ 不同 VN 之间默认完全不通（路由表隔离）★
   需要互访必须经过防火墙（Fusion Router）
```

**这叫"宏分段（Macro-segmentation）"**——大颗粒度的隔离。

#### Scalable Group Tag (SGT) —— 微分段

**在同一个 VN 内部，用 SGT 做更细粒度的策略。**

```
   VN: EMPLOYEE 内部：
   
   SGT 10 = 研发部
   SGT 20 = 财务部
   SGT 30 = 打印机
   
   策略矩阵：
              → SGT10  SGT20  SGT30
   SGT 10 研发   允许    ❌拒绝  允许
   SGT 20 财务   ❌拒绝  允许    允许
   SGT 30 打印机 ❌拒绝  ❌拒绝  ❌拒绝
```

**这叫"微分段（Micro-segmentation）"**。

**★ SGT 的革命性在于：策略与 IP 完全解耦。**

```
   传统 ACL：
   permit ip 10.1.10.0 0.0.0.255 host 10.1.30.50
              ↑ 绑定 IP 地址
   → 员工换工位换了 IP → 策略失效，要改 ACL
   → 网段调整 → 所有相关 ACL 都要改
   
   SGT 策略：
   permit SGT:10 → SGT:100
          ↑ 绑定"身份"，不是地址
   → 员工无论在哪、IP 是什么，SGT 都跟着他走
   → ★ 策略永远有效 ★
```

**SGT 的分配方式**：
- **动态**：802.1X 认证通过后，ISE 在 RADIUS Access-Accept 里下发 SGT
- **静态**：绑定到 IP、VLAN、端口或子网

**SGT 的传播（SXP）**：
```cisco
! 支持 inline SGT 的设备直接在 VXLAN/以太帧里携带 SGT
! 不支持的设备用 SXP 协议传递 IP-SGT 映射
R1(config)# cts sxp enable
R1(config)# cts sxp default password MySxpPassword
R1(config)# cts sxp connection peer 10.1.30.70 password default mode local listener
```

### 2.4 SD-Access 解决了什么

| 传统园区网的痛点 | SD-Access 的解法 |
|:--|:--|
| VLAN 只有 4094 个，跨设备延伸麻烦 | **VXLAN VNI 1600 万个**，跨三层自由延伸 |
| 终端移动要换 IP/VLAN | **LISP**，身份与位置分离，**移动时 IP 和策略不变** |
| 策略基于 IP，地址一变就失效 | **SGT**，策略基于身份标签 |
| STP 浪费带宽、收敛慢 | **三层 Underlay + ECMP**，无 STP |
| 有线和无线策略割裂 | **统一策略**（Fabric AP 也在 Fabric 内） |
| 配置分散在几十台设备上 | **DNA Center 集中编排** |

**举个具体例子**：

一个员工的笔记本从 3 楼移动到 8 楼：

| | **传统网络** | **SD-Access** |
|:--|:--|:--|
| 接入交换机 | 换了 | 换了 |
| VLAN | 换了 | **不变**（同一个 VN） |
| IP 地址 | **换了** | **不变**（LISP 映射更新） |
| 安全策略 | **失效，要重配 ACL** | **不变**（SGT 跟随身份） |
| 用户感知 | **连接中断** | **无感知** |

### 2.5 部署考量

**优势**：
- 策略与位置解耦，运维大幅简化
- 集中化管理，配置一致性有保障
- 丰富的遥测和分析（DNA Center Assurance）
- 微分段能力，符合零信任方向

**代价（必须清楚）**：

| 代价 | 说明 |
|:--|:--|
| **硬件要求高** | 必须是 Catalyst 9000 系列（老设备不支持） |
| **License 昂贵** | DNA Advantage / Premier |
| **学习曲线陡** | 要理解 LISP + VXLAN + TrustSec + ISE |
| **排障复杂** | 出问题要同时看 Underlay、Overlay、策略三层 |
| **强绑定 Cisco** | 完全的厂商锁定 |
| **DNA Center 是单点** | 需要集群部署（3 节点） |

> **实践建议**：SD-Access 适合**大型园区（1000+ 终端）、有强安全需求、且已经是 Cisco 全栈**的场景。中小企业上它是**杀鸡用牛刀**——成本和复杂度远超收益。
>
> **考试要理解概念，实际选型要理性。**

---

## ③ SD-WAN

### 3.1 传统 WAN 的痛点

```
   分支机构 A ──MPLS专线──┐
                          ├── 总部
   分支机构 B ──MPLS专线──┘
   
   问题：
   ① ★ MPLS 专线贵 ★（是宽带的 5-20 倍）
   ② 带宽扩容周期长（几周到几个月）
   ③ 访问云服务（Office 365、SaaS）要先绕回总部再出去 —— 延迟高、浪费带宽
   ④ 备份链路（4G/宽带）平时闲置，浪费
   ⑤ 每个分支要单独配置，开分店要网工出差
   ⑥ 无法感知应用，所有流量一视同仁
```

### 3.2 SD-WAN 的解法

```
                    ┌─────────────────────────┐
                    │   vManage（管理编排）     │
                    │   vSmart（控制器）        │
                    │   vBond（编排器）         │
                    └────────────┬────────────┘
                                 │ 控制平面（DTLS/TLS）
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
   ┌────▼─────┐           ┌──────▼────┐            ┌──────▼────┐
   │ vEdge/   │           │  vEdge    │            │  vEdge    │
   │ cEdge    │═══IPsec═══│           │═══IPsec════│           │
   │ 分支A     │           │  分支B     │            │  总部      │
   └──────────┘           └───────────┘            └───────────┘
        │                        │
   [MPLS + 宽带 + 4G]      [多种传输链路同时使用]
```

**核心能力**：

| 能力 | 说明 |
|:--|:--|
| **传输无关** | MPLS、宽带、4G/5G、卫星 —— 全部当作"传输通道"，上层统一 |
| **多链路同时使用** | 不再是主备，而是**根据应用智能选路** |
| **应用感知选路** | 语音走延迟低的链路，备份走带宽大的链路 |
| **零接触部署（ZTP）** | 新设备开箱上电，**自动**从 vBond 获取配置 |
| **集中策略** | 在 vManage 上定义一次，推送到所有分支 |
| **本地互联网出口** | SaaS 流量直接本地出网，不绕总部 |

### 3.3 四大组件（★ 必考）

| 组件 | 平面 | 作用 | 类比 |
|:--|:--|:--|:--|
| **vManage** | **管理平面** | 图形化管理、配置模板、监控、API | **控制台** |
| **vSmart** | **控制平面** | **用 OMP 协议分发路由和策略**，不转发数据 | **大脑**（像 BGP Route Reflector） |
| **vBond** | **编排平面** | **认证和牵引**：新设备首先联系它，它负责验证身份并告知 vSmart/vManage 的地址；还负责 **NAT 穿越** | **接待处** |
| **vEdge / cEdge** | **数据平面** | 实际转发流量的边缘路由器 | **手脚** |

**vEdge vs cEdge**：

| | **vEdge** | **cEdge** |
|:--|:--|:--|
| 来源 | Viptela 原生 | Cisco IOS-XE SD-WAN |
| 操作系统 | Viptela OS | **IOS-XE** |
| 硬件 | vEdge 100/1000/2000 | **ISR 1000/4000、ASR 1000、Catalyst 8000** |
| 现状 | 逐步淘汰 | ★ **主流** |

### 3.4 设备上线流程（ZTP，★ 考点）

```
   ① 新设备开箱上电，接上网线
        ↓
   ② 通过 DHCP 获取 IP
        ↓
   ③ 通过 DNS 或预配置找到 ★ vBond ★
        ↓
   ④ 与 vBond 建立 DTLS 连接，★ 用证书互相认证 ★
        ↓
   ⑤ vBond 验证通过 → 告诉设备 vSmart 和 vManage 的地址
        ↓
   ⑥ 设备与 vManage 建立连接 → ★ 下载配置模板 ★
        ↓
   ⑦ 设备与 vSmart 建立 ★ OMP 会话 ★ → 学习路由和策略
        ↓
   ⑧ 与其他 vEdge 自动建立 ★ IPsec 隧道 ★（数据平面）
        ↓
   ⑨ 上线，开始转发流量
```

> **ZTP 的业务价值**：开一家新分店，**快递把设备寄过去，店员插上电源和网线就完成了**。不需要网工出差，不需要现场配置。
>
> 这在连锁零售、餐饮行业是巨大的成本节省——**从"每开一店派一个工程师飞过去"变成"快递发货"**。

### 3.5 OMP（Overlay Management Protocol）

**SD-WAN 的"BGP"**，在 vSmart 和 vEdge 之间运行。

**通告三类信息**：

| 类型 | 内容 |
|:--|:--|
| **OMP Route** | 前缀（哪个分支有哪些网段） |
| **TLOC Route** | **传输定位符**（Transport Locator）：某个 vEdge 的某条链路的信息 |
| **Service Route** | 服务（防火墙、IPS 等，用于服务链） |

**TLOC 是 SD-WAN 的核心概念**：
```
   TLOC = {系统IP, 颜色(color), 封装类型}
   
   例：
   TLOC-1 = {10.1.1.1, mpls,     ipsec}
   TLOC-2 = {10.1.1.1, biz-internet, ipsec}
   TLOC-3 = {10.1.1.1, lte,      ipsec}
   
   ★ 同一台设备的三条不同链路 = 三个 TLOC ★
```

**Color（颜色）** 标识传输类型：

| 类别 | Color | 说明 |
|:--|:--|:--|
| **私有 color** | `mpls`, `metro-ethernet`, `private1-6` | 私网，**用私网 IP 建隧道** |
| **公有 color** | `biz-internet`, `public-internet`, `lte`, `3g`, `blue`, `red` 等 | 公网，**用公网 IP 建隧道，支持 NAT 穿越** |

**隧道建立规则（考点）**：
- **相同 color 之间**：直接建隧道
- **不同 color 之间**：默认也建（除非用策略限制）
- **私有 color 之间**：用私网地址
- **公有 color 之间**：用公网地址（NAT 后的）

```
   分支A: mpls + biz-internet
   分支B: mpls + biz-internet
        ↓
   建立 2×2 = 4 条 IPsec 隧道
   （mpls↔mpls, mpls↔internet, internet↔mpls, internet↔internet）
   
   ★ 流量可以根据应用和链路质量在这些隧道间智能选择 ★
```

### 3.6 应用感知路由（AAR）—— SD-WAN 的杀手锏

**根据实时测量的链路质量（延迟/抖动/丢包），自动为不同应用选择最佳路径。**

```
   持续用 BFD 测量每条隧道的质量：
   
   MPLS:         延迟 20ms  抖动 2ms   丢包 0%
   biz-internet: 延迟 45ms  抖动 15ms  丢包 1.2%
   LTE:          延迟 80ms  抖动 30ms  丢包 0.5%
        ↓
   策略：语音要求 延迟<150ms, 抖动<30ms, 丢包<1%
        ↓
   ★ 语音 → 走 MPLS ★（唯一满足 SLA 的）
   
   策略：文件备份 无 SLA 要求，选带宽大的
        ↓
   ★ 备份 → 走 biz-internet ★
```

**关键差异**：
```
   传统 WAN：链路【断了】才切换
   SD-WAN： 链路【质量下降】就切换  ← ★ 更早、更智能 ★
```

**SLA 类配置**：
```
vManage > Configuration > Policies > Custom Options > Traffic Policy
  
  SLA Class: VOICE-SLA
    Loss:    1%
    Latency: 150 ms
    Jitter:  30 ms
    
  App-Route Policy:
    Match: DSCP EF / Application: RTP
    Action: SLA Class VOICE-SLA
            Preferred Color: mpls
            Backup: biz-internet
```

**行为**：
- 优先走 MPLS
- 如果 MPLS 的实测质量**不满足 SLA**，自动切到 biz-internet
- 如果**所有链路都不满足**，走最接近的那条（或按策略丢弃）

### 3.7 直接互联网访问（DIA）

**传统方式的问题**：
```
   分支用户访问 Office 365
        ↓
   流量经 MPLS 回到总部
        ↓
   从总部的互联网出口出去
        ↓
   ★ 延迟高（绕远路）+ 浪费昂贵的 MPLS 带宽 ★
```

**SD-WAN 的 DIA**：
```
   分支用户访问 Office 365
        ↓
   ★ 直接从分支的本地宽带出网 ★
        ↓
   延迟低、不占 MPLS 带宽
   
   同时：内部业务流量仍走 IPsec 隧道回总部
```

**配合云安全（SASE）**：
```
   本地出网的流量 → 先经过云安全网关（Umbrella / Zscaler）→ 互联网
                        ↑ 保证安全策略不打折扣
```

**这就是 SASE（Secure Access Service Edge）的雏形**——网络和安全能力都下沉到边缘，以云服务的形式交付。

### 3.8 SD-WAN 的价值总结

| 维度 | 传统 WAN | SD-WAN |
|:--|:--|:--|
| **成本** | MPLS 专线昂贵 | **用宽带替代或补充 MPLS，成本降 30-60%** |
| **带宽利用** | 备份链路闲置 | **所有链路同时使用** |
| **选路** | 静态路由 / 简单主备 | **基于应用和实时质量智能选路** |
| **切换** | 链路**断了**才切 | 链路**质量下降**就切 |
| **部署** | 网工现场配置 | **ZTP 零接触** |
| **管理** | 逐台配置 | **集中策略，一次定义全网生效** |
| **云访问** | 绕回总部 | **本地直接出网（DIA）** |
| **可视化** | 靠 SNMP 猜 | **应用级可视化** |

---

## ④ SD-Access vs SD-WAN 对比

| | **SD-Access** | **SD-WAN** |
|:--|:--|:--|
| **应用场景** | **园区网（Campus）** | **广域网（WAN）** |
| **解决的核心问题** | **策略与位置解耦、微分段** | **成本、智能选路、集中管理** |
| **控制平面** | **LISP** | **OMP** |
| **数据平面** | **VXLAN** | **IPsec** |
| **策略** | **SGT / TrustSec** | 应用感知路由 |
| **控制器** | **DNA Center** | **vManage / vSmart / vBond** |
| **底层** | IS-IS（Underlay） | 任意传输（MPLS/宽带/4G） |
| **典型设备** | Catalyst 9000 | ISR 1000/4000、Catalyst 8000 |

**记忆**：
```
   SD-Access = 园区 = LISP + VXLAN + SGT + DNA Center
   SD-WAN    = 广域 = OMP  + IPsec + AAR + vManage/vSmart/vBond
```

---

## ⑤ 其他 SDN 概念（考纲要求）

### 5.1 SDN 控制器类型

| 类型 | 说明 | 例子 |
|:--|:--|:--|
| **命令式（Imperative）** | 控制器**直接控制**每台设备的转发行为，设备是"哑的" | **OpenFlow** |
| **声明式（Declarative）** | 控制器**下发意图**，设备自己决定怎么实现 | **DNA Center、ACI** |

### 5.2 北向 vs 南向接口（★ 考点）

```
        ┌────────────────────────┐
        │  应用 / 编排系统         │
        └───────────┬────────────┘
                    │ ★ 北向接口（Northbound）★
                    │   REST API / GUI
        ┌───────────▼────────────┐
        │      SDN 控制器          │
        └───────────┬────────────┘
                    │ ★ 南向接口（Southbound）★
                    │   NETCONF / OpenFlow / gRPC / CLI
        ┌───────────▼────────────┐
        │      网络设备            │
        └────────────────────────┘
```

| 接口 | 方向 | 协议 | 用途 |
|:--|:--|:--|:--|
| **北向 (Northbound)** | 控制器 **→ 上层应用** | **REST API** | 让应用和自动化系统调用控制器 |
| **南向 (Southbound)** | 控制器 **→ 网络设备** | **NETCONF / RESTCONF / OpenFlow / gRPC / SNMP / CLI** | 控制器下发配置到设备 |
| 东西向 (East-West) | 控制器 ↔ 控制器 | 各家不同 | 控制器集群同步 |

**记忆**：
- **北向 = 向上 = 面向应用 = REST API**
- **南向 = 向下 = 面向设备 = NETCONF/OpenFlow**

### 5.3 Cisco 的其他 SDN 方案

| 方案 | 场景 | 控制器 |
|:--|:--|:--|
| **SD-Access** | 园区 | DNA Center |
| **SD-WAN** | 广域网 | vManage |
| **ACI** | 数据中心 | APIC |
| **Meraki** | 中小企业/分支 | Meraki Dashboard（云） |
| **NSO** | 多厂商编排 | Network Services Orchestrator |

---

## ⑥ 考点提示 + 自测题

### 考点

- **SD-Access = LISP（控制面）+ VXLAN（数据面）+ SGT（策略面）**
- **SD-Access 的角色**：Control Plane Node / Edge Node / Border Node
- **VN（宏分段，本质是 VRF）vs SGT（微分段）**
- **SD-WAN 四大组件**：vManage / vSmart / vBond / vEdge-cEdge
- **vBond 负责认证和牵引，vSmart 跑 OMP，vManage 管理**
- **TLOC = {系统IP, color, 封装}**
- **应用感知路由（AAR）：质量下降就切，不等断**
- **北向 REST，南向 NETCONF/OpenFlow**

### 自测题

**1.** SD-Access 用到了哪些技术？各自负责什么？

<details><summary>答案</summary>

**SD-Access = LISP + VXLAN + TrustSec/SGT + IS-IS + DNA Center**

| 平面 | 技术 | 职责 |
|:--|:--|:--|
| **控制平面** | **LISP** | 维护 **EID（终端身份）→ RLOC（所在交换机）** 的映射。终端接入或移动时更新映射 |
| **数据平面** | **VXLAN** | 封装转发。**VNI** 区分不同的虚拟网络（VN） |
| **策略平面** | **Cisco TrustSec (SGT)** | 用**安全组标签**做策略，与 IP 完全解耦 |
| **Underlay** | **IS-IS**（默认） | Fabric 内部的三层路由，保证各节点 Loopback 互通 |
| **管理编排** | **DNA Center** | 图形化配置、自动化下发、策略定义、遥测分析 |

**为什么是这个组合（每个技术解决什么问题）**：

**① LISP 解决"移动性"**
```
   传统：IP 既是身份又是位置 → 移动就要换 IP → 连接中断
   LISP：EID（身份，不变）与 RLOC（位置，会变）分离
        → ★ 终端在整个园区任意移动，IP 不变 ★
```

**② VXLAN 解决"跨三层的二层 + 标识符不足"**
```
   VLAN：只有 4094 个，且跨不了三层
   VXLAN：VNI 24 位 = 1600 万个，封装进 UDP 4789 可跨任意三层网络
        → ★ 逻辑网络可以自由延伸，不受物理拓扑限制 ★
```

**③ SGT 解决"策略与地址耦合"**
```
   传统 ACL：permit ip 10.1.10.0 0.0.0.255 host 10.1.30.50
                       ↑ 绑定 IP → 地址一变策略就失效
   SGT：    permit SGT:10 → SGT:100
                  ↑ 绑定身份 → ★ 无论 IP 怎么变，策略永远有效 ★
```

**④ IS-IS 做 Underlay**
纯三层网络 + ECMP，**不用 STP**，所有链路 100% 利用，收敛快。

**⑤ DNA Center 解决"配置分散"**
一处定义策略，自动下发到全网所有设备。

**串联理解（这是 ENCOR 最喜欢考的"技术如何组合"）**：

一个员工从 3 楼走到 8 楼：
```
① 笔记本在 8 楼的 Edge Node 上认证（802.1X）
        ↓
② ISE 返回 Access-Accept，携带 ★ SGT=10（研发部）★
        ↓
③ Edge Node 向 Control Plane Node 更新 ★ LISP 映射 ★
   （"EID 10.1.200.55 现在在 RLOC 8楼交换机 后面"）
        ↓
④ 流量用 ★ VXLAN 封装 ★，VNI 对应 VN:EMPLOYEE
        ↓
⑤ 数据包里携带 ★ SGT=10 ★
        ↓
⑥ 到达目的地，目的端根据 SGT 执行策略
        ↓
★ 结果：IP 不变、VLAN 不变、策略不变、用户无感知 ★
```

**知识点串联**：
```
第 1 章 架构        → SD-Access 的物理拓扑基础
第 2 章 VRF         → Virtual Network (VN) 本质就是 VRF
第 3 章 VXLAN+LISP  → 数据平面 + 控制平面
第 13 章 802.1X/SGT → 策略平面 + 终端认证
第 14 章 API        → DNA Center 的自动化能力
```
</details>

**2.** SD-WAN 的四大组件是什么？各自的作用？

<details><summary>答案</summary>

| 组件 | 平面 | 作用 | 类比 |
|:--|:--|:--|:--|
| **vManage** | **管理平面** | 图形化管理界面、配置模板、监控告警、REST API | **控制台** |
| **vSmart** | **控制平面** | **用 OMP 协议分发路由和策略**，本身**不转发任何数据** | **大脑**（类似 BGP RR） |
| **vBond** | **编排平面** | **认证和牵引**：新设备首先联系它，它验证身份后告知 vSmart/vManage 的地址；还负责 **NAT 穿越发现** | **接待处 / 门卫** |
| **vEdge / cEdge** | **数据平面** | 分支和总部的边缘路由器，**实际转发流量** | **手脚** |

**详细职责**：

**vBond（编排器）**
- **唯一需要公网 IP 的组件**（新设备要能找到它）
- 用**证书**验证新设备的合法性
- 告诉设备去哪里找 vSmart 和 vManage
- **检测 NAT**，帮助 vEdge 之间穿越 NAT 建立隧道
- 部署后就"退居二线"（只在设备上线和重连时用到）

**vSmart（控制器）**
- 跑 **OMP** 协议，与所有 vEdge 建立会话
- 分发三类信息：
  - **OMP Route**：哪个分支有哪些网段
  - **TLOC Route**：每个 vEdge 的每条链路信息
  - **Service Route**：服务链信息
- 下发**策略**（控制策略、数据策略、应用感知路由策略）
- **不转发任何用户数据**（数据平面是 vEdge 之间的直接 IPsec 隧道）
- 类似 BGP 的 Route Reflector

**vManage（管理器）**
- 图形化配置界面
- **配置模板**（Device Template + Feature Template）
- 监控、告警、报表
- **REST API**（可以做自动化）
- 软件升级管理

**vEdge / cEdge（边缘设备）**
- 实际转发流量
- 与其他 vEdge 建立 **IPsec 隧道**
- 执行 vSmart 下发的策略
- 用 **BFD** 持续测量隧道质量

**设备上线流程（ZTP）**：
```
① 开箱上电，接网线
        ↓
② DHCP 获取 IP
        ↓
③ 通过 DNS 或预配置找到 ★ vBond ★
        ↓
④ 与 vBond 建立 DTLS，★ 证书互认 ★
        ↓
⑤ vBond 验证通过 → 返回 vSmart 和 vManage 的地址
        ↓
⑥ 连 vManage → ★ 下载配置模板 ★
        ↓
⑦ 连 vSmart → ★ 建立 OMP 会话，学习路由和策略 ★
        ↓
⑧ 与其他 vEdge ★ 自动建立 IPsec 隧道 ★
        ↓
⑨ 上线转发流量
```

**★ ZTP 的业务价值**：开新分店时，**快递把设备寄过去，店员插上电源和网线就完成部署**。不需要网工出差。

对连锁零售、餐饮这类"频繁开店"的行业，这是巨大的成本节省——从"每开一店飞一个工程师过去"变成"发个快递"。

**控制平面 vs 数据平面的分离**：
```
   控制平面：vEdge ←→ vSmart（DTLS/TLS，星型）
             ↑ 只传路由和策略
   
   数据平面：vEdge ←→ vEdge（IPsec，全互联或按策略）
             ↑ 用户流量直接走，不经过 vSmart
```

**这个分离很重要**：vSmart 挂了，**已建立的隧道继续工作**（数据平面不受影响），只是不能学习新路由和更新策略。
</details>

**3.** SD-Access 的 Virtual Network (VN) 和 SGT 有什么区别？

<details><summary>答案</summary>

| | **Virtual Network (VN)** | **Scalable Group Tag (SGT)** |
|:--|:--|:--|
| 分段粒度 | **宏分段（Macro-segmentation）** | **微分段（Micro-segmentation）** |
| 技术本质 | **VRF**（独立的路由表） | **标签**（策略标识） |
| 隔离程度 | **完全隔离**（路由表都不通） | **同一路由表内的策略控制** |
| 默认行为 | **不同 VN 之间完全不通** | 同 VN 内默认通，靠策略限制 |
| 互访方式 | 必须经过防火墙（Fusion Router） | 修改 SGT 策略矩阵 |
| 数量 | 较少（几个到几十个） | 较多（几十到几百个） |
| 对应的封装 | **VNI** | 数据包里的 **SGT 字段** |

**VN（宏分段）—— 大颗粒隔离**：
```
   VN: EMPLOYEE   → VRF EMPLOYEE   → VNI 4097
   VN: GUEST      → VRF GUEST      → VNI 4098
   VN: IOT        → VRF IOT        → VNI 4099
   VN: CAMERA     → VRF CAMERA     → VNI 4100
   
   ★ 不同 VN 之间路由表完全隔离，物理上不可能互通 ★
```

用于隔离**性质完全不同**的网络：员工网、访客网、IoT 网、摄像头网。

**SGT（微分段）—— 同一 VN 内的细粒度控制**：
```
   VN: EMPLOYEE 内部：
   
   SGT 10 = 研发部
   SGT 20 = 财务部
   SGT 30 = 销售部
   SGT 100 = 财务服务器
   SGT 110 = 代码仓库
   
   策略矩阵（在 ISE 上定义）：
   
   源 \ 目的      SGT100财务服务器  SGT110代码仓库
   SGT10 研发         ❌ 拒绝          ✅ 允许
   SGT20 财务         ✅ 允许          ❌ 拒绝
   SGT30 销售         ❌ 拒绝          ❌ 拒绝
```

**为什么需要两层**：

**只有 VN 不够**：如果把研发、财务、销售分成三个 VN，那它们之间任何互访都要经过防火墙——**但它们都需要访问共享的邮件、OA、文件服务器**，会产生大量的"绕行流量"，防火墙成为瓶颈。

**只有 SGT 不够**：访客网络和员工网络应该**从路由层面就完全隔离**，不应该依赖策略正确性来保证。策略配错了，访客就能访问内网——这个风险太大。

**正确的组合**：
```
   VN 层：性质完全不同的网络 → 路由隔离（安全边界）
        · EMPLOYEE / GUEST / IOT / CAMERA
   
   SGT 层：同一性质内的细分 → 策略控制（灵活性）
        · EMPLOYEE 内部分研发/财务/销售/服务器
```

**★ SGT 最革命性的地方：策略与 IP 完全解耦**

```
   传统 ACL：
   permit ip 10.1.10.0 0.0.0.255 host 10.1.30.50
              ↑ 绑定网段        ↑ 绑定 IP
   
   问题：
   · 员工换工位 → 换 VLAN → 换 IP → ★ 策略失效 ★
   · 网段规划调整 → 所有相关 ACL 都要改
   · 服务器迁移 → 所有引用它的 ACL 都要改
   · 一个策略要在几十台设备上配 ACL
   
   SGT 策略：
   permit SGT:10 → SGT:100
          ↑ 绑定"身份"，与地址无关
   
   优势：
   · 员工无论在哪、IP 是什么，SGT 跟着身份走 → ★ 策略永远有效 ★
   · 策略在 ISE 上集中定义，一处修改全网生效
   · 新增一个部门 = 加一个 SGT + 一行策略
```

**SGT 的分配**：
```
   动态：802.1X 认证通过后，ISE 在 RADIUS Access-Accept 里下发
        Cisco-AVPair: cts:security-group-tag=000a-00
   
   静态：绑定到 IP / VLAN / 端口 / 子网
        cts role-based sgt-map 10.1.30.50 sgt 100
```

**这就是"零信任"的落地形态**：不再基于"你在哪个网段"授权，而是基于"你是谁"授权。详见 [Security 选修第 5 章](../08-Security选修/05-零信任与网络分段.md)。
</details>

**4.** SD-WAN 的应用感知路由（AAR）和传统的浮动静态路由有什么区别？

<details><summary>答案</summary>

| | **传统浮动静态路由 / 主备切换** | **SD-WAN 应用感知路由（AAR）** |
|:--|:--|:--|
| 切换依据 | **链路 down 了**（或 IP SLA 探测失败） | **实时测量的链路质量**（延迟/抖动/丢包） |
| 切换粒度 | **整条链路的所有流量一起切** | **按应用分别选路** |
| 备份链路 | 平时**完全闲置** | **同时使用**，承载适合它的流量 |
| 感知能力 | 只知道"通/不通" | **知道"质量好不好"** |
| 决策位置 | 每台设备本地 | **控制器集中策略 + 设备本地执行** |
| 切换时机 | 断了才切 | ★ **质量下降就切** |

**传统方式的问题**：

```
   主：MPLS 20Mbps
   备：宽带 100Mbps
   
   问题 1：宽带平时完全闲置 —— 花了钱不用
   问题 2：MPLS 没断，但延迟从 20ms 涨到 300ms（拥塞）
          → ★ 不会切换 ★（因为链路还"通"）
          → 语音质量崩了，但网络"没故障"
   问题 3：所有流量一起走 MPLS，备份下载和语音抢带宽
```

**SD-WAN AAR 的做法**：

```
   持续用 ★ BFD ★ 测量每条隧道的质量（默认 1 秒一次）：
   
   ┌──────────────┬────────┬────────┬────────┐
   │ 隧道          │ 延迟    │ 抖动    │ 丢包    │
   ├──────────────┼────────┼────────┼────────┤
   │ MPLS         │ 20ms   │ 2ms    │ 0%     │
   │ biz-internet │ 45ms   │ 15ms   │ 1.2%   │
   │ LTE          │ 80ms   │ 30ms   │ 0.5%   │
   └──────────────┴────────┴────────┴────────┘
        ↓
   策略定义：
   
   SLA Class "VOICE": 延迟<150ms, 抖动<30ms, 丢包<1%
     Match: DSCP EF
     Preferred: mpls
     → ★ 语音走 MPLS ★（唯一全部满足的）
   
   SLA Class "BULK": 无 SLA 要求
     Match: 文件传输/备份
     Preferred: biz-internet
     → ★ 备份走宽带 ★（带宽大，省 MPLS）
   
   默认流量：
     → 负载分担到所有可用链路
```

**★ 最关键的差异：质量下降就切，不等断**

```
   场景：MPLS 因为拥塞，延迟涨到 300ms、丢包 3%
   
   传统方式：链路还"通"（IP SLA 的 ping 能通）
            → ★ 不切换 ★
            → 语音质量崩溃，但监控显示"一切正常"
   
   SD-WAN： BFD 测出延迟 300ms > SLA 的 150ms
            → ★ 语音自动切到 biz-internet ★
            → 用户几乎无感知
            → 同时 vManage 上告警"MPLS 链路 SLA 违约"
```

**这个能力对语音/视频业务的价值极大**——传统网络里"链路没断但质量差"是最难排查的一类问题（用户说卡，你查了半天所有指标都"正常"）。

**配置示例**：
```
vManage > Configuration > Policies:

SLA Class: VOICE-SLA
  Loss:    1 %
  Latency: 150 ms
  Jitter:  30 ms

App-Route Policy: BRANCH-AAR
  Sequence 10:
    Match:  DSCP 46 (EF)
    Action: SLA Class VOICE-SLA
            Preferred Color: mpls
            Backup SLA Preferred Color: biz-internet
  
  Sequence 20:
    Match:  Application: ms-office-365
    Action: SLA Class BUSINESS-SLA
            Preferred Color: biz-internet     ← 直接本地出网（DIA）
  
  Sequence 30:
    Match:  Application: backup / file-transfer
    Action: Preferred Color: biz-internet, lte
```

**当所有链路都不满足 SLA 时的行为**（可配置）：
- 走**最接近 SLA** 的那条
- 或按 `strict` 策略**直接丢弃**（宁可不通，也不要糟糕的体验——某些实时业务这样更好）

**补充能力：FEC 和包复制**
```
   ★ Forward Error Correction (FEC)：
     发送冗余数据，接收端可以恢复丢失的包 → 对抗丢包
   
   ★ Packet Duplication：
     同一个包同时从两条链路发出，接收端去重
     → 只要有一条链路通，业务就不受影响
     → 用于极关键的业务（金融交易、远程手术）
```

这些是传统 WAN 完全做不到的。
</details>

**5.** 北向接口和南向接口分别是什么？各自用什么协议？

<details><summary>答案</summary>

```
        ┌────────────────────────────────┐
        │  应用 / 编排系统 / 运维平台       │
        │  （Python 脚本、ITSM、CMDB）     │
        └───────────────┬────────────────┘
                        │ ★ 北向接口（Northbound）★
                        │   REST API（JSON）
        ┌───────────────▼────────────────┐
        │        SDN 控制器                │
        │  （DNA Center / vManage / APIC） │
        └───────────────┬────────────────┘
                        │ ★ 南向接口（Southbound）★
                        │   NETCONF / RESTCONF / gRPC / OpenFlow / CLI
        ┌───────────────▼────────────────┐
        │         网络设备                 │
        └────────────────────────────────┘
```

| 接口 | 方向 | 谁调用谁 | 主要协议 |
|:--|:--|:--|:--|
| **北向 (Northbound)** | 控制器 **→ 上层** | **上层应用调用控制器** | **REST API**（JSON over HTTPS） |
| **南向 (Southbound)** | 控制器 **→ 设备** | **控制器控制设备** | **NETCONF、RESTCONF、gRPC/gNMI、OpenFlow、SNMP、CLI/SSH** |
| 东西向 (East-West) | 控制器 ↔ 控制器 | 集群同步 | 各家私有 |

**记忆技巧**：
```
   ★ 北向 = 向上 = 面向【应用】= REST API
   ★ 南向 = 向下 = 面向【设备】= NETCONF / OpenFlow
```

**北向接口的实际用途**：
```python
# 通过 DNA Center 的北向 REST API 做自动化
import requests

# 获取 Token
r = requests.post(f"{DNAC}/dna/system/api/v1/auth/token", auth=(user, pwd))
token = r.json()["Token"]

# 获取所有设备
r = requests.get(f"{DNAC}/dna/intent/api/v1/network-device",
                 headers={"X-Auth-Token": token})

# 路径追踪（排障）
r = requests.post(f"{DNAC}/dna/intent/api/v1/flow-analysis",
                  headers={"X-Auth-Token": token},
                  json={"sourceIP": "10.1.10.55", "destIP": "10.1.30.100"})
```

**北向接口让"网络能力"变成"可编程的服务"**：
- ITSM 系统可以在工单审批通过后自动调用 API 开通网络
- CMDB 可以自动同步网络设备清单
- 监控平台可以调用 API 做自动化排障

**南向接口的演进**：

| 阶段 | 协议 | 特点 |
|:--|:--|:--|
| 第一代 | **CLI / SSH** | 非结构化，靠正则解析输出，脆弱 |
| 第二代 | **SNMP** | 结构化但只读为主，写操作有限 |
| 第三代 | **NETCONF / RESTCONF** | ★ **结构化 + YANG 模型 + 事务性** |
| 第四代 | **gRPC / gNMI** | 高性能、双向流、遥测 |
| 特殊 | **OpenFlow** | 直接控制转发表（命令式 SDN） |

**OpenFlow vs NETCONF（考点）**：

| | **OpenFlow** | **NETCONF** |
|:--|:--|:--|
| 控制层次 | **直接控制转发表**（流表项） | **下发配置**，设备自己决定如何实现 |
| SDN 类型 | **命令式（Imperative）** | **声明式（Declarative）** |
| 设备智能 | 设备是"哑的"，全靠控制器 | 设备保留自己的智能 |
| 控制器故障 | **设备可能停止工作** | 设备继续按现有配置工作 |
| 实际采用 | 学术/特定场景 | ★ **企业网主流** |

**为什么企业网选 NETCONF 而不是 OpenFlow**：

OpenFlow 的"完全集中控制"理论上很优雅，但实践中：
- **控制器成为绝对的单点**（挂了全网瘫痪）
- 转发表下发的延迟影响新流的建立
- 设备已有的成熟功能（路由协议、STP）被浪费

**声明式（NETCONF/DNA Center/ACI）的折中更实际**：控制器下发"意图"，设备自己用成熟的协议实现。控制器挂了，网络继续按现有配置运行。

**Cisco 各方案的接口**：

| 方案 | 北向 | 南向 |
|:--|:--|:--|
| **DNA Center** | REST API | NETCONF / CLI / SNMP |
| **vManage** | REST API | NETCONF / OMP |
| **ACI APIC** | REST API | OpFlex |
| **Meraki** | REST API（Dashboard API） | 私有（云） |

**共同点：北向全都是 REST + JSON。** 学会一个，其他的都是查文档的事。
</details>

---

## 🎓 Stage 2 结业

至此 ENCOR 350-401 全部 15 章完成。

**回到 [Stage 2 README](README.md) 完成 13 道过关自检题。** 全部答对后：
1. 做一遍 [`04-实验手册`](../04-实验手册/) 里的 Lab 01–06
2. 刷 [`06-备考与考试`](../06-备考与考试/) 里的易错点清单
3. **约考 ENCOR 350-401**

**然后进入 [Stage 3 · ENARSI 300-410](../03-ENARSI-300-410/README.md)**，把路由和排障挖到专家深度。

---

**上一章** ← [14 网络自动化](14-网络自动化-Python-REST-NETCONF-Ansible.md) ｜ **下一阶段** → [Stage 3 · ENARSI 300-410](../03-ENARSI-300-410/README.md)
