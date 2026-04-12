# Linux Privilege Escalation – Dirty Pipe

## Overview

**Dirty Pipe** is a Linux kernel vulnerability tracked as **CVE-2022-0847**. It allows an unprivileged local user to overwrite data in files they can read, even when those files are not writable to them under normal permissions.

At a high level, the bug affects how the Linux kernel handles data in **pipes**, which are a standard mechanism for one-way inter-process communication on Unix-like systems.

This issue is often compared to **Dirty COW**, but the underlying bug is different.

---

# Why It Matters

Dirty Pipe is dangerous because it can allow a local user to:

- overwrite protected file contents
- tamper with privileged configurations
- alter security-sensitive files
- escalate privileges to **root** on vulnerable systems

In practical terms, a low-privileged shell on a vulnerable host may be enough to obtain full administrative control.

---

# Affected Kernel Range

Dirty Pipe affects Linux kernels in the following range:

```text
5.8 through 5.17
```

Any target within that range should be treated as potentially vulnerable until patch status is confirmed.

---

# Quick Verification

To check the running kernel version:

```bash
uname -r
```

Example:

```text
5.13.0-46-generic
```

That falls within the affected range and would be a strong indicator to investigate further.

---

# Core Concept

Dirty Pipe is based on misuse of **pipe buffer flags** inside the Linux kernel.

Because of this flaw, an attacker may be able to:

- inject data into page cache
- overwrite file contents they should not be able to modify
- do so without traditional write permission on the target file

A useful mental model is this:

> The kernel mistakenly treats some read-only data as if it were safe to splice modified pipe data into it.

That turns a read-access foothold into a write-capable one.

---

# Prerequisites

For Dirty Pipe to be relevant during privilege escalation, the attacker generally needs:

- local code execution on the host
- a vulnerable kernel version
- read access to the target file or suitable privileged executable target

This is a **local privilege escalation** issue, not a remote exploit by itself.

---

# Common Abuse Paths

Dirty Pipe has historically been used to target:

- `/etc/passwd`
- SUID binaries
- root-owned configuration files
- authentication-related files
- other security-sensitive files that are readable but not writable

Typical outcomes include:

- creating or enabling privileged access
- obtaining a root shell
- hijacking execution flow of privileged binaries

---

# Enumeration

## 1. Check kernel version

```bash
uname -r
uname -a
```

## 2. Confirm OS context

```bash
cat /etc/os-release
cat /etc/lsb-release
```

## 3. Identify interesting privilege targets

Especially:

- authentication files
- SUID binaries
- root-owned config files
- scripts or services that execute as root

---

# Find SUID Binaries

A common enumeration step is to identify SUID binaries on the host:

```bash
find / -perm -4000 2>/dev/null
```

Example output:

```text
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/lib/snapd/snap-confine
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/eject/dmcrypt-get-device
/usr/lib/xorg/Xorg.wrap
/usr/sbin/pppd
/usr/bin/chfn
/usr/bin/su
/usr/bin/chsh
/usr/bin/umount
/usr/bin/passwd
/usr/bin/fusermount
/usr/bin/sudo
/usr/bin/vmware-user-suid-wrapper
/usr/bin/gpasswd
/usr/bin/mount
/usr/bin/pkexec
/usr/bin/newgrp
```

These binaries are valuable because they already execute with elevated privileges and may become viable post-exploitation targets on a vulnerable kernel.

---

# Practical Impact

If Dirty Pipe is exploitable on a host, it can turn a basic foothold into:

- full root compromise
- credential manipulation
- persistence
- lateral movement staging
- theft of secrets from protected locations

On enterprise systems, that can also lead to:

- access to SSH keys
- access to service credentials
- extraction of database secrets
- domain attack prep if the host is domain-joined

---

# Indicators During Assessment

Dirty Pipe should move higher in priority if you observe:

- kernel version in the affected range
- low-privileged shell on a Linux host
- no easy sudo or SUID abuse path
- readable but protected target files
- legacy or poorly patched systems

This is especially relevant on:

- older Ubuntu hosts
- lab environments
- developer workstations
- mismanaged internal servers
- appliances with stale kernels

---

# Defensive Perspective

From a blue-team or hardening perspective, the main defenses are:

## 1. Patch the kernel
Dirty Pipe is a kernel-level issue, so remediation primarily means updating to a fixed kernel.

## 2. Limit local footholds
Since this is a local privilege escalation, reducing initial code execution paths reduces exposure.

## 3. Restrict risky post-exploitation surfaces
Examples:

- minimize SUID binaries
- reduce unnecessary shell access
- harden service accounts
- use least privilege

## 4. Monitor for unusual file tampering
Watch for unexpected changes to:

- `/etc/passwd`
- SUID binaries
- authentication files
- privileged configs

## 5. EDR / audit visibility
Behavior to watch for includes:

- unusual writes to protected files
- tampering with page-cache-backed files
- suspicious local privilege escalation chains
- abnormal execution from low-privileged users

---

# Validation and Triage Checklist

Use this as a quick assessment checklist:

- [ ] Do I have local code execution?
- [ ] Is the kernel between 5.8 and 5.17?
- [ ] Is the host missing the relevant patch?
- [ ] Are there interesting readable privileged targets?
- [ ] Are there SUID binaries present?
- [ ] Is this a stable enough host to test safely?
- [ ] Is exploitation in-scope and authorized?

---

# Notes for Reporting

If you identify Dirty Pipe on an engagement, include:

- exact kernel version
- affected hostname / asset
- user context obtained initially
- privilege escalation impact
- whether root access was achieved
- risk of credential theft / persistence / lateral movement
- remediation guidance to patch kernel immediately

A concise impact statement could be:

> A local unprivileged user on the affected system can escalate privileges to root due to CVE-2022-0847 (Dirty Pipe), enabling full host compromise.

---

# Example Commands for Safe Enumeration

## Kernel version

```bash
uname -r
```

## Full kernel / architecture details

```bash
uname -a
```

## OS version

```bash
cat /etc/os-release
```

## SUID enumeration

```bash
find / -perm -4000 2>/dev/null
```

---

# Key Takeaways

- Dirty Pipe is a **local kernel privilege escalation**
- affected range is **5.8 to 5.17**
- it allows unauthorized modification of otherwise protected files
- it can lead directly to **root**
- any low-privileged shell on a vulnerable host becomes much more serious
- patching the kernel is the primary fix

---

# Tags

#linux
#privilege-escalation
#dirty-pipe
#cve-2022-0847
#kernel-exploit
#post-exploitation
#enumeration
#defensive-notes
#obsidian