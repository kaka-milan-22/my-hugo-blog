---
title: "agent-browser：AI 时代的浏览器自动化神器，从入门到精通"
date: 2026-03-03T00:00:00+08:00
draft: false
tags: ["ai", "automation", "browser", "devops", "cli", "playwright"]
categories: ["工具"]
description: "深入介绍 Vercel 出品的 agent-browser CLI —— 专为 AI Agent 设计的浏览器自动化工具，涵盖安装、核心命令、Snapshot 工作流、会话管理、网络拦截、iOS 模拟器、云端浏览器等全套用法。"
---

> GitHub: [vercel-labs/agent-browser](https://github.com/vercel-labs/agent-browser) | ⭐ 14k | Apache-2.0

---

## 为什么是 agent-browser？

Selenium 太重、Puppeteer API 太啰嗦、Playwright 对 Shell 脚本不友好……当你需要让 AI Agent 自主操控浏览器时，这些问题被放大了 10 倍：LLM 的上下文有限，每次调用都要序列化整个 DOM，效率极差。

**agent-browser** 是 Vercel Labs 开源的浏览器自动化 CLI，底层基于 Playwright，但做了专门的 AI-first 设计：

- **Rust 原生 CLI** + Node.js Daemon 架构，首命令冷启动，后续命令毫秒响应
- **Accessibility Tree（Snapshot）**代替 DOM，天然适合 LLM 解析
- **`@ref` 机制**让 AI 可以精确操作页面元素，无需反复查询 DOM
- **`--json` 输出**直接给 Agent 消费
- **多 Session、持久 Profile、云端浏览器**一应俱全

---

## 安装

```bash
# 推荐：npm 全局安装（macOS/Linux/Windows 均有原生 Rust 二进制）
npm install -g agent-browser
agent-browser install          # 下载 Chromium（首次）

# macOS 也可用 Homebrew
brew install agent-browser
agent-browser install

# 不想安装？npx 直接跑（走 Node.js，稍慢）
npx agent-browser open example.com
```

Linux 额外装系统依赖：

```bash
agent-browser install --with-deps
```

---

## 快速上手：5 分钟跑通第一个自动化流程

```bash
# 1. 打开页面
agent-browser open https://news.ycombinator.com

# 2. 拍张截图看看
agent-browser screenshot hn.png

# 3. 抓一下可交互元素（Snapshot）
agent-browser snapshot -i

# 4. 用 ref 点击第一条新闻
#    Snapshot 输出中会有 [ref=e1] 这样的标注
agent-browser click @e1

# 5. 查看当前 URL
agent-browser get url

# 6. 关闭
agent-browser close
```

你会看到类似这样的 Snapshot 输出：

```
- link "Ask HN: Who is hiring? (March 2025)" [ref=e1]
- link "Show HN: I built a Rust-based HTTP/3 server" [ref=e2]
- button "more" [ref=e3]
...
```

`@e1`、`@e2` 就是你的"遥控器"，后续命令直接用，不需要再查 DOM。

---

## 架构原理

```
┌──────────────┐   Unix Socket / HTTP   ┌──────────────────────┐
│  Rust CLI    │ ──────────────────────▶│  Node.js Daemon      │
│  (agent-     │                        │  (Playwright + Chrome)│
│   browser)   │ ◀──────────────────────│                      │
└──────────────┘   JSON Response        └──────────────────────┘
```

Daemon 在第一条命令时自动启动，之后常驻内存。每次 CLI 调用只是 IPC 通信，所以链式命令极快：

```bash
agent-browser open example.com && \
  agent-browser wait --load networkidle && \
  agent-browser snapshot -i
```

---

## 核心命令详解

### 导航与等待

```bash
agent-browser open https://example.com        # 打开 URL
agent-browser reload                          # 刷新
agent-browser back / forward                  # 前进后退

# 等待：支持元素、毫秒、文本、URL、JS 条件、加载状态
agent-browser wait "#loading"                 # 等元素出现
agent-browser wait 2000                       # 等 2 秒
agent-browser wait --text "Welcome"           # 等文字出现
agent-browser wait --url "**/dashboard"       # 等 URL 匹配
agent-browser wait --load networkidle         # 等网络空闲
agent-browser wait --fn "window.ready===true" # 等 JS 条件
```

### 元素操作

```bash
# 点击
agent-browser click @e2
agent-browser click "#submit-btn"
agent-browser dblclick ".item"

# 输入
agent-browser fill @e3 "hello@example.com"   # 清空后填入
agent-browser type @e3 "append text"          # 直接追加输入
agent-browser press Enter                     # 按键
agent-browser press "Control+a"               # 组合键
agent-browser keyboard type "raw text"        # 真实键盘事件

# 其他
agent-browser hover @e4
agent-browser check "#agree"
agent-browser select "#country" "China"
agent-browser drag ".src" ".dst"
agent-browser upload "#file" ./report.pdf
```

### 获取信息

```bash
agent-browser get text @e1          # 文本内容
agent-browser get html "#main"      # innerHTML
agent-browser get value "#input"    # 表单值
agent-browser get attr @e1 href     # 属性值
agent-browser get title             # 页面标题
agent-browser get url               # 当前 URL
agent-browser get count ".item"     # 元素数量
agent-browser get box @e1           # 边界框 {x,y,width,height}
```

### 状态检查

```bash
agent-browser is visible @e1        # → true/false
agent-browser is enabled "#btn"
agent-browser is checked "#checkbox"
```

### 语义化查找（Semantic Locators）

比 CSS selector 更语义化，适合不稳定的 DOM 结构：

```bash
agent-browser find role button click --name "Submit"
agent-browser find text "Sign In" click
agent-browser find label "Email" fill "test@test.com"
agent-browser find placeholder "Search..." fill "agent-browser"
agent-browser find testid "login-btn" click
agent-browser find first ".card" click
agent-browser find nth 2 "tr" text      # 第 2 个 tr 的文本
```

---

## Snapshot：AI 工作流的核心

Snapshot 输出的是**无障碍树（Accessibility Tree）**，而非 HTML，大幅减少 token 消耗：

```bash
agent-browser snapshot            # 完整树
agent-browser snapshot -i         # 仅可交互元素（按钮/输入/链接）
agent-browser snapshot -i -c      # 紧凑模式，去掉空结构节点
agent-browser snapshot -d 3       # 最多 3 层深度
agent-browser snapshot -s "#main" # 只看 #main 区域
agent-browser snapshot -i -C      # 包含自定义可点击元素（div onclick 等）
```

**推荐的 AI Agent 工作流：**

```bash
# Step 1: 导航
agent-browser open https://app.example.com/login

# Step 2: 等待加载完成
agent-browser wait --load networkidle

# Step 3: 获取 JSON 格式 Snapshot，喂给 LLM
agent-browser snapshot -i --json

# LLM 分析 snapshot，确定 ref → 执行操作
agent-browser fill @e3 "admin@example.com"
agent-browser fill @e4 "password123"
agent-browser click @e5

# Step 4: 页面变化后重新 snapshot
agent-browser wait --url "**/dashboard"
agent-browser snapshot -i --json
```

`--json` 输出结构：

```json
{
  "success": true,
  "data": {
    "snapshot": "- textbox \"Email\" [ref=e3]\n- textbox \"Password\" [ref=e4]\n...",
    "refs": {
      "e3": { "role": "textbox", "name": "Email" },
      "e4": { "role": "textbox", "name": "Password" }
    }
  }
}
```

---

## 截图与 PDF

```bash
# 截图
agent-browser screenshot                     # 视口截图，存临时目录
agent-browser screenshot page.png            # 指定路径
agent-browser screenshot --full page.png     # 全页截图
agent-browser screenshot --annotate          # 带编号标注（方便视觉模型）

# PDF
agent-browser pdf report.pdf
```

`--annotate` 截图会在每个交互元素上标注编号，非常适合视觉多模态模型：

```bash
agent-browser open https://example.com
agent-browser screenshot --annotate --full page_annotated.png
# 然后把图片传给 GPT-4o / Claude Vision
```

---

## Session：多 Agent 并行

每个 session 有独立的浏览器实例、Cookie、存储和历史：

```bash
# 两个 Agent 同时跑，互不干扰
agent-browser --session agent1 open site-a.com
agent-browser --session agent2 open site-b.com

agent-browser --session agent1 snapshot -i
agent-browser --session agent2 snapshot -i

# 查看所有活跃 session
agent-browser session list
# → default
# → agent1
# → agent2

# 环境变量方式
AGENT_BROWSER_SESSION=agent1 agent-browser click @e1
```

---

## 持久化 Profile：登录一次，永久复用

```bash
# 第一次：登录并保存状态
agent-browser --profile ~/.profiles/myapp open https://myapp.com/login
agent-browser --profile ~/.profiles/myapp fill @e1 "user@example.com"
agent-browser --profile ~/.profiles/myapp fill @e2 "password"
agent-browser --profile ~/.profiles/myapp click @e3
agent-browser --profile ~/.profiles/myapp wait --url "**/dashboard"

# 以后直接用，Cookie/LocalStorage 都在
agent-browser --profile ~/.profiles/myapp open https://myapp.com/dashboard

# 配置文件方式，项目内新建 agent-browser.json
# { "profile": "./browser-data" }
```

Profile 存储：Cookie、LocalStorage、IndexedDB、Service Worker、Cache。

---

## 网络拦截与 Mock

非常适合测试场景，也可以用来阻断广告请求：

```bash
# 拦截并中止请求（屏蔽广告）
agent-browser network route "**/*.doubleclick.net*" --abort

# Mock API 响应
agent-browser network route "https://api.example.com/user" \
  --body '{"id":1,"name":"Mock User","role":"admin"}'

# 查看当前请求日志
agent-browser network requests
agent-browser network requests --filter "/api/"

# 取消拦截
agent-browser network unroute
```

---

## Cookie 与 Storage 管理

```bash
# Cookie
agent-browser cookies                        # 查看所有
agent-browser cookies set token "abc123"     # 设置
agent-browser cookies clear                  # 清空

# localStorage
agent-browser storage local                  # 查看全部
agent-browser storage local "auth_token"     # 读取指定 key
agent-browser storage local set "theme" "dark"
agent-browser storage local clear

# sessionStorage 同理
agent-browser storage session set "cart" '{"items":[]}'
```

---

## 多标签页 & 弹窗处理

```bash
# 标签页
agent-browser tab                  # 列出所有 tab
agent-browser tab new              # 新建空白 tab
agent-browser tab new https://example.com
agent-browser tab 2                # 切换到第 2 个 tab
agent-browser tab close            # 关闭当前 tab

# Frame（iframe）
agent-browser frame "#payment-iframe"  # 切入 iframe
agent-browser fill @e1 "4111111111111111"
agent-browser frame main               # 切回主 frame

# Dialog（alert/confirm/prompt）
agent-browser dialog accept
agent-browser dialog accept "My input text"
agent-browser dialog dismiss
```

---

## 调试工具

```bash
# 有头模式，看着浏览器操作（调试必备）
agent-browser open example.com --headed

# 高亮元素
agent-browser highlight "#submit"

# 录制 Playwright Trace（可用 trace viewer 回放）
agent-browser trace start ./trace.zip
agent-browser open example.com
agent-browser click @e1
agent-browser trace stop ./trace.zip
# npx playwright show-trace trace.zip

# 录制视频
agent-browser record start ./demo.webm https://example.com
agent-browser click @e1
agent-browser record stop

# 控制台日志 & 页面错误
agent-browser console
agent-browser errors

# 对比 Snapshot 差异
agent-browser diff snapshot   # 当前 vs 上次

# 执行任意 JS
agent-browser eval "document.title"
agent-browser eval "window.scrollTo(0, document.body.scrollHeight)"
```

---

## CDP 模式：接管已有 Chrome

```bash
# 先启动 Chrome（带调试端口）
# macOS:
open -a "Google Chrome" --args --remote-debugging-port=9222

# 连接
agent-browser connect 9222
agent-browser snapshot -i

# 或者每次带 --cdp
agent-browser --cdp 9222 click @e1

# 自动发现（省去端口）
agent-browser --auto-connect snapshot

# 连接远程浏览器（Browserbase / 自建）
agent-browser --cdp "wss://remote.example.com/cdp?token=xxx" snapshot
```

---

## 设备模拟与浏览器设置

```bash
# 模拟手机
agent-browser set device "iPhone 14"
agent-browser open https://example.com
agent-browser screenshot mobile.png

# 设置视口
agent-browser set viewport 1920 1080

# 地理位置
agent-browser set geo 22.3193 114.1694   # 香港

# 暗黑模式
agent-browser set media dark

# 断网测试
agent-browser set offline on
agent-browser open https://example.com
agent-browser screenshot offline.png
agent-browser set offline off

# 自定义 User-Agent
agent-browser --user-agent "MyBot/1.0" open https://example.com
```

---

## iOS 模拟器：真实 Mobile Safari 测试

> 需要 macOS + Xcode + Appium

```bash
# 安装 Appium
npm install -g appium
appium driver install xcuitest

# 列出可用设备
agent-browser device list

# 在 iPhone 16 Pro 上跑 Safari
agent-browser -p ios --device "iPhone 16 Pro" open https://example.com
agent-browser -p ios snapshot -i
agent-browser -p ios tap @e1
agent-browser -p ios fill @e2 "search query"
agent-browser -p ios swipe up
agent-browser -p ios screenshot ios_test.png
agent-browser -p ios close

# 环境变量方式
export AGENT_BROWSER_PROVIDER=ios
export AGENT_BROWSER_IOS_DEVICE="iPhone 16 Pro"
agent-browser open https://example.com
```

---

## 云端浏览器集成

在 serverless 环境（CI/CD、Lambda、Vercel Function）里跑时，本地没有浏览器，用云端方案：

### Browserbase

```bash
export BROWSERBASE_API_KEY="your-key"
export BROWSERBASE_PROJECT_ID="your-project"
agent-browser -p browserbase open https://example.com
```

### Browser Use

```bash
export BROWSER_USE_API_KEY="your-key"
agent-browser -p browseruse open https://example.com
```

### Kernel（支持 Stealth 模式 + 持久 Profile）

```bash
export KERNEL_API_KEY="your-key"
export KERNEL_STEALTH=true
export KERNEL_PROFILE_NAME="myapp-profile"
agent-browser -p kernel open https://example.com
```

---

## Auth Vault：团队共享登录状态

```bash
# 保存认证 profile（会打开浏览器让你手动登录）
agent-browser auth save github --url https://github.com

# 也可以直接传账密（适合 CI）
agent-browser auth save myapp \
  --url https://app.example.com \
  --username admin@example.com \
  --password-stdin <<< "$APP_PASSWORD"

# 使用已保存的认证
agent-browser auth login github
agent-browser open https://github.com/settings/profile
agent-browser snapshot -i

# 管理
agent-browser auth list
agent-browser auth show github
agent-browser auth delete github
```

---

## 配置文件

项目根目录放 `agent-browser.json`，一次配置，全员复用：

```json
{
  "headed": false,
  "proxy": "http://localhost:7890",
  "profile": "./browser-data",
  "session": "default",
  "ignoreHttpsErrors": true
}
```

优先级从低到高：`~/.agent-browser/config.json` → `./agent-browser.json` → 环境变量 → CLI flags。

---

## 在 CI/CD 中使用

### GitHub Actions

```yaml
name: E2E Test

on: [push]

jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install agent-browser
        run: |
          npm install -g agent-browser
          agent-browser install --with-deps
      
      - name: Run tests
        run: |
          agent-browser open https://staging.example.com
          agent-browser wait --load networkidle
          agent-browser snapshot -i --json > snapshot.json
          agent-browser find role button click --name "Login"
          agent-browser fill @e1 "$TEST_USER" 
          agent-browser fill @e2 "$TEST_PASS"
          agent-browser click @e3
          agent-browser wait --url "**/dashboard"
          agent-browser screenshot dashboard.png
        env:
          TEST_USER: ${{ secrets.TEST_USER }}
          TEST_PASS: ${{ secrets.TEST_PASS }}
      
      - uses: actions/upload-artifact@v4
        with:
          name: screenshots
          path: "*.png"
```

### Docker

```dockerfile
FROM node:20-slim

RUN npm install -g agent-browser
RUN agent-browser install --with-deps

WORKDIR /app
COPY . .

CMD ["agent-browser", "open", "https://example.com"]
```

---

## 与 AI Agent 集成

### 最简方式：直接告诉 Agent

```
用 agent-browser 测试登录流程。先运行 agent-browser --help 看可用命令。
```

### 给 Claude Code / Cursor / Copilot 加技能

```bash
npx skills add vercel-labs/agent-browser
```

### 写入 CLAUDE.md / AGENTS.md

```markdown
## Browser Automation

使用 `agent-browser` 进行 Web 自动化。

核心工作流：
1. `agent-browser open <url>` — 导航到页面
2. `agent-browser snapshot -i` — 获取交互元素 ref
3. `agent-browser click @e1` / `fill @e2 "text"` — 用 ref 交互
4. 页面变化后重新 snapshot
5. 使用 `--json` 获取结构化输出

调试：加 `--headed` 看可见浏览器窗口
```

### Python 调用示例

```python
import subprocess
import json

def snapshot(session="default"):
    result = subprocess.run(
        ["agent-browser", "--session", session, "snapshot", "-i", "--json"],
        capture_output=True, text=True
    )
    return json.loads(result.stdout)

def click(ref, session="default"):
    subprocess.run(
        ["agent-browser", "--session", session, "click", f"@{ref}"],
        check=True
    )

# 使用
data = snapshot()
refs = data["data"]["refs"]
print(refs)  # {'e1': {'role': 'button', 'name': 'Login'}, ...}
click("e1")
```

### Go 调用示例

```go
package main

import (
    "encoding/json"
    "fmt"
    "os/exec"
)

func snapshot(session string) (map[string]any, error) {
    out, err := exec.Command(
        "agent-browser", "--session", session,
        "snapshot", "-i", "--json",
    ).Output()
    if err != nil {
        return nil, err
    }
    var result map[string]any
    return result, json.Unmarshal(out, &result)
}

func main() {
    exec.Command("agent-browser", "open", "https://example.com").Run()
    data, _ := snapshot("default")
    fmt.Println(data)
}
```

---

## 实战案例

### 案例 1：自动化登录并抓取数据

```bash
#!/bin/bash
set -e

# 登录
agent-browser open https://app.example.com/login
agent-browser wait --load networkidle
agent-browser find label "Email" fill "admin@example.com"
agent-browser find label "Password" fill "secret"
agent-browser find role button click --name "Sign In"
agent-browser wait --url "**/dashboard"

# 截图存档
agent-browser screenshot --full dashboard_$(date +%Y%m%d).png

# 抓取数据
agent-browser find role link click --name "Reports"
agent-browser wait --load networkidle
agent-browser get text "#report-summary" > report.txt

echo "Done: $(cat report.txt)"
agent-browser close
```

### 案例 2：Mock API 做前端测试

```bash
# 模拟后端返回 500 错误
agent-browser network route "https://api.example.com/data" \
  --body '{"error":"Internal Server Error","code":500}'

agent-browser open https://app.example.com
agent-browser wait --load networkidle
agent-browser screenshot error_state.png

# 模拟正常数据
agent-browser network unroute
agent-browser network route "https://api.example.com/data" \
  --body '{"items":[{"id":1,"name":"Test Item"}]}'

agent-browser reload
agent-browser screenshot normal_state.png
```

### 案例 3：对比两个页面的差异

```bash
agent-browser diff url \
  https://staging.example.com \
  https://production.example.com
```

### 案例 4：实时流式预览（pair browsing）

```bash
# 启动带 WebSocket 流的浏览器
AGENT_BROWSER_STREAM_PORT=9223 agent-browser open https://example.com

# 连接 ws://localhost:9223 就能实时看到浏览器画面
# 并可以注入鼠标/键盘事件，人机协作
```

---

## 常见问题

**Q: 和 Playwright 直接用有什么区别？**

Playwright 是 Node.js 库，适合代码级集成；agent-browser 是 CLI，适合 Shell 脚本、AI Agent 调用、快速原型。底层共享 Playwright，能力相同。

**Q: 多个命令共享同一个浏览器吗？**

是的。同一 session 下的命令都复用同一个 Daemon 进程，不会反复冷启动浏览器。

**Q: 能用自己已有的 Chrome 吗？**

可以。`--cdp 9222` 连接正在运行的 Chrome，或 `--auto-connect` 自动发现。

**Q: 支持 Firefox / WebKit 吗？**

通过 Daemon 支持，CLI 层目前主要是 Chromium，但 Playwright 底层具备多引擎能力。

**Q: 生产环境如何部署？**

推荐使用 Browserbase / Browser Use / Kernel 等云端浏览器方案，避免在 serverless 环境安装 Chromium。

---

## 总结

| 场景 | 推荐用法 |
|------|----------|
| AI Agent 自动化 | `snapshot -i --json` + `@ref` 操作 |
| E2E 测试 | `find role/label` 语义定位 + `--json` 断言 |
| 调试排查 | `--headed` + `trace start/stop` |
| CI/CD 无头跑 | `install --with-deps` + 云端浏览器 provider |
| 多账号并行 | `--session` 隔离 |
| 登录态复用 | `--profile` 持久化 |
| 接管现有 Chrome | `--cdp` / `--auto-connect` |
| iOS 真实 Safari | `-p ios` + Appium |

agent-browser 把浏览器操作降维成了 Unix 命令，正好卡在"CLI 够用"和"需要 AI 理解"的交叉点。对于 DevOps / 后端工程师来说，这才是最舒适的姿势。

---

*项目地址：[github.com/vercel-labs/agent-browser](https://github.com/vercel-labs/agent-browser)*
*官网文档：[agent-browser.dev](https://agent-browser.dev)*
