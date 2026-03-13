# Week-1-Linux-for-DevOps-Basic-Shell-Scripting


---

## 1️⃣ Linux Command-Line Essentials (Core OS Interaction)

* Linux filesystem hierarchy
  (standard directory structure: `/etc`, `/var`, `/home`, `/opt`, `/tmp`, `/usr`, `/proc`, `/sys` etc.)

* Tree navigation & directory exploration
  (`cd`, `pwd`, `ls` variants, `tree`, `find`, `locate`)

* File & directory operations
  (`touch`, `mkdir`, `cp`, `mv`, `rm`, `ln` — hard vs symbolic links)

* Text processing & viewing tools
  (`cat`, `less`, `more`, `head`, `tail`, `grep`, `awk` basics, `sed` basics, `cut`, `sort`, `uniq`, `wc`)

---

## 2️⃣ File Permissions & Ownership (Security Foundation)

* Understanding `chmod` numeric vs symbolic mode
* User / group / other permissions (rwx breakdown)
* Special permissions (`setuid`, `setgid`, sticky bit)
* File ownership (`chown`, `chgrp`)
* `umask` (default permissions for new files/directories)
* Extended attributes & ACLs (overview of `getfacl`, `setfacl` — advanced but useful)

---

## 3️⃣ Process Management & Troubleshooting

* Process states (running, sleeping, zombie, stopped)
* Process hierarchy (parent-child relationships, `init`/`systemd` as PID 1)
* Listing & inspecting processes (`ps` variants: `aux`, `-ef`, tree view)
* `top`/`htop` usage & interpretation (CPU, memory, sorting, signals)
* Signals (`kill`, `pkill`, `SIGTERM` vs `SIGKILL`, `SIGUSR1`, etc.)
* Priority & `nice`/`renice` (CPU scheduling basics)
* Background & foreground jobs (`&`, `nohup`, `jobs`, `fg`, `bg`, `disown`)

---

## 4️⃣ System Monitoring & Resource Usage

* Disk usage & filesystem health (`df`, `du`, `fdisk`, `lsblk` basics)
* Memory & swap (`free`, `vmstat`, key `top` fields)
* CPU & I/O stats (`mpstat`, `iostat`, `sar` basics if available)
* Network connections (`ss`/`netstat`, `lsof` basics)
* Logging locations & fundamentals (`/var/log` structure, `syslog`/`rsyslog`, `journalctl` intro)
* Common bottleneck identification
  (high CPU, memory leak indicators, disk full scenarios)

---

## 5️⃣ Shell Scripting Basics (Theory Before Writing Code)

* Shebang & script execution
  (`#!/usr/bin/env bash` vs hard-coded path)

* POSIX vs Bash specifics
  (why `/usr/bin/env` is more portable)

* `set` options (`set -e`, `-u`, `-o pipefail`, `-x` for debugging)

* Command-line arguments
  (`$0`, `$1..$9`, `$@`, `$*`, `$#`)

* Wildcards & globbing
  (`*`, `?`, `[]`, brace expansion `{}`)

* Quoting rules
  (single vs double vs no quotes; variable expansion behavior)

* Input/Output redirection & pipes
  (`>`, `>>`, `<`, `<<`, `2>`, `&>`, `|`, `tee`)

* Exit codes & error handling
  (`$?`, basic `trap` usage)

* Environment variables vs local variables
  (`export`, `env`, `printenv`)

---

## 6️⃣ Bash Scripting Control Structures

* Variables & types
  (string, integer, basic arrays)

* Parameter expansion
  (`${var:-default}`, `${var:?error}`, length, substring)

* Conditionals
  (`if/elif/else`, `[[ ]]`, `[ ]`, legacy `test`)

* String & numeric comparisons
  (`==`, `!=`, `-eq`, `-gt`, `-lt`, `=~` regex)

* File tests
  (`-f`, `-d`, `-e`, `-r`, `-w`, `-x`, etc.)

* Loops
  (`for … in`, `while`, `until`, `break`, `continue`)

* `case` statement

* Functions
  (definition, `local` variables, return values, positional parameters inside functions)

---

## 7️⃣ Logging & Alerting Concepts (Theory for Projects)

* Structured logging
  (timestamp + level + message format)

* Real-time tailing & filtering
  (`tail -f` combined with `grep`)

* Basic alerting logic
  (threshold checks → trigger action)

* Cron job basics
  (scheduling theory; practical setup later)

---
