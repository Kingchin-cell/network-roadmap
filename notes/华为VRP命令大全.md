# 华为 VRP 常用命令大全

> 按"使用场景"分类，每条都注明**什么时候用**。
> 命令基于 VRP V5/V8（eNSP 里的 AR2220 / S5700 通用）。
>
> ⚠️ 命令支持简写：`system-view` = `sys`，`interface GigabitEthernet0/0/1` = `int g0/0/1`
> ⚠️ 不确定的命令按 `Tab` 补全，按 `?` 看帮助

---

# 一、视图与基础操作

| 命令 | 作用 |
|---|---|
| `system-view` | 用户视图 `<R1>` → 系统视图 `[R1]` |
| `quit` | 返回上一级 |
| `return` 或 `Ctrl+Z` | 直接回用户视图 |
| `display current-configuration` | 看当前运行配置（**最常用**） |
| `display saved-configuration` | 看已保存的配置 |
| `save` | **保存配置**（不保存，重启就丢） |
| `reset saved-configuration` | 清空已保存配置（重启后生效） |
| `display version` | 版本信息 |
| `display clock` | 当前时间 |
| `display this` | 看当前视图下的配置 |

---

# 二、设备管理

| 命令 | 作用 |
|---|---|
| `sysname R1` | 改设备名 |
| `clock timezone CST add 8:00` | 设时区 |
| `clock datetime 12:00:00 2026-11-24` | 设时间 |
| `display device` | 看设备硬件信息 |
| `display cpu-usage` | 看 CPU 占用 |
| `display memory-usage` | 看内存占用 |
| `display temperature all` | 看温度 |
| `display power` | 看电源状态 |
| `display fan` | 看风扇状态 |
| `display logbuffer` | 看日志缓冲区 |
| `display trapbuffer` | 看告警缓冲区 |
| `info-center enable` | 开启信息中心 |

---

# 三、接口配置

| 命令 | 作用 |
|---|---|
| `interface GigabitEthernet 0/0/1` | 进接口（简写 `int g0/0/1`） |
| `ip address 192.168.1.1 24` | 配 IP（24 = 255.255.255.0） |
| `ip address 1.1.1.1 24 sub` | 配从 IP |
| `shutdown` / `undo shutdown` | 关闭 / 开启接口 |
| `description To-Core-Switch` | 加描述（**强烈建议**） |
| `duplex full` / `auto` | 双工模式 |
| `speed 1000` / `auto` | 速率 |
| `mtu 1500` | MTU |

## 查看接口

| 命令                                        | 作用                |
| ----------------------------------------- | ----------------- |
| `display interface GigabitEthernet 0/0/1` | 看某接口详情（错误包、CRC）   |
| `display interface brief`                 | **所有接口一览**（最常用）   |
| `display ip interface brief`              | 看接口 IP + 协议状态     |
| `reset counters interface`                | **清接口计数**（看丢包前先清） |
| `display interface description`           | 看所有接口描述           |

---

# 四、路由表

| 命令 | 作用 |
|---|---|
| `display ip routing-table` | **看完整路由表** |
| `display ip routing-table protocol static` | 只看静态路由 |
| `display ip routing-table protocol ospf` | 只看 OSPF 路由 |
| `display ip routing-table 192.168.1.0` | 查某网段的路由 |
| `display ip routing-table verbose` | 详细模式 |

## 华为默认优先级（越小越优先）

| 路由来源 | 默认优先级 |
|---|---|
| 直连 | **0** |
| OSPF 内部 | **10** |
| IS-IS | 15 |
| **静态路由** | **60** |
| RIP | 100 |
| OSPF 外部（ASE） | 150 |
| BGP | 255 |

---

# 五、静态路由

| 命令 | 作用 |
|---|---|
| `ip route-static 192.168.2.0 24 10.0.0.2` | 目的网段 + 掩码 + 下一跳 |
| `ip route-static 192.168.2.0 24 GigabitEthernet0/0/1` | 指定出接口 |
| `ip route-static 0.0.0.0 0 10.0.0.2` | **默认路由** |
| `ip route-static 192.168.2.0 24 10.0.0.2 preference 100` | **浮动静态路由**（主备切换） |
| `ip route-static 192.168.2.0 24 NULL0` | 黑洞路由（丢弃） |

---

# 六、OSPF

## 基础配置

| 命令 | 作用 |
|---|---|
| `ospf 1 router-id 1.1.1.1` | 启动 OSPF 进程 + 指定 RID |
| `area 0` | 进入区域 0（骨干区域） |
| `network 192.168.1.0 0.0.0.255` | 宣告网段（**反掩码**） |
| `quit` | 退出区域/进程 |

## 接口级调优

| 命令 | 作用 |
|---|---|
| `ospf cost 10` | 改接口开销 |
| `ospf dr-priority 100` | 改 DR 优先级（0 = 不参与选举） |
| `ospf timer hello 10` | 改 Hello 时间 |
| `ospf timer dead 40` | 改 Dead 时间（默认 Hello 的 4 倍） |
| `ospf network-type broadcast` | 改网络类型 |
| `ospf authentication-mode md5 1 cipher xxx` | 接口认证 |

## 查看 OSPF

| 命令 | 作用 |
|---|---|
| `display ospf peer` | **看邻居详情** |
| `display ospf peer brief` | 邻居简表（最常用） |
| `display ospf lsdb` | 看链路状态数据库 |
| `display ospf routing` | 看 OSPF 算出来的路由 |
| `display ospf interface` | 看接口 OSPF 参数 |
| `display ospf error` | 看 OSPF 错误统计 |
| `reset ospf process` | 重启 OSPF 进程（慎用） |

---

# 七、VLAN 与交换

## VLAN 配置

| 命令 | 作用 |
|---|---|
| `vlan 10` | 创建 VLAN 10 并进入 |
| `vlan batch 10 20 30` | 批量创建 |
| `vlan batch 10 to 20` | 批量创建范围 |
| `undo vlan 10` | 删除 VLAN |

## 端口类型

| 命令 | 作用 |
|---|---|
| `port link-type access` | 设为接入口 |
| `port default vlan 10` | 接入口划入 VLAN 10 |
| `port link-type trunk` | 设为干道口 |
| `port trunk allow-pass vlan 10 20` | 允许指定 VLAN 通过 |
| `port trunk allow-pass vlan all` | 允许所有 VLAN |
| `port trunk pvid vlan 10` | 修改 PVID |
| `port link-type hybrid` | 混合口（华为特色） |
| `port hybrid pvid vlan 10` | Hybrid 口 PVID |
| `port hybrid untagged vlan 10` | Hybrid 口以 untagged 方式放行 |
| `port hybrid tagged vlan 10` | Hybrid 口以 tagged 方式放行 |

## VLAN 间通信（三层）

| 命令 | 作用 |
|---|---|
| `interface Vlanif10` | 进 VLAN 虚接口 |
| `ip address 192.168.10.254 24` | 配网关 |

## 查看

| 命令 | 作用 |
|---|---|
| `display vlan` | **看所有 VLAN 和端口归属** |
| `display vlan 10` | 看 VLAN 10 详情 |
| `display port vlan` | 看端口 VLAN 信息 |
| `display mac-address` | **看 MAC 地址表** |
| `display mac-address aging-time` | MAC 老化时间 |

---

# 八、STP / 生成树

| 命令 | 作用 |
|---|---|
| `stp enable` / `undo stp enable` | 开启/关闭（默认开） |
| `stp mode stp` / `rstp` / `mstp` | 切换模式 |
| `stp priority 4096` | 改桥优先级（步长 4096） |
| `stp root primary` | 设为根桥（优先级自动 0） |
| `stp root secondary` | 设为备份根桥 |
| `stp cost 20000` | 接口开销 |
| `stp edged-port enable` | 设为边缘端口 |
| `stp bpdu-protection` | **BPDU 保护**（边缘口收到 BPDU 就 shutdown） |
| `stp root-protection` | **根保护**（防根桥被抢） |
| `stp loop-protection` | **环路保护** |
| `stp tc-protection` | TC 保护 |

## 查看 STP

| 命令 | 作用 |
|---|---|
| `display stp` | 完整 STP 信息 |
| `display stp brief` | **端口角色一览**（最常用） |
| `display stp interface GigabitEthernet0/0/1` | 某端口 STP 详情 |
| `display stp root` | 看根桥信息 |

---

# 九、链路聚合

| 命令 | 作用 |
|---|---|
| `interface Eth-Trunk 1` | 创建聚合组 |
| `mode lacp-static` | LACP 模式（推荐） |
| `mode manual load-balance` | 手工模式 |
| `trunkport GigabitEthernet 0/0/1` | 把物理口加入 |
| `trunkport GigabitEthernet 0/0/1 to 0/0/3` | 批量加入 |
| `max active-linknumber 2` | 最大活动链路数 |
| `load-balance src-dst-ip` | 负载分担方式 |

## 查看

| 命令 | 作用 |
|---|---|
| `display eth-trunk 1` | **聚合组状态** |
| `display interface Eth-Trunk 1` | 聚合口统计 |
| `display lacp peer` | LACP 邻居 |

---

# 十、ACL

## 基本 ACL（2000–2999，只匹配源 IP）

```bash
acl 2000
 rule 5 permit source 192.168.1.0 0.0.0.255
 rule 10 deny source any
quit
```

## 高级 ACL（3000–3999，源+目的+端口）

```bash
acl 3000
 rule 5 permit tcp source 192.168.1.0 0.0.0.255 destination 192.168.2.10 0 destination-port eq 80
quit
```

## 应用 ACL

| 命令 | 作用 |
|---|---|
| `traffic-filter inbound acl 3000` | 接口入方向过滤 |
| `traffic-filter outbound acl 3000` | 接口出方向过滤 |
| `nat outbound 2000` | 用于 NAT |

## 查看

| 命令 | 作用 |
|---|---|
| `display acl 3000` | 看某 ACL |
| `display acl all` | 看所有 ACL |

> 💡 **通配符（反掩码）**：`0.0.0.255` = 前 24 位固定，后 8 位任意
> `0` = 完全匹配，`255.255.255.255` = 任意

---

# 十一、NAT

| 命令 | 作用 |
|---|---|
| `nat outbound 2000` | **Easy IP**（配合基本 ACL） |
| `nat outbound 2000 address-group 1` | NAT 地址池 |
| `nat server protocol tcp global current-interface 80 inside 192.168.1.10 80` | **端口映射** |
| `nat static global 200.1.1.1 inside 192.168.1.10` | 静态一对一 |

## 查看

| 命令 | 作用 |
|---|---|
| `display nat session all` | **看 NAT 会话表** |
| `display nat outbound` | 看出方向 NAT 配置 |
| `display nat server` | 看端口映射 |
| `reset nat session` | 清 NAT 会话 |

---

# 十二、DHCP

## 全局地址池

```bash
dhcp enable                          # 先开服务（必须！）
ip pool p1
 network 192.168.1.0 mask 255.255.255.0
 gateway-list 192.168.1.254
 dns-list 223.5.5.5
 lease day 1
 quit
interface Vlanif10
 dhcp select global                  # 接口下指定用全局池
```

## DHCP 中继

```bash
dhcp enable
interface GigabitEthernet0/0/0
 dhcp select relay
 dhcp relay server-ip 192.168.1.100
```

## 查看

| 命令 | 作用 |
|---|---|
| `display ip pool` | **看地址池使用情况** |
| `display ip pool name p1` | 看指定池 |
| `display dhcp server statistics` | 统计 |

---

# 十三、AAA 与用户管理

```bash
aaa
 local-user admin password cipher Huawei@123
 local-user admin privilege level 15
 local-user admin service-type telnet ssh
quit
```

## 配 Console/Telnet 登录

```bash
# Console
user-interface console 0
 authentication-mode password
 set authentication password cipher xxx

# Telnet
user-interface vty 0 4
 authentication-mode password
 set authentication password cipher xxx
 protocol inbound telnet
```

## 查看

| 命令 | 作用 |
|---|---|
| `display local-user` | 看本地用户 |
| `display aaa` | AAA 配置 |
| `display users` | 在线用户 |

---

# 十四、VRRP（网关冗余）

| 命令 | 作用 |
|---|---|
| `vrrp vrid 1 virtual-ip 192.168.10.254` | 配虚拟 IP |
| `vrrp vrid 1 priority 120` | 改优先级（默认 100） |
| `vrrp vrid 1 preempt-mode timer delay 20` | 抢占延迟 |
| `vrrp vrid 1 track interface GigabitEthernet0/0/1 reduced 30` | 跟踪接口，故障时降优先级 |

## 查看

| 命令 | 作用 |
|---|---|
| `display vrrp` | 详细状态 |
| `display vrrp brief` | 简表 |

---

# 十五、端口安全

| 命令 | 作用 |
|---|---|
| `port-security enable` | 开启 |
| `port-security max-mac-num 2` | 限制 MAC 数量 |
| `port-security mac-address sticky` | MAC 粘滞（自动学习并绑定） |
| `port-security protect-action shutdown` | 违规时关闭端口 |

## DHCP Snooping

| 命令 | 作用 |
|---|---|
| `dhcp snooping enable` | 全局开启 |
| `dhcp snooping enable` | 接口下开启（VLAN 视图） |
| `dhcp snooping trusted` | 设为信任端口（连 DHCP 服务器的口） |
| `display dhcp snooping` | 查看 |

---

# 十六、监控与排障（最实用）

| 命令 | 作用 |
|---|---|
| `ping -c 4 8.8.8.8` | 测试连通性（发 4 个包） |
| `ping -a 192.168.1.1 8.8.8.8` | 指定源 IP ping |
| `ping -s 1500 8.8.8.8` | 指定包大小（测 MTU） |
| `tracert 8.8.8.8` | 路径追踪 |
| `display arp` | **ARP 表** |
| `display mac-address` | **MAC 表** |
| `display ip routing-table` | 路由表 |
| `display logbuffer` | 日志 |
| `terminal monitor` | 在终端显示日志 |
| `terminal debugging` | 在终端显示调试信息 |
| `debugging ip packet` | 开调试（**慎用，会刷屏**） |
| `undo debugging all` | **关所有调试** |
| `display diagnostic-information` | 一键收集诊断信息 |

## 排障口诀

```
ping 不通 → 按这个顺序查：
① 接口物理状态：display interface brief（UP 还是 DOWN？）
② IP 配置：display ip interface brief（IP 对不对？）
③ ARP：display arp（有没有学到对方 MAC？）
④ 路由：display ip routing-table（有没有去程？回程有没有？）
⑤ ACL：display acl all（是不是被拦了？）
⑥ NAT：display nat session all（NAT 生效了吗？）
```

---

# 十七、文件系统

| 命令 | 作用 |
|---|---|
| `dir` | 看当前目录文件 |
| `pwd` | 当前路径 |
| `cd flash:/` | 切目录 |
| `mkdir test` | 建目录 |
| `delete test.txt` | 删除文件 |
| `delete /unreserved test.txt` | **彻底删除**（不进回收站） |
| `rename a.txt b.txt` | 改名 |
| `copy a.txt b.txt` | 复制 |
| `display startup` | 看下次启动用哪个配置 |

---

# 十八、快捷键（提效必备）

| 键 | 作用 |
|---|---|
| `Tab` | **命令补全**（最常用） |
| `?` | 帮助（列命令/参数） |
| `Ctrl+Z` | 返回用户视图 |
| `Ctrl+A` | 光标移到行首 |
| `Ctrl+E` | 光标移到行尾 |
| `Ctrl+C` | 中断当前操作 |
| `Ctrl+U` | 删除整行 |
| `↑` / `↓` | 历史命令 |

---

# 十九、默认值速记

| 项目 | 默认值 |
|---|---|
| 端口 PVID | VLAN 1 |
| STP 优先级 | 32768 |
| STP 模式 | MSTP（部分版本） |
| OSPF Cost | 10⁸ / 带宽（百兆 = 1，千兆 = 1） |
| OSPF Hello | 10 秒（广播网络） |
| OSPF Dead | 40 秒 |
| VRRP 优先级 | 100 |
| 静态路由优先级 | 60 |
| 直连优先级 | 0 |

---

# 二十、一个"最小可用"的配置模板

**新建一台路由器，从零开始：**

```bash
system-view
sysname R1
interface GigabitEthernet0/0/0
 ip address 10.0.0.1 24
 description To-R2
 quit
ip route-static 0.0.0.0 0 10.0.0.2
quit
save
y
```

**新建一台三层交换机，配两个 VLAN：**

```bash
system-view
sysname SW1
vlan batch 10 20
interface GigabitEthernet0/0/1
 port link-type access
 port default vlan 10
 quit
interface GigabitEthernet0/0/2
 port link-type access
 port default vlan 20
 quit
interface Vlanif10
 ip address 192.168.10.254 24
 quit
interface Vlanif20
 ip address 192.168.20.254 24
 quit
save
y
```

---

# 二十一、常见报错对照

| 报错 | 原因 | 解决 |
|---|---|---|
| `Error: Unrecognized command found at '^' position.` | 命令拼错或视图不对 | 检查拼写、`Tab` 补全、确认在正确视图 |
| `Error: Wrong parameter found at '^' position.` | 参数不对（IP/掩码写错） | 检查参数格式 |
| `Error: The specified interface does not exist.` | 接口编号写错 | 确认设备型号的接口编号 |
| `Error: The IP address is in use.` | IP 已被占用 | 换一个或先删原来的 |
| `Error: Please save the configuration.` | 某些操作需先保存 | `save` |
| `Info: The configuration takes effect only after the device is restarted.` | 部分配置需重启 | 重启设备 |

---

# 二十二、用这份表的建议

```
① 不要背，用的时候查
② 按 Tab 补全，比记全拼快
③ 每个实验做完，把用到的命令抄一遍到笔记（写一遍顶看十遍）
④ 遇到不认识的命令，按 ? 看帮助
```

**最后一句**：
> **命令是"手"，原理是"脑"。**
> 只会命令不会原理 = 照抄；懂原理再查命令 = 工程师。
