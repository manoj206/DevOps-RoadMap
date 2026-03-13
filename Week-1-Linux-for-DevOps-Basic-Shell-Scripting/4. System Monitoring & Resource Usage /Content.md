# Linux Module 4: System Monitoring & Resource Usage

This module covers the **first-response toolkit** used in production when a server feels slow, is unresponsive, or is about to crash or fill up.

These commands remain essential even in modern environments (Kubernetes nodes, bare-metal troubleshooting, edge nodes, and emergency SSH sessions).

---

# 1️⃣ Disk Usage & Filesystem Health

| Command               | Purpose                                | Production examples / one-liners                           |
| --------------------- | -------------------------------------- | ---------------------------------------------------------- |
| `df -hT`              | Human-readable usage + filesystem type | `df -hT \| grep -v tmpfs \| sort -k6 -nr` — sort by % used |
| `du -sh`              | Summarize directory usage              | `sudo du -sh /* 2>/dev/null \| sort -hr`                   |
| `du -h --max-depth=1` | One-level breakdown                    | `sudo du -h --max-depth=1 /var/log \| sort -hr`            |
| `lsblk`               | Show block devices and mount points    | `lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,UUID`           |
| `df -i`               | Check inode usage                      | `df -i \| sort -k5 -nr`                                    |

### Common Production Patterns

```bash
# Quick full-disk triage
df -hT | grep -v tmpfs | sort -k6 -nr | head -10

# Find largest directories
sudo du -sh /var/lib/docker/* 2>/dev/null | sort -hr | head -8
sudo du -sh /var/log/* 2>/dev/null | sort -hr | head -8

# Inode exhaustion investigation
df -i
find /var/cache -xdev -type f -printf '%h\n' | sort | uniq -c | sort -nr | head -10
```

### Analogy

* **df** → fuel gauge on the dashboard
* **du** → opening the trunk to see what’s actually taking space

---

# 2️⃣ Memory & Swap

| Command             | Best For              | Interpretation Tips                   |
| ------------------- | --------------------- | ------------------------------------- |
| `free -wh`          | Quick overview        | Focus on **available**, not “free”    |
| `vmstat 1 5`        | Memory + CPU activity | Watch **si / so** swap columns        |
| `cat /proc/meminfo` | Detailed counters     | Look at `Committed_AS`, `CommitLimit` |

### Danger Signs (Production Red Flags)

* `available` < 10–15% of total RAM
* Swap usage growing rapidly
* `si` / `so` in `vmstat` constantly non-zero
* Kernel logs showing OOM events

Check OOM activity:

```bash
dmesg | grep -i 'out of memory'
journalctl -k -g oom
```

### Live Monitoring Pattern

```bash
watch -n 2 "free -wh && vmstat 1 4 | tail -4"
```

---

# 3️⃣ CPU & I/O Statistics

| Tool      | Purpose                     | Production Example  |
| --------- | --------------------------- | ------------------- |
| `mpstat`  | Per-core CPU usage          | `mpstat -P ALL 1 6` |
| `iostat`  | Disk latency & throughput   | `iostat -xdz 1 5`   |
| `sar`     | Historical performance data | `sar -u 1 5`        |
| `iotop`   | Per-process disk activity   | `sudo iotop -o`     |
| `pidstat` | Per-process CPU stats       | `pidstat -u 1`      |

### Important `iostat -x` Columns

| Metric          | Meaning                          |
| --------------- | -------------------------------- |
| `%util`         | Disk busy percentage             |
| `await`         | Average I/O wait time            |
| `r/s` `w/s`     | Read/write operations per second |
| `rkB/s` `wkB/s` | Throughput                       |

### Classic On-Call Investigation Flow

```bash
iostat -x 1
mpstat 1
iotop -o -b -n 5
```

---

# 4️⃣ Network Connections & Sockets

| Command    | Purpose                   | Example                   |
| ---------- | ------------------------- | ------------------------- |
| `ss -s`    | Connection summary        | `ss -s`                   |
| `ss -ltnp` | Listening ports           | `ss -ltnp`                |
| `ss -antp` | Active TCP connections    | Connection state analysis |
| `lsof -i`  | Process ↔ network mapping | `lsof -i :443`            |

### Common Production One-Liners

```bash
# Connection states summary
ss -ant | tail -n +2 | awk '{print $1}' | sort | uniq -c | sort -nr

# Top client IPs
ss -ant | tail -n +2 | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr | head -15

# Processes with most open sockets
lsof -i -n -P | awk 'NR>1 {print $2}' | sort | uniq -c | sort -nr | head -10
```

---

# 5️⃣ Logging Locations & Basics

### Classic Log Files

* `/var/log/syslog`
* `/var/log/messages`
* `/var/log/auth.log`
* `/var/log/secure`
* `/var/log/kern.log`
* `/var/log/dmesg`

View kernel messages:

```bash
dmesg -T
```

### Systemd Logging (`journalctl`)

```bash
journalctl -u nginx -n 300
journalctl -u nginx.service --since "1 hour ago"
journalctl -p err -b
journalctl -k -g "oom"
journalctl --disk-usage
journalctl --vacuum-time=3weeks
```

---

# 6️⃣ Common Bottlenecks — Production Triage Checklist

| Symptom              | Likely Causes              | First Commands to Run          |
| -------------------- | -------------------------- | ------------------------------ |
| High CPU             | CPU-bound code             | `htop`, `mpstat`, `pidstat`    |
| High iowait          | Slow disk or excessive I/O | `iostat`, `iotop`, `vmstat`    |
| Memory pressure      | Memory leak                | `free`, `vmstat`, `journalctl` |
| Disk full            | Logs / docker / cache      | `df`, `du`                     |
| Inodes exhausted     | Too many small files       | `df -i`, `find`                |
| Too many connections | TIME_WAIT flood            | `ss`, `sysctl`                 |

Example inode check:

```bash
find / -xdev -type f -size -1M | wc -l
```

---

# 7️⃣ 30-Second Production Health Snapshot

Very useful for on-call engineers.

```bash
uptime
free -wh
df -hT | grep -v tmpfs | sort -k6 -nr | head -8
iostat -x 1 3 | tail -n +3
vmstat 1 3 | tail -3
ss -s
```

Or a compact single command:

```bash
uptime; free -wh; df -hT | grep -v tmpfs | sort -k6 -nr | head -8; iostat -x 1 3 | tail -n +3; vmstat 1 3 | tail -3; ss -s
```

---

# Summary

This module covered the **core production troubleshooting toolkit**:

* Disk usage investigation
* Memory pressure diagnosis
* CPU and I/O monitoring
* Network connection analysis
* System logging inspection
* Rapid production triage workflow

These commands form the **foundation of Linux observability during incidents**.

---