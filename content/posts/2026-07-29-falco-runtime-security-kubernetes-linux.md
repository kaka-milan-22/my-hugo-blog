---
title: "Falco 深度解析：Runtime Security 架构、原理与 Kubernetes / Linux 落地"
date: 2026-07-29T08:00:00+08:00
draft: false
tags: ["Falco", "Kubernetes", "eBPF", "Runtime Security", "DevSecOps"]
categories: ["云原生", "DevOps", "Architecture"]
author: "Kaka"
description: "从 kernel event、modern eBPF、Rules Engine 和 Plugins 数据流讲清 Falco 的能力边界，并给出 Kubernetes Operator、Helm 与 Linux host 的生产落地方案。"
---

## 引言

镜像扫描、SBOM 和 admission policy 解决的是“部署前是否允许”，但无法回答容器启动后是否执行了反向 Shell、读取了 `/etc/shadow`、修改了系统二进制目录，或者通过 `setns`、`mount`、`ptrace` 等系统调用尝试逃逸。Falco 解决的正是这个运行时可见性缺口。

Falco 是 CNCF graduated 的开源 **Runtime Security detection engine**。它从 Linux Kernel 或 Plugins 持续接收事件，用 YAML Rules 做实时匹配，再把命中的安全事件发送到日志、HTTP endpoint、Falcosidekick 或 SIEM。它的核心价值是把底层 syscall 转换成包含 process、user、container 和 Kubernetes context 的可操作告警。

本文以 2026 年 7 月的 Falco v0.44 文档线为基准。这个版本已经移除 legacy eBPF、gVisor engine 和 gRPC output；kernel event 采集只保留 `modern_ebpf` 与 `kmod`。Kubernetes 1.29+ 官方优先推荐 Falco Operator，传统 Helm chart 仍然完整支持。

## Falco 能做什么

| 能力 | 实际作用 | 典型场景 |
|---|---|---|
| Linux syscall detection | 观察 process、file、network、namespace、capability 等运行时行为 | 读取敏感文件、反向 Shell、写入 `/etc`、加载 kernel module |
| Container runtime enrichment | 把 syscall 关联到 image、container ID、Pod 和 namespace | 定位异常行为属于哪个 workload |
| Kubernetes Audit detection | 通过 `k8saudit` plugin 分析 API Server audit events | 创建 privileged Pod、执行 `kubectl exec`、修改 RBAC |
| Rules Engine | 用 Rules、Macros、Lists、Exceptions 描述检测逻辑 | 官方规则、组织级基线、应用专属规则 |
| Plugin framework | 接入新的 event source 或补充字段 | AWS CloudTrail、Okta、journald、Kafka、GitLab、Keycloak |
| Alert routing | 输出到 stdout、file、syslog、program 或 HTTP(S) | 通过 Falcosidekick 转发到 70+ integrations |
| Observability | 暴露 event rate、drop、CPU、memory、rule counter 等指标 | Prometheus / Grafana 监控 Falco 自身健康 |
| Hot reload | 自动监听 configuration 和 rules 文件变化 | 不重启进程更新规则与本地配置 |
| Capture | 在 rule 触发后记录 `.scap` 事件流用于分析 | 调试、取证；当前仍应按 Sandbox feature 谨慎使用 |

Falco 的默认规则覆盖 Shell、privilege escalation、credential access、filesystem persistence、容器逃逸迹象和异常网络工具等行为。v0.44 的官方 Rules 还加入了 NPM install script 启动网络工具、Web Server 反向 Shell、cryptominer process 和 container 访问 host sensitive path 等检测。

但能力边界必须说清楚：

1. Falco 默认是 **detect and alert**，不是强制阻断器。需要自动响应时，应通过 Falcosidekick、Falco Talon 或现有 SOAR 执行隔离、删 Pod、封禁 identity 等动作。
2. Falco 按单个 event source 匹配规则，不原生关联 `syscall`、`k8s_audit`、CloudTrail 等多个 source，也不是带时间窗口和实体图谱的 SIEM correlation engine。
3. 它不替代 image scanning、Kubernetes admission、Linux Audit、EDR、Network IDS 或完整 forensic platform，而是其中的 runtime detection 层。

## Falco 的架构与数据流

Falco 的核心链路可以概括为：

```text
Linux Kernel                           External Event Sources
┌────────────────────────────┐         ┌────────────────────────┐
│ syscalls / tracepoints     │         │ K8s Audit / CloudTrail │
│ process / fd / network     │         │ Okta / journald / ...  │
└──────────────┬─────────────┘         └────────────┬───────────┘
               │                                    │
     ┌─────────▼──────────┐                ┌────────▼─────────┐
     │ Falco Driver       │                │ Falco Plugins    │
     │ modern_ebpf / kmod │                │ source/extract   │
     └─────────┬──────────┘                └────────┬─────────┘
               └─────────────────┬──────────────────┘
                                 ▼
                    ┌─────────────────────────┐
                    │ libscap                 │
                    │ capture + normalize     │
                    └────────────┬────────────┘
                                 ▼
                    ┌─────────────────────────┐
                    │ libsinsp                │
                    │ process/fd/container    │
                    │ state and enrichment    │
                    └────────────┬────────────┘
                                 ▼
                    ┌─────────────────────────┐
                    │ Falco Rules Engine      │
                    │ rules/macros/lists      │
                    │ filters/exceptions      │
                    └────────────┬────────────┘
                                 ▼
                    ┌─────────────────────────┐
                    │ Outputs                 │
                    │ JSON/HTTP/syslog/file   │
                    │ Falcosidekick → SIEM    │
                    └─────────────────────────┘
```

### 第一层：Driver 捕获 kernel events

Falco 的 `syscall` event source 由 kernelspace driver 提供。Driver 不只是把所有 syscall 原样复制到 userspace，而是根据 Falco 启用的 Rules 和维护内部状态所需的 syscall 集合做 adaptive selection，再把 event 写入 kernel/userspace 共享的 ring buffer。

| Driver | 机制 | 优点 | 代价与限制 |
|---|---|---|---|
| `modern_ebpf` | CO-RE eBPF、BTF、BPF ring buffer | 内嵌于 Falco binary，不需下载或编译 probe；默认首选 | 通常需要 Linux 5.8+ 的 BTF 与 ring buffer feature，或发行版 backport |
| `kmod` | 独立 kernel module、ring buffer、`ioctl` | 可覆盖较老 kernel，x86_64 / aarch64 支持到 3.10 | 需要 privileged、匹配 kernel 的 module，升级 kernel 后要重新加载或构建 |

`modern_ebpf` 不以 kernel version 数字做绝对判断，而是检查所需 feature 是否存在。Linux 5.8 通常已经满足要求，发行版也可能向旧 kernel backport。其 least-privileged 模式主要需要 `BPF`、`PERFMON`、`SYS_RESOURCE` 和 `SYS_PTRACE` capabilities，但在 Kubernetes 中依然需要 host PID、host `/proc` 与 tracefs 等访问，因此应放入专用 security namespace，并严格限制谁能修改 Falco workload。

`libscap` 对不同 driver 的通信细节做统一封装。启动时，它会协商 Driver API Version 和 Schema Version：前者保证 kernel/userspace 通信兼容，后者说明当前 driver 能产生哪些 event 和 fields。无论底层来自 `kmod` 还是 `modern_ebpf`，上层看到的都是一致的 SCAP event format。

### 第二层：libsinsp 维护运行时上下文

单个 `execve` 或 `openat` 本身信息有限。`libsinsp` 会持续维护 process tree、thread、file descriptor、network connection、user 和 container metadata，使 Rules 能直接使用：

```text
proc.name / proc.cmdline / proc.exepath / proc.aname
user.name / user.uid / user.loginuid
fd.name / fd.directory / fd.sip / fd.sport
container.id / container.image.repository
k8s.ns.name / k8s.pod.name
```

Falco v0.44 的 `modern_ebpf` 还可以通过 BPF iterators 初始化和修复 process / fd state，减少大规模主机上反复扫描 `/proc` 的开销。若发生 event drop，Falco 不仅可能漏掉一次检测，还可能失去后续事件所依赖的上下文，因此 drop rate 是必须监控的安全 SLI。

### 第三层：Rules Engine 做行为匹配

Rules 文件是 YAML，核心元素包括：

- **Rule**：condition 命中时生成 alert。
- **Macro**：可复用的 condition 片段。
- **List**：可复用的数据集合。
- **Exception**：按 actor、target 等字段组合定义允许行为。

下面的规则检测 production namespace 中启动交互式 Shell：

```yaml
- macro: production_container
  condition: >
    container.id != host
    and k8s.ns.name in (prod, production)

- rule: Interactive shell in production container
  desc: Detect an interactive shell started in a production container
  condition: >
    production_container
    and evt.type in (execve, execveat)
    and proc.name in (bash, sh, zsh, dash, ksh)
    and proc.tty != 0
  output: >
    Interactive shell in production
    (user=%user.name command=%proc.cmdline image=%container.image.repository
    namespace=%k8s.ns.name pod=%k8s.pod.name)
  priority: WARNING
  tags: [maturity_incubating, container, k8s, process, mitre_execution]
```

Rules 的强项是条件可读、版本可控、能直接进入 GitOps。它的难点是 false positive：相同的 Shell 在 debug Pod 中可能合法，在 payment production Pod 中却是高风险。生产调优时应优先使用“actor + target + scope”组合 Exception，例如 image repository、namespace 与目标路径，不要仅用 `proc.name` 做宽泛放行。

### 第四层：Plugins 与 Outputs 扩展边界

Falco Plugins 是动态 shared libraries，可实现 event sourcing、field extraction、event parsing 和 async event injection。常见 source 包括：

```text
syscall       → modern_ebpf / kmod
k8s_audit     → k8saudit / EKS / GKE / AKS variants
aws_cloudtrail→ cloudtrail plugin
okta          → okta plugin
journal       → journald plugin
kafka         → kafka plugin
```

每个 event source 在独立线程中消费并匹配属于自己的 Rules。多个 source 可以同时运行，但 Rules 不能直接跨 source correlation。比如 `kubectl exec` 的 API audit event 与容器内 `execve` 可以分别告警，真正把二者按 namespace、Pod、时间和 identity 串起来，应交给 SIEM 或 stream processing pipeline。

Falco v0.44 原生 output channel 是 stdout、file、syslog、program 和 HTTP(S)。生产中通常打开 JSON output，通过 HTTP 发送给 Falcosidekick，再路由到 Kafka、Loki、Elasticsearch、Splunk、Alertmanager、Slack 或其他平台。旧的 gRPC output 已在 v0.44 移除，不应再使用旧教程中的 `grpc_output` 配置。

## Kubernetes 如何配置与落地

### Operator 还是 Helm chart

| 方案 | 适用场景 | 判断 |
|---|---|---|
| Falco Operator | Kubernetes 1.29+；希望用 CRD 管理 instance、rules、plugins 和 config | 新集群首选 |
| Falco Helm chart | 已有稳定 values/GitOps 流程；不希望引入 Operator；需要成熟兼容路径 | 继续可用 |

Falco Operator 使用 `Falco`、`Component`、`Rulesfile`、`Plugin` 和 `Config` 五类 CRD。Falco instance 通常以 DaemonSet 运行，每个 Linux node 一个 sensor；Artifact Operator 作为 native sidecar，将 OCI、ConfigMap 或 inline 中的 Rules、Plugins 和 configuration fragments 写入共享 volume，并处理更新与状态。

安装 Operator：

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

helm upgrade --install falco-operator falcosecurity/falco-operator \
  --namespace falco-operator \
  --create-namespace \
  --version 0.2.0 \
  --wait

kubectl create namespace falco \
  --dry-run=client -o yaml | kubectl apply -f -
```

部署 syscall sensor、container plugin 和官方 Rules：

```yaml
apiVersion: instance.falcosecurity.dev/v1alpha1
kind: Falco
metadata:
  name: falco
  namespace: falco
spec:
  type: DaemonSet
---
apiVersion: artifact.falcosecurity.dev/v1alpha1
kind: Plugin
metadata:
  name: container
  namespace: falco
spec:
  ociArtifact:
    image:
      repository: falcosecurity/plugins/plugin/container
      tag: "0.7.1"
    registry:
      name: ghcr.io
---
apiVersion: artifact.falcosecurity.dev/v1alpha1
kind: Rulesfile
metadata:
  name: falco-rules
  namespace: falco
spec:
  ociArtifact:
    image:
      repository: falcosecurity/rules/falco-rules
      tag: "5.1.0"
    registry:
      name: ghcr.io
  priority: 50
```

这里显式 pin 了 artifact version。官方 quickstart 使用 `latest` 便于体验，但 production 应由 Renovate、GitOps pipeline 或人工 review 提交版本升级，避免 Rules 或 Plugin 在未验证时自动漂移。

如果需要 Deployment、ReplicaSet、Service 等更完整的 Kubernetes metadata，再部署 `k8s-metacollector` 与 `k8smeta` plugin。只需要 Pod name、namespace、image 等基本上下文时，container plugin 已能从 CRI socket 提取，不要把 `k8saudit` 当作 metadata collector；`k8saudit` 的职责是检测 API Server audit events。

### 使用 Helm chart 的 production values

当前 Falco chart 9.1.0 对应 Falco 0.44.1。下面是一份可作为起点的 `values-production.yaml`：

```yaml
controller:
  kind: daemonset

driver:
  enabled: true
  kind: modern_ebpf
  modernEbpf:
    leastPrivileged: true
    bufSizePreset: 4
    cpusForEachBuffer: 2

collectors:
  enabled: true
  containerEngine:
    enabled: true
  kubernetes:
    enabled: true

tty: true

resources:
  requests:
    cpu: 100m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1024Mi

falco:
  time_format_iso_8601: true
  priority: notice
  json_output: true
  stdout_output:
    enabled: true
  syslog_output:
    enabled: false

metrics:
  enabled: true
  interval: 15m
  outputRule: false
  rulesCountersEnabled: true
  resourceUtilizationEnabled: true
  stateCountersEnabled: true
  kernelEventCountersEnabled: true

serviceMonitor:
  create: true

falcosidekick:
  enabled: true

falcoctl:
  artifact:
    follow:
      enabled: false
  config:
    artifact:
      install:
        refs: [falco-rules:5.1.0]
```

部署前需要处理 Pod Security Admission。即使使用 least-privileged capabilities，Falco 仍需 host-level access，通常应把它放在专用 namespace：

```bash
kubectl create namespace falco --dry-run=client -o yaml | kubectl apply -f -
kubectl label namespace falco \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged \
  --overwrite

helm upgrade --install falco falcosecurity/falco \
  --namespace falco \
  --version 9.1.0 \
  -f values-production.yaml \
  --wait --timeout 10m
```

`ServiceMonitor` 只有在 cluster 已安装 Prometheus Operator CRD 时才能打开。Falcosidekick 的 webhook、token 和 credentials 不要放入 values Git repository，应通过 Kubernetes Secret、External Secrets 或现有 secret management pipeline 注入。

### 验证完整检测链路

先确认 DaemonSet 覆盖所有目标 Linux nodes：

```bash
kubectl get daemonset,pods -n falco -o wide
kubectl logs -n falco -l app.kubernetes.io/name=falco \
  -c falco --tail=100
```

然后只在 test namespace 触发受控事件：

```bash
kubectl create namespace falco-test
kubectl run falco-test \
  --namespace falco-test \
  --image=alpine:3.22 \
  --restart=Never \
  -- sleep 3600

kubectl wait pod/falco-test \
  --namespace falco-test \
  --for=condition=Ready \
  --timeout=120s

kubectl exec -n falco-test falco-test -- cat /etc/shadow

kubectl logs -n falco -l app.kubernetes.io/name=falco \
  -c falco --since=5m | grep -E 'Sensitive file|falco-test'

kubectl delete namespace falco-test
```

生产验收不能只看 Falco Pod 是 `Running`，还要验证 Falco → Falcosidekick → Kafka/SIEM/Alertmanager 的 end-to-end latency，并对以下指标建立告警：

```text
Falco Pod / process availability
scap event drops and drop percentage
Falco output queue drops
event rate per CPU
CPU and memory utilization
rule match counters
Falcosidekick delivery failures
```

## Linux host 如何配置与落地

Falco package 支持 x86_64 和 aarch64，bundled plugins 要求 GLIBC 2.28+。对 Ubuntu 22.04 这类现代发行版，优先使用 `modern_ebpf`，避免 DKMS、kernel headers 和 kernel upgrade 后重建 module 的维护成本。

### 安装 Falco

```bash
uname -m
ldd --version | head -n 1
test -r /sys/kernel/btf/vmlinux && echo "BTF available"

sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

curl -fsSL https://falco.org/repo/falcosecurity-packages.asc |
  sudo gpg --dearmor \
    -o /usr/share/keyrings/falco-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/falco-archive-keyring.gpg] https://download.falco.org/packages/deb stable main" |
  sudo tee /etc/apt/sources.list.d/falcosecurity.list

sudo apt-get update
sudo env \
  FALCO_FRONTEND=noninteractive \
  FALCO_DRIVER_CHOICE=modern_ebpf \
  apt-get install -y falco

sudo systemctl enable --now falco-modern-bpf.service
sudo systemctl status falco --no-pager
```

对于 kernel feature 不一致的大规模 host fleet，可以不设置 `FALCO_DRIVER_CHOICE`，让 installer 自动优先选择 `modern_ebpf`、不满足条件时回退 `kmod`。如果强制使用 `kmod`，需额外安装匹配当前 kernel 的 headers、DKMS 和 build tool，并把 kernel upgrade 纳入 driver rebuild 验证。

### 使用 configuration overlay

不要直接长期修改 package 管理的主文件。Falco 0.38+ 默认加载 `/etc/falco/config.d/`，后加载的文件覆盖主配置；`watch_config_files` 默认开启，修改 configuration 或 Rules 后会自动 hot reload。

创建 `/etc/falco/config.d/10-production.yaml`：

```yaml
engine:
  kind: modern_ebpf
  modern_ebpf:
    cpus_for_each_buffer: 2
    buf_size_preset: 4
    drop_failed_exit: false

time_format_iso_8601: true
priority: notice
json_output: true

stdout_output:
  enabled: false
syslog_output:
  enabled: true

metrics:
  enabled: true
  interval: 15m
  output_rule: false
  rules_counters_enabled: true
  resource_utilization_enabled: true
  state_counters_enabled: true
  kernel_event_counters_enabled: true
  libbpf_stats_enabled: true

webserver:
  enabled: true
  listen_address: 127.0.0.1
  listen_port: 8765
  prometheus_metrics_enabled: true
```

把组织级 Rules 放到 `/etc/falco/rules.d/`，而不是修改升级时会被覆盖的 `/etc/falco/falco_rules.yaml`。变更前先 dry-run：

```bash
sudo falco \
  --dry-run \
  -c /etc/falco/falco.yaml

sudo journalctl -u falco -f
curl -fsS http://127.0.0.1:8765/healthz
curl -fsS http://127.0.0.1:8765/metrics | head
```

Host 上的 webserver 建议只监听 loopback，由 node-local Prometheus agent 抓取；如果必须监听 `0.0.0.0`，应同时配置 host firewall、network ACL 和 TLS，不要把 health、version 与 metrics endpoint 直接暴露到不受信任网络。

## 生产落地方法

Falco 的技术安装只占很小一部分，真正决定效果的是 Rules lifecycle 和 response process。建议按以下阶段实施：

### 第一阶段：建立可见性

先启用 `maturity_stable` 官方规则和 JSON output，接入统一日志平台，连续观察 7 到 14 天。此时只告警、不自动处置，记录每条规则的触发量、业务 owner、false positive 原因与需要补充的 fields。

### 第二阶段：按环境调优

用 namespace、image digest、ServiceAccount、process ancestry、file path 和 network destination 缩小 scope。Exception 必须保留 actor 与 target 两侧约束，避免“允许某个 binary 的全部行为”。开发、CI runner、database 和 ingress node 应维护不同规则 profile。

### 第三阶段：把 Rules 当代码

Rules、Falco configuration、Helm values 和 Operator CRs 全部进入 Git。CI 至少做 YAML validation、Falco dry-run、event-generator regression 和 rendered manifest policy check。Falco、Rules、Plugins 与 chart version 都应 pin，并在 canary node pool 验证后推广。

### 第四阶段：建立安全 SLI

重点不是“Falco 运行了多久”，而是数据是否完整、告警是否送达。持续观察 event drop、output queue drop、CPU saturation、memory、rule count 和 end-to-end delivery。高负载节点发生 drop 时，先检查 Rules 和 `base_syscalls`，再按数据调整 `buf_size_preset` 与 `cpus_for_each_buffer`，不要盲目无限增大 buffer。

### 第五阶段：谨慎启用自动响应

只有当规则 false positive 已可控、资产 owner 明确且操作可回滚时，才对极少数高置信度事件启用 response。例如隔离明确的 cryptominer Pod、标记 node、冻结 compromised workload identity。自动删除 Pod 或封禁网络的 blast radius 很大，应通过 Falco Talon / SOAR 设置 allowlist、timeout、rate limit、approval 和完整审计。

## 总结

Falco 的本质是一个靠近 Linux Kernel 的实时行为检测 pipeline：`modern_ebpf` 或 `kmod` 捕获 syscall，`libscap` 统一 event format，`libsinsp` 维护 process、fd 与 container context，Rules Engine 匹配行为，Plugins 扩展 event sources，Outputs 把结果送入安全运营体系。

对 Kubernetes 1.29+ 新环境，优先评估 Falco Operator，以 CRD 管理 instance、Rules、Plugins 和 configuration；对已有成熟 GitOps 流程的集群，Falco Helm chart 仍是可靠选择。Linux host 上优先使用 `modern_ebpf` 与 configuration overlay，避免把长期配置写进 package 管理文件。

最终判断 Falco 是否落地成功，不是看 DaemonSet 是否 `Running`，而是看三件事：runtime events 是否完整、Rules 是否低噪声、告警是否能在可接受延迟内触发人工或自动响应。

## 参考资料

- [Falco 官方文档](https://falco.org/docs/)
- [Falco v0.44.0 Release Notes](https://falco.org/blog/falco-0-44-0/)
- [Kernel Events 与 Drivers](https://falco.org/docs/concepts/event-sources/kernel/)
- [Kernel Events Architecture](https://falco.org/docs/concepts/event-sources/kernel/architecture/)
- [Falco Rules](https://falco.org/docs/concepts/rules/)
- [Falco Plugins](https://falco.org/docs/concepts/plugins/)
- [Falco Outputs](https://falco.org/docs/concepts/outputs/)
- [Falco Metrics](https://falco.org/docs/concepts/metrics/)
- [Falco Operator](https://falco.org/docs/setup/operator/)
- [Falco Helm Chart](https://artifacthub.io/packages/helm/falcosecurity/falco)
- [Linux DEB / RPM 安装](https://falco.org/docs/setup/packages/)
