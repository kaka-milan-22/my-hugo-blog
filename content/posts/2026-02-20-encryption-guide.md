---
title: "运维必知的加密算法与工具实战"
date: 2026-02-20T10:00:00+08:00
draft: false
tags: ["Encryption", "Security", "TLS", "OpenSSL", "GPG"]
categories: ["DevOps"]
description: "从原理到工程实践：主流加密算法详解 + 开源工具用法速查，覆盖 OpenSSL、GPG、age、WireGuard、HashiCorp Vault 等。"
author: "Kaka"
---

> 加密无处不在：HTTPS、SSH、VPN、数据库密码存储……作为运维，不需要实现算法，但要知道用哪个、怎么用对。

---

## 一、加密算法全景

加密算法分三大类，各有分工，实际系统里往往**组合使用**：

```
┌─────────────────────────────────────────────────────────┐
│                      加密算法三大类                       │
│                                                         │
│  对称加密          非对称加密           哈希              │
│  （同一把钥匙）     （公私钥对）         （单向不可逆）     │
│                                                         │
│  AES-GCM          RSA                 SHA-256           │
│  ChaCha20         ECC / Ed25519       SHA-3             │
│                   ECDH                bcrypt / Argon2   │
│                                                         │
│  快，适合大数据     慢，用于密钥交换      不解密，用于校验   │
└─────────────────────────────────────────────────────────┘
```

**三者的协作模式（以 HTTPS 为例）：**

```
1. 非对称加密（ECC）    →  握手阶段，安全交换对称密钥
2. 对称加密（AES-GCM）  →  用交换的密钥加密实际传输数据
3. 哈希（SHA-256）      →  验证数据完整性，防篡改
```

---

## 二、对称加密

### AES（当前最主流）

**特点：**
- 密钥长度：128 / 192 / 256 bit，**AES-256 是工程标准选择**
- 速度极快，现代 CPU 有硬件指令集（AES-NI）加速
- 模式选 **AES-256-GCM**（认证加密 = 机密性 + 完整性二合一），避免老旧的 CBC/ECB

**AES 各模式对比：**

| 模式 | 完整性校验 | 是否推荐 | 备注 |
|------|----------|---------|------|
| ECB  | ❌ | ❌ 禁用 | 相同明文→相同密文，图形加密缺陷经典案例 |
| CBC  | ❌ | ⚠️ 避免 | 需要额外 HMAC，易出 Padding Oracle 攻击 |
| GCM  | ✅ | ✅ 首选 | 认证加密，同时保证机密性和完整性 |
| CCM  | ✅ | ✅ 可用 | IoT 场景常见 |

### ChaCha20-Poly1305

**特点：**
- Google 设计，专为**无 AES 硬件加速**的设备优化（ARM、移动端）
- 纯软件实现比 AES 更快，且没有时序攻击风险
- TLS 1.3 和 AES-GCM 并列推荐，WireGuard 默认使用

---

## 三、非对称加密

### RSA

**特点：**
- 历史最久，兼容性最好
- 现在最低 **2048-bit**，推荐 **4096-bit**
- 缺点：密钥长、慢，实际**不直接加密数据**，主要用于加密对称密钥或数字签名
- 安全性依赖大整数分解难题

### ECC（椭圆曲线，趋势替代 RSA）

**特点：**
- 256-bit ECC ≈ 3072-bit RSA 的安全强度，密钥短得多
- 速度比 RSA 快 10x 以上
- **Curve25519** 是目前最现代的曲线，被 WireGuard、Signal、现代 SSH 采用

**常见曲线选型：**

| 曲线 | 安全性 | 推荐度 | 使用场景 |
|------|-------|--------|---------|
| P-256 (secp256r1) | 128-bit | ✅ | TLS、HTTPS 广泛兼容 |
| P-384 | 192-bit | ✅ | 政府/金融高安全要求 |
| Curve25519 | 128-bit | ✅ 首选 | WireGuard、SSH、Signal |
| secp256k1 | 128-bit | ⚠️ | Bitcoin，通用场景不推荐 |

### ECDH / ECDHE（密钥交换）

- ECDH：静态密钥交换
- **ECDHE**：临时密钥（Ephemeral），每次会话生成新密钥对，提供**前向保密（PFS）**
- TLS 1.3 强制使用 ECDHE，废弃了静态 RSA 密钥交换

---

## 四、哈希算法

### SHA-256 / SHA-3

- **SHA-256**：目前最广泛，Bitcoin、Git、TLS 证书、Docker 镜像摘要全在用
- **SHA-3**（Keccak）：结构与 SHA-2 完全不同，防止 SHA-2 被破解时的备胎，以太坊使用
- **SHA-1**：已死，2017 年 Google SHAttered 攻击证明可碰撞，**禁止用于安全场景**
- **MD5**：更死，只适合非安全场景的文件校验（如下载完整性）

### bcrypt / Argon2（密码存储专用）

**为什么不能用 SHA-256 存密码？**

```
SHA-256("password123") 只需 0.001ms → 攻击者可以每秒算 10亿次
bcrypt("password123")  需要 100ms   → 攻击者每秒只能算 10次
```

bcrypt / Argon2 故意设计得"慢"，通过调节 cost factor 对抗暴力破解。

| 算法 | 推荐度 | 特点 |
|------|-------|------|
| MD5 | ❌ 禁用 | 存密码是灾难 |
| SHA-256 | ❌ 禁用存密码 | 太快，不适合密码 |
| bcrypt | ✅ 可用 | 经典，不支持并行 |
| scrypt | ✅ 可用 | 内存密集，防 ASIC |
| **Argon2id** | ✅ 首选 | 2015 年大赛冠军，内存+CPU 双重抗性 |

---

## 五、开源工具实战

### 5.1 OpenSSL — 瑞士军刀

几乎所有 Linux 系统自带，TLS/证书/加解密的万能工具。

#### 生成密钥和证书

```bash
# 生成 ECC 私钥（推荐 Curve25519 或 P-256）
openssl genpkey -algorithm ED25519 -out private.key

# 生成 RSA 4096 私钥
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -out rsa.key

# 生成自签名证书（测试用）
openssl req -x509 -newkey ed25519 \
  -keyout key.pem -out cert.pem \
  -days 365 -nodes \
  -subj "/CN=example.com"

# 查看证书信息
openssl x509 -in cert.pem -text -noout

# 查看证书有效期
openssl x509 -in cert.pem -noout -dates
```

#### AES 文件加解密

```bash
# 加密文件（AES-256-GCM，交互输入密码）
openssl enc -aes-256-gcm -salt -pbkdf2 -iter 100000 \
  -in secret.txt -out secret.txt.enc

# 解密
openssl enc -d -aes-256-gcm -pbkdf2 -iter 100000 \
  -in secret.txt.enc -out secret.txt

# 加密并 base64 编码（便于传输）
openssl enc -aes-256-cbc -salt -pbkdf2 -a \
  -in secret.txt -out secret.b64.enc
```

#### 哈希校验

```bash
# 计算 SHA-256
openssl dgst -sha256 file.tar.gz
sha256sum file.tar.gz  # 或者直接用这个

# 校验文件完整性
echo "abc123...  file.tar.gz" | sha256sum --check

# HMAC（带密钥的哈希，验证来源）
openssl dgst -sha256 -hmac "mysecret" file.txt
```

#### TLS 调试

```bash
# 查看远端证书链
openssl s_client -connect example.com:443 -showcerts

# 检查证书是否快过期（CI/CD 巡检常用）
openssl s_client -connect example.com:443 2>/dev/null \
  | openssl x509 -noout -enddate

# 测试支持的 TLS 版本和密码套件
openssl s_client -connect example.com:443 -tls1_3
openssl ciphers -v 'TLSv1.3'

# 生成随机密钥（生成 secret key）
openssl rand -hex 32
openssl rand -base64 32
```

---

### 5.2 GPG — 文件/邮件加密签名

适合文件传输加密、代码签名、密钥管理。

```bash
# 生成密钥对（推荐 Ed25519）
gpg --full-generate-key
# 选择: (9) ECC and ECC → Curve 25519

# 列出密钥
gpg --list-keys
gpg --list-secret-keys

# 导出公钥（给对方）
gpg --export --armor user@example.com > pubkey.asc

# 导入对方公钥
gpg --import pubkey.asc

# 加密文件（用对方公钥）
gpg --encrypt --recipient user@example.com secret.txt
# 生成 secret.txt.gpg

# 解密（用自己私钥）
gpg --decrypt secret.txt.gpg > secret.txt

# 对文件签名（证明是你发的）
gpg --sign --armor secret.txt          # 签名+内容合并
gpg --detach-sign --armor secret.txt   # 分离签名 → secret.txt.asc

# 验证签名
gpg --verify secret.txt.asc secret.txt

# 对称加密（不用密钥对，只用密码）
gpg --symmetric --cipher-algo AES256 secret.txt

# 签名 Git commit（配置后每次 commit 自动签名）
git config --global user.signingkey <KEY_ID>
git config --global commit.gpgsign true
git log --show-signature
```

---

### 5.3 age — 现代文件加密工具

比 GPG 简单很多，专注文件加密，推荐用于 **secrets 文件加密**（配合 sops）。

#### 普及程度与算法

**普及程度：** DevOps/安全圈子里已相当流行（GitHub 21k+ Stars），但还没到 OpenSSL/GPG 那种全行业量级。NixOS 社区几乎把 age 当作 secrets 管理默认方案，SOPS、Terragrunt、ArgoCD（helm-secrets）都已集成。更适合个人、小团队、GitOps 场景，企业级 secrets 管理仍以 Vault/KMS 为主。

**底层算法（全部开源标准）：**

```
密钥交换：  X25519          （Curve25519 的 ECDH，WireGuard 也用）
加密算法：  ChaCha20-Poly1305  （认证加密，Google 设计）
密码派生：  scrypt           （密码短语模式，内存密集型防暴力破解）
格式规范：  age-encryption.org/v1  （公开文档，任何人可实现）
```

没有任何私有算法，全是经过严格审计的成熟密码原语。作者 Filippo Valsorda 是 Go 语言官方密码库维护者，在密码学圈信誉极高。多语言实现：Go（官方）、Rust（rage）、TypeScript（typage）。age v1.3.0 已加入**后量子密钥支持**（ML-KEM），比 GPG 领先。

#### 基本用法

```bash
# 安装
brew install age          # macOS
apt install age           # Ubuntu 22.04+

# 生成密钥对
age-keygen -o key.txt
# 输出：Public key: age1ql3z7hjy...

# 用公钥加密
age -r age1ql3z7hjy... secret.txt > secret.txt.age

# 用密钥文件解密
age -d -i key.txt secret.txt.age > secret.txt

# 用密码加密（不用密钥对）
age -p secret.txt > secret.txt.age

# 加密整个目录（配合 tar）
tar czf - ./secrets/ | age -r age1ql3z7hjy... > secrets.tar.gz.age

# 解密
age -d -i key.txt secrets.tar.gz.age | tar xzf -

# 支持 SSH 公钥直接加密（零额外配置）
age -R ~/.ssh/id_ed25519.pub secret.txt > secret.txt.age
age -d -i ~/.ssh/id_ed25519 secret.txt.age > secret.txt

# 加密给多人（payload 只有一份，文件大小几乎不变）
age -r age1alice... -r age1bob... -r age1carol... secret.txt > secret.txt.age

# 直接从 GitHub 拉公钥加密（对方零配置）
curl https://github.com/username.keys | age -R - secret.txt > secret.txt.age
```

#### SSH 公私钥加解密的原理

`age -R ~/.ssh/id_ed25519.pub` 看起来像"公钥加密、私钥解密"，但背后原理和 RSA 不同：

**Ed25519 本身只能做签名，不能直接加密。** age 做了一个数学转换：

```
SSH Ed25519 私钥
      ↓  数学转换（Curve25519 点转换）
X25519 密钥（可做 ECDH 密钥交换）
      ↓  ECDH 派生对称密钥
ChaCha20-Poly1305 加密数据
```

Ed25519（签名用）和 X25519（加密用）都基于 Curve25519 曲线，数学上可以互相转换，age 利用了这一点。

**与 RSA 的本质区别：**

| | RSA | age + Ed25519 |
|--|-----|---------------|
| 公钥加密 | 直接用公钥加密数据 | 公钥做 ECDH 交换出对称密钥，再加密数据 |
| 私钥解密 | 直接用私钥解密 | 私钥重新做 ECDH 还原对称密钥，再解密 |
| 本质 | 非对称加密数据 | 混合加密（非对称保护对称密钥） |

所有现代加密都是混合加密，非对称只用来交换/保护那把对称密钥，真正加密数据的永远是 ChaCha20/AES 这类对称算法。

#### .age 文件格式（加密内容是什么）

执行 `age -r age1ql3z7hjy... secret.txt > secret.txt.age` 后，输出文件不只是密文，而是有完整结构：

```
┌─────────────────────────────────────────────┐
│              .age 文件结构                   │
├─────────────────────────────────────────────┤
│  Header（明文可读）                          │
│  age-encryption.org/v1                      │  ← 格式版本标识
│  -> X25519 <ephemeral_public_key>            │  ← 临时公钥（每次随机生成）
│  <wrapped_file_key>                         │  ← 被加密的文件对称密钥
│  ---                                        │  ← header 结束分隔符
├─────────────────────────────────────────────┤
│  Payload（二进制密文）                        │
│  ChaCha20-Poly1305 加密的实际数据            │  ← 真正的加密内容
└─────────────────────────────────────────────┘
```

**Header 是明文，这是故意设计的** —— 解密方需要从 header 拿到临时公钥，才能重新做 ECDH 还原出对称密钥。没有对应私钥，header 里的加密密钥无法被还原，不影响安全性。

**完整加解密流程：**

```
加密时：
  1. 随机生成 file_key（32字节对称密钥）
  2. 随机生成 ephemeral 密钥对
  3. ephemeral 私钥 + 接收方公钥 → ECDH → 派生 wrapping_key
  4. wrapping_key 加密 file_key → 写入 Header
  5. file_key 加密实际数据 → 写入 Payload
  6. 丢弃 ephemeral 私钥（提供前向保密）

解密时：
  1. 从 Header 读取 ephemeral 公钥
  2. ephemeral 公钥 + 接收方私钥 → ECDH → 还原 wrapping_key
  3. wrapping_key 解密 Header 里的 file_key
  4. file_key 解密 Payload → 原始数据
```

**加密给多人时 Header 有多条记录，Payload 只有一份：**

```
age-encryption.org/v1
-> X25519 <ephemeral_pub_for_alice>
<file_key encrypted for alice>
-> X25519 <ephemeral_pub_for_bob>
<file_key encrypted for bob>
---
[payload 只加密一次，两人用各自私钥解密出同一个 file_key]
```

加密给 10 个人，文件大小几乎不变，因为 payload 只有一份。

---

### 5.4 SOPS — Secrets 文件版本控制加密

Mozilla 开源，专为 GitOps 设计，加密 YAML/JSON 里的敏感字段，其余字段保持明文可 review。

```bash
# 安装
brew install sops

# 配置（.sops.yaml 放在项目根目录）
cat > .sops.yaml << 'EOF'
creation_rules:
  - path_regex: .*secrets.*\.yaml$
    age: age1ql3z7hjy54pw3pywj3gt303988jnsvaa2...
  - path_regex: .*\.prod\.yaml$
    kms: arn:aws:kms:us-east-1:123456789:key/abc123
EOF

# 加密 secrets 文件（只加密 value，key 保持明文）
sops -e secrets.yaml > secrets.enc.yaml

# 原始 YAML：
# db_password: mysupersecret
# api_key: sk-abc123

# 加密后：
# db_password: ENC[AES256_GCM,data:abc...,tag:xyz...,type:str]
# api_key: ENC[AES256_GCM,data:def...,tag:uvw...,type:str]

# 直接编辑加密文件（自动解密→编辑→重新加密）
sops secrets.enc.yaml

# 解密到标准输出
sops -d secrets.enc.yaml

# 取单个值
sops -d --extract '["db_password"]' secrets.enc.yaml

# 在 CI/CD 中使用
export DB_PASSWORD=$(sops -d --extract '["db_password"]' secrets.enc.yaml)
```

---

### 5.5 HashiCorp Vault — 生产级 Secrets 管理

企业级 Secrets 管理平台，支持动态密钥、租约、审计日志。

#### 先理解 Vault 解决什么问题

传统运维的痛点：

```
数据库密码写在配置文件里
    ↓
配置文件提交到 Git
    ↓
所有有 Git 权限的人都能看到密码
    ↓
密码从来不换（换了要改 N 个地方）
    ↓
某天员工离职，密码还在他脑子里
```

Vault 来解决这整条链路。核心三个能力：

---

**① 动态密钥（Dynamic Secrets）**

静态密钥：你手动创建数据库账号，密码写死，所有应用共用。

动态密钥：应用需要连数据库时，临时向 Vault 申请，Vault 实时去数据库创建专属账号，用完自动删除。

```
传统模式：
  所有服务  ──────────────────────▶  db_user/password123（永久存在）

Vault 动态模式：
  service-A 启动  ──▶  Vault  ──▶  创建 v-serviceA-abc / 随机密码（1小时有效）
  service-B 启动  ──▶  Vault  ──▶  创建 v-serviceB-xyz / 随机密码（1小时有效）
                                         ↓ 1小时后
                                    自动 DROP USER，账号消失
```

好处：密码泄露了？1小时后自动失效；每个服务有独立账号，出问题能精确定位；不存在"密码从来不换"的问题。类比：就像 AWS IAM Role 临时凭证（`aws sts assume-role`），而不是永久 Access Key。

---

**② 租约（Lease）**

动态密钥有生命周期，类似**借东西要约定归还时间**：

```
service-A 申请数据库密码，Vault 返回：
  username:       v-serviceA-abc123
  password:       Xk9#mP2...
  lease_id:       database/creds/readonly/abc123   ← 租约ID
  lease_duration: 3600s                            ← 租期1小时
  renewable:      true                             ← 可续租
```

```bash
# 服务还在跑，主动续租（不需要重启服务重新申请）
vault lease renew database/creds/readonly/abc123

# 服务下线，主动释放（立刻删账号，不等到期）
vault lease revoke database/creds/readonly/abc123

# 紧急情况：撤销某服务所有租约（所有密码立即失效）
vault lease revoke -prefix database/creds/readonly/
```

类比：就像 K8s Pod 的 `terminationGracePeriodSeconds`，有生命周期，到期自动回收，也可以提前手动终止。

---

**③ 审计日志（Audit Log）**

Vault 记录每一次密钥的申请、读取、撤销操作，全部留痕：

```json
{
  "time": "2026-02-20T20:00:01Z",
  "auth": {
    "policies": ["service-a-policy"],
    "metadata": {
      "role": "service-a",
      "pod": "api-server-7d9f-xk2p"    ← 哪个 Pod 申请的
    }
  },
  "request": {
    "path": "database/creds/readonly"  ← 申请了什么
  },
  "response": {
    "data": {
      "username": "hmac-sha256:...",   ← 敏感字段自动 HMAC，日志里看不到明文
      "password": "hmac-sha256:..."
    }
  }
}
```

注意密码在日志里是 HMAC 后的值，不是明文——可以对比"是否是同一个值"，但无法反推原始密码。

能回答的问题：谁在什么时间申请了生产数据库密码 ✅、某次泄露哪个服务的凭证被用了 ✅、密码明文是什么 ❌。

---

#### 整体运作模型

```
┌──────────────────────────────────────────────────┐
│                  HashiCorp Vault                  │
│                                                   │
│  ┌──────────┐   ┌──────────────┐  ┌───────────┐  │
│  │  KV存储   │   │  动态密钥引擎  │  │  审计日志  │  │
│  │ 静态配置  │   │  DB/AWS/SSH  │  │  全操作留痕 │  │
│  └──────────┘   └──────────────┘  └───────────┘  │
└────────────────────────┬─────────────────────────┘
                         │ 统一 API
              ┌──────────▼──────────┐
              │   你的服务 / CI/CD   │
              │  启动时去 Vault 拿凭证 │
              │  用完到期自动失效      │
              └─────────────────────┘
```

#### 常用命令

```bash
# 启动开发模式（本地测试）
vault server -dev
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='root'

# ── KV Secrets ──────────────────────────────────
vault kv put secret/myapp db_password="s3cr3t" api_key="sk-abc123"
vault kv get secret/myapp
vault kv get -field=db_password secret/myapp
vault kv get -format=json secret/myapp | jq -r '.data.data.db_password'

# ── 动态数据库密码 ──────────────────────────────
vault secrets enable database
vault write database/config/mydb \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@localhost/mydb" \
  allowed_roles="readonly" \
  username="vaultadmin" password="adminpw"

vault write database/roles/readonly \
  db_name=mydb \
  creation_statements="CREATE ROLE '{{name}}' WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO '{{name}}';" \
  default_ttl="1h" max_ttl="24h"

# 每次调用生成新临时账号，1小时后自动 DROP
vault read database/creds/readonly

# ── Transit（加密即服务）─────────────────────────
# 不存数据，只提供加解密 API（应用不持有密钥）
vault secrets enable transit
vault write -f transit/keys/myapp-key

vault write transit/encrypt/myapp-key \
  plaintext=$(echo -n "mysecret" | base64)
# 返回: ciphertext: vault:v1:abc123...

vault write transit/decrypt/myapp-key \
  ciphertext="vault:v1:abc123..." \
  | jq -r '.data.plaintext' | base64 -d
```

---

### 5.6 ssh-keygen — SSH 密钥管理

```bash
# 生成 Ed25519 密钥（现代首选，比 RSA 更短更快）
ssh-keygen -t ed25519 -C "your@email.com" -f ~/.ssh/id_ed25519

# 生成 RSA 4096（兼容老系统）
ssh-keygen -t rsa -b 4096 -C "your@email.com"

# 查看公钥指纹
ssh-keygen -lf ~/.ssh/id_ed25519.pub
ssh-keygen -lf ~/.ssh/id_ed25519.pub -E sha256   # SHA256 格式

# 从私钥提取公钥
ssh-keygen -y -f ~/.ssh/id_ed25519 > ~/.ssh/id_ed25519.pub

# 修改密钥密码
ssh-keygen -p -f ~/.ssh/id_ed25519

# 证书签名（堡垒机场景，CA 签发 SSH 证书）
# 1. 创建 CA 密钥
ssh-keygen -t ed25519 -f ssh_ca -C "SSH CA"

# 2. CA 签发用户证书（有效期 1 天）
ssh-keygen -s ssh_ca -I "user_alice" \
  -n alice,ubuntu \
  -V +1d \
  ~/.ssh/id_ed25519.pub
# 生成 id_ed25519-cert.pub

# 3. 服务端配置信任 CA（不需要逐台分发公钥）
echo "TrustedUserCAKeys /etc/ssh/ssh_ca.pub" >> /etc/ssh/sshd_config
```

---

### 5.7 cfssl — CloudFlare 的 PKI 工具链

适合搭建内部 CA，管理 mTLS 证书（服务网格、内部服务间认证）。

```bash
# 安装
go install github.com/cloudflare/cfssl/cmd/...@latest

# CA 配置文件
cat > ca-config.json << 'EOF'
{
  "signing": {
    "default": { "expiry": "8760h" },
    "profiles": {
      "server": {
        "usages": ["signing","key encipherment","server auth"],
        "expiry": "8760h"
      },
      "client": {
        "usages": ["signing","key encipherment","client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

# 生成根 CA
cat > ca-csr.json << 'EOF'
{"CN":"My Internal CA","key":{"algo":"ecdsa","size":256},"names":[{"O":"MyOrg"}]}
EOF
cfssl gencert -initca ca-csr.json | cfssljson -bare ca

# 签发服务证书
cat > server-csr.json << 'EOF'
{"CN":"my-service","hosts":["my-service","my-service.default.svc","localhost"],"key":{"algo":"ecdsa","size":256}}
EOF
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem \
  -config=ca-config.json -profile=server \
  server-csr.json | cfssljson -bare server

# 生成的文件：server.pem（证书）server-key.pem（私钥）
```

---

### 5.8 WireGuard — 基于 ChaCha20 的现代 VPN

```bash
# 安装
apt install wireguard

# 生成密钥对
wg genkey | tee privatekey | wg pubkey > publickey
wg genpsk > presharedkey  # 额外的对称预共享密钥（可选）

# 服务端配置 /etc/wireguard/wg0.conf
cat > /etc/wireguard/wg0.conf << 'EOF'
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = <server_private_key>
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <client_public_key>
PresharedKey = <preshared_key>
AllowedIPs = 10.0.0.2/32
EOF

# 启动
wg-quick up wg0
systemctl enable wg-quick@wg0

# 查看状态
wg show

# WireGuard 加密原理速览：
# 密钥交换: Curve25519 (ECDH)
# 认证加密: ChaCha20-Poly1305
# 哈希/MAC:  BLAKE2s / HMAC-SHA256
# 所有实现只有 ~4000 行代码，可审计
```

---

### 5.9 Python — 工程代码中的加密实践

```python
# pip install cryptography argon2-cffi

from cryptography.hazmat.primitives.ciphers.aead import AESGCM
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PrivateKey
from cryptography.hazmat.primitives.serialization import Encoding, PublicFormat
import os
import base64

# ── AES-256-GCM 对称加密 ────────────────────────────────
def encrypt(plaintext: bytes, key: bytes) -> tuple[bytes, bytes]:
    """返回 (nonce, ciphertext)，nonce 每次必须唯一"""
    aesgcm = AESGCM(key)
    nonce = os.urandom(12)   # 96-bit nonce for GCM
    ciphertext = aesgcm.encrypt(nonce, plaintext, None)
    return nonce, ciphertext

def decrypt(nonce: bytes, ciphertext: bytes, key: bytes) -> bytes:
    aesgcm = AESGCM(key)
    return aesgcm.decrypt(nonce, ciphertext, None)

key = AESGCM.generate_key(bit_length=256)
nonce, ct = encrypt(b"hello secret", key)
print(decrypt(nonce, ct, key))  # b'hello secret'

# ── Ed25519 签名 ────────────────────────────────────────
private_key = Ed25519PrivateKey.generate()
public_key = private_key.public_key()

message = b"deploy version 1.2.3"
signature = private_key.sign(message)

# 验证（签名不对会抛 InvalidSignature 异常）
public_key.verify(signature, message)
print("签名验证通过")

# ── Argon2 密码哈希 ─────────────────────────────────────
from argon2 import PasswordHasher

ph = PasswordHasher(
    time_cost=3,        # 迭代次数
    memory_cost=65536,  # 64MB 内存
    parallelism=2,      # 并行度
)

# 注册时
hashed = ph.hash("user_password_123")
print(hashed)  # $argon2id$v=19$m=65536,t=3,p=2$...

# 登录验证时
try:
    ph.verify(hashed, "user_password_123")
    print("密码正确")
    if ph.check_needs_rehash(hashed):
        hashed = ph.hash("user_password_123")  # 更新 hash 参数
except Exception:
    print("密码错误")
```

---

### 5.10 Go — 工程代码中的加密实践

```go
package main

import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/ed25519"
    "crypto/rand"
    "crypto/sha256"
    "encoding/hex"
    "fmt"
    "io"
    "golang.org/x/crypto/argon2"
)

// ── AES-256-GCM 加密 ────────────────────────────────────
func encrypt(plaintext, key []byte) ([]byte, error) {
    block, err := aes.NewCipher(key)
    if err != nil {
        return nil, err
    }
    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return nil, err
    }
    nonce := make([]byte, gcm.NonceSize())
    if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
        return nil, err
    }
    // nonce 前置在密文里，解密时取出
    return gcm.Seal(nonce, nonce, plaintext, nil), nil
}

func decrypt(ciphertext, key []byte) ([]byte, error) {
    block, _ := aes.NewCipher(key)
    gcm, _ := cipher.NewGCM(block)
    nonceSize := gcm.NonceSize()
    nonce, ct := ciphertext[:nonceSize], ciphertext[nonceSize:]
    return gcm.Open(nil, nonce, ct, nil)
}

// ── Ed25519 签名 ────────────────────────────────────────
func signAndVerify() {
    pub, priv, _ := ed25519.GenerateKey(rand.Reader)
    message := []byte("deploy v1.2.3")

    sig := ed25519.Sign(priv, message)
    if ed25519.Verify(pub, message, sig) {
        fmt.Println("签名验证通过")
    }
}

// ── SHA-256 哈希 ─────────────────────────────────────────
func hashFile(data []byte) string {
    h := sha256.Sum256(data)
    return hex.EncodeToString(h[:])
}

// ── Argon2id 密码哈希 ────────────────────────────────────
func hashPassword(password string) []byte {
    salt := make([]byte, 16)
    rand.Read(salt)
    // time=3, memory=64MB, threads=2, keyLen=32
    return argon2.IDKey([]byte(password), salt, 3, 64*1024, 2, 32)
}

func main() {
    key := make([]byte, 32)
    rand.Read(key)

    ct, _ := encrypt([]byte("hello secret"), key)
    pt, _ := decrypt(ct, key)
    fmt.Println(string(pt)) // hello secret

    signAndVerify()
    fmt.Println(hashFile([]byte("hello")))
}
```

---

## 六、使用场景总结

```
┌─────────────────────────────────────────────────────────────────────┐
│                        加密场景速查表                                 │
├──────────────────────┬──────────────────────┬───────────────────────┤
│  场景                 │  推荐算法/工具          │  备注                 │
├──────────────────────┼──────────────────────┼───────────────────────┤
│  HTTPS/TLS           │  TLS 1.3             │  AES-GCM + ECDHE      │
│                      │  (自动处理)            │  禁用 TLS 1.0/1.1     │
├──────────────────────┼──────────────────────┼───────────────────────┤
│  SSH 认证             │  Ed25519             │  比 RSA 更短更快       │
│                      │  ssh-keygen          │  堡垒机用证书模式       │
├──────────────────────┼──────────────────────┼───────────────────────┤
│  VPN                 │  WireGuard           │  ChaCha20-Poly1305    │
│                      │                      │  代码量少，可审计       │
├──────────────────────┼──────────────────────┼───────────────────────┤
│  文件加密传输          │  age / GPG           │  age 更简单            │
│                      │                      │  GPG 兼容性更好         │
├──────────────────────┼──────────────────────┼───────────────────────┤
│  Git 中的 Secrets    │  SOPS + age/KMS      │  加密 value，key 可见   │
│                      │                      │  适合 GitOps 工作流     │
├──────────────────────┼──────────────────────┼───────────────────────┤
│  Secrets 集中管理     │  HashiCorp Vault     │  动态密钥、租约、审计    │
│                      │                      │  生产首选               │
├──────────────────────┼──────────────────────┼───────────────────────┤
│  数据库/磁盘加密       │  AES-256-GCM         │  LUKS (磁盘)           │
│                      │  LUKS / TDE          │  PostgreSQL TDE (DB)   │
├──────────────────────┼──────────────────────┼───────────────────────┤
│  用户密码存储          │  Argon2id            │  绝对不能用 SHA/MD5     │
│                      │  bcrypt (备选)        │  cost factor 按硬件调   │
├──────────────────────┼──────────────────────┼───────────────────────┤
│  文件完整性校验        │  SHA-256             │  sha256sum 命令        │
│                      │                      │  非安全场景 MD5 也行     │
├──────────────────────┼──────────────────────┼───────────────────────┤
│  代码/制品签名         │  GPG / Sigstore      │  git commit -S         │
│                      │                      │  cosign (容器镜像签名)   │
├──────────────────────┼──────────────────────┼───────────────────────┤
│  mTLS 内部服务认证    │  ECC P-256           │  cfssl / cert-manager  │
│                      │  cfssl / Vault PKI   │  服务网格双向认证        │
├──────────────────────┼──────────────────────┼───────────────────────┤
│  代码中加解密          │  AES-256-GCM         │  Python: cryptography  │
│                      │  Ed25519             │  Go: crypto/aes        │
│                      │  Argon2id            │  Node: crypto 内置      │
├──────────────────────┼──────────────────────┼───────────────────────┤
│  JWT Token           │  ES256 (ECC)         │  RS256 (RSA) 兼容备选   │
│                      │                      │  禁用 alg: none         │
└──────────────────────┴──────────────────────┴───────────────────────┘
```

---

## 七、常见错误 / 安全反模式

```bash
# ❌ 使用弱算法
openssl enc -des ...          # DES 已破解
openssl enc -aes-256-cbc ...  # CBC 模式不带完整性校验

# ❌ 硬编码密钥
API_KEY="sk-abc123"           # 提交到 Git 是灾难

# ❌ 自己实现加密算法
def my_encrypt(data): ...     # 永远不要这样做

# ❌ nonce/IV 重用（AES-GCM 的致命错误）
nonce = b'\x00' * 12          # 固定 nonce = 完全破解 GCM

# ❌ 用 SHA-256 直接存密码
hashlib.sha256(password).hexdigest()  # 彩虹表秒破

# ✅ 正确做法清单
# 1. 用成熟库，不造轮子 (cryptography, libsodium)
# 2. 密钥存 Vault / KMS，不写代码里
# 3. nonce/salt 每次随机生成
# 4. 密码用 Argon2id
# 5. 传输用 TLS 1.3，禁用老版本
# 6. 定期轮换密钥（Vault 支持自动轮换）
```

---

## 八、后量子密码（了解趋势）

量子计算机通过 **Shor 算法**可以破解现有的 RSA 和 ECC。NIST 于 2024 年正式发布后量子标准：

| 标准 | 原名 | 用途 |
|------|------|------|
| ML-KEM | CRYSTALS-KYBER | 密钥封装/交换 |
| ML-DSA | CRYSTALS-DILITHIUM | 数字签名 |
| SLH-DSA | SPHINCS+ | 数字签名（备选） |

**当前状态：** 过渡期，生产环境暂不需要行动。CloudFlare、Google 已在 TLS 里测试**混合模式**（ECC + 后量子同时跑），两者都安全才算安全。

---

## TL;DR — 一句话选型原则

> **对称加密用 AES-256-GCM，密钥交换用 ECDHE，签名用 Ed25519，密码存储用 Argon2id，Secrets 管理用 Vault，文件加密用 age，别自己实现算法。**
