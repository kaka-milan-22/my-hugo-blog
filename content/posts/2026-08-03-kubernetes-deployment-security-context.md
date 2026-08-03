---
title: "Kubernetes Deployment SecurityContext：从默认权限到最小权限"
date: 2026-08-03T10:00:00+08:00
draft: false
tags: ["Kubernetes", "SecurityContext", "Pod Security", "容器安全", "DevOps"]
categories: ["云原生", "DevOps"]
author: "Kaka"
description: "从容器默认权限的风险出发，解释 Kubernetes Deployment 为什么需要 SecurityContext、它解决了哪些运行时安全问题，并通过一个可验证的 restricted 实例展示落地方法。"
---

## 引言

很多 Kubernetes Deployment 只定义了镜像、端口和资源限制，容器能运行就算完成。但“能运行”不等于“以合理的权限运行”：镜像可能默认使用 root，进程可能拥有不需要的 Linux capabilities，容器根文件系统也可能被任意写入。一旦应用存在 Remote Code Execution（RCE）漏洞，攻击者继承的正是这些运行时权限。

`SecurityContext` 的价值，是把 Linux 用户、group、capabilities、seccomp 和文件系统权限变成声明式的 Pod 配置。它不能消除应用漏洞，也不是完整的 container sandbox，但可以把入侵后的权限和破坏范围压到业务真正需要的最小值。

## SecurityContext 在 Deployment 的什么位置

严格来说，SecurityContext 属于 Pod 和 container，而不是 Deployment 自身。Deployment 通过 `.spec.template` 定义 Pod template，因此配置实际位于两个层级：

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      securityContext:                 # Pod-level
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: app
          securityContext:             # container-level
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
```

Pod-level 配置适合表达所有 container 共享的身份、group、volume 权限和 seccomp profile；container-level 配置适合表达某个进程独有的 privilege escalation、root filesystem 和 capabilities 限制。两层出现相同字段时，container-level 配置覆盖 Pod-level 配置。

## 1. 为什么要引入 SecurityContext

### 容器隔离不等于权限最小化

容器依赖 Linux namespaces、cgroups 等机制隔离进程，但仍与宿主机共享 kernel。容器内的 root 不等同于宿主机 root，不过它通常拥有更大的进程权限和更宽的攻击面；一旦再叠加 `privileged`、危险 capability、hostPath 或 kernel 漏洞，影响可能越过容器边界。

更常见的问题甚至不需要 container escape。攻击者只要拿到应用进程权限，就可能修改容器内的二进制或配置、在可写目录落地工具、读取挂载给 Pod 的数据，并利用 setuid 程序或多余 capability 扩大权限。因此，容器安全不能只依赖“镜像里应该配置好了”。

### 镜像默认值不是可靠的安全边界

Dockerfile 可以通过 `USER` 指定非 root 用户，但实际环境里经常存在三类偏差：基础镜像默认使用 root；升级镜像后 `USER` 被改变；不同团队构建的镜像采用不同 UID/GID。只在镜像层处理，会让平台无法从 Deployment manifest 判断工作负载最终以什么权限运行。

SecurityContext 把要求放到 Kubernetes API 中。即使镜像默认是 root，`runAsNonRoot: true` 也会阻止它以 root 启动；Deployment 每次扩容、重建和滚动发布时，ReplicaSet 创建的所有 Pod 都会继承同一套限制。这使安全配置能够进入 GitOps、code review 和 policy validation 流程，而不是散落在镜像约定或人工启动参数中。

### 安全需要“工作负载配置”和“平台强制”两条线

SecurityContext 描述一个工作负载打算怎样运行，但有权限修改 Deployment 的人也可以删除这些字段。因此它解决的是 runtime hardening 和配置一致性，不负责强制组织策略。生产环境还应通过 Pod Security Admission、Kyverno 或 OPA Gatekeeper 验证配置；本文实例使用 Kubernetes 内置的 Pod Security Admission `restricted` profile。

## 2. 引入后解决了哪些问题

| 风险 | SecurityContext 配置 | 实际作用 |
|---|---|---|
| 进程以 root 运行 | `runAsNonRoot`、`runAsUser`、`runAsGroup` | 明确进程 UID/GID，并拒绝 root 身份 |
| 子进程通过 setuid/setgid 提权 | `allowPrivilegeEscalation: false` | 为进程设置 `no_new_privs`，阻止获得高于父进程的权限 |
| 默认 Linux capabilities 过多 | `capabilities.drop: ["ALL"]` | 先移除全部 capability，只按业务需要加回最小集合 |
| 攻击者修改系统文件或落地后门 | `readOnlyRootFilesystem: true` | 将 container root filesystem 挂载为只读；必要写目录单独挂 volume |
| 应用可调用过多 system calls | `seccompProfile.type: RuntimeDefault` | 使用 container runtime 的默认 seccomp profile 过滤高风险 system calls |
| 非 root 进程无法写共享 volume | `fsGroup`、`fsGroupChangePolicy` | 为支持的 volume 设置 group ownership，让应用不必用 root 修权限 |

### 三项基础限制分别控制什么

最常见的 hardening 组合如下。如果配置放在 `containers[].securityContext`，三个字段可以写在一起；本文完整实例把所有 container 都要继承的 `runAsNonRoot` 放到了 Pod-level，效果相同。

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  runAsGroup: 10001
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

`runAsNonRoot: true` 控制“进程是谁”。Kubelet 会确保 container 的入口进程不以 UID `0` 运行；如果 image 默认用户是 root，Pod 会启动失败，而不是静默降级。生产配置最好同时明确 numeric `runAsUser` 和 `runAsGroup`，这样行为不依赖 image 内的用户名及 `/etc/passwd`。

`allowPrivilegeEscalation: false` 控制“进程以后能否获得更多权限”。它让 container process 启用 Linux `no_new_privs`：子进程通过 `execve` 启动 setuid、setgid 或带 file capability 的程序时，不能获得高于父进程的新权限。例如 Web application 被 RCE 后，即使攻击者找到一个 setuid binary，也不能借它从 UID `10001` 升级成 root。这个字段不会删除进程已经拥有的权限；当 container 使用 `privileged: true` 或拥有 `CAP_SYS_ADMIN` 时，也不能依赖它提供有效保护。

`capabilities.drop: ["ALL"]` 控制“kernel 允许进程执行哪些特权操作”。Linux capabilities 把传统 root 的全能权限拆成一组独立权限位；Kubernetes 不保证 container 的默认 capability 集合，实际默认值由 container runtime 和 OCI 配置决定。`ALL` 要求 runtime 在进程启动前清空 capability set，需要特殊 capability 的 system call 随后会被 kernel 以 `EPERM` 拒绝。

三者组合后的攻击路径就很直观：

```text
Application 出现 RCE
        ↓
攻击者拿到 UID 10001 的 shell       ← runAsNonRoot
        ↓
shell 没有额外 kernel privilege      ← capabilities.drop ALL
        ↓
执行 setuid/file-capability 程序也不能提权
                                       ← allowPrivilegeEscalation: false
```

### `drop: ["ALL"]` 具体删除了什么

Capabilities 并不是命令或程序，而是 kernel 在执行敏感操作时检查的权限位。常见 container runtime 可能提供的能力及其风险如下；具体默认集合可能因 runtime、版本与 process UID 而不同，因此不应把某个环境的默认值当成安全 contract。

| Capability | 能做什么 | 删除后的效果 |
|---|---|---|
| `CAP_CHOWN` | 任意修改文件 UID/GID ownership | 不能随意 `chown` 不属于自己的文件 |
| `CAP_DAC_OVERRIDE` | 绕过普通文件 `rwx` 权限检查 | 只能访问 UID、GID 和 mode 允许的文件 |
| `CAP_FOWNER` | 绕过“必须是文件 owner”的权限检查 | 不能随意修改其他用户文件的属性 |
| `CAP_SETUID`、`CAP_SETGID` | 切换进程 UID/GID 和 supplementary groups | 不能利用这些能力改变 process identity |
| `CAP_NET_RAW` | 创建 raw socket，构造 ICMP/IP packet | raw packet crafting 会被拒绝 |
| `CAP_NET_BIND_SERVICE` | 在需要该能力的环境中监听低于 `1024` 的端口 | 应优先监听 `8080` 等非特权端口 |
| `CAP_KILL` | 向不同 UID 的进程发送 signal | 只能 signal 普通权限允许的目标进程 |
| `CAP_MKNOD` | 创建 block/character device node | 不能创建新的 device node；device cgroup 仍是另一层限制 |
| `CAP_SETFCAP` | 给文件写入 file capability | 不能为二进制预埋额外 capability |
| `CAP_SYS_CHROOT` | 改变进程的 root directory | `chroot` 会被拒绝 |

如果应用确实需要某项 capability，正确做法仍是先删除全部，再恢复经过验证的最小集合：

```yaml
capabilities:
  drop: ["ALL"]
  add: ["NET_BIND_SERVICE"]  # 仅在确实必须监听低端口时添加
```

Pod Security Standards 的 `restricted` profile 只允许加回 `NET_BIND_SERVICE`。多数 Web application 更适合在 container 内监听 `8080`，再通过 Service 将端口映射为 `80`，这样完全不需要增加 capability。

这些控制是纵深防御关系，不是互相替代。`runAsNonRoot` 只约束 UID，不能证明 capability set 为空；`drop ALL` 也不会阻止普通 outbound TCP、读取当前 UID 有权访问的文件、访问已挂载 Secret 或执行普通 system calls。只读 root filesystem 同样不会让挂载的 `emptyDir`、PVC 或 Secret 自动变成只读，每个权限面都要单独收紧。

SecurityContext 同样有明确边界：它不管理 east-west network traffic，不能替代 NetworkPolicy；不控制谁能修改 Deployment，不能替代 RBAC；不扫描 image vulnerability，也不能替代 image signing、SBOM 和 admission policy；更不能保护被应用主动读取并泄露的 Secret。它解决的是 Pod 内进程的 runtime identity、privilege 和 filesystem access control。

## 3. 实用实例：一个符合 restricted 策略的静态服务

下面部署两个 BusyBox HTTP server 副本。进程固定使用 UID/GID `10001`，监听非特权端口 `8080`，root filesystem 只读，所有 Linux capabilities 被移除。网页来自只读 ConfigMap；如果程序需要临时写入，只允许写入有容量限制的 memory-backed `emptyDir`。

Namespace 同时启用 Pod Security Admission。`enforce` 会拒绝不符合 `restricted` profile 的新 Pod，`warn` 和 `audit` 分别给客户端与 audit log 提供迁移信号。示例固定到 `v1.36`，实际使用时应改成集群当前 minor version，并在升级前先评估新版本标准。

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: security-demo
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.36
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.36
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.36
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-content
  namespace: security-demo
data:
  index.html: |
    <!doctype html>
    <html lang="zh-CN">
      <meta charset="utf-8">
      <title>SecurityContext Demo</title>
      <body><h1>Running with least privilege</h1></body>
    </html>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: security-context-demo
  namespace: security-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: security-context-demo
  template:
    metadata:
      labels:
        app: security-context-demo
    spec:
      os:
        name: linux
      nodeSelector:
        kubernetes.io/os: linux
      automountServiceAccountToken: false

      # Pod-level：所有 container 共享的运行身份和 seccomp profile
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        fsGroupChangePolicy: OnRootMismatch
        seccompProfile:
          type: RuntimeDefault

      containers:
        - name: web
          image: busybox:1.36.1
          imagePullPolicy: IfNotPresent
          command: ["httpd", "-f", "-p", "8080", "-h", "/www"]
          ports:
            - name: http
              containerPort: 8080
          resources:
            requests:
              cpu: 10m
              memory: 16Mi
            limits:
              cpu: 100m
              memory: 32Mi
          readinessProbe:
            httpGet:
              path: /
              port: http
          livenessProbe:
            httpGet:
              path: /
              port: http

          # container-level：限制这个进程的 privilege 和 filesystem
          securityContext:
            privileged: false
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]

          volumeMounts:
            - name: web-content
              mountPath: /www
              readOnly: true
            - name: tmp
              mountPath: /tmp

      volumes:
        - name: web-content
          configMap:
            name: web-content
        - name: tmp
          emptyDir:
            medium: Memory
            sizeLimit: 16Mi
---
apiVersion: v1
kind: Service
metadata:
  name: security-context-demo
  namespace: security-demo
spec:
  selector:
    app: security-context-demo
  ports:
    - name: http
      port: 80
      targetPort: http
```

把内容保存为 `security-context-demo.yaml` 后应用，并等待 Deployment rollout 完成：

```bash
kubectl apply -f security-context-demo.yaml
kubectl rollout status deployment/security-context-demo -n security-demo
kubectl get pod -n security-demo -l app=security-context-demo
```

通过 port-forward 验证服务，不需要额外创建一个权限配置不完整的调试 Pod：

```bash
kubectl port-forward -n security-demo service/security-context-demo 8080:80
curl http://127.0.0.1:8080/
```

接下来直接检查运行时结果。`id` 应显示 UID/GID `10001`；`NoNewPrivs` 应为 `1`；`CapInh`、`CapPrm`、`CapEff`、`CapBnd` 和 `CapAmb` 是 Linux process 的不同 capability set，全部应为全零。写 container root filesystem 会失败，而单独挂载的 `/tmp` 可以写入：

```bash
kubectl exec -n security-demo deploy/security-context-demo -- id

kubectl exec -n security-demo deploy/security-context-demo -- \
  sh -c 'grep -E "^(NoNewPrivs|Cap(Inh|Prm|Eff|Bnd|Amb)):" /proc/1/status'

kubectl exec -n security-demo deploy/security-context-demo -- \
  sh -c 'touch /should-fail'
# touch: /should-fail: Read-only file system

kubectl exec -n security-demo deploy/security-context-demo -- \
  sh -c 'touch /tmp/allowed && ls -l /tmp/allowed'
```

最后可以故意删除 `allowPrivilegeEscalation: false` 再执行 `kubectl apply`，观察 Pod Security Admission 的两个检查阶段。`kubectl apply` 提交的是 Deployment object，不是 Pod；API server 会对 Deployment template 执行 `warn` 检查，因此通常会先打印 `restricted` warning，但 `enforce` 不会在这个阶段拒绝 Deployment。

Deployment 被保存后，Deployment controller 创建 ReplicaSet，ReplicaSet controller 再向 API server 请求创建真正的 Pod。此时 `enforce` 才检查 Pod；由于 `restricted` 要求显式设置 `allowPrivilegeEscalation: false`，Pod 请求会被拒绝。最终现象是 `kubectl apply` 看似成功，Deployment 和 ReplicaSet 也存在，但可用 Pod 始终创建不出来，rollout 一直失败：

```text
kubectl apply
  → Deployment 创建成功，并可能显示 restricted warning
  → Deployment controller 创建 ReplicaSet
  → ReplicaSet controller 请求创建 Pod
  → Pod Security Admission enforce 拒绝违规 Pod
  → Deployment 无法达到期望副本数，rollout 失败
```

这就是为什么上线检查不能只看 `kubectl apply` 的退出状态，还必须检查 rollout status 和 ReplicaSet event：

```bash
kubectl rollout status deployment/security-context-demo \
  -n security-demo --timeout=60s

kubectl describe replicaset -n security-demo \
  -l app=security-context-demo
```

`kubectl describe` 的 event 中会出现类似 `FailedCreate: violates PodSecurity "restricted"` 的原因。修复配置并重新 apply 后，controller 才能创建符合策略的新 Pod。

## 常见兼容性问题

| 现象 | 原因 | 推荐处理 |
|---|---|---|
| 设置 `runAsNonRoot` 后 container 无法启动 | image 依赖 root，或未提供可验证的非 root 用户 | 在 image 中创建固定 numeric UID/GID，并同步设置 `runAsUser`、`runAsGroup` |
| 开启只读 root filesystem 后报 `Permission denied` | 应用要写 `/tmp`、cache、PID 或 runtime data | 逐个识别写目录，为它们挂载有容量限制的 `emptyDir` 或 PVC |
| 非 root 进程不能监听端口 80 | 低端口需要额外权限，且不同 runtime/kernel 配置可能有差异 | 优先改为监听 `8080` 等非特权端口，由 Service 映射到 80 |
| PVC 挂载后不可写 | volume ownership 与进程 GID 不匹配 | 优先使用 `fsGroup`；同时确认 CSI driver 是否支持 ownership 管理 |
| `RuntimeDefault` 导致少数旧应用异常 | 应用依赖被默认 seccomp profile 拦截的 system call | 先观测和定位，再使用经过审核的 `Localhost` profile，不要直接改成 `Unconfined` |

生产镜像还应固定 digest，避免相同 tag 指向不同内容。若某个应用确实需要 capability，应保持 `drop: ["ALL"]`，然后只在 `add` 中恢复经过验证的单项；例如确实必须监听低端口时才考虑 `NET_BIND_SERVICE`，不要为了省事恢复默认 capability 集合。

## 总结

SecurityContext 的核心不是“增加几个安全字段”，而是把 least privilege 变成可审查、可复制的 Deployment contract。Pod-level 负责统一身份、volume group 和 seccomp，container-level 负责 privilege escalation、capabilities 与 root filesystem；两者结合，才能降低应用被攻陷后的权限和持久化能力。

落地时应从 `runAsNonRoot`、`allowPrivilegeEscalation: false`、`capabilities.drop: ["ALL"]`、`readOnlyRootFilesystem: true` 和 `RuntimeDefault` 这组基线开始，再为真实写目录和必要能力做最小例外。最后通过 Pod Security Admission 的 `restricted` profile 强制执行，否则 SecurityContext 仍只是一份可以被删掉的自律配置。

## 参考资料

- [Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [Enforcing Pod Security Standards](https://kubernetes.io/docs/setup/best-practices/enforcing-pod-security-standards/)
