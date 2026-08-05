---
title: "dig 实战指南：从 DNS 查询到故障定位"
date: 2026-08-05T22:00:00+08:00
draft: false
tags: ["DNS", "dig", "Networking", "DNSSEC", "Troubleshooting"]
categories: ["Networking", "Tools"]
author: "Kaka"
description: "系统掌握 dig 的查询模型、输出字段与 DNS 排障方法，覆盖 authoritative server、cache、negative response、DNSSEC、EDNS 和 TCP。"
---

## 引言

`dig` 的价值不只是“查一个域名对应哪个 IP”。它会把 DNS response 的 `RCODE`、flags、TTL、authoritative 信息和查询路径暴露出来，因此更像一个 DNS protocol debugger。

本文参考 Julia Evans 的 [How to use dig](https://jvns.ca/blog/2021/12/04/how-to-use-dig/) 进行中文重写，并结合 [BIND 9 dig manual](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility)、[RFC 1035](https://www.rfc-editor.org/rfc/rfc1035) 与 [RFC 2308](https://www.rfc-editor.org/rfc/rfc2308) 补充生产环境排障细节。核心原则只有一句：快速查看结果可以用 `+short`，定位故障必须阅读完整 response。

## 先建立正确的查询模型

最常用的语法可以简化成：

```bash
dig [@server] name [type] [query-options]
```

其中：

- `@server` 指定向哪台 DNS server 发起查询；省略时，`dig` 使用 `/etc/resolv.conf` 中配置的 resolver。
- `name` 是待查询的 domain name，例如 `example.com`。
- `type` 是 record type，省略时默认为 `A`。
- `query-options` 控制 recursion、transport、DNSSEC 和 output format，通常以 `+` 开头。

```bash
# 使用系统 resolver 查询 IPv4 address
dig example.com A

# 明确向 1.1.1.1 查询 IPv6 address
dig @1.1.1.1 example.com AAAA

# 查询 mail exchanger
dig example.com MX
```

常见 record type 如下：

| Type | 主要用途 |
| --- | --- |
| `A` / `AAAA` | IPv4 / IPv6 address |
| `CNAME` | alias 指向的 canonical name |
| `NS` | zone 的 authoritative name server |
| `MX` | mail exchanger 与 priority |
| `TXT` | SPF、domain verification 等 text data |
| `SOA` | zone metadata、serial 与 negative caching 参数 |
| `SRV` | service endpoint、port、priority 与 weight |
| `CAA` | 允许签发 certificate 的 CA |
| `PTR` | reverse DNS mapping |

## 完整输出应该怎么看

一次典型查询包含 `HEADER`、`QUESTION`、`ANSWER`、`AUTHORITY` 和 `ADDITIONAL`。并非每次 response 都有所有 section；现代查询通常还会看到代表 EDNS 的 `OPT PSEUDOSECTION`。

先看 `HEADER`，不要一上来只盯着 IP：

```text
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12345
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
```

`status` 是 DNS response code，也就是 `RCODE`：

| Status | 含义 | 排障方向 |
| --- | --- | --- |
| `NOERROR` | 查询被正常处理 | `ANSWER: 0` 时可能是 NODATA，不代表存在目标 type |
| `NXDOMAIN` | 该 domain name 不存在 | 检查拼写、zone delegation 和 negative cache |
| `SERVFAIL` | server 无法完成查询 | 常见于 DNSSEC validation、上游超时或 authoritative server 异常 |
| `REFUSED` | server 按 policy 拒绝查询 | 检查 ACL、recursion policy 或 source network |
| timeout | 没收到 response | 检查 network、firewall、UDP/TCP 53 与 server availability |

常见 flags 则说明 response 具有哪些属性：

- `qr`：这是 response，而不是 query。
- `aa`：回答来自该 zone 的 authoritative server。
- `rd`：client 请求 recursion。
- `ra`：server 声明支持 recursion。
- `ad`：recursive resolver 已将相关数据验证为 DNSSEC authentic data；这不代表本地 `dig` 自己完成了 validation。
- `tc`：UDP response 被截断，应改用 TCP 重新查询。

`ANSWER` 中一条 resource record 通常遵循：

```text
NAME    TTL    CLASS    TYPE    RDATA
```

例如 `example.com. 300 IN A 192.0.2.10` 中，`300` 是该 response 给出的 remaining TTL，`IN` 是 Internet class。排查 cache 与 propagation 时，TTL 往往比 IP 本身更重要。

## 控制输出：简洁不等于可靠

只看 answer section，可以使用：

```bash
dig example.com A +noall +answer
```

只输出 record data，可以使用：

```bash
dig example.com A +short
```

`+short` 很适合交互式查询，但不适合作为故障判断的唯一依据。空输出可能表示 `NXDOMAIN`、`NOERROR/NODATA`、`SERVFAIL` 或 timeout；它隐藏的恰好是 incident 中最关键的状态。

如果希望默认简化输出，可以在 `~/.digrc` 写入：

```text
+noall +answer
```

但脚本不应默默继承个人配置。较新的 BIND 版本提供 `-r` 禁用 `~/.digrc`，让 automation 的行为可复现；macOS 自带的旧版 `dig`（例如 9.10.x）并不支持该选项。因此在跨平台脚本中，应先检测 `dig -h` 是否包含 `-r`，并显式写出所需的 output 与 query option，而不是假定开发者没有配置 `~/.digrc`。

```bash
dig @1.1.1.1 example.com A +time=2 +tries=1 +noall +answer
```

生产脚本还应保留并检查 `RCODE`，不要把“命令有输出”直接等价为 DNS 正常。

## Reverse DNS：不要手写反向 zone

`PTR` 查询使用的 name 不是原始 IP，而是 `in-addr.arpa` 或 `ip6.arpa` 下的 reverse mapping。`-x` 会替你完成转换：

```bash
dig -x 203.0.113.10
dig @1.1.1.1 -x 2001:db8::10
```

需要注意，forward record 与 reverse record 是两个独立的数据集。一个 hostname 能解析到某个 IP，不意味着这个 IP 一定存在匹配的 `PTR`，更不保证两者严格互相指向。

## Recursive resolver 与 Authoritative server

日常 `dig example.com` 通常问的是 recursive resolver。它可能直接返回 cache，也可能代替 client 从 root、TLD 一路查询到 authoritative server。

排障时要把两层拆开：

```bash
# 找出 zone 的 authoritative server
dig example.com NS +short

# 绕过 recursive cache，直接询问其中一台 authoritative server
dig @hera.ns.cloudflare.com example.com A +norecurse

# 检查 zone serial
dig @hera.ns.cloudflare.com example.com SOA +norecurse
```

直接查询 authoritative server 时，正常的 authoritative answer 通常带 `aa` flag。若不同 recursive resolver 返回旧值，而各 authoritative server 已返回相同新值，问题通常在 cache；若 authoritative server 之间的 `SOA serial` 或 answer 不一致，更可能是 zone replication 或 deployment 问题。

`+norecurse` 很重要：它清除 query 中的 `RD` bit，避免误把一台同时提供 recursion 的 server 当成 authoritative source。

## 用 +trace 查看 Delegation Path

```bash
dig example.com A +trace
```

`+trace` 从 root delegation 开始，依次查询 TLD 和 authoritative server。它适合判断 delegation、glue record 或 authoritative reachability 在哪一层断裂。

但 `+trace` 不是“复刻本地 resolver 的全过程”：它绕过本地 recursive cache，自行沿 delegation path 查询；同时也可能受到本机 network policy 对外部 UDP/TCP 53 的限制。BIND 对 `+trace` 的 `@server` 处理也有特殊语义——指定的 server 主要用于最初的 root name-server 查询，而不是每一步都固定询问它。因此，`+trace` 应与普通 recursive query 对照使用，而不是单独下结论。

## Transport 与 EDNS 排障

DNS 通常先走 UDP；较大的 response、DNSSEC data 或 network middlebox 可能让问题只在特定 transport 下出现。

```bash
# 强制使用 TCP
dig @1.1.1.1 example.com A +tcp

# 禁用 EDNS，观察 legacy query 是否正常
dig @1.1.1.1 example.com A +noedns
```

如果 UDP timeout 而 TCP 正常，应检查 UDP/53、fragmentation、MTU 与 stateful firewall。如果 response 带 `tc`，client 应通过 TCP 重试。如果禁用 EDNS 后反而恢复，重点检查 outdated DNS proxy、firewall 或 load balancer 对 EDNS packet 的处理；`+noedns` 是 diagnosis 手段，不是长期修复。

## DNSSEC：AD 与 CD 到底表示什么

向支持 validation 的 recursive resolver 请求 DNSSEC record：

```bash
dig @1.1.1.1 example.com A +dnssec
```

`+dnssec` 在 EDNS 中设置 `DO` bit，表示 client 希望取得 DNSSEC-related record。返回的 `ad` flag 表示 resolver 认为数据已经通过 validation；它不是 `dig` 对 chain of trust 的本地证明。

当正常查询返回 `SERVFAIL`，而怀疑是 DNSSEC 配置错误时，可以做对照：

```bash
dig @1.1.1.1 example.com A +cd
```

`+cd` 设置 Checking Disabled bit，请 resolver 跳过 DNSSEC validation。如果 `+cd` 能返回数据、普通查询却 `SERVFAIL`，DNSSEC failure 的可能性很高，下一步应检查 `DS`、`DNSKEY`、signature validity 和 clock，而不是把 `+cd` 当成修复方案。需要在本机验证 chain of trust 时，应使用 `delv` 等 validator-aware 工具。

## 一套可执行的 DNS 排障流程

### 1. 先保留完整现场

```bash
dig example.com A
```

记录 queried server、`RCODE`、flags、TTL、query time 与 answer。不要从 `+short` 开始，否则很容易丢掉故障分类所需的信息。

### 2. 对比另一台 Recursive Resolver

```bash
dig @1.1.1.1 example.com A
```

系统 resolver 失败而公共 resolver 正常，优先检查本地 DNS path、enterprise resolver、split-horizon policy 或 cache；两者都失败，再继续检查 authoritative side。

### 3. 直接比较所有 Authoritative Server

```bash
dig example.com NS +short
dig @hera.ns.cloudflare.com example.com A +norecurse
dig @elliott.ns.cloudflare.com example.com A +norecurse
dig @hera.ns.cloudflare.com example.com SOA +norecurse
```

真实业务域名应把示例 server 替换为其 `NS` 结果，并逐台比较 answer、TTL 与 `SOA serial`。

### 4. 再检查 Delegation

```bash
dig example.com A +trace
```

重点观察 parent zone 给出的 `NS`、glue address 和下一跳是否一致，而不是只看最后一行。

### 5. 根据症状做 Controlled Comparison

```bash
# 怀疑 DNSSEC validation
dig @1.1.1.1 example.com A +dnssec
dig @1.1.1.1 example.com A +cd

# 怀疑 UDP、EDNS 或 middlebox
dig @1.1.1.1 example.com A +tcp
dig @1.1.1.1 example.com A +noedns
```

每次只改变一个变量，才能判断 failure domain。

## TTL、Propagation 与 Negative Cache

所谓 DNS propagation，主要是各层 cache 在 TTL 到期后陆续重新查询。authoritative server 通常返回 zone 中配置的 TTL；recursive resolver 返回的是 cache 中逐渐递减的 remaining TTL。因此，同一时刻从不同 resolver 看到不同值，并不自动意味着 authoritative deployment 失败。

`NXDOMAIN` 和 `NOERROR/NODATA` 也可能被 negative cache。根据 RFC 2308，negative response 中的 `SOA` 信息参与决定 negative caching TTL。修复错误 record 后，旧的“不存在”结果仍可能继续一段时间；频繁反复修改 record 只会让观测更混乱。正确做法是提前降低 TTL、等待旧 cache 过期、一次变更后同时验证 authoritative 与 recursive response。

## macOS 上的一个常见错觉

macOS 支持 scoped resolver、VPN DNS 和 per-domain search policy。`dig @server ...` 会直接询问指定 server，不一定复现 application 通过系统 resolver framework 的解析路径。

如果 browser、`curl` 或 application 的结果与 `dig` 不一致，应同时检查：

```bash
scutil --dns
```

重点确认目标 domain 被哪个 scoped resolver 接管，以及 VPN、mDNS、`/etc/hosts` 或 application cache 是否参与。此时“`dig` 正常”只能证明那次显式 DNS query 正常，不能证明 application 的完整 name-resolution path 正常。

## 结语

`dig` 最实用的用法不是记住几十个 option，而是每次都回答四个问题：请求发给了谁，server 返回什么 `RCODE`，answer 来自 recursive cache 还是 authoritative source，TTL 与 DNSSEC/transport 是否改变了结果。

探索时用 `+short`，自动化时固定所有必要 option 并注意 `-r` 的版本兼容性，incident 中保留完整 response；再用 `+norecurse`、`+trace`、`+cd`、`+tcp` 和 `+noedns` 做单变量对照。这样 `dig` 才不只是 lookup command，而是一套能够定位 DNS failure domain 的方法。

## 参考资料

- Julia Evans：[How to use dig](https://jvns.ca/blog/2021/12/04/how-to-use-dig/)
- ISC BIND 9：[dig — DNS lookup utility](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility)
- ISC Knowledge Base：[Using dig +trace](https://kb.isc.org/docs/aa-00208)
- RFC 1035：[Domain Names - Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035)
- RFC 2308：[Negative Caching of DNS Queries](https://www.rfc-editor.org/rfc/rfc2308)
