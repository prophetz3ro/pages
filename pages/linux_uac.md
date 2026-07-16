---
layout: page
title: Linux UAC
permalink: /linux_uac/
---

# Linux Live Response

## Current Users
who
w
last

Purpose:
- Active users
- Recent logins
- Suspicious sessions

## Running Processes
pstree -ap
ps -ef

Look for:
- Unknown binaries
- Processes in /tmp
- Deleted executables

# `ps` Quick Reference

| Command | What it shows |
|---------|---------------|
| `ps` | Your processes attached to the current terminal. |
| `ps -a` | Terminal (TTY) processes for all users (except session leaders). |
| `ps -x` | **All of your processes**, even those without a controlling terminal (`TTY=?`). |
| `ps -A` / `ps -e` | Every process on the system (all users, with or without a terminal). |

```bash
ps -ef
```

- `-e` = all processes
- `-f` = full-format output

```bash
ps -eo pid,ppid,user,tty,stat,cmd
```

- `-e` = all processes
- `-o` = custom output columns


```bash
ps x
```

- All of **your** processes (with and without a terminal).

```bash
ps ax
```

- Essentially all processes.

```bash
ps aux
```

- Essentially all processes with user-oriented output.

## Network Activity
ss -antup
ip addr
ip route

Look for:
- Reverse shells
- Unexpected listening ports
- External connections

## `ip route show`

Displays the routing table (how Linux decides where to send packets).

### Example

```bash
ip route show
```

Example output:

```text
default via 192.168.50.1 dev wlo1 proto dhcp src 192.168.50.200 metric 600
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
192.168.50.0/24 dev wlo1 proto kernel scope link src 192.168.50.200 metric 600
```

### Fields

| Field | Meaning |
|---------|---------|
| `default` | Catch-all route if no other route matches |
| `via` | Next hop (gateway/router) |
| `dev` | Interface used |
| `proto` | How route was created (`kernel`, `dhcp`, `static`, etc.) |
| `src` | Preferred source IP |
| `metric` | Route priority (lower = preferred) |
| `scope link` | Destination is directly reachable |
| `linkdown` | Interface/bridge currently inactive |

### Route Selection Examples

| Destination | Route Used |
|-------------|------------|
| `192.168.50.10` | `192.168.50.0/24` |
| `172.17.0.2` | `172.17.0.0/16` |
| `8.8.8.8` | `default` |

---

## `ip route get`

Shows the exact route Linux would use for a destination.

### Example

```bash
ip route get 8.8.8.8
```

Example output:

```text
8.8.8.8 via 192.168.50.1 dev wlo1 src 192.168.50.200
```

Useful commands:

```bash
ip route get 8.8.8.8
ip route get 1.1.1.1
ip route get 192.168.50.50
ip route get 172.17.0.2
```

Questions answered:

- Which interface will be used?
- Which source IP will be selected?
- Will a gateway be used?

---

## `ip addr show`

Displays network interfaces and assigned IP addresses.

### Example

```bash
ip addr show
```

Example output:

```text
2: wlo1: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 192.168.50.200/24

3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP>
    inet 172.17.0.1/16
```

### Important Fields

| Field | Meaning |
|---------|---------|
| `UP` | Interface enabled |
| `LOWER_UP` | Physical link active |
| `NO-CARRIER` | No active connection |
| `inet` | IPv4 address |
| `inet6` | IPv6 address |
| `/24` | Prefix length (subnet mask) |

### Useful Commands

```bash
ip addr show
ip addr show dev wlo1
ip -brief addr
```

Example:

```text
lo       UNKNOWN 127.0.0.1/8
wlo1     UP      192.168.50.200/24
docker0  DOWN    172.17.0.1/16
```

---

## `ss -antup`

Modern replacement for `netstat`.

Shows active TCP/UDP sockets and listening services.

### Flags

| Flag | Meaning |
|--------|---------|
| `-a` | All sockets |
| `-n` | Numeric addresses (no DNS lookup) |
| `-t` | TCP sockets |
| `-u` | UDP sockets |
| `-p` | Show process owning socket |

### Example

```bash
ss -antup
```

Example output:

```text
State  Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
LISTEN 0      4096   0.0.0.0:22         0.0.0.0:*         users:(("sshd",pid=785))
ESTAB  0      0      192.168.50.200:22  192.168.50.10:51522
```

### States

| State | Meaning |
|---------|---------|
| `LISTEN` | Waiting for incoming connections |
| `ESTAB` | Connection established |
| `SYN-SENT` | Connection attempt initiated |
| `SYN-RECV` | Waiting for ACK |
| `FIN-WAIT-1` | Connection closing |
| `FIN-WAIT-2` | Waiting for peer close |
| `TIME-WAIT` | Recently closed connection |
| `CLOSE-WAIT` | Remote side closed connection |
| `CLOSED` | No connection |

### Useful Commands

Show listening services:

```bash
ss -lntp
```

Show established TCP connections:

```bash
ss -nt state established
```

Show SSH connections:

```bash
ss -ntp | grep :22
```

Show UDP sockets:

```bash
ss -unap
```

Show all listening ports and processes:

```bash
ss -lntup
```

---

# Quick Troubleshooting Workflow

## 1. Do I have an IP address?

```bash
ip addr show
```

## 2. Do I have routes?

```bash
ip route show
```

## 3. Which route will be used?

```bash
ip route get 8.8.8.8
```

## 4. Is a service listening?

```bash
ss -lntp
```

## 5. Are connections established?

```bash
ss -nt state established
```

## 6. Can I reach the destination?

```bash
ping 8.8.8.8
ping google.com
```

## Scheduled Tasks
crontab -l
ls -la /etc/cron*

Look for:
- Persistence
- Malware execution

## Systemd Services
systemctl list-units --type=service
systemctl list-unit-files

Look for:
- New or suspicious services
- Services running from unusual paths

## SSH Artifacts
cat ~/.ssh/authorized_keys

Look for:
- Unauthorized public keys
- Recently added access

## Command History
cat ~/.bash_history
cat ~/.zsh_history

Look for:
- wget/curl
- privilege escalation
- attacker commands

## Temporary Directories
ls -alh /tmp
ls -alh /var/tmp
ls -alh /dev/shm

Look for:
- Scripts
- Payloads
- Archives

## Open Files
lsof

Useful for:
- Deleted but running binaries
- Suspicious file access

```
# Examples

# Network connections IPV4
lsof -i 4

# Don't map host names (-n), port numbers (-P), usernames (-l)
lsof -i 4 -nlP

lsof -nlP

# A protocol name - TCP, UDP
lsof -i TCP
lsof -i UDP

# All tcp connection in LISTEN state
lsof -iTCP -sTCP:LISTEN

# All tcp connection in ESTABLISHED state
lsof -iTCP -sTCP:ESTABLISHED

# Show all network connections with PID and process name
lsof -i -nP

# Only listening ports
lsof -iTCP -sTCP:LISTEN -nP

# Only established connections
lsof -iTCP -sTCP:ESTABLISHED -nP

# Deleted but still open files (VERY IMPORTANT)
lsof +L1

# Open files for a process
lsof -p <PID>

# Executable, cwd, libraries used by process
lsof -p <PID> | egrep 'txt|cwd|mem'

# Files opened by a specific user
lsof -u root

# Files under suspicious directories
lsof +D /tmp
lsof +D /var/tmp

# Which process is using a file
lsof /path/to/file

# Which process has a directory open
lsof +D /home/user

# Network connections of a specific process
lsof -p <PID> -i

# Specific remote host
lsof -i @1.2.3.4

# Specific TCP port range
lsof -i TCP:1-1024

# IPv6 only
lsof -i 6
#

```

## Loaded Kernel Modules
lsmod

Look for:
- Unknown modules
- Rootkit indicators

## Containers
docker ps -a
docker images

Look for:
- Unauthorized containers
- Crypto miners

## Memory Related
cat /proc/<pid>/cmdline
cat /proc/<pid>/environ

Useful when:
- Malware is memory-only
- Process arguments are hidden later