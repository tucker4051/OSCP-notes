# Linux Privilege Escalation – Linux Services & Internals

## Overview

Once basic environment enumeration is complete, the next step is to examine the **internal behaviour of the host** in more depth. This stage focuses on the services, processes, sockets, users, history, logs, and system internals that may reveal privilege escalation paths.

At this point, key questions include:

- What services and applications are installed?
- What services are currently running?
- What sockets and network connections are in use?
- What users, admins, and groups exist?
- Who is logged in now, and who logged in recently?
- Are any password policies enforced?
- Is the host joined to Active Directory?
- Are there useful commands, logs, backups, or history files?
- Are cron jobs running that may be hijackable?
- What tools are installed that may help post-exploitation?
- Are there interesting network interfaces, hosts entries, or pivot routes?

This phase helps move from “what exists” to “how the system really operates.”

---

# 1. Network Interfaces and IP Addressing

A good starting point is identifying the host’s network configuration.

```bash
ip a
```

Example:

```bash
2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 10.129.203.168/16
```

This tells us:

- the current IP address
- the interface name
- whether multiple interfaces exist
- whether the host may be dual-homed and usable for pivoting

If `ifconfig` is unavailable, it is often because the `net-tools` package is not installed.

---

# 2. /etc/hosts

The hosts file may contain useful internal names.

```bash
cat /etc/hosts
```

Example:

```text
127.0.0.1 localhost
127.0.1.1 nixlpe02
```

Interesting findings may include:

- internal application names
- alternate hostnames
- development or staging references
- hardcoded mappings that help pivoting or lateral movement

---

# 3. Recently Logged-in Users

Check when users last logged in:

```bash
lastlog
```

Example:

```text
mrb3n            pts/1    10.10.14.15
cliff.moore      pts/0    127.0.0.1
stacey.jenkins   pts/0    10.10.14.15
```

This helps determine:

- which accounts are genuinely used
- whether admins log in interactively
- how active the system is
- which accounts may be worth targeting

A frequently used system is more likely to contain:

- messy permissions
- useful history
- recently modified scripts
- cached credentials

---

# 4. Currently Logged-in Users

Check who is on the system right now:

```bash
w
```

Example:

```text
USER      TTY   FROM          LOGIN@   WHAT
cliff.mo  pts/0 10.10.14.16   Tue19    -bash
```

This can indicate:

- active admins or service users
- opportunities for credential theft
- the risk of disrupting another user’s work
- who may own recent files or processes

Useful alternatives:

```bash
who
finger
```

---

# 5. Command History

User history often contains highly valuable information.

```bash
history
```

Example:

```bash
1  id
2  cd /home/cliff.moore
3  exit
4  touch backup.sh
5  tail /var/log/apache2/error.log
6  ssh ec2-user@dmz02.inlanefreight.local
```

Things to look for:

- passwords passed on the command line
- SSH targets
- cron setup
- backup scripts
- git operations
- internal hostnames
- cloud logins

History often reveals how the box is used and what adjacent systems exist.

---

# 6. Searching for History Files

Some systems or tools create extra history files beyond `.bash_history`.

```bash
find / -type f \( -name *_hist -o -name *_history \) -exec ls -l {} \; 2>/dev/null
```

Example:

```text
/home/htb-student/.bash_history
```

Also check:

- `.mysql_history`
- `.psql_history`
- `.lesshst`
- custom script logs
- application audit trails

---

# 7. Cron Jobs

Cron jobs are a major privilege escalation target.

Check scheduled tasks:

```bash
ls -la /etc/cron.daily/
```

Example:

```text
-rwxr-xr-x  1 root root 1478 apt-compat
-rwxr-xr-x  1 root root 1187 dpkg
-rwxr-xr-x  1 root root 1123 man-db
```

You should also inspect:

```bash
ls -la /etc/cron.hourly/
ls -la /etc/cron.weekly/
ls -la /etc/cron.monthly/
cat /etc/crontab
crontab -l
```

Look for:

- writable scripts
- scripts using relative paths
- weak file permissions
- scripts running as root
- references to writable directories

A cron job plus a writable script is often an immediate escalation path.

---

# 8. Proc Filesystem

The `/proc` filesystem exposes live process and kernel information.

For example, process command lines can be inspected with:

```bash
find /proc -name cmdline -exec cat {} \; 2>/dev/null | tr " " "\n"
```

This can reveal:

- commands started with credentials
- ssh sessions
- service startup parameters
- tokens, secrets, or hostnames in command lines

Example indicators:

- `ssh root@host`
- agents
- custom service binaries
- alternate config paths

`/proc` is especially useful when logs are sparse but processes are live.

---

# 9. Installed Packages

Listing installed software helps identify:

- outdated packages
- niche software
- local privilege escalation candidates
- debugging or development tooling

On Debian/Ubuntu:

```bash
apt list --installed | tr "/" " " | cut -d" " -f1,3 | sed 's/[0-9]://g' | tee -a installed_pkgs.list
```

Example packages:

- `apparmor`
- `anacron`
- `apport`
- `snapd`

This list becomes more useful when matched against:

- CVE research
- GTFOBins
- known misconfigurations
- service abuse paths

---

# 10. Sudo Version

Even if `sudo -l` yields nothing useful, the installed sudo version itself may matter.

```bash
sudo -V
```

Example:

```text
Sudo version 1.8.31
```

Older sudo versions may be vulnerable to public exploits, such as:

- Baron Samedit (`CVE-2021-3156`)
- legacy heap-based issues

---

# 11. Available Binaries

List standard executable paths:

```bash
ls -l /bin /usr/bin /usr/sbin
```

This helps identify:

- shells
- interpreters
- compilers
- networking tools
- privilege escalation helpers

Interesting binaries include:

- `bash`
- `dash`
- `find`
- `cp`
- `awk`
- `tar`
- `curl`
- `wget`
- `python`
- `perl`
- `ruby`
- `gcc`
- `nmap`
- `tcpdump`
- `screen`
- `tmux`

These may later be used for:

- shell escapes
- file reads/writes
- privilege escalation
- reverse shells
- persistence

---

# 12. GTFOBins Cross-Check

GTFOBins is extremely useful for discovering binaries that can be abused.

Example one-liner to compare installed packages with GTFOBins:

```bash
for i in $(curl -s https://gtfobins.org/api.json | jq -r '.executables | keys[]'); do
    if grep -q "$i" installed_pkgs.list; then
        echo "Check for GTFO: $i"
    fi
done
```

Example results:

```text
Check GTFO for: apt
Check GTFO for: awk
Check GTFO for: bash
Check GTFO for: cp
Check GTFO for: curl
Check GTFO for: find
```

This does **not** mean those binaries are exploitable by default, only that they may become useful if combined with:

- sudo rights
- SUID
- writable PATH
- capabilities
- service misconfigurations

---

# 13. Using strace

`strace` is useful for tracing system calls made by a program.

Example:

```bash
strace ping -c1 10.129.112.20
```

This helps reveal:

- what files a program opens
- what sockets it uses
- which libraries it loads
- whether it references secrets
- whether it connects to remote hosts

Potential uses during privesc:

- understanding custom binaries
- identifying hidden config paths
- spotting credentials in process behaviour
- detecting network callbacks or auth attempts

---

# 14. Configuration Files

Readable configuration files can expose:

- usernames
- passwords
- API keys
- private file paths
- service architecture

Find them with:

```bash
find / -type f \( -name "*.conf" -o -name "*.config" \) -exec ls -l {} \; 2>/dev/null
```

Examples:

- resolver configs
- NetworkManager files
- service-specific configs
- app configs in user directories

Even if you cannot access the containing directory, world-readable config files may still reveal their contents.

---

# 15. Scripts

Shell scripts often encode operational logic that exposes escalation paths.

Search for them:

```bash
find / -type f -name "*.sh" 2>/dev/null | grep -v "src\|snap\|share"
```

Example findings:

```text
/home/htb-student/automation.sh
/etc/init.d/hwclock.sh
```

Scripts are useful because they may reveal:

- backup logic
- hardcoded credentials
- writable execution paths
- internal file locations
- service behaviour
- insecure temporary file handling

---

# 16. Running Services

Check what root is running:

```bash
ps aux | grep root
```

Example interesting services:

- `cron`
- `cupsd`
- `NetworkManager`
- `gdm3`
- `packagekitd`
- `snapd`

This helps identify:

- root-owned services
- Python services
- custom daemon processes
- software worth version-checking
- cron or unattended upgrade activity

Anything custom or unusual deserves extra attention.

---

# 17. Password Policies and Domain Clues

While Linux privilege escalation is usually local-first, it is worth checking whether the host appears to be:

- domain joined
- using centralized auth
- integrated into Active Directory

Useful indicators:

- `realm list`
- `sssd` configs
- `krb5.conf`
- `ldap.conf`
- `/etc/resolv.conf`
- usernames like `administrator.domain`

Also inspect:

```bash
cat /etc/passwd
```

for domain-style usernames.

---

# 18. Network Connections and Internal Reachability

Current and recent network connections may reveal:

- databases
- internal application servers
- jump boxes
- cloud assets
- admin workstations

Useful commands:

```bash
ss -tulpn
netstat -tulpn
arp -a
route
ip a
```

Look for:

- connections to internal RFC1918 IP ranges
- database ports
- unusual listening services
- alternate subnets reachable through the host

A dual-homed system can often become a pivot point.

---

# 19. File Modification Patterns

Files modified recently may indicate:

- cron activity
- rotating backups
- admin scripts
- temporary deployment jobs

Useful commands:

```bash
find / -type f -mtime -2 2>/dev/null
find / -type f -mmin -60 2>/dev/null
```

If the same file is repeatedly updated, it may be part of an automated task that can be hijacked.

---

# 20. Interesting Tools Installed

It is always worth checking whether the box already includes tools you would otherwise need to upload.

Common useful tools:

- `nc`
- `ncat`
- `curl`
- `wget`
- `python`
- `python3`
- `perl`
- `ruby`
- `gcc`
- `make`
- `socat`
- `tcpdump`
- `nmap`
- `git`
- `ssh`
- `scp`
- `screen`
- `tmux`

These can help with:

- reverse shells
- local compilation
- credential theft
- sniffing
- privilege escalation payloads
- pivoting

---

# 21. Why This Matters

This stage of enumeration is about understanding **how the host behaves**, not just what files exist.

By the end of services and internals enumeration, you should have a good picture of:

- active services
- likely admin activity
- internal connectivity
- cron opportunities
- readable configs and scripts
- useful binaries
- domain clues
- recent operational behaviour

That context is what makes the later privilege escalation checks far more effective.

---

# Quick Command Set

## Network and sessions

```bash
ip a
cat /etc/hosts
route
arp -a
w
lastlog
```

## Processes and services

```bash
ps aux
ps aux | grep root
ss -tulpn
```

## History and cron

```bash
history
find / -type f \( -name *_hist -o -name *_history \) 2>/dev/null
ls -la /etc/cron.daily/
cat /etc/crontab
```

## Packages and binaries

```bash
apt list --installed
sudo -V
ls -l /bin /usr/bin /usr/sbin
```

## Files and configs

```bash
find / -type f \( -name "*.conf" -o -name "*.config" \) 2>/dev/null
find / -type f -name "*.sh" 2>/dev/null
```

## Deep inspection

```bash
find /proc -name cmdline -exec cat {} \; 2>/dev/null | tr " " "\n"
strace <binary>
```

---

# Tags

#linux
#privilege-escalation
#enumeration
#services
#internals
#cron
#procfs
#gtfobins
#sudo
#post-exploitation
#obsidian
#ctf
#pentesting