---
title: "浏览器是如何既保护又泄漏你的隐私的？"
date: 2024-01-25T10:00:00+08:00
draft: false
tags: ["隐私", "Cookie", "浏览器指纹", "安全", "Web"]
categories: ["深度解析"]
author: "Kaka"
description: "从 Cookie 的诞生、第三方 Cookie 的滥用，到几乎无法阻止的浏览器指纹识别，这是一部浏览器隐私的演化史——技术如何在便利与监控之间反复拉锯。"
toc: true
---

## 一个让你细思极恐的实验

打开浏览器，访问任意一个购物网站，搜索"跑步鞋"。

然后，打开另一个标签页，去刷一下新闻网站。

大概率，你会在新闻页面的广告位上看到跑步鞋。

关掉广告拦截插件，清空 Cookie，换一个隐身窗口，再试一次。

你可能仍然会看到跑步鞋的广告。

不是魔法，是技术。理解这一切如何运作，需要从 30 年前一个叫 Lou Montulli 的程序员说起。

---

## Part 1：Cookie 的诞生——一个善意的发明

### HTTP 是健忘的

1994 年，Web 刚刚起步。HTTP 协议有一个根本性的特点：**无状态（stateless）**。

每一次请求对服务器来说都是全新的陌生人。你登录了网站，下一秒点击另一个链接，服务器完全不记得你是谁。这就像每次走进一家便利店，店员都对你没有任何印象——即使你刚刚在这里买了东西。

这对购物车来说是个灾难。你把商品加入购物车，跳转到结账页，购物车空了。

### Montulli 的解法：在客户端存一个便签

Netscape 的工程师 Lou Montulli 提出了 Cookie 的概念。思路非常简单：

> 服务器在响应里塞一个小纸条，浏览器把它存起来，下次请求时把纸条带回去。

```
第一次访问：
浏览器 → 服务器：GET /shop
服务器 → 浏览器：HTTP 200 OK
         Set-Cookie: cart_id=abc123; session_id=xyz789

之后每次访问：
浏览器 → 服务器：GET /checkout
         Cookie: cart_id=abc123; session_id=xyz789
                 ↑ 浏览器自动带上，服务器认出你了
```

这个机制在 1994 年被 Netscape 实现，1997 年成为 RFC 标准（RFC 2109）。

Cookie 本身是中性的工具。它让登录状态得以保持，让购物车不会消失，让网站记住你的语言偏好。**技术本身不坏，问题出在使用方式上。**

### Cookie 的几个关键属性

理解 Cookie 的行为，要看它的属性：

| 属性 | 含义 |
|------|------|
| `Domain` | 哪些域名可以读取这个 Cookie |
| `Path` | 哪些路径下发送 Cookie |
| `Expires / Max-Age` | Cookie 存活时间，Session Cookie 关闭浏览器即消失 |
| `Secure` | 只在 HTTPS 连接中发送 |
| `HttpOnly` | JavaScript 无法读取，防 XSS 窃取 |
| `SameSite` | 跨站请求是否携带（后面会细讲） |

设计上，Cookie 有一个重要的安全边界：**同源策略（Same-Origin Policy）**。`news.com` 设置的 Cookie，`shop.com` 读不到。浏览器严格隔离不同来源的数据。

这个边界保护了你……直到有人想出了绕过它的办法。

---

## Part 2：第三方 Cookie——广告业的"特洛伊木马"

### 一个页面可以加载来自任意域名的资源

同源策略保护的是 **JavaScript 读写**，但它并不阻止页面**加载来自其他域名的内容**。

一个 `news.com` 的页面，可以包含：
- 来自 `cdn.news.com` 的图片
- 来自 `analytics.google.com` 的脚本
- 来自 `doubleclick.net` 的广告图片

当浏览器加载 `doubleclick.net/ad.gif` 这张图片时，它会向 `doubleclick.net` 发送一个请求——这个请求会带上 `doubleclick.net` 域名下的所有 Cookie。

关键来了：`doubleclick.net` 的图片可以在响应头里设置 Cookie：
```
Set-Cookie: uid=user_12345; Domain=doubleclick.net; Expires=2025-01-01
```

这就是**第三方 Cookie（Third-Party Cookie）**：由**不是当前访问网站**的第三方域名设置和读取的 Cookie。

### 跨站追踪的完整链条

想象 DoubleClick（Google 的广告网络）把自己的广告代码嵌入到全球数十万个网站中。

```
你访问 news.com
  → 页面加载 doubleclick.net/ad
  → DoubleClick 设置 Cookie: uid=U001
  → DoubleClick 记录：U001 在 09:00 访问了 news.com

你访问 shop.com（同样嵌入了 DoubleClick）
  → 页面加载 doubleclick.net/ad
  → 浏览器自动带上 Cookie: uid=U001
  → DoubleClick 记录：U001 在 09:05 搜索了"跑步鞋"

你访问 weather.com
  → DoubleClick 记录：U001 在 09:10 查看了北京天气

现在 DoubleClick 知道：
  U001 是一个关注时事、对跑步感兴趣、住在北京的人
  → 投放跑步鞋广告
```

整个过程，你没有注册任何账号，没有同意任何条款，但你的浏览行为已经被拼成了一幅精细的画像。

这不是假设。在 2010 年代全盛时期，Facebook 的 Like 按钮、Google Analytics、各种广告网络的追踪像素，覆盖了互联网上超过 70% 的网站。**你每访问 10 个网站，其中 7 个都在悄悄向同一批第三方汇报你的行踪。**

### 浏览器的反击：SameSite 和 ITP

这场追踪游戏终于引起了足够多的反弹。

**Safari 的 ITP（Intelligent Tracking Prevention，2017）**：Apple 率先出手，Safari 开始自动限制第三方 Cookie 的存活时间，后来直接封杀。

**Firefox 的 ETP（Enhanced Tracking Protection，2019）**：默认阻止已知追踪域名的第三方 Cookie。

**Cookie 的 `SameSite` 属性（Chrome 2020）**：

```
SameSite=Strict  // 完全不在跨站请求中发送
SameSite=Lax     // 只在顶级导航（用户点击链接）时发送
SameSite=None    // 允许跨站发送，但必须配合 Secure（HTTPS）
```

Chrome 从 2020 年起将默认值改为 `SameSite=Lax`，这意味着第三方 Cookie 不再默认携带。2024 年，Chrome 宣布逐步废弃第三方 Cookie（虽然因为广告业的压力进度一再推迟）。

第三方 Cookie 的时代正在落幕。但这只是故事的上半场。

---

## Part 3：浏览器指纹——你以为删了 Cookie 就安全了？

2010 年，EFF（电子前哨基金会）发布了一项研究，令人警醒：

> 即使不使用任何 Cookie，仅凭浏览器发送的技术特征，就能在 85% 的情况下唯一识别一个用户。

这就是**浏览器指纹（Browser Fingerprinting）**。

### 你的浏览器在说什么

每次你访问一个网站，浏览器会自动透露大量技术信息：

**HTTP 请求头就泄漏了很多：**
```
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) 
            AppleWebKit/537.36 (KHTML, like Gecko) 
            Chrome/120.0.0.0 Safari/537.36
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
Accept-Encoding: gzip, deflate, br
```

光是这几行，就透露了：你用的是 Mac、Intel 芯片、macOS 10.15.7、Chrome 120，首选语言是中文。

这只是开始。通过 JavaScript，网站还可以探测：

| 信息 | 获取方式 | 独特性 |
|------|----------|--------|
| 屏幕分辨率、色深 | `screen` 对象 | 中 |
| 已安装字体列表 | Canvas/SVG 渲染测试 | **高** |
| 时区 | `Intl.DateTimeFormat` | 低（但与其他组合后高） |
| 硬件并发数（CPU核心） | `navigator.hardwareConcurrency` | 中 |
| 设备内存 | `navigator.deviceMemory` | 低 |
| 支持的 MIME 类型 | `navigator.mimeTypes` | 中 |
| 触摸屏支持 | `navigator.maxTouchPoints` | 低 |
| WebGL 渲染器型号 | WebGL API | **极高** |
| Canvas 像素级渲染差异 | Canvas API | **极高** |
| 音频处理特征 | AudioContext API | **高** |

### Canvas 指纹：最精妙的追踪技术

Canvas 指纹是最难防御的一种。原理如下：

1. JavaScript 在一个不可见的 `<canvas>` 元素上绘制相同的文本和图形
2. 导出为 Base64 图片字符串
3. 对这个字符串计算哈希值

```javascript
const canvas = document.createElement('canvas');
const ctx = canvas.getContext('2d');

// 绘制特定文字和图形
ctx.textBaseline = 'top';
ctx.font = '14px Arial';
ctx.fillStyle = '#f60';
ctx.fillRect(125, 1, 62, 20);
ctx.fillStyle = '#069';
ctx.fillText('Hello, 🌍', 2, 15);
ctx.fillStyle = 'rgba(102, 204, 0, 0.7)';
ctx.fillText('Hello, 🌍', 4, 17);

// 提取像素数据
const fingerprint = canvas.toDataURL();
// → "data:image/png;base64,iVBORw0KGGoAAAANSUhEUgAA..."
```

**为什么同样的代码，不同设备输出不同的结果？**

因为字体渲染涉及到：操作系统的字体引擎（macOS 用 CoreText，Windows 用 DirectWrite）、安装的字体版本、GPU 驱动的抗锯齿算法、显示器色彩配置文件……这些差异会累积到像素级别，产生独一无二的输出。

你的 MacBook Pro 渲染出来的那张画，和全世界其他任何一台电脑的输出都略有不同。

### WebGL 指纹：问 GPU 要身份证

WebGL 允许网站通过 JavaScript 调用 GPU 进行渲染。通过 WebGL，追踪者可以：

```javascript
const gl = canvas.getContext('webgl');
const renderer = gl.getParameter(
  gl.getExtension('WEBGL_debug_renderer_info').UNMASKED_RENDERER_WEBGL
);
// → "ANGLE (Apple, Apple M2 Pro, OpenGL 4.1)"
```

GPU 型号本身已经很有区分度，而且不同 GPU 在渲染同一个 3D 场景时，像素级别的输出也存在可测量的差异——这就是 **WebGL 渲染指纹**，独特性极高。

### 音频指纹：连声音都不放过

AudioContext API 用于处理音频，但它也被用于指纹识别：

```javascript
const ctx = new AudioContext();
const oscillator = ctx.createOscillator();
const analyser = ctx.createAnalyser();
const gain = ctx.createGain();

oscillator.connect(gain);
gain.connect(analyser);
analyser.connect(ctx.destination);

// 生成特定频率的音调，提取频率响应数据
// 不同硬件的音频处理栈会产生可测量的微小差异
```

这背后的原理与 Canvas 类似：音频处理经过操作系统、驱动、硬件，每一层的微小差异都会累积成独特的"声纹"。

### 指纹的恐怖之处：你无法删除它

Cookie 可以清除。但指纹不存储在你的设备上——**它存在于追踪者的数据库里**，是你设备特征的计算结果。

你无法：
- 删掉它（它不在你这里）
- 修改它（改了一个特征，其他特征还在）
- 逃脱它（换个 IP 没用，同一台设备指纹不变）

唯一的防御是：让自己的浏览器变得"平凡"，混入人群。

---

## Part 4：浏览器的保护机制——一场永无止境的军备竞赛

浏览器工程师们并非没有在反击。但每一次防御，都引发了新的绕过方式。

### 防御 1：限制第三方 Cookie（已接近成功）

前面提到，Safari、Firefox、Chrome 都在限制或废弃第三方 Cookie。这是目前进展最显著的方向。

但广告业不会坐以待毙，他们提出了替代方案：
- **FLoC（Federated Learning of Cohorts）**：Google 曾提议把追踪"内化"到浏览器，浏览器本地分析用户行为并给用户打标签，广告主拿到的是"人群标签"而非个人标识。但因为隐私争议太大，2022 年被 Chrome 放弃。
- **Topics API**：FLoC 的继任者，Chrome 目前在推进，让浏览器暴露用户近期感兴趣的"话题类别"，广告商据此投放。争议依然不小。

### 防御 2：Canvas 随机化

Firefox 和 Brave 会在 Canvas 导出的像素数据中加入微小的随机噪声，使得每次调用 `toDataURL()` 的结果都略有不同，破坏指纹的稳定性。

```
原始像素：[255, 128, 64, ...]
加噪后：  [254, 129, 63, ...]  // 微小随机扰动，人眼看不出差异
```

这会让 Canvas 指纹每次都不同，追踪者无法建立稳定的标识。

代价是：某些需要精确像素的 Web 应用（图形编辑器等）可能出现轻微异常。

### 防御 3：统一 User-Agent

各浏览器都在推进 **User-Agent Client Hints**：不再在默认请求头里暴露详细版本信息，只暴露最基础的标识。网站如果需要更多信息，需要明确请求，用户有机会知情。

Chrome 的 User-Agent 也在逐步"冻结"，减少版本号的变化频率，让它的区分度降低。

### 防御 4：权限隔离

现代浏览器把同一个网站的不同第三方脚本放在独立的沙箱中（Site Isolation），限制它们读取彼此的数据。Storage Partitioning（存储分区）把每个第三方在不同顶层网站下的存储隔离开——即使同一个追踪器被嵌入 `news.com` 和 `shop.com`，它在两个站点下设置的数据是隔离的，无法跨站关联。

### 防御 5：Brave 的激进路线

Brave 浏览器走得最远，它默认：
- 阻断第三方 Cookie 和追踪器
- 对 Canvas、WebGL、音频指纹加入随机化
- 对硬件 API 返回模糊值（CPU 核心数随机化，内存值随机化）
- 屏蔽 CNAME 伪装追踪（通过 DNS 别名绕过第三方限制）

但即便如此，指纹识别依然无法完全消除——只能增加追踪者的成本和不确定性。

---

## Part 5：真正的隐私，有多难？

来做一个清醒的盘点。

### 隐身模式能保护你吗？

**对本地隐私：是。** 隐身模式不保存浏览历史、Cookie、表单数据。关闭窗口后，本地痕迹消除。

**对网络追踪：否。** 你的 IP 地址、浏览器指纹，在隐身模式下完全不变。网站、广告商、ISP 照样能看到你的访问。

隐身模式保护的是"不让你的家人看到你浏览了什么"，不保护"不让互联网公司追踪你"。

### VPN 能保护你吗？

**对 IP 追踪：部分。** VPN 隐藏了你的真实 IP，服务器看到的是 VPN 提供商的 IP。

**对指纹追踪：否。** 换了 IP，你的 Canvas 指纹、WebGL 指纹、字体列表一个都没变。换个出口，指纹依旧唯一。

**代价：** 你把对 ISP 的信任转移给了 VPN 提供商。很多免费 VPN 本身就是数据收集工具。

### Cookie 同意弹窗真的保护你吗？

欧盟 GDPR 要求网站在放置非必要 Cookie 前获得用户同意，于是你看到了满网络的 Cookie 横幅。

但研究发现，大量网站的同意界面存在"黑暗模式（Dark Pattern）"：
- 同意按钮醒目，拒绝按钮藏在深处
- 默认勾选所有追踪选项
- 点击"拒绝"需要多步操作，点击"同意"只需一步

一项 2020 年的研究对 28000 个网站的 Cookie 横幅进行了分析，发现超过 95% 的横幅存在至少一种让用户难以拒绝的设计。

### 什么才真正有效？

如果你认真对待自己的网络隐私，有效的措施大致按效果排序：

1. **使用 Firefox + uBlock Origin**：uBlock Origin 是目前最有效的内容拦截器，能阻断绝大多数追踪脚本在运行之前就被执行。
2. **使用 Brave 浏览器**：内置指纹随机化和追踪阻断，无需安装插件。
3. **Firefox 的 `privacy.resistFingerprinting`**：在 about:config 中开启，Firefox 会主动伪装大量指纹特征，让你看起来像一个"通用的 Firefox 用户"。
4. **Tor 浏览器**：所有用户的指纹被强制统一，流量通过多跳匿名，是目前最强的浏览器隐私保护方案——但速度慢，很多网站会触发验证码。
5. **DNS over HTTPS（DoH）**：防止 ISP 从 DNS 查询记录中推断你访问了哪些网站。

---

## 尾声：技术中性，选择不中性

Cookie 被发明时，Lou Montulli 没有想到它会成为互联网最大的隐私问题之一。他只是想解决一个工程问题：让服务器记住购物车。

第三方 Cookie 的滥用、浏览器指纹的兴起，不是某个邪恶天才的阴谋，而是**无数工程师在追求商业目标时，在现有技术边界上一点一点推进的结果**。每一步单独来看都是"技术上可行的优化"，累积起来就成了今天这张覆盖全网的追踪网络。

浏览器工程师也在持续反击，每一次防御都真实地保护了一部分用户。但保护总是滞后于追踪——毕竟，追踪是有商业动力的，防御是一种公共品。

你能做的，是了解这些机制如何运作，在知情的情况下做出自己的选择。

这比任何技术方案都重要。

---

## 延伸阅读

- [EFF Panopticlick / CoverYourTracks](https://coveryourtracks.eff.org/) — 测试你的浏览器有多容易被追踪
- [RFC 6265](https://datatracker.ietf.org/doc/html/rfc6265) — HTTP State Management Mechanism（Cookie 标准）
- Eckersley, P. (2010), *[How Unique Is Your Web Browser?](https://coveryourtracks.eff.org/static/browser-uniqueness.pdf)* — 浏览器指纹研究原始论文
- [GDPR 与黑暗模式研究](https://arxiv.org/abs/2001.02479) — Nouwens et al., 2020
- [Privacy Sandbox](https://privacysandbox.com/) — Google 的后 Cookie 时代广告技术提案（存在争议，值得批判性阅读）
