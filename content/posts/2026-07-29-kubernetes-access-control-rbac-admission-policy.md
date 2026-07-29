---
title: "Kubernetes 权限体系全景：从 RBAC 到 Admission Policy"
date: 2026-07-29T10:00:00+08:00
draft: false
tags: ["Kubernetes", "RBAC", "Admission Control", "Security", "Zero Trust"]
categories: ["云原生", "DevOps"]
author: "Kaka"
description: "系统拆解 Kubernetes 从身份认证、授权到准入控制的完整权限链路，解释 RBAC、Node、Webhook、Pod Security 与 Admission Policy 分别解决什么问题，以及生产环境如何组合。"
---

## 引言

谈 Kubernetes 权限，很多人第一反应是 RBAC。但 RBAC 只回答一个问题：**某个身份能否对某类 API 资源执行某个动作**。它不负责确认身份真假，也不能判断一个 Deployment 是否使用了特权容器、是否挂载了不该使用的 ServiceAccount，更不能限制读请求返回的对象字段。

真正完整的 Kubernetes API 访问控制，是一条连续的决策链：

```text
Client
  │
  ├─ TLS：是否连接到了可信的 API Server
  │
  ├─ Authentication：你是谁
  │
  ├─ Authorization：你能对什么执行什么动作
  │
  ├─ Mutating Admission：需要补充或修改哪些字段
  │
  ├─ Validating Admission：这个对象最终是否符合策略
  │
  └─ Persistence：写入 etcd
          │
          └─ Audit：记录谁在何时做了什么
```

本文以 Kubernetes v1.36 为基线，覆盖 Authentication、Authorization、RBAC、Node、Webhook、ServiceAccount、impersonation、Admission Controller、Pod Security Admission、ValidatingAdmissionPolicy、MutatingAdmissionPolicy、审计与常见提权路径，并给出一套可落地的组合方式。

## 先建立正确模型：权限不是一层，而是多层

| 层次 | 回答的问题 | 主要机制 | 解决的问题 |
|---|---|---|---|
| Authentication | 你是谁？ | X.509、OIDC、ServiceAccount JWT、Webhook | 把请求映射为 `user`、`group`、`uid`、`extra` |
| Authorization | 你能做什么？ | RBAC、Node、Webhook、ABAC | 基于身份、verb、resource、namespace 等属性作出 allow/deny |
| Admission | 你提交的对象是否允许？ | built-in controller、CEL policy、webhook | 对写请求的对象字段执行 mutation 与 validation |
| Runtime isolation | Pod 运行后能碰什么？ | SecurityContext、seccomp、AppArmor/SELinux、NetworkPolicy | 限制进程、内核、网络和节点访问 |
| Audit | 实际发生了什么？ | Audit Policy、审计后端 | 追踪访问、策略命中、提权和异常行为 |

最重要的边界是：**RBAC 面向 API request attributes，Admission 面向 object contents**。RBAC 可以允许某人创建 Deployment，却无法表达“只能使用指定镜像仓库、不能使用 `hostNetwork`、只能绑定某个 ServiceAccount”；这些必须交给 Admission。反过来，Admission 不处理 `get`、`list`、`watch` 等读取请求，因此不能替代 RBAC。

## Authentication：先把身份做对

Kubernetes 自身没有通用的 `User` API。人类用户通常由外部 Identity Provider 管理，API Server 只消费认证结果；ServiceAccount 则是 Kubernetes 中的 namespaced workload identity。

| 认证方式 | 适用对象 | 设计目的与建议 |
|---|---|---|
| OIDC | 人类用户、企业 SSO | 将企业 IdP 的用户和 group 映射到 Kubernetes，适合统一生命周期、MFA 和短会话 |
| X.509 client certificate | 控制面组件、节点、少量运维场景 | 认证简单可靠，但证书吊销困难；避免把长期管理员证书广泛分发 |
| ServiceAccount token | Pod、controller、CI/CD automation | 为非人身份提供 JWT；优先使用 TokenRequest / projected bound token，不使用长期静态 token |
| Authentication Webhook | 自定义或遗留 IAM | 将 token 校验委托给外部系统，需要考虑延迟、缓存和高可用 |
| Authenticating Proxy | 已有统一认证网关 | API Server 信任代理注入的用户 header，代理证书与 request header 配置必须严密保护 |
| Bootstrap Token | kubelet TLS bootstrap | 只用于节点引导，不应作为日常用户凭据 |
| Anonymous | 健康检查或特定公开端点 | 默认收紧；匿名身份是 `system:anonymous`，所属 group 为 `system:unauthenticated` |

生产环境的常见选择是：人走 OIDC，workload 使用独立 ServiceAccount，节点通过证书和 bootstrap 流程加入集群。Kubernetes v1.34 起 structured authentication configuration 已 stable，可以用配置文件集中声明 JWT authenticator 和 anonymous access，并支持动态重载。

### ServiceAccount 不是“给 Pod 加个名字”

ServiceAccount 是 workload 调用 Kubernetes API 时的身份。每个 namespace 都有 `default` ServiceAccount，Pod 未显式指定时会自动使用它；默认 ServiceAccount 通常没有业务权限，但 token 仍可能被自动挂载。

不访问 Kubernetes API 的 workload 应关闭 token 自动挂载：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payments-runtime
  namespace: payments
automountServiceAccountToken: false
```

需要访问 API 时，为每个组件创建专用 ServiceAccount，只授予所需权限，并使用短期、带 `audience`、可轮换的 bound token。不要让多个应用共享高权限 ServiceAccount，也不要为方便而给 `default` ServiceAccount 绑定权限。

## Authorization：认证通过以后，谁来作决定

API Server 支持多种 authorizer：

| 模式 | 解决的问题 | 适用场景 | 主要限制 |
|---|---|---|---|
| RBAC | 用角色表达 API 权限 | 绝大多数用户与 workload | 规则是 additive，只能 allow，不能显式 deny |
| Node | 限制 kubelet 只能访问本节点所需对象 | 节点身份 `system:node:<name>` | 专用 authorizer，通常与 NodeRestriction 配套 |
| Webhook | 把决策委托给外部 policy engine | 需要中心化、上下文感知或组织级规则 | 同步调用带来延迟、缓存和可用性风险 |
| ABAC | 用静态 policy file 按属性授权 | 历史集群兼容 | 修改策略需改文件并重启，难审计、难运营，不建议新建 |
| AlwaysAllow / AlwaysDeny | 无条件允许或拒绝 | 测试 | `AlwaysAllow` 等于绕过授权，禁止用于生产 |

多个 authorizer 按配置顺序检查；某个模块一旦返回 allow 或 deny，决策立即结束，全部返回 no opinion 才会默认拒绝。因此顺序具有安全语义，不能认为后面的模块会“再检查一次”。常见自建集群基线是：

```text
--authorization-mode=Node,RBAC
```

只有确实需要外部决策时才增加 Webhook。比如 `Node,RBAC,Webhook` 中，被 RBAC allow 的请求不会再到 Webhook，因此此处 Webhook 只是 fallback，不是二次 guardrail；若希望外部系统拥有 veto 能力，就必须把它放在 RBAC 前面，并严谨设计 deny、no opinion 与故障时的行为。不要加入 `AlwaysAllow`；由于 RBAC 没有 deny 规则，`AlwaysAllow,RBAC` 的实际效果仍接近全部放行。

## RBAC：Kubernetes 权限控制的主干

RBAC 由四种对象组成：

| 对象 | 作用域 | 作用 |
|---|---|---|
| Role | namespace | 定义某个 namespace 内的权限集合 |
| ClusterRole | cluster | 定义 cluster-scoped 权限，或作为可被多个 namespace 复用的角色模板 |
| RoleBinding | namespace | 将 Role 或 ClusterRole 授给本 namespace 内的 subject |
| ClusterRoleBinding | cluster | 将 ClusterRole 授予 subject，并在整个集群生效 |

一个容易忽略但非常实用的组合是：**用 ClusterRole 维护标准角色模板，再用每个 namespace 的 RoleBinding 落地授权**。这样既能集中维护规则，又不会把权限扩大到所有 namespace。

### 一条 RBAC rule 到底表达什么

资源请求由 `apiGroups`、`resources`、`verbs` 和可选的 `resourceNames` 构成：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: application-deployer
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

将它只绑定到 `payments` namespace：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: payments-deployers
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

这里有几个关键细节：

`get`、`list`、`watch` 都可能返回完整对象，不能把 `list secrets` 当成“只能看 Secret 名称”。`pods/log`、`pods/exec`、`pods/attach`、`pods/portforward` 是独立 subresource；日志通常使用 `get`，exec/attach/port-forward 通常需要 `create`。只写 `pods` 不等于自动获得这些权限。

`resourceNames` 可以限制对已命名对象的部分操作，但不能限制 `create` 或 `deletecollection`，而受限的 `list/watch` 请求还需要客户端携带匹配的 field selector。它不是通用的行级或字段级权限系统。

`nonResourceURLs` 用于 `/healthz`、`/version` 等非资源路径，只能写在 ClusterRole 中。通配符 `*` 会覆盖未来新增的 API、CRD、subresource 和 verb，生产角色应尽量显式枚举。

RBAC 规则是累加的，没有 deny。想表达“允许创建 Pod，但禁止 privileged、hostPath 和指定 ServiceAccount”时，正确方案不是继续堆 RBAC，而是用 RBAC 控制 API 范围，再由 Admission 验证对象内容。

### RBAC 自带的防提权保护

Kubernetes 默认阻止用户创建一个超出自身权限的 Role/ClusterRole，也阻止其绑定一个自身无权拥有的角色。以下特殊 verb 用于显式越过这些保护：

| 特殊权限 | 能力 | 风险 |
|---|---|---|
| `escalate` on roles/clusterroles | 创建或修改包含自身未拥有权限的角色 | 可构造更高权限角色 |
| `bind` on roles/clusterroles | 将自身无权拥有的角色绑定给 subject | 可直接绑定 `cluster-admin` |
| `impersonate` on users/groups/serviceaccounts/userextras | 以其他身份发起请求 | 获得被 impersonated 身份的权限 |
| `approve` on certificatesigningrequests | 批准 CSR | 配合 signer 权限可签出新的客户端身份 |

这些 verb 应视为管理员级能力。尤其不要把用户加入 `system:masters`：该 group 会绕过 RBAC 检查，不能通过删除 RoleBinding 来收回其超级权限。

### 看似普通、实际可以提权的权限

| 权限 | 隐含能力 |
|---|---|
| `get/list/watch secrets` | 读取 Secret 数据，三者在数据暴露层面近似等价 |
| 创建 Pod/Deployment/Job | 挂载本 namespace 的 Secret、ConfigMap、PVC，并使用任意 ServiceAccount |
| `create serviceaccounts/token` | 为现有 ServiceAccount 签发 token |
| `create persistentvolumes` | 构造 `hostPath` PV，间接接触节点文件系统 |
| `get nodes/proxy` | 访问 kubelet API，可执行或 attach 到节点上的 Pod，并可能绕过常规 Audit 与 Admission |
| 修改 admission policy/webhook | 读取、修改或放行所有匹配的写请求 |
| `patch namespaces` | 放宽 Pod Security label，或改变依赖 namespace label 的 NetworkPolicy |
| 创建并批准特定 CSR | 获得新的客户端证书身份 |

这也是为什么“只能发布应用”并不天然是低权限：只要用户能创建 workload，就可能借用同 namespace 内更强的 ServiceAccount。**namespace 内部是弱安全边界**，不同 trust level 或 tenant 应拆分 namespace，并用 Admission 限制可选择的 ServiceAccount 与 Pod securityContext。

## Node Authorization 与 NodeRestriction：专门约束 kubelet

kubelet 必须读取调度到本节点的 Pod，以及这些 Pod 所需的 Secret、ConfigMap、PVC 等对象。如果直接用普通 RBAC 给每个节点手工授权，规则会复杂且容易过量。

Node authorizer 根据 `system:nodes` group、`system:node:<nodeName>` 身份和 Pod-to-Node 关系动态计算权限，只让 kubelet访问本节点职责所需对象。NodeRestriction Admission Controller 则限制 kubelet能修改哪些 Node 与 Pod，并阻止它写入受保护的 node label。

两者解决的是不同方向：Node authorizer 限制“节点能调用哪些 API”，NodeRestriction 限制“节点提交的对象内容能改到哪里”。生产环境通常同时启用，不能二选一。kubelet 自身暴露的 HTTPS API 还应单独启用 authentication 与 Webhook authorization；API Server 的 RBAC 不会自动替 kubelet endpoint 完成加固。

## Admission Control：RBAC 放行后再检查对象

Admission Controller 位于 Authentication 和 Authorization 之后、对象持久化之前，只拦截创建、修改、删除和部分 connect 类请求；普通读取请求绕过 Admission。

处理分为两个阶段：

```text
已通过 Authorization 的写请求
        │
        ├─ Mutating：补默认值、注入字段、生成 patch
        │
        └─ Validating：基于最终对象执行不变量检查
                │
                ├─ allow → persist
                └─ deny  → 返回错误
```

与权限和隔离最相关的 built-in controllers 包括：

| Controller | 解决的问题 |
|---|---|
| NamespaceLifecycle | 防止在 terminating 或不存在的 namespace 中创建对象 |
| ServiceAccount | 自动选择 ServiceAccount、注入相关信息并验证引用 |
| NodeRestriction | 限制 kubelet 对 Node、Pod 与 label 的修改范围 |
| PodSecurity | 按 Pod Security Standards 检查 Pod |
| LimitRanger | 对单对象资源 request/limit 设置默认值并限制上下界 |
| ResourceQuota | 限制 namespace 的总资源和对象数量，降低资源滥用与 DoS 风险 |
| CertificateApproval / Signing / SubjectRestriction | 约束 CSR 批准、签发和证书 subject |
| MutatingAdmissionWebhook | 调用外部 mutating webhook |
| ValidatingAdmissionWebhook | 调用外部 validating webhook |
| ValidatingAdmissionPolicy | 在 API Server 内使用 CEL 验证对象 |
| MutatingAdmissionPolicy | 在 API Server 内使用 CEL 生成 mutation |

### Pod Security Admission：阻止 workload 变成节点权限

Pod Security Admission（PSA）从 Kubernetes v1.25 起 stable，通过 namespace label 应用三档 Pod Security Standards：

| Level | 目标 |
|---|---|
| `privileged` | 不设限制，适合少量可信系统组件 |
| `baseline` | 阻止已知提权方式，同时保持较高兼容性 |
| `restricted` | 强化 least privilege，要求更严格的 securityContext |

PSA 还提供三种模式：`enforce` 拒绝违规 Pod，`warn` 向客户端返回警告，`audit` 写入审计 annotation。一个稳妥的迁移方式是先 enforce `baseline`，同时对 `restricted` 开 warn/audit；清理存量 workload 后再切换为 enforce `restricted`：

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

版本固定为 `v1.36` 可以避免 cluster upgrade 时策略语义突然改变。`kube-system` 等 privileged namespace 如果必须豁免，应同时用严格 RBAC 限制谁能写入，避免“能在豁免 namespace 创建 Pod”变成节点级权限。

### ValidatingAdmissionPolicy：优先使用的声明式验证

ValidatingAdmissionPolicy 从 Kubernetes v1.30 起 stable。它在 API Server 进程内执行 CEL，不需要外部 HTTP 调用，适合表达 deterministic、低延迟的对象约束。Policy 定义逻辑，Binding 负责选择 namespace/resource、提供参数并指定 `Deny`、`Warn`、`Audit` 动作。

下面的策略阻止 tenant Deployment 使用高权限 ServiceAccount，只允许未指定或使用自己的 runtime identity：

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: tenant-serviceaccount-guard.example.com
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
    message: "tenant Deployment may only use payments-runtime"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: tenant-serviceaccount-guard
spec:
  policyName: tenant-serviceaccount-guard.example.com
  validationActions: [Deny]
  matchResources:
    namespaceSelector:
      matchLabels:
        access.company.io/tier: tenant
        access.company.io/tenant: payments
```

实际平台应将允许的 ServiceAccount 做成 parameter resource，而不是为每个 namespace 复制一份 policy。新规则上线时先用 `Warn` 和 `Audit` 观察，再切换 `Deny`。

### MutatingAdmissionPolicy：声明式默认值与注入

Kubernetes v1.36 中 MutatingAdmissionPolicy 已 stable。它同样使用 CEL，但输出 Server-Side Apply configuration 或 JSON Patch，用于注入 label、默认 securityContext、sidecar 等内容。它解决的是“平台要自动补什么”，不是“违规对象要不要拒绝”。

mutation 必须幂等，且应由 validating policy 验证最终对象仍满足不变量。多个 mutating component 没有可以依赖的稳定执行顺序；如果策略依赖其他 mutation 的结果，应谨慎使用 `reinvocationPolicy: IfNeeded`，并避免相互覆盖字段。

### Admission Webhook：只在需要外部能力时使用

Webhook 能执行任意代码，适合镜像签名验证、外部 CMDB 查询、供应链 attestations 或 CEL 难以表达的复杂逻辑，但它把 API write path 与外部服务的延迟和故障耦合起来。

生产设计至少要考虑：

1. 使用 `matchConditions`、`namespaceSelector` 和精确 resource rule 缩小匹配范围。
2. security invariant 通常使用 `failurePolicy: Fail`；选择 `Ignore` 就是在 webhook 故障时 fail-open。
3. 设置较短 `timeoutSeconds`、多副本、PDB、监控和可靠 TLS。
4. 设置正确的 `sideEffects: None` 或 `NoneOnDryRun`，支持 dry-run。
5. mutating webhook 必须幂等；mutation 完成后再由 ValidatingAdmissionPolicy 或 validating webhook 检查结果。
6. 简单、静态、可由 CEL 表达的规则优先使用 in-process Admission Policy，减少外部依赖。

Kubernetes v1.36 还引入了 alpha 的 Manifest-Based Admission Control：策略从 API Server 本地文件加载，可在 API 服务启动前生效，并能保护 API-based admission configuration 本身，填补 bootstrap gap 和“管理员删除策略”的 self-protection gap。它适合高度管控的自建控制面，但仍是 disabled-by-default alpha，不应直接作为通用生产基线。

## 一套可落地的组合架构

单独部署 RBAC、PSA 或 policy engine 都不完整。更稳妥的组合如下：

```text
Human ── OIDC + MFA ── group ── RoleBinding ─┐
                                             │
Workload ── bound SA token ── RBAC ──────────┤
                                             ▼
                                   Node + RBAC authorizers
                                             │
                           ┌─────────────────┴─────────────────┐
                           ▼                                   ▼
                Built-in Admission                    Custom Admission
          NodeRestriction / PSA / Quota       CEL Policy → Webhook
                           │                                   │
                           └─────────────────┬─────────────────┘
                                             ▼
                          Runtime isolation + NetworkPolicy
                                             │
                                             ▼
                                      Audit + alerting
```

### 人类用户

使用企业 OIDC 与 MFA，不分发长期 `cluster-admin` certificate。权限绑定到 group，而不是逐个绑定 user；用户离职或换组时由 IdP 回收。日常操作使用 namespaced RoleBinding，break-glass 身份单独保管、短时启用并强制审计。

### Workload

每个组件使用独立 ServiceAccount；不访问 API 的 Pod 设置 `automountServiceAccountToken: false`。需要云 API 时优先使用 cloud workload identity，不要把云密钥放进 Kubernetes Secret，也不要因此扩大 Kubernetes RBAC。

### Tenant 与 namespace

按 tenant、环境和 trust level 拆 namespace。RoleBinding 控制 API 范围，PSA 控制 Pod 提权面，ValidatingAdmissionPolicy 限制 ServiceAccount、镜像来源与关键字段，ResourceQuota/LimitRange 控制资源滥用，NetworkPolicy 控制 east-west traffic。高权限 controller 与不可信 workload 不应处于同一 namespace。

### Node 与控制面

启用 Node authorizer + NodeRestriction，保护 kubelet endpoint，限制控制面端口的网络可达性。Admission policy 和 webhook configuration 本身属于高权限资源，只允许平台管理员或受控 GitOps controller 修改。

## 权限验证、审计与持续治理

不要只看 YAML 推测权限，直接让 API Server 计算：

```bash
# 当前身份能否在 payments 创建 Deployment
kubectl auth can-i create deployments -n payments

# 查看当前身份在 namespace 内的有效权限
kubectl auth can-i --list -n payments

# 管理员验证某个 ServiceAccount；执行者自身需要 impersonate 权限
kubectl auth can-i list pods \
  -n payments \
  --as system:serviceaccount:payments:payments-runtime

# 服务端校验 RBAC 与 Admission，不真正写入对象
kubectl apply --server-side --dry-run=server -f access-control.yaml
```

`kubectl auth can-i` 使用 SelfSubjectAccessReview。需要让扩展 API Server 或外部服务复用 Kubernetes 的授权结论时，可使用 SubjectAccessReview；不要在应用里复制一份 RBAC 解释器。

持续治理至少包括：

1. 定期扫描 wildcard、ClusterRoleBinding、`system:masters`、Secret read、workload create、`bind`、`escalate`、`impersonate`、CSR approval、`nodes/proxy` 和 admission configuration 权限。
2. 监控 RoleBinding、ClusterRoleBinding、ServiceAccount token、CSR、namespace security label、admission policy/webhook 的变更。
3. Audit Policy 记录高风险资源的 request/response metadata，并对 forbidden spike、impersonation、策略 fail-open 和 break-glass 行为告警。
4. 用 GitOps 管理角色与策略，变更先经过 server-side dry-run、测试集群和 `Warn/Audit` 阶段。
5. 清理离职用户、废弃 ServiceAccount 和孤立 binding；RBAC 绑定的是身份字符串，旧用户名被重新使用时可能继承历史权限。

## 常见错误

**把 RBAC 当字段级策略。** RBAC 不能按 image、label、securityContext 或 Secret 字段授权，使用 Admission。

**认为只读就是安全。** `list/watch secrets` 会返回数据，`get nodes/proxy` 甚至不是只读能力。

**给用户 Deployment create，却忽略 ServiceAccount。** workload creation 可能继承 namespace 内任意 ServiceAccount 的权限，必须做 namespace 隔离与 Admission 限制。

**用 ClusterRoleBinding 图省事。** ClusterRole 不等于必须 cluster-wide；优先用 RoleBinding 把它限制到单个 namespace。

**只上 PSA，不做 RBAC。** PSA 只检查 Pod security fields，不阻止读取 Secret、修改 RBAC 或删除 policy。

**所有策略都放 webhook。** 这会把 API Server 可用性绑定到网络服务；简单规则优先 CEL，外部上下文才用 webhook。

**用 Audit 代替 Enforcement。** Audit 只能提供证据，不能阻止请求；正确模式是先用 Audit/Warn 发现影响，再切换 Deny/Enforce。

## 总结

Kubernetes 权限设计的核心不是写出更多 Role，而是把不同问题交给正确的控制层：

Authentication 建立可信身份；RBAC 负责大多数 user、group 与 ServiceAccount 的 API 权限；Node authorizer + NodeRestriction 专门约束 kubelet；Admission 根据对象内容补默认值并执行安全不变量；PSA 提供 Pod 安全基线；CEL Admission Policy 承担大多数声明式平台规则；Webhook 只处理需要外部上下文的复杂策略；Runtime controls 限制已经启动的 workload；Audit 提供持续验证与追责。

最实用的生产原则可以压缩成一句话：**OIDC/ServiceAccount 定身份，namespaced RBAC 给最小 API 权限，PSA + CEL Policy 限制对象内容，Webhook 补外部判断，NetworkPolicy 和 runtime security 收紧运行面，Audit 验证整条链路。**

## 参考资料

- [Kubernetes API Access Control](https://kubernetes.io/docs/reference/access-authn-authz/)
- [Kubernetes Authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [Kubernetes Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
- [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [RBAC Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)
- [Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/)
- [Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [Validating Admission Policy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)
- [Mutating Admission Policy](https://kubernetes.io/docs/reference/access-authn-authz/mutating-admission-policy/)
- [Manifest-Based Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/manifest-admission-control/)
- [Kubernetes Auditing](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/)

---

*本文基于 Kubernetes v1.36 编写。使用其他 minor version 时，请核对对应版本的 feature state、API version 与默认启用的 Admission Controller。*
