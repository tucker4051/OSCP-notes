# Windows Privilege Escalation — Initial Enumeration

## Overview

During an assessment, we may gain a low-privileged shell on a Windows host and need to escalate privileges to further our access.

This host may be:

- Domain-joined
- Standalone
- A workstation
- A server
- A jump box
- A service host
- A system with access to sensitive files or internal network segments

Fully compromising the host may allow us to:

- Access sensitive files
- Access file shares
- Capture traffic
- Obtain additional credentials
- Move laterally
- Escalate privileges within Active Directory
- Potentially reach Domain Admin if privileged credentials or sessions are exposed

---

## Possible Privilege Escalation Targets

Depending on the system configuration and data encountered, we may be able to escalate to one of several useful security contexts.

| Target | Description |
|---|---|
| `NT AUTHORITY\SYSTEM` | Highly privileged local account used by many Windows services. It has more privileges than a local administrator. |
| Built-in local `Administrator` | Default local administrator account. Some organisations disable it, but many do not. It may also be reused across multiple systems. |
| Another local administrator account | Any local user that is a member of the local `Administrators` group has administrator-level privileges. |
| Domain user in local `Administrators` | A standard domain account may still have local admin rights on the host. |
| Domain Admin | A highly privileged Active Directory account. If a Domain Admin is part of the local `Administrators` group or has an active session, this may provide a path to full domain compromise. |

---

## Why Enumeration Matters

Enumeration is the key to privilege escalation.

After gaining initial access, we should collect information about:

- Operating system version
- Patch level
- Installed software
- Running services
- Running processes
- Current user context
- User privileges
- Group memberships
- Local users
- Local groups
- Password policy
- Logged-in users
- Network services
- Potentially vulnerable applications

Automated tools can help, but manual enumeration is essential when:

- Tools cannot be uploaded
- Internet access is unavailable
- PowerShell is restricted
- Antivirus or EDR blocks tools
- Application control is enforced
- We need to verify tool findings manually

---

## Key Data Points

| Data Point | Why It Matters |
|---|---|
| OS name | Identifies whether the target is a workstation or server and helps determine available tools and likely attack paths. |
| OS version | May reveal publicly known privilege escalation vulnerabilities. |
| Patch level | Missing updates may indicate exploitable CVEs. |
| Running services | Misconfigured or vulnerable services running as privileged accounts can provide escalation paths. |
| Installed software | Third-party applications may contain known vulnerabilities or stored credentials. |
| User privileges | Privileges such as `SeImpersonatePrivilege` can often be abused. |
| Group memberships | Local or domain group membership may grant useful access. |
| Logged-in users | Other users may have active sessions, files, or credentials on the host. |
| Network connections | Listening ports may reveal locally accessible services. |

---

# System Information

## Running Processes and Services

The `tasklist /svc` command shows running processes and the services associated with them.

```cmd
tasklist /svc
```

Example output:

```cmd
C:\htb> tasklist /svc

Image Name                     PID Services
========================= ======== ============================================
System Idle Process              0 N/A
System                           4 N/A
smss.exe                       316 N/A
csrss.exe                      424 N/A
wininit.exe                    528 N/A
csrss.exe                      540 N/A
winlogon.exe                   612 N/A
services.exe                   664 N/A
lsass.exe                      672 KeyIso, SamSs, VaultSvc
svchost.exe                    776 BrokerInfrastructure, DcomLaunch, LSM,
                                   PlugPlay, Power, SystemEventsBroker
svchost.exe                    836 RpcEptMapper, RpcSs
LogonUI.exe                    952 N/A
dwm.exe                        964 N/A
svchost.exe                    972 TermService
svchost.exe                   1008 Dhcp, EventLog, lmhosts, TimeBrokerSvc
svchost.exe                    364 NcbService, PcaSvc, ScDeviceEnum, TrkWks,
                                   UALSVC, UmRdpService
svchost.exe                   1468 Wcmsvc
svchost.exe                   1804 PolicyAgent
spoolsv.exe                   1884 Spooler
svchost.exe                   1988 W3SVC, WAS
svchost.exe                   1996 ftpsvc
svchost.exe                   2004 AppHostSvc
FileZilla Server.exe          1140 FileZilla Server
inetinfo.exe                  1164 IISADMIN
svchost.exe                   1736 DiagTrack
svchost.exe                   2084 StateRepository, tiledatamodelsvc
VGAuthService.exe             2100 VGAuthService
vmtoolsd.exe                  2112 VMTools
MsMpEng.exe                   2136 WinDefend
FileZilla Server Interfac     5628 N/A
jusched.exe                   5796 N/A
cmd.exe                       4132 N/A
conhost.exe                   4136 N/A
TrustedInstaller.exe          1120 TrustedInstaller
TiWorker.exe                  1816 N/A
WmiApSrv.exe                  2428 wmiApSrv
tasklist.exe                  3596 N/A
```

---

## Why `tasklist /svc` Is Useful

This command helps identify:

- Standard Windows processes
- Third-party services
- Security tools
- Web servers
- FTP servers
- Database services
- Services running under privileged contexts
- Unusual or non-standard processes

It is important to become familiar with standard Windows processes so unusual processes stand out more quickly.

---

## Common Windows Processes

| Process | Description |
|---|---|
| `smss.exe` | Session Manager Subsystem. Handles session creation. |
| `csrss.exe` | Client Server Runtime Subsystem. Core Windows process. |
| `winlogon.exe` | Handles secure user logon and logoff. |
| `lsass.exe` | Local Security Authority Subsystem Service. Handles authentication and security policy. |
| `services.exe` | Service Control Manager. Starts and manages Windows services. |
| `svchost.exe` | Generic host process for Windows services. |
| `spoolsv.exe` | Print Spooler service. Historically relevant to several privilege escalation paths. |
| `MsMpEng.exe` | Microsoft Defender antimalware process. |

---

## Interesting Findings from Process Enumeration

In the example output, the following are interesting:

| Process or Service | Why It Matters |
|---|---|
| `FileZilla Server.exe` | FTP server. May have misconfigurations, anonymous access, stored credentials, or known vulnerabilities. |
| `W3SVC` | IIS web service. May indicate a web application or writable web directory. |
| `ftpsvc` | Microsoft FTP service. Could expose files or credentials. |
| `MsMpEng.exe` | Microsoft Defender. Indicates endpoint protection is present. |
| `spoolsv.exe` | Print Spooler. Historically useful for several escalation and coercion techniques. |

If we find a third-party service such as FileZilla, we should enumerate:

- Version
- Configuration files
- Service account
- File permissions
- Anonymous access
- Stored credentials
- Known vulnerabilities
- Writable directories

---

# Environment Variables

## Display All Environment Variables

The `set` command displays environment variables.

```cmd
set
```

Example output:

```cmd
C:\htb> set

ALLUSERSPROFILE=C:\ProgramData
APPDATA=C:\Users\Administrator\AppData\Roaming
CommonProgramFiles=C:\Program Files\Common Files
CommonProgramFiles(x86)=C:\Program Files (x86)\Common Files
CommonProgramW6432=C:\Program Files\Common Files
COMPUTERNAME=WINLPE-SRV01
ComSpec=C:\Windows\system32\cmd.exe
HOMEDRIVE=C:
HOMEPATH=\Users\Administrator
LOCALAPPDATA=C:\Users\Administrator\AppData\Local
LOGONSERVER=\\WINLPE-SRV01
NUMBER_OF_PROCESSORS=6
OS=Windows_NT
Path=C:\Windows\system32;C:\Windows;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\;C:\Users\Administrator\AppData\Local\Microsoft\WindowsApps;
PATHEXT=.COM;.EXE;.BAT;.CMD;.VBS;.VBE;.JS;.JSE;.WSF;.WSH;.MSC
PROCESSOR_ARCHITECTURE=AMD64
PROCESSOR_IDENTIFIER=AMD64 Family 23 Model 49 Stepping 0, AuthenticAMD
PROCESSOR_LEVEL=23
PROCESSOR_REVISION=3100
ProgramData=C:\ProgramData
ProgramFiles=C:\Program Files
ProgramFiles(x86)=C:\Program Files (x86)
ProgramW6432=C:\Program Files
PROMPT=$P$G
PSModulePath=C:\Program Files\WindowsPowerShell\Modules;C:\Windows\system32\WindowsPowerShell\v1.0\Modules
PUBLIC=C:\Users\Public
SESSIONNAME=Console
SystemDrive=C:
SystemRoot=C:\Windows
TEMP=C:\Users\ADMINI~1\AppData\Local\Temp\1
TMP=C:\Users\ADMINI~1\AppData\Local\Temp\1
USERDOMAIN=WINLPE-SRV01
USERDOMAIN_ROAMINGPROFILE=WINLPE-SRV01
USERNAME=Administrator
USERPROFILE=C:\Users\Administrator
windir=C:\Windows
```

---

## Useful Environment Variables

| Variable | Why It Matters |
|---|---|
| `USERNAME` | Shows the current username. |
| `USERDOMAIN` | Shows the local machine or domain context. |
| `USERPROFILE` | Shows the current user's profile path. |
| `HOMEDRIVE` | May point to a local disk or network share. |
| `HOMEPATH` | Shows the user's home directory path. |
| `LOGONSERVER` | May identify a domain controller in AD environments. |
| `PATH` | Shows directories searched when running commands. |
| `TEMP` / `TMP` | Writable temporary paths. Useful for uploads or execution. |
| `APPDATA` | User-specific application data directory. May contain credentials or configs. |
| `ProgramFiles` | Location of installed applications. |

---

## Why the `PATH` Variable Matters

The `PATH` variable tells Windows where to look for executables when a command is run.

Windows generally checks:

1. The current working directory
2. Directories in the `PATH`, from left to right

This can be important because:

- Custom application paths may reveal installed software.
- Writable directories in `PATH` may enable hijacking attacks.
- Python, Java, or other interpreters in `PATH` may allow additional execution options.
- A custom path placed before `C:\Windows\System32` can be especially dangerous.

Example risk:

If a writable folder appears early in the `PATH`, and a privileged process searches for a DLL or executable by name, it may be possible to place a malicious file in that writable location.

---

## Home Drives and Roaming Profiles

The `HOMEDRIVE` and `HOMEPATH` variables may reveal useful information.

In enterprise environments, home drives may point to file shares. These shares may contain:

- User files
- Scripts
- Configuration files
- Password spreadsheets
- IT documentation
- SSH keys
- Deployment scripts
- Backup files

Roaming profiles may also allow user data to follow the user between systems.

A common startup path is:

```cmd
%USERPROFILE%\AppData\Microsoft\Windows\Start Menu\Programs\Startup
```

Files placed in this directory may execute when the user logs in.

---

# Detailed System Configuration

## View Detailed System Information

Use `systeminfo` to gather detailed host configuration information.

```cmd
systeminfo
```

Example output:

```cmd
C:\htb> systeminfo

Host Name:                 WINLPE-SRV01
OS Name:                   Microsoft Windows Server 2016 Standard
OS Version:                10.0.14393 N/A Build 14393
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User
Registered Organization:
Product ID:                00376-30000-00299-AA303
Original Install Date:     3/24/2021, 3:46:32 PM
System Boot Time:          3/25/2021, 9:24:36 AM
System Manufacturer:       VMware, Inc.
System Model:              VMware7,1
System Type:               x64-based PC
Processor(s):              3 Processor(s) Installed.
                           [01]: AMD64 Family 23 Model 49 Stepping 0 AuthenticAMD ~2994 Mhz
                           [02]: AMD64 Family 23 Model 49 Stepping 0 AuthenticAMD ~2994 Mhz
                           [03]: AMD64 Family 23 Model 49 Stepping 0 AuthenticAMD ~2994 Mhz
BIOS Version:              VMware, Inc. VMW71.00V.16707776.B64.2008070230, 8/7/2020
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume2
System Locale:             en-us;English (United States)
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC-08:00) Pacific Time (US & Canada)
Total Physical Memory:     6,143 MB
Available Physical Memory: 3,474 MB
Virtual Memory: Max Size:  10,371 MB
Virtual Memory: Available: 7,544 MB
Virtual Memory: In Use:    2,827 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    WORKGROUP
Logon Server:              \\WINLPE-SRV01
Hotfix(s):                 3 Hotfix(s) Installed.
                           [01]: KB3199986
                           [02]: KB5001078
                           [03]: KB4103723
Network Card(s):           2 NIC(s) Installed.
                           [01]: Intel(R) 82574L Gigabit Network Connection
                                Connection Name: Ethernet0
                                DHCP Enabled:    Yes
                                DHCP Server:     10.129.0.1
                                IP address(es)
                                [01]: 10.129.43.8
                                [02]: fe80::e4db:5ea3:2775:8d4d
                                [03]: dead:beef::e4db:5ea3:2775:8d4d
                           [02]: vmxnet3 Ethernet Adapter
                                Connection Name: Ethernet1
                                DHCP Enabled:    No
                                IP address(es)
                                [01]: 192.168.20.56
                                [02]: fe80::f055:fefd:b1b:9919
Hyper-V Requirements:      A hypervisor has been detected. Features required for Hyper-V will not be displayed.
```

---

## What to Look For in `systeminfo`

| Field | Why It Matters |
|---|---|
| Host Name | Identifies the machine and may reveal naming conventions. |
| OS Name | Shows workstation or server type. |
| OS Version | Helps identify public exploits for that version. |
| OS Configuration | Shows whether the host is a standalone server, domain controller, or member server. |
| System Boot Time | Long uptime may suggest missing recent patches. |
| System Manufacturer / Model | May reveal virtualisation or hardware type. |
| Domain | Shows domain or workgroup membership. |
| Hotfix(s) | Shows installed patches. |
| Network Card(s) | Helps identify multiple interfaces or dual-homed systems. |
| IP addresses | Reveals reachable network ranges. |

---

## Patch Level and Uptime

Patch level is useful because missing updates may indicate known privilege escalation vulnerabilities.

Useful indicators include:

- Installed hotfixes
- OS version
- Build number
- System boot time
- Last restart date
- Recently installed updates

If a system has not been restarted for a long time, it may also not have been patched recently.

Be careful with kernel or local privilege escalation exploits. These can cause system instability or crashes, especially on production systems.

---

# Patches and Updates

## Query Patches with WMIC

If `systeminfo` does not display hotfixes, they may be queried using WMI.

```cmd
wmic qfe
```

Example output:

```cmd
C:\htb> wmic qfe

Caption                                     CSName        Description      FixComments  HotFixID   InstallDate  InstalledBy          InstalledOn  Name  ServicePackInEffect  Status
http://support.microsoft.com/?kbid=3199986  WINLPE-SRV01  Update                        KB3199986               NT AUTHORITY\SYSTEM  11/21/2016
https://support.microsoft.com/help/5001078  WINLPE-SRV01  Security Update               KB5001078               NT AUTHORITY\SYSTEM  3/25/2021
http://support.microsoft.com/?kbid=4103723  WINLPE-SRV01  Security Update               KB4103723               NT AUTHORITY\SYSTEM  3/25/2021
```

---

## Query Patches with PowerShell

PowerShell can also be used to query installed hotfixes.

```powershell
Get-HotFix | ft -AutoSize
```

Example output:

```powershell
PS C:\htb> Get-HotFix | ft -AutoSize

Source       Description     HotFixID  InstalledBy                InstalledOn
------       -----------     --------  -----------                -----------
WINLPE-SRV01 Update          KB3199986 NT AUTHORITY\SYSTEM        11/21/2016 12:00:00 AM
WINLPE-SRV01 Update          KB4054590 WINLPE-SRV01\Administrator 3/30/2021 12:00:00 AM
WINLPE-SRV01 Security Update KB5001078 NT AUTHORITY\SYSTEM        3/25/2021 12:00:00 AM
WINLPE-SRV01 Security Update KB3200970 WINLPE-SRV01\Administrator 4/13/2021 12:00:00 AM
```

---

## Why Patch Enumeration Matters

Patch enumeration helps identify:

- Missing security updates
- Potential kernel exploits
- Known local privilege escalation vulnerabilities
- Outdated systems
- Recently patched systems
- Systems with inconsistent update history

Useful follow-up actions:

- Search KB numbers
- Compare OS build with known vulnerabilities
- Check exploit reliability
- Confirm exploit applies to the exact OS version
- Avoid unstable exploits on production systems unless explicitly authorised

---

# Installed Programs

## Query Installed Software with WMIC

Installed software may reveal vulnerable applications or tools that store credentials.

```cmd
wmic product get name
```

Example output:

```cmd
C:\htb> wmic product get name

Name
Microsoft Visual C++ 2019 X64 Additional Runtime - 14.24.28127
Java 8 Update 231 (64-bit)
Microsoft Visual C++ 2019 X86 Additional Runtime - 14.24.28127
VMware Tools
Microsoft Visual C++ 2019 X64 Minimum Runtime - 14.24.28127
Microsoft Visual C++ 2019 X86 Minimum Runtime - 14.24.28127
Java Auto Updater
```

---

## Query Installed Software with PowerShell

```powershell
Get-WmiObject -Class Win32_Product | select Name, Version
```

Example output:

```powershell
PS C:\htb> Get-WmiObject -Class Win32_Product | select Name, Version

Name                                                                    Version
----                                                                    -------
SQL Server 2016 Database Engine Shared                                  13.2.5026.0
Microsoft OLE DB Driver for SQL Server                                  18.3.0.0
Microsoft Visual C++ 2010  x64 Redistributable - 10.0.40219             10.0.40219
Microsoft Help Viewer 2.3                                               2.3.28107
Microsoft Visual C++ 2010  x86 Redistributable - 10.0.40219             10.0.40219
Microsoft Visual C++ 2013 x86 Minimum Runtime - 12.0.21005              12.0.21005
Microsoft Visual C++ 2013 x86 Additional Runtime - 12.0.21005           12.0.21005
Microsoft Visual C++ 2019 X64 Additional Runtime - 14.28.29914          14.28.29914
Microsoft ODBC Driver 13 for SQL Server                                 13.2.5026.0
SQL Server 2016 Database Engine Shared                                  13.2.5026.0
SQL Server 2016 Database Engine Services                                13.2.5026.0
SQL Server Management Studio for Reporting Services                     15.0.18369.0
Microsoft SQL Server 2008 Setup Support Files                           10.3.5500.0
SSMS Post Install Tasks                                                 15.0.18369.0
Microsoft VSS Writer for SQL Server 2016                                13.2.5026.0
Java 8 Update 231 (64-bit)                                              8.0.2310.11
Browser for SQL Server 2016                                             13.2.5026.0
Integration Services                                                    15.0.2000.130
```

---

## What to Look For in Installed Software

| Software Type | Why It Matters |
|---|---|
| FTP clients or servers | May store credentials or expose misconfigured services. |
| PuTTY / WinSCP / FileZilla | May contain saved session data or credentials. |
| SQL Server tools | May indicate database services, stored credentials, or service accounts. |
| Java | Older versions may be vulnerable or useful for execution. |
| VMware Tools | Indicates virtualised environment. |
| Backup software | Often runs with high privileges and may expose credentials. |
| Remote access tools | May contain saved sessions or lateral movement paths. |
| Custom applications | May run as services or have weak file permissions. |

---

# Network Services and Connections

## Display Running Network Connections

The `netstat` command displays active TCP and UDP connections.

```cmd
netstat -ano
```

Example output:

```cmd
PS C:\htb> netstat -ano

Active Connections

  Proto  Local Address          Foreign Address        State           PID
  TCP    0.0.0.0:21             0.0.0.0:0              LISTENING       1096
  TCP    0.0.0.0:80             0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:135            0.0.0.0:0              LISTENING       840
  TCP    0.0.0.0:445            0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:1433           0.0.0.0:0              LISTENING       3520
  TCP    0.0.0.0:3389           0.0.0.0:0              LISTENING       968
```

---

## Why `netstat -ano` Is Useful

This command helps identify:

- Listening services
- Local-only services
- Externally reachable services
- Active connections
- Remote hosts communicating with the system
- Process IDs linked to network services

The `-ano` flags mean:

| Flag | Meaning |
|---|---|
| `-a` | Displays all active connections and listening ports. |
| `-n` | Displays addresses and port numbers numerically. |
| `-o` | Shows the owning process ID. |

---

## Interesting Ports in the Example

| Port | Service | Why It Matters |
|---|---|---|
| `21` | FTP | Could indicate FileZilla or Microsoft FTP. Check anonymous access and stored credentials. |
| `80` | HTTP | Web server. Check web roots, IIS configuration, and application files. |
| `135` | RPC | Common Windows service. Useful for service enumeration. |
| `445` | SMB | File sharing. Check shares and permissions. |
| `1433` | MSSQL | SQL Server. Check service account, credentials, and database access. |
| `3389` | RDP | Remote Desktop. Useful for access validation and logged-on user context. |

To map a PID from `netstat` to a process:

```cmd
tasklist /fi "PID eq 1096"
```

---

# User and Group Information

## Why Enumerate Users and Groups?

Users are often the weakest link in an organisation.

Even if a system is patched and well-configured, privilege escalation may still be possible through:

- Weak passwords
- Password reuse
- Browsable user directories
- Stored credentials
- Interesting files
- Local admin group membership
- Over-permissive groups
- Active sessions
- Service accounts
- Misconfigured password policies

For example, a local administrator's user directory may be readable and contain a file such as:

```text
logins.xlsx
```

This could provide an easy path to privilege escalation.

---

## Logged-In Users

It is important to determine which users are logged into the system.

```cmd
query user
```

Example output:

```cmd
C:\htb> query user

 USERNAME              SESSIONNAME        ID  STATE   IDLE TIME  LOGON TIME
>administrator         rdp-tcp#2           1  Active          .  3/25/2021 9:27 AM
```

---

## Why Logged-In Users Matter

Logged-in users may reveal:

- Active administrator sessions
- RDP users
- Idle users
- Potential targets for credential capture
- Users currently working on the system
- Whether extra care is needed during evasive engagements

During an evasive engagement, we should be careful on a host where users are actively working, as noisy activity may lead to detection.

---

## Current User

Always check the user context we are operating under.

```cmd
echo %USERNAME%
```

Example output:

```cmd
C:\htb> echo %USERNAME%

htb-student
```

Also use:

```cmd
whoami
```

Sometimes we may already be running as a privileged user, service account, or even `NT AUTHORITY\SYSTEM`.

---

## Current User Privileges

User privileges can reveal direct privilege escalation paths.

```cmd
whoami /priv
```

Example output:

```cmd
C:\htb> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== ========
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Disabled
```

---

## Interesting Windows Privileges

| Privilege | Why It Matters |
|---|---|
| `SeImpersonatePrivilege` | Often abused for local privilege escalation using impersonation attacks. |
| `SeAssignPrimaryTokenPrivilege` | May support token-based privilege escalation. |
| `SeBackupPrivilege` | Can allow reading sensitive files, including registry hives. |
| `SeRestorePrivilege` | Can allow writing files to protected locations. |
| `SeDebugPrivilege` | Can allow interaction with privileged processes. |
| `SeTakeOwnershipPrivilege` | Can allow taking ownership of files or objects. |
| `SeLoadDriverPrivilege` | May allow loading kernel drivers. |
| `SeManageVolumePrivilege` | Can sometimes be abused for file system attacks. |

Example:

If we gain access as a service account with `SeImpersonatePrivilege`, it may be possible to escalate privileges using impersonation techniques.

---

## Current User Group Information

Check the current user's group memberships.

```cmd
whoami /groups
```

Example output:

```cmd
C:\htb> whoami /groups

GROUP INFORMATION
-----------------

Group Name                             Type             SID          Attributes
====================================== ================ ============ ==================================================
Everyone                               Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Desktop Users           Alias            S-1-5-32-555 Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                          Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\REMOTE INTERACTIVE LOGON  Well-known group S-1-5-14     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\INTERACTIVE               Well-known group S-1-5-4      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users       Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization         Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Local account             Well-known group S-1-5-113    Mandatory group, Enabled by default, Enabled group
LOCAL                                  Well-known group S-1-2-0      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication       Well-known group S-1-5-64-10  Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Mandatory Level Label            S-1-16-8192
```

---

## What to Look For in Group Memberships

| Group | Why It Matters |
|---|---|
| `Administrators` | Full local admin rights. |
| `Remote Desktop Users` | May allow RDP access. |
| `Remote Management Users` | May allow WinRM access. |
| `Backup Operators` | Can often read sensitive files. |
| `Print Operators` | Historically useful in domain environments. |
| `Event Log Readers` | May allow reading logs containing sensitive information. |
| `IIS_IUSRS` | May indicate web server-related access. |
| `Hyper-V Administrators` | May provide control over virtual machines. |
| Custom groups | May have delegated permissions or access to sensitive files. |

---

## Get All Local Users

List local user accounts.

```cmd
net user
```

Example output:

```cmd
C:\htb> net user

User accounts for \\WINLPE-SRV01

-------------------------------------------------------------------------------
Administrator            DefaultAccount           Guest
helpdesk                 htb-student              jordan
sarah                    secsvc
The command completed successfully.
```

---

## Why Local Users Matter

Local users may reveal:

- Administrator accounts
- Service accounts
- Helpdesk accounts
- Disabled or unused accounts
- Naming patterns
- Potential password reuse
- Accounts with interesting profile directories

Example:

If we gained access as `bob` and find a local admin account named `bob_adm`, it may be worth checking for credential reuse.

---

## Get All Local Groups

List local groups.

```cmd
net localgroup
```

Example output:

```cmd
C:\htb> net localgroup

Aliases for \\WINLPE-SRV01

-------------------------------------------------------------------------------
*Access Control Assistance Operators
*Administrators
*Backup Operators
*Certificate Service DCOM Access
*Cryptographic Operators
*Distributed COM Users
*Event Log Readers
*Guests
*Hyper-V Administrators
*IIS_IUSRS
*Network Configuration Operators
*Performance Log Users
*Performance Monitor Users
*Power Users
*Print Operators
*RDS Endpoint Servers
*RDS Management Servers
*RDS Remote Access Servers
*Remote Desktop Users
*Remote Management Users
*Replicator
*Storage Replica Administrators
*System Managed Accounts Group
*Users
The command completed successfully.
```

---

## Why Local Groups Matter

Local groups may reveal:

- Who can administer the host
- Who can access the host remotely
- Who can read event logs
- Who can perform backups
- Whether domain users have excessive rights
- Whether non-standard groups exist
- What role the server may perform

Pay particular attention to:

- `Administrators`
- `Remote Desktop Users`
- `Remote Management Users`
- `Backup Operators`
- `Event Log Readers`
- Any custom or unusual groups

---

## Details About a Specific Group

Check group membership for interesting local groups.

```cmd
net localgroup administrators
```

Example output:

```cmd
C:\htb> net localgroup administrators

Alias name     administrators
Comment        Administrators have complete and unrestricted access to the computer/domain

Members

-------------------------------------------------------------------------------
Administrator
helpdesk
sarah
secsvc
The command completed successfully.
```

---

## What to Look For in Group Details

Group details can reveal:

- Local administrator users
- Domain users with local admin access
- Service accounts with admin rights
- Helpdesk accounts
- Overly broad group membership
- Interesting group descriptions
- Possible credential reuse targets

It is worth checking non-standard groups as well. Sometimes group descriptions or memberships reveal useful information.

---

# Password Policy and Account Information

## Get Password Policy

Use `net accounts` to display password policy and account lockout settings.

```cmd
net accounts
```

Example output:

```cmd
C:\htb> net accounts

Force user logoff how long after time expires?:       Never
Minimum password age (days):                          0
Maximum password age (days):                          42
Minimum password length:                              0
Length of password history maintained:                None
Lockout threshold:                                    Never
Lockout duration (minutes):                           30
Lockout observation window (minutes):                 30
Computer role:                                        SERVER
The command completed successfully.
```

---

## Why Password Policy Matters

Password policy can reveal:

- Weak password requirements
- No lockout threshold
- Password reuse risk
- Brute-force feasibility
- Whether local password attacks may be safer
- Whether account lockouts are enforced
- Whether the host is a workstation or server

In the example above:

| Setting | Risk |
|---|---|
| Minimum password length: `0` | Very weak password policy. |
| Lockout threshold: `Never` | Password guessing may not lock accounts. |
| Password history: `None` | Users may reuse old passwords. |
| Maximum password age: `42` | Passwords may rotate regularly, depending on enforcement. |

---

# Manual Enumeration Workflow

A practical initial enumeration workflow:

## 1. Confirm User Context

```cmd
whoami
```

```cmd
echo %USERNAME%
```

```cmd
whoami /priv
```

```cmd
whoami /groups
```

---

## 2. Gather Host Information

```cmd
hostname
```

```cmd
systeminfo
```

```cmd
set
```

---

## 3. Review Processes and Services

```cmd
tasklist /svc
```

```cmd
net start
```

```cmd
sc query
```

---

## 4. Review Network Services

```cmd
netstat -ano
```

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

## 5. Review Users and Groups

```cmd
query user
```

```cmd
net user
```

```cmd
net localgroup
```

```cmd
net localgroup administrators
```

```cmd
net accounts
```

---

## 6. Review Patches and Software

```cmd
wmic qfe
```

```powershell
Get-HotFix | ft -AutoSize
```

```cmd
wmic product get name
```

```powershell
Get-WmiObject -Class Win32_Product | select Name, Version
```

---

# Common Initial Enumeration Commands

| Purpose | Command |
|---|---|
| Current user | `whoami` |
| Username variable | `echo %USERNAME%` |
| User privileges | `whoami /priv` |
| User groups | `whoami /groups` |
| Hostname | `hostname` |
| System information | `systeminfo` |
| Environment variables | `set` |
| Running processes and services | `tasklist /svc` |
| Network connections | `netstat -ano` |
| Logged-in users | `query user` |
| Local users | `net user` |
| Local groups | `net localgroup` |
| Local administrators | `net localgroup administrators` |
| Password policy | `net accounts` |
| Installed patches | `wmic qfe` |
| Installed patches with PowerShell | `Get-HotFix \| ft -AutoSize` |
| Installed software | `wmic product get name` |
| Installed software with PowerShell | `Get-WmiObject -Class Win32_Product \| select Name, Version` |

---

# Key Takeaways

- Enumeration is the foundation of Windows privilege escalation.
- Always identify the current user context first.
- Check privileges and group memberships before looking for exploits.
- Running services may reveal misconfigurations or vulnerable software.
- Environment variables can expose useful paths, shares, and execution opportunities.
- Patch level and OS version help identify potential CVEs.
- Installed software may expose stored credentials or vulnerable services.
- Logged-in users may provide useful targets or increase detection risk.
- Password policy can reveal weak local account controls.
- Manual enumeration is essential when automated tools are blocked.

---

## Summary

Initial enumeration is about building a clear picture of the host before choosing an escalation path.

The goal is to answer:

- Who am I?
- What privileges do I have?
- What system am I on?
- What software and services are running?
- What patches are installed?
- Who else uses this system?
- What groups and users exist?
- Are there obvious misconfigurations?
- Are there credentials, services, or permissions that can be abused?

Good enumeration prevents wasted effort and helps identify the most realistic privilege escalation route.

---

## Tags

#Windows #PrivilegeEscalation #WindowsPrivilegeEscalation #InitialEnumeration #Enumeration #LocalEnumeration #WindowsCommands #SystemInfo #Tasklist #Netstat #WMIC #PowerShell #UsersAndGroups #HTB #Pentesting #OSCP