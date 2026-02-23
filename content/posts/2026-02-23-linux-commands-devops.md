---
title: "Linux Commands DevOps Engineers Use Every Day"
date: 2026-02-23T10:34:14+08:00
draft: false
author: "Kaka"
tags: ["linux", "devops", "commands", "cheatsheet"]
categories: ["DevOps"]
description: "A practical reference of essential Linux commands for DevOps engineers, covering process debugging, log analysis, networking, disk management, and service control."
---

> A practical cheatsheet for daily DevOps work — from process inspection to service management.

---

## Process Debugging

| Command | Description |
|---|---|
| `ps aux` | See every process actually running |
| `top` | Live CPU + memory usage |
| `htop` | Cleaner visual process monitor |
| `kill <PID>` | Stop a process gracefully |
| `kill -9 <PID>` | Force kill (the last resort) |

---

## Logs & Search

| Command | Description |
|---|---|
| `journalctl -u nginx` | Check specific service logs |
| `tail -f /var/log/syslog` | Watch live system events |
| `grep -i "error" file.log` | Find exact issues fast |
| `less file.log` | View logs without loading the whole file |
| `find / -name "*.log"` | Locate specific log files |
| `awk '{print $1}'` | Extract specific columns |
| `sed 's/old/new/g'` | Modify text on the fly |

---

## Networking

| Command | Description |
|---|---|
| `ss -tulnp` | Check which ports are open and listening |
| `ping <host>` | Test basic connectivity |
| `traceroute <domain>` | Find where traffic stops |
| `curl localhost:8080` | Verify the app is responding |

---

## Disk & Permissions

| Command | Description |
|---|---|
| `df -h` | Check overall disk space |
| `du -sh *` | Find large directories |
| `lsblk` | View mounted disks |
| `mount` | Show mount points |
| `chmod 600 <file>` | Secure private keys (`rw-------`) |
| `chown user:group <file>` | Fix ownership issues |
| `ls -l` | View permissions and bits (r, w, x) |

---

## Service Management

| Command | Description |
|---|---|
| `systemctl status <service>` | The ultimate health check |
| `systemctl restart <service>` | Instant service restart |
| `systemctl enable <service>` | Auto-start on boot |
| `free -h` | Show available memory |
| `uname -a` | System info & kernel version |
| `watch df -h` | Monitor disk space in real-time |
| `history \| grep <keyword>` | Search your command history |

---

*Last updated: {{ .Date | dateFormat "January 2, 2006" }}*
