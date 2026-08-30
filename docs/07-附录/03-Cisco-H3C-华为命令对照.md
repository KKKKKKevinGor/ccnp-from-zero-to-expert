# 03 · Cisco / H3C / 华为 命令对照表

> **为什么需要这份表**：
> 国内大量企业用 H3C 和华为设备，但 CCNP 考的是 Cisco。
> **你上班用国产设备，考试用 Cisco，两套语法在脑子里打架。**
>
> 这份表帮你建立映射，把已有的工作经验直接迁移到 Cisco 体系上。

---

## ① 三大核心差异（先记住这三条）

```
   ★ 1. 查看命令：Cisco 用 show，H3C/华为用 display ★
   ★ 2. 撤销命令：Cisco 用 no，H3C/华为用 undo ★
   ★ 3. 模式层级：Cisco 有【特权模式】，H3C/华为没有 ★
```

**模式层级对比**：

```
   ★ Cisco ★                        ★ H3C / 华为 ★
   
   Router>          用户模式          <Router>        用户视图
      │ enable                            │ system-view
   Router#          特权模式          [Router]        系统视图
      │ configure terminal                │ interface GigabitEthernet1/0/1
   Router(config)#  全局配置          [Router-GigabitEthernet1/0/1]
      │ interface Gi0/0
   Router(config-if)# 接口配置
   
   ★ H3C/华为没有"特权模式"这一层 ★
   权限由【用户级别 0-15】控制，不需要二次提权
```

---

## ② 基础操作

| 目的 | **Cisco** | **H3C** | **华为** |
|:--|:--|:--|:--|
| 进配置模式 | `enable` → `configure terminal` | `system-view` | `system-view` |
| 退出一层 | `exit` | `quit` | `quit` |
| 直接回顶层 | `end` / `Ctrl+Z` | `return` / `Ctrl+Z` | `return` / `Ctrl+Z` |
| **保存配置** | ★ `write memory` / `copy run start` ★ | ★ `save` ★ | ★ `save` ★ |
| 查看当前配置 | `show running-config` | `display current-configuration` | `display current-configuration` |
| 查看启动配置 | `show startup-config` | `display saved-configuration` | `display saved-configuration` |
| **撤销配置** | ★ `no <cmd>` ★ | ★ `undo <cmd>` ★ | ★ `undo <cmd>` ★ |
| 设主机名 | `hostname R1` | `sysname R1` | `sysname R1` |
| 版本信息 | `show version` | `display version` | `display version` |
| 重启 | `reload` | `reboot` | `reboot` |
| **不分页** | `terminal length 0` | `screen-length disable` | `screen-length 0 temporary` |
| 输出过滤 | `\| include xxx` | `\| include xxx` | `\| include xxx` |
| 清空配置 | `write erase` + `reload` | `reset saved-configuration` + `reboot` | `reset saved-configuration` + `reboot` |
| 关闭域名解析 | `no ip domain-lookup` | `undo dns resolve` | `undo dns resolve` |
| 日志不打断输入 | `logging synchronous` | 默认 | `undo info-center enable`（或调级别） |

---

## ③ 接口配置

| 目的 | **Cisco** | **H3C** | **华为** |
|:--|:--|:--|:--|
| 进接口 | `interface GigabitEthernet0/0` | `interface GigabitEthernet 1/0/1` | `interface GigabitEthernet 0/0/1` |
| **配 IP** | ★ `ip address 1.1.1.1 255.255.255.0` ★ | ★ `ip address 1.1.1.1 24` ★ | ★ `ip address 1.1.1.1 24` ★ |
| 副地址 | `ip address ... secondary` | `ip address ... sub` | `ip address ... sub` |
| **开启接口** | ★ `no shutdown` ★ | ★ `undo shutdown` ★ | ★ `undo shutdown` ★ |
| 描述 | `description ###...###` | `description ###...###` | `description ###...###` |
| 接口概要 | `show ip interface brief` | `display ip interface brief` | `display ip interface brief` |
| 接口详情 | `show interfaces Gi0/0` | `display interface GigabitEthernet 1/0/1` | `display interface GigabitEthernet 0/0/1` |
| 接口状态 | `show interfaces status` | `display interface brief` | `display interface brief` |
| **清计数器** | `clear counters` | `reset counters interface` | `reset counters interface` |
| 批量配置 | `interface range Gi0/1-4` | `interface range GE1/0/1 to GE1/0/4` | `port-group group-member GE0/0/1 to GE0/0/4` |
| 恢复默认 | `default interface Gi0/1` | — | — |
| MTU | `mtu 9000` / `ip mtu 1400` | `mtu 9000` | `mtu 9000` |
| **TCP MSS** | ★ `ip tcp adjust-mss 1360` ★ | `tcp mss 1360` | `tcp adjust-mss 1360` |
| 光模块诊断 | `show interfaces X transceiver detail` | `display transceiver diagnosis interface X` | `display transceiver diagnosis interface X` |

> **★ 最直观的差异**：H3C/华为支持直接写前缀长度（`ip address 1.1.1.1 24`），Cisco 必须写完整点分掩码。

---

## ④ VLAN 与二层

| 目的 | **Cisco** | **H3C** | **华为** |
|:--|:--|:--|:--|
| 创建 VLAN | `vlan 10` | `vlan 10` | `vlan 10` |
| 批量创建 | `vlan 10,20,30` | `vlan batch 10 20 30` | `vlan batch 10 20 30` |
| 命名 | `name FINANCE` | `name FINANCE` | `name FINANCE` |
| **设为 Access** | ★ `switchport mode access` ★ | ★ `port link-type access` ★ | ★ `port link-type access` ★ |
| **Access 划 VLAN** | ★ `switchport access vlan 10` ★ | ★ `port access vlan 10` ★ | ★ `port default vlan 10` ★ |
| **设为 Trunk** | ★ `switchport mode trunk` ★ | ★ `port link-type trunk` ★ | ★ `port link-type trunk` ★ |
| **Trunk 放行** | ★ `switchport trunk allowed vlan 10,20` ★ | ★ `port trunk permit vlan 10 20` ★ | ★ `port trunk allow-pass vlan 10 20` ★ |
| Trunk 追加 | `switchport trunk allowed vlan **add** 30` | `port trunk permit vlan 30`（追加） | `port trunk allow-pass vlan 30`（追加） |
| **Native VLAN** | ★ `switchport trunk native vlan 999` ★ | ★ `port trunk pvid vlan 999` ★ | ★ `port trunk pvid vlan 999` ★ |
| 关闭 DTP | `switchport nonegotiate` | 无 DTP | 无 DTP |
| Hybrid 口 | 无 | `port link-type hybrid` | `port link-type hybrid` |
| Voice VLAN | `switchport voice vlan 100` | `voice-vlan 100 enable` | `voice-vlan 100 enable` |
| 查 VLAN | `show vlan brief` | `display vlan brief` | `display vlan` |
| 查 Trunk | `show interfaces trunk` | `display port trunk` | `display port vlan` |
| **MAC 表** | `show mac address-table` | `display mac-address` | `display mac-address` |
| MAC 老化 | `mac address-table aging-time 300` | `mac-address timer aging 300` | `mac-address aging-time 300` |
| 静态 MAC | `mac address-table static ...` | `mac-address static ...` | `mac-address static ...` |
| 风暴抑制 | `storm-control broadcast level 5` | `broadcast-suppression 5` | `broadcast-suppression packets 100` |

> ⚠️ **★ 重要差异**：
> - **Cisco 的 `switchport trunk allowed vlan X` 是【覆盖】**（危险！要用 `add`）
> - **H3C/华为的 `port trunk permit/allow-pass vlan X` 是【追加】**（更安全）
>
> **这个差异导致从国产设备转 Cisco 的人容易出运维事故。**

**术语对照**：**Cisco 的 Native VLAN ↔ H3C/华为的 PVID**（概念完全一样）。

---

## ⑤ 三层交换与 SVI

| 目的 | **Cisco** | **H3C** | **华为** |
|:--|:--|:--|:--|
| **创建 SVI** | ★ `interface Vlan10` ★ | ★ `interface Vlan-interface 10` ★ | ★ `interface Vlanif 10` ★ |
| **开启三层转发** | ★★ `ip routing` ★★ | ★ **默认开启** ★ | ★ **默认开启** ★ |
| 路由口 | `no switchport` | `port link-mode route` | `undo portswitch` |
| 查 SVI | `show ip interface brief \| include Vlan` | `display ip interface brief` | `display ip interface brief` |

> ⚠️ **★★ 最容易踩的坑 ★★**
>
> **Cisco 的三层交换机默认【不开启】路由转发**，必须配 `ip routing`。
> **H3C/华为默认就开着。**
>
> **症状**：SVI 配好了、状态 up/up、路由表有直连路由，但**跨 VLAN 完全不通**。
> 从国产设备转 Cisco 的人踩这个坑的概率极高。

---

## ⑥ STP

| 目的 | **Cisco** | **H3C** | **华为** |
|:--|:--|:--|:--|
| 开启 STP | 默认开启 | `stp global enable` | `stp enable` |
| **模式** | `spanning-tree mode rapid-pvst` / `mst` | `stp mode rstp` / `mstp` | `stp mode rstp` / `mstp` |
| **默认模式** | ★ **Rapid-PVST+** ★ | ★ **MSTP** ★ | ★ **MSTP** ★ |
| 设根桥 | `spanning-tree vlan 10 root primary` | `stp instance 0 root primary` | `stp instance 0 root primary` |
| 设优先级 | `spanning-tree vlan 10 priority 4096` | `stp instance 0 priority 4096` | `stp instance 0 priority 4096` |
| MST 配置模式 | `spanning-tree mst configuration` | `stp region-configuration` | `stp region-configuration` |
| MST 域名 | `name REGION-A` | `region-name REGION-A` | `region-name REGION-A` |
| MST 修订号 | `revision 1` | `revision-level 1` | `revision-level 1` |
| MST 映射 | `instance 1 vlan 10,20` | `instance 1 vlan 10 20` | `instance 1 vlan 10 20` |
| **★ 激活 MST 配置 ★** | ★ 退出即生效 ★ | ★★ **`active region-configuration`** ★★ | ★★ **`active region-configuration`** ★★ |
| 边缘端口 | `spanning-tree portfast` | `stp edged-port` | `stp edged-port enable` |
| **BPDU 保护** | `spanning-tree bpduguard enable` | `stp bpdu-protection` | `stp bpdu-protection` |
| 根保护 | `spanning-tree guard root` | `stp root-protection` | `stp root-protection` |
| 环路保护 | `spanning-tree guard loop` | `stp loop-protection` | `stp loop-protection` |
| 改开销 | `spanning-tree cost 10` | `stp cost 10` | `stp cost 10` |
| 查看 | `show spanning-tree` | `display stp brief` | `display stp brief` |
| 查 Region | `show spanning-tree mst configuration` | `display stp region-configuration` | `display stp region-configuration` |

> ⚠️ **★★ 两个致命差异 ★★**
>
> **① H3C/华为改完 MST Region 必须敲 `active region-configuration`**
> 不敲的话配置写进去了但**不生效**，`display stp region-configuration` 里的 **Oper** 部分还是旧的。
> **这是国内混合组网的头号坑。**
>
> **② Cisco 的 PVST+ 与 H3C/华为的 MSTP 不互通**
> PVST+ 是 Cisco 私有的。混合组网**必须两边都改成 MST**，且 Region 三要素（name / revision / VLAN映射）完全一致。

---

## ⑦ 链路聚合

| 目的 | **Cisco** | **H3C** | **华为** |
|:--|:--|:--|:--|
| **技术名称** | ★ EtherChannel / Port-channel ★ | ★ 链路聚合 ★ | ★ Eth-Trunk ★ |
| 创建聚合口（二层） | `interface Port-channel1` | `interface Bridge-Aggregation 1` | `interface Eth-Trunk 1` |
| 创建聚合口（三层） | `interface Port-channel1` + `no switchport` | `interface Route-Aggregation 1` | `interface Eth-Trunk 1` + `undo portswitch` |
| 成员加入 | `channel-group 1 mode active` | `port link-aggregation group 1` | `eth-trunk 1` |
| **启用 LACP** | `mode active` / `passive` | ★ `link-aggregation mode dynamic` ★ | ★ `mode lacp-static` ★ |
| **默认模式** | 需显式指定 | ★ **静态（相当于 Cisco 的 on）** ★ | ★ **手工模式** ★ |
| 负载分担 | `port-channel load-balance src-dst-ip` | `link-aggregation load-sharing mode source-ip destination-ip` | `load-balance src-dst-ip` |
| 查看 | `show etherchannel summary` | `display link-aggregation verbose` | `display eth-trunk` |
| 查 LACP | `show lacp neighbor` | `display link-aggregation verbose` | `display eth-trunk` |

**状态标志对照**：

| Cisco | H3C/华为 | 含义 |
|:--|:--|:--|
| **`(P)`** bundled | **`S`** Selected | ✅ 已捆绑 |
| **`(I)`** stand-alone | **`U`** Unselected | ❌ 参数不一致 |
| **`(s)`** suspended | `U` | ❌ LACP 协商失败 |

> ⚠️ **★ 重要差异**：**H3C/华为的聚合默认是【静态模式】**（相当于 Cisco 的 `mode on`），必须显式配 `link-aggregation mode dynamic`（H3C）或 `mode lacp-static`（华为）才启用 LACP。
>
> 跟 Cisco 对接时忘了这一步 → Cisco 侧显示 `(s)` suspended。

---

## ⑧ 静态路由

| 目的 | **Cisco** | **H3C** | **华为** |
|:--|:--|:--|:--|
| **静态路由** | ★ `ip route 192.168.2.0 255.255.255.0 10.0.0.2` ★ | ★ `ip route-static 192.168.2.0 24 10.0.0.2` ★ | ★ `ip route-static 192.168.2.0 24 10.0.0.2` ★ |
| 默认路由 | `ip route 0.0.0.0 0.0.0.0 10.0.0.2` | `ip route-static 0.0.0.0 0 10.0.0.2` | `ip route-static 0.0.0.0 0 10.0.0.2` |
| **带优先级** | `ip route ... 10.0.0.2 **200**` | `ip route-static ... **preference 200**` | `ip route-static ... **preference 200**` |
| 黑洞路由 | `ip route ... Null0` | `ip route-static ... NULL 0` | `ip route-static ... NULL 0` |
| 查路由表 | `show ip route` | `display ip routing-table` | `display ip routing-table` |
| 查特定路由 | `show ip route 1.1.1.1` | `display ip routing-table 1.1.1.1` | `display ip routing-table 1.1.1.1` |
| 查协议路由 | `show ip route ospf` | `display ip routing-table protocol ospf` | `display ip routing-table protocol ospf` |

**★ AD / Preference 默认值对照（★ 重要差异）**：

| 路由来源 | **Cisco AD** | **H3C Preference** | **华为 Preference** |
|:--|:--|:--|:--|
| 直连 | **0** | 0 | 0 |
| **静态** | ★ **1** ★ | ★ **60** ★ | ★ **60** ★ |
| **OSPF** | ★ **110** ★ | ★ **10** ★ | ★ **10** ★ |
| OSPF 外部 | 110 | 150 | 150 |
| IS-IS | 115 | 15 | 15 |
| RIP | 120 | 100 | 100 |
| **eBGP** | ★ **20** ★ | ★ **255** ★ | ★ **255** ★ |
| **iBGP** | ★ **200** ★ | ★ **255** ★ | ★ **255** ★ |

> ⚠️ **★★ 这个差异极其重要 ★★**
>
> **Cisco：静态(1) < OSPF(110)** → 静态路由更优先
> **H3C/华为：OSPF(10) < 静态(60)** → **OSPF 更优先！**
>
> **混合组网时，同样的配置在两边会有完全不同的选路结果。**

---

## ⑨ OSPF

| 目的 | **Cisco** | **H3C** | **华为** |
|:--|:--|:--|:--|
| 启动进程 | `router ospf 1` | `ospf 1` | `ospf 1` |
| Router ID | `router-id 1.1.1.1` | `ospf 1 router-id 1.1.1.1` | `ospf 1 router-id 1.1.1.1` |
| **通告网段** | ★ `network 192.168.1.0 0.0.0.255 **area 0**` ★ | ★ `area 0` → `network 192.168.1.0 0.0.0.255` ★ | ★ `area 0` → `network 192.168.1.0 0.0.0.255` ★ |
| 接口启用 | `ip ospf 1 area 0` | `ospf enable 1 area 0` | `ospf enable 1 area 0` |
| **静默接口** | ★ `passive-interface Gi0/0` ★ | ★ `silent-interface Gi1/0/1` ★ | ★ `silent-interface GE0/0/1` ★ |
| 接口 cost | `ip ospf cost 50` | `ospf cost 50` | `ospf cost 50` |
| DR 优先级 | `ip ospf priority 255` | `ospf dr-priority 255` | `ospf dr-priority 255` |
| 网络类型 | `ip ospf network point-to-point` | `ospf network-type p2p` | `ospf network-type p2p` |
| **参考带宽** | `auto-cost reference-bandwidth 100000` | `bandwidth-reference 100000` | `bandwidth-reference 100000` |
| Stub 区域 | `area 1 stub` | `stub`（area 视图下） | `stub` |
| Totally Stub | `area 1 stub no-summary` | `stub no-summary` | `stub no-summary` |
| NSSA | `area 1 nssa` | `nssa` | `nssa` |
| **区域间汇总** | `area 1 range X Y` | `abr-summary X Y`（area 视图） | `abr-summary X Y` |
| **外部汇总** | `summary-address X Y` | `asbr-summary X Y` | `asbr-summary X Y` |
| 虚链路 | `area 1 virtual-link 2.2.2.2` | `vlink-peer 2.2.2.2` | `vlink-peer 2.2.2.2` |
| 默认路由 | `default-information originate` | `default-route-advertise` | `default-route-advertise` |
| 认证 | `ip ospf message-digest-key 1 md5 KEY` | `ospf authentication-mode md5 1 plain KEY` | `ospf authentication-mode md5 1 plain KEY` |
| 查邻居 | `show ip ospf neighbor` | `display ospf peer` | `display ospf peer` |
| 查 LSDB | `show ip ospf database` | `display ospf lsdb` | `display ospf lsdb` |
| 查接口 | `show ip ospf interface` | `display ospf interface` | `display ospf interface` |

> **★ 结构差异**：Cisco 的 `network` 命令直接带 `area` 参数；**H3C/华为要先进 `area` 视图，再写 `network`**。

---

## ⑩ BGP

| 目的 | **Cisco** | **H3C** | **华为** |
|:--|:--|:--|:--|
| 启动 | `router bgp 65001` | `bgp 65001` | `bgp 65001` |
| Router ID | `bgp router-id 1.1.1.1` | `router-id 1.1.1.1` | `router-id 1.1.1.1` |
| **邻居** | ★ `neighbor 1.1.1.1 remote-as 100` ★ | ★ `peer 1.1.1.1 as-number 100` ★ | ★ `peer 1.1.1.1 as-number 100` ★ |
| **更新源** | ★ `neighbor X update-source Lo0` ★ | ★ `peer X connect-interface Lo0` ★ | ★ `peer X connect-interface Lo0` ★ |
| **next-hop-self** | ★ `neighbor X next-hop-self` ★ | ★ `peer X next-hop-local` ★ | ★ `peer X next-hop-local` ★ |
| eBGP 多跳 | `neighbor X ebgp-multihop 2` | `peer X ebgp-max-hop 2` | `peer X ebgp-max-hop 2` |
| RR 客户端 | `neighbor X route-reflector-client` | `peer X reflect-client` | `peer X reflect-client` |
| **通告网络** | ★ `network 192.168.0.0 **mask** 255.255.252.0` ★ | ★ `network 192.168.0.0 22` ★ | ★ `network 192.168.0.0 22` ★ |
| 聚合 | `aggregate-address X Y summary-only` | `aggregate X Y detail-suppressed` | `aggregate X Y detail-suppressed` |
| **应用策略** | ★ `neighbor X route-map RM **in/out**` ★ | ★ `peer X route-policy RP **import/export**` ★ | ★ `peer X route-policy RP **import/export**` ★ |
| 前缀过滤 | `neighbor X prefix-list PL in` | `peer X ip-prefix PL import` | `peer X ip-prefix PL import` |
| 认证 | `neighbor X password KEY` | `peer X password simple KEY` | `peer X password simple KEY` |
| 最大前缀 | `neighbor X maximum-prefix 1000` | `peer X route-limit 1000` | `peer X route-limit 1000` |
| 查邻居 | `show ip bgp summary` | `display bgp peer` | `display bgp peer` |
| 查 BGP 表 | `show ip bgp` | `display bgp routing-table` | `display bgp routing-table` |
| **软重置** | ★ `clear ip bgp X soft in` ★ | ★ `refresh bgp X import` ★ | ★ `refresh bgp X import` ★ |

**★ 术语对照**：

| Cisco | H3C / 华为 |
|:--|:--|
| **route-map** | ★ **route-policy** ★ |
| **in / out** | ★ **import / export** ★ |
| **next-hop-self** | ★ **next-hop-local** ★ |
| **neighbor** | ★ **peer** ★ |
| prefix-list | ip-prefix |
| clear ... soft | refresh |

---

## ⑪ ACL

| 目的 | **Cisco** | **H3C** | **华为** |
|:--|:--|:--|:--|
| **标准/基本 ACL** | `access-list **10** permit ...` | `acl **basic 2000**` → `rule permit source ...` | `acl **2000**` → `rule permit source ...` |
| **扩展/高级 ACL** | `access-list **100** permit tcp ...` | `acl **advanced 3000**` → `rule permit tcp ...` | `acl **3000**` → `rule permit tcp ...` |
| 命名 ACL | `ip access-list extended NAME` | `acl advanced name NAME` | `acl name NAME advance` |
| **应用到接口** | ★ `ip access-group 100 in` ★ | ★ `packet-filter 3000 inbound` ★ | ★ `traffic-filter inbound acl 3000` ★ |
| **保护 VTY** | ★ `access-class 10 in` ★ | `user-interface vty 0 4` → `acl 2000 inbound` | `user-interface vty 0 4` → `acl 2000 inbound` |
| 查看 | `show access-lists` | `display acl all` | `display acl all` |

**★ 编号范围对照**：

| 类型 | **Cisco** | **H3C / 华为** |
|:--|:--|:--|
| 标准 / 基本 | ★ **1–99, 1300–1999** ★ | ★ **2000–2999** ★ |
| 扩展 / 高级 | ★ **100–199, 2000–2699** ★ | ★ **3000–3999** ★ |
| 二层 | 700–799 | ★ **4000–4999** ★ |
| 用户自定义 | — | 5000–5999 |

> ⚠️ **★ 编号范围完全不同，容易混。** Cisco 的 2000 是扩展 ACL，H3C/华为的 2000 是基本 ACL。

---

## ⑫ NAT

| 目的 | **Cisco** | **H3C** | **华为** |
|:--|:--|:--|:--|
| **标记内网口** | ★★ `ip nat inside` ★★ | ★ **不需要** ★ | ★ **不需要** ★ |
| **标记外网口** | ★★ `ip nat outside` ★★ | ★ **不需要** ★ | ★ **不需要** ★ |
| **PAT（出接口）** | `ip nat inside source list 1 interface Gi0/1 overload` | `nat outbound 2000`（接口下） | `nat outbound 2000`（接口下） |
| PAT（地址池） | `ip nat pool P ...` + `... pool P overload` | `nat address-group 1` + `nat outbound 2000 address-group 1` | `nat address-group 1` + `nat outbound 2000 address-group 1` |
| 静态 NAT | `ip nat inside source static <私> <公>` | `nat static outbound <私> <公>` | `nat static global <公> inside <私>` |
| **端口转发** | `ip nat inside source static tcp <私> 80 <公> 80` | ★ `nat server protocol tcp global <公> 80 inside <私> 80` ★ | ★ `nat server protocol tcp global <公> 80 inside <私> 80` ★ |
| 查转换表 | `show ip nat translations` | `display nat session` | `display nat session all` |
| 查统计 | `show ip nat statistics` | `display nat all` | `display nat outbound` |

> ⚠️ **★★ 架构差异 ★★**
>
> **Cisco 必须显式标记 `ip nat inside/outside`**，方向是显式的。
> **H3C/华为不需要**，NAT 直接配在出接口上，方向是隐含的。
>
> **从国产设备转 Cisco 的人，最容易忘掉标记接口** → NAT 完全不工作（且配置看起来完全正确）。

**★ 术语**：Cisco 的"端口转发"在 H3C/华为叫 **`nat server`**（NAT 服务器映射），这个名字其实更直观。

---

## ⑬ DHCP

| 目的 | **Cisco** | **H3C** | **华为** |
|:--|:--|:--|:--|
| 全局开启 | 默认开启 | `dhcp enable` | `dhcp enable` |
| 地址池 | `ip dhcp pool NAME` | `dhcp server ip-pool NAME` | `ip pool NAME` |
| 网段 | `network 192.168.10.0 255.255.255.0` | `network 192.168.10.0 mask 255.255.255.0` | `network 192.168.10.0 mask 24` |
| 网关 | `default-router 192.168.10.1` | `gateway-list 192.168.10.1` | `gateway-list 192.168.10.1` |
| DNS | `dns-server 10.0.0.53` | `dns-list 10.0.0.53` | `dns-list 10.0.0.53` |
| 排除地址 | `ip dhcp excluded-address X Y` | `forbidden-ip X Y` | `excluded-ip-address X Y` |
| 租期 | `lease 0 8 0` | `expired day 0 hour 8` | `lease day 0 hour 8` |
| **DHCP 中继** | ★ `ip helper-address 10.0.0.53` ★ | ★ `dhcp relay server-address 10.0.0.53` ★ | ★ `dhcp relay server-ip 10.0.0.53` ★ |
| **Snooping** | `ip dhcp snooping` | `dhcp snooping enable` | `dhcp snooping enable` |
| **信任口** | ★ `ip dhcp snooping trust` ★ | ★ `dhcp snooping trust` ★ | ★ `dhcp snooping trusted` ★ |
| 查绑定 | `show ip dhcp binding` | `display dhcp server ip-in-use` | `display ip pool` |

---

## ⑭ VRRP（★ HSRP 是 Cisco 私有的）

| 目的 | **Cisco (VRRP)** | **H3C** | **华为** |
|:--|:--|:--|:--|
| 虚拟 IP | `vrrp 10 ip 192.168.10.1` | `vrrp vrid 10 virtual-ip 192.168.10.1` | `vrrp vrid 10 virtual-ip 192.168.10.1` |
| 优先级 | `vrrp 10 priority 110` | `vrrp vrid 10 priority 110` | `vrrp vrid 10 priority 110` |
| 抢占 | 默认开启 | 默认开启 | 默认开启 |
| 抢占延迟 | `vrrp 10 preempt delay minimum 60` | `vrrp vrid 10 preempt-mode delay 60` | `vrrp vrid 10 preempt-mode timer delay 60` |
| **跟踪** | ★ `vrrp 10 track 1 decrement 20` ★ | ★ `vrrp vrid 10 track interface X **reduced** 20` ★ | ★ `vrrp vrid 10 track interface X **reduced** 20` ★ |
| 认证 | `vrrp 10 authentication md5 key-string KEY` | `vrrp vrid 10 authentication-mode md5 KEY` | `vrrp vrid 10 authentication-mode md5 KEY` |
| 查看 | `show vrrp brief` | `display vrrp` | `display vrrp` |

> ⚠️ **★ HSRP 和 GLBP 是 Cisco 私有的，H3C/华为不支持。**
> **混合厂商组网必须用 VRRP。**
>
> **★ 术语差异**：Cisco 用 `decrement`（减少），H3C/华为用 `reduced`（减少）。

---

## ⑮ VRF / VPN-instance

| 目的 | **Cisco** | **H3C** | **华为** |
|:--|:--|:--|:--|
| **定义** | ★ `vrf definition NAME` ★ | ★ `ip vpn-instance NAME` ★ | ★ `ip vpn-instance NAME` ★ |
| RD | `rd 65000:100` | `route-distinguisher 65000:100` | `route-distinguisher 65000:100` |
| RT | `route-target both 65000:100` | `vpn-target 65000:100 both` | `vpn-target 65000:100 both` |
| **接口绑定** | ★ `vrf forwarding NAME` ★ | ★ `ip binding vpn-instance NAME` ★ | ★ `ip binding vpn-instance NAME` ★ |
| 查路由表 | `show ip route vrf NAME` | `display ip routing-table vpn-instance NAME` | `display ip routing-table vpn-instance NAME` |
| **ping** | ★ `ping vrf NAME <ip>` ★ | ★ `ping -vpn-instance NAME <ip>` ★ | ★ `ping -vpn-instance NAME <ip>` ★ |
| traceroute | `traceroute vrf NAME <ip>` | `tracert -vpn-instance NAME <ip>` | `tracert -vpn-instance NAME <ip>` |

**★ 术语**：Cisco 叫 **VRF**，H3C/华为叫 **VPN-instance**。概念完全一样。

---

## ⑯ GRE / IPsec

| 目的 | **Cisco** | **H3C** | **华为** |
|:--|:--|:--|:--|
| 隧道接口 | `interface Tunnel0` | `interface Tunnel0 **mode gre**` | `interface Tunnel0` |
| 隧道模式 | `tunnel mode gre ip` | （创建时指定） | `tunnel-protocol gre` |
| 隧道源 | `tunnel source Gi0/1` | `source GigabitEthernet 1/0/1` | `source GigabitEthernet 0/0/1` |
| 隧道目的 | `tunnel destination 203.2.2.2` | `destination 203.2.2.2` | `destination 203.2.2.2` |
| IKE 提案 | `crypto ikev2 proposal` | `ike proposal` | `ike proposal` |
| IPsec 转换集 | `crypto ipsec transform-set` | `ipsec transform-set` | `ipsec proposal` |
| IPsec profile | `crypto ipsec profile` | `ipsec profile` | `ipsec profile` |
| 应用到隧道 | `tunnel protection ipsec profile X` | `tunnel protection ipsec profile X` | `ipsec profile X` |
| 查 IKE SA | `show crypto ikev2 sa` | `display ike sa` | `display ike sa` |
| 查 IPsec SA | `show crypto ipsec sa` | `display ipsec sa` | `display ipsec sa` |

---

## ⑰ 网络服务与管理

| 目的 | **Cisco** | **H3C** | **华为** |
|:--|:--|:--|:--|
| **NTP 客户端** | `ntp server 10.0.0.61` | `ntp-service unicast-server 10.0.0.61` | `ntp-service unicast-server 10.0.0.61` |
| NTP 服务端 | `ntp master 3` | `ntp-service refclock-master 3` | `ntp-service refclock-master 3` |
| **时区** | `clock timezone CST 8` | `clock timezone CST add 8` | `clock timezone CST add 08:00:00` |
| 查 NTP | `show ntp status` | `display ntp-service status` | `display ntp-service status` |
| **Syslog 服务器** | `logging host 10.0.0.210` | `info-center loghost 10.0.0.210` | `info-center loghost 10.0.0.210` |
| 日志级别 | `logging trap informational` | `info-center source default loghost level informational` | `info-center loghost X level informational` |
| 查日志 | `show logging` | `display logbuffer` | `display logbuffer` |
| **SNMP 团体** | `snmp-server community X RO` | `snmp-agent community read X` | `snmp-agent community read X` |
| SNMP v3 | `snmp-server user ...` | `snmp-agent usm-user v3 ...` | `snmp-agent usm-user v3 ...` |
| **IP SLA / NQA** | ★ `ip sla 1` ★ | ★ `nqa entry admin test` ★ | ★ `nqa test-instance admin test` ★ |
| **Track** | `track 1 ip sla 1 reachability` | `track 1 nqa entry admin test reaction 1` | `track 1 nqa entry admin test reaction 1` |
| **NetFlow / NetStream** | ★ `flow monitor` / `ip flow ingress` ★ | ★ `ip netstream inbound` ★ | ★ `ip netstream inbound` ★ |
| 邻居发现 | `show cdp neighbors` | `display lldp neighbor-information` | `display lldp neighbor` |
| 堆叠 | `show switch`（StackWise/VSS） | `display irf`（**IRF**） | `display stack`（**CSS/iStack**） |

**★ 术语对照**：

| Cisco | H3C / 华为 |
|:--|:--|
| **IP SLA** | ★ **NQA**（Network Quality Analyzer）★ |
| **NetFlow** | ★ **NetStream** ★ |
| **StackWise / VSS** | ★ **IRF**（H3C）/ **CSS、iStack**（华为）★ |
| **CDP** | LLDP（国产设备用标准的 LLDP） |

---

## ⑱ ★ 混合组网速查：最容易踩的 10 个坑 ★

```
   ★ 1. Cisco 忘配 ip routing ★
      国产设备默认开三层转发，Cisco 不开
      症状：同 VLAN 通，跨 VLAN 全不通，但 SVI 和路由表都正常
   
   ★ 2. H3C/华为忘敲 active region-configuration ★
      MST Region 配了不生效，Digest 对不上
   
   ★ 3. Cisco PVST+ 与国产 MSTP 不互通 ★
      必须两边统一改成 MST，且 name/revision/映射三者完全一致
   
   ★ 4. Cisco 的 trunk allowed vlan 是【覆盖】 ★
      国产是【追加】。Cisco 上要用 add，否则整条 Trunk 断
   
   ★ 5. AD/Preference 默认值完全不同 ★
      Cisco: 静态1 < OSPF110（静态优先）
      国产:  OSPF10 < 静态60（★ OSPF 优先 ★）
      → 同样的配置，选路结果完全不同
   
   ★ 6. Cisco 需要标记 ip nat inside/outside ★
      国产不需要，NAT 直接配在出接口
   
   ★ 7. H3C/华为的链路聚合默认是静态模式 ★
      需显式配 link-aggregation mode dynamic 才启用 LACP
   
   ★ 8. HSRP/GLBP 是 Cisco 私有 ★
      混合组网只能用 VRRP
   
   ★ 9. ACL 编号范围完全不同 ★
      Cisco: 标准 1-99，扩展 100-199
      国产:  基本 2000-2999，高级 3000-3999
   
   ★ 10. EIGRP 是 Cisco 私有 ★
       国产设备不支持，混合环境必须用 OSPF/IS-IS
```

---

## ⑲ ★ 给国内网工的迁移建议 ★

**如果你日常用 H3C/华为，要学 Cisco**：

```
   ★ 第 1 天：建立命令映射 ★
      · show ↔ display
      · no ↔ undo
      · write memory ↔ save
      · 记住 Cisco 多了一层"特权模式"
   
   ★ 第 2 天：踩三个必踩的坑 ★
      · 配三层交换机时忘 ip routing
      · 改 Trunk 时用了覆盖写法
      · 配 NAT 时忘标记 inside/outside
      ★ 亲手踩一遍，印象最深 ★
   
   ★ 第 3-5 天：对照本表逐节过一遍 ★
      重点：VLAN/Trunk、STP、路由协议、ACL
   
   ★ 之后：直接进 ENCOR ★
      你已有的运维经验（VLAN、路由、ACL、故障处理）
      可以直接迁移，只是换个语法
```

**如果你日常用 Cisco，要接手国产设备**：

```
   ★ 重点注意 ★
   · display 不是 show
   · undo 不是 no
   · AD 值完全不同（可能导致选路和你预期不一样）
   · MST 要 active region-configuration
   · 链路聚合默认静态
   · 没有 EIGRP、HSRP、GLBP
```

---

**上一节** ← [02 中英术语对照表](02-中英术语对照表.md) ｜ **下一节** → [04 资源与书单](04-资源与书单.md)
