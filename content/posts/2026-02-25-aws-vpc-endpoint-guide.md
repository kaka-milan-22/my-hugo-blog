---
title: "AWS VPC Endpoint 完全指南：运维工程师实战手册"
date: 2026-02-25T17:00:00+08:00
draft: false
tags: ["AWS", "VPC", "Network", "DevOps"]
categories: ["DevOps", "网络"]
author: "Kaka"
description: "从零理解 AWS VPC Endpoint，搞清楚哪些服务需要 Endpoint、该建哪种、怎么配安全组、Private DNS 怎么回事。"
---

## 写在前面

每次新建私网环境，总有人问：

> "EC2 访问 S3，要不要建 Endpoint？"
> "SQS Endpoint 要配什么安全组？"
> "RDS 需要 Endpoint 吗？"

这篇文章从运维视角把这件事讲清楚，附快速判断口诀和生产最小配置清单。

---

## 一、先搞清楚两类服务

AWS 的服务在网络层面分两种，搞清楚这个，后面全部自洽。

### 第一类：数据面在你 VPC 里的服务

这些服务创建时，会让你**选子网、选安全组、分配私网 IP**。

```
EC2 / RDS / ElastiCache / ALB / NLB / EKS Node
```

它们本质是一张 ENI 插在你的 VPC 里，天然就有私网 IP。

```
Client → 私网 IP → 直接访问 ✅
```

**不需要 Endpoint。**

**一句话判断法：** 创建时需要选子网 / 安全组 → 资源在你 VPC 里 → 直连即可。

---

### 第二类：AWS 托管的公共服务（控制面在 AWS 公网）

这些服务没有 VPC 概念，默认走公网：

```
S3 / SQS / SNS / CloudWatch / STS / ECR / Secrets Manager / KMS / DynamoDB
```

如果你的 EC2 在私网（没有 NAT），直接访问这些服务会：

```
EC2 → ❌ 访问失败（没有出路）
```

有 NAT 的话：

```
EC2 → NAT Gateway → 公网 → AWS Public Endpoint → 服务
```

流量出公网，产生 NAT 流量费，而且安全性较差。

**想在私网访问这类服务 → 就需要 VPC Endpoint。**

---

## 二、Endpoint 两种类型，本质完全不同

### Gateway Endpoint（路由型）

类比：**在路由表里加了一条"走内网捷径"的路由。**

| 属性 | 说明 |
|------|------|
| 创建方式 | 修改 Route Table |
| 有没有 ENI | ❌ 无 |
| 有没有安全组 | ❌ 无 |
| 费用 | **免费** |
| 支持服务 | **仅 S3、DynamoDB** |

工作原理：

```
# Route Table 里多了这条
Destination: pl-xxxxxxxx (S3 Prefix List)
Target:      vpce-xxxxxxxxx (Gateway Endpoint)
```

流量依然是 HTTPS，但在 AWS 内网骨干网转发，不出公网。

**适用场景：** EC2 / EKS / Lambda 私网访问 S3，同时省掉 NAT 流量费。

---

### Interface Endpoint（网卡型）

类比：**AWS 在你 VPC 里插了一张网卡，这张网卡就是某个 AWS 服务的私网入口。**

| 属性 | 说明 |
|------|------|
| 创建方式 | 在子网创建 ENI |
| 有没有 ENI | ✅ 有，有私网 IP |
| 有没有安全组 | ✅ 必须配 |
| 费用 | 按小时 + 流量计费 |
| 支持服务 | SQS / SNS / ECR / STS / KMS / CloudWatch 等几乎所有 AWS API |

工作原理：

```
EC2 → 443 → Endpoint ENI (10.x.x.x) → AWS 骨干网 → 服务
```

理解模型：

> Interface Endpoint = AWS 把 SQS 的入口"搬"到你 VPC 里一个私网 IP。

---

## 三、哪些服务需要手动建 Endpoint？

**AWS 不会自动帮你建 Endpoint。**

只要满足以下两个条件，就需要手动建：

1. 服务不在你的 VPC（即第二类服务）
2. 你希望走私网访问

### 几个常见误区

#### 误区 1：RDS 需要 Endpoint？

RDS 创建时要选子网和安全组 → 它就在你 VPC 里 → 直接私网访问，不需要 Endpoint。

#### 误区 2：EC2 内网访问 S3 默认走私网？

❌ 错。S3 默认是公网服务。

- 没有 NAT → 访问直接失败
- 有 NAT → 走公网，产生费用

必须创建 **S3 Gateway Endpoint** 才走私网。

#### 误区 3：EKS 控制台帮我建了 Endpoint，是自动的？

❌ 不是。那是控制台向导替你执行了创建操作，本质仍是你主动触发的，不是 AWS 系统自动维护的。

---

## 四、Interface Endpoint 的安全组怎么配？

这是最容易踩坑的地方。

### Endpoint 安全组规则（入方向）

```
Type:     HTTPS (443)
Protocol: TCP
Port:     443
Source:   Client 的 Security Group ID
```

**注意：Source 填的是安全组 ID，不是 IP 段。**

这是 AWS 基于身份的网络控制模型：

> **安全组的 Source 代表"任何挂了这个 SG 的 ENI"**，包括 EC2、ECS Task、Lambda ENI、EKS Pod ENI 等。

实际上，只要 EC2 / Pod 挂了 Client SG，流量就允许通过，不用管它的私网 IP 是什么。

### 完整示例

假设：EC2 挂了 `sg-app`，Endpoint 挂了 `sg-endpoint`。

```
sg-endpoint Inbound:
  TCP 443  Source: sg-app  ✅
```

```
sg-app Outbound:
  TCP 443  Destination: sg-endpoint  ✅ (也可以开 0.0.0.0/0)
```

---

## 五、Private DNS 是什么？为什么重要？

Interface Endpoint 有一个选项：**Enable Private DNS**。

### 开启后的效果

```
# DNS 解析变化
sqs.ap-southeast-1.amazonaws.com
→ 原来解析公网 IP
→ 开启后解析为 10.x.x.x (Endpoint ENI 私网 IP)
```

客户端代码不用改，SDK 照常访问 `sqs.ap-southeast-1.amazonaws.com`，流量自动走 Endpoint。

### 关闭后的效果

DNS 仍解析到公网 IP，Endpoint 形同虚设，流量还是走公网。

**生产建议：Interface Endpoint 必须开启 Private DNS。**

### 验证方法

在 EC2 上执行：

```bash
# 查看 SQS 域名解析
dig sqs.ap-southeast-1.amazonaws.com

# 看 ANSWER SECTION
# 私网 IP (10.x.x.x) → Endpoint 生效 ✅
# 公网 IP             → 未走 Endpoint ❌
```

---

## 六、快速判断口诀

```
创建时需要子网/SG → 在 VPC → 不需要 Endpoint
创建时只有 ARN/region → 不在 VPC → 需要 Endpoint

S3 / DynamoDB → Gateway Endpoint（免费，改路由）
其它 AWS API  → Interface Endpoint（收费，有 ENI）
```

---

## 七、生产最小 Endpoint 清单

私网计算环境（没有 NAT Gateway）的最基础 Endpoint 配置：

| 服务 | 类型 | 用途 |
|------|------|------|
| S3 | Gateway | 拉配置文件、日志归档 |
| ECR API | Interface | 拉取镜像 API 层 |
| ECR DKR | Interface | 拉取镜像数据层 |
| STS | Interface | SDK 获取临时凭证 |
| CloudWatch Logs | Interface | 写日志 |
| SSM | Interface | Systems Manager |
| EC2Messages | Interface | SSM Session Manager |
| SSMMessages | Interface | SSM Session Manager |

**缺少会报错的典型症状：**

```
缺 ECR → EKS/ECS 拉镜像失败
缺 STS → SDK 报 credential 错误
缺 SSM 相关 → 无法 Session Manager 远程
缺 CloudWatch → 日志出公网或丢失
```

---

## 八、Endpoint Policy：更细粒度的访问控制

Interface Endpoint 和 Gateway Endpoint 都支持 **Endpoint Policy**，可以精细控制谁能通过这个 Endpoint 访问什么资源。

例如，限制只允许访问特定 S3 Bucket：

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-private-bucket",
        "arn:aws:s3:::my-private-bucket/*"
      ]
    }
  ]
}
```

**生产建议：** 至少设置允许访问 Amazon 官方镜像仓库（否则 ECR 会出问题）：

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "*",
      "Resource": "*"
    }
  ]
}
```

默认 Policy 是全允许，安全敏感场景下建议收紧。

---

## 九、费用估算参考

| Endpoint 类型 | 费用构成 |
|--------------|---------|
| Gateway (S3/DynamoDB) | **免费** |
| Interface | $0.01/AZ/小时 + $0.01/GB 流量 |

一个 Interface Endpoint 单 AZ：约 $7.2/月（不含流量）。

多 AZ 高可用部署（3 AZ）：约 $21.6/月，加上流量费。

> 如果没有 NAT Gateway，Interface Endpoint 的费用远低于 NAT 流量费（NAT 约 $0.045/GB）。

---

## 总结一张图

```
你的 VPC
│
├── EC2 / RDS / ElastiCache / ALB
│     └── 直接私网访问，无需 Endpoint
│
├── S3 / DynamoDB
│     └── Gateway Endpoint（路由型，免费）
│
└── SQS / SNS / ECR / STS / CloudWatch / KMS ...
      └── Interface Endpoint（ENI 型，有费用）
            ├── 配安全组（TCP 443，Source = Client SG）
            └── 开 Private DNS（让域名解析到私网）
```

---

## 参考资料

- [AWS VPC Endpoints 官方文档](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html)
- [AWS PrivateLink 定价](https://aws.amazon.com/privatelink/pricing/)
- [Interface Endpoint 支持服务列表](https://docs.aws.amazon.com/vpc/latest/privatelink/aws-services-privatelink-support.html)
