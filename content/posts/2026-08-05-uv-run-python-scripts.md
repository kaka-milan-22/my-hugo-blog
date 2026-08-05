---
title: "用 uv 运行单文件 Python 脚本：依赖、Python 版本与 lockfile 一次解决"
date: 2026-08-05T20:00:00+08:00
draft: false
tags: ["Python", "uv", "PEP 723", "Automation", "DevOps"]
categories: ["Python", "Tools"]
author: "Kaka"
description: "使用 uv 管理单文件 Python 脚本的运行环境、inline dependencies、Python 版本和 lockfile，让运维脚本无需手工维护 virtualenv 也能可靠复现。"
---

## 引言

单文件 Python 脚本经常从十几行开始：查询一个 API、检查一批 endpoint、清理临时资源。随后它会悄悄引入 `requests`、`rich` 等依赖，最后变成“只有作者电脑能运行”的脚本：不知道该用哪个 Python，也不知道 package version，更不知道某个 virtualenv 是否还在。

`uv run` 解决的不是如何少敲一次 `python`，而是把 interpreter、dependencies 和 environment resolution 放进脚本的执行流程。简单脚本可以直接运行；临时依赖可以通过 `--with` 注入；需要共享的脚本则把 PEP 723 inline metadata 写进文件自身，再用 lockfile 固定 resolution。

本文基于 Astral 官方的 [Running scripts](https://docs.astral.sh/uv/guides/scripts/) 指南重新整理，聚焦 DevOps automation 的实际用法。如果需要先了解 uv 的完整工具链，可以阅读此前的 [uv：Python 工具链的革命性突破](/posts/2026-02-16-uv-python-game-changer/)。

## 从最简单的 `uv run` 开始

没有第三方依赖的脚本可以直接执行：

```python
# hello.py
import platform
import sys

print(f"Python: {sys.version.split()[0]}")
print(f"Platform: {platform.platform()}")
print(f"Arguments: {sys.argv[1:]}")
```

```bash
uv run hello.py
uv run hello.py --verbose
```

`uv` 会选择合适的 Python environment，并把后续参数交给脚本。如果只想执行一段临时代码，也可以从 stdin 读取：

```bash
uv run - <<'PY'
from datetime import datetime, timezone

print(datetime.now(timezone.utc).isoformat())
PY
```

这里有一个容易踩的坑：在包含 `pyproject.toml` 的 project 目录中运行 `uv run script.py`，uv 默认会同步 project environment，并在执行前安装当前 project。如果脚本与 project 无关，应明确跳过：

```bash
# --no-project 必须放在脚本名称前面
uv run --no-project script.py
```

## 临时依赖：使用 `--with`

一次性检查或临时调试不值得创建 project。可以使用 `--with` 为当前 invocation 增加 dependency：

```bash
uv run --with httpx --with rich check_api.py
```

dependency 可以带 version constraint：

```bash
uv run --with 'httpx>=0.28,<1' check_api.py
```

在 project 目录中，`--with` 默认是在 project dependencies 之上增加临时 package；如果想要完全独立的环境，应组合 `--no-project`：

```bash
uv run --no-project --with 'httpx<1' check_api.py
```

`--with` 适合只运行一次的命令、验证 package behavior 或临时补充 debug tool。脚本一旦要进入 Git、交给同事或放进 CI，就不应该让使用者从 README 猜 dependency，应该改用 inline script metadata。

## Inline script metadata：让脚本自带运行说明

PEP 723 定义了一种写在 Python comment block 中的 metadata 格式。它可以声明 `requires-python` 和 `dependencies`，普通 Python interpreter 会把它视为 comment，uv 则把它当作 environment contract。

可以先初始化脚本：

```bash
uv init --script probe.py --python 3.12
```

再让 uv 修改 dependency metadata，不必手写 TOML：

```bash
uv add --script probe.py 'httpx>=0.28,<1' 'rich<15'
```

最终文件顶部会出现这样的 block：

```python
# /// script
# requires-python = ">=3.12"
# dependencies = [
#   "httpx>=0.28,<1",
#   "rich<15",
# ]
# ///
```

`dependencies` 即使为空也必须存在。运行时，uv 会按 metadata 创建隔离 environment、解析并缓存 dependencies，然后执行脚本：

```bash
uv run probe.py
```

与普通 `uv run` 有一个重要区别：脚本一旦包含 inline metadata，即使它位于某个 uv project 中，project dependencies 也会被忽略。这个行为保证脚本的 dependency contract 自包含，但也意味着脚本不能暗中依赖当前 project 中未声明的 package。

## 实例：一个可以直接交付的 endpoint probe

下面的脚本接收一组 URL，以表格显示 status code 与 latency；任何 endpoint 失败时返回非零 exit code，适合直接用于 CI smoke test 或发布后的快速验证。

```python
# /// script
# requires-python = ">=3.12"
# dependencies = [
#   "httpx>=0.28,<1",
#   "rich<15",
# ]
# ///

from __future__ import annotations

import argparse
import time

import httpx
from rich.console import Console
from rich.table import Table


def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(description="Probe HTTP endpoints")
    parser.add_argument("urls", nargs="+", help="One or more HTTP(S) URLs")
    parser.add_argument("--timeout", type=float, default=5.0)
    return parser.parse_args()


def main() -> int:
    args = parse_args()
    table = Table("URL", "Status", "Latency", "Result")
    failed = False

    with httpx.Client(
        follow_redirects=True,
        timeout=args.timeout,
    ) as client:
        for url in args.urls:
            started = time.perf_counter()
            try:
                response = client.get(url)
                latency_ms = (time.perf_counter() - started) * 1000
                ok = response.is_success
                failed |= not ok
                table.add_row(
                    url,
                    str(response.status_code),
                    f"{latency_ms:.1f} ms",
                    "OK" if ok else "FAILED",
                )
            except httpx.HTTPError as exc:
                latency_ms = (time.perf_counter() - started) * 1000
                failed = True
                table.add_row(
                    url,
                    "-",
                    f"{latency_ms:.1f} ms",
                    f"ERROR: {exc}",
                )

    Console().print(table)
    return 1 if failed else 0


if __name__ == "__main__":
    raise SystemExit(main())
```

使用者只需要安装 uv，不需要提前执行 `pip install` 或激活 virtualenv：

```bash
uv run probe.py \
  https://example.com \
  https://example.com/healthz \
  --timeout 3
```

第一次运行需要 resolve 和下载 dependencies，后续运行会复用 uv cache。脚本、Python constraint 与直接 dependencies 都在同一个文件中，因此 code review 能看到运行环境发生了什么变化。

## 固定 Python 版本

inline metadata 的 `requires-python` 表达兼容范围：

```python
# /// script
# requires-python = ">=3.12,<3.15"
# dependencies = []
# ///
```

`uv run` 会寻找满足 constraint 的 interpreter；如果本机没有且 uv 允许自动管理 Python，它可以下载合适版本。也可以在 invocation 中明确请求版本，但所选版本仍需满足脚本声明：

```bash
uv run --python 3.13 probe.py https://example.com
```

`requires-python` 比只在文档里写“请使用 Python 3.12”更可靠，因为不兼容时会在执行前失败，而不是运行到某个新语法或标准库 API 才报错。

## Lockfile：从“可运行”升级到“可复现”

version constraint 只描述允许范围，今天和三个月后可能 resolve 到不同 transitive dependencies。对于 CI、定时任务和长期保存的运维脚本，应为 script 单独创建 lockfile：

```bash
uv lock --script probe.py
```

uv 会在脚本旁创建 `probe.py.lock`。应把脚本和 lockfile 一起提交；运行时使用 `--locked`，确保 metadata 与 lockfile 不一致时直接失败，而不是在 CI 中偷偷更新 resolution：

```bash
uv run --locked probe.py https://example.com/healthz
```

如果需要限制 resolver 只考虑某个时间点之前发布的 distribution，还可以在 inline metadata 中配置 `exclude-newer`：

```python
# /// script
# requires-python = ">=3.12"
# dependencies = ["httpx<1"]
# [tool.uv]
# exclude-newer = "2026-08-01T00:00:00Z"
# ///
```

`exclude-newer` 适合重建历史环境或降低未来 resolution 漂移，但它不是 vulnerability scanning，也不能替代 lockfile review。dependency update 仍应通过显式变更、测试和 code review 完成。

## 变成真正的 executable script

在支持 `env -S` 的 Unix-like 系统上，可以增加 shebang，让使用者直接执行文件：

```python
#!/usr/bin/env -S uv run --script
#
# /// script
# requires-python = ">=3.12"
# dependencies = ["httpx<1"]
# ///

import httpx

print(httpx.get("https://example.com", timeout=5).status_code)
```

```bash
chmod +x probe
./probe
```

这非常适合放在 repository 的 `scripts/` 目录或个人 `PATH` 中。脚本依然是单个文件，但具备明确的 Python 和 dependency contract。

## Private index 的处理原则

uv 支持把 alternative package index 写入 script metadata：

```bash
uv add \
  --index https://packages.example.com/simple \
  --script probe.py \
  internal-sdk
```

不要把 username、password 或 token 直接写进 index URL，因为脚本会进入 Git 和日志。metadata 只保留不含 credential 的 endpoint，authentication 通过受控的 credential provider、环境注入或 secret manager 解决。

## 如何选择执行方式

| 场景 | 推荐方式 | 原因 |
|---|---|---|
| 无第三方 dependency 的一次性脚本 | `uv run script.py` | 最少配置 |
| 临时验证一个 package | `uv run --with package script.py` | 不污染长期 environment |
| 需要共享的单文件 automation | PEP 723 inline metadata | Python 与 dependencies 自包含 |
| CI、cron、长期运维脚本 | inline metadata + `.lock` + `--locked` | resolution 可审查、可复现 |
| 多 module application、library 或 test suite | 正式 uv project | 需要 package structure、dependency groups 与统一 lockfile |

判断边界很简单：单文件仍能完整表达 dependency 和生命周期，就使用 script metadata；一旦出现多个 internal module、dev dependency、build、test 或 publish 需求，就应该升级为 project，而不是把所有东西继续塞进一个脚本。

## 总结

`uv run` 把单文件 Python automation 从“依赖作者电脑环境”变成了声明式执行单元。`--with` 解决临时 dependency，PEP 723 inline metadata 固定 Python compatibility 和直接 dependencies，script lockfile 则把 transitive resolution 纳入版本控制。

我的默认做法是：本地临时执行用 `--with`；需要交付的脚本使用 inline metadata；进入 CI 或 cron 后立刻增加 lockfile 和 `--locked`。这样既保留单文件脚本的轻量，也不会重新制造一批无人知道如何复现的 virtualenv。

## 参考资料

- [Astral uv：Running scripts](https://docs.astral.sh/uv/guides/scripts/)
- [Python Packaging User Guide：Inline script metadata](https://packaging.python.org/en/latest/specifications/inline-script-metadata/)
- [Astral uv：Command reference](https://docs.astral.sh/uv/reference/cli/)
