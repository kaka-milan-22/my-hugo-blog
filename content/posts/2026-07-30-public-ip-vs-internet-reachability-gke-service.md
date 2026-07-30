---
title: "Public IP 不等于公网可达：从 GKE Redis 的 34.118.234.50 讲清地址与路由"
date: 2026-07-30T08:00:00+08:00
draft: true
publish: false
tags: ["Kubernetes", "GKE", "Network", "Redis", "IP Address"]
categories: ["云原生", "网络"]
author: "Kaka"
description: "以 GKE Redis Service 的 34.118.234.50 为例，区分 IP 地址分类、注册归属、BGP 路由、VPC 可达性与 Kubernetes ClusterIP 虚拟转发。"
---

## 引言

排查 GKE 中的 Redis 时，看到下面的输出很容易产生疑问：

```text
$ kubectl -n demo get service redis
NAME    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
redis   ClusterIP   34.118.234.50   <none>        6379/TCP
```

`34.118.234.50` 不属于 RFC 1918，看起来像一个 public IP。它是否意味着 Redis 已经暴露到 Internet？

答案是：**不意味着。**

更准确的判断是：

| 维度 | `34.118.234.50` 的结论 |
|---|---|
| 地址分类 | 位于普通分配的 IPv4 address space，不是 RFC 1918 |
| 注册归属 | 所在的 `34.64.0.0/10` 注册给 Google |
| GKE 用途 | 位于 GKE-managed Service range `34.118.224.0/20` |
| VPC 路由 | 不是普通 VPC route destination |
| Internet 路由 | Google 不为该 Service range 发布公网路由 |
| Kubernetes 语义 | 同集群内由 iptables/eBPF 实现的虚拟 `ClusterIP` |
| Redis 公网暴露 | 没有；`EXTERNAL-IP` 仍是 `<none>` |

本文不会使用任何真实环境 IP。`34.118.234.50` 是从 Google 官方 GKE Service range 中选取的公开示例；其他客户端和 endpoint 地址使用 RFC 5737 documentation ranges 或虚构的 RFC 1918 地址。

## “公网 IP”其实有多种语义

运维沟通中，“这是公网 IP”至少可能表达三件不同的事。

### 语义一：地址分类

它是不是 RFC 1918、Loopback、Link-local、CGNAT、Multicast、Documentation、Benchmarking 或 Reserved address？

如果排除这些特殊用途范围，剩余地址通常可以称为 ordinary allocated、global 或 public address space。

### 语义二：路由可达性

当前 Internet Default-Free Zone 中是否存在覆盖这个地址的 BGP route？从指定网络的 routing table 是否存在有效 next hop？

这是一个动态状态，可能随时间、region、VRF、route policy 和观察位置变化。

### 语义三：服务暴露

即使网络上存在 route，目标是否真的接受连接？中间还可能经过：

```text
Firewall
Security Policy
ACL
NAT
Load Balancer
Service listener
TLS / Authentication
Application policy
```

因此：

```text
public address space
    ≠ Internet 上存在 route
    ≠ 从我的位置可达
    ≠ 端口开放
    ≠ 应用允许访问
```

反过来也一样：RFC 1918 地址可以通过 VPN、Interconnect 或同一 VPC 正常访问，但它仍然不是 public address space。

## 五层判断模型

不要只问“这是公有还是私有”，而要逐层回答：

| 层次 | 核心问题 | 典型数据源 |
|---|---|---|
| Address classification | 这个 bit pattern 属于哪类地址？ | IANA registry、RFC、语言标准库 |
| Registration | 这个地址块由谁管理？ | RIR RDAP / WHOIS |
| Routing control plane | 哪些网络宣布了覆盖 prefix？ | BGP RIB、route collector、云路由表 |
| Data-plane reachability | 从指定 source 到 destination 能否转发？ | `ip route get`、flow log、traceroute、TCP test |
| Service semantics | 这个地址代表 interface、VIP、NAT 还是 application endpoint？ | Kubernetes Service、Load Balancer、dataplane 配置 |

这些答案可能完全不同。RIR registration 说明地址资源的管理关系，不代表 registrant 必须发布 BGP route；BGP 中有 route 也不代表 firewall 和应用会放行。

## 地址分类：先看 IANA Special-Purpose Registry

常见 IPv4 特殊用途范围包括：

| 范围 | 类型 | Globally Reachable |
|---|---|---|
| `10.0.0.0/8` | RFC 1918 Private-Use | False |
| `172.16.0.0/12` | RFC 1918 Private-Use | False |
| `192.168.0.0/16` | RFC 1918 Private-Use | False |
| `100.64.0.0/10` | RFC 6598 Shared Address Space / CGNAT | False |
| `127.0.0.0/8` | Loopback | False |
| `169.254.0.0/16` | Link Local | False |
| `192.0.2.0/24` | TEST-NET-1 Documentation | False |
| `198.51.100.0/24` | TEST-NET-2 Documentation | False |
| `203.0.113.0/24` | TEST-NET-3 Documentation | False |
| `198.18.0.0/15` | Benchmarking | False |
| `224.0.0.0/4` | Multicast，不属于普通 unicast | N/A |
| `240.0.0.0/4` | Reserved | False |

IANA registry 的 `Globally Reachable` 字段描述一个 special-purpose prefix 是否可以被转发到指定 administrative domain 之外。IANA 同时明确指出：**registry 中的地址前缀不保证在任何特定 local 或 global context 中可路由。**

由此可以建立一个重要边界：

```text
IANA Special-Purpose Registry 是地址语义数据库
Internet BGP table 才是某一时刻的路由控制面状态
```

`34.118.234.50` 不属于 IANA 通用的 IPv4 Special-Purpose ranges，所以从分类角度，它不是 RFC 1918 或其他通用特殊地址。

## `is_global` 只能回答分类问题

Python `ipaddress` 可以快速检查标准库对地址的分类：

```bash
uv run python - <<'PY'
import ipaddress

for value in [
    "34.118.234.50",
    "10.44.1.12",
    "192.0.2.10",
    "100.64.0.1",
]:
    ip = ipaddress.ip_address(value)
    print(
        value,
        f"private={ip.is_private}",
        f"global={ip.is_global}",
        f"reserved={ip.is_reserved}",
    )
PY
```

在当前 Python 中，结果类似：

```text
34.118.234.50 private=False global=True  reserved=False
10.44.1.12    private=True  global=False reserved=False
192.0.2.10    private=True  global=False reserved=False
100.64.0.1    private=False global=False reserved=False
```

这里有两个陷阱。

第一，`is_private` 不等于“是否属于 RFC 1918”。Python 根据 IANA 特殊用途语义处理 documentation 等地址，所以 `192.0.2.10` 也可能返回 `private=True`。

第二，`is_global=True` 不会查询 BGP、VPC route、firewall 或 Kubernetes Service。它只说明标准库没有把该地址归入 non-global 特殊类别。

因此下面的推断是错误的：

```text
ipaddress.is_global == True
所以目标一定能从 Internet 访问
```

## Registration 也不等于 Route

查询 ARIN RDAP 可以看到 `34.118.234.50` 所在地址块的注册信息：

```bash
curl -fsS \
  https://rdap.arin.net/registry/ip/34.118.234.50 |
  jq '{
    name,
    startAddress,
    endAddress,
    entities: [.entities[]? | {handle, roles}]
  }'
```

简化结果：

```json
{
  "name": "GOOGL-2",
  "startAddress": "34.64.0.0",
  "endAddress": "34.127.255.255",
  "entities": [
    {
      "handle": "GOOGL-2",
      "roles": ["registrant"]
    }
  ]
}
```

这证明地址空间注册给 Google，但不能推出：

1. Google 正在向 Internet 宣布 `34.118.224.0/20`。
2. 这个 IP 绑定在某张 network interface 上。
3. 从当前 source network 存在有效 route。
4. Redis 正在监听这个地址。
5. Firewall 允许访问 TCP/6379。

RDAP 回答“由谁管理”，BGP 和实际 dataplane 才回答“流量如何到达”。

## 为什么 GKE 会使用 `34.118.224.0/20`

Google Cloud 当前文档规定：

```text
GKE Autopilot 1.27+
GKE Standard 1.29+
```

默认可以从 GKE-managed range `34.118.224.0/20` 为 Service 分配 IPv4 地址。这个设计避免每个集群都从 VPC subnet secondary range 中单独预留 Service CIDR，也不消耗 subnet secondary range quota。

Google 对该范围的处理非常明确：

1. 将 `34.118.224.0/20` 用于 GKE 内部 Service。
2. 不为该范围发布 public Internet route。
3. 不能把该范围用于资源的 external IPv4 address。
4. Service IP 不在 cluster VPC 中作为普通目标路由。

所以 `34.118.234.50` 同时具有两种看起来矛盾、实际并不冲突的属性：

```text
地址块层面：来自注册给 Google 的普通 IPv4 address space
使用层面：被 GKE 当成不可公网路由的内部 Service VIP
```

还要再补一个细节：地址块的 allocation 是全球唯一的，但一个具体 Service IP 不一定对应全球唯一 endpoint。多个彼此隔离的 GKE cluster 可以内部使用相同 Service range，甚至出现相同的 `ClusterIP`；真正的 endpoint identity 还包含 cluster context。

## ClusterIP 不是普通 Interface Address

Kubernetes `ClusterIP` 是 virtual IP。它通常不会像 VM address 那样绑定到一张可从 VPC 路由到达的普通 network interface。

在 GKE 中，Service 和 EndpointSlice 被转换为节点 dataplane 规则。一个 Pod 访问 Redis 时，路径类似：

```text
Client Pod
10.44.2.21
    │
    │ dst=34.118.234.50:6379
    ▼
Node iptables / eBPF
    │
    │ DNAT
    ▼
Redis Pod
10.44.1.12:6379
```

节点 dataplane 捕获匹配 `Service IP + protocol + port` 的流量，把 destination 改写为某个 ready endpoint 的 Pod IP，然后再按 Pod route 转发。

这解释了三个现象。

### VPC 中没有普通 route

Service IP 不是通过 VPC route 直接送到某个 node。Google 官方文档说明，它只供同一 cluster 内的 client Pod 使用。

### 多个 cluster 可以复用

每个 cluster 在自己的 nodes 上维护 dataplane 规则。即使两个 cluster 都存在 `34.118.234.50`，它们也会各自转发到自己的 endpoints。

### `ping` 不能证明 Service 是否正常

Kubernetes Service 定义的是 TCP、UDP 或 SCTP port，不是 ICMP endpoint。iptables/eBPF 规则匹配的是 Service protocol 与 port，`ping ClusterIP` 成功或失败都不是可靠的健康结论。

正确测试应该使用实际应用协议，例如：

```bash
redis-cli -h redis.demo.svc.cluster.local -p 6379 PING
```

## 实战：在 GKE 中验证 Redis Service

下面是一个只用于网络演示的 Redis。它没有持久化、认证和高可用配置，不能直接用于生产。

### 创建 Redis 与 ClusterIP Service

保存为 `redis-demo.yaml`：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7.4-alpine
        args:
        - redis-server
        - --save
        - ""
        - --appendonly
        - "no"
        - --protected-mode
        - "no"
        ports:
        - name: redis
          containerPort: 6379
        readinessProbe:
          exec:
            command: ["redis-cli", "PING"]
          initialDelaySeconds: 2
          periodSeconds: 5
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 250m
            memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: demo
spec:
  type: ClusterIP
  selector:
    app: redis
  ports:
  - name: redis
    protocol: TCP
    port: 6379
    targetPort: redis
```

应用并等待 Pod Ready：

```bash
kubectl create namespace demo \
  --dry-run=client \
  -o yaml | kubectl apply -f -

kubectl apply --server-side --dry-run=server -f redis-demo.yaml
kubectl apply -f redis-demo.yaml
kubectl -n demo rollout status deployment/redis
```

这里没有手工设置 `clusterIP`，由 GKE 自动分配。下面使用 `34.118.234.50` 模拟输出，避免引用任何真实 cluster：

```text
$ kubectl -n demo get service redis -o wide
NAME    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    SELECTOR
redis   ClusterIP   34.118.234.50   <none>        6379/TCP   app=redis
```

关键字段是：

```text
TYPE=ClusterIP
EXTERNAL-IP=<none>
```

这两个字段已经说明它不是公网入口。

### 查看虚拟 Service 与真实 Endpoint

```bash
kubectl -n demo get service redis -o yaml

kubectl -n demo get endpointslice \
  -l kubernetes.io/service-name=redis \
  -o wide
```

概念上的映射是：

```text
Service VIP
34.118.234.50:6379
        │
        └── EndpointSlice
              └── 10.44.1.12:6379
```

`34.118.234.50` 是 stable virtual frontend，`10.44.1.12` 是会随 Pod 重建变化的 backend endpoint。

### 从同一 cluster 测试

```bash
kubectl -n demo run redis-client \
  --rm \
  --stdin \
  --tty \
  --restart=Never \
  --image=redis:7.4-alpine \
  -- redis-cli \
     -h redis.demo.svc.cluster.local \
     -p 6379 \
     PING
```

预期返回：

```text
PONG
```

DNS 最终解析为 ClusterIP，node dataplane 再把连接转发给 ready Redis Pod。

### 为什么 VPC VM 或 Internet Client 访问失败

假设 VPC VM 使用虚构地址 `10.20.0.10`，Internet client 使用 RFC 5737 地址 `198.51.100.10`：

```text
10.20.0.10      ──X──> 34.118.234.50:6379
198.51.100.10   ──X──> 34.118.234.50:6379
```

VPC VM 的流量不会自然经过某个 GKE node 的 Service DNAT 规则；Internet 上又没有该 GKE-managed range 的公开路由。失败原因不是 Redis 密码，也不是 TCP/6379 firewall，而是更早的 routing / service semantics 层就不成立。

## 如果 VPC 中的客户端确实需要访问 Redis

保留 ClusterIP 给 cluster 内调用，再创建独立的 internal LoadBalancer frontend：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-internal
  namespace: demo
  annotations:
    networking.gke.io/load-balancer-type: "Internal"
spec:
  type: LoadBalancer
  selector:
    app: redis
  ports:
  - name: redis
    protocol: TCP
    port: 6379
    targetPort: redis
```

创建完成后，这个 Service 会同时拥有两类地址：

```yaml
spec:
  clusterIP: 34.118.234.50
status:
  loadBalancer:
    ingress:
    - ip: 10.20.0.25
```

以上两个地址仍然只是示例：

| 字段 | 含义 |
|---|---|
| `spec.clusterIP` | cluster 内部 virtual Service IP |
| `status.loadBalancer.ingress[].ip` | VPC client 实际访问的 internal Load Balancer frontend |

不要看到 Service `type: LoadBalancer` 就把 `clusterIP` 当成入口。真正的 Load Balancer address 在 `status.loadBalancer.ingress`。

对于生产 Redis，更合理的选择通常是：

1. 同集群应用：保留 `ClusterIP`。
2. VPC 内应用：internal LoadBalancer，并限制 firewall source。
3. 跨网络：VPN、Interconnect、Private Service Connect 或受控 proxy。
4. 托管场景：使用 Memorystore 等 managed service。
5. 不建议直接创建 public LoadBalancer 暴露 Redis。

Redis 自身还必须配置 authentication、TLS、ACL、persistence、replication、backup 和 resource isolation。网络可达不等于访问安全。

## 一套可落地的排查顺序

### 第一步：确认你看到的是什么字段

```bash
kubectl -n demo get service redis \
  -o jsonpath='{.spec.type}{"\t"}{.spec.clusterIP}{"\t"}{.status.loadBalancer.ingress}{"\n"}'
```

先区分 ClusterIP 与 Load Balancer ingress，不要只复制一个 IP 去搜索。

### 第二步：检查 Service 是否有 ready endpoints

```bash
kubectl -n demo get endpointslice \
  -l kubernetes.io/service-name=redis

kubectl -n demo get pods \
  -l app=redis \
  -o wide
```

ClusterIP 存在但没有 ready endpoint，连接仍会失败。

### 第三步：从正确的观察点测试

```text
ClusterIP       → 从同 cluster Pod 测试
Internal LB     → 从同 VPC / 已连接网络测试
External LB     → 从允许的 Internet source 测试
Pod IP          → 按 CNI/VPC routing 范围测试
```

“从我的 laptop 不通”不能直接证明 cluster 内也不通。

### 第四步：使用真实协议

Redis 使用：

```bash
redis-cli -h HOST -p 6379 PING
```

HTTP 使用 `curl`，TLS 使用 `openssl s_client`，TCP listener 可以用 `nc`。不要用 `ping` 替代 L4/L7 检查。

### 第五步：最后再看 route、firewall 与 dataplane

如果 Service、EndpointSlice 和应用协议都正确，再按路径检查：

```text
Source route
VPC route / BGP route
Firewall / NetworkPolicy
Load Balancer health
Node iptables / eBPF
Pod route
Application listener
```

按层排查比反复修改 firewall 更快，也能避免把 virtual IP 当成 interface address。

## 对 `34.118.234.50` 的准确表述

不严谨的说法：

```text
它不是私网 IP，所以它一定是公网 IP。
```

按地址分类更准确的说法：

```text
34.118.234.50 位于注册给 Google 的 ordinary/global IPv4 address space，
不属于 RFC 1918 或 IANA 通用 Special-Purpose ranges。
```

按 GKE 实际用途更准确的说法：

```text
34.118.234.50 位于 GKE-managed Service range 34.118.224.0/20，
在当前 cluster 中作为 virtual ClusterIP 使用。
Google 不为该范围发布 public Internet route，
该 Service IP 也不是 VPC 内的普通 route destination。
```

按运维语义最直接的说法：

```text
这是一个长得像 public IP 的 GKE internal Service VIP，
不是 Redis 的公网入口。
```

三种说法关注不同层次，并不互相矛盾。

## 常见误区

### 不在 RFC 1918 就一定公网可达

错误。还存在 CGNAT、Documentation、Benchmarking、Link-local、Reserved 等范围；即使属于普通分配空间，也可能没有 BGP route，或被组织内部私用。

### RDAP 显示 Google 就是 Google 公网服务

错误。RDAP 说明 address resource registration，不说明具体 product、route 或 endpoint。

### ClusterIP 可以从同 VPC VM 直接访问

GKE Service IP 默认不是 VPC-routable address。需要 internal LoadBalancer、proxy 或其他明确的暴露机制。

### `ping` 不通说明 Redis 挂了

错误。ClusterIP 是基于 protocol/port 的 virtual Service，应该用 `redis-cli` 检查 Redis protocol。

### LoadBalancer Service 的 `clusterIP` 就是入口

错误。Load Balancer frontend 查看 `.status.loadBalancer.ingress`，`clusterIP` 仍服务于 cluster 内部流量。

### “全球唯一地址”对应全球唯一 Service

错误。全球唯一描述 address allocation。GKE 可以在隔离的 cluster context 中复用 Service range，同一个数值可能对应不同 cluster 的不同 Service。

## 总结

判断一个 IP 时，必须把五层问题分开：

```text
地址分类：它属于哪种地址空间？
注册归属：由哪个组织管理？
路由状态：当前是否存在有效 route？
数据面：从指定 source 是否真的可达？
服务语义：它代表 interface、Load Balancer、NAT 还是 virtual Service？
```

`34.118.234.50` 从分类上位于 Google 管理的 ordinary/global IPv4 address space，但在 GKE 中可以被用作 internal virtual Service IP。Google 不发布 `34.118.224.0/20` 的公网路由，GKE Service IP 也不在 VPC 中作为普通路由目标；同集群 Pod 的流量依靠 node 上的 iptables/eBPF DNAT 才能到达 Redis endpoint。

所以最终结论不是简单的“公网 IP”或“私网 IP”，而是：

> **地址属性属于 public/global address space；运行时语义是 GKE ClusterIP；Internet 与普通 VPC routing 均不可达。**

## 参考资料

- [Google Cloud：VPC-native clusters 与 GKE-managed Service range](https://cloud.google.com/kubernetes-engine/docs/concepts/alias-ips)
- [Google Cloud：GKE Internal Load Balancing](https://cloud.google.com/kubernetes-engine/docs/how-to/internal-load-balancing)
- [Kubernetes：Virtual IPs and Service Proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/)
- [IANA IPv4 Special-Purpose Address Registry](https://www.iana.org/assignments/iana-ipv4-special-registry/iana-ipv4-special-registry.xhtml)
- [IANA IPv4 Address Space Registry](https://www.iana.org/assignments/ipv4-address-space/ipv4-address-space.xhtml)
- [RFC 5737：IPv4 Address Blocks Reserved for Documentation](https://www.rfc-editor.org/rfc/rfc5737)
- [RFC 1918：Address Allocation for Private Internets](https://www.rfc-editor.org/rfc/rfc1918)
- [RFC 6598：Shared Address Space for CGNAT](https://www.rfc-editor.org/rfc/rfc6598)
- [ARIN RDAP：34.118.234.50](https://rdap.arin.net/registry/ip/34.118.234.50)

---

*本文基于 2026 年 7 月的 Google Cloud、Kubernetes、IANA 与 IETF 官方资料编写。所有环境名称与 IP 均为公开文档地址或虚构示例，不包含真实生产环境信息。*
