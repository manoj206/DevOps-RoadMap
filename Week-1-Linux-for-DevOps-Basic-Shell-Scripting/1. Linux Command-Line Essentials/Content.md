# Linux Command-Line Essentials (Core OS Interaction)

Everything in Linux revolves around the **command line (terminal)**.
This section covers the foundational skills needed to confidently navigate, manage files, and process text in Linux.

---

## 1️⃣ Linux Filesystem Hierarchy

Linux uses a single **upside-down tree** structure starting from the **root directory (`/`)**.
There are **no drive letters** (C:, D:, etc.). Everything exists under `/`.

| Directory | Purpose | Analogy |
|-----------|---------|---------|
| `/` | Root of entire filesystem | Foundation of the house |
| `/home` | User home directories | Personal bedrooms (`/home/username`) |
| `/etc` | System-wide configuration files | Control panel / brain |
| `/var` | Variable data (logs, DBs, cache) | Storage room (`/var/log`) |
| `/tmp` | Temporary files | Trash bin (often cleared on reboot) |
| `/usr` | User programs & libraries | Toolbox — most installed software |
| `/opt` | Optional third-party software | Garage — large manually installed apps |
| `/proc` | Virtual filesystem (process info) | Live X-ray of the running system |
| `/sys` | Virtual filesystem (kernel/devices) | Electrical wiring |
| `/boot` | Bootloader & kernel files | Starter motor |
| `/dev` | Device files | Plugs & sockets (`/dev/null`, `/dev/sda`) |
| `/bin` | Essential user binaries | Basic tools (`ls`, `cp`, `cat`) |
| `/sbin` | Admin binaries | System-level tools (`fdisk`, `iptables`) |
| `/mnt` | Temporary mount points (external disks, NFS) | Docking bay |
| `/media` | Auto-mounted removable media (USB, CD) | USB slot |

> 💡 **Tip:** `/mnt` and `/media` are frequently encountered when working with cloud storage volumes or network file systems — something you'll hit early in DevOps.

### Quick Check
```bash
ls /
```

---

## 2️⃣ Tree Navigation & Directory Exploration

### Absolute vs Relative Paths *(important foundation)*

Before navigating, understand how paths work:

| Type | Description | Example |
|------|-------------|---------|
| **Absolute** | Always starts from `/` — works from anywhere | `/home/user/projects/app` |
| **Relative** | Starts from your *current* directory | `../projects/app` |

> 💡 Use absolute paths in scripts — relative paths break when a script is called from a different location.

### Navigation Commands

| Command | Purpose | Examples |
|---------|---------|---------|
| `pwd` | Print current directory | `pwd` |
| `cd` | Change directory | `cd /etc`, `cd ..`, `cd -`, `cd ~`, `cd ~/projects` |
| `ls` | List contents | `ls -la`, `ls -lh`, `ls -R` |
| `tree` | Visual directory structure | `tree -L 2`, `tree -a` |
| `find` | Search files/dirs | `find ~ -name "*.pdf"` |
| `locate` | Fast indexed search | `locate nginx.conf` |

### `cd` Shortcuts Worth Memorising

```bash
cd ~        # Go to your home directory (/home/username)
cd -        # Jump back to previous directory (very handy!)
cd ..       # One level up
cd ../..    # Two levels up
```

### `ls` Variants Explained

```bash
ls -l       # Long format (permissions, owner, size, date)
ls -a       # Show hidden files (dotfiles like .bashrc)
ls -la      # Both combined — most commonly used
ls -lh      # Human-readable file sizes (KB, MB, GB)
ls -lt      # Sort by modification time (newest first)
ls -R       # Recursive — lists all subdirectories too
```

### `find` — Beyond Basic Name Search

```bash
find ~ -name "*.pdf"                      # Find by name
find /etc -type f -name "*.conf"          # Only files (-type f), not dirs
find /home -type d -name "backup"         # Only directories (-type d)
find /var/log -name "*.log" -mtime -7     # Modified in last 7 days
find /tmp -name "*.sh" -exec chmod +x {} \;  # Find AND run a command on results
```

> 💡 `2>/dev/null` suppresses "Permission denied" noise:
> ```bash
> find / -name "secret.txt" 2>/dev/null
> ```

### `locate` vs `find`

| | `find` | `locate` |
|--|--------|---------|
| How it works | Scans filesystem in real time | Queries a pre-built index |
| Speed | Slower (thorough) | Very fast |
| Always up-to-date? | ✅ Yes | ❌ No — index can be stale |
| Fix stale index | — | `sudo updatedb` |

---

## 3️⃣ File & Directory Operations

| Command | Action | Examples |
|---------|--------|---------|
| `touch` | Create empty file / update timestamp | `touch notes.txt`, `touch file{1..5}.txt` |
| `mkdir` | Create directory | `mkdir -p projects/web/css` |
| `cp` | Copy files/directories | `cp file.txt /tmp/`, `cp -r folder/ backup/` |
| `mv` | Move or rename | `mv old.txt new.txt`, `mv file.txt ~/backup/` |
| `rm` | Remove files/directories | `rm file.txt`, `rm -r folder/`, `rm -rf temp/` ⚠️ |
| `ln` | Create links | `ln original hardlink`, `ln -s original symlink` |

### Key Flags Worth Knowing

```bash
cp -r         # Recursive — required for copying directories
cp -p         # Preserve timestamps & permissions (important in deployments)
cp -v         # Verbose — shows what's being copied
rm -rf        # ⚠️ Force-removes recursively — no confirmation, no undo
mv -i         # Interactive — prompts before overwriting (safe habit)
mkdir -p      # Creates full nested path; no error if dir already exists
```

### Hard Link vs Symbolic Link

| | Hard Link | Symbolic Link |
|--|-----------|--------------|
| Points to | Same inode (same data block) | A file path |
| If original is deleted | Still works ✅ | Breaks ❌ (dangling symlink) |
| Cross filesystems | ❌ No | ✅ Yes |
| Link directories | ❌ Usually not | ✅ Yes |
| Common use case | Backup/deduplication | Config aliasing, versioned binaries |

```bash
ln docs/report1.md hard-report1.md       # Hard link
ln -s docs/report1.md latest.md          # Symbolic link

ls -li        # -i shows inode numbers — hard links share the same inode
```

> 💡 **Real DevOps use:** Symlinks are used constantly — e.g., `nginx`'s `sites-enabled` folder is just symlinks pointing to `sites-available`. You enable a site by creating a symlink; you disable it by removing one.

### Quick Practice Structure

```bash
mkdir -p test/{docs,code,backup}
touch test/docs/report{1..3}.md
ln -s docs/report1.md test/latest.md
ln test/docs/report1.md test/hard-report1.md
ls -li test/docs/
```

---

## 4️⃣ Text Processing & Viewing Tools

### Viewing Files

```bash
cat file.txt                    # Print entire file (best for short files)
less large.log                  # Paginate — best for big files
more large.log                  # Older paginator — forward-only (less is better)
head -n 10 access.log           # First 10 lines
tail -n 20 error.log            # Last 20 lines
tail -f /var/log/syslog         # Follow live — great for watching logs in real time
```

> 💡 `less` shortcuts: `Space` = next page, `b` = back, `/pattern` = search, `q` = quit.
> `more` only goes forward — use `less` by default.

---

### `grep` — Searching & Filtering

```bash
grep "error" /var/log/syslog         # Basic match
grep -i "error" logfile              # Case-insensitive
grep -r "TODO" ~/projects/           # Recursive search across directories
grep -v "DEBUG" logfile              # Invert — show lines that DON'T match
grep -n "FAIL" deploy.log            # Show line numbers with matches
grep -c "404" access.log             # Count matching lines only
grep -E "error|warn|fatal" app.log   # Extended regex — match multiple patterns
grep -A 3 "FAILED" deploy.log        # Show 3 lines AFTER each match
grep -B 2 "FAILED" deploy.log        # Show 2 lines BEFORE each match
```

> 💡 `-A` and `-B` are extremely useful in real debugging — you need the **context** around an error, not just the error line itself.

---

### `awk` — Column-Based Processing

```bash
# Print specific column ($1 = first field, $2 = second, etc.)
awk '{print $1}' access.log

# Custom field separator (e.g., colon-separated /etc/passwd)
awk -F: '{print $1}' /etc/passwd            # Print all usernames

# With condition
awk -F: '$3 > 1000 {print $1}' /etc/passwd  # Users with UID > 1000

# Print multiple columns
awk '{print $1, $7}' access.log              # IP and request path
```

> 💡 Think of `awk` as: *"For each line, split into columns, apply condition, print result."*

---

### `sed` — Stream Editing

```bash
sed 's/old/new/' file.txt            # Replace first occurrence per line
sed 's/old/new/g' file.txt           # Replace ALL occurrences (global)
sed -i 's/old-text/new-text/g' config.conf   # Edit file in-place ⚠️
sed -i.bak 's/old/new/g' config.conf         # In-place edit WITH backup (safer)
sed '/^#/d' config.conf              # Delete all comment lines
sed -n '5,10p' file.txt              # Print only lines 5 to 10
```

> ⚠️ Always use `-i.bak` instead of bare `-i` when editing files in-place — it creates a backup (`config.conf.bak`) before modifying.

---

### Quick Reference — Small Power Tools

| Command | Typical Use | Example |
|---------|------------|---------|
| `cut` | Extract columns by delimiter | `cut -d: -f1 /etc/passwd` → print usernames |
| `sort` | Sort lines | `sort -n` (numeric), `-r` (reverse), `-k2` (by column 2) |
| `uniq` | Remove/count duplicates | `sort file \| uniq -c` — must sort first! |
| `wc` | Count lines/words/bytes | `wc -l file.txt`, `wc -w`, `wc -c` |
| `awk` | Column-based processing | See above |
| `sed` | Stream editing | See above |

> 💡 `uniq` only removes **adjacent** duplicates — always `sort` first before piping to `uniq`.

---

### Processing Pipelines (The Real Power)

```bash
# Count unique IP addresses in a web access log
awk '{print $1}' access.log | sort | uniq -c | sort -nr | head -n 10

# Find all users with UID > 1000
awk -F: '$3 > 1000 {print $1}' /etc/passwd

# Replace text in a config file (with backup)
sed -i.bak 's/old-text/new-text/g' config.conf

# Count how many 404 errors in a log
grep "404" access.log | wc -l

# Most used commands in your history
history | awk '{print $2}' | sort | uniq -c | sort -nr | head -15
```

---

## Recommended Practice

```bash
# 1. Explore structure
tree /etc -L 2

# 2. Find files
find ~ -type f -name "*.txt" 2>/dev/null

# 3. Build a test structure with links
mkdir -p test/{docs,code,backup}
touch test/docs/report{1..3}.md
ln -s docs/report1.md test/latest.md
ln test/docs/report1.md test/hard-report1.md
ls -li test/

# 4. Simulate log analysis
grep -E "ERROR|WARN" /var/log/syslog | tail -20
awk '{print $1}' /var/log/syslog | sort | uniq -c | sort -nr | head -5

# 5. Analyse your own command usage
history | awk '{print $2}' | sort | uniq -c | sort -nr | head -15
```

---

## Key Takeaways

- Everything in Linux lives under `/` — know the major directories by heart
- Absolute paths are safe in scripts; relative paths depend on *where* you run from
- `find` searches in real time; `locate` is faster but needs `updatedb` to stay current
- `less` is better than `more` — use it by default for large files
- `tail -f` is your go-to for watching live logs
- `grep -A/-B` gives you context around matches — critical for real debugging
- Always use `sed -i.bak` when editing files in-place — backups save you
- `uniq` needs `sort` first — they almost always go together
- The real power of Linux is **composing small tools into pipelines** with `|`

---