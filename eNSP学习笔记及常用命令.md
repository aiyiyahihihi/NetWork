# eNSP学习笔记及常用命令

> **eNSP (Enterprise Network Simulation Platform)** 是华为推出的图形化网络仿真平台，用于模拟华为路由器、交换机等网络设备，支持 HCIA/HCIP/HCIE 等级别的学习与实验。

---

## eNSP基础

### 软件组成

| 组件 | 作用 |
|------|------|
| eNSP主程序 | 图形化拓扑编辑与设备管理 |
| VirtualBox | 设备虚拟化底层 |
| Wireshark | 抓包分析 |
| WinPcap/Npcap | 网络数据包捕获驱动 |

### 常用设备型号

| 设备类型 | 型号 | 接口默认命名 |
|----------|------|-------------|
| 路由器 | AR2220 / AR3260 | `GigabitEthernet0/0/0` |
| 交换机 | S5700 / S3700 | `GigabitEthernet0/0/1` |
| 三层交换机 | S5700 | 支持路由功能 |
| 防火墙 | USG6000V | `GigabitEthernet1/0/0` |
| 无线AC | AC6005 | — |
| 云设备 | Cloud | 用于桥接物理网络 |

### 拓扑搭建步骤

1. 从左侧设备列表拖拽设备到工作区
2. 选择线缆类型连接设备接口
3. 点击"启动"按钮启动所有设备
4. 双击设备图标进入 CLI 命令行界面

### 快捷键

| 快捷键 | 功能 |
|:------:|------|
| `Ctrl + A` | 打开设备 CLI |
| `Ctrl + C` | 停止ping等持续命令 |
| `Ctrl + Z` | 从配置模式返回用户视图 |
| `Tab` | 命令自动补全 |
| `?` | 显示当前可用的命令或参数提示 |

---

## 命令行基础

### 视图层级结构

```
用户视图 → <system-view> → 系统视图 → 各种子视图（接口视图、协议视图等）
<Huawei>                 [Huawei]  [Huawei-GigabitEthernet0/0/0]
```

### 视图切换命令

```bash
<Huawei>                          # 用户视图：查看状态，权限最低
<Huawei> system-view              # 进入系统视图（必须从这里开始配置）
[Huawei]                          # 系统视图：进行全局配置

# 进入接口视图
[Huawei] interface GigabitEthernet 0/0/0
[Huawei-GigabitEthernet0/0/0]

# 进入VLAN视图
[Huawei] vlan 10
[Huawei-vlan10]

# 进入VTY视图（远程管理）
[Huawei] user-interface vty 0 4
[Huawei-ui-vty0-4]

# 进入OSPF进程视图
[Huawei] ospf 1
[Huawei-ospf-1]

# 返回上一级
[Huawei-GigabitEthernet0/0/0] quit
# 或简写为：
[Huawei-GigabitEthernet0/0/0] q

# 直接返回用户视图（任何配置模式下均生效）
[Huawei-GigabitEthernet0/0/0] return
<Huawei>
```

### 帮助系统

```bash
?              # 显示当前视图下所有可用命令
s?             # 显示以s开头的所有命令
system-view ?  # 显示该命令后可接的参数
interface ?    # 显示interface后可接的类型
```

---

## 设备基础配置

### 设备命名

```bash
[Huawei] sysname R1  # 将设备命名为R1
[R1]
```

### 登录信息（标题/标语）

```bash
[R1] header login information "Welcome to R1"     # 登录前显示
[R1] header shell information "You are in R1"     # 登录后显示
```

### 密码设置

```bash
# 设置Console口密码
[R1] user-interface console 0
[R1-ui-console0] authentication-mode password
[R1-ui-console0] set authentication password cipher huawei123

# 设置特权模式密码（进入系统视图密码）
[R1] super password cipher huawei456
```

### 接口配置

```bash
[R1] interface GigabitEthernet 0/0/0
[R1-GigabitEthernet0/0/0] ip address 192.168.1.1 255.255.255.0   # 配置IP
[R1-GigabitEthernet0/0/0] undo shutdown                          # 开启接口
[R1-GigabitEthernet0/0/0] description TO-SW1                     # 接口描述
[R1-GigabitEthernet0/0/0] display this                           # 查看当前接口配置
```

### 时间与系统配置

```bash
[R1] clock timezone BJ add 08:00:00      # 设置北京时区
[R1] clock datetime 10:30:00 2026-07-13  # 设置日期时间
[R1] display clock                       # 查看当前时间
```

---

## VLAN配置

### 交换机VLAN基础

```bash
# 创建VLAN
[SW1] vlan 10
[SW1-vlan10] description SALES  # VLAN描述
[SW1-vlan10] q
[SW1] vlan batch 10 20 30       # 批量创建VLAN

# 将接口加入VLAN（Access模式）
[SW1] interface GigabitEthernet 0/0/1
[SW1-GigabitEthernet0/0/1] port link-type access
[SW1-GigabitEthernet0/0/1] port default vlan 10

# 将接口加入VLAN（Access模式，一条命令）
[SW1] interface GigabitEthernet 0/0/2
[SW1-GigabitEthernet0/0/2] port link-type access
[SW1-GigabitEthernet0/0/2] port default vlan 20

# Trunk模式配置（交换机间互联口）
[SW1] interface GigabitEthernet 0/0/24
[SW1-GigabitEthernet0/0/24] port link-type trunk
[SW1-GigabitEthernet0/0/24] port trunk allow-pass vlan 10 20
[SW1-GigabitEthernet0/0/24] port trunk allow-pass vlan all    # 允许所有VLAN通过
```

### VLAN查看命令

```bash
[SW1] display vlan                             # 查看所有VLAN
[SW1] display vlan 10                          # 查看VLAN 10的详细信息
[SW1] display port vlan                        # 查看端口VLAN信息
[SW1] display port vlan GigabitEthernet 0/0/1  # 查看指定端口的VLAN
```

### VLAN间通信 —— 三层交换机SVI

```bash
[SW1] interface Vlanif 10
[SW1-Vlanif10] ip address 192.168.10.254 255.255.255.0
[SW1-Vlanif10] q

[SW1] interface Vlanif 20
[SW1-Vlanif20] ip address 192.168.20.254 255.255.255.0
```

### VLAN 间通信 —— 单臂路由（路由器子接口）

```bash
[R1] interface GigabitEthernet 0/0/0.10
[R1-GigabitEthernet0/0/0.10] dot1q termination vid 10
[R1-GigabitEthernet0/0/0.10] ip address 192.168.10.1 255.255.255.0
[R1-GigabitEthernet0/0/0.10] arp broadcast enable                   # 开启ARP广播
```

------

## STP生成树协议

### STP模式设置

```bash
# 华为默认MSTP模式，可切换
[SW1] stp mode stp      # 普通生成树
[SW1] stp mode rstp     # 快速生成树（推荐）
[SW1] stp mode mstp     # 多实例生成树（默认）

# 启用/禁用 STP
[SW1] stp enable        # 全局启用
[SW1] stp disable       # 全局禁用
[SW1] undo stp enable
```

### 根桥与优先级调整

```bash
[SW1] stp root primary           # 指定为主根桥（优先级自动设为0）
[SW2] stp root secondary         # 指定为备用根桥（优先级=4096）

# 手动调整优先级（必须为4096的倍数）
[SW1] stp priority 4096

# 接口 STP 配置
[SW1-GigabitEthernet0/0/1] stp edged-port enable    # 边缘端口（连PC口）
[SW1-GigabitEthernet0/0/1] stp disable              # 接口禁用 STP
```

### STP查看命令

```bash
[SW1] display stp                          # 查看 STP 状态
[SW1] display stp brief                    # 简要信息（端口角色/状态）
[SW1] display stp interface GigabitEthernet 0/0/1  # 接口 STP 状态
```

---

## 链路聚合（Eth-Trunk）

### 6.1 手动模式

```bash
[SW1] interface Eth-Trunk 1
[SW1-Eth-Trunk1] mode manual load-balance     # 手动模式（默认）
[SW1-Eth-Trunk1] trunkport GigabitEthernet 0/0/1 to 0/0/2
[SW1-Eth-Trunk1] port link-type trunk
[SW1-Eth-Trunk1] port trunk allow-pass vlan all
```

### 6.2 LACP 模式（推荐）

```bash
[SW1] interface Eth-Trunk 1
[SW1-Eth-Trunk1] mode lacp-static             # LACP 静态模式
[SW1-Eth-Trunk1] trunkport GigabitEthernet 0/0/1 to 0/0/2
[SW1-Eth-Trunk1] port link-type trunk
[SW1-Eth-Trunk1] port trunk allow-pass vlan all
```

### 6.3 查看命令

```bash
[SW1] display eth-trunk 1                     # 查看聚合口详情
[SW1] display interface Eth-Trunk 1           # 查看聚合口状态
```

---

## 静态路由与默认路由

### 静态路由

```bash
# ip route-static 目标网络 掩码 下一跳IP
[R1] ip route-static 192.168.2.0 255.255.255.0 10.0.0.2

# ip route-static 目标网络 掩码 出接口
[R1] ip route-static 192.168.2.0 255.255.255.0 GigabitEthernet 0/0/1

# 推荐
# ip route-static 目标网络 掩码 出接口 下一跳
[R1] ip route-static 192.168.2.0 255.255.255.0 GigabitEthernet 0/0/1 10.0.0.2
```

### 默认路由

```bash
# 全零路由，匹配所有目标
[R1] ip route-static 0.0.0.0 0.0.0.0 10.0.0.2

# 等价默认路由出处接口
[R1] ip route-static 0.0.0.0 0.0.0.0 GigabitEthernet 0/0/0 10.0.0.2
```

### 浮动静态路由（备份路由）

> preference：路由优先级

```bash
# 通过设置较高优先级（数值越大优先级越低）实现备份
# ip route-static 目标网络 掩码 下一跳 preference 优先值
[R1] ip route-static 192.168.2.0 255.255.255.0 10.0.0.2 preference 60    # 主路由
[R1] ip route-static 192.168.2.0 255.255.255.0 20.0.0.2 preference 80    # 备份路由
```

### 路由查看命令

> protocol：路由协议

```bash
[R1] display ip routing-table                  # 查看路由表
[R1] display ip routing-table protocol static  # 查看静态路由
[R1] display ip routing-table 192.168.1.0      # 查看到特定网络的路由
```

---

## RIP路由协议

### RIPv2基本配置

```bash
[R1] rip 1                      # 进入RIP进程，进程ID为1
[R1-rip-1] version 2            # 使用RIPv2（支持无类路由）
[R1-rip-1] undo summary         # 关闭自动汇总（必须关闭）
[R1-rip-1] network 192.168.1.0  # 宣告直连主类网络
[R1-rip-1] network 10.0.0.0
```

### 高级配置

```bash
# 静默接口（只收不发）
[R1-rip-1] silent-interface GigabitEthernet 0/0/0

# 手动汇总
[R1-GigabitEthernet0/0/0] rip summary-address 192.168.0.0 255.255.252.0

# 水平分割（默认开启）
[R1-GigabitEthernet0/0/0] rip split-horizon    # 如果是 poison-reverse 用 undo split-horizon + poison-reverse

# 引入外部路由时的默认度量值
[R1-rip-1] default-cost 3
```

### 查看命令

```bash
[R1] display rip             # 查看RIP进程信息
[R1] display rip 1 route     # 查看RIP路由表
[R1] display rip 1 database  # 查看RIP数据库
```

---

## OSPF路由协议

### 单区域OSPF基本配置

```bash
[R1] ospf 1 router-id 1.1.1.1                           # 启动OSPF进程并指定Router-ID
[R1-ospf-1] area 0                                      # 进入骨干区域
[R1-ospf-1-area-0.0.0.0] network 192.168.1.0 0.0.0.255  # 宣告网络（反掩码）
[R1-ospf-1-area-0.0.0.0] network 10.0.0.0 0.0.0.3
```

> **反掩码规则**：反掩码 = `255.255.255.255 - 子网掩码`，例如：`/24 = 0.0.0.255`，`/30 = 0.0.0.3`

### 多区域OSPF

```bash
# 区域0（骨干区域）
[R1] ospf 1 router-id 1.1.1.1
[R1-ospf-1] area 0
[R1-ospf-1-area-0.0.0.0] network 10.0.12.0 0.0.0.3

# 区域1（普通区域）
[R1-ospf-1] area 1
[R1-ospf-1-area-0.0.0.1] network 192.168.1.0 0.0.0.255

# Stub 区域配置
[R1-ospf-1] area 1
[R1-ospf-1-area-0.0.0.1] stub
```

### 接口精确宣告（推荐方式）

```bash
[R1] ospf 1 router-id 1.1.1.1
[R1-ospf-1] area 0
[R1-ospf-1-area-0.0.0.0] quit
[R1-ospf-1] quit

[R1] interface GigabitEthernet 0/0/0
[R1-GigabitEthernet0/0/0] ospf enable 1 area 0   # 接口使能 OSPF
```

### OSPF高级配置

```bash
# 修改接口优先级（影响 DR/BDR 选举，0=不参与选举）
[R1-GigabitEthernet0/0/0] ospf dr-priority 100

# 修改网络类型
[R1-GigabitEthernet0/0/0] ospf network-type p2p   # 点对点（收敛更快）

# 修改接口 Cost
[R1-GigabitEthernet0/0/0] ospf cost 10

# 修改参考带宽（默认100Mbps）
[R1-ospf-1] bandwidth-reference 1000   # 设为1000Mbps（千兆）

# 引入静态路由
[R1-ospf-1] import-route static

# 引入直连路由
[R1-ospf-1] import-route direct
```

### OSPF查看命令

```bash
[R1] display ospf peer                         # 查看 OSPF 邻居
[R1] display ospf peer brief                    # 邻居简要信息
[R1] display ospf routing                       # 查看 OSPF 路由表
[R1] display ospf lsdb                          # 查看链路状态数据库
[R1] display ospf interface                     # OSPF 接口信息
[R1] display ospf brief                         # OSPF 进程简要信息
[R1] display ospf error                         # OSPF 错误信息（排错用）
```

---

## DHCP配置

### 基于全局地址池

```bash
# 开启 DHCP
[R1] dhcp enable

# 创建地址池
[R1] ip pool POOL1
[R1-ip-pool-POOL1] network 192.168.1.0 mask 255.255.255.0
[R1-ip-pool-POOL1] gateway-list 192.168.1.1
[R1-ip-pool-POOL1] dns-list 8.8.8.8 114.114.114.114
[R1-ip-pool-POOL1] excluded-ip-address 192.168.1.1 192.168.1.10   # 排除地址范围
[R1-ip-pool-POOL1] lease day 3                                     # 租期3天
[R1-ip-pool-POOL1] quit

# 接口启用
[R1] interface GigabitEthernet 0/0/0
[R1-GigabitEthernet0/0/0] dhcp select global
```

### 基于接口地址池（简单场景）

```bash
[R1] dhcp enable
[R1] interface GigabitEthernet 0/0/0
[R1-GigabitEthernet0/0/0] ip address 192.168.1.1 255.255.255.0
[R1-GigabitEthernet0/0/0] dhcp select interface
[R1-GigabitEthernet0/0/0] dhcp server dns-list 8.8.8.8
[R1-GigabitEthernet0/0/0] dhcp server excluded-ip-address 192.168.1.1 192.168.1.10
```

### DHCP中继（跨网段）

```bash
[R1] dhcp enable
[R1] interface GigabitEthernet 0/0/0     # 连接客户端的接口
[R1-GigabitEthernet0/0/0] dhcp select relay
[R1-GigabitEthernet0/0/0] dhcp relay server-ip 10.0.0.100   # 指定 DHCP 服务器 IP
```

### 查看命令

```bash
[R1] display ip pool                           # 查看地址池信息
[R1] display ip pool name POOL1 used           # 查看已分配地址
[R1] display dhcp server statistics            # DHCP 服务器统计
```

---

## ACL访问控制列表

### 基本 ACL（2000-2999，基于源IP）

```bash
# 拒绝单个主机
[R1] acl 2000
[R1-acl-basic-2000] rule 5 deny source 192.168.1.100 0

# 允许网段
[R1-acl-basic-2000] rule 10 permit source 192.168.1.0 0.0.0.255
```

### 高级ACL（3000-3999，基于五元组）

```bash
[R1] acl 3000
# 拒绝某主机访问某服务器的 TCP 80端口
[R1-acl-adv-3000] rule 5 deny tcp source 192.168.1.100 0 destination 10.0.0.1 0 destination-port eq 80

# 允许其他网段 HTTP 访问
[R1-acl-adv-3000] rule 10 permit tcp source 192.168.1.0 0.0.0.255 destination any destination-port eq 80

# 拒绝 ICMP（禁止ping）
[R1-acl-adv-3000] rule 15 deny icmp source 192.168.1.0 0.0.0.255 destination any
```

### ACL应用

```bash
# 在接口上应用（入方向）
[R1-GigabitEthernet0/0/0] traffic-filter inbound acl 3000

# 在接口上应用（出方向）
[R1-GigabitEthernet0/0/0] traffic-filter outbound acl 3000

# 在 VTY 中应用（限制远程登录）
[R1] user-interface vty 0 4
[R1-ui-vty0-4] acl 2000 inbound
```

### 查看命令

```bash
[R1] display acl 3000                       # 查看ACL规则
[R1] display traffic-filter applied-record  # 查看ACL应用情况
```

---

## NAT网络地址转换

### 静态NAT（一对一）

```bash
[R1] interface GigabitEthernet 0/0/1                  # 公网接口
[R1-GigabitEthernet0/0/1] nat static global 200.1.1.100 inside 192.168.1.10
```

### 动态NAT

```bash
# 定义地址池
[R1] nat address-group 1 200.1.1.50 200.1.1.100

# 定义需转换的内网（ACL）
[R1] acl 2000
[R1-acl-basic-2000] rule 5 permit source 192.168.1.0 0.0.0.255

# 接口应用
[R1-GigabitEthernet0/0/1] nat outbound 2000 address-group 1
```

### NAPT（多对一，最常用）

```bash
# Easy IP 方式（直接使用公网接口 IP）
[R1] acl 2000
[R1-acl-basic-2000] rule 5 permit source 192.168.1.0 0.0.0.255
[R1-acl-basic-2000] quit

[R1] interface GigabitEthernet 0/0/1
[R1-GigabitEthernet0/0/1] nat outbound 2000       # Easy IP
```

### NAT Server（端口映射，内网服务器发布）

```bash
# 将公网IP的80端口映射到内网服务器
[R1-GigabitEthernet0/0/1] nat server protocol tcp global 200.1.1.1 80 inside 192.168.1.10 80

# 映射 FTP（20,21端口）
[R1-GigabitEthernet0/0/1] nat server protocol tcp global 200.1.1.1 ftp inside 192.168.1.10 ftp
```

### 查看命令

```bash
[R1] display nat session all             # 查看 NAT 会话表
[R1] display nat outbound                # 查看出方向 NAT 配置
[R1] display nat server                  # 查看 NAT Server 映射
[R1] display nat statistics              # NAT 统计信息
```

---

## VRRP虚拟路由冗余协议

### 基本配置

```bash
# 主路由器 R1
[R1] interface Vlanif 10
[R1-Vlanif10] ip address 192.168.1.1 255.255.255.0
[R1-Vlanif10] vrrp vrid 1 virtual-ip 192.168.1.254     # 虚拟 IP
[R1-Vlanif10] vrrp vrid 1 priority 120                  # 优先级（默认100，越大越优先）
[R1-Vlanif10] vrrp vrid 1 preempt-mode timer delay 5    # 抢占模式，延时5秒
[R1-Vlanif10] vrrp vrid 1 track interface GigabitEthernet 0/0/1 reduced 30  # 上行链路跟踪

# 备份路由器 R2
[R2] interface Vlanif 10
[R2-Vlanif10] ip address 192.168.1.2 255.255.255.0
[R2-Vlanif10] vrrp vrid 1 virtual-ip 192.168.1.254     # 同一 VRID，同一虚拟 IP
[R2-Vlanif10] vrrp vrid 1 priority 100                  # 或者不设置（默认100）
```

### 查看命令

```bash
[R1] display vrrp                          # 查看 VRRP 状态
[R1] display vrrp brief                    # 简要信息
[R1] display vrrp interface Vlanif 10      # 指定接口的 VRRP
```

---

## Telnet / SSH远程管理

### Telnet配置

```bash
# 服务端配置
[R1] telnet server enable
[R1] user-interface vty 0 4
[R1-ui-vty0-4] authentication-mode password
[R1-ui-vty0-4] set authentication password cipher huawei123
[R1-ui-vty0-4] protocol inbound telnet       # 仅允许 Telnet
[R1-ui-vty0-4] user privilege level 3        # 登录后权限级别

# 客户端连接
<R2> telnet 192.168.1.1
```

### SSH配置（推荐）

```bash
# 服务端配置
[R1] stelnet server enable
[R1] rsa local-key-pair create               # 生成密钥对
[R1] ssh user admin authentication-type password
[R1] ssh user admin service-type stelnet

[R1] user-interface vty 0 4
[R1-ui-vty0-4] authentication-mode aaa
[R1-ui-vty0-4] protocol inbound ssh

# 创建本地用户
[R1] aaa
[R1-aaa] local-user admin password cipher huawei123
[R1-aaa] local-user admin privilege level 3
[R1-aaa] local-user admin service-type ssh

# 客户端连接
<R2> ssh client first-time enable             # 首次连接
<R2> stelnet 192.168.1.1
```

### 查看命令

```bash
[R1] display users                            # 查看在线用户
[R1] display ssh server status                # SSH 服务状态
[R1] display ssh server session               # SSH 会话信息
```

---

## PPP与广域网

### PPP基本配置

```bash
# 两端均需配置
[R1] interface Serial 1/0/0
[R1-Serial1/0/0] link-protocol ppp
[R1-Serial1/0/0] ip address 10.0.0.1 255.255.255.252
```

### PAP认证

```bash
# 被认证方
[R1-Serial1/0/0] ppp pap local-user R1 password cipher huawei123

# 认证方
[R2] aaa
[R2-aaa] local-user R1 password cipher huawei123
[R2-aaa] local-user R1 service-type ppp
[R2-aaa] quit
[R2] interface Serial 1/0/0
[R2-Serial1/0/0] ppp authentication-mode pap
```

### CHAP认证（更安全）

```bash
# 认证方
[R2] aaa
[R2-aaa] local-user R1 password cipher huawei123
[R2-aaa] local-user R1 service-type ppp
[R2-aaa] quit
[R2] interface Serial 1/0/0
[R2-Serial1/0/0] ppp authentication-mode chap

# 被认证方
[R1-Serial1/0/0] ppp chap user R1
[R1-Serial1/0/0] ppp chap password cipher huawei123
```

---

## 常用查询与排错命令

### 接口与设备信息

```bash
display version                          # 查看设备版本
display device                           # 查看设备硬件信息（单板、电源等）
display current-configuration            # 查看当前生效配置
display saved-configuration              # 查看已保存配置
display this                             # 查看当前视图下的配置
display ip interface brief               # 查看接口 IP 简要信息 ★常用
display interface brief                  # 查看所有接口简要状态
display interface GigabitEthernet 0/0/0  # 查看指定接口详细信息
display cpu-usage                        # CPU 使用率
display memory-usage                     # 内存使用率
```

### 路由相关

```bash
display ip routing-table                  # 查看路由表 ★常用
display ip routing-table protocol static  # 只看静态路由
display ip routing-table protocol ospf    # 只看OSPF路由
display fib                               # 查看FIB转发表
```

### ARP与MAC

```bash
display arp                                   # 查看 ARP 表 ★常用
display arp all                               # 查看所有 ARP 条目
display mac-address                           # MAC地址表（交换机）★常用
display mac-address vlan 10                   # 指定 VLAN 的 MAC 表
display mac-address dynamic                   # 查看动态 MAC
```

### 连通性测试

```bash
ping 192.168.1.1                              # 测试连通性 ★最常用
ping -c 10 192.168.1.1                        # 发送10个包
ping -s 1500 192.168.1.1                      # 指定数据包大小
ping -a 192.168.1.1 192.168.2.1               # 指定源IP

tracert 192.168.2.1                           # 追踪路由路径 ★常用
tracert -d 192.168.2.1                        # 不解析域名
```

### 排错诊断

```bash
display logbuffer                             # 查看日志缓冲区
display trapbuffer                            # 查看告警缓冲区
reset logbuffer                               # 清除日志缓冲区
terminal debugging                            # 开启终端调试显示
terminal monitor                              # 开启终端监控
debugging ip icmp                             # 开启 ICMP 调试（慎用）
display debugging                             # 查看当前调试开关
undo debugging all                            # 关闭所有调试
```

### LLDP（邻居发现）

```bash
lldp enable                                   # 全局开启 LLDP
display lldp neighbor                         # 查看 LLDP 邻居 ★常用
display lldp neighbor brief                   # 简要信息
display lldp neighbor interface GigabitEthernet 0/0/1
```

---

## 设备保存与文件管理

```bash
# 保存配置
save                                          # 保存当前配置到 flash
save config.cfg                               # 另存为指定文件名
compare configuration                         # 比较当前配置与已保存配置的差异

# 文件管理
dir                                           # 查看文件列表
dir flash:/                                   # 查看 flash 目录
copy flash:/vrpcfg.zip flash:/vrpcfg_bak.zip  # 复制文件
delete flash:/old.cfg                         # 删除文件（进入回收站）
reset recycle-bin                             # 清空回收站
undelete                                       # 恢复删除文件

# 恢复出厂设置
reset saved-configuration                     # 清除已保存配置
reboot                                        # 重启设备（重启后恢复出厂）
```

---

## eNSP使用技巧与排错

### eNSP常见问题

| 问题 | 原因 | 解决方法 |
|------|------|---------|
| 设备启动失败（###） | VirtualBox未注册或版本不兼容 | 确保 VirtualBox 版本与 eNSP 兼容（建议 5.2.x），管理员身份运行 eNSP |
| 设备一直显示"正在启动" | 虚拟网卡被占用 | 重新注册 esight 工具，或重启 eNSP |
| AR 路由器无法启动 | 内存不足 | 关闭不必要设备，减少拓扑规模 |
| 抓包无数据 | 未在接口右键选择"抓包" | 正确操作：右键端口 → 选择"开始数据抓包" |

### 工作流程建议

1. **先画拓扑**，再配置IP地址表
2. **自底向上**配置：先二层（VLAN/STP/Trunk），再三层（IP/路由）
3. **边配边测**：`ping`每一步都验证
4. **保存配置**：每个关键阶段`save`
5. **导出配置**：右键设备 → 导出配置，方便复习

### 在PC上绑定IP

```bash
# eNSP中的PC设备命令
ipconfig /all        # 查看IP
ping 192.168.1.1     # 测试连通性
tracert 192.168.2.1  # 追踪路由
```

---

## 附录：命令速查表

### 最常用 Top 20

| 序号 | 命令 | 用途 |
|------|------|------|
| 1 | `system-view` | 进入系统视图 |
| 2 | `sysname R1` | 修改设备名 |
| 3 | `interface GigabitEthernet 0/0/0` | 进入接口视图 |
| 4 | `ip address x.x.x.x x.x.x.x` | 配置接口 IP |
| 5 | `undo shutdown` | 开启接口 |
| 6 | `vlan batch 10 20` | 批量创建 VLAN |
| 7 | `port link-type trunk` | 设置 Trunk 模式 |
| 8 | `port default vlan 10` | 设置 Access VLAN |
| 9 | `ip route-static 目标 掩码 下一跳` | 添加静态路由 |
| 10 | `ospf 1` | 进入 OSPF 进程 |
| 11 | `area 0` | 进入 OSPF 区域 |
| 12 | `acl 2000` / `acl 3000` | 创建 ACL |
| 13 | `nat outbound 2000` | 配置 Easy IP NAT |
| 14 | `dhcp enable` | 开启 DHCP |
| 15 | `user-interface vty 0 4` | 进入 VTY 配置 |
| 16 | `display ip routing-table` | 查看路由表 |
| 17 | `display ip interface brief` | 查看接口 IP 摘要 |
| 18 | `display vlan` | 查看 VLAN |
| 19 | `ping x.x.x.x` | 测试连通性 |
| 20 | `save` | 保存配置 |

### 接口命名速查

| 接口类型 | 命名格式 | 示例 |
|----------|---------|------|
| 千兆以太口 | `GigabitEthernet x/x/x` | `GigabitEthernet 0/0/0` |
| 万兆以太口 | `XGigabitEthernet x/x/x` | `XGigabitEthernet 0/0/1` |
| 串行接口 | `Serial x/x/x` | `Serial 1/0/0` |
| 回环接口 | `LoopBack x` | `LoopBack 0` |
| VLAN 虚接口 | `Vlanif x` | `Vlanif 10` |
| 聚合接口 | `Eth-Trunk x` | `Eth-Trunk 1` |
| 子接口 | `GigabitEthernet x/x/x.VID` | `GigabitEthernet 0/0/0.10` |
| NULL 接口 | `NULL 0` | `NULL 0` |

### 常用协议号与端口号

| 端口号 | 协议/服务 |
|--------|----------|
| 20/21 | FTP |
| 22 | SSH |
| 23 | Telnet |
| 25 | SMTP |
| 53 | DNS |
| 67/68 | DHCP |
| 80 | HTTP |
| 110 | POP3 |
| 161 | SNMP |
| 443 | HTTPS |
| 520 | RIP |

---

> **学习建议**：先掌握 VLAN、静态路由、OSPF、ACL、NAT 五大核心模块，再逐步扩展到 DHCP、VRRP、STP 等高级功能。多动手搭拓扑，遇到问题多用 `display` 类命令排查。
