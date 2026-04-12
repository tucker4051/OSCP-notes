# Protected File Transfers

## Overview

During CTFs, labs, and real-world pentests, we may obtain highly sensitive files such as:

- credential material
- password hashes
- NTDS.dit
- enumeration output
- infrastructure data
- Active Directory data
- user lists
- configuration files containing secrets

Whenever possible, these files should be transferred using **encrypted channels** or **encrypted containers/files**.

Preferred secure transport methods:

- SSH
- SCP
- SFTP
- HTTPS
- WinRM over HTTPS

If those are not available, encrypt the file **before transfer**.

---

## Important Handling Notes

- do not exfiltrate real sensitive client data unless explicitly authorised
- if testing DLP / egress controls, use **dummy data** that mimics the protected data type
- use strong, unique passwords for encrypted files
- use a different encryption password per client / engagement
- verify integrity after transfer
- clean up temporary copies where appropriate

Data leakage during an engagement can have serious consequences for:

- the client
- the consultancy
- the tester

Treat all collected data as sensitive.

---

## Quick Workflow

1. identify whether the file is sensitive
2. prefer encrypted transfer channel if available
3. if not, encrypt file locally first
4. transfer encrypted file
5. decrypt only in authorised environment
6. verify integrity and handle securely

---

# Windows File Encryption

## Overview

A simple approach on Windows is using a PowerShell AES encryption script such as:

```powershell
Invoke-AESEncryption.ps1
```

This can encrypt:

- strings
- files

Encrypted files are written with the `.aes` extension.

---

## Import the Module

```powershell
Import-Module .\Invoke-AESEncryption.ps1
```

---

## Encrypt a String

```powershell
Invoke-AESEncryption -Mode Encrypt -Key "p@ssw0rd" -Text "Secret Text"
```

Returns Base64 ciphertext.

---

## Decrypt a String

```powershell
Invoke-AESEncryption -Mode Decrypt -Key "p@ssw0rd" -Text "BASE64_CIPHERTEXT"
```

Returns plaintext.

---

## Encrypt a File

```powershell
Invoke-AESEncryption -Mode Encrypt -Key "p4ssw0rd" -Path .\scan-results.txt
```

Example output:

```text
File encrypted to C:\htb\scan-results.txt.aes
```

---

## Decrypt a File

```powershell
Invoke-AESEncryption -Mode Decrypt -Key "p4ssw0rd" -Path .\scan-results.txt.aes
```

This restores the original file.

---

## Example Workflow

```powershell
Import-Module .\Invoke-AESEncryption.ps1

Invoke-AESEncryption -Mode Encrypt -Key "p4ssw0rd" -Path .\scan-results.txt
```

Check resulting files:

```powershell
ls
```

Example:

```text
Invoke-AESEncryption.ps1
scan-results.txt
scan-results.txt.aes
```

---

## Windows Notes

- use strong, unique passwords
- do not reuse one password across clients
- transfer the `.aes` file, not the plaintext original
- keep decryption limited to authorised systems

---

# Linux File Encryption

## Overview

On Linux, `openssl` is a simple and common way to encrypt files before transfer.

Recommended options:

- `-aes256`
- `-iter 100000`
- `-pbkdf2`

This improves resistance against brute-force attacks.

---

## Encrypt a File

```bash
openssl enc -aes256 -iter 100000 -pbkdf2 -in /etc/passwd -out passwd.enc
```

You will be prompted for an encryption password.

---

## Decrypt a File

```bash
openssl enc -d -aes256 -iter 100000 -pbkdf2 -in passwd.enc -out passwd
```

You will be prompted for the decryption password.

---

## Example Workflow

Encrypt file:

```bash
openssl enc -aes256 -iter 100000 -pbkdf2 -in loot.txt -out loot.txt.enc
```

Transfer encrypted file using your preferred method.

Decrypt later:

```bash
openssl enc -d -aes256 -iter 100000 -pbkdf2 -in loot.txt.enc -out loot.txt
```

---

## Linux Notes

- `openssl` is commonly present on Linux systems
- prefer strong passphrases
- encrypted archive/file is safer than raw transfer over plaintext channels
- still prefer SSH/SFTP/HTTPS where available

---

# Preferred Secure Transport Options

If available, prefer these over plaintext transfer methods:

| Method | Notes |
|--------|------|
| SCP | simple encrypted file transfer over SSH |
| SFTP | interactive encrypted file transfer |
| SSH | secure transport tunnel |
| HTTPS | good for web-based transfer |
| WinRM over HTTPS | useful in Windows environments |

If none are available, use:

1. local encryption
2. transfer encrypted file
3. decrypt later in safe environment

---

# Practical Operator Guidance

## Sensitive Data Examples

Treat these as protected by default:

- NTDS.dit
- SAM / SYSTEM / SECURITY
- password dumps
- kerberos tickets
- API keys
- SSH keys
- internal network maps
- AD enumeration output
- cloud credentials
- database dumps

---

## Good Practice

- minimise copies
- minimise time at rest on disk
- use strong unique passwords
- avoid plaintext transfer where possible
- document handling decisions in notes if needed
- remove temporary unencrypted files when no longer needed

---

## If Testing DLP / Egress Controls

Unless explicitly authorised, do **not** exfiltrate real:

- PII
- financial records
- trade secrets
- regulated data

Instead, generate dummy files that simulate the target data type.

---

# When to Use What

| Scenario | Best Option |
|----------|-------------|
| SSH available | SCP / SFTP |
| HTTPS available | HTTPS upload/download |
| only plaintext methods available | encrypt first, then transfer |
| Windows host | PowerShell AES script |
| Linux host | OpenSSL |

---

# Cheatsheet

## Windows

```powershell
# import module
Import-Module .\Invoke-AESEncryption.ps1
```

```powershell
# encrypt string
Invoke-AESEncryption -Mode Encrypt -Key "StrongPassword" -Text "Secret Text"
```

```powershell
# decrypt string
Invoke-AESEncryption -Mode Decrypt -Key "StrongPassword" -Text "BASE64_CIPHERTEXT"
```

```powershell
# encrypt file
Invoke-AESEncryption -Mode Encrypt -Key "StrongPassword" -Path .\file.txt
```

```powershell
# decrypt file
Invoke-AESEncryption -Mode Decrypt -Key "StrongPassword" -Path .\file.txt.aes
```

## Linux

```bash
# encrypt file
openssl enc -aes256 -iter 100000 -pbkdf2 -in file.txt -out file.txt.enc
```

```bash
# decrypt file
openssl enc -d -aes256 -iter 100000 -pbkdf2 -in file.txt.enc -out file.txt
```

---

# Key Points

- encrypted transport is preferred
- if encrypted transport is unavailable, encrypt the file first
- use strong unique passwords
- do not unnecessarily exfiltrate real client-sensitive data
- protect all collected loot as if it were production-sensitive

---

# Tags

#file-transfer #encryption #openssl #powershell #sensitive-data #opsec #pentest #ctf #oscp