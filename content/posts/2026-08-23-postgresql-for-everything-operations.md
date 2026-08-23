---
title: "PostgreSQL for Everything？运维视角下的能力边界与生产基线"
date: 2026-08-23T10:00:00+08:00
draft: false
tags: ["PostgreSQL", "Database", "Architecture", "DevOps", "JSONB", "pgvector"]
categories: ["Database", "Architecture", "DevOps"]
author: "Kaka"
description: "PostgreSQL 能否替代搜索、文档数据库、消息队列、时序数据库、Vector Database 与 Redis？从运维成本、数据一致性、生产基线和退出条件给出可落地判断。"
---

## 引言

“The answer to everything is PostgreSQL。”这句话当然带着玩笑成分，但背后有一个非常严肃的架构问题：为了全文搜索、JSON document、异步任务、time series、vector search 和 cache，我们真的需要一开始就部署六套不同的系统吗？

Raphael A. Bauer 在 [PostgreSQL for Everything](https://www.raphaelbauer.com/posts/postgresql-everything/) 中给出了明确主张：PostgreSQL 稳定、容易获得，而且通过内置能力和 extension 可以承担远超传统 RDBMS 的职责。这个方向我基本赞同，但从运维视角必须补上后半句：**PostgreSQL first，不是 PostgreSQL forever。**

站内已有一篇 [《2026 年了，直接用 PostgreSQL 吧》](/posts/2026-02-15-postgres-2026/)，重点介绍 PostgreSQL 的广泛能力。本文不再重复“它什么都能做”，而是回答三个更接近生产的问题：什么时候应该先用 PostgreSQL，怎样避免把所有 workload 塞进同一个 failure domain，以及出现哪些信号时必须拆出专用系统。

> 本文参考 Raphael A. Bauer 原文的核心观点重新撰写，不是逐句翻译；内容结合 PostgreSQL 18 official docs，增加了 queue correctness、UNLOGGED table 风险、PITR、autovacuum、connection pooling、extension governance 和 migration exit criteria。

## 真正昂贵的不是组件，而是组件之间的边

引入一个专用系统，成本远不止多一个 Deployment 或 VM。你还会得到一组新的 connection credential、TLS policy、backup、restore、upgrade、monitoring、capacity model、on-call runbook 和 security boundary。更麻烦的是数据同步：PostgreSQL 是 system of record，Elasticsearch、Redis、Vector Database 和 analytics store 各自保存副本，CDC pipeline 一旦延迟或失败，系统便同时存在多个“真相”。

```text
单一 PostgreSQL

Application ── transaction ──> PostgreSQL
                       一套 backup / HA / audit / access control

多种专用系统

Application ──> PostgreSQL ── CDC ──> Search
       │             │          └────> Analytics
       │             └───────────────> Cache invalidation
       └─────────────────────────────> Vector Database

复杂度主要增长在同步、重放、顺序、幂等和故障恢复这些“边”上。
```

因此默认从 PostgreSQL 开始，价值不是 PostgreSQL 在每个 benchmark 中都最快，而是它让 transaction boundary、authorization、backup 和 observability 保持集中。只有专用系统带来的收益已经超过新增的 operational cost，拆分才成立。

## 全文搜索：先用内置 FTS，复杂 relevance 再拆

PostgreSQL 内置 `tsvector`、`tsquery`、ranking、dictionary 与 GIN index。搜索对象本来就在业务表中，且需求主要是关键词匹配、简单 ranking、filter 和 transaction consistency 时，内置 Full Text Search 通常是最低成本方案。

```sql
CREATE TABLE articles (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title text NOT NULL,
    body text NOT NULL,
    published_at timestamptz,
    search_vector tsvector GENERATED ALWAYS AS (
        setweight(to_tsvector('simple', coalesce(title, '')), 'A') ||
        setweight(to_tsvector('simple', coalesce(body, '')), 'B')
    ) STORED
);

CREATE INDEX articles_search_idx
    ON articles USING GIN (search_vector);

SELECT id,
       title,
       ts_rank(search_vector, websearch_to_tsquery('simple', $1)) AS rank
FROM articles
WHERE search_vector @@ websearch_to_tsquery('simple', $1)
ORDER BY rank DESC
LIMIT 20;
```

示例使用 `simple` configuration 只是为了展示语法，不代表它适合所有语言。上线前应通过 `\dF` 检查可用 text search configuration；中文等需要 word segmentation 的内容通常要引入经过治理的 tokenizer extension，或者直接采用更成熟的外部 search engine。

它的优势是没有同步延迟：row 和 search index 在同一个 transaction 中提交。边界也很清楚：如果业务依赖 typo tolerance、复杂 synonym pipeline、BM25 tuning、faceting、高亮、跨大量 index 搜索、独立扩缩容或日志级 ingestion throughput，Elasticsearch/OpenSearch 仍然更合适。不要用“PostgreSQL 也能搜索”推导出“它应该承载所有搜索”。

## JSONB：用 hybrid schema，不要把关系模型扔掉

`jsonb` 适合结构有一定弹性、但仍需要 join、transaction 和统一权限模型的场景。最稳妥的设计不是把整张表变成一个 `payload jsonb`，而是把 identity、tenant、状态、金额、时间等强约束字段保留为 typed columns，只把变化快、稀疏的 attributes 放进 `jsonb`。

```sql
CREATE TABLE orders (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id uuid NOT NULL,
    status text NOT NULL CHECK (status IN ('pending', 'paid', 'cancelled')),
    amount numeric(18, 2) NOT NULL CHECK (amount >= 0),
    attributes jsonb NOT NULL DEFAULT '{}'::jsonb,
    created_at timestamptz NOT NULL DEFAULT clock_timestamp()
);

-- 适合以 @> containment 为主的查询。
CREATE INDEX orders_attributes_gin
    ON orders USING GIN (attributes jsonb_path_ops);

-- 高频固定字段更适合 expression index。
CREATE INDEX orders_external_id_idx
    ON orders ((attributes ->> 'external_id'));
```

`jsonb_path_ops` index 通常更小，适合 `@>`、`@?`、`@@`，但不支持 key-exists `?`。不要无脑给整个 document 建 GIN：index 会放大 write、VACUUM 和 storage cost。大 JSON document 的任意更新还会锁整行并产生新的 row version；当不同字段需要高并发独立更新时，应拆成关系表，而不是继续向 JSON 深处嵌套。

## Queue：`SKIP LOCKED` 很好用，但它不是 Kafka

PostgreSQL queue 最大的价值是业务写入和 job enqueue 可以处于同一个 transaction，不需要解决“双写成功了一半”。多个 worker 可以利用 `FOR UPDATE SKIP LOCKED` 并发领取不同 job：

```sql
WITH picked AS (
    SELECT id
    FROM jobs
    WHERE status = 'ready'
      AND run_at <= clock_timestamp()
    ORDER BY priority DESC, id
    FOR UPDATE SKIP LOCKED
    LIMIT 1
)
UPDATE jobs AS j
SET status = 'running',
    locked_by = $1,
    locked_at = clock_timestamp(),
    attempts = attempts + 1
FROM picked
WHERE j.id = picked.id
RETURNING j.*;
```

worker 应快速提交领取 transaction，再执行外部任务；不要在调用 third-party API 的几分钟里一直持有 row lock。还需要 lease timeout、retry/backoff、dead-letter state、idempotency key 和 stuck-job recovery，例如将超过 lease 且未完成的 job 重新置为 `ready`。

它适合 background job、transactional outbox、小中规模 work queue。它不适合高吞吐 append-only event log、大量 consumer group、长周期 replay、跨 region stream processing 或严格 partition ordering；这些是 Kafka/Pulsar 的设计中心。RabbitMQ 的 routing、ack、priority 和 delivery semantics 也不能简单等同于一张表。

## Time Series：partitioning 是起点，不是自动驾驶

原生 declarative partitioning 可以按时间拆表，通过 partition pruning 减少扫描，并通过 `DROP TABLE` 或 `DETACH PARTITION` 快速执行 retention，避免海量 `DELETE` 带来的 VACUUM 压力：

```sql
CREATE TABLE metrics (
    tenant_id uuid NOT NULL,
    ts timestamptz NOT NULL,
    metric_name text NOT NULL,
    value double precision NOT NULL
) PARTITION BY RANGE (ts);

CREATE TABLE metrics_2026_08
    PARTITION OF metrics
    FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');
```

但 partition 不会自动解决 cardinality、compression、downsampling、late arrival、retention policy 和 hot/cold storage。TimescaleDB 在 PostgreSQL 上增加 hypertable 与 continuous aggregate，适合希望保留 SQL/transaction 能力的时序 workload；当 workload 以大规模 append、列式 scan、极高压缩率和 OLAP aggregation 为中心时，ClickHouse 之类 columnar system 可能具有更好的成本曲线。

运维上必须控制 partition 数量，并为未来 partition 自动创建和监控。漏建下个月 partition 会直接让 insert 失败；过多 partition 又会增加 planning 和 catalog overhead。partitioned parent 不存 row，autovacuum 不会替 parent 自动更新统计信息，数据分布明显变化后要主动 `ANALYZE` parent。

## Vector Search：pgvector 适合“关系数据旁边的向量”

`pgvector` 的核心优势不是宣称替代所有 Vector Database，而是向量、metadata、tenant policy 和业务 transaction 可以共存。它支持 exact search，以及 HNSW、IVFFlat approximate index：

```sql
CREATE EXTENSION vector;

CREATE TABLE documents (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id uuid NOT NULL,
    content text NOT NULL,
    embedding vector(1536) NOT NULL
);

CREATE INDEX CONCURRENTLY documents_embedding_hnsw
    ON documents USING hnsw (embedding vector_cosine_ops);

SELECT id, content
FROM documents
WHERE tenant_id = $1
ORDER BY embedding <=> $2::vector
LIMIT 10;
```

HNSW 需要在 recall、latency、memory、build time 和 write amplification 之间调参；metadata filtering 还会影响结果数量和 index scan 行为。数据量、QPS 和 tenant 数继续增长时，应持续用真实 query distribution 做 recall benchmark，而不是只测单次 latency。若 vector index 已经主导 memory、VACUUM、replication lag 和 storage，或者需要独立扩缩容，它就应该拥有独立的 serving path。

## Redis、文件系统和 Graph：能做不等于应该做

原文提出用 UNLOGGED table 做 cache。它确实不写 WAL，write performance 更高，但语义必须说完整：server crash 或 unclean shutdown 后内容会被清空，也不会复制到 standby；它仍然消耗 shared buffer、connection、MVCC、lock、autovacuum、CPU 和 I/O。TTL 也不是 PostgreSQL 自动提供的能力，仍需 background cleanup。把 cache load 放进主库，可能让一个本可丢弃的 workload 拖慢真正不能丢的数据。

同样，`bytea`、Large Object 可以保存 binary data，transaction consistency 很方便；但大型文件会放大 WAL、base backup、restore time 和 replica traffic。常见生产边界是：小而强事务关联的 blob 放 PostgreSQL，大对象放 S3-compatible object storage，数据库只存 metadata、checksum、object key 和 lifecycle state。

树形结构可用 recursive CTE 或 `ltree`，property graph 还能借助 Apache AGE。若查询以高 fan-out、多跳 traversal、graph algorithm 和 graph-specific tooling 为主，专用 Graph Database 更自然。PostgreSQL 可以生成 JSON、暴露 function，甚至承担 API 层的一部分，但把全部业务逻辑压进 stored procedure 会让 database 成为 release bottleneck 和更大的 blast radius。

## 一张表决定是否应该拆分

| 需求 | PostgreSQL-first 适用条件 | 应考虑专用系统的信号 |
|---|---|---|
| Full Text Search | 数据已在 PostgreSQL、基础 relevance、强一致 | 复杂 ranking/faceting、独立扩缩容、搜索吞吐主导资源 |
| JSON document | hybrid schema、需要 transaction/join | 超大 document、schema 完全动态、document write pattern 主导 |
| Job Queue | transactional enqueue、有限 consumer、任务可幂等 | 大规模 fan-out/replay、跨 region、严格 stream ordering |
| Time Series | 中等 ingest、SQL aggregation、简单 retention | 列式 scan/压缩决定成本，数据规模长期压倒 OLTP |
| Vector Search | 向量和业务 metadata 强关联 | index 主导 memory/I/O，需要独立 scaling 或更高 recall/QPS |
| Cache | 小规模、可丢弃、无需复杂 eviction | sub-millisecond SLA、TTL/eviction/pub-sub、cache load 影响主库 |
| Blob | 小对象、强事务关联 | 大文件、CDN、生命周期管理、backup/replication 成本明显 |
| Graph | tree、有限层级 traversal | 高 fan-out、多跳 graph query 与 algorithm 是核心 workload |

判断标准不应该是“能不能写出 SQL”，而应是 workload 是否还能共享同一组 SLO、resource envelope、failure domain 和 scaling model。

## PostgreSQL-first 的生产基线

把更多能力集中进 PostgreSQL，会减少组件数量，却提高数据库本身的重要性。以下基线必须先于 extension 和高级功能。

### Version lifecycle

截至 2026-08-23，current stable major 是 PostgreSQL 18，受支持版本为 14–18；PostgreSQL 14 将在 2026-11-12 EOL。community major version 的支持周期约五年，minor release 主要包含 bug、security 和 data corruption fix。生产环境应持续跟进当前 minor，而不是因为“数据库很稳定”长期不升级。

major upgrade 需要 `pg_upgrade`、dump/restore 或 logical replication 等迁移路径。每年提前做 extension compatibility matrix，尤其检查 managed service 是否支持目标 extension/version；extension 能进入 database backend process，其 upgrade、ABI、权限和 supply chain 应按生产代码治理。

### Connection budget

PostgreSQL 不是让 application 无限创建 connection 的 serverless endpoint。先给每个 application、migration、monitoring、admin 和 failover reserve 分配 connection budget，再决定 `max_connections`。短连接或大量 application replica 通常需要 PgBouncer；transaction pooling 能显著复用 server connection，但会破坏部分 session-level feature，不能无脑切换。

```sql
SELECT application_name,
       state,
       count(*) AS connections
FROM pg_stat_activity
GROUP BY application_name, state
ORDER BY connections DESC;
```

### Autovacuum 与 bloat

MVCC update/delete 会留下 dead tuple。autovacuum 同时负责复用空间、更新 planner statistics、维护 visibility map 和防止 transaction ID wraparound。高 churn queue、JSONB 和 cache table 往往比默认参数更早需要 per-table tuning。

```sql
SELECT relname,
       n_live_tup,
       n_dead_tup,
       last_autovacuum,
       last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

不要把定时 `VACUUM FULL` 当日常维护；它需要 `ACCESS EXCLUSIVE` lock。正确方向是让普通 autovacuum 足够及时，监控 dead tuple、freeze age、I/O 和 vacuum progress，并对异常 hot table 单独配置 threshold 与 scale factor。

### Backup、PITR 与 restore drill

replication 不是 backup：误删、错误 migration 和逻辑损坏会快速复制到 standby。生产至少需要 base backup + continuous WAL archive，才能做 Point-in-Time Recovery（PITR）；还要明确 RPO、RTO、retention、encryption、off-site copy 和 immutable policy。

真正的验收标准不是“backup job succeeded”，而是在隔离环境完成自动 restore，校验 recovery target、关键 row count、schema、extension 和 application smoke test。没有定期 restore drill 的 backup，只是未经验证的愿望。

### HA 与故障域

streaming replica 可以缩短 host failure 的恢复时间，但 failover 还涉及 leader election、client routing、fencing、replication lag 和 split-brain prevention。把 queue、search、vector 和 OLTP 全部集中在一个 cluster 后，必须决定它们是否允许共享 CPU、I/O、WAL 和 failover event。

最少要做 workload isolation：独立 role/database/schema、statement timeout、connection pool、resource-aware query policy，以及必要时独立 instance。逻辑上使用 PostgreSQL，不代表物理上只能有一个 cluster。

### Observability

仅监控 CPU、memory、disk utilization 不够。至少覆盖 connection saturation、transaction rate、slow query、lock wait、deadlock、cache hit、WAL generation、archive failure、replication lag、checkpoint、autovacuum、bloat、table/index growth 和 disk forecast。

```sql
-- 谁正在等待，等什么。
SELECT pid,
       application_name,
       wait_event_type,
       wait_event,
       clock_timestamp() - query_start AS running_for,
       query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY running_for DESC;

-- WAL archive 是否持续成功。
SELECT archived_count,
       failed_count,
       last_archived_time,
       last_failed_time
FROM pg_stat_archiver;
```

启用 `pg_stat_statements` 后再基于 total execution time、mean latency、calls 和 temporary I/O 找真正的 workload，而不是凭感觉调 `shared_buffers`。PostgreSQL 18 还提供 `pg_stat_io` 等更细的 I/O 统计，但 metric 只有进入 dashboard、alert 和 runbook 才有价值。

## 从一开始就设计退出路径

PostgreSQL-first 最怕两种极端：一种是一开始就为不存在的规模部署专用系统；另一种是 PostgreSQL 已经明显超出边界，团队仍以“减少组件”为理由拒绝拆分。正确方式是在 Architecture Decision Record 中提前写清楚 exit criteria。

例如 Vector Search 可以规定：当 HNSW index 超过可分配 memory、replication lag 连续违反 SLO、p95 超过 80 ms 或真实 recall 低于目标，就启动专用 Vector Database 评估。Queue 可以规定：当 retention、consumer group、replay 或跨 region ordering 成为硬需求，就迁移到 Kafka/Pulsar，而不是继续扩展一张 jobs 表。

拆分也不应该从 application dual-write 开始。优先使用 transactional outbox 或 PostgreSQL logical replication/CDC，把变更可靠地送往新系统；为 backfill、增量追平、校验、cutover 和 rollback 分别定义步骤。专用系统先作为 derived read model，PostgreSQL 保持 system of record，直到迁移被证明可逆且一致。

```text
Start in PostgreSQL
        │
        ▼
定义 SLO、容量预算、exit criteria
        │
        ├─ SLO 满足 ─────────────> 保持简单，继续优化
        │
        └─ 持续违反且已定位瓶颈
                    │
                    ▼
             CDC / outbox 建立副本
                    │
                    ▼
             校验、灰度、可回滚 cutover
```

## 总结

PostgreSQL 不是所有 workload 的最佳终点，但它往往是最好的起点。Full Text Search、JSONB、`SKIP LOCKED` queue、partitioning、pgvector、`ltree` 和 extension ecosystem 可以让团队在业务尚未证明需要专用系统时，避免过早承担同步、备份、权限和 on-call 复杂度。

真正成熟的原则不是“PostgreSQL 替代一切”，而是：**先用 PostgreSQL 保持 transaction 和运维边界简单；用 production metric 判断能力边界；只有收益明确超过新增系统的全生命周期成本时才拆分。**

当你选择 PostgreSQL for more things，也必须相应提高它的 production discipline：current minor、connection budget、autovacuum、PITR restore drill、HA fencing、extension governance、observability 和明确的 exit strategy。简单架构不是少装几个软件，而是更少的隐式故障模式。

## 参考资料

- [Raphael A. Bauer：PostgreSQL for Everything](https://www.raphaelbauer.com/posts/postgresql-everything/)
- [PostgreSQL Versioning Policy](https://www.postgresql.org/support/versioning/)
- [PostgreSQL Full Text Search](https://www.postgresql.org/docs/current/textsearch.html)
- [PostgreSQL JSON Types](https://www.postgresql.org/docs/current/datatype-json.html)
- [PostgreSQL SELECT / SKIP LOCKED](https://www.postgresql.org/docs/current/sql-select.html)
- [PostgreSQL Table Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)
- [PostgreSQL Routine Vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html)
- [PostgreSQL Continuous Archiving and PITR](https://www.postgresql.org/docs/current/continuous-archiving.html)
- [PostgreSQL Monitoring Database Activity](https://www.postgresql.org/docs/current/monitoring.html)
- [PostgreSQL UNLOGGED Tables](https://www.postgresql.org/about/featurematrix/detail/unlogged-tables/)
- [pgvector official repository](https://github.com/pgvector/pgvector)
- [PgBouncer feature matrix](https://www.pgbouncer.org/features.html)
- [Apache AGE](https://age.apache.org/)
