# Tools & Quick Commands — John The Ripper (JtR)

## Basic Syntax

```bash
john [options] <hash_file>
```

Show cracked passwords:

```bash
john --show <hash_file>
```

Show cracking status:

```bash
john --status
```

Resume session:

```bash
john --restore
```

---

# Common Attack Modes

## Single Crack Mode

Uses username & account data

```bash
john --single hashes.txt
```

Best for:

- passwd/shadow files
- usernames known
- OS credential dumps

---

## Wordlist Mode

Dictionary attack

```bash
john --wordlist=rockyou.txt hashes.txt
```

With rules:

```bash
john --wordlist=rockyou.txt --rules hashes.txt
```

Multiple wordlists:

```bash
john --wordlist=list1.txt,list2.txt hashes.txt
```

---

## Incremental Mode (Brute Force)

```bash
john --incremental hashes.txt
```

Specify charset:

```bash
john --incremental=ASCII hashes.txt
john --incremental=Latin1 hashes.txt
john --incremental=UTF8 hashes.txt
```

Limit length:

```bash
john --incremental --min-length=6 --max-length=10 hashes.txt
```

---

# Specify Hash Format

```bash
john --format=<format> hashes.txt
```

Examples:

```bash
john --format=nt hashes.txt
john --format=raw-md5 hashes.txt
john --format=sha512crypt hashes.txt
john --format=netntlmv2 hashes.txt
john --format=mssql hashes.txt
```

List supported formats:

```bash
john --list=formats
```

---

# Identify Hash Type

Using hashid:

```bash
hashid hash.txt
```

With JtR format hint:

```bash
hashid -j hash.txt
```

---

# Show Cracked Passwords

```bash
john --show hashes.txt
```

---

# Performance Tuning

Limit CPU threads:

```bash
john --fork=4 hashes.txt
```

Limit length:

```bash
john --min-length=6 hashes.txt
john --max-length=12 hashes.txt
```

Custom charset:

```bash
john --incremental=Custom hashes.txt
```

---

# 2john File Conversion Tools

Convert files to JtR format:

```bash
<tool> <file> > hash.txt
```

Examples:

## ZIP

```bash
zip2john file.zip > zip.hash
john zip.hash
```

## RAR

```bash
rar2john file.rar > rar.hash
john rar.hash
```

## PDF

```bash
pdf2john file.pdf > pdf.hash
john pdf.hash
```

## SSH private key

```bash
ssh2john id_rsa > ssh.hash
john ssh.hash
```

## KeePass database

```bash
keepass2john database.kdbx > keepass.hash
john keepass.hash
```

## Office documents

```bash
office2john file.docx > office.hash
john office.hash
```

## WPA handshake

```bash
hccap2john capture.hccap > wifi.hash
john wifi.hash
```

---

# Common Hash Attack Examples

## NTLM

```bash
john --format=nt --wordlist=rockyou.txt ntlm.txt
```

## Linux shadow hash

```bash
john --format=sha512crypt shadow.hash
```

## NetNTLMv2

```bash
john --format=netntlmv2 hashes.txt
```

## MSSQL hash

```bash
john --format=mssql hashes.txt
```

## MD5

```bash
john --format=raw-md5 hashes.txt
```

---

# Useful Wordlists

Default rockyou:

```bash
/usr/share/wordlists/rockyou.txt
```

SecLists:

```bash
/usr/share/seclists/
```

Custom list:

```bash
cewl example.com > wordlist.txt
```

---

# Useful Locations

John config:

```bash
/etc/john/john.conf
```

Wordlists:

```bash
/usr/share/wordlists/
```

---

# Quick Workflow

Identify hash:

```bash
hashid hash.txt
```

Wordlist attack:

```bash
john --wordlist=rockyou.txt hash.txt
```

Apply rules:

```bash
john --wordlist=rockyou.txt --rules hash.txt
```

Fallback brute force:

```bash
john --incremental hash.txt
```

Show results:

```bash
john --show hash.txt
```

---

# Tags

#tools #john #password-cracking #cheatsheet #oscp