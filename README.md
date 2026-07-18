# Penetration Testing Knowledge Book

A structured collection of penetration testing, offensive security, and lab reference notes maintained in Markdown.

This repository is designed as a practical knowledge base for penetration testing study, certification preparation, security labs, CTFs, and authorised security assessments. The notes focus on repeatable methodology, technical concepts, enumeration techniques, exploitation workflows, privilege escalation, Active Directory, pivoting, credential attacks, and commonly used tools.

The knowledge base is primarily maintained in [Obsidian](https://obsidian.md/), but all notes use standard Markdown and can be viewed directly through GitHub or any Markdown-compatible editor.

---

## Purpose

The purpose of this repository is to provide a central and logically organised reference for penetration testing activities.

Rather than functioning as a collection of isolated commands, the notes aim to document:

* Why a technique is used
* When it is appropriate
* How the technique works
* Commands and tools commonly associated with it
* Common problems and troubleshooting steps
* Links between related attack techniques
* Practical workflows that can be reused during labs and assessments

The structure broadly follows the lifecycle of a penetration test, from initial reconnaissance through exploitation, privilege escalation, pivoting, and post-exploitation activity.

---

## Repository Structure

```text
Knowledge Book
├── 00 - Start Here
├── 01 - Reconnaissance & Enumeration
├── 02 - Vulnerability Discovery & Exploit Research
├── 03 - Web Application Attacks
├── 04 - Exploitation & Initial Access
├── 05 - Privilege Escalation
├── 06 - Credential Attacks
├── 07 - Active Directory
├── 08 - Pivoting & Tunnelling
├── 09 - Post Exploitation & Evasion
├── 10 - Tools & Quick Commands
└── 99 - Templates & Checklists
```

### 00 - Start Here

Entry-point notes covering the overall penetration testing workflow, CTF methodology, and common ports.

### 01 - Reconnaissance & Enumeration

Notes covering external reconnaissance, OSINT, service discovery, protocol enumeration, network services, databases, and remote management interfaces.

### 02 - Vulnerability Discovery & Exploit Research

Guidance for vulnerability scanning, application discovery, exploit research, public exploit review, default credentials, misconfiguration testing, and findings prioritisation.

### 03 - Web Application Attacks

Detailed notes covering common web application vulnerabilities and attack techniques, including:

* Command injection
* SQL injection
* File inclusion
* File upload attacks
* Cross-site scripting
* Authentication weaknesses
* Access-control vulnerabilities
* API attacks
* Server-side attacks

### 04 - Exploitation & Initial Access

Practical material covering exploitation methodology, service exploitation, application exploitation, shells, payload generation, file transfers, Metasploit, and initial-access troubleshooting.

### 05 - Privilege Escalation

Separate Linux and Windows privilege-escalation sections covering enumeration, permissions, services, credentials, token abuse, scheduled tasks, path hijacking, containers, kernel exploits, and other common escalation paths.

### 06 - Credential Attacks

Notes covering credential discovery, password attacks, hash cracking, credential storage, authenticated services, and credential reuse techniques.

### 07 - Active Directory

Active Directory attack methodology covering:

* Enumeration
* Password policies
* Password spraying
* LLMNR and NBT-NS poisoning
* Security controls
* Kerberos attacks
* Credential attacks
* Lateral movement

### 08 - Pivoting & Tunnelling

Techniques for reaching internal networks through compromised systems, including SSH forwarding, SOCKS proxies, Chisel, Socat, Windows pivoting, Meterpreter, DNS tunnelling, and ICMP tunnelling.

### 09 - Post Exploitation & Evasion

Post-exploitation and defensive-evasion concepts, including AV evasion and bypass techniques.

### 10 - Tools & Quick Commands

Condensed tool references and command examples for commonly used penetration testing utilities.

Examples include:

* Nmap
* enum4linux-ng
* rpcclient
* ffuf
* Gobuster
* smbclient
* Evil-WinRM
* Impacket
* Responder
* Kerbrute
* Hashcat
* John the Ripper
* Hydra
* Metasploit
* MSFVenom
* proxychains

### 99 - Templates & Checklists

Reusable penetration testing checklists and a structured note template for documenting labs, CTFs, and authorised assessments.

---

## Suggested Starting Points

New readers may find the following notes useful:

```text
00 - Start Here
├── Pentest Workflow
├── CTF Box Methodology
└── Common Ports
```

For practical testing, the general workflow is:

```text
Reconnaissance
    ↓
Enumeration
    ↓
Vulnerability discovery
    ↓
Exploit research
    ↓
Exploitation
    ↓
Initial access
    ↓
Privilege escalation
    ↓
Credential attacks
    ↓
Pivoting and lateral movement
    ↓
Post exploitation
```

This is not a rigid sequence. Real assessments are iterative, and new information frequently requires returning to earlier phases.

---

## Note Naming Convention

Notes use numbered prefixes to keep related subjects grouped together.

Example:

```text
03.02 - Injection Attacks
├── 03.02.10 - Command Injection
│   ├── 03.02.10.01 - Command Injection Overview.md
│   ├── 03.02.10.02 - Detection.md
│   └── 03.02.10.03 - Injection Operators.md
│
└── 03.02.20 - SQL Injection
    ├── 03.02.20.01 - SQL Injection Overview.md
    ├── 03.02.20.02 - Subverting Query Logic.md
    └── 03.02.20.03 - Using Comments.md
```

Numbering gaps are intentional. They allow additional sections or notes to be inserted later without requiring the entire knowledge base to be renumbered.

---

## Using the Notes with Obsidian

To use the repository as an Obsidian vault:

1. Clone the repository.
2. Open Obsidian.
3. Select **Open folder as vault**.
4. Choose the repository or `Knowledge Book` directory.
5. Allow Obsidian to index the Markdown files and internal links.

Clone the repository with:

```bash
git clone <repository-url>
```

Then open the cloned directory in Obsidian.

The `.obsidian` directory is intentionally excluded from version control so that local workspace settings, themes, and plugins are not committed.

Example `.gitignore` entry:

```gitignore
.obsidian/
```

---

## Penetration Test Note Template

The repository includes a reusable template for documenting a penetration test, CTF, or lab machine.

The template covers:

```text
00 - Overview
01 - Enumeration
02 - Findings & Attack Paths
03 - Exploitation & Access
04 - Privilege Escalation
05 - Loot & Evidence
06 - Scripts & Payloads
07 - Writeup
```

This provides a consistent structure for recording:

* Scan results
* Service enumeration
* Potential attack paths
* Confirmed vulnerabilities
* Exploit attempts
* Initial foothold
* Privilege escalation
* Credentials and hashes
* Evidence
* Custom scripts
* Final attack chain
* Lessons learned

---

## Content Status

This repository is continuously updated as new subjects, labs, tools, and techniques are studied.

Some notes are comprehensive reference pages, while others are concise command guides or study summaries. Certain sections may therefore be more developed than others.

Notes may be reorganised, expanded, or consolidated as the knowledge base grows.

---

## Legal and Ethical Use

This repository is intended solely for:

* Authorised penetration testing
* Cybersecurity training
* Certification study
* Capture-the-flag challenges
* Personal lab environments
* Defensive security research

Do not use any technique documented in this repository against systems, applications, networks, or accounts without explicit authorisation.

The repository owner accepts no responsibility for unlawful, unauthorised, or unethical use of the material.

Always ensure that testing activities are covered by clear permission and an agreed scope.

---

## Disclaimer

The commands and techniques contained in these notes may be incomplete, environment-specific, outdated, or potentially disruptive.

Before running any command:

* Understand what it does
* Confirm the target is within scope
* Consider the effect on the target system
* Test in a controlled environment where possible
* Adapt commands to the relevant operating system and tool version

These notes should be treated as a learning and reference resource rather than a substitute for professional judgement.

---

## Contributions

This is primarily a personal knowledge base, but corrections and constructive suggestions are welcome.

When proposing changes:

* Preserve the existing folder logic
* Use clear Markdown formatting
* Keep commands inside properly closed code blocks
* Avoid adding unverified techniques
* Include context rather than isolated commands
* Do not include real credentials, client information, flags, or sensitive data

---

## Author

Maintained by Scott Tucker as part of ongoing penetration testing, offensive security, and cybersecurity study.

---

## Licence

Unless a separate licence file is provided, the material in this repository should be treated as personal reference content.

Third-party commands, tools, screenshots, quotations, and course-derived material remain subject to their respective owners' licences and terms.
