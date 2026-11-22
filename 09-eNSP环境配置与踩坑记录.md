# 09 · eNSP 环境配置与踩坑记录（Win11 24H2 血泪版）

> **这份文件记录了 2026-11-23 那一晚，在一台 Windows 11 24H2 机器上把 eNSP 跑起来
> 踩过的全部坑、试过的全部错路，以及最终的解法。**
>
> 目的：以后重装系统 / 帮同学装 / eNSP 又坏了，直接照着做，不用再花 3 小时。

---

## 一、最终结论（先看这个）

**在 Windows 11 24H2 上让 eNSP 可用，只需要两步：**

```
① 用微软官方工具 DG_Readiness_Tool_v3.6.ps1 -Disable
② 重启时，在 "Credential Guard Opt-out Tool" 界面按 F3
```

**其他所有方法（关内存完整性、关 VBS 注册表、禁服务、关 Secure Boot、组策略）
在 24H2 上都不够。**

### ⚠️ 最重要的一条：**这个修复只对「下一次启动」有效**

2026-11-24 实测确认：

| 操作 | 结果 |
|---|---|
| 跑 DG 工具 → 重启 → 按 F3 | ✅ eNSP 可用 |
| 之后再普通重启（不跑工具） | ❌ **失效，hypervisor 又回来了** |

**所以实际用法是：**

```
每次想用 eNSP 时：
   ① 双击桌面「修复eNSP环境.ps1」（右键 → 用 PowerShell 运行）
   ② 脚本最后问"现在重启吗" → 输 y 自动重启
   ③ 重启时看到黑底界面 → 按 F3 → 按任意键
   ④ 进系统后 eNSP 就能用
```

**省事的技巧**：平时用完电脑用「**睡眠**」而不是「关机/重启」，配置就不会丢。
只有真需要重启时，才走一遍上面的流程。

---

## 二、本机环境（出问题的那台）

| 项目 | 值 |
|---|---|
| 机器 | MECHREVO KUANGSHI Series |
| CPU | Intel Core i5-14450HX |
| 内存 | 15.7 GB |
| 系统 | **Windows 11 24H2**（Build 26100.6584，家庭中文版） |
| 注 | 注册表 `ProductName` 显示 "Windows 10 Home China"，**这是微软的兼容字段，实际是 Win11** |
| BIOS | American Megatrends N.1.21MRO16（2025/7/24） |
| Secure Boot | 排查过程中被手动关闭过（后来发现**不是必需的**） |
| BitLocker | 未启用 |

### 装的软件

| 软件 | 版本 | 说明 |
|---|---|---|
| **eNSP** | **1.3.00.200T** | 装在 `D:\eNSP\`，发布者 Huawei ✅ |
| **VirtualBox** | **5.2.44r139111** | ⚠️ **必须是 5.2.x**，6.x/7.x 不行 |
| **WinPcap** | 4.1.3 | 抓包驱动 |
| **Wireshark** | 4.6.10 | 可选但强烈建议 |

---

## 三、症状

### 现象 1：eNSP 拖出设备，一直红色，启动失败

### 现象 2：直接用 VBoxManage 启动会报这个错（最准确的诊断入口）

```
VBoxManage.exe: error: Raw-mode is unavailable courtesy of Hyper-V.
                (VERR_SUPDRV_NO_RAW_MODE_HYPER_V_ROOT)
VBoxManage.exe: error: Details: code E_FAIL (0x80004005),
                component ConsoleWrap, interface IConsole
```

### 现象 3：VBoxManage 检测不到硬件虚拟化

```
> VBoxManage list hostinfo
Processor supports HW virtualization: no     ← 这就是问题
```

### 现象 4：系统层面

```powershell
(Get-CimInstance Win32_ComputerSystem).HypervisorPresent   # True  ← 有问题
(Get-CimInstance Win32_Processor).VirtualizationFirmwareEnabled  # False ← 有问题
(Get-CimInstance -Namespace root\Microsoft\Windows\DeviceGuard -ClassName Win32_DeviceGuard).VirtualizationBasedSecurityStatus  # 2 = 运行中
```

---

## 四、根因

```
eNSP 只认 VirtualBox 5.2（它的 tools 目录里有 VBox42/43/50/51/52.dll，没有 60/70）
        ↓
VirtualBox 5.2 只能用"raw mode"独占 VT-x（不支持跟 Hyper-V 共存）
        ↓
Win11 24H2 的 VBS（基于虚拟化的安全）占了 VT-x
        ↓
VirtualBox 抢不到 → 报 Raw-mode unavailable → eNSP 设备起不来
```

**Win11 24H2 的特殊性**：VBS 锁得比其他版本死，手动改注册表/禁服务都关不掉，必须用官方工具。

---

## 五、✅ 正确的解决步骤（可复现）

### 第 1 步：下载微软官方 DG 工具

```
官方页面：https://www.microsoft.com/en-us/download/details.aspx?id=53337
短链     ：https://aka.ms/dgwhcr
直链     ：https://download.microsoft.com/download/b/d/8/bd821b1f-05f2-4a7e-aa03-df6c4f687b07/dgreadiness_v3.6.zip

下 dgreadiness_v3.6.zip（约 32 KB）
解压到 C:\Users\<你的用户名>\DGReadiness\tool\
→ 里面应该有 DG_Readiness_Tool_v3.6.ps1（76 KB）
```

**验证签名**（重要，别用来路不明的脚本）：
```powershell
Get-AuthenticodeSignature "C:\Users\21651\DGReadiness\tool\dgreadiness_v3.6\DG_Readiness_Tool_v3.6.ps1"
# 应该显示 Status = Valid，Signer = CN=Microsoft Corporation
```

### 第 2 步：以管理员运行 DG 工具

```powershell
# 管理员 PowerShell
Set-ExecutionPolicy RemoteSigned -Scope Process -Force
cd C:\Users\21651\DGReadiness\tool\dgreadiness_v3.6
.\DG_Readiness_Tool_v3.6.ps1 -Disable
```

**输出里会有一些 "REG DELETE ... 找不到" 的报错——这是正常的**（说明那些值本来就不存在）。

### 第 3 步：设置 bcdedit

```cmd
# 管理员 CMD
bcdedit /set hypervisorlaunchtype off
bcdedit /deletevalue vsmlaunchtype
```

验证：
```cmd
bcdedit /enum {current} | findstr hypervisor
# 应该显示：hypervisorlaunchtype    Off
```

### 第 4 步：重启，**按 F3** ⭐⭐⭐

```
重启电脑
        ↓
⚠️ 过程中会出现黑底界面，标题类似：
     "Credential Guard Opt-out Tool"
     "按 F3 禁用 Credential Guard..."
        ↓
  >>> 立刻按 F3 <<<
        ↓
（界面只停 3-5 秒，错过就得重来）
```

**这一步是整个流程的成败关键。**

### 第 5 步：验证

```powershell
# 1) Hypervisor 是否关掉
(Get-CimInstance Win32_ComputerSystem).HypervisorPresent
# 应该是 False

# 2) 硬件虚拟化是否可用
(Get-CimInstance Win32_Processor).VirtualizationFirmwareEnabled
# 应该是 True

# 3) VirtualBox 是否检测到
& 'C:\Program Files\Oracle\VirtualBox\VBoxManage.exe' list hostinfo
# 应该是：Processor supports HW virtualization: yes

# 4) 直接启动 eNSP 的模板虚拟机
& 'C:\Program Files\Oracle\VirtualBox\VBoxManage.exe' startvm "AR_Base" --type headless
# 应该是：VM "AR_Base" has been successfully started.
# （测试完记得关掉：VBoxManage controlvm "AR_Base" poweroff）

# 5) 最后：打开 eNSP，拖一台 AR2220，右键启动 → 1-2 分钟变绿
```

---

## 六、❌ 试过但没用的方法（别浪费时间）

> 这些在**普通 Win10/Win11 23H2** 上可能有效，但**在 Win11 24H2 上不够**。

| # | 尝试 | 结果 | 为什么没用 |
|---|---|---|---|
| 1 | 关"内存完整性"（HVCI） | ❌ | 只是 VBS 的一个子功能，VBS 平台还在 |
| 2 | 关 VirtualMachinePlatform 功能 | ❌ | 只是触发器之一 |
| 3 | `bcdedit hypervisorlaunchtype off` 单独用 | ❌ | 需要配合 DG 工具 |
| 4 | 改注册表 `EnableVirtualizationBasedSecurity = 0` | ❌ | 值改了但 VBS 仍在运行 |
| 5 | 禁用 WslService / VMSMP / HvHost / vmcompute | ❌ | 服务不是根因 |
| 6 | 关 Secure Boot（进 BIOS） | ❌ | **不是它的问题，不用关** |
| 7 | `bcdedit vsmlaunchtype off` 单独用 | ❌ | 同上 |
| 8 | 在 Win11 虚拟机里装（VMware 套娃） | ❌ | Guru Meditation（Win11 里也有 VBS） |
| 9 | VirtualBox 6.x / 7.x | ❌ | eNSP 不认（版本校验） |

**关键教训**：
> **Win11 24H2 上，VBS 必须用微软官方 DG_Readiness_Tool 才能彻底关掉，
> 而且必须在重启时按 F3 确认。**

---

## 七、⚠️ 这个配置很脆弱，会被这些操作破坏

| 操作 | 后果 |
|---|---|
| **Windows 大版本更新** | 可能重置 VBS 配置 |
| **安装 Docker Desktop** | 会重新启用 Hyper-V |
| **安装/更新 WSL2** | 会重新启用虚拟机平台 |
| 某些"系统优化"工具 | 可能改动 DeviceGuard |
| 手动跑 `bcdedit /set hypervisorlaunchtype auto` | 直接失效 |

**所以：如果哪天 eNSP 又坏了，跑一遍桌面上的 `修复eNSP环境.ps1`，重启时按 F3。**

---

## 八、💔 代价：WSL2 / Docker 不能用了

```
原因：WSL2 和 Docker 都需要 hypervisor
     hypervisor 关了 → 它们就用不了
```

**WSL 启动会报：**
```
错误代码: Wsl/Service/CreateInstance/CreateVm/HCS/HCS_E_HYPERV_NOT_INSTALLED
```

**需要用 Linux 时怎么办**：

| 方案 | 说明 |
|---|---|
| **VMware 装 Ubuntu** ⭐ 推荐 | 完整 Linux，能 tcpdump 抓包，网工实验最合适 |
| WSL1（降级） | 只能跑基础命令，**抓不了包** |
| 换一台电脑做 Linux 实验 | — |

**注意：关掉 VBS 后，VMware 的性能反而变好了**（之前走 WHP 模式很慢，现在能独占 VT-x）。

---

## 九、🔐 安全提醒

关掉 VBS / HVCI / Credential Guard 之后，系统少了内核级保护。

**对这台机器的特别提醒**（它在 2026 年 9 月和 11 月各中过一次木马）：

```
❌ 绝对不要执行：irm xxx.com | iex
❌ 不要从来路不明的网站下载 .exe / .bat / .ps1
❌ 不要用"破解版""绿色版"软件
✅ 装软件只从官方渠道
✅ 火绒保持运行
```

---

## 十、一键修复脚本

桌面上有 **`修复eNSP环境.ps1`**（v2），它做了：
1. 检查 DG 工具是否存在（并验证微软签名）
2. 调用 `DG_Readiness_Tool_v3.6.ps1 -Disable`
3. 设置 `bcdedit hypervisorlaunchtype off`
4. 用大字提示"重启时按 F3"
5. 日志写到 `C:\Users\<用户名>\ensp_fix_log.txt`

**用法**：右键 → 使用 PowerShell 运行

---

## 十一、故障排查速查表

| 现象 | 可能原因 | 怎么办 |
|---|---|---|
| `Raw-mode is unavailable courtesy of Hyper-V` | hypervisor 在跑 | 跑修复脚本 + 重启按 F3 |
| `VirtualBox version is not supported` | VirtualBox 版本不对 | 卸载，装 **5.2.44** |
| 设备一直红色，无报错 | hypervisor / 虚拟网卡 | 先看 `VBoxManage list hostinfo` |
| 错误代码 40 | 虚拟网卡 / DHCP | 关 VirtualBox 的 DHCP 服务器；检查 Host-Only 网卡 IP 是 192.168.56.1 |
| 命令行一直输出 `####` | 杀软拦截设备通信 | 临时退出火绒试试 |
| `Guru Meditation` | 虚拟化层面严重错误 | 一般说明 hypervisor 还在 |
| 设备能起但 ping 不通 | 网卡 / 防火墙 | 检查 `Host-Only Network` 是否 Up、防火墙是否放行 ensp_vboxserver |

### 诊断三连（出问题先跑这三个）

```powershell
# 1) 系统层面
(Get-CimInstance Win32_ComputerSystem).HypervisorPresent        # 要 False
(Get-CimInstance Win32_Processor).VirtualizationFirmwareEnabled # 要 True

# 2) VirtualBox 层面
& 'C:\Program Files\Oracle\VirtualBox\VBoxManage.exe' list hostinfo | Select-String 'HW virtualization'
# 要 yes

# 3) 实际测试
& 'C:\Program Files\Oracle\VirtualBox\VBoxManage.exe' startvm "AR_Base" --type headless
# 要 successfully started（测完 poweroff 关掉）
```

---

## 十二、一句话总结

> **Win11 24H2 + eNSP 的组合，唯一可靠的解法是：
> 微软 DG_Readiness_Tool -Disable + 重启时按 F3。
> 其他所有教程上的方法，在这个版本上都不够。**
