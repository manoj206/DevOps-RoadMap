# Linux Module 3: Processes & Job Control

Processes are **running instances of programs**.
Mastering how to find, inspect, control, prioritize, and terminate them is one of the most frequently used skill sets on Linux servers, development machines, and production debugging sessions.

---

## 1️⃣ Process States

Seen in the `STAT` column of `ps` and in tools like `top` / `htop`.

| State | Full Name             | Meaning                                          | Killable?                | Notes                              |
| ----- | --------------------- | ------------------------------------------------ | ------------------------ | ---------------------------------- |
| **R** | Running / Runnable    | Actively using CPU or waiting in run queue       | Yes                      | Ready to execute                   |
| **S** | Interruptible Sleep   | Waiting for event (I/O, timer, network)          | Yes                      | Normal sleeping state              |
| **D** | Uninterruptible Sleep | Waiting on kernel I/O (usually disk)             | No (until I/O completes) | Often indicates I/O bottleneck     |
| **T** | Stopped / Traced      | Paused via `SIGSTOP` or `Ctrl+Z`                 | No (until continued)     | Can be resumed                     |
| **Z** | Zombie                | Process finished; parent hasn’t collected status | No (harmless)            | Cleared when parent calls `wait()` |

### Notes

* Zombies consume almost no resources.
* Many zombies usually indicate a parent process that is not calling `wait()`.
* Orphaned processes are automatically adopted by PID 1 (`systemd`).

---

## 2️⃣ Process Hierarchy (Parent–Child Tree)

* Every process has a **PID** (Process ID)
* Every process has a **PPID** (Parent Process ID)
* **PID 1** = `systemd` (on modern Linux systems)
* If a parent dies, child processes become orphans and are adopted by PID 1

View hierarchy:

```bash
pstree -p
pstree -aup
systemd-cgls
ps -ef --forest
```

Understanding hierarchy is essential for debugging runaway workers and orphaned services.

---

## 3️⃣ Listing & Inspecting Processes

Common tools:

```bash
ps aux
ps -ef
ps -eF
ps xawf
pgrep -fl python
```

Sorting examples:

```bash
ps aux --sort=-%cpu | head -15
ps -eo pid,ppid,state,nice,%cpu,%mem,cmd --sort=-%mem | head
```

Live monitoring:

```bash
top
htop
btop
```

`htop` is generally preferred for clarity and interactive controls.

---

## 4️⃣ Signals

Signals are software interrupts used to control processes.

| Signal    | Number | Meaning                               | Catchable? |
| --------- | ------ | ------------------------------------- | ---------- |
| SIGTERM   | 15     | Graceful termination (default `kill`) | Yes        |
| SIGKILL   | 9      | Immediate forced termination          | No         |
| SIGINT    | 2      | Interrupt (Ctrl+C)                    | Yes        |
| SIGSTOP   | 19/17  | Stop immediately                      | No         |
| SIGCONT   | 18/19  | Resume stopped process                | —          |
| SIGTSTP   | 20/18  | Interactive stop (Ctrl+Z)             | Yes        |
| SIGHUP    | 1      | Hangup (often reload config)          | Varies     |
| SIGUSR1/2 | 10/12  | User-defined purposes                 | Varies     |

Useful commands:

```bash
kill -l
pkill -f "python.*worker"
killall -u user firefox
```

Always try `SIGTERM` first before using `SIGKILL`.

---

## 5️⃣ CPU Scheduling, Priority & Nice Values

Linux uses the **Completely Fair Scheduler (CFS)**.

### 🔹 Nice Value (User Hint)

Range:

```
-20  (highest priority)
  0  (default)
+19  (lowest priority)
```

Lower number = more CPU share
Higher number = more polite (wait longer)

Start process with nice value:

```bash
nice -n 10 ./long-task.sh
```

Change priority of running process:

```bash
renice 5 -p 1234
renice -n -5 -u username   # requires root
```

---

### 🔹 How Linux Actually Uses It (Dynamic Priority)

Linux does **not** schedule purely based on nice.

Instead:

* Each process accumulates *virtual runtime*
* CPU time is distributed proportionally
* Nice value influences scheduling weight
* Heavy CPU users gradually receive less CPU share
* Sleeping or interactive processes get preference when waking up

Nice values are converted into weights (approximate examples):

| Nice | Approx Weight |
| ---- | ------------- |
| -5   | ~3350         |
| 0    | ~1024         |
| +5   | ~335          |
| +10  | ~110          |

CPU time is distributed proportionally to weight.

Example scenario:

If two CPU-heavy processes run:

```bash
yes > /dev/null
nice -n 10 yes > /dev/null
```

The default nice (0) process will receive significantly more CPU time than the +10 process.

---

### Important Notes

* Nice affects **CPU scheduling only**
* It does NOT affect:

    * I/O priority
    * Memory allocation
    * Network bandwidth

For disk I/O priority:

```bash
ionice -c3 updatedb
```

---

## 6️⃣ Background & Foreground Jobs (Shell Job Control)

| Action              | Command                          |
| ------------------- | -------------------------------- |
| Run in background   | `command &`                      |
| Suspend foreground  | `Ctrl+Z`                         |
| List jobs           | `jobs`                           |
| Resume foreground   | `fg %1`                          |
| Resume background   | `bg %1`                          |
| Detach safely       | `nohup command > out.log 2>&1 &` |
| Remove job tracking | `disown %1`                      |

Safe long-task pattern:

```bash
nohup ./train-model.py --epochs 300 > train.log 2>&1 &
disown
```

---

## 7️⃣ Practice Commands

Safe to experiment:

```bash
sleep 600 &        # background
jobs
fg %1
Ctrl+Z
bg
pkill sleep

pstree -p | grep sleep
```

---

## Summary

This module covered:

* Process states and lifecycle
* Parent-child hierarchy
* Process inspection
* Signals
* Job control
* Nice values
* Dynamic priority under CFS
* Safe background execution

