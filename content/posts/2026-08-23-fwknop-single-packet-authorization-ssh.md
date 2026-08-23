---
title: "让公网看不见 SSH 22 端口：fwknop 与 Single Packet Authorization 实战"
date: 2026-08-23T09:00:00+08:00
draft: false
tags: ["SSH", "fwknop", "Single Packet Authorization", "UFW", "Ansible", "Security"]
categories: ["Linux", "网络", "DevOps"]
author: "Kaka"
description: "在 Ubuntu 22.04 上用 fwknop 和 Single Packet Authorization 隐藏 SSH：包含威胁模型、UFW 配置、Mac client、Ansible 灰度发布、验证、回滚与故障排查。"
---

## 引言

SSH hardening 通常停在禁用 password login、禁止 root login、只允许 public key，再加一层 fail2ban。这些措施都应该保留，但它们没有改变一个事实：只要 TCP/22 对公网开放，scanner 就能连接到 OpenSSH，读取 banner，并在新漏洞出现时直接触达 daemon。

Single Packet Authorization（SPA）的思路更激进：SSH 仍在服务器本地监听 22 端口，但 host firewall 默认丢弃公网连接；只有 client 先发出一个通过 encryption 和 HMAC 验证的 SPA packet，`fwknopd` 才为这个 source IP 临时插入 allow rule。超时后 rule 自动删除，已经建立的 SSH session 通常由 conntrack 继续维持。

先给结论：SPA 不是 SSH authentication 的替代品，也不是万能的 Zero Trust。它是一层 service concealment 和 pre-authentication gate，适合公网小规模主机、homelab、break-glass access，或者作为 VPN/Tailscale 之外的独立路径。对企业生产环境，优先级仍应是 private network、VPN/overlay network、identity-aware access 和可靠的 out-of-band console。

> 本文基于 Michele Bologna 的 [Why I close SSH port 22 entirely (and what I use instead)](https://www.michelebologna.net/2026/ssh-port-22-fwknop-single-packet-authorization/) 翻译并重构，原文采用 [CC BY 3.0](https://creativecommons.org/licenses/by/3.0/)；本文增加了 Ubuntu 22.04 实测、cloud firewall 边界、两阶段 rollout、Ansible、key rotation、监控和故障排查，并修正了部分实现细节。

## SPA 不是传统 Port Knocking

传统 Port Knocking 依赖一串固定的端口访问顺序：例如依次连接 7000、8000、9000，server 看到正确序列后开放 22。序列本身通常是明文，可被旁路观察和 replay，本质上更接近 shared secret + obscurity。

`fwknop` 只发送一个 SPA packet。典型 symmetric 模式会先加密 payload，再对 ciphertext 做 HMAC，也就是 encrypt-then-authenticate。payload 中包含随机数据、timestamp、请求开放的 protocol/port 和 client IP。`fwknopd` 同时检查 packet age，并持久化已接受 packet 的 digest；被捕获的旧 packet 即使原样重发，也会因为超时或 digest replay check 被拒绝。这里不是所谓“单次 sequence counter”，而是 freshness check 与 replay cache 共同工作。

```text
未授权扫描器
    │ TCP SYN :22
    ▼
[ Host firewall: DROP ] ──────> 无 banner、无 RST、Nmap 显示 filtered

已授权运维终端
    │ encrypted + HMAC SPA packet / UDP 62201
    ▼
[ fwknopd 验证身份、时间与 replay cache ]
    │
    ├─ invalid ───────────────> 静默丢弃
    │
    └─ valid ─────────────────> 仅为 source IP 开放 TCP/22，持续 60 秒
                                      │
                                      ▼
                                  SSH 正常认证
```

关键边界是：SPA 降低了 OpenSSH 对 Internet 的可达性，但 `fwknopd`、libpcap、Netfilter 和 key distribution 成为新的 trusted computing base。OpenSSH 的 key-only authentication、patching、least privilege 和 audit 一项都不能省。

## 上线前先解决“怎么救回来”

不要在唯一一条 SSH session 中直接删除 22 端口规则。最低限度需要保持当前 session 不退出，并准备一个独立的恢复通道：cloud serial console、provider web console、Tailscale/VPN，或者同网段的 management host。恢复路径必须和 fwknop 配置独立，否则它不叫 fallback。

上线前还要确认以下条件：client 与 server 都有可靠的 time synchronization；明确公网 ingress interface；确认 client 是否位于 NAT/CGNAT 后；检查 cloud Security Group/NACL 与 host firewall 两层策略；同时验证 IPv4 和 IPv6。只挡住 IPv4 的 TCP/22，而 IPv6 仍公开，是非常常见的“看起来已经隐藏”。

在 AWS、GCP、Azure 等 cloud 环境，SPA packet 必须先通过 provider firewall 才能到达 VM。若 client egress IP 不固定，可以在 Security Group 放行 UDP/62201 到 instance，再由 host 上的 `fwknopd` 做 cryptographic authorization；TCP/22 则不再对公网放行。Security Group 如果先把 UDP/62201 丢掉，VM 上的 libpcap 什么也看不到。

## Ubuntu 22.04 Server 配置

本文以 Ubuntu 22.04、UFW、默认 `iptables-nft` compatibility backend 为例。Jammy repository 提供的是 `fwknop-server 2.6.10-13build1`；部署前应通过 `apt-cache policy fwknop-server` 核对当前 mirror 和 security policy，不要把 distro package version 当成永远不变的事实。

先收集实际状态，不要猜 interface 名称：

```bash
sudo apt update
sudo apt install fwknop-server

ip -brief link
timedatectl status
sudo ss -lntp 'sport = :22'
sudo ufw status verbose
```

假设公网 interface 是 `ens3`，SPA 使用 UDP/62201，编辑 `/etc/fwknop/fwknopd.conf`，只覆盖真正需要的参数：

```text
PCAP_INTF                   ens3;
PCAP_FILTER                 udp dst port 62201;
ENABLE_PCAP_PROMISC         N;
ENABLE_SPA_PACKET_AGING     Y;
MAX_SPA_PACKET_AGE          120;
ENABLE_DIGEST_PERSISTENCE   Y;
```

默认 capture path 使用 libpcap 被动抓包，不需要在 UFW 中 `allow 62201/udp`。packet 即使随后被 Netfilter 丢弃，`fwknopd` 仍能看到它；普通 allow rule 反而可能让 kernel 对一个没有 socket 监听的 UDP port 返回 ICMP port-unreachable，增加外部可观测性。只有明确启用 `ENABLE_UDP_SERVER`、让 daemon 通过 UDP socket 收包时，才按该 mode 的网络路径重新设计 firewall。

### 生成并管理两把 key

在可信 client 上生成独立的 encryption key 和 HMAC key：

```bash
umask 077
fwknop --key-gen
```

输出中的 `KEY_BASE64` 和 `HMAC_KEY_BASE64` 都是 secret。不要放进 shell history、工单、Chat、Git repository 或普通 dotfiles template；应写入 Ansible Vault、SOPS、password manager 或企业 secret manager，再在部署时渲染。也不要让整个 fleet 共用一组永久 key：更实用的模型是“每个 operator/device 一组 key，只分发到其被授权管理的 host group”，这样丢失一台 laptop 时可以定点 revoke，而不是全网轮换。

server 端 `/etc/fwknop/access.conf` 的最小 hardening stanza 如下：

```text
SOURCE                  ANY
OPEN_PORTS              tcp/22
REQUIRE_SOURCE_ADDRESS  Y
FW_ACCESS_TIMEOUT       60
MAX_FW_TIMEOUT          60
HMAC_DIGEST_TYPE        SHA512
KEY_BASE64              <ENCRYPTION_KEY_BASE64>
HMAC_KEY_BASE64         <HMAC_KEY_BASE64>
```

`OPEN_PORTS` 必须显式限制为 `tcp/22`。如果省略，client 可以在 SPA request 中申请其他 port。`MAX_FW_TIMEOUT` 则防止 client 用 `--fw-timeout` 请求过长的开放窗口。`REQUIRE_SOURCE_ADDRESS Y` 要求 packet 内嵌明确的 allow IP，避免把未经约束的 packet source 直接当成授权目标。

多个 operator/device 使用多个 stanza。key 应彼此独立，文件权限保持为 root-only：

```bash
sudo chown root:root /etc/fwknop/fwknopd.conf /etc/fwknop/access.conf
sudo chmod 600 /etc/fwknop/fwknopd.conf /etc/fwknop/access.conf
sudo fwknopd --exit-parse-config
```

最后一条命令只解析并校验配置，不修改 firewall，适合放进 CI/CD 或 Ansible handler 前置检查。通过后再启动 daemon：

```bash
sudo systemctl enable --now fwknop-server
sudo systemctl is-active fwknop-server
sudo fwknopd --fw-list
```

Jammy 的 native systemd unit 名为 `fwknop-server.service`，直接执行 `/usr/sbin/fwknopd`。网上很多旧教程要求修改 `/etc/default/fwknop-server` 中的 `START_DAEMON="yes"`，但该 native unit 并不读取这个变量；使用 `systemctl enable --now` 即可。

此时先不要删除现有的 public SSH allow rule。server 端准备完成，不等于 end-to-end path 已经验证。

## macOS / Linux Client 配置

macOS 可以直接安装 Homebrew formula；Ubuntu client 使用 distro package：

```bash
# macOS
brew install fwknop

# Ubuntu / Debian
sudo apt install fwknop-client
```

将 client 配置写入 `~/.fwknoprc`。下面的 profile 名为 `prod-bastion`：

```text
[prod-bastion]
SPA_SERVER             203.0.113.10
SPA_SERVER_PORT        62201
SPA_SERVER_PROTO       udp
ALLOW_IP               resolve
ACCESS                 tcp/22
USE_HMAC               Y
HMAC_DIGEST_TYPE       SHA512
KEY_BASE64             <ENCRYPTION_KEY_BASE64>
HMAC_KEY_BASE64        <HMAC_KEY_BASE64>
```

```bash
chmod 600 ~/.fwknoprc
fwknop -n prod-bastion
```

`ALLOW_IP resolve` 适合 NAT 后且 egress IP 动态变化的 laptop，但它依赖 external IP resolution endpoint。production 可以通过 `RESOLVE_URL` 指向自己控制的 HTTPS endpoint；固定办公 egress 则直接配置明确 IP，减少外部依赖。不要为了方便改成 `ALLOW_IP source`，因为它与 server 端的 `REQUIRE_SOURCE_ADDRESS Y` hardening 目标冲突。

SPA 使用 UDP，单个 packet 可能在网络中丢失。重新执行 `fwknop -n prod-bastion` 会生成新的随机 packet，可以安全重试；被抓包后原样 replay 的 packet 才会命中 digest cache。

## 先验证，再关闭公网 SSH

打开第二个 terminal，从真实外网路径执行 SPA 并建立新的 SSH session：

```bash
fwknop -n prod-bastion
ssh ops@203.0.113.10
```

同时在保留的旧 session 或 out-of-band console 中观察：

```bash
sudo journalctl -u fwknop-server -f
sudo fwknopd --fw-list
sudo iptables -S FWKNOP_INPUT
```

`ufw status` 不会展示 `fwknopd` 动态维护的 rule，因为后者直接管理自己的 Netfilter chain。Ubuntu 22.04 的 `iptables` 通常是 nftables compatibility frontend，因此排障时还可以用 `sudo nft list ruleset` 观察最终 rule graph，但不要同时手工修改两套规则来源。

end-to-end 验证通过后，才删除原有的 public SSH rule。此时必须确认 UFW 已启用，`ufw status verbose` 显示 incoming default policy 为 `deny`；否则删除 allow rule 并不会让 22 端口不可达。必须按现有 rule 类型精确删除：

```bash
# 如果原规则是 allow
sudo ufw delete allow 22/tcp

# 如果原规则是 limit
sudo ufw delete limit 22/tcp
```

cloud Security Group 中的 public TCP/22 ingress 也要同步删除。然后从一台没有 SPA key 的外部主机验证：

```bash
nmap -Pn -p 22 203.0.113.10
```

预期结果是 `filtered`，而不是 `open` 或 `closed`。再从授权 client 发送 SPA，确认 60 秒窗口内可以建立新连接；窗口过期后，新连接重新变为不可达。最后补测 IPv6、server reboot、UFW reload、`fwknop-server` restart 和 client 网络切换。

## 不要用固定 sleep 粘合 SSH

原始做法常在 `ProxyCommand` 中写成：

```text
fwknop -n prod-bastion; sleep 2; nc %h %p
```

它能工作，但 `sleep 2` 既不能保证慢路径已经更新 firewall，也会在快路径白等。更稳妥的是使用一个小 wrapper：发送 SPA 后轮询 TCP/22，成功才把 stdio 交给 `nc`；`fwknop` 的 stdout 必须丢弃，避免污染 SSH binary stream。

保存为 `~/.local/bin/ssh-spa-proxy`：

```sh
#!/bin/sh
set -eu

profile=$1
host=$2
port=$3

fwknop -n "$profile" >/dev/null

attempt=0
while [ "$attempt" -lt 10 ]; do
    if nc -z -w 1 "$host" "$port" >/dev/null 2>&1; then
        exec nc "$host" "$port"
    fi
    attempt=$((attempt + 1))
    sleep 0.2
done

echo "SPA accepted path did not expose ${host}:${port}" >&2
exit 1
```

```bash
chmod 700 ~/.local/bin/ssh-spa-proxy
```

再在 SSH client config 中引用：

```sshconfig
Host prod-bastion
    HostName 203.0.113.10
    User ops
    ProxyCommand ~/.local/bin/ssh-spa-proxy prod-bastion %h %p
```

之后仍然是普通体验：

```bash
ssh prod-bastion
scp release.tar.gz prod-bastion:/tmp/
```

## 用 Ansible 做两阶段灰度发布

自动化的关键不是“一个 play 完成所有动作”，而是把 lockout risk 变成显式 state transition。第一阶段只安装、渲染、校验并启动 `fwknopd`，保留 public SSH；从 controller 验证 SPA path 后，第二阶段才启用 `fwknop_lockdown_enabled` 删除原规则。生产 rollout 使用 `serial: 1`，一次只动一台。

核心 tasks 可以写成：

```yaml
- name: Install fwknop server
  ansible.builtin.apt:
    name: fwknop-server
    state: present
    update_cache: true

- name: Deploy fwknop daemon configuration
  ansible.builtin.template:
    src: fwknopd.conf.j2
    dest: /etc/fwknop/fwknopd.conf
    owner: root
    group: root
    mode: "0600"
  notify: Restart fwknop server

- name: Deploy fwknop access policy
  ansible.builtin.template:
    src: access.conf.j2
    dest: /etc/fwknop/access.conf
    owner: root
    group: root
    mode: "0600"
  no_log: true
  diff: false
  notify: Restart fwknop server

- name: Validate fwknop configuration
  ansible.builtin.command: fwknopd --exit-parse-config
  changed_when: false

- name: Enable fwknop server
  ansible.builtin.systemd_service:
    name: fwknop-server
    enabled: true
    state: started

- name: Remove the exact pre-existing public SSH rule
  community.general.ufw:
    rule: "{{ fwknop_existing_ssh_ufw_rule }}"
    port: "{{ fwknop_ssh_port }}"
    proto: tcp
    delete: true
  when: fwknop_lockdown_enabled | bool
```

对应的 default 应让 lockdown 默认关闭：

```yaml
fwknop_pcap_interface: "{{ ansible_default_ipv4.interface }}"
fwknop_spa_port: 62201
fwknop_ssh_port: "22"
fwknop_access_timeout: 60
fwknop_existing_ssh_ufw_rule: limit
fwknop_lockdown_enabled: false
```

`access.conf.j2` 中只引用 encrypted variable，不放明文：

```jinja2
SOURCE                  ANY
OPEN_PORTS              tcp/{{ fwknop_ssh_port }}
REQUIRE_SOURCE_ADDRESS  Y
FW_ACCESS_TIMEOUT       {{ fwknop_access_timeout }}
MAX_FW_TIMEOUT          {{ fwknop_access_timeout }}
HMAC_DIGEST_TYPE        SHA512
KEY_BASE64              {{ vault_fwknop_spa_key }}
HMAC_KEY_BASE64         {{ vault_fwknop_hmac_key }}
```

client 侧的 wrapper 也可以复用于 Ansible SSH transport。要点是 `fwknoprc` profile 的名字应稳定匹配 `ansible_host` 使用的 DNS/IP，且 `ProxyCommand` 必须写 controller 上的绝对路径，不能假设 automation runner 的 home directory：

```yaml
ansible_ssh_common_args: >-
  -o ProxyCommand="/opt/ops/bin/ssh-spa-proxy %h %h %p"
  -o ConnectTimeout=10
```

初次 provisioning 仍走现有 SSH path。只有 server role 已完成、controller 上已经具备对应 key、第二个 SSH connection 验证成功，才把 inventory 中的 `fwknop_lockdown_enabled` 切换为 `true`。UFW reload 可能影响 runtime chain，相关 handler 应在 reload 后 restart `fwknop-server`，并纳入 reboot test。

## 运维故障排查矩阵

| 现象 | 优先检查 | 典型原因 |
|---|---|---|
| `fwknopd` 完全没有收到 packet | `tcpdump`、Security Group、`PCAP_INTF`、`PCAP_FILTER` | cloud firewall 提前丢包、interface/port 配错、酒店网络拦 UDP |
| log 出现 HMAC/decrypt error | 两端 key 与 `HMAC_DIGEST_TYPE` | key 串台、profile 选错、rotation 未完成 |
| packet 被判定为 expired | `timedatectl`、NTP/chrony | client/server clock drift 超过 `MAX_SPA_PACKET_AGE` |
| 已创建 rule，但 SSH 仍 timeout | `fwknopd --fw-list`、allow IP、rule order | NAT 后解析出错误 egress IP、iptables backend/rule ordering 异常 |
| `ufw status` 看不到临时 rule | `fwknopd --fw-list`、`iptables -S FWKNOP_INPUT` | 正常现象；动态 rule 不由 UFW database 管理 |
| reboot 或 UFW reload 后失效 | systemd 状态、最终 Netfilter graph | daemon 未 enable、UFW reload 刷新了 fwknop chain/jump |
| 偶发第一次连接失败 | 重发 SPA、观察 packet loss | UDP 无 delivery guarantee、network path jitter |
| IPv4 已隐藏但 scanner 仍看到 SSH | `nmap -6`、IPv6 firewall | IPv6 ingress 未按同一 policy 管理 |

日常监控至少覆盖 `fwknop-server.service` 存活、config parse、time sync、digest cache 持久化和关键日志异常。可以对 valid SPA acceptance 建 audit event，但不要把 key、完整 packet 或 client secret 写进日志。定期做一次外部 black-box probe，比只看 `systemctl is-active` 更能证明真正的安全状态。

## Key rotation 与撤权

rotation 不要直接覆盖旧 key。先新增一个新 stanza，将新 key 安全下发到目标 client，验证新 profile 后再删除旧 stanza；全程 `serial: 1` 并保留 out-of-band path。完整流程是 expand、verify、contract，而不是一次性 replace。

共享一组 fleet-wide key 虽然最省事，但 blast radius 最大：任意一台 client 泄露就能触发所有 server 的 access window，而且无法区分 operator。更好的 compromise 是 per-device key + host-group distribution；规模更大时，SPA 的 key matrix 和 revoke workflow 会逐渐变成负担，这正是应该转向集中式 VPN、device identity 或 short-lived certificate access 的信号。

还要注意 NAT 的授权粒度。firewall rule 只识别 source public IP；如果多个用户共享同一个 office NAT/CGNAT，那么 60 秒内该出口后的其他设备也能到达 TCP/22。不过它们仍必须通过 SSH authentication，所以 SPA 与 SSH key 是两道独立门，而不是二选一。

## 什么时候不该用 fwknop

如果组织已经有稳定的 Tailscale/WireGuard、bastion、Teleport 或 cloud-native identity-aware access，让 SSH 只绑定 private interface 通常更容易集中审计、撤权和自动化。SPA 更适合“不希望常驻 overlay control plane，但又必须保留公网 emergency path”的场景。

它也不适合把所有复杂度藏起来后无人维护。fwknop upstream 当前最新 release 是 2.6.11，而 Ubuntu 22.04 repository 是带 distro patch 的 2.6.10；release cadence、package ownership、root daemon、libpcap capture、firewall compatibility 和 secret rotation 都应进入 risk register。减少 OpenSSH 暴露面，不代表系统攻击面归零。

## 总结

fwknop 的价值不是把 22 换成一个没人知道的端口，而是把 TCP connection 建立前置于 cryptographic authorization：未持有 key 的 scanner 无法触达 OpenSSH，合法 client 只为自己的 source IP 获得一个短暂窗口。

真正可用于运维的实现必须同时具备五个条件：保留 SSH hardening；有独立 out-of-band path；passive capture 模式下不误开 UDP host firewall；按 operator/device 管理 key；用两阶段 rollout、外部验证和明确回滚避免自锁。少任何一个，SPA 都可能从安全增强变成新的单点故障。

## 参考资料

- [Michele Bologna：Why I close SSH port 22 entirely](https://www.michelebologna.net/2026/ssh-port-22-fwknop-single-packet-authorization/)
- [fwknop upstream repository](https://github.com/mrash/fwknop)
- [fwknop client manual](https://www.cipherdyne.org/fwknop/docs/manpages/fwknop.html)
- [fwknopd server manual](https://www.cipherdyne.org/fwknop/docs/manpages/fwknopd.html)
- [Ubuntu 22.04 fwknopd manpage](https://manpages.ubuntu.com/manpages/jammy/man8/fwknopd.8.html)
- [Ubuntu Server firewall documentation](https://documentation.ubuntu.com/server/how-to/security/firewalls/)
- [Homebrew fwknop formula](https://formulae.brew.sh/formula/fwknop)
