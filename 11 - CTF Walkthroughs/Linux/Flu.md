
## Overview

**Platform:** OffSec Proving Grounds **Operating System:** Linux (Ubuntu) **Web Application:** Atlassian Confluence 7.13.6 **Initial Access:** Unauthenticated OGNL injection RCE (CVE-2022-26134) **Initial Access Type:** Web application exploitation **Privilege Escalation:** Writable root cron script (`/opt/log-backup.sh`) **Final Access:** Root

### Key Techniques

- Nmap service + version fingerprinting
- Service fingerprint reading (identifying Confluence from raw HTTP headers, not just a banner)
- CVE lookup from version number
- Unauthenticated OGNL injection RCE
- Exploit-DB PoC usage as an alternative to a Metasploit module
- pspy process monitoring
- Writable cron script abuse
- Sudoers file injection (as an alternative to SUID bash)

---

# Attack Path

```text
Nmap: 22, 8090
        ↓
Fingerprint Identifies Confluence (via headers, not a clean banner)
        ↓
Version 7.13.6 Confirmed on Web UI
        ↓
CVE-2022-26134 (Unauthenticated OGNL Injection)
        ↓
Exploit-DB PoC (50952) Instead of Metasploit Module
        ↓
Command Execution as confluence
        ↓
pspy Reveals Root Cron: /opt/log-backup.sh
        ↓
Script Writable by confluence
        ↓
Overwrite Script to Add Sudoers Entry
        ↓
Cron Executes as root
        ↓
sudo su → Root Shell
```

---

# 1. Nmap Enumeration

```bash
sudo nmap -sVSC -T5 <target>
```

```text
22/tcp    ssh    OpenSSH 9.0p1 Ubuntu
8090/tcp  ?      unrecognized by Nmap's service DB
```

Nmap couldn't cleanly identify the service on `8090`, but the raw fingerprint data returned in the scan included distinctive strings:

```text
X-Confluence-Request-Time: ...
Location: http://localhost:8090/login.action?os_destination=...
```

> [!tip] Recognition Pattern Don't rely solely on Nmap's service-name guess (`opsmessaging?` in this case, clearly wrong). When a service comes back unrecognized, read the raw fingerprint/banner text Nmap still captures — headers, redirect paths, and cookie names are often product-specific even when Nmap's own database doesn't have a match. `X-Confluence-Request-Time` and the `.action` URL suffix are both distinctly Atlassian/Confluence.

Browsing to `http://<target>:8090/` confirmed **Atlassian Confluence, version 7.13.6**, directly in the web UI.

---

# 2. Vulnerability Identification

Confluence 7.13.6 matches **CVE-2022-26134** — an unauthenticated OGNL (Object-Graph Navigation Language) injection vulnerability. OGNL is the expression language Confluence uses internally for templating; the bug allows attacker-controlled input to be evaluated as an OGNL expression, which in turn can invoke arbitrary Java (and by extension, OS command execution) without any authentication.

- [Official Atlassian Advisory](https://confluence.atlassian.com/) (search CVE-2022-26134)

> [!tip] Version → CVE Recognition Confluence has a well-known history of high-severity unauthenticated RCEs (this one, plus others in the same OGNL-injection family). Any time Confluence shows up with a specific version number, checking known CVEs for that exact build should be an immediate, near-automatic step.

---

# 3. Getting Code Execution

## What I did differently

Rather than the Metasploit module `multi/http/atlassian_confluence_namespace_ognl_injection`, used the **Exploit-DB PoC**: [https://www.exploit-db.com/exploits/50952](https://www.exploit-db.com/exploits/50952) — a standalone script that sends the OGNL injection directly as an HTTP request rather than going through Metasploit's exploit/handler framework.

```bash
python3 50952.py <target_url> "<command>"
```

> [!note] Metasploit vs. Standalone PoC Both routes exploit the same underlying vulnerability, but a standalone script gives more visibility into the exact request being sent (useful for understanding the bug, and for environments where Metasploit isn't available or its module happens to be flaky). Metasploit is faster to get a full session/meterpreter going; a raw PoC is better for understanding the mechanics or for one-off command execution without needing a persistent handler.

Command execution confirmed:

```bash
id
# uid=1001(confluence) gid=1001(confluence) groups=1001(confluence)
```

```bash
cat /home/confluence/local.txt
```

---

# 4. Privilege Escalation Enumeration

Uploaded and ran **pspy** to observe scheduled/background processes as an unprivileged user.

```text
2023/12/13 09:52:01 CMD: UID=0  PID=2434  | /bin/sh /opt/log-backup.sh
```

A root-owned cron job executing `/opt/log-backup.sh` on a schedule.

```bash
ls -alh /opt/log-backup.sh
# -rwxr-xr-x 1 confluence confluence 20 Dec 13 09:00 /opt/log-backup.sh
```

The script is **owned by `confluence`** — the exact user our shell is running as — despite being executed by root via cron. Full write access confirmed.

> [!tip] Recognition Pattern A script executed by root via cron, but _owned by the low-privileged user you already are_, is about as clean a privesc primitive as you'll find. No permission bypass or race condition needed — you already own the file outright.

---

# 5. Privilege Escalation — Writable Cron Script

## What I did differently

The reference walkthrough overwrites the script with `chmod u+s /bin/bash` to set the SUID bit. Instead, used the same writable-script primitive to **add a permanent sudoers entry**:

```bash
echo "confluence ALL=(root) NOPASSWD: ALL" >> /opt/log-backup.sh
```

After the cron job fires and executes the script as root, this line gets appended to `/etc/sudoers`, granting the `confluence` user unrestricted, passwordless `sudo` access.

```bash
sudo su
```

```bash
id
# uid=0(root) gid=0(root) groups=0(root)
```

```bash
cat /root/proof.txt
```

> [!tip] SUID Bash vs. Sudoers Injection Both are valid endpoints for "I have one shot at arbitrary code as root" — the choice mostly comes down to persistence and flexibility:
> 
> - **SUID bash** (`chmod u+s /bin/bash`) — fast, simple, but leaves a fairly conspicuous world-readable SUID binary sitting on the filesystem, and only grants a root _shell_, not root privileges for arbitrary future commands run normally.
> - **Sudoers injection** (`<user> ALL=(root) NOPASSWD: ALL`) — grants full, clean `sudo` access as your existing user going forward, which is arguably stealthier (no unusual SUID bit to trip a file-integrity check) and more flexible (can run any command with `sudo`, not just get dropped into a shell).
> 
> Worth having both in the toolkit and picking based on what the writable-file primitive allows and what the engagement calls for.

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|Nmap can't identify a service|Read the raw fingerprint text for product-specific headers/strings anyway|
|Confluence identified with a specific version|Check known CVEs immediately — history of severe unauthenticated RCEs|
|Public exploit exists as both a Metasploit module and a standalone PoC|Standalone PoC gives more visibility into the actual request/mechanism|
|pspy shows a root cron job|Check the owner and permissions of the executed script, not just its existence|
|Cron script owned by your current low-priv user|Full write access = full privesc primitive, no further trick needed|
|Need a one-shot "run this as root" opportunity|Choose SUID bash for speed/simplicity, or a sudoers entry for persistence/flexibility|

---

# Key Lessons

> [!tip] Don't Trust Nmap's Service Guess Alone When a port comes back as unrecognized, the raw fingerprint data Nmap captured is often still identifiable by eye — headers, cookie names, and redirect paths frequently give away the exact product even when Nmap's signature database doesn't have a match.

> [!tip] Confluence + a Version Number = Immediate CVE Check Confluence has a strong track record of unauthenticated RCEs via OGNL injection. Treat a version-fingerprinted Confluence instance as high-priority for CVE lookup before anything else.

> [!tip] A Standalone PoC Is a Valid Alternative to Metasploit Don't treat "the walkthrough used Metasploit" as the only path — a raw Exploit-DB script targeting the same CVE is often just as effective, and teaches you more about what the exploit is actually doing on the wire.

> [!tip] Cron Script Ownership Matters More Than Its Existence Finding a root cron job via pspy is only step one — the real question is always "can I write to what it executes?" A script owned by your current user, run by root, is a complete privesc primitive on its own.

> [!tip] Keep Multiple "Cash In Root" Techniques Ready SUID bash and sudoers injection both convert "one arbitrary command as root" into lasting privilege — pick based on whether you want a quick shell or persistent, flexible sudo access.