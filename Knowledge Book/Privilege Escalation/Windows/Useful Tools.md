# Windows Privilege Escalation — Useful Tools

## Overview

When enumerating Windows systems for privilege escalation opportunities, there are many tools that can help identify common and obscure misconfigurations.

These tools can speed up enumeration, highlight interesting findings, and present information in a more readable format. However, they should not replace manual understanding.

```
title: Key Point
Privilege escalation tools are useful for narrowing down possible attack paths, but they can produce false positives, false negatives, and large amounts of noisy output.

Manual enumeration skills are still essential.
```

---

## Useful Windows Privilege Escalation Tools

| Tool | Description |
|---|---|
| [Seatbelt](https://github.com/GhostPack/Seatbelt) | C# project for performing a wide variety of local privilege escalation checks. |
| [winPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/winPEAS) | Script that searches for possible Windows privilege escalation paths. The checks are explained in the HackTricks Windows privilege escalation checklist. |
| [PowerUp](https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Privesc/PowerUp.ps1) | PowerShell script for finding common Windows privilege escalation vectors caused by misconfigurations. It can also exploit some identified issues. |
| [SharpUp](https://github.com/GhostPack/SharpUp) | C# version of PowerUp. |
| [JAWS](https://github.com/411Hall/JAWS) | PowerShell script for enumerating privilege escalation vectors, written in PowerShell 2.0. |
| [SessionGopher](https://github.com/Arvanaghi/SessionGopher) | PowerShell tool that finds and decrypts saved session information for remote access tools such as PuTTY, WinSCP, SuperPuTTY, FileZilla, and RDP. |
| [Watson](https://github.com/rasta-mouse/Watson) | .NET tool used to enumerate missing KBs and suggest privilege escalation exploits. |
| [LaZagne](https://github.com/AlessandroZ/LaZagne) | Tool used to retrieve passwords stored locally by browsers, chat tools, databases, Git, email clients, memory dumps, PHP tools, sysadmin tools, wireless configurations, and Windows credential storage mechanisms. |
| [Windows Exploit Suggester - Next Generation](https://github.com/bitsadmin/wesng) | Tool based on the output of Windows `systeminfo`. It identifies vulnerabilities and possible exploits affecting the target operating system. |
| [Sysinternals Suite](https://docs.microsoft.com/en-us/sysinternals/downloads/sysinternals-suite) | Microsoft tool suite useful for Windows enumeration. Relevant tools include AccessChk, PipeList, and PsService. |

---

## Pre-Compiled Binaries

Some tools have pre-compiled binaries available:

| Tool | Pre-Compiled Source |
|---|---|
| Seatbelt | [GhostPack Compiled Binaries](https://github.com/r3motecontrol/Ghostpack-CompiledBinaries) |
| SharpUp | [GhostPack Compiled Binaries](https://github.com/r3motecontrol/Ghostpack-CompiledBinaries) |
| LaZagne | [LaZagne Releases](https://github.com/AlessandroZ/LaZagne/releases/) |

```
In a real client environment, it is recommended to compile tools from source rather than relying on unknown or pre-compiled binaries.

This reduces supply chain risk and allows better control over what is being executed.
```

---

## Common Upload Location

Depending on how access is gained to a Windows system, the current user may not have write access to many directories.

A commonly writable location is:

```powershell
C:\Windows\Temp
```

This is often useful because the `BUILTIN\Users` group usually has write access.

```
Practical Tip
If unsure where to upload tools during a lab or assessment, `C:\Windows\Temp` is often a safe first choice.
```

---

## Why These Tools Are Useful

Windows enumeration can produce a huge amount of information. Privilege escalation tools help by:

- Automating repetitive checks
- Highlighting common misconfigurations
- Presenting output in a structured format
- Reducing the time needed to identify likely attack paths
- Helping focus manual investigation

Examples of checks these tools may perform include:

- Service permission issues
- Weak file or folder permissions
- Unquoted service paths
- Stored credentials
- Missing patches
- AutoLogon credentials
- AlwaysInstallElevated settings
- Scheduled task misconfigurations
- Interesting registry keys
- Privileged token or group membership issues

---

## Tools Are a Double-Edged Sword

Automated tools can be extremely helpful, but they also have limitations.

### Advantages

- Faster enumeration
- More complete initial coverage
- Easier to spot obvious issues
- Useful for comparing systems
- Helpful during time-limited assessments

### Disadvantages

- Can produce too much output
- May miss issues
- May report false positives
- May trigger antivirus or EDR
- May be blocked from running
- Can create over-reliance on automation

```
Do Not Rely Blindly on Tools
Tools should support enumeration, not replace understanding.

A tester should know what each tool is checking, why the result matters, and how to manually verify the finding.
```

---

## Manual Enumeration Is Essential

Even when using tools, manual enumeration remains important.

Manual skills help when:

- A tool fails to run
- The host blocks script execution
- Antivirus or EDR removes the tool
- The environment is air-gapped
- Uploading binaries is not possible
- Tool output is unclear
- A finding needs to be manually validated
- The client asks for a technical explanation

```
Manual First Mindset
A good approach is to understand the manual technique first, then use tools to speed up the process.

This makes it easier to troubleshoot tool output and explain findings clearly.
```

---

## Exploitation Should Also Be Understood

The same principle applies to exploitation.

It is not enough to run an "autopwn" script without understanding what it does. During a professional assessment, the tester should be able to explain:

- What vulnerability or misconfiguration exists
- Why it is exploitable
- What the exploit does
- What permissions are required
- What the expected impact is
- What risks the exploit introduces
- How to remediate the issue

```
Avoid Blind Exploitation
Do not run exploit scripts blindly, especially in client environments.

Understand the exploit path, expected behaviour, and potential impact before execution.
```

---

## Writing Custom Scripts

It is acceptable, and often encouraged, to write custom scripts for enumeration or exploitation.

Custom scripts can be useful because they are:

- Easier to control
- Easier to explain
- Easier to tailor to the environment
- Less noisy than large automated tools
- Sometimes less likely to be detected than common public tools

However, custom scripts should still be tested carefully before use.

---

## Use Cases Beyond Penetration Testing

These tools are not only useful for penetration testers. They can also help:

- System administrators
- Security engineers
- Internal audit teams
- Compliance teams
- Infrastructure teams
- Security architects

Common defensive use cases include:

- Identifying low-hanging fruit before an assessment
- Periodic checks of machine security posture
- Reviewing the impact of upgrades or configuration changes
- Testing new gold images before production deployment
- Supporting internal security reviews
- Validating hardening standards

```
Defensive Use Case
Before deploying a new Windows gold image, a security engineer could run tools such as Seatbelt, winPEAS, or PowerUp to check for weak permissions, stored credentials, or common privilege escalation paths.
```

---

## Risks of Running Enumeration Tools

Like any automation, privilege escalation tools can introduce risk.

Potential risks include:

- Triggering antivirus alerts
- Triggering EDR detections
- Creating noisy logs
- Causing instability on fragile systems
- Running checks too aggressively
- Accessing sensitive credential stores
- Producing findings that require careful handling

```
Fragile Systems
Although rare, excessive enumeration may cause issues on unstable or fragile systems.

This is especially important in production environments.
```

---

## Antivirus and EDR Detection

Many well-known privilege escalation tools are detected by antivirus and EDR products.

Examples of products that may detect or block these tools include:

- Microsoft Defender
- Cylance
- Carbon Black
- CrowdStrike
- SentinelOne
- Sophos
- Trend Micro

For example, a precompiled `LaZagne` binary may be heavily detected because it is a known credential recovery tool.

```
Detection Example
At the time of the original module, uploading the LaZagne version 2.4.3 binary to VirusTotal showed that 47 out of 70 products detected it.
```

---

## Engagement Context Matters

During some assessments, the client may want to test detection and response capability. In that type of engagement, running noisy public tools may be part of the objective.

During other assessments, the goal may simply be to identify as many weaknesses as possible. In that case, the client may temporarily configure security tools, such as Microsoft Defender, not to block testing activity.

```
Always Confirm Rules of Engagement
Before running privilege escalation tools, confirm whether the assessment allows:

- Uploading tools
- Running public exploit binaries
- Running credential recovery tools
- Bypassing antivirus
- Disabling security controls
- Performing noisy enumeration
```

---

## AV Bypass Considerations

Some techniques can be used to modify tools so they are less likely to be detected by antivirus products.

Examples include:

- Removing comments
- Renaming functions
- Changing strings
- Recompiling from source
- Encrypting or packing executables
- Modifying signatures
- Changing execution methods

```
Scope Reminder
AV bypass techniques should only be used when explicitly authorised and within the rules of engagement.

For this module, the focus is Windows privilege escalation, not AV evasion.
```

---

## Practical Workflow

A sensible approach during Windows privilege escalation enumeration is:

1. Perform basic manual enumeration.
2. Identify the current user, groups, privileges, hostname, domain, and OS version.
3. Check writable directories.
4. Upload or execute tools only if permitted.
5. Run targeted enumeration tools.
6. Review the output carefully.
7. Manually validate interesting findings.
8. Exploit only where authorised and understood.
9. Document the finding, impact, evidence, and remediation.

---

## Example Initial Checks

Before using automated tools, gather basic system context.

```cmd
whoami
```

```cmd
whoami /priv
```

```cmd
whoami /groups
```

```cmd
hostname
```

```cmd
systeminfo
```

```cmd
ipconfig /all
```

```cmd
net user
```

```cmd
net localgroup
```

```cmd
net localgroup administrators
```

---

## Example Upload Directory Check

Check whether `C:\Windows\Temp` is writable.

```cmd
echo test > C:\Windows\Temp\test.txt
```

```cmd
type C:\Windows\Temp\test.txt
```

```cmd
del C:\Windows\Temp\test.txt
```

---

## Key Takeaways

- Windows privilege escalation tools can greatly speed up enumeration.
- Tools should support manual investigation, not replace it.
- Public tools are commonly detected by antivirus and EDR.
- Precompiled binaries should be treated with caution in client environments.
- Always understand what a tool is doing before running it.
- Always validate tool findings manually.
- The rules of engagement determine what tools and techniques are appropriate.
- Manual enumeration remains essential for professional penetration testing.

---

## Quick Reference

| Area | Key Point |
|---|---|
| Tool usage | Helpful for fast enumeration and narrowing down attack paths. |
| Manual skills | Required to verify findings and troubleshoot tool limitations. |
| Upload location | `C:\Windows\Temp` is commonly writable by standard users. |
| Detection risk | Public tools are often detected by AV and EDR. |
| Client work | Prefer compiling from source and follow rules of engagement. |
| Exploitation | Avoid blindly running autopwn scripts. |
| Defensive value | These tools can also help administrators harden systems. |

```ad-summary
title: Summary
Privilege escalation tools are powerful enumeration aids, but they are not magic.

The best approach is to understand the underlying techniques, use tools to speed up checks, manually validate findings, and always work within the agreed scope.
```

#privilege-escalation #windows #tools 