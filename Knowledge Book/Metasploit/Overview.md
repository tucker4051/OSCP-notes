# Metasploit Framework — Overview

## Purpose

Metasploit is a modular exploitation framework used to:

- identify vulnerabilities
- develop and execute exploits
- generate payloads
- maintain access
- automate post-exploitation tasks

Understanding its structure helps:

- select correct modules faster
- troubleshoot issues
- create custom modules
- operate efficiently under exam conditions

---

# 1. Metasploit Architecture

Default installation path:

```bash
/usr/share/metasploit-framework
```

---

## Core Directories

### Base Framework Components

```bash
ls /usr/share/metasploit-framework
```

Key directories:

| Directory | Purpose |
|----------|---------|
| data | configuration files, wordlists, binaries |
| documentation | framework documentation |
| lib | core libraries powering MSF functionality |

---

# 2. Modules Structure

Modules are the functional core of Metasploit.

```bash
ls /usr/share/metasploit-framework/modules
```

Output:

```text
auxiliary
encoders
evasion
exploits
nops
payloads
post
```

---

## Module Categories

### Exploits

Code that takes advantage of vulnerabilities.

```text
exploit/windows/smb/ms17_010_eternalblue
```

---

### Payloads

Code executed after successful exploitation.

Examples:

```text
reverse shells
meterpreter shells
bind shells
```

---

### Auxiliary

Scanning, fuzzing, enumeration tools.

Examples:

```text
port scanning
SMB enumeration
SNMP enumeration
service discovery
```

---

### Post

Post-exploitation modules.

Examples:

```text
credential dumping
privilege escalation
persistence
data extraction
```

---

### Encoders

Modify payloads to avoid detection.

Example use:

```text
AV evasion
signature bypass
```

---

### NOPs

No-operation instructions used to pad buffers.

Purpose:

```text
ensure reliable shellcode execution
avoid memory corruption issues
```

---

### Evasion

Modules designed to bypass:

```text
IDS
IPS
AV
EDR
```

---

# 3. Plugins

Plugins extend functionality of msfconsole.

```bash
ls /usr/share/metasploit-framework/plugins
```

Examples:

| Plugin | Function |
|--------|----------|
| db_tracker | track credentials |
| session_notifier | session alerts |
| sqlmap | integration |
| nessus | vulnerability scanning |
| openvas | vulnerability scanning |

---

# 4. Scripts

Contains scripts used by Meterpreter and automation features.

```bash
ls /usr/share/metasploit-framework/scripts
```

Directories:

```text
meterpreter
shell
resource
ps
```

---

# 5. Tools Directory

Standalone utilities:

```bash
ls /usr/share/metasploit-framework/tools
```

Examples:

| Directory | Purpose |
|----------|---------|
| exploit | exploit helpers |
| payloads | payload generation |
| recon | reconnaissance tools |
| password | password utilities |

---

# 6. Launching Metasploit

Start console:

```bash
msfconsole
```

Quiet mode:

```bash
msfconsole -q
```

---

# Update Framework

```bash
sudo apt update && sudo apt install metasploit-framework
```

---

# 7. Basic MSF Commands

Show help menu:

```bash
help
```

List modules:

```bash
show modules
```

Search modules:

```bash
search smb
```

Use module:

```bash
use exploit/windows/smb/psexec
```

Show options:

```bash
options
```

---

# 8. Enumeration Importance

Metasploit depends heavily on accurate enumeration.

Key questions:

- which services are exposed?
- which versions are running?
- which vulnerabilities apply?
- what attack surface exists?

Example enumeration tools:

```text
nmap
nikto
enum4linux
ldapsearch
snmpwalk
```

Version identification is critical:

```text
vulnerabilities are version dependent
```

---

# 9. MSF Engagement Structure

Metasploit aligns with pentesting workflow:

---

## 1. Enumeration

Identify:

- services
- versions
- attack vectors

Example modules:

```text
auxiliary/scanner/*
```

---

## 2. Preparation

Select:

- exploit
- payload
- target parameters

Example:

```text
set RHOSTS
set LHOST
set payload
```

---

## 3. Exploitation

Execute exploit:

```bash
exploit
run
```

Goal:

gain shell/session.

---

## 4. Privilege Escalation

Increase privileges:

```text
SYSTEM
root
administrator
```

Example modules:

```text
post/multi/recon/*
post/windows/escalate/*
```

---

## 5. Post-Exploitation

Actions:

- credential dumping
- persistence
- pivoting
- data exfiltration

Example modules:

```text
post/windows/gather/*
post/linux/gather/*
```

---

# 10. Mental Model

Think of Metasploit as:

```text
database of exploits
+ payload generator
+ session manager
+ post exploitation toolkit
```

---

# Quick Reference

## start msf

```bash
msfconsole
```

---

## quiet mode

```bash
msfconsole -q
```

---

## search modules

```bash
search <service>
```

---

## use module

```bash
use <module>
```

---

## show options

```bash
options
```

---

## run exploit

```bash
exploit
```

---

# Related Notes

- msfvenom
- meterpreter
- auxiliary modules
- payload selection
- pivoting
- post exploitation

---

# Tags

#metasploit #msfconsole #exploitation #oscp #pentesting #framework