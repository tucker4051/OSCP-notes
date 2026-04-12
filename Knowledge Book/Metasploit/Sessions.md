# Metasploit Sessions & Jobs

## Overview

Metasploit can manage multiple active interactions at the same time.

Two core concepts make this possible:

- **Sessions** → active communication channels with target hosts
- **Jobs** → background tasks running inside Metasploit

This is one of the biggest strengths of `msfconsole`, especially during:

- multi-host exploitation
- post-exploitation
- pivoting
- handler management
- long-running scans / listeners

---

# 1. Sessions

## What a Session Is

A **session** is an active connection to a target created after a successful payload or compatible module execution.

Examples:

- Meterpreter session
- command shell session
- PowerShell session

A session gives you an interface to interact with the compromised host.

---

## Session Characteristics

- remains active after exploitation
- can be backgrounded
- can be revisited later
- may die if communication breaks
- can be used by post modules

---

# 2. Listing Sessions

To see active sessions:

```bash
sessions
```

Example output:

```text
Active sessions
===============

  Id  Name  Type                     Information                 Connection
  --  ----  ----                     -----------                 ----------
  1         meterpreter x86/windows  NT AUTHORITY\SYSTEM @ MS01  10.10.10.129:443 -> 10.10.10.205:50501
```

---

# 3. Interacting with a Session

To interact with a specific session:

```bash
sessions -i 1
```

Example:

```text
[*] Starting interaction with 1...

meterpreter >
```

This is one of the most-used session commands in Metasploit.

---

# 4. Backgrounding a Session

If you want to keep a session alive but return to the main `msfconsole` prompt:

## From Meterpreter

```bash
background
```

or press:

```text
CTRL + Z
```

After confirmation, you return to:

```text
msf6 >
```

The session continues running in the background.

---

## Why Background a Session?

You may want to:

- run a different exploit
- launch a post module
- start a listener
- inspect another host
- pivot using the existing session

Backgrounding is essential in multi-step workflows.

---

# 5. Using a Session with Other Modules

Many post-exploitation modules can run against an existing session.

Typical workflow:

1. exploit target
2. get session
3. background session
4. search/select post module
5. set `SESSION <id>`
6. run module

This is especially common with:

- local exploit suggesters
- credential gatherers
- token modules
- network scanners
- persistence modules

---

# 6. Jobs

## What a Job Is

A **job** is a background task running inside Metasploit.

Examples:

- reverse TCP handler
- auxiliary scanner
- passive listener
- long-running exploit / module

Jobs are about **tasks**.  
Sessions are about **access to targets**.

---

## Session vs Job

| Concept | Meaning |
|--------|---------|
| Session | active shell / Meterpreter channel to target |
| Job | background Metasploit task such as handler or scanner |

---

# 7. Why Jobs Matter

Jobs are useful when you need to:

- keep a handler running
- keep a passive listener alive
- run multiple tasks simultaneously
- free up the console while a task continues
- avoid terminating Metasploit functionality accidentally

---

# 8. Job Help

Show job help:

```bash
jobs -h
```

Common options:

| Option | Purpose |
|--------|---------|
| `jobs -l` | list jobs |
| `jobs -i <id>` | detailed info on job |
| `jobs -k <id>` | kill a specific job |
| `jobs -K` | kill all jobs |
| `jobs -p <id>` | persist job |
| `jobs -P` | persist all jobs |

---

# 9. Running an Exploit as a Job

To run a module in the background as a job:

```bash
exploit -j
```

or

```bash
run -j
```

Example:

```text
[*] Exploit running as background job 0.
[*] Started reverse TCP handler on 10.10.14.34:4444
```

This is very common with:

- `multi/handler`
- passive listeners
- long-running modules

---

# 10. Listing Jobs

List running jobs:

```bash
jobs -l
```

Example:

```text
Jobs
====

 Id  Name                    Payload                    Payload opts
 --  ----                    -------                    ------------
 0   Exploit: multi/handler  generic/shell_reverse_tcp  tcp://10.10.14.34:4444
```

---

# 11. Killing Jobs

Kill a specific job:

```bash
jobs -k 0
```

Kill all jobs:

```bash
jobs -K
```

Use this when:

- a handler is holding a port
- a background task is no longer needed
- you want to clean up Metasploit state

---

# 12. Why Killing the Session May Not Free the Port

Important distinction:

- ending a **session** does not necessarily stop the **job**
- a handler running as a job may continue listening on a port even if no session exists

Example issue:

You stop interacting with a shell, but `4444` is still in use.

Cause:

- background `multi/handler` job still running

Fix:

```bash
jobs -l
jobs -k <id>
```

---

# 13. Common Workflow Example

## Start a Handler as a Job

```bash
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 10.10.14.34
set LPORT 4444
exploit -j
```

## List Jobs

```bash
jobs -l
```

## Wait for Session

```bash
sessions
```

## Interact

```bash
sessions -i 1
```

## Background Session

```bash
background
```

## Kill Old Handler if Needed

```bash
jobs -k 0
```

---

# 14. Useful Session Commands

## List sessions

```bash
sessions
```

## Interact with session

```bash
sessions -i 1
```

## Background session

```bash
background
```

## Kill a session

```bash
sessions -k 1
```

## Upgrade shell session

```bash
sessions -u 1
```

Useful when upgrading a basic shell to Meterpreter if supported.

---

# 15. Useful Job Commands

## List jobs

```bash
jobs -l
```

## Detailed job info

```bash
jobs -i 0
```

## Kill job

```bash
jobs -k 0
```

## Kill all jobs

```bash
jobs -K
```

---

# 16. Practical OSCP Guidance

Use **sessions** when you want to manage compromised hosts.

Use **jobs** when you want to manage background Metasploit tasks.

Typical pattern:

- exploit → session created
- background session
- run post modules against session
- keep handler as job
- kill old jobs when ports need freeing

---

# 17. Common Pitfalls

- forgetting a handler is still running
- confusing sessions with jobs
- killing session but not job
- leaving old listeners on ports you want to reuse
- backgrounding a fragile shell and losing it
- failing to note session IDs during multi-target work

---

# 18. Key Takeaways

- sessions are active target connections
- jobs are background tasks inside Metasploit
- backgrounding a session does not kill it
- killing a session does not always stop the handler job
- `sessions -i` and `jobs -l` are core commands to memorise
- `exploit -j` is especially useful for handlers

---

# Cheatsheet

## Sessions

```bash
sessions
sessions -i 1
background
sessions -k 1
sessions -u 1
```

## Jobs

```bash
jobs -h
jobs -l
jobs -i 0
jobs -k 0
jobs -K
```

## Run as job

```bash
exploit -j
run -j
```

## Typical handler workflow

```bash
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 10.10.14.34
set LPORT 4444
exploit -j
jobs -l
sessions
sessions -i 1
```

---

# Related Notes

- Metasploit overview
- payloads
- Meterpreter commands
- multi/handler
- post modules
- pivoting

---

# Tags

#metasploit #sessions #jobs #meterpreter #handler #postexploitation #oscp