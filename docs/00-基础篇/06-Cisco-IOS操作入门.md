# 06 · Cisco IOS 操作入门

## ① 这章解决什么问题

你在 EVE-NG 里开了一台路由器，双击进去，看到一个 `Router>` 提示符。然后呢？

这一章解决三件事：**怎么把命令敲进去、怎么保证不丢配置、怎么在改错的时候不把自己锁在门外**。

最后一条尤其重要。远程配置设备时改错一条 ACL 或者接口地址，你就再也连不上它了——要么开车去机房，要么等着挨骂。IOS 提供了几个救命机制，这章会讲透。

---

## ② 原理讲透

### 2.1 命令模式层级

```
   用户模式 (User EXEC)
   Router>
   │  只能看基本信息，不能改配置
   │
   │  enable
   ▼
   特权模式 (Privileged EXEC)
   Router#
   │  可以看全部信息、重启、保存配置、调试
   │  ⚠️ 但仍然不能改配置
   │
   │  configure terminal
   ▼
   全局配置模式 (Global Configuration)
   Router(config)#
   │  改主机名、路由、ACL 等全局配置
   │
   ├── interface Gi0/0        →  Router(config-if)#     接口配置
   ├── router ospf 1          →  Router(config-router)# 路由协议配置
   ├── line vty 0 4           →  Router(config-line)#   线路配置
   ├── ip access-list ext X   →  Router(config-ext-nacl)# ACL 配置
   └── vlan 10                →  Router(config-vlan)#   VLAN 配置
```

**退出层级**：
| 命令 | 作用 |
|:--|:--|
| `exit` | 退回上一层 |
| `end` 或 `Ctrl+Z` | **直接跳回特权模式**（不管你在多深的层级） |
| `disable` | 从特权模式退回用户模式 |

**跨模式执行命令的技巧**：
```cisco
! 在配置模式下想执行特权模式的命令，前面加 do
Router(config)# do show ip interface brief
Router(config-if)# do show running-config interface Gi0/0
```
`do` 能省掉大量 `end` → 执行 → `conf t` → 重新进接口的来回切换，非常提升效率。

### 2.2 两份配置文件（最重要的概念）

| 名称 | 位置 | 特点 | 查看命令 |
|:--|:--|:--|:--|
| **running-config** | **RAM** | 当前生效的配置，**断电丢失** | `show running-config` |
| **startup-config** | **NVRAM** | 开机加载的配置，断电保留 | `show startup-config` |

**关键机制**：你在配置模式敲的每一条命令，**回车即刻生效**，写入 running-config。但**不会自动保存**到 startup-config。

```
         configure terminal 敲命令
                  ↓
          running-config (RAM)  ←── 立即生效
                  ↓
        copy running-config startup-config   ← 必须手动执行！
                  ↓
          startup-config (NVRAM)  ←── 重启后加载
```

**保存命令**（三种写法等价）：
```cisco
Router# copy running-config startup-config
Router# write memory
Router# wr                 ! 缩写，最常用
```

> **新手最常犯的错**：配了一下午，测试都正常，设备重启后配置全没了。因为忘了 `wr`。
>
> **老手的习惯**：改完一个模块就 `wr` 一次。但**大改动前先 `wr` 一次**，这样改坏了可以用 `reload` 回滚（见 2.6 节）。

### 2.3 命令行技巧

| 快捷键 / 技巧 | 作用 |
|:--|:--|
| `Tab` | 补全命令 |
| `?` | 查看当前可用命令 |
| `sh ip int br` | **命令可以缩写**，只要不产生歧义 |
| `show ?` | 查看 show 后面能跟什么 |
| `show ip ?` | 逐级探索 |
| `↑` / `Ctrl+P` | 上一条历史命令 |
| `↓` / `Ctrl+N` | 下一条历史命令 |
| `Ctrl+A` | 光标跳到行首 |
| `Ctrl+E` | 光标跳到行尾 |
| `Ctrl+W` | 删除光标前一个单词 |
| `Ctrl+U` | 删除整行 |
| `Ctrl+C` | 中断当前命令 |
| `Ctrl+Shift+6` | **中断卡住的 ping / traceroute / telnet** |
| `no <命令>` | 撤销一条配置 |

**输出过滤（管道，极其常用）**：
```cisco
Router# show running-config | include ip route
Router# show running-config | begin interface        ! 从匹配处开始显示
Router# show running-config | section interface Gi0/0 ! 只显示这一段
Router# show running-config | exclude !               ! 排除注释行
Router# show ip route | include 192.168
Router# show interfaces | include (Ethernet|error)    ! 支持正则
Router# show ip route | count 192.168                 ! 计数
```

**关掉分页（脚本化和复制配置时必备）**：
```cisco
Router# terminal length 0        ! 当前会话不分页
Router(config)# line console 0
Router(config-line)# length 0    ! 永久
```

### 2.4 基础必配项（每台新设备都要做）

```cisco
! ── 主机名 ──
Router(config)# hostname R1

! ── 关闭域名解析（防止敲错命令时卡住 30 秒去做 DNS 查询）──
R1(config)# no ip domain-lookup
! ↑ 强烈建议！敲错命令时 IOS 会把它当主机名去 DNS 解析，卡半天，非常烦人

! ── 日志不打断输入 ──
R1(config)# line console 0
R1(config-line)# logging synchronous
R1(config-line)# exec-timeout 0 0        ! 实验环境不超时；生产环境应设 5 0
R1(config-line)# exit

! ── 特权密码（加密存储）──
R1(config)# enable secret Cisco123!
! ⚠️ 不要用 enable password，那是明文/弱加密（type 7 可秒破）

! ── 控制台密码 ──
R1(config)# line console 0
R1(config-line)# password ConsolePass
R1(config-line)# login
R1(config-line)# exit

! ── 加密配置文件里的明文密码 ──
R1(config)# service password-encryption
! 注意：这只是 type 7 弱加密，防"肩窥"而已，不防真正的破解

! ── SSH 远程管理（生产必须，Telnet 明文禁用）──
R1(config)# ip domain-name example.com
R1(config)# crypto key generate rsa modulus 2048
R1(config)# username admin privilege 15 secret AdminPass123!
R1(config)# ip ssh version 2
R1(config)# line vty 0 15
R1(config-line)# transport input ssh          ! 只允许 SSH，禁用 Telnet
R1(config-line)# login local                  ! 使用本地用户名密码
R1(config-line)# exec-timeout 5 0
R1(config-line)# exit

! ── Banner 警告（法律意义：证明入侵者知道这是未授权访问）──
R1(config)# banner motd #
  ***  AUTHORIZED ACCESS ONLY  ***
  Unauthorized access is prohibited and will be prosecuted.
#

! ── 保存 ──
R1# write memory
```

> **`no ip domain-lookup` 是实验时最值得第一个敲的命令。** 不配的话，你手滑敲错一个命令，IOS 会尝试把它解析成主机名，Console 卡死 30 秒以上，`Ctrl+Shift+6` 才能中断。血泪教训。

### 2.5 接口配置

```cisco
R1(config)# interface GigabitEthernet0/0
R1(config-if)# description ### To-Core-Switch-Gi1/0/24 ###
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit

! 批量配置多个接口
R1(config)# interface range GigabitEthernet0/0-3
R1(config-if-range)# no shutdown

! 不连续的接口范围
R1(config)# interface range Gi0/1 , Gi0/3 , Gi0/5 - 8
```

> **`description` 是良心配置。** 半年后回来看配置，没有描述的接口你根本不知道对端是谁。**规范写法：`### To-<对端设备>-<对端接口> | <用途> ###`**。这在故障应急时能省下十几分钟。

**接口状态解读**（`show ip interface brief`）：

| Status | Protocol | 含义 | 排查方向 |
|:--|:--|:--|:--|
| up | up | ✅ 正常 | — |
| **administratively down** | down | 接口被 `shutdown` 了 | 敲 `no shutdown` |
| **down** | down | 物理层问题 | 网线、光模块、对端接口 shutdown |
| **up** | **down** | 物理通但二层不通 | 封装不匹配、keepalive、时钟（串口）、对端 VLAN 不对 |
| up (looped) | up | 检测到环回 | 线路有环回，查运营商 |

### 2.6 救命机制（远程操作必学）

#### 机制 1：`reload in` —— 定时重启保险绳

远程改高危配置（ACL、路由、接口地址）前：

```cisco
R1# reload in 10
Reload scheduled in 10 minutes by admin on vty0
Proceed with reload? [confirm] y
```

设备会在 10 分钟后自动重启，**加载 startup-config**（也就是你改动之前的配置）。

然后你放心去改：
- **改对了，能连上** → `reload cancel` 取消重启，然后 `wr` 保存
- **改错了，断连了** → 什么都不用做，10 分钟后设备自动重启回到旧配置

```cisco
R1# reload cancel
```

> ⚠️ **前提：改动之前 startup-config 必须是正确的。** 所以标准流程是：`wr` → `reload in 10` → 改配置 → 验证 → `reload cancel` → `wr`。

#### 机制 2：配置归档与回滚（IOS 12.3+）

```cisco
! 开启归档
R1(config)# archive
R1(config-archive)#  path flash:backup-config
R1(config-archive)#  maximum 10
R1(config-archive)#  write-memory          ! 每次 wr 自动归档一份
R1(config-archive)#  time-period 1440      ! 每天自动归档

! 手工存一份检查点
R1# archive config

! 查看归档列表
R1# show archive

! 回滚到某个版本
R1# configure replace flash:backup-config-1 
```

#### 机制 3：`configure terminal revert` —— 试运行（推荐）

```cisco
! 进入配置模式，设置 5 分钟后自动回滚
R1# configure terminal revert timer 5
R1(config)# ... 改配置 ...
R1(config)# end

! 如果配置正确，确认提交：
R1# configure confirm

! 如果 5 分钟内没有 confirm，配置自动回滚到修改前的状态
```

这比 `reload in` 更优雅——**不需要重启设备**。生产环境改核心设备强烈推荐这个。

### 2.7 密码恢复与常用维护

```cisco
! ── 查看设备信息 ──
R1# show version                    ! IOS 版本、运行时间、配置寄存器、硬件型号
R1# show inventory                  ! 硬件模块/序列号
R1# show processes cpu sorted       ! CPU 占用排序（排障必备）
R1# show processes memory sorted
R1# show logging                    ! 日志缓冲区
R1# show clock

! ── 文件系统 ──
R1# dir flash:
R1# show flash:
R1# copy running-config tftp://192.168.1.100/R1-backup.cfg
R1# copy tftp://192.168.1.100/R1.cfg running-config
R1# delete flash:old-image.bin

! ── 清空配置恢复出厂 ──
R1# write erase
R1# reload
! 注意：write erase 只清 startup-config，running-config 还在，所以必须 reload

! ── 配置寄存器（密码恢复用）──
R1# show version | include register
Configuration register is 0x2102        ! 正常值
! 0x2142 = 启动时跳过 startup-config（密码恢复用）
R1(config)# config-register 0x2102
```

**密码恢复流程**（路由器）：
1. 断电重启，开机 60 秒内按 `Ctrl+Break` 进入 ROMmon
2. `rommon> confreg 0x2142`（跳过 startup-config）
3. `rommon> reset`
4. 设备以空配置启动，进入特权模式
5. `copy startup-config running-config`（把原配置读进来，此时你已经在特权模式，不需要密码）
6. 改密码：`enable secret NewPass`
7. **接口都是 shutdown 状态，需要 `no shutdown`**
8. `config-register 0x2102` 改回正常
9. `write memory`

---

## ③ 完整的新设备初始化模板

把这个存起来，每次开新设备直接粘：

```cisco
enable
configure terminal

! ===== 基础 =====
hostname R1
no ip domain-lookup
ip domain-name lab.local
clock timezone CST 8

! ===== 控制台 =====
line console 0
 logging synchronous
 exec-timeout 15 0
 exit

! ===== 认证 =====
enable secret Str0ngEnablePass!
username admin privilege 15 secret Str0ngAdminPass!
service password-encryption

! ===== SSH =====
crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
line vty 0 15
 transport input ssh
 login local
 exec-timeout 10 0
 exit

! ===== 日志与时间 =====
service timestamps log datetime msec localtime
service timestamps debug datetime msec localtime
logging buffered 65536 informational
ntp server 203.107.6.88

! ===== 配置归档 =====
archive
 path flash:archive-config
 maximum 10
 write-memory
 exit

! ===== Banner =====
banner motd ^
*******************************************************
*  AUTHORIZED ACCESS ONLY. Activity is monitored.     *
*  Unauthorized use will be prosecuted.               *
*******************************************************
^

end
write memory
```

### 三厂商对照

| 目的 | Cisco | H3C | 华为 |
|:--|:--|:--|:--|
| 进特权/系统视图 | `enable` → `configure terminal` | `system-view` | `system-view` |
| 退出到上层 | `exit` | `quit` | `quit` |
| 直接回顶层 | `end` / `Ctrl+Z` | `return` / `Ctrl+Z` | `return` / `Ctrl+Z` |
| 保存配置 | `write memory` / `copy run start` | `save` | `save` |
| 查看当前配置 | `show running-config` | `display current-configuration` | `display current-configuration` |
| 查看启动配置 | `show startup-config` | `display saved-configuration` | `display saved-configuration` |
| 撤销配置 | `no <cmd>` | `undo <cmd>` | `undo <cmd>` |
| 设主机名 | `hostname R1` | `sysname R1` | `sysname R1` |
| 进接口 | `interface Gi0/0` | `interface GigabitEthernet 0/0` | `interface GigabitEthernet 0/0/0` |
| 开启接口 | `no shutdown` | `undo shutdown` | `undo shutdown` |
| 版本信息 | `show version` | `display version` | `display version` |
| 不分页 | `terminal length 0` | `screen-length disable` | `screen-length 0 temporary` |
| 输出过滤 | `\| include xxx` | `\| include xxx` | `\| include xxx` |
| 重启 | `reload` | `reboot` | `reboot` |

> **最容易混的两点**：
> 1. **Cisco 用 `show`，H3C/华为用 `display`**。
> 2. **Cisco 用 `no`，H3C/华为用 `undo`**。
>
> 另外，H3C/华为**没有"特权模式"这一层**，`system-view` 直接进配置视图，权限靠用户级别（0-15）控制。

---

## ④ 配套实验：设备初始化 + 远程救命演练

### 实验 A：完整初始化

在 EVE-NG 里新建一台路由器，**盲敲**完成：
1. 改主机名为 `LAB-R1`
2. 关闭域名解析
3. 配置 Gi0/0 为 `192.168.1.1/24` 并开启
4. 配置 SSH 允许 `admin/Cisco123!` 登录
5. 保存配置
6. `reload` 后验证配置还在

**验证命令**：
```cisco
LAB-R1# show ip interface brief
LAB-R1# show running-config | section line vty
LAB-R1# show ip ssh
```

### 实验 B：把自己锁在门外（然后救回来）

**这个实验必须做，因为它教你的东西，你迟早会在生产环境用上。**

**Step 1：先建立一个 SSH 会话**，从另一台设备（或宿主机）：
```
ssh admin@192.168.1.1
```

**Step 2：确保 startup-config 是好的**
```cisco
LAB-R1# write memory
```

**Step 3：设置保险绳**
```cisco
LAB-R1# reload in 5
System configuration has been modified. Save? [yes/no]: no
Reload scheduled in 5 minutes
Proceed with reload? [confirm]
```

**Step 4：故意把自己锁出去**
```cisco
LAB-R1(config)# interface GigabitEthernet0/0
LAB-R1(config-if)# shutdown
```
→ SSH 会话立刻断开。

**Step 5：等 5 分钟**（或者在 EVE-NG 里加速），设备自动重启，加载 startup-config（接口是 `no shutdown` 状态），**SSH 恢复可用**。

**Step 6：正确流程重做一遍**
```cisco
LAB-R1# write memory
LAB-R1# reload in 10
LAB-R1# configure terminal
LAB-R1(config)# ... 改一个安全的配置，比如加个 loopback ...
LAB-R1(config)# end
LAB-R1# show ip interface brief          ← 验证没断连
LAB-R1# reload cancel                     ← 取消保险绳
LAB-R1# write memory                      ← 保存
```

### 实验 C：configure revert（更优雅的方案）

```cisco
LAB-R1# configure terminal revert timer 3
LAB-R1(config)# interface Gi0/0
LAB-R1(config-if)# shutdown
LAB-R1(config-if)# end
```
→ 断连。等 3 分钟，**配置自动回滚，接口自动恢复，设备不重启**。

对比一下：`reload in` 会中断所有业务几分钟；`configure revert` 只回滚配置，业务几乎无感。生产环境优先用后者。

---

## ⑤ 排障思路

| 症状 | 原因 | 解决 |
|:--|:--|:--|
| 敲错命令后卡住 30 秒 | IOS 在做 DNS 解析 | `Ctrl+Shift+6` 中断；配 `no ip domain-lookup` |
| 重启后配置全没了 | 忘了 `write memory` | 养成改完就 `wr` 的习惯 |
| SSH 连不上，Telnet 能连 | RSA 密钥没生成 / `transport input` 没配 ssh | `crypto key generate rsa`，`transport input ssh` |
| `crypto key generate rsa` 报错 | 没配 `ip domain-name` | 先配域名再生成密钥 |
| 日志把正在输入的命令冲乱 | 未配 `logging synchronous` | `line console 0` 下配 |
| 接口 admin down | 被 shutdown | `no shutdown` |
| 接口 up/down（串口） | 封装/时钟不匹配 | 检查 `encapsulation`，DCE 侧配 `clock rate` |
| 改配置后失联 | 没设保险绳 | 下次先 `reload in` 或 `configure revert timer` |
| `show run` 输出翻页太慢 | 分页开启 | `terminal length 0` |
| 忘记密码 | — | 配置寄存器 `0x2142` 密码恢复流程 |

---

## ⑥ 考点提示 + 自测题

### 考点

- **running-config vs startup-config** 是送分题。
- **SSH 配置的四要素**（hostname、domain-name、rsa key、username）经常考漏了哪一步。
- **`configure replace` / `archive`** 在 ENCOR 的运维管理部分会出现。
- **`reload in` 保险绳**是实战最有价值的技巧，虽然不一定考。

### 自测题

**1.** 你在配置模式下改了配置，然后设备意外断电。重启后配置还在吗？

<details><summary>答案</summary>

**不在**。

配置模式敲的命令写入 **running-config（RAM）**，立即生效但**断电即失**。只有执行了 `copy running-config startup-config`（或 `write memory` / `wr`），配置才会写进 **NVRAM 的 startup-config**，重启后才会被加载。

**推论（也是实战技巧）**：如果你改配置改坏了，**只要还没保存，直接 `reload` 就能回到修改前的状态**。这是最原始的"回滚"手段。
</details>

**2.** 配置 SSH 需要哪几个前置步骤？漏了哪一步会失败？

<details><summary>答案</summary>

四个必需步骤，缺一不可：

```cisco
1. hostname R1                              ! 必须不是默认的 "Router"
2. ip domain-name example.com               ! 必须配，否则第 3 步报错
3. crypto key generate rsa modulus 2048     ! 生成密钥对（这一步依赖前两步）
4. line vty 0 15
     transport input ssh                    ! 允许 SSH
     login local                            ! 使用本地账号
   username admin privilege 15 secret Pass  ! 建本地账号
```

**最常漏的是第 2 步**。因为 RSA 密钥的标识符是 `<hostname>.<domain-name>`，没有域名就无法生成，会报：
```
% Please define a domain-name first.
```

**其次常漏的是 `login local`**。只配了 `transport input ssh` 但用 `login`（而非 `login local`），会去找 line 密码而不是用户名密码，SSH 客户端提示输入用户名时无法通过。

**验证命令**：
```cisco
R1# show ip ssh              ! 看 SSH 是否启用、版本
R1# show crypto key mypubkey rsa    ! 看密钥是否生成
```
</details>

**3.** 远程 SSH 到一台生产核心交换机，要改一条可能影响管理网段的 ACL。改之前你应该做什么？

<details><summary>答案</summary>

**方案 A：`configure revert`（推荐，不重启）**
```cisco
SW1# write memory                       ! 先保存已知正确的配置
SW1# configure terminal revert timer 5  ! 5 分钟内不确认就自动回滚
SW1(config)# ... 改 ACL ...
SW1(config)# end
! 验证：SSH 还通吗？业务正常吗？
SW1# configure confirm                  ! 确认，取消自动回滚
SW1# write memory
```
如果改坏了断连，5 分钟后配置自动回滚，**设备不重启，业务不中断**。

**方案 B：`reload in`（老办法，会重启）**
```cisco
SW1# write memory
SW1# reload in 10
SW1# configure terminal
SW1(config)# ... 改 ACL ...
SW1(config)# end
! 验证
SW1# reload cancel
SW1# write memory
```
如果断连，10 分钟后设备重启加载 startup-config。但**重启会中断所有业务几分钟**，核心设备上代价很高。

**方案 C：ACL 特有的技巧**
改 ACL 时，**先在最前面加一条放行你自己管理 IP 的规则**：
```cisco
SW1(config)# ip access-list extended MGMT-ACL
SW1(config-ext-nacl)# 1 permit ip host <你的IP> any    ! 序号 1，排最前
```
这样无论后面改成什么样，你的管理连接都不会被切断。

**核心设备优先用方案 A + 方案 C 组合。**
</details>

**4.** `show ip interface brief` 显示接口是 `up / down`（Status=up, Protocol=down），可能是什么原因？

<details><summary>答案</summary>

`Status=up` 说明**物理层正常**（有电信号/光信号），`Protocol=down` 说明**数据链路层没起来**。

常见原因：

| 接口类型 | 可能原因 |
|:--|:--|
| **串口 Serial** | 封装不匹配（一端 PPP 一端 HDLC）；DCE 侧没配 `clock rate`；keepalive 收不到 |
| **以太网** | 对端接口在错误的 VLAN；两端 Trunk/Access 模式不匹配；子接口没配 `encapsulation dot1q` |
| **Tunnel** | 隧道目的地址不可达；`tunnel source` 接口 down；递归路由问题 |
| **通用** | 对端 keepalive 未响应；线路有环回 |

**排查命令**：
```cisco
R1# show interfaces Serial0/0        ! 看 Encapsulation 和 keepalive
R1# show controllers Serial0/0       ! 看是 DCE 还是 DTE 端
R1# show interfaces Tunnel0          ! Tunnel 看 source/destination 状态
```

**对比记忆**：
- `down/down` = 物理层问题（线断了、对端 shutdown 了）
- `up/down` = 二层协商问题
- `administratively down/down` = 本端被 `shutdown` 了
</details>

**5.** H3C 设备上要查看当前生效的配置，命令是什么？和 Cisco 有什么区别？

<details><summary>答案</summary>

**H3C/华为**：`display current-configuration`
**Cisco**：`show running-config`

两个关键差异：
1. **`show` vs `display`** —— Cisco 系用 `show`，H3C/华为（VRP 及其衍生）用 `display`
2. **`no` vs `undo`** —— 撤销配置时，Cisco 用 `no xxx`，H3C/华为用 `undo xxx`

还有一个结构性差异：**H3C/华为没有 Cisco 那样独立的"特权模式"**。Cisco 是 `用户模式 → enable → 特权模式 → configure terminal → 配置模式` 三层；H3C/华为登录后直接 `system-view` 进配置视图，权限由**用户级别（0–15）**控制，不需要二次提权。

对应的保存命令也不同：Cisco 是 `write memory` / `copy run start`，H3C/华为是 `save`。

完整对照见 [附录 03 三厂商命令对照](../07-附录/03-Cisco-H3C-华为命令对照.md)。
</details>

---

## 🎓 Stage 0 结业

至此基础篇完成。回到 [Stage 0 README](README.md) 完成过关自检的 6 道题，全部答对后进入 Stage 1。

**上一章** ← [05 TCP/UDP 与常见应用协议](05-TCP-UDP与常见应用协议.md) ｜ **下一阶段** → [Stage 1 · CCNA 补齐篇](../01-CCNA补齐篇/README.md)
