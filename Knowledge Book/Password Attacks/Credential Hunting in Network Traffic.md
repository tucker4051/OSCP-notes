# Password Attacks - Credential Hunting in Network Traffic

## Overview

In modern environments, most applications use **TLS encryption** to protect credentials in transit. However, plaintext protocols still appear in:

- legacy systems
- misconfigured services
- internal lab environments
- development or staging servers
- poorly secured embedded devices
- internal monitoring traffic
- outdated applications

Whenever traffic is not encrypted, attackers may be able to recover:

- usernames
- passwords
- session cookies
- authentication tokens
- NTLM hashes
- API keys
- SNMP community strings

Credential hunting in network traffic is therefore a valuable technique during:

- internal penetration tests
- red team engagements
- CTF environments
- assumed breach scenarios
- post-exploitation pivoting

In this note we cover:

- identifying plaintext protocols
- extracting credentials using Wireshark filters
- automating extraction using Pcredz

---

# Plaintext vs Encrypted Protocols

Historically, many protocols transmitted authentication data in cleartext.

Modern equivalents usually support encryption, but insecure configurations still occur.

| Unencrypted Protocol | Encrypted Counterpart | Description |
|---|---|---|
| HTTP | HTTPS | Transfers web content |
| FTP | FTPS / SFTP | File transfer |
| SNMP | SNMPv3 | Network device monitoring |
| POP3 | POP3S | Email retrieval |
| IMAP | IMAPS | Mailbox access |
| SMTP | SMTPS | Email sending |
| LDAP | LDAPS | Directory services |
| RDP | RDP with TLS | Remote desktop |
| DNS | DoH / DoT | Domain resolution |
| SMB | SMB3 with TLS | File sharing |
| VNC | VNC with TLS | Remote desktop control |

### Why this matters

If encryption is not enabled, credentials may appear directly inside packet captures.

Example risks:

- HTTP login forms transmitting passwords in cleartext
- FTP credentials visible in network captures
- SNMP community strings exposed
- NTLM authentication hashes captured over SMB
- LDAP binds exposing usernames

---

# Wireshark

## Overview

Wireshark is a packet capture and analysis tool commonly included in penetration testing distributions.

It allows:

- live traffic inspection
- PCAP file analysis
- filtering specific protocols
- searching for strings inside packets
- reconstructing sessions

Think of Wireshark as a microscope for network traffic.

---

## Useful Wireshark Filters

Wireshark filters help narrow down traffic quickly.

| Filter | Purpose |
|---|---|
| ip.addr == 56.48.210.13 | Traffic involving a specific IP |
| tcp.port == 80 | Traffic on port 80 (HTTP) |
| http | HTTP traffic only |
| dns | DNS queries |
| icmp | Ping traffic |
| tcp.flags.syn == 1 && tcp.flags.ack == 0 | Detect connection attempts |
| http.request.method == "POST" | POST requests (login forms) |
| tcp.stream eq 53 | Specific TCP conversation |
| eth.addr == 00:11:22:33:44:55 | Filter by MAC address |
| ip.src == 192.168.1.10 && ip.dst == 192.168.1.20 | Traffic between hosts |

---

## Identifying Credential Data in HTTP

Many login forms send credentials via POST requests.

Filter for POST traffic:

```
http.request.method == "POST"
```

Search for password strings:

```
http contains "pass"
```

Other useful search strings:

```
http contains "user"
http contains "login"
http contains "token"
http contains "Authorization"
```

---

## Searching for Strings in Wireshark

You can search packet contents using:

### Display filter

```
http contains "passw"
```

### Manual search

```
Edit → Find Packet
```

Search examples:

- password
- passwd
- pwd
- login
- token
- auth
- key
- session
- bearer

---

## Tracking Sessions

To follow a conversation:

```
tcp.stream eq 5
```

Then:

```
Right-click → Follow → TCP Stream
```

This reconstructs full communication between client and server.

Useful when credentials are split across multiple packets.

---

## Protocols commonly exposing credentials

### HTTP

Credentials often appear in POST bodies:

```
username=admin&password=password123
```

### FTP

FTP transmits credentials in plaintext:

```
USER admin
PASS password123
```

### SNMP

Community strings may act as passwords:

```
public
private
monitor
admin
```

### SMTP / POP3 / IMAP

Email credentials may appear in authentication commands.

### SMB / NTLM

Hashes may appear during authentication handshakes.

---

# Pcredz

## Overview

Pcredz automates credential extraction from PCAP files or live traffic.

It identifies credentials and hashes across many protocols.

Supported extractions include:

- credit card numbers
- POP credentials
- SMTP credentials
- IMAP credentials
- SNMP community strings
- FTP credentials
- HTTP Basic authentication
- NTLM hashes
- Kerberos hashes

---

## Installing Pcredz

Clone repository:

```
git clone https://github.com/lgandx/PCredz
cd PCredz
```

Alternatively use the Docker container described in the repository README.

---

## Running Pcredz against a capture file

```
./Pcredz -f demo.pcapng -t -v
```

### Parameter explanation

| Option | Meaning |
|---|---|
| -f | input pcap file |
| -t | include timestamps |
| -v | verbose output |

---

## Example output

```
protocol: udp 192.168.31.211:59022 > 192.168.31.238:161
Found SNMPv2 Community string: secret

protocol: tcp 192.168.31.243:55707 > 192.168.31.211:21
FTP User: admin
FTP Pass: password123
```

---

## Why Pcredz is useful

Instead of manually inspecting thousands of packets, Pcredz:

- extracts credentials automatically
- identifies hashes for offline cracking
- highlights weak protocols quickly
- saves significant analysis time

---

# Practical Workflow

When analysing network traffic:

### Step 1 — Identify plaintext protocols

Look for:

- HTTP
- FTP
- Telnet
- SMTP
- POP3
- IMAP
- SNMP
- LDAP
- SMB

---

### Step 2 — Filter interesting traffic

Examples:

```
http
ftp
ldap
smb
dns
```

---

### Step 3 — Search for credential keywords

Examples:

```
http contains "pass"
http contains "user"
http contains "token"
http contains "auth"
```

---

### Step 4 — Follow TCP streams

Reconstruct sessions:

```
Follow TCP stream
```

Look for:

- login requests
- session cookies
- tokens
- credentials
- API keys

---

### Step 5 — Run automated extraction tools

Use:

- Pcredz
- tshark
- custom grep pipelines

---

# High-Value Findings

Credential hunting in traffic may reveal:

- plaintext passwords
- NTLM hashes
- Kerberos tickets
- SNMP community strings
- API tokens
- session cookies
- internal hostnames
- authentication headers
- basic auth credentials
- bearer tokens

---

# Quick Filters

## HTTP credentials

```
http.request.method == "POST"
http contains "password"
http contains "login"
```

## FTP credentials

```
ftp
```

## SNMP community strings

```
snmp
```

## DNS reconnaissance

```
dns
```

## Identify scanning activity

```
tcp.flags.syn == 1 && tcp.flags.ack == 0
```

## Track host communication

```
ip.src == 192.168.1.10 && ip.dst == 192.168.1.20
```

---

# Quick Commands

## Run Pcredz

```
./Pcredz -f capture.pcapng -t -v
```

## Basic Wireshark filters

```
http
ftp
dns
icmp
```

## Search credentials in HTTP

```
http contains "pass"
http contains "user"
http contains "auth"
```

## Filter POST login requests

```
http.request.method == "POST"
```

## Track TCP stream

```
tcp.stream eq 5
```

---

# Key Takeaways

- Not all environments use encryption correctly.
- Plaintext protocols still appear in real networks.
- Credentials may be directly visible in packet captures.
- Wireshark filters allow fast identification of sensitive traffic.
- Pcredz automates credential extraction.
- Network traffic analysis is a valuable post-exploitation technique.
- Always check for HTTP, FTP, SNMP, LDAP, and SMB traffic.

---

# Tags

#password-attacks
#credential-hunting
#network
#pcap
#wireshark
#pcredz
#post-exploitation
#enumeration
#obsidian