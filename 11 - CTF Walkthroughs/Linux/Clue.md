## Overview

**Platform:** OffSec Proving Grounds **Operating System:** Linux (Debian) **Services:** Apache (80), Samba (139/445), Cassandra Web (3000), FreeSWITCH (8021) **Initial Access:** FreeSWITCH `mod_event_socket` command execution, using a password recovered from an SMB-mounted backup share **Initial Access Type:** Service exploitation, credential-pattern discovery via file share **Privilege Escalation:** `sudo`-permitted re-run of `cassandra-web` (a remote-file-read-vulnerable service) as root, reused against `/etc/shadow` and SSH keys **Final Access:** Root

### Key Techniques

- AutoRecon for initial broad enumeration
- Recognizing and recovering from an automated tool's missed port
- Manual re-enumeration as a stuck-point recovery strategy
- Cassandra Web remote file read (Exploit-DB)
- `/etc/passwd` enumeration for valid usernames
- `sshd_config` review to identify which users can actually SSH in
- SMB guest mount
- Bash one-liner for bulk password-hunting across a mounted share
- Directory-pattern reasoning to find an undocumented but structurally similar credential file
- FreeSWITCH command execution exploit, patched with a discovered (non-default) password
- Reading and manually replicating an exploit's HTTP request instead of running it as-is
- `sudo`-permitted re-invocation of a vulnerable local service as root
- Path traversal via raw `curl` request reproduction
- Private key theft and reuse across usernames

---

# Attack Path

```text
AutoRecon
        ↓
Ports Found: 22, 80, 139, 3000, 8021 (445 MISSED)
        ↓
Cassandra Web (3000) — Remote File Read Exploit
        ↓
/etc/passwd Enumerated — Valid Usernames Found
        ↓
sshd_config Read — Only root & anthony Permitted to SSH
        ↓
Password for 'cassie' Found via Cassandra Web RFR — Doesn't Work for SSH
        ↓
STUCK — FreeSWITCH Default Creds Fail, Hints Don't Help
        ↓
Manual Re-Scan (nmap -sV -sC) Reveals Port 445 (SMB) — AutoRecon Missed It
        ↓
SMB Guest Mount of 'backup' Share
        ↓
Bulk Password-Hunting One-Liner Across Mounted Share
        ↓
Default 'cluecon' Password Found (Doesn't Work)
        ↓
Directory-Pattern Reasoning Finds a Second, Undocumented Config Path
        ↓
Real FreeSWITCH Password Recovered
        ↓
FreeSWITCH RCE Exploit, Patched With Real Password
        ↓
Reverse Shell (Foothold)
        ↓
cassie's SSH Private Key Found on Disk
        ↓
su cassie (Using Earlier-Found Password)
        ↓
sudo -l: cassie May Re-Run /usr/local/bin/cassandra-web as Root
        ↓
Manually Replicate the Remote-File-Read Exploit as a Raw curl Request
        ↓
Read /etc/shadow, Attempt root's SSH Key (Fails/Missing)
        ↓
Read anthony's SSH Private Key Instead
        ↓
SSH as anthony Fails, but Same Key Works for root
        ↓
Root Shell
```

---

# 1. Initial Enumeration

```bash
sudo autorecon <target>
```

Served the results over a quick local web server for easier browsing:

```bash
python3 -m http.server 80
```

**Ports found (per AutoRecon):**

```text
22    ssh
80    http (403 Forbidden)
3000  http — Cassandra Web
8021  freeswitch-event
```

> [!warning] Automated Recon Tools Can Miss Ports AutoRecon did not detect port `445` (SMB) on this box — a gap that became a significant roadblock later, since the SMB share turned out to hold the credential needed to move forward. No automated tool's port coverage should be treated as complete; if enumeration stalls, re-running a plain, explicit `nmap -sV -sC` scan (or a full `-p-` sweep) is worth doing even after a "thorough" automated tool has already run.

---

# 2. Cassandra Web — Remote File Read

Port `3000` runs **Cassandra Web**, with a known remote file read vulnerability:

- [Exploit-DB — Cassandra Web 0.5.0 Remote File Read](https://www.exploit-db.com/exploits/) (search "Cassandra Web 0.5.0")

Used it to read `/etc/passwd`, confirming valid system usernames, including `cassie`.

The Exploit-DB page's notes pointed toward a specific directory worth checking for leaked passwords — following that lead recovered:

```text
cassie:SecondBiteTheApple330
```

**This password did not work for SSH.** Checking `sshd_config` explains why:

```text
AllowUsers root anthony
```

Only `root` and `anthony` are permitted to authenticate via SSH at all — `cassie`'s valid password is simply not usable for that particular login path, regardless of correctness.

> [!tip] Recognition Pattern A correct password that "doesn't work" for a specific service is worth checking against that service's own access-control configuration before assuming the credential itself is wrong. `sshd_config`'s `AllowUsers`/`DenyUsers` directives can silently reject an otherwise-valid login.

---

# 3. Hitting a Roadblock

Port `8021` (FreeSWITCH) looked like the natural next target, but:

- The commonly-referenced default FreeSWITCH password (`ClueCon`) didn't work.
- A known FreeSWITCH credential-leak directory path (from public research) didn't contain anything useful on this box.
- The lab's own provided hint didn't immediately resolve things either.

At this point, genuinely stuck — worth documenting as its own lesson rather than glossing over.

> [!tip] Key Principle Getting stuck on a Hard-rated box is normal, not a sign of doing something wrong. The recovery move that worked here wasn't a clever new technique — it was **going back to basics and re-enumerating from scratch**, this time manually rather than relying on the same automated tool that had already run once. When stuck, questioning whether your enumeration was actually complete (rather than just re-trying the same exploitation ideas repeatedly) is often the more productive path.

---

# 4. Re-Enumeration — Finding the Missed Port

```bash
sudo nmap -sV -sC <target>
```

```text
22/tcp    ssh     OpenSSH 7.9p1 Debian
80/tcp    http    Apache 2.4.38 (403 Forbidden)
139/tcp   smb     Samba smbd 3.X-4.X
445/tcp   smb     Samba smbd 4.9.5-Debian
3000/tcp  http    Cassandra Web (Thin httpd)
8021/tcp  freeswitch-event  FreeSWITCH mod_event_socket
```

**Port 445 — completely absent from the original AutoRecon output.** This directly unblocks the rest of the chain.

---

# 5. SMB Enumeration

```bash
mkdir backup
mount -t cifs //<target>/backup backup/ -o guest
```

A `backup` share, mountable as guest — matching the `/backup` directory gobuster had already found (but couldn't access) on port 80, strongly suggesting this share backs that same web content.

## Bulk password-hunting

Used a bash one-liner (sourced from public research on searching mounted shares for credential-shaped strings) to sweep the mounted share for anything password-related — the same general technique as `grep -rinE 'password|user|pass|key|...'` used elsewhere, applied here across a full mounted filesystem rather than just a handful of downloaded files.

This surfaced a known FreeSWITCH config path containing the **default** `cluecon` password — already known not to work.

## Pattern-based reasoning to find the real credential

Noticing the found config file's path had a distinct, recognizable structure (matching a directory layout referenced in public FreeSWITCH research), reasoned that a **structurally similar directory** might hold a second, non-default config — and it did, containing the actual working password for this specific box.

> [!tip] Key Principle When a known "leaky" file/path pattern only yields default/placeholder values, don't stop there — the same _directory structure_ is worth probing for sibling files that might hold the environment-specific real values. Public research often documents the canonical leaky path; box-specific customizations sometimes live one directory over.

---

# 6. Exploitation — FreeSWITCH Command Execution

```text
FreeSWITCH 1.10.1 - Command Execution (Exploit-DB)
```

The public exploit defaults to trying the well-known `ClueCon` password, which fails on this box. Edited the exploit script directly to substitute the **actual recovered password**.

```bash
python3 exploit.py --rhost=<target> --password=<recovered_password>
```

Verified working with a read of `/etc/passwd`, then pivoted to a full reverse shell — matching the listener port to the one Cassandra Web was hosted on for convenience.

```bash
nc -lvnp <port>
```

Reverse shell received.

> [!tip] Recognition Pattern Public exploits with a hardcoded default credential often fail silently or with an unhelpful error against boxes that changed the default — always check whether a script accepts a credential override, and if not, patch the source directly rather than assuming the exploit doesn't apply to the target.

---

# 7. Local Enumeration and User Pivot

Found `cassie`'s SSH private key sitting readable on disk in her home directory. Since the earlier-found password (`SecondBiteTheApple330`) was valid for the _account_ even though it couldn't be used over SSH directly:

```bash
su cassie
# Password: SecondBiteTheApple330
```

Successfully switched to `cassie` locally (not remotely — `AllowUsers` only restricts SSH, not local `su`), and retrieved her private key from there.

---

# 8. Privilege Escalation

```bash
sudo -l
```

`cassie` may run **`/usr/local/bin/cassandra-web`** as an elevated user — effectively permission to launch another instance of the same Cassandra Web application, but running with elevated privileges this time.

Since Cassandra Web itself is vulnerable to the same **remote file read** bug already used for the initial foothold, running a privileged instance of it and exploiting _that_ instance turns the vulnerability into a **root-privileged arbitrary file read**.

## Manually replicating the exploit

Attempting to transfer and run the original exploit script directly against the newly-spawned local instance didn't work as expected. Rather than debugging the script, read through its logic and found the actual technique was simple enough to reproduce as a raw HTTP request:

```bash
curl --path-as-is http://0.0.0.0:4444/../../../../../../../../../<file>
```

Just a path-traversal GET request — the exploit script's complexity was mostly wrapper code around this one core request.

> [!tip] Key Principle Reading and understanding a public exploit thoroughly, rather than treating it as an opaque tool, pays off directly here — once the underlying mechanism (a raw path-traversal GET request) was clear, replicating it with `curl --path-as-is` was faster and more reliable than fighting with the original script's transfer/execution assumptions.

**Reading `/etc/shadow`:**

```bash
curl --path-as-is http://0.0.0.0:4444/../../../../../../../../../etc/shadow
```

**Attempting root's SSH private key** — not present or not accessible. Pivoted instead to reading **`anthony`'s** private key (recall: `anthony` was one of the two users permitted by `sshd_config` to SSH in at all):

```bash
curl --path-as-is http://0.0.0.0:4444/../../../../../../../../../home/anthony/.ssh/id_rsa
```

## The final quirk

```bash
ssh -i anthony_id_rsa anthony@<target>
# Failed
ssh -i anthony_id_rsa root@<target>
# Worked!
```

The stolen private key, despite being retrieved from `anthony`'s home directory, actually authenticated successfully as **root** rather than `anthony` — suggesting either a shared/reused key pair between the two accounts, or the key file's actual association didn't match its filesystem location.

```bash
id
# uid=0(root)
```

> [!tip] Recognition Pattern An SSH private key's location on disk (whose home directory it was found in) doesn't guarantee which account it actually authenticates as — a key pair can be shared or reused across accounts. If a stolen key fails against the "obvious" matching username, it's worth trying it against other permitted accounts (per `sshd_config`'s `AllowUsers`) before concluding the key is useless.

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|Automated recon tool (AutoRecon, etc.) completes|Don't treat its port list as final — re-run a manual `nmap -sV -sC` if enumeration stalls|
|A correct password fails for a specific login|Check that service's own access-control config (`sshd_config`'s `AllowUsers`, etc.) before doubting the credential|
|A known "leaky" config path only has default/placeholder values|Probe structurally similar sibling paths for environment-specific real values|
|Public exploit uses a hardcoded default credential|Check for a credential-override flag, or patch the script directly|
|A public exploit script fails to run against a variant target|Read its actual request/logic and consider replicating manually (`curl`, raw sockets)|
|`sudo -l` permits re-running a vulnerable local service|The same vulnerability can often be reused against the privileged instance|
|A stolen credential/key doesn't work for the account it was found under|Try it against other accounts permitted to log in, per server config|

---

# Key Lessons

> [!tip] No Automated Tool's Enumeration Is Guaranteed Complete AutoRecon missing SMB entirely on this box was the single biggest roadblock in the whole chain. When stuck, re-running enumeration manually — even after "thorough" automated tooling — is a legitimate and often necessary recovery step.

> [!tip] A Failed Login Doesn't Always Mean a Wrong Credential Server-side access restrictions (`AllowUsers`, similar allowlists) can reject a completely correct password. Check the service's own config before discarding a credential as invalid.

> [!tip] Leaky Config Paths Have Siblings When a known-leaky file only yields a default/placeholder value, check nearby or structurally similar paths — box-specific real values sometimes live in a parallel location to the documented one.

> [!tip] Understand Exploits Well Enough to Replicate Them When a public exploit script doesn't work cleanly against a variant of the target (e.g. a locally-relaunched service on a different port), reading its core logic and reproducing the underlying request manually is often faster than debugging the script itself — and builds a much better understanding of the actual vulnerability.

> [!tip] A Sudo-Permitted Re-Run of a Vulnerable Service Is a Direct Privilege Multiplier If a service with a known vulnerability can be relaunched with elevated privileges via sudo, the original exploit technique often still applies — just against the new, more-privileged instance.

> [!tip] Getting Stuck Is Normal — Re-Enumerate Rather Than Force It The biggest unlock on this box wasn't a clever exploit — it was going back and questioning whether enumeration had actually been thorough, rather than continuing to hammer the same exploitation attempts.