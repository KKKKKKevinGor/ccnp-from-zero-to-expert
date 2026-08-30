# Lab 03 · HSRP 高可用与接口跟踪

**对应章节**：[ENCOR 05](../02-ENCOR-350-401/05-一层二层排障与高可用-HSRP-VRRP.md)　**难度**：★★　**时长**：1.5 小时

## 目标

1. 配置 HSRP，验证主备切换
2. **亲眼看到"没配 track 时上行断了却不切换"的问题**
3. 用 IP SLA + Track 实现真正的端到端故障检测
4. 配置多组 HSRP 实现负载分担，并验证与 STP 根桥对齐的重要性

---

## 拓扑

```
                    [ 上游 / Internet ]
                     10.0.11.1    10.0.22.1
                        │              │
                     Gi0/1          Gi0/1
              ┌─────────┴────┐  ┌──────┴─────────┐
              │     SW1      │══│     SW2        │
              │ Vlan10 .10.2 │  │ Vlan10 .10.3   │
              │ Vlan20 .20.2 │  │ Vlan20 .20.3   │
              └──────┬───────┘  └───────┬────────┘
                     │                  │
                     └────────┬─────────┘
                          ┌───┴────┐
                          │  SW3   │ (接入)
                          └───┬────┘
                              │
                        [PC1] [PC2]
                      VLAN10  VLAN20
                    GW:.10.1  GW:.20.1
```

---

## Part 1：基础 HSRP

### Step 1：配置

```cisco
! ══════ SW1 ══════
SW1(config)# ip routing
SW1(config)# vlan 10,20

SW1(config)# interface Vlan10
SW1(config-if)# ip address 192.168.10.2 255.255.255.0
SW1(config-if)# standby version 2                       ! ★ 建议用 v2
SW1(config-if)# standby 10 ip 192.168.10.1              ! 虚拟 IP
SW1(config-if)# standby 10 priority 110
SW1(config-if)# ★ standby 10 preempt ★                  ! ★★★ 最容易忘
SW1(config-if)# standby 10 name VLAN10-GW
SW1(config-if)# no shutdown

SW1(config)# interface Vlan20
SW1(config-if)# ip address 192.168.20.2 255.255.255.0
SW1(config-if)# standby version 2
SW1(config-if)# standby 20 ip 192.168.20.1
SW1(config-if)# standby 20 priority 100                 ! VLAN 20 是备
SW1(config-if)# standby 20 preempt
SW1(config-if)# no shutdown

! ══════ SW2 ══════
SW2(config)# ip routing

SW2(config)# interface Vlan10
SW2(config-if)# ip address 192.168.10.3 255.255.255.0
SW2(config-if)# standby version 2
SW2(config-if)# standby 10 ip 192.168.10.1
SW2(config-if)# standby 10 priority 100                 ! VLAN 10 是备
SW2(config-if)# standby 10 preempt

SW2(config)# interface Vlan20
SW2(config-if)# ip address 192.168.20.3 255.255.255.0
SW2(config-if)# standby version 2
SW2(config-if)# standby 20 ip 192.168.20.1
SW2(config-if)# standby 20 priority 110                 ! VLAN 20 是主
SW2(config-if)# standby 20 preempt
```

### Step 2：验证

```cisco
SW1# show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State   Active         Standby        Virtual IP
Vlan10      10   110 ★P★ Active  local        192.168.10.3   192.168.10.1
Vlan20      20   100 P Standby 192.168.20.3   local          192.168.20.1
                      ↑
              ★ P 列必须有，否则没配抢占 ★
```

**在 PC 上验证虚拟 MAC**：
```
PC1> ping 192.168.10.1
PC1> arp -a
  192.168.10.1    00-00-0c-9f-f0-0a    动态
                  ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
        HSRPv2 虚拟 MAC = 0000.0C9F.Fxxx
        xxx = 组号的十六进制（组 10 = 00a）
```

**HSRPv1 vs v2 的虚拟 MAC**：
| 版本 | 虚拟 MAC | 组号范围 |
|:--|:--|:--|
| v1 | `0000.0C07.AC<组号十六进制>` | 0-255 |
| **v2** | `0000.0C9F.F<组号三位十六进制>` | **0-4095** |

### Step 3：基础切换测试

```cisco
! 在 PC1 上持续 ping 外网：ping 8.8.8.8 -t

! 关掉 SW1 的 SVI
SW1(config)# interface Vlan10
SW1(config-if)# shutdown
```

**记录丢包数**：__________

```cisco
SW2# show standby brief
Interface   Grp  Pri P State   Active    Standby   Virtual IP
Vlan10      10   100 P ★Active★ local    unknown   192.168.10.1
                        ↑ SW2 接管了 ✓
```

<details><summary>预期结果</summary>

**默认 timers (Hello 3 秒 / Hold 10 秒)：丢 3-10 个包。**

**加快切换**：
```cisco
SW1(config-if)# standby 10 timers msec 200 msec 750
SW2(config-if)# standby 10 timers msec 200 msec 750
```
再测：**丢 1 个包以内**。

⚠️ **不建议低于 `msec 200 / msec 750`**——CPU 高负载或链路抖动时可能误判，导致主备频繁切换（震荡），比切换慢更糟。
</details>

**恢复**：
```cisco
SW1(config-if)# no shutdown
```
观察 SW1 是否**自动抢回** Active（因为配了 preempt）。

---

## Part 2：★ 核心实验 —— 没有 track 的致命问题

### Step 1：制造"上行断了但 HSRP 不切换"的场景

```cisco
! 确保 SW1 是 VLAN 10 的 Active
SW1# show standby brief | include Vlan10
Vlan10      10   110 P Active  local  ...

! ★ 断掉 SW1 的【上行】接口（不是 SVI）★
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# shutdown
```

### Step 2：观察（★ 这是本实验最重要的观察）

```cisco
SW1# show standby brief
Interface   Grp  Pri P State    Active   Standby        Virtual IP
Vlan10      10   110 P ★Active★ local    192.168.10.3   192.168.10.1
                        ↑↑↑↑↑↑
              ★ SW1 依然是 Active！HSRP 认为一切正常 ★
```

```
PC1> ping 8.8.8.8
Request timed out.
Request timed out.
Request timed out.
★ 用户断网了 ★
```

**✅ 这就是"HSRP 显示一切正常，但用户断网"的经典场景。**

**为什么**：HSRP 只关心**自己的 SVI 是否 up**，完全不知道上行链路的状态。SW1 的 Vlan10 依然 up，所以它继续当 Active，继续吸引流量——**然后把流量送进黑洞**。

### Step 3：加接口跟踪修复

```cisco
! ── 定义 track 对象 ──
SW1(config)# track 1 interface GigabitEthernet0/1 line-protocol
SW1(config-track)#  delay down 5 up 20                  ! ★ 防抖

! ── HSRP 挂钩 track ──
SW1(config)# interface Vlan10
SW1(config-if)# standby 10 track 1 decrement 20
!                                            ↑
!              ★ 必须保证：110 - 20 = 90 < 100（SW2 的优先级）★
```

### Step 4：再次测试

```cisco
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# shutdown
```

**观察**：
```cisco
SW1# show track
Track 1
  Interface GigabitEthernet0/1 line-protocol
  ★ Line protocol is Down ★
    1 change, last change 00:00:08
  Delay up 20 secs, down 5 secs
  Tracked by:
    HSRP Vlan10 10

SW1# show standby Vlan10 10 | include Priority
  ★ Priority 90 (configured 110) ★
    Track object 1 state Down decrement 20

SW1# show standby brief
Interface   Grp  Pri P State    Active         Standby   Virtual IP
Vlan10      10   ★90★ P ★Standby★ 192.168.10.3  local   192.168.10.1
                                    ↑ SW2 接管了 ✓
```

```
PC1> ping 8.8.8.8
Reply from 8.8.8.8: bytes=32 time=15ms          ★ 用户正常了 ✓
```

### Step 5：decrement 值的计算验证

**故意把 decrement 设小，看会怎样**：
```cisco
SW1(config-if)# standby 10 track 1 decrement 5
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# shutdown
```

```cisco
SW1# show standby brief
Vlan10      10   ★105★ P ★Active★  local  ...
                   ↑           ↑
        110-5=105 > 100    ★ 依然是 Active，没切换！★
```

**★ 规则**：`本端优先级 − decrement < 对端优先级`

```
   110 - 5  = 105 > 100  ❌ 不切换
   110 - 10 = 100 = 100  ❌ 平局也不切换（HSRP 不抢占平局）
   110 - 20 = 90  < 100  ✅ 切换
```

**建议 decrement 留足余量**（比如差值的 2 倍）。

---

## Part 3：IP SLA + Track（比接口跟踪更可靠）

### 为什么需要

**接口跟踪的局限**：如果上行经过交换机、光猫、运营商设备，**远端故障时本地接口依然 up**。

```
   SW1 ─── [交换机] ─── [光猫] ─── [运营商] ─── Internet
    ↑
  Gi0/1 一直 up（它连的是交换机）
  
  运营商侧故障 → 实际不通
  但 track interface 检测不到
```

### 配置

```cisco
! ── 探测两个不同的目标（提高可靠性）──
SW1(config)# ip sla 1
SW1(config-ip-sla)#  icmp-echo 114.114.114.114 source-interface GigabitEthernet0/1
SW1(config-ip-sla-echo)#   frequency 5
SW1(config-ip-sla-echo)#   timeout 2000
SW1(config-ip-sla-echo)#   threshold 1000
SW1(config-ip-sla-echo)#   tag "TELECOM-DNS"
SW1(config)# ip sla schedule 1 life forever start-time now

SW1(config)# ip sla 2
SW1(config-ip-sla)#  icmp-echo 223.5.5.5 source-interface GigabitEthernet0/1
SW1(config-ip-sla-echo)#   frequency 5
SW1(config-ip-sla-echo)#   timeout 2000
SW1(config)# ip sla schedule 2 life forever start-time now

! ── 绑定 track ──
SW1(config)# track 10 ip sla 1 reachability
SW1(config-track)#  delay down 10 up 30
SW1(config)# track 11 ip sla 2 reachability
SW1(config-track)#  delay down 10 up 30

! ── ★ 组合跟踪：任一目标通就算通（避免单目标故障误切换）──
SW1(config)# track 20 list boolean or
SW1(config-track)#  object 10
SW1(config-track)#  object 11
SW1(config-track)#  delay down 15 up 60

! ── 同时跟踪接口和端到端可达性 ──
SW1(config)# track 1 interface GigabitEthernet0/1 line-protocol
SW1(config)# track 30 list boolean and
SW1(config-track)#  object 1                        ! 接口必须 up
SW1(config-track)#  object 20                       ! 且端到端可达

! ── 应用到 HSRP ──
SW1(config)# interface Vlan10
SW1(config-if)# no standby 10 track 1 decrement 20
SW1(config-if)# ★ standby 10 track 30 decrement 20 ★
```

### 验证

```cisco
SW1# show ip sla statistics
IPSLA operation id: 1
        Latest RTT: 15 milliseconds
Latest operation return code: ★ OK ★
Number of successes: 1245
Number of failures: 3

SW1# show track brief
Track  Object                    Parameter    Value   Last Change
1      interface Gi0/1           line-protocol Up     00:20:15
10     ip sla 1                  state         Up     00:20:12
11     ip sla 2                  state         Up     00:20:10
20     list boolean or           boolean       Up     00:20:10
30     list boolean and          boolean       ★Up★   00:20:15
                                                ↑ 最终用于 HSRP 的对象
```

### 测试远端故障

```cisco
! 用 ACL 阻断探测目标，模拟"链路 up 但远端不通"
SW1(config)# ip access-list extended BLOCK-PROBE
SW1(config-ext-nacl)#  deny icmp any host 114.114.114.114
SW1(config-ext-nacl)#  deny icmp any host 223.5.5.5
SW1(config-ext-nacl)#  permit ip any any
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# ip access-group BLOCK-PROBE out
```

**观察（约 15 秒后，因为配了 `delay down 15`）**：
```cisco
SW1# show track 20
Track 20
  List boolean or
  ★ Boolean OR is Down ★
    object 10 Down
    object 11 Down

SW1# show standby brief
Vlan10      10   90  P ★Standby★ 192.168.10.3  local  192.168.10.1
                        ↑ 切换了 ✓

SW1# show interfaces GigabitEthernet0/1 | include line protocol
GigabitEthernet0/1 is up, line protocol is ★ up ★
                                            ↑ ★ 接口依然 up！
```

**✅ 这证明了 IP SLA 能检测到接口跟踪检测不到的故障。**

---

## Part 4：负载分担 + STP 对齐

### Step 1：确认 HSRP 负载分担

```cisco
SW1# show standby brief
Interface   Grp  Pri P State    Active         Standby        Virtual IP
Vlan10      10   110 P ★Active★  local         192.168.10.3   192.168.10.1
Vlan20      20   100 P Standby  192.168.20.3   local          192.168.20.1

SW2# show standby brief
Vlan10      10   100 P Standby  192.168.10.2   local          192.168.10.1
Vlan20      20   110 P ★Active★  local         192.168.20.2   192.168.20.1
```

**VLAN 10 走 SW1，VLAN 20 走 SW2** ✓

### Step 2：★ 关键 —— STP 根桥必须与 HSRP 对齐

**先看不对齐的情况**：
```cisco
! 故意让 SW1 当所有 VLAN 的根桥
SW1(config)# spanning-tree vlan 10,20 root primary
SW2(config)# spanning-tree vlan 10,20 root secondary
```

**观察 SW3（接入层）的转发路径**：
```cisco
SW3# show spanning-tree vlan 20
Root ID    Priority    24596
           Address     0000.0000.1111                  ← SW1 是根
           Cost        4
           Port        1 (GigabitEthernet1/0/1)         ← 根端口朝 SW1

Interface        Role Sts Cost      Prio.Nbr Type
Gi1/0/1          Root FWD 4         128.1    P2p       ← 朝 SW1 转发
Gi1/0/2          Altn BLK 4         128.2    P2p       ← 朝 SW2 阻塞
```

**问题**：
```
   VLAN 20 的 HSRP Active 是 ★ SW2 ★
   但 VLAN 20 的 STP 根桥是 ★ SW1 ★，SW3 朝 SW2 的口被阻塞
        ↓
   PC2 发包给网关（虚拟 MAC 在 SW2 上）
        ↓
   SW3 按 STP 只能往 SW1 发
        ↓
   SW1 发现目的 MAC 是 SW2 的虚拟 MAC
        ↓
   ★ SW1 通过横向链路转给 SW2 ★
        ↓
   ★ 多走了一跳，横向链路承载了不该有的流量 ★
```

**验证次优路径**：
```cisco
SW1# show interfaces GigabitEthernet1/0/23 | include rate
  5 minute input rate ★ 45000000 ★ bits/sec
  5 minute output rate ★ 45000000 ★ bits/sec
  ↑ 横向链路流量异常高
```

### Step 3：对齐修复

```cisco
! ── SW1：VLAN 10 的 HSRP Active + STP 根桥 ──
SW1(config)# spanning-tree vlan 10 root primary
SW1(config)# spanning-tree vlan 20 root secondary

! ── SW2：VLAN 20 的 HSRP Active + STP 根桥 ──
SW2(config)# spanning-tree vlan 20 root primary
SW2(config)# spanning-tree vlan 10 root secondary
```

**验证对齐**：
```cisco
SW1# show standby brief | include Vlan10
Vlan10  10  110 P ★Active★  local ...            ← HSRP Active

SW1# show spanning-tree vlan 10 | include root
             ★ This bridge is the root ★          ← STP 根桥
                                                   ✅ 对齐

SW2# show standby brief | include Vlan20
Vlan20  20  110 P ★Active★  local ...

SW2# show spanning-tree vlan 20 | include root
             ★ This bridge is the root ★
                                                   ✅ 对齐
```

**再看横向链路流量**：应该大幅下降（只剩必要的控制流量）。

**★ 原则：让所有"主"角色集中在同一台设备上。**

---

## Part 5：故障注入

### 故障 A：忘配 preempt

```cisco
SW1(config)# interface Vlan10
SW1(config-if)# no standby 10 preempt

! 重启 HSRP 或让 SW2 先成为 Active
SW1(config-if)# shutdown
SW1(config-if)# no shutdown
```

<details><summary>症状与排查</summary>

```cisco
SW1# show standby brief
Interface   Grp  Pri  P  State    Active         Standby   Virtual IP
Vlan10      10   110 ★ ★ Standby  192.168.10.3   local     192.168.10.1
                       ↑
              ★ P 列是空的 = 没配抢占 ★
              优先级 110 更高，但依然是 Standby
```

**修复**：`standby 10 preempt`

**建议同时配抢占延迟**：
```cisco
SW1(config-if)# standby 10 preempt delay minimum 60
```

**为什么需要延迟**：设备重启后接口很快 up，但**路由协议还没收敛完**（OSPF 邻居建立 + LSDB 同步可能要 30-60 秒）。这时立即抢占成 Active，流量涌进来但路由表不完整 → **黑洞**。
</details>

### 故障 B：两台都是 Active（脑裂）

```cisco
! 在 SW1 和 SW2 之间断开 VLAN 10 的二层连通
SW1(config)# interface GigabitEthernet1/0/23
SW1(config-if)# switchport trunk allowed vlan remove 10
```

<details><summary>症状与排查</summary>

```cisco
SW1# show standby brief
Vlan10      10   110 P ★Active★  local  ★unknown★  192.168.10.1
                                          ↑ 看不到对端

SW2# show standby brief
Vlan10      10   100 P ★Active★  local  ★unknown★  192.168.10.1
                        ↑ ★ 两台都是 Active ★
```

**后果**：
- 两台设备都用相同的虚拟 IP 和虚拟 MAC 响应 ARP
- **交换机的 MAC 表会震荡**（同一个虚拟 MAC 从两个方向学到）
- 流量随机分配到两台，可能造成会话中断

**排查**：
```cisco
! ① 检查 HSRP 报文能否互通
SW1# ping 192.168.10.3                          ← 直接 ping 对端的真实 IP
! 不通 → 二层不通

! ② 检查 Trunk 放行
SW1# show interfaces trunk | include Vlan
Gi1/0/23    ★ 20,30,40 ★                        ← VLAN 10 不见了

! ③ 检查认证/版本/组号
SW1# show standby Vlan10 10 | include auth|version
SW2# show standby Vlan10 10 | include auth|version
```

**修复**：
```cisco
SW1(config-if)# switchport trunk allowed vlan add 10
```

**其他可能导致脑裂的原因**：
- HSRP 认证不匹配
- HSRP 版本不一致（v1 vs v2）
- 组号不一致
- 中间设备的 ACL 拦了 UDP 1985（v1/v2）或组播地址
</details>

### 故障 C：主备频繁切换（震荡）

```cisco
! timers 设得过于激进
SW1(config-if)# standby 10 timers msec 50 msec 150
SW2(config-if)# standby 10 timers msec 50 msec 150

! 同时给设备制造一些 CPU 负载
SW1# debug ip packet                            ! ⚠️ 只在实验环境
```

<details><summary>症状与排查</summary>

```cisco
SW1# show logging | include HSRP
%HSRP-5-STATECHANGE: Vlan10 Grp 10 state Standby -> Active
%HSRP-5-STATECHANGE: Vlan10 Grp 10 state Active -> Speak
%HSRP-5-STATECHANGE: Vlan10 Grp 10 state Speak -> Standby
%HSRP-5-STATECHANGE: Vlan10 Grp 10 state Standby -> Active
★ 反复切换 ★
```

**根因**：Hello 间隔太短（50ms），CPU 忙时来不及处理，被误判为对端失效。

**修复**：
```cisco
SW1(config-if)# standby 10 timers msec 200 msec 750
! 或用默认值
SW1(config-if)# standby 10 timers 3 10
```

**★ 震荡比切换慢更糟**：每次切换都会中断 TCP 会话、触发 ARP 更新、可能引发上游的连锁反应。
</details>

---

## 实验检查清单

```
□ ① HSRP 基础配置，主备状态正确
□ ② 验证了虚拟 MAC（PC 的 ARP 表）
□ ③ ★ 亲眼看到"没配 track 时上行断了不切换"
□ ④ 配置接口跟踪并验证切换
□ ⑤ 验证了 decrement 值太小时不切换
□ ⑥ 配置 IP SLA + Track，验证远端故障也能检测
□ ⑦ 配置多组 HSRP 负载分担
□ ⑧ ★ 验证 HSRP Active 与 STP 根桥对齐
□ ⑨ 三个故障都亲手制造并修复
□ ⑩ 写了实验笔记
```

## 关键命令总结

| 命令 | 用途 |
|:--|:--|
| **`show standby brief`** | ★ 第一命令，看 P 列和 State |
| `show standby Vlan10 10` | 看实际优先级和 track 影响 |
| **`show track`** / `show track brief` | ★ 跟踪对象状态 |
| `show ip sla statistics` | 探测结果 |
| `show spanning-tree root` | ★ 验证与 HSRP 对齐 |
| `show logging \| include HSRP` | 状态变化历史 |

## 核心结论

| 要点 | 说明 |
|:--|:--|
| **必须配 `preempt`** | HSRP 默认不抢占 |
| **必须配 track** | 否则上行断了不切换 |
| **decrement 要够大** | `优先级 − decrement < 对端优先级` |
| **IP SLA > 接口跟踪** | 能检测远端故障 |
| **必须配 delay 防抖** | 避免震荡 |
| **HSRP Active 与 STP 根桥必须对齐** | 否则次优路径 |

---

**上一个** ← [Lab 02](lab-02-STP与EtherChannel.md) ｜ **下一个** → [Lab 04: OSPF 多区域](lab-04-OSPF多区域.md)
