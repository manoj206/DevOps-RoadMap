# Linux Command Line Practice Notes

This document contains practical examples and explanations for working with:

* Hard Links vs Symbolic Links
* `sort` usage
* `awk` field processing
* `sed` transformations
* `tree`, `find`, `grep`, and directory structure operations

---

# Hard Links vs Symbolic Links

## Create a Sample File

```bash
echo "DevOps is engineering under uncertainty." > original.txt
```

Check inode:

```bash
ls -li original.txt
```

Example output:

```
123456 -rw-r--r-- 1 user user 43 original.txt
```

The number `123456` is the inode number.

---

## Hard Link

```bash
ln original.txt hardlink.txt
```

Inspect:

```bash
ls -li
```

Example:

```
123456 -rw-r--r-- 2 user user 43 hardlink.txt
123456 -rw-r--r-- 2 user user 43 original.txt
```

### Observations

* Both files share the same inode.
* Link count is 2.
* They reference the same data on disk.

### Edit through hard link

```bash
echo "Chaos is common." >> hardlink.txt
cat original.txt
```

Output:

```
DevOps is engineering under uncertainty.
Chaos is common.
```

Both names refer to the same data.

### Delete original

```bash
rm original.txt
cat hardlink.txt
```

File still exists because one reference remains.

---

## Symbolic Link

Recreate original:

```bash
echo "DevOps is engineering under uncertainty." > original.txt
```

Create symlink:

```bash
ln -s original.txt symlink.txt
```

Inspect:

```bash
ls -li
```

Example:

```
123470 lrwxrwxrwx 1 user user 13 symlink.txt -> original.txt
123469 -rw-r--r-- 1 user user 43 original.txt
```

### Observations

* Different inode.
* Symlink stores path reference.

### Delete original

```bash
rm original.txt
cat symlink.txt
```

Output:

```
No such file or directory
```

Symlink is broken because it points to a missing path.

---

# `sort` Examples

## Basic Sort

`file.txt`:

```
banana
apple
Mango
cherry
```

```bash
sort file.txt
```

Output:

```
Mango
apple
banana
cherry
```

Sort is lexicographical by default (ASCII-based).

---

## Numeric Reverse Sort

`numbers.txt`:

```
10
2
50
3
25
```

```bash
sort numbers.txt
```

Output (incorrect numerically):

```
10
2
25
3
50
```

Correct numeric reverse sort:

```bash
sort -nr numbers.txt
```

Output:

```
50
25
10
3
2
```

Flags:

* `-n` → numeric
* `-r` → reverse

---

## Sorting Process Memory

```bash
ps aux | sort -nk4
```

* `-k4` → sort by 4th column (%MEM)
* `-n` → numeric sort

To get highest memory first:

```bash
ps aux | sort -nrk4
```

---

# `awk` Field Processing

## Example with `/etc/passwd`

Colon-separated file:

```
root:x:0:0:root:/root:/bin/bash
```

### Explicit Field Separator

```bash
awk -F: '{print $1 " has UID " $3}' /etc/passwd
```

Output:

```
root has UID 0
```

`-F:` sets field separator to colon.

---

## Conditional Filtering

Correct version:

```bash
awk -F: '$3 > 1000 {print $1}' /etc/passwd
```

Prints usernames where UID > 1000 (typically regular users).

Without `-F`, default separator is whitespace.

---

# `sed` Examples

Assume `file.txt`:

```
old value here
this is old configuration
keep this line
another old setting
final old line
stable config
```

---

## Substitute (Print Only)

```bash
sed 's/old/new/g' file.txt
```

Output:

```
new value here
this is new configuration
keep this line
another new setting
final new line
stable config
```

File remains unchanged.

---

## In-place Edit

```bash
sed -i 's/old/new/g' file.txt
```

Modifies file directly.

Safer option:

```bash
sed -i.bak 's/old/new/g' file.txt
```

Creates backup.

---

## Delete Lines

```bash
sed '1,5d' file.txt
```

Deletes lines 1–5 in output stream.

Result:

```
stable config
```

Original file unchanged (unless `-i` used).

---

# `tree`

```bash
tree /etc -L 2
```

* `/etc` → starting directory
* `-L 2` → limit depth to 2 levels

Sample output:

```
/etc
├── nginx
│   ├── nginx.conf
│   └── sites-enabled
├── passwd
└── systemd
    ├── system
```

Useful for visualizing directory structure.

---

# `find`

```bash
find ~ -name "*.txt" 2>/dev/null
```

* Searches home directory
* Ignores permission errors (`2>/dev/null`)

Example output:

```
/home/user/notes.txt
/home/user/projects/readme.txt
```

---

# Directory Structure and Links Practice

Create structure:

```bash
mkdir -p demo/app/config
mkdir -p demo/app/logs
```

Create files:

```bash
echo "server=prod" > demo/app/config/settings.txt
cp demo/app/config/settings.txt demo/app/logs/log.txt
```

Create links:

```bash
ln demo/app/config/settings.txt demo/app/config/hardlink.txt
ln -s demo/app/config/settings.txt demo/app/config/symlink.txt
```

Inspect:

```bash
ls -li demo/app/config
```

Example:

```
123456 -rw-r--r-- 2 user user 12 settings.txt
123456 -rw-r--r-- 2 user user 12 hardlink.txt
123789 lrwxrwxrwx 1 user user 29 symlink.txt -> demo/app/config/settings.txt
```

---

# Recursive Grep

```bash
grep -r "TODO" ~/projects
```

Searches recursively for the string "TODO".

Example output:

```
/home/user/projects/app.py:# TODO: improve error handling
/home/user/projects/config/settings.yaml:# TODO: refactor config format
```

Useful for:

* Searching codebases
* Finding leftover debug markers
* Auditing configurations

---

# Key Takeaways

* Hard links share the same inode; symlinks reference a path.
* `sort` defaults to lexicographical; use `-n` for numeric.
* `awk` requires correct field separator for structured files.
* `sed` modifies streams unless `-i` is used.
* `tree`, `find`, `grep` are essential filesystem navigation tools.
* Always inspect inode behavior using `ls -li` when working with links.

---

# Analyzes the following command usage pattern

```bash
history | awk '{print $1}' | sort | uniq -c | sort -nr | head -15
```

---

# Step 1 — What `history` Looks Like

Typical history output looks like this:

```bash
  1  ls
  2  cd projects
  3  vim main.py
  4  git status
  5  ls
  6  docker build .
  7  git status
  8  ls
  9  git commit -m "init"
 10  docker ps
 11  git status
 12  ls
```

Format:

```
<command_number>  <actual command>
```

Important:
The first column is just a counter.

---

# Step 2 — `awk '{print $1}'`

This extracts the first column only.

So the stream becomes:

```bash
1
2
3
4
5
6
7
8
9
10
11
12
```

Wait.

That seems useless. Why?

Because in many shells, `$1` is the history number, not the command.

That means this pipeline depends on how your shell prints history.

In Bash, actual useful version usually is:

```bash
history | awk '{print $2}'
```

Because:

Column 1 → history ID
Column 2 → first word of command

So let’s assume what you actually want is counting commands used most often.

Corrected logical version:

```bash
history | awk '{print $2}'
```

This produces:

```bash
ls
cd
vim
git
ls
docker
git
ls
git
docker
git
ls
```

---

# Step 3 — `sort`

```bash
... | sort
```

Now output becomes:

```bash
cd
docker
docker
git
git
git
git
ls
ls
ls
ls
vim
```

Sorting groups identical commands together.

---

# Step 4 — `uniq -c`

`uniq -c` counts consecutive duplicates.

So now:

```bash
... | uniq -c
```

Produces:

```bash
      1 cd
      2 docker
      4 git
      4 ls
      1 vim
```

This means:
You used git 4 times.
You used ls 4 times.
Docker twice.
Etc.

---

# Step 5 — `sort -nr`

Numeric reverse sort.

```bash
... | sort -nr
```

Now it ranks most-used commands first:

```bash
      4 git
      4 ls
      2 docker
      1 vim
      1 cd
```

---

# Step 6 — `head -15`

Show top 15 lines.

Since example is small, it prints everything.

---

# Final Output Example

```bash
4 git
4 ls
2 docker
1 vim
1 cd
```

---

# What This Pipeline Actually Does

It answers:

“What are my most frequently used commands?”

This is command-line self-observation.

---

# Let’s Rebuild the Pipeline Mechanically

```bash
history
```

Raw data.

```bash
| awk '{print $2}'
```

Extract command.

```bash
| sort
```

Group identical commands together.

```bash
| uniq -c
```

Count frequency.

```bash
| sort -nr
```

Rank by usage.

```bash
| head -15
```

Top 15 commands.

---

# Why This Is Powerful

Because this is a pattern:

Extract → Group → Count → Rank → Trim

That pattern is used everywhere in DevOps:

* Count most common IP hitting server
* Find top error types
* Identify memory-heavy processes
* Detect log frequency spikes

It’s primitive analytics built from tiny Unix tools.

---

# Subtle Gotcha

If a command has arguments:

```bash
git status
git pull
git commit
```

Using `$2` groups them all as `git`.

But if your history formatting differs, `$1` might already be the command.

Always inspect raw `history` first.

---

# The Deeper Skill

This isn’t about counting commands.

It’s about learning to think:

“Everything is text.”
“Text can be piped.”
“Pipes can be shaped into insight.”

That mindset is what differentiates someone who *runs commands* from someone who *builds command pipelines*.


