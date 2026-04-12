# Password Attacks — Linux Authentication, passwd, shadow, and opasswd

## Overview

Linux systems support multiple authentication mechanisms, but one of the most common is **PAM** (Pluggable Authentication Modules).

PAM handles:

- user authentication
- password validation
- session management
- account restrictions
- password changes

When a user authenticates or changes a password, PAM modules such as `pam_unix.so` are involved in reading and updating credential-related files.

The most important files for local Linux authentication are:

- `/etc/passwd`
- `/etc/shadow`
- `/etc/security/opasswd`

Understanding how these files work is useful for:

- post-exploitation credential gathering
- password cracking
- identifying misconfigurations
- spotting weak or legacy hashing algorithms
- recovering old password material

---

# PAM

## Overview

PAM stands for:

```text
Pluggable Authentication Modules
```

It provides a modular authentication framework used by Linux services and local login mechanisms.

Typical PAM modules on Debian-based systems are found under:

```text
/usr/lib/x86_64-linux-gnu/security/
```

Examples include:

- `pam_unix.so`
- `pam_unix2.so`

PAM is used by many components, including:

- local password logins
- `passwd`
- `su`
- `sudo`
- LDAP integrations
- Kerberos authentication
- mount/auth-related services

---

# /etc/passwd

## Overview

The `/etc/passwd` file contains a record for every user on the system.

It is typically:

- world-readable
- structured
- essential for account identification

Each entry has seven colon-separated fields.

Example:

```text
htb-student:x:1000:1000:,,,:/home/htb-student:/bin/bash
```

---

## Field Breakdown

| Field | Example |
|------|---------|
| Username | `htb-student` |
| Password | `x` |
| User ID (UID) | `1000` |
| Group ID (GID) | `1000` |
| GECOS | `,,,` |
| Home Directory | `/home/htb-student` |
| Default Shell | `/bin/bash` |

---

## Password Field in /etc/passwd

This field is especially important.

Common values:

| Value | Meaning |
|------|---------|
| `x` | password hash is stored in `/etc/shadow` |
| actual hash | legacy / insecure setup |
| empty | no password required |

Modern systems usually use:

```text
x
```

meaning the actual hash is stored in `/etc/shadow`.

---

## Dangerous Misconfiguration: Writable /etc/passwd

If `/etc/passwd` is writable by mistake, an attacker may be able to modify user entries.

Example dangerous root entry:

```text
root::0:0:root:/root:/bin/bash
```

Notice the empty password field.

This can allow:

```bash
su
```

with no password prompt.

This is rare, but high impact.

---

# /etc/shadow

## Overview

The `/etc/shadow` file stores password hashes and password policy metadata.

Unlike `/etc/passwd`, it is normally readable only by privileged users.

If a user exists in `/etc/passwd` but has no valid `/etc/shadow` entry, that account may be unusable for normal local authentication.

Example entry:

```text
htb-student:$y$j9T$3QSBB6CbHEu...SNIP...f8Ms:18955:0:99999:7:::
```

---

## Field Breakdown

| Field | Meaning |
|------|---------|
| Username | user account |
| Password | hashed password |
| Last Change | days since epoch |
| Min Age | minimum password age |
| Max Age | maximum password age |
| Warning Period | days before expiry warning |
| Inactivity Period | disabled after expiry |
| Expiration Date | account expiry |
| Reserved | unused / reserved |

---

## Special Password Field Values

| Value | Meaning |
|------|---------|
| `!` | password login disabled |
| `*` | password login disabled |
| empty | no password required |

Important:

Even if password login is disabled, other auth methods may still work, such as:

- SSH keys
- Kerberos
- service-based authentication

---

# Hash Format in /etc/shadow

The password field generally follows:

```text
$<id>$<salt>$<hash>
```

Example:

```text
$6$salt$hash
```

This structure tells us:

- which algorithm is used
- what salt is present
- what hash value must be cracked

---

# Common Linux Hash IDs

| ID | Algorithm |
|----|-----------|
| `1` | MD5 |
| `2a` | Blowfish |
| `5` | SHA-256 |
| `6` | SHA-512 |
| `sha1` | SHA1crypt |
| `y` | Yescrypt |
| `gy` | Gost-yescrypt |
| `7` | Scrypt |

---

## Practical Note

Modern Linux distributions often use:

```text
y = yescrypt
```

Older systems may still use:

- MD5
- SHA-256
- SHA-512

Older / weaker hash types may be much easier to crack.

---

# /etc/security/opasswd

## Overview

PAM can prevent password reuse by storing historical password hashes.

These old password hashes are kept in:

```text
/etc/security/opasswd
```

This file is usually only readable by root.

---

## Why opasswd Matters

This file can reveal:

- previous user passwords
- old hash algorithms
- password reuse patterns
- downgrade opportunities when old hashes are weaker

Example:

```text
cry0l1t3:1000:2:$1$HjFAfYTG$qNDkF0zJ3v8ylCOrKB0kt0,$1$kcUjWZJX$E9uMSmiQeRh4pAAgzuvkq1
```

Important observation:

```text
$1$ = MD5
```

MD5-based password history entries are much weaker than modern yescrypt or SHA-512 hashes.

Even if the current password is strong, an older cracked password may reveal:

- reuse patterns
- company naming habits
- password mutation style

---

# Cracking Linux Credentials

## Why unshadow Is Needed

John the Ripper expects user context alongside the shadow hashes.

`unshadow` combines:

- `/etc/passwd`
- `/etc/shadow`

into a crackable file format.

---

## Workflow

### Step 1 — Copy files

```bash
sudo cp /etc/passwd /tmp/passwd.bak
sudo cp /etc/shadow /tmp/shadow.bak
```

### Step 2 — Combine with unshadow

```bash
unshadow /tmp/passwd.bak /tmp/shadow.bak > /tmp/unshadowed.hashes
```

### Step 3 — Crack with Hashcat or John

Example with Hashcat:

```bash
hashcat -m 1800 -a 0 /tmp/unshadowed.hashes rockyou.txt -o /tmp/unshadowed.cracked
```

---

# Hashcat Modes Commonly Relevant Here

| Mode | Hash Type |
|------|-----------|
| `500` | md5crypt |
| `7400` | sha256crypt |
| `1800` | sha512crypt |
| yescrypt | check current Hashcat support/version |

Note:

Not every Linux hash type uses the same Hashcat mode. Always verify based on the hash ID.

---

# Practical Attack Notes

## /etc/passwd Misconfigurations

Check for:

- writable `/etc/passwd`
- empty root password field
- hashes stored directly in `/etc/passwd`

---

## /etc/shadow Opportunities

Check for:

- weak hash algorithms
- disabled but interesting accounts
- empty password fields
- service accounts with valid shells
- password age / expiry clues

---

## opasswd Opportunities

Check for:

- old MD5 hashes
- reused salts / repeated patterns
- historical password reuse
- predictable mutation patterns

---

# Why This Matters in Real Assessments

These files can provide:

- recoverable credentials
- evidence of poor password hygiene
- access to reused passwords
- privilege escalation opportunities
- lateral movement opportunities if the same password is reused elsewhere

A weak old password is like an old spare key still hidden under the doormat. Even if the front door lock changed, people often keep hiding the new key in the same place.

---

# Typical Workflow

1. obtain root or sufficient privilege
2. inspect `/etc/passwd`
3. inspect `/etc/shadow`
4. inspect `/etc/security/opasswd`
5. identify weak / legacy hash types
6. combine passwd + shadow with `unshadow`
7. crack hashes offline
8. test recovered passwords for reuse

---

# Key Takeaways

- PAM manages local Linux authentication workflows
- `/etc/passwd` is world-readable and holds account metadata
- `/etc/shadow` stores password hashes and password policy data
- `/etc/security/opasswd` stores previous password hashes
- weak or legacy hashes in `opasswd` may be easier to crack
- `unshadow` is the standard way to prepare Linux hashes for John
- recovered passwords often lead to reuse wins elsewhere

---

# Related Notes

- Password Attacks — John The Ripper
- Password Attacks — Hashcat
- Password Attacks — Custom Wordlists & Rules
- Password Attacks — Dumping LSASS and Extracting In-Memory Credentials
- Privilege Escalation — Linux

---

# Tags

#password-attacks #linux #pam #passwd #shadow #opasswd #hashcat #john #oscp