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

组件里有些操作不属于"渲染 UI"本身，比如：请求 API、设置定时器、操作 DOM。这些叫**副作用**，用 `useEffect` 处理。

```jsx
import { useState, useEffect } from 'react';

function ServerStatus() {
  const [data, setData] = useState(null);

  useEffect(() => {
    // 组件挂载后执行（类似服务启动后的 health check）
    fetch('/api/servers')
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

## TL;DR — 五行总结

1. **组件**：UI 的积木块，函数返回 JSX
2. **State**：数据变 → UI 自动变，你只管改数据
3. **Props**：父传子，单向流动，只读
4. **useEffect**：处理 API 请求、定时器等副作用
5. **Virtual DOM**：React 自己做 diff，只更新变化部分，你不用手动操作 DOM

> 接下来最好的学习路径：把一个你熟悉的运维工具（比如服务状态看板）用 React 写一遍，边查边学。光看不练永远停在概念层面。
