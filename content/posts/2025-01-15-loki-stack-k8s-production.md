---
title: "Loki Stack 生产实践：K8s 环境下日均 1T 日志的高可用架构"
date: 2025-01-15T10:00:00+08:00
lastmod: 2025-01-15T10:00:00+08:00
draft: false
tags: ["loki", "kubernetes", "observability", "logging", "devops", "grafana", "fluent-bit"]
categories: ["可观测性", "运维架构"]
author: "Platform Engineering"
description: "深入剖析 Loki Stack 在 Kubernetes 生产环境中的落地实践，涵盖 Ingester Ring 原理、横向扩展策略、Fluent Bit 采集配置与日均 1TB 日志场景的完整方案。"
summary: "从命名由来出发，系统介绍 Loki Stack 的核心优势，深入讲解 Ingester Ring 一致性哈希机制与副本写入原理，以 K8s 生产集群日均 1TB 日志收集为例，详细说明 Fluent Bit 替代 Promtail 的原因与完整配置方案。"
toc: true
weight: 1
---

## 为什么叫 Loki Stack？

Loki 的命名来自北欧神话中的「洛基」——奥丁的义兄，一位善于变形、以诡计著称的神灵。Grafana 团队选用这个名字，有意借用了洛基的两个核心特质：

**轻盈变形**：Loki 不像 Elasticsearch 那样对日志内容做全量索引，它只对**标签（Labels）**建立索引，日志正文以压缩块的形式原始存储。这正是洛基「变形」的隐喻——表面轻量，内藏乾坤。

**狡黠高效**：通过放弃全文索引换取极低的存储和查询开销，在资源紧张的 K8s 环境里游刃有余。

**Loki Stack** 则是围绕 Loki 构建的完整可观测链路组合：

| 组件 | 角色 | 类比 Prometheus 生态 |
|---|---|---|
| **Fluent Bit** | 日志采集 Agent | Node Exporter |
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
原始日志：       1,000 GB/day
Loki 压缩存储：  ≈ 100～200 GB/day  ✅
ES 同等场景：    ≈ 3,000～5,000 GB/day  ❌
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
Query Frontend  →  Querier
```

在 K8s 中，可针对流量瓶颈单独对 Ingester 或 Querier 进行扩容，无需整体扩容。

### 4. 多租户原生支持

通过 `X-Scope-OrgID` Header 实现天然的租户隔离，适合平台团队为多个业务方提供统一日志服务，配合 Grafana 的 Team/Org 体系实现精细化访问控制。

### 5. 运维复杂度低

相比 EFK（Elasticsearch + Fluentd + Kibana）：

- 无需管理 Shard、Replica、Index Template
- 无需 JVM 调优（Loki 是 Go 编写，内存行为可预期）
- 对象存储天然具备高可用与持久化，无需额外的存储集群

---

## 架构全景：K8s 日均 1TB 日志场景

```
┌─────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                        │
│                                                                  │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐    │
│  │  Node 1  │   │  Node 2  │   │  Node 3  │   │  Node N  │    │
│  │Fluent Bit│   │Fluent Bit│   │Fluent Bit│   │Fluent Bit│    │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘   └────┬─────┘    │
│       └───────────────┴──────────────┴───────────────┘         │
│                              │                                   │
│                     ┌────────▼────────┐                         │
│                     │   Distributor   │  ← 接收 & 路由          │
│                     │   (3 replicas)  │                         │
│                     └────────┬────────┘                         │
│               ┌──────────────┼──────────────┐                   │
│      ┌────────▼───┐  ┌───────▼────┐  ┌──────▼─────┐           │
│      │ Ingester-0 │  │ Ingester-1 │  │ Ingester-2 │           │
│      │ [Ring 节点]│  │ [Ring 节点]│  │ [Ring 节点]│           │
│      └────────┬───┘  └───────┬────┘  └──────┬─────┘           │
│               └──────────────┴──────────────┘                   │
│                              │  flush                            │
│                     ┌────────▼────────┐                         │
│                     │    Compactor    │                         │
│                     └────────┬────────┘                         │
└──────────────────────────────┼──────────────────────────────────┘
                               │
                     ┌─────────▼──────────┐
                     │   Object Storage   │
                     │  (S3 / MinIO / GCS)│
                     └─────────┬──────────┘
                               │
               ┌───────────────┴──────────────┐
               │                              │
      ┌────────▼────────┐          ┌──────────▼──────┐
      │  Query Frontend │          │      Ruler       │
      │  (2 replicas)   │          │   (Alert Rules)  │
      └────────┬────────┘          └──────────────────┘
               │
      ┌────────▼────────┐
      │     Querier      │
      │  (3 replicas)   │
      └────────┬────────┘
               │
      ┌────────▼────────┐
      │     Grafana      │
      └─────────────────┘
```

### 容量估算（日均 1TB 原始日志）

| 指标 | 数值 |
|---|---|
| 原始日志总量 | 1 TB/day |
| 压缩后存储 | ~150 GB/day |
| 峰值写入速率 | ~150 MB/s（早高峰 3x 均值） |
| Ingester 推荐内存 | 每实例 16～24 GB |
| Ingester 推荐副本数 | **6～8 个**（replication_factor=3） |
| 对象存储月均成本（S3） | ~4,500 GB × $0.023 ≈ **$103/月** |

---

## Ingester Ring 深度解析

Ingester 是 Loki 写路径上最核心的组件，也是最需要理解其内部机制的部分。理解了 Ring，才能真正理解为什么要多副本、要大内存。

### Ring 是什么？

Ring 本质是一个**一致性哈希环**，所有 Ingester 节点启动时会通过 **Memberlist（Gossip 协议）** 互相发现，并在这个虚拟环上"占位"，每个节点持有若干个 **Token**（虚拟位置点，默认 256 个）。

```
                      Token 0°
                         │
           ┌─────────────┴─────────────┐
      270° │                           │ 90°
           │                           │
Ingester-2 │          Ring             │ Ingester-0
(token:    │                           │ (token:
 210,270,  │                           │  30, 90,
   330°)   │                           │  150°)
           └─────────────┬─────────────┘
                         │
                       180°
                    Ingester-1
                 (token: 60,120,
                   180,240,300°)
```

当一条日志流进来，Distributor 对这条流的**标签集做哈希**，得到一个 0～360° 的值，然后**顺时针找到第一个 Token**，那个 Ingester 就是这条流的写入起点。

### 副本不是备份，是并发写入

`replication_factor: 3` 的含义是：**每条日志流同时写入 3 个 Ingester**，而不是写 1 个再异步复制。3 个节点完全对等，没有主从之分。

```
                     Distributor
                          │
           ┌──────────────┼──────────────┐
           │              │              │
           ▼              ▼              ▼
      Ingester-0     Ingester-1     Ingester-2
      (节点 1/3)     (节点 2/3)     (节点 3/3)
           │              │              │
           └──────────────┴──────────────┘
             同一条日志流，3 份同时并发写入
             每个节点都持有完整的这条流数据
```

**为什么要这么设计？**

Ingester 是内存服务，数据在 flush 到 S3 之前都活在内存里。如果只写 1 个节点，节点一旦崩溃，最近 30 分钟内未 flush 的日志**永久丢失**，对生产环境不可接受。3 份并发写入，挂 1 个还有 2 份，满足生产可靠性要求。

### Quorum 写入机制

Distributor 并发写 3 个节点，但**不需要等全部成功**，只需要 **Quorum（法定人数）** 响应即可：

```
Quorum = floor(replication_factor / 2) + 1
       = floor(3 / 2) + 1
       = 2

写入流程：
Distributor ──并发写──▶ Ingester-0  ✅ 响应成功
                    ──▶ Ingester-1  ✅ 响应成功   → 2/3 满足 Quorum → 立即返回客户端成功
                    ──▶ Ingester-2  ⏳ 响应慢...  → 后台继续等待，不影响写入响应
```

### 查询时如何处理 3 份数据？

Querier 访问 3 个节点后按**纳秒时间戳 + 内容哈希**做唯一键去重，用户看到的结果不会有重复条目：

```
Querier 收到查询请求
    │
    ├─ 确定这条流在哪 3 个 Ingester 上（查 Ring）
    ├─ 并发请求这 3 个节点
    ├─ 收到 3 份数据（内容相同，有重复）
    └─ 按 [timestamp + hash(log_line)] 去重 → 返回唯一结果
```

### 节点故障时的行为

```
正常：RF=3，全部 ACTIVE
┌──────────┐  ┌──────────┐  ┌──────────┐
│Ingester-0│  │Ingester-1│  │Ingester-2│
│  ACTIVE  │  │  ACTIVE  │  │  ACTIVE  │
└──────────┘  └──────────┘  └──────────┘
写入：2/3 Quorum ✅    查询：3 个节点去重 ✅

──────────────────────────────────────────

Ingester-1 挂掉：
┌──────────┐  ┌──────────┐  ┌──────────┐
│Ingester-0│  │Ingester-1│  │Ingester-2│
│  ACTIVE  │  │   DOWN   │  │  ACTIVE  │
└──────────┘  └──────────┘  └──────────┘
写入：2/3 仍满足 Quorum ✅
查询：2 个节点仍有完整数据 ✅
（集群降级运行，服务不中断）

──────────────────────────────────────────

Ingester-1 + Ingester-2 同时挂掉：
写入：1/3 不满足 Quorum ❌ → 写入失败
查询：部分数据缺失 ❌
```

RF=3 能容忍 **1 个节点故障**不影响读写，这是生产环境的最低要求。

### 为什么 Ingester 要大内存？

每条活跃日志流在 Ingester 内存中的结构如下：

```
Stream {app="order-api", namespace="payment", level="error"}
    │
    └── Chunk（压缩块，目标大小 1.5MB）
          ├── 日志条目 [timestamp, log line]
          ├── 日志条目 [timestamp, log line]
          ├── ... （持续追加）
          └── 攒满 1.5MB 或超过 max_chunk_age=2h → flush 到 S3
```

内存占用 = 活跃流数量 × 每个 Chunk 平均大小。日均 1TB 场景下：

```
50,000 条活跃流 × 256KB/Chunk ≈ 12.5 GB（仅 Chunk 数据）
+ WAL 缓冲 + Go runtime + 索引结构 ≈ 16～20 GB 实际占用
```

**内存不足时最隐蔽的症状**不是 OOM，而是 Ingester 被迫**提前 flush 小 Chunk**：

```
内存充足：攒满 1.5MB 再 flush  → 压缩率高，S3 小文件少，查询快 ✅
内存不足：256KB 就 flush       → 小文件爆炸，查询需扫描 6x 文件 → 查询变慢 ❌
```

监控这个指标可以提前发现内存压力：

```bash
# Chunk 平均大小，健康值应接近 chunk_target_size（1.5MB）
# 持续低于 500KB 说明内存压力触发了提前 flush
loki_ingester_chunk_size_bytes_sum / loki_ingester_chunk_size_bytes_count
```

### 为什么多个小节点优于少量大节点？

这是最常见的误判——"给单个 Ingester 升到 64GB 是不是比 6 个 16GB 更稳？"

**不是，理由有三：**

**故障爆炸半径不同：**
```
方案 A：3 个 Ingester × 32GB
  → 1 个挂掉，损失 1/3 容量
  → 剩余 2 个承压可能连锁 OOM ← 危险

方案 B：6 个 Ingester × 16GB（推荐）
  → 1 个挂掉，损失 1/6 容量
  → 剩余 5 个轻松吸收 ← 稳定
```

**flush 恢复时间成正比：** 节点越大，重启时 WAL 回放时间越长，影响写入的窗口越大。

**Ring 分布更均匀：** 节点越多，Token 越密集，单节点被集中分配高写入量流的概率越低，避免热点。

**各场景推荐配置：**

| 日均写入量 | Ingester 配置 | 总内存 |
|---|---|---|
| < 100 GB/day | 3 × 8GB（RF=3） | 24 GB |
| 100G～500G/day | 4 × 16GB（RF=3） | 64 GB |
| 500G～1T/day | 6 × 16GB（RF=3） | 96 GB |
| > 1T/day | 8～10 × 16GB（RF=3） | 128～160 GB |

---

## 为什么用 Fluent Bit 替代 Promtail？

K8s 社区已经全面转向 Fluent Bit 作为标准日志采集 Agent，Promtail 逐渐退出主力位置。

| 对比维度 | Promtail | Fluent Bit |
|---|---|---|
| 定位 | 专为 Loki 设计 | 通用日志采集器 |
| 实现语言 | Go | **C** |
| 内存占用 | ~50MB | **~5MB** |
| 输出目标 | 只能到 Loki | Loki / ES / Kafka / S3 / 几十种 |
| 多目标同时输出 | ❌ | ✅ |
| K8s metadata 注入 | 基础 | 原生支持，功能更强 |
| CNCF 状态 | 非 CNCF | **CNCF 毕业项目** |
| 社区活跃度 | 进入维护模式 | 非常活跃 |

**最关键的差异**：生产环境通常需要日志同时打到 Loki（实时查询）和 Kafka（审计归档）或 ES（全文检索），Fluent Bit 一个 DaemonSet 多路输出直接搞定，Promtail 做不到。

---

## 标签设计（Label Design）

标签是 Loki 的核心设计哲学，**标签设计的好坏直接决定 Ring 上的流分布、Ingester 内存压力与查询性能**。

### 黄金法则：Low Cardinality（低基数）

Loki 为每个唯一的标签组合（Label Set）创建一个独立的**日志流（Stream）**，流的数量直接影响 Ingester 内存占用、对象存储文件数量和查询扫描范围。

| 流数量 | 评估 |
|---|---|
| < 10,000 | 健康 ✅ |
| 10,000 ～ 100,000 | 需要关注 ⚠️ |
| > 100,000 | 危险，Ingester 可能 OOM ❌ |

### K8s 生产环境推荐标签集

```logql
{
  cluster   = "prod-cn-north",   # 集群标识，极低基数
  namespace = "payment",         # K8s namespace
  app       = "order-service",   # 来自 Pod label: app.kubernetes.io/name
  env       = "production",      # 环境标识
  level     = "error"            # 日志级别，基数 ≤ 4
}
```

### 绝对禁止放入标签的字段

```yaml
- pod_name    # Pod 每次重建名字不同，基数爆炸
- request_id  # UUID，基数 = 请求总数
- user_id     # 用户量级，基数不可控
- trace_id    # 放在日志正文，通过 LogQL 过滤
- ip_address  # IP 动态变化，基数高
- version     # 每次发版都变（app.kubernetes.io/version）
- pod_hash    # pod-template-hash，每个 RS 不同
```

### 正确做法：结构化正文 + LogQL 过滤

```bash
# 高基数字段放在日志正文（JSON），通过 LogQL 精准过滤
{namespace="payment", app="order-service"}
  | json
  | trace_id="3fa2c1d4-8b7e-11ef-a3d2-0242ac120002"
  | line_format "{{.level}} {{.msg}}"
```

---

## 完整配置示例

### 目录结构

```
loki-stack/
├── fluent-bit-configmap.yaml     # Fluent Bit 采集配置
├── fluent-bit-daemonset.yaml     # Fluent Bit DaemonSet + RBAC
├── loki-values.yaml              # Loki Helm values
├── ingester-storageclass.yaml    # Ingester WAL 存储类
├── object-storage-secret.yaml    # 对象存储凭证
├── loki-rules-configmap.yaml     # LogQL 告警规则
└── grafana-datasource.yaml       # Grafana 数据源
```

---

### 1. 对象存储凭证

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

### 2. Ingester WAL 存储类

```yaml
# ingester-storageclass.yaml
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

### 3. Loki Helm Values

```yaml
# loki-values.yaml
# helm install loki grafana/loki -n logging -f loki-values.yaml

loki:
  # ── 认证 ────────────────────────────────────────────────
  auth_enabled: true             # 生产环境开启多租户

  # ── 服务器 ──────────────────────────────────────────────
  server:
    http_listen_port: 3100
    grpc_listen_port: 9095
    log_level: info
    grpc_server_max_recv_msg_size: 104857600  # 100MB
    grpc_server_max_send_msg_size: 104857600

  # ── Memberlist（Gossip Ring） ────────────────────────────
  memberlist:
    join_members:
      - loki-memberlist.logging.svc.cluster.local:7946
    gossip_interval: 2s
    gossip_nodes: 3
    retransmit_factor: 4

  # ── 通用配置 ─────────────────────────────────────────────
  common:
    path_prefix: /loki
    replication_factor: 3        # 每条流写入 3 个 Ingester
    ring:
      kvstore:
        store: memberlist

  # ── 摄入限制 ─────────────────────────────────────────────
  limits_config:
    ingestion_rate_mb: 64        # 每租户 64MB/s 写入速率
    ingestion_burst_size_mb: 128
    per_stream_rate_limit: 10MB
    per_stream_rate_limit_burst: 20MB
    max_label_names_per_series: 15    # 标签数量上限，防止基数爆炸
    max_label_name_length: 1024
    max_label_value_length: 2048
    max_streams_per_user: 100000      # 每租户最多 10 万流
    max_query_series: 5000
    max_query_parallelism: 32
    max_entries_limit_per_query: 50000
    query_timeout: 5m
    retention_period: 30d
    unordered_writes: true            # 容忍网络抖动导致的乱序写入
    max_chunk_age: 2h

  # ── Ingester 配置（核心） ────────────────────────────────
  ingester:
    wal:
      enabled: true
      dir: /loki/wal
      flush_on_shutdown: true    # 下线前强制 flush，数据不丢
      replay_memory_ceiling: 6GB # WAL 回放内存上限
    lifecycler:
      ring:
        replication_factor: 3
        heartbeat_timeout: 1m
      heartbeat_period: 5s
      join_after: 30s            # 等待 Ring 稳定后再接受写入
      final_sleep: 30s           # 下线前等待 Ring 收敛
      num_tokens: 256            # 环上虚拟节点数，越多分布越均匀
    chunk_idle_period: 30m       # 流空闲超过 30 分钟触发 flush
    chunk_retain_period: 1m
    max_chunk_age: 2h            # 超过 2 小时强制 flush
    chunk_target_size: 1572864   # 目标 Chunk 大小 1.5MB
    chunk_encoding: snappy       # 压缩算法
    concurrent_flushes: 32       # 并发 flush goroutine 数
    max_transfer_retries: 0      # 使用 WAL 替代节点间 transfer

  # ── Distributor 配置 ─────────────────────────────────────
  distributor:
    ring:
      kvstore:
        store: memberlist

  # ── Compactor 配置 ───────────────────────────────────────
  compactor:
    working_directory: /loki/compactor
    shared_store: s3
    compaction_interval: 10m
    retention_enabled: true
    retention_delete_delay: 2h
    retention_delete_worker_count: 150

  # ── 存储配置（S3）────────────────────────────────────────
  storage_config:
    aws:
      s3: s3://ap-northeast-1/loki-prod-chunks
      region: ap-northeast-1
      access_key_id: ${AWS_ACCESS_KEY_ID}
      secret_access_key: ${AWS_SECRET_ACCESS_KEY}
      s3forcepathstyle: false
      http_config:
        idle_conn_timeout: 90s
    tsdb_shipper:
      active_index_directory: /loki/tsdb-index
      cache_location: /loki/tsdb-index-cache
      cache_ttl: 24h
      shared_store: s3

  # ── Schema 配置 ──────────────────────────────────────────
  schema_config:
    configs:
      - from: "2024-01-01"
        store: tsdb              # Loki 2.8+ 推荐，性能更优
        object_store: s3
        schema: v12
        index:
          prefix: loki_prod_index_
          period: 24h

  # ── Query 缓存 ───────────────────────────────────────────
  query_range:
    align_queries_with_step: true
    max_retries: 5
    cache_results: true
    results_cache:
      cache:
        redis:
          endpoint: redis:6379
          expiration: 1h

  querier:
    max_concurrent: 20
    query_ingesters_within: 3h   # 3h 内数据优先查 Ingester 热数据

  chunk_store_config:
    chunk_cache_config:
      redis:
        endpoint: redis:6379
        expiration: 2h

  frontend:
    log_queries_longer_than: 5s
    compress_responses: true
    max_outstanding_per_tenant: 2048

# ── K8s 组件资源配置 ─────────────────────────────────────────

# Ingester：有状态 StatefulSet，6 副本
ingester:
  replicas: 6
  maxUnavailable: 1
  persistence:
    enabled: true
    storageClass: loki-wal-ssd
    size: 50Gi                   # WAL 存储，按峰值 2h 写入量估算
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

# Distributor：无状态，自动扩缩
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

# Query Frontend
queryFrontend:
  replicas: 2
  resources:
    requests:
      cpu: 1
      memory: 2Gi
    limits:
      cpu: 2
      memory: 4Gi

# Querier
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

# Compactor：单实例（Leader 选举保证唯一）
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

# Redis 缓存（Query 结果 + Chunk 缓存）
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

### 4. Fluent Bit 采集配置

```yaml
# fluent-bit-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush           5
        Daemon          Off
        Log_Level       info
        Parsers_File    parsers.conf
        HTTP_Server     On
        HTTP_Listen     0.0.0.0
        HTTP_Port       2020        # 暴露 /metrics 给 Prometheus 抓取

    # ── INPUT：读取所有 Pod 日志 ─────────────────────────────
    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Exclude_Path      /var/log/containers/*_logging_*.log  # 排除自身
        Parser            cri
        DB                /var/log/flb_kube.db  # 记录读取位置，重启不丢失
        DB.Sync           Normal
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On
        Refresh_Interval  10

    # ── FILTER 1：注入 K8s metadata ─────────────────────────
    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On      # 将 JSON 正文字段合并到顶层
        Merge_Log_Key       log_processed
        Keep_Log            Off     # 合并后丢弃原始 log 字段
        Annotations         Off     # 不注入 annotation（避免基数爆炸）
        Labels              On      # 注入 Pod labels

    # ── FILTER 2：提升 JSON 日志字段到顶层 ──────────────────
    [FILTER]
        Name          nest
        Match         kube.*
        Operation     lift
        Nested_under  log_processed
        Add_prefix    parsed_

    # ── FILTER 3：丢弃 debug 日志（生产降噪）────────────────
    [FILTER]
        Name    grep
        Match   kube.*
        Exclude parsed_level ^debug$

    # ── OUTPUT：推送到 Loki ──────────────────────────────────
    [OUTPUT]
        Name            loki
        Match           kube.*
        Host            loki-distributor.logging.svc.cluster.local
        Port            3100
        Labels          job=fluentbit, cluster=prod-cn-north
        # 核心：只映射低基数字段为 Loki 标签
        Label_Keys      $kubernetes['namespace_name'],$kubernetes['labels']['app.kubernetes.io/name'],$parsed_level
        Tenant_ID       production
        Line_Format     json
        # ⚠️ 关键：禁止自动映射所有 k8s label，否则基数爆炸
        Auto_Kubernetes_Labels Off
        Retry_Limit     5

  parsers.conf: |
    # containerd / CRI 日志格式
    [PARSER]
        Name        cri
        Format      regex
        Regex       ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<message>.*)$
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z

    # 应用结构化 JSON 日志
    [PARSER]
        Name        json
        Format      json
        Time_Key    timestamp
        Time_Format %Y-%m-%dT%H:%M:%S.%LZ
```

> **关于 `Auto_Kubernetes_Labels` 的警告**
>
> 开启这个选项会把所有 Pod labels 自动映射为 Loki 标签，包括 `app.kubernetes.io/version`（每次发版都变）、`pod-template-hash`（每个 ReplicaSet 不同）等高基数字段，几分钟内 Loki 流数量可以从几千飙到几十万，直接把 Ingester 打爆。**生产环境务必保持 `Off`，手动指定低基数字段。**

---

### 5. Fluent Bit DaemonSet + RBAC

```yaml
# fluent-bit-daemonset.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluent-bit
  namespace: logging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluent-bit
rules:
  - apiGroups: [""]
    resources: ["pods", "namespaces", "nodes"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluent-bit
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: fluent-bit
subjects:
  - kind: ServiceAccount
    name: fluent-bit
    namespace: logging
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "2020"
        prometheus.io/path: "/api/v1/metrics/prometheus"
    spec:
      serviceAccountName: fluent-bit
      tolerations:
        - operator: Exists             # 采集所有节点，含 master/taint 节点
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:3.2
          resources:
            requests:
              cpu: 100m
              memory: 64Mi            # 对比 Promtail 的 256Mi，节省 4 倍
            limits:
              cpu: 500m
              memory: 128Mi
          ports:
            - containerPort: 2020
              name: metrics
          livenessProbe:
            httpGet:
              path: /
              port: 2020
            initialDelaySeconds: 10
            periodSeconds: 30
          volumeMounts:
            - name: config
              mountPath: /fluent-bit/etc/
            - name: varlog
              mountPath: /var/log
              readOnly: true
            - name: db
              mountPath: /var/log/flb_kube.db
      volumes:
        - name: config
          configMap:
            name: fluent-bit-config
        - name: varlog
          hostPath:
            path: /var/log
        - name: db
          hostPath:
            path: /var/log/flb_kube.db
```

---

### 6. Ingester KEDA 弹性扩容

Ingester 建议基于 Chunk 创建速率扩容，而非 CPU，因为 CPU 无法反映内存 Chunk 积压压力：

```yaml
# ingester-scaledobject.yaml
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
        threshold: "500"          # 每实例超过 500 chunks/s 触发扩容
        query: |
          sum(rate(loki_ingester_chunks_created_total[2m])) /
          count(up{job="loki-ingester"})
```

---

### 7. 告警规则（LogQL Rules）

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
          # 错误日志速率过高
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
                过去 5 分钟错误日志速率 {{ $value | humanize }} 条/秒，超过阈值 10 条/秒。

          # OOM 事件检测
          - alert: PodOOMKilled
            expr: |
              count_over_time({namespace=~".+"} |= "OOMKilled" [5m]) > 0
            labels:
              severity: critical
            annotations:
              summary: "检测到 OOMKilled 事件"

          # 日志写入速率骤降（采集断路预警）
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
          # Ingester 流数量预警（标签基数健康度）
          - alert: LokiHighStreamCount
            expr: |
              sum(loki_ingester_streams_created_total) by (pod) > 50000
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "{{ $labels.pod }} 流数量过高，检查标签基数"

          # Chunk 平均大小过小（内存压力导致提前 flush）
          - alert: LokiSmallChunkSize
            expr: |
              (loki_ingester_chunk_size_bytes_sum / loki_ingester_chunk_size_bytes_count) < 524288
            for: 10m
            labels:
              severity: warning
            annotations:
              summary: "Ingester Chunk 平均大小 < 512KB，可能存在内存压力"

          # Ingester 内存 Chunks 积压
          - alert: LokiFlushQueueDepth
            expr: loki_ingester_memory_chunks > 50000
            for: 10m
            labels:
              severity: warning
            annotations:
              summary: "Ingester 内存 Chunks 积压，Flush 速率可能不足"
```

---

### 8. Grafana 数据源

```yaml
# grafana-datasource.yaml
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
    editable: true
```

---

### 9. 安装命令

```bash
# 0. 准备 namespace 和前置资源
kubectl create namespace logging
kubectl apply -f object-storage-secret.yaml
kubectl apply -f ingester-storageclass.yaml

# 1. 添加 Helm 仓库
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# 2. 安装 Loki（分布式模式）
helm install loki grafana/loki \
  --namespace logging \
  --version 6.x.x \
  -f loki-values.yaml \
  --set loki.storage.s3.accessKeyId="${AWS_ACCESS_KEY_ID}" \
  --set loki.storage.s3.secretAccessKey="${AWS_SECRET_ACCESS_KEY}"

# 3. 部署 Fluent Bit
kubectl apply -f fluent-bit-configmap.yaml
kubectl apply -f fluent-bit-daemonset.yaml

# 4. 部署 KEDA 弹性扩容（需提前安装 KEDA）
kubectl apply -f ingester-scaledobject.yaml

# 5. 应用告警规则
kubectl apply -f loki-rules-configmap.yaml

# 6. 验证
kubectl get pods -n logging
# 检查 Ring 状态（所有 Ingester 应为 ACTIVE）
kubectl exec -n logging deploy/loki-distributor -- \
  curl -s http://localhost:3100/ring | python3 -m json.tool
# 检查 Fluent Bit 写入成功率
kubectl exec -n logging ds/fluent-bit -- \
  curl -s http://localhost:2020/api/v1/metrics/prometheus | grep fluentbit_output_retried_records
```

---

## 运维排查速查

### 常用 LogQL

```logql
# 查看指定服务最近 error 日志
{namespace="payment", app="order-api", level="error"} | limit 100

# 按 app 统计 5 分钟错误速率
sum(rate({namespace="payment", level="error"}[5m])) by (app)

# 解析 JSON 日志，按 trace_id 追踪
{namespace="payment", app="order-api"}
  | json
  | trace_id="3fa2c1d4-8b7e-11ef-a3d2-0242ac120002"

# 统计各 namespace 日志量占比（容量规划）
sum(bytes_over_time({cluster="prod-cn-north"}[24h])) by (namespace)
  / sum(bytes_over_time({cluster="prod-cn-north"}[24h]))

# 检查活跃流数量（标签基数健康度）
count(count_over_time({cluster="prod-cn-north"}[5m])) by (namespace)
```

### Ingester Ring 健康检查

```bash
# 查看 Ring 状态（所有节点应为 ACTIVE）
curl -s http://loki-ingester:3100/ring | python3 -m json.tool

# 查看关键指标
curl -s http://loki-ingester:3100/metrics | grep -E \
  "loki_ingester_(streams|chunks|memory|flush|chunk_size)"

# 手动触发 Flush（维护前执行）
curl -X POST http://loki-ingester-0:3100/flush

# 优雅下线（缩容前执行，等待 flush 完成后再删 Pod）
curl -X POST http://loki-ingester-5:3100/ingester/shutdown
```

---

## 小结

| 关键决策 | 推荐方案 |
|---|---|
| 日志采集 Agent | **Fluent Bit**（替代 Promtail） |
| 存储后端 | S3 / GCS（成本低，免运维） |
| 索引类型 | TSDB（Loki 2.8+） |
| Ingester replication_factor | **3**（容忍 1 节点故障） |
| Ingester 部署策略 | **多个小节点**优于少量大节点 |
| Ingester 扩容依据 | Chunk 创建速率（非 CPU） |
| 标签数量 | ≤ 8 个，全部低基数 |
| 日志格式 | 结构化 JSON，高基数字段放正文 |
| 缓存层 | Redis（Query 结果 + Chunk 缓存） |
| 保留策略 | 热数据 7 天（SSD），冷数据 30 天（S3） |

Loki Stack 最大的价值不在于「功能最强」，而在于**在 Prometheus 生态下以最低的运维成本换取足够好的可观测能力**。理解 Ingester Ring 的写入机制是调优的关键——副本数、节点规模、内存配置、标签设计，四个维度相互影响，牵一发而动全身。对于日均 1TB 的 K8s 生产场景，本文给出的配置已在多个生产集群验证，可作为团队标准化基线直接落地。
