# Linux Module 7: Logging & Alerting Concepts

---

## 📌 Why This Module Matters

In real-world projects — DevOps pipelines, backup scripts, server automation, monitoring tools — **95% of debugging time is spent reading logs**.

Good logging turns your script from a black box into a transparent system that clearly shows:
- What happened
- When it happened
- Why it failed
- What action was taken

Good alerting means problems are **caught and escalated before users notice them**.

---

## 1️⃣ Structured Logging

### Why Structure Matters

An unstructured log is just noise:
```
backup started
error connecting
done
```

A structured log is scannable, greppable, and parseable by external systems:
```
[2026-03-13 07:45:12 UTC] [INFO]     Backup started for /var/www/myapp
[2026-03-13 07:45:15 UTC] [ERROR]    Failed to connect to S3: Connection timeout
[2026-03-13 07:45:20 UTC] [WARN]     Disk usage at 87% — approaching threshold
[2026-03-13 07:45:21 UTC] [DEBUG]    Retry attempt 1 of 3
```

### The Three Non-Negotiable Fields

| Field | Purpose | Example |
|-------|---------|---------|
| **Timestamp** | When — essential for correlating events across systems | `2026-03-13 07:45:12 UTC` |
| **Level** | Severity — lets you filter signal from noise | `ERROR` |
| **Message** | What happened + enough context to act on it | `S3 upload failed: timeout after 30s` |

> Always use **UTC** timestamps in production logs. Local time creates confusion when servers are in different timezones or when daylight saving changes happen. A log entry at "2:30 AM" during a DST transition is ambiguous — UTC never is.

### Log Levels

| Level | When to use | Action required? |
|-------|------------|-----------------|
| `DEBUG` | Detailed internal state — variable values, loop iterations | No — disabled in production by default |
| `INFO` | Normal operation milestones — start, finish, checkpoints | No |
| `WARN` | Unusual but non-fatal — approaching threshold, retrying | Monitor — might need attention |
| `ERROR` | An operation failed — something didn't work as expected | Yes — investigate |
| `CRITICAL` | System-level failure requiring immediate action | Yes — page someone |

---

### Production-Grade Logging Functions

```bash
#!/usr/bin/env bash
set -euo pipefail

LOGFILE="/var/log/myproject.log"
DEBUG="${DEBUG:-false}"           # Default: debug off. Enable with: DEBUG=true ./script.sh

# ── Core logging engine ──────────────────────────────────────────────────────
log() {
    local level="$1"
    local message="$2"
    local timestamp

    timestamp=$(date --utc +"%Y-%m-%d %H:%M:%S UTC")   # UTC always — never local time

    printf '[%s] [%-8s] %s\n' \
        "$timestamp" \
        "$level" \
        "$message" \
    | tee -a "$LOGFILE"           # tee: write to both stdout (terminal) and the log file
}

# ── Level-specific wrappers ──────────────────────────────────────────────────
log_info()  { log "INFO"     "$1"; }               # Normal operation — goes to stdout
log_warn()  { log "WARN"     "$1"; }               # Unusual — goes to stdout
log_error() { log "ERROR"    "$1" >&2; }           # Failure — redirected to stderr
log_debug() {                                       # Only emits when DEBUG=true
    [[ "${DEBUG}" == "true" ]] && log "DEBUG" "$1" || true
}
log_critical() {                                    # Fatal — logs to stderr then exits non-zero
    log "CRITICAL" "$1" >&2
    exit 1
}
```

### Why Each Design Decision Was Made

```bash
# printf vs echo
printf '[%s] [%s] %s\n' "$ts" "$level" "$msg"   # printf: predictable formatting,
                                                  # handles special chars safely
echo "[$ts] [$level] $msg"                        # echo: interprets -e, \n etc.
                                                  # — can corrupt log lines with special chars

# tee -a vs > or >>
printf '...' | tee -a "$LOGFILE"   # tee -a: appends to file AND shows on terminal
printf '...' >> "$LOGFILE"         # >>: appends to file only — you're flying blind
                                   # during manual runs or debugging

# >&2 on errors
log "ERROR" "$msg" >&2             # Errors → stderr: allows callers to capture stdout
                                   # separately from error output in pipelines and CI logs

# DEBUG gate
[[ "${DEBUG}" == "true" ]] && log "DEBUG" "$1" || true
# || true prevents set -e from exiting when DEBUG is false
# (the [[ ]] returns 1 when false, which would trigger -e without || true)
```

### Usage in a Script

```bash
log_info     "Backup started for /var/www/myapp"
log_debug    "Config loaded: HOST=$DB_HOST PORT=$DB_PORT"   # Only visible with DEBUG=true
log_warn     "S3 response slow — 4200ms (threshold: 3000ms)"
log_error    "Database connection failed after 3 retries"
log_critical "Disk at 100% — cannot continue"               # Logs then exits with code 1
```

---

### JSON Structured Logging — The Production Upgrade

Plain-text logs work for humans reading a terminal. But log pipelines — ELK, Splunk, Loki/Grafana, Datadog — work best with **JSON logs**. Each log line becomes a structured object that can be queried, aggregated, and alerted on with no parsing configuration.

```bash
log_json() {
    local level="$1"
    local message="$2"
    local timestamp

    timestamp=$(date --utc +"%Y-%m-%dT%H:%M:%SZ")   # ISO 8601 — standard machine-readable format

    printf '{"timestamp":"%s","level":"%s","message":"%s","host":"%s","script":"%s"}\n' \
        "$timestamp" \
        "$level" \
        "$message" \
        "$(hostname)" \               # Which server produced this log
        "$(basename "$0")" \          # Which script produced this log
    | tee -a "$LOGFILE"
}
```

Output:
```json
{"timestamp":"2026-03-13T07:45:12Z","level":"ERROR","message":"S3 upload failed","host":"web-01","script":"backup.sh"}
```

> You don't have to start with JSON. Start with plain structured text — it's human-readable and still greppable. Upgrade to JSON when you connect to a log aggregation system.

---

## 2️⃣ Real-Time Tailing & Filtering

### The Core Pattern — `tail -f`

```bash
tail -f /var/log/myproject.log      # -f = follow: streams new lines as they are written
                                    # stays open until you Ctrl+C
```

> `tail -f` is your primary tool during a live deployment or incident. You run it in one terminal window and watch the log scroll in real time as your script executes.

### `--line-buffered` — Why It Matters in Pipelines

By default, `grep` buffers output — it waits to collect a full buffer before printing. In a `tail -f` pipeline, this means you see nothing for minutes, then a burst. `--line-buffered` forces grep to flush every line immediately:

```bash
# Without --line-buffered: output is delayed (buffered in chunks)
tail -f app.log | grep "ERROR"

# With --line-buffered: every matching line appears instantly as it's written
tail -f app.log | grep --line-buffered "ERROR"
```

> Always use `--line-buffered` when piping `tail -f` into `grep`. Without it, real-time monitoring is silently broken.

### Filtering Patterns

```bash
# Watch only errors — useful during a deployment
tail -f /var/log/myproject.log \
    | grep --line-buffered "\[ERROR\]"

# Watch errors and critical together — use extended regex
tail -f /var/log/myproject.log \
    | grep --line-buffered -E "\[ERROR\]|\[CRITICAL\]"

# Add colour — highlights matching text in the terminal
tail -f /var/log/myproject.log \
    | grep --line-buffered -E "\[ERROR\]|\[CRITICAL\]" --color=always

# Suppress debug noise — see everything except DEBUG lines
tail -f /var/log/myproject.log \
    | grep --line-buffered -v "\[DEBUG\]"     # -v inverts match — exclude DEBUG

# Narrow further — errors from a specific component
tail -f /var/log/myproject.log \
    | grep --line-buffered "\[ERROR\].*backup"   # ERROR lines that also mention "backup"

# Multiple greps in a pipeline — each stage narrows the stream
tail -f /var/log/myproject.log \
    | grep --line-buffered -v "\[DEBUG\]" \      # Remove debug noise first
    | grep --line-buffered -E "\[WARN\]|\[ERROR\]|\[CRITICAL\]"  # Keep only concerning levels
```

### Split the Stream — Watch and Archive Simultaneously

```bash
# Full log streams to screen AND errors are separately archived
tail -f "$LOGFILE" \
    | tee >(grep --line-buffered "\[ERROR\]" \
            >> /var/log/errors-only.log)   # Process substitution: fork the stream
                                           # Full stream → terminal
                                           # ERROR lines → separate error log
```

> This pattern is useful when you want a dedicated error log for alerting systems to watch, while the full log retains the complete audit trail.

### Other Useful Monitoring Approaches

```bash
# less +F — like tail -f but you can scroll back with Ctrl+C then navigate
less +F /var/log/myproject.log        # Ctrl+C to stop following, then navigate normally
                                      # F again to resume following

# watch — re-run a command every N seconds and show its output
watch -n 2 "tail -n 20 /var/log/myproject.log"   # Refresh last 20 lines every 2 seconds
watch -n 5 "grep '\[ERROR\]' /var/log/myproject.log | wc -l"   # Monitor error count over time
```

---

## 3️⃣ Basic Alerting Logic

Alerting is the bridge between logging and action. The pattern is always the same:

```
measure something → compare to threshold → if threshold crossed → act
```

### Disk Usage Alert

```bash
check_disk_usage() {
    local mount="${1:-/}"             # Default to / if no argument given
    local threshold="${2:-90}"        # Default threshold: 90%
    local usage

    # df shows disk stats — tail -1 gets the data row, awk extracts column 5 (%used)
    # tr -d '%' strips the percent sign so we can do numeric comparison
    usage=$(df "$mount" \
        | tail -1 \
        | awk '{print $5}' \
        | tr -d '%')

    log_info "Disk usage on $mount: ${usage}%"

    if [[ "$usage" -ge "$threshold" ]]; then
        log_critical "Disk usage CRITICAL: ${usage}% on $mount (threshold: ${threshold}%)"
        # log_critical calls exit 1 — everything below this line is for WARN case
    elif [[ "$usage" -ge $(( threshold - 10 )) ]]; then
        log_warn "Disk usage elevated: ${usage}% on $mount — approaching ${threshold}% threshold"
    fi
}

# Call with defaults (/ at 90%)
check_disk_usage

# Call with custom mount and threshold
check_disk_usage "/data" 80
```

> Adding a **warn zone** (threshold - 10%) is important — it gives you advance warning before the system hits the critical limit. One level of alert is better than none; two levels (warn + critical) is production-grade.

### CPU Usage Alert

```bash
check_cpu_usage() {
    local threshold="${1:-85}"
    local cpu_idle
    local cpu_used

    # top -bn1: batch mode, one iteration — suitable for scripting (not interactive)
    # grep "Cpu(s)": find the CPU summary line
    # awk: extract the idle percentage (field 8 in most top versions)
    cpu_idle=$(top -bn1 \
        | grep "Cpu(s)" \
        | awk '{print $8}' \
        | cut -d. -f1)            # Integer part only — strip decimal

    cpu_used=$(( 100 - cpu_idle ))   # used = 100 - idle

    log_info "CPU usage: ${cpu_used}% (idle: ${cpu_idle}%)"

    if [[ "$cpu_used" -gt "$threshold" ]]; then
        log_warn "High CPU usage: ${cpu_used}% (threshold: ${threshold}%)"

        # Log the top 5 CPU consumers for immediate context
        log_info "Top CPU processes:"
        ps aux --sort=-%cpu \
            | head -6 \
            | tail -5 \
            | while IFS= read -r line; do
                log_info "  $line"
            done
    fi
}
```

### Memory Usage Alert

```bash
check_memory_usage() {
    local threshold="${1:-90}"
    local mem_total mem_used mem_pct

    # /proc/meminfo is the authoritative source for memory stats
    mem_total=$(grep MemTotal /proc/meminfo | awk '{print $2}')   # kB
    mem_avail=$(grep MemAvailable /proc/meminfo | awk '{print $2}') # kB (includes cache)
    mem_used=$(( mem_total - mem_avail ))
    mem_pct=$(( mem_used * 100 / mem_total ))

    log_info "Memory usage: ${mem_pct}% (${mem_used}kB used of ${mem_total}kB)"

    if [[ "$mem_pct" -ge "$threshold" ]]; then
        log_warn "High memory usage: ${mem_pct}% (threshold: ${threshold}%)"
    fi
}
```

> Use `MemAvailable` from `/proc/meminfo`, not `MemFree`. `MemFree` is literally unused memory — it looks alarming because Linux aggressively uses free RAM as disk cache. `MemAvailable` is what the kernel estimates is actually available for new allocations, accounting for reclaimable cache.

### Service Health Check

```bash
check_service() {
    local service="$1"

    # systemctl is-active returns 0 if running, non-zero otherwise
    if ! systemctl is-active --quiet "$service"; then
        log_error "Service is not running: $service"

        # Log the last 10 lines of the service journal for immediate triage
        log_error "Last journal entries for $service:"
        journalctl -u "$service" -n 10 --no-pager \
            | while IFS= read -r line; do
                log_error "  $line"
            done

        return 1
    fi

    log_info "Service is running: $service"
}

# HTTP endpoint health check
check_http_endpoint() {
    local url="$1"
    local expected_code="${2:-200}"
    local actual_code

    # curl -s: silent, -o /dev/null: discard body, -w: write status code only
    actual_code=$(curl -s -o /dev/null -w "%{http_code}" \
        --connect-timeout 5 \          # Give up connecting after 5s
        --max-time 10 \                # Give up entire request after 10s
        "$url")

    if [[ "$actual_code" != "$expected_code" ]]; then
        log_error "Health check failed: $url returned HTTP $actual_code (expected $expected_code)"
        return 1
    fi

    log_info "Health check passed: $url returned HTTP $actual_code"
}
```

### Alerting Integrations

Once a threshold is crossed, logging alone is not enough — alerts need to reach people:

```bash
send_alert() {
    local subject="$1"
    local message="$2"

    # Email — requires mail/sendmail configured on the server
    echo "$message" | mail -s "$subject" "ops@yourcompany.com"

    # Slack webhook
    curl -s -X POST "https://hooks.slack.com/services/YOUR/WEBHOOK/URL" \
        -H 'Content-type: application/json' \
        --data "{\"text\":\"🚨 *${subject}*\n${message}\"}"

    # PagerDuty (for on-call escalation)
    curl -s -X POST "https://events.pagerduty.com/v2/enqueue" \
        -H "Content-Type: application/json" \
        --data "{
            \"routing_key\": \"YOUR_INTEGRATION_KEY\",
            \"event_action\": \"trigger\",
            \"payload\": {
                \"summary\": \"$subject\",
                \"severity\": \"critical\",
                \"source\": \"$(hostname)\"
            }
        }"
}
```

### 🏭 Complete System Health Monitor

```bash
#!/usr/bin/env bash
set -euo pipefail

LOGFILE="/var/log/system-monitor.log"
DEBUG="${DEBUG:-false}"

# ... (logging functions from section 1 go here) ...

monitor_system() {
    log_info "=== System health check started on $(hostname) ==="

    check_disk_usage "/"  90
    check_disk_usage "/data" 80          # Check secondary mounts with custom thresholds
    check_cpu_usage   85
    check_memory_usage 90

    for svc in nginx postgresql redis; do
        check_service "$svc"
    done

    check_http_endpoint "http://localhost:8080/health" 200

    log_info "=== System health check completed ==="
}

monitor_system
```

Run manually with debug output:
```bash
DEBUG=true ./monitor.sh
```

Watch it live while it runs:
```bash
tail -f /var/log/system-monitor.log | grep --line-buffered -v "\[DEBUG\]"
```

---

## 4️⃣ Cron Job Basics

Cron is Linux's built-in task scheduler. It runs commands automatically at defined intervals — no manual intervention needed.

### The Cron Expression Format

```
┌─────────────── minute        (0–59)
│   ┌─────────── hour          (0–23)
│   │   ┌─────── day of month  (1–31)
│   │   │   ┌─── month         (1–12)
│   │   │   │   ┌ day of week  (0–7, both 0 and 7 = Sunday)
│   │   │   │   │
*   *   *   *   *   command to run
```

### Common Schedules

| Expression | Meaning | Typical use |
|-----------|---------|------------|
| `* * * * *` | Every minute | Testing only — never leave in production |
| `*/5 * * * *` | Every 5 minutes | Health checks, monitoring |
| `0 * * * *` | Every hour (at :00) | Log rotation, cleanup |
| `30 2 * * *` | 2:30 AM daily | Backups, reports |
| `0 3 * * 0` | 3:00 AM every Sunday | Weekly maintenance |
| `0 0 1 * *` | Midnight, 1st of month | Monthly reports |
| `0 9-17 * * 1-5` | Every hour, 9am–5pm, weekdays | Business hours checks |
| `*/15 8-20 * * *` | Every 15 min, 8am–8pm | Frequent daytime monitoring |

### Managing Crontab

```bash
crontab -l                   # List current user's crontab — see all scheduled jobs
crontab -e                   # Edit current user's crontab (opens in $EDITOR)
crontab -r                   # ⚠️ Remove ALL cron jobs — no confirmation, no undo
crontab -u alice -l          # List another user's crontab (requires root)

# System-wide cron directories (no crontab -e needed — just drop files in):
/etc/cron.daily/             # Scripts run once per day by the OS
/etc/cron.hourly/            # Scripts run once per hour
/etc/cron.weekly/            # Scripts run once per week
/etc/cron.monthly/           # Scripts run once per month
```

### Production Cron Rules

**1. Always use absolute paths — for everything:**
```bash
# ❌ Wrong — cron's PATH is minimal (/usr/bin:/bin) and won't find your scripts
30 2 * * * backup.sh

# ✅ Correct — explicit absolute path to both the interpreter and the script
30 2 * * * /usr/bin/env bash /home/alice/scripts/backup.sh
```

**2. Always redirect output to a log file:**
```bash
# ❌ Wrong — output goes to local mail (usually ignored), errors are invisible
30 2 * * * /usr/bin/env bash /home/alice/scripts/backup.sh

# ✅ Correct — stdout and stderr both captured to a log
30 2 * * * /usr/bin/env bash /home/alice/scripts/backup.sh >> /var/log/backup.log 2>&1
```

**3. Use `flock` to prevent overlapping runs for long jobs:**
```bash
# Without flock: if backup takes 3 hours, the 2 AM job and next day's 2 AM job overlap
30 2 * * * /usr/bin/env bash /home/alice/scripts/backup.sh >> /var/log/backup.log 2>&1

# With flock: second instance exits immediately if first is still running
30 2 * * * flock -n /tmp/backup.lock /usr/bin/env bash /home/alice/scripts/backup.sh >> /var/log/backup.log 2>&1
# -n = non-blocking: if the lock can't be acquired, exit immediately rather than waiting
```

**4. Set `MAILTO` to suppress or route mail:**
```bash
# At the top of crontab:
MAILTO=""                    # Suppress all cron email notifications
MAILTO="ops@company.com"     # Or route them to a real address
```

### Cron's Environment Problem

Cron runs with a **minimal, stripped-down environment** — not your login shell's environment. This is the most common reason cron jobs fail silently when run manually they work fine.

```bash
# What cron's PATH typically looks like:
/usr/bin:/bin

# What your login shell's PATH typically looks like:
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/home/alice/.local/bin
```

**Fixes:**

```bash
# Option 1 — set PATH explicitly at the top of the crontab
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Option 2 — set environment inside the script itself
export PATH="/usr/local/bin:/usr/bin:/bin:$PATH"

# Option 3 — use /usr/bin/env bash and absolute paths everywhere (most robust)
30 2 * * * /usr/bin/env bash /home/alice/scripts/backup.sh >> /var/log/backup.log 2>&1
```

**Debugging a cron environment mismatch:**
```bash
# Add this to crontab temporarily — it dumps what cron actually sees to a file
* * * * * env > /tmp/cron-environment.txt

# Compare with your login session:
diff /tmp/cron-environment.txt <(env | sort)
# Anything in your session but not in cron-environment.txt is a potential source of failure
```

### Complete Production Crontab Example

```bash
# Crontab for production server monitoring and backup
# Managed by: ops team
# Last updated: 2026-03-13

MAILTO=""                    # Suppress noisy mail — all output goes to log files instead
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin  # Explicit full PATH

# System health monitoring — every 5 minutes
*/5 * * * *   flock -n /tmp/monitor.lock /usr/bin/env bash /opt/scripts/monitor.sh >> /var/log/monitor.log 2>&1

# Daily database backup — 2:30 AM (off-peak)
30 2 * * *    flock -n /tmp/backup.lock  /usr/bin/env bash /opt/scripts/backup.sh  >> /var/log/backup.log  2>&1

# Weekly log archive and cleanup — 3:00 AM every Sunday
0 3 * * 0     /usr/bin/env bash /opt/scripts/archive-logs.sh >> /var/log/archive.log 2>&1

# Monthly report generation — midnight on the 1st
0 0 1 * *     /usr/bin/env bash /opt/scripts/monthly-report.sh >> /var/log/reports.log 2>&1
```

---

## Key Takeaways

- Always use UTC timestamps — local time causes confusion across timezones and DST changes
- `printf` over `echo` for log formatting — handles special characters predictably
- `tee -a` over `>>` alone — you see output live AND it's saved to the log file
- Route errors to `stderr` (`>&2`) — lets CI/CD systems and callers distinguish output from errors
- `--line-buffered` is mandatory when piping `tail -f` into `grep` — without it, real-time monitoring is silently broken
- Build alerting in two stages — a **warn zone** before the critical threshold gives advance notice
- Use `MemAvailable` not `MemFree` from `/proc/meminfo` — `MemFree` excludes reclaimable cache and makes memory look falsely scarce
- In cron, use absolute paths for everything — cron's `PATH` is minimal and will silently fail to find your tools
- Always redirect cron output to a log file with `>> /var/log/job.log 2>&1` — unredirected output disappears
- Use `flock -n` on long-running cron jobs — prevents the next scheduled run from overlapping with a still-running one
- Debug cron failures by dumping `env > /tmp/cron-env.txt` and comparing with your login session environment

---