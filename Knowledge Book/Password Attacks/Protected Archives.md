# Cracking Protected Archives / BitLocker

## Overview

Encrypted archives and full-disk encryption containers often contain high-value data:

- customer data
- configuration backups
- credentials
- source code
- SSH keys
- internal documentation
- database exports

Many archive formats support password protection, and hashes can often be extracted for offline cracking using John the Ripper or Hashcat.

Common targets:

| Type | Examples |
|------|---------|
| ZIP archives | backups, exports |
| 7zip archives | compressed backups |
| RAR archives | proprietary data |
| GZIP encrypted files | manually encrypted data |
| BitLocker VHD | full disk encryption |
| encrypted backups | company data containers |

---

# Cracking ZIP Files

ZIP archives are widely used in Windows environments.

## Extract hash

```bash
zip2john ZIP.zip > zip.hash
```

Example hash:

```text
ZIP.zip/customers.csv:$pkzip2$1*2*2*0*2a*1e*490e7510*0*42*0*2a*490e*409b*ef1e7feb7c1cf701a6ada7132e6a5c6c84c032401536faf7493df0294b0d5afc3464f14ec081cc0e18cb*$/pkzip2$
```

## Crack with John

```bash
john --wordlist=rockyou.txt zip.hash
```

## Show result

```bash
john zip.hash --show
```

Example output:

```text
ZIP.zip/customers.csv:1234
```

---

# Identifying OpenSSL Encrypted GZIP Files

Sometimes files appear normal but are encrypted with OpenSSL.

Check file type:

```bash
file GZIP.gzip
```

Example output:

```text
openssl enc'd data with salted password
```

---

# Brute Forcing OpenSSL Encrypted GZIP

When hash extraction is unreliable, direct brute force may be required.

Example loop using rockyou:

```bash
for i in $(cat rockyou.txt); do
    openssl enc -aes-256-cbc -d -in GZIP.gzip -k $i 2>/dev/null | tar xz
done
```

Notes:

- errors are expected during attempts
- successful password results in extracted archive
- check output directory after completion

```bash
ls
```

---

# BitLocker Overview

BitLocker is Microsoft's full disk encryption solution.

Characteristics:

- AES encryption (128 or 256 bit)
- used on Windows systems
- protects full disk or virtual drives (.vhd)
- commonly used in enterprise environments

Authentication methods:

- password
- PIN
- recovery key (48-digit)

Recovery keys are typically impractical to brute force.

Focus is usually on cracking the user password.

---

# Extract BitLocker Hash

Use bitlocker2john:

```bash
bitlocker2john -i Backup.vhd > backup.hashes
```

Filter usable hash:

```bash
grep "bitlocker\$0" backup.hashes > backup.hash
```

Example hash:

```text
$bitlocker$0$16$02b329c0453b9273f2fc1b927443b5fe$1048576$12$00b0a67f961dd80103000000$60$d59f37e70696f7eab6b8f95ae93bd53f3f7067d5e33c0394b3d8e2d1fdb885cb86c1b978f6cc12ed26de0889cd2196b0510bbcd2a8c89187ba8ec54f
```

---

# Crack BitLocker Hash with Hashcat

Hashcat mode for BitLocker:

```text
-m 22100
```

Example:

```bash
hashcat -a 0 -m 22100 backup.hash /usr/share/wordlists/rockyou.txt
```

Example result:

```text
Recovered........: 1/1
Password.........: 1234qwer
Speed............: 25 H/s
```

Important:

BitLocker cracking is slow due to:

- strong AES encryption
- high iteration count
- key stretching
- large keyspace

---

# Mounting BitLocker VHD (Windows)

Double-click VHD file:

```text
Backup.vhd
```

Windows prompts for password.

Once unlocked, drive becomes accessible.

---

# Mounting BitLocker VHD (Linux)

Install dislocker:

```bash
sudo apt-get install dislocker
```

Create mount directories:

```bash
sudo mkdir -p /media/bitlocker
sudo mkdir -p /media/bitlockermount
```

Attach VHD as loop device:

```bash
sudo losetup -f -P Backup.vhd
```

Identify partition:

```bash
sudo fdisk -l
```

Decrypt volume:

```bash
sudo dislocker /dev/loop0p2 -uPASSWORD -- /media/bitlocker
```

Mount decrypted filesystem:

```bash
sudo mount -o loop /media/bitlocker/dislocker-file /media/bitlockermount
```

Browse files:

```bash
cd /media/bitlockermount
ls -la
```

Unmount after analysis:

```bash
sudo umount /media/bitlockermount
sudo umount /media/bitlocker
```

---

# General Archive Cracking Workflow

## Step 1 — identify archive

Common extensions:

```text
.zip
.rar
.7z
.gz
.tar
.tar.gz
.vhd
.vhdx
.iso
```

## Step 2 — extract hash

```bash
zip2john
rar2john
7z2john
bitlocker2john
```

## Step 3 — crack hash

```bash
john --wordlist wordlist.txt hash.txt
```

or

```bash
hashcat -a 0 -m <mode> hash.txt wordlist.txt
```

## Step 4 — access extracted data

- credentials
- configs
- keys
- documents
- backup files

---

# Why Archives Are Valuable Targets

Archives often contain:

- database dumps
- backup credentials
- configuration exports
- private keys
- sensitive internal data
- offline password storage

Common real-world scenarios:

- backup.zip on file share
- dev_backup.7z on web server
- credentials.rar on desktop
- encrypted VHD backup from sysadmin
- exported KeePass database

---

# Performance Expectations

Relative cracking difficulty:

| Target | Difficulty |
|-------|-----------|
| ZIP weak encryption | low |
| PDF | medium |
| Office docs | medium |
| RAR5 | high |
| 7zip AES | high |
| BitLocker | very high |

Wordlist quality is critical.

---

# Quick Target Checklist

Search for:

```text
*.zip
*.rar
*.7z
*.tar
*.gz
*.vhd
*.vhdx
*.iso
*.backup
*.bak
*.old
*.save
*.kdbx
```

---

# Key Takeaways

- archives frequently contain high-value data
- many archive formats allow hash extraction
- BitLocker cracking is slow but possible
- wordlist quality determines success probability
- always check backup files during post-exploitation
- extracted archives often lead to credential reuse

---

# Tags

#password-attacks #archives #bitlocker #hashcat #john #post-exploitation #credential-access #oscp