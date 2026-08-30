# 14 · 网络自动化：Python / REST / NETCONF / Ansible

> **ENCOR 考纲权重 15%。这是传统网工丢分最多的一章，也是最能拉开差距的一章。**

## ① 这章解决什么问题

你有 200 台交换机，现在要：
- 给所有设备加一条 NTP 服务器配置
- 检查所有设备的 IOS 版本是否有已知漏洞
- 每天备份所有设备的配置
- 上线一个新 VLAN，要在 50 台设备上配 Trunk 放行

**手工做**：每台设备 SSH 登录、敲命令、检查、退出。200 台 × 2 分钟 = **6.7 小时**，还容易出错（漏一台、敲错一个字符）。

**自动化做**：一个脚本，**5 分钟**，而且**每台设备的配置绝对一致**。

更重要的是：**自动化让"验证"变得可行**。手工检查 200 台设备的配置一致性几乎不可能，脚本可以每天跑一遍。

---

## ② 数据格式：JSON / XML / YAML

自动化的第一步是**理解机器可读的数据格式**。

### 2.1 三种格式对比

| | **JSON** | **XML** | **YAML** |
|:--|:--|:--|:--|
| 用于 | **REST API** | **NETCONF** | **Ansible / 配置文件** |
| 可读性 | 中 | 差 | ★ **最好** |
| 结构标识 | `{}` `[]` | 标签 `<tag>` | **缩进** |
| 支持注释 | ❌ | ✅ | ✅ |
| 数据类型 | 有（字符串/数字/布尔/null） | 都是字符串 | 有 |

### 2.2 同一份数据的三种表示

**JSON**：
```json
{
  "device": {
    "hostname": "R1",
    "interfaces": [
      {
        "name": "GigabitEthernet0/1",
        "ip": "10.0.12.1",
        "mask": "255.255.255.252",
        "enabled": true
      },
      {
        "name": "GigabitEthernet0/2",
        "ip": "10.0.13.1",
        "mask": "255.255.255.252",
        "enabled": false
      }
    ]
  }
}
```

**XML**：
```xml
<device>
  <hostname>R1</hostname>
  <interfaces>
    <interface>
      <name>GigabitEthernet0/1</name>
      <ip>10.0.12.1</ip>
      <mask>255.255.255.252</mask>
      <enabled>true</enabled>
    </interface>
    <interface>
      <name>GigabitEthernet0/2</name>
      <ip>10.0.13.1</ip>
      <mask>255.255.255.252</mask>
      <enabled>false</enabled>
    </interface>
  </interfaces>
</device>
```

**YAML**：
```yaml
device:
  hostname: R1
  interfaces:
    - name: GigabitEthernet0/1
      ip: 10.0.12.1
      mask: 255.255.255.252
      enabled: true
    - name: GigabitEthernet0/2
      ip: 10.0.13.1
      mask: 255.255.255.252
      enabled: false
```

**考点：识别格式**
- 看到 `{` `}` `[` `]` `"key": value` → **JSON**
- 看到 `<tag>...</tag>` → **XML**
- 看到**缩进 + `key: value` + `- ` 列表项** → **YAML**

> **YAML 的坑：缩进必须用空格，不能用 Tab。** 而且缩进必须一致。这是初学者最常见的错误。

---

## ③ 网络 API 三剑客

### 3.1 三者对比（★ 高频考点）

| | **NETCONF** | **RESTCONF** | **REST API** |
|:--|:--|:--|:--|
| 标准 | **RFC 6241** | **RFC 8040** | 无统一标准 |
| 传输 | **SSH（端口 830）** | **HTTPS（端口 443）** | HTTPS（443） |
| 数据格式 | **XML** | **JSON 或 XML** | 通常 **JSON** |
| 数据模型 | **YANG** | **YANG** | 厂商自定义 |
| 操作 | `<get>` `<get-config>` `<edit-config>` `<commit>` | **HTTP 动词**（GET/POST/PUT/PATCH/DELETE） | HTTP 动词 |
| **事务** | ✅ **支持**（candidate + commit + rollback） | ❌ 有限 | ❌ |
| 学习曲线 | 陡（XML 复杂） | ★ **平缓** | 平缓 |

**选型建议**：
- **需要事务性配置**（多条配置要么全成功要么全回滚）→ **NETCONF**
- **简单的读写操作** → **RESTCONF**
- **对接控制器/管理平台**（DNA Center、vManage、ISE）→ **REST API**

### 3.2 HTTP 方法与状态码（必背）

| 方法 | 用途 | 幂等 |
|:--|:--|:--|
| **GET** | **读取**资源 | ✅ |
| **POST** | **创建**新资源 | ❌ |
| **PUT** | **替换**整个资源 | ✅ |
| **PATCH** | **部分更新** | ❌ |
| **DELETE** | **删除** | ✅ |

**状态码**：

| 码 | 含义 |
|:--|:--|
| **200 OK** | 成功 |
| **201 Created** | 创建成功 |
| **204 No Content** | 成功但无返回内容（常见于 DELETE） |
| **400 Bad Request** | 请求格式错误 |
| **401 Unauthorized** | **未认证**（没提供凭据或凭据错） |
| **403 Forbidden** | **已认证但无权限** |
| **404 Not Found** | 资源不存在 |
| **405 Method Not Allowed** | 方法不支持（比如对只读资源用 POST） |
| **500 Internal Server Error** | 服务器错误 |

> **考点：401 vs 403** —— 401 是"你是谁我不知道"（认证问题），403 是"我知道你是谁，但你没这个权限"（授权问题）。

### 3.3 YANG 数据模型

**YANG 是描述"网络配置数据的结构"的建模语言。**

```yang
module ietf-interfaces {
  container interfaces {
    list interface {
      key "name";
      leaf name { type string; }
      leaf description { type string; }
      leaf enabled { type boolean; default "true"; }
      leaf type { type identityref; }
    }
  }
}
```

**关键节点类型**：

| 类型 | 说明 |
|:--|:--|
| **container** | 容器（分组，无数据） |
| **list** | 列表（多个实例，有 key） |
| **leaf** | 叶子（单个值） |
| **leaf-list** | 叶子列表（多个值） |

**两类模型**：

| 类型 | 说明 | 例子 |
|:--|:--|:--|
| **开放模型** | IETF / OpenConfig 定义，**跨厂商通用** | `ietf-interfaces`、`openconfig-bgp` |
| **原生模型** | 厂商自定义，**功能全但不通用** | `Cisco-IOS-XE-native` |

> **实践权衡**：开放模型可移植但功能受限（很多厂商特性没覆盖），原生模型功能全但换厂商就要重写。**通用配置用开放模型，特殊功能用原生模型。**

### 3.4 RESTCONF 实操

**启用**：
```cisco
R1(config)# ip http secure-server
R1(config)# restconf
R1(config)# username api-user privilege 15 secret ApiPassword
R1(config)# aaa new-model
R1(config)# aaa authentication login default local
R1(config)# aaa authorization exec default local
```

**URL 结构**：
```
https://<设备IP>/restconf/data/<YANG模块>:<容器>/<列表>=<key>
                    ↑          ↑
                 固定前缀   模型路径
```

**curl 示例**：
```bash
# ── 读取所有接口 ──
curl -k -u api-user:ApiPassword \
  -X GET \
  -H "Accept: application/yang-data+json" \
  https://10.0.0.1/restconf/data/ietf-interfaces:interfaces

# ── 读取单个接口 ──
curl -k -u api-user:ApiPassword \
  -X GET \
  -H "Accept: application/yang-data+json" \
  "https://10.0.0.1/restconf/data/ietf-interfaces:interfaces/interface=GigabitEthernet2"

# ── 修改接口描述（PATCH 部分更新）──
curl -k -u api-user:ApiPassword \
  -X PATCH \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "ietf-interfaces:interface": {
      "name": "GigabitEthernet2",
      "description": "### Updated via RESTCONF ###"
    }
  }' \
  "https://10.0.0.1/restconf/data/ietf-interfaces:interfaces/interface=GigabitEthernet2"

# ── 创建新的 Loopback（PUT 替换/创建）──
curl -k -u api-user:ApiPassword \
  -X PUT \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "ietf-interfaces:interface": {
      "name": "Loopback100",
      "description": "Created via RESTCONF",
      "type": "iana-if-type:softwareLoopback",
      "enabled": true,
      "ietf-ip:ipv4": {
        "address": [{"ip": "100.100.100.100", "netmask": "255.255.255.255"}]
      }
    }
  }' \
  "https://10.0.0.1/restconf/data/ietf-interfaces:interfaces/interface=Loopback100"

# ── 删除 ──
curl -k -u api-user:ApiPassword \
  -X DELETE \
  "https://10.0.0.1/restconf/data/ietf-interfaces:interfaces/interface=Loopback100"
```

### 3.5 NETCONF 实操

**启用**：
```cisco
R1(config)# netconf-yang
R1(config)# netconf-yang feature candidate-datastore
```

**验证**：
```cisco
R1# show netconf-yang sessions
R1# show netconf-yang datastores
R1# show platform software yang-management process
```

**三个数据存储（Datastore）**：

| Datastore | 说明 |
|:--|:--|
| **running** | 当前生效的配置 |
| **candidate** | **草稿区**，可以改，改完 commit 才生效 |
| **startup** | 启动配置 |

**★ candidate + commit 是 NETCONF 相对 RESTCONF 的核心优势**：
```
   ① 把多条配置写进 candidate（还没生效）
   ② 检查是否有语法错误（validate）
   ③ commit → 一次性全部生效
   ④ 有问题 → discard-changes 或 rollback
   
   ★ 要么全成功，要么全不生效 —— 这就是事务性 ★
```

---

## ④ Python 网络自动化

### 4.1 常用库

| 库 | 用途 | 特点 |
|:--|:--|:--|
| **Netmiko** | **SSH 连接设备，敲 CLI** | ★ 最容易上手，支持多厂商 |
| **Paramiko** | 底层 SSH 库 | Netmiko 的基础 |
| **NAPALM** | **多厂商统一 API** | 抽象层，同一段代码支持多厂商 |
| **ncclient** | **NETCONF 客户端** | XML 操作 |
| **requests** | HTTP/REST | 通用 |
| **Nornir** | 自动化框架 | 比 Ansible 更 Python 化，性能好 |
| **pyATS/Genie** | Cisco 测试框架 | **结构化解析 show 命令输出** |
| **TextFSM / ntc-templates** | 解析 CLI 输出 | 把文本变成结构化数据 |

### 4.2 Netmiko 示例

```python
#!/usr/bin/env python3
"""批量采集设备信息"""
from netmiko import ConnectHandler
from netmiko.exceptions import NetmikoTimeoutException, NetmikoAuthenticationException
import json

devices = [
    {"device_type": "cisco_ios", "host": "10.0.0.1", "username": "admin", "password": "pass"},
    {"device_type": "cisco_ios", "host": "10.0.0.2", "username": "admin", "password": "pass"},
    {"device_type": "hp_comware", "host": "10.0.0.3", "username": "admin", "password": "pass"},
]

results = {}

for device in devices:
    host = device["host"]
    try:
        with ConnectHandler(**device) as conn:
            hostname = conn.find_prompt().strip("#>")
            # use_textfsm=True 会把输出解析成结构化数据（需要 ntc-templates）
            version = conn.send_command("show version", use_textfsm=True)
            interfaces = conn.send_command("show ip interface brief", use_textfsm=True)

            results[host] = {
                "hostname": hostname,
                "version": version[0].get("version") if version else None,
                "uptime": version[0].get("uptime") if version else None,
                "interfaces": interfaces,
            }
            print(f"[OK]   {host} ({hostname})")

    except NetmikoAuthenticationException:
        print(f"[FAIL] {host} - 认证失败")
        results[host] = {"error": "auth_failed"}
    except NetmikoTimeoutException:
        print(f"[FAIL] {host} - 连接超时")
        results[host] = {"error": "timeout"}
    except Exception as e:
        print(f"[FAIL] {host} - {e}")
        results[host] = {"error": str(e)}

with open("inventory.json", "w", encoding="utf-8") as f:
    json.dump(results, f, indent=2, ensure_ascii=False)
```

**批量配置（带幂等检查）**：
```python
#!/usr/bin/env python3
"""批量下发 NTP 配置（幂等）"""
from netmiko import ConnectHandler

NTP_CONFIG = [
    "ntp server 10.1.30.61 prefer",
    "ntp server 10.1.30.62",
    "clock timezone CST 8",
]

def configure_ntp(device):
    with ConnectHandler(**device) as conn:
        # ★ 幂等：先检查，已配置就跳过
        current = conn.send_command("show running-config | include ^ntp server")
        if "10.1.30.61" in current and "10.1.30.62" in current:
            return f"{device['host']}: 已配置，跳过"

        output = conn.send_config_set(NTP_CONFIG)
        conn.save_config()

        # 验证
        verify = conn.send_command("show ntp status")
        status = "synchronized" if "synchronized" in verify else "not yet synced"
        return f"{device['host']}: 已配置 ({status})"

for device in devices:
    print(configure_ntp(device))
```

**配置备份（生产实用）**：
```python
#!/usr/bin/env python3
"""每日配置备份 + 差异检测"""
from netmiko import ConnectHandler
from datetime import datetime
import difflib
import os

BACKUP_DIR = "backups"

def backup_config(device):
    host = device["host"]
    os.makedirs(f"{BACKUP_DIR}/{host}", exist_ok=True)

    with ConnectHandler(**device) as conn:
        conn.send_command("terminal length 0")
        config = conn.send_command("show running-config")

    # 去掉每次都变的行（时间戳等），避免误报差异
    lines = [l for l in config.splitlines()
             if not l.startswith("! Last configuration change")
             and not l.startswith("! NVRAM config last updated")]
    config_clean = "\n".join(lines)

    today = datetime.now().strftime("%Y%m%d")
    path = f"{BACKUP_DIR}/{host}/{today}.cfg"

    # 与上一次备份做 diff
    existing = sorted(os.listdir(f"{BACKUP_DIR}/{host}"))
    if existing:
        with open(f"{BACKUP_DIR}/{host}/{existing[-1]}", encoding="utf-8") as f:
            old = f.read()
        diff = list(difflib.unified_diff(
            old.splitlines(), config_clean.splitlines(),
            fromfile="previous", tofile="current", lineterm=""))
        if diff:
            print(f"⚠️  {host} 配置发生变更：")
            print("\n".join(diff[:30]))
        else:
            print(f"✅ {host} 配置无变更")

    with open(path, "w", encoding="utf-8") as f:
        f.write(config_clean)
```

### 4.3 requests + RESTCONF

```python
#!/usr/bin/env python3
import requests
import json
from requests.auth import HTTPBasicAuth
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

HOST = "10.0.0.1"
AUTH = HTTPBasicAuth("api-user", "ApiPassword")
HEADERS = {
    "Accept": "application/yang-data+json",
    "Content-Type": "application/yang-data+json",
}
BASE = f"https://{HOST}/restconf/data"

# ── 读取所有接口 ──
r = requests.get(f"{BASE}/ietf-interfaces:interfaces",
                 auth=AUTH, headers=HEADERS, verify=False, timeout=10)

if r.status_code == 200:
    data = r.json()
    for intf in data["ietf-interfaces:interfaces"]["interface"]:
        name = intf["name"]
        enabled = intf.get("enabled", False)
        desc = intf.get("description", "")
        print(f"{name:30} enabled={enabled!s:6} {desc}")
elif r.status_code == 401:
    print("认证失败：检查用户名密码")
elif r.status_code == 404:
    print("资源不存在：检查 YANG 模块名和路径")
else:
    print(f"错误 {r.status_code}: {r.text}")

# ── 创建 Loopback ──
payload = {
    "ietf-interfaces:interface": {
        "name": "Loopback100",
        "description": "Created by automation",
        "type": "iana-if-type:softwareLoopback",
        "enabled": True,
        "ietf-ip:ipv4": {
            "address": [{"ip": "100.100.100.100", "netmask": "255.255.255.255"}]
        }
    }
}
r = requests.put(f"{BASE}/ietf-interfaces:interfaces/interface=Loopback100",
                 auth=AUTH, headers=HEADERS, json=payload, verify=False)
print(f"创建 Loopback100: {r.status_code}")   # 201 Created 或 204 No Content
```

### 4.4 ncclient + NETCONF

```python
#!/usr/bin/env python3
from ncclient import manager
import xml.dom.minidom

with manager.connect(
    host="10.0.0.1",
    port=830,                       # ★ NETCONF over SSH
    username="admin",
    password="password",
    hostkey_verify=False,
    device_params={"name": "iosxe"},
) as m:
    # ── 查看设备支持的能力 ──
    for cap in m.server_capabilities:
        if "ietf-interfaces" in cap:
            print(cap)

    # ── 读取配置 ──
    filter_xml = """
    <filter>
      <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces"/>
    </filter>
    """
    result = m.get_config(source="running", filter=filter_xml)
    print(xml.dom.minidom.parseString(result.xml).toprettyxml(indent="  "))

    # ── 修改配置（事务性）──
    config_xml = """
    <config>
      <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
        <interface>
          <name>Loopback200</name>
          <description>Created via NETCONF</description>
          <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">
            ianaift:softwareLoopback
          </type>
          <enabled>true</enabled>
        </interface>
      </interfaces>
    </config>
    """
    m.edit_config(target="candidate", config=config_xml)   # ★ 先写草稿
    m.validate(source="candidate")                          # ★ 校验
    m.commit()                                              # ★ 提交生效
    # 有问题可以 m.discard_changes()
```

### 4.5 ⚠️ 上面代码里的两个安全妥协（实验可以，生产不行）

本章的示例代码为了简化，做了两处妥协。**生产环境必须修掉**，而且这也是面试和 Code Review 会问到的点。

#### 妥协 1：`verify=False` 关闭了 TLS 证书校验

```python
requests.get(url, verify=False)                    # ❌ 生产禁止
urllib3.disable_warnings()                          # ❌ 连警告都关掉了
```

**风险**：关闭证书校验 = **任何中间人都可以伪装成你的设备**，截获你的 API 凭据和配置数据。你以为在跟核心交换机通信，实际可能在跟攻击者的服务器通信。

**正确做法**：

```python
# ── 方案 1（推荐）：把设备/内部 CA 的证书加入信任链 ──
requests.get(url, verify="/etc/ssl/certs/corp-ca.crt")

# 或设置环境变量，全局生效
# export REQUESTS_CA_BUNDLE=/etc/ssl/certs/corp-ca.crt

# ── 方案 2：给设备签发内部 CA 的正式证书 ──
# 设备侧：
#   crypto pki trustpoint CORP-CA
#    enrollment terminal
#    subject-name CN=R1.corp.example.com
#   crypto pki authenticate CORP-CA
#   crypto pki enroll CORP-CA
#   ip http secure-trustpoint CORP-CA
# 然后 Python 侧就可以正常 verify=True（默认值）
requests.get(url)                                   # ✅
```

**只有在隔离的实验环境**（EVE-NG 里的设备用自签名证书）才可以用 `verify=False`，而且应该显式注释说明。

#### 妥协 2：用 stdlib 的 XML 解析器处理外部数据

```python
import xml.dom.minidom                              # ⚠️ 有 XXE / billion-laughs 风险
xml.dom.minidom.parseString(result.xml)
```

**风险**：Python 标准库的 XML 解析器默认会处理**外部实体引用（XXE）** 和**实体展开**，恶意 XML 可以：
- 读取解析主机上的任意文件（`<!ENTITY xxe SYSTEM "file:///etc/passwd">`）
- 通过嵌套实体展开耗尽内存（billion-laughs 攻击）

在网络自动化场景里，XML 来自你自己的设备，风险相对低。但如果 XML 可能来自不完全可信的来源（第三方控制器、多租户环境、被攻陷的设备），就是真实的漏洞。

**正确做法**：

```bash
pip install defusedxml
```

```python
# ❌ 不安全
import xml.dom.minidom
xml.dom.minidom.parseString(data)

# ✅ 安全（API 完全兼容，改个 import 就行）
import defusedxml.minidom
defusedxml.minidom.parseString(data)

# ElementTree 同理
# import xml.etree.ElementTree as ET        ❌
import defusedxml.ElementTree as ET       # ✅
```

`defusedxml` 是 stdlib XML 模块的安全替代，**API 完全一致**，禁用了外部实体和实体展开。改一行 import 的成本，换掉一整类漏洞。

#### 自动化脚本的其他安全要点

| 要点 | 说明 |
|:--|:--|
| **凭据不写死在代码里** | 用环境变量、`ansible-vault`、HashiCorp Vault、AWS Secrets Manager |
| **脚本用的账号最小权限** | 只读采集就用只读账号，不要一律 privilege 15 |
| **代码进 Git 前扫一遍** | `git-secrets`、`truffleHog` 检查有没有误提交密码 |
| **审计日志** | 自动化操作也要记录到 TACACS+ 计费，能追溯"是哪个脚本改的" |
| **限制脚本的来源 IP** | 设备侧用 `access-class` 只允许自动化服务器访问 |

```python
# ✅ 凭据从环境变量读
import os
username = os.environ["NET_USER"]
password = os.environ["NET_PASS"]        # 不要写死！
```

```yaml
# ✅ Ansible 用 vault 加密
# ansible-vault encrypt_string 'MyPassword' --name 'ansible_password'
ansible_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  62313365396662343061393464336163383734...
```

---

## ⑤ Ansible

### 5.1 核心概念

| 概念 | 说明 |
|:--|:--|
| **Inventory** | 设备清单（哪些设备，怎么分组） |
| **Playbook** | 任务剧本（YAML 格式） |
| **Module** | 具体执行的功能模块（`ios_config`、`ios_facts`） |
| **Task** | 单个任务 |
| **Role** | 可复用的任务集合 |
| **Variables** | 变量（设备特有的参数） |
| **Template** | Jinja2 模板（生成配置） |
| **Idempotency** | ★ **幂等性**：执行多次结果相同 |

**★ 幂等性是 Ansible 的核心价值**：
```
   第一次运行：配置被添加（changed=1）
   第二次运行：发现已存在，不做任何事（changed=0, ok=1）
```
这意味着**你可以放心地反复运行同一个 playbook**，不用担心重复配置或破坏现有状态。

### 5.2 Inventory

```yaml
# inventory.yml
all:
  children:
    routers:
      hosts:
        R1:
          ansible_host: 10.0.0.1
        R2:
          ansible_host: 10.0.0.2
      vars:
        ansible_network_os: ios
    switches:
      hosts:
        SW1:
          ansible_host: 10.0.0.11
        SW2:
          ansible_host: 10.0.0.12
      vars:
        ansible_network_os: ios
  vars:
    ansible_connection: network_cli
    ansible_user: admin
    ansible_password: "{{ vault_password }}"      # 用 ansible-vault 加密
    ansible_become: yes
    ansible_become_method: enable
```

### 5.3 Playbook 示例

```yaml
---
# ntp.yml - 批量配置 NTP
- name: 配置全网 NTP
  hosts: all
  gather_facts: no
  connection: network_cli

  vars:
    ntp_servers:
      - 10.1.30.61
      - 10.1.30.62
    timezone_name: CST
    timezone_offset: 8

  tasks:
    - name: 配置 NTP 服务器
      cisco.ios.ios_config:
        lines:
          - "ntp server {{ item }}"
        # ★ Ansible 会自动检查是否已存在，幂等
      loop: "{{ ntp_servers }}"

    - name: 配置时区
      cisco.ios.ios_config:
        lines:
          - "clock timezone {{ timezone_name }} {{ timezone_offset }}"

    - name: 配置日志时间戳
      cisco.ios.ios_config:
        lines:
          - "service timestamps log datetime msec localtime show-timezone"
          - "service timestamps debug datetime msec localtime show-timezone"

    - name: 保存配置
      cisco.ios.ios_config:
        save_when: modified                       # ★ 只在有变更时保存

    - name: 验证 NTP 同步状态
      cisco.ios.ios_command:
        commands:
          - show ntp status
      register: ntp_result

    - name: 显示结果
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }}: {{ 'SYNCED' if 'synchronized' in ntp_result.stdout[0] else 'NOT SYNCED' }}"
```

**配置备份**：
```yaml
---
- name: 备份所有设备配置
  hosts: all
  gather_facts: no

  tasks:
    - name: 采集运行配置
      cisco.ios.ios_config:
        backup: yes
        backup_options:
          filename: "{{ inventory_hostname }}-{{ lookup('pipe', 'date +%Y%m%d') }}.cfg"
          dir_path: ./backups
```

**基于模板批量生成配置**：
```yaml
---
- name: 用模板配置接口
  hosts: switches
  gather_facts: no

  tasks:
    - name: 下发接口配置
      cisco.ios.ios_config:
        src: templates/access_port.j2
```

```jinja
{# templates/access_port.j2 #}
{% for port in access_ports %}
interface {{ port.name }}
 description ### {{ port.desc }} ###
 switchport mode access
 switchport access vlan {{ port.vlan }}
 switchport nonegotiate
 spanning-tree portfast
 spanning-tree bpduguard enable
 switchport port-security
 switchport port-security maximum 3
 switchport port-security violation restrict
 no shutdown
{% endfor %}
```

**执行**：
```bash
# 检查语法
ansible-playbook -i inventory.yml ntp.yml --syntax-check

# ★ 预演（dry-run，不实际改配置）
ansible-playbook -i inventory.yml ntp.yml --check --diff

# 实际执行
ansible-playbook -i inventory.yml ntp.yml

# 只对某一组执行
ansible-playbook -i inventory.yml ntp.yml --limit switches

# 只对一台执行（先测试）
ansible-playbook -i inventory.yml ntp.yml --limit SW1
```

> **★ `--check --diff` 是生产环境的必备习惯**：先预演，看清楚会改什么，确认无误再实际执行。

### 5.4 Ansible vs Python 脚本

| | **Ansible** | **Python (Netmiko/Nornir)** |
|:--|:--|:--|
| 学习曲线 | ★ 平缓（YAML，不用写代码） | 陡（要会 Python） |
| 幂等性 | ★ **内置** | 要自己实现 |
| 并发 | 内置（forks） | 要自己写（threading/asyncio） |
| 灵活性 | 受模块限制 | ★ **无限** |
| 复杂逻辑 | 难（YAML 表达能力有限） | ★ 容易 |
| 性能 | 较慢（每个 task 一次连接） | ★ 快（可复用连接） |
| 适合 | **标准化的配置下发** | **复杂的采集、分析、判断** |

**实践建议**：
- **配置下发、合规检查** → Ansible
- **数据采集、分析、复杂逻辑** → Python
- **两者结合**：Python 生成数据 → Ansible 下发

---

## ⑥ Cisco 平台的自动化能力

### 6.1 DNA Center（园区网）

**Cisco 的园区网控制器，提供完整的 REST API。**

```python
import requests
from requests.auth import HTTPBasicAuth
import urllib3
urllib3.disable_warnings()

DNAC = "https://dnac.example.com"

# ① 获取 Token（有效期 1 小时）
r = requests.post(f"{DNAC}/dna/system/api/v1/auth/token",
                  auth=HTTPBasicAuth("admin", "password"), verify=False)
token = r.json()["Token"]

headers = {"X-Auth-Token": token, "Content-Type": "application/json"}

# ② 获取所有网络设备
r = requests.get(f"{DNAC}/dna/intent/api/v1/network-device",
                 headers=headers, verify=False)
for dev in r.json()["response"]:
    print(f"{dev['hostname']:20} {dev['managementIpAddress']:16} "
          f"{dev['platformId']:20} {dev['softwareVersion']}")

# ③ 获取设备健康状况
r = requests.get(f"{DNAC}/dna/intent/api/v1/device-health",
                 headers=headers, verify=False)

# ④ 路径追踪（排障神器）
payload = {"sourceIP": "10.1.10.55", "destIP": "10.1.30.100", "protocol": "tcp"}
r = requests.post(f"{DNAC}/dna/intent/api/v1/flow-analysis",
                  headers=headers, json=payload, verify=False)
```

### 6.2 模型驱动遥测（Model-Driven Telemetry, MDT）

**取代 SNMP 轮询的现代方案**：

| | **SNMP 轮询** | **MDT（Telemetry）** |
|:--|:--|:--|
| 模式 | **拉（Pull）** —— NMS 主动问 | **推（Push）** —— 设备主动报 |
| 频率 | 受轮询间隔限制（通常 5 分钟） | **秒级甚至亚秒级** |
| 效率 | 低（每次都是完整请求-响应） | ★ **高**（只推变化的数据） |
| 扩展性 | 差（设备多了 NMS 扛不住） | ★ **好** |
| 数据模型 | MIB | **YANG** |

```cisco
! ── 配置遥测订阅 ──
R1(config)# telemetry ietf subscription 101
R1(config-mdt-subs)#  encoding encode-kvgpb
R1(config-mdt-subs)#  filter xpath /interfaces-ios-xe-oper:interfaces/interface/statistics
R1(config-mdt-subs)#  source-address 10.0.0.1
R1(config-mdt-subs)#  stream yang-push
R1(config-mdt-subs)#  update-policy periodic 1000          ! 每 10 秒推一次（单位 厘秒）
R1(config-mdt-subs)#  receiver ip address 10.1.30.240 57500 protocol grpc-tcp

R1# show telemetry ietf subscription all
R1# show telemetry ietf subscription 101 detail
```

**典型技术栈**：设备 → gRPC/gNMI → Telegraf → InfluxDB → Grafana

### 6.3 Cisco 其他平台的 API

| 平台 | 用途 | API |
|:--|:--|:--|
| **DNA Center** | 园区网 | REST |
| **vManage** | SD-WAN | REST |
| **ACI APIC** | 数据中心 | REST |
| **ISE** | 身份策略 | REST (ERS) |
| **Meraki Dashboard** | 云管理 | REST |
| **Webex** | 协作 | REST |

**共同点**：都是 **REST + JSON + Token 认证**。会了一个，其他的都是查文档的事。

---

## ⑦ 配套实验：从手工到自动化

### 实验 A：用 RESTCONF 读取和修改配置

**Step 1：设备端启用**
```cisco
R1(config)# ip http secure-server
R1(config)# restconf
R1(config)# username api-user privilege 15 secret ApiPass123
R1(config)# aaa new-model
R1(config)# aaa authentication login default local
R1(config)# aaa authorization exec default local
```

**Step 2：验证服务**
```cisco
R1# show platform software yang-management process
confd            : Running
nesd             : Running
syncfd           : Running
ncsshd           : Running
dmiauthd         : Running
nginx            : Running                      ← RESTCONF 依赖 nginx
ndbmand          : Running
pubd             : Running
```

**Step 3：curl 测试**
```bash
curl -k -u api-user:ApiPass123 \
  -X GET \
  -H "Accept: application/yang-data+json" \
  https://10.0.0.1/restconf/data/ietf-interfaces:interfaces \
  | python -m json.tool
```

**预期输出**：
```json
{
  "ietf-interfaces:interfaces": {
    "interface": [
      {
        "name": "GigabitEthernet1",
        "description": "MANAGEMENT",
        "type": "iana-if-type:ethernetCsmacd",
        "enabled": true,
        "ietf-ip:ipv4": {
          "address": [{"ip": "10.0.0.1", "netmask": "255.255.255.0"}]
        }
      }
    ]
  }
}
```

**Step 4：创建 Loopback**
```bash
curl -k -u api-user:ApiPass123 \
  -X PUT \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "ietf-interfaces:interface": {
      "name": "Loopback100",
      "description": "Created via RESTCONF",
      "type": "iana-if-type:softwareLoopback",
      "enabled": true,
      "ietf-ip:ipv4": {
        "address": [{"ip": "100.100.100.100", "netmask": "255.255.255.255"}]
      }
    }
  }' \
  "https://10.0.0.1/restconf/data/ietf-interfaces:interfaces/interface=Loopback100" \
  -w "\nHTTP Status: %{http_code}\n"
```

**Step 5：在设备上验证**
```cisco
R1# show ip interface brief | include Loopback100
Loopback100    100.100.100.100  YES other  up      up

R1# show running-config interface Loopback100
interface Loopback100
 description Created via RESTCONF
 ip address 100.100.100.100 255.255.255.255
```

**✅ 配置真的通过 API 生效了。**

### 实验 B：Python 批量巡检

```python
#!/usr/bin/env python3
"""网络巡检脚本：检查关键项并输出报告"""
from netmiko import ConnectHandler
from datetime import datetime
import json

DEVICES = [
    {"device_type": "cisco_ios", "host": "10.0.0.1", "username": "admin", "password": "pass"},
    {"device_type": "cisco_ios", "host": "10.0.0.2", "username": "admin", "password": "pass"},
]

# 巡检项：(名称, 命令, 判断函数)
CHECKS = [
    ("NTP同步",   "show ntp status",        lambda o: "synchronized" in o and "unsynchronized" not in o),
    ("CPU负载",   "show processes cpu | include five minutes",
                                            lambda o: int(o.split("five minutes: ")[1].split("%")[0]) < 70 if "five minutes:" in o else False),
    ("配置已保存", "show running-config | include ^!",  lambda o: True),   # 占位
    ("STP根桥",   "show spanning-tree root", lambda o: "This bridge is the root" in o),
    ("接口错误",   "show interfaces | include CRC",
                                            lambda o: all(int(l.split()[0]) == 0 for l in o.splitlines() if l.strip() and l.split()[0].isdigit())),
]

def inspect(device):
    host = device["host"]
    report = {"host": host, "timestamp": datetime.now().isoformat(), "checks": {}}
    try:
        with ConnectHandler(**device) as conn:
            report["hostname"] = conn.find_prompt().strip("#>")
            for name, cmd, judge in CHECKS:
                output = conn.send_command(cmd)
                try:
                    passed = judge(output)
                except Exception:
                    passed = None
                report["checks"][name] = {
                    "passed": passed,
                    "output": output[:200],
                }
    except Exception as e:
        report["error"] = str(e)
    return report

if __name__ == "__main__":
    reports = [inspect(d) for d in DEVICES]

    # 输出人类可读的摘要
    print(f"{'设备':<18}{'主机名':<15}" + "".join(f"{n:<12}" for n, _, _ in CHECKS))
    print("-" * 90)
    for r in reports:
        if "error" in r:
            print(f"{r['host']:<18}连接失败: {r['error']}")
            continue
        line = f"{r['host']:<18}{r.get('hostname',''):<15}"
        for name, _, _ in CHECKS:
            p = r["checks"].get(name, {}).get("passed")
            line += f"{'✅' if p else ('❌' if p is False else '⚠️'):<12}"
        print(line)

    with open(f"inspect-{datetime.now():%Y%m%d}.json", "w", encoding="utf-8") as f:
        json.dump(reports, f, indent=2, ensure_ascii=False)
```

### 实验 C：Ansible 批量配置

```bash
# 目录结构
automation/
├── inventory.yml
├── group_vars/
│   ├── all.yml
│   └── switches.yml
├── templates/
│   └── access_port.j2
└── playbooks/
    ├── ntp.yml
    ├── backup.yml
    └── compliance.yml
```

```yaml
# playbooks/compliance.yml —— 合规性检查
---
- name: 网络配置合规性检查
  hosts: all
  gather_facts: no

  vars:
    required_config:
      - "no ip domain-lookup"
      - "service timestamps log datetime msec"
      - "logging host 10.1.30.210"
      - "ntp server 10.1.30.61"

  tasks:
    - name: 获取运行配置
      cisco.ios.ios_command:
        commands: show running-config
      register: running

    - name: 检查必需配置项
      ansible.builtin.assert:
        that:
          - item in running.stdout[0]
        fail_msg: "❌ {{ inventory_hostname }} 缺少配置: {{ item }}"
        success_msg: "✅ {{ item }}"
      loop: "{{ required_config }}"
      ignore_errors: yes
      register: compliance

    - name: 生成合规报告
      ansible.builtin.debug:
        msg: >-
          {{ inventory_hostname }}:
          通过 {{ compliance.results | selectattr('failed','equalto',false) | list | length }}/
          {{ required_config | length }}
```

```bash
# 先预演
ansible-playbook -i inventory.yml playbooks/ntp.yml --check --diff --limit SW1

# 单台测试
ansible-playbook -i inventory.yml playbooks/ntp.yml --limit SW1

# 全网执行
ansible-playbook -i inventory.yml playbooks/ntp.yml
```

---

## ⑧ 考点提示 + 自测题

### 考点

- **JSON / XML / YAML 的识别**
- **NETCONF (SSH 830, XML) vs RESTCONF (HTTPS 443, JSON/XML)**
- **HTTP 方法与状态码**（尤其 **401 vs 403**）
- **YANG 数据模型的节点类型**（container/list/leaf）
- **开放模型 vs 原生模型**
- **Ansible 的幂等性**
- **Puppet/Chef（拉模式）vs Ansible/SaltStack（推模式）**
- **MDT（推）vs SNMP（拉）**

### 自测题

**1.** NETCONF 和 RESTCONF 有什么区别？分别用什么端口和数据格式？

<details><summary>答案</summary>

| | **NETCONF** | **RESTCONF** |
|:--|:--|:--|
| 标准 | **RFC 6241** | **RFC 8040** |
| 传输 | **SSH，端口 830** | **HTTPS，端口 443** |
| 数据格式 | **XML** | **JSON 或 XML** |
| 数据模型 | **YANG** | **YANG** |
| 操作方式 | RPC 操作：`<get>` `<get-config>` `<edit-config>` `<commit>` | **标准 HTTP 动词**：GET/POST/PUT/PATCH/DELETE |
| **事务支持** | ✅ **candidate + validate + commit + rollback** | ❌ 有限 |
| 学习曲线 | 陡（XML 复杂、RPC 概念多） | ★ 平缓（就是普通的 REST） |
| 工具生态 | ncclient | curl、requests、Postman 等所有 HTTP 工具 |

**NETCONF 的核心优势：事务性（★ 最重要的区别）**

```
   ① edit-config 到 candidate 数据存储（草稿区，还没生效）
   ② validate（校验语法和语义）
   ③ commit（一次性全部生效）
   ④ 有问题 → discard-changes 或 rollback
   
   ★ 要么全成功，要么全不生效 ★
```

**为什么重要**：假设你要下发 10 条配置。用 RESTCONF/CLI 的话，如果第 5 条失败了，**前 4 条已经生效了，设备处于中间状态**——可能是不可用的状态（比如改了 ACL 只改了一半）。

NETCONF 的 candidate + commit 保证了原子性。

**RESTCONF 的优势：简单**

```bash
# 就是普通的 HTTP 请求，任何工具都能发
curl -k -u user:pass \
  -H "Accept: application/yang-data+json" \
  https://10.0.0.1/restconf/data/ietf-interfaces:interfaces
```

不需要学 XML、不需要学 RPC、不需要特殊的客户端库。

**启用**：
```cisco
! NETCONF
R1(config)# netconf-yang
R1(config)# netconf-yang feature candidate-datastore

! RESTCONF
R1(config)# ip http secure-server
R1(config)# restconf

! 两者都需要 AAA
R1(config)# aaa new-model
R1(config)# aaa authentication login default local
R1(config)# aaa authorization exec default local
R1(config)# username api-user privilege 15 secret Pass123
```

**验证**：
```cisco
R1# show netconf-yang sessions
R1# show netconf-yang datastores
R1# show platform software yang-management process
! 所有进程都应该是 Running
```

**选型建议**：
| 场景 | 推荐 |
|:--|:--|
| 简单的读写操作、快速原型 | **RESTCONF** |
| 复杂的多步配置，需要原子性 | **NETCONF** |
| 团队 Python 基础弱 | RESTCONF |
| 需要 rollback 能力 | NETCONF |
</details>

**2.** HTTP 状态码 401 和 403 有什么区别？

<details><summary>答案</summary>

| 码 | 名称 | 含义 | 通俗理解 |
|:--|:--|:--|:--|
| **401** | **Unauthorized** | **未认证** | "**你是谁？**" —— 你没提供凭据，或凭据无效 |
| **403** | **Forbidden** | **已认证但无权限** | "**我知道你是谁，但你不能做这个**" |

**401 的典型原因**：
- 没有提供 `Authorization` 头
- 用户名/密码错误
- Token 过期或无效
- 认证方式不对（比如服务器要 Bearer Token，你发的是 Basic Auth）

```bash
# 没提供凭据
curl -k https://10.0.0.1/restconf/data/ietf-interfaces:interfaces
# → 401 Unauthorized

# 密码错了
curl -k -u api-user:WrongPassword https://10.0.0.1/restconf/...
# → 401 Unauthorized
```

**403 的典型原因**：
- 认证成功了，但这个用户的**权限级别不够**（比如只有 privilege 1，试图修改配置）
- ACL 限制了来源 IP
- 试图访问不属于自己的资源
- RBAC 策略拒绝

```bash
# 用只读账号试图修改配置
curl -k -u readonly-user:CorrectPassword \
  -X PUT \
  -d '{"...": "..."}' \
  https://10.0.0.1/restconf/data/...
# → 403 Forbidden
```

**排障流程**：
```
   收到 401
        ↓
   ① 检查用户名密码是否正确
   ② 检查 Token 是否过期（DNA Center 的 Token 只有 1 小时）
   ③ 检查认证头格式（Basic vs Bearer）
   ④ 在设备上确认账号存在：show running-config | include username

   收到 403
        ↓
   ① 检查用户的 privilege level（RESTCONF 通常需要 15）
   ② 检查 AAA 授权配置：aaa authorization exec default local
   ③ 检查 ACL 是否限制了来源 IP
   ④ 检查请求的操作是否被允许（只读账号不能 PUT/POST/DELETE）
```

**其他常见状态码**：

| 码 | 含义 | 网络自动化中的典型场景 |
|:--|:--|:--|
| **200 OK** | 成功 | GET 读取成功 |
| **201 Created** | 创建成功 | POST/PUT 创建了新资源 |
| **204 No Content** | 成功但无返回体 | PUT 更新成功、DELETE 成功 |
| **400 Bad Request** | 请求格式错 | JSON 语法错、必填字段缺失 |
| **404 Not Found** | 资源不存在 | **YANG 模块名写错**、接口名不存在 |
| **405 Method Not Allowed** | 方法不支持 | 对只读资源用 POST |
| **409 Conflict** | 冲突 | 资源已存在（用 POST 创建已有资源） |
| **500 Internal Server Error** | 服务器错误 | 设备侧异常，看设备日志 |

**404 在网络自动化中特别常见**：
```bash
# 模块名写错
https://10.0.0.1/restconf/data/ietf-interface:interfaces      ← 少了 s
                                            ↑ 404

# 正确
https://10.0.0.1/restconf/data/ietf-interfaces:interfaces

# 查看设备支持哪些模块
curl -k -u user:pass https://10.0.0.1/restconf/data/ietf-yang-library:modules-state
```
</details>

**3.** 什么是 Ansible 的幂等性？为什么它很重要？

<details><summary>答案</summary>

**幂等性（Idempotency）= 执行一次和执行多次的结果相同。**

```
   第一次运行 playbook：
   TASK [配置 NTP] ***********
   changed: [SW1]                    ← 配置被添加了
   PLAY RECAP: SW1 : ok=1 changed=1

   第二次运行同一个 playbook：
   TASK [配置 NTP] ***********
   ok: [SW1]                         ← ★ 发现已存在，什么都没做
   PLAY RECAP: SW1 : ok=1 changed=0
```

**为什么重要**：

**① 可以放心地反复运行**
你不需要记住"这台设备配过了没有"。直接跑，Ansible 自己判断。这让自动化脚本可以**定时运行**（比如每天跑一次确保配置正确）。

**② 声明式而非命令式**
```
   命令式（传统脚本）：
   "执行这些步骤" → 要自己处理"如果已经存在怎么办"
   
   声明式（Ansible）：
   ★ "我要的最终状态是这样" ★ → Ansible 负责让现状变成目标状态
```

这是个思维方式的转变。你描述"期望的状态"，工具负责"如何达到"。

**③ 配置漂移检测**
```bash
ansible-playbook site.yml --check --diff
```
预演模式会告诉你**哪些设备的配置偏离了标准**（`changed` 的就是有偏差的），但不实际修改。这是**合规性审计**的利器。

**④ 安全性**
不会因为重复执行而产生副作用（比如把 ACL 规则加两遍、把 VLAN 列表覆盖掉）。

**Ansible 如何实现幂等**：

`ios_config` 模块的工作方式：
```
   ① 先读取设备当前的 running-config
   ② 与 playbook 里声明的 lines 做对比
   ③ 只下发【缺失的】那些行
   ④ 如果全部已存在 → changed=0，什么都不做
```

**⚠️ 注意：不是所有模块都天然幂等**

```yaml
# ❌ ios_command 不幂等（它只是执行命令，不做状态判断）
- name: 重启设备
  cisco.ios.ios_command:
    commands: reload
# 每次运行都会重启！

# ✅ ios_config 是幂等的
- name: 配置 NTP
  cisco.ios.ios_config:
    lines:
      - ntp server 10.1.30.61
```

**自己写 Python 时也要实现幂等**：
```python
def configure_ntp(conn):
    # ★ 先检查
    current = conn.send_command("show running-config | include ^ntp server")
    if "10.1.30.61" in current:
        return "already configured"          # 幂等：已存在就跳过
    
    # 再配置
    conn.send_config_set(["ntp server 10.1.30.61"])
    return "configured"
```

**幂等性的边界情况（需要注意）**：

| 情况 | 问题 |
|:--|:--|
| 命令的顺序有意义（如 ACL） | Ansible 可能只加缺失的行，导致顺序错乱 |
| 配置有多种等价写法 | 设备显示 `ip route 10.0.0.0 255.0.0.0 X`，你写 `ip route 10.0.0.0/8 X` → 判断为不存在 |
| 需要"删除多余配置" | 默认只加不删，需要用 `replace: block` |

**处理 ACL 这类有序配置**：
```yaml
- name: 配置 ACL（整块替换）
  cisco.ios.ios_config:
    lines:
      - permit tcp any host 10.0.0.100 eq 80
      - permit tcp any host 10.0.0.100 eq 443
      - deny ip any any log
    parents: ip access-list extended WEB-ACL
    before: no ip access-list extended WEB-ACL      # ★ 先删掉再重建
    match: exact
    replace: block
```
</details>

**4.** SNMP 轮询和模型驱动遥测（MDT）有什么区别？

<details><summary>答案</summary>

| | **SNMP 轮询** | **MDT（Model-Driven Telemetry）** |
|:--|:--|:--|
| 模式 | **拉（Pull）** —— NMS 主动问 | **推（Push）** —— 设备主动报 |
| 触发 | NMS 定时发 GET 请求 | 设备按周期或**事件驱动**推送 |
| 频率 | 受轮询间隔限制（通常 **1-5 分钟**） | **秒级甚至亚秒级** |
| 数据模型 | **MIB**（OID 数字串，可读性差） | **YANG**（结构化，语义清晰） |
| 传输 | UDP 161 | **gRPC / gNMI / NETCONF** |
| 编码 | BER | **GPB (Protocol Buffers) / JSON** |
| 效率 | 低（每次完整请求-响应，大量冗余） | ★ **高**（只推变化的数据） |
| 扩展性 | **差**（设备多了 NMS 成为瓶颈） | ★ **好**（设备分担了工作） |

**SNMP 轮询的三个根本问题**：

**① 时间粒度太粗**
5 分钟轮询一次，意味着：
- 一次 30 秒的流量突发**完全看不到**（被平均掉了）
- 故障发生后最多 5 分钟才发现
- 微突发（microburst）导致的丢包完全无法诊断

**② NMS 成为瓶颈**
```
   1000 台设备 × 每台 50 个接口 × 每分钟轮询
   = 每分钟 50,000 次 SNMP 请求
   → NMS 的 CPU 和网络成为瓶颈
   → 只能拉长轮询间隔 → 时间粒度更粗
```

**③ 数据模型落后**
MIB 是 OID 数字串（`1.3.6.1.2.1.2.2.1.10`），可读性极差，扩展新数据类型很麻烦。

**MDT 的解法**：

```
   设备主动推送
        ↓
   ① 不需要 NMS 轮询 → NMS 不再是瓶颈
   ② 可以做到秒级/亚秒级
   ③ 可以是【事件驱动】的（状态变化立即推送，而不是等下次轮询）
   ④ 用 YANG 模型，数据自描述
```

**配置**：
```cisco
R1(config)# telemetry ietf subscription 101
R1(config-mdt-subs)#  encoding encode-kvgpb                    ! Google Protocol Buffers
R1(config-mdt-subs)#  filter xpath /interfaces-ios-xe-oper:interfaces/interface/statistics
R1(config-mdt-subs)#  source-address 10.0.0.1
R1(config-mdt-subs)#  stream yang-push
R1(config-mdt-subs)#  update-policy periodic 1000              ! 10 秒（单位厘秒）
R1(config-mdt-subs)#  receiver ip address 10.1.30.240 57500 protocol grpc-tcp

! 事件驱动（状态变化才推）
R1(config-mdt-subs)#  update-policy on-change
```

**两种订阅模式**：

| 模式 | 说明 | 用途 |
|:--|:--|:--|
| **periodic** | 按固定周期推送 | 流量统计、CPU/内存 |
| **on-change** | **状态变化时才推** | 接口 up/down、BGP 邻居状态、路由变化 |

**`on-change` 是 MDT 最有价值的能力**——接口 down 的那一瞬间就推送，而不是等到下一个轮询周期。

**典型技术栈**：
```
   [网络设备] --gRPC--> [Telegraf] --> [InfluxDB] --> [Grafana]
                            ↑              ↑            ↑
                        采集器          时序数据库      可视化
   
   或者：
   [网络设备] --gNMI--> [Kafka] --> [Elasticsearch] --> [Kibana]
```

**现实中的选择**：

**SNMP 不会消失**，因为：
- 生态成熟，所有设备都支持
- 老设备不支持 MDT
- 简单场景够用

**实践建议**：
| 场景 | 用什么 |
|:--|:--|
| 老设备、基础监控 | SNMP |
| 新设备、需要高精度 | **MDT** |
| 大规模（>500 台） | **MDT**（SNMP 扛不住） |
| 需要秒级故障发现 | **MDT (on-change)** |
| 流量成分分析 | **NetFlow**（两者都不能替代） |

**三者是互补的**：
- **MDT/SNMP** → 设备状态和性能指标
- **NetFlow** → 流量成分（谁在用带宽）
- **Syslog** → 事件记录
</details>

**5.** 你要给 200 台交换机加一条 NTP 配置。用 Ansible 还是 Python？为什么？

<details><summary>答案</summary>

**这个场景用 Ansible。**

**理由**：

**① 任务是"标准化配置下发"，正是 Ansible 的强项**
```yaml
- name: 配置 NTP
  hosts: all
  gather_facts: no
  tasks:
    - name: 下发 NTP 配置
      cisco.ios.ios_config:
        lines:
          - ntp server 10.1.30.61 prefer
          - ntp server 10.1.30.62
          - clock timezone CST 8
        save_when: modified
```
**十几行 YAML 就搞定，不用写一行代码。**

**② 内置幂等性**
不需要自己写"检查是否已配置"的逻辑。已经配过的设备会显示 `ok`（changed=0），没配的显示 `changed`。

**③ 内置并发**
```bash
ansible-playbook -i inventory.yml ntp.yml -f 50    # 50 台并发
```
Python 要自己写 threading 或 asyncio。

**④ 内置预演（`--check --diff`）**
```bash
ansible-playbook -i inventory.yml ntp.yml --check --diff --limit SW1
```
**先看会改什么，确认无误再执行。** 这在生产环境是必备的安全措施。

**⑤ 有清晰的执行报告**
```
PLAY RECAP ***********************************************
SW1  : ok=3  changed=1  unreachable=0  failed=0
SW2  : ok=3  changed=0  unreachable=0  failed=0    ← 已配过
SW3  : ok=0  changed=0  unreachable=1  failed=0    ← 连不上
...
```
一眼看出哪些成功、哪些跳过、哪些失败。

**⑥ 团队协作友好**
YAML 可读性好，不会 Python 的同事也能看懂和维护。可以进 Git 做版本管理和 Code Review。

---

**什么时候用 Python**：

| 场景 | 为什么 Ansible 不够 |
|:--|:--|
| **复杂的判断逻辑** | "如果接口有 CRC 错误且错误率 > 阈值，就把它 shutdown 并发告警"—— YAML 表达这种逻辑很痛苦 |
| **数据采集与分析** | 采集所有设备的 MAC 表，找出重复的 MAC —— 需要跨设备的数据处理 |
| **对接外部系统** | 从 CMDB 拉设备清单，处理完把结果写回工单系统 |
| **性能敏感** | Ansible 每个 task 都可能重新建连接，Python (Nornir) 可以复用连接，快得多 |
| **自定义解析** | 解析非标准的 show 命令输出 |

**Python 示例（复杂逻辑）**：
```python
# 这种逻辑用 Ansible 很难表达
for device in devices:
    with ConnectHandler(**device) as conn:
        interfaces = conn.send_command("show interfaces", use_textfsm=True)
        for intf in interfaces:
            crc = int(intf.get("crc", 0))
            packets = int(intf.get("input_packets", 1))
            error_rate = crc / packets if packets else 0
            
            if error_rate > 0.001:                    # 错误率 > 0.1%
                print(f"⚠️ {device['host']} {intf['interface']} 错误率 {error_rate:.2%}")
                # 发告警到企业微信/飞书
                send_alert(device, intf, error_rate)
                # 如果错误率极高，自动 shutdown
                if error_rate > 0.05:
                    conn.send_config_set([
                        f"interface {intf['interface']}",
                        "shutdown"
                    ])
```

---

**最佳实践：两者结合**

```
   ① Python 脚本从 CMDB/Excel 生成 Ansible inventory 和变量
        ↓
   ② Ansible 负责标准化的配置下发
        ↓
   ③ Python 脚本做验证、采集结果、生成报告、推送通知
```

**完整的自动化工作流**：
```bash
# 1. Python 生成 inventory
python generate_inventory.py --from-cmdb > inventory.yml

# 2. Ansible 预演
ansible-playbook -i inventory.yml ntp.yml --check --diff | tee dryrun.log

# 3. 人工审核 dryrun.log

# 4. 灰度执行（先 5 台）
ansible-playbook -i inventory.yml ntp.yml --limit 'all[0:4]'

# 5. 验证
python verify_ntp.py --devices inventory.yml

# 6. 全量执行
ansible-playbook -i inventory.yml ntp.yml

# 7. 生成报告
python generate_report.py --output report.html
```

**★ 无论用什么工具，生产环境的铁律**：
1. **先在 1 台设备上测试**
2. **用 `--check --diff` 预演**
3. **灰度推进**（5 台 → 50 台 → 全量）
4. **每一步都验证**
5. **准备回滚方案**
</details>

---

**上一章** ← [13 网络安全](13-网络安全-AAA-802.1X-控制平面保护.md) ｜ **下一章** → [15 SD-Access 与 SD-WAN](15-SD-Access与SD-WAN.md)
