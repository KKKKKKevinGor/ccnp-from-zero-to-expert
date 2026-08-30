# Stage 1 · CCNA 补齐篇

> **目标**：学完这一篇，你能独立从零搭出一个可用的中小企业局域网——多 VLAN、冗余链路、三层互通、能上网、有基本安全策略。

## 为什么这一篇不能跳

如果你直接从 ENCOR 开始，会发现书里张口就是"MST 实例映射"、"OSPF 的 Type-5 LSA 在 NSSA 里转成 Type-7"——这些都建立在你已经彻底搞懂 VLAN、STP、OSPF 单区域的基础上。

**CCNP 不是 CCNA 的续集，是 CCNA 的深水区。** 浅水区没走稳，深水区必淹。

## 章节列表

| # | 章节 | 核心问题 | 后续依赖 |
|:--|:--|:--|:--|
| 01 | [VLAN 与 Trunk](01-VLAN与Trunk.md) | 怎么把一个物理网切成多个逻辑网 | VXLAN、SD-Access、二层排障 |
| 02 | [STP 生成树协议](02-STP生成树协议.md) | 有冗余链路时怎么不成环 | RSTP/MST、二层排障 |
| 03 | [EtherChannel 链路聚合](03-EtherChannel链路聚合.md) | 怎么把多条链路当一条用 | vPC/StackWise、ENCOR 交换章 |
| 04 | [OSPF 单区域](04-OSPF单区域.md) | 路由器怎么自动学到路由 | **OSPF 进阶、ENARSI 全部排障** |
| 05 | [ACL 访问控制列表](05-ACL访问控制列表.md) | 怎么控制谁能访问谁 | 路由策略、QoS 分类、安全 |
| 06 | [NAT 地址转换](06-NAT地址转换.md) | 私网怎么访问公网 | ENCOR IP 服务、IPsec NAT-T |
| 07 | [DHCP/DNS/NTP 基础服务](07-DHCP-DNS-NTP基础服务.md) | 终端怎么自动拿到网络参数 | ENARSI 基础设施服务排障 |
| 08 | [无线与安全基础](08-无线与安全基础.md) | 无线跟有线有什么本质不同 | ENCOR 无线章、802.1X |

## 学习建议

**重点排序**（时间不够时优先砸这几章）：

1. **04 OSPF 单区域** —— ENCOR 和 ENARSI 里 OSPF 的分量最重，而所有进阶内容都建立在单区域的邻居建立、LSDB、SPF 计算之上。
2. **02 STP** —— 选举逻辑是二层排障的通用语言，MST/RSTP 只是变体。
3. **01 VLAN** —— 后面 VXLAN、SD-Access 的"逻辑隔离"思想都源自这里。
4. **05 ACL** —— 通配符掩码 + 匹配逻辑，是 route-map、prefix-list、QoS class-map 的共同基础。

**如果你已经在做网络运维**（H3C/华为环境），建议：先直接做本篇 8 章的**自测题**，只回头补做错的章节。预计 3 天可以刷完。

## 过关自检

学完本篇，闭卷回答，全对再进 Stage 2：

1. 两台交换机之间的 Trunk，一端 Native VLAN 是 1，另一端是 99。会发生什么？为什么这是安全隐患？
2. 给定四台交换机的桥优先级和 MAC，怎么一步步推出根桥、每台的根端口、每条链路的指定端口？
3. LACP 的 `active` + `passive` 能起来吗？`passive` + `passive` 呢？为什么？
4. OSPF 邻居卡在 `EXSTART` 状态，最可能是什么原因？
5. 标准 ACL 应该放在靠近源还是靠近目的的位置？为什么？扩展 ACL 呢？
6. `ip nat inside source list 1 interface Gi0/1 overload` 这条命令里，`overload` 是什么意思？
7. DHCP 客户端和服务器不在同一个网段，需要配什么？`giaddr` 字段起什么作用？
8. WPA2-PSK 和 WPA2-Enterprise 的区别是什么？后者需要什么额外组件？

答案分散在各章的「自测题」小节。
