# Password Attacks — Credential Hunting in Windows

## Overview

Credential hunting is the process of searching a compromised system for stored credentials.

Once access is obtained (for example via RDP to an admin workstation), credentials may be discovered in:

- configuration files
- scripts
- password managers
- browser storage
- system memory
- registry locations
- Group Policy files
- user documents
- network shares

Credential hunting is highly effective because users and administrators frequently store credentials insecurely for convenience.

Rather than blindly guessing passwords, we search for credentials that already exist on the system.

---

# Search-Centric Approach

Modern operating systems and applications provide powerful search functionality.

These same features can be leveraged during an engagement to locate credentials quickly.

The key idea:

Focus on how the system is used.

Example scenario:

We have RDP access to an IT administrator workstation.

An IT admin commonly works with:

- servers
- scripts
- configuration files
- remote management tools
- SSH clients
- databases
- cloud dashboards
- network devices
- backup systems

All of these may require stored credentials.

---

# Keywords to Search For

Useful keywords when searching files or registry entries:

```text
password
passphrase
secret
credential
creds
login
username
user
key
passkey
configuration
dbcredential
dbpassword
pwd
token
apikey
connectionstring
```

Also search for:

```text
*.config
*.xml
*.ini
*.txt
*.env
*.ps1
*.bat
*.vbs
*.yml
*.json
```

---

# Built-in Search Tools

## Windows Search (GUI)

When GUI access is available:

Use Windows Search bar to search for:

```text
password
creds
config
key
```

Windows Search indexes:

- documents
- text files
- config files
- scripts
- emails
- application data

Example locations worth browsing manually:

```text
Desktop
Documents
Downloads
AppData
OneDrive
Shared folders
Network drives
```

---

# Automated Credential Extraction Tools

## LaZagne

LaZagne is a credential recovery tool capable of extracting stored passwords from many common applications.

It contains multiple modules targeting different credential storage mechanisms.

Example modules:

| Module | Description |
|--------|-------------|
| browsers | Chrome, Firefox, Edge, Opera |
| chats | Skype and messaging apps |
| mails | Outlook, Thunderbird |
| memory | KeePass and LSASS memory |
| sysadmin | WinSCP, OpenVPN configs |
| windows | Credential Manager, LSA secrets |
| wifi | stored wireless passwords |

LaZagne supports credential extraction from many popular applications and can often reveal credentials in plaintext.

---

# Running LaZagne

After transferring LaZagne to the target:

```cmd
start LaZagne.exe all
```

Verbose mode:

```cmd
LaZagne.exe all -vv
```

Example output:

```text
------------------- Winscp passwords -----------------

[+] Password found !!!
URL: 10.129.202.51
Login: admin
Password: SteveisReallyCool123
Port: 22
```

---

# Browser Credential Storage

Browsers frequently store credentials for convenience.

Common targets:

- Chrome
- Edge
- Firefox
- Opera

Although credentials are encrypted, decryption tools exist:

Examples:

```text
firefox_decrypt
decrypt-chrome-passwords
LaZagne browser module
```

Browser credentials often provide access to:

- admin portals
- SaaS dashboards
- internal tools
- email accounts
- VPN portals

---

# Important File Locations

## SYSVOL share

Group Policy files may contain credentials:

```text
\\domain\SYSVOL\
```

Look for:

```text
Groups.xml
Services.xml
ScheduledTasks.xml
Scripts
Policies
```

---

## Scripts on IT Shares

Admins often embed credentials in scripts:

```text
.ps1
.bat
.vbs
.sh
.py
```

Look for:

```text
net use
runas
password=
credential=
```

---

## web.config files

Common on development machines and IIS servers:

```text
web.config
app.config
```

Often contain:

```text
database credentials
connection strings
API keys
service credentials
```

Example patterns:

```xml
connectionString=
password=
user id=
```

---

## unattended installation files

Example file:

```text
unattend.xml
```

May contain:

```text
local administrator credentials
domain join credentials
deployment credentials
```

Common location:

```text
C:\Windows\Panther\
C:\Windows\System32\Sysprep\
```

---

## Active Directory description fields

Admins sometimes store credentials in AD object descriptions.

Examples:

```text
password in user description
service account notes
temporary credentials
device credentials
```

These can sometimes be discovered through LDAP enumeration.

---

## KeePass Databases

KeePass password vaults may be found on:

```text
user desktops
network shares
Documents folders
shared IT folders
```

File extensions:

```text
.kdbx
.kdb
```

If master password can be guessed or cracked, large numbers of credentials may be recovered.

---

## Documents Containing Credentials

Look for files such as:

```text
passwords.txt
passwords.xlsx
creds.docx
accounts.txt
vpn.txt
backup.txt
notes.txt
```

Common locations:

```text
Desktop
Documents
Downloads
Sharepoint sync folders
Network shares
OneDrive folders
```

---

# Additional Credential Sources

Common credential storage locations:

```text
Windows Credential Manager
LSA Secrets
Registry
Scheduled Tasks
Service account configs
RDP connection files
VPN configuration files
Database config files
Cloud CLI configuration files
```

---

# Practical Workflow

1. identify system purpose
2. determine likely credential storage locations
3. search using relevant keywords
4. inspect configuration files
5. inspect scripts
6. inspect user documents
7. run automated credential tools
8. inspect shares and mapped drives
9. review browser credential storage
10. test discovered credentials across environment

---

# Why Credential Hunting Works

Users prioritize convenience over security.

Examples of common behaviour:

- storing passwords in text files
- embedding credentials in scripts
- saving passwords in browsers
- reusing passwords across systems
- storing credentials in documentation

Credential hunting is often one of the fastest ways to escalate privileges.

Instead of breaking the lock, we look for where the key was left lying around.

---

# Key Takeaways

- credential hunting focuses on discovery, not brute force
- understanding user behaviour improves success rate
- administrators often store credentials insecurely
- browsers and config files are high-value targets
- SYSVOL and scripts frequently contain passwords
- automated tools like LaZagne accelerate discovery
- KeePass databases may contain large credential sets
- credential reuse enables lateral movement

---

# Related Notes

- Password Attacks — Windows Credential Manager
- Password Attacks — Dumping LSASS Credentials
- Password Attacks — Active Directory NTDS.dit Extraction
- Enumeration — SMB Shares
- Enumeration — LDAP
- Privilege Escalation — Windows

---

# Tags

#password-attacks #credential-hunting #windows #lazagne #oscp #postexploitation #credential-access