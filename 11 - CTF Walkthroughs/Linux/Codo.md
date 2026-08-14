
## Summary

Foothold is CodoForum (a PHP forum app) exposing a version number in a leaked `README.md`, which maps to a known unrestricted file-upload RCE CVE. The published exploit script wasn't needed in the end — manually replicating the upload with a standard PHP reverse shell got the same result. Privesc is just hardcoded credentials found during enumeration, reused directly for `su root`.

**Chain:** version disclosure via README → CVE lookup → default/registered creds → manual file-upload RCE → hardcoded creds in filesystem → `su root`

---

## Enumeration

### Nmap

Only two ports open:

- `22/tcp` — OpenSSH 8.2p1 Ubuntu 4ubuntu0.7
- `80/tcp` — Apache 2.4.41, page title "All topics | CODOLOGIC"

Small attack surface (just SSH + web) usually means the web app itself is the whole game.

### Dirsearch

With only a web server to work with, ran Dirsearch with common web extensions:

```bash
dirsearch -e php,aspx,jsp,html,js -u http://$IP/
```

**Notable hits:**

- `/admin/`, `/admin/login.php` — admin panel
- `/cache/` — nothing interesting inside
- `/README.md` — 24KB, worth reading in full
- `/sites` — CMS-style directory structure (hints this is a known open-source app, not custom code)

**Pattern to remember:** always fetch and actually read any exposed `README.md`, `CHANGELOG`, `package.json`, or similar metadata file. These routinely leak the exact software version, which turns "unknown PHP app" into "known CVE" in one step.

### Identifying the app + version

The README identified the app as **CodoForum**, version ~5.2. Searching for CVEs against CodoForum turned up:

- [CodoForum v5.1 — Unrestricted File Upload / RCE — CVE-2022-31854 (Exploit-DB)](https://www.exploit-db.com/exploits/) — file-upload vulnerability leading to remote code execution

**Pattern to remember:** identify app + version from any exposed metadata → search `<app> <version> CVE` → check Exploit-DB/GitHub for a public PoC before trying anything manual.

---

## Exploitation

### Getting an authenticated session

The CVE requires an authenticated user to reach the vulnerable upload functionality. Two options both worked here:

- Self-register a new account through the forum's normal sign-up flow
- Default credentials `admin:admin` also worked on the admin login

**Pattern to remember:** for any CVE that needs "authenticated user" — don't assume that's a hard blocker. Self-registration is often open, and default/admin creds are always worth a quick try before writing the requirement off as unmet.

### When the public exploit script isn't reliable

The public Exploit-DB script for CVE-2022-31854 didn't behave as expected here. Rather than fighting the script, the underlying vulnerability class (unrestricted file upload) was replicated manually:

1. Generated a PHP reverse shell payload using [revshells.com](https://www.revshells.com/) (effectively the classic pentestmonkey PHP reverse shell, just via a convenient generator).
2. Uploaded the payload through the forum's file/avatar/attachment upload feature — the same upload path the CVE targets.
3. The uploaded file lands in a predictable path under the site's asset directory:

```
http://<target>/sites/default/assets/img/attachments/payload.php
```

4. Set up a listener and requested that URL directly to trigger the reverse shell:

```bash
nc -lvnp <port>
curl http://<target>/sites/default/assets/img/attachments/payload.php
```

**Pattern to remember:** a public exploit script is just one implementation of an underlying bug. When the script fails, go back to the CVE description/advisory, understand _what the actual vulnerability is_ (here: unrestricted upload + predictable upload path), and reproduce it manually with a generic tool (revshells.com, pentestmonkey shells, msfvenom). This is often faster than debugging someone else's script.

---

## Privilege Escalation

### Enumeration on the box

After stabilizing the shell (`python3 -c 'import pty; pty.spawn("/bin/bash")'` or similar), basic enumeration of the filesystem turned up **hardcoded credentials** — likely in a config file or setup script belonging to the app.

```
Username: codo
Password: FatPanda123
```

### Credential reuse straight to root

Tried the discovered credentials directly against `su root` — no additional privesc technique needed.

```bash
su root
# password: FatPanda123
```

Worked immediately.

**Pattern to remember:** always grep the filesystem for credentials after landing a shell — config files, `.env`, setup/install scripts, and database connection strings are common leak points. Then try any found credentials against `su`, SSH, and any other login prompt on the box before reaching for a "real" privesc technique. Sometimes the whole privesc step is just password reuse.

---

## Lessons / Transferable Techniques

1. **Read exposed metadata files fully** (`README.md`, changelogs). Version disclosure there is often the fastest route from "web app" to "known CVE."
2. **Authentication requirements on a CVE aren't automatically a blocker.** Try self-registration and default creds first.
3. **When a public exploit script fails, fall back to understanding the vulnerability class and reproducing it manually.** Generic tools (revshells.com, pentestmonkey PHP shell, msfvenom) cover most upload-based RCE scenarios.
4. **Grep for hardcoded credentials after landing a shell**, and try them everywhere — `su`, SSH, any other panel. Not every privesc needs a kernel exploit or GTFOBins; sometimes it's just reused passwords.
