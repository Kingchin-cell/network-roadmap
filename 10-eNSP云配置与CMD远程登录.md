# 10 · eNSP 云配置与 CMD 远程登录路由器（实测可用）

> **目标**：在本机 CMD 里 `telnet` 到 eNSP 里的路由器，像老师那样多窗口操作。
>
> **这份方案是 2026-11-24 在 Win11 24H2 + eNSP 1.3 + VirtualBox 5.2.44 上实测打通的**，
> 中间踩过 VirtualBox Host-Only 不通的坑，最后用 **VMware VMnet1** 解决。

---

## 一、最终可用配置（结论先看）

| 项目 | 值 |
|---|---|
| **云端口 1** | `UDP`（Internal）← 连路由器 |
| **云端口 2** | `VMware Network Adapter VMnet1`（Public，IP = 192.168.95.1） |
| **端口映射** | `1→2` 和 `2→1`，**两条都勾「双向通道」** |
| **路由器接口** | `GigabitEthernet0/0/0` → `192.168.95.10/24` |
| **本机验证** | `ping 192.168.95.10` |
| **远程登录** | `telnet 192.168.95.10`，账号密码见下 |

---

## 二、完整步骤

### 第 1 步：拖一个云（Cloud）

```
eNSP 左侧设备栏 → Cloud → 拖到画布
```

### 第 2 步：配置云端口（注意顺序）

**双击云 → 打开配置窗口**

**① 先加 UDP 端口（对内）**
```
绑定信息：   UDP
端口类型：   Ethernet
监听端口：   30000（默认）
   ↓
点【增加】
```
→ 生成 **端口 1**，状态 `Internal`

**② 再加虚拟网卡端口（对外）**
```
绑定信息：   VMware Network Adapter VMnet1
端口类型：   Ethernet
端口号：     2
   ↓
点【增加】
```
→ 生成 **端口 2**，状态 `Public`

> ⚠️ **为什么用 VMware VMnet1 而不是 VirtualBox Host-Only？**
> 在本机上实测：VirtualBox Host-Only 网卡**收发严重不平衡**（发送 11.5 GB / 接收 224 KB），
> 打不通。VMware 的 VMnet1 一次就通。
> 如果你机器上 VirtualBox Host-Only 正常，也可以用它（IP 换成 192.168.56.1，路由器配 192.168.56.x）。

### 第 3 步：配置端口映射（关键，必须两条）

```
第一条：
  端口类型：    Ethernet
  入端口编号：  1
  出端口编号：  2
  ☑ 双向通道
  → 点【增加】

第二条：
  端口类型：    Ethernet
  入端口编号：  2
  出端口编号：  1
  ☑ 双向通道
  → 点【增加】
```

**「端口映射表」里应该有 2 行：**
```
| No. | 入端口编号 | 出端口编号 | 端口类型 |
|  1  |     1      |     2      | Ethernet |
|  2  |     2      |     1      | Ethernet |
```

**→ 点「应用」→ 关闭**

> 💡 只配一条会单向不通（能发不能收），表现就是 ping 全部超时。

### 第 4 步：启动云 ⚠️

```
选中云 → 右键 → 启动（或点上方绿色 ▶）
云的图标要变绿
```

**云不启动 = 不转发任何流量。**

### 第 5 步：连线

```
用「Auto」或「Electrical」线：
  路由器 GigabitEthernet0/0/0  ←→  云的 Ethernet 1（UDP/Internal 那个）
```

> ⚠️ **必须连 Internal（UDP）端口**，不能连 Public（网卡）端口。
> 因为 Public 端口的使命是通往本机，不是接设备。

### 第 6 步：路由器配 IP

```bash
<Huawei> system-view
[Huawei] sysname R1

[R1] interface GigabitEthernet0/0/0
[R1-GigabitEthernet0/0/0] ip address 192.168.95.10 24
[R1-GigabitEthernet0/0/0] quit
```

**验证接口起来了：**
```bash
[R1] display ip interface brief
```
**必须是：**
```
GigabitEthernet0/0/0    192.168.95.10/24    up    up
```
> `up up` 才算好。如果是 `down down` → 云没启动或连线不对。
> 如果是 `*down` → 被 shutdown 了，用 `undo shutdown`。

### 第 7 步：本机验证

```cmd
ping 192.168.95.10
```

**通了再往下走。不通先排查（见第五节）。**

### 第 8 步：路由器配置 telnet

```bash
[R1] telnet server enable

[R1] user-interface vty 0 4
[R1-ui-vty0-4] authentication-mode password
[R1-ui-vty0-4] set authentication password simple Huawei123
[R1-ui-vty0-4] user privilege level 15
[R1-ui-vty0-4] protocol inbound telnet
[R1-ui-vty0-4] quit
[R1] quit
<R1> save
```

**⚠️ 三条不能少：**
| 命令 | 不加的后果 |
|---|---|
| `telnet server enable` | 连接被拒绝 |
| `user privilege level 15` | 登录后只能看不能配 |
| `protocol inbound telnet` | 协议被挡 |

**验证服务开了：**
```bash
<R1> display telnet server status
# 要显示 TELNET IPV4 server : Enable
```

### 第 9 步：本机 telnet

**先装 telnet 客户端（Windows 默认没装）：**
```
控制面板 → 程序 → 启用或关闭 Windows 功能
→ 勾选「Telnet 客户端」→ 确定
```
或管理员 PowerShell：
```powershell
dism /online /Enable-Feature /FeatureName:TelnetClient
```

**然后：**
```cmd
telnet 192.168.95.10
```

```
Password:            ← 输入 Huawei123
                       ⚠️ 输入时屏幕不显示任何字符（连 * 都没有），这是正常的！
                       输完直接回车
```

**看到 `<R1>` 提示符 = 成功！**

---

## 三、本机 VMnet1 的 IP 是多少？

```powershell
Get-NetIPAddress -AddressFamily IPv4 | Where-Object {$_.InterfaceAlias -like '*VMware*'}
```

**常见值：**
| 网卡 | 默认 IP |
|---|---|
| VMware VMnet1（Host-Only） | **192.168.95.1/24** |
| VMware VMnet8（NAT） | 192.168.253.1/24 |

**路由器 IP 必须在同一网段**：VMnet1 是 `192.168.95.1` → 路由器配 `192.168.95.x`。

---

## 四、为什么要用 VMnet1 而不是 VirtualBox Host-Only

**实测数据（本机）：**

```
VirtualBox Host-Only Network:
  发送 11,531,544,213 字节（11.5 GB）
  接收 224,218 字节（224 KB）
  → 严重不平衡，包发出去收不回来

VMware VMnet1:
  一次就通 ✅
```

**原因推测**：VirtualBox 5.2 在 Win11 24H2 上的 Host-Only 驱动行为异常
（还出现过"已关闭的虚拟机 MAC 残留在 ARP 表"的幽灵现象）。

---

## 五、排查清单（ping 不通时按顺序查）

| 步骤 | 检查 | 命令/位置 | 正常值 |
|---|---|---|---|
| 1 | 云启动了吗 | eNSP 里云的颜色 | 绿色 |
| 2 | 端口映射是两条吗 | 双击云 → 端口映射表 | 2 行 |
| 3 | 连线连的是 Internal 口吗 | 选中线看两端 | UDP 那个端口 |
| 4 | 路由器接口 UP 吗 | `display ip interface brief` | `up up` |
| 5 | 路由器 IP 同网段吗 | 同上 | 192.168.95.x/24 |
| 6 | 本机 ARP 学到路由器了吗 | `arp -a` | 有 192.168.95.10 |
| 7 | Windows 防火墙拦 ICMP 吗 | 见下 | 放行 |

### Windows 防火墙要放行 ICMP

**Windows 默认禁止外部 ping 本机**，会导致"路由器 ping 不通本机"：

```powershell
# 管理员 PowerShell
Enable-NetFirewallRule -DisplayName "文件和打印机共享(回显请求 - ICMPv4-In)"
```

**或者新建专用规则：**
```powershell
New-NetFirewallRule -DisplayName "Allow ICMPv4-In (eNSP Lab)" `
  -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow -Profile Any
```

**注意**：蓝屏/重启后这条规则可能被系统恢复成禁用，需要重新开。

---

## 六、telnet vs SSH（建议用 SSH）

**telnet 是明文传输**，密码能被 Wireshark 抓到。**SSH 是加密的。**

### SSH 配置

```bash
[R1] rsa local-key-pair create
   （密钥长度选 2048）

[R1] stelnet server enable

[R1] aaa
[R1-aaa] local-user admin password simple Huawei123
[R1-aaa] local-user admin privilege level 15
[R1-aaa] local-user admin service-type ssh
[R1-aaa] quit

[R1] user-interface vty 0 4
[R1-ui-vty0-4] authentication-mode aaa
[R1-ui-vty0-4] protocol inbound ssh
[R1-ui-vty0-4] quit
```

### 本机连接（Windows 10/11 自带 OpenSSH）

```cmd
ssh admin@192.168.95.10
```

---

## 七、一个"最小可用"的完整命令清单

**从零到能 telnet，复制粘贴即可：**

```bash
# ===== 路由器侧 =====
system-view
sysname R1

interface GigabitEthernet0/0/0
 ip address 192.168.95.10 24
 quit

telnet server enable

user-interface vty 0 4
 authentication-mode password
 set authentication password simple Huawei123
 user privilege level 15
 protocol inbound telnet
 quit
quit
save
y
```

**本机侧：**
```cmd
ping 192.168.95.10
telnet 192.168.95.10
```

---

## 八、这个配置能干什么

**打通之后，你可以：**

| 场景 | 好处 |
|---|---|
| **多个 CMD 窗口同时登录** | 一台设备配完，切到另一个窗口配另一台 |
| **复制粘贴配置** | eNSP 内置窗口不能粘贴长配置，CMD 里可以 |
| **保存命令历史** | 用 Xshell/PuTTY 可以存日志 |
| **配合脚本自动化** | Python + Netmiko 直接连（大三学自动化时会用） |
| **截图/录屏干净** | 不用截整个 eNSP 界面 |

---

## 九、一句话总结

> **云 = 一座桥，需要两个桥墩（一个对内 UDP，一个对外网卡）+ 两条双向映射。**
> **VirtualBox Host-Only 打不通就换 VMware VMnet1。**
> **telnet 必须 `telnet server enable` + `privilege level 15`。**
