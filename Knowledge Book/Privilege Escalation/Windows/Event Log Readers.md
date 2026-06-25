## Overview

The `Event Log Readers` group allows members to read event logs from a local Windows machine.

This can become valuable during an assessment because event logs may contain sensitive information, including command-line arguments that expose credentials.

One especially important event is:

```text
Event ID 4688: A new process has been created
```

If auditing of process creation events and command-line values is enabled, Windows records process execution details in the Security event log.

---

# Why Event Log Readers Matters

Organizations may enable process creation logging to help defenders identify suspicious activity.

This visibility can help detect commands often used after initial access, such as:

- `whoami`
- `netstat`
- `tasklist`
- `ipconfig`
- `systeminfo`
- `net view`
- `net use`
- `wmic`
- `reg`

This data may be:

- saved locally in the Windows Security event log
- forwarded to a SIEM
- ingested into a search platform such as ElasticSearch
- reviewed by defenders during threat hunting
- used to alert on unusual command execution

> [!important]
> The same logging that helps defenders can also expose sensitive command-line values to users who can read the logs.

---

# Defensive Context

Process creation and command-line auditing can provide strong host-level visibility without requiring a full enterprise EDR deployment.

Organizations can use built-in Microsoft tooling to:

- detect suspicious command execution
- identify binaries that should not be present
- investigate lateral movement
- monitor post-exploitation reconnaissance
- restrict specific commands with AppLocker rules

> [!tip]
> For organizations with limited security budgets, process creation auditing, command-line logging, Windows Event Forwarding, and AppLocker can provide high-impact visibility and control.

---

# Offensive Value

From an attacker or tester perspective, access to event logs may reveal:

- passwords passed on the command line
- usernames used with remote commands
- mapped network drives
- backup share paths
- administrative commands
- scripts executed by admins
- service account usage
- remote hostnames and file shares

Many Windows commands support passing credentials as parameters. If command-line auditing is enabled, those credentials may be captured in event logs.

---

# Who Can Read These Logs?

The following users commonly have access:

- local administrators
- members of `Event Log Readers`

Administrators may add power users, developers, or support staff to `Event Log Readers` so they can troubleshoot systems without full administrative rights.

---

# Confirming Event Log Readers Membership

Check local group membership:

```cmd
net localgroup "Event Log Readers"
```

Example output:

```cmd
C:\htb> net localgroup "Event Log Readers"

Alias name     Event Log Readers
Comment        Members of this group can read event logs from local machine

Members

-------------------------------------------------------------------------------
logger
The command completed successfully.
```

In this example, the user `logger` is a member of the group.

---

# Key Event – Process Creation

## Event ID

```text
4688
```

## Event name

```text
A new process has been created
```

## Why It Matters

When command-line auditing is enabled, Event ID `4688` can include the process command line.

This may expose credentials when users run commands such as:

```cmd
net use T: \\fs01\backups /user:tim MyStr0ngP@ssword
```

---

# Method 1 – Search Security Logs with wevtutil

## Goal

Query the Security event log from the command line and search for command-line arguments containing credential-related strings.

---

## Search for `/user`

```powershell
wevtutil qe Security /rd:true /f:text | Select-String "/user"
```

Example output:

```powershell
PS C:\htb> wevtutil qe Security /rd:true /f:text | Select-String "/user"

        Process Command Line:   net use T: \\fs01\backups /user:tim MyStr0ngP@ssword
```

### Command breakdown

| Part | Purpose |
|---|---|
| `wevtutil` | Windows Event Log utility |
| `qe Security` | Query events from the Security log |
| `/rd:true` | Return events in reverse direction, newest first |
| `/f:text` | Format output as text |
| `Select-String "/user"` | Search output for `/user` |

---

# Method 2 – Query Remote Logs with wevtutil Credentials

## Goal

Use alternate credentials with `wevtutil` to query logs from a remote host.

---

## Syntax

```cmd
wevtutil qe Security /rd:true /f:text /r:<remote_host> /u:<username> /p:<password> | findstr "/user"
```

Example:

```cmd
wevtutil qe Security /rd:true /f:text /r:share01 /u:julie.clay /p:Welcome1 | findstr "/user"
```

### Command breakdown

| Part | Purpose |
|---|---|
| `/r:share01` | Query remote host `share01` |
| `/u:julie.clay` | Username for remote authentication |
| `/p:Welcome1` | Password for remote authentication |
| `findstr "/user"` | Search output for `/user` |

> [!warning]
> Passing credentials on the command line can itself create new logged credentials if process command-line auditing is enabled.

---

# Method 3 – Search Security Logs with Get-WinEvent

## Goal

Use PowerShell to filter for process creation events containing `/user` in the command line.

---

## Query Event ID 4688

```powershell
Get-WinEvent -LogName security | where { $_.ID -eq 4688 -and $_.Properties[8].Value -like '*/user*'} | Select-Object @{name='CommandLine';expression={ $_.Properties[8].Value }}
```

Example output:

```powershell
PS C:\htb> Get-WinEvent -LogName security | where { $_.ID -eq 4688 -and $_.Properties[8].Value -like '*/user*'} | Select-Object @{name='CommandLine';expression={ $_.Properties[8].Value }}

CommandLine
-----------
net use T: \\fs01\backups /user:tim MyStr0ngP@ssword
```

### Command breakdown

| Part | Purpose |
|---|---|
| `Get-WinEvent -LogName security` | Read the Security event log |
| `$_.ID -eq 4688` | Filter for process creation events |
| `$_.Properties[8].Value` | Access the command-line field in this example |
| `-like '*/user*'` | Find command lines containing `/user` |
| `Select-Object` | Display only the command line |

> [!note]
> Searching the Security event log with `Get-WinEvent` requires administrator access or adjusted permissions on the registry key:
>
> ```text
> HKLM\System\CurrentControlSet\Services\Eventlog\Security
> ```
>
> Membership in only `Event Log Readers` is not sufficient for this specific `Get-WinEvent` Security log method.

---

# Running Get-WinEvent as Another User

`Get-WinEvent` supports alternate credentials through the `-Credential` parameter.

General pattern:

```powershell
Get-WinEvent -LogName security -Credential <credential_object>
```

> [!tip]
> Use this when valid credentials are available for a user with sufficient event log access.

---

# Other Useful Logs

## PowerShell Operational Log

The PowerShell Operational log may contain sensitive information if script block logging or module logging is enabled.

Important detail:

```text
PowerShell Operational log is accessible to unprivileged users.
```

Useful log location:

```text
Microsoft-Windows-PowerShell/Operational
```

Potentially exposed information:

- script contents
- command arguments
- module activity
- credentials accidentally embedded in scripts
- administrative automation details

---

# Search Targets

When hunting event logs for exposed credentials, useful search terms may include:

```text
/user
```

```text
/pass
```

```text
/password
```

```text
pwd
```

```text
net use
```

```text
runas
```

```text
cmdkey
```

```text
PowerShell
```

```text
ConvertTo-SecureString
```

```text
-credential
```

```text
-PassThru
```

> [!note]
> These search terms are practical extensions for log review. The source example specifically demonstrates searching for `/user`.

---

# Practical Decision Tree

## Is the current user in Event Log Readers?

Check:

```cmd
net localgroup "Event Log Readers"
```

If the current user or a controlled user is listed, continue.

---

## Is process command-line auditing enabled?

Look for Event ID `4688` entries containing command-line data.

Search with `wevtutil`:

```powershell
wevtutil qe Security /rd:true /f:text | Select-String "/user"
```

---

## Does wevtutil return command lines with credentials?

Example exposed value:

```cmd
net use T: \\fs01\backups /user:tim MyStr0ngP@ssword
```

Use the discovered information only within the authorized assessment scope.

---

## Does Get-WinEvent fail?

Possible reason:

```text
Membership in Event Log Readers alone is not sufficient for this Security log query.
```

Use:

- administrator context
- adjusted Security event log registry permissions
- `wevtutil` if permitted
- alternate credentials with sufficient rights

---

## Are PowerShell logs enabled?

Check the PowerShell Operational log for sensitive script or module activity.

```text
Microsoft-Windows-PowerShell/Operational
```

---

# Troubleshooting

## No results for `/user`

Possible reasons:

- process command-line auditing is not enabled
- Security log has rolled over
- no users passed credentials on the command line
- logs are forwarded and cleared locally
- query syntax is too narrow
- relevant activity is in another log

Try searching for other terms:

```text
password
```

```text
pass
```

```text
cmdkey
```

```text
net use
```

```text
runas
```

---

## Access denied reading Security log

Possible causes:

- current user is not an administrator
- current user is not in `Event Log Readers`
- Security log permissions are restricted
- `Get-WinEvent` requires additional permissions for this query

---

## Get-WinEvent returns no CommandLine field

Possible causes:

- process command-line auditing is disabled
- Event ID `4688` exists but does not include command-line values
- event property index differs across versions or event formats
- the queried log does not contain matching process creation events

---

## Remote wevtutil query fails

Check:

- remote hostname is correct
- credentials are valid
- firewall permits remote event log access
- account has permissions to read the target log
- remote event log service is running
- command-line credentials may be logged

---

# Command Reference

## Check Event Log Readers members

```cmd
net localgroup "Event Log Readers"
```

## Query Security log with wevtutil

```powershell
wevtutil qe Security /rd:true /f:text
```

## Search Security log for `/user`

```powershell
wevtutil qe Security /rd:true /f:text | Select-String "/user"
```

## Query remote Security log with credentials

```cmd
wevtutil qe Security /rd:true /f:text /r:<remote_host> /u:<username> /p:<password> | findstr "/user"
```

## Example remote query

```cmd
wevtutil qe Security /rd:true /f:text /r:share01 /u:julie.clay /p:Welcome1 | findstr "/user"
```

## Search Event ID 4688 with Get-WinEvent

```powershell
Get-WinEvent -LogName security | where { $_.ID -eq 4688 -and $_.Properties[8].Value -like '*/user*'} | Select-Object @{name='CommandLine';expression={ $_.Properties[8].Value }}
```

## PowerShell Operational log path

```text
Microsoft-Windows-PowerShell/Operational
```

---

# Defensive Notes

## Benefits of Process Creation Auditing

Process creation auditing can help identify:

- suspicious reconnaissance commands
- unexpected administrative binaries
- command execution from unusual workstations
- lateral movement tooling
- malware staging or spreading behavior

Examples of commands defenders may monitor:

```text
tasklist
ver
ipconfig
systeminfo
dir
net view
ping
net use
type
at
reg
wmic
wusa
```

---

# Defensive Hardening Ideas

Organizations can reduce risk by:

- enabling process creation auditing
- enabling command-line logging for process creation
- forwarding logs to a SIEM
- tuning detections for suspicious command usage
- limiting membership in `Event Log Readers`
- avoiding credentials on command lines
- using managed service accounts where possible
- reviewing PowerShell logging exposure
- implementing AppLocker rules for restricted commands
- reviewing built-in group memberships regularly

> [!warning]
> Command-line logging is useful for detection, but it can also store sensitive data. Avoid passing passwords directly as command arguments.

---

# Key Takeaways

- `Event Log Readers` can provide access to logs that reveal sensitive command-line data.
- Event ID `4688` records process creation events.
- If command-line auditing is enabled, Event ID `4688` may include full process command lines.
- Credentials passed to commands such as `net use` may be exposed in logs.
- `wevtutil` can query Security logs from the command line.
- `wevtutil` supports alternate credentials with `/u` and `/p`.
- `Get-WinEvent` can filter Event ID `4688`, but Security log access may require more than `Event Log Readers`.
- The PowerShell Operational log may expose sensitive information and can be readable by unprivileged users.
- Process creation auditing is valuable for defenders but must be managed carefully to avoid sensitive data exposure.

---

# Related Notes

- [[Windows Privilege Escalation]]
- [[Windows Built-In Groups]]
- [[Event Log Readers]]
- [[Windows Event Logs]]
- [[Event ID 4688]]
- [[wevtutil]]
- [[Get-WinEvent]]
- [[PowerShell Logging]]
- [[AppLocker]]
- [[Knowledge Book/Privilege Escalation/Linux/Credential Hunting]]
- [[SIEM]]
- [[Windows Security Log]]

---

# Tags

#windows
#privilege-escalation
#event-log-readers
#event-logs
#security-log
#event-id-4688
#process-creation
#command-line-logging
#credential-hunting
#wevtutil
#get-winevent
#powershell-operational
#applocker
#siem
#pentesting