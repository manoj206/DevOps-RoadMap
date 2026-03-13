# Linux Module 3: Process Management & Troubleshooting

Processes are **running instances of programs**.
Mastering how to find, inspect, control, prioritize, and terminate them is one of the most frequently used skill sets on Linux servers, development machines, and production debugging sessions.

---

## 1️⃣ Process States

Every process on Linux is always in one of a handful of states. You see these in the `STAT` column of `ps` and in `top`/`htop`.

| State | Full Name | Meaning | Killable? |
|-------|-----------|---------|-----------|
| **R** | Running / Runnable | Actively using CPU or waiting in run queue | Yes |
| **S** | Interruptible Sleep | Waiting for an event — I/O, timer, network | Yes |
| **D** | Uninterruptible Sleep | Waiting on kernel-level I/O (usually disk) | ❌ Not until I/O completes |
| **T** | Stopped | Paused via `SIGSTOP` or `Ctrl+Z` | Not until resumed |
| **Z** | Zombie | Process finished; parent hasn't collected its exit status yet | ❌ Cannot be killed |

### State Modifiers (Additional Characters After the Main State)

`ps aux` often shows states like `Ss`, `R+`, `Sl`. The extra characters are modifiers:

| Modifier | Meaning |
|----------|---------|
| `s` | Session leader (e.g. a shell) |
| `+` | In the foreground process group |
| `l` | Multi-threaded |
| `<` | High priority (nice value is negative) |
| `N` | Low priority (nice value is positive) |

So `Ss` means: sleeping + session leader. `R+` means: running in the foreground.

### Important Nuances

**Zombie processes (`Z`):**
- Consume almost no resources — just a PID slot in the process table
- Cannot be killed with any signal (they are already dead)
- Cleared automatically when the parent calls `wait()` to collect the exit status
- Many zombies piling up usually means the **parent process has a bug** and is not calling `wait()`

**Uninterruptible Sleep (`D`):**
- Cannot be killed until the I/O completes — even `kill -9` has no effect
- A process stuck in `D` for a long time usually signals an **I/O bottleneck** — slow disk, hung NFS mount, or a storage issue
- Counts toward system load average even though it's not using CPU

**Orphaned processes:**
- If a parent dies before its children, the children become orphans
- Linux automatically re-parents orphans to **PID 1** (`systemd`)
- This is normal and harmless — systemd will collect their exit status properly

---

## 2️⃣ Process Hierarchy — Parent, Child, PID 1

Every process has:
- A **PID** (Process ID) — its own unique identifier
- A **PPID** (Parent Process ID) — the PID of whoever spawned it

This creates a tree rooted at **PID 1**.

### The Boot Chain

```
PID 1: systemd
  ├── sshd (listens for SSH connections)
  │     └── sshd (your session) ← spawned when you connected
  │           └── bash (your shell)
  │                 └── ps (the command you just ran)
  ├── nginx
  │     ├── nginx worker 1
  │     └── nginx worker 2
  └── cron
        └── backup.sh (currently running job)
```

### Viewing the Process Tree

```bash
pstree -p              # Show full tree with PIDs in parentheses
pstree -aup            # Show tree with user (-u), PIDs (-p), and arguments (-a)
ps -ef --forest        # ps output with ASCII tree lines showing parent-child relationships
ps -eo pid,ppid,cmd    # Show PID, PPID, and command — useful for tracing parentage manually
```

### Finding a Process's Parent

```bash
ps -o pid,ppid,cmd -p 1234    # Show PID, PPID, and command for a specific PID
cat /proc/1234/status          # Kernel's view — shows PPid: line among other details
```

> 💡 Understanding the process tree is essential for debugging runaway workers. If a web server spawns 50 children and one goes rogue, knowing the parent lets you kill the right process without taking down the entire service.

---

## 3️⃣ Listing & Inspecting Processes

### `ps` — The Core Tool

`ps` takes a snapshot of current processes. Two common formats:

```bash
ps aux          # BSD-style: all processes, with CPU/memory, user-oriented
ps -ef          # UNIX-style: all processes, with PPID, full command line
ps -eF          # Like -ef but adds extra columns (priority, nice, memory size)
```

### Understanding `ps aux` Columns

```bash
ps aux
# USER   PID  %CPU %MEM    VSZ   RSS TTY   STAT  START   TIME COMMAND
# alice  1234  45.2  2.1  98304 43520 pts/0  R+   10:00   0:03 python train.py
```

| Column | Meaning |
|--------|---------|
| `USER` | Who owns the process |
| `PID` | Process ID |
| `%CPU` | CPU usage since process started |
| `%MEM` | Percentage of physical RAM used |
| `VSZ` | Virtual memory size (includes everything mapped — often misleading) |
| `RSS` | Resident Set Size — actual physical RAM currently in use (more meaningful than VSZ) |
| `STAT` | Process state (see section 1) |
| `TIME` | Total accumulated CPU time |
| `COMMAND` | The command and its arguments |

> 💡 **RSS vs VSZ:** VSZ includes memory mapped but not yet loaded (shared libs, mmap'd files). RSS is what's actually in RAM right now. When diagnosing memory pressure, use RSS.

### Sorting and Filtering `ps`

```bash
ps aux --sort=-%cpu | head -10           # Top 10 CPU consumers (- means descending)
ps aux --sort=-%mem | head -10           # Top 10 memory consumers
ps -eo pid,ppid,stat,nice,%cpu,%mem,cmd --sort=-%mem | head -15   # Custom columns, sorted by memory
ps aux | grep nginx                      # Filter to just nginx processes
ps aux | grep -v grep | grep python      # Filter python processes, excluding the grep itself
```

### `pgrep` — Find PIDs by Name

```bash
pgrep nginx                   # List PIDs of all processes named nginx
pgrep -l nginx                # Include process name in output
pgrep -fl python              # -f matches against full command line (not just process name)
pgrep -u www-data             # All processes owned by user www-data
```

> 💡 `pgrep` is cleaner than `ps aux | grep` for scripts — it returns just PIDs, ready to pipe into `kill` or other commands.

### Inspecting a Specific Process via `/proc`

Every running process has a directory under `/proc/<PID>/`:

```bash
cat /proc/1234/cmdline | tr '\0' ' '   # Full command line (null-separated, so translate to spaces)
cat /proc/1234/status                  # State, PID, PPID, memory stats, threads
cat /proc/1234/environ | tr '\0' '\n'  # Environment variables the process started with
ls -l /proc/1234/fd                    # All open file descriptors (files, sockets, pipes)
cat /proc/1234/limits                  # Resource limits (open files, max processes, etc.)
```

> 💡 `/proc/<PID>/fd` is invaluable for debugging — it shows everything a process has open. Seeing unexpected file handles or sockets here often explains mysterious resource leaks.

---

## 4️⃣ `top` and `htop` — Live Monitoring

### `top` — Always Available

```bash
top               # Launch top
top -u www-data   # Show only processes owned by www-data
top -p 1234,5678  # Watch specific PIDs only
```

### Reading the `top` Header

```
top - 14:32:01 up 3 days, 2:11,  2 users,  load average: 1.42, 1.10, 0.95
Tasks: 210 total,   2 running, 207 sleeping,   0 stopped,   1 zombie
%Cpu(s): 23.4 us,  4.1 sy,  0.0 ni, 70.2 id,  2.1 wa,  0.0 hi,  0.2 si
MiB Mem:   7850.0 total,   512.3 free,  5421.1 used,  1916.6 buff/cache
MiB Swap:  2048.0 total,  1984.2 free,    63.8 used.  2130.4 avail Mem
```

**Load average** (`1.42, 1.10, 0.95`): average number of processes wanting CPU over the last 1, 5, and 15 minutes. On a 4-core machine, a load of 4.0 means fully utilized. Above your core count = the system is overloaded.

**CPU breakdown:**

| Field | Meaning |
|-------|---------|
| `us` | User space — your applications |
| `sy` | System/kernel — OS work |
| `ni` | Niced processes |
| `id` | Idle — free CPU |
| `wa` | I/O wait — CPU idle but waiting for disk/network |
| `hi` / `si` | Hardware/software interrupt handling |

> 💡 High `wa` (I/O wait) means your bottleneck is disk or network, not CPU. Adding more CPU won't help — you need faster I/O.

### `top` Interactive Keys

| Key | Action |
|-----|--------|
| `k` | Kill a process (prompts for PID then signal) |
| `r` | Renice a process |
| `P` | Sort by CPU (default) |
| `M` | Sort by memory |
| `T` | Sort by cumulative CPU time |
| `u` | Filter by user |
| `f` | Add/remove/reorder columns |
| `1` | Toggle per-CPU breakdown (multicore) |
| `q` | Quit |

---

### `htop` — The Better Alternative

`htop` gives everything `top` does but with colour, mouse support, and much easier interaction. Install it if not already present: `sudo apt install htop` or `sudo yum install htop`.

```bash
htop                    # Launch with full interactive UI
htop -u www-data        # Filter to a specific user
htop -p 1234,5678       # Watch specific PIDs
htop -d 5               # Refresh every 0.5 seconds (d = tenths of a second)
```

### Reading the `htop` Header

```
  CPU[|||||||||||          35.2%]    Tasks: 89, 142 thr; 2 running
  Mem[||||||||||||||  3.42G/7.65G]   Load average: 1.42 1.10 0.95
  Swp[              0K/2.00G    ]    Uptime: 3 days, 02:11:40
```

- CPU bar fills left to right. Green = user, red = kernel, blue = low priority (niced).
- Memory bar: green = used, blue = buffers, yellow = cache. Cache is not "wasted" — Linux uses free RAM as disk cache and releases it instantly when needed.

### `htop` Interactive Keys

| Key | Action |
|-----|--------|
| `F2` | Setup — change columns, colors, meters |
| `F3` / `/` | Search by process name |
| `F4` | Filter — show only matching processes |
| `F5` | Toggle tree view (shows parent-child hierarchy) |
| `F6` | Choose sort column |
| `F9` | Send signal to selected process |
| `F10` / `q` | Quit |
| `Space` | Tag a process (for bulk operations) |
| `u` | Filter by user |
| `t` | Toggle tree view |

> 💡 `htop` tree view (`F5` or `t`) is the fastest way to visually understand which worker processes belong to which parent — essential when a service spawns multiple children.

---

## 5️⃣ Signals

Signals are software interrupts sent to processes to notify them of events or instruct them to take action.

### Common Signals

| Signal | Number | Meaning | Catchable? | Typical Use |
|--------|--------|---------|-----------|-------------|
| `SIGTERM` | 15 | Graceful termination request | ✅ Yes | Default `kill` — gives the process time to clean up |
| `SIGKILL` | 9 | Immediate forced termination | ❌ No | Last resort — bypasses process entirely |
| `SIGINT` | 2 | Interrupt | ✅ Yes | What `Ctrl+C` sends |
| `SIGTSTP` | 20 | Interactive stop | ✅ Yes | What `Ctrl+Z` sends |
| `SIGSTOP` | 19 | Unconditional stop | ❌ No | Cannot be caught or ignored — always pauses |
| `SIGCONT` | 18 | Continue a stopped process | — | Resume after `SIGSTOP` or `SIGTSTP` |
| `SIGHUP` | 1 | Hangup | ✅ Yes | Originally: terminal closed. Now: reload config |
| `SIGUSR1/2` | 10/12 | User-defined | ✅ Yes | App-specific actions — e.g. rotate logs, dump stats |

```bash
kill -l       # List all signal names and their numbers
```

### `SIGTERM` vs `SIGKILL` — Why It Matters

`SIGTERM` (15) is a **request**. The process receives it, can run cleanup code (flush buffers, close DB connections, finish in-flight requests), then exits cleanly. Well-behaved services handle SIGTERM gracefully.

`SIGKILL` (9) is **not a request** — it's the kernel terminating the process immediately. The process never even sees it. No cleanup happens. This can cause:
- Corrupted files if writes were in progress
- Orphaned DB connections
- Lost in-flight work

> **Rule:** Always try `SIGTERM` first. Give the process a few seconds. Only escalate to `SIGKILL` if it refuses to stop.

### Sending Signals

```bash
kill 1234                  # Send SIGTERM (15) to PID 1234 — the default
kill -15 1234              # Same as above, explicit
kill -TERM 1234            # Same, using signal name
kill -9 1234               # Send SIGKILL — forced, no cleanup
kill -9 -1                 # ⚠️ SIGKILL to all your processes — use with extreme care
```

### `pkill` and `killall` — Kill by Name

```bash
pkill nginx                        # Send SIGTERM to all processes named nginx
pkill -9 nginx                     # Send SIGKILL to all processes named nginx
pkill -f "python.*worker"          # -f matches full command line (not just process name)
pkill -u alice                     # Terminate all processes owned by user alice

killall firefox                    # Send SIGTERM to all firefox processes
killall -u bob python              # Kill all python processes owned by bob
```

> 💡 `pkill -f` is extremely useful because it matches the full command line including arguments. `pkill python` would kill any process named `python`, but `pkill -f "python manage.py"` is surgical — it only kills Django management processes.

### `SIGHUP` — Config Reload Without Restart

Many daemons treat `SIGHUP` as "reload your config file" without fully restarting:

```bash
kill -HUP $(pgrep nginx)           # Reload nginx config without dropping connections
kill -1 $(pgrep sshd)              # Reload sshd config (same using signal number)
```

> This is how you apply config changes to a running service with zero downtime.

### `SIGUSR1` / `SIGUSR2` — Application-Specific Actions

```bash
kill -USR1 $(pgrep apache2)        # Tell Apache to rotate its log files
kill -USR1 $(pgrep -f gunicorn)    # Tell Gunicorn to gracefully reload workers
```

The exact behaviour of USR1/USR2 depends entirely on the application — always check its documentation.

---

## 6️⃣ Priority & Nice Values

### The Concept

Linux gives every process a **nice value** that hints to the scheduler how much CPU priority it should get.

```
-20  ←  highest priority (gets the most CPU)
  0  ←  default for all new processes
+19  ←  lowest priority (most polite — yields to everything else)
```

Lower number = more CPU. Higher number = more polite, gets less CPU.

> The name "nice" comes from the idea of a process being "nice" to other processes by voluntarily asking for less CPU time.

### Starting a Process With a Nice Value

```bash
nice -n 10 ./backup.sh              # Run with priority +10 — low priority, won't starve others
nice -n 19 ./index-builder.sh       # Lowest possible priority — runs in background scraps
nice -n -5 ./critical-task.sh       # Higher than default — requires root to go negative
```

### Changing Priority of a Running Process

```bash
renice 10 -p 1234                   # Set nice value of PID 1234 to +10
renice -n -5 -p 1234               # Decrease nice (increase priority) — requires root
renice 15 -u alice                  # Set all of alice's processes to nice +15
```

> Only root can set **negative** nice values (higher priority). Any user can make their own processes more polite (positive nice), but can't increase their own priority without `sudo`.

### How the Kernel Actually Uses It — CFS

Linux uses the **Completely Fair Scheduler (CFS)**. It doesn't simply pick the highest-priority process and run it exclusively. Instead:

- Each process accumulates **virtual runtime** (vruntime)
- The scheduler always picks the process with the **lowest vruntime** to run next
- Nice values are converted to **weights** that affect how fast vruntime accumulates

| Nice | Approximate Weight |
|------|--------------------|
| -20 | 88761 |
| -5 | 3121 |
| 0 | 1024 |
| +5 | 335 |
| +10 | 110 |
| +19 | 15 |

A process with nice `-5` (weight 3121) vs a process at nice `0` (weight 1024) will get roughly **3× more CPU** when both are runnable.

> 💡 **Key insight:** Nice values only matter when processes are **competing for CPU**. If the system is lightly loaded, a nice +19 process still runs at full speed — there's nothing else competing. Nice only kicks in under contention.

### I/O Priority — `ionice`

Nice values control **CPU scheduling only**. They do not affect disk I/O priority. For that, use `ionice`:

```bash
ionice -c 3 ./updatedb              # Class 3 = Idle — only runs I/O when disk is otherwise free
ionice -c 2 -n 4 ./backup.sh       # Class 2 = Best-effort, level 4 (0=highest, 7=lowest)
ionice -p 1234                      # Check current I/O class of a running process
```

| Class | Meaning |
|-------|---------|
| 1 | Realtime — highest, can starve others |
| 2 | Best-effort — default for most processes |
| 3 | Idle — only uses I/O when nothing else needs it |

> 💡 Always run long backup or indexing jobs with `ionice -c 3` on production servers — otherwise their disk I/O competes with your application and causes latency spikes.

---

## 7️⃣ Background & Foreground Jobs

### The Basics

By default, a command runs in the **foreground** — it holds your terminal until it finishes. Job control lets you manage multiple tasks in one shell session.

```bash
./long-task.sh &            # The & sends the process to the background immediately
                            # Shell prints: [1] 4823 — job number and PID
```

```bash
jobs                        # List all background jobs in this shell session
jobs -l                     # Same but include PIDs
```

Output:
```
[1]-  Running    ./backup.sh &
[2]+  Stopped    ./interactive-tool.sh
```

`+` = current job (default for `fg`/`bg`). `-` = previous job.

### Moving Between Foreground and Background

```bash
Ctrl+Z                      # Suspend (pause) the currently running foreground process — sends SIGTSTP
jobs                        # Confirm it's now listed as Stopped
bg %1                       # Resume job 1 in the background — sends SIGCONT
fg %1                       # Bring job 1 back to the foreground
fg                          # Bring the most recent job to foreground (no number needed)
```

### A Complete Job Control Workflow

```bash
./build.sh                  # Start a build in the foreground
# Ctrl+Z                    # Pause it — need the terminal for something else
jobs                        # Confirm: [1]+ Stopped ./build.sh
bg %1                       # Let the build continue in background
./test.sh                   # Run another task in the foreground
fg %1                       # Bring the build back when ready to watch it
```

---

### `nohup` — Survive Terminal Disconnect

Processes started in a shell are tied to that shell's **session**. When you disconnect from SSH or close the terminal, the shell sends `SIGHUP` to all its children — killing your background jobs.

`nohup` makes a process **immune to SIGHUP**:

```bash
nohup ./train-model.py > train.log 2>&1 &
#     ↑ immune to SIGHUP  ↑ stdout to file ↑ stderr → stdout ↑ background
```

- `> train.log` — redirect stdout to a file (required since there's no terminal)
- `2>&1` — redirect stderr into stdout (so both go to the log)
- `&` — send to background immediately

> Without `> train.log 2>&1`, nohup writes output to `nohup.out` in the current directory automatically — useful to know if you forget to redirect.

### `disown` — Detach a Job Already Running

If you forgot to use `nohup` and the process is already running:

```bash
./long-task.sh &            # Started in background but without nohup
jobs                        # Shows: [1]+ Running ./long-task.sh
disown %1                   # Remove job 1 from shell's job table
                            # Shell will no longer send SIGHUP to it on disconnect
jobs                        # Job is gone from the list — it's now fully detached
```

`disown` is your escape hatch when you realize mid-session that you forgot `nohup`.

### `nohup` vs `disown` — When to Use Which

| | `nohup` | `disown` |
|--|---------|---------|
| When | Before starting the process | After the process is already running |
| What it does | Ignores SIGHUP signal | Removes from shell's job table |
| Output | Redirected to `nohup.out` if not specified | Wherever it was already going |
| Typical use | Planned long-running job | Forgot nohup, process already started |

### 🏭 Production-Grade Pattern — Safe Long-Running Task

```bash
nohup ./data-pipeline.sh \          # Make immune to terminal disconnect
  > /var/log/pipeline.log \         # Capture stdout to a named log file
  2>&1 \                            # Merge stderr into stdout
  &                                 # Send to background

echo "Pipeline PID: $!"             # $! = PID of the last backgrounded command
                                    # Save this so you can monitor or kill it later

disown                              # Extra safety — detach from shell's job table too
```

```bash
# Later — check if it's still running
ps -p <PID>                         # Is that PID still alive?
tail -f /var/log/pipeline.log       # Watch its output live
```

---

## Quick Reference

```bash
# Process inspection
ps aux --sort=-%cpu | head -10      # Top CPU consumers
ps aux --sort=-%mem | head -10      # Top memory consumers
ps -ef --forest                     # Full tree with parent-child lines
pstree -p                           # Clean tree view with PIDs
pgrep -fl python                    # Find python processes with full command
cat /proc/<PID>/status              # Kernel-level process details

# Signals
kill -15 <PID>                      # Graceful termination (always try first)
kill -9 <PID>                       # Forced kill (last resort)
kill -HUP <PID>                     # Reload config
pkill -f "pattern"                  # Kill by matching full command line

# Priority
nice -n 10 ./task.sh                # Start with lower priority
renice 10 -p <PID>                  # Change priority of running process
ionice -c 3 ./backup.sh             # Idle I/O priority

# Job control
command &                           # Start in background
Ctrl+Z → bg %1                      # Suspend then resume in background
fg %1                               # Bring to foreground
nohup command > out.log 2>&1 &      # Survive disconnect
disown %1                           # Detach already-running job
```

---

## Key Takeaways

- Process state `D` (uninterruptible sleep) cannot be killed — it indicates an I/O problem, not a misbehaving process
- Zombies are harmless but many of them signal a parent with a `wait()` bug
- `RSS` is more meaningful than `VSZ` for diagnosing actual memory usage
- **Always try `SIGTERM` before `SIGKILL`** — forced kills skip cleanup and can corrupt state
- `SIGHUP` is a reload signal for daemons — use it to apply config changes without restarting
- `pkill -f` matches the full command line — far more surgical than `pkill` by process name alone
- Nice values only have effect under CPU **contention** — a nice +19 process still runs fast when the system is idle
- Nice controls CPU only — use `ionice -c 3` for background jobs that do heavy disk I/O
- `nohup` before starting + `disown` after = maximum insurance against losing a background job to a disconnect
- `$!` gives you the PID of the last backgrounded command — always save it for long-running tasks

---