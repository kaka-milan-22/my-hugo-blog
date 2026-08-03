---
title: "REST vs gRPC：下一套 API 到底该怎么选？"
date: 2026-08-03T10:00:00+08:00
draft: false
tags: ["REST", "gRPC", "API", "Protocol Buffers", "HTTP"]
categories: ["Architecture", "云原生"]
author: "Kaka"
description: "从 API 模型、contract、传输格式、streaming、browser compatibility 和运维成本对比 REST 与 gRPC，并给出面向真实场景的选型原则。"
---

## 引言

REST 和 gRPC 经常被放在一起比较，但它们并不是同一层面的两个协议。REST 是一种以 resource 和 representation 为中心的 architectural style；gRPC 是一套以远程方法调用为中心、通常使用 Protocol Buffers 定义 contract 的 RPC framework。

本文参考了 Kreya 的 [A detailed comparison of REST and gRPC](https://kreya.app/blog/rest-vs-grpc/) 的问题框架，并结合 HTTP、gRPC、Protocol Buffers 与 OpenAPI 官方资料重新整理。重点不是宣布谁取代谁，而是回答一个更实用的问题：什么边界应该使用 REST，什么流量更适合 gRPC？

先给结论：public API、browser client、开放生态集成，默认优先 REST + JSON + OpenAPI；内部高频调用、强类型 contract、多语言 client generation 或 streaming 场景，优先考虑 gRPC。大型系统通常不必二选一，边界使用 REST，内部使用 gRPC 往往更合理。

## 核心差异一览

| 维度 | REST/HTTP API | gRPC |
|---|---|---|
| 抽象模型 | 围绕 resource、URI 和 HTTP method | 围绕 service 与 method |
| Contract | 可使用 OpenAPI，但也可能只有文档或实现 | 通常先定义 `.proto`，再生成 client/server stub |
| 常见编码 | JSON，可读、生态成熟 | Protobuf，binary、strongly typed |
| 传输 | 可运行于不同 HTTP version | 标准 gRPC 依赖 HTTP/2 |
| 调用模式 | 典型 API 是 request/response；streaming 需结合 HTTP stream、SSE 或 WebSocket | 原生定义 unary、server streaming、client streaming、bidirectional streaming |
| Browser | `fetch`、XHR 原生支持 | 通常需要 gRPC-Web 和 proxy，能力也不完全等同于原生 gRPC |
| Cache | 可直接利用 HTTP cache semantics、CDN 和 conditional request | 通常由 application 自己设计 cache |
| 调试 | `curl`、browser DevTools、API gateway 支持成熟 | 需要理解 `.proto`、metadata、status 和 trailers |
| 适合边界 | Public API、Web、第三方 integration | Service-to-service、内部 data plane、实时 stream |

## REST：围绕 resource 组织系统

REST 风格的 API 通常把业务对象表示为 URI，再利用 HTTP method 表达意图。例如订单系统可以这样设计：

```http
GET    /v1/orders/ord-123
GET    /v1/orders?customer_id=c-42&page_token=...
POST   /v1/orders
PATCH  /v1/orders/ord-123
DELETE /v1/orders/ord-123
```

这种模型的最大价值不是 JSON，而是复用 HTTP 已经定义好的 semantics。`GET` 是 safe method，`PUT` 和 `DELETE` 具有 idempotent semantics；status code、content negotiation、ETag、Cache-Control、conditional request 与 CDN cache 都有成熟的协议和中间件支持。

REST 并不强制使用 JSON，也不等于某一份具体 API specification。现实中的“REST API”大多指 HTTP + JSON API，但设计一致性仍要由团队负责：URI 命名、错误结构、pagination、idempotency key 和 versioning 如果没有统一规范，不同 service 很快就会形成不同方言。

REST 也不是只能 code-first。使用 OpenAPI 先定义 schema、operation 和 response，再生成 client 或 server stub，同样可以走 design-first。问题在于 OpenAPI 并非 REST API 的必选项；如果 contract 只是从实现代码反向生成，内部 model 的意外变化就可能泄漏为 breaking API change。

当业务天然是 action 而不是 resource 时，REST model 也会变得别扭。例如批量创建订单、触发结算、执行风控重算，团队可能陷入 `/orders/batch`、`/orders/actions/recalculate` 是否“足够 RESTful”的争论。技术上都能实现，但抽象成本确实存在。

## gRPC：把远程调用定义成强类型方法

gRPC 从 service method 出发。下面这份 `.proto` 同时定义了 method、request、response 和 message field，之后可以为 Go、Java、Python 等受支持语言生成 client 与 server code：

```protobuf
syntax = "proto3";

package orders.v1;

service OrderService {
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc WatchOrders(WatchOrdersRequest) returns (stream OrderEvent);
  rpc ImportOrders(stream CreateOrderRequest) returns (ImportOrdersResponse);
  rpc SyncOrders(stream OrderEvent) returns (stream OrderEvent);
}

message GetOrderRequest {
  string order_id = 1;
}

message WatchOrdersRequest {
  string customer_id = 1;
}

message CreateOrderRequest {
  string customer_id = 1;
  int64 amount_cents = 2;
}

message ImportOrdersResponse {
  uint32 accepted = 1;
}

message Order {
  string order_id = 1;
  string status = 2;
}

message OrderEvent {
  string order_id = 1;
  string status = 2;
  int64 occurred_at_unix = 3;
}
```

这份 contract 直接表达了四种调用模式：`GetOrder` 是 unary；`WatchOrders` 是 server streaming；`ImportOrders` 是 client streaming；`SyncOrders` 是 bidirectional streaming。streaming 是 service definition 的一部分，client/server code generation、message ordering、deadline 和 cancellation 都围绕同一个 RPC model 工作。

gRPC 的另一个优势是 contract 不容易被实现细节悄悄改变。修改 request 或 response 必须先修改 `.proto`，CI 可以对 schema 做 breaking-change 检查。但 Protobuf evolution 也有纪律要求：已发布的 field number 不能重新分配给不同含义，删除字段后应保留原 number，避免旧数据或旧 client 被错误解析。

## JSON 与 Protobuf：不要只看 payload 大小

JSON 是 text format，人可以直接阅读，browser、日志系统和命令行工具都能轻松处理。代价是 field name 会重复出现在 payload 中，类型系统相对有限，而且不同语言对 number、null、missing field 和 timestamp 的处理容易产生边界差异。

Protobuf 使用 field number 编码，payload 通常更紧凑，并从 schema 生成 strongly typed language binding。它适合大量结构化小消息和跨语言 service contract，但 binary payload 本身不可读，没有对应 descriptor 或 `.proto` 就不能完整解释数据。这会把调试能力从“直接看文本”转移到 reflection、schema registry 和专用 client 工具上。

“gRPC 一定比 REST 快”是一个不可靠的结论。Protobuf serialization、较小 payload、HTTP/2 multiplexing 和 persistent channel 可能显著降低高频小请求的开销，但真实 latency 还取决于 TLS、proxy hop、compression、message size、connection management 与业务处理时间。选型前应该使用自己的 payload 和 traffic pattern 做 benchmark，而不是引用脱离环境的倍数。

对于超大文件，gRPC 也不是天然更优。Protobuf message 通常在内存中完成 serialization，各语言实现还可能设置 message size limit。镜像、备份和大对象更适合走 object storage + signed URL 或标准 HTTP streaming；必须通过 gRPC 传输时，应显式设计 chunking、flow control、checksum 和 retry semantics。

## Streaming 与 Browser compatibility

说“REST 只支持 unary”并不严谨。HTTP response body 本身可以 streaming，应用也可以结合 Server-Sent Events 或 WebSocket 实现实时通信；区别在于 REST resource model 没有像 gRPC service definition 那样，把四类 stream 统一成一等 RPC contract。

Browser 是两者差距最明显的边界。REST/JSON 可以直接通过 `fetch` 调用，并天然进入 browser DevTools、CORS 和 HTTP cache 体系。browser 无法直接使用原生 gRPC 所需的全部 HTTP/2 能力，因此通常需要 gRPC-Web client 和 Envoy 等 proxy；官方 gRPC-Web 当前支持 unary 和有限的 server streaming，不支持 client streaming 与 bidirectional streaming。

如果主要 consumer 是 browser 或外部合作方，直接暴露 REST 往往更省成本。若内部已经统一使用 gRPC，可以在边界增加 API gateway、gRPC-Web proxy 或 JSON transcoding，但要明确 authentication、error mapping、timeout 和 versioning 到底由哪一层负责。

## 运维视角：gRPC 的成本常被低估

REST 基于通用 HTTP semantics，大部分 ingress、WAF、API gateway、CDN 和 observability platform 都能直接理解 method、path 和 status code。出现故障时，使用 `curl -v`、抓取 access log 或查看 browser network panel 就能完成第一轮定位。

gRPC 则要求链路中的 load balancer、service mesh 和 ingress 正确支持 HTTP/2、TLS ALPN、trailers 和长连接。监控不仅要看 HTTP status，还要提取 gRPC status；路由通常围绕 `/package.Service/Method`；持续存在的 HTTP/2 connection 也会改变 connection-level load balancing 的效果。

生产 gRPC client 必须显式设置 deadline，并设计 cancellation propagation。retry 只能用于确认安全的操作，还要配置 retry budget 和 backoff，否则一次局部故障可能被 client amplification 放大成 retry storm。streaming RPC 还需要考虑 flow control、慢 consumer、连接中断后的 resume position，以及滚动发布时如何 graceful drain。

这些都不是 gRPC 的缺点，而是它把更多 distributed systems 能力带进了调用框架。团队如果没有相应的 tracing、metrics、reflection、health checking 和 schema governance，gRPC 的理论收益会被运维复杂度抵消。

## 到底该怎么选

| 场景 | 建议 | 主要原因 |
|---|---|---|
| Public API、第三方 integration | REST + JSON + OpenAPI | 兼容性、可发现性和工具生态更好 |
| Browser 或 mobile backend API | 默认 REST；明确需要时使用 gRPC-Web | 直接调用和调试成本更低 |
| 内部微服务、高频小消息 | gRPC | 强 contract、code generation、连接复用 |
| 多语言 service-to-service | gRPC | `.proto` 作为跨语言 source of truth |
| 双向实时数据流 | 原生 gRPC streaming | streaming contract 与 flow control 更完整 |
| CDN cacheable content | REST/HTTP | 可以利用标准 cache semantics 与中间节点 |
| 大文件上传下载 | HTTP/object storage | streaming、range、重试和 CDN 支持更成熟 |
| 团队 HTTP/2 可观测性不成熟 | 先使用 REST | 降低基础设施与故障定位成本 |

一个常见且实用的组合架构如下：

```text
Browser / Partner / CLI
          │
          │ REST + JSON
          ▼
     API Gateway / BFF
          │
          │ gRPC
          ▼
  Internal Services ─── gRPC streaming ─── Data Services
```

这种架构让外部 consumer 获得稳定、通用的 HTTP API，同时让内部 service 使用强类型 contract 和 streaming。但 gateway 不是免费的：REST 与 gRPC 的 error model、pagination、field naming 和 timeout 必须有确定映射，最好从同一份 contract 生成或在 CI 中验证，避免维护两套逐渐漂移的 API。

## 总结

REST 的核心优势是通用性：resource-oriented semantics、browser 原生支持、HTTP cache 与成熟的中间件生态。gRPC 的核心优势是确定性：`.proto` contract、generated stub、compact binary message，以及四种一等 streaming mode。

我的默认原则很简单：系统边界优先 REST，系统内部按收益选择 gRPC；没有明确的 contract、streaming 或吞吐需求，就不要只为“性能更高”引入 gRPC。反过来，如果内部已经有成熟的 HTTP/2、observability 和 schema governance，继续手写大量脆弱的 JSON client 也没有必要。

## 参考资料

- [Kreya：A detailed comparison of REST and gRPC](https://kreya.app/blog/rest-vs-grpc/)
- [gRPC Introduction](https://grpc.io/docs/what-is-grpc/introduction/)
- [gRPC Core concepts](https://grpc.io/docs/what-is-grpc/core-concepts/)
- [Protocol Buffers Overview](https://protobuf.dev/overview/)
- [gRPC-Web](https://github.com/grpc/grpc-web)
- [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
- [RFC 9110：HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
- [RFC 9111：HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html)
