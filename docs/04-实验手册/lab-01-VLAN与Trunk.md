# Lab 01 · VLAN 与 Trunk

**对应章节**：[CCNA补齐 01](../01-CCNA补齐篇/01-VLAN与Trunk.md)　**难度**：★　**时长**：1 小时

## 目标

1. 划分 VLAN，配置 Access 和 Trunk 端口
2. 用三层交换机的 SVI 实现 VLAN 间路由
3. **亲手制造 4 个典型故障并排查**

---

## 拓扑

```
   PC1(VLAN10)   PC2(VLAN20)              PC3(VLAN10)   PC4(VLAN30)
   192.168.10.10 192.168.20.10            192.168.10.11 192.168.30.10
        │            │                          │            │
     Gi1/0/1      Gi1/0/2                    Gi1/0/1      Gi1/0/2
        └─────┬──────┘                          └─────┬──────┘
        ┌─────┴──────┐      Trunk           ┌─────────┴────┐
        │    SW1     │Gi1/0/24 ══════ Gi1/0/24│    SW2      │
        │  (三层)     │                       │   (二层)     │
        └────────────┘                       └──────────────┘
```

| VLAN | 名称 | 网段 | 网关（SW1 的 SVI） |
|:--|:--|:--|:--|
| 10 | FINANCE | 192.168.10.0/24 | 192.168.10.1 |
| 20 | SALES | 192.168.20.0/24 | 192.168.20.1 |
| 30 | IT | 192.168.30.0/24 | 192.168.30.1 |
| 999 | NATIVE-UNUSED | — | 不用，仅作 Native VLAN |

---

## Part 1：基础配置

### Step 1：SW2（二层接入交换机）

**先自己想怎么配，再往下看。**

<details><summary>参考配置</summary>

```cisco
SW2(config)# hostname SW2
SW2(config)# no ip domain-lookup
SW2(config)# line console 0
SW2(config-line)# logging synchronous
SW2(config-line)# exec-timeout 0 0
SW2(config-line)# exit

! ── 创建 VLAN ──
SW2(config)# vlan 10
SW2(config-vlan)#  name FINANCE
SW2(config)# vlan 20
SW2(config-vlan)#  name SALES
SW2(config)# vlan 30
SW2(config-vlan)#  name IT
SW2(config)# vlan 999
SW2(config-vlan)#  name NATIVE-UNUSED
SW2(config-vlan)# exit

! ── 接入端口 ──
SW2(config)# interface GigabitEthernet1/0/1
SW2(config-if)#  description ### PC3 - FINANCE ###
SW2(config-if)#  switchport mode access
SW2(config-if)#  switchport access vlan 10
SW2(config-if)#  switchport nonegotiate
SW2(config-if)#  spanning-tree portfast
SW2(config-if)#  spanning-tree bpduguard enable
SW2(config-if)#  no shutdown

SW2(config)# interface GigabitEthernet1/0/2
SW2(config-if)#  description ### PC4 - IT ###
SW2(config-if)#  switchport mode access
SW2(config-if)#  switchport access vlan 30
SW2(config-if)#  switchport nonegotiate
SW2(config-if)#  spanning-tree portfast
SW2(config-if)#  spanning-tree bpduguard enable
SW2(config-if)#  no shutdown

! ── 上联 Trunk ──
SW2(config)# interface GigabitEthernet1/0/24
SW2(config-if)#  description ### To SW1 Gi1/0/24 ###
SW2(config-if)#  switchport trunk encapsulation dot1q
SW2(config-if)#  switchport mode trunk
SW2(config-if)#  switchport trunk native vlan 999
SW2(config-if)#  switchport trunk allowed vlan 10,20,30
SW2(config-if)#  switchport nonegotiate
SW2(config-if)#  no shutdown
```
</details>

### Step 2：SW1（三层核心）

<details><summary>参考配置</summary>

```cisco
SW1(config)# hostname SW1
SW1(config)# no ip domain-lookup
SW1(config)# ★ ip routing ★                        ! ★★★ 最容易忘的一条

SW1(config)# vlan 10
SW1(config-vlan)#  name FINANCE
SW1(config)# vlan 20
SW1(config-vlan)#  name SALES
SW1(config)# vlan 30
SW1(config-vlan)#  name IT
SW1(config)# vlan 999
SW1(config-vlan)#  name NATIVE-UNUSED

! ── 接入端口 ──
SW1(config)# interface GigabitEthernet1/0/1
SW1(config-if)#  switchport mode access
SW1(config-if)#  switchport access vlan 10
SW1(config-if)#  switchport nonegotiate
SW1(config-if)#  spanning-tree portfast
SW1(config-if)#  spanning-tree bpduguard enable

SW1(config)# interface GigabitEthernet1/0/2
SW1(config-if)#  switchport mode access
SW1(config-if)#  switchport access vlan 20
SW1(config-if)#  switchport nonegotiate
SW1(config-if)#  spanning-tree portfast
SW1(config-if)#  spanning-tree bpduguard enable

! ── 下联 Trunk（参数必须和 SW2 完全一致）──
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)#  description ### To SW2 Gi1/0/24 ###
SW1(config-if)#  switchport trunk encapsulation dot1q
SW1(config-if)#  switchport mode trunk
SW1(config-if)#  switchport trunk native vlan 999
SW1(config-if)#  switchport trunk allowed vlan 10,20,30
SW1(config-if)#  switchport nonegotiate

! ── 网关 SVI ──
SW1(config)# interface Vlan10
SW1(config-if)#  description ### FINANCE Gateway ###
SW1(config-if)#  ip address 192.168.10.1 255.255.255.0
SW1(config-if)#  no shutdown

SW1(config)# interface Vlan20
SW1(config-if)#  description ### SALES Gateway ###
SW1(config-if)#  ip address 192.168.20.1 255.255.255.0
SW1(config-if)#  no shutdown

SW1(config)# interface Vlan30
SW1(config-if)#  description ### IT Gateway ###
SW1(config-if)#  ip address 192.168.30.1 255.255.255.0
SW1(config-if)#  no shutdown
```
</details>

### Step 3：验证（逐项确认，不要只 ping）

```cisco
! ── ① VLAN 与端口归属 ──
SW1# show vlan brief
VLAN Name              Status    Ports
---- ----------------- --------- -------------------------------
10   FINANCE           active    Gi1/0/1
20   SALES             active    Gi1/0/2
30   IT                active
999  NATIVE-UNUSED     active

! ── ② Trunk 状态（★ 最重要的一条命令）──
SW1# show interfaces trunk

Port        Mode  Encapsulation  Status    Native vlan
Gi1/0/24    on    802.1q         trunking  ★ 999 ★           ← 两端必须一致

Port        Vlans allowed on trunk
Gi1/0/24    10,20,30                                          ← 配置的放行列表

Port        Vlans allowed and active in management domain
Gi1/0/24    10,20,30                                          ← 实际存在的 VLAN

Port        Vlans in spanning tree forwarding state and not pruned
Gi1/0/24    10,20,30                                          ← ★ 真正转发的

! ── ③ SVI 状态 ──
SW1# show ip interface brief | include Vlan
Vlan10   192.168.10.1   YES manual  ★ up      up ★
Vlan20   192.168.20.1   YES manual  up        up
Vlan30   192.168.30.1   YES manual  up        up

! ── ④ 路由表 ──
SW1# show ip route connected
C    192.168.10.0/24 is directly connected, Vlan10
C    192.168.20.0/24 is directly connected, Vlan20
C    192.168.30.0/24 is directly connected, Vlan30

! ── ⑤ ip routing 是否开启（★ 必查）──
SW1# show running-config | include ^ip routing
ip routing                                                    ← 必须有这一行
```

**连通性测试矩阵**：

| # | 测试 | 预期 | 验证什么 |
|:--|:--|:--|:--|
| 1 | PC1 → 192.168.10.1 | ✅ | 网关可达 |
| 2 | PC1 → PC3（同 VLAN 跨交换机） | ✅ | **Trunk 正常** |
| 3 | PC1 → PC2（跨 VLAN 同交换机） | ✅ | **SVI 路由** |
| 4 | PC1 → PC4（跨 VLAN 跨交换机） | ✅ | 全链路 |
| 5 | PC2 → PC4 | ✅ | 全链路 |

---

## Part 2：故障注入（★ 本实验最有价值的部分）

**每个故障：先观察症状 → 独立推导 → 再看答案。**

### 故障 A：Native VLAN 不匹配

```cisco
SW2(config)# interface GigabitEthernet1/0/24
SW2(config-if)# switchport trunk native vlan 1
```

**观察**（等 60 秒让 CDP 发现）：

<details><summary>预期症状与排查</summary>

**症状**：Console 打印告警
```
%CDP-4-NATIVE_VLAN_MISMATCH: Native VLAN mismatch discovered on 
GigabitEthernet1/0/24 (999), with SW2 GigabitEthernet1/0/24 (1).
```

**排查**：
```cisco
SW1# show interfaces trunk
Port        Mode  Encapsulation  Status    Native vlan
Gi1/0/24    on    802.1q         trunking  ★ 999 ★

SW2# show interfaces trunk
Port        Mode  Encapsulation  Status    Native vlan
Gi1/0/24    on    802.1q         trunking  ★ 1 ★             ← 找到了
```

**危害**：VLAN 999 和 VLAN 1 被"焊接"在一起，两个本该隔离的广播域互通了。

**修复**：
```cisco
SW2(config-if)# switchport trunk native vlan 999
```

**加固**（强制给 Native VLAN 也打标签）：
```cisco
SW1(config)# vlan dot1q tag native
SW2(config)# vlan dot1q tag native
```
</details>

### 故障 B：Trunk 未放行 VLAN

```cisco
SW1(config)# interface GigabitEthernet1/0/24
SW1(config-if)# switchport trunk allowed vlan 10,20
```

**观察**：

<details><summary>预期症状与排查</summary>

**症状**：PC4（VLAN 30）ping 不通网关和其他 VLAN，但 PC3（VLAN 10）正常。

**排查（★ 用四段输出法）**：
```cisco
SW1# show interfaces trunk

Port        Vlans allowed on trunk
Gi1/0/24    ★ 10,20 ★                          ← 第 2 段：30 不见了！

Port        Vlans allowed and active in management domain
Gi1/0/24    10,20

Port        Vlans in spanning tree forwarding state and not pruned
Gi1/0/24    10,20
```

**★ 排障技巧：从第 4 段往上看**
- 第 4 段没有 VLAN 30 → 往上看
- 第 3 段也没有 → 往上看
- 第 2 段也没有 → **配置里就没放行**

如果第 2 段有但第 3 段没有 → **VLAN 没创建**
如果第 3 段有但第 4 段没有 → **被 STP 阻塞了**

**修复（★ 注意用 add）**：
```cisco
SW1(config-if)# switchport trunk allowed vlan ★ add ★ 30
```

**⚠️ 危险对比**：
```cisco
! ❌ 这样会把 10 和 20 都覆盖掉！
SW1(config-if)# switchport trunk allowed vlan 30

! ✅ 正确
SW1(config-if)# switchport trunk allowed vlan add 30
```

**在生产网络敲错这个 = 整栋楼断网。**
</details>

### 故障 C：忘了 `ip routing`（★ 最迷惑的故障）

```cisco
SW1(config)# no ip routing
```

**观察**：

<details><summary>预期症状与排查</summary>

**症状**：
- ✅ 同 VLAN 内通信正常（PC1 ↔ PC3）
- ❌ **跨 VLAN 全部不通**（PC1 → PC2 不通）
- ⚠️ **但所有表面证据都正常**！

```cisco
SW1# show ip interface brief | include Vlan
Vlan10   192.168.10.1   YES manual  ★ up      up ★         ← SVI 正常
Vlan20   192.168.20.1   YES manual  up        up

SW1# show ip route connected
C    192.168.10.0/24 is directly connected, Vlan10         ← 路由表正常
C    192.168.20.0/24 is directly connected, Vlan20
```

**★ 这就是这个故障的迷惑之处**：SVI up/up、路由表有条目，看起来一切正常，但就是不转发。

**排查（唯一的线索）**：
```cisco
SW1# show running-config | include ^ip routing
! ★ 空输出 = 没开启 ★

! 对比正常时：
SW1# show running-config | include ^ip routing
ip routing
```

**另一个线索**：
```cisco
SW1# show ip route
! 没开 ip routing 时，路由表顶部不会显示 "Gateway of last resort"
! 而且学不到任何动态路由
```

**修复**：
```cisco
SW1(config)# ip routing
```

**★ 记住这个症状**：**同 VLAN 通、跨 VLAN 全不通、SVI 状态正常** → 立刻查 `ip routing`。

**厂商差异**：H3C 和华为的三层交换机**默认就开启路由转发**，没有这个开关。从国产设备转 Cisco 的人特别容易踩。
</details>

### 故障 D：SVI 因为没有 up 的端口而 down

```cisco
SW1(config)# interface GigabitEthernet1/0/2
SW1(config-if)# shutdown
! 假设 VLAN 20 只有这一个端口，且 Trunk 上 VLAN 20 也没有活动主机
```

**观察**：

<details><summary>预期症状与排查</summary>

**症状**：
```cisco
SW1# show ip interface brief | include Vlan20
Vlan20   192.168.20.1   YES manual  up        ★ down ★
                                              ↑ line protocol down
```

**原因：SVI up 的三个条件**
1. VLAN 存在于 VLAN 数据库 ✓
2. **★ 该 VLAN 内至少有一个 up 的物理端口 ★** ← 不满足
3. SVI 本身没有 shutdown ✓

**排查**：
```cisco
SW1# show vlan brief | include ^20
20   SALES             active    ★ (空的，没有端口) ★

SW1# show interfaces status | include Gi1/0/2
Gi1/0/2                      disabled     20         auto   auto
                             ↑ 被 shutdown 了

SW1# show interfaces trunk | include 20
! Trunk 上虽然放行了 VLAN 20，但如果对端也没有活动主机，
! 该 VLAN 在 Trunk 上可能不处于 forwarding 状态
```

**修复**：
```cisco
SW1(config-if)# no shutdown
```

**强制让 SVI 保持 up（测试用）**：
```cisco
SW1(config)# interface Vlan20
SW1(config-if)# ★ no autostate ★
```

**`no autostate` 的适用场景**：
- SVI 只用作管理接口，不需要有活动端口
- 该 VLAN 的流量全部通过 Trunk，本地没有 Access 口
- 测试环境

**生产环境慎用**——autostate 是个有用的保护机制（防止流量被送到一个实际没有终端的 VLAN）。
</details>

---

## Part 3：进阶挑战

### 挑战 1：VLAN 跳跃攻击验证

```cisco
! 把某个接入端口改回默认的 dynamic 模式
SW1(config)# interface GigabitEthernet1/0/3
SW1(config-if)# switchport mode dynamic desirable
```

**在这个端口上接另一台交换机**，观察它们自动协商成了 Trunk：
```cisco
SW1# show interfaces GigabitEthernet1/0/3 switchport | include Mode
Administrative Mode: dynamic desirable
Operational Mode: ★ trunk ★                      ← 自动变成了 Trunk！
```

**这就是 Switch Spoofing 攻击的原理。**

**加固**：
```cisco
SW1(config-if)# switchport mode access
SW1(config-if)# switchport nonegotiate
```

### 挑战 2：Voice VLAN

```cisco
SW1(config)# vlan 100
SW1(config-vlan)#  name VOICE

SW1(config)# interface GigabitEthernet1/0/5
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10           ! 数据 VLAN（给 PC）
SW1(config-if)# switchport voice vlan 100           ! 语音 VLAN（给 IP 电话）
SW1(config-if)# mls qos trust device cisco-phone
SW1(config-if)# spanning-tree portfast
```

**验证**：
```cisco
SW1# show interfaces GigabitEthernet1/0/5 switchport | include Voice
Voice VLAN: 100 (VOICE)
```

**理解**：这个端口同时承载两个 VLAN——**数据 VLAN 不打标签（Native），语音 VLAN 打标签**。IP 电话内置一个小交换机，把 PC 的流量原样转发，把自己的语音流量打上 VLAN 100 标签。

### 挑战 3：完整的接入端口安全模板

把学到的都用上：
```cisco
SW1(config)# interface range GigabitEthernet1/0/1-20
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 100
 switchport nonegotiate                       ! 防 VLAN 跳跃
 
 spanning-tree portfast
 spanning-tree bpduguard enable               ! 防私接交换机
 
 switchport port-security
 switchport port-security maximum 3
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 
 storm-control broadcast level 5.00
 storm-control action trap
 
 no shutdown
```

---

## 实验检查清单

```
□ ① 所有 VLAN 都创建了（show vlan brief）
□ ② Trunk 两端的 Native VLAN 一致
□ ③ Trunk 放行了所有需要的 VLAN
□ ④ 三层交换机开启了 ip routing
□ ⑤ 所有 SVI 都是 up/up
□ ⑥ 五项连通性测试全部通过
□ ⑦ 四个故障都亲手制造并修复了
□ ⑧ 写了实验笔记
```

## 关键命令总结

| 命令 | 用途 |
|:--|:--|
| **`show interfaces trunk`** | ★ Trunk 排障第一命令，四段输出法 |
| `show vlan brief` | VLAN 与端口归属 |
| `show ip interface brief \| include Vlan` | SVI 状态 |
| **`show run \| include ^ip routing`** | ★ 三层转发是否开启 |
| `show interfaces Gi1/0/1 switchport` | 单个端口的详细模式 |
| `switchport trunk allowed vlan **add** X` | ★ 追加而非覆盖 |

## 实验笔记

```markdown
# Lab 01 笔记

## 我踩的坑
1. 
2. 

## 最有用的命令
1. show interfaces trunk（四段输出法）
2. 
3. 

## 还没搞懂的
1. 
```

---

**下一个实验** → [Lab 02: STP 与 EtherChannel](lab-02-STP与EtherChannel.md)
