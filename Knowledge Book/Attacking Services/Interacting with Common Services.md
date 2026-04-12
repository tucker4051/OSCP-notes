# Attacking Common Services - Interacting with Common Services

## Overview

To attack a service effectively, we need to understand:

- **its purpose**
- **how to interact with it**
- **what tools are available**
- **what actions are possible once access is gained**

This note covers how to interact with several **common internal services**, focusing on:

- **file share services**
- **email services**
- **databases**

The goal here is not exploitation yet, but building the operational knowledge needed to **enumerate, access, search, and work with these services efficiently**.

---

## Why This Matters

Many vulnerabilities are discovered by people who understand a protocol or service deeply.

If you know how a service normally works, you are much better at spotting:

- weak permissions
- misconfigurations
- sensitive file exposure
- anonymous access
- credential storage
- abuse paths after authentication

Think of this as learning how to “speak the language” of a service before trying to break it.

---

# File Share Services

## Overview

A **file sharing service** provides, mediates, or monitors file transfer between systems.

Historically, organizations relied heavily on **internal file sharing services**, such as:

- **SMB**
- **NFS**
- **FTP**
- **TFTP**
- **SFTP**

Modern environments may also use cloud-backed storage such as:

- Dropbox
- Google Drive
- OneDrive
- SharePoint
- AWS S3
- Azure Blob Storage
- Google Cloud Storage

Even when cloud services are involved, data is often **synced locally** to servers or workstations, so local enumeration techniques still matter.

This section focuses on **internal services**, especially **SMB**.

---

# Server Message Block (SMB)

## Overview

**SMB** is one of the most common file sharing protocols in Windows environments.

It is used for:

- shared folders
- departmental file repositories
- application data shares
- scripts
- backups
- printer sharing

You can interact with SMB using:

- **GUI**
- **command line**
- **specialized tools**

---

# Interacting with SMB from Windows

## GUI Access

A simple way to browse an SMB share in Windows is:

1. Press **`WIN + R`**
2. Enter the UNC path:

```text
\\192.168.220.129\Finance\
```

### What happens next

- If the share allows **anonymous access**, or your current session already has access, it opens directly.
- If not, Windows prompts for authentication.

This is the quickest manual way to test access.

---

## Windows CMD

Windows has two main shell environments:

- **Command Prompt (CMD)**
- **PowerShell**

CMD is useful for simple share interaction.

---

### Listing Share Contents with `dir`

```cmd
dir \\192.168.220.129\Finance\
```

### What this does

Lists files and folders inside the remote share.

Useful for:

- confirming access
- identifying top-level folders
- checking whether the share is browsable

---

### Mapping a Share with `net use`

```cmd
net use n: \\192.168.220.129\Finance
```

This maps the remote share to drive letter **`N:`**.

### Why this helps

Once mapped, the share behaves much like a local drive, which makes searching and scripting easier.

---

### Mapping with Credentials

```cmd
net use n: \\192.168.220.129\Finance /user:plaintext Password123
```

### What this does

Authenticates to the share using the supplied username and password.

Useful when:

- guest access is disabled
- alternate credentials are required
- testing known credentials

---

### Counting Files Recursively

```cmd
dir n: /a-d /s /b | find /c ":\"
```

### Breakdown

| Part | Meaning |
|---|---|
| `dir` | Lists contents |
| `n:` | Target mapped drive |
| `/a-d` | Show files only, not directories |
| `/s` | Recurse into subdirectories |
| `/b` | Bare format |
| `find /c ":\\"` | Count matching output lines |

### Why this matters

If a share contains tens of thousands of files, manual searching becomes slow. This helps estimate scale quickly.

---

### Searching for Interesting Filenames

Examples:

```cmd
dir n:\*cred* /s /b
dir n:\*secret* /s /b
```

### Good filename search terms

- `cred`
- `password`
- `users`
- `secret`
- `key`

### Also search common source code / config extensions

- `.cs`
- `.c`
- `.go`
- `.java`
- `.php`
- `.asp`
- `.aspx`
- `.html`

---

### Searching Inside Files with `findstr`

```cmd
findstr /s /i cred n:\*.*
```

### What this does

- `/s` = recurse
- `/i` = case-insensitive

This searches file contents for the string `cred`.

Very useful for finding:

- credentials
- API keys
- tokens
- usernames
- comments referencing secrets

---

# Interacting with SMB from PowerShell

PowerShell is more flexible than CMD and is usually better for automation.

---

## Listing Share Contents

```powershell
Get-ChildItem \\192.168.220.129\Finance\
```

Short alias:

```powershell
gci \\192.168.220.129\Finance\
```

---

## Mapping a Share with `New-PSDrive`

```powershell
New-PSDrive -Name "N" -Root "\\192.168.220.129\Finance" -PSProvider "FileSystem"
```

### Why use this

This is the PowerShell equivalent of `net use`.

---

## Mapping with Credentials

```powershell
$username = 'plaintext'
$password = 'Password123'
$secpassword = ConvertTo-SecureString $password -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential $username, $secpassword
New-PSDrive -Name "N" -Root "\\192.168.220.129\Finance" -PSProvider "FileSystem" -Credential $cred
```

### What this does

Creates a credential object, then uses it to authenticate to the SMB share.

---

## Counting Files Recursively

```powershell
(Get-ChildItem -File -Recurse | Measure-Object).Count
```

If you are inside the mapped drive:

```powershell
N:
(Get-ChildItem -File -Recurse | Measure-Object).Count
```

---

## Searching for Interesting Filenames

```powershell
Get-ChildItem -Recurse -Path N:\ -Include *cred* -File
```

### Why this is useful

PowerShell handles filtering very cleanly and is ideal for scripted searching.

---

## Searching File Contents with `Select-String`

```powershell
Get-ChildItem -Recurse -Path N:\ | Select-String "cred" -List
```

### Why this matters

`Select-String` is roughly the PowerShell equivalent of:

- `grep` in Linux
- `findstr` in CMD

It is powerful for searching for:

- passwords
- tokens
- usernames
- configuration secrets
- code comments referencing creds

---

# Interacting with SMB from Linux

Linux can also browse and mount SMB shares, whether the server is:

- a Windows system
- a Samba server
- another SMB-capable host

---

## Mounting an SMB Share

```bash
sudo mkdir /mnt/Finance
sudo mount -t cifs -o username=plaintext,password=Password123,domain=. //192.168.220.129/Finance /mnt/Finance
```

### What this does

- creates a local mount point
- mounts the remote SMB share locally
- allows you to interact with it like a local directory

---

## Mounting with a Credential File

```bash
mount -t cifs //192.168.220.129/Finance /mnt/Finance -o credentials=/path/credentialfile
```

Credential file format:

```text
username=plaintext
password=Password123
domain=.
```

### Why use a credential file

This avoids putting credentials directly in the shell history or process list.

---

## Requirement

Install CIFS support if needed:

```bash
sudo apt install cifs-utils
```

---

## Searching for Interesting Filenames

```bash
find /mnt/Finance/ -name *cred*
```

Example result:

```text
/mnt/Finance/Contracts/private/credentials.txt
```

---

## Searching Inside Files

```bash
grep -rn /mnt/Finance/ -ie cred
```

### What this does

- `-r` = recursive
- `-n` = show line number
- `-i` = case-insensitive
- `-e` = search expression

Useful for quickly finding:

- credentials
- secrets
- usernames
- passwords
- internal notes

---

## Key Idea

Once a remote file service is mounted locally, you can use normal local tools:

- `find`
- `grep`
- `strings`
- `file`
- `cat`
- `less`

This is important because the service itself may be remote, but the workflow becomes local.

---

# Other File Share Services

Other file-sharing services you may encounter include:

- **FTP**
- **TFTP**
- **NFS**
- **SFTP**

The key lesson is this:

> Once you understand how to mount or connect to the service, you can often use your normal local tools to search and analyse the exposed files.

When you encounter a new service, ask:

- How do I authenticate?
- Can I browse anonymously?
- Can I mount it?
- What tools support it?
- Can I search files efficiently after connecting?

---

# Email Services

## Overview

Email typically relies on **one protocol for sending** and **another for retrieving** messages.

### Common protocols

| Protocol | Purpose |
|---|---|
| SMTP | Sending email |
| POP3 | Retrieving email (download-style) |
| IMAP | Retrieving and managing email on the server |

---

## Why email matters during assessments

Mailbox access can expose:

- credentials
- password reset links
- MFA setup messages
- internal documents
- sensitive communications
- service registration messages
- developer notifications

If credentials are valid for email, they may also work elsewhere.

---

## Using a GUI Mail Client

A common Linux mail client is **Evolution**.

Install it:

```bash
sudo apt-get install evolution
```

If sandboxing causes startup issues:

```bash
export WEBKIT_FORCE_SANDBOX=0 && evolution
```

---

## Connecting to Mail Services

When configuring a client, you typically need:

- mail server hostname or IP
- username
- password
- protocol type
- encryption method

### Common combinations

- **SMTP** or **SMTPS**
- **IMAP** or **IMAPS**
- **POP3** or **POP3S**
- **STARTTLS** where supported

### Why this matters

If you have credentials and the right ports are open, a mail client can give you immediate access to mailbox contents.

---

# Databases

## Overview

Databases are common in enterprise environments and often store:

- application data
- user accounts
- credentials
- internal business records
- audit logs
- configuration data

The two most common relational database platforms you will see are:

- **MySQL**
- **MSSQL**

---

## Common Ways to Interact with Databases

1. **Command-line utilities**
2. **Programming languages**
3. **GUI applications**

---

# MSSQL Interaction

## Linux with `sqsh`

```bash
sqsh -S 10.129.20.13 -U username -P Password123
```

### Why `sqsh` is useful

It behaves much like a shell and supports:

- variables
- aliases
- pipes
- redirection
- command history

Useful when working from Linux against MSSQL.

---

## Windows with `sqlcmd`

```cmd
sqlcmd -S 10.129.20.13 -U username -P Password123
```

### Why this matters

`sqlcmd` is the standard Microsoft command-line tool for interacting with MSSQL from Windows.

---

# MySQL Interaction

## Linux

```bash
mysql -u username -pPassword123 -h 10.129.20.13
```

## Windows

```cmd
mysql.exe -u username -pPassword123 -h 10.129.20.13
```

### Note

`-pPassword123` is written without a space in this syntax.

---

# GUI Database Tools

## Common Tools

- **MySQL Workbench**
- **SQL Server Management Studio (SSMS)**
- **DBeaver**

### DBeaver

DBeaver is especially useful because it is:

- cross-platform
- multi-database
- easy to use
- supports MySQL, MSSQL, PostgreSQL, and others

---

## Installing DBeaver

```bash
sudo dpkg -i dbeaver-<version>.deb
```

Run it with:

```bash
dbeaver &
```

### What you need to connect

- target IP / hostname
- port
- database engine
- username
- password

---

## Why database access matters

Once connected, you may be able to:

- enumerate databases
- enumerate tables
- retrieve sensitive data
- locate usernames and passwords
- identify application structure
- execute commands in some cases
- abuse service account privileges

---

# Tools for Interacting with Common Services

## SMB

- `smbclient`
- `CrackMapExec` / `NetExec`
- `SMBMap`
- `Impacket`
- `psexec.py`
- `smbexec.py`

## FTP

- `ftp`
- `lftp`
- `ncftp`
- `filezilla`
- `crossftp`

## Email

- `Thunderbird`
- `Claws`
- `Geary`
- `MailSpring`
- `mutt`
- `mailutils`
- `sendEmail`
- `swaks`
- `sendmail`

## Databases

- `mssql-cli`
- `mycli`
- `mssqlclient.py`
- `dbeaver`
- `MySQL Workbench`
- `SSMS`

---

# General Troubleshooting

When you cannot access a service, common causes include:

- **Authentication**
- **Privileges**
- **Network connectivity**
- **Firewall rules**
- **Protocol support**
- **Client/server version mismatch**
- **Encryption requirements**
- **Disabled anonymous access**

---

## Why error messages matter

Error messages often reveal useful clues.

Examples:

- invalid credentials
- access denied
- unsupported protocol version
- SMB signing issues
- TLS negotiation failures
- authentication method mismatch

These can guide the next step in troubleshooting or enumeration.

---

# Key Takeaways

- Understanding how to **interact** with a service is the foundation of attacking it.
- SMB shares can be accessed from both **Windows and Linux** using GUI and CLI methods.
- Email access can reveal highly sensitive operational information.
- Databases often contain the most valuable information in an environment.
- GUI tools are useful, but **CLI tools are faster, scriptable, and better for scale**.
- Troubleshooting access issues is part of service enumeration.

---

# Quick Commands

## SMB - Windows CMD

```cmd
dir \\192.168.220.129\Finance\
net use n: \\192.168.220.129\Finance
net use n: \\192.168.220.129\Finance /user:plaintext Password123
dir n: /a-d /s /b | find /c ":\"
dir n:\*cred* /s /b
findstr /s /i cred n:\*.*
```

## SMB - PowerShell

```powershell
Get-ChildItem \\192.168.220.129\Finance\
New-PSDrive -Name "N" -Root "\\192.168.220.129\Finance" -PSProvider "FileSystem"
(Get-ChildItem -File -Recurse | Measure-Object).Count
Get-ChildItem -Recurse -Path N:\ -Include *cred* -File
Get-ChildItem -Recurse -Path N:\ | Select-String "cred" -List
```

## SMB - Linux

```bash
sudo mkdir /mnt/Finance
sudo mount -t cifs -o username=plaintext,password=Password123,domain=. //192.168.220.129/Finance /mnt/Finance
mount -t cifs //192.168.220.129/Finance /mnt/Finance -o credentials=/path/credentialfile
find /mnt/Finance/ -name *cred*
grep -rn /mnt/Finance/ -ie cred
```

## Email

```bash
sudo apt-get install evolution
export WEBKIT_FORCE_SANDBOX=0 && evolution
```

## MSSQL

```bash
sqsh -S 10.129.20.13 -U username -P Password123
```

```cmd
sqlcmd -S 10.129.20.13 -U username -P Password123
```

## MySQL

```bash
mysql -u username -pPassword123 -h 10.129.20.13
```

```cmd
mysql.exe -u username -pPassword123 -h 10.129.20.13
```

## DBeaver

```bash
sudo dpkg -i dbeaver-<version>.deb
dbeaver &
```

---

# Search Priorities

When interacting with shares, mail, or databases, prioritize:

## File Shares

- credentials
- secrets
- configuration files
- source code
- documents
- scripts
- backups
- private folders

## Email

- password reset emails
- onboarding emails
- credentials in messages
- internal attachments
- MFA setup messages
- notifications from systems

## Databases

- user tables
- password fields
- API keys
- tokens
- config tables
- stored procedures
- service accounts

---

# High-Value Findings

- anonymous SMB access
- writable SMB shares
- sensitive files in shared folders
- valid email credentials
- mailbox access with reused passwords
- database credentials that work elsewhere
- database tables containing users and passwords
- service accounts with excessive privileges
- command execution via database features

---

# Tags

#attacking-common-services  
#common-services  
#smb  
#email  
#databases  
#windows  
#linux  
#mysql  
#mssql  
#enumeration  
#obsidian