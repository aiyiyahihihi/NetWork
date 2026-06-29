# DNS（Domain Name System）学习笔记

> 域名系统 —— 互联网的电话簿，将人类可读的域名转换为机器可读的 IP 地址。

---

## 目录

1. [概述与历史](#1-概述与历史)
2. [域名层级结构](#2-域名层级结构)
3. [DNS 解析流程](#3-dns-解析流程)
4. [DNS 服务器类型](#4-dns-服务器类型)
5. [DNS 记录类型](#5-dns-记录类型)
6. [DNS 协议与报文格式](#6-dns-协议与报文格式)
7. [DNS 缓存机制](#7-dns-缓存机制)
8. [DNS 安全](#8-dns-安全)
9. [高级主题](#9-高级主题)
10. [常用命令与工具](#10-常用命令与工具)
11. [故障排查指南](#11-故障排查指南)
12. [最佳实践](#12-最佳实践)

---

## 1. 概述与历史

### 1.1 什么是 DNS

DNS（Domain Name System，域名系统）是一个**分层、分布式**的命名系统，核心功能是：

```
域名 (www.example.com)  ──解析──▶  IP 地址 (93.184.216.34)
```

### 1.2 为什么需要 DNS

| 问题 | 说明 |
|------|------|
| 人类记忆 | 人类难以记忆大量 IP 地址（如 `142.250.80.46`），域名更直观 |
| IP 可变性 | 服务器 IP 可能变更，域名提供稳定的访问入口 |
| 负载均衡 | 同一域名可解析到多个 IP，实现流量分发 |
| 服务抽象 | 域名与底层基础设施解耦 |

### 1.3 历史演进

| 时期 | 事件 |
|------|------|
| 1970s | ARPANET 时代，使用 `HOSTS.TXT` 文件（单点维护，不可扩展） |
| 1983 | Paul Mockapetris 发布 RFC 882/883，提出 DNS 设计 |
| 1987 | RFC 1034/1035 发布，成为现代 DNS 基础规范 |
| 1997 | RFC 2065 引入 DNSSEC 概念 |
| 2010s | 根区正式部署 DNSSEC |
| 2016 | DNS over TLS (RFC 7858) |
| 2018 | DNS over HTTPS (RFC 8484) |

### 1.4 设计目标

1. **分布式**：无单点故障，分级委派
2. **一致性**：全局统一的命名空间
3. **缓存**：减少解析延迟和根/顶级服务器负载
4. **可扩展**：支持新记录类型和功能

---

## 2. 域名层级结构

### 2.1 树形命名空间

```
                        [根域 "."]
                            │
        ┌───────────────────┼───────────────────┐
       com.              org.                cn.
        │                  │                   │
   example.com.      wikipedia.org.       gov.cn.
        │                                     │
   www.example.com.                     beijing.gov.cn.
```

### 2.2 域名组成

```
┌─────────────────────────────────────────────┐
│  https://www.example.com.cn:443/path?q=1    │
└─────────────────────────────────────────────┘
          │                  │
          └── FQDN ─────────┘
          www.example.com.cn.
           │   │           │
           │   │           └── 顶级域 (TLD): cn
           │   └── 二级域 (SLD): example
           └── 子域 (Subdomain): www
```

#### 完全限定域名（FQDN）

- 以点 `.` 结尾表示绝对域名，如 `www.example.com.`
- 不以点结尾为相对域名，解析时系统会自动补全

### 2.3 域名的层级划分

| 层级 | 说明 | 示例 |
|------|------|------|
| **根域** | 用 `.` 表示，13 组根服务器（字母 A-M） | `.` |
| **顶级域 (TLD)** | 通用 TLD + 国家/地区代码 TLD | `.com`, `.cn`, `.org` |
| **二级域** | 用户注册的域名 | `example.com` |
| **子域** | 域所有者自行划分 | `www.example.com` |
| **主机名** | 最左侧标签 | `www` |

### 2.4 TLD 分类

```
顶级域 (TLD)
├── 通用顶级域 (gTLD)
│   ├── 非赞助: .com .net .org .info .biz
│   ├── 赞助: .edu .gov .mil .int
│   └── 新 gTLD: .app .dev .blog .shop .xyz ...
├── 国家/地区代码顶级域 (ccTLD)
│   └── .cn .us .uk .jp .de .kr .hk .tw ...
└── 基础设施 TLD: .arpa (用于反向解析)
```

### 2.5 域名命名规则

- 每个标签最长 **63 个字符**
- 完整域名最长 **253 个字符**（含点号）
- 允许字符：字母 `a-z`、数字 `0-9`、连字符 `-`
- 标签不能以连字符开头或结尾
- **国际化域名（IDN）**：通过 Punycode 编码支持非 ASCII 字符（如中文域名）

---

## 3. DNS 解析流程

### 3.1 递归解析与迭代解析

```
┌──────────┐    递归查询     ┌──────────┐
│  客户端   │ ◄────────────► │ 本地 DNS  │
│ (Stub)   │    （一次请求    │ 解析器    │
└──────────┘    一次应答）    └────┬─────┘
                                  │
                     迭代查询      │
              ┌───────────────────┼────────────┐
              ▼                   ▼            ▼
         ┌────────┐          ┌────────┐   ┌────────┐
         │根服务器│          │TLD 服务│   │权威服务│
         └────────┘          └────────┘   └────────┘
```

### 3.2 完整迭代解析过程（解析 `www.example.com`）

```
步骤 1: 客户端 → 本地 DNS 解析器 (递归)
  "请解析 www.example.com"

步骤 2: 本地解析器 → 根服务器 (迭代)
  请求: "www.example.com 的 A 记录？"
  根服务器: "我不知道，但 .com 的权威服务器在 a.gtld-servers.net (192.5.6.30)"

步骤 3: 本地解析器 → .com TLD 服务器 (迭代)
  请求: "www.example.com 的 A 记录？"
  TLD 服务器: "我不知道，但 example.com 的权威 DNS 在 ns1.example.com"

步骤 4: 本地解析器 → example.com 权威服务器 (迭代)
  请求: "www.example.com 的 A 记录？"
  权威服务器: "www.example.com 的 A 记录是 93.184.216.34"

步骤 5: 本地解析器 → 客户端 (返回结果)
  "www.example.com 解析为 93.184.216.34"
```

### 3.3 优化：缓存加速

```
实际解析流程中，步骤 1 之前先检查：
┌──────────────────────────────────┐
│ 1. 浏览器 DNS 缓存              │
│ 2. 操作系统 DNS 缓存 (hosts)    │
│ 3. 本地 DNS 解析器缓存          │
└──────────────────────────────────┘
如果任一命中缓存，直接返回，无需递归查询。
```

### 3.4 解析类型对比

| 类型 | 请求方 → 服务器 | 服务器行为 |
|------|----------------|-----------|
| **递归查询** | 客户端 → 本地解析器 | 服务器负责完整解析，返回最终结果 |
| **迭代查询** | 解析器 → 各级服务器 | 服务器只返回下一步线索（Referral） |
| **反向查询** | PTR 记录查询 | IP → 域名 |

---

## 4. DNS 服务器类型

### 4.1 四种服务器角色

```
┌─────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  DNS 解析器  │  │  根服务器     │  │  TLD 服务器   │  │ 权威 DNS 服务器│
│ (Resolver)  │  │ (Root Server) │  │(TLD Server)  │  │(Authoritative)│
├─────────────┤  ├──────────────┤  ├──────────────┤  ├──────────────┤
│ 代表客户端   │  │ 13 组全球部署 │  │ 管理每个 TLD  │  │ 存放域名的   │
│ 完成完整解析 │  │ 响应根区查询  │  │ 的权威信息    │  │ 最终 DNS 记录│
│ 通常由 ISP   │  │              │  │              │  │              │
│ 或公共 DNS   │  │              │  │              │  │              │
│ 提供        │  │              │  │              │  │              │
└─────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
```

### 4.2 根服务器（Root Servers）

- 共 **13 组**根服务器，命名为 `a.root-servers.net` 到 `m.root-servers.net`
- 采用 **Anycast 任播技术**，全球部署超过 1600 个节点
- 由 12 个独立组织运营
- 根区文件由 **IANA** 管理

```
字母    运营组织                      IPv4
a       Verisign                    198.41.0.4
b       USC-ISI                     170.247.170.2
c       Cogent                      192.33.4.12
d       University of Maryland      199.7.91.13
e       NASA                        192.203.230.10
f       ISC                         192.5.5.241
g       US DoD NIC                  192.112.36.4
h       US Army Research Lab        198.97.190.53
i       Netnod                      192.36.148.17
j       Verisign                    192.58.128.30
k       RIPE NCC                    193.0.14.129
l       ICANN                       199.7.83.42
m       WIDE Project                202.12.27.33
```

### 4.3 权威 DNS 服务器

- 存放域名的**权威记录**（SOA 记录标识权威来源）
- 由域名所有者或 DNS 托管服务商维护
- 分为**主服务器（Master/Primary）**和**辅助服务器（Slave/Secondary）**
- 辅助服务器通过**区域传输（Zone Transfer）**同步数据

### 4.4 常见公共 DNS 解析器

| 服务商 | 主 DNS | 备用 DNS | 特性 |
|--------|--------|---------|------|
| Google | `8.8.8.8` | `8.8.4.4` | 支持 DoH/DoT |
| Cloudflare | `1.1.1.1` | `1.0.0.1` | 隐私优先，WARP |
| Quad9 | `9.9.9.9` | `149.112.112.112` | 安全过滤 |
| 阿里 | `223.5.5.5` | `223.6.6.6` | 国内推荐 |
| 腾讯 DNSPod | `119.29.29.29` | `182.254.116.116` | 国内推荐 |
| 114DNS | `114.114.114.114` | `114.114.115.115` | 国内通用 |

---

## 5. DNS 记录类型

### 5.1 常用记录速查

| 类型 | 全称 | RR 值 | 用途 | 示例 |
|------|------|-------|------|------|
| **A** | Address | 1 | IPv4 地址记录 | `example.com. A 93.184.216.34` |
| **AAAA** | IPv6 Address | 28 | IPv6 地址记录 | `example.com. AAAA 2606:2800:220:1:248:1893:25c8:1946` |
| **CNAME** | Canonical Name | 5 | 别名记录（一个域名指向另一个域名） | `www.example.com. CNAME example.com.` |
| **MX** | Mail Exchange | 15 | 邮件服务器记录 | `example.com. MX 10 mail.example.com.` |
| **NS** | Name Server | 2 | 域的权威 DNS 服务器 | `example.com. NS ns1.example.com.` |
| **SOA** | Start of Authority | 6 | 域的管理信息（权威声明） | 见 5.2 |
| **TXT** | Text | 16 | 文本信息（SPF、DKIM、验证） | `example.com. TXT "v=spf1 ..."` |
| **PTR** | Pointer | 12 | 反向解析（IP → 域名） | `34.216.184.93.in-addr.arpa. PTR example.com.` |
| **SRV** | Service | 33 | 服务定位记录 | `_sip._tcp.example.com. SRV 10 60 5060 sip.example.com.` |
| **CAA** | Certification Authority Authorization | 257 | 指定可签发证书的 CA | `example.com. CAA 0 issue "letsencrypt.org"` |
| **NSEC** | Next Secure | 47 | DNSSEC 证明记录不存在 | 用于 DNSSEC |
| **RRSIG** | RR Signature | 46 | DNSSEC 数字签名 | 用于 DNSSEC |
| **DNSKEY** | DNS Key | 48 | DNSSEC 公钥 | 用于 DNSSEC |
| **DS** | Delegation Signer | 43 | DNSSEC 委派签名者 | 用于 DNSSEC |

### 5.2 SOA 记录详解

```
example.com.  IN  SOA  ns1.example.com. admin.example.com. (
                2024060101   ; 序列号（Serial）- 用于判断区域是否更新
                7200         ; 刷新间隔（Refresh）- 辅助服务器同步频率
                3600         ; 重试间隔（Retry）- 同步失败重试间隔
                1209600      ; 过期时间（Expire）- 辅助服务器数据有效期
                86400        ; 最小 TTL（Minimum TTL）- 否定缓存 TTL
)
```

### 5.3 MX 记录优先级

```
example.com.  MX  10  mail1.example.com.    ← 优先使用
example.com.  MX  20  mail2.example.com.    ← 备用1
example.com.  MX  30  mail3.example.com.    ← 备用2
```

数字越小，优先级越高。

### 5.4 CNAME 的限制

```
❌ 错误：CNAME 不能与其他记录共存
example.com.  CNAME  other.example.com.
example.com.  MX 10 mail.example.com.     ← 冲突！

✅ 正确：CNAME 仅用于子域
www.example.com.  CNAME  example.com.
```

> **关键规则**：RFC 规定，如果一条记录是 CNAME，则不能同时存在其他类型的同名记录（DNSSEC 相关记录除外）。

### 5.5 SRV 记录格式

```
_service._proto.name. TTL class SRV priority weight port target
        ──┬── ──┬─           ──┬── ──┬── ──┬─ ──┬──
        服务   协议           优先级 权重  端口  目标主机
```

示例：
```
_http._tcp.example.com.  86400  IN  SRV  10  5  80  webserver.example.com.
```

### 5.6 TXT 记录的常见用途

| 用途 | 格式示例 |
|------|---------|
| SPF（发件人策略） | `v=spf1 mx include:_spf.google.com ~all` |
| DKIM（域名密钥） | `v=DKIM1; k=rsa; p=MIGfMA0GCSq...` |
| DMARC | `v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com` |
| 域名验证 | `google-site-verification=xxxxx` |
| 泛用键值 | `key=value` |

---

## 6. DNS 协议与报文格式

### 6.1 传输协议

| 协议 | 端口 | 用途 |
|------|------|------|
| **UDP** | 53 | 标准查询（报文 < 512 字节） |
| **TCP** | 53 | 区域传输（AXFR/IXFR）、大响应（EDNS0 扩展后 > 512B） |
| **DoT** | 853 | DNS over TLS（加密） |
| **DoH** | 443 | DNS over HTTPS（加密） |

### 6.2 DNS 报文结构

```
┌─────────────────────────────────────────┐
│              Header (12 字节)             │
├─────────────────────────────────────────┤
│              Question Section             │  查询的问题
├─────────────────────────────────────────┤
│              Answer Section               │  回答的资源记录
├─────────────────────────────────────────┤
│              Authority Section            │  权威服务器的资源记录
├─────────────────────────────────────────┤
│              Additional Section           │  附加的资源记录
└─────────────────────────────────────────┘
```

#### Header 字段（12 字节）

```
 0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15
┌───────────────────┬───────────────────────────┐
│       ID          │ QR│ Opcode │AA│TC│RD│RA│ Z │  ← 标志位
├───────────────────┼───────────────────────────┤
│                  QDCOUNT                       │  ← 问题数
├───────────────────────────────────────────────┤
│                  ANCOUNT                       │  ← 回答数
├───────────────────────────────────────────────┤
│                  NSCOUNT                       │  ← 授权记录数
├───────────────────────────────────────────────┤
│                  ARCOUNT                       │  ← 附加记录数
└───────────────────────────────────────────────┘
```

| 标志位 | 含义 |
|--------|------|
| **QR** | 0=查询, 1=响应 |
| **AA** | Authoritative Answer（权威回答） |
| **TC** | Truncated（被截断，需用 TCP 重试） |
| **RD** | Recursion Desired（期望递归查询） |
| **RA** | Recursion Available（支持递归查询） |
| **RCODE** | 响应码（0=无错, 1=格式错, 2=服务器失败, 3=域名不存在 NXDOMAIN） |

### 6.3 EDNS0（RFC 6891）

- 扩展 DNS 报文大小限制（通过 OPT 伪记录）
- 支持更大的 UDP 报文（通常 1232-4096 字节）
- 支持 DNSSEC（返回 RRSIG 等大记录）
- 支持 Client Subnet（EDNS Client Subnet，传递客户端子网信息）

### 6.4 常见响应码（RCODE）

| RCODE | 名称 | 说明 |
|-------|------|------|
| 0 | NOERROR | 查询成功 |
| 1 | FORMERR | DNS 报文格式错误 |
| 2 | SERVFAIL | DNS 服务器内部错误 |
| 3 | **NXDOMAIN** | 域名不存在 |
| 4 | NOTIMP | 请求类型不支持 |
| 5 | REFUSED | 服务器拒绝应答 |

---

## 7. DNS 缓存机制

### 7.1 缓存层级

```
┌──────────────────┐
│   浏览器缓存      │  Chrome: chrome://net-internals/#dns
│   (1-5 分钟)     │
├──────────────────┤
│   操作系统缓存    │  Windows: ipconfig /displaydns
│   (TTL 决定)     │  Linux: systemd-resolved / nscd
├──────────────────┤
│   路由器缓存      │  家用路由器通常也缓存 DNS
├──────────────────┤
│   ISP/递归解析器  │  公共 DNS (8.8.8.8 / 1.1.1.1)
│   缓存           │
└──────────────────┘
```

### 7.2 TTL（Time To Live）

- TTL 由域名的权威服务器在 SOA 或具体记录中指定
- 表示记录在缓存中的有效时间（秒）
- **到期前**：缓存直接返回结果
- **到期后**：缓存服务器重新向权威服务器查询

```
常见 TTL 设置：
- 静态网站: 3600-86400 (1小时~24小时)
- 频繁变更: 300-600 (5~10分钟)
- CDN 切换需求: 60-300 (1~5分钟)
- 最低实用 TTL: 30 秒（更短会显著增加查询量）
```

### 7.3 否定缓存（Negative Caching）

- 当查询返回 **NXDOMAIN**（域名不存在）时也缓存该结果
- TTL 由 SOA 记录的 **Minimum TTL** 字段控制
- 防止对不存在域名的重复查询

---

## 8. DNS 安全

### 8.1 常见攻击与防御

| 攻击类型 | 描述 | 防御措施 |
|---------|------|---------|
| **DNS 缓存投毒** | 向解析器注入伪造的 DNS 响应 | DNSSEC、源端口随机化、查询 ID 随机化 |
| **DNS 劫持** | 篡改客户端/路由器 DNS 设置 | 使用加密 DNS（DoH/DoT） |
| **DNS 放大攻击** | 利用 DNS 响应比请求大的特性进行 DDoS | RRL（Response Rate Limiting）、BCP38 |
| **DNS 隧道** | 通过 DNS 协议传输非 DNS 数据 | 深度包检测、DNS 查询速率限制 |
| **域名劫持** | 未授权转移域名所有权 | 注册商锁定、多因素认证 |
| **中间人攻击** | 拦截并篡改 DNS 流量 | DNSSEC、DoH、DoT、DNSCrypt |

### 8.2 DNSSEC（DNS Security Extensions）

#### 核心目标

- **数据来源验证**：证明 DNS 记录来自正确的权威源
- **数据完整性**：证明 DNS 记录在传输中未被篡改
- **否定存在证明**：证明某个域名确实不存在

> **注意**：DNSSEC 不提供机密性（不加密），不防 DDoS，不防中间人（那是 DoT/DoH 的职责）。

#### 信任链（Chain of Trust）

```
根区 DNSKEY
    │ (DS 记录哈希验证)
    ▼
.com TLD DNSKEY
    │ (DS 记录哈希验证)
    ▼
example.com DNSKEY
    │ (DNSKEY 私钥签名)
    ▼
www.example.com A 记录的 RRSIG 签名
```

#### 关键记录

```
DNSKEY   — 区域的公钥（ZSK 用于签名记录，KSK 用于签名 DNSKEY）
DS       — 委派签名者，父域存放子域 KSK 的哈希
RRSIG    — 记录的数字签名
NSEC/NSEC3 — 证明某记录不存在的签名链
```

#### 签名验证流程

```
1. 请求 example.com 的 DNSKEY
2. 请求 .com 的 DS 记录（包含 example.com KSK 的哈希）
3. 验证 DS 哈希 == DNSKEY 的 KSK 哈希 ✓
4. 使用 DNSKEY 中的 ZSK 验证记录的 RRSIG 签名 ✓
5. 递归验证至根
```

### 8.3 加密 DNS

| 协议 | 全称 | 端口 | 特点 |
|------|------|------|------|
| **DoT** | DNS over TLS | 853 | 独立端口，易被防火墙识别 |
| **DoH** | DNS over HTTPS | 443 | 混入 HTTPS 流量，难以阻止 |
| **DoQ** | DNS over QUIC | 853 | 基于 QUIC，低延迟 |
| **DNSCrypt** | - | 自定义 | 较早期的加密 DNS 方案 |

### 8.4 电子邮件安全记录（SPF / DKIM / DMARC）

```
发件验证三角：

SPF（Sender Policy Framework）
└── 验证发件服务器 IP 是否被域名授权
    记录: v=spf1 include:_spf.google.com ~all

DKIM（DomainKeys Identified Mail）
└── 验证邮件内容未被篡改（数字签名）
    记录: v=DKIM1; k=rsa; p=<公钥>

DMARC（Domain-based Message Authentication）
└── 告诉收件服务器如何处理 SPF/DKIM 失败的邮件
    记录: v=DMARC1; p=quarantine; pct=100; rua=mailto:dmarc@example.com
```

---

## 9. 高级主题

### 9.1 智能 DNS（GeoDNS / Traffic Management）

```
用户请求 www.example.com
         │
         ▼
┌─────────────────────┐
│  智能 DNS 解析器     │
│  根据以下条件决策：   │
│  - 用户地理位置      │
│  - 服务器健康状态    │
│  - 当前负载         │
│  - 响应延迟         │
└──────┬──────────────┘
       │
   ┌───┼───┐
   ▼   ▼   ▼
 北京 上海 深圳
 节点 节点 节点
```

常见实现方式：
- **基于 IP 库的 GeoDNS**：根据请求来源 IP 返回最近节点
- **基于 EDNS Client Subnet**：传递客户端子网给权威 DNS 做精确决策
- **Anycast + 健康检查**：任播路由 + 后端健康探测

### 9.2 CDN 与 DNS 的关系

```
常规：DNS 返回源站 IP
CDN：DNS 返回 CDN 边缘节点 IP

用户 → DNS 解析 → CDN 边缘节点 IP → 边缘缓存 → (如需要) → 回源到源站
```

CDN 厂商（Cloudflare、Akamai、阿里云 CDN、腾讯云 CDN）通常要求用户将域名的 NS 记录指向 CDN 提供的 DNS 服务器，或使用 CNAME 方式接入。

### 9.3 区域传输（Zone Transfer）

- **AXFR**（Full Zone Transfer）：传输整个区域文件
- **IXFR**（Incremental Zone Transfer）：仅传输变更部分（RFC 1995）
- **安全**：应限制 AXFR 仅允许授权的辅助服务器 IP，防止信息泄露

```
主 DNS ── NOTIFY (区域有更新) ──▶ 辅助 DNS
主 DNS ◄── SOA 查询（获取序列号）── 辅助 DNS
主 DNS ── AXFR/IXFR (传输数据) ──▶ 辅助 DNS
```

### 9.4 动态 DNS（DDNS）

- 允许客户端动态更新 DNS 记录
- 常用于家庭宽带动态 IP 场景
- 协议：RFC 2136
- 常见服务商：DynDNS、No-IP、Cloudflare DDNS

### 9.5 DNS 负载均衡

```
example.com.  300  IN  A  1.2.3.4
example.com.  300  IN  A  5.6.7.8
example.com.  300  IN  A  9.10.11.12
```

- **DNS 轮询（Round Robin）**：返回多个 IP，客户端随机选择
- **局限性**：不考虑服务器真实负载、客户端可能缓存导致不均
- **改进**：配合健康检查的智能 DNS

### 9.6 反向 DNS 解析（PTR）

```
正向：域名 → IP
反向：IP → 域名（用于验证服务器身份，如邮件服务器反查）

IPv4 反向区域格式：
IP: 192.0.2.34
PTR 查询: 34.2.0.192.in-addr.arpa

IPv6 反向区域格式：
IP: 2001:db8::567:89ab
PTR 查询: b.a.9.8.7.6.5.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa
```

---

## 10. 常用命令与工具

### 10.1 基本查询工具

#### `nslookup` — 经典 DNS 查询

```bash
# 基本 A 记录查询
nslookup example.com

# 指定 DNS 服务器查询
nslookup example.com 8.8.8.8

# 查询 MX 记录
nslookup -type=MX example.com

# 查询所有记录
nslookup -type=ANY example.com

# 反向查询
nslookup 93.184.216.34

# 交互模式
nslookup
> set type=NS
> example.com
```

#### `dig` — 功能强大的 DNS 诊断工具

```bash
# 基本查询
dig example.com

# 简化输出（只显示答案）
dig example.com +short

# 查询指定记录类型
dig example.com AAAA
dig example.com MX
dig example.com NS
dig example.com TXT
dig example.com SOA
dig example.com CAA

# 指定 DNS 服务器
dig @8.8.8.8 example.com

# 追踪解析路径
dig example.com +trace

# 反向查询
dig -x 93.184.216.34

# 显示统计信息
dig example.com +stats

# DNSSEC 验证
dig example.com +dnssec

# 仅显示答案段
dig example.com +noall +answer

# 查询所有记录（可能被部分服务器过滤）
dig example.com ANY

# 批量查询
dig -f domains.txt +short
```

#### `host` — 简洁查询工具

```bash
host example.com
host -t MX example.com
host 93.184.216.34
host -a example.com   # 查询所有记录
```

### 10.2 Windows 专用命令

```cmd
REM 查看 DNS 缓存
ipconfig /displaydns

REM 清除 DNS 缓存
ipconfig /flushdns

REM 查看 DNS 配置
ipconfig /all

REM 注册 DNS（动态更新）
ipconfig /registerdns

REM 释放并更新 IP
ipconfig /release && ipconfig /renew
```

### 10.3 在线 DNS 工具

| 工具 | 用途 | 地址 |
|------|------|------|
| DNSdumpster | DNS 侦察 | dnsdumpster.com |
| dnschecker.org | 全球 DNS 传播检查 | dnschecker.org |
| whatsmydns.net | 全球 DNS 检查 | whatsmydns.net |
| DNSSEC Analyzer | DNSSEC 验证 | dnsviz.net |
| intoDNS | DNS 配置检查 | intodns.com |
| MXToolbox | 邮件/dns/黑名单检查 | mxtoolbox.com |

### 10.4 编程查询 API

**Python (dnspython)**：
```python
import dns.resolver

# A 记录查询
answers = dns.resolver.resolve('example.com', 'A')
for rdata in answers:
    print(f'A 记录: {rdata.address}')

# MX 记录查询
answers = dns.resolver.resolve('example.com', 'MX')
for rdata in answers:
    print(f'MX 记录: {rdata.preference} {rdata.exchange}')
```

**使用系统调用**：
```python
import socket
ip = socket.getaddrinfo('example.com', 80)
print(ip)
```

---

## 11. 故障排查指南

### 11.1 排查流程图

```
用户报告：网站无法访问
         │
         ▼
┌────────────────────┐
│ 检查本地 DNS 缓存   │  ipconfig /flushdns 后重试
└────────┬───────────┘
         │ 问题依旧
         ▼
┌────────────────────┐
│ 检查 HOSTS 文件     │  Windows: C:\Windows\System32\drivers\etc\hosts
│                     │  Linux/Mac: /etc/hosts
└────────┬───────────┘
         │ 无误
         ▼
┌────────────────────┐
│ 手动 DNS 查询       │  dig example.com @8.8.8.8
│ 是否返回正确 IP？   │  nslookup example.com 1.1.1.1
└────┬───────┬───────┘
     │       │
   正常     异常
     │       │
     ▼       ▼
┌────────┐ ┌──────────────────┐
│ 网络层  │ │ 换 DNS 服务器重试  │  dig @1.1.1.1 example.com
│ 排查    │ │ 对比不同 DNS 结果  │  dig @8.8.8.8 example.com
└────────┘ └────────┬─────────┘
                    │
              ┌─────┴─────┐
           都正常      仅某个DNS异常
              │             │
              ▼             ▼
        ┌──────────┐  ┌──────────────┐
        │ ISP 问题  │  │ 该 DNS 服务器 │
        │ 联系 ISP  │  │ 缓存或配置问题│
        └──────────┘  └──────────────┘

如果所有 DNS 都异常 → 域名可能到期或 DNS 配置错误
```

### 11.2 常见问题与解决

| 症状 | 可能原因 | 排查命令 |
|------|---------|---------|
| 无法解析任何域名 | DNS 服务器不可达、网络断开 | `ping 8.8.8.8` |
| 仅无法解析某个域名 | 域名过期、DNS 配置错误 | `dig NS example.com` |
| 解析到错误 IP | DNS 缓存投毒、hosts 文件 | `ipconfig /flushdns` |
| 间歇性解析失败 | DNS 服务器过载、网络抖动 | `dig +stats example.com` |
| 子域名不生效 | TTL 未过期、未配置 | `dig +trace sub.example.com` |
| DNSSEC 验证失败 | DNSSEC 配置错误 | `dig +dnssec example.com` |
| 部分地域慢/不通 | GeoDNS 或 CDN 问题 | dnschecker.org 检查 |
| 邮件被拒收 | SPF/DKIM/DMARC 未配置 | `dig TXT example.com` |

### 11.3 诊断技巧

```bash
# 1. 查看完整权威链
dig +trace example.com

# 2. 对比不同 DNS 服务器的结果
diff <(dig @8.8.8.8 example.com +short) <(dig @1.1.1.1 example.com +short)

# 3. 测试 DNSSEC
dig example.com +dnssec +multi

# 4. 检查 NS 一致性
dig NS example.com @a.gtld-servers.net   # 从 TLD 查看
dig NS example.com @ns1.example.com       # 从权威查看

# 5. 测量解析耗时
dig example.com +stats | grep "Query time"

# 6. 检查 CNAME 链
dig www.example.com +trace
```

---

## 12. 最佳实践

### 12.1 域名配置

```
✅ 始终配置至少 2 个 NS 服务器（不同地理位置/网络）
✅ SOA 中设置合理的 TTL 值
✅ 使用 CAA 记录限制证书颁发
✅ 配置 SPF、DKIM、DMARC 保护邮件域名
✅ 启用 DNSSEC（如支持）
✅ 定期检查域名到期日期，开启自动续费
✅ 使用注册商锁定防止未授权转移
```

### 12.2 TTL 设置建议

| 场景 | 推荐 TTL | 原因 |
|------|---------|------|
| 稳定生产环境 | 3600-86400 | 减少查询量，提高缓存命中 |
| 计划变更期间 | 300 | 变更前先降低 TTL，确保快速生效 |
| CDN / 故障切换 | 60-300 | 快速切换节点 |
| 开发/测试环境 | 30-300 | 灵活调整 |
| Apex/裸域 | 300-3600 | 平衡稳定性与灵活性 |

### 12.3 安全加固清单

- [ ] 使用 DNSSEC 签名区域
- [ ] 启用 DoH 或 DoT（客户端侧）
- [ ] 限制区域传输（仅允许授权的辅助服务器 IP）
- [ ] 配置 CAA 记录
- [ ] 配置 SPF、DKIM、DMARC
- [ ] 使用注册商提供的安全功能（锁定、两步验证）
- [ ] 监控 DNS 解析异常
- [ ] 定期审计 DNS 记录

### 12.4 区域文件示例

```
$ORIGIN example.com.
$TTL 86400

; SOA 记录（必须）
@   IN  SOA  ns1.example.com. admin.example.com. (
        2024060101  ; Serial
        7200        ; Refresh
        3600        ; Retry
        1209600     ; Expire
        86400       ; Minimum TTL
)

; NS 记录
@   IN  NS   ns1.example.com.
@   IN  NS   ns2.example.com.

; A 记录
@   IN  A    93.184.216.34
www IN  A    93.184.216.34

; AAAA 记录
@   IN  AAAA 2606:2800:220:1:248:1893:25c8:1946

; MX 记录
@   IN  MX   10  mail.example.com.

; CNAME 记录
blog IN  CNAME  example.blogspot.com.

; TXT 记录（SPF）
@   IN  TXT  "v=spf1 mx -all"

; CAA 记录
@   IN  CAA  0 issue "letsencrypt.org"
```

---

## 附录 A：常见顶级域与用途

| TLD | 用途 |
|-----|------|
| `.com` | 商业组织（最常用） |
| `.org` | 非营利组织 |
| `.net` | 网络服务提供商 |
| `.edu` | 教育机构 |
| `.gov` | 美国政府机构 |
| `.mil` | 美国军事机构 |
| `.int` | 国际组织 |
| `.cn` | 中国 |
| `.io` | 英属印度洋领地（常用于科技公司） |
| `.dev` | 开发者（强制 HTTPS） |
| `.app` | 应用（强制 HTTPS） |

## 附录 B：DNS 相关 RFC 速查

| RFC | 标题 | 年份 |
|-----|------|------|
| RFC 1034 | Domain Names — Concepts and Facilities | 1987 |
| RFC 1035 | Domain Names — Implementation and Specification | 1987 |
| RFC 2181 | Clarifications to the DNS Specification | 1997 |
| RFC 6891 | Extension Mechanisms for DNS (EDNS0) | 2013 |
| RFC 4033-4035 | DNSSEC | 2005 |
| RFC 7858 | DNS over TLS | 2016 |
| RFC 8484 | DNS over HTTPS | 2018 |
| RFC 7871 | EDNS Client Subnet | 2016 |

## 附录 C：推荐阅读与资源

- **IANA 根服务器**: https://www.iana.org/domains/root/servers
- **DNS 根服务器运营**: https://root-servers.org
- **Cloudflare DNS 学习中心**: https://www.cloudflare.com/learning/dns/
- **DNS 可视化工具**: https://dnsviz.net
- **DNSPod 帮助文档**: https://docs.dnspod.cn

---

> **结语**：DNS 是互联网最基础也最重要的基础设施之一。理解 DNS 的工作原理、记录类型、安全机制和故障排查方法，是每一位网络工程师和后端开发者的必备技能。本文档涵盖了 DNS 的核心知识点，建议结合实际操作（`dig`、`nslookup`）反复练习以加深理解。
