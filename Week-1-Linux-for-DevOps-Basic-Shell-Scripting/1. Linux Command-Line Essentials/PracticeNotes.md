# Linux Command-Line Practice Notes

---

## 1️⃣ Hard Links vs Symbolic Links

### Setup

```bash
echo "DevOps is engineering under uncertainty." > original.txt  # Create a file with content
ls -li original.txt                                             # -i shows inode number, -l shows full details
```

Expected output:
```
123456 -rw-r--r-- 1 user user 43 original.txt
```
`123456` is the **inode** — the actual data address on disk. The filename is just a label pointing to it.

---

### Hard Link

```bash
ln original.txt hardlink.txt    # Create a hard link — a second name for the same inode
ls -li                          # Both files will show the same inode number
```

Expected output:
```
123456 -rw-r--r-- 2 user user 43 hardlink.txt
123456 -rw-r--r-- 2 user user 43 original.txt
```

The link count is now `2` — two names, one data block.

```bash
echo "Chaos is common." >> hardlink.txt   # Append through the hard link
cat original.txt                          # Verify original reflects the change — it will
```

```bash
rm original.txt       # Remove one name (label)
cat hardlink.txt      # Data still accessible — inode still has one reference left
```

> 💡 A hard link deletion only removes a name. The actual data is deleted only when the link count drops to **zero**.

---

### Symbolic Link

```bash
echo "DevOps is engineering under uncertainty." > original.txt  # Recreate the original file
ln -s original.txt symlink.txt                                  # Create a symlink — stores the path, not the inode
ls -li                                                          # Symlink has a different inode; shows arrow (->)
```

Expected output:
```
123470 lrwxrwxrwx 1 user user 13 symlink.txt -> original.txt
123469 -rw-r--r-- 1 user user 43 original.txt
```

```bash
rm original.txt      # Delete the original file
cat symlink.txt      # Fails — symlink points to a path that no longer exists (dangling symlink)
```

Output:
```
cat: symlink.txt: No such file or directory
```

> 💡 **Real DevOps use:** Nginx's `sites-enabled/` folder is entirely symlinks pointing into `sites-available/`. Enabling a site = create symlink. Disabling = remove symlink. No file is ever duplicated.

---

### 🏭 Production-Grade Example — Versioned Binary with Symlink

A common real-world pattern when managing multiple versions of a tool (e.g., Java, Node, Python):

```bash
# Simulate installing two versions of a tool
mkdir -p /opt/myapp/v1.0/bin                        # Create directory for version 1
mkdir -p /opt/myapp/v2.0/bin                        # Create directory for version 2

echo '#!/bin/bash\necho "myapp v1.0"' > /opt/myapp/v1.0/bin/myapp   # Fake v1 binary
echo '#!/bin/bash\necho "myapp v2.0"' > /opt/myapp/v2.0/bin/myapp   # Fake v2 binary

chmod +x /opt/myapp/v1.0/bin/myapp                  # Make v1 executable
chmod +x /opt/myapp/v2.0/bin/myapp                  # Make v2 executable

ln -s /opt/myapp/v1.0/bin/myapp /usr/local/bin/myapp  # Point "current" to v1

myapp                                                 # Runs v1 — outputs "myapp v1.0"

ln -sf /opt/myapp/v2.0/bin/myapp /usr/local/bin/myapp # -f forces update of existing symlink — now points to v2

myapp                                                 # Runs v2 — zero downtime version switch
```

> `ln -sf` is how deployment pipelines do **zero-downtime binary upgrades** — the symlink swap is atomic.

---

## 2️⃣ `sort` Examples

### Basic Lexicographical Sort

```bash
sort file.txt          # Default sort — lexicographical (ASCII order, uppercase first)
```

Input (`file.txt`):
```
banana
apple
Mango
cherry
```

Output:
```
Mango       ← uppercase M comes before lowercase in ASCII
apple
banana
cherry
```

> 💡 Use `sort -f` to ignore case if you want true alphabetical order regardless of case.

---

### Numeric Sort

```bash
sort numbers.txt        # Lexicographical — WRONG for numbers
sort -n numbers.txt     # -n → numeric sort — CORRECT
sort -nr numbers.txt    # -n → numeric, -r → reverse (largest first)
```

Without `-n`, `10` sorts before `2` (because `"1" < "2"` in ASCII). Always use `-n` for numbers.

---

### Sort by Specific Column

```bash
ps aux | sort -nk4       # Sort all running processes numerically (-n) by column 4 (%MEM)
ps aux | sort -nrk4      # Same but reversed — highest memory consumers at top
ps aux | sort -nrk3      # Sort by column 3 (%CPU) — find CPU-hungry processes
```

> 💡 `-k4` means "use column 4 as the sort key." Combined with `-nr` this becomes your quick **resource audit** without needing `htop`.

---

### 🏭 Production-Grade Example — Log Volume by Date

```bash
# Count how many log lines exist per date in a log file
# Assumes log lines start with a date like: 2024-03-10 ...

awk '{print $1}' /var/log/app.log \    # Extract just the date field (column 1)
  | sort \                             # Group identical dates together
  | uniq -c \                         # Count occurrences of each date
  | sort -nr \                        # Sort by count, highest first
  | head -10                          # Show top 10 busiest days
```

---

## 3️⃣ `awk` Field Processing

### How `awk` Thinks

> For every line → split into columns → apply condition → print result.

Default separator is **whitespace**. Use `-F` to set a custom one.

---

### Basic Column Extraction

```bash
awk '{print $1}' access.log              # Print first column of every line
awk '{print $1, $7}' access.log          # Print column 1 (IP) and column 7 (URL path)
awk -F: '{print $1}' /etc/passwd         # -F: sets colon as separator; print first field (username)
awk -F: '{print $1 " has UID " $3}' /etc/passwd  # Combine fields with a custom string
```

---

### Conditional Filtering

```bash
awk -F: '$3 > 1000 {print $1}' /etc/passwd    # Print usernames where UID (col 3) is > 1000
awk '$9 == "404" {print $7}' access.log        # Print URLs (col 7) where HTTP status (col 9) is 404
awk '$9 >= 500 {print $0}' access.log          # Print entire line for any 5xx server error
```

---

### `awk` with `BEGIN` and `END` Blocks

```bash
awk 'BEGIN {print "--- Report Start ---"} \    # Runs once before processing any lines
     {print $1, $9} \                          # Runs for every line — print IP and status code
     END {print "--- Report End ---"}' \        # Runs once after all lines are processed
     access.log
```

> 💡 `BEGIN`/`END` blocks are where you print headers, initialize counters, or print totals — very common in log report scripts.

---

### 🏭 Production-Grade Example — Count HTTP Status Codes

```bash
# Summarize all HTTP response codes from an nginx/apache access log
# Access log format: IP - - [date] "METHOD URL HTTP" STATUS_CODE size

awk '{print $9}' /var/log/nginx/access.log \   # Extract column 9 — the HTTP status code
  | sort \                                      # Group identical codes together
  | uniq -c \                                  # Count how many times each code appears
  | sort -nr                                   # Show most frequent codes first
```

Expected output:
```
  9420 200
  1023 304
   312 404
    45 500
     3 502
```

> Immediately tells you: are you serving mostly success (200s), or are there spikes in 404s or 500s?

---

### 🏭 Production-Grade Example — Detect Top Hitting IPs

```bash
# Find which IPs are making the most requests — useful for detecting abuse or DDoS

awk '{print $1}' /var/log/nginx/access.log \   # Extract source IP (column 1)
  | sort \                                      # Sort to group same IPs together
  | uniq -c \                                  # Count requests per IP
  | sort -nr \                                 # Highest request count first
  | head -10                                   # Show top 10 IPs
```

---

## 4️⃣ `sed` Examples

### Substitute (Preview Only — File Unchanged)

```bash
sed 's/old/new/' file.txt        # Replace first occurrence of 'old' per line — preview only
sed 's/old/new/g' file.txt       # g flag → replace ALL occurrences per line — preview only
```

---

### In-Place Edit

```bash
sed -i 's/old/new/g' file.txt          # Modify file directly ⚠️ — no backup
sed -i.bak 's/old/new/g' file.txt      # Modify file — creates file.txt.bak first (always prefer this)
```

> ⚠️ Never use bare `-i` on production config files. Always use `-i.bak` — one typo in a sed command can corrupt a config.

---

### Delete Lines

```bash
sed '1,5d' file.txt           # Delete lines 1 through 5 — output only, file unchanged
sed -i '1,5d' file.txt        # Delete lines 1–5 in place
sed '/^#/d' file.txt          # Delete all lines starting with # (comment lines)
sed '/^$/d' file.txt          # Delete all blank/empty lines
sed -n '5,10p' file.txt       # -n suppresses default output; p prints only lines 5–10
```

---

### 🏭 Production-Grade Example — Sanitize a Config Before Deployment

A common CI/CD step — strip all comments and blank lines from a config before shipping it:

```bash
# Start with the raw config file
cp app.conf app.conf.bak                    # Always back up before any sed operation

sed -i '/^#/d' app.conf                     # Delete all comment lines (lines starting with #)
sed -i '/^$/d' app.conf                     # Delete all empty/blank lines
sed -i 's/localhost/prod-db.internal/g' app.conf   # Swap dev DB host for production DB host

cat app.conf                                # Review the result before deploying
```

---

### 🏭 Production-Grade Example — Bulk Rename Config Values Across Multiple Files

```bash
# Scenario: you need to update an old API endpoint URL across 12 config files at once

find /etc/myapp -name "*.conf" \            # Find all .conf files under /etc/myapp
  | xargs sed -i.bak \                      # Run sed in-place on each file, with .bak backup
    's|api.old-domain.com|api.new-domain.com|g'  # Replace old URL with new (| used as delimiter to avoid escaping /)
```

> Using `|` instead of `/` as the sed delimiter avoids having to escape slashes in URLs — a common production trick.

---

## 5️⃣ `tree`, `find`, `grep` in Practice

### `tree`

```bash
tree /etc -L 2                  # Show /etc directory, 2 levels deep only
tree -a /home/user              # -a includes hidden files (dotfiles)
tree -d /var                    # -d shows only directories, not files
tree /etc -L 2 > structure.txt  # Save the directory structure to a file (useful for docs)
```

---

### `find`

```bash
find ~ -name "*.txt" 2>/dev/null              # Find .txt files; 2>/dev/null suppresses "Permission denied" errors
find /etc -type f -name "*.conf"              # -type f → files only (not directories)
find /var/log -name "*.log" -mtime -7         # Files modified in the last 7 days
find /tmp -name "*.sh" -exec chmod +x {} \;  # Find .sh files and make each one executable
                                              # {} = placeholder for each found file; \; = end of -exec command
```

---

### `grep`

```bash
grep "error" /var/log/syslog          # Find lines containing "error"
grep -i "error" /var/log/syslog       # -i → case-insensitive match
grep -n "FAIL" deploy.log             # -n → show line numbers with each match
grep -v "DEBUG" app.log               # -v → invert: show lines that do NOT match
grep -c "404" access.log              # -c → count matching lines only (no content printed)
grep -E "error|warn|fatal" app.log    # -E → extended regex; match any of the three patterns
grep -A 3 "FAILED" deploy.log         # -A 3 → show 3 lines AFTER each match (context)
grep -B 2 "FAILED" deploy.log         # -B 2 → show 2 lines BEFORE each match
grep -r "TODO" ~/projects/            # -r → recursive search through all subdirectories
```

---

### 🏭 Production-Grade Example — Triage a Failing Deployment

```bash
# Scenario: deployment just failed. You need to quickly find out what went wrong.

grep -i "error\|exception\|failed" /var/log/deploy.log \  # Find any error-like lines, case-insensitive
  | grep -v "INFO" \                                        # Exclude noise — lines that only have INFO level
  | tail -20                                               # Focus on the last 20 — most recent failures first
```

```bash
# Deeper triage — get context around the failure, not just the error line itself
grep -n "FAILED" /var/log/deploy.log \     # Find the exact line number of failure
  | head -5                                # Get the first few failure occurrences

grep -A 5 -B 2 "FAILED" /var/log/deploy.log   # Show 2 lines before and 5 after each FAILED — full context
```

---

## 6️⃣ The Pipeline Pattern — Full Breakdown

### The Command

```bash
history | awk '{print $2}' | sort | uniq -c | sort -nr | head -15
#         ↑ extract cmd     ↑ group  ↑ count   ↑ rank      ↑ trim
```

> Note: The original notes used `$1` — this is a **bug**. In Bash, `history` output is `<id>  <command>`, so `$1` extracts the number, not the command. `$2` is correct.

### Step-by-Step Breakdown

```bash
history                     # Step 1: Raw history — format is: <id>  <command> <args>

| awk '{print $2}'          # Step 2: Extract column 2 — the command name (ls, git, docker...)
                            # $1 would give the history ID (useless), $2 gives the actual command

| sort                      # Step 3: Sort alphabetically — groups all identical commands together
                            # uniq only counts CONSECUTIVE duplicates, so sort must come first

| uniq -c                   # Step 4: Count consecutive duplicates — outputs "  N commandname"

| sort -nr                  # Step 5: -n numeric sort, -r reverse — highest count at the top

| head -15                  # Step 6: Trim output to top 15 — no need to see 500 lines
```

### Expected Output

```
     47 git
     38 ls
     21 docker
     14 kubectl
      9 vim
      7 cd
      4 ssh
      3 grep
      2 cat
      1 top
```

---

### The Extract → Group → Count → Rank → Trim Pattern

This is the most reusable pattern in Linux text processing. It appears everywhere in DevOps:

```bash
# Most common IPs hitting your server
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -10

# Most frequent error types in app logs
grep "ERROR" app.log | awk '{print $5}' | sort | uniq -c | sort -nr | head -10

# Largest directories consuming disk space
du -sh /var/log/* | sort -rh | head -10    # -h → human-readable sizes; -rh sorts by size descending
```

> The tools change. The pattern doesn't.

---

## Key Takeaways

- Hard links share an inode — deleting one name doesn't delete the data until all names are gone
- Symlinks store a path — if the target moves or is deleted, the symlink breaks
- `ln -sf` is the atomic pattern for zero-downtime binary/config version switching
- `sort -n` is mandatory for numbers — lexicographical sort silently gives wrong results
- `awk '{print $2}'` not `$1` for extracting commands from `history` — always inspect raw output first
- `sed -i.bak` over bare `-i` — always leave yourself a backup before in-place edits
- `grep -A/-B` gives you context around matches — critical for real log triage
- The **Extract → Group → Count → Rank → Trim** pipeline pattern is universal in DevOps observability
- `find -exec` lets you act on results inline — eliminates the need for a separate loop

---