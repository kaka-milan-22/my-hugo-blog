---
title: "Git 三棵树模型深度解析"
date: 2026-03-05T10:30:00+08:00
draft: false
tags: ["Git", "DevOps", "版本控制", "原理"]
categories: ["DevOps", "Tools"]
author: "Kaka"
description: "彻底理解 Git 最核心的心智模型：Working Directory、Staging Index、Repository 三棵树的运作机制，以及所有常用命令背后的本质"
---

## 为什么要理解三棵树？

很多开发者用了多年 Git，遇到问题还是只会 `git reset --hard` 硬来，或者 `git stash` 逃跑。根本原因是**没有建立正确的心智模型**。

Git 所有的操作——add、commit、reset、checkout、merge、rebase——本质上都是**在三棵树之间移动数据**。理解了三棵树，所有命令的行为都变得可以推理，而不是死记硬背。

---

## 一、三棵树总览

Git 在本地维护三个独立的数据结构，每一个都可以理解为一个"文件系统快照"：

```
┌──────────────────────────────────────────────────────────────────┐
│                         Git 三棵树                                │
│                                                                  │
│  ┌─────────────────┐   ┌─────────────────┐   ┌───────────────┐  │
│  │  Working Tree   │   │  Staging Index  │   │  Repository   │  │
│  │  （工作目录）    │   │   （暂存区）     │   │  （本地仓库）  │  │
│  │                 │   │                 │   │               │  │
│  │  你实际看到和    │   │  下一次 commit  │   │  所有历史     │  │
│  │  编辑的文件      │   │  将要包含的内容  │   │  commit 快照  │  │
│  │                 │   │                 │   │               │  │
│  │  磁盘真实文件    │   │  .git/index     │   │  .git/objects │  │
│  └────────┬────────┘   └────────┬────────┘   └───────┬───────┘  │
│           │                     │                    │           │
│           │   git add ────────▶ │                    │           │
│           │                     │  git commit ─────▶ │           │
│           │   git checkout ◀─────────────────────────│           │
│           │   git restore  ◀────│                    │           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 二、第一棵树：Working Tree（工作目录）

### 本质

Working Tree 就是你在磁盘上看到的那些文件，是你直接用编辑器、IDE 操作的地方。**它是三棵树里唯一"肉眼可见"的**。

```
project/
├── main.go          ← 你正在编辑的文件
├── config.yaml      ← 你刚刚改过的配置
└── README.md        ← 没有修改
```

### 文件的四种状态

Working Tree 中的每个文件，在 Git 眼里处于四种状态之一：

```
┌─────────────────────────────────────────────────────────┐
│                    文件生命周期                           │
│                                                         │
│  Untracked ──── git add ────▶ Staged ──── git commit ──▶ Committed
│      │                          │                            │
│      │                          │ git restore --staged       │
│      │                          ▼                            │
│      │                      Unstaged (Modified)              │
│      │                          │                            │
│      │                          │ git restore                │
│      │                          ▼                            │
│      │                       Unmodified ◀─────────────────── │
│      │                                                       │
│   git rm --cached / .gitignore                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

| 状态            | 含义                              | `git status` 显示         |
|----------------|-----------------------------------|--------------------------|
| **Untracked**  | Git 从未追踪过此文件               | `?? new_file.go`         |
| **Modified**   | 已追踪，但自上次 commit 后有改动   | `M  main.go`             |
| **Staged**     | 已 `git add`，等待下次 commit      | `A  new_file.go`         |
| **Unmodified** | 与最新 commit 完全一致             | （不显示）                |

### 实际观察

```bash
echo "hello" > new_file.go       # Untracked
git add new_file.go              # → Staged
echo "world" >> new_file.go      # 同一个文件现在同时出现在 Staged 和 Modified！

git status
# Changes to be committed:     ← Staging Index 里的版本（只有 "hello"）
#   new file: new_file.go
#
# Changes not staged:          ← Working Tree 里的版本（有 "hello\nworld"）
#   modified: new_file.go
```

这个现象揭示了一个关键事实：**`git add` 拍摄的是那一刻文件内容的快照**，后续对文件的修改不会自动更新暂存区。

---

## 三、第二棵树：Staging Index（暂存区）

### 本质

Staging Index 存储在 `.git/index` 文件中，是一个**二进制格式的目录树**，记录了"下一次 commit 将会包含的完整文件快照"。

它是 Git 与其他版本控制系统最重要的差异点，**SVN、Mercurial 都没有这一层**。

### 内部结构

`.git/index` 本质上是一张表，每一行代表一个被追踪的文件：

```
┌──────────────┬──────────────┬────────────┬──────────────────┐
│  ctime/mtime │  文件大小    │  SHA-1     │  文件路径        │
├──────────────┼──────────────┼────────────┼──────────────────┤
│  1709600000  │  1024        │  3b18e512  │  src/main.go     │
│  1709600100  │  256         │  8ab4c3d1  │  config.yaml     │
│  1709598000  │  512         │  f1e2d3c4  │  README.md       │
└──────────────┴──────────────┴────────────┴──────────────────┘
```

每个 SHA-1 对应 `.git/objects/` 中的一个 blob 对象（文件内容的压缩存储）。

```bash
# 查看暂存区当前内容
git ls-files --stage
# 100644 3b18e512dba79e4c8300dd08aeb37f8e728b8dad 0  src/main.go
# 100644 8ab4c3d1...                              0  config.yaml

# 查看暂存区与工作目录的差异
git diff                    # Working Tree vs Staging Index

# 查看暂存区与最新 commit 的差异
git diff --staged           # Staging Index vs Repository（即 commit 后会有什么变化）
git diff --cached           # 同上，两者等价
```

### 暂存区的核心价值：精细控制提交粒度

假设你同时修改了三件事：修复了一个 bug、优化了一个函数、更新了文档。没有暂存区，你只能一次性提交。有了暂存区：

```bash
# 场景：同一个文件里有 bug fix 和 feature 两类修改
git add -p src/handler.go
# Git 会逐个 hunk 询问你是否暂存：
#
# @@ -10,6 +10,7 @@ func HandleRequest(...)
# -    return nil
# +    return fmt.Errorf("invalid input: %w", err)  ← bug fix
#
# Stage this hunk [y,n,q,a,d,s,?]? y   ← 只暂存 bug fix 的部分
#
# @@ -25,4 +26,8 @@ func processData(...)
# +    // new feature code...              ← 新功能
#
# Stage this hunk [y,n,q,a,d,s,?]? n   ← 不暂存新功能

git commit -m "fix: handle invalid input error"
# 第一个 commit 只包含 bug fix

git add -p src/handler.go
git commit -m "feat: add data processing optimization"
# 第二个 commit 只包含新功能
```

这就是**原子提交（Atomic Commits）**的实现方式，让 `git log`、`git bisect`、code review 都变得更清晰。

### 暂存区的特殊模式（Merge 冲突时）

当合并冲突发生时，暂存区会为同一个文件维护**三个版本**（stage 0/1/2/3）：

```bash
git ls-files --stage
# 100644 <hash1> 1  conflicted.go   ← stage 1: 共同祖先版本 (base)
# 100644 <hash2> 2  conflicted.go   ← stage 2: 当前分支版本 (ours)
# 100644 <hash3> 3  conflicted.go   ← stage 3: 被合并分支版本 (theirs)

# 解决冲突后，stage 编号归零，表示已解决
git add conflicted.go
git ls-files --stage
# 100644 <hash4> 0  conflicted.go   ← stage 0: 正常状态
```

---

## 四、第三棵树：Repository（本地仓库）

### 本质

Repository 就是 `.git/objects/` 目录，是一个**内容寻址的 key-value 数据库**，存储所有历史快照。它是三棵树里唯一**永久且不可变的**。

一旦数据写入 Repository（即执行 `git commit`），在不强制覆盖的情况下，数据永远不会丢失。

### 对象图谱（Object Graph）

Repository 存储的是一个**有向无环图（DAG）**，每次 commit 指向其父 commit，形成完整的历史链。

```
Repository 内部结构：

commit C3  ──────────────────────────────────────────────
│  tree: a1b2                                           │
│  parent: C2                                           │
│  author: dev@example.com                              │
│  message: "feat: add OAuth login"                     │
└──▶ tree a1b2 ──▶ blob 3b18 (main.go v3)
                ├──▶ blob 8ab4 (config.yaml)     ← 未变更，共享同一 blob
                └──▶ tree f1e2 (src/)
                          └──▶ blob 9c3d (handler.go v2)

commit C2  ──────────────────────────────────────────────
│  tree: d3e4
│  parent: C1
└──▶ tree d3e4 ──▶ blob 2a1b (main.go v2)
                ├──▶ blob 8ab4 (config.yaml)     ← 同一个 blob！内容没变
                └──▶ tree f1e2 (src/)
                          └──▶ blob 7b2c (handler.go v1)

commit C1  ──────────────────────────────────────────────
│  tree: e5f6
│  parent: (none, 初始提交)
└──▶ ...
```

**关键洞察：Git 存储的是快照，不是差异（diff）。** 但因为内容相同的文件共享同一个 blob 对象，实际磁盘空间非常高效。

### HEAD 与分支指针

```
Repository 的引用系统：

HEAD  ──▶  refs/heads/main  ──▶  commit C3
                                       │
                                       ▼
            refs/heads/feature  ──▶  commit C4
                                       │
                                       ▼ (parent)
                                     commit C3

# HEAD 是"你在哪"
# 分支名是"分支末端在哪"
# 它们都只是指向某个 commit 的指针
```

```bash
# 验证这些引用的物理存储
cat .git/HEAD
# ref: refs/heads/main

cat .git/refs/heads/main
# a1b2c3d4e5f6...（一个 commit 的 SHA-1）

# 查看 commit 对象内容
git cat-file -p HEAD
# tree f7a8b9...
# parent d3e4f5...
# author Dev <dev@example.com> 1709600000 +0800
# committer Dev <dev@example.com> 1709600000 +0800
#
# feat: add OAuth login
```

---

## 五、命令本质：三棵树之间的数据移动

理解了三棵树，所有命令的本质一目了然：

### 5.1 正向流：Working Tree → Index → Repository

```
┌───────────────┐    git add     ┌───────────────┐   git commit  ┌───────────────┐
│  Working Tree │ ─────────────▶ │ Staging Index │ ─────────────▶│  Repository   │
│               │                │               │               │               │
│  你的修改      │                │  下次提交快照  │               │  永久历史      │
└───────────────┘                └───────────────┘               └───────────────┘
```

### 5.2 `git reset` 的三种模式（反向移动）

`git reset` 是三棵树模型理解最重要的命令，它移动的是 **HEAD（和当前分支指针）**，并根据参数决定是否同步更新 Index 和 Working Tree。

```
git reset 移动目标：Repository → Index → Working Tree

                   Repository         Index          Working Tree
                  (HEAD 移动到)    (是否同步)        (是否同步)
 ─────────────────────────────────────────────────────────────────
 --soft   HEAD~1       ✅ 移动           ❌ 不变          ❌ 不变
 --mixed  HEAD~1       ✅ 移动           ✅ 同步          ❌ 不变   ← 默认
 --hard   HEAD~1       ✅ 移动           ✅ 同步          ✅ 同步
```

用场景来理解：

```bash
# 场景 1：刚 commit，发现漏加了一个文件
git reset --soft HEAD~1
# HEAD 回退一步，但暂存区保持不变
# 相当于把 commit "拆包"回暂存区
git add forgotten_file.go
git commit -m "feat: complete implementation"   # 重新提交


# 场景 2：暂存了一堆东西，想重新整理要提交哪些
git reset HEAD         # 等价于 git reset --mixed HEAD
# HEAD 不动（已经是最新），但暂存区清空
# 工作目录的修改保留，可以重新 git add -p


# 场景 3：本地实验性修改全部废弃，回到上一个 commit 的状态
git reset --hard HEAD
# 三棵树全部同步到 HEAD 状态
# ⚠️ 工作目录未暂存的修改全部丢失，无法通过 reflog 找回
```

**图解 --soft / --mixed / --hard 的差异：**

```
初始状态：
  Repository:   A ──▶ B ──▶ C   (HEAD → main → C)
  Index:        文件内容 = C 的快照
  Working Tree: 文件内容 = C 的快照（无未保存修改）

执行 git reset HEAD~1（即回退到 B）后：

  --soft:
    Repository:   A ──▶ B ──▶ C   (HEAD → main → B，C 仍存在)
    Index:        文件内容 = C 的快照  ← 未变！
    Working Tree: 文件内容 = C 的快照  ← 未变！
    效果：C 的所有改动出现在暂存区，等待重新提交

  --mixed (默认):
    Repository:   A ──▶ B   (HEAD → main → B)
    Index:        文件内容 = B 的快照  ← 同步到 B
    Working Tree: 文件内容 = C 的快照  ← 未变！
    效果：C 的所有改动出现在工作目录（未暂存），可 git add 重新整理

  --hard:
    Repository:   A ──▶ B   (HEAD → main → B)
    Index:        文件内容 = B 的快照  ← 同步到 B
    Working Tree: 文件内容 = B 的快照  ← 同步到 B ⚠️
    效果：完全回到 B 的状态，C 的改动全部消失
```

---

### 5.3 `git checkout` / `git restore` 的本质

```
git checkout <branch>  ──▶  1. 移动 HEAD 到目标分支
                            2. 用 Repository 更新 Index
                            3. 用 Index 更新 Working Tree
                            （三棵树全部同步）

git restore <file>     ──▶  用 Index 覆盖 Working Tree 中的指定文件
                            （撤销工作目录的未暂存修改）

git restore --staged <file>  ──▶  用 Repository(HEAD) 覆盖 Index 中的指定文件
                                  （撤销暂存，但保留工作目录修改）
```

```
git restore 的数据流向：

  git restore <file>:
    Repository  ──X──  Index ─────────────▶  Working Tree
                         ↑
                     从 Index 读取，覆盖工作目录

  git restore --staged <file>:
    Repository ───────────────▶  Index      Working Tree（不变）
                         ↑
                     从 Repository 读取，覆盖暂存区
```

---

### 5.4 命令与三棵树映射全表

```
命令                            Working Tree   Index   Repository   HEAD
 ─────────────────────────────────────────────────────────────────────────
 git add <file>                      读          写
 git commit                                      读         写        移动
 git status                          读          读         读
 git diff                            读          读
 git diff --staged                               读         读

 git reset --soft  <commit>                                            移动
 git reset --mixed <commit>                      写                    移动
 git reset --hard  <commit>          写          写                    移动

 git restore <file>                  写          读
 git restore --staged <file>                     写         读

 git checkout <branch>               写          写                    移动
 git stash                           写          写
 git stash pop                       写          写

 git merge <branch>                  写          写         写        移动
 git rebase                          写          写         写        移动

 git cherry-pick <commit>            写          写         写        移动
```

---

## 六、三棵树与 Stash 的关系

`git stash` 本质上是将三棵树的当前状态打包成一个特殊的 commit 对象存入 Repository，然后将 Working Tree 和 Index 还原到 HEAD 状态。

```
执行 git stash 时发生了什么：

  1. 将 Index 的快照打包为一个 commit 对象（stash index tree）
  2. 将 Working Tree 的快照打包为另一个 commit 对象（stash working tree）
  3. 这两个 commit 被记录到 refs/stash
  4. 将 Working Tree 和 Index 都还原为 HEAD 的状态

  stash@{0}
  ├── stash commit (message: "WIP on main: a1b2 feat: xxx")
  │     ├── parent: HEAD commit
  │     ├── parent: index tree commit    ← 暂存区的快照
  │     └── parent: untracked commit     ← 未追踪文件（-u 时）
  └── working directory commit           ← 工作目录快照
```

```bash
# git stash -u 将未追踪文件也纳入储藏
# git stash --keep-index 只储藏工作目录，保留暂存区
# git stash branch <name> 将储藏恢复为新分支（最安全的恢复方式）
```

---

## 七、一个完整的操作示例

用三棵树视角，追踪一个完整的开发周期：

```
Step 0: 初始状态（三棵树一致，都是 commit C1 的内容）
  Repository (HEAD=C1): { main.go: v1, config.yaml: v1 }
  Index:                { main.go: v1, config.yaml: v1 }
  Working Tree:         { main.go: v1, config.yaml: v1 }

Step 1: 修改文件 main.go
  Repository:   { main.go: v1, config.yaml: v1 }  ← 不变
  Index:        { main.go: v1, config.yaml: v1 }  ← 不变
  Working Tree: { main.go: v2, config.yaml: v1 }  ← 变 了 ← "Modified"

Step 2: git add main.go
  Repository:   { main.go: v1, config.yaml: v1 }  ← 不变
  Index:        { main.go: v2, config.yaml: v1 }  ← 更新了 ← "Staged"
  Working Tree: { main.go: v2, config.yaml: v1 }

Step 3: 再次修改 main.go（添加了注释）
  Repository:   { main.go: v1, config.yaml: v1 }
  Index:        { main.go: v2, config.yaml: v1 }  ← 还是 v2（上次 add 的快照）
  Working Tree: { main.go: v3, config.yaml: v1 }  ← 变成 v3

  ⚠️ 此时 git status 会显示 main.go 既在 "staged" 又在 "modified"！

Step 4: git commit -m "feat: update main"
  Repository:   { main.go: v2, config.yaml: v1 }  ← 提交的是 Index 的快照（v2）！
  Index:        { main.go: v2, config.yaml: v1 }  ← 不变
  Working Tree: { main.go: v3, config.yaml: v1 }  ← 不变，v3 的修改还在

  ⚠️ 注意：v3 的改动（注释）没有进入本次 commit！
```

---

## 总结

```
┌──────────────────────────────────────────────────────────────────┐
│                       三棵树要点速记                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Working Tree  = 你的磁盘 = 唯一肉眼可见的                        │
│  Index         = 下一次 commit 的预览 = .git/index               │
│  Repository    = 永久历史 = .git/objects/ = 不可变 DAG           │
│                                                                  │
│  git add      : Working Tree  ──▶  Index                        │
│  git commit   : Index         ──▶  Repository                   │
│  git restore  : Index         ──▶  Working Tree                 │
│  git reset    : Repository    ──▶  Index (──▶ Working Tree)     │
│  git checkout : Repository    ──▶  Index  ──▶  Working Tree     │
│                                                                  │
│  reset --soft  = 只移动 HEAD                                     │
│  reset --mixed = 移动 HEAD + 更新 Index               (默认)    │
│  reset --hard  = 移动 HEAD + 更新 Index + 更新 Working Tree     │
│                                                                  │
│  git add -p 是精细控制 Index 内容的最佳实践                       │
│  git diff          = Working Tree vs Index                       │
│  git diff --staged = Index vs Repository                         │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

理解三棵树之后，Git 就不再是一个神秘的"黑盒"，而是一个行为完全可预测的工具。每当你不确定某个命令会做什么，问自己一句：**"它会把哪棵树的数据写入哪棵树？"** 答案往往就清晰了。
