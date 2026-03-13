# Linux Module 6: Bash Scripting — Control Structures

Control structures are what transform a list of commands into **logic**. This module covers everything needed to write scripts that make decisions, repeat work, handle data, and encapsulate reusable logic into functions.

---

## 1️⃣ Variables & Types

Bash is loosely typed — variables have no enforced type by default. Everything is a string unless you tell bash otherwise.

### Declaring Variables

```bash
NAME="alice"                    # String — no spaces around =, ever
COUNT=42                        # Looks like an integer, but is still stored as a string
declare -i COUNT=42             # Explicitly declared integer — arithmetic is enforced
declare -r MAX=100              # Read-only — same as readonly MAX=100
declare -l username="ALICE"     # Lowercase — stored as "alice" automatically
declare -u status="active"      # Uppercase — stored as "ACTIVE" automatically
```

> ⚠️ `VAR = "value"` (with spaces) is a syntax error — bash interprets it as running a command named `VAR` with arguments `=` and `"value"`. Always write `VAR="value"` with no spaces.

### String Variables

```bash
GREETING="Hello, World"
EMPTY=""                        # Valid — explicitly empty string
MULTIWORD="hello world"         # Fine as long as you quote it when using it

echo "$GREETING"                # Always quote — prevents word splitting
echo "${GREETING}"              # Braces are optional here, but required when followed by more text
echo "${GREETING}!"             # Without braces: $GREETING! would look for variable GREETING!
```

### Integer Variables and Arithmetic

```bash
declare -i X=10
declare -i Y=3

# Arithmetic with $(( ))
RESULT=$((X + Y))               # Addition → 13
RESULT=$((X * Y))               # Multiplication → 30
RESULT=$((X / Y))               # Integer division → 3 (truncates)
RESULT=$((X % Y))               # Modulo (remainder) → 1
RESULT=$((X ** 2))              # Exponentiation → 100

# Increment / decrement
((COUNT++))                     # Increment in place
((COUNT--))                     # Decrement in place
((COUNT += 5))                  # Add 5 in place

# Arithmetic in conditions — (( )) returns 0 (true) if result is non-zero
if (( X > Y )); then
    echo "$X is greater"
fi
```

> 💡 `$(( ))` is for arithmetic **substitution** (you want the result). `(( ))` is for arithmetic **evaluation** (you want the true/false outcome, e.g. in an if statement). Both are bash built-ins — no external commands needed.

### Array Variables

```bash
# Indexed array — declare and populate
FRUITS=("apple" "banana" "cherry")        # Declare with values
FRUITS[3]="date"                          # Add a fourth element
declare -a SERVERS                        # Declare empty indexed array

# Accessing elements
echo "${FRUITS[0]}"                       # First element: apple
echo "${FRUITS[2]}"                       # Third element: cherry
echo "${FRUITS[@]}"                       # All elements: apple banana cherry date
echo "${#FRUITS[@]}"                      # Number of elements: 4
echo "${FRUITS[@]:1:2}"                   # Slice — elements 1 and 2: banana cherry

# Modifying arrays
FRUITS+=("elderberry")                    # Append an element
unset "FRUITS[1]"                         # Remove element at index 1 (leaves a gap — index 1 is now unset)
FRUITS=("${FRUITS[@]}")                   # Re-index to close gaps

# Iterating
for fruit in "${FRUITS[@]}"; do           # Always use "${array[@]}" — quotes preserve elements with spaces
    echo "Fruit: $fruit"
done
```

### Associative Arrays (Key-Value Maps)

```bash
declare -A CONFIG                         # Must explicitly declare with -A

CONFIG["host"]="prod-db.internal"
CONFIG["port"]="5432"
CONFIG["name"]="appdb"

echo "${CONFIG["host"]}"                  # prod-db.internal
echo "${!CONFIG[@]}"                      # All keys: host port name
echo "${CONFIG[@]}"                       # All values
echo "${#CONFIG[@]}"                      # Number of pairs: 3

# Iterating key-value pairs
for key in "${!CONFIG[@]}"; do
    echo "$key = ${CONFIG[$key]}"
done
```

> 💡 Associative arrays require `declare -A` — without it, bash creates a regular indexed array and silently discards the key names.

---

## 2️⃣ Parameter Expansion

Parameter expansion is bash's built-in mechanism for transforming variable values — defaults, error handling, substring extraction, and string manipulation — all without spawning external processes.

### Default Value Operators

```bash
# ${var:-default} — use default if var is unset OR empty
DB_HOST="${DB_HOST:-localhost}"           # If DB_HOST is unset or "", use "localhost"

# ${var-default} — use default only if var is unset (not if empty)
DB_PORT="${DB_PORT-5432}"                # If DB_PORT is "", keep ""; only use 5432 if unset

# ${var:=default} — assign default back to var if unset or empty
: "${LOG_DIR:=/var/log/myapp}"           # Sets LOG_DIR to /var/log/myapp AND keeps it for later use

# ${var:?error message} — exit with error if unset or empty
: "${API_KEY:?'API_KEY must be set before running this script'}"

# ${var:+alternate} — use alternate value only if var IS set and non-empty
DEBUG_FLAG="${VERBOSE:+--verbose}"       # If VERBOSE is set, DEBUG_FLAG="--verbose"; else ""
```

### Length and Substring

```bash
STR="Hello, World"

echo "${#STR}"                           # Length: 13
echo "${STR:7}"                          # Substring from index 7: World
echo "${STR:7:5}"                        # Substring from index 7, length 5: World
echo "${STR: -5}"                        # Last 5 characters: World  (space before - is required)
echo "${STR:0:5}"                        # First 5 characters: Hello
```

### String Removal (Pattern-Based)

These are among the most useful parameter expansions in real scripts:

```bash
FILEPATH="/var/log/nginx/access.log"

# Remove shortest match from the FRONT (# = front, like a hat)
echo "${FILEPATH#*/}"                    # var/log/nginx/access.log  (removed leading /)

# Remove longest match from the FRONT
echo "${FILEPATH##*/}"                   # access.log  (removed everything up to last /)

# Remove shortest match from the END (% = end, like a tail)
echo "${FILEPATH%.*}"                    # /var/log/nginx/access  (removed .log)

# Remove longest match from the END
echo "${FILEPATH%%/*}"                   # (empty — removed everything from first /)
```

> 💡 Memory trick: `#` removes from the front (like a git branch prefix). `%` removes from the end. Double `##` or `%%` = greedy (longest match). This is how you extract filenames, extensions, and directory paths without `basename` or `dirname`.

```bash
# Practical examples:
FILE="deploy-v1.2.3.tar.gz"
echo "${FILE##*.}"                       # gz — file extension
echo "${FILE%.*}"                        # deploy-v1.2.3.tar — strip last extension
echo "${FILE%%.*}"                       # deploy-v1 — strip all extensions
```

### String Replacement

```bash
STR="the cat sat on the mat"

echo "${STR/cat/dog}"                    # Replace first match: the dog sat on the mat
echo "${STR//at/ot}"                     # Replace ALL matches: the cot sot on the mot
echo "${STR/#the/a}"                     # Replace at START only: a cat sat on the mat
echo "${STR/%mat/rug}"                   # Replace at END only: the cat sat on the rug
```

### Case Conversion (Bash 4+)

```bash
STR="Hello World"

echo "${STR,,}"                          # All lowercase: hello world
echo "${STR^^}"                          # All uppercase: HELLO WORLD
echo "${STR,}"                           # Lowercase first char only: hELLO WORLD
echo "${STR^}"                           # Uppercase first char only: Hello World
```

---

## 3️⃣ Conditionals

### `if / elif / else` Structure

```bash
#!/usr/bin/env bash
set -euo pipefail

if [[ "$APP_ENV" == "prod" ]]; then
    echo "Production — full checks enabled"
elif [[ "$APP_ENV" == "staging" ]]; then
    echo "Staging — partial checks"
else
    echo "Development — minimal checks"
fi
```

### `[[ ]]` vs `[ ]` vs `test` — Which to Use

| Style | Name | Behaviour | Recommendation |
|-------|------|-----------|---------------|
| `[[ ]]` | Bash keyword | Safer, supports `&&`, `\|\|`, `=~`, no word splitting | ✅ Use this in bash scripts |
| `[ ]` | POSIX `test` | Portable, but variables must be quoted carefully | Use only in `#!/bin/sh` scripts |
| `test` | External command | Identical to `[ ]` | Rarely used directly |
| `(( ))` | Arithmetic | For integer comparisons only | Use for numbers |

```bash
# [[ ]] advantages over [ ]:

# No word splitting — safe even with unquoted variables (though still quote them)
[[ $FILENAME == *.log ]]          # Works — glob pattern matching built-in
[ $FILENAME == *.log ]            # WRONG — glob not expanded in [ ]

# Logical operators without escaping
[[ -f "$FILE" && -r "$FILE" ]]    # && and || work directly
[ -f "$FILE" -a -r "$FILE" ]      # Old style — -a and -o are deprecated

# Regex matching
[[ "$VERSION" =~ ^v[0-9]+\.[0-9]+ ]]   # =~ only works in [[ ]]
```

### Combining Conditions

```bash
# Logical AND — both must be true
if [[ -f "$CONFIG" && -r "$CONFIG" ]]; then
    echo "Config exists and is readable"
fi

# Logical OR — at least one must be true
if [[ "$ENV" == "dev" || "$ENV" == "staging" ]]; then
    echo "Non-production environment"
fi

# Negation
if [[ ! -d "$DEPLOY_DIR" ]]; then
    mkdir -p "$DEPLOY_DIR"
fi
```

---

## 4️⃣ String & Numeric Comparisons

### String Comparisons (inside `[[ ]]`)

| Operator | Meaning | Example |
|----------|---------|---------|
| `==` | Equal | `[[ "$A" == "$B" ]]` |
| `!=` | Not equal | `[[ "$A" != "$B" ]]` |
| `<` | Less than (lexicographic) | `[[ "$A" < "$B" ]]` |
| `>` | Greater than (lexicographic) | `[[ "$A" > "$B" ]]` |
| `-z` | String is empty (zero length) | `[[ -z "$VAR" ]]` |
| `-n` | String is non-empty | `[[ -n "$VAR" ]]` |
| `=~` | Matches extended regex | `[[ "$VER" =~ ^v[0-9]+ ]]` |

```bash
NAME="alice"

[[ "$NAME" == "alice" ]]         # true
[[ "$NAME" != "bob" ]]           # true
[[ -z "$EMPTY_VAR" ]]            # true if EMPTY_VAR is unset or ""
[[ -n "$NAME" ]]                 # true — NAME is non-empty
[[ "$NAME" == al* ]]             # true — glob pattern matching (no quotes on pattern)
[[ "$NAME" =~ ^[a-z]+$ ]]       # true — NAME is all lowercase letters (regex)
```

> 💡 When using `=~`, do **not** quote the regex pattern — quoting it forces a literal string comparison, disabling regex matching.

### Numeric Comparisons (inside `[[ ]]` or `(( ))`)

| Operator | Meaning | In `[[ ]]` | In `(( ))` |
|----------|---------|-----------|-----------|
| Equal | `==` | `-eq` | `==` |
| Not equal | `!=` | `-ne` | `!=` |
| Less than | `<` | `-lt` | `<` |
| Less or equal | `<=` | `-le` | `<=` |
| Greater than | `>` | `-gt` | `>` |
| Greater or equal | `>=` | `-ge` | `>=` |

```bash
COUNT=5

# Using [[ ]] with -eq style operators (for numbers)
if [[ "$COUNT" -gt 3 ]]; then echo "More than 3"; fi
if [[ "$COUNT" -eq 5 ]]; then echo "Exactly 5"; fi

# Using (( )) — cleaner for numeric logic
if (( COUNT > 3 )); then echo "More than 3"; fi
if (( COUNT >= 5 && COUNT <= 10 )); then echo "Between 5 and 10"; fi

# ⚠️ Never use < or > for numbers inside [[ ]] — they are lexicographic:
[[ "10" > "9" ]]     # FALSE — "1" < "9" lexicographically
(( 10 > 9 ))         # TRUE — correct numeric comparison
```

### Regex Matching with `=~`

```bash
VERSION="v1.23.4"
IP="192.168.1.100"
EMAIL="alice@example.com"

# Validate version format
if [[ "$VERSION" =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
    echo "Valid semver: $VERSION"
fi

# Capture groups — matched groups land in ${BASH_REMATCH[@]}
if [[ "$IP" =~ ^([0-9]+)\.([0-9]+)\.([0-9]+)\.([0-9]+)$ ]]; then
    echo "First octet: ${BASH_REMATCH[1]}"   # 192
    echo "Second octet: ${BASH_REMATCH[2]}"  # 168
fi

# Validate email format (basic)
if [[ "$EMAIL" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]; then
    echo "Valid email"
fi
```

---

## 5️⃣ File Tests

File tests check properties of files and directories before operating on them — essential for safe scripts.

### Common File Test Operators

| Operator | True when |
|----------|----------|
| `-e FILE` | File exists (any type) |
| `-f FILE` | Exists and is a regular file |
| `-d FILE` | Exists and is a directory |
| `-L FILE` | Exists and is a symbolic link |
| `-r FILE` | Exists and is readable |
| `-w FILE` | Exists and is writable |
| `-x FILE` | Exists and is executable |
| `-s FILE` | Exists and has size > 0 (non-empty) |
| `-z FILE` | File exists and is empty (size = 0) |
| `FILE1 -nt FILE2` | FILE1 is newer than FILE2 (by modification time) |
| `FILE1 -ot FILE2` | FILE1 is older than FILE2 |
| `FILE1 -ef FILE2` | FILE1 and FILE2 refer to the same inode (hard links) |

```bash
CONFIG="/etc/myapp/config.yml"

# Check before reading
if [[ ! -f "$CONFIG" ]]; then
    echo "Error: config file not found: $CONFIG" >&2
    exit 1
fi

if [[ ! -r "$CONFIG" ]]; then
    echo "Error: config file is not readable: $CONFIG" >&2
    exit 1
fi

# Check before writing
if [[ ! -d "$LOG_DIR" ]]; then
    mkdir -p "$LOG_DIR"
fi

if [[ ! -w "$LOG_DIR" ]]; then
    echo "Error: cannot write to log directory: $LOG_DIR" >&2
    exit 1
fi

# Check before executing
if [[ ! -x "$BINARY" ]]; then
    echo "Error: $BINARY is not executable" >&2
    exit 126           # 126 = command not executable (standard exit code)
fi

# Check if a file has content (non-empty)
if [[ -s "/tmp/errors.log" ]]; then
    echo "Errors were found — check /tmp/errors.log"
fi

# Check if two files are the same inode
if [[ "sites-enabled/app.conf" -ef "sites-available/app.conf" ]]; then
    echo "sites-enabled/app.conf is a hard link to sites-available/app.conf"
fi
```

### 🏭 Production-Grade Pattern — Pre-flight Checks

```bash
#!/usr/bin/env bash
set -euo pipefail

preflight_checks() {
    local errors=0

    # Required files
    [[ -f "/etc/myapp/config.yml" ]] || { echo "Missing: config.yml" >&2; ((errors++)); }
    [[ -f "/etc/myapp/secrets.env" ]] || { echo "Missing: secrets.env" >&2; ((errors++)); }

    # Required directories and permissions
    [[ -d "/var/log/myapp" && -w "/var/log/myapp" ]] || { echo "Log dir missing or not writable" >&2; ((errors++)); }
    [[ -d "/var/run/myapp" && -w "/var/run/myapp" ]] || { echo "Run dir missing or not writable" >&2; ((errors++)); }

    # Required binaries
    [[ -x "$(command -v docker)" ]] || { echo "docker not found or not executable" >&2; ((errors++)); }
    [[ -x "$(command -v jq)" ]] || { echo "jq not found" >&2; ((errors++)); }

    if (( errors > 0 )); then
        echo "$errors pre-flight check(s) failed. Aborting." >&2
        exit 1
    fi

    echo "All pre-flight checks passed."
}

preflight_checks
```

---

## 6️⃣ Loops

### `for ... in` — Iterate Over a List

```bash
# Iterate over a literal list
for ENV in dev staging prod; do
    echo "Checking $ENV..."
done

# Iterate over an array
SERVICES=("nginx" "postgres" "redis")
for SERVICE in "${SERVICES[@]}"; do
    echo "Restarting $SERVICE"
    systemctl restart "$SERVICE"
done

# Iterate over files (always use nullglob for safety)
shopt -s nullglob
for LOG in /var/log/myapp/*.log; do
    echo "Processing: $LOG"
    gzip "$LOG"
done
shopt -u nullglob

# C-style numeric loop
for (( i=1; i<=5; i++ )); do
    echo "Step $i of 5"
done

# Brace expansion loop
for i in {1..10}; do
    echo "Item $i"
done
```

### `while` — Loop While Condition Is True

```bash
# Count down
COUNT=5
while (( COUNT > 0 )); do
    echo "T-minus $COUNT"
    (( COUNT-- ))
done

# Read file line by line — the correct pattern
while IFS= read -r line; do      # IFS= prevents trimming leading/trailing whitespace
    echo "Line: $line"           # -r prevents backslash interpretation
done < /etc/hosts

# Wait for a service to become ready
MAX_WAIT=30
ELAPSED=0
while ! curl -sf "http://localhost:8080/health" &>/dev/null; do
    if (( ELAPSED >= MAX_WAIT )); then
        echo "Service did not start within ${MAX_WAIT}s" >&2
        exit 1
    fi
    echo "Waiting for service... (${ELAPSED}s)"
    sleep 2
    (( ELAPSED += 2 ))
done
echo "Service is ready"
```

> 💡 `while IFS= read -r line` is the canonical pattern for reading files line by line. Without `IFS=`, leading tabs and spaces are stripped. Without `-r`, backslashes are interpreted. Both omissions silently corrupt data.

### `until` — Loop Until Condition Becomes True

`until` is the inverse of `while` — it loops as long as the condition is **false**:

```bash
# Equivalent to the while health check above — sometimes reads more naturally
until curl -sf "http://localhost:8080/health" &>/dev/null; do
    echo "Waiting for service..."
    sleep 2
done
echo "Service is ready"
```

### `break` and `continue`

```bash
# break — exit the loop immediately
for FILE in /var/log/*.log; do
    if [[ "$(wc -l < "$FILE")" -gt 100000 ]]; then
        echo "File too large to process: $FILE"
        break                           # Stop processing — first oversized file found
    fi
    process_log "$FILE"
done

# continue — skip to the next iteration
for HOST in "${HOSTS[@]}"; do
    if ! ping -c1 -W1 "$HOST" &>/dev/null; then
        echo "Host unreachable: $HOST — skipping"
        continue                        # Don't deploy to unreachable host
    fi
    deploy_to "$HOST"
done

# break/continue with nested loops — use level numbers
for ENV in dev staging prod; do
    for SERVICE in nginx redis; do
        if [[ "$ENV" == "prod" && "$SERVICE" == "redis" ]]; then
            continue 2              # Skip to next iteration of the OUTER loop (level 2)
        fi
        echo "Deploy $SERVICE to $ENV"
    done
done
```

---

## 7️⃣ `case` Statement

`case` is cleaner than long `if/elif` chains when matching a variable against multiple patterns.

### Basic Syntax

```bash
case "$VARIABLE" in
    pattern1)
        # commands
        ;;          # ;; = break (end this branch)
    pattern2|pattern3)
        # commands — | separates multiple patterns for the same branch
        ;;
    *)
        # default — matches anything not caught above
        ;;
esac
```

### Practical Examples

```bash
# Environment routing
case "$APP_ENV" in
    prod)
        REPLICAS=5
        LOG_LEVEL="warn"
        ;;
    staging)
        REPLICAS=2
        LOG_LEVEL="info"
        ;;
    dev)
        REPLICAS=1
        LOG_LEVEL="debug"
        ;;
    *)
        echo "Unknown environment: $APP_ENV" >&2
        exit 2
        ;;
esac

# File type detection using glob patterns
case "$FILENAME" in
    *.tar.gz|*.tgz)    tar xzf "$FILENAME" ;;
    *.tar.bz2)         tar xjf "$FILENAME" ;;
    *.zip)             unzip "$FILENAME" ;;
    *.gz)              gunzip "$FILENAME" ;;
    *)
        echo "Unknown archive format: $FILENAME" >&2
        exit 1
        ;;
esac

# HTTP status code handling
case "$STATUS_CODE" in
    200|201|204)
        echo "Success"
        ;;
    301|302)
        echo "Redirect — check location header"
        ;;
    4[0-9][0-9])         # Glob pattern — matches 400-499
        echo "Client error: $STATUS_CODE"
        ;;
    5[0-9][0-9])         # Matches 500-599
        echo "Server error: $STATUS_CODE" >&2
        exit 1
        ;;
    *)
        echo "Unexpected status: $STATUS_CODE"
        ;;
esac
```

### `;&` and `;;&` — Fall-Through (Bash 4+)

```bash
case "$LEVEL" in
    debug)
        echo "Debug info"
        ;;&             # ;& = fall through to NEXT branch regardless
                        # ;;& = continue testing subsequent patterns
    info)
        echo "Info message"
        ;;
    *)
        echo "Unknown level"
        ;;
esac
# With ;;&, "debug" matches debug branch AND continues testing — will also match *
# This is rarely needed but useful for layered logging handlers
```

---

## 8️⃣ Functions

Functions let you name, reuse, and isolate blocks of logic. In bash, they behave like mini-scripts within your script.

### Definition Syntax

Both styles are valid — the `function` keyword is optional:

```bash
# Style 1 — with function keyword (bash-specific)
function greet() {
    echo "Hello, $1"
}

# Style 2 — POSIX-compatible (preferred)
greet() {
    echo "Hello, $1"
}
```

> Functions must be **defined before they are called** — bash reads the script top to bottom.

### Positional Parameters Inside Functions

Inside a function, `$1`, `$2`, `$@`, `$#` refer to the **function's arguments**, not the script's:

```bash
deploy() {
    local env="$1"           # Function's first argument
    local version="$2"       # Function's second argument
    echo "Deploying $version to $env"
}

deploy "staging" "v1.2.3"   # Call the function with arguments
```

### `local` Variables — Essential for Correctness

Without `local`, variables set inside a function pollute the global scope — a common source of hard-to-debug bugs:

```bash
STATUS="idle"

update_status() {
    STATUS="running"         # ⚠️ Modifies the GLOBAL STATUS — probably not intended
    local TEMP="working"     # ✅ local — scoped to this function only
}

update_status
echo "$STATUS"               # Prints "running" — global was changed
echo "$TEMP"                 # Prints "" — TEMP was local, not visible here
```

```bash
# Safe pattern — use local for ALL function-internal variables
process_file() {
    local filepath="$1"          # local copy of the argument
    local linecount              # declare local before assigning
    linecount=$(wc -l < "$filepath")
    echo "File $filepath has $linecount lines"
}
```

### Return Values

Bash functions can only `return` an integer (0–255), used as an **exit code**. To return a string, use command substitution:

```bash
# Return exit code (0=success, non-zero=failure)
is_valid_env() {
    local env="$1"
    case "$env" in
        dev|staging|prod) return 0 ;;    # 0 = true/success
        *) return 1 ;;                    # 1 = false/failure
    esac
}

if is_valid_env "$APP_ENV"; then
    echo "Valid environment"
else
    echo "Invalid environment: $APP_ENV" >&2
fi

# Return a string — use echo and capture with $()
get_timestamp() {
    echo "$(date +%Y%m%d-%H%M%S)"        # "Return" value via stdout
}

TIMESTAMP="$(get_timestamp)"             # Capture the output
echo "Timestamp: $TIMESTAMP"
```

### 🏭 Production-Grade Example — Full Deployment Script Structure

```bash
#!/usr/bin/env bash
set -euo pipefail

# ── Constants ────────────────────────────────────────────────
readonly DEPLOY_DIR="/var/www/myapp"
readonly LOG_FILE="/var/log/deploy.log"

# ── Logging helpers ──────────────────────────────────────────
log()  { echo "[$(date +%H:%M:%S)] INFO  $*" | tee -a "$LOG_FILE"; }
warn() { echo "[$(date +%H:%M:%S)] WARN  $*" | tee -a "$LOG_FILE" >&2; }
die()  {
    echo "[$(date +%H:%M:%S)] ERROR $*" | tee -a "$LOG_FILE" >&2
    exit 1
}

# ── Validation ───────────────────────────────────────────────
validate_args() {
    local env="$1"
    local version="$2"

    [[ -n "$env" ]]     || die "Environment cannot be empty"
    [[ -n "$version" ]] || die "Version cannot be empty"

    case "$env" in
        dev|staging|prod) ;;
        *) die "Invalid environment '$env'. Must be dev, staging, or prod" ;;
    esac

    [[ "$version" =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]] \
        || die "Invalid version '$version'. Must match vMAJOR.MINOR.PATCH"
}

# ── Health check ─────────────────────────────────────────────
wait_for_healthy() {
    local url="$1"
    local max_wait="${2:-60}"
    local elapsed=0

    log "Waiting for $url to become healthy..."
    until curl -sf "$url" &>/dev/null; do
        (( elapsed >= max_wait )) && die "Service not healthy after ${max_wait}s"
        sleep 3
        (( elapsed += 3 ))
    done
    log "Service is healthy after ${elapsed}s"
}

# ── Cleanup ──────────────────────────────────────────────────
cleanup() {
    local code=$?
    [[ -d "${TMPDIR:-}" ]] && rm -rf "$TMPDIR"
    (( code == 0 )) && log "Deploy completed successfully" \
                    || warn "Deploy exited with code $code"
}
trap cleanup EXIT

# ── Main ─────────────────────────────────────────────────────
main() {
    local env="$1"
    local version="$2"

    validate_args "$env" "$version"

    TMPDIR=$(mktemp -d)              # Temp dir — cleaned up by trap
    log "Starting deploy of $version to $env"

    # ... deployment steps ...

    wait_for_healthy "http://localhost:8080/health"
    log "Deploy of $version to $env complete"
}

main "$@"                            # Pass all script arguments to main
```

> Structuring scripts with a `main()` function called at the bottom is considered best practice — it keeps the script readable top-to-bottom, makes functions testable in isolation, and ensures `trap` is always registered before anything runs.

---

## Quick Reference

```bash
# Variables
VAR="string"                          # String
declare -i N=42                       # Integer
ARR=("a" "b" "c")                    # Indexed array
declare -A MAP                        # Associative array

# Parameter expansion
${VAR:-default}                       # Default if unset or empty
${VAR:?'error'}                       # Exit with error if unset
${#VAR}                               # String length
${VAR:2:4}                            # Substring: start=2, length=4
${VAR##*/}                            # Remove longest prefix up to /
${VAR%.*}                             # Remove shortest suffix from .
${VAR//old/new}                       # Replace all occurrences

# Comparisons
[[ "$A" == "$B" ]]                    # String equality
[[ "$A" =~ regex ]]                   # Regex match
(( A > B ))                           # Numeric comparison
[[ -f "$F" && -r "$F" ]]              # File exists and is readable

# Loops
for item in "${array[@]}"; do ...     # Array iteration
while IFS= read -r line; do ...       # File line-by-line
for (( i=0; i<n; i++ )); do ...       # C-style numeric

# Functions
func() { local var="$1"; ... }        # Define with local vars
result="$(func arg)"                  # Capture string return value
func arg && echo "success"            # Test exit code return value
```

---

## Key Takeaways

- `declare -i` for integers, `declare -A` for associative arrays — without these, bash treats everything as a string
- `$(( ))` evaluates arithmetic; `(( ))` tests arithmetic — know which returns a value vs a true/false
- `"${array[@]}"` always over `"${array[*]}"` — preserves element boundaries just like `"$@"` vs `"$*"`
- `${VAR##*/}` and `${VAR%.*}` extract filenames and extensions without forking `basename` or `dirname`
- `[[ ]]` over `[ ]` in bash scripts — safer, supports `&&`/`||`, glob patterns, and `=~`
- Never use `<` or `>` for numeric comparison inside `[[ ]]` — they are lexicographic; use `-lt`/`-gt` or `(( ))`
- `while IFS= read -r line` is the only correct pattern for reading files line by line
- `shopt -s nullglob` before any glob-based loop — prevents the literal `*.log` bug
- Every function variable should be `local` — global variable pollution causes the hardest bugs to trace
- Functions can only `return` 0–255; return strings via `echo` and capture with `$()`
- Structure scripts with a `main()` function — keeps logic testable, readable, and trap-safe

---