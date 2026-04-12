# Enumerating Security Controls

## Overview

After gaining a foothold, one of the first things we should do is assess the **defensive state** of the environment.

This helps us answer questions like:

- What protections are in place on this host?
- Will common AD enumeration tools get blocked?
- Do we need to live off the land instead of dropping tooling?
- Are controls applied consistently across the environment?

Understanding security controls early helps shape the rest of the assessment. It influences:

- tool choice
- execution method
- OPSEC considerations
- privilege escalation strategy
- post-exploitation decisions

Not every host is protected equally. One workstation may be tightly locked down, while another may have weaker controls or looser policy enforcement.

> **Note:** This note is about identifying common security controls in an AD environment. It is not a bypass guide.

---

# Why Enumerate Security Controls?

Security tooling changes how we operate.

For example:

- **Defender** may block PowerView or unsigned tooling
- **AppLocker** may prevent execution of PowerShell or binaries from user-writable paths
- **Constrained Language Mode** may cripple PowerShell functionality
- **LAPS** may prevent local admin password reuse, but also become a target for delegated password readers

Think of this phase like checking the tripwires in a building before walking deeper inside.

---

# Windows Defender

## Overview

**Windows Defender** (later branded **Microsoft Defender**) has become much more effective over time and can block many common offensive tools by default.

A common first check is whether real-time protection is enabled.

## Checking Defender Status

```powershell
Get-MpComputerStatus
```

## Example output

```powershell
AMEngineVersion                 : 1.1.17400.5
AMProductVersion                : 4.10.14393.0
AMServiceEnabled                : True
AMServiceVersion                : 4.10.14393.0
AntispywareEnabled              : True
AntispywareSignatureAge         : 1
AntispywareSignatureLastUpdated : 9/2/2020 11:31:50 AM
AntispywareSignatureVersion     : 1.323.392.0
AntivirusEnabled                : True
AntivirusSignatureAge           : 1
AntivirusSignatureLastUpdated   : 9/2/2020 11:31:51 AM
AntivirusSignatureVersion       : 1.323.392.0
BehaviorMonitorEnabled          : False
ComputerID                      : 07D23A51-F83F-4651-B9ED-110FF2B83A9C
ComputerState                   : 0
FullScanAge                     : 4294967295
FullScanEndTime                 :
FullScanStartTime               :
IoavProtectionEnabled           : False
LastFullScanSource              : 0
LastQuickScanSource             : 2
NISEnabled                      : False
NISEngineVersion                : 0.0.0.0
NISSignatureAge                 : 4294967295
NISSignatureLastUpdated         :
NISSignatureVersion             : 0.0.0.0
OnAccessProtectionEnabled       : False
QuickScanAge                    : 0
QuickScanEndTime                : 9/3/2020 12:50:45 AM
QuickScanStartTime              : 9/3/2020 12:49:49 AM
RealTimeProtectionEnabled       : True
RealTimeScanDirection           : 0
PSComputerName                  :
```

## What to look for

The most important line in many cases is:

```powershell
RealTimeProtectionEnabled : True
```

This tells us Defender is actively protecting the host in real time.

### Why this matters

If Defender is enabled, tools such as:

- PowerView
- SharpView
- custom payloads
- known offensive scripts

may be blocked, quarantined, or generate alerts.

---

# AppLocker

## Overview

**AppLocker** is Microsoft’s application whitelisting solution.

It allows administrators to define which users or groups can run:

- executables
- scripts
- MSI files
- DLLs
- packaged apps

The goal is to stop unapproved software from running.

In practice, this often means:

- blocking `cmd.exe`
- blocking `powershell.exe`
- blocking execution from user-writable folders
- allowing only approved apps from Windows and Program Files

## Why it matters

AppLocker can seriously restrict offensive operations, especially if we rely on:

- PowerShell
- dropped binaries
- scripts executed from Desktop, Temp, Downloads, or AppData

However, AppLocker policies are often incomplete or inconsistently applied.

For example, defenders may block one PowerShell path but forget another.

---

## Checking AppLocker Policy

```powershell
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
```

## Example output

```powershell
PathConditions      : {%SYSTEM32%\WINDOWSPOWERSHELL\V1.0\POWERSHELL.EXE}
PathExceptions      : {}
PublisherExceptions : {}
HashExceptions      : {}
Id                  : 3d57af4a-6cf8-4e5b-acfc-c2c2956061fa
Name                : Block PowerShell
Description         : Blocks Domain Users from using PowerShell on workstations
UserOrGroupSid      : S-1-5-21-2974783224-3764228556-2640795941-513
Action              : Deny

PathConditions      : {%PROGRAMFILES%\*}
PathExceptions      : {}
PublisherExceptions : {}
HashExceptions      : {}
Id                  : 921cc481-6e17-4653-8f75-050b80acca20
Name                : (Default Rule) All files located in the Program Files folder
Description         : Allows members of the Everyone group to run applications that are located in the Program Files folder.
UserOrGroupSid      : S-1-1-0
Action              : Allow

PathConditions      : {%WINDIR%\*}
PathExceptions      : {}
PublisherExceptions : {}
HashExceptions      : {}
Id                  : a61c8b2c-a319-4cd0-9690-d2177cad7b51
Name                : (Default Rule) All files located in the Windows folder
Description         : Allows members of the Everyone group to run applications that are located in the Windows folder.
UserOrGroupSid      : S-1-1-0
Action              : Allow

PathConditions      : {*}
PathExceptions      : {}
PublisherExceptions : {}
HashExceptions      : {}
Id                  : fd686d83-a829-4351-8ff4-27c7de5755d2
Name                : (Default Rule) All files
Description         : Allows members of the local Administrators group to run all applications.
UserOrGroupSid      : S-1-5-32-544
Action              : Allow
```

---

## What this tells us

### Deny rule

```powershell
%SYSTEM32%\WINDOWSPOWERSHELL\V1.0\POWERSHELL.EXE
```

Domain users are denied access to the standard 64-bit PowerShell executable in `System32`.

### Allow rules

Everyone is allowed to run applications from:

- `%PROGRAMFILES%\*`
- `%WINDIR%\*`

Local administrators can run all files.

---

## Why this matters

This type of policy often **looks stricter than it really is**.

If only the standard `System32` PowerShell path is blocked, other executables may still work, such as:

- `%SystemRoot%\SysWOW64\WindowsPowerShell\v1.0\powershell.exe`
- `PowerShell_ISE.exe`

This is a common defensive blind spot.

Think of it like locking the front door but leaving the side entrance open.

---

# PowerShell Constrained Language Mode

## Overview

**Constrained Language Mode (CLM)** limits PowerShell functionality to reduce abuse.

When enabled, PowerShell loses access to many features useful for offensive operations, including:

- COM objects
- many .NET types
- PowerShell classes
- XAML workflows
- other advanced scripting features

This can make many PowerShell-based tools partially or completely unusable.

---

## Checking Language Mode

```powershell
$ExecutionContext.SessionState.LanguageMode
```

## Example output

```powershell
ConstrainedLanguage
```

## Possible values

Common values include:

- `FullLanguage`
- `ConstrainedLanguage`

---

## Why this matters

If the result is:

```powershell
ConstrainedLanguage
```

then PowerShell is heavily restricted.

This can affect:

- PowerView usage
- script-based enumeration
- in-memory execution workflows
- reflective loading techniques
- COM or .NET heavy tooling

In short, PowerShell may still be available, but it will behave more like a toy knife than a multi-tool.

---

# LAPS

## Overview

**LAPS** stands for **Local Administrator Password Solution**.

It is designed to prevent one of the most common AD weaknesses:

- local administrator password reuse across multiple hosts

LAPS works by:

- randomizing the local admin password on each machine
- rotating that password on a schedule
- storing the password in Active Directory attributes
- restricting who can read it

---

## Why LAPS matters

From an attacker perspective, LAPS changes the game.

Without LAPS:
- local admin password reuse can enable lateral movement across many machines

With LAPS:
- each machine may have a unique local admin password
- local admin spraying becomes much less effective

However, LAPS introduces a different angle:

- **who can read those stored passwords?**

If we can identify users or groups with LAPS read rights, we may find another route to privileged access.

---

# LAPSToolkit Functions

Several useful PowerShell functions from **LAPSToolkit** can help enumerate LAPS configuration and access.

---

## Find-LAPSDelegatedGroups

This function identifies groups that have been delegated permission to read LAPS passwords.

```powershell
Find-LAPSDelegatedGroups
```

## Example output

```powershell
OrgUnit                                             Delegated Groups
-------                                             ----------------
OU=Servers,DC=INLANEFREIGHT,DC=LOCAL                INLANEFREIGHT\Domain Admins
OU=Servers,DC=INLANEFREIGHT,DC=LOCAL                INLANEFREIGHT\LAPS Admins
OU=Workstations,DC=INLANEFREIGHT,DC=LOCAL           INLANEFREIGHT\Domain Admins
OU=Workstations,DC=INLANEFREIGHT,DC=LOCAL           INLANEFREIGHT\LAPS Admins
OU=Web Servers,OU=Servers,DC=INLANEFREIGHT,DC=LOCAL INLANEFREIGHT\Domain Admins
OU=Web Servers,OU=Servers,DC=INLANEFREIGHT,DC=LOCAL INLANEFREIGHT\LAPS Admins
OU=SQL Servers,OU=Servers,DC=INLANEFREIGHT,DC=LOCAL INLANEFREIGHT\Domain Admins
OU=SQL Servers,OU=Servers,DC=INLANEFREIGHT,DC=LOCAL INLANEFREIGHT\LAPS Admins
OU=File Servers,OU=Servers,DC=INLANEFREIGHT,DC=L... INLANEFREIGHT\Domain Admins
OU=File Servers,OU=Servers,DC=INLANEFREIGHT,DC=L... INLANEFREIGHT\LAPS Admins
OU=Contractor Laptops,OU=Workstations,DC=INLANEF... INLANEFREIGHT\Domain Admins
OU=Contractor Laptops,OU=Workstations,DC=INLANEF... INLANEFREIGHT\LAPS Admins
OU=Staff Workstations,OU=Workstations,DC=INLANEF... INLANEFREIGHT\Domain Admins
OU=Staff Workstations,OU=Workstations,DC=INLANEF... INLANEFREIGHT\LAPS Admins
OU=Executive Workstations,OU=Workstations,DC=INL... INLANEFREIGHT\Domain Admins
OU=Executive Workstations,OU=Workstations,DC=INL... INLANEFREIGHT\LAPS Admins
OU=Mail Servers,OU=Servers,DC=INLANEFREIGHT,DC=L... INLANEFREIGHT\Domain Admins
OU=Mail Servers,OU=Servers,DC=INLANEFREIGHT,DC=L... INLANEFREIGHT\LAPS Admins
```

### What this tells us

Here, the main delegated groups are:

- `INLANEFREIGHT\Domain Admins`
- `INLANEFREIGHT\LAPS Admins`

If we compromise a member of one of these groups, LAPS passwords may become accessible.

---

## Find-AdmPwdExtendedRights

This function checks which identities have rights over LAPS-enabled computers, including delegated groups and users with **All Extended Rights**.

```powershell
Find-AdmPwdExtendedRights
```

## Example output

```powershell
ComputerName                Identity                    Reason
------------                --------                    ------
EXCHG01.INLANEFREIGHT.LOCAL INLANEFREIGHT\Domain Admins Delegated
EXCHG01.INLANEFREIGHT.LOCAL INLANEFREIGHT\LAPS Admins   Delegated
SQL01.INLANEFREIGHT.LOCAL   INLANEFREIGHT\Domain Admins Delegated
SQL01.INLANEFREIGHT.LOCAL   INLANEFREIGHT\LAPS Admins   Delegated
WS01.INLANEFREIGHT.LOCAL    INLANEFREIGHT\Domain Admins Delegated
WS01.INLANEFREIGHT.LOCAL    INLANEFREIGHT\LAPS Admins   Delegated
```

### Why this matters

This helps identify:

- who can read LAPS passwords
- whether rights are explicitly delegated
- where to focus if you want to target a user who can retrieve local admin credentials

A particularly important case is users with **All Extended Rights**, because they may not stand out as obviously as delegated admin groups.

---

## Get-LAPSComputers

This function can enumerate LAPS-enabled computers and, if the current user has permission, retrieve the passwords in cleartext.

```powershell
Get-LAPSComputers
```

## Example output

```powershell
ComputerName                Password       Expiration
------------                --------       ----------
DC01.INLANEFREIGHT.LOCAL    6DZ[+A/[]19d$F 08/26/2020 23:29:45
EXCHG01.INLANEFREIGHT.LOCAL oj+2A+[hHMMtj, 09/26/2020 00:51:30
SQL01.INLANEFREIGHT.LOCAL   9G#f;p41dcAe,s 09/26/2020 00:30:09
WS01.INLANEFREIGHT.LOCAL    TCaG-F)3No;l8C 09/26/2020 00:46:04
```

### What this tells us

This reveals:

- which computers have LAPS enabled
- their current local admin password
- password expiration time

If your account can see the `Password` column, that is a major finding.

---

# Why LAPS Enumeration Is Valuable

LAPS can both **defend** and **expose**.

It defends by preventing reused local admin passwords.

But if you compromise a user or group with rights to read LAPS attributes, it can hand you clean local admin credentials for many systems.

So LAPS is not just something to note as “enabled” or “disabled”.

It is something to map:

- where it is enabled
- who can read it
- which systems are covered
- which systems are missing it

---

# What to Look for During Enumeration

When assessing security controls, focus on the following questions:

## Defender

- Is real-time protection enabled?
- Are AV signatures current?
- Is behavior monitoring on?
- Is on-access protection active?

## AppLocker

- Are scripts blocked?
- Is PowerShell blocked only by path?
- Are writable directories executable?
- Are default allow rules too broad?

## PowerShell Language Mode

- Is it `FullLanguage` or `ConstrainedLanguage`?
- Will PowerShell-based tooling function correctly?

## LAPS

- Is it deployed?
- Which OUs use it?
- Who can read passwords?
- Are there systems without LAPS?
- Can the current user retrieve any passwords?

---

# Practical Interpretation

## If Defender is enabled

Expect common offensive tooling to be flagged or blocked.

## If AppLocker is present

Expect execution restrictions, especially for:

- PowerShell
- dropped EXEs
- scripts from user directories

## If Constrained Language Mode is enabled

Expect PowerShell functionality to be reduced significantly.

## If LAPS is deployed correctly

Expect local admin password reuse to be limited.

## If LAPS delegation is weak

Expect opportunities to retrieve local admin passwords through privilege abuse.

---

# Quick Commands

## Check Defender status

```powershell
Get-MpComputerStatus
```

## Check effective AppLocker policy

```powershell
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
```

## Check PowerShell language mode

```powershell
$ExecutionContext.SessionState.LanguageMode
```

## Find LAPS delegated groups

```powershell
Find-LAPSDelegatedGroups
```

## Find extended rights over LAPS-enabled computers

```powershell
Find-AdmPwdExtendedRights
```

## Enumerate LAPS-enabled computers and visible passwords

```powershell
Get-LAPSComputers
```

---

# Key Takeaways

- Enumerating security controls early helps guide safe and effective operations
- Defender may block common scripts and payloads
- AppLocker can restrict execution, but policies are often incomplete
- Constrained Language Mode can cripple PowerShell-heavy workflows
- LAPS prevents local admin reuse, but delegated readers become valuable targets
- Controls are often inconsistent across hosts, so one machine’s restrictions may not reflect another’s

---

# Conclusion

Security control enumeration is about understanding the terrain before moving.

You are not just asking:

- “What tools do I want to use?”

You are asking:

- “What will this host allow me to do?”
- “What will get blocked?”
- “What will get logged?”
- “Where are the gaps in control coverage?”

That awareness helps you move smarter, quieter, and more effectively through an AD environment.

---

# Tags

#active-directory  
#enumeration  
#security-controls  
#windows-defender  
#applocker  
#powershell  
#constrained-language-mode  
#laps  
#defense  
#post-exploitation  
#obsidian