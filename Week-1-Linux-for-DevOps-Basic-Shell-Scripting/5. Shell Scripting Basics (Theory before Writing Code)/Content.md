# Linux Module 5: Shell Scripting Basics (Theory Before Writing Code)

This module focuses on **foundational concepts** that make the difference between fragile one-liners and reliable, production-grade automation scripts (used in CI/CD, deployment, monitoring, backups, etc.).

Production scripts in modern environments almost always include:

* `#!/usr/bin/env bash`
* `set -euo pipefail`
* Strict quoting
* Proper argument handling
* Meaningful exit codes and cleanup
* Clean I/O redirection

---

# 1️⃣ Shebang & Script Execution

The shebang (`#!`) **must be the very first line** of the script.

```bash
#!/usr/bin/env bash
```

### Why `/usr/bin/env bash` instead of `#!/bin/bash`?

| Style                 | Pros                        | Cons                                         | Recommendation        |
| --------------------- | --------------------------- | -------------------------------------------- | --------------------- |
| `#!/bin/bash`         | Explicit path               | Breaks on systems where Bash lives elsewhere | Avoid unless required |
| `#!/usr/bin/env bash` | Uses `$PATH` to locate Bash | Slightly slower startup                      | **Best practice**     |

### Correct Execution Methods

```bash
chmod +x myscript.sh
./myscript.sh staging v1.2.3

bash myscript.sh           # ignores shebang (good for testing)
source myscript.sh         # runs in current shell
```

⚠️ Avoid:

```bash
sh myscript.sh
```

Because `sh` may be `dash` or `ash`, which lacks Bash features.

---

# 2️⃣ POSIX sh vs Bash

Use `#!/bin/sh` only if your script is **strictly POSIX-compatible**.

Examples of **Bash-only features**:

* Arrays
* `[[ ]]`
* Brace expansion `{1..10}`
* Process substitution `<(cmd)`

Modern rule:

> Default to **Bash** unless strict portability is required.

---

# 3️⃣ Strict Mode — The Three Most Important Lines

```bash
#!/usr/bin/env bash
set -euo pipefail
```

| Option            | Meaning                             | Why It Matters              |
| ----------------- | ----------------------------------- | --------------------------- |
| `set -e`          | Exit immediately if a command fails | Prevents cascading failures |
| `set -u`          | Error on undefined variables        | Prevents typos              |
| `set -o pipefail` | Pipeline fails if any command fails | Prevents silent failures    |

Example failure without pipefail:

```bash
false | true
```

Without `pipefail` → success
With `pipefail` → failure

### Debug Mode

```bash
set -x
set +x
```

Or run:

```bash
bash -x myscript.sh
```

---

# 4️⃣ Command-Line Arguments

| Variable  | Meaning                        | Correct Quoting  |
| --------- | ------------------------------ | ---------------- |
| `$0`      | Script name                    | Usually unquoted |
| `$1`–`$9` | Positional arguments           | `"$1"`           |
| `$@`      | All arguments (separate words) | `"$@"`           |
| `$*`      | All arguments as one string    | Rarely correct   |
| `$#`      | Number of arguments            | Unquoted         |

### Example: Argument Validation

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
    cat << 'EOF'
Usage:
    ./deploy.sh <environment> <version>

Examples:
    ./deploy.sh staging v1.2.3
    ./deploy.sh prod latest
EOF
    exit 1
}

if [ $# -ne 2 ]; then
    usage
fi

ENVIRONMENT="$1"
VERSION="$2"

case "$ENVIRONMENT" in
    dev|staging|prod)
        ;;
    *)
        echo "Unknown environment: $ENVIRONMENT" >&2
        usage
        ;;
esac

echo "Deploying $VERSION to $ENVIRONMENT"
```

---

# 5️⃣ Quoting Rules

Quoting errors are **the most common Bash bugs**.

| Quotes     | Expansion | Protection                |
| ---------- | --------- | ------------------------- |
| `'single'` | No        | Strongest protection      |
| `"double"` | Yes       | Recommended for variables |
| none       | Yes       | Dangerous                 |

Example bug:

```bash
file="Important Report (Final).pdf"

rm $file
```

Correct version:

```bash
rm -- "$file"
```

The `--` prevents filenames beginning with `-` from being interpreted as flags.

---

# 6️⃣ Input / Output Redirection & Pipes

| Symbol  | Meaning                  | Example                |        |                |
| ------- | ------------------------ | ---------------------- | ------ | -------------- |
| `>`     | overwrite file           | `echo ok > file.txt`   |        |                |
| `>>`    | append                   | `echo log >> file.log` |        |                |
| `<`     | input redirection        | `command < input.txt`  |        |                |
| `<<EOF` | here-document            | inline input           |        |                |
| `2>`    | redirect stderr          | `cmd 2> errors.log`    |        |                |
| `&>`    | redirect stdout + stderr | `script &> log.txt`    |        |                |
| `       | tee`                     | display + log          | `build | tee build.log` |

### Example: Log Everything

```bash
exec > >(tee -a deploy.log) 2>&1
```

All script output goes to both terminal and log.

---

# 7️⃣ Exit Codes & Error Handling

Standard exit codes:

| Code  | Meaning                |
| ----- | ---------------------- |
| `0`   | success                |
| `1`   | general error          |
| `126` | command not executable |
| `127` | command not found      |
| `130` | Ctrl+C interruption    |

### Example Cleanup with `trap`

```bash
cleanup() {
    local code=$?
    echo "Cleaning up..."
    rm -rf "/tmp/myapp.$$"
    exit "$code"
}

trap cleanup EXIT
```

This ensures cleanup runs even if the script fails.

---

# 8️⃣ Environment vs Local Variables

| Type               | Example                | Scope                      |
| ------------------ | ---------------------- | -------------------------- |
| Local variable     | `var="value"`          | current script             |
| Exported variable  | `export VAR="value"`   | visible to child processes |
| Read-only variable | `readonly VAR="value"` | cannot change              |

### Example: Default Environment Variable

```bash
export APP_ENV="${APP_ENV:-staging}"

if [[ "$APP_ENV" != "prod" && "$APP_ENV" != "staging" ]]; then
    echo "Invalid APP_ENV value"
    exit 2
fi
```

---

# 9️⃣ Additional Helpful Bash Concepts

### Command Substitution

```bash
current_user="$(whoami)"
today="$(date +%F)"
```

---

### Safe File Loop

```bash
while read -r line; do
    echo "Processing: $line"
done < input.txt
```

`-r` prevents escape interpretation.

---

### Basic `getopts` Argument Parsing

```bash
while getopts "e:v:" opt; do
  case $opt in
    e) ENV="$OPTARG" ;;
    v) VERSION="$OPTARG" ;;
  esac
done
```

Example:

```bash
./deploy.sh -e prod -v v1.2.3
```

---

# Summary

This module introduced the foundations of **reliable Bash scripting**:

* Shebang portability
* Strict error handling (`set -euo pipefail`)
* Argument parsing
* Safe quoting
* I/O redirection
* Exit codes and traps
* Environment variable management

These patterns prevent many real-world failures in deployment scripts, automation pipelines, and system administration tasks.

---