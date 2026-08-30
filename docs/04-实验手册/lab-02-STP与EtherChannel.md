# Lab 02 · STP 与 EtherChannel

**对应章节**：[CCNA补齐 02](../01-CCNA补齐篇/02-STP生成树协议.md)、[03](../01-CCNA补齐篇/03-EtherChannel链路聚合.md)、[ENCOR 04](../02-ENCOR-350-401/04-交换进阶-STP进阶与排障.md)
**难度**：★★　**时长**：2 小时

## 目标

1. **手工推导** STP 选举结果，再用设备验证
2. 配置 MST 实现负载分担
3. 配置 LACP 聚合，对比 STP 冗余与聚合冗余的收敛速度
4. 制造并排查 5 个典型故障

---

## 拓扑

```
              ┌──────────┐          ┌──────────┐
              │   SW1    │══════════│   SW2    │
              │ 0000.1111│  Po1     │ 0000.2222│
              └─┬──────┬─┘          └─┬──────┬─┘
                │      │              │      │
                │      └──────┬───────┘      │
                │             ╳              │
                │      ┌──────┴───────┐      │
              ┌─┴──────┴─┐          ┌─┴──────┴─┐
              │   SW3    │══════════│   SW4    │
              │ 0000.3333│          │ 0000.4444│
              └──────────┘          └──────────┘
```

VLAN 10, 20 → MST 实例 1；VLAN 30, 40 → MST 实例 2

---

## Part 1：手工推导（★ 先别碰设备）

### Step 1：默认优先级下的 STP 结果

**给定**：
- 所有交换机优先级默认 **32768**
- MAC：SW1 = `0000.0000.1111`，SW2 = `...2222`，SW3 = `...3333`，SW4 = `...4444`
- 所有链路 1Gbps（cost = 4）
- 拓扑：SW1-SW2、SW1-SW3、SW2-SW4、SW3-SW4、SW1-SW4、SW2-SW3（全互联）

**填表**（用三步法推导）：

| 交换机 | 到 SW1 的端口 | 到 SW2 | 到 SW3 | 到 SW4 |
|:--|:--|:--|:--|:--|
| SW1 | — | ? | ? | ? |
| SW2 | ? | — | ? | ? |
| SW3 | ? | ? | — | ? |
| SW4 | ? | ? | ? | — |

<details><summary>推导答案</summary>

**第 1 步：选根桥**
- 优先级都是 32768 → 比 MAC
- `0000.0000.1111` 最小 → **SW1 是根桥**
- **SW1 的所有端口都是指定端口（Desg/FWD）**

**第 2 步：选根端口**（每台非根交换机一个，到根桥开销最小）
- **SW2**：直连 SW1 开销 4；经 SW3/SW4 都是 8 → **朝 SW1 的口是 RP**
- **SW3**：直连 SW1 开销 4 → **朝 SW1 的口是 RP**
- **SW4**：直连 SW1 开销 4 → **朝 SW1 的口是 RP**

**第 3 步：选指定端口**（每条链路一个）

| 链路 | 两端到根开销 | 比 Bridge ID | 结果 |
|:--|:--|:--|:--|
| SW1-SW2 | SW1=0 | SW1 赢（根桥） | SW1端=Desg，SW2端=RP |
| SW1-SW3 | SW1=0 | SW1 赢 | SW1端=Desg，SW3端=RP |
| SW1-SW4 | SW1=0 | SW1 赢 | SW1端=Desg，SW4端=RP |
| **SW2-SW3** | 都是 4 | **SW2(...2222) < SW3(...3333)** | **SW2端=Desg，SW3端=BLK** |
| **SW2-SW4** | 都是 4 | **SW2 < SW4** | **SW2端=Desg，SW4端=BLK** |
| **SW3-SW4** | 都是 4 | **SW3 < SW4** | **SW3端=Desg，SW4端=BLK** |

**最终阻塞的端口**：
- SW3 朝 SW2 的口
- SW4 朝 SW2 的口
- SW4 朝 SW3 的口

**验证无环**：从任一点沿转发路径走，回不到起点 ✓

**逻辑拓扑**（一棵以 SW1 为根的树）：
```
            [SW1]
          /   |   \
      [SW2] [SW3] [SW4]
```
</details>

### Step 2：设备验证

```cisco
SW3# show spanning-tree vlan 10

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------
Gi1/0/1          Root FWD 4         128.1    P2p      ← 朝 SW1
Gi1/0/2          Altn BLK 4         128.2    P2p      ← 朝 SW2，阻塞
Gi1/0/3          Desg FWD 4         128.3    P2p      ← 朝 SW4
```

**和你的推导一致吗？** 不一致的话回去重新推一遍，找出哪一步想错了。

---

## Part 2：MST 配置与负载分担

### Step 1：全部切到 MST

```cisco
! ★ 四台都执行完全相同的配置 ★
SWx(config)# spanning-tree mode mst
SWx(config)# spanning-tree mst configuration
SWx(config-mst)#  name CAMPUS
SWx(config-mst)#  revision 1
SWx(config-mst)#  instance 1 vlan 10,20
SWx(config-mst)#  instance 2 vlan 30,40
SWx(config-mst)#  show pending                    ! ★ 提交前预览
SWx(config-mst)#  exit                            ! 退出即生效
```

### Step 2：验证 Region 一致性（★ 关键）

```cisco
SW1# show spanning-tree mst configuration digest
Name      [CAMPUS]
Revision  1     Instances configured 3
★ Digest    0x5B4D2E1F8A9C3D7E6F5A4B3C2D1E0F9A ★

SW2# show spanning-tree mst configuration digest
★ Digest    0x5B4D2E1F8A9C3D7E6F5A4B3C2D1E0F9A ★     ← 必须完全相同
```

**Digest 是三个参数（name、revision、VLAN映射）的哈希值。** 对比 Digest 比逐条对比映射表快得多。

**查看完整映射**：
```cisco
SW1# show spanning-tree mst configuration
Name      [CAMPUS]
Revision  1     Instances configured 3

Instance  Vlans mapped
--------  ---------------------------------------------------------
0         1-9,11-19,21-29,31-39,41-4094           ← ★ 实例 0 包含所有未映射的
1         10,20
2         30,40
```

### Step 3：配置负载分担

```cisco
! SW1：实例 1 的主根，实例 2 的备根
SW1(config)# spanning-tree mst 1 root primary
SW1(config)# spanning-tree mst 2 root secondary

! SW2：相反
SW2(config)# spanning-tree mst 2 root primary
SW2(config)# spanning-tree mst 1 root secondary
```

### Step 4：验证负载分担生效

```cisco
SW3# show spanning-tree mst 1

##### MST1    vlans mapped:   10,20
Bridge        address 0000.0000.3333  priority  32769
Root          address 0000.0000.1111  priority  ★ 4097 ★         ← SW1 是根
              port    Gi1/0/1

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------
Gi1/0/1          Root ★ FWD ★ 20000  128.1    P2p               ← 朝 SW1 转发
Gi1/0/2          Altn ★ BLK ★ 20000  128.2    P2p               ← 朝 SW2 阻塞
```

```cisco
SW3# show spanning-tree mst 2

##### MST2    vlans mapped:   30,40
Root          address 0000.0000.2222  priority  ★ 4098 ★         ← SW2 是根

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------
Gi1/0/1          Altn ★ BLK ★ 20000  128.1    P2p               ← 朝 SW1 阻塞
Gi1/0/2          Root ★ FWD ★ 20000  128.2    P2p               ← 朝 SW2 转发
```

**✅ 负载分担成功**：
- VLAN 10/20 的流量走 SW1
- VLAN 30/40 的流量走 SW2
- **两条上行链路都在使用**

---

## Part 3：EtherChannel

### Step 1：SW1-SW2 之间配 LACP

```cisco
! ── SW1 ──
SW1(config)# interface range GigabitEthernet1/0/23-24
SW1(config-if-range)# shutdown                              ! ★ 先关，避免配置过程中成环
SW1(config-if-range)# switchport trunk encapsulation dot1q
SW1(config-if-range)# switchport mode trunk
SW1(config-if-range)# switchport trunk native vlan 999
SW1(config-if-range)# switchport trunk allowed vlan 10,20,30,40
SW1(config-if-range)# switchport nonegotiate
SW1(config-if-range)# channel-protocol lacp
SW1(config-if-range)# ★ channel-group 1 mode active ★
SW1(config-if-range)# no shutdown

! ── SW2 完全相同 ──
```

### Step 2：验证捆绑

```cisco
SW1# show etherchannel summary
Flags:  D - down        P - bundled in port-channel
        I - stand-alone s - suspended
        R - Layer3      S - Layer2
        U - in use

Group  Port-channel  Protocol    Ports
------+-------------+-----------+------------------------------
1      ★ Po1(SU) ★    LACP       Gi1/0/23★(P)★  Gi1/0/24★(P)★
          ↑↑                              ↑            ↑
      S=二层 U=在用                   已捆绑       已捆绑

SW1# show lacp neighbor
Channel group 1 neighbors
                  LACP port                     Admin  Oper   Port
Port      Flags   Priority  Dev ID        Age   Key    Key    Number
Gi1/0/23  ★ SA ★  32768     0000.0000.2222 12s  0x0    0x1    0x17
                                                ↑
                              S=Slow, A=Active（对端确实在 active）

SW1# show interfaces Port-channel1 | include BW
  MTU 1500 bytes, BW ★ 2000000 ★ Kbit/sec           ← 2Gbps ✓
```

### Step 3：验证 STP 把 Po1 当一个端口

```cisco
SW1# show spanning-tree mst 1

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------
★ Po1 ★          Desg FWD ★ 10000 ★ 128.65   P2p
                                    ↑
                    ★ 物理口 Gi1/0/23 和 Gi1/0/24 不再单独出现 ★
                    ★ cost 从 20000 降到 10000（带宽翻倍）★
```

---

## Part 4：收敛速度对比实验（★ 最有价值的部分）

### 实验 A：STP 冗余的收敛速度

**准备**：在 SW3 后面接 PC，持续 ping SW1 的某个 SVI。

```cisco
! 测试 1：Rapid-PVST / MST（快速）
SWx(config)# spanning-tree mode mst

! 断开 SW3 的根端口
SW3(config)# interface GigabitEthernet1/0/1
SW3(config-if)# shutdown
```

**记录丢包数**：__________

<details><summary>预期结果</summary>

**MST/RSTP：丢 0-2 个包（< 1 秒）**

原因：Alternate Port 是预计算好的热备份，根端口失效时立即接替。
</details>

```cisco
! 测试 2：切回经典 PVST（慢速）
SWx(config)# spanning-tree mode pvst

! 再次断开
SW3(config)# interface GigabitEthernet1/0/1
SW3(config-if)# shutdown
```

**记录丢包数**：__________

<details><summary>预期结果</summary>

**经典 PVST：丢 30-50 个包（30-50 秒）**

原因：需要重新计算 → Listening(15s) → Learning(15s) → Forwarding。

**★ 30 倍的差距。** 这就是为什么现代网络必须用 RSTP/MST。
</details>

### 实验 B：EtherChannel 冗余的收敛速度

```cisco
! 切回 MST
SWx(config)# spanning-tree mode mst

! 断开 Po1 的一个成员链路（不是整个 Po1）
SW1(config)# interface GigabitEthernet1/0/23
SW1(config-if)# shutdown
```

**记录丢包数**：__________

<details><summary>预期结果</summary>

**丢 0-1 个包（甚至无丢包）**

```cisco
SW1# show etherchannel summary
1      Po1(SU)        LACP       Gi1/0/23★(D)★  Gi1/0/24(P)
                                          ↑ down，但 Po1 依然是 SU

SW1# show interfaces Port-channel1 | include BW
  MTU 1500 bytes, BW ★ 1000000 ★ Kbit/sec          ← 带宽降到 1Gbps，但没断
```

**★ 关键：STP 完全不感知这次变化**（Po1 依然是一个 up 的端口），**不触发任何 STP 重收敛**。

**三种冗余方式的收敛速度对比**：

| 方式 | 收敛时间 | STP 是否重收敛 |
|:--|:--|:--|
| 经典 STP | **30-50 秒** | ✅ 全网重算 |
| RSTP/MST | **< 1 秒** | ✅ 局部快速收敛 |
| **EtherChannel 成员切换** | **< 100 毫秒** | ❌ **完全不感知** |

**这就是为什么现代设计优先用 EtherChannel（配合 VSS/Stack）而不是靠 STP 做冗余。**
</details>

---

## Part 5：故障注入

### 故障 A：MST Region 不匹配

```cisco
SW4(config)# spanning-tree mst configuration
SW4(config-mst)#  revision 2                      ! 其他是 1
SW4(config-mst)#  exit
```

<details><summary>症状与排查</summary>

**症状**：
```cisco
SW3# show spanning-tree mst 1

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- ------------------
Gi1/0/3          Desg FWD 20000     128.3    P2p ★ Bound(RSTP) ★
                                                   ↑↑↑↑↑↑↑↑↑↑↑
                                        Boundary = 对面是不同 Region
```

**后果**：负载分担在这个方向失效，SW4 被当作一个"外部网桥"。

**排查**：
```cisco
SW3# show spanning-tree mst configuration digest
Digest    ★ 0x5B4D2E1F... ★

SW4# show spanning-tree mst configuration digest
Digest    ★ 0x8A2C7E9F... ★                      ← 不同！
```

**修复**：`revision 1`

**常见的不一致原因**：
| 原因 | 注意 |
|:--|:--|
| **Region Name 大小写** | `CAMPUS` ≠ `campus`（区分大小写） |
| Revision 号 | 新设备默认 0 |
| VLAN 映射差一个数字 | 逐条核对 |
| **H3C/华为忘了 `active region-configuration`** | ★ 混合组网头号坑 |
</details>

### 故障 B：LACP 双 passive

```cisco
SW1(config)# interface range GigabitEthernet1/0/23-24
SW1(config-if-range)# channel-group 1 mode passive
SW2(config)# interface range GigabitEthernet1/0/23-24
SW2(config-if-range)# channel-group 1 mode passive
```

<details><summary>症状与排查</summary>

```cisco
SW1# show etherchannel summary
1      Po1★(SD)★      LACP       Gi1/0/23★(s)★  Gi1/0/24★(s)★
          ↑                              ↑
      S=二层 D=down                  s=suspended（挂起）

SW1# show lacp neighbor
Channel group 1 neighbors
! ★ 空输出 = 对端没发 LACPDU ★
```

**根因**：双方都是 passive（被动等待），谁也不主动发起协商。

**规则**：**至少一方必须是 active**。

| SW1 \ SW2 | active | passive | on |
|:--|:--|:--|:--|
| **active** | ✅ | ✅ | ❌ |
| **passive** | ✅ | ❌ | ❌ |
| **on** | ❌ | ❌ | ✅（危险） |

**修复**：至少一端改 active（**建议两端都 active**）。
</details>

### 故障 C：同一聚合组成员口参数不一致

```cisco
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# switchport mode access             ! 另一个是 trunk
SW1(config-if)# switchport access vlan 10
```

<details><summary>症状与排查</summary>

```cisco
SW1# show etherchannel summary
1      Po1(SU)        LACP       Gi1/0/23(P)  Gi1/0/24★(I)★
                                                       ↑
                                            I = stand-alone（被踢出聚合组）

SW1# show logging | include EC-5
%EC-5-CANNOT_BUNDLE2: Gi1/0/24 is not compatible with Gi1/0/23 and will be 
suspended (vlan mode of Gi1/0/24 is access, Gi1/0/23 is trunk)
                       ↑ ★ 日志明确告诉你哪个参数不一致 ★
```

**必须一致的参数**：速率、双工、交换模式（access/trunk）、Access VLAN、Native VLAN、Allowed VLAN、MTU、STP 参数。

**修复**：
```cisco
SW1(config)# interface Port-channel1
SW1(config-if)# switchport mode trunk
! ★ 在 Port-channel 接口上配置，会自动同步到所有成员口 ★
```

**★ 最佳实践：捆好之后所有配置都在 `interface Port-channel1` 下改，不要单独改物理口。**
</details>

### 故障 D：BPDU Guard 触发

```cisco
SW3(config)# interface GigabitEthernet1/0/10
SW3(config-if)# switchport mode access
SW3(config-if)# spanning-tree portfast
SW3(config-if)# spanning-tree bpduguard enable
! 在这个口接一台交换机
```

<details><summary>症状与排查</summary>

```
%SPANTREE-2-BLOCK_BPDUGUARD: Received BPDU on port Gi1/0/10 with BPDU Guard 
enabled. Disabling port.
%PM-4-ERR_DISABLE: bpduguard error detected on Gi1/0/10, putting Gi1/0/10 in 
err-disable state
```

```cisco
SW3# show interfaces status err-disabled
Port      Name    Status              Reason               
Gi1/0/10          ★ err-disabled ★    ★ bpduguard ★
```

**恢复**：
```cisco
! 手工
SW3(config-if)# shutdown
SW3(config-if)# no shutdown

! ★ 自动恢复（推荐配上）
SW3(config)# errdisable recovery cause bpduguard
SW3(config)# errdisable recovery interval 300
SW3# show errdisable recovery
```

**这就是防止"员工私接交换机造成全楼环路"最有效的机制。**
</details>

### 故障 E：Root Guard 触发

```cisco
SW1(config)# interface GigabitEthernet1/0/3           ! 朝向 SW3
SW1(config-if)# spanning-tree guard root

! 让 SW3 试图抢根桥
SW3(config)# spanning-tree mst 1 priority 0
```

<details><summary>症状与排查</summary>

```
%SPANTREE-2-ROOTGUARD_BLOCK: Root guard blocking port GigabitEthernet1/0/3 on MST1.
```

```cisco
SW1# show spanning-tree inconsistentports
Name                 Interface              Inconsistency
-------------------- ---------------------- ------------------
MST1                 GigabitEthernet1/0/3   ★ Root Inconsistent ★
```

**恢复**：把 SW3 的优先级改回去，端口会**自动恢复**（不需要人工干预）。

**Root Guard 的价值**：防止有人接一台优先级极低的交换机进来抢走根桥，导致全网流量绕路。

**配置位置**：**汇聚交换机朝向接入层的所有端口**。
</details>

---

## Part 6：完整的加固配置

```cisco
! ── 全局 ──
SWx(config)# spanning-tree mode mst
SWx(config)# spanning-tree portfast default
SWx(config)# spanning-tree portfast bpduguard default
SWx(config)# spanning-tree loopguard default
SWx(config)# udld aggressive
SWx(config)# errdisable recovery cause bpduguard
SWx(config)# errdisable recovery cause psecure-violation
SWx(config)# errdisable recovery cause link-flap
SWx(config)# errdisable recovery interval 300

! ── 根桥规划（★ 必做）──
SW1(config)# spanning-tree mst 1 root primary
SW1(config)# spanning-tree mst 2 root secondary
SW2(config)# spanning-tree mst 2 root primary
SW2(config)# spanning-tree mst 1 root secondary

! ── 汇聚朝下游 ──
SW1(config)# interface range GigabitEthernet1/0/1-20
SW1(config-if-range)# spanning-tree guard root

! ── 接入端口 ──
SW3(config)# interface range GigabitEthernet1/0/1-20
SW3(config-if-range)# switchport mode access
SW3(config-if-range)# switchport nonegotiate
SW3(config-if-range)# spanning-tree portfast
SW3(config-if-range)# spanning-tree bpduguard enable
SW3(config-if-range)# storm-control broadcast level 5.00
SW3(config-if-range)# storm-control action trap
```

---

## 实验检查清单

```
□ ① 手工推导的 STP 结果与设备验证一致
□ ② MST Region 的 Digest 四台完全相同
□ ③ 两个实例的根桥分别在 SW1 和 SW2
□ ④ 验证了负载分担（两个实例的阻塞口不同）
□ ⑤ LACP 捆绑成功，两个成员都是 (P)
□ ⑥ 测量并记录了三种收敛速度（PVST / MST / EtherChannel）
□ ⑦ 五个故障都亲手制造并修复了
□ ⑧ 写了实验笔记
```

## 关键命令总结

| 命令 | 用途 |
|:--|:--|
| **`show spanning-tree mst configuration digest`** | ★ 验证 Region 一致性 |
| `show spanning-tree mst <实例>` | 看某个实例的角色和状态 |
| **`show etherchannel summary`** | ★ 聚合排障第一命令，看标志位 |
| `show lacp neighbor` | 确认对端 LACP 模式 |
| `show spanning-tree inconsistentports` | Root/Loop Guard 触发情况 |
| `show interfaces status err-disabled` | 被保护机制关闭的端口 |
| `show spanning-tree detail \| include occurr\|from` | ★ 找拓扑震荡源头 |

---

**上一个** ← [Lab 01](lab-01-VLAN与Trunk.md) ｜ **下一个** → [Lab 03: HSRP 高可用](lab-03-HSRP高可用.md)
