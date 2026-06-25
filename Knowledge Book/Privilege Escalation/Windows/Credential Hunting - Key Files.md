## Overview

Credentials can unlock many paths during an assessment.

Discovered credentials may provide:

- local administrator access
- a foothold into Active Directory
- access to additional hosts
- access to application services
- privilege escalation within the domain
- access to sensitive data
- access to administrative tooling

Credential hunting is a key part of Windows privilege escalation because passwords and secrets are often stored in places that are easy to overlook.

---

# Common Credential Locations

Useful places to check include:

- application configuration files
- IIS `web.config` files
- dictionary files
- unattended installation files
- PowerShell history files
- exported PowerShell credentials
- scripts used for automation
- backup folders
- deployment folders
- user profile directories

> [!important]
> Always re-check credential locations after gaining higher privileges. Local admin access may allow reading files from other users’ profiles.

---

# Method 1 – Application Configuration Files

## Why Config Files Matter

Applications sometimes store credentials in cleartext configuration files.

This may include credentials for:

- databases
- service accountscd 
- API keys
- local admin accounts
- domain users
- backup services
- web applications
- vCenter or cloud tooling

Common file extensions:

```text
.txt
.ini
.cfg
.config
.xml
```

---

# Search for Passwords with findstr

Use `findstr` to recursively search for password strings in common configuration and text files.

```powershell
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml
```

## Command Breakdown

| Part | Purpose |
|---|---|
| `findstr` | Search for strings in files |
| `/S` | Search recursively |
| `/I` | Case-insensitive search |
| `/M` | Print only filenames containing matches |
| `/C:"password"` | Search for the literal string `password` |
| `*.txt *.ini *.cfg *.config *.xml` | File types to search |

> [!tip]
> Start with filenames only using `/M`, then inspect interesting files manually to avoid excessive output.

---

# IIS web.config Files

IIS applications often store sensitive values in:

```text
web.config
```

For the default IIS site, the path may be:

```text
C:\inetpub\wwwroot\web.config
```

There may be multiple `web.config` files across the system, especially on servers hosting multiple sites or applications.

Search recursively for them:

```powershell
Get-ChildItem -Path C:\ -Filter web.config -Recurse -ErrorAction SilentlyContinue
```

Common secrets in `web.config`:

- SQL connection strings
- application pool credentials
- service account passwords
- API keys
- encryption keys
- LDAP bind credentials

---

# Method 2 – Dictionary Files

## Why Dictionary Files Matter

Some applications, browsers, and email clients underline words they do not recognize. Users may accidentally add passwords or sensitive strings to a custom dictionary to remove the underline.

This can expose secrets in user profile files.

---

# Chrome Custom Dictionary

Example Chrome dictionary path:

```text
C:\Users\<username>\AppData\Local\Google\Chrome\User Data\Default\Custom Dictionary.txt
```

Search for password-like values:

```powershell
gc 'C:\Users\htb-student\AppData\Local\Google\Chrome\User Data\Default\Custom Dictionary.txt' | Select-String password
```

Example output:

```powershell
Password1234!
```

## Useful Search Terms

```text
password
pass
pwd
admin
vpn
domain
sql
svc
secret
token
key
```

---

# Method 3 – Unattended Installation Files

## Why Unattend Files Matter

Unattended Windows installation files can define:

- auto-logon settings
- local administrator passwords
- domain join information
- accounts created during installation
- setup commands

Passwords in `unattend.xml` may be:

- plaintext
- base64 encoded

These files should be deleted after installation, but copies are often left behind during image development or deployment testing.

---

# Example unattend.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<unattend xmlns="urn:schemas-microsoft-com:unattend">
    <settings pass="specialize">
        <component name="Microsoft-Windows-Shell-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS" xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <AutoLogon>
                <Password>
                    <Value>local_4dmin_p@ss</Value>
                    <PlainText>true</PlainText>
                </Password>
                <Enabled>true</Enabled>
                <LogonCount>2</LogonCount>
                <Username>Administrator</Username>
            </AutoLogon>
            <ComputerName>*</ComputerName>
        </component>
    </settings>
</unattend>
```

Interesting values:

```text
<Value>local_4dmin_p@ss</Value>
<PlainText>true</PlainText>
<Username>Administrator</Username>
```

This indicates plaintext auto-logon credentials for the local `Administrator` account.

---

# Common Unattended File Locations

Check common locations:

```text
C:\unattend.xml
C:\Windows\Panther\Unattend.xml
C:\Windows\Panther\Unattended.xml
C:\Windows\System32\Sysprep\unattend.xml
C:\Windows\System32\Sysprep\Panther\unattend.xml
```

Search recursively:

```powershell
Get-ChildItem -Path C:\ -Include unattend.xml,unattended.xml,autounattend.xml -Recurse -ErrorAction SilentlyContinue
```

---

# Method 4 – PowerShell History Files

## Why PowerShell History Matters

Starting with PowerShell 5.0 on Windows 10, PowerShell stores command history through PSReadLine.

Default path:

```text
C:\Users\<username>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

Administrators frequently pass credentials on the command line for troubleshooting, automation, or remote access.

These credentials may remain in PowerShell history.

---

# Confirm PowerShell History Save Path

```powershell
(Get-PSReadLineOption).HistorySavePath
```

Example output:

```powershell
PS C:\htb> (Get-PSReadLineOption).HistorySavePath

C:\Users\htb-student\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

---

# Read Current User PowerShell History

```powershell
gc (Get-PSReadLineOption).HistorySavePath
```

Example output:

```powershell
PS C:\htb> gc (Get-PSReadLineOption).HistorySavePath

dir
cd Temp
md backups
cp c:\inetpub\wwwroot\* .\backups\
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://www.powershellgallery.com/packages/MrAToolbox/1.0.1/Content/Get-IISSite.ps1'))
. .\Get-IISsite.ps1
Get-IISsite -Server WEB02 -web "Default Web Site"
wevtutil qe Application "/q:*[Application [(EventID=3005)]]" /f:text /rd:true /u:WEB02\administrator /p:5erv3rAdmin! /r:WEB02
```

Credentials exposed:

```text
/u:WEB02\administrator
/p:5erv3rAdmin!
```

---

# Read All Accessible PowerShell History Files

Use this one-liner to read all default PowerShell history files accessible to the current user:

```powershell
foreach($user in ((ls C:\users).fullname)){cat "$user\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt" -ErrorAction SilentlyContinue}
```

Example output:

```powershell
dir
cd Temp
md backups
cp c:\inetpub\wwwroot\* .\backups\
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://www.powershellgallery.com/packages/MrAToolbox/1.0.1/Content/Get-IISSite.ps1'))
. .\Get-IISsite.ps1
Get-IISsite -Server WEB02 -web "Default Web Site"
wevtutil qe Application "/q:*[Application [(EventID=3005)]]" /f:text /rd:true /u:WEB02\administrator /p:5erv3rAdmin! /r:WEB02
```

> [!tip]
> Run this again after gaining local admin access. More user profiles may become readable.

---

# Method 5 – PowerShell Exported Credentials

## Why Exported Credentials Matter

PowerShell is often used for automation and scripting.

Administrators may store credentials using:

```powershell
Get-Credential | Export-Clixml
```

The resulting XML file contains encrypted credentials protected by DPAPI.

DPAPI protection usually means the credential can only be decrypted by:

```text
the same user
on the same computer
```

If command execution is obtained as that user, the credential can often be recovered in cleartext.

---

# Example Automation Script

Example script:

```powershell
# Connect-VC.ps1
# Get-Credential | Export-Clixml -Path 'C:\scripts\pass.xml'
$encryptedPassword = Import-Clixml -Path 'C:\scripts\pass.xml'
$decryptedPassword = $encryptedPassword.GetNetworkCredential().Password
Connect-VIServer -Server 'VC-01' -User 'bob_adm' -Password $decryptedPassword
```

## What It Does

1. A credential was previously exported to:

```text
C:\scripts\pass.xml
```

2. The script imports it:

```powershell
Import-Clixml -Path 'C:\scripts\pass.xml'
```

3. The script decrypts the password:

```powershell
$encryptedPassword.GetNetworkCredential().Password
```

4. The script uses it to connect to vCenter:

```text
VC-01
```

Target username:

```text
bob_adm
```

---

# Decrypt PowerShell Credentials

If running as the same user on the same system, import the credential file.

```powershell
$credential = Import-Clixml -Path 'C:\scripts\pass.xml'
```

Retrieve the username:

```powershell
$credential.GetNetworkCredential().username
```

Example output:

```powershell
bob
```

Retrieve the password:

```powershell
$credential.GetNetworkCredential().password
```

Example output:

```powershell
Str0ng3ncryptedP@ss!
```

---

# Practical Decision Tree

## Are there obvious config files?

Search:

```powershell
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml
```

Then inspect files manually.

---

## Is IIS installed?

Check common web roots:

```text
C:\inetpub\wwwroot\
```

Search for:

```text
web.config
```

Look for connection strings and application credentials.

---

## Are browser or application dictionaries present?

Check user profiles for dictionary files.

Example:

```text
Chrome\User Data\Default\Custom Dictionary.txt
```

Search for sensitive strings.

---

## Are unattended install files present?

Search for:

```text
unattend.xml
unattended.xml
autounattend.xml
```

Look for:

```text
AutoLogon
Password
PlainText
Username
```

---

## Is PowerShell history available?

Check current user:

```powershell
(Get-PSReadLineOption).HistorySavePath
gc (Get-PSReadLineOption).HistorySavePath
```

Check all accessible users:

```powershell
foreach($user in ((ls C:\users).fullname)){cat "$user\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt" -ErrorAction SilentlyContinue}
```

---

## Are exported PowerShell credentials present?

Search for likely credential files:

```powershell
Get-ChildItem -Path C:\ -Include *.xml,*.clixml -Recurse -ErrorAction SilentlyContinue
```

Look for scripts using:

```powershell
Import-Clixml
Export-Clixml
GetNetworkCredential()
```

If running as the same user who created the file, attempt decryption with:

```powershell
Import-Clixml
```

---

# Useful Search Terms

Search for these terms in files and scripts:

```text
password
passwd
pwd
pass
credential
creds
secret
token
apikey
api_key
connectionString
connstr
username
user
admin
domain
ldap
sql
vcenter
viserver
```

---

# Troubleshooting

## findstr returns too much output

Use `/M` to show filenames only:

```powershell
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml
```

Then open selected files.

---

## Access denied when reading user profiles

Current user may not have permission.

Re-check after privilege escalation or local admin access.

---

## PowerShell history file does not exist

Possible causes:

- PSReadLine not used
- PowerShell version older than 5.0
- user never used PowerShell interactively
- history was cleared
- non-default profile path
- running as a service/non-interactive account

---

## Export-Clixml credential does not decrypt

Common causes:

- wrong user context
- wrong machine
- DPAPI master key unavailable
- credential file was copied from another host
- profile data is missing
- credential object is not a PSCredential

---

## Password is base64 in unattend.xml

Some unattended files store values as base64.

Check whether the XML indicates plaintext:

```xml
<PlainText>true</PlainText>
```

If not plaintext, decode and validate carefully.

---

# Command Reference

## Search common files for password string

```powershell
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml
```

## Search recursively for web.config

```powershell
Get-ChildItem -Path C:\ -Filter web.config -Recurse -ErrorAction SilentlyContinue
```

## Read Chrome custom dictionary

```powershell
gc 'C:\Users\htb-student\AppData\Local\Google\Chrome\User Data\Default\Custom Dictionary.txt' | Select-String password
```

## Search for unattended install files

```powershell
Get-ChildItem -Path C:\ -Include unattend.xml,unattended.xml,autounattend.xml -Recurse -ErrorAction SilentlyContinue
```

## Get current PowerShell history path

```powershell
(Get-PSReadLineOption).HistorySavePath
```

## Read current PowerShell history

```powershell
gc (Get-PSReadLineOption).HistorySavePath
```

## Read all accessible PowerShell history files

```powershell
foreach($user in ((ls C:\users).fullname)){cat "$user\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt" -ErrorAction SilentlyContinue}
```

## Import exported PowerShell credential

```powershell
$credential = Import-Clixml -Path 'C:\scripts\pass.xml'
```

## Retrieve username from PSCredential

```powershell
$credential.GetNetworkCredential().username
```

## Retrieve password from PSCredential

```powershell
$credential.GetNetworkCredential().password
```

## Search for PowerShell credential artifacts

```powershell
Get-ChildItem -Path C:\ -Include *.xml,*.clixml -Recurse -ErrorAction SilentlyContinue
```

---

# Cleanup Checklist

Credential hunting is usually read-only, but cleanup may still be required.

Remove any temporary copies created during analysis:

```text
copied config files
exported search results
screenshots
notes containing cleartext credentials
temporary credential dumps
```

Securely handle discovered credentials:

- store only in approved assessment notes
- avoid unnecessary duplication
- do not leave credentials on target systems
- report exact file path and exposure
- recommend rotation for any exposed passwords

---

# Defensive Notes

## Common Causes

Credentials are often exposed because of:

- cleartext configuration files
- scripts created for convenience
- copied deployment answer files
- PowerShell history retention
- passwords passed on command lines
- insecure automation practices
- weak secret management

---

# Defensive Controls

Organizations should:

- avoid storing plaintext passwords in config files
- use managed service accounts where possible
- restrict access to configuration files
- remove unattended installation files after deployment
- disable or manage PowerShell history where appropriate
- prevent credentials from being passed on command lines
- use secure secret vaults
- rotate exposed credentials
- monitor for credential strings in source and config repositories
- audit scripts for `Import-Clixml`, `Export-Clixml`, and hardcoded passwords

---

# Key Takeaways

- Credentials are one of the most valuable privilege escalation findings.
- Config files frequently expose application, database, or service account passwords.
- IIS `web.config` files are high-value targets.
- Browser dictionary files can accidentally contain passwords.
- `unattend.xml` may expose local administrator or auto-logon credentials.
- PowerShell history can preserve commands containing passwords.
- Exported PowerShell credentials are DPAPI-protected but may be decrypted by the same user on the same host.
- Always re-check credential locations after gaining higher privileges.
- Treat discovered credentials as sensitive evidence and recommend rotation.

---

# Related Notes

- [[Windows Privilege Escalation]]
- [[Credential Hunting - Key Files]]
- [[Application Configuration Files]]
- [[IIS]]
- [[web.config]]
- [[PowerShell History]]
- [[DPAPI]]
- [[Unattend.xml]]
- [[PSCredential]]
- [[findstr]]
- [[Active Directory]]
- [[Post-Exploitation]]

---

# Tags

#windows
#privilege-escalation
#credential-hunting
#cleartext-passwords
#configuration-files
#web-config
#powershell-history
#dpapi
#unattend-xml
#pscredential
#findstr
#iis
#pentesting