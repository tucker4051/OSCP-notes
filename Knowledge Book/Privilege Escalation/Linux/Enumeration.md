# Linux Privilege Escalation – Enumeration

## Overview

After obtaining an initial shell on a Linux system, the next objective is typically **privilege escalation to root**.

Root access allows:

- full control of the operating system
- credential harvesting
- access to sensitive configuration files
- persistence mechanisms
- pivoting into other systems
- traffic capture and credential interception
- extraction of domain credentials if domain-joined

Privilege escalation is rarely a single-step exploit — it is usually the result of **systematic enumeration**.

Enumeration is essentially building a complete map of:

> what the system is, what is running on it, who has access, and where weaknesses exist.

Automation tools like **LinPEAS**, **LinEnum**, and **lse** are extremely useful, but manual enumeration is critical for understanding attack paths.

---

# 1. Situational Awareness – First Commands

Immediately after gaining shell access, gather basic context:

```bash
whoami
id
hostname
ip a
sudo -l
```

These answer:

| Question | Why it matters |
|----------|---------------|
| who am I? | confirms current privilege level |
| what groups am I in? | group membership often grants hidden privileges |
| what system is this? | naming conventions may reveal role |
| what network am I in? | possible pivot opportunities |
| do I have sudo rights? | often the quickest escalation path |

Example:

```bash
whoami
htb-student

id
uid=1008(htb-student) gid=1008(htb-student) groups=1008(htb-student),27(sudo)
```

Membership in `sudo` is immediately interesting.

---

# 2. Operating System & Distribution

Understanding the OS helps identify:

- available tools
- default file locations
- package manager
- potential vulnerabilities

```bash
cat /etc/os-release
```

Example:

```bash
NAME="Ubuntu"
VERSION="20.04.4 LTS (Focal Fossa)"
```

Key insight:

Different distributions imply different attack paths:

| Distro | Common package manager |
|--------|------------------------|
| Ubuntu/Debian | apt |
| CentOS/RHEL | yum/dnf |
| Arch | pacman |
| SUSE | zypper |

Outdated distributions often expose known privilege escalation exploits.

---

# 3. Kernel Version

Kernel exploits can provide direct root access.

```bash
uname -a
```

Example:

```bash
Linux nixlpe02 5.4.0-122-generic x86_64 GNU/Linux
```

Search exploit databases for:

```text
Linux kernel 5.4.0 privilege escalation exploit
```

Kernel exploits can crash systems, so should be used carefully in real environments.

---

# 4. Environment Variables

Environment variables sometimes expose credentials or misconfigurations.

```bash
env
```

Example interesting values:

- PATH
- HOME
- USER
- API keys
- tokens
- passwords

Check PATH specifically:

```bash
echo $PATH
```

Misconfigured PATH variables can allow command hijacking.

---

# 5. Running Processes

Processes running as root are prime targets.

```bash
ps aux | grep root
```

Look for:

- custom scripts
- unusual binaries
- outdated software
- services known to contain vulnerabilities

Example vulnerable services historically:

- Nagios
- Exim
- Samba
- ProFTPd

Misconfigured services running as root often lead to escalation.

---

# 6. Installed Packages

Outdated software may contain known privilege escalation exploits.

Check installed packages:

```bash
dpkg -l
```

or:

```bash
rpm -qa
```

Example:

Screen version 4.05.00 is vulnerable to privilege escalation.

---

# 7. Logged-in Users

Identify other active users:

```bash
ps au
```

Active sessions may reveal:

- administrators logged in
- shared accounts
- lateral movement opportunities

Example:

```bash
cliff.moore pts/0 bash
root tty1 sudo su
```

---

# 8. User Accounts

List all users:

```bash
cat /etc/passwd
```

Key fields:

| Field | Meaning |
|------|--------|
| username | account name |
| UID | user ID |
| GID | group ID |
| home directory | possible credential storage |
| shell | login capability |

Example:

```text
stacey.jenkins:x:1007:1007::/home/stacey.jenkins:/bin/bash
```

Accounts with valid shells are more useful targets.

Filter login-enabled users:

```bash
grep "sh$" /etc/passwd
```

---

# 9. User Groups

Group membership often grants hidden privileges.

```bash
cat /etc/group
```

Check interesting groups:

| Group | Why important |
|------|--------------|
| sudo | admin privileges |
| adm | log file access |
| docker | container breakout potential |
| lxd | container root escape |
| disk | raw disk read access |

Example:

```bash
getent group sudo
```

---

# 10. Home Directories

User home directories often contain:

- credentials
- scripts
- SSH keys
- configuration files
- API tokens

```bash
ls /home
```

Check each directory:

```bash
ls -la /home/username
```

Look for:

- .bash_history
- .ssh/
- config files
- backup scripts

Example:

```bash
ls -la /home/stacey.jenkins
```

---

# 11. SSH Keys

SSH private keys can allow:

- persistent access
- lateral movement
- pivoting into internal systems

```bash
ls -l ~/.ssh
```

Example:

```bash
id_rsa
id_rsa.pub
```

Private keys may allow login to:

- same host
- other hosts
- domain-connected machines

---

# 12. Bash History

Users often leave sensitive commands in history:

```bash
history
```

Look for:

- passwords in commands
- SSH connections
- backup scripts
- database access commands

Example:

```bash
ssh ec2-user@dmz02.inlanefreight.local
```

This reveals another host.

---

# 13. Sudo Privileges

One of the most common escalation paths.

```bash
sudo -l
```

Example:

```bash
(root) NOPASSWD: /usr/sbin/tcpdump
```

NOPASSWD allows execution without credentials.

Some binaries allow shell escape.

Always check:

GTFOBins reference.

---

# 14. Configuration Files

Configuration files frequently contain credentials.

Search for:

```bash
find / -name "*.conf" 2>/dev/null
find / -name "*.config" 2>/dev/null
```

Look for:

- database credentials
- API tokens
- service passwords

---

# 15. Password Hashes

Check:

```bash
cat /etc/shadow
```

if readable.

Hashes can be cracked offline.

Hash types:

| prefix | algorithm |
|--------|----------|
| $1$ | MD5 |
| $5$ | SHA-256 |
| $6$ | SHA-512 |
| $2a$ | bcrypt |
| $argon2 | Argon2 |

Hashes sometimes appear directly in `/etc/passwd`.

---

# 16. Cron Jobs

Cron jobs often execute scripts as root.

```bash
ls -la /etc/cron.daily/
```

Look for:

- writable scripts
- scripts using relative paths
- scripts calling other scripts

Writable cron scripts often lead to privilege escalation.

---

# 17. Mounted Drives

Additional drives may contain:

- backups
- credentials
- scripts
- database dumps

Check block devices:

```bash
lsblk
```

Check mounted filesystems:

```bash
df -h
```

Check fstab:

```bash
cat /etc/fstab
```

Credentials may appear inside mount definitions.

---

# 18. Network Information

Understanding network layout helps identify pivot paths.

Routing table:

```bash
route
```

ARP cache:

```bash
arp -a
```

DNS config:

```bash
cat /etc/resolv.conf
```

Internal DNS servers often indicate domain membership.

---

# 19. Writable Directories

Writable directories allow:

- tool uploads
- script modification
- cron abuse

Find writable directories:

```bash
find / -path /proc -prune -o -type d -perm -o+w 2>/dev/null
```

Common writable locations:

- /tmp
- /var/tmp
- /dev/shm

---

# 20. Writable Files

Writable scripts may be executed by root.

```bash
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null
```

Look for:

- backup scripts
- cron scripts
- service configs

Example:

```bash
/etc/cron.daily/backup
```

---

# 21. SUID / SGID Binaries

SUID allows execution as file owner (often root).

Search:

```bash
find / -perm -4000 -type f 2>/dev/null
```

Many SUID binaries allow shell escape.

---

# 22. Hidden Files

Hidden files may contain credentials or notes.

Find hidden files:

```bash
find / -type f -name ".*" 2>/dev/null
```

Hidden directories:

```bash
find / -type d -name ".*" 2>/dev/null
```

Common examples:

- .ssh
- .git
- .env
- .config

---

# 23. Temporary Directories

Temporary directories often contain:

- logs
- credentials
- scripts
- application artifacts

Check:

```bash
ls -l /tmp
ls -l /var/tmp
ls -l /dev/shm
```

Differences:

| directory | persistence |
|----------|-------------|
| /tmp | cleared on reboot |
| /var/tmp | longer persistence |
| /dev/shm | memory-backed storage |

---

# 24. Defensive Controls

Identify security controls:

- AppArmor
- SELinux
- Fail2ban
- iptables
- ufw
- Snort

These affect exploitability.

---

# 25. Automation Tools

Helpful enumeration tools:

- LinPEAS
- LinEnum
- lse

Manual enumeration remains essential when:

- tools cannot be uploaded
- environment restricts execution
- stealth is required

---

# 26. Enumeration Mindset

Privilege escalation is pattern recognition.

We are searching for:

- trust boundaries
- permission mistakes
- exposed credentials
- unsafe automation
- outdated software
- unintended execution paths

Enumeration builds the dataset required to identify these weaknesses.

---

# 27. Quick Enumeration Checklist

Initial orientation:

```bash
whoami
id
hostname
ip a
sudo -l
```

System info:

```bash
cat /etc/os-release
uname -a
env
echo $PATH
```

Users:

```bash
cat /etc/passwd
cat /etc/group
ls /home
```

Processes:

```bash
ps aux
ps au
```

Files:

```bash
find / -perm -4000 -type f 2>/dev/null
find / -type d -perm -o+w 2>/dev/null
find / -type f -perm -o+w 2>/dev/null
```

Storage:

```bash
lsblk
df -h
cat /etc/fstab
```

Network:

```bash
ip a
route
arp -a
cat /etc/resolv.conf
```

History & credentials:

```bash
history
ls -la ~/.ssh
```

Cron:

```bash
ls -la /etc/cron*
```

---

# Tags

#linux
#privilege-escalation
#enumeration
#linpeas
#linenum
#suid
#sudo
#cron
#linux-security
#pentesting
#ctf
#post-exploitation
#obsidian