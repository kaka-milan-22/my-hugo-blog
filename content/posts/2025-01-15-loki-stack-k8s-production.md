---
title: "Loki Stack 生产实践：K8s 环境下日均 1T 日志的高可用架构"
date: 2025-01-15T10:00:00+08:00
lastmod: 2025-01-15T10:00:00+08:00
draft: false
tags: ["loki", "kubernetes", "observability", "logging", "devops", "grafana"]
categories: ["可观测性", "运维架构"]
author: "Platform Engineering"
description: "深入剖析 Loki Stack 在 Kubernetes 生产环境中的落地实践，涵盖 Ingestor 横向扩展、标签设计策略与日均 1TB 日志场景的完整配置方案。"
summary: "本文从命名由来出发，系统介绍 Loki Stack 的核心优势，并以 K8s 生产集群日均 1TB 日志收集为例，详细讲解 Ingestor 横向扩展机制与标签设计最佳实践，附带完整可落地的配置示例。"
toc: true
weight: 1
---

## 为什么叫 Loki Stack？

Loki 的命名来自北欧神话中的「洛基」——奥丁的义兄，一位善于变形、以诡计著称的神灵。Grafana 团队选用这个名字，有意借用了洛基的两个核心特质：

1. **轻盈变形**：Loki 不像 Elasticsearch 那样对日志内容做全量索引，它只对**标签（Labels）**建立索引，日志正文以压缩块的形式原始存储。这正是洛基「变形」的隐喻——表面轻量，内藏乾坤。
2. **狡黠高效**：通过放弃全文索引换取极低的存储和查询开销，在资源紧张的 K8s 环境里游刃有余。

**Loki Stack** 则是围绕 Loki 构建的完整可观测链路组合：

| 组件 | 角色 | 类比 Prometheus 生态 |
|---|---|---|
| **Promtail / Alloy** | 日志采集 Agent | Node Exporter |
| **Loki** | 日志存储与查询引擎 | Prometheus Server |
| **Grafana** | 可视化与告警 | Grafana |
| **Ruler** | LogQL 告警规则评估 | Alertmanager Rules |

这套 Stack 与 Prometheus + Grafana 共用同一套标签体系，天然构成 **Metrics + Logs** 的统一可观测视图，这是它被称为 "Loki Stack" 而非单独的 "Loki" 的核心原因。

---

## Loki Stack 核心优势

### 1. 极低的存储成本

Elasticsearch 的全文索引通常会让存储膨胀至原始日志的 **3～5 倍**；而 Loki 只对少量标签建索引，日志正文经 **Snappy / zstd 压缩**后存入对象存储（S3、GCS、MinIO），实测压缩比可达 **5～10:1**。

以日均 1TB 原始日志为例：

```
原始日志：1 TB/day
Loki 压缩存储：≈ 100～200 GB/day  ✓
ES 同等场景：≈ 3～5 TB/day       ✗
```

### 2. 与 Prometheus 标签体系天然融合

Loki 的查询语言 **LogQL** 与 PromQL 高度一致，同样支持标签过滤、聚合运算：

```logql
# 统计 production namespace 下 error 日志的速率
sum(rate({namespace="production", level="error"}[5m])) by (app)
```

在 Grafana 面板中，同一行可混排 Metrics 图表与 Logs 面板，且跳转时**自动继承时间范围与标签上下文**，排障效率大幅提升。

### 3. 微服务架构，按需弹性

Loki 采用组件化架构，各角色可独立扩缩容：

```
Distributor  →  Ingester  →  Compactor
                              ↕
                         Object Storage
Query Frontend → Querier
```

在 K8s 中，可针对流量瓶颈单独对 Ingester 或 Querier 进行 HPA，无需整体扩容。

### 4. 多租户原生支持

通过 `X-Scope-OrgID` Header 实现天然的租户隔离，适合平台团队为多个业务方提供统一日志服务，配合 Grafana 的 Team/Org 体系实现精细化访问控制。

### 5. 运维复杂度低

相比 EFK（Elasticsearch + Fluentd + Kibana）：
- **无需管理 Shard、Replica、Index Template**
- **无需 JVM 调优**（Loki 是 Go 编写，内存行为可预期）
- 对象存储天然具备高可用与持久化，无需额外的存储集群

---

## 架构全景：K8s 日均 1TB 日志场景

```
┌─────────────────────────────────────────────────────────────┐
│                      Kubernetes Cluster                      │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  Node 1  │  │  Node 2  │  │  Node 3  │  │  Node N  │   │
│  │Promtail  │  │Promtail  │  │Promtail  │  │Promtail  │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
│       └──────────────┴─────────────┴──────────────┘        │
│                             │                               │
│                    ┌────────▼────────┐                      │
│                    │   Distributor   │  ← 接收 & 路由       │
│                    │   (3 replicas)  │                      │
│                    └────────┬────────┘                      │
│              ┌──────────────┼──────────────┐                │
│     ┌────────▼───┐  ┌───────▼────┐  ┌─────▼──────┐        │
│     │ Ingester-0 │  │ Ingester-1 │  │ Ingester-2 │        │
│     │  (ring)    │  │  (ring)    │  │  (ring)    │        │
│     └────────┬───┘  └───────┬────┘  └─────┬──────┘        │
│              └──────────────┴──────────────┘                │
│                             │  flush                        │
│                    ┌────────▼────────┐                      │
│                    │    Compactor    │                      │
│                    └────────┬────────┘                      │
└─────────────────────────────┼───────────────────────────────┘
                              │
                    ┌─────────▼──────────┐
                    │   Object Storage   │
                    │  (S3 / MinIO / GCS)│
                    └─────────┬──────────┘
                              │
              ┌───────────────┴──────────────┐
              │                              │
     ┌────────▼────────┐          ┌──────────▼──────┐
     │  Query Frontend │          │     Ruler        │
     │  (2 replicas)   │          │  (Alert Rules)   │
     └────────┬────────┘          └─────────────────-┘
              │
     ┌────────▼────────┐
     │    Querier       │
     │  (3 replicas)   │
     └────────┬────────┘
              │
     ┌────────▼────────┐
     │    Grafana       │
     └─────────────────┘
```

### 容量估算（日均 1TB 原始日志）

| 指标 | 数值 |
|---|---|
| 原始日志总量 | 1 TB/day |
| 压缩后存储 | ~150 GB/day |
| 峰值写入速率 | ~150 MB/s（假设早高峰 3x 均值） |
| Ingester 内存缓冲 | 每个实例建议 16～32 GB |
| Ingester 推荐副本数 | **6～8 个**（含 replication_factor=3） |
| 对象存储月均成本（S3 标准） | ~4,500 GB × $0.023 ≈ **$103/月** |

---

## Ingestor 横向扩展

### Ingester 的工作原理

Ingester 是 Loki 写路径上的核心组件，负责：

1. **接收** Distributor 路由过来的日志流
2. **缓存** 日志块（Chunks）到内存中的 WAL（Write-Ahead Log）
3. **周期性 Flush** 压缩后的 Chunks 到对象存储
4. **响应** Querier 对"热数据"（未 flush 部分）的查询请求

Ingester 之间通过 **Memberlist（Gossip 协议）** 形成一致性哈希环（Ring），每条日志流根据其标签集的 fingerprint 被路由到环上的特定节点。

### 扩展的核心挑战

Ingester 是**有状态服务**，扩缩容时需要优雅处理：

- **数据不丢失**：新节点加入时，已有 Chunks 不能丢弃
- **查询一致性**：查询时需覆盖所有持有该流数据的 Ingester
- **Flush 时序**：节点下线前必须将内存数据 flush 到对象存储

### 扩容流程（K8s StatefulSet）

```bash
# 1. 修改 StatefulSet replicas（以 Helm values 为例）
# ingester.replicas: 6 → 8

# 2. Loki 的 Ring 会自动感知新节点，Gossip 完成后开始分配流量
# 观察 Ring 状态
kubectl exec -n logging loki-ingester-0 -- \
  curl -s http://localhost:3100/ring | jq '.[] | select(.state != "ACTIVE")'

# 3. 验证新节点已 ACTIVE
kubectl logs -n logging loki-ingester-6 | grep "joined ring"

# 4. 缩容时，触发 /ingester/shutdown 让节点优雅退出
kubectl exec -n logging loki-ingester-7 -- \
  curl -X POST http://localhost:3100/ingester/shutdown
```

### WAL（Write-Ahead Log）配置要点

启用 WAL 后，Ingester 重启时可从本地磁盘恢复未 flush 的数据，是生产环境的强制要求：

```yaml
# Loki ingester WAL 配置
ingester:
  wal:
    enabled: true
    dir: /loki/wal          # 需挂载 PVC
    flush_on_shutdown: true # 下线时强制 flush
    replay_memory_ceiling: 4GB
  lifecycler:
    ring:
      replication_factor: 3  # 每条流写入 3 个 Ingester
    final_sleep: 30s          # 等待 Ring 收敛
  chunk_idle_period: 30m
  chunk_retain_period: 1m
  max_chunk_age: 2h
  chunk_target_size: 1572864  # 1.5MB，平衡压缩率与查询性能
```

### HPA 配置（基于自定义指标）

Ingester 建议使用**手动扩缩容**（根据写入速率预估），而非 CPU HPA，因为 CPU 并不能准确反映 Chunks 的内存压力。推荐基于 `loki_ingester_chunks_created_total` 速率来触发：

```yaml
# 使用 KEDA 基于 Prometheus 指标扩容 Ingester
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: loki-ingester-scaler
  namespace: logging
spec:
  scaleTargetRef:
    name: loki-ingester
  minReplicaCount: 6
  maxReplicaCount: 16
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus:9090
      metricName: loki_ingester_chunks_per_second
      threshold: "500"        # 每个实例超过 500 chunks/s 时触发扩容
      query: |
        sum(rate(loki_ingester_chunks_created_total[2m])) /
        count(up{job="loki-ingester"})
```

---

## 标签设计（Label Design）

标签是 Loki 的核心设计哲学，**标签设计的好坏直接决定查询性能与存储成本**。

### 黄金法则：Low Cardinality（低基数）

Loki 为每个唯一的标签组合（Label Set）创建一个独立的**日志流（Stream）**，流的数量直接影响：
- Ingester 内存占用
- 对象存储文件数量
- 查询扫描的 Chunk 数

**标签基数（Cardinality）控制目标**：

| 场景 | 流数量 | 评估 |
|---|---|---|
| < 10,000 流 | 健康 | ✅ |
| 10,000 ～ 100,000 流 | 警惕 | ⚠️ |
| > 100,000 流 | 危险 | ❌ |

### K8s 生产环境推荐标签集

```logql
# 推荐的标签维度（固定在 Promtail pipeline 中提取）
{
  cluster    = "prod-cn-north",  # 集群标识，低基数
  namespace  = "payment",        # K8s namespace
  app        = "order-service",  # 来自 Pod label: app.kubernetes.io/name
  env        = "production",     # 环境标识
  node       = "node-001"        # 节点名（可选，按需添加）
}
```

### ❌ 反模式：高基数标签

```yaml
# 绝对禁止放入标签的字段
- pod_name     # Pod 每次重建名字不同，基数爆炸
- request_id   # UUID，基数 = 请求总数
- user_id      # 用户量级，基数不可控
- trace_id     # 应放在日志正文，通过 LogQL 过滤
- ip_address   # IP 基数高，且动态变化
- timestamp    # 绝对禁止，Loki 内部管理时间
```

### ✅ 正确做法：结构化正文 + 标签过滤

```bash
# 日志正文（JSON 格式）包含高基数字段，通过 LogQL 的 line_format 过滤
{namespace="payment", app="order-service"} 
  | json 
  | trace_id="abc-123-xyz"
  | line_format "{{.level}} {{.msg}}"
```

### 标签提取 Pipeline（Promtail 配置）

```yaml
# Promtail pipeline stages：从 K8s Pod 日志中提取标准标签
pipeline_stages:
  # Stage 1：解析 Docker/containerd JSON 日志格式
  - cri: {}

  # Stage 2：从日志正文解析 JSON（适合结构化日志应用）
  - json:
      expressions:
        level:   level
        msg:     message
        traceId: trace_id
        service: service_name

  # Stage 3：提升关键字段为标签（仅低基数字段）
  - labels:
      level:   ""   # error/warn/info/debug，基数 ≤ 4
      # 注意：traceId 不提升为标签！

  # Stage 4：丢弃 debug 日志（生产降噪）
  - drop:
      expression: '.*level="debug".*'
      drop_counter_reason: debug_log_dropped

  # Stage 5：匹配异常日志，添加额外标签（用于告警）
  - match:
      selector: '{level="error"}'
      stages:
        - labels:
            error_type: ""
```

### 标签命名规范（团队标准化）

```
格式：snake_case，全小写，使用下划线
禁止：大写字母、连字符（会导致 LogQL 引用时需加引号）

推荐标签清单：
  cluster        集群名称       prod-us-east / staging-cn
  namespace      K8s namespace  payment / user-service
  app            应用名         order-api（来自 k8s label）
  env            环境           production / staging / dev
  level          日志级别       error / warn / info
  component      子组件         grpc-server / worker / scheduler
```

---

## 完整配置示例

以下为基于 **Helm + values.yaml** 的完整生产配置，覆盖 Loki、Promtail 及相关 K8s 资源清单。

### 目录结构

```
loki-stack/
├── loki-values.yaml          # Loki 主配置
├── promtail-values.yaml      # Promtail Agent 配置
├── ingester-pvc.yaml         # Ingester WAL 持久卷
├── object-storage-secret.yaml # 对象存储凭证
└── grafana-datasource.yaml   # Grafana 数据源
```

---

### 1. 对象存储凭证（Secret）

```yaml
# object-storage-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: loki-s3-credentials
  namespace: logging
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: "YOUR_ACCESS_KEY"
  AWS_SECRET_ACCESS_KEY: "YOUR_SECRET_KEY"
```

---

### 2. Ingester WAL 持久卷（PVC）

```yaml
# ingester-pvc.yaml
# 为 StatefulSet volumeClaimTemplates 提供模板
# 在 loki-values.yaml 中引用，此处展示原始 YAML 结构
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: loki-wal-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

---

### 3. Loki 主配置（loki-values.yaml）

```yaml
# loki-values.yaml
# helm install loki grafana/loki -n logging -f loki-values.yaml

loki:
  # ── 认证 ──────────────────────────────────────────────
  auth_enabled: true  # 生产环境开启多租户

  # ── 服务器 ─────────────────────────────────────────────
  server:
    http_listen_port: 3100
    grpc_listen_port: 9095
    log_level: info
    grpc_server_max_recv_msg_size: 104857600  # 100MB
    grpc_server_max_send_msg_size: 104857600

  # ── 通用配置 ───────────────────────────────────────────
  common:
    path_prefix: /loki
    replication_factor: 3
    ring:
      kvstore:
        store: memberlist

  # ── Memberlist（Gossip Ring） ──────────────────────────
  memberlist:
    join_members:
      - loki-memberlist.logging.svc.cluster.local:7946
    gossip_interval: 2s
    gossip_nodes: 3
    retransmit_factor: 4

  # ── 摄入限制（全局 + 租户级可覆盖） ────────────────────
  limits_config:
    # 写入限制
    ingestion_rate_mb: 64            # 每租户 64MB/s 写入速率
    ingestion_burst_size_mb: 128     # 突发 128MB
    per_stream_rate_limit: 10MB      # 单流限速
    per_stream_rate_limit_burst: 20MB

    # 标签限制（防止基数爆炸）
    max_label_names_per_series: 20   # 每个流最多 20 个标签
    max_label_name_length: 1024
    max_label_value_length: 2048
    max_streams_per_user: 100000     # 每租户最多 10 万流

    # 查询限制
    max_query_series: 5000
    max_query_parallelism: 32
    max_entries_limit_per_query: 50000
    query_timeout: 5m

    # 保留策略（按租户）
    retention_period: 30d            # 默认保留 30 天

    # 乱序写入容忍（应对网络抖动）
    unordered_writes: true
    max_chunk_age: 2h

  # ── Ingester 配置 ──────────────────────────────────────
  ingester:
    wal:
      enabled: true
      dir: /loki/wal
      flush_on_shutdown: true
      replay_memory_ceiling: 6GB
    lifecycler:
      ring:
        replication_factor: 3
        heartbeat_timeout: 1m
      heartbeat_period: 5s
      join_after: 30s
      final_sleep: 30s
      num_tokens: 256              # Ring 上的虚拟节点数
    chunk_idle_period: 30m
    chunk_retain_period: 1m
    max_chunk_age: 2h
    chunk_target_size: 1572864     # 1.5MB
    chunk_encoding: snappy
    concurrent_flushes: 32
    max_transfer_retries: 0        # 使用 WAL 替代 transfer

  # ── Distributor 配置 ────────────────────────────────────
  distributor:
    ring:
      kvstore:
        store: memberlist

  # ── Compactor 配置 ──────────────────────────────────────
  compactor:
    working_directory: /loki/compactor
    shared_store: s3
    compaction_interval: 10m
    retention_enabled: true        # 开启自动过期删除
    retention_delete_delay: 2h
    retention_delete_worker_count: 150
    delete_request_cancel_period: 24h

  # ── 存储配置（S3）──────────────────────────────────────
  storage_config:
    aws:
      s3: s3://ap-northeast-1/loki-prod-chunks
      region: ap-northeast-1
      access_key_id: ${AWS_ACCESS_KEY_ID}
      secret_access_key: ${AWS_SECRET_ACCESS_KEY}
      s3forcepathstyle: false
      http_config:
        idle_conn_timeout: 90s
        response_header_timeout: 0
    boltdb_shipper:
      active_index_directory: /loki/index
      cache_location: /loki/index_cache
      cache_ttl: 24h
      shared_store: s3
    tsdb_shipper:                  # Loki 2.8+ 推荐使用 TSDB 索引
      active_index_directory: /loki/tsdb-index
      cache_location: /loki/tsdb-index-cache
      cache_ttl: 24h
      shared_store: s3

  # ── Schema 配置 ────────────────────────────────────────
  schema_config:
    configs:
      - from: "2024-01-01"
        store: tsdb              # 使用 TSDB 索引（性能更优）
        object_store: s3
        schema: v12
        index:
          prefix: loki_prod_index_
          period: 24h            # 每天一个索引文件

  # ── Query Frontend 配置 ────────────────────────────────
  query_range:
    align_queries_with_step: true
    max_retries: 5
    cache_results: true
    results_cache:
      cache:
        redis:
          endpoint: redis:6379
          expiration: 1h

  # ── Querier 配置 ───────────────────────────────────────
  querier:
    max_concurrent: 20
    query_ingesters_within: 3h   # 3h 内数据优先查 Ingester

  # ── Chunk 缓存（读加速）────────────────────────────────
  chunk_store_config:
    max_look_back_period: 0s
    chunk_cache_config:
      redis:
        endpoint: redis:6379
        expiration: 2h

  # ── 前缀路由（多组件部署）────────────────────────────────
  frontend:
    log_queries_longer_than: 5s
    compress_responses: true
    max_outstanding_per_tenant: 2048

# ── K8s 资源配置 ─────────────────────────────────────────

# Ingester：有状态服务，StatefulSet
ingester:
  replicas: 6                    # 日均 1TB 场景推荐 6 起步
  maxUnavailable: 1
  persistence:
    enabled: true
    storageClass: loki-wal-ssd
    size: 50Gi                   # WAL 存储，按 2h 峰值写入估算
  resources:
    requests:
      cpu: 2
      memory: 16Gi
    limits:
      cpu: 4
      memory: 24Gi
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app.kubernetes.io/component: ingester
          topologyKey: kubernetes.io/hostname  # 禁止同节点部署
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "loki"
      effect: "NoSchedule"

# Distributor：无状态，Deployment
distributor:
  replicas: 3
  resources:
    requests:
      cpu: 1
      memory: 2Gi
    limits:
      cpu: 2
      memory: 4Gi
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 10
    targetCPUUtilizationPercentage: 60

# Query Frontend：无状态
queryFrontend:
  replicas: 2
  resources:
    requests:
      cpu: 1
      memory: 2Gi
    limits:
      cpu: 2
      memory: 4Gi

# Querier：无状态
querier:
  replicas: 3
  resources:
    requests:
      cpu: 2
      memory: 8Gi
    limits:
      cpu: 4
      memory: 16Gi
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 12
    targetCPUUtilizationPercentage: 70

# Compactor：单实例（Leader 选举）
compactor:
  replicas: 1
  resources:
    requests:
      cpu: 500m
      memory: 2Gi
    limits:
      cpu: 2
      memory: 4Gi

# Ruler
ruler:
  enabled: true
  replicas: 2
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 1
      memory: 2Gi

# Redis 缓存
redis:
  enabled: true
  auth:
    enabled: true
    password: "YOUR_REDIS_PASSWORD"
  master:
    resources:
      requests:
        cpu: 500m
        memory: 4Gi
```

---

### 4. Promtail 配置（promtail-values.yaml）

```yaml
# promtail-values.yaml
# helm install promtail grafana/promtail -n logging -f promtail-values.yaml

config:
  # Loki 推送地址
  clients:
    - url: http://loki-distributor.logging.svc.cluster.local:3100/loki/api/v1/push
      tenant_id: production          # X-Scope-OrgID，对应租户
      batchwait: 1s
      batchsize: 1048576             # 1MB 批量推送
      timeout: 10s
      backoff_config:
        min_period: 500ms
        max_period: 5m
        max_retries: 10
      external_labels:
        cluster: prod-cn-north       # 注入集群标签（所有日志统一打）

  # 采集目标
  scrape_configs:
    # ── 采集所有 K8s Pod 日志 ─────────────────────────────
    - job_name: kubernetes-pods
      kubernetes_sd_configs:
        - role: pod
      pipeline_stages:
        # Stage 1：解析 CRI 格式（containerd）
        - cri: {}

        # Stage 2：尝试 JSON 解析（结构化日志）
        - json:
            expressions:
              log_level: level
              log_msg:   message
              trace_id:  traceId

        # Stage 3：非 JSON 日志的正则兜底（传统日志）
        - regex:
            expression: >-
              (?P<log_level>ERROR|WARN|INFO|DEBUG|TRACE|error|warn|info|debug)

        # Stage 4：标准化 level 值
        - template:
            source: log_level
            template: '{{ ToLower .Value }}'

        # Stage 5：提升低基数字段为标签
        - labels:
            level: log_level

        # Stage 6：丢弃 debug 日志（可按 namespace 选择性保留）
        - match:
            selector: '{namespace!~"platform|infra"}'
            stages:
              - drop:
                  expression: '(?i)level=debug'
                  drop_counter_reason: debug_dropped_prod

        # Stage 7：从 K8s metadata 提取并注入标签
        - labeldrop:
            - filename                 # 去除冗余的文件路径标签

      relabel_configs:
        # 只采集 Running 状态的 Pod
        - source_labels: [__meta_kubernetes_pod_phase]
          action: keep
          regex: Running

        # 排除 logging namespace 自身（避免循环采集）
        - source_labels: [__meta_kubernetes_namespace]
          action: drop
          regex: logging

        # 提取关键 K8s 元数据作为标签
        - source_labels: [__meta_kubernetes_namespace]
          target_label: namespace
        - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_name]
          target_label: app
        - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_component]
          target_label: component
        - source_labels: [__meta_kubernetes_pod_node_name]
          target_label: node
        - source_labels: [__meta_kubernetes_pod_container_name]
          target_label: container

        # 从 annotation 读取租户 ID（多租户路由）
        - source_labels: [__meta_kubernetes_pod_annotation_loki_tenant]
          target_label: __tenant_id__

# DaemonSet 配置（每个节点部署一个 Promtail）
daemonset:
  resources:
    requests:
      cpu: 200m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
  tolerations:
    - operator: Exists               # 采集所有节点，含 master/taint 节点
  volumeMounts:
    - name: pods-logs
      mountPath: /var/log/pods
      readOnly: true
    - name: containers-logs
      mountPath: /var/lib/docker/containers
      readOnly: true
  volumes:
    - name: pods-logs
      hostPath:
        path: /var/log/pods
    - name: containers-logs
      hostPath:
        path: /var/lib/docker/containers
```

---

### 5. 告警规则（LogQL Rules）

```yaml
# loki-rules-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: loki-alert-rules
  namespace: logging
data:
  production-rules.yaml: |
    groups:
      - name: production.critical
        interval: 1m
        rules:
          # 错误日志速率告警
          - alert: HighErrorLogRate
            expr: |
              sum(rate({namespace=~"payment|order|user", level="error"}[5m])) by (namespace, app)
              > 10
            for: 2m
            labels:
              severity: warning
              team: backend
            annotations:
              summary: "{{ $labels.app }} 错误日志速率过高"
              description: |
                应用 {{ $labels.app }}（namespace: {{ $labels.namespace }}）
                过去 5 分钟错误日志速率为 {{ $value | humanize }} 条/秒，超过阈值 10 条/秒。

          # OOM 检测
          - alert: PodOOMKilled
            expr: |
              count_over_time(
                {namespace=~".+"}
                |= "OOMKilled"
                [5m]
              ) > 0
            labels:
              severity: critical
            annotations:
              summary: "检测到 OOMKilled 事件"

          # 日志写入速率突降（可能是采集断路）
          - alert: LokiIngestionDropped
            expr: |
              sum(rate(loki_distributor_lines_received_total[5m]))
              < sum(rate(loki_distributor_lines_received_total[5m] offset 30m)) * 0.5
            for: 5m
            labels:
              severity: critical
              team: platform
            annotations:
              summary: "Loki 日志写入速率骤降超过 50%"

      - name: loki.infrastructure
        interval: 2m
        rules:
          # Ingester 流数量预警
          - alert: LokiHighStreamCount
            expr: |
              sum(loki_ingester_streams_created_total) by (pod)
              > 50000
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "Ingester {{ $labels.pod }} 流数量过高，请检查标签基数"

          # Chunk Flush 积压
          - alert: LokiFlushQueueDepth
            expr: |
              loki_ingester_memory_chunks > 50000
            for: 10m
            labels:
              severity: warning
            annotations:
              summary: "Ingester 内存 Chunks 积压，Flush 速率可能不足"
```

---

### 6. 安装命令

```bash
# 0. 准备 namespace 和 secret
kubectl create namespace logging
kubectl apply -f object-storage-secret.yaml -n logging
kubectl apply -f ingester-pvc.yaml          # StorageClass

# 1. 添加 Grafana Helm 仓库
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# 2. 安装 Loki（分布式模式）
helm install loki grafana/loki \
  --namespace logging \
  --version 6.x.x \
  -f loki-values.yaml \
  --set loki.storage.s3.accessKeyId="${AWS_ACCESS_KEY_ID}" \
  --set loki.storage.s3.secretAccessKey="${AWS_SECRET_ACCESS_KEY}"

# 3. 安装 Promtail DaemonSet
helm install promtail grafana/promtail \
  --namespace logging \
  -f promtail-values.yaml

# 4. 应用告警规则
kubectl apply -f loki-rules-configmap.yaml -n logging

# 5. 验证部署
kubectl get pods -n logging
kubectl exec -n logging deploy/loki-distributor -- \
  curl -s http://localhost:3100/ready
```

---

### 7. Grafana 数据源配置

```yaml
# grafana-datasource.yaml（Grafana Provisioning）
apiVersion: 1
datasources:
  - name: Loki-Production
    type: loki
    access: proxy
    url: http://loki-query-frontend.logging.svc.cluster.local:3100
    jsonData:
      maxLines: 5000
      timeout: 300
      httpHeaderName1: "X-Scope-OrgID"
    secureJsonData:
      httpHeaderValue1: "production"
    isDefault: false
    version: 1
    editable: true
```

---

## 运维排查速查

### 常用 LogQL

```logql
# 查看指定服务最近 100 条 error 日志
{namespace="payment", app="order-api", level="error"} | limit 100

# 按 app 统计 5 分钟内的错误速率
sum(rate({namespace="payment", level="error"}[5m])) by (app)

# 解析 JSON 日志并按 trace_id 追踪
{namespace="payment", app="order-api"}
  | json
  | trace_id="3fa2c1d4-8b7e-11ef-a3d2-0242ac120002"

# 统计各服务日志量占比（容量规划）
sum(bytes_over_time({cluster="prod-cn-north"}[24h])) by (namespace)
  / sum(bytes_over_time({cluster="prod-cn-north"}[24h]))

# 检查当前活跃流数量（评估基数健康度）
count(count_over_time({cluster="prod-cn-north"}[5m])) by (namespace)
```

### Ingester Ring 健康检查

```bash
# 查看 Ring 状态（所有节点应为 ACTIVE）
curl -s http://loki-ingester:3100/ring | python3 -m json.tool

# 查看 Ingester 指标
curl -s http://loki-ingester:3100/metrics | grep -E \
  "loki_ingester_(streams|chunks|memory|flush)"

# 手动触发 Flush（维护前）
curl -X POST http://loki-ingester-0:3100/flush
```

---

## 小结

| 关键决策 | 推荐方案 |
|---|---|
| 存储后端 | S3 / GCS（成本低，免运维） |
| 索引类型 | TSDB（Loki 2.8+） |
| 采集 Agent | Promtail（或 Grafana Alloy） |
| Ingester 扩容 | StatefulSet 手动 + KEDA 辅助 |
| 标签数量 | ≤ 8 个，全部低基数 |
| 日志格式 | 结构化 JSON，正文含高基数字段 |
| 保留策略 | 热数据 7 天（SSD），冷数据 30 天（S3） |
| 缓存层 | Redis（Query 结果 + Chunk 缓存） |

Loki Stack 最大的价值不在于「功能最强」，而在于**在 Prometheus 生态下以最低的运维成本换取足够好的可观测能力**。对于日均 1TB 的 K8s 生产场景，本文给出的配置已在多个生产集群验证，可作为团队标准化基线直接落地。
