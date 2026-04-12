# Password Attacks — Credential Hunting in Network Shares

## Overview

Network shares frequently contain sensitive information such as:

- plaintext credentials
- configuration files
- deployment scripts
- database connection strings
- backup files
- automation scripts
- password manager databases

Because shares are commonly used by:

- IT teams
- developers
- DevOps engineers
- administrators
- support teams

they often contain reusable credentials.

Credential hunting in shares is often faster and quieter than password brute forcing.

---

# Common Credential Indicators

Before using automated tools, understand typical patterns that indicate stored credentials.

---

## Keywords to Search For

Common strings:

```text
password
pass
passwd
pwd
secret
token
key
apikey
credential
creds
login
connectionstring
dbpassword
dbuser
```

Domain-specific searching:

```text
DOMAIN\
COMPANY\
INLANEFREIGHT\
.local
.internal
corp
svc
admin
```

Localized keywords (important in real environments):

| English | German | French |
|--------|--------|--------|
| user | benutzer | utilisateur |
| password | passwort | motdepasse |
| admin | administrator | administrateur |

---

## File Extensions Often Containing Credentials

```text
.ini
.cfg
.config
.xml
.json
.yml
.env
.ps1
.bat
.cmd
.vbs
.py
.txt
.docx
.xlsx
.csv
.kdbx
```

---

## Interesting File Names

Look for:

```text
config
settings
backup
passwords
creds
accounts
users
deploy
initial
setup
database
connection
automation
script
```

Examples:

```text
passwords.xlsx
creds.txt
config.ini
deploy.ps1
web.config
backup.zip
accounts.csv
vpn.txt
db.env
```

---

# Strategy

Avoid scanning everything blindly.

Prioritize shares likely to contain credentials:

| High Value Share | Why |
|------------------|-----|
| IT | admin scripts |
| DEV | config files |
| SYSVOL | GPO credentials |
| NETLOGON | scripts |
| Backup | archived configs |
| Finance | database exports |
| HR | spreadsheets |
| Admin home dirs | notes/scripts |

Large environments may contain thousands of files.

Targeting high-value shares first saves time.

---

# Manual Searching from Windows

Basic recursive search:

```powershell
Get-ChildItem -Recurse \\SERVER\Share -Include *.config,*.xml,*.ini,*.txt,*.ps1 |
Select-String -Pattern "password"
```

Search for domain strings:

```powershell
Select-String -Path \\SERVER\Share\* -Pattern "DOMAIN\\"
```

---

# Hunting from Windows (Automated)

## Snaffler

Snaffler automatically discovers accessible network shares and searches for credential patterns.

Basic scan:

```cmd
Snaffler.exe -s
```

Snaffler can:

- enumerate domain computers
- locate readable shares
- search file contents
- detect credential patterns
- highlight interesting files

Example discovery:

```text
\\DC01\SYSVOL
\\DC01\IT
\\DC01\Finance
\\DC01\HR
```

Example finding:

```text
unattend.xml → AdministratorPassword
```

---

## Useful Snaffler Parameters

Retrieve usernames from AD:

```cmd
-u
```

Include specific shares:

```cmd
-i
```

Exclude noisy shares:

```cmd
-n
```

Helpful for reducing false positives.

---

## PowerHuntShares

PowerShell-based share auditing tool.

Generates HTML report for easy review.

Basic usage:

```powershell
Invoke-HuntSMBShares -Threads 100 -OutputDirectory C:\Users\Public
```

PowerHuntShares can:

- enumerate domain computers
- detect SMB shares
- identify permissions
- highlight high-risk shares
- produce HTML and CSV reports

Useful for large environments.

---

# Hunting from Linux

## MANSPIDER

Search SMB shares remotely from Linux.

Recommended via Docker:

```bash
docker run --rm -v ./manspider:/root/.manspider \
blacklanternsecurity/manspider <target> \
-c 'passw' \
-u '<user>' \
-p '<password>'
```

Example search terms:

```text
passw
cred
secret
token
key
config
```

MANSPIDER:

- downloads matching files
- searches file contents
- supports multithreading
- handles large environments well

Loot stored in:

```text
/root/.manspider/loot
```

---

## NetExec Spidering

NetExec includes share crawling functionality.

Search share contents:

```bash
nxc smb <target> \
-u <user> \
-p <password> \
--spider IT \
--content \
--pattern "passw"
```

Spider all accessible shares:

```bash
nxc smb <target> -u <user> -p <password> --spider *
```

Search for keywords:

```bash
--pattern "password"
--pattern "creds"
--pattern "token"
--pattern "secret"
```

---

# High Value File Targets

Particularly important:

```text
web.config
unattend.xml
Groups.xml
Services.xml
ScheduledTasks.xml
backup.zip
id_rsa
id_ed25519
config.ini
.env
KeePass.kdbx
database.yml
settings.py
```

---

# SYSVOL Specific Targets

Common GPO credential exposure locations:

```text
\\domain\SYSVOL\domain\Policies\
```

Search for:

```text
Groups.xml
Services.xml
ScheduledTasks.xml
Drives.xml
Printers.xml
Registry.xml
```

These may contain:

```text
embedded passwords
service credentials
scheduled task credentials
```

---

# Typical Workflow

1. enumerate SMB shares
2. identify readable shares
3. prioritize IT / DEV shares
4. search manually for keywords
5. run automated tools
6. review false positives
7. extract credentials
8. attempt reuse
9. escalate privileges
10. pivot laterally

---

# Why This Works

Administrators often:

- embed credentials in scripts
- store passwords in spreadsheets
- reuse credentials across systems
- store config backups
- leave deployment files accessible

Credential reuse is extremely common in enterprise environments.

Finding one password may unlock:

- VPN access
- domain admin accounts
- database access
- cloud dashboards
- backup systems

---

# Key Takeaways

- network shares frequently contain reusable credentials
- keyword-based searching is highly effective
- prioritize high-value shares first
- automated tools improve efficiency
- manual review is still necessary
- credentials found here often enable lateral movement

---

# Related Notes

- Enumeration — SMB
- Password Attacks — Credential Hunting in Windows
- Password Attacks — Windows Credential Manager
- Active Directory — SYSVOL Enumeration
- Privilege Escalation — Windows

---

# Tags

#password-attacks #credential-hunting #smb #shares #snaffler #manspider #netexec #oscp