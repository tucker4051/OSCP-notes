# File Upload Attacks – Absent Validation

## Overview

The simplest file upload vulnerability occurs when an application allows files to be uploaded **without enforcing meaningful validation**.

If the application does not properly check:

- file extension
- MIME type
- file signature
- execution risk
- allowed file categories

then we may be able to upload a malicious script directly and execute it on the server.

In the most straightforward case, this means:

- upload a web shell
- browse to the uploaded file
- gain command execution

Think of it like a building with no security checks. If the server accepts anything, we can walk straight in with a weaponised file.

---

# 1. What Absent Validation Means

A file upload feature is vulnerable when it allows files to be uploaded **without filtering what is being accepted**.

This may allow:

- arbitrary file upload
- web shell upload
- reverse shell upload
- server-side code execution
- full compromise of the host

If the uploaded file lands inside a web-accessible directory and the server executes that file type, the upload becomes an immediate RCE vector.

## Typical signs of weak or absent validation

- upload form does not mention allowed file types
- file picker shows `All Files`
- script extensions like `.php` can be selected
- unusual file types upload successfully
- uploaded files can be accessed directly afterwards

Front-end weakness alone does **not confirm vulnerability**, but it is a strong indicator that the back-end may also be poorly enforced.

---

# 2. Why This Matters

If the server accepts arbitrary executable files, the upload function becomes more than just storage.

It can become:

- a code execution mechanism
- a persistence method
- an initial access vector
- a pivot point into the underlying host

A successful arbitrary upload often leads directly to remote code execution if:

1. the file is stored in a web-accessible directory
2. the server executes that file type
3. we can reach it via URL

---

# 3. Initial Clues in the Upload Form

Before uploading anything malicious, inspect the upload feature itself.

## Things to look for

- no mention of allowed extensions
- no client-side file restrictions
- upload widget accepts any file type
- browser file selector displays `All Files`
- no error shown when selecting executable files

These signs suggest the application may lack:

- front-end validation
- back-end validation
- secure file handling controls

## Important note

Client-side acceptance alone does not prove vulnerability.

The real question is:

**Does the back-end accept and store dangerous files?**

---

# 4. Arbitrary File Upload

If the application accepts any file type, we may be able to upload a malicious server-side script directly.

Common payload types:

- web shells
- reverse shells
- code execution test files
- staged payloads

Example harmless execution test:

```php
<?php echo "Hello HTB"; ?>
```

This confirms whether the uploaded file is executed by the server.

---

# 5. Identify the Web Framework First

Before uploading a malicious script, identify what language or framework the application uses.

A web shell must typically be written in the same language executed by the server.

## Common examples

- PHP → `.php`
- ASP → `.asp`
- ASP.NET → `.aspx`
- Java → `.jsp`

A PHP payload will not execute on an ASP.NET server, and vice versa.

---

# 6. Manual Framework Identification

One quick manual technique is checking common index extensions.

If the site loads at:

```
http://<SERVER_IP>:<PORT>/
```

try:

```
http://<SERVER_IP>:<PORT>/index.php
http://<SERVER_IP>:<PORT>/index.asp
http://<SERVER_IP>:<PORT>/index.aspx
http://<SERVER_IP>:<PORT>/index.jsp
```

If one returns the same page, this often reveals the stack.

## Example

If `/index.php` loads correctly, the application is likely PHP-based.

## Why this works

Many applications hide the index file from the URL but still rely on it internally.

---

# 7. Alternative Stack Identification Methods

Extension probing is useful but not always reliable.

Some frameworks use:

- routing systems
- extensionless URLs
- controllers
- rewrite rules

Other identification techniques include:

- Wappalyzer
- Burp Suite scanner
- OWASP ZAP
- HTTP response headers
- error message analysis

## These tools can reveal

- language
- framework
- web server
- OS
- libraries
- version information

Knowing the stack ensures the uploaded payload is compatible.

---

# 8. Testing for Code Execution

After identifying the likely language, upload a harmless script in that language.

Example PHP test file:

```php
<?php echo "Hello HTB"; ?>
```

Save as:

```
test.php
```

Upload it through the application.

## Expected behaviours

### Successful upload message

```
File successfully uploaded
```

Suggests minimal or absent validation.

### Access the uploaded file

If execution occurs, the output will appear:

```
Hello HTB
```

If code is shown instead:

```php
<?php echo "Hello HTB"; ?>
```

then the file is being served as text rather than executed.

---

# 9. Upload Success vs Execution Success

Uploading a file does not automatically mean exploitation is possible.

Confirm:

- the file uploads successfully
- the file is accessible
- the file executes as server-side code
- the file is not renamed or sanitised
- the file is stored in a web-accessible location

## Example workflow

### Step 1 – Upload test script

```php
<?php echo "Hello HTB"; ?>
```

### Step 2 – confirm upload success

```
File successfully uploaded
```

### Step 3 – browse to uploaded file

### Step 4 – confirm execution output

```
Hello HTB
```

Execution confirms code runs on the server.

---

# 10. What This Confirms

If a `.php` file uploads and executes successfully, the application is vulnerable to:

- arbitrary file upload
- remote code execution
- potential full compromise of the server

At this point, the upload feature becomes an RCE entry point.

---

# 11. Conditions Required for Exploitation

Absent validation typically leads to RCE when:

- dangerous extensions are allowed
- files are stored server-side
- files are web accessible
- the server executes the file type
- files are not sanitised or relocated

If these conditions are met, exploitation is usually straightforward.

---

# 12. Potential Impact

## Direct outcomes

- web shell access
- command execution
- reverse shell callback

## Secondary impact

- credential extraction
- configuration disclosure
- database access
- pivoting internally
- persistence mechanisms

Even when full shell access is not immediate, arbitrary upload often leads to sensitive information disclosure.

---

# 13. Practical Testing Workflow

## Step 1 – inspect upload feature

Look for signs of weak validation.

## Step 2 – identify application language

Determine likely server technology.

## Step 3 – create harmless test payload

Start simple before using shells.

## Step 4 – upload the file

Observe system behaviour.

## Step 5 – access uploaded file

Use provided link or locate path manually.

## Step 6 – confirm execution

Verify whether server-side code runs.

## Step 7 – escalate carefully

Only introduce weaponised payloads once behaviour is confirmed.

---

# 14. Example Payloads

## PHP execution test

```php
<?php echo "Hello HTB"; ?>
```

## Basic PHP web shell

```php
<?php system($_GET['cmd']); ?>
```

## Example command execution

```
http://<SERVER_IP>:<PORT>/uploads/shell.php?cmd=id
```

---

# 15. Important Notes

## Upload success alone is not enough

Execution must be confirmed.

## Front-end controls are not security controls

Back-end validation determines real security posture.

## Correct language matters

Payload must match server technology.

## Routing may affect accessibility

Uploaded files are not always directly reachable.

## Start small

Test behaviour before attempting full exploitation.

---

# 16. Quick Checks

## Test index extensions

```
/index.php
/index.asp
/index.aspx
/index.jsp
```

## PHP test payload

```php
<?php echo "Hello HTB"; ?>
```

## PHP web shell

```php
<?php system($_GET['cmd']); ?>
```

## Example execution

```
http://<SERVER_IP>:<PORT>/uploads/shell.php?cmd=whoami
```

---

# 17. Key Takeaways

- absent validation allows arbitrary file upload
- executable uploads can lead directly to RCE
- identifying server technology helps choose correct payload
- execution confirmation is more important than upload success
- uploaded scripts must be reachable and executable to exploit
- file upload flaws frequently become full compromise paths

---

# Tags

#file-upload
#arbitrary-file-upload
#web-shell
#rce
#php
#validation
#web-attacks
#pentesting
#ctf
#obsidian