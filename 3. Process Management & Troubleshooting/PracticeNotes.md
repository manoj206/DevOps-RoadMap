# Background Process Control Examples

---

## 1️⃣ Running a Long Process Safely in Background

### Command

```bash
nohup python train_model.py --epochs 500 > train.log 2>&1 &
disown
```

### What It Does

* `nohup`
  Prevents the process from terminating when the terminal session closes (ignores `SIGHUP`).

* `python train_model.py --epochs 500`
  Runs a long training job.

* `> train.log`
  Redirects standard output (stdout) to `train.log`.

* `2>&1`
  Redirects standard error (stderr) to the same file as stdout.

* `&`
  Runs the process in the background.

* `disown`
  Removes the process from the shell’s job control table (fully detaches it from the current session).

### Result

* The training continues even if:

    * SSH disconnects
    * Terminal closes
* All logs are written to `train.log`
* The process runs independently of the shell session

To monitor:

```bash
tail -f train.log
```

To verify running:

```bash
ps aux | grep train_model.py
```

---

## 2️⃣ Job Control & Process Management Example

### Step 1: Start a Background Process

```bash
sleep 600 &
```

Output example:

```
[1] 23847
```

* `[1]` → job number
* `23847` → process ID (PID)

---

### Step 2: List Background Jobs

```bash
jobs
```

Example output:

```
[1]+  Running  sleep 600 &
```

---

### Step 3: Bring Job to Foreground

```bash
fg %1
```

Moves job number `1` to foreground.
Terminal becomes occupied by the process again.

---

### Step 4: Suspend Process

Press:

```
Ctrl + Z
```

Example output:

```
[1]+  Stopped  sleep 600
```

This sends `SIGTSTP` (pause signal).

---

### Step 5: Resume in Background

```bash
bg
```

Example output:

```
[1]+ sleep 600 &
```

Process resumes execution in background.

---

### Step 6: Kill the Process

```bash
pkill sleep
```

Sends `SIGTERM` to all processes named `sleep`.

Verify:

```bash
jobs
```

Example:

```
[1]+  Terminated  sleep 600
```

---

### Step 7: View Process Tree

Start another sleep process:

```bash
sleep 600 &
```

Then inspect:

```bash
pstree -p | grep sleep
```

Example output:

```
bash(23100)─┬─sleep(24122)
```

* `bash(23100)` → parent shell process
* `sleep(24122)` → child process

Shows parent-child relationship.

---

## Summary of Commands

| Command       | Purpose                              |
| ------------- | ------------------------------------ |
| `sleep 600 &` | Run process in background            |
| `jobs`        | List current shell jobs              |
| `fg %1`       | Bring job #1 to foreground           |
| `Ctrl + Z`    | Suspend foreground process           |
| `bg`          | Resume stopped process in background |
| `pkill sleep` | Terminate process by name            |
| `pstree -p`   | View process hierarchy with PIDs     |

---
