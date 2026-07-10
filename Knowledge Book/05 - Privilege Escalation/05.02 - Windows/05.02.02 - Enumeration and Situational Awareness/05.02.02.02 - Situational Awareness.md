## Overview

Situational awareness is the first structured enumeration phase after gaining access to a Windows host.

The goal is to understand:

- who we are
- what host we are on
- what privileges and groups we have
- what users and groups exist locally
- what operating system and architecture are in use
- what network interfaces, routes, and connections exist
- whether the host is domain joined
- what applications are installed
- what processes and services are running
- what defensive controls may block tooling or trigger alerts
- whether the host provides access to additional networks or systems

> [!important]
> Do not jump straight into exploitation. Situational awareness helps decide whether local privilege escalation is worthwhile, which techniques are likely to work, and whether the host can support lateral movement.

---

# Why Situational Awareness Matters

Situational awareness can reveal:

- local privilege escalation paths
- reachable internal networks
- dual-homed hosts
- active administrator sessions
- domain controller or DNS information
- interesting local users
- privileged groups
- remote access groups
- installed applications with saved credentials
- running services such as web servers or databases
- AV, EDR, Defender, or AppLocker controls
- allowed and blocked execution paths

Not every compromised host needs to be rooted. In a real assessment, the better question is:

```text
Will privileged access on this host help me move toward the objective?
```

For example:

- A workstation with an active admin RDP session may be high value.
- A dual-homed server may provide access to a hidden subnet.
- A host running KeePass, FileZilla, XAMPP, or MySQL may contain credentials or sensitive data.
- A machine with restrictive AppLocker rules may require different tooling.

---

# First-Pass Workflow

Run these checks first to orient yourself.

```cmd
hostname
whoami
whoami /priv
whoami /groups
systeminfo
ipconfig /all
arp -a
route print
netstat -ano
```

Then continue with:

```cmd
net user
net localgroup
```

PowerShell equivalents:

```powershell
Get-LocalUser
Get-LocalGroup
Get-LocalGroupMember <group>
Get-Process
```

---

# 1. Identify Current User and Host

## Current User

```cmd
whoami
```

Example:

```cmd
C:\Users\dave> whoami

clientwk220\dave
```

This tells us:

| Item | Meaning |
|---|---|
| `clientwk220` | Hostname or local computer name |
| `dave` | Current user context |

Hostnames can reveal system purpose.

Examples:

```text
WEB01
MSSQL01
CLIENTWK220
DC01
BACKUP01
FILE01
```

Useful assumptions:

| Hostname Pattern | Possible Role |
|---|---|
| `WEB` | Web server |
| `SQL`, `MSSQL`, `DB` | Database server |
| `DC` | Domain controller |
| `FILE` | File server |
| `BK`, `BACKUP` | Backup server |
| `CLIENT`, `WK`, `LAPTOP` | Workstation |

> [!tip]
> Hostnames are not proof, but they often provide useful direction for further enumeration.

---

# 2. Enumerate Current User Privileges

## Check Token Privileges

```cmd
whoami /priv
```

Look for privileges such as:

| Privilege | Why It Matters |
|---|---|
| `SeImpersonatePrivilege` | May enable Potato-style privilege escalation. |
| `SeAssignPrimaryTokenPrivilege` | May support token-based privilege escalation. |
| `SeBackupPrivilege` | Can read protected files and registry hives. |
| `SeRestorePrivilege` | Can overwrite protected files. |
| `SeDebugPrivilege` | Can access/debug privileged processes. |
| `SeTakeOwnershipPrivilege` | Can take ownership of files or objects. |
| `SeLoadDriverPrivilege` | May allow driver-based escalation. |
| `SeManageVolumePrivilege` | Can sometimes be abused for file access/escalation. |

Example high-value finding:

```text
SeImpersonatePrivilege        Enabled
```

Potential follow-up:

```text
JuicyPotato
PrintSpoofer
RoguePotato
GodPotato
```

---

# 3. Enumerate Current User Groups

## Check Group Memberships

```cmd
whoami /groups
```

PowerShell equivalent:

```powershell
[System.Security.Principal.WindowsIdentity]::GetCurrent().Groups
```

Example useful groups:

| Group | Why It Matters |
|---|---|
| `Administrators` | Full local administrative access. |
| `Remote Desktop Users` | Can log in over RDP. |
| `Remote Management Users` | Can use WinRM. |
| `Backup Operators` | Can back up and restore files regardless of ACLs. |
| `Event Log Readers` | Can read event logs, sometimes useful for credentials or activity. |
| `DnsAdmins` | Domain privilege escalation possibility in AD environments. |
| `Print Operators` | Can have elevated rights on certain systems. |
| Custom groups | May imply business-specific permissions. |

Example:

```text
CLIENTWK220\helpdesk
BUILTIN\Remote Desktop Users
BUILTIN\Users
```

Interpretation:

- `helpdesk` may have extra access compared to a normal user.
- `Remote Desktop Users` means credentials for this user may allow GUI access over RDP.
- Custom groups should be investigated.

---

# 4. Enumerate Local Users

## Using net user

```cmd
net user
```

## Using PowerShell

```powershell
Get-LocalUser
```

Example:

```powershell
Get-LocalUser

Name               Enabled Description
----               ------- -----------
Administrator      False   Built-in account for administering the computer/domain
BackupAdmin        True
dave               True    dave
daveadmin          True
DefaultAccount     False   A user account managed by the system.
Guest              False   Built-in account for guest access to the computer/domain
offsec             True
steve              True
```

## What to Look For

| Item | Why It Matters |
|---|---|
| Enabled admin-like users | Potential privilege escalation or credential targets. |
| Disabled Administrator | Built-in admin may not be usable. |
| Users ending in `admin` | May indicate privileged companion accounts. |
| Backup-related accounts | May have access to sensitive files or systems. |
| Descriptions | Sometimes contain operational notes or passwords. |
| Service-like users | May be tied to applications or scheduled tasks. |

Interesting examples:

```text
daveadmin
BackupAdmin
sqlsvc
sccm_svc
backup_svc
```

> [!tip]
> Admins often have separate daily-use and privileged accounts, such as `dave` and `daveadmin`.

---

# 5. Enumerate Local Groups

## Using net localgroup

```cmd
net localgroup
```

## Using PowerShell

```powershell
Get-LocalGroup
```

Example:

```powershell
Get-LocalGroup

Name                                Description
----                                -----------
adminteam                           Members of this group are admins to all workstations on the second floor
BackupUsers
helpdesk
Administrators                      Administrators have complete and unrestricted access to the computer/domain
Remote Desktop Users                Members in this group are granted the right to logon remotely
```

## What to Look For

| Group | Why It Matters |
|---|---|
| `Administrators` | Local admin users. |
| `Remote Desktop Users` | Users who can RDP. |
| `Remote Management Users` | Users who can use WinRM. |
| `Backup Operators` | Can read protected files via backup rights. |
| Custom admin groups | May imply delegated admin rights. |
| Backup groups | May control backup software or sensitive file access. |
| Helpdesk groups | May have support or workstation admin privileges. |

Interesting custom group:

```text
adminteam
```

Description:

```text
Members of this group are admins to all workstations on the second floor
```

This may not immediately give local admin on the current host, but it may become useful after identifying systems on the second floor.

---

# 6. Enumerate Group Members

Use PowerShell:

```powershell
Get-LocalGroupMember Administrators
```

```powershell
Get-LocalGroupMember "Remote Desktop Users"
```

```powershell
Get-LocalGroupMember "Remote Management Users"
```

```powershell
Get-LocalGroupMember "Backup Operators"
```

Check custom groups:

```powershell
Get-LocalGroupMember adminteam
```

Example:

```powershell
Get-LocalGroupMember Administrators

ObjectClass Name                      PrincipalSource
----------- ----                      ---------------
User        CLIENTWK220\Administrator Local
User        CLIENTWK220\daveadmin     Local
User        CLIENTWK220\backupadmin   Local
User        CLIENTWK220\offsec        Local
```

## Why This Matters

This identifies:

- local administrator targets
- users worth password hunting for
- users who can RDP or WinRM
- custom delegated admin groups
- backup users with broad access

High-value findings:

```text
CLIENTWK220\daveadmin
CLIENTWK220\backupadmin
```

---

# 7. Identify OS, Version, Architecture, and Patch Context

## System Information

```cmd
systeminfo
```

Useful fields:

| Field | Why It Matters |
|---|---|
| `OS Name` | Identifies Windows edition. |
| `OS Version` | Helps determine exploit compatibility. |
| `Build` | Used to map patch level and feature release. |
| `System Type` | Determines x86 vs x64 payload/tooling. |
| `Hotfix(s)` | Shows installed KBs. |
| `Domain` | Indicates domain membership. |
| `Logon Server` | May reveal domain controller. |
| `Original Install Date` | Older installs may have legacy misconfigs. |

Example:

```text
OS Name: Microsoft Windows 11 Pro
OS Version: 10.0.22621 N/A Build 22621
System Type: x64-based PC
```

Interpretation:

```text
Windows 11 Pro 22H2
64-bit system
```

> [!important]
> Architecture matters. A 64-bit tool will not run on a 32-bit system. A 32-bit shell on a 64-bit system may also affect exploit behavior.

---

# 8. Enumerate Installed Hotfixes

## WMIC

```cmd
wmic qfe
```

## PowerShell

```powershell
Get-HotFix
```

## Why This Matters

Installed KBs help identify:

- missing patch privilege escalation paths
- legacy systems
- vulnerable OS builds
- patch gaps
- exploit compatibility

If needed, save `systeminfo` output and analyze offline with tools such as:

```text
Windows-Exploit-Suggester
Sherlock
wesng
```

---

# 9. Gather Network Interface and DNS Information

## ipconfig /all

```cmd
ipconfig /all
```

This reveals:

- IP addresses
- network adapters
- subnet masks
- default gateways
- DNS servers
- DHCP servers
- DNS suffixes
- NetBIOS status
- multiple network interfaces

## What to Look For

| Field | Why It Matters |
|---|---|
| Host Name | Confirms system name. |
| Primary DNS Suffix | May indicate domain membership. |
| DNS Suffix Search List | May reveal internal domains. |
| IPv4 Address | Identifies current subnet. |
| Subnet Mask | Helps determine network size. |
| Default Gateway | Shows route out of subnet. |
| DNS Servers | Often domain controllers in AD environments. |
| DHCP Server | Infrastructure target. |
| Multiple Adapters | May indicate dual-homed host. |
| NetBIOS over TCP/IP | May enable legacy enumeration. |

Example:

```text
IPv4 Address: 192.168.50.220
Subnet Mask: 255.255.255.0
Default Gateway: 192.168.50.254
DNS Servers: 8.8.8.8
```

---

# 10. Identify Dual-Homed Hosts

A dual-homed host has two or more network interfaces or routes into multiple networks.

Indicators:

```text
Multiple Ethernet adapters
Multiple IPv4 addresses
Multiple default gateways
Multiple internal routes
Different DNS suffixes per adapter
Routes to separate subnets
```

Why this matters:

```text
A dual-homed host may provide access to an internal subnet that is unreachable from the original attack box.
```

Check with:

```cmd
ipconfig /all
```

```cmd
route print
```

```cmd
arp -a
```

---

# 11. Review ARP Cache

## Command

```cmd
arp -a
```

## Why Check ARP?

The ARP cache shows systems the host has recently communicated with on local network segments.

This can reveal:

- nearby hosts
- gateways
- recently contacted systems
- potential lateral movement targets
- admin workstations
- servers accessed by the current host
- hosts not discovered from the attack box

## What to Look For

| Item | Meaning |
|---|---|
| Dynamic entries | Recently contacted hosts. |
| Gateway IP | Router/default gateway. |
| Multiple interfaces | Possible dual-homed host. |
| Unknown internal IPs | Potential targets. |
| Repeated entries | Frequently contacted systems. |

Example:

```cmd
arp -a
```

```text
Interface: 10.129.43.8 --- 0x4
  Internet Address      Physical Address      Type
  10.129.0.1            00-50-56-b9-4d-df     dynamic
  10.129.43.12          00-50-56-b9-da-ad     dynamic
  10.129.43.13          00-50-56-b9-5b-9f     dynamic
```

Potential follow-up:

```text
10.129.43.12
10.129.43.13
```

---

# 12. Review Routing Table

## Command

```cmd
route print
```

## Why Check Routes?

The routing table shows how traffic leaves the host.

It can reveal:

- directly connected networks
- hidden internal subnets
- persistent routes
- multiple default routes
- IPv6 routes
- preferred interfaces
- additional reachable networks

## What to Look For

| Item | Why It Matters |
|---|---|
| `0.0.0.0/0` | Default route. |
| Multiple default routes | Multiple network paths. |
| On-link routes | Directly reachable networks. |
| Persistent routes | Manually configured routes. |
| Interface metrics | Lower metric is usually preferred. |
| Internal ranges | Extra subnets to investigate. |
| IPv6 routes | IPv6 may reveal missed attack paths. |

Example interesting routes:

```text
10.129.0.0/16
192.168.20.0/24
```

This suggests the host can reach at least two networks.

---

# 13. Review Active Network Connections

## Command

```cmd
netstat -ano
```

Flags:

| Flag | Purpose |
|---|---|
| `-a` | Show all active connections and listening ports. |
| `-n` | Do not resolve names. |
| `-o` | Show process ID. |

Example:

```cmd
netstat -ano
```

```text
Proto  Local Address          Foreign Address        State           PID
TCP    0.0.0.0:80             0.0.0.0:0              LISTENING       3340
TCP    0.0.0.0:443            0.0.0.0:0              LISTENING       3340
TCP    0.0.0.0:3306           0.0.0.0:0              LISTENING       3508
TCP    0.0.0.0:3389           0.0.0.0:0              LISTENING       1148
TCP    192.168.50.220:3389    192.168.48.3:33770     ESTABLISHED     1148
TCP    192.168.50.220:4444    192.168.48.3:58386     ESTABLISHED     2064
```

## What to Look For

| Port / State | Meaning |
|---|---|
| `80`, `443` listening | Web server. |
| `3306` listening | MySQL. |
| `3389` listening | RDP enabled. |
| `5985`, `5986` listening | WinRM enabled. |
| `445` listening | SMB. |
| Established RDP | Another user may be logged in. |
| Unknown outbound connections | Possible agents, apps, or C2-like traffic. |

Map PID to process:

```cmd
tasklist /FI "PID eq 3508"
```

PowerShell:

```powershell
Get-Process -Id 3508
```

> [!important]
> An active RDP connection from another host may indicate a logged-on user. After privilege escalation, this could become relevant for credential access or session enumeration.

---

# 14. Check Domain Context

## Environment Variables

```cmd
echo %USERDOMAIN%
```

```cmd
echo %LOGONSERVER%
```

```cmd
echo %COMPUTERNAME%
```

## Workstation Configuration

```cmd
net config workstation
```

Useful fields:

| Field | Why It Matters |
|---|---|
| Workstation domain | Domain or workgroup membership. |
| Logon domain | Current logon context. |
| Logon server | May reveal domain controller. |
| DNS suffix | Internal AD namespace. |

## Domain Controller Discovery

```cmd
nltest /dsgetdc:
```

If a domain name is known:

```cmd
nltest /dsgetdc:<domain>
```

PowerShell:

```powershell
[System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
```

> [!note]
> Some commands may fail if the host is not domain joined, DNS is misconfigured, or current context lacks domain access.

---

# 15. Enumerate Installed Applications

Installed applications may reveal:

- exploitable software
- credential stores
- password managers
- FTP clients
- remote access tools
- web stacks
- database clients
- VPN software
- backup software
- virtualization tools

## Check Program Files

```cmd
dir "C:\Program Files"
```

```cmd
dir "C:\Program Files (x86)"
```

Also check user directories:

```cmd
dir "%USERPROFILE%\Downloads"
```

```cmd
dir "%USERPROFILE%\Desktop"
```

```cmd
dir "%LOCALAPPDATA%\Programs"
```

---

# 16. Enumerate Installed Applications via Registry

## 32-bit Applications

```powershell
Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | select DisplayName
```

## 64-bit Applications

```powershell
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" | select DisplayName
```

## Combined View with Version and Install Path

```powershell
$INSTALLED = Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* | Select-Object DisplayName, DisplayVersion, InstallLocation
$INSTALLED += Get-ItemProperty HKLM:\Software\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\* | Select-Object DisplayName, DisplayVersion, InstallLocation
$INSTALLED | Where-Object { $_.DisplayName -ne $null } | Sort-Object -Property DisplayName -Unique | Format-Table -AutoSize
```

## Interesting Applications

| Application | Why It Matters |
|---|---|
| KeePass | Password database may exist. |
| FileZilla | Saved FTP credentials may exist. |
| WinSCP | Saved sessions/credentials. |
| PuTTY | Saved sessions, keys, proxy creds. |
| mRemoteNG | Saved RDP/SSH/VNC credentials. |
| TeamViewer | Remote access configuration. |
| OpenVPN | VPN profiles and keys. |
| XAMPP | Apache/MySQL configs and web roots. |
| MySQL/MSSQL clients | Database access. |
| Browsers | Saved passwords, cookies, tokens. |
| Backup software | High-value access and restored data. |
| VMware/VirtualBox tools | Virtualization context. |

---

# 17. Enumerate Running Processes

## PowerShell

```powershell
Get-Process
```

## CMD

```cmd
tasklist
```

With services:

```cmd
tasklist /svc
```

Map process to PID from `netstat`:

```powershell
Get-Process -Id <PID>
```

Example:

```powershell
Get-Process -Id 3508
```

## What to Look For

| Process | Possible Meaning |
|---|---|
| `httpd.exe` | Apache web server. |
| `mysqld.exe` | MySQL database. |
| `w3wp.exe` | IIS worker process. |
| `sqlservr.exe` | MSSQL server. |
| `filezilla.exe` | FTP client or server. |
| `keepass.exe` | Password manager running. |
| `TeamViewer.exe` | Remote access tool. |
| `powershell.exe` | Scripts or admin activity. |
| `cmd.exe` | Interactive shells or scripts. |
| AV/EDR processes | Defensive tooling. |

Interpretation example:

```text
httpd.exe + mysqld.exe + XAMPP installed = likely local web stack with database-backed app.
```

---

# 18. Enumerate Services

Services can reveal:

- privilege escalation vectors
- unquoted service paths
- weak service permissions
- interesting applications
- service accounts
- startup binaries
- web/database stacks

## List Services

```cmd
sc query
```

PowerShell:

```powershell
Get-Service
```

WMI with paths:

```cmd
wmic service get name,displayname,pathname,startmode
```

PowerShell with paths:

```powershell
Get-CimInstance Win32_Service | Select-Object Name, DisplayName, State, StartMode, StartName, PathName
```

## What to Look For

| Item | Why It Matters |
|---|---|
| `StartName` | Shows service account. |
| `PathName` | Shows executable path and arguments. |
| Unquoted paths | Potential service path hijack. |
| Writable service binary | Replace binary for escalation. |
| Custom services | More likely to be misconfigured. |
| Backup/database services | Often privileged and sensitive. |

---

# 19. Enumerate Security Protections

## Why This Matters

Security tools can block or detect:

- public exploits
- credential dumping
- PowerShell abuse
- encoded commands
- reverse shells
- LOLBAS abuse
- suspicious parent-child process chains
- unsigned binaries
- script execution
- tool upload and execution

Before running tools, identify what protections are active.

---

# 20. Check Windows Defender

## PowerShell

```powershell
Get-MpComputerStatus
```

Important fields:

| Field | Meaning |
|---|---|
| `AMServiceEnabled` | Defender antimalware service enabled. |
| `AntivirusEnabled` | Antivirus enabled. |
| `AntispywareEnabled` | Antispyware enabled. |
| `RealTimeProtectionEnabled` | Real-time scanning enabled. |
| `BehaviorMonitorEnabled` | Behavior monitoring enabled. |
| `IoavProtectionEnabled` | Scans downloaded/attachment files. |
| `OnAccessProtectionEnabled` | Scans files when accessed. |
| `AntivirusSignatureLastUpdated` | Signature freshness. |

## Service Check

```cmd
sc query windefend
```

## Security Center AV Query

```cmd
wmic /namespace:\\root\SecurityCenter2 path AntiVirusProduct get displayName,pathToSignedProductExe
```

> [!tip]
> Do not only check whether Defender exists. Check whether real-time protection and behavior monitoring are enabled.

---

# 21. Identify AV / EDR Processes

## tasklist

```cmd
tasklist
```

## WMIC

```cmd
wmic process get ProcessId,Name,ExecutablePath
```

Look for vendor processes related to:

```text
Defender
CrowdStrike
SentinelOne
Sophos
Carbon Black
McAfee
Trellix
Symantec
Trend Micro
Elastic
Cylance
ESET
Bitdefender
Kaspersky
Palo Alto Cortex
```

> [!warning]
> Some EDRs alert on ordinary commands when executed from suspicious contexts. Tool choice and process chain matter.

---

# 22. Enumerate AppLocker

## What AppLocker Controls

AppLocker can restrict:

- executables
- scripts
- Windows Installer files
- DLLs
- packaged apps

Rules may be based on:

- path
- hash
- publisher signature

Common default allow paths:

```text
%WINDIR%\*
%PROGRAMFILES%\*
```

Common blocked user-writable paths:

```text
C:\Users\<user>\Desktop
C:\Users\<user>\Downloads
C:\Users\<user>\AppData
C:\Users\Public
C:\Windows\Temp
```

---

# 23. List Effective AppLocker Rules

```powershell
Get-AppLockerPolicy -Effective | Select-Object -ExpandProperty RuleCollections
```

What to inspect:

| Item | Why It Matters |
|---|---|
| `Action` | Allow or deny. |
| `UserOrGroupSid` | Who rule applies to. |
| `PathConditions` | Allowed or blocked paths. |
| `PublisherConditions` | Signed binary rules. |
| `HashExceptions` | Specific file exceptions. |
| `PathExceptions` | Excluded paths. |
| Script rules | Affect `.ps1`, `.bat`, `.cmd`, `.vbs`, `.js`. |
| Executable rules | Affect `.exe`. |
| MSI rules | Affect installer techniques. |
| DLL rules | Affect DLL loading. |

---

# 24. Test AppLocker Policy

Test whether a file would be allowed or denied.

```powershell
Get-AppLockerPolicy -Local | Test-AppLockerPolicy -Path C:\Windows\System32\cmd.exe -User Everyone
```

Example output:

```text
FilePath                    PolicyDecision MatchingRule
--------                    -------------- ------------
C:\Windows\System32\cmd.exe Denied         c:\windows\system32\cmd.exe
```

If `cmd.exe` or `powershell.exe` is denied, enumeration and exploitation methods must adapt.

Test other paths:

```powershell
Get-AppLockerPolicy -Local | Test-AppLockerPolicy -Path C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -User Everyone
```

```powershell
Get-AppLockerPolicy -Local | Test-AppLockerPolicy -Path C:\Users\Public\tool.exe -User Everyone
```

---

# 25. Check PowerShell Language Mode

```powershell
$ExecutionContext.SessionState.LanguageMode
```

Possible values:

```text
FullLanguage
ConstrainedLanguage
NoLanguage
RestrictedLanguage
```

Why it matters:

| Mode | Impact |
|---|---|
| `FullLanguage` | Normal PowerShell functionality. |
| `ConstrainedLanguage` | Blocks many .NET and advanced features. |
| `NoLanguage` | No script text execution. |
| `RestrictedLanguage` | Heavily limited syntax. |

---

# 26. Check PowerShell Execution Policy

```powershell
Get-ExecutionPolicy -List
```

Current-process bypass:

```powershell
Set-ExecutionPolicy Bypass -Scope Process
```

> [!note]
> Execution policy is not a security boundary, but it can affect how scripts run.

---

# 27. Pull Findings Together

After running situational awareness checks, summarize the host.

Example summary:

```text
Current user: CLIENTWK220\dave
Groups: helpdesk, Remote Desktop Users, Users
Privileged targets: daveadmin, BackupAdmin
OS: Windows 11 Pro Build 22621 x64
Network: 192.168.50.220/24 via 192.168.50.254
Listening services: 80, 443, 3306, 3389
Active sessions: RDP from 192.168.48.3
Installed apps: KeePass, FileZilla, XAMPP, 7-Zip
Running apps: httpd, mysqld, filezilla
Protections: Defender status checked, AppLocker checked
Potential next steps: credential hunting, web/database config review, KeePass search, RDP/session investigation, service enumeration
```

This converts raw enumeration into an actionable plan.

---

# Practical Operator Workflow

## Phase 1 – Identity and Access

```cmd
hostname
whoami
whoami /priv
whoami /groups
```

Then:

```powershell
Get-LocalUser
Get-LocalGroup
Get-LocalGroupMember Administrators
Get-LocalGroupMember "Remote Desktop Users"
Get-LocalGroupMember "Remote Management Users"
Get-LocalGroupMember "Backup Operators"
```

Ask:

```text
Who am I?
Am I privileged?
Can I RDP or WinRM?
Are there admin-like local users?
Are there custom groups with useful descriptions?
```

---

## Phase 2 – Host and OS

```cmd
systeminfo
wmic qfe
```

```powershell
Get-HotFix
```

Ask:

```text
What OS is this?
What architecture is it?
Is it old or missing patches?
Can known local exploits apply?
```

---

## Phase 3 – Network

```cmd
ipconfig /all
arp -a
route print
netstat -ano
```

Ask:

```text
What subnet am I on?
Is this host dual-homed?
What systems has it contacted?
What services are listening?
Are other users connected?
Can this host reach networks I cannot?
```

---

## Phase 4 – Domain Context

```cmd
echo %USERDOMAIN%
echo %LOGONSERVER%
net config workstation
nltest /dsgetdc:
```

Ask:

```text
Is the host domain joined?
What domain context exists?
Can I identify a domain controller?
```

---

## Phase 5 – Applications and Processes

```cmd
dir "C:\Program Files"
dir "C:\Program Files (x86)"
dir "%USERPROFILE%\Downloads"
tasklist
tasklist /svc
```

```powershell
Get-Process
Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | select DisplayName
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" | select DisplayName
```

Ask:

```text
What software is installed?
What software is running?
Are there password managers, FTP clients, VPNs, web stacks, databases, or backup tools?
```

---

## Phase 6 – Services

```cmd
sc query
wmic service get name,displayname,pathname,startmode
```

```powershell
Get-CimInstance Win32_Service | Select-Object Name, DisplayName, State, StartMode, StartName, PathName
```

Ask:

```text
Are custom services running?
Do services run as privileged accounts?
Are service paths misconfigured?
Are service binaries writable?
```

---

## Phase 7 – Protections and Execution Constraints

```powershell
Get-MpComputerStatus
Get-AppLockerPolicy -Effective | Select-Object -ExpandProperty RuleCollections
Get-AppLockerPolicy -Local | Test-AppLockerPolicy -Path C:\Windows\System32\cmd.exe -User Everyone
$ExecutionContext.SessionState.LanguageMode
Get-ExecutionPolicy -List
```

```cmd
sc query windefend
tasklist
wmic /namespace:\\root\SecurityCenter2 path AntiVirusProduct get displayName,pathToSignedProductExe
```

Ask:

```text
What security controls are running?
Will common tools be detected?
Is PowerShell restricted?
Is AppLocker enforcing path or publisher rules?
Where can binaries/scripts execute from?
```

---

# Command Reference

## Identity

```cmd
hostname
```

```cmd
whoami
```

```cmd
whoami /priv
```

```cmd
whoami /groups
```

---

## Users and Groups

```cmd
net user
```

```cmd
net localgroup
```

```powershell
Get-LocalUser
```

```powershell
Get-LocalGroup
```

```powershell
Get-LocalGroupMember Administrators
```

```powershell
Get-LocalGroupMember "Remote Desktop Users"
```

```powershell
Get-LocalGroupMember "Remote Management Users"
```

```powershell
Get-LocalGroupMember "Backup Operators"
```

---

## OS and Patches

```cmd
systeminfo
```

```cmd
wmic qfe
```

```powershell
Get-HotFix
```

---

## Network

```cmd
ipconfig /all
```

```cmd
arp -a
```

```cmd
route print
```

```cmd
netstat -ano
```

---

## Domain Context

```cmd
echo %USERDOMAIN%
```

```cmd
echo %LOGONSERVER%
```

```cmd
echo %COMPUTERNAME%
```

```cmd
net config workstation
```

```cmd
nltest /dsgetdc:
```

---

## Installed Applications

```cmd
dir "C:\Program Files"
```

```cmd
dir "C:\Program Files (x86)"
```

```cmd
dir "%USERPROFILE%\Downloads"
```

```powershell
Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | select DisplayName
```

```powershell
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" | select DisplayName
```

```powershell
$INSTALLED = Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* | Select-Object DisplayName, DisplayVersion, InstallLocation
$INSTALLED += Get-ItemProperty HKLM:\Software\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\* | Select-Object DisplayName, DisplayVersion, InstallLocation
$INSTALLED | Where-Object { $_.DisplayName -ne $null } | Sort-Object -Property DisplayName -Unique | Format-Table -AutoSize
```

---

## Processes and Services

```cmd
tasklist
```

```cmd
tasklist /svc
```

```cmd
tasklist /FI "PID eq <PID>"
```

```powershell
Get-Process
```

```powershell
Get-Process -Id <PID>
```

```cmd
sc query
```

```cmd
wmic service get name,displayname,pathname,startmode
```

```powershell
Get-CimInstance Win32_Service | Select-Object Name, DisplayName, State, StartMode, StartName, PathName
```

---

## Defender and AV

```powershell
Get-MpComputerStatus
```

```cmd
sc query windefend
```

```cmd
wmic /namespace:\\root\SecurityCenter2 path AntiVirusProduct get displayName,pathToSignedProductExe
```

```cmd
tasklist
```

---

## AppLocker and PowerShell Restrictions

```powershell
Get-AppLockerPolicy -Effective | Select-Object -ExpandProperty RuleCollections
```

```powershell
Get-AppLockerPolicy -Local | Test-AppLockerPolicy -Path C:\Windows\System32\cmd.exe -User Everyone
```

```powershell
Get-AppLockerPolicy -Local | Test-AppLockerPolicy -Path C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -User Everyone
```

```powershell
$ExecutionContext.SessionState.LanguageMode
```

```powershell
Get-ExecutionPolicy -List
```

```powershell
Set-ExecutionPolicy Bypass -Scope Process
```

---

# Interpreting Findings

## High-Value User Findings

| Finding | Possible Follow-Up |
|---|---|
| User in `Remote Desktop Users` | Try RDP if credentials are known. |
| User in `Remote Management Users` | Try WinRM if credentials are known. |
| User has `SeImpersonatePrivilege` | Test Potato-style escalation. |
| User has `SeBackupPrivilege` | Attempt protected file/registry hive reads. |
| Local admin-like accounts exist | Hunt credentials for those users. |
| Helpdesk/custom support groups | Check delegated permissions. |
| Backup users/groups | Investigate backup tools and file access. |

---

## High-Value Host Findings

| Finding | Possible Follow-Up |
|---|---|
| Legacy OS or missing patches | Check local exploit compatibility. |
| x64 OS with x86 shell | Consider architecture before running exploits. |
| Dual-homed host | Investigate pivoting opportunities. |
| Active RDP session | Check logged-on users after privilege escalation. |
| Web/database ports open | Review application configs. |
| KeePass installed | Search for `.kdbx` files. |
| FileZilla installed | Search for saved FTP credentials. |
| XAMPP installed/running | Inspect web roots and database configs. |
| AppLocker enabled | Identify allowed execution paths. |
| Defender/EDR active | Avoid noisy public tooling. |

---

# Common Next Steps After Situational Awareness

Depending on findings, continue into:

- [[Windows Credential Hunting]]
- [[Windows Service Enumeration]]
- [[Windows Privilege Escalation – Installed Applications]]
- [[Windows Privilege Escalation – Missing Patches]]
- [[Windows Privilege Escalation – Scheduled Tasks]]
- [[Windows Privilege Escalation – AlwaysInstallElevated]]
- [[Windows Privilege Escalation – JuicyPotato]]
- [[Windows Privilege Escalation – AppLocker Bypass]]
- [[Windows Post-Exploitation – Pillaging]]
- [[Active Directory Enumeration]]
- [[Lateral Movement]]

---

# Troubleshooting

## PowerShell is blocked

Try CMD equivalents:

```cmd
whoami /groups
net user
net localgroup
ipconfig /all
route print
netstat -ano
tasklist
sc query
wmic qfe
```

---

## Commands are blocked by AppLocker

Check policy:

```powershell
Get-AppLockerPolicy -Effective | Select-Object -ExpandProperty RuleCollections
```

Identify allowed paths and signed binaries.

---

## `Get-LocalUser` is unavailable

Use legacy commands:

```cmd
net user
```

```cmd
net localgroup
```

---

## `wmic` is unavailable

Use PowerShell alternatives:

```powershell
Get-HotFix
```

```powershell
Get-CimInstance Win32_OperatingSystem
```

```powershell
Get-CimInstance Win32_Service
```

---

## AV blocks tools

Use built-in commands first.

Delay tool execution until you understand:

- Defender status
- AppLocker policy
- EDR processes
- allowed execution paths
- rules of engagement

---

# Key Takeaways

- Situational awareness should come before exploitation.
- Start with identity, privileges, groups, OS, and network context.
- Local users and groups reveal privilege targets and access paths.
- Network routes, ARP cache, and active connections reveal reachable systems and possible pivots.
- Installed applications and running processes reveal credential stores, services, and attack surfaces.
- Defender, EDR, AppLocker, and PowerShell restrictions shape what tools and techniques are safe or possible.
- Convert raw enumeration into an actionable plan before attempting privilege escalation.
- Manual commands are essential when automated tools are blocked or too noisy.

---

# Related Notes

- [[Windows Privilege Escalation]]
- [[Windows Enumeration]]
- [[Windows Credential Hunting]]
- [[Windows Services]]
- [[Windows Networking]]
- [[AppLocker]]
- [[Windows Defender]]
- [[EDR]]
- [[PowerShell]]
- [[Local Users and Groups]]
- [[SeImpersonatePrivilege]]
- [[Windows Post-Exploitation]]
- [[Active Directory Enumeration]]

---

# Tags

#windows
#privilege-escalation
#situational-awareness
#enumeration
#network-enumeration
#users-and-groups
#installed-applications
#running-processes
#services
#defender
#edr
#applocker
#powershell
#pentesting
#oscp