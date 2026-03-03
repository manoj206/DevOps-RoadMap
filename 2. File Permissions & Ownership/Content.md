## Linux Module 2: File Permissions & Ownership (Security Foundation)

File permissions are the **core security mechanism of Linux**.
They control **who can read, write, or execute** files and directories.

Misconfigured permissions are one of the fastest ways to:

* Create security vulnerabilities
* Break application deployments
* Lock yourself (or a service) out of required files

---

## 1️⃣ Basic Permission Classes & rwx Breakdown

Every file/directory has three classes of users:

* **owner** (`u`) — the file’s owner
* **group** (`g`) — users in the assigned group
* **others** (`o`) — everyone else

Each class has three permission types:

| Permission | Meaning | On Files              | On Directories                             | Numeric |
| ---------- | ------- | --------------------- | ------------------------------------------ | ------- |
| **r**      | read    | View file contents    | List contents (`ls`)                       | 4       |
| **w**      | write   | Modify or delete file | Create, rename, delete items inside        | 2       |
| **x**      | execute | Run program or script | Enter directory (`cd`) and access contents | 1       |

Example:

```bash
-rwxr-xr--  1 alice developers 4096 Mar 3 18:30 script.sh
```

Breakdown:

* `rwx` → owner (`alice`) has full access
* `r-x` → group (`developers`) can read & execute
* `r--` → others can only read

---

## 2️⃣ `chmod` — Change Mode

### 🔹 Symbolic Mode (Recommended for Day-to-Day Use)

Format:

```
chmod [who][operator][permissions] file
```

* who → `u`, `g`, `o`, `a`
* operator → `+`, `-`, `=`
* permission → `r`, `w`, `x`

Examples:

```bash
chmod u+x script.sh
chmod g-w report.txt
chmod o-rwx secret.key
chmod a+x runme.sh
chmod u=rw,g=r,o= config.ini
chmod +x *.sh
```

---

### 🔹 Numeric (Octal) Mode

Each class gets a number from 0–7 (sum of 4+2+1).

| Octal | Meaning   | Common Use Case              |
| ----- | --------- | ---------------------------- |
| 644   | rw-r--r-- | Standard files               |
| 755   | rwxr-xr-x | Executables & public folders |
| 700   | rwx------ | Private files                |
| 750   | rwxr-x--- | Team-shared executables      |
| 600   | rw------- | Private keys                 |

Examples:

```bash
chmod 644 public.txt
chmod 755 myscript.sh
chmod 700 ~/.ssh/id_ed25519
chmod 4755 /usr/local/bin/mysudo   # special bit included
```

---

## 3️⃣ Special Permissions (setuid, setgid, sticky bit)

These modify the **execute position**.

| Bit    | Octal | Symbolic | Effect on Files    | Effect on Directories           | Common Example          |
| ------ | ----- | -------- | ------------------ | ------------------------------- | ----------------------- |
| setuid | 4xxx  | u+s      | Run as file owner  | Rarely used                     | `/usr/bin/passwd`       |
| setgid | 2xxx  | g+s      | Run as group owner | New files inherit group         | Shared team directories |
| sticky | 1xxx  | +t       | Mostly ignored     | Only file owner/root can delete | `/tmp`                  |

Examples:

```bash
chmod u+s dangerous-tool
chmod g+s /shared/team
chmod +t /myproject/uploads
```

Real-world analogy:

* setuid → Users can run a tool, but it behaves as the owner.
* setgid (directory) → Everything created inside belongs to the team.
* sticky bit → Public folder where only creators can delete their own files.

---

## 4️⃣ Ownership Management

### Change Owner

```bash
chown alice script.sh
chown alice:developers report.xlsx
chown -R www-data:www-data /var/www
```

### Change Group Only

```bash
chgrp devs /shared/folder
```

Only root (or privileged users) can transfer ownership.

---

## 5️⃣ `umask` — Default Permissions

New files start at:

* Files → 666
* Directories → 777

`umask` subtracts permissions from this maximum.

| umask | New File | New Directory | Scenario               |
| ----- | -------- | ------------- | ---------------------- |
| 022   | 644      | 755           | Default desktop/server |
| 002   | 664      | 775           | Collaborative teams    |
| 077   | 600      | 700           | High security          |

Examples:

```bash
umask
umask -S
umask 027
```

Think of `umask` as the template that removes permissions before files are created.

---

## 6️⃣ ACLs — Access Control Lists (Extended Permissions)

Traditional Unix permissions are limited to owner/group/others.
ACLs allow per-user and per-group rules.

View ACLs:

```bash
getfacl important.txt
```

Add ACL:

```bash
setfacl -m u:bob:rw important.txt
```

Recursive ACL:

```bash
setfacl -m g:interns:rX -R /reports/
```

Set default ACL for new files:

```bash
setfacl -m d:u:bob:rw /shared-folder/
```

Remove ACL entry:

```bash
setfacl -x u:bob file.txt
```

Remove all ACLs:

```bash
setfacl -b file.txt
```

---

## 7️⃣ Common Real-World Scenarios

### Permission Denied Debugging

Checklist:

1. Check file permissions → `ls -l`
2. Check ownership → `ls -l`
3. Check directory permissions (often overlooked)
4. Check group membership → `groups username`
5. Check ACLs → `getfacl file`
6. Check umask (for recently created files)

---

### Recursive Risks

Be cautious with:

```bash
chmod -R 777 /
chmod -R 755 /var
```

Recursive permission changes are powerful — and destructive if misused.

Always test on a sample directory first.

---

## Quick Reference – Most Used Commands

```bash
ls -l
chmod 755 script.sh
chmod -R 750 /srv/app
chmod u+s binary
chmod g+s /data/team
chmod +t /shared/uploads
chown -R www-data:www-data /var/www/html
umask 027
getfacl file.txt
setfacl -m u:deploy:rwx key.pem
```

---

## Summary

This module covers:

* Permission model (rwx)
* Numeric vs symbolic chmod
* Special permission bits
* Ownership control
* Default permission behavior (umask)
* Advanced access control (ACLs)
* Practical troubleshooting patterns

These are foundational skills for secure deployments, service reliability, and production-safe automation.

---