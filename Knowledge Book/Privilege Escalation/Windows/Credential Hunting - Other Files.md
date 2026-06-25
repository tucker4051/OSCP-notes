## Overview

Credentials and sensitive operational data are often stored in local files or network share drives.

These files may provide:

- local administrator credentials
- domain user credentials
- service account credentials
- SSH private keys
- database credentials
- VPN credentials
- backup data
- virtual machine disks
- configuration data for privilege escalation

In Active Directory environments, network shares are especially valuable because users often store sensitive files in shared folders without realizing other domain users can read them.

---

# Why File and Share Hunting Matters

Common real-world findings include:

- `passwords.txt`
- Excel files containing credentials
- Word documents with onboarding passwords
- OneNote notebooks with admin notes
- SSH private keys
- KeePass databases
- virtual hard drives
- configuration backups
- exported registry hives
- VPN profiles
- remote desktop files
- VNC configuration files

Interesting file extensions include:

```text
.kdbx
.vmdk
.vhd
.vhdx
.ppk
.rdp
.config
.vnc
.cred
.xml
.ini
.txt
```

> [!important]
> A single password found on a local drive or network share can lead to initial access, lateral movement, or privilege escalation.

---

# Network Share Context

Many organizations provide each employee with a folder on a file share.

Example:

```text
\\FILE01\users\bjones
```

A common misconfiguration is granting broad read access such as:

```text
Domain Users: Read
```

This can expose personal files and sensitive notes across the entire domain.

---

# Tooling – Snaffler

In Active Directory environments, `Snaffler` can crawl network shares and identify interesting files.

Useful targets include:

- password files
- configuration files
- KeePass databases
- SSH keys
- virtual disks
- scripts
- backups
- documents with sensitive names or contents

Use Snaffler when you need automated share discovery and triage, but still understand manual searching so you can validate findings and adapt searches.

---

# Manual File Content Searching

## Search File Contents for a String – Example 1

Change to a target directory and search common file types for the string `password`.

```cmd
cd c:\Users\htb-student\Documents & findstr /SI /M "password" *.xml *.ini *.txt
```

Example output:

```cmd
stuff.txt
```

## Command Breakdown

| Part | Purpose |
|---|---|
| `cd c:\Users\htb-student\Documents` | Move into target directory |
| `&` | Run the next command after changing directory |
| `findstr` | Search file contents |
| `/S` | Search subdirectories |
| `/I` | Case-insensitive search |
| `/M` | Print matching filenames only |
| `"password"` | Search string |
| `*.xml *.ini *.txt` | File types to search |

---

# Search File Contents for a String – Example 2

Search and show matching lines.

```cmd
findstr /si password *.xml *.ini *.txt *.config
```

Example output:

```cmd
stuff.txt:password: l#-x9r11_2_GL!
```

This returns the filename and matching content.

---

# Search File Contents for a String – Example 3

Search all files recursively and include line numbers.

```cmd
findstr /spin "password" *.*
```

Example output:

```cmd
stuff.txt:1:password: l#-x9r11_2_GL!
```

## Flag Breakdown

| Flag | Purpose |
|---|---|
| `/S` | Search current directory and subdirectories |
| `/P` | Skip files with non-printable characters |
| `/I` | Case-insensitive search |
| `/N` | Print line number |

---

# Search File Contents with PowerShell

Use `Select-String` to search file contents.

```powershell
Select-String -Path C:\Users\htb-student\Documents\*.txt -Pattern password
```

Example output:

```powershell
stuff.txt:1:password: l#-x9r11_2_GL!
```

---

# Searching for Interesting Filenames and Extensions

## Search for File Extensions – Example 1

```cmd
dir /S /B *pass*.txt == *pass*.xml == *pass*.ini == *cred* == *vnc* == *.config*
```

Example output:

```cmd
c:\inetpub\wwwroot\web.config
```

## Search for File Extensions – Example 2

Use `where` to search recursively for `.config` files.

```cmd
where /R C:\ *.config
```

Example output:

```cmd
c:\inetpub\wwwroot\web.config
```

---

# Search for File Extensions with PowerShell

```powershell
Get-ChildItem C:\ -Recurse -Include *.rdp, *.config, *.vnc, *.cred -ErrorAction Ignore
```

Example output:

```powershell
Directory: C:\inetpub\wwwroot

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         5/25/2021   9:59 AM            329 web.config

<SNIP>
```

---

# Useful File Extension Targets

Search for files matching:

```text
*.txt
*.xml
*.ini
*.config
*.cfg
*.conf
*.json
*.yml
*.yaml
*.ps1
*.bat
*.cmd
*.vbs
*.kdbx
*.ppk
*.pem
*.key
*.rdp
*.vnc
*.cred
*.vhd
*.vhdx
*.vmdk
*.ovpn
*.sql
*.bak
*.zip
*.7z
*.rar
*.doc
*.docx
*.xls
*.xlsx
*.one
```

Useful filename patterns:

```text
*pass*
*password*
*cred*
*credential*
*secret*
*admin*
*vpn*
*backup*
*config*
*login*
*account*
*key*
*ssh*
```

---

# Sticky Notes Passwords

## Why Sticky Notes Matter

Users often store passwords and administrative notes in the Windows Sticky Notes app.

Sticky Notes data is stored in a SQLite database, not a simple text file.

Database path:

```text
C:\Users\<user>\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite
```

Related files:

```text
plum.sqlite
plum.sqlite-shm
plum.sqlite-wal
```

These files are worth collecting and reviewing.

---

# Locate Sticky Notes Database Files

Navigate to the Sticky Notes `LocalState` directory.

Example listing:

```powershell
ls
```

Example output:

```powershell
Directory: C:\Users\htb-student\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         5/25/2021  11:59 AM          20480 15cbbc93e90a4d56bf8d9a29305b8981.storage.session
-a----         5/25/2021  11:59 AM            982 Ecs.dat
-a----         5/25/2021  11:59 AM           4096 plum.sqlite
-a----         5/25/2021  11:59 AM          32768 plum.sqlite-shm
-a----         5/25/2021  12:00 PM         197792 plum.sqlite-wal
```

---

# Review Sticky Notes with DB Browser for SQLite

Copy these files to the analysis machine:

```text
plum.sqlite
plum.sqlite-shm
plum.sqlite-wal
```

Open `plum.sqlite` in DB Browser for SQLite.

Run:

```sql
SELECT Text FROM Note;
```

Look for:

- passwords
- usernames
- service names
- hostnames
- admin notes
- application names
- meeting notes that reveal tooling

---

# Review Sticky Notes with PowerShell

## Step 1 – Allow Module Import for Current Process

```powershell
Set-ExecutionPolicy Bypass -Scope Process
```

Example prompt:

```text
Execution Policy Change

The execution policy helps protect you from scripts that you do not trust. Changing the execution policy might expose
you to the security risks described in the about_Execution_Policies help topic.

Do you want to change the execution policy?
[Y] Yes  [A] Yes to All  [N] No  [L] No to All  [S] Suspend  [?] Help (default is "N"): A
```

## Step 2 – Import PSSQLite

```powershell
cd .\PSSQLite\
```

```powershell
Import-Module .\PSSQLite.psd1
```

## Step 3 – Set Database Path

```powershell
$db = 'C:\Users\htb-student\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite'
```

## Step 4 – Query Notes

```powershell
Invoke-SqliteQuery -Database $db -Query "SELECT Text FROM Note" | ft -wrap
```

Example output:

```powershell
Text
----
\id=de368df0-6939-4579-8d38-0fda521c9bc4 vCenter
\id=e4adae4c-a40b-48b4-93a5-900247852f96
\id=1a44a631-6fff-4961-a4df-27898e9e1e65 root:Vc3nt3R_adm1n!
\id=c450fc5f-dc51-4412-b4ac-321fd41c522a Thycotic demo tomorrow at 10am
```

Interesting credential:

```text
root:Vc3nt3R_adm1n!
```

Related target context:

```text
vCenter
```

---

# Review Sticky Notes with strings

Copy the SQLite files to the attack host and run `strings`.

```bash
strings plum.sqlite-wal
```

Example output excerpt:

```text
CREATE TABLE "Note" (
"Text" varchar ,
"WindowPosition" varchar ,
"IsOpen" integer ,
"IsAlwaysOnTop" integer ,
"CreationNoteIdAnchor" varchar ,
"Theme" varchar ,
"IsFutureNote" integer ,
"RemoteId" varchar ,
"ChangeKey" varchar ,
"LastServerVersion" varchar ,
"RemoteSchemaVersion" integer ,
"IsRemoteDataInvalid" integer ,
"PendingInsightsScan" integer ,
"Type" varchar ,
"Id" varchar primary key not null ,
"ParentId" varchar ,
"CreatedAt" bigint ,
"DeletedAt" bigint ,
"UpdatedAt" bigint )'
indexsqlite_autoindex_Note_1Note

<SNIP>

\id=011f29a4-e37f-451d-967e-c42b818473c2 vCenter
\id=34910533-ddcf-4ac4-b8ed-3d1f10be9e61 alright*
\id=ffaea2ff-b4fc-4a14-a431-998dc833208c root:Vc3nt3R_adm1n!ManagedPosition=Yellow93b49900-6530-42e0-b35c-2663989ae4b3af907b1b-1eef-4d29-b238-3ea74f7ffe5c

<SNIP>
```

> [!tip]
> `strings` is quick but less precise than querying the SQLite database directly.

---

# Other Files of Interest

Review these files and paths where accessible:

```text
%SYSTEMDRIVE%\pagefile.sys
%WINDIR%\debug\NetSetup.log
%WINDIR%\repair\sam
%WINDIR%\repair\system
%WINDIR%\repair\software
%WINDIR%\repair\security
%WINDIR%\iis6.log
%WINDIR%\system32\config\AppEvent.Evt
%WINDIR%\system32\config\SecEvent.Evt
%WINDIR%\system32\config\default.sav
%WINDIR%\system32\config\security.sav
%WINDIR%\system32\config\software.sav
%WINDIR%\system32\config\system.sav
%WINDIR%\system32\CCM\logs\*.log
%USERPROFILE%\ntuser.dat
%USERPROFILE%\LocalS~1\Tempor~1\Content.IE5\index.dat
%WINDIR%\System32\drivers\etc\hosts
C:\ProgramData\Configs\*
C:\Program Files\Windows PowerShell\*
```

---

# Why These Files Matter

| File / Path | Why It May Be Interesting |
|---|---|
| `pagefile.sys` | May contain memory remnants, including secrets |
| `NetSetup.log` | Domain join and setup information |
| `%WINDIR%\repair\sam` | Old SAM hive backup |
| `%WINDIR%\repair\system` | SYSTEM hive needed to process SAM |
| `%WINDIR%\repair\security` | Security hive backup |
| `iis6.log` | Legacy IIS activity or configuration clues |
| `AppEvent.Evt` | Legacy application event log |
| `SecEvent.Evt` | Legacy security event log |
| `*.sav` registry hives | Saved registry hive copies |
| `CCM\logs\*.log` | SCCM client logs, deployment details |
| `ntuser.dat` | User registry hive |
| `Content.IE5\index.dat` | Legacy browser cache/index data |
| `hosts` | Hostname mappings and environment clues |
| `C:\ProgramData\Configs\*` | Vendor or custom config files |
| `Windows PowerShell\*` | Scripts, modules, or automation secrets |

---

# Practical Decision Tree

## Are network shares available?

Enumerate shares and inspect accessible directories.

Look for:

```text
users
home
profiles
backups
IT
admin
scripts
configs
deploy
software
```

Use Snaffler or manual search depending on tooling availability.

---

## Are there obvious credential filenames?

Search for:

```text
*pass*
*cred*
*secret*
*admin*
*vpn*
*key*
```

---

## Are there interesting file extensions?

Search for:

```text
*.kdbx
*.ppk
*.vhdx
*.vmdk
*.rdp
*.config
*.vnc
*.cred
```

---

## Are Sticky Notes present?

Check:

```text
C:\Users\<user>\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\
```

If present, review:

```text
plum.sqlite
plum.sqlite-shm
plum.sqlite-wal
```

---

## Is local admin access obtained later?

Re-run searches after escalation.

Local admin may allow access to:

- other user profiles
- protected config directories
- backup files
- registry hive copies
- service account working directories

---

# Troubleshooting

## Search commands return access denied

Use `-ErrorAction SilentlyContinue` or `-ErrorAction Ignore` in PowerShell.

Example:

```powershell
Get-ChildItem C:\ -Recurse -Include *.config -ErrorAction Ignore
```

Re-check after privilege escalation.

---

## Search takes too long

Limit scope first:

```text
C:\Users
C:\ProgramData
C:\inetpub
C:\Scripts
network shares
```

Then expand to full disk if needed.

---

## Sticky Notes database appears empty

Check related files:

```text
plum.sqlite-shm
plum.sqlite-wal
```

Recent notes may be stored in WAL data.

Try:

```bash
strings plum.sqlite-wal
```

or copy all three files and open with DB Browser for SQLite.

---

## SQLite database is locked

Copy all related files first:

```text
plum.sqlite
plum.sqlite-shm
plum.sqlite-wal
```

Then analyze the copies offline.

---

## Snaffler finds too much data

Triage high-value extensions first:

```text
.kdbx
.ppk
.config
.vhdx
.vmdk
.rdp
.vnc
.cred
```

Then review filename hits containing:

```text
pass
cred
admin
secret
key
```

---

# Command Reference

## Search file contents and show filenames only

```cmd
cd c:\Users\htb-student\Documents & findstr /SI /M "password" *.xml *.ini *.txt
```

## Search file contents and show matching lines

```cmd
findstr /si password *.xml *.ini *.txt *.config
```

## Search all files recursively with line numbers

```cmd
findstr /spin "password" *.*
```

## Search file contents with PowerShell

```powershell
Select-String -Path C:\Users\htb-student\Documents\*.txt -Pattern password
```

## Search for interesting filenames

```cmd
dir /S /B *pass*.txt == *pass*.xml == *pass*.ini == *cred* == *vnc* == *.config*
```

## Search for config files

```cmd
where /R C:\ *.config
```

## Search for interesting extensions with PowerShell

```powershell
Get-ChildItem C:\ -Recurse -Include *.rdp, *.config, *.vnc, *.cred -ErrorAction Ignore
```

## Sticky Notes database path

```text
C:\Users\<user>\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite
```

## Import PSSQLite

```powershell
Import-Module .\PSSQLite.psd1
```

## Query Sticky Notes

```powershell
Invoke-SqliteQuery -Database $db -Query "SELECT Text FROM Note" | ft -wrap
```

## strings on Sticky Notes WAL

```bash
strings plum.sqlite-wal
```

---

# Cleanup Checklist

Credential hunting is usually read-only, but cleanup may still be necessary.

Remove temporary copies of:

```text
plum.sqlite
plum.sqlite-shm
plum.sqlite-wal
search result exports
copied documents
copied database files
temporary scripts
```

Securely handle any discovered credentials:

- avoid leaving credentials on target systems
- store findings only in approved notes
- report exact file paths
- include evidence with minimal exposure
- recommend credential rotation
- recommend permission remediation

---

# Defensive Notes

## Common Causes

Credential exposure occurs when users:

- store passwords in documents
- keep credentials in personal share folders
- save secrets in Sticky Notes
- store private keys on shared drives
- keep old VM disks or backups on accessible shares
- place config files in broadly readable folders
- assume mapped home folders are private

---

# Defensive Controls

Organizations should:

- restrict permissions on home folders and shares
- remove broad `Domain Users` read access where inappropriate
- scan shares for secrets regularly
- deploy data loss prevention controls
- block storage of credentials in user documents
- use password managers or enterprise secret vaults
- educate users about Sticky Notes and file share exposure
- monitor for sensitive file extensions on shares
- protect backup and virtual disk files
- review SCCM and deployment logs for exposed secrets
- rotate credentials found in files

---

# Key Takeaways

- Local file systems and network shares often contain credentials.
- User share folders may be readable by more people than intended.
- Snaffler is useful for crawling Active Directory file shares.
- Manual searching is still important when tools are unavailable or incomplete.
- `findstr`, `where`, `dir`, `Select-String`, and `Get-ChildItem` are useful built-in search options.
- Sticky Notes stores data in a SQLite database named `plum.sqlite`.
- Sticky Notes WAL files can contain recent note contents.
- Interesting files include registry backups, SCCM logs, config folders, PowerShell directories, browser cache files, and virtual disk files.
- Always re-run credential searches after privilege escalation.
- Discovered credentials should be handled carefully and rotated after reporting.

---

# Related Notes

- [[Windows Privilege Escalation]]
- [[Credential Hunting]]
- [[Network Shares]]
- [[Snaffler]]
- [[Sticky Notes]]
- [[SQLite]]
- [[PSSQLite]]
- [[findstr]]
- [[Select-String]]
- [[Windows File System]]
- [[SCCM]]
- [[KeePass]]
- [[SSH Keys]]
- [[Virtual Hard Disks]]

---

# Tags

#windows
#privilege-escalation
#credential-hunting
#file-system
#network-shares
#snaffler
#sticky-notes
#sqlite
#pssqlite
#findstr
#select-string
#sensitive-files
#pentesting