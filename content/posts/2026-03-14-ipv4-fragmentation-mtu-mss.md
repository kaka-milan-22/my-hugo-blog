---
title: "IPv4 分片问题 | MTU 和 MSS 的核心区别"
date: 2026-03-14T00:00:00+08:00
draft: false
tags: ["网络", "Linux", "IPv4", "MTU", "MSS", "TCP"]
categories: ["网络"]
author: "Kaka"
description: "从零理解 IPv4 分片的来龙去脉，深入剖析 MTU 与 MSS 的本质区别，并用 Linux 命令亲手验证每一个概念。"
---

> **一句话总结**：MTU 是网络层的物理限制，MSS 是传输层的主动协商——理解这两者的区别，是搞懂网络性能问题的关键第一步。

---

## 问题的起源：数据包为什么需要分片？

想象你要把一张超大的海报邮寄出去，但邮局规定每个包裹最长不能超过 1 米。你只有两个选择：

1. 把海报裁成几段，分多个包裹寄出，收件人再拼起来。
2. 在寄出之前，就把海报折叠到合适的尺寸。

IPv4 的世界里，这两种方案分别对应 **IP 分片（Fragmentation）** 和 **MSS 协商**。前者是不得已而为之，后者是更聪明的提前规划。

---

## 核心概念一：MTU（Maximum Transmission Unit）

### 是什么？

MTU 是**链路层**允许传输的最大数据帧大小，单位是字节。它是一个**物理/链路层的硬性限制**，由网络介质决定，不是软件可以随意更改的。

| 网络类型 | 典型 MTU |
|---------|---------|
| 以太网（Ethernet） | **1500 字节**（最常见） |
| 本地回环（loopback） | 65535 字节 |
| PPPoE（ADSL 拨号） | 1492 字节 |
| Wi-Fi（802.11） | 2304 字节 |
| VPN（如 WireGuard） | 1420 字节左右 |

### 解决了什么问题？

不同物理网络的帧大小不同。MTU 定义了"这条路上的卡车最大能装多少货"，让网络层知道自己最多能发多大的包。

### Linux 命令验证

```bash
# 查看所有网卡的 MTU
ip link show

# 只看 eth0 的 MTU
ip link show eth0

# 临时修改 MTU（需要 root）
sudo ip link set eth0 mtu 1400

# 用 ping 测试 MTU 边界（-s 指定数据大小，-M do 禁止分片）
# ICMP header 8字节 + IP header 20字节 = 28字节
# 所以测试 1500 MTU 时，-s 应该是 1472
ping -s 1472 -M do 8.8.8.8

# 如果包太大且禁止分片，会看到：
# ping: local error: message too long, mtu=1500
ping -s 1473 -M do 8.8.8.8
```

**实验结果解读**：
- `-s 1472` 成功 → 说明链路 MTU 是 1500（1472 + 8 ICMP header + 20 IP header = 1500）
- `-s 1473` 失败 → 包超过 MTU，且我们禁止了分片，所以直接报错

---

## 核心概念二：IP 分片（IPv4 Fragmentation）

### 是什么？

当一个 IP 数据包大于路径上某个节点的 MTU 时，路由器（或发送方）会把这个包**切割成多个更小的分片**，每个分片独立传输，最终由**目标主机重组**。

### IPv4 分片头部字段

```
IP Header（20字节）包含三个关键字段：
┌─────────────────┬──────────────────────────────────────┐
│ Identification  │ 16位，同一个原始包的分片共享同一个 ID   │
│ Flags           │ 3位：DF（Don't Fragment）、MF（More Fragments）│
│ Fragment Offset │ 13位，当前分片在原始数据中的偏移量（单位8字节）│
└─────────────────┴──────────────────────────────────────┘
```

### 分片过程举例

假设要发送一个 3000 字节的 UDP 包，链路 MTU = 1500：

```
原始包: [IP Header 20B][UDP Header 8B][Data 2972B] = 3000B

分片1: [IP Header][Data 0~1479B]   offset=0,   MF=1
分片2: [IP Header][Data 1480~2959B] offset=185, MF=1
分片3: [IP Header][Data 2960~2971B] offset=370, MF=0
```

> offset 单位是 8 字节，所以 1480 / 8 = 185

### 为什么 IP 分片是个坏东西？

1. **性能差**：分片和重组消耗 CPU，增加延迟。
2. **丢一片全重传**：TCP 层不知道分片的存在，只要有一个分片丢失，整个 TCP 段都要重传。
3. **防火墙/NAT 问题**：很多防火墙只检查第一个分片（含端口号），后续分片可能被错误处理或丢弃。
4. **IPv6 已废弃路由器分片**：IPv6 中只有发送方才能分片，路由器不再做分片，遇到大包直接发 ICMPv6 "包太大" 报错。

### Linux 命令验证

```bash
# 用 tcpdump 抓包，观察 IP 分片
# 先在一个终端启动抓包
sudo tcpdump -i eth0 -n 'host 目标IP' -v

# 另一个终端发送大 UDP 包（故意触发分片）
# hping3 发送 3000 字节 UDP 包
sudo hping3 -2 -d 3000 目标IP

# 或者用 Python 发送大 UDP 包
python3 -c "
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.sendto(b'x' * 3000, ('目标IP', 9999))
"

# 在 tcpdump 输出中，你会看到：
# flags [+]  表示 MF=1（还有更多分片）
# flags [none] 表示最后一个分片
# frag XXXX:0   第一个分片，offset=0
# frag XXXX:185 第二个分片，offset=185（×8字节=1480）

# 查看系统 IP 分片统计
cat /proc/net/snmp | grep -i frag
# 或者
nstat | grep -i frag
```

---

## 核心概念三：MSS（Maximum Segment Size）

### 是什么？

MSS 是 **TCP 层**的概念，表示 TCP 单次可以发送的**最大数据载荷**大小（不含 TCP/IP 头部）。它在 **TCP 三次握手时协商**，写在 SYN 包的 Options 字段中。

```
MSS = MTU - IP Header(20B) - TCP Header(20B)
MSS = 1500 - 20 - 20 = 1460 字节（以太网标准情况）
```

### MSS 解决了什么问题？

MSS 是"聪明的预防"——**在发送之前就把数据切到合适的大小**，让 TCP 段永远不超过路径 MTU，从根本上避免了 IP 层的分片。

> 对比一下：
> - IP 分片 = 货物已经装车了，过不了收费站，被迫在收费站拆包
> - MSS = 发货前就按收费站限制打包好，一路顺畅

### TCP 握手中的 MSS 协商

```
Client ──SYN (MSS=1460)──────────→ Server
Client ←──SYN-ACK (MSS=1460)───── Server
Client ──ACK──────────────────────→ Server

双方取较小的 MSS 作为实际使用值。
```

### Linux 命令验证

```bash
# 用 ss 命令查看 TCP 连接的 MSS
ss -tin dst 8.8.8.8

# 输出示例：
# skmem:(r0,rb212992,t0,tb212992,f0,w0,o0,bl0,d0)
# ts sack cubic wscale:7,7 rto:204 rtt:4.123/1.234 mss:1460 ...

# 用 tcpdump 抓 TCP 握手，查看 MSS 协商
sudo tcpdump -i eth0 -n 'tcp[tcpflags] & (tcp-syn) != 0' -v

# 在另一个终端发起连接
curl -s https://example.com > /dev/null

# tcpdump 输出中可以看到：
# Options [mss 1460, ...]

# 手动设置连接的 MSS（用于测试）
sudo ip route add 目标IP/32 via 网关IP mtu 1400
# 这会让内核对该路由使用更小的 MTU，从而协商更小的 MSS

# 查看路由的 MTU 设置
ip route show cache
```

---

## MTU vs MSS：一张图说清楚

```
以太网帧（最大 1518 字节）
┌──────────────────────────────────────────────────────┐
│ Ethernet Header │        IP Packet（MTU=1500）        │ FCS │
│     14B         │                                    │  4B │
│                 │ IP Header │    TCP Segment          │     │
│                 │   20B     │                        │     │
│                 │           │ TCP Header │  Data     │     │
│                 │           │    20B     │ (MSS=1460)│     │
└──────────────────────────────────────────────────────┘

MTU = 1500（IP 层看到的最大包大小）
MSS = 1460（TCP 层实际能装的数据大小）
```

| 对比项 | MTU | MSS |
|-------|-----|-----|
| 所在层 | 网络层 / 链路层边界 | 传输层（TCP） |
| 单位含义 | 整个 IP 包最大大小 | TCP 数据载荷最大大小 |
| 设置方式 | 由网络接口决定 | TCP 握手时协商 |
| 超出后果 | 触发 IP 分片或丢包 | 不会超出（提前协商好了） |
| 适用协议 | IP（含 UDP、TCP） | 仅 TCP |

---

## 进阶：PMTUD（路径 MTU 发现）

如果路径上有中间节点 MTU 更小（比如过了一段 VPN），怎么办？

**Path MTU Discovery (PMTUD)** 的原理：
1. 发送方发出 DF=1（禁止分片）的包。
2. 如果某个路由器的 MTU 不够，它会丢掉这个包，并回复 **ICMP Type 3 Code 4（需要分片但 DF 位已置位）**，同时告知自己的 MTU。
3. 发送方收到 ICMP 后，缩小包大小重试。

```bash
# 查看本机路由缓存中的 PMTU 信息
ip route show cache

# 手动触发 PMTUD 测试（路径上逐步减小包大小）
for size in 1472 1400 1300 1200; do
  result=$(ping -s $size -M do -c 1 8.8.8.8 2>&1)
  echo "size=$size: $(echo $result | grep -oE 'mtu=[0-9]+|1 received|100% packet loss')"
done

# 如果 ICMP 被防火墙屏蔽（常见的 VPN/云环境问题），PMTUD 会失效
# 这时候可以用 TCP MSS Clamping 作为补救
# 在 iptables 中限制 MSS
sudo iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --clamp-mss-to-pmtu
```

---

## 实战：排查 MTU/MSS 导致的网络问题

### 症状

- 小数据包（ping、DNS）正常，大文件传输卡死或超慢
- SSH 连接可以建立，但执行命令时没有输出
- HTTPS 页面加载一半就卡住

**这是经典的 "MTU Black Hole" 问题**，通常发生在 VPN、PPPoE、或 GRE 隧道环境中。

### 排查步骤

```bash
# 步骤1：确认正常路径 MTU
ping -s 1472 -M do -c 3 目标IP   # 如果成功，MTU 是 1500
ping -s 1452 -M do -c 3 目标IP   # 如果成功，MTU 是 1480（PPPoE 典型）

# 步骤2：检查 ICMP 是否被防火墙屏蔽
sudo tcpdump -i eth0 icmp -n

# 步骤3：检查当前 TCP 连接的实际 MSS
ss -tin | grep mss

# 步骤4：用 tracepath 探测路径上每跳的 MTU
tracepath 目标IP
# 输出示例：
# 1: 192.168.1.1  (pmtu 1500)
# 2: 10.0.0.1    (pmtu 1492)  ← 这里 MTU 变小了

# 步骤5：如果确认 MTU 问题，临时修复
sudo ip link set eth0 mtu 1400
# 或者在内核强制 MSS Clamping（见上方 iptables 命令）
```

---

## 总结

| 场景 | 发生了什么 | 应该怎么做 |
|-----|-----------|-----------|
| 以太网直连 | MTU=1500，MSS=1460，无分片 | 正常，无需操作 |
| PPPoE 宽带 | MTU=1492，MSS=1452 | 路由器自动处理，或手动设置 MSS |
| VPN 隧道 | 隧道头占用额外字节，有效 MTU 下降 | 设置 VPN 接口 MTU，或启用 MSS Clamping |
| UDP 大包 | 可能触发 IP 分片 | 应用层自己控制数据大小，或改用 TCP |
| MTU Black Hole | ICMP 被屏蔽，PMTUD 失效 | 降低 MTU / 启用 iptables MSS Clamping |

**核心记忆点**：
- **MTU** = 链路的物理上限，网络层被动遵守
- **MSS** = TCP 主动协商，目的是避免 IP 分片
- **IP 分片** = 性能杀手，现代网络应尽量避免
- **PMTUD** = 自动发现路径 MTU 的机制，依赖 ICMP 畅通

---

*参考资料：RFC 791（IPv4）、RFC 879（MSS）、RFC 1191（PMTUD）*
