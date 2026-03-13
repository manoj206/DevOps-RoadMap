## 🧪 Scenario 1 — System Health Check Script

**What it covers:** File permissions, process inspection, text processing pipelines, structured logging, functions, exit codes, `trap`

**The brief:** Write a script that checks disk, memory, and a running process — logs results in structured format — and exits with a meaningful code.

### Step 1 — Set up the environment

```bash
mkdir -p ~/scripts/health-check
cd ~/scripts/health-check
touch monitor.sh
chmod 755 monitor.sh
```

### Step 2 — Write the script

```bash
nano monitor.sh
```

```bash
#!/usr/bin/env bash
set -euo pipefail

# ── Config ────────────────────────────────────────────────────
LOGFILE="/tmp/health-check.log"
DISK_THRESHOLD=80
MEM_THRESHOLD=80
CHECK_PROCESS="bash"              # Change to nginx/sshd if available on the playground

# ── Logging ───────────────────────────────────────────────────
log() {
    local level="$1"
    local message="$2"
    local ts
    ts=$(date --utc +"%Y-%m-%d %H:%M:%S UTC")
    printf '[%s] [%-8s] %s\n' "$ts" "$level" "$message" | tee -a "$LOGFILE"
}
log_info()  { log "INFO"     "$1"; }
log_warn()  { log "WARN"     "$1"; }
log_error() { log "ERROR"    "$1" >&2; }

# ── Cleanup on exit ───────────────────────────────────────────
cleanup() {
    local code=$?
    (( code == 0 )) \
        && log_info "Health check finished cleanly." \
        || log_error "Health check exited with code $code."
}
trap cleanup EXIT

# ── Checks ────────────────────────────────────────────────────
check_disk() {
    local usage
    usage=$(df / | tail -1 | awk '{print $5}' | tr -d '%')
    log_info "Disk usage on /: ${usage}%"
    [[ "$usage" -ge "$DISK_THRESHOLD" ]] \
        && log_warn "Disk above threshold (${usage}% >= ${DISK_THRESHOLD}%)"
}

check_memory() {
    local total avail used pct
    total=$(grep MemTotal     /proc/meminfo | awk '{print $2}')
    avail=$(grep MemAvailable /proc/meminfo | awk '{print $2}')
    used=$(( total - avail ))
    pct=$(( used * 100 / total ))
    log_info "Memory usage: ${pct}% (${used}kB / ${total}kB)"
    [[ "$pct" -ge "$MEM_THRESHOLD" ]] \
        && log_warn "Memory above threshold (${pct}% >= ${MEM_THRESHOLD}%)"
}

check_process() {
    local proc="$1"
    if pgrep -x "$proc" > /dev/null 2>&1; then
        local pid count
        pid=$(pgrep -x "$proc" | head -1)
        count=$(pgrep -x "$proc" | wc -l)
        log_info "Process '$proc' is running — PID $pid ($count instance(s))"
    else
        log_error "Process '$proc' is NOT running"
        return 1
    fi
}

# ── Main ──────────────────────────────────────────────────────
main() {
    log_info "=== Health check started on $(hostname) ==="
    check_disk
    check_memory
    check_process "$CHECK_PROCESS"
    log_info "=== Health check complete ==="
}

main
```

### Step 3 — Run and observe

```bash
./monitor.sh                          # Normal run
cat /tmp/health-check.log             # Review the structured log
grep "\[WARN\]" /tmp/health-check.log # Pull only warnings
```

### Step 4 — Break it deliberately (this is where the learning is)

```bash
chmod 000 /tmp/health-check.log       # Remove write access to the log
./monitor.sh                          # Watch it fail — what's the exit code?
echo $?                               # Check it

chmod 644 /tmp/health-check.log       # Restore access
```

**Interview angle:** *"Walk me through what set -euo pipefail does in this script and why you'd use trap."*

---

## 🧪 Scenario 2 — Log Analyser Pipeline

**What it covers:** Text processing (`grep`, `awk`, `sort`, `uniq`, `sed`, `wc`), pipelines, file operations, the Extract→Group→Count→Rank pattern

**The brief:** Generate a fake application log, then write a pipeline-based analyser that answers real ops questions from it.

### Step 1 — Generate a realistic fake log

```bash
mkdir -p ~/scripts/log-analysis
cd ~/scripts/log-analysis

# Generate 200 fake log lines
cat > generate-log.sh << 'EOF'
#!/usr/bin/env bash

LEVELS=("INFO" "INFO" "INFO" "WARN" "ERROR" "DEBUG")
MESSAGES=(
    "User login successful"
    "Database query executed"
    "Cache miss on key user_session"
    "Response time exceeded 2000ms"
    "Failed to connect to upstream service"
    "Retrying request attempt 2 of 3"
    "Payment processed successfully"
    "Invalid token rejected"
    "Config reloaded"
    "Disk usage check passed"
)
IPS=("10.0.1.5" "10.0.1.12" "10.0.2.8" "10.0.2.19" "10.0.3.3")

for i in $(seq 1 200); do
    TS=$(date --utc +"%Y-%m-%d %H:%M:%S UTC")
    LEVEL=${LEVELS[$((RANDOM % ${#LEVELS[@]}))]}
    MSG=${MESSAGES[$((RANDOM % ${#MESSAGES[@]}))]}
    IP=${IPS[$((RANDOM % ${#IPS[@]}))]}
    printf '[%s] [%-5s] [%s] %s\n' "$TS" "$LEVEL" "$IP" "$MSG"
done
EOF

chmod +x generate-log.sh
./generate-log.sh > app.log
wc -l app.log                         # Confirm 200 lines were created
```

### Step 2 — Answer ops questions using pipelines

Now open `nano analyser.sh` and build each answer:

```bash
#!/usr/bin/env bash
# Log analyser — run against app.log

LOGFILE="${1:-app.log}"

echo "======================================"
echo " Log Analysis Report: $LOGFILE"
echo "======================================"

echo ""
echo "── Total lines ──────────────────────"
wc -l < "$LOGFILE"                               # Total line count

echo ""
echo "── Line count by log level ──────────"
grep -oE '\[(INFO|WARN|ERROR|DEBUG)\]' "$LOGFILE" \  # Extract just the level token
    | sort \                                          # Group identical levels
    | uniq -c \                                       # Count each
    | sort -nr                                        # Most frequent first

echo ""
echo "── Error messages (full lines) ──────"
grep "\[ERROR\]" "$LOGFILE"                      # All ERROR lines

echo ""
echo "── Most active source IPs ───────────"
grep -oE '\[([0-9]{1,3}\.){3}[0-9]{1,3}\]' "$LOGFILE" \  # Extract IP tokens
    | tr -d '[]' \                                         # Strip brackets
    | sort \                                               # Group by IP
    | uniq -c \                                            # Count per IP
    | sort -nr \                                           # Highest first
    | head -5                                              # Top 5 only

echo ""
echo "── Most frequent log messages ───────"
awk '{print $NF}' "$LOGFILE" \              # Last field — won't work perfectly but teaches $NF
    | sort | uniq -c | sort -nr | head -5

echo ""
echo "── Sanitised log (remove DEBUG) ─────"
sed '/\[DEBUG\]/d' "$LOGFILE" | wc -l       # Count lines after stripping DEBUG
echo "lines remain after removing DEBUG entries"
```

```bash
chmod +x analyser.sh
./analyser.sh app.log
```

### Step 3 — Extend it yourself (the actual practice)

Try answering these questions by writing your own one-liners:

```bash
# 1. How many unique IPs appear in the log?
grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' app.log | sort -u | wc -l

# 2. Replace all WARN with WARNING in a copy of the log
cp app.log app-modified.log
sed -i.bak 's/\[WARN \]/[WARNING]/' app-modified.log
diff app.log app-modified.log | head -10     # Verify the change

# 3. Show only lines from a specific IP
grep "10.0.1.5" app.log | tail -10
```

**Interview angle:** *"Given an nginx access log, how would you find the top 10 IPs hitting a 500 error?"* — you can answer this now.

---

## 🧪 Scenario 3 — Automated Backup Script With Cron

**What it covers:** Variables, functions, file operations, permissions, `find`, exit codes, `trap`, cron scheduling, structured logging — everything together

**The brief:** Write a backup script that archives a target directory, keeps only the last 3 backups, and is safe to schedule with cron.

### Step 1 — Create a fake app directory to back up

```bash
mkdir -p ~/myapp/{config,data,logs}

echo "db_host=prod-db.internal" > ~/myapp/config/database.yml
echo "port=8080"                >> ~/myapp/config/database.yml
echo "User data here"           > ~/myapp/data/users.db
touch ~/myapp/logs/app.log
```

### Step 2 — Write the backup script

```bash
mkdir -p ~/scripts/backup
nano ~/scripts/backup/backup.sh
```

```bash
#!/usr/bin/env bash
set -euo pipefail

# ── Config ────────────────────────────────────────────────────
SOURCE_DIR="${SOURCE_DIR:-$HOME/myapp}"       # What to back up (overridable via env)
BACKUP_DIR="${BACKUP_DIR:-/tmp/backups}"      # Where to store archives
KEEP_LAST=3                                   # How many backups to retain
LOGFILE="/tmp/backup.log"

# ── Logging ───────────────────────────────────────────────────
log() {
    local level="$1" message="$2" ts
    ts=$(date --utc +"%Y-%m-%d %H:%M:%S UTC")
    printf '[%s] [%-8s] %s\n' "$ts" "$level" "$message" | tee -a "$LOGFILE"
}
log_info()  { log "INFO"  "$1"; }
log_warn()  { log "WARN"  "$1"; }
log_error() { log "ERROR" "$1" >&2; }

# ── Cleanup ───────────────────────────────────────────────────
TMPDIR_WORK=""
cleanup() {
    local code=$?
    [[ -n "$TMPDIR_WORK" && -d "$TMPDIR_WORK" ]] && rm -rf "$TMPDIR_WORK"
    (( code == 0 )) \
        && log_info "Backup script finished successfully" \
        || log_error "Backup script failed with exit code $code"
}
trap cleanup EXIT

# ── Validation ────────────────────────────────────────────────
validate() {
    [[ -d "$SOURCE_DIR" ]] \
        || { log_error "Source directory not found: $SOURCE_DIR"; exit 1; }

    mkdir -p "$BACKUP_DIR"

    [[ -w "$BACKUP_DIR" ]] \
        || { log_error "Backup directory not writable: $BACKUP_DIR"; exit 1; }

    log_info "Validation passed — source: $SOURCE_DIR, destination: $BACKUP_DIR"
}

# ── Create backup ─────────────────────────────────────────────
create_backup() {
    local timestamp
    timestamp=$(date +"%Y%m%d-%H%M%S")
    local archive="$BACKUP_DIR/backup-${timestamp}.tar.gz"

    TMPDIR_WORK=$(mktemp -d)          # Temp working dir — cleaned up by trap

    log_info "Creating archive: $archive"

    tar -czf "$archive" -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"

    local size
    size=$(du -sh "$archive" | cut -f1)
    log_info "Archive created: $archive ($size)"
}

# ── Rotate old backups ────────────────────────────────────────
rotate_backups() {
    local count
    count=$(find "$BACKUP_DIR" -name "backup-*.tar.gz" | wc -l)
    log_info "Current backup count: $count (keeping last $KEEP_LAST)"

    if (( count > KEEP_LAST )); then
        local to_delete=$(( count - KEEP_LAST ))
        log_info "Removing $to_delete old backup(s)"

        # find: sorted by time (oldest first), delete the excess
        find "$BACKUP_DIR" -name "backup-*.tar.gz" \
            | sort \                           # Alphabetical = chronological (timestamp in name)
            | head -n "$to_delete" \           # Grab only the oldest ones
            | while IFS= read -r old; do
                log_info "Deleting old backup: $old"
                rm -f "$old"
            done
    fi
}

# ── Summary ───────────────────────────────────────────────────
print_summary() {
    log_info "── Backup summary ──"
    find "$BACKUP_DIR" -name "backup-*.tar.gz" \
        | sort \
        | while IFS= read -r f; do
            log_info "  $(du -sh "$f" | cut -f1)  $f"
        done
}

# ── Main ──────────────────────────────────────────────────────
main() {
    log_info "=== Backup started ==="
    validate
    create_backup
    rotate_backups
    print_summary
    log_info "=== Backup complete ==="
}

main
```

```bash
chmod 755 ~/scripts/backup/backup.sh
```

### Step 3 — Run it multiple times to trigger rotation

```bash
~/scripts/backup/backup.sh           # Run 1
sleep 2
~/scripts/backup/backup.sh           # Run 2
sleep 2
~/scripts/backup/backup.sh           # Run 3
sleep 2
~/scripts/backup/backup.sh           # Run 4 — should now delete the oldest

ls -lh /tmp/backups/                  # Should only see 3 archives
cat /tmp/backup.log                   # Review the full structured log
```

### Step 4 — Override via environment variables

```bash
# Test with a different source and destination without editing the script
SOURCE_DIR=/etc BACKUP_DIR=/tmp/etc-backups ~/scripts/backup/backup.sh
```

### Step 5 — Schedule it with cron

```bash
crontab -e
```

Add this line:
```bash
*/2 * * * * /usr/bin/env bash /root/scripts/backup/backup.sh >> /tmp/backup-cron.log 2>&1
```

```bash
# Wait 2 minutes then verify it ran
crontab -l                              # Confirm the entry is there
cat /tmp/backup-cron.log               # Output from cron run
ls -lh /tmp/backups/                   # Archives accumulating
```

Remove the cron job when done:
```bash
crontab -e                             # Delete the line you added
```

---

## What to Focus on While Doing These

As you work through each one, pause and ask yourself:

| Question | Where it applies |
|----------|-----------------|
| Why does `set -euo pipefail` matter here? | All three scripts |
| What happens if I remove the `trap`? | Scenario 1 & 3 |
| Why `"$@"` and not `$@`? | Scenario 1 & 3 |
| Can I break this with a filename that has spaces? | Scenario 3 |
| What does `$?` return after each check? | All three |
| Why does the cron job need absolute paths? | Scenario 3 |

---