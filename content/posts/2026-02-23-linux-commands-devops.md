---
title: "Linux Commands DevOps Engineers Use Every Day"
date: 2026-02-23T10:34:14+08:00
draft: false
author: "Kaka"
tags: ["linux", "devops", "commands", "cheatsheet", "sre"]
categories: ["DevOps"]
description: "A comprehensive reference of essential Linux commands for DevOps/SRE engineers — process debugging, logs, networking, DNS, disk, containers, performance, cron, package management, and more."
toc: true
---

> A battle-tested cheatsheet for daily DevOps/SRE work. Bookmark it, internalize it, go home on time.

---

## Process Debugging

| Command | Description |
|---|---|
| `ps aux` | See every process actually running |
| `ps aux \| grep <name>` | Filter to a specific process |
| `top` | Live CPU + memory usage |
| `htop` | Cleaner visual process monitor |
| `pgrep nginx` | Get PID by process name |
| `lsof -p <PID>` | List all files opened by a process |
| `lsof -i :8080` | Which process is using port 8080 |
| `kill <PID>` | Stop a process gracefully (SIGTERM) |
| `kill -9 <PID>` | Force kill — the last resort (SIGKILL) |
| `pkill -f <name>` | Kill by name pattern |
| `nohup <cmd> &` | Run a command immune to hangups |
| `strace -p <PID>` | Trace system calls of a running process |

---

## Logs & Search

| Command | Description |
|---|---|
| `journalctl -u nginx` | Check specific service logs |
| `journalctl -u nginx --since "1 hour ago"` | Narrow to a time window |
| `journalctl -f` | Follow system journal in real time |
| `tail -f /var/log/syslog` | Watch live system events |
| `tail -n 100 file.log` | Last 100 lines |
| `grep -i "error" file.log` | Case-insensitive search |
| `grep -C 5 "panic" file.log` | Show 5 lines of context around match |
| `grep -rn "TODO" /app/` | Recursive search with line numbers |
| `grep -v "debug" file.log` | Exclude lines matching a pattern |
| `less file.log` | Page through logs without loading all |
| `find / -name "*.log" -mtime -1` | Logs modified in the last 24h |

---

## Text Processing

| Command | Description |
|---|---|
| `awk '{print $1}'` | Extract the first column |
| `awk -F: '{print $1}' /etc/passwd` | Custom delimiter, print first field |
| `awk '/ERROR/ {print NR, $0}' app.log` | Print matching lines with line numbers |
| `awk '{sum+=$3} END {print sum}'` | Sum a numeric column |
| `sed 's/old/new/g'` | Global in-place text substitution |
| `sed -n '10,20p' file` | Print lines 10 to 20 |
| `sed '/^#/d' config.conf` | Delete comment lines |
| `cut -d',' -f1,3 file.csv` | Extract columns from a delimited file |
| `cut -c1-10 file.txt` | Extract first 10 characters per line |
| `sort file.txt` | Sort lines alphabetically |
| `sort -n -k2 file.txt` | Sort numerically by second column |
| `sort -rh \| head -10` | Reverse human-readable sort, top 10 |
| `uniq -c` | Count consecutive duplicate lines |
| `sort \| uniq -c \| sort -rn` | Frequency count of all values |
| `wc -l file.log` | Count lines |
| `wc -w file.txt` | Count words |
| `tr 'a-z' 'A-Z'` | Translate characters (lowercase → upper) |
| `tr -d '\r'` | Strip Windows carriage returns |
| `column -t -s','` | Pretty-print CSV as aligned table |
| `jq '.items[].name'` | Parse and query JSON |
| `jq -r '.[] \| select(.status=="error")'` | Filter JSON array by field value |

---

## Networking

| Command | Description |
|---|---|
| `ss -tulnp` | Check open/listening ports with PID |
| `netstat -anp` | All connections (if ss unavailable) |
| `ping <host>` | Test basic connectivity |
| `traceroute <domain>` | Find where traffic drops |
| `mtr <domain>` | Continuous traceroute — better than traceroute |
| `curl -I localhost:8080` | Check HTTP headers and response code |
| `curl -o /dev/null -sw "%{http_code}" <url>` | Just the HTTP status code |
| `curl -v <url>` | Verbose: show full request + response |
| `wget -q --spider <url>` | Check if a URL is reachable |
| `iptables -L -n` | List current firewall rules |
| `tcpdump -i eth0 port 80` | Capture live packets on a port |
| `tcpdump -i eth0 -w cap.pcap` | Save capture to file for Wireshark |

---

## DNS & Network Diagnostics

| Command | Description |
|---|---|
| `dig <domain>` | Full DNS query with all record details |
| `dig +short <domain>` | Just the resolved IP(s) |
| `dig <domain> MX` | Query MX records |
| `dig <domain> TXT` | Query TXT records (SPF, DKIM, etc.) |
| `dig @8.8.8.8 <domain>` | Query against a specific DNS server |
| `dig -x <ip>` | Reverse DNS lookup (PTR record) |
| `nslookup <domain>` | Quick DNS lookup (interactive or one-shot) |
| `host <domain>` | Simple forward/reverse DNS lookup |
| `resolvectl status` | Show DNS servers in use (systemd-resolved) |
| `cat /etc/resolv.conf` | Check configured DNS resolvers |
| `nmap -p 22,80,443 <host>` | Scan specific ports |
| `nmap -sV <host>` | Detect service versions on open ports |
| `nmap -sn 192.168.1.0/24` | Ping scan a subnet (host discovery) |
| `whois <domain>` | Domain registration and ownership info |
| `openssl s_client -connect <host>:443` | Inspect TLS certificate details |

---

## SSH & File Transfer

| Command | Description |
|---|---|
| `ssh user@host` | Connect to a remote server |
| `ssh -i ~/.ssh/id_ed25519 user@host` | Connect with a specific key |
| `ssh -p 2222 user@host` | Connect on a non-standard port |
| `ssh -L 8080:localhost:8080 user@host` | Local port forwarding |
| `ssh -R 9090:localhost:9090 user@host` | Remote port forwarding |
| `ssh -N -D 1080 user@host` | Dynamic SOCKS proxy |
| `ssh -J bastion user@target` | Jump host / bastion proxy |
| `scp file.txt user@host:/remote/path/` | Copy a file to remote |
| `scp -r dir/ user@host:/remote/path/` | Copy a directory to remote |
| `rsync -avz /src/ user@host:/dst/` | Sync files (incremental + compressed) |
| `rsync -avz --delete /src/ /dst/` | Sync and remove files not in source |
| `rsync --dry-run -avz /src/ /dst/` | Preview what rsync would do |
| `ssh-keygen -t ed25519` | Generate a modern SSH key pair |
| `ssh-copy-id user@host` | Install your public key on remote host |
| `ssh-add ~/.ssh/id_ed25519` | Add key to SSH agent |

---

## Disk & Permissions

| Command | Description |
|---|---|
| `df -h` | Overall disk usage |
| `df -ih` | Inode usage — often the hidden culprit! |
| `du -sh *` | Size of each item in current directory |
| `du -sh /* 2>/dev/null \| sort -rh \| head -10` | Top 10 largest directories |
| `lsblk` | View block devices and mount structure |
| `mount` | Show all mount points |
| `lsof +D /path` | Find processes holding a directory open |
| `chmod 600 <file>` | Secure private keys (`rw-------`) |
| `chmod 755 <dir>` | Standard directory permissions |
| `chown user:group <file>` | Fix file ownership |
| `chown -R user:group <dir>` | Recursive ownership fix |
| `ls -lah` | Detailed listing including hidden files |
| `stat <file>` | Full metadata: size, permissions, timestamps |
| `tar -czf backup.tar.gz /path/` | Create a compressed archive |
| `tar -xzf backup.tar.gz` | Extract an archive |
| `tar -tzf backup.tar.gz` | List archive contents without extracting |

---

## Performance Analysis

| Command | Description |
|---|---|
| `free -h` | Memory usage at a glance |
| `vmstat 1 5` | CPU, memory, I/O stats — 5 snapshots, 1s apart |
| `iostat -xz 1` | Per-device disk I/O utilization |
| `sar -u 1 5` | Historical and live CPU usage |
| `sar -r` | Memory usage history |
| `sar -d` | Disk activity history |
| `uptime` | Load averages for 1/5/15 minutes |
| `dmesg -T \| tail -20` | Recent kernel messages with timestamps |
| `perf top` | Live kernel + userspace CPU profiling |
| `nice -n 10 <cmd>` | Run a command with lower CPU priority |
| `renice -n 5 -p <PID>` | Change the priority of a running process |

---

## Docker & Containers

| Command | Description |
|---|---|
| `docker ps` | List running containers |
| `docker ps -a` | All containers including stopped ones |
| `docker logs -f <container>` | Follow container logs |
| `docker logs --tail 100 <container>` | Last 100 lines of container logs |
| `docker exec -it <container> bash` | Shell into a running container |
| `docker inspect <container>` | Full container config/state as JSON |
| `docker inspect -f '{{.State.Status}}' <c>` | Extract a specific field from inspect |
| `docker stats` | Live CPU/memory for all containers |
| `docker top <container>` | Processes running inside a container |
| `docker image ls` | List local images |
| `docker image prune -af` | Remove unused images |
| `docker system df` | Docker disk usage overview |
| `docker system prune -af` | Remove all unused resources |
| `docker network ls` | List Docker networks |
| `docker network inspect <net>` | Inspect a network (IPs, containers) |
| `docker volume ls` | List Docker volumes |
| `docker cp <container>:/path ./local` | Copy files out of a container |

---

## Service Management

| Command | Description |
|---|---|
| `systemctl status <service>` | Health check with recent log tail |
| `systemctl restart <service>` | Restart a service |
| `systemctl reload <service>` | Reload config without full restart |
| `systemctl enable <service>` | Auto-start on boot |
| `systemctl disable <service>` | Remove from boot sequence |
| `systemctl list-units --failed` | Show all failed units |
| `systemctl list-timers` | List all systemd timers |
| `uname -a` | Kernel version and architecture |
| `hostnamectl` | Hostname, OS, and kernel info |
| `timedatectl` | System time and timezone |

---

## Scheduled Tasks (Cron)

| Command | Description |
|---|---|
| `crontab -l` | List current user's cron jobs |
| `crontab -e` | Edit current user's cron jobs |
| `crontab -u <user> -l` | List another user's cron jobs |
| `cat /etc/crontab` | System-wide crontab |
| `ls /etc/cron.d/` | Drop-in cron job files |
| `ls /etc/cron.daily/` | Scripts run daily by cron |
| `systemctl list-timers --all` | List all systemd timers (modern cron) |
| `systemctl status <name>.timer` | Check a specific timer |

**Cron expression cheatsheet:**

```
# ┌───── minute (0–59)
# │ ┌───── hour (0–23)
# │ │ ┌───── day of month (1–31)
# │ │ │ ┌───── month (1–12)
# │ │ │ │ ┌───── day of week (0–7, 0=Sun)
# │ │ │ │ │
# * * * * *  command

0 2 * * *          # Every day at 02:00
*/5 * * * *        # Every 5 minutes
0 9 * * 1          # Every Monday at 09:00
0 0 1 * *          # First day of every month at midnight
@reboot            # Once at system startup
```

---

## Package Management

### Debian / Ubuntu (apt)

| Command | Description |
|---|---|
| `apt update` | Refresh package index |
| `apt upgrade` | Upgrade all installed packages |
| `apt install <pkg>` | Install a package |
| `apt remove <pkg>` | Remove a package (keep config) |
| `apt purge <pkg>` | Remove package + config files |
| `apt autoremove` | Remove orphaned dependencies |
| `apt search <keyword>` | Search available packages |
| `apt show <pkg>` | Show package details and dependencies |
| `dpkg -l \| grep <name>` | Check if a package is installed |
| `dpkg -L <pkg>` | List files installed by a package |

### RHEL / CentOS / Amazon Linux (yum / dnf)

| Command | Description |
|---|---|
| `yum update` / `dnf update` | Update all packages |
| `yum install <pkg>` | Install a package |
| `yum remove <pkg>` | Remove a package |
| `yum search <keyword>` | Search available packages |
| `yum info <pkg>` | Package details |
| `rpm -qa \| grep <name>` | Check installed RPM packages |
| `rpm -ql <pkg>` | List files in an installed package |

### macOS (Homebrew)

| Command | Description |
|---|---|
| `brew update` | Update Homebrew itself |
| `brew upgrade` | Upgrade all installed formulae |
| `brew install <pkg>` | Install a package |
| `brew uninstall <pkg>` | Uninstall a package |
| `brew list` | List installed packages |
| `brew search <keyword>` | Search available packages |
| `brew info <pkg>` | Package details |
| `brew doctor` | Diagnose Homebrew issues |
| `brew cleanup` | Remove old versions and cache |

---

## Environment Variables & Shell Productivity

| Command | Description |
|---|---|
| `env` | Print all environment variables |
| `export VAR=value` | Set an env variable for current session |
| `unset VAR` | Remove an env variable |
| `printenv PATH` | Print a specific variable |
| `source ~/.bashrc` | Reload shell config without restarting |
| `which <cmd>` | Find where a command binary lives |
| `type <cmd>` | Is it a binary, alias, or shell builtin? |
| `alias ll='ls -lah'` | Create a shortcut |
| `history \| grep <keyword>` | Search command history |
| `Ctrl+R` | Reverse-search through history interactively |
| `!!` | Repeat the last command |
| `sudo !!` | Repeat last command with sudo |
| `!$` | Reuse last argument from previous command |
| `!^` | Reuse first argument from previous command |
| `cd -` | Jump back to previous directory |
| `watch -n 2 df -h` | Repeat a command every 2 seconds |
| `time <cmd>` | Measure command execution time |
| `xargs` | Build and execute commands from stdin |
| `xargs -P 4` | Run up to 4 processes in parallel |
| `tee output.log` | Write to file AND keep stdout visible |
| `command1 && command2` | Run command2 only if command1 succeeds |
| `command1 \|\| command2` | Run command2 only if command1 fails |
| `set -e` | Exit script on first error |
| `set -x` | Print each command before executing (debug) |

---

## Quick Incident Runbook

When something breaks in production, run through this sequence:

```bash
# 1. Is the service up?
systemctl status <service>

# 2. Any recent errors in logs?
journalctl -u <service> --since "10 minutes ago" | grep -i "error\|warn\|fatal"

# 3. Is the port open and listening?
ss -tulnp | grep <port>

# 4. Is the app actually responding?
curl -I localhost:<port>

# 5. CPU / memory pressure?
top -bn1 | head -20
free -h

# 6. Disk full?
df -h
df -ih   # also check inodes!
du -sh /var/log/* | sort -rh | head -5

# 7. Network and DNS OK?
ping -c 3 <upstream_host>
dig +short <domain>

# 8. Container issues? (if applicable)
docker ps -a
docker logs --tail 100 <container>
docker stats --no-stream
```

---

*Last updated: February 2026 · Submit PRs with your own additions.*
