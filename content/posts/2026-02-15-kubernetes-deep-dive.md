---
title: "Kubernetes 深度解析：为什么它成为容器编排的事实标准？"
date: 2026-02-15T11:00:00+08:00
draft: false
tags: ["kubernetes", "k8s", "container", "orchestration", "devops"]
categories: ["Container", "Cloud Native", "DevOps"]
author: "Kaka"
description: "深入剖析 Kubernetes 的核心架构、组件功能及其成为容器编排标准的原因，帮助运维工程师全面掌握 K8s 技术栈"
---

![k8s_blog_800w.jpg](https://img.kakacn.com/file/1771166552159_k8s_blog_800w.jpg)

## 引言

在云原生时代，Kubernetes（简称 K8s）已经成为容器编排领域的事实标准。从初创公司到 Google、Netflix 等科技巨头，无数组织都在使用 K8s 管理他们的容器化应用。作为运维工程师，深入理解 K8s 的架构和运作机制，已经成为必备技能。

本文将从运维视角深入剖析 Kubernetes，解答三个核心问题：
1. 为什么 K8s 如此流行？
2. K8s 的整体架构是怎样的？
3. 各个核心组件如何协同工作？

## 为什么 Kubernetes 如此流行？

### 微服务时代的运维挑战

在传统单体应用时代，运维相对简单：

```
单体应用模式：
部署：手动上传 WAR/JAR → 重启 Tomcat
扩展：购买更大的服务器（垂直扩展）
监控：单一进程，易于追踪
```

但随着微服务架构的兴起，一切都变了：

```
微服务架构：
应用被拆分为 → 50+ 独立服务
每个服务 → 多个实例（3-10个）
总容器数量 → 200-500+ 个容器

手动管理？几乎不可能！
```

**真实场景示例**

假设你在运维一个电商平台：
- 用户服务：10 个实例
- 订单服务：15 个实例
- 支付服务：8 个实例
- 商品服务：12 个实例
- 推荐服务：20 个实例
- ... 还有更多服务

现在遇到促销活动，流量突增 10 倍：
- ❌ **没有 K8s**：凌晨 2 点被叫醒，手动 SSH 到服务器，一个个启动容器，配置负载均衡...
- ✅ **有了 K8s**：修改副本数配置，K8s 自动完成扩容、负载均衡、健康检查

### Kubernetes 解决的核心问题

**1. 高可用性（High Availability）**

K8s 通过多种机制保证应用持续运行：

```yaml
# 声明期望状态
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  replicas: 3  # 期望 3 个副本
  selector:
    matchLabels:
      app: payment
  template:
    metadata:
      labels:
        app: payment
    spec:
      containers:
      - name: payment
        image: payment:v2.0
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

当某个 Pod 失败时：
```
时刻 T0: [Pod-1] [Pod-2] [Pod-3]  ← 3个运行中
时刻 T1: [Pod-1] [Pod-2] [×]      ← Pod-3 崩溃
时刻 T2: [Pod-1] [Pod-2] [Pod-4]  ← K8s 自动创建 Pod-4
                        ↑
              无需人工干预！
```

**2. 弹性伸缩（Auto-scaling）**

根据负载自动调整资源：

```yaml
# 水平 Pod 自动伸缩器（HPA）
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # CPU 使用率超过 70% 时扩容
```

实际效果：
```
正常时段：   3 个 Pod  (节省资源)
促销开始：   CPU 使用率 ↑ → 自动扩容到 15 个 Pod
流量高峰：   自动扩容到 50 个 Pod
活动结束：   逐步缩减到 3 个 Pod
```

**3. 自我修复（Self-healing）**

K8s 持续监控集群状态并自动修复问题：

| 问题场景 | K8s 的反应 |
|---------|-----------|
| Pod 崩溃 | 自动重启 Pod |
| 容器健康检查失败 | 重启容器或替换 Pod |
| 节点宕机 | 在健康节点上重新调度 Pod |
| 资源不足 | 驱逐低优先级 Pod，保证核心服务 |

```yaml
# 健康检查配置
spec:
  containers:
  - name: app
    image: myapp:1.0
    livenessProbe:      # 存活探针：检测容器是否运行
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:     # 就绪探针：检测容器是否准备好接收流量
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

**4. 声明式配置（Declarative Configuration）**

传统运维方式（命令式）：
```bash
# 需要记住一堆命令，容易出错
ssh server1
docker run -d --name app1 myapp:1.0
ssh server2
docker run -d --name app2 myapp:1.0
# ... 重复 N 次
```

K8s 方式（声明式）：
```yaml
# 只需描述期望状态，K8s 负责实现
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 10  # 我要 10 个实例
  # ... 其他配置
```

```bash
# 一条命令应用配置
kubectl apply -f deployment.yaml

# K8s 会自动：
# ✓ 拉取镜像
# ✓ 调度到合适的节点
# ✓ 启动容器
# ✓ 配置网络
# ✓ 设置健康检查
# ✓ 持续监控状态
```

**5. 跨平台可移植性**

同一套配置可以运行在任何环境：

```
开发环境（Mac/Windows）
    ↓ 相同的 YAML 配置
测试环境（本地集群）
    ↓ 相同的 YAML 配置
预发布环境（AWS）
    ↓ 相同的 YAML 配置
生产环境（GCP/Azure/私有云）
```

避免"在我机器上可以运行"的经典问题。

## Kubernetes 整体架构

K8s 采用主从架构（Master-Worker），分为控制平面和数据平面。

### 架构全景图

```
┌─────────────────────────────────────────────────────────────┐
│                        Control Plane                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  API Server  │  │  Scheduler   │  │  Controller  │      │
│  │              │  │              │  │   Manager    │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
│         │                  │                  │              │
│         └──────────────────┼──────────────────┘              │
│                            │                                 │
│                    ┌───────▼────────┐                        │
│                    │     etcd       │                        │
│                    │  (配置存储)     │                        │
│                    └────────────────┘                        │
└─────────────────────────────────────────────────────────────┘
                             │
                             │ (gRPC/HTTPS)
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
┌───────▼─────────┐  ┌───────▼─────────┐  ┌───────▼─────────┐
│   Worker Node 1 │  │   Worker Node 2 │  │   Worker Node 3 │
│                 │  │                 │  │                 │
│  ┌───────────┐  │  │  ┌───────────┐  │  │  ┌───────────┐  │
│  │  Kubelet  │  │  │  │  Kubelet  │  │  │  │  Kubelet  │  │
│  └─────┬─────┘  │  │  └─────┬─────┘  │  │  └─────┬─────┘  │
│        │        │  │        │        │  │        │        │
│  ┌─────▼─────┐  │  │  ┌─────▼─────┐  │  │  ┌─────▼─────┐  │
│  │Container  │  │  │  │Container  │  │  │  │Container  │  │
│  │ Runtime   │  │  │  │ Runtime   │  │  │  │ Runtime   │  │
│  │(containerd│  │  │  │(containerd│  │  │  │(containerd│  │
│  │  /CRI-O)  │  │  │  │  /CRI-O)  │  │  │  │  /CRI-O)  │  │
│  └───────────┘  │  │  └───────────┘  │  │  └───────────┘  │
│                 │  │                 │  │                 │
│  ┌───────────┐  │  │  ┌───────────┐  │  │  ┌───────────┐  │
│  │ kube-proxy│  │  │  │ kube-proxy│  │  │  │ kube-proxy│  │
│  └───────────┘  │  │  └───────────┘  │  │  └───────────┘  │
│                 │  │                 │  │                 │
│  [Pod] [Pod]    │  │  [Pod] [Pod]    │  │  [Pod] [Pod]    │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

### 控制平面组件（Control Plane）

控制平面是集群的"大脑"，负责全局决策和管理。

#### 1. API Server

**职责**：集群的统一入口，所有操作都通过 API Server

```
kubectl 命令 ───┐
                │
Web UI 控制台 ──┼──→ API Server ──→ 验证 ──→ 授权 ──→ 准入控制 ──→ 持久化到 etcd
                │
自动化脚本 ─────┘
```

**关键特性**：
- RESTful API 接口
- 身份验证和鉴权
- 准入控制（Admission Control）
- 数据验证
- 集群状态的唯一真实来源

**运维视角**：
```bash
# API Server 是所有操作的入口
kubectl get pods        # 实际是 GET /api/v1/namespaces/default/pods
kubectl create -f app.yaml  # 实际是 POST /api/v1/namespaces/default/pods

# 可以直接访问 API
curl -k https://<api-server>:6443/api/v1/pods \
  -H "Authorization: Bearer $TOKEN"
```

#### 2. etcd

**职责**：分布式键值存储，保存集群的所有配置和状态

```
etcd 存储的数据：
├── /registry/pods/default/myapp-xxx
├── /registry/services/default/myservice
├── /registry/deployments/default/myapp
├── /registry/secrets/default/db-password
└── ... 所有集群资源
```

**关键特性**：
- 强一致性（使用 Raft 协议）
- 键值对存储
- 支持 Watch 机制（监听变化）
- 支持快照和备份

**运维要点**：
```bash
# etcd 是集群的"数据库"，必须定期备份
ETCDCTL_API=3 etcdctl snapshot save backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 恢复集群
ETCDCTL_API=3 etcdctl snapshot restore backup.db
```

**生产环境建议**：
- 至少 3 个 etcd 节点（奇数个以支持选举）
- 使用 SSD 存储（etcd 对 I/O 敏感）
- 定期备份（每天或每小时）
- 监控 etcd 性能指标

#### 3. Scheduler

**职责**：决定 Pod 应该运行在哪个节点上

**调度流程**：

```
1. Watch API Server 获取未调度的 Pod
          ↓
2. 过滤阶段（Filtering）
   筛选出满足条件的节点：
   - 资源是否充足？ (CPU/Memory)
   - 节点是否健康？
   - 有 NodeSelector/Affinity 约束吗？
   - 端口是否冲突？
          ↓
3. 打分阶段（Scoring）
   给每个候选节点打分：
   - 资源利用率均衡
   - Pod 亲和性/反亲和性
   - 数据本地性
          ↓
4. 选择得分最高的节点
          ↓
5. 绑定 Pod 到节点（通过 API Server）
```

**调度策略示例**：

```yaml
# Pod 亲和性：让相关的 Pod 运行在一起
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - cache
        topologyKey: kubernetes.io/hostname

---
# Pod 反亲和性：让副本分散到不同节点
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web
            topologyKey: kubernetes.io/hostname  # 同一节点不运行多个副本
```

**节点选择器**：

```yaml
# 将数据库 Pod 调度到 SSD 节点
apiVersion: v1
kind: Pod
metadata:
  name: mysql
spec:
  nodeSelector:
    disktype: ssd
    zone: us-west-1a
  containers:
  - name: mysql
    image: mysql:8.0
```

#### 4. Controller Manager

**职责**：运行控制器，持续监控集群状态并采取行动

**核心控制器**：

**Deployment Controller**
```
期望状态：3 个副本运行
实际状态：2 个副本运行 (1 个崩溃)

Controller 行动：
1. 检测到差异
2. 创建新的 ReplicaSet
3. 启动 1 个新 Pod
4. 持续监控直到达到期望状态
```

**Node Controller**
```
监控节点健康状态：
- 每 5 秒检查一次心跳
- 40 秒无响应 → 标记为 NotReady
- 5 分钟无响应 → 驱逐该节点上的 Pod
```

**ReplicaSet Controller**
```yaml
# 确保副本数量
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
```

**Endpoint Controller**
```
监控 Service 和 Pod：
Service "web" (selector: app=web)
    ↓ 自动发现匹配的 Pod
Pod1: 10.244.1.5:8080 (Ready)
Pod2: 10.244.2.3:8080 (Ready)
Pod3: 10.244.3.7:8080 (NotReady) ← 不加入 Endpoints
    ↓
创建 Endpoints 对象：
web → [10.244.1.5:8080, 10.244.2.3:8080]
```

### 工作节点组件（Worker Nodes）

工作节点是实际运行应用容器的地方。

#### 1. Kubelet

**职责**：节点上的代理，负责管理本节点的 Pod 和容器

**核心功能**：

```
Kubelet 的工作流程：

1. 监听 API Server
   获取分配到本节点的 Pod 定义
          ↓
2. 调用 Container Runtime
   拉取镜像、创建容器
          ↓
3. 挂载 Volume
   配置存储卷
          ↓
4. 配置网络
   为 Pod 分配 IP
          ↓
5. 执行健康检查
   Liveness/Readiness Probe
          ↓
6. 上报状态
   将 Pod 状态报告给 API Server
```

**健康检查示例**：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: myapp:1.0
    # 存活探针：失败则重启容器
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
    
    # 就绪探针：失败则从 Service 中移除
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      successThreshold: 1
    
    # 启动探针：用于慢启动应用
    startupProbe:
      httpGet:
        path: /startup
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
```

**资源管理**：

```yaml
# Kubelet 根据资源请求和限制管理容器
spec:
  containers:
  - name: app
    resources:
      requests:       # 调度时保证的资源
        memory: "256Mi"
        cpu: "250m"   # 0.25 核
      limits:         # 容器可使用的最大资源
        memory: "512Mi"
        cpu: "500m"
```

#### 2. Container Runtime

**职责**：实际运行容器的底层软件

**发展历程**：

```
Docker 时代 (2014-2020)：
K8s → dockershim → Docker Engine → containerd → runc

CRI 标准化后 (2020+)：
K8s → CRI → containerd/CRI-O → runc
              ↑
         直接对接，更高效
```

**主流 Container Runtime**：

**containerd**（推荐）
```bash
# 安装 containerd
apt-get install containerd

# 配置 CRI 插件
cat > /etc/containerd/config.toml <<EOF
version = 2
[plugins."io.containerd.grpc.v1.cri"]
  [plugins."io.containerd.grpc.v1.cri".containerd]
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
        runtime_type = "io.containerd.runc.v2"
EOF

# 重启服务
systemctl restart containerd
```

**CRI-O**（专为 K8s 设计）
```bash
# 轻量级，只实现 CRI 接口
# 没有 Docker 的额外功能
apt-get install cri-o
systemctl start crio
```

**运维命令对比**：

```bash
# Docker 时代
docker ps
docker logs <container-id>
docker exec -it <container-id> /bin/bash

# containerd 时代
crictl ps          # 查看容器
crictl logs <id>   # 查看日志
crictl exec -it <id> /bin/bash

# 或使用 kubectl（推荐）
kubectl get pods
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- /bin/bash
```

#### 3. kube-proxy

**职责**：维护节点上的网络规则，实现 Service 的负载均衡

**工作模式**：

**iptables 模式**（默认）
```bash
# kube-proxy 创建 iptables 规则
# Service "web-service" (ClusterIP: 10.96.0.10)
# Backend Pods: 10.244.1.5, 10.244.2.3, 10.244.3.7

iptables -t nat -A KUBE-SERVICES \
  -d 10.96.0.10/32 -p tcp --dport 80 \
  -j KUBE-SVC-WEB

iptables -t nat -A KUBE-SVC-WEB \
  -m statistic --mode random --probability 0.33 \
  -j KUBE-SEP-POD1

iptables -t nat -A KUBE-SVC-WEB \
  -m statistic --mode random --probability 0.50 \
  -j KUBE-SEP-POD2

iptables -t nat -A KUBE-SVC-WEB \
  -j KUBE-SEP-POD3
```

**IPVS 模式**（高性能）
```bash
# 使用 Linux 内核的 IPVS 模块
# 支持更多负载均衡算法

# 查看 IPVS 规则
ipvsadm -Ln

# 输出示例：
TCP  10.96.0.10:80 rr
  -> 10.244.1.5:8080         Masq    1      0          0
  -> 10.244.2.3:8080         Masq    1      0          0
  -> 10.244.3.7:8080         Masq    1      0          0
```

**Service 类型**：

```yaml
# ClusterIP：集群内部访问
apiVersion: v1
kind: Service
metadata:
  name: internal-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080

---
# NodePort：通过节点 IP 访问
apiVersion: v1
kind: Service
metadata:
  name: nodeport-service
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # 可在任意节点的 30080 端口访问

---
# LoadBalancer：云厂商提供的负载均衡器
apiVersion: v1
kind: Service
metadata:
  name: lb-service
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

## 容器网络（Container Networking）

K8s 网络遵循以下原则：
- 所有 Pod 可以互相通信（无需 NAT）
- 所有节点可以和所有 Pod 通信
- Pod 看到的自己的 IP 和其他 Pod 看到的一致

### 网络模型

```
┌─────────────────────────────────────────────────┐
│                  Service Layer                   │
│  ClusterIP / NodePort / LoadBalancer / Ingress  │
└────────────────┬────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────┐
│              CNI Plugin Layer                    │
│   Flannel / Calico / Weave / Cilium / etc.     │
└────────────────┬────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────┐
│             Pod Network (Overlay)                │
│   Virtual Network: 10.244.0.0/16                │
│   ┌──────┐  ┌──────┐  ┌──────┐                 │
│   │ Pod1 │  │ Pod2 │  │ Pod3 │                 │
│   └──────┘  └──────┘  └──────┘                 │
└────────────────┬────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────┐
│           Physical Network                       │
│   Node1: 192.168.1.10                           │
│   Node2: 192.168.1.11                           │
│   Node3: 192.168.1.12                           │
└─────────────────────────────────────────────────┘
```

### 主流 CNI 插件对比

**Flannel**（简单易用）
```bash
# 特点：
# - 最简单的 CNI 实现
# - 使用 VXLAN 或 host-gw 模式
# - 适合小型集群

# 安装
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

**Calico**（功能强大）
```yaml
# 特点：
# - 支持网络策略（Network Policy）
# - BGP 路由模式，性能好
# - 支持加密

# 安装
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# 网络策略示例
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend  # 只允许 backend Pod 访问数据库
    ports:
    - protocol: TCP
      port: 3306
```

**Cilium**（eBPF 技术）
```bash
# 特点：
# - 基于 eBPF，性能极佳
# - 提供 L7 网络策略
# - 内置服务网格功能

# 安装
helm install cilium cilium/cilium --version 1.14.0 \
  --namespace kube-system
```

### Pod 间通信示例

```
Pod A (10.244.1.5) 在 Node1
    ↓ 访问 Pod B (10.244.2.3) 在 Node2
    
1. Pod A 发送数据包：
   Src: 10.244.1.5, Dst: 10.244.2.3

2. Node1 的 CNI Plugin：
   查找路由表 → Pod B 在 Node2
   封装数据包（VXLAN/BGP）

3. 物理网络传输：
   Node1 (192.168.1.10) → Node2 (192.168.1.11)

4. Node2 的 CNI Plugin：
   解封装数据包
   转发到 Pod B (10.244.2.3)

5. Pod B 接收数据包
```

## 容器存储（Container Storage）

K8s 提供多种存储抽象来解耦 Pod 和具体存储实现。

### 存储层次结构

```
┌─────────────────────────────────────┐
│         Application Layer            │
│   Pod 使用 Volume 存储数据           │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│      PersistentVolumeClaim (PVC)    │
│   "我需要 10GB 的 SSD 存储"          │
└──────────────┬──────────────────────┘
               │ 绑定
┌──────────────▼──────────────────────┐
│      PersistentVolume (PV)          │
│   实际的存储资源                     │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│       Storage Backend               │
│  NFS / Ceph / AWS EBS / GCE PD     │
└─────────────────────────────────────┘
```

### Volume 类型

**临时存储（emptyDir）**
```yaml
# 与 Pod 生命周期绑定，Pod 删除后数据丢失
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: cache
      mountPath: /cache
  volumes:
  - name: cache
    emptyDir: {}  # 节点上的临时目录
```

**主机路径（hostPath）**
```yaml
# 挂载节点上的目录或文件
# ⚠️ 注意：Pod 重新调度到其他节点时数据不可用
apiVersion: v1
kind: Pod
metadata:
  name: log-collector
spec:
  containers:
  - name: collector
    image: fluentd
    volumeMounts:
    - name: varlog
      mountPath: /var/log
      readOnly: true
  volumes:
  - name: varlog
    hostPath:
      path: /var/log  # 节点的 /var/log 目录
      type: Directory
```

**配置文件（ConfigMap & Secret）**
```yaml
# ConfigMap 存储配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_url: "mysql://db:3306/myapp"
  log_level: "info"

---
# 挂载为 Volume
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: app-config

---
# Secret 存储敏感信息
apiVersion: v1
kind: Secret
metadata:
  name: db-password
type: Opaque
data:
  password: cGFzc3dvcmQxMjM=  # base64 编码

---
# 作为环境变量使用
spec:
  containers:
  - name: app
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-password
          key: password
```

### 持久化存储（PV & PVC）

**静态供给**（手动创建 PV）

```yaml
# 1. 管理员创建 PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteMany  # 多个 Pod 可同时读写
  persistentVolumeReclaimPolicy: Retain  # PVC 删除后保留数据
  nfs:
    server: nfs.example.com
    path: /exported/path

---
# 2. 用户创建 PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 5Gi  # 请求 5GB，会绑定到上面的 10Gi PV

---
# 3. Pod 使用 PVC
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
```

**动态供给**（自动创建 PV）

```yaml
# 1. 定义 StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs  # 使用 AWS EBS
parameters:
  type: gp3  # SSD 类型
  iopsPerGB: "10"
allowVolumeExpansion: true  # 允许扩容

---
# 2. PVC 引用 StorageClass
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  storageClassName: fast-ssd  # 使用 fast-ssd StorageClass
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
# K8s 会自动创建 20GB 的 AWS EBS 卷

---
# 3. StatefulSet 中使用
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:  # 每个 Pod 自动创建独立的 PVC
  - metadata:
      name: data
    spec:
      storageClassName: fast-ssd
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 20Gi
```

### 存储类型对比

| 存储类型 | 使用场景 | 数据持久性 | 访问模式 |
|---------|---------|-----------|---------|
| emptyDir | 临时缓存、中间文件 | Pod 删除后丢失 | RWO |
| hostPath | 节点日志收集 | 节点级持久 | RWO |
| NFS | 共享文件系统 | 独立于 Pod | RWX |
| Ceph RBD | 块存储（数据库） | 独立于 Pod | RWO |
| AWS EBS | 云上块存储 | 独立于 Pod | RWO |
| GCE PD | 云上块存储 | 独立于 Pod | RWO/ROX |

**访问模式说明**：
- **RWO** (ReadWriteOnce)：单个节点读写
- **ROX** (ReadOnlyMany)：多节点只读
- **RWX** (ReadWriteMany)：多节点读写

### CSI（Container Storage Interface）

现代存储驱动标准：

```yaml
# CSI Driver 示例：AWS EBS CSI
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com  # CSI 驱动
parameters:
  type: gp3
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer  # 延迟绑定，确保 PV 在 Pod 所在可用区创建
```

## 核心资源对象

### Pod

最小的部署单元，可包含一个或多个容器：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  # Init Container：在主容器前运行
  initContainers:
  - name: init-db
    image: busybox
    command: ['sh', '-c', 'until nslookup db; do sleep 2; done']
  
  # 主容器
  containers:
  - name: app
    image: myapp:1.0
    ports:
    - containerPort: 8080
    env:
    - name: DB_HOST
      value: "mysql"
  
  # Sidecar：辅助容器
  - name: log-forwarder
    image: fluentd
    volumeMounts:
    - name: logs
      mountPath: /logs
  
  volumes:
  - name: logs
    emptyDir: {}
```

### Deployment

管理无状态应用：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate  # 滚动更新
    rollingUpdate:
      maxSurge: 1        # 最多额外创建 1 个 Pod
      maxUnavailable: 0  # 最多 0 个 Pod 不可用
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:1.21
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

**更新流程**：

```bash
# 更新镜像
kubectl set image deployment/web-app web=nginx:1.22

# K8s 自动执行：
# 1. 创建新版本的 Pod
# 2. 等待新 Pod Ready
# 3. 删除旧版本的 Pod
# 4. 重复，直到所有 Pod 更新完成

# 回滚
kubectl rollout undo deployment/web-app

# 查看历史
kubectl rollout history deployment/web-app
```

### StatefulSet

管理有状态应用（数据库、消息队列）：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql  # Headless Service
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

**StatefulSet 特性**：

```
Pod 命名：有序且稳定
mysql-0
mysql-1
mysql-2

网络标识：稳定的 DNS
mysql-0.mysql.default.svc.cluster.local
mysql-1.mysql.default.svc.cluster.local
mysql-2.mysql.default.svc.cluster.local

存储：独立的 PVC
mysql-0 → data-mysql-0 (10Gi)
mysql-1 → data-mysql-1 (10Gi)
mysql-2 → data-mysql-2 (10Gi)
```

### DaemonSet

在每个节点上运行一个 Pod（日志收集、监控代理）：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true  # 使用主机网络
      containers:
      - name: node-exporter
        image: prom/node-exporter
        ports:
        - containerPort: 9100
          hostPort: 9100
```

## 生产环境最佳实践

### 1. 资源配额和限制

**Namespace 级别配额**：

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    requests.cpu: "100"      # 总 CPU 请求
    requests.memory: "200Gi" # 总内存请求
    limits.cpu: "200"
    limits.memory: "400Gi"
    pods: "50"              # 最多 50 个 Pod
```

**LimitRange**：

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: pod-limits
  namespace: production
spec:
  limits:
  - max:
      cpu: "2"
      memory: "4Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
    default:           # 默认 limit
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:    # 默认 request
      cpu: "250m"
      memory: "256Mi"
    type: Container
```

### 2. 高可用部署

**多副本 + Pod 反亲和性**：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-app
spec:
  replicas: 5
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - critical-app
            topologyKey: kubernetes.io/hostname
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - critical-app
              topologyKey: topology.kubernetes.io/zone
```

**PodDisruptionBudget**（保证最小可用实例）：

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2  # 至少保持 2 个 Pod 运行
  selector:
    matchLabels:
      app: web
```

### 3. 监控和日志

**Prometheus 监控**：

```yaml
# ServiceMonitor：自动发现监控目标
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: app-monitor
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    interval: 30s
```

**关键指标**：

```bash
# 节点资源使用率
node_cpu_usage
node_memory_usage
node_disk_usage

# Pod 资源使用
container_cpu_usage_seconds_total
container_memory_usage_bytes

# 应用指标
http_requests_total
http_request_duration_seconds
```

**日志收集架构**：

```
应用容器日志 (stdout/stderr)
    ↓
DaemonSet (Fluentd/Filebeat)
    ↓
Elasticsearch / Loki
    ↓
Kibana / Grafana
```

### 4. 安全最佳实践

**Pod Security Standards**：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted  # 最严格
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**NetworkPolicy**：

```yaml
# 默认拒绝所有流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# 允许特定流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

## 运维常见问题与解决

### Pod 一直处于 Pending 状态

**排查步骤**：

```bash
# 1. 查看 Pod 事件
kubectl describe pod <pod-name>

# 常见原因和解决方案：
# FailedScheduling: 0/3 nodes are available: insufficient cpu
# → 增加节点或减少 CPU 请求

# FailedScheduling: 0/3 nodes match node selector
# → 检查 nodeSelector 配置

# FailedScheduling: pod has unbound immediate PersistentVolumeClaims
# → 检查 PVC 是否成功绑定

# 2. 查看集群资源
kubectl top nodes
kubectl describe nodes
```

### Pod 不断重启（CrashLoopBackOff）

```bash
# 1. 查看日志
kubectl logs <pod-name>
kubectl logs <pod-name> --previous  # 查看上一次的日志

# 2. 检查健康检查配置
kubectl describe pod <pod-name>

# 常见原因：
# - 应用启动失败
# - livenessProbe 配置不当
# - 资源限制过低（OOMKilled）
```

### Service 无法访问

```bash
# 1. 检查 Service 和 Endpoints
kubectl get svc
kubectl get endpoints <service-name>

# 2. 如果 Endpoints 为空，检查 Pod 标签
kubectl get pods --show-labels

# 3. 测试服务连通性
kubectl run test --image=busybox -it --rm -- wget -O- <service-name>

# 4. 检查 NetworkPolicy
kubectl get networkpolicy
```

### 节点 NotReady

```bash
# 1. 查看节点状态
kubectl describe node <node-name>

# 2. 登录节点检查
ssh <node-ip>
systemctl status kubelet
journalctl -u kubelet -f

# 常见原因：
# - 磁盘空间不足
# - 网络问题
# - Container Runtime 故障
```

## 权衡与选择

### 优势

✅ **自动化运维**：自我修复、自动扩展、滚动更新
✅ **高可用性**：多副本、健康检查、故障转移
✅ **可移植性**：跨云平台、跨环境一致性
✅ **声明式配置**：GitOps 友好，易于版本控制
✅ **生态丰富**：海量的工具和插件

### 劣势

❌ **复杂性高**：学习曲线陡峭，需要专业团队
❌ **资源开销**：控制平面需要额外资源
❌ **过度设计**：小型应用可能不需要 K8s
❌ **运维成本**：需要持续维护和监控

### 适用场景

**适合使用 K8s**：
- 微服务架构，服务数量 > 10
- 需要频繁部署和扩展
- 多环境部署（开发/测试/生产）
- 有专业的运维团队

**可以考虑替代方案**：
- 简单应用：Docker Compose
- Serverless 场景：AWS Lambda, Cloud Functions
- 托管服务：Cloud Run, App Engine

### 托管 K8s 服务

如果团队规模有限，建议使用托管服务：

| 服务 | 提供商 | 特点 |
|------|--------|------|
| EKS | AWS | 深度集成 AWS 服务 |
| GKE | Google Cloud | Google 原生支持，体验最好 |
| AKS | Azure | 与 Azure 服务集成 |
| ACK | 阿里云 | 国内延迟低 |
| TKE | 腾讯云 | 国内云服务 |

**托管服务的优势**：
- 控制平面由云厂商管理
- 自动升级和打补丁
- 集成云平台服务（负载均衡、存储、监控）
- 降低运维复杂度

## 总结

Kubernetes 之所以成为容器编排的事实标准，是因为它系统性地解决了大规模容器化应用的管理难题：

**核心价值**：
1. 通过声明式 API 简化运维
2. 自动化实现高可用和弹性伸缩
3. 提供标准化的容器运行环境
4. 丰富的生态系统和工具链

**架构优势**：
- 控制平面负责决策，工作节点执行任务
- 松耦合的组件设计，易于扩展
- 插件化的网络和存储方案
- 强大的控制器模式

**运维要点**：
- 理解各组件的职责和协作方式
- 掌握网络和存储的工作原理
- 建立完善的监控和日志体系
- 遵循安全和资源管理最佳实践

对于运维工程师而言，K8s 不仅是一个工具，更是一套完整的云原生运维理念。虽然学习曲线陡峭，但一旦掌握，它将极大提升团队的交付效率和系统的可靠性。

## 参考资源

- [Kubernetes 官方文档](https://kubernetes.io/docs/)
- [CNCF Landscape](https://landscape.cncf.io/)
- [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- [K8s 生产最佳实践](https://learnk8s.io/production-best-practices)

---

*本文基于 Kubernetes 1.28+ 版本编写，适用于运维工程师深入学习 K8s 架构和实践。*
