---
title: "看起来像，本质是镜像：sops 与一个自研本地 KMS 的密码学分野"
date: 2026-06-01T10:00:00+08:00
draft: false
tags: ["Secrets Management", "sops", "age", "Cryptography", "Envelope Encryption", "KMS", "GitOps", "Security"]
categories: ["Security", "Tools"]
author: "Kaka"
description: "sops 和我自研的本地 KMS（AnB）第一眼看上去做的是同一件事——让 secret 不以明文裸奔。但顺着 envelope encryption 一路挖到封装密钥的那一层，会发现两者其实是一棵树的两个对称分叉，而岔路只有一个：包裹密钥用的是非对称还是对称。"
---

## 引言：一个"感觉很像"的直觉

我有两套都在用的 secret 管理工具。一套是 [sops](https://github.com/getsops/sops)，CNCF 旗下的老牌方案，配合 [age](https://github.com/FiloSottile/age) 做 backend；另一套是我自己用 Go 写的本地 KMS——代号 AnB，一个绑在 `127.0.0.1` 上、走 mTLS 的常驻 daemon，secret 以 `<agent-vault:key>` 占位符的形式暴露给上层 agent，真值由 daemon 在最后一刻替换。

某天我盯着这两个东西，冒出一个直觉：**它们好像在做同一件事**。都是"让 secret 不以明文形态停留在不该停留的地方"，都能让配置文件无明文进 git。那它们到底是不是冗余？我该不该把其中一个砍掉？

这篇文章是那个直觉的完整复盘。结论先放这儿：**它们确实共享同一个密码学底座，但从那个底座往上，几乎每一个设计选择都是镜像对称的——而所有分歧最终都能归因到一个点：包裹密钥的那一层，用的是非对称还是对称。** 这是一个很好的例子，说明"功能看起来重叠"和"架构本质相同"是两回事。

## 一、相同的底座：都是 envelope encryption

先说为什么"感觉像"是对的。两者的核心都是 **envelope encryption（信封加密）**，而且 AEAD 都用 **AES-256-GCM**。

envelope encryption 的思想是"用密钥加密密钥"，分层：

1. 一个随机生成的 **data key（数据密钥）** 负责加密真正的业务数据（secret value）。
2. data key 自己不裸存，而是被一个更高层的 **KEK（key-encryption-key，密钥加密密钥）** 包裹后存起来。
3. 解密时先解开 data key，再用 data key 解业务数据。

sops 是教科书式的 envelope：每个文件一个随机 data key，用 AES-256-GCM 加密文件里的各个 value，data key 再被 age / KMS 公钥加密塞进文件末尾的 metadata。

我的 AnB 是同一套结构，只是层数更显式：

```
operator passphrase
      │  Argon2id (M=64MiB, T=3, P=1, 16B salt)
      ▼
     KEK ──── AES-256-GCM ────► 封装 master key（存进 on-disk envelope）
      │
      ▼
  master key ── AES-256-GCM ──► 封装每一个 secret value（存进 vault）
```

落到代码上，最底层是一个通用的 `Seal` / `Open`，wire 格式是 `ivHex:tagHex:ctHex`（12 字节 nonce、16 字节 tag）：

```go
// Seal encrypts plaintext under key and returns "ivHex:tagHex:ctHex".
func Seal(key, plaintext []byte) (string, error) {
    gcm, _ := newGCM(key)              // AES-256-GCM
    iv := make([]byte, 12)
    rand.Read(iv)
    sealed := gcm.Seal(nil, iv, plaintext, nil) // ct || tag
    ct, tag := sealed[:len(sealed)-16], sealed[len(sealed)-16:]
    return hex(iv) + ":" + hex(tag) + ":" + hex(ct), nil
}
```

master key 是 32 字节 CSPRNG 随机，被 `Wrap` 用 KEK 封装；KEK 由 operator 的 master password 经 **Argon2id** 派生。一个干净的细节：passphrase 错误不需要单独校验，直接表现为 GCM 认证失败：

```go
// Unwrap reverses Wrap. A GCM auth failure surfaces as ErrBadPassword.
func Unwrap(env *Envelope, password string) ([]byte, error) {
    kek := DeriveKEK(password, salt, env.Params)  // Argon2id
    defer Wipe(kek)
    key, err := Open(kek, env.IV+":"+env.Tag+":"+env.Wrapped)
    if err != nil {
        return nil, ErrBadPassword  // 用 AEAD tag 当密码校验
    }
    return key, nil
}
```

到这一层，两者**真的像**：都是 AES-256-GCM 的信封加密，都让 secret 不裸奔。直觉没错。

## 二、第一层分野：无状态文件 vs 有状态 daemon

往上走第一步，形态就分叉了。

**sops 是无状态、文件式的。** 它没有常驻进程，加密产物**就是文件本身**——一个 `secrets.yaml`，里面是自包含的密文：

```yaml
db_host: prod-pg.internal          # 明文，key 和非敏感值不动
db_password: ENC[AES256_GCM,data:xK...,iv:...,tag:...,type:str]
```

secret 静态躺在 git / 磁盘里，用的时候 `sops -d` 当场解密。它本质上是"一个会自己加解密的文件"。

**AnB 是有状态、C/S 的。** Bob 是常驻 KMS daemon，绑 mTLS；Alice 是 client。secret 由 daemon 托管，运行时按需吐。它本质上是"一个会说话的保险柜"。

这个差异本身不分高下，但它牵出了下一个更本质的分野。

## 三、第二层分野：自包含密文 vs 引用 indirection

这一层是关键。两者都能让**配置文件无明文进 git**，但方式截然相反。

- **sops 文件里是 self-contained ciphertext（自包含密文）**：`ENC[AES256_GCM,...]` 本身就携带加密后的 secret，真值（密文形态）**在文件里**。
- **AnB 配置里是 reference（引用）**：`<agent-vault:db-password>` 只是个指针，真值**不在文件里、也不在仓库里**，而在 Bob 的 vault state 里。

一个是"secret 跟着文件走"，一个是"文件只带引用、secret 留在保险柜"。

这里要纠正一个容易踩的误区：很多人以为"占位符方案"只能做运行时注入、不能落盘。其实不然——AnB 同时提供两条出口，而且它们和 sops 的两个子命令几乎一一对应：

| 出口形态 | AnB | sops 对应物 |
|---|---|---|
| 注入进程 env、明文零落盘 | `alice exec` | `sops exec-env` |
| 物化明文进文件 | `alice template` / `write` | `sops exec-file` / `sops -d > file` |

`alice exec` 这条路只把占位符解析进**子进程的 env**，经 mTLS 向 daemon 要明文，在内存里替换后 `syscall.Exec` 替换进程镜像，明文从不落盘——alice 自己的堆被内核回收。这里有个值得学的安全细节：**argv 故意不扫占位符**，因为 `/proc/<pid>/cmdline` 默认 world-readable，secret 进 argv 会泄露给同机任意 uid，所以只走 env：

```go
// Argv (everything after --) is NOT scanned for placeholders — Linux
// /proc/<pid>/cmdline is world-readable by default, so secrets in argv
// would leak to any uid on the box. --env values land in child env only.
```

`alice template` 这条路才是"物化进文件"：读含占位符的源文件，渲染出明文，用 `tmp + rename` 原子写入，默认 `mode 0600`。而且它被刻意 gate 成 **TTY-only**——往磁盘写明文这件事，要求人在场才能做。

所以"占位符也能落盘进 git"这个判断是对的。但本质分野仍然只有一个：**谁是 source of truth。** sops 的源文件是自包含密文（secret 以加密形态躺在文件里），AnB 的源永远是纯引用模板（真值永驻中心 vault）。

## 四、最深一层：封装密钥用非对称还是对称

挖到这里才到根。前面所有的差异——单机 vs GitOps、引用 vs 自包含密文、要不要 daemon——**全都能归因到一个密码学决策：包裹 data key / master key 的那一层，用的是非对称还是对称。**

| | 包裹密钥的算法 | 必然的分发属性 |
|---|---|---|
| **sops** | **非对称**：age（X25519）/ KMS 封 data key | 公钥加密 → 多 recipient、GitOps 分发，无需 passphrase、无需常驻组件 |
| **AnB** | **对称**：Argon2id(passphrase) 派生 KEK 封 master key | 解封必须有 master password → 中心化、单 operator，天然配 daemon 托管 |

AEAD 两边都是 AES-256-GCM，完全一样。**真正的岔路是 wrapping 那层是非对称还是对称。**

- sops 选**非对称**（age / KMS）。所以"任何人拿公钥就能加密、各 recipient 用各自私钥解密"。这是它能进 GitOps、能配多环境 recipient 的密码学根因——加密侧只需要公钥（可以放进 CI），解密侧各持私钥。
- AnB 选 **Argon2id + passphrase 对称封装**。所以它**必然**是"一个 master password 守一个 vault"的中心化模型——这正是为什么它要 Bob 这个常驻 daemon、为什么 secret 留在 vault 只发引用、为什么适合单机 / agent 而非 GitOps 拉取。

换句话说，前面那些看似独立的架构差异，不是巧合，全是这一层算法选择的下游。**选了非对称，你就走向分发友好；选了对称 passphrase，你就走向中心托管。** 这是一条因果链，不是一堆并列的特性。

## 五、两个 sops 没有的细节

公平起见，AnB 这套对称中心模型也带来了 sops 给不了的东西：

**版本化 master key + lazy rewrap。** on-disk 的 envelope 支持多个 master key 版本，每段密文带 `v<N>:` 前缀标明用哪一版封的。轮换时加一把新 key、bump current 指针，旧 secret 仍能用旧 key 解；下次用到某条旧密文时，**顺手用 current key 重新封装写回**（lazy rewrap）。对比 sops 轮换得 `sops updatekeys` 整文件重加密、重新 commit，AnB 是"用到才迁移"的渐进式轮换。

**运行时审计与即时轮换。** 中心化 daemon 天然有"谁、何时、取了哪个 key"的统一审计点；改 vault 即生效，所有占位符下次注入自动拿新值。sops 没有运行时审计——谁 `sops -d` 了文件，它并不知道。

**诚实的 Wipe。** 代码里尽力把派生出的 KEK 字节清零，但注释直接承认"Go string 擦不掉"——不吹牛，这种地方反而让人放心。

## 总结：同一棵树的两个分叉

回到最初那个"它们是不是冗余"的问题。答案是：**不冗余，互补。**

它们的相同处是真实的——都是 AES-256-GCM 的 envelope encryption，都让 secret 不以明文裸奔。但从 envelope 往上的每一个分歧，都能一路归因到**封装层选了对称（Argon2id passphrase）还是非对称（age 公钥）**这一个决策：

- 要把 secret 推给一群"拉"模式、且你不完全信任的目标（K8s 集群、CI runner）——**sops 的自包含密文 + 非对称多 recipient 更顺**。
- 要在你自己掌控的主机上给 agent / 进程供值，还想要即时轮换 + 统一审计 + secret 永不落 git——**AnB 的引用 indirection + 中心 vault 更优雅**。

所以我两个都留着：**sops 管 GitOps 的静态配置，AnB 管本地 agent 的运行时注入。** 一个把真值（密文）分发进文件本身，一个把真值锁在中心 vault、文件只留指针。

最后一个值得记住的方法论：当你觉得两个工具"好像在做同一件事"时，别停在功能层比对——**一路挖到它们的密码学/数据模型底座，往往会发现表面的相似只是同一个数学原语的两种封装，而真正决定它们各自适合什么场景的，是那个你一开始根本没注意到的、藏在最底下的算法选择。**

---

*注：文中 AnB 是我自研的本地 KMS 实验项目，设计上兼容 agent-vault 的 `<agent-vault:key>` 占位符语法。本文聚焦设计哲学与密码学权衡，不涉及任何具体凭据或部署细节。*
