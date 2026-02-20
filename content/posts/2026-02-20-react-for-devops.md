---
title: "写给运维的 React 快速入门"
date: 2026-02-20T22:18:00+08:00
draft: false
tags: ["React", "Frontend", "DevOps"]
categories: ["DevOps"]
description: "从运维视角理解 React 的核心概念与工作原理，不纠结细节，只建立框架认知。"
author: "Kaka"
---

> 你已经能部署 React 项目了，现在我们来搞清楚它到底是怎么运作的。

---

## React 是什么？一句话定位

React 是一个 **UI 构建库**，不是完整框架（这点和 Angular 不同）。它只负责一件事：**把数据渲染成页面，并在数据变化时高效更新页面**。

类比运维视角：React 就像一个 **配置管理工具**，你描述"期望状态"（想要的 UI），它负责让实际状态收敛过去。

```
数据（State）  →  React  →  DOM（页面）
数据变了      →  React  →  只更新变化的部分
```

---

## React 的本质：声明式 UI

React 的核心不只是"把组件封装成函数"，函数只是载体。本质是一个更根本的思想转变：

**UI 是状态的"快照"**

```
UI = f(State)
```

在 React 之前（jQuery 时代），你写的是**命令式**代码：

```js
// 亲自指挥每一步，手动找元素、手动改
const el = document.getElementById('count')
el.innerText = count + 1
```

React 之后，你写的是**声明式**代码：

```jsx
// 只描述"应该长什么样"，不管怎么变过去
return <p>当前计数: {count}</p>
```

**类比运维你秒懂：**

| 时代 | 风格 | 运维类比 |
|------|------|---------|
| jQuery | 命令式：一步步告诉浏览器怎么改 DOM | Shell 脚本：`apt install`、`systemctl start`、`sed -i`... 逐条执行 |
| React | 声明式：描述期望状态，框架负责收敛 | Ansible/K8s：写 YAML 描述期望状态，工具算出 diff 自己执行 |

> **React 本质上就是 UI 领域的「声明式配置管理」。你告诉它 *what*，它负责 *how*。**

---

## 核心概念一：组件（Component）

**组件是 React 的最小单位**，本质就是一个返回 UI 的 JavaScript 函数。

```jsx
// 一个最简单的组件
function ServerCard({ name, status }) {
  return (
    <div>
      <h2>{name}</h2>
      <p>状态: {status}</p>
    </div>
  );
}

// 使用它
<ServerCard name="nginx-prod-01" status="running" />
```

**类比理解：**

| 运维概念 | React 概念 |
|---------|-----------|
| Ansible Role | Component（可复用单元） |
| Role 调用 Role | 组件嵌套组件 |
| 变量（vars） | Props（从外部传入的数据） |
| 本地状态文件 | State（组件内部管理的数据） |

**页面 = 组件树**，一个复杂页面是很多小组件嵌套组合的结果：

```
App
├── Header
│   └── NavMenu
├── Dashboard
│   ├── ServerCard  (复用 × N)
│   └── AlertList
└── Footer
```

---

## 核心概念二：JSX（长得像 HTML 的 JS）

你看 React 代码会发现 JS 里写着 HTML，这叫 **JSX**。它不是真的 HTML，是语法糖，会被编译成 JS 函数调用。

```jsx
// 你写的 JSX
const el = <h1 className="title">Hello</h1>;

// 编译后实际执行的
const el = React.createElement("h1", { className: "title" }, "Hello");
```

**记住两个区别就够了：**

- HTML 用 `class`，JSX 用 `className`（因为 `class` 是 JS 保留字）
- JSX 里用 `{}` 插入 JS 表达式，就像 Shell 里的 `$()`

```jsx
const cpu = 87;
return <p style={{ color: cpu > 80 ? 'red' : 'green' }}>CPU: {cpu}%</p>;
```

---

## 核心概念三：State（状态驱动 UI）

这是 React 最核心的思想：**UI 是 State 的函数**。

```
UI = f(State)
```

State 变 → React 自动重新渲染相关组件 → 页面更新。你不需要手动操作 DOM（不需要 `document.getElementById` 那套）。

```jsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);  // 声明一个状态，初始值 0

  return (
    <div>
      <p>当前计数: {count}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
    </div>
  );
}
```

**工作流程：**

```
用户点击按钮
    ↓
调用 setCount(count + 1)
    ↓
React 检测到 state 变化
    ↓
重新执行组件函数，生成新 UI
    ↓
对比新旧 UI（Virtual DOM Diff）
    ↓
只更新页面上变化的那个数字
```

---

## 核心概念四：Props（组件间通信）

Props 是父组件向子组件传递数据的方式，**单向流动，只读**。

```jsx
// 父组件
function Dashboard() {
  const servers = [
    { id: 1, name: 'nginx-01', status: 'running' },
    { id: 2, name: 'redis-01', status: 'stopped' },
  ];

  return (
    <div>
      {servers.map(s => (
        <ServerCard key={s.id} name={s.name} status={s.status} />
      ))}
    </div>
  );
}

// 子组件（只读取 props，不修改）
function ServerCard({ name, status }) {
  return <div>{name} - {status}</div>;
}
```

**数据流方向：**

```
Dashboard（父）
    ↓  props 向下传
ServerCard（子）
    ↑  事件/回调向上报
```

子组件想"修改"数据？不行，它只能通过调用父组件传下来的**回调函数**来通知父组件去改。

---

## 核心概念五：useEffect（处理副作用）

### 先搞清楚：什么是纯函数？

React 组件本质上应该是一个**纯函数**。

**纯函数 = 同样的输入，永远得到同样的输出，并且不影响外部世界。**

```js
// ✅ 纯函数
function add(a, b) {
  return a + b   // 只依赖入参，不碰外面任何东西
}
add(1, 2)  // 永远是 3，调用100次都是3
```

### 什么是副作用？

**副作用 = 函数做了"返回值以外"的任何事情。**

理解副作用有两个维度，很多人只知道第一个：

**1. 写出去（对外产生影响）**

```js
let count = 0
function fn() { count++ }         // 修改外部变量 → 副作用
function fn() { fetch('/api') }   // 网络请求 → 副作用
function fn() { localStorage.setItem('k', 'v') }  // 写存储 → 副作用
```

**2. 读进来（依赖外部可变状态）**  ← 这个容易被忽略

```js
let multiplier = 2
function fn(x) { return x * multiplier }  // 也是副作用！
// 同样输入 fn(3)，multiplier 被人改过后结果就不同了
// "同输入同输出"的保证被打破了
```

**一刀切的判断方法：**

> 把这个函数调用 100 次，结果和影响完全一样吗？

```js
add(1, 2)        // 100次都是3               ✅ 纯函数
new Date()       // 100次时间都不同           ❌ 副作用
db.insert(user)  // 插了100条数据            ❌ 副作用
```

**类比运维：** 纯函数就像**幂等操作**。`kubectl apply` 同一份 YAML 执行 10 次，结果应该一样——这就是幂等/纯的思想。

### 为什么 React 组件要是纯函数？

React 在某些情况下会**多次调用你的组件函数**（StrictMode 下故意调用两次来检测问题）。

```jsx
// ✅ 纯组件：调用多少次结果都一样
function ServerCard({ name, status }) {
  return <div>{name}: {status}</div>
}

// ❌ 有副作用的组件：第二次渲染结果不一样，出 bug
let renderCount = 0
function ServerCard({ name }) {
  renderCount++              // 修改了外部变量！
  return <div>{name} - 渲染第{renderCount}次</div>
}
// React 多调用一次，UI 就显示错误数字 → 不可预测 = bug 的温床
```

### 副作用跑不掉，用 useEffect 来兜底

副作用本身没有错，**没有副作用的程序是没有意义的程序**。你总要请求 API、写数据库、发日志。

React 的做法是：**把副作用从渲染逻辑里隔离出去，统一放到 useEffect 管理。**

```jsx
import { useState, useEffect } from 'react';

function ServerStatus() {
  const [data, setData] = useState(null);

  // 渲染函数本身：纯的，只管描述 UI ↓
  // 副作用：隔离在 useEffect 里 ↓
  useEffect(() => {
    fetch('/api/servers')          // 网络请求 = 副作用，关进这里
      .then(r => r.json())
      .then(d => setData(d));

    // 返回清理函数（类似 systemd 的 ExecStop）
    return () => {
      console.log('组件卸载，清理资源');
    };
  }, []); // [] 表示只在挂载时执行一次

  return <div>{data ? JSON.stringify(data) : '加载中...'}</div>;
}
```

**`useEffect` 的执行时机由第二个参数控制：**

```jsx
useEffect(() => { ... });          // 每次渲染后都执行
useEffect(() => { ... }, []);      // 只在组件挂载时执行一次
useEffect(() => { ... }, [count]); // count 变化时执行
```

**职责分离总结：**

```
组件函数本身  →  纯函数，只负责"数据 → UI"的映射
useEffect    →  副作用的隔离区，API请求/定时器/DOM操作全放这

                    ┌─────────────────┐
  Props             │                 │
  State    ──────▶  │   组件函数       │  ──────▶  UI 快照
                    │   (纯函数)       │
                    └─────────────────┘
                             │
                       有副作用？
                             │
                    ┌────────▼────────┐
                    │   useEffect     │  ──────▶  API / Timer / DOM
                    └─────────────────┘
```

> 就像 Go 里你会把 I/O 操作推到函数边界，让业务逻辑保持可测试——React 的 useEffect 是同一个思想。

---

## 核心概念六：Virtual DOM（高效更新机制）

React 不直接操作浏览器的真实 DOM（操作真实 DOM 很慢），而是维护一个 **Virtual DOM**（内存中的 JS 对象树）。

```
State 变化
    ↓
生成新的 Virtual DOM 树
    ↓
Diff 算法对比新旧两棵树（找出差异）
    ↓
只把"差异部分"同步到真实 DOM
```

**类比：** 就像 `ansible --check` 先算出 diff，再只执行有变化的 task，而不是每次都重新 apply 全量配置。

---

## 整体架构鸟瞰

```
┌─────────────────────────────────────────────┐
│                  React App                  │
│                                             │
│   ┌──────────┐     ┌──────────────────────┐ │
│   │  State   │────▶│   Component Tree     │ │
│   │ (数据源)  │     │  App                 │ │
│   └──────────┘     │  ├── Header          │ │
│         ▲          │  ├── Main            │ │
│         │          │  │   ├── Card × N    │ │
│   用户交互          │  └── Footer          │ │
│   API 响应         └──────────────────────┘ │
│                            │                │
│                    Virtual DOM Diff         │
│                            │                │
│                     真实 DOM 更新            │
└─────────────────────────────────────────────┘
```

---

## 生态系统速查（知道有这些东西就行）

| 需求 | 工具 | 类比 |
|------|------|------|
| 项目脚手架 | `Vite` / `Create React App` | `ansible-galaxy init` |
| 路由（多页面） | `React Router` | nginx location 路由 |
| 全局状态管理 | `Redux` / `Zustand` | 配置中心 Consul/etcd |
| 请求 API | `axios` / `React Query` | curl / httpie |
| UI 组件库 | `Ant Design` / `MUI` | 直接用现成的，别造轮子 |
| 类型检查 | `TypeScript` | 强烈推荐，相当于带类型的配置校验 |

---

## 一个最小可运行的 React 项目结构

```bash
my-app/
├── public/
│   └── index.html        # 入口 HTML，React 挂载点
├── src/
│   ├── main.jsx          # 入口 JS，把 App 挂到 #root
│   ├── App.jsx           # 根组件
│   └── components/       # 你写的组件们
│       └── ServerCard.jsx
├── package.json          # 依赖声明（类似 requirements.txt）
└── vite.config.js        # 构建配置
```

```jsx
// main.jsx - 启动入口
import ReactDOM from 'react-dom/client';
import App from './App';

ReactDOM.createRoot(document.getElementById('root')).render(<App />);
```

---

## TL;DR — 核心总结

1. **本质**：声明式 UI，描述期望状态，React 负责收敛。类比 K8s/Ansible，你写 what，框架做 how
2. **组件**：UI 的积木块，纯函数，同样输入永远同样输出
3. **State**：数据变 → UI 自动变，你只管改数据，不要手动操作 DOM
4. **Props**：父传子，单向流动，只读
5. **副作用**：函数做了"返回值以外"的事（改外部变量、网络请求、依赖外部可变状态）都算副作用
6. **useEffect**：副作用的隔离区，把 API 请求、定时器等统统关进去，让组件函数保持纯净
7. **Virtual DOM**：React 自己做 diff，只更新变化部分，不用手动操作 DOM

> 接下来最好的学习路径：把一个你熟悉的运维工具（比如服务状态看板）用 React 写一遍，边查边学。光看不练永远停在概念层面。
