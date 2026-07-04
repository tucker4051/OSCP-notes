## Overview

Windows privileges are rights granted to an account that allow it to perform specific operations on the local system.

These operations may include:

- Managing services
- Loading drivers
- Shutting down the system
- Debugging applications
- Backing up files
- Restoring files
- Taking ownership of objects
- Impersonating users

Privileges are different from **access rights**.

| Concept | Meaning |
|---|---|
| Privileges | Rights that allow an account to perform system-level actions. |
| Access rights | Permissions used to grant or deny access to securable objects such as files, folders, registry keys, services, and processes. |

Privileges are granted through a user's **access token** when they log on to the system.

---

## Why Windows Privileges Matter

The goal of a privilege escalation assessment is often to gain administrative control over one or more systems.

If we can log in as a user with specific privileges, we may be able to abuse those privileges to:

- Escalate to local administrator
- Escalate to `NT AUTHORITY\SYSTEM`
- Access sensitive files
- Dump credentials
- Modify services
- Load malicious drivers
- Access registry hives
- Move laterally
- Escalate further in Active Directory

Some privileges are extremely powerful and should be treated as equivalent to administrative access in certain contexts.

---

# Windows Authorization Process

## Security Principals

A **security principal** is anything that can be authenticated by Windows.

Examples include:

- User accounts
- Computer accounts
- Service accounts
- Processes running under a user or computer context
- Security groups

Security principals are used to control access to resources on Windows systems.

Each security principal has a unique identifier called a **Security Identifier**, or **SID**.

---

## Security Identifiers

A **SID** uniquely identifies a security principal.

When a user, group, or computer account is created, Windows assigns it a SID. This SID remains associated with that principal for its lifetime.

Example SID format:

```text
S-1-5-21-...
```

SIDs are used heavily in Windows access control decisions.

---

## Access Tokens

When a user logs in, Windows creates an **access token** for that logon session.

The token contains information such as:

- User SID
- Group SIDs
- Assigned privileges
- Integrity level
- Logon session information
- Other access control data

When the user tries to access a resource, Windows compares the user's token against the resource's security descriptor.

---

## Securable Objects

A **securable object** is a Windows object that can have permissions applied to it.

Examples include:

- Files
- Folders
- Registry keys
- Services
- Processes
- Threads
- Printers
- Active Directory objects
- Named pipes

---

## Security Descriptors

A **security descriptor** contains security information about a securable object.

This may include:

| Component | Description |
|---|---|
| Owner SID | Identifies the object owner. |
| Group SID | Identifies the primary group. |
| DACL | Discretionary Access Control List. Defines who is allowed or denied access. |
| SACL | System Access Control List. Defines auditing rules. |

---

## Access Control Entries

A **DACL** contains one or more **Access Control Entries**, known as ACEs.

Each ACE defines access for a user or group.

An ACE may include:

- The security principal
- Whether access is allowed or denied
- The specific permissions granted or denied
- Inheritance settings

---

## High-Level Authorization Flow

When a user attempts to access a resource, Windows performs an access check.

The process is roughly:

1. A user attempts to access a securable object.
2. Windows reviews the user's access token.
3. Windows reviews the object's security descriptor.
4. The token SIDs are compared against the object's ACEs.
5. Windows checks whether the required access is allowed or denied.
6. Access is granted or denied.

In privilege escalation, we often try to abuse:

- Assigned privileges
- Weak DACLs
- Misconfigured groups
- Over-permissive services
- Poorly protected files
- Token behaviour
- Delegated rights

---

# Rights and Privileges in Windows

Windows contains many built-in groups that grant powerful rights and privileges.

Some groups can be abused to escalate privileges on:

- Standalone Windows hosts
- Domain-joined workstations
- Member servers
- Domain Controllers
- Active Directory environments

Depending on the group and environment, these rights may lead to:

- Local administrator access
- `NT AUTHORITY\SYSTEM`
- Domain Admin access
- Domain persistence
- Lateral movement

---

## Powerful Windows and Active Directory Groups

| Group | Description |
|---|---|
| Default Administrators | Domain Admins and Enterprise Admins are highly privileged “super” groups. |
| Server Operators | Members can modify services, access SMB shares, and back up files. |
| Backup Operators | Members can log on locally to Domain Controllers and should often be treated as highly privileged. They may create shadow copies of SAM or NTDS databases, read registry remotely, and access files via SMB. |
| Print Operators | Members can log on locally to Domain Controllers and may be able to abuse driver loading behaviour. |
| Hyper-V Administrators | If virtual Domain Controllers exist, virtualisation administrators should be treated as highly privileged. |
| Account Operators | Members can modify non-protected users and groups in the domain. |
| Remote Desktop Users | Members can use RDP where allowed. This group often receives additional rights in real environments. |
| Remote Management Users | Members can log on using PowerShell Remoting where allowed. |
| Group Policy Creator Owners | Members can create new GPOs, but need additional delegation to link them to domains or OUs. |
| Schema Admins | Members can modify the Active Directory schema and potentially backdoor future objects. |
| DNS Admins | Members may be able to load a DLL on a Domain Controller through the DNS service. This can be abused for privilege escalation or persistence. |

---

## Why Group Membership Matters

Group membership can grant powerful rights indirectly.

For example:

- A user may not be a local administrator directly.
- The user may belong to a group that has dangerous privileges.
- That group may be added to another local or domain group.
- The resulting effective permissions may allow privilege escalation.

Always check both:

```cmd
whoami /groups
```

and local group memberships:

```cmd
net localgroup
```

```cmd
net localgroup administrators
```

---

# User Rights Assignment

## What Is User Rights Assignment?

**User Rights Assignment** defines what users and groups are allowed to do on a Windows system.

These rights may be assigned locally or through Group Policy.

Examples include rights to:

- Log on locally
- Log on through Remote Desktop
- Access the computer from the network
- Shut down the system
- Back up files
- Restore files
- Debug programs
- Load drivers
- Impersonate users

These settings apply to the local host and can strongly affect privilege escalation paths.

---

## Key User Rights Assignments

| Constant | Setting Name | Standard Assignment | Description |
|---|---|---|---|
| `SeNetworkLogonRight` | Access this computer from the network | Administrators, Authenticated Users | Determines which users can connect to the device from the network. Required by protocols such as SMB, NetBIOS, CIFS, and COM+. |
| `SeRemoteInteractiveLogonRight` | Allow log on through Remote Desktop Services | Administrators, Remote Desktop Users | Determines which users can access the login screen through RDP. |
| `SeBackupPrivilege` | Back up files and directories | Administrators | Allows users to bypass file, directory, registry, and object permissions for backup purposes. |
| `SeSecurityPrivilege` | Manage auditing and security log | Administrators | Allows users to manage audit options and view or clear the Security log. |
| `SeTakeOwnershipPrivilege` | Take ownership of files or other objects | Administrators | Allows users to take ownership of securable objects, including files, registry keys, services, processes, and Active Directory objects. |
| `SeDebugPrivilege` | Debug programs | Administrators | Allows users to attach to or open processes they do not own, including sensitive system processes. |
| `SeImpersonatePrivilege` | Impersonate a client after authentication | Administrators, Local Service, Network Service, Service | Allows a process to impersonate another user after authentication. Commonly abused for privilege escalation. |
| `SeLoadDriverPrivilege` | Load and unload device drivers | Administrators | Allows users to load and unload drivers. Drivers run as highly privileged code. |
| `SeRestorePrivilege` | Restore files and directories | Administrators | Allows users to bypass permissions when restoring files and set valid security principals as object owners. |
| `SeTcbPrivilege` | Act as part of the operating system | Administrators, Local Service, Network Service, Service | Allows a process to assume the identity of any user. This is extremely sensitive and should be tightly restricted. |

---

# Viewing Current User Privileges

## `whoami /priv`

Use the following command to list privileges assigned to the current user:

```cmd
whoami /priv
```

This shows:

- Privilege name
- Description
- Current state

A privilege may be shown as:

| State | Meaning |
|---|---|
| Enabled | The privilege is active in the current token. |
| Disabled | The privilege is assigned but not currently enabled. |
| Not listed | The privilege is not assigned to the current token. |

---

## Disabled Does Not Mean Useless

If a privilege is listed as `Disabled`, it still means the account has that privilege assigned.

It cannot be used until enabled in the current access token, but the privilege may still be abuseable if we can enable it within a process.

Windows does not provide a simple built-in command or PowerShell cmdlet to enable privileges directly. Enabling specific privileges normally requires scripting or token manipulation.

---

# Elevated vs Non-Elevated Rights

## User Account Control

User Account Control, or **UAC**, was introduced to restrict applications from running with full administrative permissions unless required.

This means an administrator may have different privileges depending on whether the shell is:

- Non-elevated
- Elevated

An administrative user in a non-elevated shell may not see or use all privileges.

An administrative user in an elevated shell can access the full set of assigned administrative privileges.

---

# Local Administrator Rights — Elevated

## Check Current User

```powershell
whoami
```

Example output:

```powershell
PS C:\htb> whoami

winlpe-srv01\administrator
```

---

## Check Privileges from Elevated Shell

```powershell
whoami /priv
```

Example output:

```powershell
PS C:\htb> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                            Description                                                        State
========================================= ================================================================== ========
SeIncreaseQuotaPrivilege                  Adjust memory quotas for a process                                 Disabled
SeSecurityPrivilege                       Manage auditing and security log                                   Disabled
SeTakeOwnershipPrivilege                  Take ownership of files or other objects                           Disabled
SeLoadDriverPrivilege                     Load and unload device drivers                                     Disabled
SeSystemProfilePrivilege                  Profile system performance                                         Disabled
SeSystemtimePrivilege                     Change the system time                                             Disabled
SeProfileSingleProcessPrivilege           Profile single process                                             Disabled
SeIncreaseBasePriorityPrivilege           Increase scheduling priority                                       Disabled
SeCreatePagefilePrivilege                 Create a pagefile                                                  Disabled
SeBackupPrivilege                         Back up files and directories                                      Disabled
SeRestorePrivilege                        Restore files and directories                                      Disabled
SeShutdownPrivilege                       Shut down the system                                               Disabled
SeDebugPrivilege                          Debug programs                                                     Disabled
SeSystemEnvironmentPrivilege              Modify firmware environment values                                 Disabled
SeChangeNotifyPrivilege                   Bypass traverse checking                                           Enabled
SeRemoteShutdownPrivilege                 Force shutdown from a remote system                                Disabled
SeUndockPrivilege                         Remove computer from docking station                               Disabled
SeManageVolumePrivilege                   Perform volume maintenance tasks                                   Disabled
SeImpersonatePrivilege                    Impersonate a client after authentication                          Enabled
SeCreateGlobalPrivilege                   Create global objects                                              Enabled
SeIncreaseWorkingSetPrivilege             Increase a process working set                                     Disabled
SeTimeZonePrivilege                       Change the time zone                                               Disabled
SeCreateSymbolicLinkPrivilege             Create symbolic links                                              Disabled
SeDelegateSessionUserImpersonatePrivilege Obtain an impersonation token for another user in the same session Disabled
```

---

## Notes on Local Admin Privileges

A local administrator may have many powerful privileges assigned, but many are disabled by default.

Important privileges in the example include:

| Privilege | Why It Matters |
|---|---|
| `SeDebugPrivilege` | Can be used to interact with sensitive processes. |
| `SeBackupPrivilege` | Can read protected files during backup operations. |
| `SeRestorePrivilege` | Can write protected files during restore operations. |
| `SeTakeOwnershipPrivilege` | Can take ownership of securable objects. |
| `SeLoadDriverPrivilege` | Can load drivers, which run as privileged code. |
| `SeImpersonatePrivilege` | Can impersonate authenticated clients. |
| `SeManageVolumePrivilege` | Can perform volume maintenance tasks. |

---

# Standard User Rights

A standard user has far fewer privileges than a local administrator.

## Check Current User

```powershell
whoami
```

Example output:

```powershell
PS C:\htb> whoami

winlpe-srv01\htb-student
```

---

## Check Standard User Privileges

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
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Disabled
```

---

## Notes on Standard User Privileges

A standard user usually has limited privileges.

Common privileges include:

| Privilege | Description |
|---|---|
| `SeChangeNotifyPrivilege` | Allows bypass traverse checking. This is commonly enabled and not usually useful for escalation by itself. |
| `SeIncreaseWorkingSetPrivilege` | Allows increasing a process working set. Usually not a direct escalation path. |

If a standard user has additional privileges, they should be investigated carefully.

---

# Backup Operators Rights

Users in specific groups may receive additional rights.

For example, members of the **Backup Operators** group may have rights that are restricted by UAC but still visible in the privilege list.

## Check Backup Operator Privileges

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
SeShutdownPrivilege           Shut down the system           Disabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Disabled
```

---

## Why Backup Operators Matter

Backup Operators can be dangerous because they may be able to:

- Log on locally to Domain Controllers
- Access sensitive files
- Read registry hives
- Create shadow copies
- Access the file system via SMB
- Interfere with critical systems

Even if the user does not appear to be a direct administrator, membership in Backup Operators should be treated as highly sensitive.

---

# Privilege Enumeration Workflow

## 1. Check Current User

```cmd
whoami
```

---

## 2. Check Current Privileges

```cmd
whoami /priv
```

---

## 3. Check Group Memberships

```cmd
whoami /groups
```

---

## 4. Check Local Administrators

```cmd
net localgroup administrators
```

---

## 5. Check Other Interesting Groups

```cmd
net localgroup "Backup Operators"
```

```cmd
net localgroup "Remote Desktop Users"
```

```cmd
net localgroup "Remote Management Users"
```

```cmd
net localgroup "Hyper-V Administrators"
```

```cmd
net localgroup "Event Log Readers"
```

---

## 6. Check User Rights Assignment Locally

The following command can export local security policy:

```cmd
secedit /export /cfg C:\Windows\Temp\secpol.cfg
```

Review the exported file:

```cmd
type C:\Windows\Temp\secpol.cfg
```

Look for rights such as:

```text
SeBackupPrivilege
SeRestorePrivilege
SeDebugPrivilege
SeImpersonatePrivilege
SeLoadDriverPrivilege
SeTakeOwnershipPrivilege
SeRemoteInteractiveLogonRight
SeNetworkLogonRight
```

---

# High-Value Privileges for Privilege Escalation

| Privilege                       | Potential Abuse                                                   |
| ------------------------------- | ----------------------------------------------------------------- |
| `SeImpersonatePrivilege`        | Token impersonation attacks to escalate to SYSTEM.                |
| `SeAssignPrimaryTokenPrivilege` | Token abuse and process creation with alternate tokens.           |
| `SeDebugPrivilege`              | Dump credentials or interact with privileged processes.           |
| `SeBackupPrivilege`             | Read protected files such as `SAM`, `SYSTEM`, or `NTDS.dit`.      |
| `SeRestorePrivilege`            | Write files to protected locations or overwrite service binaries. |
| `SeTakeOwnershipPrivilege`      | Take ownership of protected files, services, or registry keys.    |
| `SeLoadDriverPrivilege`         | Load a vulnerable or malicious driver.                            |
| `SeManageVolumePrivilege`       | Potential file system abuse.                                      |
| `SeTcbPrivilege`                | Act as part of the operating system. Extremely powerful.          |

---

# Detection

## Event ID 4672

Windows can generate the following event when special privileges are assigned to a new logon session:

```text
4672: Special privileges assigned to new logon
```

This event can be useful for detecting accounts receiving sensitive privileges during logon.

---

## Detection Ideas

Defenders may monitor for:

- Sensitive privileges assigned to unusual users
- `SeDebugPrivilege` assigned outside administrative roles
- `SeBackupPrivilege` assigned to unexpected accounts
- `SeImpersonatePrivilege` assigned to unusual service accounts
- `SeLoadDriverPrivilege` usage
- New logons by privileged accounts
- Local group membership changes
- Use of token manipulation tools
- Access to LSASS
- Shadow copy creation
- Registry hive backup
- Driver loading events

---

## Defensive Considerations

To reduce abuse of Windows privileges:

- Apply least privilege.
- Review User Rights Assignment regularly.
- Restrict local administrator membership.
- Restrict Backup Operators membership.
- Monitor Event ID `4672`.
- Monitor changes to privileged groups.
- Avoid assigning powerful privileges to broad groups.
- Use separate admin accounts.
- Enforce UAC where appropriate.
- Review service account privileges.
- Remove unnecessary rights from service accounts.

---

# Quick Reference

| Purpose | Command |
|---|---|
| Show current user | `whoami` |
| Show current privileges | `whoami /priv` |
| Show current groups | `whoami /groups` |
| Show local administrators | `net localgroup administrators` |
| Show local groups | `net localgroup` |
| Export local security policy | `secedit /export /cfg C:\Windows\Temp\secpol.cfg` |
| Read exported security policy | `type C:\Windows\Temp\secpol.cfg` |

---

# Key Takeaways

- Windows privileges are rights that allow accounts to perform system-level actions.
- Privileges are granted through access tokens at logon.
- Privileges are different from access rights on securable objects.
- Security principals are identified by SIDs.
- Windows compares access tokens against security descriptors to make access decisions.
- Group membership can grant powerful privileges indirectly.
- `whoami /priv` is essential for privilege enumeration.
- Disabled privileges may still be useful if they are assigned to the token.
- UAC affects what privileges are available in elevated versus non-elevated sessions.
- Some groups, such as Backup Operators and DNS Admins, can be extremely dangerous in Active Directory environments.
- Event ID `4672` can help defenders detect special privileges assigned at logon.

---

## Summary

Windows privilege escalation often depends on understanding what privileges and rights the current user has.

The most important questions are:

- What user am I?
- What groups am I in?
- What privileges are assigned to my token?
- Are any privileges disabled but available?
- Am I in a group with dangerous rights?
- Can I abuse these rights locally or in Active Directory?
- Would UAC affect my ability to use these privileges?

Understanding Windows privileges helps identify direct escalation paths and avoid wasting time on techniques that do not apply to the current user context.

---

## Tags

#Windows #PrivilegeEscalation #WindowsPrivilegeEscalation #WindowsPrivileges #UserRightsAssignment #AccessTokens #SecurityPrincipals #SID #DACL #SACL #ACE #Whoami #UAC #SeImpersonatePrivilege #SeDebugPrivilege #SeBackupPrivilege #SeRestorePrivilege #SeTakeOwnershipPrivilege #HTB #Pentesting #OSCP