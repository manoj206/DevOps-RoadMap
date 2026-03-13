# Linux Module 2: File Permissions & Ownership — Practice Notes

This document contains practical examples and detailed explanations for working with:

* Reading and interpreting permission strings
* `chmod` — symbolic and numeric modes
* Special permissions — setuid, setgid, sticky bit
* `chown` and `chgrp` — ownership management
* `umask` — default permission behavior
* ACLs — `getfacl` and `setfacl`
* Real-world permission debugging

---

# Reading Permission Strings

## Create a Sample File and Script

```bash
echo "database_host=prod-db.internal" > config.txt   # Create a config file with content
touch deploy.sh                                        # Create an empty shell script
ls -l                                                  # View permissions of both files
```

Expected output:
```
-rw-r--r-- 1 alice devs 30 Mar 10 config.txt
-rw-r--r-- 1 alice devs  0 Mar 10 deploy.sh
```

### Breaking Down the Permission String

```
-  rw-  r--  r--
│   │    │    │
│   │    │    └── others  : r-- → read only
│   │    └─────── group   : r-- → read only
│   └──────────── owner   : rw- → read + write
└──────────────── file type: - = file, d = directory, l = symlink
```

### File Type Characters

```bash
ls -l /etc/hosts          # - at the start → regular file
ls -ld /etc/              # d at the start → directory
ls -l /etc/localtime      # l at the start → symbolic link
ls -l /dev/sda            # b at the start → block device
```

### Inspecting Full File Metadata

```bash
stat config.txt           # Shows permissions in both octal and symbolic form,
                          # plus inode, owner, size, and all timestamps
```

Expected output:
```
  File: config.txt
  Size: 30
  Inode: 123456
  Access: (0644/-rw-r--r--)   Uid: (1001/alice)   Gid: (1002/devs)
  Access: 2024-03-10 09:00:00
  Modify: 2024-03-10 09:00:00
```

> `stat` gives you the octal value directly — useful when you need to confirm the exact permission number without calculating it manually.

---

# `chmod` — Symbolic Mode

## Add and Remove Permissions

```bash
chmod u+x deploy.sh         # Add execute to owner — now owner can run the script
ls -l deploy.sh             # Verify: should now show -rwxr--r--

chmod g+w config.txt        # Add write permission for group members
chmod o-r config.txt        # Remove read from others — others can no longer view the file
ls -l config.txt            # Verify: should show -rw-rw----
```

## The `=` Operator — Set Exactly

```bash
chmod u=rw,g=r,o= config.txt   # Set precisely: owner=rw, group=r, others=nothing
                                # = replaces whatever was there — does not add or remove
ls -l config.txt               # Should show: -rw-r-----
```

> Use `=` when you want to guarantee the full permission state, not just add or remove one bit.

## Apply to Multiple Files

```bash
chmod +x *.sh                  # Add execute to ALL .sh files in current directory
ls -l *.sh                     # Verify all scripts are now executable
```

---

# `chmod` — Numeric (Octal) Mode

## The Calculation

Each permission type has a fixed value:

```
r = 4
w = 2
x = 1
- = 0
```

Add the values per class:

```
rwx = 4+2+1 = 7
r-x = 4+0+1 = 5
r-- = 4+0+0 = 4
--- = 0+0+0 = 0
```

So `chmod 754 file` means: owner=`rwx`(7), group=`r-x`(5), others=`r--`(4)

## Standard Values in Practice

```bash
chmod 644 config.txt           # rw-r--r-- : owner reads/writes, everyone else reads
                               # Standard for config files, web assets, text files

chmod 755 deploy.sh            # rwxr-xr-x : owner full, group+others read+execute
                               # Standard for scripts and public-facing directories

chmod 600 secret.key           # rw------- : owner reads/writes only, nobody else sees it
                               # Required for SSH private keys — SSH rejects weaker permissions

chmod 700 private-script.sh    # rwx------ : only owner can read, write, execute
                               # Personal scripts with sensitive logic
```

## Verify the Octal Value of Existing Files

```bash
stat -c "%a %n" config.txt     # %a = octal permissions, %n = filename
stat -c "%a %n" *.sh           # Check octal on all .sh files at once
```

## Recursive chmod

```bash
chmod -R 750 /srv/app/         # Apply 750 recursively to all files and dirs inside
                               # ⚠️ This gives execute to ALL files including non-scripts
                               # Use with caution — test on a subdirectory first
```

### Safer Recursive Pattern — Separate Files and Directories

```bash
# Give directories 755 and files 644 — without making everything executable
find /srv/app -type d -exec chmod 755 {} \;   # Apply 755 to directories only
find /srv/app -type f -exec chmod 644 {} \;   # Apply 644 to files only
find /srv/app -name "*.sh" -exec chmod 755 {} \;  # Then explicitly fix scripts
```

> This is the production-safe approach to recursive permission changes.

---

# Special Permissions

## Setup

```bash
mkdir -p /tmp/special-demo/{uploads,team,tools}   # Create directories for each demo
ls -ld /tmp/special-demo/                         # Verify directory was created
```

---

## Sticky Bit — Shared Upload Directory

```bash
chmod 1777 /tmp/special-demo/uploads    # 1 = sticky bit, 777 = world-writable
                                        # Anyone can create files, but only owners can delete theirs
ls -ld /tmp/special-demo/uploads        # Should show: drwxrwxrwt — the 't' is the sticky bit
```

### Observe Sticky Bit Behaviour

```bash
# As user alice — create a file
touch /tmp/special-demo/uploads/alice-file.txt

# As a different user (bob) — try to delete alice's file
rm /tmp/special-demo/uploads/alice-file.txt
# Output: rm: cannot remove 'alice-file.txt': Operation not permitted
# Because sticky bit protects files from deletion by non-owners
```

### Real-world reference

```bash
ls -ld /tmp          # /tmp on any Linux system already has the sticky bit set
                     # Output: drwxrwxrwt — this is why multiple users can share /tmp safely
```

---

## setgid — Shared Team Directory

```bash
groupadd devteam                              # Create a shared group (run as root)
chgrp devteam /tmp/special-demo/team/         # Assign the team group to the directory
chmod 2775 /tmp/special-demo/team/            # 2 = setgid, 775 = group can write
                                              # New files created inside will inherit 'devteam' group
ls -ld /tmp/special-demo/team/               # Should show: drwxrwsr-x — 's' in group position
```

### Observe setgid Behaviour

```bash
# As any user in 'devteam' — create a file inside
touch /tmp/special-demo/team/shared-doc.txt
ls -l /tmp/special-demo/team/shared-doc.txt
# Output: -rw-r--r-- 1 alice devteam ... shared-doc.txt
#                              ↑ group is 'devteam', not alice's primary group
```

> Without setgid, the file would inherit the creating user's default group. With it, all files automatically belong to the team — no manual `chgrp` needed every time.

---

## setuid — Inspect a Real Example

```bash
ls -l /usr/bin/passwd            # Check the passwd binary on your system
# Output: -rwsr-xr-x 1 root root ... /usr/bin/passwd
#               ↑ 's' in owner position = setuid
#         Any user runs this binary, but it executes AS root
#         This is how a regular user can change their own password
#         (which requires writing to /etc/shadow, a root-only file)
```

```bash
# Verify setuid is set using stat
stat -c "%a %n" /usr/bin/passwd  # Should show 4755 — the 4 is the setuid digit
```

> ⚠️ Never set setuid on shell scripts — Linux ignores it for security reasons. It only works on compiled binaries. Setting setuid on a script is a common mistake that silently does nothing.

### Spotting a Misconfigured Special Bit

```bash
chmod u+s config.txt             # Set setuid WITHOUT execute permission already set
ls -l config.txt                 # Shows: -rwSr--r-- — uppercase 'S' = setuid set but no 'x'
                                 # This is almost always a misconfiguration
chmod u+x config.txt             # Add execute — now it shows lowercase 's' (correct)
```

---

# `chown` and `chgrp`

## Basic Ownership Changes

```bash
touch testfile.txt                          # Create a test file
ls -l testfile.txt                          # Note current owner and group

sudo chown bob testfile.txt                 # Change owner to bob
ls -l testfile.txt                          # Verify — owner is now bob

sudo chown alice:devs testfile.txt          # Change owner AND group in one command
ls -l testfile.txt                          # Verify — owner=alice, group=devs

sudo chown :devops testfile.txt             # Change group only (colon with no user)
ls -l testfile.txt                          # Verify — group changed, owner unchanged
```

## Checking Group Membership

```bash
id                             # Show your current uid, gid, and all groups
groups                         # Simpler — just list group names
groups alice                   # Check group membership for a specific user
```

> If you add a user to a group and the change seems to have no effect on their session, they need to **log out and log back in** for the new group membership to be reflected.

## Recursive Ownership Change

```bash
sudo chown -R www-data:www-data /var/www/myapp/   # Recursively change all files and subdirs
                                                   # Essential after deploying app files as root
                                                   # www-data is the typical web server user (nginx/apache)
ls -la /var/www/myapp/                             # Verify ownership changed throughout
```

---

## 🏭 Production-Grade Example — Post-Deployment Ownership Fix

```bash
# Scenario: CI/CD pipeline copied new app files to server as root
# Problem: app runs as 'www-data' and can't read its own files
# Fix: correct ownership and permissions after deployment

sudo chown -R www-data:www-data /var/www/myapp/       # Give web server user ownership of all files

find /var/www/myapp -type d -exec chmod 750 {} \;     # Directories: owner full, group read+execute
find /var/www/myapp -type f -exec chmod 640 {} \;     # Files: owner read/write, group read only

chmod 640 /var/www/myapp/config/database.yml          # Config with credentials — extra restrictive
chmod 750 /var/www/myapp/bin/*.sh                     # Deployment scripts need execute

ls -la /var/www/myapp/                                # Final verification
```

---

# `umask` — Default Permissions

## Observe the Current umask

```bash
umask              # Display current umask as octal (e.g. 0022)
umask -S           # Display in symbolic form: u=rwx,g=rx,o=rx
```

## How umask Subtracts from the Maximum

```bash
# Files start at 666 maximum (no execute by default)
# Dirs start at 777 maximum

# With umask 022:
#   666 - 022 = 644  → new files are rw-r--r--
#   777 - 022 = 755  → new dirs are rwxr-xr-x

touch newfile.txt      # Create a file with current umask active
mkdir newdir           # Create a directory with current umask active
ls -l newfile.txt      # Should show 644 (with default umask 022)
ls -ld newdir/         # Should show 755
```

## Change umask and Observe the Difference

```bash
umask 077              # Set strict umask — owner only, no group or others access
touch private.txt      # Create a file under this new umask
mkdir private-dir      # Create a directory under this new umask
ls -l private.txt      # Should show 600 (rw-------)
ls -ld private-dir/    # Should show 700 (rwx------)

umask 022              # Reset back to standard default
```

## Team Collaboration umask

```bash
umask 002              # Allow group to write (used in team environments)
                       # Files → 664, Dirs → 775
touch team-file.txt    # Create file under collaborative umask
ls -l team-file.txt    # Should show 664 (rw-rw-r--)
```

---

## 🏭 Production-Grade Example — Service umask in a Startup Script

```bash
# Scenario: a background service writes log and temp files
# You want those files to be readable by the app's group but invisible to others

umask 027              # Files created by this service will be 640, dirs will be 750
./start-service.sh     # Any files the service creates inherit this umask

# Verify: files created by the service
ls -l /var/log/myapp/  # Should show 640 for log files, 750 for subdirectories
```

### Making umask Persistent

```bash
# For a single user — add to their shell config
echo "umask 027" >> ~/.bashrc       # Takes effect on next login/new shell
source ~/.bashrc                     # Apply immediately to current session

# For a systemd service unit (preferred for services)
# Add to the [Service] section of the unit file:
# UMask=0027
```

---

# ACLs — `getfacl` and `setfacl`

## Why ACLs Exist

Standard permissions give you one owner and one group. ACLs let you add permissions for
any specific user or group without changing ownership.

```bash
touch report.txt                   # Create a test file
ls -l report.txt                   # Standard permissions — one owner, one group
getfacl report.txt                 # View ACL — initially matches standard permissions
```

Expected getfacl output (no ACLs set yet):
```
# file: report.txt
# owner: alice
# group: devs
user::rw-          ← owner's permissions
group::r--         ← group's permissions
other::r--         ← others' permissions
```

## Add a User ACL

```bash
setfacl -m u:bob:rw report.txt    # -m = modify; give user bob read+write on this file
                                   # Does not change owner or group
getfacl report.txt                 # Verify — bob's entry now appears
ls -l report.txt                   # A '+' appears at end of permission string: -rw-r--r--+
                                   # The '+' signals that ACL entries exist
```

## Add a Group ACL

```bash
setfacl -m g:interns:r report.txt  # Give the interns group read-only access
getfacl report.txt                  # Both bob and interns entries now visible
```

## Recursive ACL — Directory Tree

```bash
mkdir -p /shared/reports/                               # Create a shared directory
setfacl -R -m u:deploy:rwX /shared/reports/            # -R = recursive
                                                        # X (capital) = execute on dirs only
                                                        # NOT on regular files (lowercase x would make files executable)
```

> Capital `X` is critical in recursive ACL operations. Lowercase `x` would make every file executable, which is almost never intended.

## Default ACL — New Files Inherit Permissions

```bash
# Scenario: bob should automatically get rw on every new file created in /shared/reports/
setfacl -m d:u:bob:rw /shared/reports/    # d: prefix = default ACL
                                           # Only applies to NEW files created inside this dir
                                           # Does not affect existing files

touch /shared/reports/new-report.txt       # Create a new file inside
getfacl /shared/reports/new-report.txt    # Verify — bob:rw was automatically inherited
```

## Remove ACL Entries

```bash
setfacl -x u:bob report.txt    # Remove only bob's ACL entry (other entries untouched)
getfacl report.txt              # Verify bob's entry is gone

setfacl -b report.txt           # Remove ALL ACL entries completely
ls -l report.txt                # The '+' should be gone — back to standard permissions
getfacl report.txt              # Confirms clean state
```

---

## 🏭 Production-Grade Example — CI/CD Deploy User Without Changing Ownership

```bash
# Scenario: app is owned by www-data (the web server)
# CI/CD pipeline runs as 'deploy' user and needs to push new files
# Problem: you can't give deploy ownership without breaking the web server's access
# Solution: use ACLs to grant deploy the access it needs without changing ownership

# Grant deploy user access to the app directory
setfacl -R -m u:deploy:rwX /var/www/myapp/          # Recursive on existing files
setfacl -R -m d:u:deploy:rwX /var/www/myapp/        # Default — new files also get deploy access

# Verify both standard permissions and ACL entries
ls -la /var/www/myapp/           # '+' on files confirms ACL is active
getfacl /var/www/myapp/          # Full ACL breakdown — confirm deploy entry is present

# After deployment, optionally tighten back down
# (remove deploy's write access once deployment is done)
setfacl -R -m u:deploy:rX /var/www/myapp/           # Downgrade deploy to read+execute only
```

---

# Permission Debugging — Full Walkthrough

## Scenario: Application Fails With "Permission Denied"

This is a structured checklist with commands to run at each step.

```bash
# Step 1 — Identify the exact file the error is about
# (Check your application's error log)
tail -20 /var/log/myapp/error.log       # Find the file path mentioned in the error

# Step 2 — Check the file's own permissions and owner
ls -l /var/www/myapp/config/app.yml    # Is the owner correct? Are permissions right?

# Step 3 — Check the PARENT directory (most overlooked step)
ls -ld /var/www/myapp/config/          # Directory needs 'x' for the app to enter it
ls -ld /var/www/myapp/                 # Check each level up the path
ls -ld /var/www/                       # A missing 'x' anywhere in the chain blocks access

# Step 4 — Confirm what user the application runs as
ps aux | grep myapp                    # Look at the USER column — that's the effective user
                                       # Could be www-data, myapp, deploy, etc.

# Step 5 — Check that user's group membership
id www-data                            # What groups does www-data belong to?
groups www-data                        # Simpler group check

# Step 6 — Check if ACLs are involved
getfacl /var/www/myapp/config/app.yml  # Any ACL entries? Do they grant or block access?

# Step 7 — Simulate access as the actual service user
sudo -u www-data cat /var/www/myapp/config/app.yml   # Can www-data actually read this file?
sudo -u www-data ls /var/www/myapp/config/            # Can www-data list this directory?
```

### Common Root Causes and Fixes

```bash
# Cause 1: File owned by root, app runs as www-data
sudo chown www-data:www-data /var/www/myapp/config/app.yml

# Cause 2: Directory missing execute bit
sudo chmod 750 /var/www/myapp/config/

# Cause 3: App user not in the right group
sudo usermod -aG devs www-data         # Add www-data to 'devs' group
# (www-data must log out and back in, or restart the service for group to take effect)

# Cause 4: Wrong umask created files with bad permissions
stat -c "%a %n" /var/www/myapp/config/app.yml    # Check actual octal value
chmod 640 /var/www/myapp/config/app.yml           # Fix if needed
```

---

# Key Takeaways

* Always check **parent directory permissions** first when debugging — missing `x` on a directory blocks everything inside regardless of file permissions
* `chmod -R` on a mixed file/directory tree can silently break scripts — use `find` with `-type f` and `-type d` separately for safer recursive changes
* Uppercase `S` or `T` in a permission string means the special bit is set but has no effect — the underlying `x` is missing; this is almost always a misconfiguration
* `stat -c "%a %n"` is the fastest way to get the raw octal value of a file's permissions
* Capital `X` (not lowercase) in `setfacl -R` adds execute to directories only — always use it over lowercase `x` in recursive ACL operations
* `umask 027` is the recommended default for services — files land at `640`, directories at `750`; others get nothing
* ACLs are the right tool when standard owner/group model isn't flexible enough — common for CI/CD pipelines and shared service accounts
* The `+` at the end of `ls -l` output is your signal that ACLs are active — run `getfacl` to see the full picture

---