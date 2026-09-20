---
title: "2025 年的 Self-Hosting：从容器到安全入口的实用工具栈"
date: 2026-09-20T09:30:00+08:00
draft: false
tags: ["Self-Hosting", "Docker", "Podman", "Kubernetes", "Caddy", "Homelab"]
categories: ["DevOps", "Tools", "网络"]
author: "Kaka"
description: "围绕容器运行时、Web 管理、reverse proxy、VPN、监控与通知，梳理一套克制、可维护的现代 Self-Hosting 工具栈，并补上安全和备份基线。"
---

## 引言

Self-Hosting 的吸引力很直接：数据和服务掌握在自己手里，不必把照片、密码、书签、监控数据和自动化流程全部交给 SaaS。真正困难的却不是“把服务跑起来”，而是半年后仍能更新、排障、恢复，并且不让一个暴露在公网的应用成为家庭网络的入口。

[Self-Hosting Like It's 2025](https://web.archive.org/web/20250401115923/https://kiranet.org/posts/self-hosting-like-its-2025/) 从 Homelab 使用者的角度盘点了 Docker、Podman、Kubernetes、Portainer、Dockge、Pangolin、Caddy、NetBird、Uptime Kuma 和 Gotify 等工具。本文沿着这条主线重新整理，并加入运维上不可省略的安全、更新和 backup 约束。它不是一份“最佳软件排行榜”，而是一套降低长期维护成本的选择方法。

## 先定义 Self-Hosting 的目标

Self-Hosting 至少包含三种不同需求：仅供家庭 LAN 使用的服务、需要人在外部访问的私有服务，以及任何人都能访问的公开网站。它们的网络入口和风险模型完全不同。

```text
仅限 LAN                 私有远程访问                 公开服务

Client ──> Service      Client ── VPN/Mesh ──> App   Internet ──> HTTPS Proxy ──> App
无需公网入口              默认不公开 origin             必须持续修补公网攻击面
```

如果服务只给自己和家人使用，优先让它留在私网，通过 WireGuard、NetBird 或 Tailscale 一类 overlay network 访问。只有确实需要匿名公网访问时，才开放 HTTPS 入口。不要因为 reverse proxy 配置方便，就把整个 Homelab 变成 public attack surface。

## 容器运行时：选最小的复杂度

### Docker：生态和可复制性优先

Docker 的最大优势不是技术最先进，而是它已经成为 Self-Hosting 软件事实上的交付格式。大量项目直接提供 `compose.yaml`，文档、示例和故障经验也最多。对于单机 Homelab，Docker Compose 通常已经足够：配置可以进入 Git，volume 和 network 清晰可见，迁移时也不依赖某个管理界面的内部数据库。

```yaml
services:
  app:
    image: ghcr.io/example/app:1.4.2
    restart: unless-stopped
    read_only: true
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    volumes:
      - ./data:/var/lib/app
    networks:
      - backend

networks:
  backend:
    internal: true
```

示例刻意不映射 host port。应用先进入 internal network，再由 reverse proxy 精确接入；同时 pin 明确版本，避免 `latest` 在无人值守更新时引入不可预测的 breaking change。`read_only`、`no-new-privileges` 和 `cap_drop` 不能消除漏洞，但能收窄容器被攻破后的权限。

### Podman：rootless 与 systemd 集成

Podman 提供接近 Docker CLI 的操作体验，不依赖常驻 daemon，并把 rootless container 作为重要使用方式。对希望减少 rootful daemon 权限、或者本来就用 Fedora/RHEL 系生态的人，它很有吸引力。

Quadlet 可以用 systemd unit 风格声明 container、volume、network 和 image，让启动顺序、restart policy、日志与系统服务管理保持一致。代价是兼容性并非绝对：Docker Compose 示例、socket、network 行为以及某些依赖 Docker API 的工具仍需逐项验证。不要仅因为“rootless 更安全”就假设迁移没有成本。

### Kubernetes：为学习或真实的编排需求买单

Kubernetes 能处理声明式部署、service discovery、rolling update、health check、secret/config 分发和多节点调度，但这些能力伴随 control plane、CNI、CSI、Ingress、certificate 和版本升级的维护成本。

单台机器、少量服务、能够接受短暂停机时，Compose 往往更可靠，因为系统中需要理解的状态更少。使用 k3s、k0s 或完整 Kubernetes 的合理理由，是你确实需要多节点调度和统一 reconciliation，或者 Homelab 本身就是学习环境。为了运行十个家庭应用而复制企业平台，通常是在用 orchestration complexity 替代业务复杂度。

| 方案 | 适合场景 | 主要代价 |
|---|---|---|
| Docker Compose | 单机、多数社区应用、快速部署 | rootful 默认值与供应链治理要自行处理 |
| Podman + Quadlet | rootless、systemd-first、RHEL/Fedora 系 | Compose 与第三方工具兼容性需验证 |
| Kubernetes | 多节点、平台学习、真实编排需求 | control plane 和周边组件维护成本高 |

## Web 管理：界面是入口，不是配置的唯一真相

### Portainer

Portainer 覆盖 Docker、Swarm 和 Kubernetes 等环境，功能完整，适合查看 container 状态、日志、image、network 和 volume。它也因此拥有很高的控制权限：管理面一旦暴露或账号失守，攻击者得到的通常不只是某个应用，而是整个 container host。

### Dockge

Dockge 更聚焦 Compose stack，界面和心智模型都更简单。如果目标只是管理一组 `compose.yaml`，它比完整平台更轻。无论选择哪一个，Compose 文件都应保存在普通目录或 Git repository 中，确保离开 Web UI 后仍能用标准 CLI 恢复。

管理面应只通过 LAN 或 VPN 访问，启用 MFA（若产品支持），并避免直接暴露到公网。Web UI 提升的是日常操作体验，不应成为无法重建的 state store。

## Reverse proxy 与 VPN：先选择访问模型

### Pangolin：把公网入口和家庭网络分开

Pangolin 把 tunneled reverse proxy、mesh connectivity、identity access 与管理界面组合起来。典型拓扑是在公网 VPS 运行入口，家中的 connector 主动向外建立连接，因此不必在家庭 router 上做 port forwarding，也不会直接公开 home IP。

```text
Internet
   │
   ▼
Public VPS / Pangolin ── encrypted tunnel ──> Home connector ──> Private app
        │
        └── TLS / identity policy / public ingress
```

这种设计减少了入站网络配置，但没有让安全责任消失。VPS、Pangolin server、connector 和后端应用都要及时更新；identity policy 也不能替代应用自身的 authorization。关键数据仍需独立 backup，不能把 tunnel 可用性误当成 disaster recovery。

### Caddy：配置克制，自动处理 HTTPS

Caddy 适合愿意维护文本配置的人。一个最小 reverse proxy 只需：

```caddyfile
photos.example.com {
    encode zstd gzip
    reverse_proxy photos:8080
}
```

它会处理 certificate issuance 和 renewal，配置也容易进入 Git。缺少 Web UI 不一定是缺点：可 review、可 diff、可从 backup 还原的配置，长期通常比只能点击生成的状态更可靠。

### Nginx Proxy Manager：易上手，但要保留退出路径

Nginx Proxy Manager 用 Web UI 封装 Nginx、certificate 和 access list，适合快速起步。需要注意的是，UI 抽象也会隐藏实际生成的配置；遇到 authentication、WebSocket、TCP/UDP proxy 或 upgrade 问题时，仍然要理解底层 Nginx。部署前应测试 configuration export 和完整恢复，而不只是测试“能否创建 proxy host”。

### NetBird：私有服务默认走 overlay network

NetBird 基于 WireGuard 管理 peer、route 和 access policy，可以自托管 control plane，也可使用其托管服务。它与 Tailscale 属于相似的问题空间：让设备像在同一私网中通信，而不要求每个应用暴露公网端口。

对管理面、NAS、dashboard、SSH 和内部 API，VPN/mesh 应是默认方案。公开 reverse proxy 只服务真正需要公开的站点；这条边界比堆叠更多 WAF rule 更容易推理。

## 监控和通知：先覆盖“服务死了”

Uptime Kuma 的定位非常清楚：用 HTTP、TCP、DNS、Ping 等 probe 判断服务是否可用，并把告警送往常见渠道。它不能替代 Prometheus、Grafana、Loki 或 distributed tracing，但对 Homelab 来说，先可靠回答“服务是否还能被用户访问”，价值往往高于一开始就部署完整 observability stack。

Gotify 提供简单的 self-hosted push notification。应用或监控系统通过 HTTP API 发消息，手机端接收通知。它适合承接 Uptime Kuma、backup job、certificate renewal 和磁盘阈值告警。需要单独考虑一个问题：如果 Gotify 与被监控服务在同一台 host 上，整机断电时通知也发不出去。至少保留一个外部 dead-man check，或者把关键告警通道放到不同 failure domain。

## 原文之外最重要的三条生产基线

### Backup 必须以 restore 为终点

备份 Compose 文件远远不够。真正需要保护的是 application state、database dump、uploaded files、encryption key 和恢复顺序。建议遵循 3-2-1：至少三份副本、两种介质、一份异地，并定期在干净环境执行 restore drill。

对 PostgreSQL、MySQL 等数据库，不要把运行中的 volume 直接压缩后当成一致性备份。使用数据库原生 dump 或物理备份机制，必要时配合 filesystem snapshot。`rclone` 能传输备份，却不会自动创造 application-consistent snapshot。

### 更新要可控，也要及时

自动更新所有 image 看似省事，实际可能在夜间同时升级 schema、runtime 和 application。更稳妥的流程是订阅 release/security notification，固定版本，先备份和阅读 migration note，再按服务逐个更新；高危安全修复则应有快速通道。无论手动还是自动，失败后必须能回滚 image 和 data schema。

### 默认拒绝横向移动

公网应用被攻破应被视为必然会发生的事件，而不是纯理论风险。不同 trust level 的服务不要共享 Docker socket、host network、可写 host path 或同一个高权限 database account。将 IoT、server、管理终端和普通家庭设备放进不同 VLAN 或 policy group，并限制 east-west traffic。container isolation 是一道边界，不是完整 sandbox。

## 一套克制的落地组合

对单机或少量节点的家庭环境，一个可维护的默认方案可以是：Docker Compose 或 Podman Quadlet 运行应用；Caddy 代理少量公开服务；NetBird 一类 mesh network 承载管理面和私有服务；Uptime Kuma 做 availability check；Gotify 接收通知；独立 storage 执行 versioned、off-site backup。

需要 Web 管理时再加入 Dockge 或 Portainer，需要隐藏 home origin 并统一公网入口时再评估 Pangolin。Kubernetes 只有在学习目标或 workload 明确要求时才进入架构。Self-Hosting 的成熟标志不是 dashboard 数量，而是任何一台机器损坏后，你知道数据在哪里、如何重建，以及恢复需要多久。

## 总结

2025 年的 Self-Hosting 工具已经足够友好，难点也从“能否部署”转向“能否长期拥有”。Docker、Podman 和 Kubernetes 解决的是不同规模的运行问题；Portainer、Dockge 改善操作体验；Caddy、Pangolin、NetBird 决定访问边界；Uptime Kuma 与 Gotify 负责发现故障。最终选择应服务于简单、可恢复和最小暴露面，而不是追逐最多的功能。

参考资料：

- [Self-Hosting Like It's 2025（Wayback Machine）](https://web.archive.org/web/20250401115923/https://kiranet.org/posts/self-hosting-like-its-2025/)
- [Awesome-Selfhosted](https://github.com/awesome-selfhosted/awesome-selfhosted)
- [selfh.st](https://selfh.st/)
- [awesome-docker-compose](https://github.com/Haxxnet/Compose-Examples)

> 本文根据原文主题进行中文重写，并加入安全、backup 与恢复视角，不是逐句翻译。工具功能和授权模式会变化，部署前应核对各项目的 current documentation。
