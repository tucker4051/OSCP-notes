# Password Attacks — Hashcat

## Overview

Hashcat is a powerful password cracking tool designed to recover plaintext passwords from hashes using a variety of attack modes.

It is widely used because it supports:

- dictionary attacks
- rule-based attacks
- mask attacks
- brute-force attacks
- hybrid attacks
- combinator attacks
- GPU acceleration
- hundreds of hash formats

Hashcat is often the preferred choice when:

- speed matters
- GPU acceleration is available
- the hash type is known
- structured password guesses are possible

---

# Basic Syntax

General syntax:

```bash
hashcat -a <attack_mode> -m <hash_mode> <hashes> [wordlist, rule, mask, ...]
```

Breakdown:

| Option | Purpose |
|--------|---------|
| `-a` | attack mode |
| `-m` | hash mode / hash type |
| `<hashes>` | single hash or file containing hashes |
| extra arguments | depends on selected attack mode |

Example:

```bash
hashcat -a 0 -m 0 hashes.txt /usr/share/wordlists/rockyou.txt
```

---

# Hash Modes

Hashcat supports hundreds of different hash types.

Each hash type is assigned a numeric identifier.

Show hash modes:

```bash
hashcat --help
```

Examples:

| Hash Mode | Type |
|----------|------|
| `0` | MD5 |
| `100` | SHA1 |
| `1400` | SHA256 |
| `1700` | SHA512 |
| `1000` | NTLM |
| `5600` | NetNTLMv2 |
| `1800` | sha512crypt |
| `500` | md5crypt |
| `13100` | Kerberos 5 TGS-REP |
| `22000` | WPA-PBKID-PMKID+EAPOL |

Correctly identifying the hash mode is critical.

---

# Identifying Hash Types

If the hash type is unknown, tools like `hashid` can help.

Example:

```bash
hashid -m '$1$FNr44XZC$wQxY6HHLrgrGX0e1195k.1'
```

Output may show:

```text
MD5 Crypt [Hashcat Mode: 500]
Cisco-IOS(MD5) [Hashcat Mode: 500]
FreeBSD MD5 [Hashcat Mode: 500]
```

Important note:

context matters.

A hash alone may match multiple possible formats, so consider:

- where the hash came from
- surrounding file structure
- OS or application context
- known naming conventions

---

# Attack Modes

Hashcat supports many attack modes, but the most common for pentesting are:

- dictionary attack
- mask attack

---

# Dictionary Attack (`-a 0`)

Dictionary mode tests each candidate from a wordlist against the supplied hash.

Syntax:

```bash
hashcat -a 0 -m <hash_mode> <hashes> <wordlist>
```

Example:

```bash
hashcat -a 0 -m 0 e3e3ec5831ad5e7288241960e5d4fdb8 /usr/share/wordlists/rockyou.txt
```

Use dictionary mode when:

- passwords are likely human-created
- common password reuse is expected
- you have a good wordlist
- you want a fast first attempt

---

## Why Dictionary Attacks Matter

Most users do not create fully random passwords.

They often use:

- common words
- names
- company terms
- seasons / years
- simple patterns

This makes dictionary attacks highly effective.

---

# Rule-Based Dictionary Attack

A plain wordlist often is not enough.

Rules allow Hashcat to modify candidates automatically, for example:

- add digits
- capitalize letters
- use leetspeak substitutions
- append symbols
- duplicate characters

Example:

```bash
hashcat -a 0 -m 0 1b0556a75770563578569ae21392630c /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

This is one of the most useful real-world attack patterns.

---

## Built-In Rule Files

Default rules are usually found in:

```bash
/usr/share/hashcat/rules
```

List them:

```bash
ls -l /usr/share/hashcat/rules
```

Common useful rules:

- `best64.rule`
- `rockyou-30000.rule`
- `dive.rule`
- `generated.rule`
- `leetspeak.rule`

---

## When to Use Rules

Use rules when:

- a wordlist-only attack failed
- you suspect password mutation
- passwords likely include years / digits / symbols
- you want better coverage without going full brute force

---

# Mask Attack (`-a 3`)

Mask mode is a structured brute-force attack.

Instead of guessing blindly, you define the password pattern.

Example scenario:

- password starts with uppercase letter
- followed by 4 lowercase letters
- then 1 digit
- then 1 symbol

Mask:

```text
?u?l?l?l?l?d?s
```

Attack:

```bash
hashcat -a 3 -m 0 1e293d6912d074c0fd15844d803400dd '?u?l?l?l?l?d?s'
```

---

## Built-In Mask Symbols

| Symbol | Charset |
|--------|---------|
| `?l` | lowercase letters |
| `?u` | uppercase letters |
| `?d` | digits |
| `?h` | lowercase hex |
| `?H` | uppercase hex |
| `?s` | symbols |
| `?a` | all printable ASCII |
| `?b` | full byte range |

---

## Custom Charsets

Custom charsets can be defined with:

- `-1`
- `-2`
- `-3`
- `-4`

Then referenced using:

- `?1`
- `?2`
- `?3`
- `?4`

Example:

```bash
hashcat -a 3 -m 1000 hashes.txt -1 ?l?d '?1?1?1?1?1?1'
```

This tries six-character passwords made from lowercase letters and digits.

---

## When to Use Mask Attacks

Mask attacks are useful when you know or suspect password structure, such as:

- fixed length
- known suffixes
- standard complexity policy
- company password format
- seasonal password patterns

Examples:

- `Spring2024!`
- `Password1!`
- `Admin123`
- `Capital letter + lowercase + digits`

---

# Performance Considerations

Hashcat is very fast, especially with GPU support, but attack size grows quickly.

The size of the search space depends on:

- character set
- password length
- attack mode
- rules used

Mask attacks can become huge if the pattern is too broad.

To optimize:

- keep masks as narrow as possible
- use rules before brute force
- use context to guide attacks
- start with likely formats first

---

# Typical Workflow

A practical cracking workflow is usually:

1. identify the hash type
2. choose the correct hash mode
3. try a dictionary attack
4. try dictionary + rules
5. use mask attack for likely patterns
6. escalate to broader brute-force only if justified

---

# Example Workflow

## Step 1 — Identify Hash

```bash
hashid -m hash.txt
```

## Step 2 — Dictionary Attack

```bash
hashcat -a 0 -m 0 hash.txt /usr/share/wordlists/rockyou.txt
```

## Step 3 — Dictionary + Rules

```bash
hashcat -a 0 -m 0 hash.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

## Step 4 — Mask Attack

```bash
hashcat -a 3 -m 0 hash.txt '?u?l?l?l?l?d?s'
```

---

# Status Output

Hashcat provides detailed runtime information such as:

- status
- speed
- recovered count
- progress
- candidate range
- estimated time
- hardware utilization

Important fields:

| Field | Meaning |
|-------|---------|
| `Status` | Running / Cracked / Exhausted |
| `Hash.Mode` | selected hash mode |
| `Recovered` | how many hashes cracked |
| `Progress` | guesses completed |
| `Speed` | cracking speed |
| `Candidates` | candidate range currently being tested |

---

# Common Hashcat Use Cases

Hashcat is commonly used for cracking:

- NTLM hashes
- NetNTLMv2 challenge-response
- Linux shadow hashes
- database password hashes
- web app password hashes
- archive/document passwords
- Wi-Fi captures

---

# Key Takeaways

- Hashcat is extremely fast and flexible
- correct hash mode selection is essential
- dictionary + rules is usually the best first attack
- mask attacks are powerful when password structure is partly known
- context matters when identifying unknown hashes
- avoid jumping to full brute force too early

---

# Tags

#password-attacks #hashcat #hash-cracking #rules #masks #gpu #oscp