# Linux Command-Line Essentials (Core OS Interaction)

Everything in Linux revolves around the **command line (terminal)**.
This section covers the foundational skills needed to confidently navigate, manage files, and process text in Linux.

---

## 1️⃣ Linux Filesystem Hierarchy (Standard Directory Structure)

Linux uses a single **upside-down tree** structure starting from the **root directory (`/`)**.

There are **no drive letters** (C:, D:, etc.).
Everything exists under `/`.

| Directory | Purpose                             | Analogy                                   |
| --------- | ----------------------------------- | ----------------------------------------- |
| `/`       | Root of entire filesystem           | Foundation of the house                   |
| `/home`   | User home directories               | Personal bedrooms (`/home/username`)      |
| `/etc`    | System-wide configuration files     | Control panel / brain                     |
| `/var`    | Variable data (logs, DBs, cache)    | Storage room (`/var/log`)                 |
| `/tmp`    | Temporary files                     | Trash bin (often cleared on reboot)       |
| `/usr`    | User programs & libraries           | Toolbox — most installed software         |
| `/opt`    | Optional third-party software       | Garage — large manually installed apps    |
| `/proc`   | Virtual filesystem (process info)   | Live X-ray of the running system          |
| `/sys`    | Virtual filesystem (kernel/devices) | Electrical wiring                         |
| `/boot`   | Bootloader & kernel files           | Starter motor                             |
| `/dev`    | Device files                        | Plugs & sockets (`/dev/null`, `/dev/sda`) |
| `/bin`    | Essential user binaries             | Basic tools (`ls`, `cp`, `cat`)           |
| `/sbin`   | Admin binaries                      | System-level tools                        |

### Quick Check

```bash
ls /
```

---

## 2️⃣ Tree Navigation & Directory Exploration

These commands help you move and explore efficiently.

| Command  | Purpose                    | Examples                                            |
| -------- | -------------------------- | --------------------------------------------------- |
| `pwd`    | Print current directory    | `pwd`                                               |
| `cd`     | Change directory           | `cd /etc`, `cd ..`, `cd -`, `cd ~`, `cd ~/Projects` |
| `ls`     | List contents              | `ls -la`, `ls -lh`, `ls -R`                         |
| `tree`   | Visual directory structure | `tree -L 2`, `tree -a`                              |
| `find`   | Search files               | `find ~ -name "*.pdf"`                              |
| `locate` | Fast indexed search        | `locate nginx.conf`                                 |

> Tip: Run `sudo updatedb` periodically so `locate` stays updated.

---

## 3️⃣ File & Directory Operations

| Command | Action                   | Examples                                          |
| ------- | ------------------------ | ------------------------------------------------- |
| `touch` | Create empty file        | `touch notes.txt`, `touch file{1..5}.txt`         |
| `mkdir` | Create directory         | `mkdir -p projects/web/css`                       |
| `cp`    | Copy files/directories   | `cp file.txt /tmp/`, `cp -r folder/ backup/`      |
| `mv`    | Move or rename           | `mv old.txt new.txt`, `mv file.txt ~/backup/`     |
| `rm`    | Remove files/directories | `rm file.txt`, `rm -r folder/`, `rm -rf temp/` ⚠️ |
| `ln`    | Create links             | `ln original hardlink`, `ln -s original symlink`  |

### Hard Link vs Symbolic Link

| Hard Link                              | Symbolic Link                      |
| -------------------------------------- | ---------------------------------- |
| Points to same inode (same data)       | Points to file path                |
| Survives deletion of original filename | Breaks if original file is deleted |
| Cannot cross filesystems               | Can cross filesystems              |
| Usually cannot link directories        | Can link directories               |

Inspect inode numbers:

```bash
ls -li
```

---

## 4️⃣ Text Processing & Viewing Tools

### Viewing Files

```bash
cat file.txt
less large.log        # Better for big files (Space = next, /search, q = quit)
head -n 10 access.log
tail -n 20 error.log
tail -f /var/log/syslog   # Follow live logs
```

---

### Searching & Filtering

```bash
grep "error" /var/log/syslog
grep -ir "TODO" ~/projects/
grep -v "DEBUG" logfile
```

---

### Processing Pipelines (Very Common Pattern)

#### Count Unique IP Addresses

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -nr | head -n 10
```

#### List Users With UID > 1000

```bash
awk -F: '$3 > 1000 {print $1}' /etc/passwd
```

#### Replace Text in File (In-place)

```bash
sed -i 's/old-text/new-text/g' config.conf
```

---

## Quick Reference – Small Power Tools

| Command | Typical Use                                |
| ------- | ------------------------------------------ |
| `cut`   | Extract columns                            |
| `sort`  | Sort lines (`-n`, `-r`, `-k`)              |
| `uniq`  | Remove/count duplicates (after sort)       |
| `wc`    | Count lines/words/bytes (`-l`, `-w`, `-c`) |
| `awk`   | Column-based processing                    |
| `sed`   | Stream editing (replace/delete lines)      |

---

## Recommended Practice Commands

### View Structure

```bash
tree /etc -L 2
```

### Find Files in Home Directory

```bash
find ~ -type f -name "*.txt" 2>/dev/null
```

### Create a Test Structure

```bash
mkdir -p test/{docs,code,backup}
touch test/docs/report{1..3}.md
ln -s docs/report1.md test/latest.md
ln docs/report1.md test/hard-report1.md
ls -li test/
```

### Analyze Command Usage

```bash
history | awk '{print $2}' | sort | uniq -c | sort -nr | head -15
```

---

## Key Takeaways

* Everything in Linux exists under `/`
* Files are references to inodes
* Links behave differently at filesystem level
* Linux tools are composable using pipes (`|`)
* Most real-world debugging involves combining small tools into pipelines

---

This completes **Module 1: Linux Command-Line Essentials**.

---
