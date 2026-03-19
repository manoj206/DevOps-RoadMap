# Linux Module 5: Shell Scripting Basics (Theory Before Writing Code)

This module focuses on **foundational concepts** that make the difference between fragile one-liners and reliable, production-grade automation scripts used in CI/CD, deployments, monitoring, and backups.

Production scripts in modern environments almost always include:
- `#!/usr/bin/env bash`
- `set -euo pipefail`
- Strict quoting
- Proper argument handling
- Meaningful exit codes and cleanup
- Clean I/O redirection

---

## 1️⃣ Shebang & Script Execution

The shebang (`#!`) tells the kernel **which interpreter to use** to run the file. It must be the **very first line** — not even a blank line before it.

```bash
#!/usr/bin/env bash
```

### `#!/usr/bin/env bash` vs `#!/bin/bash`

| Style | How it works | Risk | Recommendation |
|-------|-------------|------|---------------|
| `#!/bin/bash` | Hardcoded path to bash | Breaks on systems where bash lives elsewhere (e.g. `/usr/local/bin/bash` on some BSDs, NixOS, Homebrew environments) | Avoid unless you control every target system |
| `#!/usr/bin/env bash` | Asks `env` to find `bash` in `$PATH` | A different bash version might be found if PATH is unusual | ✅ Best practice — portable across most environments |
| `#!/bin/sh` | Uses the system's POSIX shell | May be `dash`, `ash`, or `busybox sh` — not bash | Only use when writing strictly POSIX-compatible scripts |

> 💡 `/usr/bin/env` is a small utility whose sole job is to find a program in `$PATH` and execute it. It is almost universally located at `/usr/bin/env` across all Unix-like systems — which is why it's safe to hardcode *that* path.

### Script Execution Methods

```bash
chmod +x deploy.sh          # Make the script executable first
./deploy.sh                 # Execute using the shebang interpreter
bash deploy.sh              # Explicitly use bash — shebang is ignored (good for testing)
source deploy.sh            # Run in the CURRENT shell session (no subshell spawned)
. deploy.sh                 # Same as source — POSIX-compatible shorthand
```

### Why `source` is Different

When you run `./deploy.sh`, it spawns a **child process** (subshell). Variables set inside die when the script exits. When you `source` a script, it runs **in your current shell** — any variables or functions it sets persist in your session.

```bash
# This is how shell config files work:
source ~/.bashrc             # Reload your bash config without opening a new terminal
. ~/.profile                 # Same — all functions and exports land in your current shell
```

> ⚠️ Avoid `sh deploy.sh` — `sh` may invoke `dash`, `ash`, or `busybox sh` depending on the system, none of which support Bash-specific syntax. If your script uses `[[`, arrays, or brace expansion, it will silently fail or produce wrong results under `sh`.

---

## 2️⃣ POSIX `sh` vs Bash

`#!/bin/sh` means "I commit to being POSIX-compatible and will not use any Bash-specific features." Unless you have a strong reason for that commitment, default to Bash.

### Features That Are Bash-Only (Will Break Under `sh`)

| Feature | Bash | POSIX `sh` |
|---------|------|-----------|
| `[[ ]]` double bracket tests | ✅ | ❌ — use `[ ]` |
| Arrays (`arr=(a b c)`) | ✅ | ❌ |
| Associative arrays (`declare -A`) | ✅ | ❌ |
| Brace expansion (`{1..10}`) | ✅ | ❌ |
| Process substitution `<(cmd)` | ✅ | ❌ |
| `=~` regex matching in `[[ ]]` | ✅ | ❌ |
| `local` keyword in functions | ✅ | ❌ in strict POSIX (works in most `sh` impls but not guaranteed) |
| `readonly` and `declare` | ✅ | Limited |

### The Practical Rule

> Default to `#!/usr/bin/env bash`. Only drop to `#!/bin/sh` if your script must run on minimal embedded systems, Docker `scratch` images, or environments where bash is genuinely not available.

---

## 3️⃣ `set` Options — Strict Mode

These three lines together form the **standard production Bash header**:

```bash
#!/usr/bin/env bash
set -euo pipefail
```

### What Each Option Does

| Option | Long form | What it does | Why it matters |
|--------|-----------|-------------|----------------|
| `-e` | `set -e` | Exit immediately if any command returns a non-zero exit code | Prevents cascading failures — without this, a failed step is silently skipped and the script keeps going |
| `-u` | `set -u` | Treat unset variables as errors | Catches typos in variable names before they cause data loss (e.g. `rm -rf "$DIIR/"` would expand to `rm -rf "/"` without `-u`) |
| `-o pipefail` | — | The entire pipeline fails if **any** command in it fails | Without this, `false \| true` succeeds — the failure of `false` is hidden by `true` succeeding |
| `-x` | `set -x` | Print each command before executing it (debug mode) | Shows exactly what the shell is running, with all variables expanded |

### The Danger Without `pipefail`

```bash
# Without pipefail — this silently succeeds even though grep failed:
grep "critical" /var/log/app.log | mail -s "Alert" ops@company.com
# If the log file doesn't exist, grep exits 1 — but the pipeline exit code is mail's (0)
# You never get the alert, and the script doesn't stop

# With set -o pipefail — grep's failure propagates:
set -o pipefail
grep "critical" /var/log/app.log | mail -s "Alert" ops@company.com
# grep fails → pipeline fails → script stops (with set -e active)
```

### The Danger Without `-u`

```bash
# A typo in a variable name — catastrophic without -u:
DEPLOY_DIR="/var/www/myapp"
rm -rf "$DEPOY_DIR/"          # Typo: DEPOY instead of DEPLOY
# Without -u: DEPOY_DIR is unset, expands to empty string
# This runs: rm -rf "/"      ← deletes your entire filesystem

# With set -u: script immediately exits with:
# bash: DEPOY_DIR: unbound variable
```

### Debug Mode — `set -x`

```bash
set -x           # Turn on — every subsequent command is printed with a '+' prefix before running
./deploy.sh
set +x           # Turn off — stop printing commands

# Or run an entire script in debug mode from outside:
bash -x deploy.sh
```

Output with `-x` looks like:
```
+ ENVIRONMENT=staging
+ VERSION=v1.2.3
+ echo 'Deploying v1.2.3 to staging'
Deploying v1.2.3 to staging
```

> 💡 Use `bash -x` on a failing script as your first debugging step — seeing the exact command that ran with variables fully expanded usually reveals the problem immediately.

### Combining Options

```bash
set -euxo pipefail     # All four at once — maximum strictness + debug output
set -eo pipefail       # Standard production (without debug noise)
set +e                 # Temporarily disable exit-on-error (useful around commands expected to fail)
```

#### Temporarily Disabling `-e` for an Expected Failure

```bash
set -eo pipefail

# This command might legitimately return non-zero (e.g. grep returns 1 when no match found)
set +e
grep "pattern" file.txt
GREP_STATUS=$?
set -e

if [ "$GREP_STATUS" -eq 0 ]; then
    echo "Pattern found"
elif [ "$GREP_STATUS" -eq 1 ]; then
    echo "Pattern not found — normal"
else
    echo "grep failed with unexpected error" >&2
    exit 1
fi
```

---

## 4️⃣ Command-Line Arguments

When a script is called with arguments, bash populates a set of special variables automatically.

### Special Variables

| Variable | Meaning | Example |
|----------|---------|---------|
| `$0` | Script name (as it was invoked) | `./deploy.sh` or `/usr/local/bin/deploy.sh` |
| `$1` – `$9` | Positional arguments | `$1` = first argument, `$2` = second, etc. |
| `${10}` and beyond | Arguments past 9 | Must use braces: `${10}`, `${11}` |
| `$#` | Number of arguments passed | `3` if three arguments were given |
| `$@` | All arguments as **separate words** | `"$@"` — each argument is its own quoted string |
| `$*` | All arguments as **one string** | `"$*"` joins all args with first char of `$IFS` (usually space) |
| `$?` | Exit code of last command | `0` = success, non-zero = failure |
| `$$` | PID of the current script | Useful for creating unique temp files: `/tmp/myapp.$$` |
| `$!` | PID of last backgrounded command | Save this to monitor or kill a background job |

### `"$@"` vs `"$*"` — The Critical Difference

```bash
# Script called with: ./script.sh "hello world" "foo bar"

for arg in "$@"; do echo "$arg"; done
# Output:
# hello world       ← treated as one argument (correct)
# foo bar           ← treated as one argument (correct)

for arg in "$*"; do echo "$arg"; done
# Output:
# hello world foo bar   ← all merged into one string (almost always wrong)
```

> Always use `"$@"` when passing arguments to another command or iterating over them. `"$*"` is rarely the right choice.

### Argument Validation Pattern

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
    cat <<EOF
Usage:
    $0 <environment> <version>

Arguments:
    environment    Target environment (dev | staging | prod)
    version        Version tag to deploy (e.g. v1.2.3 or latest)

Examples:
    $0 staging v1.2.3
    $0 prod latest
EOF
    exit 1
}

# Validate argument count
if [ "$#" -ne 2 ]; then
    echo "Error: expected 2 arguments, got $#" >&2
    usage
fi

ENVIRONMENT="$1"
VERSION="$2"

# Validate environment value
case "$ENVIRONMENT" in
    dev|staging|prod) ;;                            # Valid — do nothing, fall through
    *)
        echo "Error: unknown environment '$ENVIRONMENT'" >&2
        usage
        ;;
esac

echo "Deploying $VERSION to $ENVIRONMENT"
```

### `shift` — Consuming Arguments One at a Time

```bash
while [ "$#" -gt 0 ]; do
    case "$1" in
        --env)
            ENVIRONMENT="$2"
            shift 2             # Consume both --env and its value
            ;;
        --version)
            VERSION="$2"
            shift 2
            ;;
        --dry-run)
            DRY_RUN=true
            shift               # Consume just this flag
            ;;
        *)
            echo "Unknown argument: $1" >&2
            exit 1
            ;;
    esac
done
```

### `getopts` — POSIX-style Flag Parsing

```bash
while getopts "e:v:d" opt; do
    case "$opt" in
        e) ENVIRONMENT="$OPTARG" ;;   # -e staging  — : means takes an argument
        v) VERSION="$OPTARG" ;;       # -v v1.2.3
        d) DRY_RUN=true ;;            # -d  — no : means it's a boolean flag
        ?) echo "Unknown option" >&2; exit 1 ;;
    esac
done

shift $((OPTIND - 1))    # Remove processed options — remaining args are now $1, $2, etc.
```

Usage: `./deploy.sh -e prod -v v1.2.3 -d`

> `getopts` only handles single-character flags (`-e`, `-v`). For long flags (`--env`, `--version`), use the `while/case/shift` pattern above or the external `getopt` utility.

---

## 5️⃣ Wildcards & Globbing

Globbing is **filename expansion** performed by the shell before the command sees its arguments. It is not regex.

### Wildcard Characters

| Pattern | Matches | Example |
|---------|---------|---------|
| `*` | Any string of characters (including empty) | `*.log` matches `app.log`, `error.log`, `access.log` |
| `?` | Exactly one character | `file?.txt` matches `file1.txt`, `fileA.txt` but not `file10.txt` |
| `[abc]` | Any one character in the set | `file[123].txt` matches `file1.txt`, `file2.txt`, `file3.txt` |
| `[a-z]` | Any one character in the range | `report[0-9].md` matches `report1.md` through `report9.md` |
| `[^abc]` | Any one character NOT in the set | `file[^0-9].txt` matches `fileA.txt` but not `file1.txt` |
| `{a,b,c}` | Brace expansion — each value | `{dev,staging,prod}` expands to three separate words |
| `{1..5}` | Brace expansion — numeric range | `file{1..5}.txt` → `file1.txt file2.txt ... file5.txt` |

### Globbing in Practice

```bash
ls *.log                          # All files ending in .log
ls /var/log/nginx/*.log           # All nginx logs
cp config.{yml,yml.bak}           # Expands to: cp config.yml config.yml.bak — quick backup
mkdir -p environments/{dev,staging,prod}/{config,logs,data}  # Create entire directory tree at once
touch report{1..5}.md             # Create report1.md through report5.md
echo file?.txt                    # Lists file1.txt, fileA.txt — any single char
```

### Brace Expansion — No Filesystem Required

Unlike `*` and `?`, brace expansion does **not** require matching files to exist. It just generates strings:

```bash
echo {a,b,c}.conf                 # a.conf b.conf c.conf — even if no files exist
mkdir {2022,2023,2024}-{01..12}  # Creates 36 directories: 2022-01, 2022-02, ..., 2024-12
```

### Globbing Inside Scripts — the `nullglob` Problem


## Understanding `nullglob` in Bash

When working with wildcards like `*.log` in shell scripts, Bash has a subtle behavior that can cause unexpected bugs. The `nullglob` option changes this behavior to make scripts safer.

---

## The Example

```bash
# Iterate over files (always use nullglob for safety)
shopt -s nullglob

for LOG in /var/log/myapp/*.log; do
    echo "Processing: $LOG"
    gzip "$LOG"
done

shopt -u nullglob
```

---

## What’s Actually Happening

The pattern:

```bash
/var/log/myapp/*.log
```

is expanded by the shell **before** the loop runs.

---

## Default Behavior (without `nullglob`)

### When files exist

If the directory contains:

```bash
app.log
server.log
```

The shell expands to:

```bash
/var/log/myapp/app.log /var/log/myapp/server.log
```

The loop runs normally.

---

### When no files exist ❌

If there are **no `.log` files**, the shell does this:

```bash
/var/log/myapp/*.log   →   "/var/log/myapp/*.log"
```

Yes — it keeps the pattern **as-is**.

So your loop becomes:

```bash
for LOG in /var/log/myapp/*.log
```

Which runs **once**, with:

```bash
LOG="/var/log/myapp/*.log"
```

Then your script tries:

```bash
gzip /var/log/myapp/*.log
```

Result:

```bash
gzip: /var/log/myapp/*.log: No such file or directory
```

---

## With `nullglob` Enabled ✅

```bash
shopt -s nullglob
```

This tells Bash:

> If a wildcard matches nothing, return **nothing** instead of the pattern.

---

### When no files exist

```bash
/var/log/myapp/*.log   →   (empty)
```

Now the loop becomes:

```bash
for LOG in
```

Which means:

* The loop runs **zero times**
* No errors
* Clean behavior

---

## Why This Matters

Without `nullglob`, your script can:

* Process fake filenames
* Throw misleading errors
* Behave unpredictably in empty directories

With `nullglob`, your script:

* Only runs when real files exist
* Avoids edge-case bugs
* Becomes safer and more predictable

---

## Mental Model

Think of wildcard expansion like a fishing net:

| Scenario        | Default Behavior   | With `nullglob` |
| --------------- | ------------------ | --------------- |
| Matches files   | Return files       | Return files    |
| Matches nothing | Return the pattern | Return nothing  |

---

## Why Turn It Off?

```bash
shopt -u nullglob
```

Shell options are **global to the current session**.
Good practice is to reset them after use to avoid side effects in other parts of your script.

---

## Quick Demo

```bash
echo *.log
```

### Without `nullglob`

```bash
*.log
```

### With `nullglob`

```bash
# (no output)
```

---

## Bonus Tip

There’s a stricter alternative:

```bash
shopt -s failglob
```

This will:

* Throw an error if a wildcard matches nothing
* Prevent silent failures

Useful when you want **fail-fast behavior** instead of silent skipping.

---


```bash
# Default behaviour — if no files match, the glob is passed literally:
for f in *.txt; do
    echo "$f"           # If no .txt files exist, prints literally: *.txt
done

# Fix with nullglob — unmatched globs expand to nothing:
shopt -s nullglob
for f in *.txt; do
    echo "$f"           # If no .txt files exist, the loop simply doesn't run
done
shopt -u nullglob       # Turn off after use to avoid unexpected behaviour elsewhere
```
## Takeaway

If your script depends on wildcard expansion, enabling `nullglob` is a small change that prevents subtle and frustrating bugs.
> ⚠️ Always use `shopt -s nullglob` before glob-based loops in scripts. Without it, the loop will process the literal string `*.txt` if no files match — silently corrupting your logic.


---

## 6️⃣ Quoting Rules

Quoting errors are **the most common class of Bash bugs**. The rule is simple: quote everything that could contain spaces, special characters, or be empty.

### The Three Quoting Modes

| Style | Variable Expansion | Glob Expansion | Use Case |
|-------|-------------------|---------------|----------|
| `'single quotes'` | ❌ No | ❌ No | Literal strings — nothing is interpreted |
| `"double quotes"` | ✅ Yes | ❌ No | Variables and command substitution — globs suppressed |
| No quotes | ✅ Yes | ✅ Yes | Word splitting and globbing happen — almost always dangerous |

### The Classic Quoting Bug

```bash
filename="Important Report (Final).pdf"

rm $filename            # WRONG — shell splits on spaces, runs: rm Important Report (Final).pdf
                        # That's trying to delete 4 separate arguments, none of which exist as named

rm "$filename"          # CORRECT — the whole thing is one argument: "Important Report (Final).pdf"
```

### Single vs Double — When to Use Which

```bash
echo 'Hello $USER'           # Prints literally: Hello $USER  (no expansion)
echo "Hello $USER"           # Prints: Hello alice  (variable expanded)
echo "Today is $(date +%F)"  # Prints: Today is 2024-03-10  (command substitution works inside "")
echo 'No $(expansion) here'  # Prints literally: No $(expansion) here
```

### The `--` Convention — Protecting Against `-` Filenames

```bash
file="-rf important-data"

rm "$file"       # Still dangerous — rm receives -rf as a flag, then important-data as path
rm -- "$file"    # CORRECT — -- signals end of options; everything after is a filename
```

> `--` works with most Unix commands (`rm`, `grep`, `mv`, `cp`). It's a good habit any time a filename could start with a `-`.

### Nested Quoting in Command Substitution

```bash
# Command substitution inside double quotes — works correctly:
result="$(grep "pattern" "$filename")"    # Both the outer "" and inner "" are handled correctly
                                           # Bash parses nested quotes in $() independently

# Same with backticks — requires escaping (another reason to prefer $() ):
result="`grep \"pattern\" \"$filename\"`"  # Harder to read and error-prone
```

> Always use `$(...)` for command substitution, never backticks. `$(...)` nests cleanly and is easier to read.

### The Practical Quoting Rule

> **Quote everything by default.** Only leave things unquoted when you specifically need word splitting or glob expansion — and document why.

```bash
# These should always be quoted:
cp "$source" "$destination"
mkdir -p "$base_dir/$env/logs"
[[ "$APP_ENV" == "prod" ]]

# Intentionally unquoted — we WANT glob expansion:
shopt -s nullglob
for log in /var/log/myapp/*.log; do
    process_log "$log"      # The variable itself is still quoted
done
```

---

## 7️⃣ Input / Output Redirection & Pipes

Every process has three standard streams:
- **stdin** (`0`) — input
- **stdout** (`1`) — normal output
- **stderr** (`2`) — error output

### Redirection Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| `>` | Redirect stdout to file (overwrite) | `echo "ok" > status.txt` |
| `>>` | Redirect stdout to file (append) | `echo "log" >> app.log` |
| `<` | Redirect file to stdin | `sort < unsorted.txt` |
| `2>` | Redirect stderr to file | `cmd 2> errors.log` |
| `2>>` | Append stderr to file | `cmd 2>> errors.log` |
| `&>` | Redirect both stdout and stderr to file | `script &> all.log` |
| `2>&1` | Redirect stderr into stdout stream | `cmd > out.log 2>&1` |
| `>/dev/null` | Discard stdout | `cmd > /dev/null` |
| `2>/dev/null` | Discard stderr (suppress error messages) | `find / -name x 2>/dev/null` |

### `2>&1` — Order Matters

```bash
# CORRECT — redirect stdout to file first, then merge stderr into stdout:
cmd > out.log 2>&1          # Both stdout and stderr go to out.log

# WRONG — stderr still goes to terminal:
cmd 2>&1 > out.log          # stderr merges into stdout (terminal), THEN stdout redirected to file
                             # Counterintuitive — the order of redirections is left-to-right
```

### Here-Documents (`<<EOF`)

A here-document feeds a multi-line string as stdin to a command — without needing a separate file:

```bash
cat <<EOF
Server: $HOSTNAME
Environment: $APP_ENV
Deployed by: $(whoami)
EOF
# Variables and command substitution work inside a regular heredoc

cat <<'EOF'
This is literal: $HOSTNAME $(whoami)
No expansion happens inside single-quoted heredoc delimiter
EOF
```

Use `<<'EOF'` (single-quoted delimiter) when you want literal text with no expansion — common for generating scripts or config files that themselves contain `$` characters.

### Here-String (`<<<`)

```bash
grep "error" <<< "no errors here"      # Feed a single string as stdin — no file needed
wc -w <<< "count these words"          # Quick way to pass a string to a command expecting stdin
```

### `tee` — Write to File AND See Output Live

```bash
./build.sh | tee build.log              # Output appears on terminal AND is saved to build.log
./deploy.sh 2>&1 | tee deploy.log       # Capture both stdout and stderr, show and save both
./test.sh | tee -a test.log             # -a = append to log instead of overwriting
```

### Redirect All Script Output to a Log

```bash
#!/usr/bin/env bash
set -euo pipefail

exec > >(tee -a /var/log/deploy.log) 2>&1    # From this line on, ALL output goes to both
                                               # terminal and the log file — stdout and stderr
echo "Deploy started at $(date)"
```

> `exec > >(...)` permanently redirects the script's stdout for the rest of its execution. This is the cleanest way to add logging to an entire script without wrapping every command.

### `/dev/null` — The Discard Device

```bash
cmd > /dev/null             # Discard stdout — suppress normal output
cmd 2> /dev/null            # Discard stderr — suppress error messages
cmd &> /dev/null            # Discard everything — run silently
cmd > /dev/null 2>&1        # Same — older, more portable form
```

---

## 8️⃣ Exit Codes & Error Handling

Every command exits with a numeric code. `0` always means success. Any non-zero value means failure.

### `$?` — The Exit Code of the Last Command

```bash
grep "pattern" file.txt
echo $?              # 0 if pattern was found, 1 if not found, 2 if file didn't exist

cp source.txt dest.txt
if [ $? -ne 0 ]; then
    echo "Copy failed" >&2
    exit 1
fi

# Better pattern — test the command directly:
if ! cp source.txt dest.txt; then
    echo "Copy failed" >&2
    exit 1
fi
```

> Prefer `if ! command; then` over running a command and then checking `$?`. By the time you check `$?`, another command may have run in between and overwritten it.

### Standard Exit Code Conventions

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | General error |
| `2` | Misuse of shell builtins (e.g. wrong arguments) |
| `126` | Command found but not executable |
| `127` | Command not found |
| `128+n` | Killed by signal `n` (e.g. `130` = killed by `Ctrl+C`, which is SIGINT=2, so 128+2) |

### Explicit Exit Codes

```bash
#!/usr/bin/env bash
set -euo pipefail

if [ "$#" -ne 2 ]; then
    echo "Usage: $0 <env> <version>" >&2
    exit 2           # 2 = usage error (distinct from runtime error)
fi

if ! ping -c1 "$DB_HOST" &>/dev/null; then
    echo "Cannot reach database host: $DB_HOST" >&2
    exit 1           # 1 = runtime error
fi

echo "All checks passed"
exit 0               # Explicit success (optional — falling off the end also exits 0)
```

### `trap` — Guaranteed Cleanup

`trap` registers commands to run when the script exits or receives a signal — even if it exits due to an error.

```bash
#!/usr/bin/env bash
set -euo pipefail

TMPDIR="/tmp/deploy.$$"       # $$ = this script's PID — unique temp directory

cleanup() {
    local exit_code=$?                        # Capture the exit code before it gets overwritten
    echo "Cleaning up $TMPDIR..." >&2
    rm -rf "$TMPDIR"                          # Remove temp files regardless of how we exited
    exit "$exit_code"                         # Preserve the original exit code
}

trap cleanup EXIT             # Run cleanup() on ANY exit — success, failure, or signal

trap 'echo "Interrupted" >&2; exit 130' INT TERM   # Handle Ctrl+C and kill gracefully

mkdir -p "$TMPDIR"
# ... rest of script — TMPDIR will always be cleaned up
```

### `trap` Signals Worth Knowing

| Signal | When it fires | Typical use |
|--------|--------------|-------------|
| `EXIT` | On any exit — normal or error | Cleanup temp files, release locks |
| `INT` | `Ctrl+C` | Cancel gracefully |
| `TERM` | `kill` sent to script | Stop gracefully |
| `ERR` | Any command returns non-zero (with `set -e`) | Log the failing line |

```bash
trap 'echo "Error on line $LINENO" >&2' ERR    # $LINENO tells you exactly where it failed
```

---

## 9️⃣ Environment Variables vs Local Variables

### The Three Variable Types

| Type | Syntax | Scope |
|------|--------|-------|
| Local (shell) variable | `VAR="value"` | Current shell only — child processes cannot see it |
| Exported (environment) variable | `export VAR="value"` | Visible to current shell AND all child processes it spawns |
| Read-only variable | `readonly VAR="value"` | Cannot be changed or unset — good for constants |

```bash
NAME="alice"                    # Local — only this shell sees it
export DB_HOST="prod-db.internal"  # Exported — subshells and child processes inherit it
readonly MAX_RETRIES=3          # Read-only — attempting to change it causes an error
```

### Subshell Inheritance

```bash
SECRET="hunter2"               # Local variable
export APP_ENV="prod"          # Exported variable

bash -c 'echo "SECRET=$SECRET"'      # Prints: SECRET=  (empty — not exported, child can't see it)
bash -c 'echo "APP_ENV=$APP_ENV"'    # Prints: APP_ENV=prod (exported — child sees it)
```

### Inspecting Environment Variables

```bash
env                             # Print all exported environment variables
printenv                        # Same — slightly more portable
printenv APP_ENV                # Print value of one specific variable
printenv | sort | grep "^APP"   # Filter exported vars by prefix — useful for checking config
set                             # Print ALL variables (local + exported) and functions — very verbose
declare -p VAR                  # Show a specific variable's value, type, and flags
```

### Default Values — The `:-` Operator

```bash
# Use a default if the variable is unset or empty:
DB_HOST="${DB_HOST:-localhost}"        # If DB_HOST is unset, use "localhost"
APP_ENV="${APP_ENV:-staging}"          # If APP_ENV is unset, use "staging"
LOG_LEVEL="${LOG_LEVEL:-info}"

# Use a default only if unset (not if empty):
DB_PORT="${DB_PORT-5432}"             # - without : — only triggers if truly unset, not if ""

# Fail with a message if unset:
API_KEY="${API_KEY:?'API_KEY must be set'}"    # Script exits with the message if API_KEY is unset
```

### 🏭 Production-Grade Pattern — Config Validation at Script Start

```bash
#!/usr/bin/env bash
set -euo pipefail

# Validate all required environment variables before doing anything
: "${APP_ENV:?'APP_ENV must be set (dev|staging|prod)'}"
: "${DB_HOST:?'DB_HOST must be set'}"
: "${API_KEY:?'API_KEY must be set'}"

# Apply safe defaults for optional variables
LOG_LEVEL="${LOG_LEVEL:-info}"
MAX_RETRIES="${MAX_RETRIES:-3}"
DEPLOY_TIMEOUT="${DEPLOY_TIMEOUT:-300}"

# Validate APP_ENV value
case "$APP_ENV" in
    dev|staging|prod) ;;
    *) echo "Error: APP_ENV must be dev, staging, or prod — got '$APP_ENV'" >&2; exit 2 ;;
esac

echo "Config validated. Deploying to $APP_ENV"
```

> The `: "${VAR:?...}"` pattern uses the `:` no-op command to trigger the `?` expansion — if `VAR` is unset, the script immediately exits with your error message. This is the cleanest way to enforce required environment variables at the top of a script.

---

## Quick Reference

```bash
# Header
#!/usr/bin/env bash
set -euo pipefail

# Arguments
$0 $1 $2    # script name, first arg, second arg
$#          # argument count
"$@"        # all args, separately quoted — always use this over "$*"
shift       # consume $1, shift all others down

# Quoting
"$var"      # always quote variables
'literal'   # no expansion at all
-- "$file"  # protect against filenames starting with -

# Redirection
cmd > out.log 2>&1          # both stdout and stderr to file
cmd 2>/dev/null             # suppress errors
cmd | tee out.log           # show output AND save it

# Exit codes
$?                          # exit code of last command
exit 0                      # explicit success
exit 1                      # general failure
trap cleanup EXIT           # run cleanup on any exit

# Variables
export VAR="val"            # make visible to child processes
readonly CONST="val"        # cannot be changed
VAR="${VAR:-default}"       # use default if unset
VAR="${VAR:?'must be set'}" # exit with error if unset
printenv VAR                # print exported variable
```

---

## Key Takeaways

- `#!/usr/bin/env bash` is portable; `#!/bin/bash` is fragile on non-standard systems
- Never use `sh script.sh` on a Bash script — silent failures when `sh` is `dash`
- `set -euo pipefail` is non-negotiable in production scripts — add it to every script, no exceptions
- `"$@"` preserves argument boundaries; `"$*"` collapses them — always use `"$@"`
- Quote all variables by default — unquoted variables with spaces cause silent, hard-to-debug failures
- `--` signals end of flags — use it with `rm`, `mv`, `grep` when filenames could start with `-`
- Redirect order matters: `> file 2>&1` is correct; `2>&1 > file` leaves stderr on the terminal
- `trap cleanup EXIT` guarantees cleanup runs even if the script crashes mid-execution
- `${VAR:?'message'}` is the cleanest way to enforce required environment variables at script start
- `$(...)` over backticks — nests cleanly, easier to read, no escaping headaches

---