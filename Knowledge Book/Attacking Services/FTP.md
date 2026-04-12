# Attacking Common Services - Attacking FTP

## Overview

**FTP (File Transfer Protocol)** is a standard network protocol used to transfer files between systems. It also supports common file and directory operations such as:

- listing files
- changing directories
- uploading files
- downloading files
- renaming files
- deleting files and folders

By default, FTP listens on **TCP port 21**.

From an attacker’s point of view, FTP is interesting because it often exposes:

- **anonymous access**
- **weak credentials**
- **misconfigured permissions**
- **sensitive files**
- **upload functionality**
- **legacy software vulnerabilities**

Many organizations have historically used FTP during:

- website development
- software deployment
- internal file exchange
- staging and publishing workflows

Because of that, FTP servers often contain useful or sensitive content.

---

## Why FTP Matters in an Assessment

Once you gain access to an FTP service, the immediate goal is usually to answer these questions:

- Can I log in anonymously?
- Can I read files?
- Can I upload files?
- Are there sensitive configs, credentials, or source code present?
- Is the service running an old or vulnerable version?
- Can the FTP server be abused to reach other systems?

FTP is one of those services where **simple access often leads to valuable data exposure**, even without exploitation.

---

# Enumeration

## Basic Nmap Enumeration

A strong starting point is:

```bash
sudo nmap -sC -sV -p 21 192.168.2.142
```

### What this does

- `-sC` runs default NSE scripts, including useful FTP checks such as `ftp-anon`
- `-sV` performs version detection
- `-p 21` targets the FTP port specifically

### Why this is useful

This can reveal:

- whether anonymous login is enabled
- directory listings exposed to anonymous users
- writable directories
- the FTP banner
- server software and version

---

## Example Output

```text
PORT   STATE SERVICE
21/tcp open  ftp
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--   1 1170     924            31 Mar 28  2001 .banner
| d--x--x--x   2 root     root         1024 Jan 14  2002 bin
| d--x--x--x   2 root     root         1024 Aug 10  1999 etc
| drwxr-srwt   2 1170     924          2048 Jul 19 18:48 incoming [NSE: writeable]
| d--x--x--x   2 root     root         1024 Jan 14  2002 lib
| drwxr-sr-x   2 1170     924          1024 Aug  5  2004 pub
|_Only 6 shown. Use --script-args ftp-anon.maxlist=-1 to see all.
```

### Important things to notice

- **Anonymous FTP login allowed**
- **directory listing exposed**
- **writable folder detected**
- possible old server layout
- potential sensitive file names such as `.banner`

### If the listing is truncated

Use:

```bash
sudo nmap -sC -sV -p 21 --script-args ftp-anon.maxlist=-1 192.168.2.142
```

This tells Nmap to show the full anonymous listing where possible.

---

# Misconfigurations

## Anonymous Authentication

A common FTP weakness is **anonymous login**.

You can usually test it with:

```bash
ftp 192.168.2.142
```

Then authenticate as:

```text
Name: anonymous
Password: <blank or any value>
```

### Example

```text
Connected to 192.168.2.142.
220 (vsFTPd 2.3.4)
Name (192.168.2.142:kali): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 0        0               9 Aug 12 16:51 test.txt
226 Directory send OK.
```

---

## Why Anonymous Access Is Dangerous

Anonymous access becomes especially risky when the anonymous user has:

- **read access** to sensitive files
- **write access** to upload folders
- **traversal access** to parent or sibling directories

This could allow an attacker to:

- download source code
- retrieve credentials or configuration files
- discover deployment files
- upload malicious files
- plant web shells if the FTP root overlaps with a web root

A writable FTP upload folder combined with a vulnerable web application can be particularly dangerous.

Example chain:

1. FTP allows anonymous upload
2. Uploaded file lands in a web-accessible location
3. A path traversal or predictable path reveals the file
4. The attacker executes it as PHP, ASPX, or another server-side script

---

# Manual FTP Interaction

Once logged in, you should treat the session like a remote file system and enumerate carefully.

## Common FTP commands

| Command | Purpose |
|---|---|
| `ls` | List files |
| `cd` | Change directory |
| `pwd` | Print working directory |
| `get` | Download one file |
| `mget` | Download multiple files |
| `put` | Upload one file |
| `mput` | Upload multiple files |
| `delete` | Delete file |
| `mkdir` | Create directory |
| `rmdir` | Remove directory |
| `help` | Show available commands |

---

## Practical Enumeration Goals

After login, look for:

- credentials
- configs
- source code
- backup files
- hidden files
- upload folders
- deployment artifacts
- web content
- scripts
- logs

### Interesting filenames

- `config`
- `backup`
- `db`
- `cred`
- `secret`
- `key`
- `pass`
- `.env`
- `.git`
- `.htaccess`
- `web.config`

---

# Brute Forcing FTP

## Overview

If anonymous login is disabled, FTP may still be vulnerable to credential attacks.

One option is **Medusa**.

### Example

```bash
medusa -u fiona -P /usr/share/wordlists/rockyou.txt -h 10.129.203.7 -M ftp
```

### What this does

- `-u fiona` targets a single username
- `-P` provides a password list
- `-h` specifies the host
- `-M ftp` selects the FTP module

---

## Example Result

```text
ACCOUNT FOUND: [ftp] Host: 10.129.203.7 User: fiona Password: family [SUCCESS]
```

---

## Notes on Brute Force

Classic brute forcing is often less effective today because many services now implement:

- account lockout
- rate limiting
- IDS/IPS monitoring
- failed login alerts

Because of that, **password spraying** is often the safer and more realistic technique in enterprise environments.

Still, brute force remains useful in:

- labs
- CTFs
- legacy systems
- badly configured internal services
- old appliances

---

# FTP Bounce Attack

## Overview

An **FTP bounce attack** abuses the FTP server’s handling of the `PORT` command to make it send traffic to another target.

In effect, the attacker uses the FTP server as a proxy to interact with another host that may not be directly reachable.

This can be useful for:

- internal port scanning
- service discovery
- reaching segmented hosts indirectly

---

## Conceptual Example

Assume:

- `FTP_DMZ` is internet-facing
- `Internal_DMZ` is not internet-facing
- the attacker can reach the FTP server
- the FTP server is misconfigured and permits bounce behaviour

The attacker may be able to use the FTP server to probe internal systems.

---

## Nmap FTP Bounce Syntax

```bash
nmap -Pn -v -n -p80 -b anonymous:password@10.10.110.213 172.17.0.2
```

### Breakdown

| Option | Meaning |
|---|---|
| `-Pn` | Skip host discovery |
| `-v` | Verbose |
| `-n` | No DNS resolution |
| `-p80` | Scan port 80 on the target |
| `-b anonymous:password@10.10.110.213` | Use the FTP server as a bounce proxy |
| `172.17.0.2` | Internal target being scanned |

---

## Example Result

```text
PORT   STATE  SERVICE
80/tcp open http
```

### Why this matters

This reveals internal service exposure without direct connectivity from the attacker to the target.

---

## Modern Reality

Most modern FTP servers block bounce attacks by default.

However, the attack may still work if:

- the server is old
- security settings are weak
- the `PORT` handling is misconfigured
- administrators disabled protections

So it is less common today, but still worth knowing.

---

# Common Attack Paths Against FTP

## 1. Anonymous Read Access

**Impact:**

- source code exposure
- backup leakage
- credential discovery
- internal file disclosure

---

## 2. Anonymous Write Access

**Impact:**

- file planting
- web shell upload
- malicious file staging
- content tampering

---

## 3. Weak Credentials

**Impact:**

- authenticated file access
- privilege escalation within the FTP hierarchy
- discovery of internal processes and deployment pipelines

---

## 4. Vulnerable FTP Software

**Impact:**

- service compromise
- remote code execution
- authentication bypass
- information disclosure

Always note the exact FTP version from banners and version detection.

---

## 5. FTP Bounce

**Impact:**

- indirect port scanning
- internal discovery
- segmentation bypass in edge cases

---

# Enumeration Priorities

When you find an FTP service, check these in order:

1. **Version and banner**
2. **Anonymous login**
3. **Directory listing exposure**
4. **Writable folders**
5. **Interesting files**
6. **Credential reuse possibilities**
7. **Upload potential**
8. **Bounce support**
9. **Known vulnerabilities in the FTP version**

---

# High-Value Findings

The following are especially valuable:

- anonymous login enabled
- writable anonymous directories
- upload directory mapped into a web root
- exposed credentials or config files
- backup archives
- source code
- outdated FTP server version
- successful credential brute force
- FTP bounce capability

---

# Practical Workflow

## 1. Scan the service

```bash
sudo nmap -sC -sV -p 21 <target>
```

## 2. Test anonymous login

```bash
ftp <target>
```

Use:

```text
anonymous
```

## 3. Enumerate files and folders

```text
ls
cd <dir>
pwd
```

## 4. Download interesting files

```text
get filename
mget *
```

## 5. Test upload capability if permitted

```text
put test.txt
```

## 6. If anonymous access fails, test credentials

```bash
medusa -u <user> -P <wordlist> -h <target> -M ftp
```

## 7. Check for bounce possibilities if relevant

```bash
nmap -Pn -v -n -p80 -b user:pass@<ftp_server> <internal_target>
```

---

# Quick Commands

## Nmap FTP Enumeration

```bash
sudo nmap -sC -sV -p 21 <target>
sudo nmap -sC -sV -p 21 --script-args ftp-anon.maxlist=-1 <target>
```

## FTP Client Login

```bash
ftp <target>
```

## Common FTP Commands

```text
ls
cd
pwd
get <file>
mget *
put <file>
mput *
help
```

## Medusa FTP Brute Force

```bash
medusa -u <user> -P /usr/share/wordlists/rockyou.txt -h <target> -M ftp
medusa -U users.txt -P passwords.txt -h <target> -M ftp
```

## FTP Bounce with Nmap

```bash
nmap -Pn -v -n -p80 -b anonymous:password@<ftp_server> <internal_target>
```

---

# Key Takeaways

- FTP is simple, but often dangerously misconfigured.
- Anonymous access should always be tested.
- Readable or writable shares can quickly lead to compromise.
- FTP upload + web exposure is a classic attack path.
- Weak credentials still appear on internal services.
- FTP bounce is less common now, but still worth understanding.
- After gaining FTP access, the real value is usually in the **files**, not the protocol itself.

---

# Tags

#attacking-common-services  
#ftp  
#enumeration  
#file-transfer  
#medusa  
#nmap  
#anonymous-login  
#misconfiguration  
#obsidian