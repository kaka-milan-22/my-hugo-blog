---
title: "Kubernetes ServiceAccount 实战：Pod 身份、Token 与跨 Namespace 授权"
date: 2026-07-29T08:00:00+08:00
draft: false
tags: ["Kubernetes", "ServiceAccount", "RBAC", "Security", "Workload Identity"]
categories: ["云原生", "DevOps"]
author: "Kaka"
description: "解释 ServiceAccount 如何成为 Pod 的 Kubernetes 身份，演示最小权限绑定、token automount、projected token，以及不使用 ClusterRoleBinding 的跨 namespace 授权。"
---

## 引言

RBAC 定义“某个身份能做什么”，但 Pod 首先要有身份。Kubernetes 使用 ServiceAccount 为 workload、controller、Job 和自动化程序提供 non-human identity。

理解 ServiceAccount 最重要的一句话是：

```text
ServiceAccount 提供身份
RBAC 给这个身份授权
Token 只是身份凭据
```

ServiceAccount 本身没有权限。创建一个 ServiceAccount 不会自动获得 Pod、Secret 或 Deployment 的访问能力；只有 RoleBinding 或 ClusterRoleBinding 把角色授给它以后，API Server 才会允许对应请求。

> 本文是 Kubernetes 权限系列第 2/4 篇，重点是 Pod identity、token 与跨 namespace 授权。

## ServiceAccount 到底是什么

ServiceAccount 是 Kubernetes API 中的 namespaced object：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: inventory-reader
  namespace: applications
```

它在认证系统中的完整用户名是：

```text
system:serviceaccount:<namespace>:<name>
```

上面的 ServiceAccount 对应：

```text
system:serviceaccount:applications:inventory-reader
```

同时属于以下 group：

```text
system:serviceaccounts
system:serviceaccounts:applications
system:authenticated
```

这意味着 RBAC 既可以绑定单个 ServiceAccount，也可以绑定某个 namespace 或整个集群中的所有 ServiceAccount。后两种范围很大，生产环境应谨慎使用。

## User、ServiceAccount 与 Pod 的关系

| 身份 | 存放位置 | 典型使用者 | 生命周期 |
|---|---|---|---|
| User / Group | 外部 IdP 或证书系统 | 人类用户 | 由外部 IAM 管理 |
| ServiceAccount | Kubernetes API，属于 namespace | Pod、controller、CI/CD | 跟随集群对象管理 |
| Pod | 通过 `serviceAccountName` 选择 ServiceAccount | workload process | Pod 生命周期 |

Pod spec 中的 `serviceAccountName` 决定它以哪个 ServiceAccount 身份调用 API：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: inventory-agent
  namespace: applications
spec:
  serviceAccountName: inventory-reader
  containers:
  - name: agent
    image: registry.k8s.io/pause:3.10
```

Pod 创建后不能修改 `serviceAccountName`。如果没有显式指定，Kubernetes 会使用当前 namespace 中名为 `default` 的 ServiceAccount。

## default ServiceAccount 的风险

每个 namespace 创建时都会自动获得一个 `default` ServiceAccount。它默认通常只有 API discovery 等基础权限，但 Pod 仍可能自动挂载它的 token。

不要为了方便给 `default` ServiceAccount 绑定 `view`、`edit` 或自定义高权限角色。否则任何没有显式声明 `serviceAccountName` 的 Pod 都会继承这些权限。

更稳妥的设计是：

```text
default ServiceAccount：无业务权限，关闭 automount
每个 workload：使用独立 ServiceAccount
每个 ServiceAccount：只绑定组件所需的最小 Role
```

可以在 ServiceAccount 层关闭 token 自动挂载：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web-frontend
  namespace: applications
automountServiceAccountToken: false
```

Pod 也可以单独设置：

```yaml
spec:
  serviceAccountName: web-frontend
  automountServiceAccountToken: false
```

如果 ServiceAccount 和 Pod 都设置了该字段，Pod spec 的值优先。一个 workload 不访问 Kubernetes API 时，即使需要 ServiceAccount 名称用于其他 identity integration，也应关闭 API token automount。

## ServiceAccount Token 是怎样工作的

现代 Kubernetes 默认通过 TokenRequest API 给 Pod 提供短期 bound token。kubelet 把它作为 projected volume 挂载，并在过期前自动轮换。

默认挂载目录是：

```text
/var/run/secrets/kubernetes.io/serviceaccount/
├── ca.crt
├── namespace
└── token
```

Token 通常包含：

| Claim | 作用 |
|---|---|
| `sub` | ServiceAccount 身份 |
| `aud` | 允许接收该 token 的 audience |
| `exp` | 过期时间 |
| bound object reference | 将 token 生命周期绑定到 Pod、Secret 或其他对象 |

与旧式永久 ServiceAccount token Secret 相比，projected bound token 具有短生命周期、audience 限制和自动轮换能力。Kubernetes v1.24 起不再为每个 ServiceAccount 自动创建永久 token Secret。

不要把 ServiceAccount token 当成普通配置：

1. 不写入 Git、ConfigMap、镜像或 CI log。
2. 不复制 Pod 内 token 给另一个长期运行的服务。
3. 不创建永久 Secret token 解决临时集成问题。
4. 接收 token 的服务必须验证 signature、expiry、audience 与 bound object。

## 实战一：给 Pod 最小的本 namespace 读取权限

### 场景与目标

`applications` namespace 中有一个 inventory agent，它需要观察 Pod 和 EndpointSlice，但不能：

1. 读取 Secret。
2. 创建或删除 Pod。
3. 修改 Deployment。
4. 访问其他 namespace。

先准备实验 namespace：

```bash
kubectl create namespace applications \
  --dry-run=client \
  -o yaml | kubectl apply -f -
```

### 第一步：创建 ServiceAccount 和 Role

保存为 `inventory-reader.yaml`：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: inventory-reader
  namespace: applications
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: inventory-reader
  namespace: applications
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["discovery.k8s.io"]
  resources: ["endpointslices"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: inventory-reader
  namespace: applications
subjects:
- kind: ServiceAccount
  name: inventory-reader
  namespace: applications
roleRef:
  kind: Role
  name: inventory-reader
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Pod
metadata:
  name: inventory-agent
  namespace: applications
spec:
  serviceAccountName: inventory-reader
  containers:
  - name: agent
    image: registry.k8s.io/pause:3.10
```

先做 server-side dry-run，再应用：

```bash
kubectl apply --server-side --dry-run=server -f inventory-reader.yaml
kubectl apply -f inventory-reader.yaml
```

### 第二步：直接验证 ServiceAccount 的有效权限

不需要进入 Pod，也不需要手工读取 token：

```bash
sa='system:serviceaccount:applications:inventory-reader'

kubectl auth can-i list pods \
  -n applications \
  --as "$sa"

kubectl auth can-i watch endpointslices.discovery.k8s.io \
  -n applications \
  --as "$sa"

kubectl auth can-i get secrets \
  -n applications \
  --as "$sa"

kubectl auth can-i create pods \
  -n applications \
  --as "$sa"

kubectl auth can-i list pods \
  -n payments \
  --as "$sa"
```

预期结果：

```text
applications/list pods           yes
applications/watch endpointslice yes
applications/get secrets         no
applications/create pods         no
payments/list pods               no
```

执行测试的管理员需要 `impersonate` ServiceAccount 的权限。应用自身可以通过 Kubernetes client library 的 in-cluster configuration 使用自动挂载的 token 和 CA，不应在代码中硬编码 token 路径之外的凭据。

## 实战二：跨 namespace 授权

### 场景与错误做法

`ops` namespace 中的 `release-auditor` 需要读取 `payments` namespace 的 Deployment。

常见错误是创建 ClusterRoleBinding：

```text
ClusterRoleBinding → release-auditor 可以读取所有 namespace
```

需求只是访问一个目标 namespace，正确做法是在**目标 namespace** 创建 Role 和 RoleBinding。RoleBinding 的位置决定权限作用范围，subject 可以来自另一个 namespace。

准备两个 namespace：

```bash
kubectl create namespace ops \
  --dry-run=client \
  -o yaml | kubectl apply -f -

kubectl create namespace payments \
  --dry-run=client \
  -o yaml | kubectl apply -f -
```

### 创建跨 namespace binding

保存为 `cross-namespace-auditor.yaml`：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: release-auditor
  namespace: ops
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: release-reader
  namespace: payments
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: release-auditor-from-ops
  namespace: payments
subjects:
- kind: ServiceAccount
  name: release-auditor
  namespace: ops
roleRef:
  kind: Role
  name: release-reader
  apiGroup: rbac.authorization.k8s.io
```

应用并验证：

```bash
kubectl apply -f cross-namespace-auditor.yaml

sa='system:serviceaccount:ops:release-auditor'

kubectl auth can-i list deployments.apps \
  -n payments \
  --as "$sa"

kubectl auth can-i list deployments.apps \
  -n orders \
  --as "$sa"

kubectl auth can-i update deployments.apps \
  -n payments \
  --as "$sa"
```

预期只有 `payments/list deployments` 返回 `yes`。如果同一个 ServiceAccount 还要访问 `orders`，就在 `orders` 再创建一份 RoleBinding，而不是扩大为 ClusterRoleBinding。

## 实战三：显式 projected token 与自定义 audience

有时 workload 不需要 Kubernetes API token，却需要向内部服务证明自己的 Kubernetes identity。可以关闭默认 automount，只投射一枚面向指定 audience 的短期 token：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: internal-api-client
  namespace: applications
automountServiceAccountToken: false
---
apiVersion: v1
kind: Pod
metadata:
  name: internal-api-client
  namespace: applications
spec:
  serviceAccountName: internal-api-client
  automountServiceAccountToken: false
  containers:
  - name: client
    image: registry.k8s.io/pause:3.10
    volumeMounts:
    - name: identity-token
      mountPath: /var/run/secrets/internal-api
      readOnly: true
  volumes:
  - name: identity-token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          audience: https://internal-api.example.com
          expirationSeconds: 3600
```

这枚 token 的 audience 不是 Kubernetes API Server，而是 `https://internal-api.example.com`。内部服务应通过 TokenReview API 或 Kubernetes OIDC discovery 验证它，并显式检查 audience；使用 TokenReview 时，验证服务自身还需要调用 `tokenreviews.authentication.k8s.io` 的最小 RBAC 权限。kubelet 会在过期前更新 projected volume，应用必须重新读取文件，不能只在启动时缓存 token。

临时调试可以请求短期 token：

```bash
kubectl create token internal-api-client \
  -n applications \
  --audience=https://internal-api.example.com \
  --duration=10m
```

该命令会把 token 输出到终端，只适合受控调试。不要把输出复制到脚本、工单、聊天工具或 CI log。

## 401 与 403 应该怎样排查

调用 API 失败时，先区分 Authentication 与 Authorization：

| 返回结果 | 含义 | 常见原因 |
|---|---|---|
| `401 Unauthorized` | API Server 没有接受身份 | token 过期、audience 不匹配、signature 无效 |
| `403 Forbidden` | 身份已认证，但权限不足 | Role/Binding 缺失、namespace 错误、verb 或 subresource 不匹配 |

排查顺序：

```bash
# 1. 确认 Pod 实际使用哪个 ServiceAccount
kubectl get pod inventory-agent \
  -n applications \
  -o jsonpath='{.spec.serviceAccountName}{"\n"}'

# 2. 确认 RoleBinding 的 subject 与目标 namespace
kubectl get rolebinding inventory-reader \
  -n applications \
  -o yaml

# 3. 让 API Server 直接计算授权结果
kubectl auth can-i list pods \
  -n applications \
  --as system:serviceaccount:applications:inventory-reader
```

不要通过解码或打印真实 token 来排查 RBAC。403 已经说明 Authentication 成功，继续查看 token 内容通常没有价值。

## ServiceAccount 的高风险权限

以下授权应作为 privilege escalation risk 审核：

| 权限 | 风险 |
|---|---|
| `get/list/watch secrets` | 直接读取 Secret data |
| `create serviceaccounts/token` | 为其他 ServiceAccount 签发 token |
| 创建 Pod/Deployment/Job | 可以选择 namespace 内其他 ServiceAccount |
| `impersonate serviceaccounts` | 以其他 workload identity 发起请求 |
| 给 `system:serviceaccounts` 绑定角色 | 给集群内所有 ServiceAccount 授权 |
| 给 `system:serviceaccounts:<ns>` 绑定角色 | 给 namespace 内所有 Pod identity 授权 |

即使某个 ServiceAccount 自身权限很小，只要它能创建 workload，就可能让新 Pod 使用更强的 ServiceAccount。这正是下一篇要解决的问题。

## 常见错误

### 给 default ServiceAccount 授权

任何未显式指定身份的 Pod 都会继承权限，授权范围难以追踪。

### 用 ClusterRoleBinding 实现单 namespace 访问

跨 namespace 不等于 cluster-wide。在目标 namespace 创建 RoleBinding 即可。

### 把长期 token 放到 Secret

永久 token 不自动过期，泄露后的有效窗口不可控。优先 TokenRequest 和 projected bound token。

### 认为 automount false 等于 ServiceAccount 无效

它只是不把 Kubernetes API credential 自动放进 Pod。Pod 仍然声明了 ServiceAccount identity，可用于 admission rule、云 workload identity 或显式 projected token。

### 为每个 Pod 创建 ServiceAccount

ServiceAccount 应按 workload security boundary 建模，而不是机械地一 Pod 一个。Deployment 的所有 replicas 通常共享同一 ServiceAccount，但不同组件或不同权限级别不能共享。

## 总结

ServiceAccount 是 workload identity，不是权限集合。Pod 通过 `serviceAccountName` 选择身份，API Server 使用短期 projected token 完成认证，RBAC 再决定这个身份能访问哪些资源。

生产设计应做到：一个组件一个清晰身份、默认关闭不需要的 token、使用 bound token、在目标 namespace 做跨 namespace RoleBinding，并通过 `kubectl auth can-i --as system:serviceaccount:...` 验证最终权限。

下一篇将处理 ServiceAccount 带来的关键安全边界：为什么一个只有 `create deployments` 的用户，仍可能借用 namespace 内的高权限身份，以及如何用 Pod Security 与 Admission Policy 阻止它。

## 系列文章

1. [Kubernetes RBAC 基础与实战：四个对象讲清最小权限](/posts/2026-07-29-kubernetes-rbac-basics-practice/)
2. Kubernetes ServiceAccount 实战：Pod 身份、Token 与跨 Namespace 授权（本文）
3. [为什么 Kubernetes 只有 RBAC 不够：从隐式提权到 Admission Policy](/posts/2026-07-29-why-kubernetes-rbac-is-not-enough/)
4. [Kubernetes 权限体系进阶：完整 Access Control 架构](/posts/2026-07-29-kubernetes-access-control-rbac-admission-policy/)

## 参考资料

- [Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/)
- [Configure Service Accounts for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
- [Managing Service Accounts](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)
- [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [TokenRequest API](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-request-v1/)

---

*本文基于 Kubernetes v1.36 编写。*
