---
title: "为什么 Kubernetes 只有 RBAC 不够：从隐式提权到 Admission Policy"
date: 2026-07-29T09:00:00+08:00
draft: false
tags: ["Kubernetes", "RBAC", "Admission Policy", "Pod Security", "Security"]
categories: ["云原生", "DevOps"]
author: "Kaka"
description: "通过一个 Deployment 借用高权限 ServiceAccount 的实战场景，解释 RBAC 为什么无法约束对象字段，以及如何组合 Pod Security Admission 与 ValidatingAdmissionPolicy。"
---

## 引言

假设开发团队只有以下 RBAC 权限：

```yaml
apiGroups: ["apps"]
resources: ["deployments"]
verbs: ["get", "list", "watch", "create", "update", "patch"]
```

他们不能读取 Secret、不能修改 RBAC，也不能直接创建 Pod。这个权限看起来很安全，但只要同一个 namespace 中存在高权限 ServiceAccount，团队就可能创建一个 Deployment，让 Pod 使用这个 ServiceAccount。

问题的根源是：

```text
RBAC 只判断“能否 create deployments”
RBAC 不检查 Deployment spec 里面写了什么
```

这不是 RBAC 的漏洞，而是职责边界。RBAC 负责 API request authorization；对象字段、securityContext、image 和 ServiceAccount 选择必须由 Admission 处理。

> 本文是 Kubernetes 权限系列第 3/4 篇。前两篇已经建立 RBAC 与 ServiceAccount 模型，本文专门处理它们组合后的安全缺口。

## RBAC 能看见什么

一个 API request 到达 RBAC 时，authorizer 主要看到：

```text
user / group / extra
namespace
apiGroup
resource / subresource
verb
resourceName
nonResourceURL
```

它可以判断：

```text
alice 能否在 payments 创建 Deployment？
```

但无法判断：

```text
Deployment 使用哪个 ServiceAccount？
容器是否 privileged？
是否启用 hostNetwork 或 hostPID？
是否挂载 hostPath？
镜像是否来自可信 registry？
是否缺少 required label？
replicas 是否超过平台限制？
```

这些字段只有在 Admission 阶段检查完整 object 才有意义。

## 实战风险：Deployment 借用高权限 ServiceAccount

### 场景

`payments` namespace 中有两个身份：

```text
payments-runtime   普通业务应用使用，无 Kubernetes API 权限
backup-controller  备份组件使用，可以读取 Secret
```

平台给开发团队授予 Deployment create/update 权限，但没有限制 `.spec.template.spec.serviceAccountName`。

一个通过 RBAC 检查的 Deployment 可能是：

保存为 `unexpected-workload.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: unexpected-workload
  namespace: payments
spec:
  replicas: 1
  selector:
    matchLabels:
      app: unexpected-workload
  template:
    metadata:
      labels:
        app: unexpected-workload
    spec:
      serviceAccountName: backup-controller
      containers:
      - name: app
        image: registry.k8s.io/pause:3.10
```

RBAC 只检查创建 Deployment 的权限，因此请求会被允许。ReplicaSet controller 随后创建 Pod，Pod 自动获得 `backup-controller` 的 token。真实攻击中，用户只需换成自己控制的镜像，就可以使用该身份调用 API。

这里发生了权限传递：

```text
create Deployment
      │
      ▼
选择任意 ServiceAccount
      │
      ▼
Pod 获得该 ServiceAccount token
      │
      ▼
继承 ServiceAccount 的 API 权限
```

同样地，workload creation 还可能隐含：

| 能力 | 原因 |
|---|---|
| 读取 Secret / ConfigMap | Pod 可以把 namespace 内对象挂载为 volume 或 env |
| 使用 PVC | Pod 可以挂载已有存储 |
| 访问节点 | privileged、hostPath、hostPID 等配置可能突破容器边界 |
| 访问云 metadata | Pod 网络路径可能接触 node 或 cloud instance identity |
| 借用其他 workload identity | Pod 可以选择 namespace 内其他 ServiceAccount |

因此 Kubernetes 官方 RBAC good practices 明确把 workload creation 视为 privilege escalation risk。namespace 内部不是强 tenant boundary。

## 为什么不能继续用 RBAC 修补

可以把用户的 RBAC 缩小为“只能创建 Deployment，不能创建 Pod”，但 Deployment controller 最终仍会创建 Pod。也可以不给用户 Secret read，但 kubelet 仍可以为被调度的 Pod 挂载它引用的 Secret。

RBAC 的正确职责是控制 API surface：

```text
允许哪些身份访问哪些 API resource
```

Admission 的正确职责是控制 object invariants：

```text
允许写入的对象最终必须满足哪些条件
```

两者应该组合，而不是互相替代。

## Admission 在请求链路中的位置

```text
Authentication
      │
Authorization / RBAC
      │
      ├─ deny  → 403 Forbidden
      │
      └─ allow
           │
           ▼
Mutating Admission
           │
           ▼
Validating Admission
      │               │
    allow            deny
      │               │
   写入 etcd       返回策略错误
```

Admission 只处理 create、update、patch、delete 和部分 connect 类请求。`get`、`list`、`watch` 不经过 Admission，因此 Admission Policy 不能阻止一个已经拥有 Secret read 权限的身份读取数据。

## 第一层：用 namespace 建立 trust boundary

如果用户可以在一个 namespace 创建 workload，就应假设其可能接触该 namespace 中大部分 workload-level 资源。不同 tenant、环境和 trust level 应拆 namespace：

```text
payments-dev
payments-prod
platform-system
security-controllers
```

高权限 controller 不应和普通业务 workload 放在同一 namespace。namespace 分离仍然需要 RBAC：开发团队不能写入 `platform-system`，业务 Pod 的 ServiceAccount 也不能跨 namespace 获得多余权限。

## 第二层：Pod Security Admission 限制运行权限

Pod Security Admission（PSA）是 Kubernetes 内置的 validating admission controller，从 v1.25 起 stable。它按 namespace 应用 Pod Security Standards：

| Level | 目标 |
|---|---|
| `privileged` | 不施加限制，适合少量可信系统组件 |
| `baseline` | 阻止已知 privilege escalation，同时保持较高兼容性 |
| `restricted` | 强化 least privilege，要求更严格的 securityContext |

PSA 支持三种模式：

| Mode | 行为 |
|---|---|
| `enforce` | 拒绝违规 Pod |
| `warn` | 请求继续，但向客户端返回 warning |
| `audit` | 请求继续，但在 Audit event 中增加 annotation |

### 实战：给 payments 增加安全基线

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments
  labels:
    access.company.io/tier: tenant
    access.company.io/tenant: payments
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: v1.36
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.36
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.36
```

这个配置先强制 `baseline`，同时观察迁移到 `restricted` 会影响哪些 workload。固定 `v1.36` 可以避免集群升级时策略语义静默变化。

用一个 privileged Pod 验证：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-test
  namespace: payments
spec:
  containers:
  - name: test
    image: registry.k8s.io/pause:3.10
    securityContext:
      privileged: true
```

执行 server-side dry-run：

```bash
kubectl apply --server-side --dry-run=server -f privileged-test.yaml
```

预期被 Pod Security `baseline` 拒绝。

需要注意：PSA 的 `enforce` 针对实际 Pod。创建 Deployment 等 workload object 时，`warn` 和 `audit` 会检查 Pod template 并提前提示，但 `enforce` 最终发生在 controller 创建 Pod 时。因此一个违规 Deployment 可能先创建成功，随后因为 Pod 被拒绝而一直没有可用副本。需要在 Deployment 层立即失败时，可以补充 ValidatingAdmissionPolicy。

### PSA 解决不了什么

PSA 只检查 Pod Security Standards 定义的字段。它不负责：

1. 限制 Deployment 可以选择哪些 ServiceAccount。
2. 强制镜像来自企业 registry。
3. 要求 owner、cost-center 等组织 label。
4. 限制业务字段或 CRD spec。
5. 阻止 Secret volume 的业务级越权引用。

这些规则需要通用 Admission Policy。

## 第三层：用 ValidatingAdmissionPolicy 限制对象内容

ValidatingAdmissionPolicy（VAP）从 Kubernetes v1.30 起 stable。它在 API Server 内使用 CEL 执行验证，不依赖外部 webhook，适合 deterministic、低延迟的规则。

一条策略通常由两部分组成：

```text
ValidatingAdmissionPolicy
  定义匹配哪些资源、执行哪些 CEL validation

ValidatingAdmissionPolicyBinding
  决定策略应用到哪里，以及使用 Deny / Warn / Audit
```

### 实战：限制 Deployment 可使用的 ServiceAccount

保存为 `payments-serviceaccount-policy.yaml`：

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: payments-serviceaccount-guard.example.com
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: ["apps"]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["deployments"]
  validations:
  - expression: >-
      !has(object.spec.template.spec.serviceAccountName) ||
      object.spec.template.spec.serviceAccountName == "payments-runtime"
    message: >-
      payments Deployment may only use the default identity
      or payments-runtime
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: payments-serviceaccount-guard
spec:
  policyName: payments-serviceaccount-guard.example.com
  validationActions: [Warn, Audit]
  matchResources:
    namespaceSelector:
      matchLabels:
        access.company.io/tier: tenant
        access.company.io/tenant: payments
```

这里先使用 `Warn` 和 `Audit`，不立即阻断存量发布：

```bash
kubectl apply --server-side --dry-run=server \
  -f payments-serviceaccount-policy.yaml

kubectl apply -f payments-serviceaccount-policy.yaml
```

然后对前面的危险 Deployment 做 dry-run：

```bash
kubectl apply --server-side --dry-run=server \
  -f unexpected-workload.yaml
```

客户端会收到 warning，Audit event 也会记录 validation failure。确认合法 workload 都已改用 `payments-runtime` 后，将 binding 切换为 `Deny`：

```bash
kubectl patch validatingadmissionpolicybinding \
  payments-serviceaccount-guard \
  --type=merge \
  -p '{"spec":{"validationActions":["Deny"]}}'
```

再次提交相同 Deployment，请求会在写入 etcd 前被拒绝。

示例只匹配 Deployment，因为前面的开发角色也只允许创建 Deployment。如果用户还能创建 StatefulSet、DaemonSet、Job、CronJob 或直接创建 Pod，必须为这些 resource 增加对应规则；否则未匹配的 workload 会成为绕过路径。

### 为什么 Policy 与 Binding 要分开

Policy 可以作为组织级模板，例如“workload 只能使用允许的 ServiceAccount”；Binding 决定应用到哪些 namespace，并可引用 parameter resource 提供每个 tenant 的 allowlist。

上面的示例为便于理解直接写死 `payments-runtime`。规模化平台应通过 `paramKind` 读取 CRD 或 ConfigMap 参数，复用一份 policy，而不是为每个 namespace 复制 CEL。

## 第四层：什么时候需要 Admission Webhook

CEL Policy 适合只依赖请求对象、旧对象、namespace、参数和 authorizer 信息的规则。以下场景通常需要 webhook：

1. 向外部 image registry 查询签名或 attestations。
2. 查询 CMDB、ticket system 或组织级 policy service。
3. 执行 CEL 难以表达的复杂逻辑。
4. 与已有 OPA、Kyverno、Gatekeeper 等策略平台集成。

Webhook 能力更强，但位于 API write path：

| 设计项 | 风险与建议 |
|---|---|
| `failurePolicy: Fail` | 安全规则不会在 webhook 故障时绕过，但会影响 API 可用性 |
| `failurePolicy: Ignore` | webhook 故障时 fail-open，不能用于关键 security invariant |
| `timeoutSeconds` | 过长会放大 API latency，应该短且可观测 |
| `namespaceSelector` / `matchConditions` | 精确缩小调用范围，减少故障半径 |
| 多副本与 PDB | 防止单点故障阻塞集群写请求 |
| mutation 幂等性 | 防止 reinvocation 或重试产生重复注入 |

简单规则优先 ValidatingAdmissionPolicy；只有真正依赖外部信息时才引入 webhook。

## 把三层控制组合起来

一个 tenant namespace 的权限设计可以表达为：

```text
RBAC
  允许 team-payments 创建和更新 Deployment
  不允许读取 Secret、修改 RBAC、创建 Pod

ServiceAccount
  payments-runtime 只拥有应用运行所需权限
  高权限 controller 放到独立 namespace

Pod Security Admission
  enforce baseline
  warn + audit restricted

ValidatingAdmissionPolicy
  只允许使用 payments-runtime
  限制镜像 registry、required labels、host options

ResourceQuota / LimitRange
  限制对象数量和资源消耗

NetworkPolicy / Runtime Security
  限制 Pod 启动后的网络和进程能力

Audit
  记录策略命中、拒绝和高风险资源变更
```

各层解决不同问题：

| 控制 | 核心职责 |
|---|---|
| RBAC | 谁能调用哪些 API |
| ServiceAccount | Pod 使用哪个 API identity |
| PSA | Pod 是否符合通用安全基线 |
| VAP / Webhook | 对象是否满足平台和业务规则 |
| NetworkPolicy / runtime controls | workload 启动后能访问什么 |
| Audit | 发生过什么 |

## 策略本身也需要保护

Admission Policy、Binding、WebhookConfiguration 和 namespace label 都是高权限配置。如果业务用户能修改它们，就可以先放宽策略再提交 workload。

至少限制以下资源：

```text
validatingadmissionpolicies
validatingadmissionpolicybindings
mutatingadmissionpolicies
mutatingadmissionpolicybindings
validatingwebhookconfigurations
mutatingwebhookconfigurations
namespaces
```

这些权限只授给平台管理员或受控 GitOps controller，并对变更启用 Audit alert。Kubernetes v1.36 的 Manifest-Based Admission Control 可以进一步解决 API-based policy 的 bootstrap 和 self-protection gap，但目前仍是 alpha，不应作为普通集群的默认基线。

## 常见错误

### 认为没有 Secret read 就读不到 Secret

能创建 workload 的用户可能把 Secret 挂载进 Pod，再由容器读取。

### 认为不能创建 Pod 就安全

Deployment、Job、DaemonSet 等 controller 最终都会创建 Pod，并且 Pod 可以选择 ServiceAccount。

### 只启用 PSA

PSA 不限制 ServiceAccount、镜像来源、组织 label 或任意 CRD 字段。

### 直接把新策略设为 Deny

先用 `Warn + Audit` 观察实际影响，修复存量对象和自动化，再切换 `Deny`。关键策略还要准备 break-glass 与回滚路径。

### 使用 namespaceSelector 却允许业务修改 namespace

如果用户可以删除或修改用于匹配的 label，就能让 namespace 脱离策略范围。selector label 本身也必须由 RBAC 保护。

### 用 Admission 代替 RBAC

读取请求绕过 Admission。Secret、ConfigMap 和日志等 read permission 仍必须由 RBAC 严格控制。

## 总结

只有 RBAC 不够，不是因为 RBAC 太弱，而是因为它只负责授权 API request：

```text
RBAC：你能不能创建 Deployment
Admission：这个 Deployment 里面允许写什么
```

workload creation 会带来 ServiceAccount 借权、Secret 挂载和 Pod privilege escalation 等隐式能力。正确方案是先按 trust level 拆 namespace，用最小 RBAC 控制 API surface，再用 PSA 建立通用 Pod 基线，用 ValidatingAdmissionPolicy 约束 ServiceAccount 和业务字段，外部上下文才交给 webhook。

下一篇进阶总览会把 Authentication、Node Authorization、RBAC、Admission、runtime isolation 和 Audit 串成完整的 Kubernetes Access Control 架构。

## 系列文章

1. [Kubernetes RBAC 基础与实战：四个对象讲清最小权限](/posts/2026-07-29-kubernetes-rbac-basics-practice/)
2. [Kubernetes ServiceAccount 实战：Pod 身份、Token 与跨 Namespace 授权](/posts/2026-07-29-kubernetes-serviceaccount-practice/)
3. 为什么 Kubernetes 只有 RBAC 不够：从隐式提权到 Admission Policy（本文）
4. [Kubernetes 权限体系进阶：完整 Access Control 架构](/posts/2026-07-29-kubernetes-access-control-rbac-admission-policy/)

## 参考资料

- [RBAC Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)
- [Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Validating Admission Policy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)
- [Admission Webhook Good Practices](https://kubernetes.io/docs/concepts/cluster-administration/admission-webhooks-good-practices/)

---

*本文基于 Kubernetes v1.36 编写。*
