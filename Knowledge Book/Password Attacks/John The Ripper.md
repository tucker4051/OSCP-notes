# Password Attacks — John The Ripper (JtR)

## Overview

John the Ripper (JtR or john) is a widely used password cracking tool capable of performing:

- dictionary attacks
- brute-force attacks
- rule-based attacks
- hybrid attacks
- file password cracking
- hash cracking

JtR is commonly included in most pentesting distributions.

The **jumbo version** is recommended because it includes:

- support for hundreds of hash formats
- optimized performance
- additional wordlists
- extended rule sets
- support for encrypted files (ZIP, PDF, SSH keys, etc.)

---

# How John Works

John attempts to recover plaintext passwords by:

1. taking a hash as input
2. generating candidate passwords
3. hashing candidates using the same algorithm
4. comparing results against the target hash

If hashes match:

```
password recovered
```

---

# Supported Hash Types

John supports many hash formats including:

| Category | Examples |
|----------|----------|
| Windows | NTLM, NetNTLMv2, MSCash |
| Linux | SHA512crypt, MD5crypt |
| Databases | MSSQL, MySQL, Oracle |
| Web apps | phpass, bcrypt |
| Files | ZIP, RAR, PDF |
| Network auth | Kerberos, WPA |
| SSH | private keys |

Hash format can be specified using:

```bash
--format=
```

Example:

```bash
john --format=nt hashes.txt
```

---

# Cracking Modes

John provides multiple cracking modes depending on the scenario.

---

# Single Crack Mode

Single mode generates password guesses using information about the user.

Uses:

- username
- real name
- home directory
- GECOS information

Then applies transformation rules such as:

- capitalization
- number suffixes
- character substitutions

Example passwd entry:

```
r0lf:$6$ues25dIanlctrWxg$nZHVz2z4kCy1760Ee28M1xtHdGoy0C2cYzZ8l2sVa1kIa8K9gAcdBP.GI6ng/qA4oaMrgElZ1Cb9OeXO4Fvy3/:0:0:Rolf Sebastian:/home/r0lf:/bin/bash
```

Attack:

```bash
john --single passwd
```

Typical success cases:

```
username-based passwords
company naming patterns
real-name passwords
```

Best used when:

```
passwd file available
AD user list available
real names available
```

---

# Wordlist Mode

Dictionary attack using a wordlist.

Tests each word in list against hash.

Example:

```bash
john --wordlist=rockyou.txt hashes.txt
```

Rules can be applied to modify words:

```bash
john --wordlist=rockyou.txt --rules hashes.txt
```

Rules generate variations:

```
Password
Password1
Password!
P@ssword
```

Common wordlists:

```
rockyou.txt
SecLists
custom lists
```

Best used when:

```
human-created passwords expected
common password reuse likely
```

---

# Incremental Mode

Brute-force attack using statistical modelling (Markov chains).

Generates password guesses dynamically.

Example:

```bash
john --incremental hashes.txt
```

Characteristics:

- tests all character combinations
- prioritizes likely passwords
- slow but thorough
- does not require wordlist

Custom charsets defined in:

```
/etc/john/john.conf
```

Example configuration:

```
[Incremental:ASCII]
MinLen = 0
MaxLen = 13
CharCount = 95
```

Best used when:

```
wordlist fails
password complexity unknown
time is not a constraint
```

---

# Identifying Hash Formats

Sometimes hash format is unknown.

Example hash:

```
193069ceb0461e1d40d216e32c79c704
```

Tools to identify hash type:

### hashid

```bash
hashid -j <hash>
```

Example output:

```
MD5
NTLM
RIPEMD-128
```

Context often determines correct format.

Examples:

| Context | Likely format |
|--------|--------------|
| Windows SAM | NTLM |
| Linux shadow | SHA512crypt |
| MSSQL | mssql |
| web app | md5 or bcrypt |

---

# Common Hash Formats

| Format | Description |
|--------|-------------|
| nt | Windows NTLM |
| raw-md5 | MD5 |
| raw-sha1 | SHA1 |
| raw-sha256 | SHA256 |
| bcrypt | Blowfish |
| mssql | MSSQL hashes |
| netntlmv2 | NTLM challenge-response |
| krb5 | Kerberos |
| zip | zip archives |
| rar | rar archives |
| pdf | pdf passwords |
| ssh | ssh private keys |

Example usage:

```bash
john --format=nt hashes.txt
```

---

# Cracking Password-Protected Files

JtR includes conversion tools called:

```
*2john
```

These convert files into hash format compatible with John.

General syntax:

```bash
<tool> <file> > hash.txt
```

---

# Common 2john Tools

| Tool | Purpose |
|------|---------|
| zip2john | crack ZIP files |
| rar2john | crack RAR archives |
| pdf2john | crack PDF passwords |
| ssh2john | crack SSH private keys |
| keepass2john | crack KeePass databases |
| office2john | crack MS Office files |
| pfx2john | crack PKCS#12 certificates |
| hccap2john | crack WPA handshakes |
| putty2john | crack PuTTY keys |
| truecrypt_volume2john | crack TrueCrypt volumes |

Example:

```bash
zip2john file.zip > zip.hash
john zip.hash
```

---

# Outputting Cracked Passwords

Show cracked credentials:

```bash
john --show hashes.txt
```

Example output:

```
user:password
```

---

# Session Management

Resume cracking session:

```bash
john --restore
```

Show cracking status:

```bash
john --status
```

---

# Performance Optimization

Limit password length:

```bash
--min-length=
--max-length=
```

Specify format manually:

```bash
--format=
```

Use rules:

```bash
--rules
```

Use custom wordlists:

```bash
--wordlist=
```

---

# Typical Workflow

1. obtain hash
2. identify hash format
3. attempt wordlist attack
4. apply rules
5. attempt single mode
6. attempt incremental mode
7. review cracked credentials

---

# Example Workflow

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

Brute-force fallback:

```bash
john --incremental hash.txt
```

Show results:

```bash
john --show hash.txt
```

---

# Key Takeaways

- very flexible password cracking tool
- supports hundreds of formats
- integrates with many pentesting workflows
- rule-based attacks significantly improve success rate
- 2john tools allow cracking encrypted files
- selecting correct cracking mode is critical

---

# Tags

#password-attacks #john #hash-cracking #wordlists #oscp #credential-access