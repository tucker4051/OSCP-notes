## Overview

Situational awareness means understanding the system and environment we have landed in before deciding what to do next.

When we gain access to a Windows host, we should avoid jumping straight into exploitation. First, we need to orient ourselves by identifying:

- Where the host sits on the network
- What interfaces and routes it has
- Whether it is joined to a domain
- What other systems it has communicated with
- What security protections are running
- Whether application control is in place
- What tools or commands may be blocked

**Key idea:** Situational awareness helps us make informed decisions. Instead of reacting blindly, we use system and network context to plan privilege escalation, lateral movement, and tool execution more effectively.

---

## Why Situational Awareness Matters

When landing on a Windows or Linux host during a penetration test, there are several things we should always check before attempting privilege escalation.

We may discover:

- Other reachable hosts
- Multiple network interfaces
- Additional internal networks
- Domain controller information
- Recently contacted systems
- Routes into restricted network segments
- Antivirus or EDR products
- Application whitelisting controls
- Tooling restrictions
- Commands that are likely to trigger alerts

This information may directly support local privilege escalation, or it may reveal a better route through the environment.

For example, a host may be **dual-homed**, meaning it has access to two networks. Compromising that host could provide a route into an internal subnet that was not reachable from our original attack box.

---

# Network Information

## Why Gather Network Information?

Gathering network information is a crucial part of enumeration.

It can help us identify:

- The host's IP addresses
- Network interfaces
- DNS servers
- Default gateways
- Domain-related DNS suffixes
- Other local subnets
- Recently contacted hosts
- Routing paths
- Possible lateral movement targets

Network information can support both:

1. **Privilege escalation**, by identifying local context and restrictions.
2. **Lateral movement**, by identifying reachable systems and network paths.

---

## Dual-Homed Hosts

A **dual-homed host** is a system connected to two or more networks.

This usually means the host has multiple physical or virtual network interfaces.

A dual-homed host can act like a bridge between network segments. If we compromise it, we may be able to access another part of the network that was previously unreachable.

Useful indicators include:

- Multiple Ethernet adapters
- Multiple IPv4 addresses
- Multiple default gateways
- Routes to different internal ranges
- DNS suffixes linked to internal domains

---

## Interface, IP Address, and DNS Information

Use `ipconfig /all` to gather detailed network interface information.

```cmd
ipconfig /all
```

Example output:

```cmd
C:\htb> ipconfig /all

Windows IP Configuration

   Host Name . . . . . . . . . . . . : WINLPE-SRV01
   Primary Dns Suffix  . . . . . . . :
   Node Type . . . . . . . . . . . . : Hybrid
   IP Routing Enabled. . . . . . . . : No
   WINS Proxy Enabled. . . . . . . . : No
   DNS Suffix Search List. . . . . . : .htb

Ethernet adapter Ethernet1:

   Connection-specific DNS Suffix  . :
   Description . . . . . . . . . . . : vmxnet3 Ethernet Adapter
   Physical Address. . . . . . . . . : 00-50-56-B9-C5-4B
   DHCP Enabled. . . . . . . . . . . : No
   Autoconfiguration Enabled . . . . : Yes
   Link-local IPv6 Address . . . . . : fe80::f055:fefd:b1b:9919%9(Preferred)
   IPv4 Address. . . . . . . . . . . : 192.168.20.56(Preferred)
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.20.1
   DNS Servers . . . . . . . . . . . : 8.8.8.8
   NetBIOS over Tcpip. . . . . . . . : Enabled

Ethernet adapter Ethernet0:

   Connection-specific DNS Suffix  . : .htb
   Description . . . . . . . . . . . : Intel(R) 82574L Gigabit Network Connection
   Physical Address. . . . . . . . . : 00-50-56-B9-90-94
   DHCP Enabled. . . . . . . . . . . : Yes
   Autoconfiguration Enabled . . . . : Yes
   IPv6 Address. . . . . . . . . . . : dead:beef::e4db:5ea3:2775:8d4d(Preferred)
   Link-local IPv6 Address . . . . . : fe80::e4db:5ea3:2775:8d4d%4(Preferred)
   IPv4 Address. . . . . . . . . . . : 10.129.43.8(Preferred)
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : fe80::250:56ff:feb9:4ddf%4
                                       10.129.0.1
   DHCP Server . . . . . . . . . . . : 10.129.0.1
   DNS Servers . . . . . . . . . . . : 1.1.1.1
                                       8.8.8.8
   NetBIOS over Tcpip. . . . . . . . : Enabled
```

---

## What to Look For in `ipconfig /all`

| Field | Why It Matters |
|---|---|
| Host Name | Identifies the machine name. May reveal role or naming convention. |
| DNS Suffix | May indicate domain membership or internal DNS namespace. |
| IPv4 Address | Shows the current network range. |
| Subnet Mask | Helps determine network size. |
| Default Gateway | Shows the router used to reach other networks. |
| DNS Servers | May reveal domain controllers or internal DNS infrastructure. |
| Multiple Adapters | May indicate a dual-homed host. |
| DHCP Server | Can reveal infrastructure systems. |
| NetBIOS over TCP/IP | May support legacy name resolution or enumeration. |

**Practical tip:** Pay close attention to DNS servers. In Active Directory environments, DNS servers are often domain controllers.

---

# ARP Table

## Why Check the ARP Cache?

The ARP cache shows IP addresses that the host has recently communicated with on the local network.

This can help identify:

- Nearby hosts
- Gateways
- Recently contacted systems
- Potential lateral movement targets
- Systems administrators may have connected to
- RDP or WinRM targets used from this host

Use the following command:

```cmd
arp -a
```

Example output:

```cmd
C:\htb> arp -a

Interface: 10.129.43.8 --- 0x4
  Internet Address      Physical Address      Type
  10.129.0.1            00-50-56-b9-4d-df     dynamic
  10.129.43.12          00-50-56-b9-da-ad     dynamic
  10.129.43.13          00-50-56-b9-5b-9f     dynamic
  10.129.255.255        ff-ff-ff-ff-ff-ff     static
  224.0.0.22            01-00-5e-00-00-16     static
  224.0.0.252           01-00-5e-00-00-fc     static
  224.0.0.253           01-00-5e-00-00-fd     static
  239.255.255.250       01-00-5e-7f-ff-fa     static
  255.255.255.255       ff-ff-ff-ff-ff-ff     static

Interface: 192.168.20.56 --- 0x9
  Internet Address      Physical Address      Type
  192.168.20.255        ff-ff-ff-ff-ff-ff     static
  224.0.0.22            01-00-5e-00-00-16     static
  224.0.0.252           01-00-5e-00-00-fc     static
  239.255.255.250       01-00-5e-7f-ff-fa     static
  255.255.255.255       ff-ff-ff-ff-ff-ff     static
```

---

## What to Look For in the ARP Table

| Item | Meaning |
|---|---|
| Dynamic entries | Hosts recently contacted by the system. |
| Gateway IP | Usually the local router or default gateway. |
| Multiple interfaces | May indicate the host sits across multiple networks. |
| Unknown internal IPs | Potential lateral movement targets. |
| Repeated communication targets | May indicate important servers or admin activity. |

The ARP cache is useful because it shows systems the host has already interacted with. This can reveal targets that may not have appeared in earlier external scanning.

---

# Routing Table

## Why Check the Routing Table?

The routing table shows how the host sends traffic to different networks.

It can reveal:

- Connected networks
- Default routes
- Internal subnets
- Persistent routes
- Preferred interfaces
- IPv4 and IPv6 routing paths
- Whether the host can reach networks we cannot directly access

Use the following command:

```cmd
route print
```

Example output:

```cmd
C:\htb> route print

===========================================================================
Interface List
  9...00 50 56 b9 c5 4b ......vmxnet3 Ethernet Adapter
  4...00 50 56 b9 90 94 ......Intel(R) 82574L Gigabit Network Connection
  1...........................Software Loopback Interface 1
  3...00 00 00 00 00 00 00 e0 Microsoft ISATAP Adapter
  5...00 00 00 00 00 00 00 e0 Teredo Tunneling Pseudo-Interface
 13...00 00 00 00 00 00 00 e0 Microsoft ISATAP Adapter #2
===========================================================================

IPv4 Route Table
===========================================================================
Active Routes:
Network Destination        Netmask          Gateway       Interface  Metric
          0.0.0.0          0.0.0.0       10.129.0.1      10.129.43.8     25
          0.0.0.0          0.0.0.0     192.168.20.1    192.168.20.56    271
       10.129.0.0      255.255.0.0         On-link       10.129.43.8    281
      10.129.43.8  255.255.255.255         On-link       10.129.43.8    281
   10.129.255.255  255.255.255.255         On-link       10.129.43.8    281
        127.0.0.0        255.0.0.0         On-link         127.0.0.1    331
        127.0.0.1  255.255.255.255         On-link         127.0.0.1    331
  127.255.255.255  255.255.255.255         On-link         127.0.0.1    331
     192.168.20.0    255.255.255.0         On-link     192.168.20.56    271
    192.168.20.56  255.255.255.255         On-link     192.168.20.56    271
   192.168.20.255  255.255.255.255         On-link     192.168.20.56    271
        224.0.0.0        240.0.0.0         On-link         127.0.0.1    331
        224.0.0.0        240.0.0.0         On-link       10.129.43.8    281
        224.0.0.0        240.0.0.0         On-link     192.168.20.56    271
  255.255.255.255  255.255.255.255         On-link         127.0.0.1    331
  255.255.255.255  255.255.255.255         On-link       10.129.43.8    281
  255.255.255.255  255.255.255.255         On-link     192.168.20.56    271
===========================================================================

Persistent Routes:
  Network Address          Netmask  Gateway Address  Metric
          0.0.0.0          0.0.0.0     192.168.20.1  Default
===========================================================================

IPv6 Route Table
===========================================================================
Active Routes:
 If Metric Network Destination      Gateway
  4    281 ::/0                     fe80::250:56ff:feb9:4ddf
  1    331 ::1/128                  On-link
  4    281 dead:beef::/64           On-link
  4    281 dead:beef::e4db:5ea3:2775:8d4d/128
                                    On-link
  4    281 fe80::/64                On-link
  9    271 fe80::/64                On-link
  4    281 fe80::e4db:5ea3:2775:8d4d/128
                                    On-link
  9    271 fe80::f055:fefd:b1b:9919/128
                                    On-link
  1    331 ff00::/8                 On-link
  4    281 ff00::/8                 On-link
  9    271 ff00::/8                 On-link
===========================================================================

Persistent Routes:
  None
```

---

## What to Look For in the Routing Table

| Item | Why It Matters |
|---|---|
| `0.0.0.0/0` | Default route used when no specific route exists. |
| Multiple default routes | May indicate multiple network paths. |
| On-link networks | Networks directly reachable from the host. |
| Persistent routes | Manually configured routes that survive reboot. |
| Interface metrics | Lower metric usually means preferred route. |
| Internal ranges | May reveal additional reachable subnets. |
| IPv6 routes | IPv6 may expose paths missed during IPv4-only enumeration. |

Routing information may not directly give us local admin rights, but it can reveal additional attack paths. A host with access to another subnet may become more valuable after privilege escalation.

---

# Enumerating Protections

## Why Enumerate Security Protections?

Most modern Windows environments use security tools to monitor, alert on, and block suspicious activity.

These may include:

- Antivirus
- Endpoint Detection and Response
- Application whitelisting
- Script blocking
- PowerShell logging
- Attack Surface Reduction rules
- Device control
- Controlled folder access

These protections can interfere with enumeration and exploitation.

Public privilege escalation tools and proof-of-concept exploits are commonly detected by antivirus and EDR products. Before running tools, it is important to understand what protections are active.

---

## Antivirus and EDR

Antivirus and EDR products may detect:

- Known public tools
- Credential dumping behaviour
- Suspicious PowerShell usage
- Abnormal process chains
- Common living-off-the-land binary abuse
- Exploit behaviour
- Encoded commands
- Reverse shells
- Unusual parent-child process relationships

Some EDR tools may even detect or block common administrative commands such as:

```cmd
net.exe
```

```cmd
tasklist.exe
```

```cmd
whoami.exe
```

```cmd
powershell.exe
```

```cmd
cmd.exe
```

Context matters. For example, an accountant's workstation launching unusual binaries through `cmd.exe` may be treated as suspicious, even if the same activity would be normal on an administrator's jump box.

---

## Check Windows Defender Status

Use PowerShell to check Windows Defender status:

```powershell
Get-MpComputerStatus
```

Example output:

```powershell
PS C:\htb> Get-MpComputerStatus

AMEngineVersion                 : 1.1.17900.7
AMProductVersion                : 4.10.14393.2248
AMServiceEnabled                : True
AMServiceVersion                : 4.10.14393.2248
AntispywareEnabled              : True
AntispywareSignatureAge         : 1
AntispywareSignatureLastUpdated : 3/28/2021 2:59:13 AM
AntispywareSignatureVersion     : 1.333.1470.0
AntivirusEnabled                : True
AntivirusSignatureAge           : 1
AntivirusSignatureLastUpdated   : 3/28/2021 2:59:12 AM
AntivirusSignatureVersion       : 1.333.1470.0
BehaviorMonitorEnabled          : False
ComputerID                      : 54AF7DE4-3C7E-4DA0-87AC-831B045B9063
ComputerState                   : 0
FullScanAge                     : 4294967295
FullScanEndTime                 :
FullScanStartTime               :
IoavProtectionEnabled           : False
LastFullScanSource              : 0
LastQuickScanSource             : 0
NISEnabled                      : False
NISEngineVersion                : 0.0.0.0
NISSignatureAge                 : 4294967295
NISSignatureLastUpdated         :
NISSignatureVersion             : 0.0.0.0
OnAccessProtectionEnabled       : False
QuickScanAge                    : 4294967295
QuickScanEndTime                :
QuickScanStartTime              :
RealTimeProtectionEnabled       : False
RealTimeScanDirection           : 0
PSComputerName                  :
```

---

## Useful Windows Defender Fields

| Field | Meaning |
|---|---|
| `AMServiceEnabled` | Defender antimalware service is enabled. |
| `AntivirusEnabled` | Antivirus functionality is enabled. |
| `AntispywareEnabled` | Antispyware functionality is enabled. |
| `RealTimeProtectionEnabled` | Real-time protection status. |
| `BehaviorMonitorEnabled` | Behaviour monitoring status. |
| `IoavProtectionEnabled` | Scans files downloaded from the internet or opened from attachments. |
| `OnAccessProtectionEnabled` | Indicates whether files are scanned when accessed. |
| `AntivirusSignatureLastUpdated` | Shows how recent the AV signatures are. |

**Practical tip:** Do not only check whether Defender is installed. Check whether real-time protection, behaviour monitoring, and on-access protection are actually enabled.

---

# Application Whitelisting

## What Is Application Whitelisting?

Application whitelisting controls which applications, scripts, installers, or files users are allowed to run.

In Windows environments, this may be implemented using:

- AppLocker
- Windows Defender Application Control
- Third-party application control products

Application whitelisting may block non-admin users from running tools such as:

```cmd
cmd.exe
```

```powershell
powershell.exe
```

```cmd
net.exe
```

It may also restrict specific file types, such as:

- `.exe`
- `.ps1`
- `.bat`
- `.cmd`
- `.vbs`
- `.js`
- `.msi`
- `.dll`

If application control is enforced, our usual enumeration or exploitation tools may not run. We may need to find allowed paths, allowed binaries, or alternative execution methods.

---

## AppLocker

AppLocker is a Microsoft application control feature that can restrict which users or groups are allowed to run specific files.

AppLocker can control:

- Executables
- Scripts
- Windows Installer files
- DLLs
- Packaged apps

Rules may be based on:

- File path
- File hash
- Publisher signature

---

## List Effective AppLocker Rules

Use the following PowerShell command to view effective AppLocker rules:

```powershell
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
```

Example output:

```powershell
PS C:\htb> Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections

PublisherConditions : {*\*\*,0.0.0.0-*}
PublisherExceptions : {}
PathExceptions      : {}
HashExceptions      : {}
Id                  : a9e18c21-ff8f-43cf-b9fc-db40eed693ba
Name                : (Default Rule) All signed packaged apps
Description         : Allows members of the Everyone group to run packaged apps that are signed.
UserOrGroupSid      : S-1-1-0
Action              : Allow

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

PublisherConditions : {*\*\*,0.0.0.0-*}
PublisherExceptions : {}
PathExceptions      : {}
HashExceptions      : {}
Id                  : b7af7102-efde-4369-8a89-7a6a392d1473
Name                : (Default Rule) All digitally signed Windows Installer files
Description         : Allows members of the Everyone group to run digitally signed Windows Installer files.
UserOrGroupSid      : S-1-1-0
Action              : Allow

PathConditions      : {%WINDIR%\Installer\*}
PathExceptions      : {}
PublisherExceptions : {}
HashExceptions      : {}
Id                  : 5b290184-345a-4453-b184-45305f6d9a54
Name                : (Default Rule) All Windows Installer files in %systemdrive%\Windows\Installer
Description         : Allows members of the Everyone group to run all Windows Installer files located in %systemdrive%\Windows\Installer.
UserOrGroupSid      : S-1-1-0
Action              : Allow

PathConditions      : {*.*}
PathExceptions      : {}
PublisherExceptions : {}
HashExceptions      : {}
Id                  : 64ad46ff-0d71-4fa0-a30b-3f3d30c5433d
Name                : (Default Rule) All Windows Installer files
Description         : Allows members of the local Administrators group to run all Windows Installer files.
UserOrGroupSid      : S-1-5-32-544
Action              : Allow

PathConditions      : {%PROGRAMFILES%\*}
PathExceptions      : {}
PublisherExceptions : {}
HashExceptions      : {}
Id                  : 06dce67b-934c-454f-a263-2515c8796a5d
Name                : (Default Rule) All scripts located in the Program Files folder
Description         : Allows members of the Everyone group to run scripts that are located in the Program Files folder.
UserOrGroupSid      : S-1-1-0
Action              : Allow

PathConditions      : {%WINDIR%\*}
PathExceptions      : {}
PublisherExceptions : {}
HashExceptions      : {}
Id                  : 9428c672-5fc3-47f4-808a-a0011f36dd2c
Name                : (Default Rule) All scripts located in the Windows folder
Description         : Allows members of the Everyone group to run scripts that are located in the Windows folder.
UserOrGroupSid      : S-1-1-0
Action              : Allow

PathConditions      : {*}
PathExceptions      : {}
PublisherExceptions : {}
HashExceptions      : {}
Id                  : ed97d0cb-15ff-430f-b82c-8d7832957725
Name                : (Default Rule) All scripts
Description         : Allows members of the local Administrators group to run all scripts.
UserOrGroupSid      : S-1-5-32-544
Action              : Allow
```

---

## What to Look For in AppLocker Rules

| Item | Why It Matters |
|---|---|
| `Action` | Shows whether the rule allows or denies execution. |
| `UserOrGroupSid` | Shows who the rule applies to. |
| `PathConditions` | Shows allowed or blocked file paths. |
| `PublisherConditions` | Shows whether signed binaries are allowed. |
| `HashExceptions` | Shows specific file hashes excluded from rules. |
| `PathExceptions` | Shows paths excluded from a rule. |
| Script rules | May affect PowerShell, batch, VBS, or JavaScript execution. |
| Executable rules | May affect `.exe` payloads and tools. |
| MSI rules | May affect installer-based techniques. |

Default AppLocker rules often allow execution from:

- `%WINDIR%\*`
- `%PROGRAMFILES%\*`

They may block execution from user-writable directories such as:

- `Downloads`
- `Desktop`
- `Temp`
- `AppData`
- `C:\Users\Public`

---

## Test AppLocker Policy

We can test whether a specific file would be allowed or denied.

```powershell
Get-AppLockerPolicy -Local | Test-AppLockerPolicy -Path C:\Windows\System32\cmd.exe -User Everyone
```

Example output:

```powershell
PS C:\htb> Get-AppLockerPolicy -Local | Test-AppLockerPolicy -Path C:\Windows\System32\cmd.exe -User Everyone

FilePath                    PolicyDecision MatchingRule
--------                    -------------- ------------
C:\Windows\System32\cmd.exe         Denied c:\windows\system32\cmd.exe
```

In this example, `cmd.exe` is denied for the `Everyone` group.

If `cmd.exe` or `powershell.exe` is denied, this can significantly affect enumeration and exploitation. Alternative methods may be required, depending on the rules of engagement.

---

# Practical Situational Awareness Workflow

A practical first-pass workflow after gaining access to a Windows host:

---

## 1. Identify Basic Host Information

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

```cmd
systeminfo
```

---

## 2. Gather Network Context

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

## 3. Check Domain Context

```cmd
echo %USERDOMAIN%
```

```cmd
echo %LOGONSERVER%
```

```cmd
net config workstation
```

```cmd
nltest /dsgetdc:
```

---

## 4. Check Security Protections

```powershell
Get-MpComputerStatus
```

```cmd
sc query windefend
```

```cmd
tasklist
```

```cmd
wmic /namespace:\\root\SecurityCenter2 path AntiVirusProduct get displayName,pathToSignedProductExe
```

---

## 5. Check Application Control

```powershell
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
```

```powershell
Get-AppLockerPolicy -Local | Test-AppLockerPolicy -Path C:\Windows\System32\cmd.exe -User Everyone
```

---

# How This Supports Privilege Escalation

Situational awareness can help identify privilege escalation opportunities by revealing:

- Missing patches from OS version information
- Weak host placement in the network
- Interesting network interfaces
- Reachable internal services
- Security controls that must be avoided
- Allowed execution paths
- Blocked tools or binaries
- Whether public tooling is likely to be detected

Example:

If AppLocker blocks execution from `C:\Users\Public\Downloads` but allows execution from `%WINDIR%`, we may need to look for writable subdirectories inside allowed paths or use trusted binaries already present on the system.

---

# Key Takeaways

- Situational awareness should be performed before attempting privilege escalation.
- Network information can reveal additional reachable networks and lateral movement paths.
- The ARP cache can identify systems the host has recently communicated with.
- The routing table can reveal dual-homed systems and hidden network paths.
- Defender, AV, EDR, and AppLocker may interfere with tooling.
- AppLocker rules can restrict common binaries, scripts, and installers.
- Understanding protections early helps avoid wasted time and unnecessary detections.
- Manual checks are essential when tools cannot run or are likely to be blocked.

---

# Quick Reference

| Area | Command |
|---|---|
| Interface and DNS info | `ipconfig /all` |
| ARP cache | `arp -a` |
| Routing table | `route print` |
| Current user | `whoami` |
| User privileges | `whoami /priv` |
| User groups | `whoami /groups` |
| Host information | `systeminfo` |
| Defender status | `Get-MpComputerStatus` |
| AppLocker rules | `Get-AppLockerPolicy -Effective \| select -ExpandProperty RuleCollections` |
| Test AppLocker path | `Get-AppLockerPolicy -Local \| Test-AppLockerPolicy -Path <path> -User Everyone` |

---

## Summary

Situational awareness is about understanding the host, network, and defensive controls before taking action.

Good enumeration helps us choose the right privilege escalation path, avoid blocked techniques, and identify wider opportunities for lateral movement.

---

## Tags

#Windows #PrivilegeEscalation #WindowsPrivilegeEscalation #SituationalAwareness #Enumeration #NetworkEnumeration #ARP #RoutingTable #AppLocker #WindowsDefender #EDR #AV #HTB #Pentesting #OSCP