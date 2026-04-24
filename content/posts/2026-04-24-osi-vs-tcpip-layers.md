---
title: "OSI 七层 vs TCP/IP 四层：一次 HTTPS 请求的逐层解剖"
date: 2026-04-24T10:00:00+08:00
draft: false
tags: ["网络", "OSI", "TCP/IP", "HTTP", "TLS", "Wireshark", "Linux"]
categories: ["网络"]
author: "Kaka"
description: "别再背 OSI 七层模型了——用一次真实的 curl https://example.com 请求，亲手在 Linux 上把每一层抓出来看清楚，从此对网络分层有肌肉记忆。"
---

> **一句话总结**：OSI 是教科书里的"理想模型"，TCP/IP 是互联网上真正在跑的"工程实现"。理解两者差异的最好方式，不是背表格，而是抓一次真实的 HTTPS 请求，看每一层到底长什么样。

---

## 为什么要分层？

想象一下，你要从北京给东京的朋友寄一个包裹。你不需要亲自开车把它送到日本——你只要：

1. 把礼物装进盒子（应用层：你和朋友关心的内容）
2. 贴上地址标签（网络层：谁送给谁）
3. 交给快递员（传输层：保证送达）
4. 快递员装上飞机（链路层/物理层：实际的运输工具）

每一层只管自己的事，不操心上下层怎么实现。这就是**网络分层**的本质——**关注点分离（Separation of Concerns）**。

网络历史上出现过两套分层模型：

- **OSI 七层模型**：1984 年 ISO 提出，理论完备，是学术界和教材的标准。
- **TCP/IP 四层模型**：1970 年代随 ARPANET 实际演进出来的，互联网就是跑在它上面的。

一个是蓝图，一个是成品。它们的关系就像 UML 图和实际跑起来的代码。

---

## OSI 七层模型：理想主义者的蓝图

从上到下，记住口诀："**应、表、会、传、网、数、物**"。

| 层级 | 名称 | 核心职责 | 代表协议/技术 |
|------|------|----------|---------------|
| **L7** | 应用层 (Application) | 面向用户的协议 | HTTP、FTP、SMTP、DNS |
| **L6** | 表示层 (Presentation) | 加解密、编码、压缩 | TLS/SSL、JPEG、ASCII |
| **L5** | 会话层 (Session) | 建立/维护/终止会话 | NetBIOS、RPC |
| **L4** | 传输层 (Transport) | 端到端可靠传输 | TCP、UDP |
| **L3** | 网络层 (Network) | 路由与逻辑寻址 | IP、ICMP、路由协议 |
| **L2** | 数据链路层 (Data Link) | 相邻节点帧传输 | Ethernet、Wi-Fi、MAC |
| **L1** | 物理层 (Physical) | 比特流传输 | 网线、光纤、电信号 |

### OSI 的几个"理想化"设计

- **表示层和会话层独立存在**：这是 OSI 最理论化的部分，实际中几乎没有独立协议真正只做"会话管理"。
- **严格分层**：每层只能和相邻层交互，不能跨层。
- **完美但复杂**：正因为太完备，OSI 的网络栈实现起来非常重——这也是它输给 TCP/IP 的历史原因之一。

---

## TCP/IP 四层模型：实用主义者的胜利

TCP/IP 出生在真实的网络对抗中（冷战时期的 ARPANET），它只关心"能跑、能扩展、不出事"。

| 层级 | 名称 | 对应 OSI | 代表协议 |
|------|------|----------|----------|
| **L4** | 应用层 (Application) | OSI L5+L6+L7 | HTTP、HTTPS、DNS、SSH、SMTP |
| **L3** | 传输层 (Transport) | OSI L4 | TCP、UDP、QUIC |
| **L2** | 网络层 (Internet) | OSI L3 | IP、ICMP、ARP |
| **L1** | 网络接口层 (Link) | OSI L1+L2 | Ethernet、Wi-Fi、PPP |

注意几个关键点：

1. **应用层"吞并"了 OSI 的表示层和会话层**。HTTPS 里的 TLS 加密（本应是表示层）、HTTP/2 的连接复用（本应是会话层）——这些都打包进应用层解决了。
2. **网络接口层合并了物理层和数据链路层**。TCP/IP 不关心你用的是光纤还是 Wi-Fi，只要能把帧传出去就行。
3. **没有"严格分层"的洁癖**。TLS 同时用到了传输层的 TCP 和应用层的握手，跨层调用很常见。

---

## 核心对比：一张图说清楚

```
┌─────────────────────┐       ┌─────────────────────┐
│   OSI 七层模型      │       │   TCP/IP 四层模型   │
├─────────────────────┤       ├─────────────────────┤
│ L7 应用层           │       │                     │
├─────────────────────┤       │                     │
│ L6 表示层           │──────▶│   应用层            │
├─────────────────────┤       │   (HTTP, TLS, DNS)  │
│ L5 会话层           │       │                     │
├─────────────────────┤       ├─────────────────────┤
│ L4 传输层           │──────▶│ 传输层 (TCP/UDP)    │
├─────────────────────┤       ├─────────────────────┤
│ L3 网络层           │──────▶│ 网络层 (IP)         │
├─────────────────────┤       ├─────────────────────┤
│ L2 数据链路层       │       │                     │
├─────────────────────┤──────▶│ 网络接口层          │
│ L1 物理层           │       │ (Ethernet, Wi-Fi)   │
└─────────────────────┘       └─────────────────────┘
```

### 关键差异总结

| 维度 | OSI | TCP/IP |
|------|-----|--------|
| **层数** | 7 层 | 4 层 |
| **诞生背景** | 学术/标准化组织 | 工程/实战 |
| **分层严格度** | 严格，每层定义清晰 | 宽松，允许跨层 |
| **设计顺序** | 先有模型，再找协议 | 先有协议，再总结模型 |
| **当前角色** | 教学、分析问题用 | 互联网实际运行用 |
| **表示层/会话层** | 独立存在 | 并入应用层 |

---

## 实战：抓一次 `curl https://example.com`，看清每一层

光讲理论没意思，我们来做一次"网络解剖"。目标：从一次 HTTPS 请求中，把每一层的真实数据都抓出来。

### 准备工作

```bash
# Ubuntu / Debian
sudo apt install -y tcpdump curl dnsutils iproute2

# macOS
brew install wireshark  # 或直接用系统自带的 tcpdump
```

### Step 1：物理层 / 链路层（L1 + L2）

#### 查看你用的是什么物理介质

```bash
# 网卡状态（速率、双工、介质类型）
ethtool eth0
# 或在 macOS 上
networksetup -listallhardwareports
```

输出示例：

```
Settings for eth0:
    Speed: 1000Mb/s         # ← 1Gbps 千兆以太网
    Duplex: Full            # ← 全双工
    Port: Twisted Pair      # ← 双绞线
    Link detected: yes      # ← 物理层连通
```

这就是 **L1 物理层**——电信号、光信号、射频信号的世界。

#### 查看 MAC 地址（链路层身份证）

```bash
ip link show eth0
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
#     link/ether a0:36:9f:12:34:56 brd ff:ff:ff:ff:ff:ff
```

那个 `a0:36:9f:12:34:56` 就是 **L2 数据链路层**的 MAC 地址——在本地局域网内唯一标识这块网卡。

#### 看 ARP 表（L2 如何找到下一跳）

```bash
ip neigh show
# 192.168.1.1 dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
```

这告诉你：要把数据包发给网关 `192.168.1.1`，必须把以太网帧的目标 MAC 写成 `aa:bb:cc:dd:ee:ff`。**L3 的 IP 地址需要通过 ARP 协议翻译成 L2 的 MAC 地址**——这是典型的跨层协作。

### Step 2：网络层（L3）

#### DNS 解析：应用层触发，但走的是整个栈

```bash
dig +short example.com
# 93.184.216.34
```

拿到 IP 后，内核决定走哪条路由：

```bash
ip route get 93.184.216.34
# 93.184.216.34 via 192.168.1.1 dev eth0 src 192.168.1.100 uid 1000
```

这就是 **L3 网络层**的工作：**根据目标 IP 查路由表，决定下一跳**。

#### 用 traceroute 看 L3 的全程路径

```bash
traceroute -n example.com
#  1  192.168.1.1      0.5 ms
#  2  10.0.0.1         3.2 ms
#  3  203.0.113.1     12.8 ms
#  ...
# 15  93.184.216.34  128.4 ms
```

每一跳都是一个 L3 路由器，它们只关心 IP 头部，从不拆开看里面的 TCP 或 HTTP。

### Step 3：传输层（L4）

开一个终端抓包：

```bash
sudo tcpdump -i eth0 -nn -vv 'host 93.184.216.34 and port 443' -w /tmp/https.pcap
```

另开一个终端发请求：

```bash
curl -v https://example.com > /dev/null
```

回来看抓包结果：

```bash
sudo tcpdump -r /tmp/https.pcap -nn | head -10
```

你会看到经典的 **TCP 三次握手**：

```
14:23:45.123 IP 192.168.1.100.54321 > 93.184.216.34.443: Flags [S], seq 1000
14:23:45.173 IP 93.184.216.34.443 > 192.168.1.100.54321: Flags [S.], seq 2000, ack 1001
14:23:45.174 IP 192.168.1.100.54321 > 93.184.216.34.443: Flags [.], ack 2001
```

这就是 **L4 传输层**：端口号（54321 → 443）、序列号、ACK、握手——这些都是 TCP 提供的可靠性保证，IP 层（L3）完全不知道。

### Step 4：应用层（L5 + L6 + L7，在 TCP/IP 里合并成一层）

TCP 握手完成后，立刻是 **TLS 握手**（OSI 里的 L6 表示层，TCP/IP 里是应用层的一部分）：

```
Client Hello  →  支持的加密套件、SNI (example.com)
Server Hello  →  选定的加密套件、证书
Client Key Exchange + Finished
Server Finished
```

用 `openssl` 可以亲眼看到：

```bash
openssl s_client -connect example.com:443 -servername example.com -showcerts </dev/null 2>&1 | head -30
```

输出里会看到：

```
CONNECTED(00000003)
Protocol: TLSv1.3
Cipher: TLS_AES_256_GCM_SHA384
Server certificate
subject=CN=www.example.org
issuer=C=US, O=DigiCert Inc, CN=DigiCert TLS RSA SHA256 2020 CA1
```

这就是 **L6 表示层**干的活——**加密**。在 TCP/IP 的世界里，它被归入"应用层"，但本质上它做的事情（加解密、格式转换）是纯粹的表示层职责。

TLS 完成后，才是真正的 **HTTP 报文（L7 应用层）**：

```
GET / HTTP/1.1
Host: example.com
User-Agent: curl/8.0.1
Accept: */*
```

但注意：在网络上传输时，这段明文 HTTP **已经被 TLS 加密过了**，抓包也只能看到一堆乱码。这就是为什么抓包工具需要配合 SSL key log 才能看到明文 HTTP。

```bash
# 让 curl 导出 TLS 会话密钥
SSLKEYLOGFILE=/tmp/sslkeys.log curl -v https://example.com > /dev/null

# Wireshark 可以用这个 key 文件解密抓包
```

### Step 5：把整个栈画在一张图上

当你执行 `curl https://example.com`，数据在本机是这样往下走的：

```
┌──────────────────────────────────────────────────────────┐
│ 应用层：HTTP 报文                                         │
│  GET / HTTP/1.1\r\nHost: example.com\r\n...              │
├──────────────────────────────────────────────────────────┤
│ 表示层（TLS）：加密成 TLS Record                          │
│  17 03 03 00 20 [encrypted payload...]                   │
├──────────────────────────────────────────────────────────┤
│ 传输层：加上 TCP 头部                                     │
│  src_port=54321 dst_port=443 seq=... ack=...             │
├──────────────────────────────────────────────────────────┤
│ 网络层：加上 IP 头部                                      │
│  src=192.168.1.100 dst=93.184.216.34 ttl=64              │
├──────────────────────────────────────────────────────────┤
│ 链路层：加上以太网帧头 + 尾                               │
│  dst_mac=aa:bb:cc:dd:ee:ff src_mac=a0:36:9f:12:34:56 CRC │
├──────────────────────────────────────────────────────────┤
│ 物理层：变成电信号从网线发出去                            │
│  010110100111010...                                      │
└──────────────────────────────────────────────────────────┘
```

对端收到后，从下往上**逐层剥皮**，最后 Nginx 看到的就是那段明文的 HTTP GET。

---

## 实战案例：当某一层出问题了怎么排查？

分层模型最大的工程价值，是让你**按层定位故障**。记住这个排查口诀：**从下往上，逐层验证**。

### 场景：网页打不开

```bash
# L1 物理层：网线插了吗？
cat /sys/class/net/eth0/carrier
# 1 = 有物理连接，0 = 断了

# L2 链路层：能发出以太网帧吗？
ping 192.168.1.1  # 网关
arp -n            # ARP 表有没有填好

# L3 网络层：能路由到外网吗？
ping 8.8.8.8            # 公网 IP 能通 → 路由 OK
traceroute 8.8.8.8      # 看卡在哪一跳

# L4 传输层：TCP 能握手吗？
nc -zv example.com 443
# Connection to example.com 443 port [tcp/https] succeeded!

# L6 表示层：TLS 握手正常吗？
openssl s_client -connect example.com:443 -servername example.com </dev/null
# 看证书是否有效、是否过期

# L7 应用层：HTTP 本身正常吗？
curl -v https://example.com
# 看状态码、响应头
```

### 真实案例对照表

| 症状 | 最可能出问题的层 | 排查命令 |
|------|------------------|----------|
| 网线灯不亮 | L1 | `ethtool eth0` |
| 本机能 ping 网关但 ping 不通其他主机 | L2（ARP/交换机） | `ip neigh`、`arping` |
| ping 不通任何外网 IP | L3（路由/网关） | `ip route`、`traceroute` |
| ping 通但浏览器打不开 | L4/L7（端口/DNS） | `dig`、`nc -zv` |
| `curl: (60) SSL certificate problem` | L6（TLS） | `openssl s_client` |
| 返回 502/503 | L7（应用） | `curl -v`、看应用日志 |
| HTTP 响应很慢但 ping 很快 | L4 或 L7（不是 L3） | 抓包看 TCP 窗口、RTT |

**一个典型陷阱**：很多人一看到"网页打不开"就去重启路由器——但如果是 L6 证书过期问题，重启路由器根本没用。分层思维能让你少走一半弯路。

---

## 常见疑问

### Q1：为什么教材都讲 OSI，但实际用的是 TCP/IP？

OSI 的**词汇**（七层、表示层、会话层）已经成为行业通用语言——即使我们实际在用 TCP/IP，讨论问题时还是会说"这是 L4 的事"、"TLS 是 L6"。

就像我们聊计算机体系结构时会说"寄存器"、"缓存"，即使你写的是 Python，这些词依然适用。OSI 提供了**语言**，TCP/IP 提供了**实现**。

### Q2：QUIC 到底是哪一层？

QUIC 是个有趣的例子：

- 它跑在 **UDP（L4）** 之上；
- 但它又自己实现了**可靠传输、流控、拥塞控制**（L4 的职责）；
- 它还把 **TLS 加密（L6）** 直接融合进来；
- 最终承载 **HTTP/3（L7）**。

从 OSI 严格分层的角度看，QUIC 是一个"违反分层"的怪物。但从 TCP/IP 务实的角度看，它完美诠释了"只要跑得好，管它什么层"。

### Q3：为什么说"OSI 是理想，TCP/IP 是现实"？

因为 TCP/IP 在 1970 年代就已经在真实网络上跑起来了，而 OSI 直到 1980 年代中期才有完整标准。等 OSI 标准出炉时，互联网已经被 TCP/IP 占领了。**先跑起来的实现，会自然成为事实标准**——这在软件工程里是一条铁律。

---

## 总结：记住三件事就够了

1. **OSI 是思维框架，TCP/IP 是工程实现**。面试、排查、学习时用 OSI 的词汇；写代码、配网络、抓包时用 TCP/IP 的思路。
2. **分层的价值不在于"每层严格隔离"，而在于"出问题时能按层定位"**。记住那张"症状 → 层级 → 排查命令"的表。
3. **抓一次包胜过读十本书**。下次觉得某个概念模糊时，打开 `tcpdump`，让数据自己说话。

---

## 参考资料

- [RFC 1122 - Requirements for Internet Hosts](https://datatracker.ietf.org/doc/html/rfc1122) — TCP/IP 四层模型的正式定义
- [ISO/IEC 7498-1 - OSI Reference Model](https://www.iso.org/standard/20269.html) — OSI 七层模型原始标准
- [Wireshark 官方教程](https://www.wireshark.org/docs/) — 最好的分层学习工具
- 本博客相关文章：[TCP 拥塞控制深度解析](/posts/2026-02-18-tcp-congestion-control-deep-dive/)、[IPv4 分片与 MTU/MSS](/posts/2026-03-14-ipv4-fragmentation-mtu-mss/)
