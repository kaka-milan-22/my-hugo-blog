---
title: "Building an AI-Agent-Native EVM Wallet From Scratch: When MetaMask No Longer Fits the LLM Era"
date: 2026-05-13T10:00:00+08:00
draft: false
tags: ["Wallet", "EVM", "Ethereum", "AI Agent", "Web3", "DeFi", "Aave", "Uniswap", "Python", "Security"]
categories: ["Tools", "Architecture"]
author: "Kaka"
description: "An EVM wallet built for LLM agents: pure CLI, JSON I/O, policy gating, FIFO secret transport, and direct Aave / Uniswap integration — with a full borrow/swap flow proven on Sepolia."
---

## Why We Need an Agent-Native Wallet

Mainstream EVM wallets — MetaMask, Rabby, Phantom — are all fundamentally **built for humans**: browser extensions, WalletConnect QR codes, `window.ethereum` injection, click-to-sign popups. The moment you want an LLM agent to autonomously execute a workflow like *"swap ETH to USDC on Uniswap, deposit the USDC into Aave for yield, and auto-repay if HF drops below 1.5"*, the entire browser-wallet stack flips from "safety rails" into "an unbridgeable chasm."

A common shortcut is to just feed the private key to the agent. That degrades the wallet to a single `web3.eth.account.from_key()` call, letting a model that hallucinates operate your funds with zero fail-safe. One prompt injection, one wrong contract address, one "I thought it would dry-run but it broadcast instead" — and the money is gone.

This post introduces an open-source project I built called `wallet`: an **AI-agent-native EVM wallet**. It's a Python CLI where every write operation flows through a six-stage pipeline: *preview → policy → idempotency → confirmation → signing → audit*. The whole design has been validated on Sepolia with real on-chain transfers, Uniswap V3 swaps, and a full Aave V3 borrow/repay loop — every step backed by an Etherscan transaction hash.

It is not a MetaMask replacement. MetaMask serves the "human clicking buttons in a browser" scenario; this wallet serves the "agent running cron tasks in a terminal" scenario. The two responsibilities are orthogonal.

## Overall Architecture: Four Layers of Defense

The core design principle is exactly one rule: **the agent may invoke any read operation, but every write operation must pass through four independent security gates.**

| Layer | Location | Failure mode it stops |
|---|---|---|
| **skill** | `docs/skills/wallet-agent.skill.md` | Tells the agent how to use the tool: always dry-run first, use a fresh request-id per broadcast, never pass `--policy-bypass` |
| **policy** | `~/.wallet/policy.json` | Hard pre-signing intercepts: amount caps, recipient allowlist, contract allowlist, refuse unlimited approvals, `min_health_factor` |
| **idempotency** | `~/.wallet/idempotency.json` | The same `--request-id` replayed returns the cached result; never re-broadcasts |
| **audit** | `~/.wallet/audit.log` | Append-only JSON-lines log of every signing attempt. **No CLI command reads it** — the agent cannot reverse-engineer state from the log |

The four are AND-composed: any single denial fails closed. Policy also has one counter-intuitive default: **when policy is uninitialized, everything is denied** — not "open by default." The reasoning: a freshly-installed wallet should not be able to move a single cent until you've explicitly declared who you trust.

```
src/wallet/
  cli/_common.py       # confirm_and_broadcast pipeline
  core/policy.py       # policy evaluation
  core/signer.py       # mnemonic -> privkey -> signature (in-memory only)
  storage/audit.py     # append-only audit
  storage/idempotency.py
  storage/vault.py     # agent-vault adapter + FIFO transport
```

## Account Management: Mnemonics Never Touch Disk

Standard HD wallet (BIP-39 + BIP-44), but with extra care around secret storage.

```sh
# Run this in your terminal (NOT in an agent context)
wallet account create main
# Prints the 12-word mnemonic + tells you to run:
agent-vault set wallet-main-mnemonic
# Paste the mnemonic into agent-vault
```

Key points:

1. `account create` / `account import` are **hard-coded as TTY-only**. Running them under an agent context returns an immediate refusal, because mnemonics would necessarily print to stdout and enter the LLM's context window — effectively publishing them.
2. The mnemonic lives encrypted in `agent-vault` (a local vault that's invisible to the agent).
3. At signing time, the mnemonic flows from the vault process to the signer process through a **Unix FIFO (named pipe)**. FIFOs are kernel pipes — nothing hits disk, and the buffer vanishes the moment a process exits. The fallback path uses short-lived `/tmp/wallet-secret-*` files, but the FIFO path never triggers it.

You can verify with this watcher script that nothing leaks:

```sh
while true; do ls /tmp/wallet-secret-* 2>/dev/null && echo LEAK; done &
wallet send <to> 0.0001 --broadcast
# A LEAK line means the fallback was hit
```

A single mnemonic can derive arbitrarily many addresses:

```sh
wallet account derive main --index 1 --as second
# main:   0x34a9...36c7  (m/44'/60'/0'/0/0)
# second: 0xFb0b...d126  (m/44'/60'/0'/0/1)
# shared vault key: wallet-main-mnemonic
```

## Transfers: Dry-Run by Default, Confirmation Required

The most common EVM wallet accident is "I sent it to the wrong address." This wallet's default behavior is dry-run — you must explicitly add `--broadcast` to actually sign and submit:

```sh
wallet send 0xabc... 0.01                              # preview only, no signing
wallet send @alice 0.01 --broadcast                    # prompts y/N
wallet send @alice 0.01 --broadcast --yes              # skip prompt (required for agent mode)
wallet send 0xabc... 50 --token USDC --broadcast --yes # ERC-20 transfer
```

`@alice` is an address-book alias (`wallet book add alice 0xabc...`). For agent calls, I recommend using raw `0x...` addresses directly to avoid polluting the address book.

On-chain proof (Sepolia):

| Operation | Amount | Hash |
|---|---|---|
| First ETH transfer | 0.001 ETH | `0x5abce81b...` |
| Transfer with `--request-id` | 0.0001 ETH | `0xca5882b3...` |
| Larger transfer | 0.02 ETH | `0x3a1e1c16...` |

## ERC-20 Approvals: Unlimited Approvals Denied by Default

The most popular phishing playbook is to trick the user into signing `approve(spender, 2^256-1)`, after which the attacker can drain the token at any time. This wallet's default policy sets `deny_unlimited_approve: true`, so any `--unlimited` call is rejected:

```sh
wallet approve set USDC <spender> 100               # bounded approval
wallet approve set USDC <spender> --unlimited       # blocked by policy
wallet approve show USDC --spender 0x...            # inspect current allowance
wallet approve revoke USDC <spender>                # one-shot revoke
```

If you genuinely need unlimited approval (e.g., to save gas with certain routers), you have to flip the policy field manually in a TTY. **The agent cannot modify policy files.**

## Swap: 0x Aggregator + Uniswap V3 Fallback

The default `--via auto` first tries the 0x aggregator (best quote across ~100 DEXes); if 0x is unavailable (no API key, no testnet liquidity), it automatically falls back to calling Uniswap V3 SwapRouter02 directly:

```sh
wallet swap ETH USDC 0.001                             # dry-run, auto route
wallet swap USDC WETH 100 --slippage-bps 50            # 0.5% slippage
wallet swap ETH USDC 0.001 --broadcast --yes           # real signing
wallet swap ETH USDC 0.001 --via uniswap-v3            # force UniV3 direct
wallet swap ETH USDC 0.001 --via 0x                    # force aggregator
```

The preview spells out route, expected output, and minimum acceptable output:

```
swap route          ETH > 500bps > USDC
expected out        15.565555 USDC
min out (0.5% slip) 15.487727 USDC
```

Sepolia run: 0.001 ETH swapped for 15.565555 USDC through the Uniswap V3 0.05% pool (`0xe96c972c...`). Testnet liquidity is so thin that the fill exactly matched the quote; on mainnet there will be slippage, with `min_out` as the backstop.

## Aave V3 Borrowing: The Full Loop

This is where "agent-native" really pays off. A complete supply → withdraw → borrow → repay cycle, all in the terminal:

### Read operations (free RPC queries)

```sh
wallet aave positions               # your deposits / debt + Health Factor
wallet aave rates                   # supply / variable-borrow APR per pool
wallet aave rates --token USDC      # filter a single token
```

### Faucet: Get test tokens without MetaMask

The Aave staging.aave.com faucet officially requires MetaMask to mint mock tokens. But the faucet contract itself is permissionless, so I just wrapped it as a CLI command:

```sh
wallet aave faucet WBTC 0.01 --broadcast --yes
# on-chain: 0x9b5264e7...
```

### Full borrow loop

The actual on-chain sequence I ran:

| Step | Operation | Hash |
|---|---|---|
| 1 | approve WBTC → Aave Pool | `0x3675d91e...` |
| 2 | supply 0.005 WBTC | `0x78ae3f30...` |
| 3 | withdraw 0.002 WBTC (partial) | `0xd651449b...` |
| 4 | attempt borrow 150 USDC | **blocked by policy** (estimated HF 1.500 < min 2.0) |
| 5 | borrow 100 USDC | `0x9e4cbabe...` (HF 2.250 ✓) |
| 6 | repay USDC --max ($100.0003) | `0x069b74d7...` |
| 7 | withdraw WBTC --max | `0xc6dd9bad...` |

After the loop completed the position was zeroed out with no leftover debt.

### `min_health_factor`: The Killer Feature for Autonomous Borrowing

This is the policy field I find most valuable. Aave's on-chain check only reverts when HF < 1 (i.e., "already liquidatable") — but by then it's too late: the next price tick will liquidate you.

`min_health_factor: 2.0` makes the wallet compute the estimated post-tx HF **before signing**, and block anything that falls below the threshold:

```
Preview output:
  current HF            ∞
  estimated HF after    1.500
  liquidation HF        1.000 (Aave revert threshold)

→ error: policy_block — hf-would-drop-below-min:1.500<2.0
```

The HF logic reads Aave oracle's `getAssetPrice` plus each reserve's liquidation threshold:

- borrow: `new_debt = current_debt + amount × price`
- withdraw: `new_weighted_collateral = current_weighted - amount × price × asset_LT`

This is **far stricter** than Aave's own check, specifically to stop an agent (which has no real-time market judgment) from "borrowing right up to the cliff edge."

## Security in Action: Each Gate, Caught in the Wild

### Policy uninitialized → deny everything

```sh
wallet send 0xabc... 0.001 --broadcast --yes
# error: policy_block — no-policy-configured-run-wallet-policy-init
```

### Recipient not in allowlist

```sh
wallet send 0xUNKNOWN 0.001 --broadcast --yes
# error: policy_block — recipient-not-in-allowlist
```

### Contract not in allowlist

Before invoking the Uniswap router / Aave Pool / Aave Faucet, the wallet checks whether the contract address sits in `contract_allowlist`:

```sh
wallet swap USDC WETH 1 --broadcast --yes
# error: policy_block — swap-router-not-in-contract-allowlist
```

### Insufficient allowance → emits a fix-it command

```sh
wallet aave repay USDC 99.5 --broadcast --yes
# error: insufficient_allowance
# data.suggested_command: "wallet approve set USDC 0x6Ae4...8951 99.5"
```

The `suggested_command` field tells the agent exactly what to run next, so it doesn't waste gas on transactions that are guaranteed to revert.

### Aave error code decoding

When the Aave Pool reverts, it returns a numeric error code. The wallet translates it to plain English:

```
error: simulation_reverted — aave:51 (SUPPLY_CAP_EXCEEDED — Aave's supply cap is full
  (common on testnets). Try a smaller amount or a different reserve.)
```

In JSON output, `data.aave_error_code: "51"` lets the agent branch on the numeric code.

### Idempotency: Same request-id never broadcasts twice

```sh
RID=$(uuidgen)
wallet send second 0.0001 --broadcast --yes --request-id "$RID"
# → submitted: 0x12e11533...  (broadcast)

wallet send second 0.0001 --broadcast --yes --request-id "$RID"
# → submitted: 0x12e11533...  (replayed_idempotent, same hash, no re-broadcast)
```

`~/.wallet/idempotency.json` stores `(request_id, fingerprint, tx_hash)` triples. If the same id arrives with different parameters, it raises `idempotency_mismatch` — protecting the agent from accidentally double-spending on retry.

## JSON Output: Built for Agents

Every command supports `--json` (or `WALLET_JSON=1`), emitting a single-line JSON envelope. **stderr never pollutes stdout:**

```sh
WALLET_JSON=1 wallet send second 0.0001 --broadcast --yes --request-id "$(uuidgen)" | jq .
```

```jsonc
// Success
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

// Error (all error codes share the same schema)
{
  "ok": false,
  "command": "aave.borrow",
  "error": "policy_block",
  "reason": "hf-would-drop-below-min:1.500<2.0",
  "data": {...}
}
```

The closed set of error codes: `validation_error`, `policy_block`, `idempotency_mismatch`, `not_found`, `rpc_error`, `vault_error`, `simulation_reverted`, `aborted`, `missing_request_id`, `confirmation_required`, `tty_required`, `no_route`, `insufficient_allowance`.

All amount fields are strings (to avoid JS bigint precision loss), and every address is in EIP-55 checksum form.

## Audit Log: Full Traceability

Every signing attempt — broadcast, rejected, replayed, user_aborted — appends one JSON line to `~/.wallet/audit.log`. The file is opened `O_APPEND`, so concurrent writes don't tear. **No CLI command reads this file** — that's intentional, so the agent can't reverse-engineer wallet state from the log.

```jsonc
{"kind":"aave_faucet","outcome":"broadcast","hash":"0x9b5264..."}     // 0.01 WBTC mint
{"kind":"approve","outcome":"broadcast","hash":"0x3675d9..."}          // WBTC → Aave Pool
{"kind":"aave_supply","outcome":"broadcast","hash":"0x78ae3f..."}      // 0.005 WBTC deposit
{"kind":"aave_borrow","outcome":"rejected"}                            // 150 USDC blocked by policy
{"kind":"aave_borrow","outcome":"broadcast","hash":"0x9e4cba..."}      // 100 USDC passed
{"kind":"aave_repay","outcome":"broadcast","hash":"0x069b74..."}       // repay --max
```

A human can fully reconstruct, offline, what the wallet did and when — independent of the on-chain explorer (which has no "who called this" tag).

## What the Agent Can / Cannot Do

| Operation | Agent | TTY |
|---|---|---|
| `balance` / `history` / `portfolio` | ✓ | ✓ |
| `send` / `approve` / `swap` / `aave *` (dry-run) | ✓ | ✓ |
| `send` / `swap` / `aave borrow` (broadcast) | ✓ but only if policy passes + `--request-id` provided | ✓ |
| `--policy-bypass` | denied | warns, then allows |
| `--unlimited` approve | denied by default policy | denied by default policy |
| `account create` / `import` | denied (TTY only) | ✓ |
| `policy init` / editing policy.json | denied | ✓ |

## What's Deliberately Out of Scope (For Now)

Things consciously left out so the project doesn't bloat:

- **Hardware wallets (Ledger / Trezor)** — the only hard blocker before serious mainnet sums. The code gates are wired up; we're waiting on the PR.
- **WalletConnect / browser dApp injection** — intentionally not implemented. If you need a dApp UI, just use MetaMask alongside this one.
- **Stuck-tx replace / cancel** — mainnet base-fee spikes can leave transactions pending for hours; Tier 2 roadmap.
- **Multi-RPC consensus reads** — currently single RPC, so there's an SPOF; Tier 2.
- **Permit2 / EIP-712 signing** — 0x v2 supports it, but the policy explicitly refuses unlimited approval (which Permit2 assumes), so it's deferred.
- **Compound V3 / Spark / Morpho** — currently Aave V3 only.
- **Cross-chain bridges** — single-chain focus.

## Running It Yourself

```sh
# 0. Install
git clone git@github.com:kaka-milan-22/wallet.git && cd wallet && uv sync

# 1. Configure (TTY, one-shot)
export WALLET_SEPOLIA_RPC=https://ethereum-sepolia.publicnode.com
export ETHERSCAN_API_KEY=…
uv run wallet account create main
agent-vault set wallet-main-mnemonic
uv run wallet policy init
# Edit policy.json: add the Uniswap V3 SwapRouter02, Aave Pool, and Aave Faucet
# addresses to contract_allowlist

# 2. Get Sepolia ETH
open https://sepolia-faucet.pk910.de/                                    # PoW
open https://cloud.google.com/application/web3/faucet/ethereum/sepolia   # Google

# 3. Run the full demo
uv run wallet swap ETH USDC 0.001 --broadcast --yes
uv run wallet aave faucet WBTC 0.01 --broadcast --yes
uv run wallet approve set 0x29f2D40B... 0x6Ae43d3... 0.01 --broadcast --yes
uv run wallet aave supply WBTC 0.005 --broadcast --yes
uv run wallet aave borrow USDC 100 --broadcast --yes
uv run wallet aave repay USDC --max --broadcast --yes
uv run wallet aave withdraw WBTC --max --broadcast --yes
```

The whole demo costs essentially nothing on Sepolia (testnet ETH is free), and every step exercises the full dry-run → policy → idempotency → audit pipeline, firing each of the four gates at least once.

## Summary

This wallet isn't trying to be "yet another MetaMask." It's trying to answer the question: *when autonomous agents enter the workflow, what should a wallet look like?* The core stances are:

1. **Fail-closed is the default** — no policy means no funds move; no request-id means no broadcast; no allowlist means no transfer.
2. **Every layer works independently** — the skill is a soft suggestion, policy is a hard wall, idempotency stops replay, audit leaves a trail. If any single layer breaks, the other three still hold.
3. **Don't give the agent capabilities it doesn't need** — the agent can't read the audit log, can't edit the policy, can't create accounts, can't retrieve the mnemonic.
4. **JSON I/O first** — closed-set error codes, string-encoded amounts, no stderr-to-stdout pollution: the caller stays stable and reliable.

Current status: Sepolia is fully working end-to-end; small mainnet amounts are usable, but **wait for the Ledger integration before going large** — a hardware wallet is the last cold-key gate, and no matter how careful software is, it's still a hot key. That roadmap item is written up; issues and PRs welcome.

Repository: [github.com/kaka-milan-22/wallet](https://github.com/kaka-milan-22/wallet).
