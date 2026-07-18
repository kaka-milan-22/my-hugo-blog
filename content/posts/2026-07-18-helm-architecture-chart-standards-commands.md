---
title: "Helm 实战指南：架构、Chart 规范与 Kubernetes 运维常用命令"
date: 2026-07-18T08:00:00+08:00
draft: false
tags: ["Kubernetes", "Helm", "DevOps", "Cloud Native", "Operations"]
categories: ["云原生", "DevOps"]
author: "Kaka"
description: "从 Kubernetes 原生 YAML 的工程化痛点出发，系统讲清 Helm 的设计动机、运行架构、Chart 规范，以及运维工程师高频使用的安装、升级、排障和回滚命令。"
---

## 引言

Kubernetes 提供了强大的声明式 API，但它没有直接解决应用的打包、参数化、版本管理和跨环境复用问题。一个生产应用往往包含 Deployment、Service、ConfigMap、Ingress、ServiceAccount、RBAC、HPA 等多个资源；当环境从一套变成十套、应用版本不断迭代时，复制 YAML 和手工执行 `kubectl apply` 很快就会失控。

Helm 的定位不是替代 Kubernetes，而是在 Kubernetes API 之上提供应用级的 package management。它把一组相关资源封装为可版本化的 **chart**，将 chart 与配置合并后安装为可追踪、可升级、可回滚的 **release**。对运维工程师来说，Helm 最重要的价值不是“少写几份 YAML”，而是建立稳定的应用交付边界。

## 为什么 Kubernetes 需要 Helm

### 原生 YAML 的四个工程化缺口

直接维护 Kubernetes manifests 在小规模场景中很清晰，但规模扩大后通常会遇到四类问题。

第一是重复。开发、测试、预发布和生产环境的 manifests 大部分相同，只有镜像、域名、副本数和资源配额不同。复制目录会带来配置漂移，批量替换又容易误改。

第二是缺少应用边界。`kubectl` 面向单个或一组 Kubernetes resources，却不知道哪些对象共同组成某个应用版本，也不原生维护应用级 revision history。

第三是依赖与分发。一个应用可能依赖 Redis、PostgreSQL 或 ingress controller，需要统一描述依赖、版本和默认配置，并通过 repository 或 OCI registry 分发。

第四是生命周期。生产发布需要 dry-run、差异检查、原子升级、等待资源就绪、失败回滚和历史审计，仅靠散落的 YAML 与 shell scripts 很难形成一致接口。

Helm 正是为这些问题而生。它最初由 Deis 团队在 2015 年创建，源于 Deis 从 Fleet 迁移到 Kubernetes 后重写安装工具的需求，设计思路借鉴 Homebrew、apt 和 yum。2016 年 Helm Classic 与 Google 的 Kubernetes Deployment Manager 合并；Helm 3 在 2019 年移除集群内的 Tiller，重新成为直接访问 Kubernetes API 的 client-side 工具；Helm 4 则在 2025 年发布，引入 Server-Side Apply、改进的资源状态监控和新的 plugin 架构。这个演进过程始终围绕同一个目标：让 Kubernetes 应用可以被打包、共享、安装和持续管理。

## Helm 的核心模型与架构

### Chart、Values、Release 与 Repository

理解 Helm，先区分四个对象：

| 对象 | 含义 | 运维视角 |
|---|---|---|
| Chart | Kubernetes 应用包，包含 metadata、默认 values 和 templates | 类似 RPM、Deb 或 Homebrew formula |
| Values | Chart 的输入参数 | 环境差异和发布配置 |
| Release | Chart 在某个 cluster/namespace 中的一次安装实例 | 有名称、状态和 revision history 的应用实例 |
| Repository / OCI Registry | Chart 的存储与分发位置 | 软件源或 artifact registry |

同一个 chart 可以安装多次，每次形成独立 release。例如用同一份 `redis` chart 创建 `redis-session` 和 `redis-cache`，它们可以使用不同 values、位于不同 namespace，并拥有各自的升级历史。

### 运行架构

Helm 4 的核心仍是 client-side 架构，不需要在 cluster 内长期运行一个 Helm server：

```text
Chart repository / OCI registry
              │ pull
              ▼
┌──────────────────────────────┐
│ Helm CLI / Helm Go library   │
│                              │
│ Chart + Values               │
│      │                       │
│      ▼                       │
│ Go template rendering       │
│      │                       │
│      ▼                       │
│ Kubernetes manifests        │
└──────────────┬───────────────┘
               │ Kubernetes REST API
               ▼
┌──────────────────────────────┐
│ Kubernetes API Server        │
│  ├─ creates/updates resources│
│  └─ stores release metadata  │
│     as namespace Secrets     │
└──────────────────────────────┘
```

`helm` 读取当前 kubeconfig 和 context，加载 chart，按优先级合并 values，渲染 Go templates，再通过 Kubernetes client library 调用 API Server。release metadata 默认保存在 release 所在 namespace 的 Secrets 中，因此 Helm 不需要独立数据库；但这也意味着拥有 Secret 读取权限的主体可能看到 chart、values 甚至其中的敏感配置，生产环境必须用 RBAC 与 etcd encryption 控制访问。

Values 的常见优先级从低到高如下：

```text
chart 的 values.yaml
  < parent chart values
  < helm install/upgrade -f values.yaml
  < --set / --set-string / --set-file
```

后提供的 values file 会覆盖前一个文件。生产发布应尽量将配置固化在版本库中的 values files，避免堆叠难以审计的 `--set`。

### 一次 upgrade 实际发生了什么

执行下面的命令时：

```bash
helm upgrade --install payment ./payment \
  --namespace payment-prod \
  --create-namespace \
  -f values-prod.yaml \
  --wait --timeout 10m \
  --rollback-on-failure
```

Helm 会查找 release；不存在就执行 install，存在则读取上一 revision。随后 Helm 合并 values、渲染 manifests、验证资源并提交给 API Server，等待 workloads 达到就绪状态，最后写入新的 release revision。若操作失败，`--rollback-on-failure` 会触发回滚。Helm 4 中该 flag 是原 `--atomic` 的新名称，旧名称仍兼容但已进入 deprecated 流程。

需要注意：Helm 4 对新 release 默认使用 Server-Side Apply；从 Helm 3 升级而来的 release 会延续原有 client-side apply 行为，以避免升级时突然改变 field ownership。多控制器共同管理同一资源时，应把 managed fields 冲突纳入发布验证。

## Helm Chart 的标准结构

运行 `helm create payment` 会生成一个基础 chart。典型目录如下：

```text
payment/
├── Chart.yaml              # Chart metadata，必需
├── Chart.lock              # dependency lock，执行 dependency update 后生成
├── values.yaml             # 默认配置，必需
├── values.schema.json      # Values 的 JSON Schema，推荐
├── charts/                 # 下载或内嵌的依赖 charts
├── crds/                   # CRDs，安装早于 templates
├── templates/
│   ├── _helpers.tpl        # 可复用的 named templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt           # 安装后的操作提示
│   └── tests/
│       └── test-connection.yaml
├── README.md
└── .helmignore
```

### Chart.yaml：区分 chart 版本与应用版本

```yaml
apiVersion: v2
name: payment
description: Payment service for Kubernetes
type: application
version: 1.4.2
appVersion: "2026.07.18"
kubeVersion: ">=1.31.0-0"
dependencies:
  - name: redis
    version: "~20.6.0"
    repository: "oci://registry.example.com/charts"
    condition: redis.enabled
```

`version` 是 chart 自身的 SemVer 版本，只要 template、default values 或依赖发生变化就应递增。`appVersion` 表示被部署应用的版本，只是信息字段，不参与 Helm 的版本计算。把两者混为一谈会导致 chart 无法独立演进，也会破坏 release 审计。

### values.yaml：设计稳定的配置接口

```yaml
replicaCount: 3

image:
  repository: registry.example.com/payment
  tag: "2026.07.18"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080

resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    memory: 512Mi
```

Values 是 chart 对使用者暴露的 API，应保持命名稳定、层级适中、类型明确。变量以 lowercase 开头并采用 camelCase；字符串建议加引号；容易被 CLI 覆盖的字段不要设计得过深。不要把 password、private key 或 token 直接写入 values 文件或用 `--set` 传入 shell history，应引用 External Secrets、SOPS 等受控的 secrets workflow。

### Templates：生成 YAML，不是隐藏业务逻辑

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "payment.fullname" . }}
  labels:
    {{- include "payment.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "payment.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "payment.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: payment
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

Template 应保持可读、可预测。把重复命名和 labels 抽到 `_helpers.tpl`，使用 `include` 配合 `nindent`；对必填配置使用 `required`，对可选值使用 `default`。不要为了“通用”而把所有 Kubernetes fields 都模板化，否则 values 会变成另一套更难维护的 Kubernetes API。

### Chart 规范与生产最佳实践

Chart name 应符合 DNS-1123 label：只使用 lowercase letters、数字和连字符，最长 63 字符；YAML 使用两个空格缩进。资源应统一使用 `app.kubernetes.io/name`、`app.kubernetes.io/instance`、`app.kubernetes.io/managed-by` 和 `helm.sh/chart` 等标准 labels，selector labels 必须稳定，不能包含随升级变化的 chart version。

不要在 templates 的 `metadata.namespace` 中硬编码 namespace，应由 `--namespace` 决定安装目标。Container image 应使用固定 tag，关键生产负载最好锁定 digest，避免 `latest` 等 floating tags。为 values 添加 `values.schema.json`，让类型和必填项在渲染前失败；为 chart 同时提供 README、NOTES、tests，并在 CI 中执行 `helm lint`、`helm template` 和 server-side dry-run。

CRDs 放在 `crds/` 时会在普通 templates 之前安装，但 Helm 不会像普通资源一样升级或删除 CRDs。这是为了避免误删 custom resources，却也意味着 CRD lifecycle 应由单独 chart、operator 或明确的升级流程管理。

## Kubernetes 运维工程师常用命令

下面以 `payment` release、`payment-prod` namespace 为例。生产操作前先确认 context，永远不要把 namespace 和 cluster context 当作默认值。

### 环境确认与 Chart 检查

```bash
# 确认 Helm、Kubernetes client 和当前 context
helm version
kubectl version --client
kubectl config current-context

# Repository 管理
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo bitnami/redis --versions

# 查看远端 chart 的 metadata、默认 values 和全部内容
helm show chart bitnami/redis
helm show values bitnami/redis > values-reference.yaml
helm show all bitnami/redis

# OCI chart：拉取指定版本到本地
helm pull oci://registry.example.com/charts/payment \
  --version 1.4.2 --untar
```

`helm repo update` 只刷新 repository index，不会升级 cluster 内的 release。正式环境必须显式指定 chart version 或 OCI digest，避免 repository 更新后拿到不可预期版本。

### 渲染、校验与 Dry-run

```bash
# 静态检查 chart
helm lint ./payment -f values-prod.yaml --strict

# 本地渲染，适合代码审查和接入 kubeconform 等工具
helm template payment ./payment \
  --namespace payment-prod \
  -f values-prod.yaml > rendered.yaml

# 连接 API Server 做 dry-run，可发现 schema、RBAC 和 admission 问题
helm upgrade --install payment ./payment \
  --namespace payment-prod \
  -f values-prod.yaml \
  --dry-run=server --debug
```

`helm template` 不访问 cluster，因此无法完整模拟 lookup、CRD discovery、admission webhook 和当前 API capabilities。上线前应优先补一次 `--dry-run=server`，但要注意 debug 或 dry-run 输出可能包含渲染后的 Secret。

### 安装与幂等升级

```bash
# 固定 chart version 安装第三方组件
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --version 4.13.2 \
  -f values-prod.yaml \
  --wait --timeout 10m

# CI/CD 常用的 install-or-upgrade 模式
helm upgrade --install payment oci://registry.example.com/charts/payment \
  --namespace payment-prod \
  --create-namespace \
  --version 1.4.2 \
  -f values-common.yaml \
  -f values-prod.yaml \
  --wait --timeout 10m \
  --rollback-on-failure
```

`--wait` 会等待 Pods、Deployments、StatefulSets、Services 等资源满足条件，`--timeout` 则限制等待时间。不要为了“提高成功率”盲目放大 timeout；先确认 readinessProbe、PodDisruptionBudget、quota 和 rollout strategy 是否合理。

### 查看状态与排障

```bash
# 查看当前 namespace 或所有 namespace 的 releases
helm list -n payment-prod
helm list -A --all

# 查看 release 状态、历史和实际使用的 values
helm status payment -n payment-prod --show-resources
helm history payment -n payment-prod
helm get values payment -n payment-prod --all

# 查看某个 revision 的渲染结果或完整 release 信息
helm get manifest payment -n payment-prod --revision 7
helm get all payment -n payment-prod --revision 7

# Helm 与 Kubernetes 联合排障
kubectl get all -n payment-prod \
  -l app.kubernetes.io/instance=payment
kubectl get events -n payment-prod --sort-by=.lastTimestamp
kubectl rollout status deployment/payment -n payment-prod
```

Helm 的 `STATUS: deployed` 只说明 release 操作成功，不等于业务健康。最终判断仍要结合 Deployment conditions、Pod readiness、events、logs、metrics 和 service-level probes。

### 回滚、测试与卸载

```bash
# 先查看 revision，再回滚到明确版本
helm history payment -n payment-prod
helm rollback payment 6 \
  --namespace payment-prod \
  --wait --timeout 10m

# 执行 chart 内 templates/tests 定义的测试 Pod
helm test payment -n payment-prod --logs

# 卸载 release；默认同时删除 release history
helm uninstall payment -n payment-prod

# 保留历史，便于审计或后续恢复
helm uninstall payment -n payment-prod --keep-history
```

回滚不是数据库回滚。Helm 只能恢复它管理的 Kubernetes manifests 与 release 配置，无法自动逆转 schema migration、外部数据库写入或不可逆的 CRD 变更。涉及数据面的发布必须设计 backward-compatible migration 与独立 recovery plan。

### Chart 开发与依赖管理

```bash
# 创建 chart scaffold
helm create payment

# 更新 dependencies 并生成 Chart.lock
helm dependency update ./payment

# 严格按 Chart.lock 恢复 dependencies
helm dependency build ./payment

# 打包 chart
helm package ./payment --destination ./dist

# 推送到 OCI registry
helm registry login registry.example.com
helm push ./dist/payment-1.4.2.tgz \
  oci://registry.example.com/charts
```

`dependency update` 会重新解析 `Chart.yaml` 中的版本范围并更新 lock；`dependency build` 则根据现有 lock 重建 `charts/`，更适合可复现的 CI build。Chart package、container image 与 Git commit 应建立可追踪关系，并在供应链中加入 provenance、signature 或 OCI digest verification。

## 一套可落地的发布流程

生产环境可以把 Helm workflow 固化成以下 pipeline：开发阶段执行 `helm lint` 和 unit tests；构建阶段用 `helm dependency build` 与 `helm package` 生成 immutable artifact；验证阶段执行 `helm template`、Kubernetes schema validation 和 policy checks；部署前使用 server-side dry-run；部署时固定 chart version/digest，并启用 `--wait`、合理 timeout 与 rollback-on-failure；部署后执行 `helm test`、rollout validation 和业务 smoke test。

Helm 负责 package 与 release lifecycle，GitOps controller 负责持续 reconciliation，两者并不冲突。在 Argo CD 或 Flux 场景中，应由 controller 成为唯一 deployment owner，避免工程师同时从本地执行 `helm upgrade`，否则会形成双写和 configuration drift。

## 总结

Helm 被开发出来，是因为 Kubernetes 擅长管理 resources，却缺少应用级的 package、configuration、distribution 和 release lifecycle。它通过 Chart + Values + Release 把零散 YAML 提升为可版本化的交付单元，并以 client-side CLI/Go library 直接对接 Kubernetes API。

对 Kubernetes 运维工程师而言，真正需要熟练掌握的是三条主线：用 `lint`、`template` 和 server-side dry-run 提前发现问题；用 `upgrade --install`、固定版本、wait 和失败回滚实现可控发布；用 `status`、`history`、`get`、`rollback` 联合 `kubectl` 完成审计和排障。Chart 设计则应被当作稳定 API 维护，而不是把复杂性从 Kubernetes YAML 转移到 Go templates。

## 参考资料

- [Helm Introduction：核心概念与架构](https://helm.sh/docs/intro/introduction/)
- [Helm Project History](https://helm.sh/community/history/)
- [Helm Charts Format](https://helm.sh/docs/topics/charts/)
- [Helm Chart Best Practices](https://helm.sh/docs/chart_best_practices/)
- [Helm 4 Overview](https://helm.sh/docs/overview/)
- [CNCF Helm Project Journey Report](https://www.cncf.io/reports/helm-project-journey-report/)
