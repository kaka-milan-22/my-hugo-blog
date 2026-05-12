---
title: "从零打造 AI Agent 原生 EVM 钱包：当 MetaMask 不再适合 LLM 时代"
date: 2026-05-12T10:00:00+08:00
draft: false
tags: ["Wallet", "EVM", "Ethereum", "AI Agent", "Web3", "DeFi", "Aave", "Uniswap", "Python", "Security"]
categories: ["Tools", "Architecture"]
author: "Kaka"
description: "一个为 LLM Agent 而生的 EVM 钱包：纯 CLI、JSON I/O、策略门禁、FIFO 密钥传输、Aave / Uniswap 直连，在 Sepolia 上完整跑通借贷 / 兑换全流程。"
---

## 引言：为什么需要一个 Agent 原生的钱包

主流的 EVM 钱包，无论 MetaMask、Rabby 还是 Phantom，本质都是**给人用的**：浏览器扩展、WalletConnect 二维码、`window.ethereum` 注入、点击签名弹窗。当你想让一个 LLM Agent 自主完成"先在 Uniswap 上把 ETH 换成 USDC，再把 USDC 存进 Aave 收利息，HF 跌破 1.5 就自动还款"这种工作流时，整个浏览器钱包技术栈瞬间从"安全护栏"变成"无法跨越的鸿沟"。

很多人会想：那就把私钥直接喂给 Agent 行不行？这是把钱包退化成一行 `web3.eth.account.from_key()`，等于让一个会幻觉的模型直接操作你的资产，没有任何 fail-safe。任何一次 prompt injection、任何一次合约地址写错、任何一次"我以为它会 dry-run 结果直接 broadcast"，资金就没了。

这篇文章介绍我自己做的一个开源项目 `wallet`：一个 **AI-agent-native EVM 钱包**，Python CLI，所有写操作都走"预览 → 策略 → 幂等 → 确认 → 签名 → 审计"六段式管道。整套设计已经在 Sepolia 测试网上跑通了真实的转账、Uniswap V3 兑换、Aave V3 借贷全流程，每一步都有 Etherscan 上的链上交易作证。

它不是 MetaMask 的替代品。MetaMask 服务的是"人坐在浏览器前点按钮"的场景；这个钱包服务的是"Agent 在终端里跑 cron 任务"的场景。两者职责正交。

## 整体架构：四层防御

设计原则只有一条：**Agent 可以调用任何读操作，但任何写操作都必须穿过四层独立的安全闸门**。

| 层 | 位置 | 拦截的失败模式 |
|---|---|---|
| **skill** | `docs/skills/wallet-agent.skill.md` | 告诉 Agent 怎么用：永远先 dry-run、每次 broadcast 用新的 request-id、永远不要 `--policy-bypass` |
| **policy** | `~/.wallet/policy.json` | 硬性预签名拦截：金额上限、收款人白名单、合约白名单、拒绝无限授权、`min_health_factor` |
| **idempotency** | `~/.wallet/idempotency.json` | 同一个 `--request-id` 重放只会返回缓存结果，绝不重复广播 |
| **audit** | `~/.wallet/audit.log` | append-only JSON-lines，记录每一次签名尝试。**没有任何 CLI 命令能读它**，Agent 无法用日志反推下一步 |

四层是 AND 关系，任一拒绝就 fail-closed。policy 还有一个反常识的默认：**没初始化 policy 时全部拒绝**，不是"开放全部"。这个决策的逻辑是，新装的钱包在你没显式声明信任谁之前，应该一分钱都动不了。

```
src/wallet/
  cli/_common.py       # confirm_and_broadcast 管道
  core/policy.py       # 策略求值
  core/signer.py       # mnemonic -> 私钥 -> 签名（仅在内存中）
  storage/audit.py     # append-only 审计
  storage/idempotency.py
  storage/vault.py     # agent-vault 适配 + FIFO 传输
```

## 账户管理：助记词永不落盘

HD 钱包标准的 BIP-39 + BIP-44，但秘密存储这里下了点功夫。

```sh
# 在你的终端里跑（不是 Agent 上下文）
wallet account create main
# 会打印 12 词助记词 + 提示运行：
agent-vault set wallet-main-mnemonic
# 把助记词粘贴进 agent-vault
```

关键点：

1. `account create` / `account import` **被硬编码为 TTY-only**。在 Agent 上下文里跑会直接拒绝，因为助记词必然要打印到 stdout，会立刻进入 LLM 的 context window，等于公开。
2. 助记词加密存在 `agent-vault`（一个本地 vault，对 Agent 不可见）。
3. 签名时，助记词通过 **Unix FIFO（命名管道）** 从 vault 进程传到签名进程。FIFO 是内核管道，不写磁盘，进程退出立刻消失。fallback 路径走 `/tmp/wallet-secret-*` 短暂文件，但 FIFO 路径下连这个 fallback 都不会触发。

可以用下面这个观察脚本验证没有泄漏：

```sh
while true; do ls /tmp/wallet-secret-* 2>/dev/null && echo LEAK; done &
wallet send <to> 0.0001 --broadcast
# 看到 LEAK 行才说明 fallback 被触发了
```

一份助记词可以派生任意多个地址：

```sh
wallet account derive main --index 1 --as second
# main:   0x34a9...36c7  (m/44'/60'/0'/0/0)
# second: 0xFb0b...d126  (m/44'/60'/0'/0/1)
# 共享 vault key: wallet-main-mnemonic
```

## 转账：默认 dry-run，强制确认

EVM 钱包最常见的事故是"误转账"。这个钱包的默认行为是 dry-run，你必须显式加 `--broadcast` 才会真正签名上链：

```sh
wallet send 0xabc... 0.01                              # 预览，不签名
wallet send @alice 0.01 --broadcast                    # 会弹 y/N 确认
wallet send @alice 0.01 --broadcast --yes              # 跳过确认（Agent 模式必须显式）
wallet send 0xabc... 50 --token USDC --broadcast --yes # ERC-20 转账
```

`@alice` 是地址本里的别名（`wallet book add alice 0xabc...`）。Agent 调用时建议直接用 `0x...`，避免地址本被污染。

链上记录（Sepolia）：

| 操作 | 金额 | 哈希 |
|---|---|---|
| 第一笔 ETH 转账 | 0.001 ETH | `0x5abce81b...` |
| 带 `--request-id` 的转账 | 0.0001 ETH | `0xca5882b3...` |
| 较大转账 | 0.02 ETH | `0x3a1e1c16...` |

## ERC-20 授权：默认拒绝无限授权

钓鱼合约最常用的套路就是诱导用户签一个 `approve(spender, 2^256-1)`，之后随时把账户的 token 拉走。这个钱包默认策略里 `deny_unlimited_approve: true`，任何 `--unlimited` 调用都会被拒绝：

```sh
wallet approve set USDC <spender> 100               # 有限授权
wallet approve set USDC <spender> --unlimited       # 被策略阻断
wallet approve show USDC --spender 0x...            # 查看当前 allowance
wallet approve revoke USDC <spender>                # 一键撤销
```

如果你确实需要无限授权（比如对某些路由器优化 gas），需要在 TTY 里手动把策略里的这一条改成 `false`，**Agent 不能修改 policy 文件**。

## Swap：0x 聚合器 + Uniswap V3 兜底

默认 `--via auto` 优先尝试 0x 聚合器（拿 ~100 个 DEX 的最佳报价），如果 0x 不可用（比如测试网没 API key 或没流动性），自动 fallback 到直接调用 Uniswap V3 SwapRouter02：

```sh
wallet swap ETH USDC 0.001                             # dry-run, auto route
wallet swap USDC WETH 100 --slippage-bps 50            # 0.5% 滑点
wallet swap ETH USDC 0.001 --broadcast --yes           # 真签名
wallet swap ETH USDC 0.001 --via uniswap-v3            # 强制直连 UniV3
wallet swap ETH USDC 0.001 --via 0x                    # 强制走聚合器
```

预览里会显式列出路由 + 期望输出 + 最小可接受输出：

```
swap route          ETH > 500bps > USDC
expected out        15.565555 USDC
min out (0.5% slip) 15.487727 USDC
```

Sepolia 实测：0.001 ETH 在 Uniswap V3 的 0.05% 费率池换出 15.565555 USDC（`0xe96c972c...`），由于测试网流动性极薄，实际成交与报价完全一致；主网会有滑点，由 min_out 兜底。

## Aave V3 借贷：完整循环

这是最能体现"Agent 原生"价值的功能。一个完整的 supply → withdraw → borrow → repay 循环，全部在终端里跑通：

### 读操作（免费 RPC 查询）

```sh
wallet aave positions               # 你的存款 / 借款 + Health Factor
wallet aave rates                   # 每个池子的 supply / variable-borrow APR
wallet aave rates --token USDC      # 过滤单个 token
```

### Faucet：免 MetaMask 领测试币

Aave 官方 staging.aave.com 的 faucet 必须用 MetaMask 才能领 mock token。但 faucet 合约本身是 permissionless 的，所以我直接把它做成了 CLI 命令：

```sh
wallet aave faucet WBTC 0.01 --broadcast --yes
# 链上记录: 0x9b5264e7...
```

### 完整借贷循环

实际跑通的链上序列：

| 步骤 | 操作 | 哈希 |
|---|---|---|
| 1 | approve WBTC → Aave Pool | `0x3675d91e...` |
| 2 | supply 0.005 WBTC | `0x78ae3f30...` |
| 3 | withdraw 0.002 WBTC（部分） | `0xd651449b...` |
| 4 | 尝试 borrow 150 USDC | **被策略拦截**（预估 HF 1.500 < min 2.0） |
| 5 | borrow 100 USDC | `0x9e4cbabe...`（HF 2.250 ✓） |
| 6 | repay USDC --max（$100.0003） | `0x069b74d7...` |
| 7 | withdraw WBTC --max | `0xc6dd9bad...` |

整个循环跑完后仓位归零，没有残留债务。

### `min_health_factor`：自主借贷的杀手锏

这是我觉得最有价值的一个策略字段。Aave 链上检查只在 HF < 1 时 revert（也就是"已经可被清算"），但等到那时已经晚了——下一次价格 tick 你就被清算了。

`min_health_factor: 2.0` 让钱包在 **签名之前** 就算出预估 HF，低于阈值直接拦截：

```
预览输出：
  current HF            ∞
  estimated HF after    1.500
  liquidation HF        1.000 (Aave revert threshold)

→ error: policy_block — hf-would-drop-below-min:1.500<2.0
```

HF 计算逻辑读 Aave oracle 的 `getAssetPrice` 和每个 reserve 的清算阈值：

- borrow: `new_debt = current_debt + amount × price`
- withdraw: `new_weighted_collateral = current_weighted - amount × price × asset_LT`

这个比 Aave 自己的检查**严格得多**，专门防止 Agent 在没有实时市场判断能力的情况下"借到悬崖边"。

## 安全机制实战：每一道闸门的真实拦截

### 策略未初始化 → 拒绝一切

```sh
wallet send 0xabc... 0.001 --broadcast --yes
# error: policy_block — no-policy-configured-run-wallet-policy-init
```

### 收款人不在白名单

```sh
wallet send 0xUNKNOWN 0.001 --broadcast --yes
# error: policy_block — recipient-not-in-allowlist
```

### 合约不在白名单

调用 Uniswap 路由 / Aave Pool / Aave Faucet 前，钱包先检查合约地址是否在 `contract_allowlist` 里：

```sh
wallet swap USDC WETH 1 --broadcast --yes
# error: policy_block — swap-router-not-in-contract-allowlist
```

### 授权额度不足 → 直接给出修复命令

```sh
wallet aave repay USDC 99.5 --broadcast --yes
# error: insufficient_allowance
# data.suggested_command: "wallet approve set USDC 0x6Ae4...8951 99.5"
```

`suggested_command` 字段让 Agent 直接知道下一步该跑什么，避免烧 gas 在必定 revert 的交易上。

### Aave 错误码解码

Aave Pool revert 时返回的是数字错误码，钱包把它翻译成人话：

```
error: simulation_reverted — aave:51 (SUPPLY_CAP_EXCEEDED — Aave's supply cap is full
  (common on testnets). Try a smaller amount or a different reserve.)
```

JSON 输出里 `data.aave_error_code: "51"` 让 Agent 可以基于数字码分支决策。

### 幂等性：同一个 request-id 不会重复广播

```sh
RID=$(uuidgen)
wallet send second 0.0001 --broadcast --yes --request-id "$RID"
# → submitted: 0x12e11533...  (broadcast)

wallet send second 0.0001 --broadcast --yes --request-id "$RID"
# → submitted: 0x12e11533...  (replayed_idempotent, 同一个 hash，没有重新广播)
```

`~/.wallet/idempotency.json` 存 `(request_id, fingerprint, tx_hash)` 三元组。如果同一个 id 配上不同的参数，会抛 `idempotency_mismatch`，防止 Agent 在重试时不小心 double-spend。

## JSON 输出：为 Agent 而生

每个命令都支持 `--json`（或 `WALLET_JSON=1`），输出单行 JSON envelope，**stderr 永不污染 stdout**：

```sh
WALLET_JSON=1 wallet send second 0.0001 --broadcast --yes --request-id "$(uuidgen)" | jq .
```

```jsonc
// 成功
{
  "ok": true,
  "command": "send",
  "chain": "sepolia",
  "data": {
    "phase": "broadcast",
    "kind": "native_transfer",
    "from": "0x34a9…",
    "to": "0xFb0b…",
    "amount_wei": "100000000000000",
    "amount": "0.0001",
    "tx_hash": "0xca5882b3…",
    "explorer_url": "https://sepolia.etherscan.io/tx/…",
    "request_id": "ab81ab82-...",
    "outcome": "broadcast",
    "nonce": 5,
    "gas": 21000
  }
}

// 错误（所有错误码共享同一个 schema）
{
  "ok": false,
  "command": "aave.borrow",
  "error": "policy_block",
  "reason": "hf-would-drop-below-min:1.500<2.0",
  "data": {...}
}
```

可枚举的错误码：`validation_error`、`policy_block`、`idempotency_mismatch`、`not_found`、`rpc_error`、`vault_error`、`simulation_reverted`、`aborted`、`missing_request_id`、`confirmation_required`、`tty_required`、`no_route`、`insufficient_allowance`。

所有金额字段都是字符串（避免 JS bigint 精度丢失），所有地址都是 EIP-55 checksum 格式。

## 审计日志：完整可追溯

每一次签名尝试（broadcast、rejected、replayed、user_aborted）都会往 `~/.wallet/audit.log` append 一行 JSON。文件用 `O_APPEND` 打开，并发写不会撕裂。**没有任何 CLI 命令读这个文件**——这是故意的，避免 Agent 用日志反推钱包状态。

```jsonc
{"kind":"aave_faucet","outcome":"broadcast","hash":"0x9b5264..."}     // 0.01 WBTC 铸造
{"kind":"approve","outcome":"broadcast","hash":"0x3675d9..."}          // WBTC → Aave Pool
{"kind":"aave_supply","outcome":"broadcast","hash":"0x78ae3f..."}      // 0.005 WBTC 存入
{"kind":"aave_borrow","outcome":"rejected"}                            // 150 USDC 被策略拦截
{"kind":"aave_borrow","outcome":"broadcast","hash":"0x9e4cba..."}      // 100 USDC 通过
{"kind":"aave_repay","outcome":"broadcast","hash":"0x069b74..."}       // repay --max
```

人类可以离线用这个日志完整还原钱包做了什么、什么时候做的，独立于链上 explorer（explorer 没有"调用方是谁"的标签）。

## Agent 能做什么 / 不能做什么

| 操作 | Agent | TTY |
|---|---|---|
| `balance` / `history` / `portfolio` | ✓ | ✓ |
| `send` / `approve` / `swap` / `aave *` （dry-run） | ✓ | ✓ |
| `send` / `swap` / `aave borrow`（broadcast） | ✓ 但必须策略通过 + 带 `--request-id` | ✓ |
| `--policy-bypass` | 拒绝 | 警告后允许 |
| `--unlimited` approve | 默认策略拒绝 | 默认策略拒绝 |
| `account create` / `import` | 拒绝（TTY only） | ✓ |
| `policy init` / 修改 policy.json | 拒绝 | ✓ |

## 什么不在范围内（暂时）

刻意没做的事，避免一次性铺得太大：

- **硬件钱包（Ledger / Trezor）**——这是主网非小额资金前的唯一硬阻塞。代码 gate 还在，要等 PR 落地。
- **WalletConnect / 浏览器 dApp 注入**——故意不做。需要 dApp UI 就用 MetaMask，两个钱包并存即可。
- **Stuck-tx replace / cancel**——主网 base-fee 飙升时交易可能挂几小时，Tier 2 路线图。
- **多 RPC 共识读**——目前是单 RPC，有 SPOF 风险，Tier 2。
- **Permit2 / EIP-712 签名**——0x v2 支持，但策略明确拒绝无限授权（Permit2 的前提），暂不集成。
- **Compound V3 / Spark / Morpho**——目前只做了 Aave V3。
- **跨链桥**——单链聚焦。

## 怎么自己跑一遍

```sh
# 0. 安装
git clone git@github.com:kaka-milan-22/wallet.git && cd wallet && uv sync

# 1. 配置（TTY，一次性）
export WALLET_SEPOLIA_RPC=https://ethereum-sepolia.publicnode.com
export ETHERSCAN_API_KEY=…
uv run wallet account create main
agent-vault set wallet-main-mnemonic
uv run wallet policy init
# 编辑 policy.json，把 Uniswap V3 SwapRouter02、Aave Pool、Aave Faucet
# 三个地址加进 contract_allowlist

# 2. 领 Sepolia ETH
open https://sepolia-faucet.pk910.de/                                    # PoW
open https://cloud.google.com/application/web3/faucet/ethereum/sepolia   # Google

# 3. 跑完整 demo
uv run wallet swap ETH USDC 0.001 --broadcast --yes
uv run wallet aave faucet WBTC 0.01 --broadcast --yes
uv run wallet approve set 0x29f2D40B... 0x6Ae43d3... 0.01 --broadcast --yes
uv run wallet aave supply WBTC 0.005 --broadcast --yes
uv run wallet aave borrow USDC 100 --broadcast --yes
uv run wallet aave repay USDC --max --broadcast --yes
uv run wallet aave withdraw WBTC --max --broadcast --yes
```

整套 demo 在 Sepolia 上几乎零成本（测试网 ETH 免费），每一步都走 dry-run → policy → idempotency → audit 的完整管道，把四道闸门各触发一次。

## 总结

这个钱包想解决的不是"再做一个 MetaMask"，而是"当我们的工作流里出现自主 Agent 时，钱包应该长什么样"。核心立场是：

1. **fail-closed 是默认**——没有策略就不能动钱，没有 request-id 就不能 broadcast，没有 allowlist 就不能转账。
2. **每一层都独立工作**——skill 是软建议，policy 是硬墙，idempotency 防重放，audit 留痕迹。任何一层失效，其它三层依然能拦住事故。
3. **不让 Agent 拥有它不需要的能力**——Agent 不能读审计日志、不能改策略、不能创建账户、不能领 mnemonic。
4. **JSON I/O 优先**——错误码可枚举、金额用字符串、stderr 不污染 stdout，让 Agent 调用方稳定可靠。

目前的状态：Sepolia 测试网完整跑通，主网小额可用，但**大额请等 Ledger 集成**——硬件钱包是 cold key 的最后一道门，软件钱包再小心，本质仍是 hot key。这条 ROADMAP 已经写好，欢迎 issue / PR。

代码仓库：[github.com/kaka-milan-22/wallet](https://github.com/kaka-milan-22/wallet)。
