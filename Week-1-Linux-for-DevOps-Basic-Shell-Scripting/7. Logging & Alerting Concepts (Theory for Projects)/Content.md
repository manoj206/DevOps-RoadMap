# Module 7: Logging & Alerting Concepts (Theory for Projects)

---

# 📌 Why This Module Matters

In real-world projects (DevOps, monitoring tools, backup scripts, CI/CD pipelines, server automation), **95% of debugging time is spent reading logs**.

Good logging and alerting turns your script from a **black box** into a transparent system that clearly shows:

* What happened
* When it happened
* Why it failed
* What action was taken

This allows engineers to **detect problems before users notice them**.

---

# Core Philosophy

Remember these rules:

* Every important action must be logged with **timestamp + level + context**
* Logs must be **structured**
* Logs should be easy to **grep, parse, and ship to systems like ELK, Splunk, or Grafana**
* Alerting = **if something bad happens → act immediately**
* Cron = **your automated night-shift worker**

---

# 1️⃣ Structured Logging (timestamp + level + message)

## Recommended Log Format

Example production log lines:

```text
[2026-03-13 07:45:12 UTC] [INFO] Backup started for /etc
[2026-03-13 07:45:15 UTC] [ERROR] Failed to connect to S3: Connection timeout
[2026-03-13 07:45:20 UTC] [WARN] Disk usage at 87%
```

### Why this format?

| Component | Purpose                       |
| --------- | ----------------------------- |
| Timestamp | Shows when the event occurred |
| Level     | Indicates severity            |
| Message   | Human-readable description    |

Typical log levels:

| Level    | Meaning                            |
| -------- | ---------------------------------- |
| INFO     | Normal operation                   |
| WARN     | Something unusual but not fatal    |
| ERROR    | Operation failed                   |
| DEBUG    | Detailed debugging info            |
| CRITICAL | Immediate failure requiring action |

---

# Production-Grade Logging Functions

```bash
#!/usr/bin/env bash
set -euo pipefail

LOGFILE="/var/log/myproject.log"

log() {
    local level="$1"
    local message="$2"
    local timestamp

    timestamp=$(date --utc +"%Y-%m-%d %H:%M:%S UTC")

    printf '[%s] [%s] %s\n' "$timestamp" "$level" "$message" | tee -a "$LOGFILE"
}

log_info()     { log "INFO" "$1"; }
log_warn()     { log "WARN" "$1"; }
log_error()    { log "ERROR" "$1" >&2; }
log_debug()    { [[ "${DEBUG:-false}" == "true" ]] && log "DEBUG" "$1"; }
log_critical() { log "CRITICAL" "$1" >&2; exit 1; }
```

---

# Line-by-Line Explanation

| Component         | Explanation                    |
| ----------------- | ------------------------------ |
| `date --utc`      | Always use UTC timestamps      |
| `printf`          | Safer than `echo`              |
| `tee -a`          | Writes to both screen and file |
| `$DEBUG` variable | Enables optional debug logging |

Example usage:

```bash
log_info "Backup started"
log_warn "Disk usage is high"
log_error "Database connection failed"
log_debug "Raw response: $response"
```

---

# Real Project Upgrade (JSON Logging)

Later, logs can be structured as JSON:

```json
{"timestamp":"2026-03-13T07:45:12Z","level":"ERROR","message":"S3 failed","service":"backup","host":"server01"}
```

This format works well with log pipelines like:

* Elasticsearch / ELK stack
* Splunk
* Loki / Grafana
* Datadog

---

# 2️⃣ Real-Time Tailing & Filtering (tail + grep)

When monitoring logs, constantly reopening the file is inefficient.

Instead use **live log streaming**.

### Basic Log Watching

```bash
tail -f /var/log/myproject.log
```

---

### Watch Only Errors

```bash
tail -f /var/log/myproject.log | grep --line-buffered "\[ERROR\]"
```

---

### Watch Errors and Critical Alerts

```bash
tail -f /var/log/myproject.log \
| grep --line-buffered -E "\[ERROR\]|\[CRITICAL\]" --color=always
```

---

### Hide Debug Noise

```bash
tail -f /var/log/myproject.log | grep --line-buffered -v "\[DEBUG\]"
```

---

### Filter by Multiple Fields

```bash
tail -f /var/log/myproject.log \
| grep --line-buffered "\[ERROR\].*backup"
```

---

# Command Breakdown

| Command                | Purpose                            |
| ---------------------- | ---------------------------------- |
| `tail -f`              | Follow log updates in real time    |
| `grep --line-buffered` | Immediate output without buffering |
| `-E`                   | Extended regex support             |
| `--color=always`       | Highlight matches                  |

---

# Pro Monitoring Trick

Split log streams:

```bash
tail -f "$LOGFILE" | tee >(grep "\[ERROR\]" > /var/log/errors-only.log)
```

This allows:

* full logs in one file
* critical errors extracted separately

---

# Additional Useful Monitoring Tools

```bash
less +F logfile.log
watch -n 2 "tail -n 20 logfile.log"
```

These are often used during production debugging.

---

# 3️⃣ Basic Alerting Logic

Alerting means:

> If a threshold is crossed → trigger an action.

---

## Disk Usage Alert Example

```bash
check_disk_usage() {
    local threshold=90
    local usage

    usage=$(df / | tail -1 | awk '{print $5}' | tr -d '%')

    log_info "Disk usage on / is ${usage}%"

    if [[ "$usage" -ge "$threshold" ]]; then
        log_critical "Disk usage CRITICAL: ${usage}% (threshold ${threshold}%)"
    fi
}
```

---

# Full Monitoring Example

```bash
monitor_system() {
    log_info "Starting system health check..."

    local cpu
    cpu=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d. -f1)

    if [[ "$cpu" -gt 85 ]]; then
        log_warn "High CPU usage: ${cpu}%"
    fi

    check_disk_usage

    log_info "Health check completed successfully"
}
```

---

# Explanation of Key Commands

| Command | Purpose            |
| ------- | ------------------ |
| `df`    | disk usage         |
| `awk`   | extract fields     |
| `tr -d` | remove characters  |
| `-ge`   | numeric comparison |

---

# Real Alerting Integrations

Later you can add:

```bash
mail -s "Disk Alert" admin@example.com
curl -X POST https://hooks.slack.com/...
curl -X POST https://api.pagerduty.com/...
```

These connect your scripts to **real incident systems**.

---

# 4️⃣ Cron Job Basics (Scheduling)

Cron is Linux’s built-in scheduler.

### Cron Format

```
* * * * * command
│ │ │ │ │
│ │ │ │ └── Day of week (0–7)
│ │ │ └──── Month
│ │ └────── Day of month
│ └──────── Hour
└────────── Minute
```

---

# Common Cron Schedules

| Schedule      | Meaning         | Example     |
| ------------- | --------------- | ----------- |
| `* * * * *`   | every minute    | testing     |
| `*/5 * * * *` | every 5 minutes | monitoring  |
| `0 * * * *`   | hourly          | cleanup     |
| `30 2 * * *`  | daily backup    | night jobs  |
| `0 3 * * 0`   | weekly          | maintenance |
| `0 0 1 * *`   | monthly         | reports     |

---

# Managing Cron

```bash
crontab -l
crontab -e
crontab -r
```

---

# Production Cron Rules

Always follow these:

1. Use **absolute paths**
2. Log all output
3. Test schedules before deploying

Example:

```cron
30 2 * * * /usr/bin/env bash /home/manoj/scripts/backup.sh >> /var/log/backup.log 2>&1
```

---

# Real Cron Example

```cron
# Daily system monitoring
0 3 * * * /home/manoj/scripts/monitor.sh >> /var/log/monitor.log 2>&1
```

---

# Cron Debugging Tip

Cron runs with a **minimal environment**.

If something fails:

```bash
env > cron-env.txt
```

This helps compare environment variables.

---

# Complete Production Example Script

```bash
#!/usr/bin/env bash
set -euo pipefail

LOGFILE="/var/log/system-monitor.log"
DEBUG=${DEBUG:-false}

monitor_system() {
    log_info "=== System Health Check Started ==="

    check_disk_usage

    log_info "=== System Health Check Completed ==="
}

monitor_system
```

---

# Running the Script

Manual run:

```bash
DEBUG=true ./monitor.sh
```

Monitor logs:

```bash
tail -f /var/log/system-monitor.log
```

---

# Summary

This module introduced **core logging and alerting principles used in real DevOps automation**.

You learned:

* Structured logging
* Log levels and formatting
* Real-time log monitoring
* Basic alerting logic
* Threshold detection
* Cron-based scheduling
* Production logging practices

These concepts form the foundation for **building reliable monitoring scripts and automation pipelines**.

---