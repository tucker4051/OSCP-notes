# Windows Privilege Escalation — Processes, Network Services and Named Pipes

## Overview

One of the best places to look for privilege escalation opportunities is the list of processes and services running on the system.

Even if a process is not running as an administrator, it may still provide a route to additional privileges. A common example is discovering a web server such as IIS, XAMPP, or another application server running on the host.

If we can upload or place a web shell, such as an `aspx` or `php` shell, we may gain code execution as the user running the web server.

That user may not be an administrator, but it may have useful privileges such as:

```text
SeImpersonatePrivilege
```

This privilege can often be abused using techniques or tools such as:

- Juicy Potato
- Rogue Potato
- Lonely Potato
- PrintSpoofer
- GodPotato

The general idea is that a service account may not be an administrator directly, but its token privileges may still allow escalation to `NT AUTHORITY\SYSTEM`.

---

# Access Tokens

## What Are Access Tokens?

In Windows, **access tokens** describe the security context of a process or thread.

An access token contains information about:

- The user account identity
- Group memberships
- Assigned privileges
- Integrity level
- Logon session
- Security attributes

When a user authenticates to a Windows system, their credentials are verified. If authentication succeeds, Windows assigns the user an access token.

Whenever that user interacts with a process, a copy of the token is used to determine what the process is allowed to do.

---

## Why Access Tokens Matter for Privilege Escalation

Access tokens are important because Windows uses them to make access control decisions.

If a process has a token with powerful privileges, that process may be useful for privilege escalation.

Examples of useful privileges include:

| Privilege | Why It Matters |
|---|---|
| `SeImpersonatePrivilege` | Can often be abused to impersonate privileged tokens. |
| `SeAssignPrimaryTokenPrivilege` | Can allow assigning tokens to processes. |
| `SeDebugPrivilege` | Can allow interaction with privileged processes. |
| `SeBackupPrivilege` | Can allow reading sensitive files. |
| `SeRestorePrivilege` | Can allow writing to protected locations. |
| `SeTakeOwnershipPrivilege` | Can allow taking ownership of files or objects. |

---

## Service Accounts and Tokens

Services often run using accounts that have more privileges than a normal user.

Common service contexts include:

| Context | Description |
|---|---|
| `NT AUTHORITY\SYSTEM` | Highly privileged local system account. |
| `NT AUTHORITY\LOCAL SERVICE` | Limited local service account. |
| `NT AUTHORITY\NETWORK SERVICE` | Service account with network authentication capability. |
| Custom domain service account | Domain account used to run services. May have access to other systems. |
| Local administrator account | Local account with admin rights on the host. |

If a web server, database service, or application service runs with a useful token, it may provide an escalation path.

---

# Enumerating Network Services

## Why Enumerate Network Services?

Processes commonly interact through network sockets.

Examples include:

- DNS
- HTTP
- SMB
- FTP
- RDP
- WinRM
- MSSQL
- Local administrative interfaces

The `netstat` command can show active TCP and UDP connections. This helps us identify:

- Listening services
- Local-only services
- Externally reachable services
- Active outbound connections
- Process IDs associated with ports
- Potentially vulnerable services
- Services only reachable from localhost

A local-only service may be especially interesting because developers sometimes assume localhost services are safe and therefore configure them insecurely.

---

## Display Active Network Connections

Use the following command:

```cmd
netstat -ano
```

Example output:

```cmd
C:\htb> netstat -ano

Active Connections

  Proto  Local Address          Foreign Address        State           PID
  TCP    0.0.0.0:21             0.0.0.0:0              LISTENING       3812
  TCP    0.0.0.0:80             0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:135            0.0.0.0:0              LISTENING       836
  TCP    0.0.0.0:445            0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:3389           0.0.0.0:0              LISTENING       936
  TCP    0.0.0.0:5985           0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:8080           0.0.0.0:0              LISTENING       5044
  TCP    0.0.0.0:47001          0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:49664          0.0.0.0:0              LISTENING       528
  TCP    0.0.0.0:49665          0.0.0.0:0              LISTENING       996
  TCP    0.0.0.0:49666          0.0.0.0:0              LISTENING       1260
  TCP    0.0.0.0:49668          0.0.0.0:0              LISTENING       2008
  TCP    0.0.0.0:49669          0.0.0.0:0              LISTENING       600
  TCP    0.0.0.0:49670          0.0.0.0:0              LISTENING       1888
  TCP    0.0.0.0:49674          0.0.0.0:0              LISTENING       616
  TCP    10.129.43.8:139        0.0.0.0:0              LISTENING       4
  TCP    10.129.43.8:3389       10.10.14.3:63191       ESTABLISHED     936
  TCP    10.129.43.8:49671      40.67.251.132:443      ESTABLISHED     1260
  TCP    10.129.43.8:49773      52.37.190.150:443      ESTABLISHED     2608
  TCP    10.129.43.8:51580      40.67.251.132:443      ESTABLISHED     3808
  TCP    10.129.43.8:54267      40.67.254.36:443       ESTABLISHED     3808
  TCP    10.129.43.8:54268      40.67.254.36:443       ESTABLISHED     1260
  TCP    10.129.43.8:54269      64.233.184.189:443     ESTABLISHED     2608
  TCP    10.129.43.8:54273      216.58.210.195:443     ESTABLISHED     2608
  TCP    127.0.0.1:14147        0.0.0.0:0              LISTENING       3812
  TCP    192.168.20.56:139      0.0.0.0:0              LISTENING       4
  TCP    [::]:21                [::]:0                 LISTENING       3812
  TCP    [::]:80                [::]:0                 LISTENING       4
  TCP    [::]:135               [::]:0                 LISTENING       836
  TCP    [::]:445               [::]:0                 LISTENING       4
  TCP    [::]:3389              [::]:0                 LISTENING       936
  TCP    [::]:5985              [::]:0                 LISTENING       4
  TCP    [::]:8080              [::]:0                 LISTENING       5044
  TCP    [::]:47001             [::]:0                 LISTENING       4
  TCP    [::]:49664             [::]:0                 LISTENING       528
  TCP    [::]:49665             [::]:0                 LISTENING       996
  TCP    [::]:49666             [::]:0                 LISTENING       1260
  TCP    [::]:49668             [::]:0                 LISTENING       2008
  TCP    [::]:49669             [::]:0                 LISTENING       600
  TCP    [::]:49670             [::]:0                 LISTENING       1888
  TCP    [::]:49674             [::]:0                 LISTENING       616
  TCP    [::1]:14147            [::]:0                 LISTENING       3812
  UDP    0.0.0.0:123            *:*                                    1104
  UDP    0.0.0.0:500            *:*                                    1260
  UDP    0.0.0.0:3389           *:*                                    936
```

---

## Understanding the `netstat -ano` Flags

| Flag | Meaning |
|---|---|
| `-a` | Displays all active connections and listening ports. |
| `-n` | Displays addresses and port numbers numerically. |
| `-o` | Displays the owning process ID. |

The process ID can then be mapped to a process using:

```cmd
tasklist /fi "PID eq 3812"
```

---

## What to Look For in Active Network Connections

The main things to look for are:

- Services listening on all interfaces
- Services listening only on localhost
- Unusual high ports
- Known administrative interfaces
- Services linked to third-party applications
- Established outbound connections
- Ports associated with privileged services
- Services running under privileged accounts

---

## Localhost Services

Pay special attention to services listening on loopback addresses:

```text
127.0.0.1
```

```text
::1
```

These are only accessible from the local host.

The reason these are interesting is that developers and administrators often assume localhost services are not exposed to the network and therefore do not need strong authentication or secure configuration.

Examples of loopback-only entries:

```text
127.0.0.1:14147
```

```text
[::1]:14147
```

In the example, port `14147` stands out because it is used by the FileZilla administrative interface.

---

## FileZilla Administrative Interface Example

FileZilla Server commonly uses port:

```text
14147
```

If this port is listening only on localhost, it may still be accessible from our shell on the compromised host.

Potential attack opportunities include:

- Connecting to the administrative interface locally
- Extracting FTP credentials
- Modifying FTP shares
- Creating a share pointing to `C:\`
- Abusing the FileZilla service account context

If FileZilla is running as an administrator or privileged service account, this may allow privilege escalation.

---

## Common Interesting Ports

| Port | Service | Why It May Matter |
|---|---|---|
| `21` | FTP | May expose credentials, anonymous access, writable directories, or FileZilla. |
| `80` | HTTP | Web service. May allow web shell upload or application exploitation. |
| `135` | RPC | Windows RPC endpoint mapper. Useful for service enumeration. |
| `139` | NetBIOS | Legacy Windows file sharing and name service. |
| `445` | SMB | File shares, named pipes, remote administration. |
| `3389` | RDP | Remote Desktop access and active session indicators. |
| `5985` | WinRM HTTP | Remote PowerShell access. |
| `5986` | WinRM HTTPS | Remote PowerShell over HTTPS. |
| `8080` | HTTP alternate | Often used by web applications, proxies, or admin panels. |
| `14147` | FileZilla Admin | Local administrative interface for FileZilla Server. |
| `1433` | MSSQL | SQL Server. May expose database access or service account abuse. |
| `25672` | Erlang | Erlang distribution port, often relevant to RabbitMQ and similar services. |

---

# Additional Network Service Examples

## Splunk Universal Forwarder

A useful example of service-based privilege escalation is the **Splunk Universal Forwarder**.

Splunk Universal Forwarder is commonly installed on endpoints to send logs into Splunk.

Historically, some default configurations:

- Did not require authentication
- Allowed application deployment
- Ran as `NT AUTHORITY\SYSTEM`

This could allow code execution as SYSTEM if the forwarder is misconfigured.

Things to check:

- Whether Splunk Universal Forwarder is installed
- Whether its management port is listening
- Whether authentication is required
- Whether apps can be deployed
- What account the service runs as
- Whether the configuration files are writable

---

## Erlang Port

Another local privilege escalation vector is the Erlang distribution port:

```text
25672
```

Erlang is a programming language designed around distributed systems. Erlang nodes can join clusters using a shared secret called a **cookie**.

If the cookie is weak or readable, an attacker may be able to connect to the Erlang node and execute code.

Applications that may use Erlang include:

- RabbitMQ
- CouchDB
- SolarWinds products

RabbitMQ commonly uses the default cookie:

```text
rabbit
```

Things to check:

- Whether port `25672` is listening
- Whether Erlang-based software is installed
- Whether the Erlang cookie is weak
- Whether the cookie file is readable
- What account the Erlang service runs as

---

# Named Pipes

## What Are Named Pipes?

Named pipes are a Windows inter-process communication mechanism.

They allow processes to communicate with each other using shared memory.

A named pipe can be thought of as a temporary file-like communication channel stored in memory. Data written to the pipe can be read by another process.

There are two main types of pipes:

| Pipe Type | Description |
|---|---|
| Named pipe | Has a specific name and can be used for communication between unrelated processes. |
| Anonymous pipe | Usually used between related processes, such as parent and child processes. |

Example named pipe path:

```text
\\.\pipe\ExampleNamedPipeServer
```

---

## How Named Pipes Work

Windows uses a client-server model for named pipe communication.

| Role | Description |
|---|---|
| Pipe server | The process that creates the named pipe. |
| Pipe client | The process that connects to and communicates with the pipe. |

Named pipes can be:

| Mode | Description |
|---|---|
| Half-duplex | One-way communication. Usually the client writes and the server reads. |
| Duplex | Two-way communication. The client and server can both send and receive data. |

Every active connection to a named pipe server creates a new pipe instance. These instances share the same pipe name but use different data buffers.

---

## Why Named Pipes Matter

Named pipes are useful to enumerate because they may expose:

- Inter-process communication channels
- Service control interfaces
- Weak permissions
- Local privilege escalation opportunities
- Red team tooling artefacts
- Security product communication
- Application-specific control channels

If a named pipe allows low-privileged users to write data, and the pipe server runs as a privileged account, it may be possible to influence privileged behaviour.

---

## Cobalt Strike and Named Pipes

Cobalt Strike commonly uses named pipes for command execution and inter-process communication.

A simplified workflow:

1. Beacon starts a named pipe, for example:

```text
\\.\pipe\msagent_12
```

2. Beacon starts a new process and injects the command into that process.

3. The command output is written to the named pipe.

4. Beacon reads the output from the pipe and displays it to the operator.

This approach helps protect the beacon. If the injected command crashes or is flagged by antivirus, it may not directly crash the main beacon process.

---

## Suspicious Named Pipe Names

Cobalt Strike operators often rename named pipes to look like legitimate software.

Common masquerading examples include names related to:

- Chrome
- Edge
- Firefox
- Teams
- Microsoft Office
- Windows services

One common example is:

```text
mojo
```

Chrome commonly uses named pipes containing `mojo`.

If a system has `mojo` named pipes but Chrome is not installed or running, that may be suspicious.

This can indicate:

- Red team activity
- Malware
- C2 framework usage
- Suspicious process injection
- Masquerading behaviour

---

# Listing Named Pipes

## Listing Named Pipes with PipeList

`PipeList` is part of the Sysinternals Suite.

Use it to enumerate open named pipes:

```cmd
pipelist.exe /accepteula
```

Example output:

```cmd
C:\htb> pipelist.exe /accepteula

PipeList v1.02 - Lists open named pipes
Copyright (C) 2005-2016 Mark Russinovich
Sysinternals - www.sysinternals.com

Pipe Name                                    Instances       Max Instances
---------                                    ---------       -------------
InitShutdown                                      3               -1
lsass                                             4               -1
ntsvcs                                            3               -1
scerpc                                            3               -1
Winsock2\CatalogChangeListener-340-0              1                1
Winsock2\CatalogChangeListener-414-0              1                1
epmapper                                          3               -1
Winsock2\CatalogChangeListener-3ec-0              1                1
Winsock2\CatalogChangeListener-44c-0              1                1
LSM_API_service                                   3               -1
atsvc                                             3               -1
Winsock2\CatalogChangeListener-5e0-0              1                1
eventlog                                          3               -1
Winsock2\CatalogChangeListener-6a8-0              1                1
spoolss                                           3               -1
Winsock2\CatalogChangeListener-ec0-0              1                1
wkssvc                                            4               -1
trkwks                                            3               -1
vmware-usbarbpipe                                 5               -1
srvsvc                                            4               -1
ROUTER                                            3               -1
vmware-authdpipe                                  1                1
```

---

## Listing Named Pipes with PowerShell

Named pipes can also be listed with PowerShell using `Get-ChildItem`.

```powershell
gci \\.\pipe\
```

Equivalent full command:

```powershell
Get-ChildItem \\.\pipe\
```

Example output:

```powershell
PS C:\htb> gci \\.\pipe\

    Directory: \\.\pipe

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
------       12/31/1600   4:00 PM              3 InitShutdown
------       12/31/1600   4:00 PM              4 lsass
------       12/31/1600   4:00 PM              3 ntsvcs
------       12/31/1600   4:00 PM              3 scerpc

    Directory: \\.\pipe\Winsock2

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
------       12/31/1600   4:00 PM              1 Winsock2\CatalogChangeListener-34c-0

    Directory: \\.\pipe

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
------       12/31/1600   4:00 PM              3 epmapper
```

---

# Reviewing Named Pipe Permissions

## Why Check Pipe Permissions?

After listing named pipes, we can review their permissions.

This is useful because weak permissions may allow low-privileged users to:

- Read from a pipe
- Write to a pipe
- Modify pipe data
- Interact with a privileged service
- Abuse a service control interface
- Trigger privileged behaviour

Windows permissions are controlled through a **Discretionary Access Control List** or **DACL**.

A DACL shows which users or groups have permissions to a resource.

---

## Reviewing LSASS Named Pipe Permissions

Use `AccessChk` from Sysinternals to check named pipe permissions.

```cmd
accesschk.exe /accepteula \\.\Pipe\lsass -v
```

Example output:

```cmd
C:\htb> accesschk.exe /accepteula \\.\Pipe\lsass -v

Accesschk v6.12 - Reports effective permissions for securable objects
Copyright (C) 2006-2017 Mark Russinovich
Sysinternals - www.sysinternals.com

\\.\Pipe\lsass
  Untrusted Mandatory Level [No-Write-Up]
  RW Everyone
        FILE_READ_ATTRIBUTES
        FILE_READ_DATA
        FILE_READ_EA
        FILE_WRITE_ATTRIBUTES
        FILE_WRITE_DATA
        FILE_WRITE_EA
        SYNCHRONIZE
        READ_CONTROL
  RW NT AUTHORITY\ANONYMOUS LOGON
        FILE_READ_ATTRIBUTES
        FILE_READ_DATA
        FILE_READ_EA
        FILE_WRITE_ATTRIBUTES
        FILE_WRITE_DATA
        FILE_WRITE_EA
        SYNCHRONIZE
        READ_CONTROL
  RW APPLICATION PACKAGE AUTHORITY\Your Windows credentials
        FILE_READ_ATTRIBUTES
        FILE_READ_DATA
        FILE_WRITE_ATTRIBUTES
        FILE_WRITE_EA
        SYNCHRONIZE
        READ_CONTROL
  RW BUILTIN\Administrators
        FILE_ALL_ACCESS
```

In this example, only `BUILTIN\Administrators` have full access to the LSASS pipe, which is expected.

---

## Check All Named Pipe Permissions

To review DACLs for all named pipes:

```cmd
accesschk.exe /accepteula \pipe\
```

To search for named pipes where write access is available:

```cmd
accesschk.exe -w \pipe\* -v
```

---

## Useful AccessChk Flags

| Flag | Meaning |
|---|---|
| `/accepteula` | Accepts the Sysinternals EULA. Useful for non-interactive use. |
| `-w` | Shows objects with write access. |
| `-v` | Verbose output. |
| `\pipe\*` | Targets all named pipes. |

---

# Named Pipes Attack Example

## WindscribeService Named Pipe Privilege Escalation

A practical example of named pipe abuse is the **WindscribeService Named Pipe Privilege Escalation** vulnerability.

The issue is that the `WindscribeService` named pipe allowed overly permissive access.

Using `AccessChk`, we can check the pipe permissions.

```cmd
accesschk.exe -accepteula -w \pipe\WindscribeService -v
```

Example output:

```cmd
C:\htb> accesschk.exe -accepteula -w \pipe\WindscribeService -v

Accesschk v6.13 - Reports effective permissions for securable objects
Copyright (C) 2006-2020 Mark Russinovich
Sysinternals - www.sysinternals.com

\\.\Pipe\WindscribeService
  Medium Mandatory Level (Default) [No-Write-Up]
  RW Everyone
        FILE_ALL_ACCESS
```

---

## Why This Is Dangerous

The output shows:

```text
Everyone
FILE_ALL_ACCESS
```

This means all users have full access to the named pipe.

If the pipe server runs as a privileged service, a low-privileged user may be able to interact with it in a way that causes privileged actions.

This can potentially lead to escalation to:

```text
NT AUTHORITY\SYSTEM
```

---

## Named Pipe Privilege Escalation Pattern

A common named pipe privilege escalation pattern is:

1. Identify named pipes.
2. Find a named pipe belonging to a privileged service.
3. Check the pipe permissions.
4. Confirm low-privileged users can write to or fully control the pipe.
5. Understand how the service handles pipe input.
6. Abuse the pipe interaction to trigger privileged behaviour.
7. Escalate privileges.

---

# Practical Enumeration Workflow

## 1. Check Active Network Services

```cmd
netstat -ano
```

Look for:

- Localhost-only services
- Unusual listening ports
- Administrative interfaces
- Services mapped to privileged processes

---

## 2. Map Ports to Processes

```cmd
tasklist /fi "PID eq <PID>"
```

Example:

```cmd
tasklist /fi "PID eq 3812"
```

---

## 3. Investigate Interesting Services

For each interesting service, check:

- Service name
- Executable path
- Running user
- Version
- Configuration files
- File permissions
- Known vulnerabilities
- Local admin interfaces
- Stored credentials

Example:

```cmd
sc query
```

```cmd
sc qc <service_name>
```

---

## 4. List Named Pipes

Using PipeList:

```cmd
pipelist.exe /accepteula
```

Using PowerShell:

```powershell
gci \\.\pipe\
```

---

## 5. Check Named Pipe Permissions

Check a specific pipe:

```cmd
accesschk.exe /accepteula \\.\Pipe\<PipeName> -v
```

Check for writable pipes:

```cmd
accesschk.exe -w \pipe\* -v
```

---

## 6. Investigate Weak Pipe Permissions

Focus on pipes where low-privileged users or broad groups have excessive access.

Interesting groups include:

- `Everyone`
- `Authenticated Users`
- `Users`
- `ANONYMOUS LOGON`
- `INTERACTIVE`

Interesting permissions include:

- `FILE_WRITE_DATA`
- `FILE_WRITE_ATTRIBUTES`
- `FILE_WRITE_EA`
- `FILE_ALL_ACCESS`

---

# Key Takeaways

- Running processes and services are a key source of privilege escalation opportunities.
- Web services may allow shell upload or code execution as a service account.
- Service accounts may have useful privileges such as `SeImpersonatePrivilege`.
- `netstat -ano` can reveal local and externally exposed services.
- Localhost-only services are often misconfigured because they are assumed to be safe.
- FileZilla's admin port `14147` is a useful example of a localhost service to investigate.
- Splunk Universal Forwarder and Erlang-based services can provide privilege escalation opportunities if misconfigured.
- Named pipes are used for inter-process communication.
- Weak named pipe permissions can lead to privilege escalation.
- `PipeList`, PowerShell, and `AccessChk` are useful for named pipe enumeration.
- Always manually validate tool findings before attempting exploitation.

---

# Quick Reference

| Purpose | Command |
|---|---|
| Show active network connections | `netstat -ano` |
| Map PID to process | `tasklist /fi "PID eq <PID>"` |
| Query services | `sc query` |
| Show service configuration | `sc qc <service_name>` |
| List named pipes with PipeList | `pipelist.exe /accepteula` |
| List named pipes with PowerShell | `gci \\.\pipe\` |
| Check pipe permissions | `accesschk.exe /accepteula \\.\Pipe\<PipeName> -v` |
| Check all named pipe DACLs | `accesschk.exe /accepteula \pipe\` |
| Find writable named pipes | `accesschk.exe -w \pipe\* -v` |

---

## Summary

Processes, network services, and named pipes can expose powerful local privilege escalation paths.

The most useful checks are:

- What services are running?
- What ports are listening?
- Are any services only exposed on localhost?
- Which process owns each port?
- What account does the service run as?
- Are any named pipes writable by low-privileged users?
- Does a privileged service trust input from a weakly protected pipe?

Local services and named pipes are often overlooked, but they can provide direct escalation opportunities when permissions or configurations are weak.

---

## Tags

#Windows #PrivilegeEscalation #WindowsPrivilegeEscalation #Processes #NetworkServices #Netstat #NamedPipes #AccessTokens #AccessChk #PipeList #Sysinternals #SeImpersonatePrivilege #FileZilla #Splunk #Erlang #HTB #Pentesting #OSCP