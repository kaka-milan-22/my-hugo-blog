---
title: "Kubernetes RBAC 基础与实战：四个对象讲清最小权限"
date: 2026-07-29T07:00:00+08:00
draft: false
tags: ["Kubernetes", "RBAC", "Security", "Authorization", "DevOps"]
categories: ["云原生", "DevOps"]
author: "Kaka"
description: "从 subject、scope、resource、verb 四个维度理解 Kubernetes RBAC，并通过 namespace 级应用发布权限完成一套可验证的最小权限配置。"
---

## 引言

Kubernetes RBAC 看起来有四种对象、多个 API group 和一组不太直观的 verb，但它的核心模型并不复杂：

```text
谁（subject）
  在哪里（scope）
  对什么资源（resource / subresource）
  做什么操作（verb）
```

只要先回答这四个问题，再选择 Role、ClusterRole、RoleBinding 和 ClusterRoleBinding，绝大多数权限配置都可以直接推导出来。

> 本文是 Kubernetes 权限系列第 1/4 篇，只讨论 RBAC 授权模型。下一篇再讨论 Pod 如何通过 ServiceAccount 获得身份。

## RBAC 解决什么问题

Authentication 负责确认请求者是谁，Authorization 决定这个身份能做什么。RBAC（Role-Based Access Control）是 Kubernetes 最常用的 authorizer，它根据 API request attributes 作出授权判断：

```text
user/group/serviceaccount
        +
namespace
        +
apiGroup/resource/subresource
        +
verb
        =
allow 或 no opinion
```

RBAC 规则是 additive：所有 RoleBinding 和 ClusterRoleBinding 的授权结果会累加。它没有显式 deny，也不检查对象中的 image、label、securityContext 等字段。

例如，RBAC 可以表达：

```text
允许 team-payments 在 payments namespace 更新 Deployment
```

但不能表达：

```text
允许更新 Deployment，但镜像必须来自 registry.example.com，
且不允许 privileged container
```

后一个问题属于 Admission Policy，不属于 RBAC。

## 四种 RBAC 对象

RBAC 把“权限定义”和“把权限交给谁”拆成两组对象：

| 对象 | 作用域 | 职责 |
|---|---|---|
| Role | namespace | 定义某个 namespace 内的权限 |
| ClusterRole | cluster | 定义 cluster-scoped 权限，或提供可复用的 namespaced 角色模板 |
| RoleBinding | namespace | 在一个 namespace 内给 subject 绑定 Role 或 ClusterRole |
| ClusterRoleBinding | cluster | 把 ClusterRole 的权限授予整个集群范围 |

最容易混淆的是 `ClusterRole`：它不等于“使用后一定拥有全局权限”。一个 ClusterRole 如果通过 RoleBinding 绑定，其 namespaced rules 只在 RoleBinding 所在 namespace 生效。

```text
ClusterRole + RoleBinding        → 只在一个 namespace 生效
ClusterRole + ClusterRoleBinding → 在所有 namespace / cluster scope 生效
```

这使平台团队可以维护一份标准角色模板，再由每个业务 namespace 自己绑定，避免复制大量 Role。

## Role：定义 namespace 内的权限

下面的 Role 允许读取 `payments` namespace 中的 Pod：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: payments
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

Role 只定义权限，不指定用户。真正把权限授予用户、group 或 ServiceAccount，需要 RoleBinding。

## ClusterRole：集群权限或角色模板

ClusterRole 有两种典型用途。

第一种是访问 cluster-scoped resource，例如 Node、Namespace、PersistentVolume：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

第二种是定义可复用的 namespaced role template：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: application-viewer
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log", "services"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch"]
```

之后可以在多个 namespace 中分别创建 RoleBinding，复用同一份角色定义。

## RoleBinding 与 ClusterRoleBinding：把权限交给谁

RoleBinding 支持三类 subject：

| Subject | 典型用途 |
|---|---|
| User | 单个人类身份，通常来自 X.509 或 OIDC |
| Group | 企业团队或岗位，通常来自 OIDC group claim |
| ServiceAccount | Pod、controller、CI/CD 等非人身份 |

生产环境的人类权限应优先绑定 Group，而不是逐个绑定 User。这样人员入职、转组和离职由 Identity Provider 管理，不需要不断修改 Kubernetes RBAC。

下面示例中的 `oidc:` 假设 API Server 配置了对应的 group prefix；实际名称必须与 OIDC authenticator 最终写入请求的 group 字符串一致，prefix 本身不是 Kubernetes 的固定格式。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: payments-viewers
  namespace: payments
subjects:
- kind: Group
  name: oidc:team-payments
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

`roleRef` 创建后不能修改。如果要换角色，应删除并重建 RoleBinding，或使用 `kubectl auth reconcile` 管理声明式 RBAC。

## 一条 rule 是怎样匹配请求的

### apiGroups

Pod、Service、Secret 等核心资源属于 core API group，在 RBAC 中写成空字符串：

```yaml
apiGroups: [""]
```

Deployment 属于 `apps`，Job 属于 `batch`，Role 和 RoleBinding 属于 `rbac.authorization.k8s.io`：

```yaml
apiGroups: ["apps"]
resources: ["deployments"]
```

可以用 `kubectl api-resources` 查询资源的 group、是否 namespaced 以及资源名：

```bash
kubectl api-resources
kubectl api-resources --api-group=apps
```

### resources 与 subresources

RBAC 使用 API resource 的复数名，不使用 Kind：

```text
Kind: Deployment → resource: deployments
Kind: Pod        → resource: pods
```

subresource 必须单独授权：

```yaml
rules:
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
- apiGroups: [""]
  resources: ["pods/exec", "pods/attach", "pods/portforward"]
  verbs: ["create"]
```

拥有 `get pods` 不代表能看日志，拥有 `update pods` 也不代表能执行 `kubectl exec`。

### verbs

资源请求常见 verb 如下：

| HTTP 行为 | Kubernetes verb |
|---|---|
| 读取单个对象 | `get` |
| 读取对象集合 | `list` |
| 持续监听变化 | `watch` |
| 创建对象 | `create` |
| 完整替换对象 | `update` |
| 部分修改对象 | `patch` |
| 删除单个对象 | `delete` |
| 批量删除 | `deletecollection` |

`get`、`list` 和 `watch` 都可能返回完整对象。特别是 Secret，`list secrets` 并不是只看名称，而是可能拿到所有 Secret data。

### resourceNames

`resourceNames` 可以把部分权限限制到指定对象：

```yaml
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["application-config"]
  verbs: ["get", "update", "patch"]
```

但它不能限制 `create`，因为创建请求发生时对象尚未存在；也不能限制 `deletecollection`。受限的 `list/watch` 还要求客户端使用匹配的 field selector，因此它不是通用的行级权限方案。

## 实战：给应用团队最小发布权限

### 场景与目标

假设 `oidc:team-payments` 负责 `payments` namespace，团队需要：

1. 查看 Deployment、ReplicaSet、Pod、Event 和 Pod logs。
2. 创建与更新 Deployment。
3. 可以执行 `kubectl scale`。
4. 不能读取 Secret。
5. 不能直接创建 Pod。
6. 不能 `exec` 进入容器，也不能删除 namespace。

这里选择 `ClusterRole + RoleBinding`：ClusterRole 作为平台标准模板，RoleBinding 将作用域限制在 `payments`。

先准备实验 namespace：

```bash
kubectl create namespace payments \
  --dry-run=client \
  -o yaml | kubectl apply -f -
```

### 第一步：创建角色模板

保存为 `payments-rbac.yaml`：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: application-deployer
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: ["apps"]
  resources: ["deployments/scale"]
  verbs: ["get", "update", "patch"]
- apiGroups: ["apps"]
  resources: ["replicasets"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods", "services", "events"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: application-deployers
  namespace: payments
subjects:
- kind: Group
  name: oidc:team-payments
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: application-deployer
  apiGroup: rbac.authorization.k8s.io
```

应用前先让 API Server 做校验：

```bash
kubectl apply --server-side --dry-run=server -f payments-rbac.yaml
kubectl apply -f payments-rbac.yaml
```

### 第二步：验证授权结果

管理员可以通过 impersonation 验证 group 的有效权限：

```bash
kubectl auth can-i create deployments \
  -n payments \
  --as alice \
  --as-group oidc:team-payments

kubectl auth can-i patch deployments \
  --subresource=scale \
  -n payments \
  --as alice \
  --as-group oidc:team-payments

kubectl auth can-i get pods \
  --subresource=log \
  -n payments \
  --as alice \
  --as-group oidc:team-payments

kubectl auth can-i get secrets \
  -n payments \
  --as alice \
  --as-group oidc:team-payments

kubectl auth can-i create pods \
  -n payments \
  --as alice \
  --as-group oidc:team-payments
```

预期结果：

```text
create deployments       yes
patch deployments/scale  yes
get pods/log             yes
get secrets              no
create pods              no
```

执行 `--as` 和 `--as-group` 的管理员自身需要相应 impersonation 权限。普通团队成员验证自己时不需要这两个参数：

```bash
kubectl auth can-i --list -n payments
```

### 第三步：验证 namespace 边界

同一个 group 在 `orders` namespace 不应获得权限：

```bash
kubectl auth can-i create deployments \
  -n orders \
  --as alice \
  --as-group oidc:team-payments
```

预期为 `no`。如果返回 `yes`，检查是否存在额外的 ClusterRoleBinding 或其他 RoleBinding：

```bash
kubectl get rolebindings -A
kubectl get clusterrolebindings
```

## 默认角色应该怎样使用

Kubernetes 提供 `view`、`edit`、`admin` 和 `cluster-admin` 等默认 ClusterRole：

| 角色 | 典型能力 | 注意事项 |
|---|---|---|
| `view` | 读取大多数 namespaced resource | 默认不读取 Secret，也不能查看 Role/RoleBinding |
| `edit` | 读写大多数 namespaced resource | 可访问 Secret、创建 workload，通常能借用 namespace 内 ServiceAccount |
| `admin` | 管理 namespace 内大部分资源和 RBAC | 不等于 cluster admin，但已是高权限 |
| `cluster-admin` | 完整集群控制 | 只用于严格控制的管理员或 break-glass |

默认角色适合快速授权，但生产平台通常需要基于职责建立更窄的自定义 ClusterRole。尤其不要因为“开发需要发布”就直接绑定 `edit`。

## 常见错误

### 把 ClusterRole 当成必然全局

权限范围由 binding 决定。可复用的 ClusterRole 通过 RoleBinding 绑定，仍然只在单个 namespace 生效。

### 使用通配符

```yaml
apiGroups: ["*"]
resources: ["*"]
verbs: ["*"]
```

这不只覆盖当前资源，也覆盖未来新增的 CRD、subresource 和 verb。平台升级或安装 Operator 后，旧角色可能自动获得新权限。

### 忽略 subresource

`pods`、`pods/log`、`pods/exec` 是不同授权目标。`pods/exec` 能进入容器，不能混进普通只读角色。

### 只检查 Role，不检查 binding

Role 只是权限定义，真正的有效权限来自所有 binding 的并集。排障时要同时检查 RoleBinding、ClusterRoleBinding 和 subject 的 group。

### 认为 RBAC 可以 deny

RBAC 只能增加 allow。想禁止特定字段、镜像或 securityContext，应使用 Admission，而不是继续叠加 Role。

## 总结

Kubernetes RBAC 可以压缩为四个问题：

```text
谁：User、Group、ServiceAccount
在哪里：namespace 或 cluster
对什么：API group、resource、subresource
做什么：verb
```

Role/ClusterRole 定义权限，RoleBinding/ClusterRoleBinding 分配权限。生产环境优先使用 Group、namespaced RoleBinding、显式 resource 和显式 verb，并始终用 `kubectl auth can-i` 验证最终结果。

下一篇将进入 workload identity：ServiceAccount 如何成为 Pod 身份，token 为什么会自动挂载，以及怎样实现安全的跨 namespace 授权。

## 系列文章

1. Kubernetes RBAC 基础与实战：四个对象讲清最小权限（本文）
2. [Kubernetes ServiceAccount 实战：Pod 身份、Token 与跨 Namespace 授权](/posts/2026-07-29-kubernetes-serviceaccount-practice/)
3. [为什么 Kubernetes 只有 RBAC 不够：从隐式提权到 Admission Policy](/posts/2026-07-29-why-kubernetes-rbac-is-not-enough/)
4. [Kubernetes 权限体系进阶：完整 Access Control 架构](/posts/2026-07-29-kubernetes-access-control-rbac-admission-policy/)

## 参考资料

- [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Authorization Overview](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
- [kubectl auth can-i](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_auth/kubectl_auth_can-i/)
- [RBAC Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)

---

*本文基于 Kubernetes v1.36 编写。*
