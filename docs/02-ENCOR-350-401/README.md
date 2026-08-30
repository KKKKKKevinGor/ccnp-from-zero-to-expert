# Stage 2 · ENCOR 350-401

> **目标**：覆盖 CCNP 核心考试（Implementing and Operating Cisco Enterprise Network Core Technologies）的全部知识点，并且每一项都能配、能验、能排障。

## 考试概览

| 项目 | 说明 |
|:--|:--|
| 考试代码 | **350-401 ENCOR** |
| 时长 | 120 分钟 |
| 题量 | 90–110 题 |
| 题型 | 单选、多选、拖拽、模拟题（Sim）、实验题（Lab） |
| 通过后 | 获得 **CCNP Enterprise 核心资格** + **Cisco Certified Specialist – Enterprise Core** |
| 有效期 | 3 年 |

**官方考纲权重**（作为学习时间分配参考）：

| 域 | 权重 | 对应本篇章节 |
|:--|:--|:--|
| 1.0 架构 Architecture | **15%** | 01, 15 |
| 2.0 虚拟化 Virtualization | **10%** | 02, 03 |
| 3.0 基础设施 Infrastructure | **30%** ← 最重 | 04, 05, 06, 07, 08, 09, 11, 12 |
| 4.0 网络保障 Network Assurance | **10%** | 12 |
| 5.0 安全 Security | **20%** | 13 |
| 6.0 自动化 Automation | **15%** | 14 |

> **注意**：3.0 基础设施占 30%，其中**路由协议（OSPF、EIGRP、BGP）是绝对核心**。但别忽略自动化（15%）——很多传统网工在这块丢分最多。

## 本篇的组织逻辑

我没有严格按考纲顺序排章节，而是按**知识依赖关系**排：

```
   ① 架构（宏观视角，知道自己在造什么）
        ↓
   ② 虚拟化（VRF → GRE/IPsec → VXLAN/LISP）
        ↓ 这是一条"抽象层层叠加"的主线：
        ↓ VLAN 抽象二层 → VRF 抽象三层 → 隧道抽象链路 → VXLAN 抽象跨三层的二层
        ↓
   ③ 交换与高可用（STP 进阶、FHRP）
        ↓
   ④ 路由协议（EIGRP → OSPF → BGP）  ← 分量最重
        ↓
   ⑤ 增值服务（组播、QoS、无线、IP 服务）
        ↓
   ⑥ 安全与自动化
        ↓
   ⑦ SD-Access / SD-WAN（前面所有技术的集大成）
```

## 章节列表

| # | 章节 | 权重 | 难度 |
|:--|:--|:--|:--|
| 01 | [企业网络架构与设计](01-企业网络架构与设计.md) | ★★★ | ★★ |
| 02 | [网络虚拟化：VRF / GRE / IPsec](02-网络虚拟化-VRF-GRE-IPsec.md) | ★★★★ | ★★★ |
| 03 | [Overlay：VXLAN 与 LISP](03-Overlay-VXLAN与LISP.md) | ★★★ | ★★★★ |
| 04 | [交换进阶：STP 进阶与排障](04-交换进阶-STP进阶与排障.md) | ★★★★ | ★★★ |
| 05 | [一二层排障与高可用：HSRP/VRRP/GLBP](05-一层二层排障与高可用-HSRP-VRRP.md) | ★★★★ | ★★★ |
| 06 | [EIGRP](06-EIGRP.md) | ★★★★ | ★★★ |
| 07 | [OSPF 进阶](07-OSPF进阶.md) | ★★★★★ | ★★★★ |
| 08 | [BGP](08-BGP.md) | ★★★★★ | ★★★★★ |
| 09 | [组播 Multicast](09-组播Multicast.md) | ★★ | ★★★ |
| 10 | [QoS 服务质量](10-QoS服务质量.md) | ★★★ | ★★★ |
| 11 | [无线架构与漫游](11-无线架构与漫游.md) | ★★★ | ★★★ |
| 12 | [IP 服务：NTP/NAT/SLA/NetFlow/SNMP/Syslog](12-IP服务-NTP-NAT-SLA-NetFlow-SNMP-Syslog.md) | ★★★ | ★★ |
| 13 | [网络安全：AAA/802.1X/控制平面保护](13-网络安全-AAA-802.1X-控制平面保护.md) | ★★★★ | ★★★ |
| 14 | [网络自动化：Python/REST/NETCONF/Ansible](14-网络自动化-Python-REST-NETCONF-Ansible.md) | ★★★★ | ★★★ |
| 15 | [SD-Access 与 SD-WAN](15-SD-Access与SD-WAN.md) | ★★★ | ★★★ |

## 学习建议

**时间分配（10 周）**：

| 周 | 章节 | 重点 |
|:--|:--|:--|
| W1 | 01, 02 | 建立宏观框架，搞懂 VRF-Lite |
| W2 | 03, 04 | VXLAN 概念 + MST 实操 |
| W3 | 05, 06 | HSRP 实验 + EIGRP DUAL 手算 |
| W4–W5 | **07 OSPF 进阶** | **LSA 类型是重中之重，值得花两周** |
| W6–W7 | **08 BGP** | **选路顺序 + 属性操控，同样值得两周** |
| W8 | 09, 10, 11 | 组播、QoS、无线 |
| W9 | 12, 13 | IP 服务与安全 |
| W10 | 14, 15 | 自动化与 SDN，然后总复习 |

**三条硬性要求**：

1. **每章的实验必须自己敲**。ENCOR 有模拟题和实验题，只看不练必挂。
2. **LSA 类型表和 BGP 选路顺序必须能默写**。这两个是全书最高频的考点。
3. **自动化章节不要跳过**。它占 15%，而且多数传统网工在这里得分最低——这恰恰是拉开差距的地方。

## 过关自检

学完本篇，闭卷回答，答对 8 题以上再考虑约考：

1. 三层架构（接入/汇聚/核心）和 Spine-Leaf 各自适用什么场景？为什么数据中心要用后者？
2. VRF-Lite 和 MPLS L3VPN 的区别是什么？
3. GRE over IPsec 和 IPsec over GRE 有什么不同？为什么通常用前者？
4. VXLAN 的 VNI 是多少位？它解决了 VLAN 的什么问题？
5. MST 中，哪三个参数必须在同一个 Region 内完全一致？
6. HSRP、VRRP、GLBP 三者最大的区别是什么？
7. EIGRP 的 FD 和 RD 分别是什么？可行性条件（FC）怎么判断？
8. OSPF 的 1/2/3/4/5/7 类 LSA 分别由谁产生、传播到哪里、作用是什么？
9. BGP 选路的前 6 步是什么？
10. `ip tcp adjust-mss` 解决什么问题？它和 `ip mtu` 有什么区别？
11. CoPP 保护的是什么？为什么需要它？
12. RESTCONF 和 NETCONF 的区别？分别用什么端口？
13. SD-WAN 的四大组件是什么？各自的作用？

答案分散在各章。
