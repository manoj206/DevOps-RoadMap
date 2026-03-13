# Module 6: Bash Scripting – Control Structures

---

# Golden Rules for Production Scripts

Always start scripts like this:

```bash
#!/usr/bin/env bash
set -euo pipefail
```

Explanation:

| Line                  | Meaning                               |
| --------------------- | ------------------------------------- |
| `#!/usr/bin/env bash` | Portable way to locate Bash           |
| `set -e`              | Exit immediately if any command fails |
| `set -u`              | Treat unset variables as errors       |
| `set -o pipefail`     | Fail pipelines if any command fails   |

---

# Preferred Modern Bash Style

Follow these conventions:

* Use `[[ ... ]]` instead of `[ ... ]`
* Quote variables → `"$var"`
* Use `local` inside functions
* Return meaningful exit codes (`0 = success`)
* Use `"$@"` when passing arguments
* Avoid unnecessary subshells when possible

---

# 1️⃣ Variables & Basic Types

Bash variables are **untyped** by default. Everything starts as a string.

Example:

```bash
#!/usr/bin/env bash
set -euo pipefail

name="Manoj"
echo "Hello $name"

count=42
((count++))
echo "Next: $((count + 10))"
```

### Explicit Integer Variables

```bash
declare -i number=075
echo $number
```

Output:

```
61
```

Because `075` is interpreted as **octal**.

---

# Arrays

### Indexed Arrays

```bash
files=("report1.txt" "data.csv" "backup.tar.gz")

echo "First: ${files[0]}"
echo "All: ${files[@]}"
echo "Count: ${#files[@]}"
```

### Associative Arrays (Bash 4+)

```bash
declare -A config
config["env"]="production"
config["log_level"]="debug"

echo "Env: ${config[env]}"
```

Analogy:
Variables are **sticky notes**. Bash lets you write text, numbers, or lists on them.

---

# 2️⃣ Parameter Expansion (Extremely Powerful)

Parameter expansion allows powerful variable manipulation.

```bash
backup_dir="${BACKUP_DIR:-/var/backups}"
logfile="${LOGFILE:=/var/log/app.log}"
db_host="${DB_HOST:?Database host is required}"
```

Meaning:

| Pattern           | Meaning              |
| ----------------- | -------------------- |
| `${var:-default}` | Use default if unset |
| `${var:=default}` | Assign default       |
| `${var:?error}`   | Fail if missing      |

---

### String Operations

```bash
echo "Length: ${#name}"

version="14.04.5"
echo "Major: ${version:0:2}"
```

---

### Path Manipulation

```bash
path="/home/manoj/projects/repo"

echo "Basename: ${path##*/}"
echo "Dirname: ${path%/*}"
```

---

Analogy:
Parameter expansion is like a **mini text-processing engine built into every variable**.

---

# 3️⃣ Conditionals – `if / elif / else`

Modern Bash style:

```bash
file="/etc/important.conf"

if [[ -f "$file" && -r "$file" ]]; then
    echo "Config is good"
elif [[ ! -e "$file" ]]; then
    echo "Creating default config..."
    echo "# Auto-generated" > "$file"
else
    echo "Permissions issue!" >&2
    exit 2
fi
```

### Why `[[ ]]` is Better

* Handles spaces safely
* Supports regex matching
* Supports logical operators (`&&`, `||`)
* Avoids word splitting issues

---

# 4️⃣ Comparison Cheat Sheet

| Type    | Operator | Meaning        | Example                          |
| ------- | -------- | -------------- | -------------------------------- |
| String  | `==`     | equal          | `[[ "$env" == "prod" ]]`         |
| String  | `!=`     | not equal      | `[[ "$env" != "dev" ]]`          |
| Regex   | `=~`     | regex match    | `[[ "$ip" =~ ^[0-9]+\.[0-9]+ ]]` |
| Numeric | `-eq`    | equal          | `[[ $count -eq 0 ]]`             |
| Numeric | `-lt`    | less than      | `[[ $age -lt 18 ]]`              |
| Numeric | `-gt`    | greater than   | `[[ $age -gt 18 ]]`              |
| File    | `-f`     | regular file   | `[[ -f "$path" ]]`               |
| File    | `-d`     | directory      | `[[ -d "$path" ]]`               |
| File    | `-r`     | readable       | `[[ -r "$file" ]]`               |
| File    | `-w`     | writable       | `[[ -w "$file" ]]`               |
| File    | `-x`     | executable     | `[[ -x "$script" ]]`             |
| File    | `-s`     | file not empty | `[[ -s "$log" ]]`                |

---

# 5️⃣ Loops

## `for` Loop (Most Common)

```bash
for file in *.log; do
    [[ -f "$file" ]] || continue
    gzip -9 "$file"
done
```

### Safe Array Loop

```bash
services=("nginx" "postgres" "redis")

for svc in "${services[@]}"; do
    systemctl restart "$svc"
done
```

---

## `while` Loop

```bash
count=0

while [[ $count -lt 10 ]]; do
    echo "Count: $count"
    ((count++))
done
```

---

## `until` Loop

```bash
until ping -c1 8.8.8.8 &>/dev/null; do
    echo "Waiting for internet..."
    sleep 5
done
```

---

### `break` and `continue`

| Command    | Purpose                |
| ---------- | ---------------------- |
| `break`    | Exit loop immediately  |
| `continue` | Skip current iteration |

---

### Reading Files Safely

```bash
while read -r line; do
    echo "Processing $line"
done < input.txt
```

`-r` prevents escape interpretation.

---

Analogy:
Loops are **conveyor belts**.

* `for` → fixed list
* `while` → keep working while condition holds
* `until` → wait until condition becomes true

---

# 6️⃣ `case` Statement

Cleaner alternative to many `elif` blocks.

```bash
action="${1:-help}"

case "$action" in
    start|up)
        docker compose up -d
        ;;
    stop|down)
        docker compose down
        ;;
    restart)
        "$0" stop && "$0" start
        ;;
    *)
        echo "Usage: $0 {start|stop|restart|status}"
        exit 1
        ;;
esac
```

Analogy:
`case` is like a **switchboard operator routing calls to the correct department**.

---

# 7️⃣ Functions – The Backbone of Clean Scripts

Example functions:

```bash
log_error() {
    local msg="$1"
    echo "[$(date --iso-8601=seconds)] ERROR: $msg" >&2
    return 1
}
```

---

### Check Open Port

```bash
is_port_open() {
    local port="$1"
    nc -z -w 2 localhost "$port" 2>/dev/null
}
```

---

### Backup Function

```bash
backup_directory() {
    local src="$1"
    local dest="$2"
    local stamp
    stamp=$(date +%Y%m%d-%H%M%S)

    if [[ ! -d "$src" ]]; then
        log_error "Source $src missing"
        return 1
    fi

    tar -czf "$dest/backup-${stamp}.tar.gz" -C "$src" .
    echo "Backup created: $dest/backup-${stamp}.tar.gz"
}
```

---

# Main Function Pattern

```bash
main() {
    backup_directory "/etc" "/backups" || exit 10
    is_port_open 5432 || { log_error "Postgres down"; exit 20; }
}

main "$@"
```

---

# Function Best Practices

| Rule                         | Reason                            |
| ---------------------------- | --------------------------------- |
| Use `local` variables        | Prevent global variable pollution |
| Pass arguments with `"$@"`   | Safe argument handling            |
| Return meaningful exit codes | Makes automation reliable         |

---

Analogy:
Functions are **machines on the factory floor**.

You feed them inputs → they perform work → return a status light.

---

# Summary

You now know how to use:

* Variables and arrays
* Parameter expansion
* Conditionals
* Comparisons
* Loops
* Case statements
* Functions
* Clean script structure

These tools are enough to build **professional Bash automation scripts used in real DevOps environments**.

---
