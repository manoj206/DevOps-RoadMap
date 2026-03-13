# Linux Module 2: File Permissions & Ownership

File permissions are the **core security mechanism of Linux**.
They control **who can read, write, or execute** files and directories.

Misconfigured permissions are one of the fastest ways to:
- Create security vulnerabilities
- Break application deployments
- Lock yourself (or a service) out of required files

---

## 1️⃣ User / Group / Other — rwx Breakdown

Every file and directory has **three permission classes** and **three permission types**.

### The Three Classes

| Class | Symbol | Who it applies to |
|-------|--------|------------------|
| Owner | `u` | The user who owns the file |
| Group | `g` | Users belonging to the file's assigned group |
| Others | `o` | Everyone else on the system |

### The Three Permission Types

| Permission | Symbol | Numeric | On Files | On Directories |
|------------|--------|---------|----------|----------------|
| read | `r` | 4 | View file contents | List contents with `ls` |
| write | `w` | 2 | Modify or delete file | Create, rename, delete items inside |
| execute | `x` | 1 | Run as a program/script | Enter with `cd` and access contents |

> ⚠️ **Directory execute is frequently misunderstood.** Without `x` on a directory, you cannot `cd` into it or access anything inside — even if the files themselves have open permissions. This is a very common source of "Permission denied" in deployments.

### Reading a Permission String

```bash
ls -l script.sh
# -rwxr-xr--  1 alice developers 4096 Mar 3 18:30 script.sh
```

Break it down character by character:

```
- rwx r-x r--
│ │   │   │
│ │   │   └── others  : r-- → read only (4)
│ │   └─────── group  : r-x → read + execute (5)
│ └─────────── owner  : rwx → full access (7)
└───────────── file type: - = regular file, d = directory, l = symlink
```

### Verifying Permissions in Practice

```bash
ls -l file.txt           # View permissions of a specific file
ls -la                   # Include hidden files (dotfiles)
ls -ld /etc/nginx/       # -d shows the directory itself, not its contents
stat file.txt            # Full metadata — permissions, inode, timestamps, owner
```

> 💡 Always use `ls -ld` when debugging **directory permission** issues — `ls -l` shows the contents, not the directory itself.

---

## 2️⃣ `chmod` — Change Mode

### Symbolic Mode (Recommended for Day-to-Day Use)

Format: `chmod [who][operator][permission] file`

| Who | Meaning |
|-----|---------|
| `u` | owner |
| `g` | group |
| `o` | others |
| `a` | all three (u + g + o) |

| Operator | Meaning |
|----------|---------|
| `+` | Add permission |
| `-` | Remove permission |
| `=` | Set exactly (replaces existing) |

```bash
chmod u+x script.sh          # Give owner execute permission
chmod g-w report.txt         # Remove write from group
chmod o-rwx secret.key       # Remove all permissions from others
chmod a+x runme.sh           # Give everyone execute
chmod u=rw,g=r,o= config.ini # Set exactly: owner=rw, group=r, others=nothing
chmod +x *.sh                # Add execute to all .sh files in current directory
```

> 💡 **When to use symbolic:** When you want to *add or remove* a single permission without touching the rest. Safer for one-off changes.

---

### Numeric (Octal) Mode

Each permission type has a value: `r=4`, `w=2`, `x=1`. Add them up per class.

```
rwx = 4+2+1 = 7
r-x = 4+0+1 = 5
r-- = 4+0+0 = 4
--- = 0+0+0 = 0
```

So `chmod 754 file` means: owner=`rwx`, group=`r-x`, others=`r--`

### Standard Octal Values to Memorise

| Octal | Symbolic | Meaning | Typical Use |
|-------|----------|---------|-------------|
| `644` | rw-r--r-- | Owner writes, everyone reads | Config files, web assets |
| `755` | rwxr-xr-x | Owner full, everyone can read+execute | Scripts, public directories |
| `700` | rwx------ | Owner only, completely private | Personal scripts |
| `750` | rwxr-x--- | Owner full, group can read+execute | Team-shared executables |
| `600` | rw------- | Owner read/write only | SSH private keys, secrets |
| `400` | r-------- | Owner read-only | Sensitive credentials (read once) |

```bash
chmod 644 config.yml         # Standard config file
chmod 755 deploy.sh          # Executable script
chmod 600 ~/.ssh/id_ed25519  # Private key — SSH will reject it if this is wrong
chmod -R 750 /srv/app        # Recursive — apply to all files and subdirs ⚠️
```

> 💡 **When to use numeric:** When you want to set the **complete** permission state precisely. Better for scripts and automation.

> ⚠️ `chmod -R` is powerful and dangerous. A common mistake is running `chmod -R 644` on a directory tree that includes scripts — it removes execute from everything, silently breaking all your scripts. Always test on a subdirectory first.

---

## 3️⃣ Special Permissions — setuid, setgid, Sticky Bit

These are a **fourth permission digit** prepended to the usual three. They modify behaviour beyond basic rwx.

| Bit | Octal | Symbolic | On Files | On Directories |
|-----|-------|----------|----------|----------------|
| setuid | `4xxx` | `u+s` | File runs as its **owner**, not the caller | Rarely used |
| setgid | `2xxx` | `g+s` | File runs as its **group** | New files inside **inherit the directory's group** |
| sticky | `1xxx` | `+t` | No modern effect | Only the **file's owner** (or root) can delete it |

### How They Appear in `ls -l`

```bash
-rwsr-xr-x   # setuid set — 's' replaces 'x' in owner position
-rwxr-sr-x   # setgid set — 's' replaces 'x' in group position
drwxrwxrwt   # sticky set — 't' replaces 'x' in others position
```

If the underlying `x` bit is **not** set, the special bit appears as uppercase (`S` or `T`) — a warning that the bit is set but has no effect.

### setuid

```bash
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root ... /usr/bin/passwd
```

Any user can run `passwd` — but it executes **as root** (the owner), so it can write to `/etc/shadow`. Without setuid, only root could change passwords.

```bash
chmod u+s /usr/local/bin/mytool   # Set setuid symbolically
chmod 4755 /usr/local/bin/mytool  # Set setuid numerically (4 = setuid digit)
```

> ⚠️ Setuid on shell scripts is **ignored** by Linux for security reasons. It only works on compiled binaries.

### setgid

```bash
chmod g+s /shared/team      # Symbolic
chmod 2755 /shared/team     # Numeric
```

Any file created inside `/shared/team` will automatically inherit the directory's group — not the creating user's default group. Essential for shared team directories so everyone can access each other's files.

### Sticky Bit

```bash
ls -ld /tmp
# drwxrwxrwt 10 root root ... /tmp
```

`/tmp` is world-writable — anyone can create files. But the sticky bit (`t`) means **you can only delete your own files**, even though you have write on the directory.

```bash
chmod +t /myproject/uploads    # Symbolic
chmod 1777 /myproject/uploads  # Numeric — world-writable + sticky
```

> 💡 **Real DevOps use:** Set the sticky bit on any shared upload or scratch directory — prevents one user's process from accidentally deleting another's files.

---

## 4️⃣ Ownership Management — `chown` and `chgrp`

### `chown` — Change Owner (and optionally Group)

```bash
chown alice script.sh                    # Change owner to alice
chown alice:developers report.xlsx       # Change owner AND group in one command
chown :developers report.xlsx            # Change group only (colon with no user)
chown -R www-data:www-data /var/www/     # Recursive — essential after deploying web app files
```

> ⚠️ Only **root** (or a user with `sudo`) can change file ownership. A regular user cannot give their file to someone else.

### `chgrp` — Change Group Only

```bash
chgrp devs /shared/folder               # Change group of a single item
chgrp -R devs /shared/                  # Recursive group change
```

> 💡 `chown :groupname file` and `chgrp groupname file` do the same thing. `chown` is more commonly used because it handles both owner and group in one command.

### Checking Group Membership

```bash
groups                    # Show all groups your current user belongs to
groups alice              # Show groups for a specific user
id                        # Detailed: uid, gid, and all group memberships
id alice                  # Same for another user
```

> 💡 If you add a user to a group and the change seems to have no effect, they likely need to **log out and back in** for the new group membership to be picked up by their session.

### 🏭 Production-Grade Example — Post-Deployment Ownership Fix

A very common situation: you deploy files as root (via CI/CD or scp), but your app runs as `www-data`. Result: the app can't read its own files.

```bash
# After deploying new app files to the server
chown -R www-data:www-data /var/www/myapp/     # Give the web server user ownership of all app files
chmod -R 750 /var/www/myapp/                   # Owner full access, group read+execute, others nothing
chmod 640 /var/www/myapp/config/database.yml   # Config file — owner read/write, group read, others nothing
```

---

## 5️⃣ `umask` — Default Permissions for New Files

When any file or directory is created, Linux starts with a **maximum permission**, then subtracts the `umask` to get the actual result.

| Type | Maximum | umask `022` subtracted | Result |
|------|---------|----------------------|--------|
| File | `666` | `022` | `644` (rw-r--r--) |
| Directory | `777` | `022` | `755` (rwxr-xr-x) |

> ⚠️ Linux never grants execute to new **files** automatically (max is `666`, not `777`). Execute must always be set explicitly with `chmod`. Directories start at `777` because `x` is needed to enter them.

### Common umask Values

| umask | New File | New Directory | When to use |
|-------|----------|---------------|-------------|
| `022` | `644` | `755` | Default — safe for most servers |
| `002` | `664` | `775` | Collaborative teams — group can write |
| `027` | `640` | `750` | Stricter — others get no permissions |
| `077` | `600` | `700` | High security — owner only |

```bash
umask           # View current umask (shows as e.g. 0022)
umask -S        # Show in symbolic form: u=rwx,g=rx,o=rx
umask 027       # Set for current shell session only
```

### Making umask Persistent

```bash
# Add to ~/.bashrc or ~/.profile for a single user
echo "umask 027" >> ~/.bashrc

# Add to /etc/profile or /etc/login.defs for system-wide default
```

### 🏭 Production-Grade Example — Secure Service User umask

For a service like a web app that should never expose files to others:

```bash
# In the service's startup script or systemd unit file
umask 027                    # New files: 640 (owner rw, group r, others nothing)
                             # New dirs:  750 (owner rwx, group rx, others nothing)

./start-app.sh               # Any files the app creates will follow this umask
```

---

## 6️⃣ ACLs — Access Control Lists

Standard Unix permissions only give you **one owner** and **one group**. ACLs let you grant permissions to **any specific user or group** without changing ownership.

> 💡 Think of ACLs as a permission extension layer on top of standard rwx — standard permissions still apply, ACLs add extra rules on top.

### View ACLs

```bash
getfacl file.txt              # Show all ACL entries for a file
getfacl -R /shared/           # Recursive — show ACLs for entire directory tree
```

A `+` at the end of `ls -l` output signals that ACL entries exist:
```bash
ls -l important.txt
# -rw-r--r--+ 1 alice devs ... important.txt
#            ↑ this + means ACLs are set
```

### Set ACLs

```bash
setfacl -m u:bob:rw file.txt         # Give user bob read+write on this file
setfacl -m g:interns:r /reports/     # Give interns group read-only on a directory
setfacl -m u:deploy:rwx deploy.sh    # Give deploy user full access to a specific script
```

### Recursive ACL

```bash
setfacl -R -m g:interns:rX /reports/   # Recursive: r = read files, X = execute dirs only (not files)
```

> 💡 Capital `X` (not lowercase `x`) is important here — it adds execute only to **directories**, not regular files. Using lowercase `x` recursively would make all files executable, which is almost never what you want.

### Default ACLs (Inherited by New Files)

```bash
setfacl -m d:u:bob:rw /shared-folder/    # d: prefix = default ACL
                                          # Any new file created inside will automatically get bob:rw
```

### Remove ACLs

```bash
setfacl -x u:bob file.txt     # Remove only bob's ACL entry (other entries untouched)
setfacl -b file.txt           # Remove ALL ACL entries from the file entirely
```

### 🏭 Production-Grade Example — CI/CD Deploy User Access

Scenario: A CI/CD pipeline runs as user `deploy`. It needs write access to `/var/www/app` but you don't want to change the ownership away from `www-data`.

```bash
# Grant deploy user read+write+execute on the app directory
setfacl -R -m u:deploy:rwX /var/www/app/        # Recursive on existing files
setfacl -R -m d:u:deploy:rwX /var/www/app/      # Default — applies to new files created inside too

# Verify
getfacl /var/www/app/
```

This is cleaner than adding `deploy` to the `www-data` group, which would give broader access than needed.

---

## 7️⃣ Permission Denied — Debugging Checklist

When you hit `Permission denied`, work through this in order:

```bash
# Step 1 — Check the file's permissions and owner
ls -l /path/to/file

# Step 2 — Check the PARENT DIRECTORY's permissions (often the real culprit)
ls -ld /path/to/

# Step 3 — Check what groups your user belongs to
id
groups

# Step 4 — Check if ACLs are involved (look for + in ls -l output)
getfacl /path/to/file

# Step 5 — Check effective permissions as a specific user
sudo -u www-data ls /var/www/app/    # Can www-data actually see this?

# Step 6 — Check umask if the file was just created
umask
```

---

## Quick Reference

```bash
ls -l                                     # View permissions
ls -ld /dir/                              # View directory's own permissions
stat file.txt                             # Full metadata including permissions

chmod 755 script.sh                       # Numeric chmod
chmod u+x script.sh                       # Symbolic chmod
chmod -R 750 /srv/app                     # Recursive chmod ⚠️

chown alice:developers file.txt           # Change owner and group
chown -R www-data:www-data /var/www/      # Recursive ownership change

umask                                     # View current umask
umask 027                                 # Set stricter umask for this session

getfacl file.txt                          # View ACL entries
setfacl -m u:deploy:rwx file.txt         # Add ACL entry
setfacl -R -m d:u:deploy:rX /shared/     # Recursive + default ACL
setfacl -b file.txt                       # Remove all ACLs
```

---

## Key Takeaways

- Permissions apply to three classes — owner, group, others — in that priority order
- **Directory `x` is not optional** — without it, nothing inside is accessible regardless of file permissions
- Numeric mode sets permissions absolutely; symbolic mode adds or removes without touching the rest
- `S` or `T` (uppercase) in permission strings means the special bit is set but the underlying `x` is missing — usually a misconfiguration
- `umask 022` is the safe default; `002` for team collaboration; `077` for secrets
- ACLs solve the "one group is not enough" problem — use them for CI/CD users and service accounts
- Capital `X` in `setfacl` applies execute to directories only — always use it over lowercase `x` in recursive ACL operations
- When debugging `Permission denied`, check the **parent directory** first — it's the most overlooked step

---
