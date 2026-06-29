# DHCP 学习笔记（Dynamic Host Configuration Protocol）

# 1. DHCP 概述

## 1.1 什么是 DHCP

DHCP（Dynamic Host Configuration Protocol，动态主机配置协议）是一种应用层协议，用于自动为网络中的主机分配：

- IP 地址
- 子网掩码（Subnet Mask）
- 默认网关（Default Gateway）
- DNS 服务器地址
- 租约时间（Lease Time）
- 其他网络参数

DHCP 的主要目的是：

> 实现 IP 地址的自动管理，减少人工配置工作量，避免 IP 地址冲突。

---

# 2. DHCP 的作用

## 手工配置 IP 的问题

假设一个公司有：

- 500 台电脑
- 100 台打印机
- 50 台服务器

如果全部手动配置：

- 工作量巨大
- 容易输错
- IP 冲突频繁
- 更换网络时配置困难

因此需要 DHCP 自动分配。

---

# 3. DHCP 工作模式

DHCP 采用：

- Client/Server（客户端/服务器）模式

组成：

### DHCP Client

需要获取 IP 地址的终端：

- PC
- 手机
- 平板
- 虚拟机

### DHCP Server

负责分配 IP 地址。

例如：

- Windows Server DHCP
- Linux DHCP Server
- 路由器 DHCP 服务
- 企业交换机 DHCP 服务

---

# 4. DHCP 使用的端口

DHCP 使用 UDP 协议。

| 设备 | 端口 |
|------|------|
| DHCP Client | UDP 68 |
| DHCP Server | UDP 67 |

记忆：

```text
客户端：68
服务器：67
```

---

# 5. DHCP 工作原理（重点）

DHCP 采用 DORA 四步过程：

```text
Discover
Offer
Request
Ack
```

简称：

```text
DORA
```

---

# 6. DORA 详细流程

## 第一步：Discover（发现）

客户端刚开机：

```text
IP：0.0.0.0
```

不知道：

- 自己的 IP
- DHCP 服务器地址

因此发送：

```text
DHCP Discover
```

特点：

```text
源IP：0.0.0.0
目标IP：255.255.255.255
```

广播发送。

---

## 第二步：Offer（提供）

DHCP Server 收到广播后：

提供一个可用 IP。

例如：

```text
192.168.1.100
```

发送：

```text
DHCP Offer
```

内容包括：

- IP地址
- 子网掩码
- 网关
- DNS
- 租约时间

---

## 第三步：Request（请求）

客户端收到 Offer 后：

向服务器发送：

```text
DHCP Request
```

表示：

```text
我要使用这个 IP。
```

仍然采用广播。

---

## 第四步：ACK（确认）

服务器发送：

```text
DHCP ACK
```

表示：

```text
IP地址正式租给客户端。
```

客户端开始使用该地址。

---

# 7. DORA 流程图

```text
Client                              Server
   |------Discover-------------------->|
   |<-------Offer----------------------|
   |------Request--------------------->|
   |<--------ACK-----------------------|
```

考试和面试经常考。

---

# 8. DHCP 地址租约（Lease）

DHCP 分配的 IP 不是永久的。

例如：

```text
Lease Time = 8 天
```

到期后需要续租。

---

# 9. DHCP 续租机制（重点）

## T1 时间

默认：

```text
50%
```

例如：

租约：

```text
8天
```

T1：

```text
4天
```

客户端向原服务器单播续租。

---

## T2 时间

默认：

```text
87.5%
```

例如：

```text
7天
```

若原服务器没有响应：

客户端广播寻找其他 DHCP Server。

---

## 租约到期

如果仍然无法续租：

```text
IP地址失效
```

客户端重新执行 DORA 流程。

---

# 10. DHCP 地址池（Pool）

DHCP Server 管理一个地址池。

例如：

```text
192.168.1.100
~
192.168.1.200
```

可分配：

```text
101 个地址
```

---

# 11. 地址池组成

通常包括：

```text
网络地址
子网掩码
网关
DNS
租约时间
排除地址
```

---

# 12. DHCP 地址分配方式

## 1）自动分配（Automatic Allocation）

永久分配。

---

## 2）动态分配（Dynamic Allocation）

最常见。

有租约时间。

---

## 3）手工分配（Manual Allocation）

又叫：

```text
静态绑定
MAC绑定
地址保留
```

根据 MAC 分配固定 IP。

例如：

```text
00-11-22-33-44-55
↓

192.168.1.10
```

---

# 13. DHCP 中继（DHCP Relay）★★★★★

## 为什么需要 DHCP Relay？

广播不能跨三层设备。

例如：

```text
PC ---------- Router ---------- DHCP Server
```

Discover 广播：

```text
255.255.255.255
```

路由器不会转发。

因此：

客户端无法获取 IP。

---

## 解决办法

配置：

```text
DHCP Relay
```

或者：

```text
ip helper-address
```

---

## DHCP Relay 工作过程

```text
PC
 ↓
广播Discover
 ↓
路由器Relay
 ↓
单播给DHCP Server
 ↓
Server回复
 ↓
Relay转发给PC
```

---

# 14. DHCP Relay 工作图

```text
PC ---- SW ---- Router ---- DHCP Server
             DHCP Relay
```

企业网络大量使用。

---

# 15. DHCP 常见配置参数

## IP Address

```text
192.168.1.100
```

## Subnet Mask

```text
255.255.255.0
```

## Default Gateway

```text
192.168.1.1
```

## DNS

```text
8.8.8.8
114.114.114.114
```

## Lease

```text
8 days
```

---

# 16. DHCP 常见报文

| 报文 | 作用 |
|------|------|
| Discover | 查找服务器 |
| Offer | 提供IP |
| Request | 请求IP |
| ACK | 确认租约 |
| NAK | 拒绝请求 |
| Release | 释放地址 |
| Decline | 拒绝地址 |
| Inform | 获取其他参数 |

---

# 17. DHCP NAK

如果：

```text
IP地址已经失效
```

或者：

```text
请求非法
```

服务器发送：

```text
DHCP NAK
```

客户端必须重新申请。

---

# 18. DHCP Release

客户端主动释放：

```text
ipconfig /release
```

服务器回收地址。

---

# 19. DHCP Renew

Windows：

```cmd
ipconfig /renew
```

重新申请租约。

---

# 20. 查看 DHCP 信息

## Windows

查看：

```cmd
ipconfig /all
```

查看 DHCP：

```cmd
ipconfig
```

释放：

```cmd
ipconfig /release
```

更新：

```cmd
ipconfig /renew
```

---

## Linux

查看：

```bash
ip addr
```

查看租约：

```bash
cat /var/lib/dhcp/*.leases
```

重新获取：

```bash
dhclient
```

---

# 21. DHCP 抓包分析（重点）

过滤条件：

```text
bootp
```

或者：

```text
dhcp
```

典型顺序：

```text
Discover
Offer
Request
ACK
```

考试经常给抓包图让判断。

---

# 22. DHCP 优点

## 自动管理

减少运维工作。

## 防止 IP 冲突

统一分配。

## 便于迁移

更换网络无需重新配置。

## 集中管理

统一配置 DNS、网关。

---

# 23. DHCP 缺点

## 依赖服务器

服务器故障：

```text
新客户端无法获取IP。
```

## 安全风险

非法 DHCP Server：

```text
DHCP Spoofing
```

---

# 24. DHCP 攻击

## DHCP Starvation（地址耗尽攻击）

攻击者不断申请地址：

```text
地址池耗尽
```

正常用户无法获取 IP。

---

## DHCP Spoofing（伪造服务器）

攻击者伪造 DHCP Server：

下发错误：

- 网关
- DNS

导致：

- 中间人攻击
- 流量劫持

---

# 25. DHCP 安全技术

## DHCP Snooping★★★★★

交换机只允许：

```text
Trusted Port
```

发送 DHCP Server 报文。

非法服务器直接丢弃。

---

## 生成绑定表

记录：

- MAC
- IP
- VLAN
- Interface

可用于：

- IP Source Guard
- Dynamic ARP Inspection

---

# 26. DHCP Snooping 端口

## Trusted

连接：

```text
DHCP Server
```

## Untrusted

连接：

```text
普通用户
```

默认都是：

```text
Untrusted
```

---

# 27. DHCP 常见面试题

### DHCP 使用什么协议？

```text
UDP
```

---

### DHCP 端口是多少？

```text
Server：67
Client：68
```

---

### DHCP 四个过程是什么？

```text
Discover
Offer
Request
ACK
```

---

### 为什么 Discover 使用广播？

因为客户端：

```text
没有IP
不知道DHCP服务器地址
```

---

### 为什么需要 DHCP Relay？

```text
广播不能跨路由器。
```

---

### DHCP 与静态 IP 的区别？

DHCP：

```text
自动分配。
```

静态：

```text
人工配置。
```

---

# 28. 企业实际部署建议

## 服务器

固定 IP。

---

## 网络设备

固定 IP。

---

## 打印机

固定 IP。

---

## 办公网终端

DHCP。

---

## 无线终端

DHCP。

---

# 29. DHCP 核心知识总结

```text
作用：
自动分配IP。

协议：
UDP。

端口：
67/68。

流程：
DORA。

Discover：
广播。

地址：
租约机制。

跨网段：
DHCP Relay。

安全：
DHCP Snooping。

攻击：
Starvation
Spoofing。
```

---

# 30. 一张图记住 DHCP

```text
Client                    Server
0.0.0.0

Discover  ------------->

             <----------- Offer

Request   ------------->

             <----------- ACK

获得：
IP
Mask
Gateway
DNS
Lease
```

---

# 学习口诀

```text
DHCP四步别记差，
Discover先广播；
Offer提供可用IP，
Request申请别弄丢；
ACK确认正式用，
67、68记心头；
跨网段用Relay，
安全靠Snooping守。
```
