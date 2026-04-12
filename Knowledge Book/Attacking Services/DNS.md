# Attacking Common Services - Attacking DNS

## Overview

The **Domain Name System (DNS)** translates human-readable domain names into IP addresses.

Examples:

- `hackthebox.com` → `104.17.42.72`

DNS primarily uses:

- **UDP 53**
- **TCP 53**

Although DNS is often thought of as "UDP only", it has always supported both UDP and TCP. UDP is typically used for normal queries because it is lightweight and fast, while TCP is used when:

- the response is too large for a single UDP packet
- a DNS zone transfer is performed
- reliability is required

Because almost every networked application depends on DNS, attacks against DNS can be extremely valuable during an assessment.

---

## Why DNS Matters in an Assessment

DNS can reveal:

- internal hostnames
- subdomains
- mail servers
- third-party providers
- cloud services
- hidden environments
- legacy infrastructure
- potential takeover opportunities
- opportunities for spoofing or redirection

Think of DNS like an organisation’s phonebook. If you can read it, copy it, or alter it, you can learn a huge amount about the environment or misdirect users and systems.

---

# Enumeration

## Basic Nmap Enumeration

The Nmap default scripts and version detection can be used for quick initial DNS enumeration.

```bash
nmap -p53 -Pn -sV -sC 10.10.110.213
```

### Example Output

```text
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-29 03:47 EDT
Nmap scan report for 10.10.110.213
Host is up (0.017s latency).

PORT    STATE  SERVICE     VERSION
53/tcp  open   domain      ISC BIND 9.11.3-1ubuntu1.2 (Ubuntu Linux)
```

### What this tells you

This output reveals:

- DNS is running on TCP 53
- the DNS software is **ISC BIND**
- the likely host OS is **Ubuntu Linux**

### Why this matters

Version information can help you:

- identify outdated software
- search for known issues
- determine likely configuration styles
- understand whether the target is a Linux DNS server, Windows DNS server, or something else

---

# DNS Zone Transfer

## Overview

A **DNS zone** is a part of the DNS namespace managed by a particular DNS server or administrator.

DNS servers often replicate zone data to other DNS servers using **zone transfers**.

If this is misconfigured, an attacker may be able to request the entire zone database.

This is one of the highest-value DNS misconfigurations you can find because it can expose:

- all subdomains
- internal naming conventions
- mail routing
- administrative hosts
- hidden services
- sometimes even TXT-based secrets or operational notes

---

## Why Zone Transfers Are Dangerous

Zone transfers typically do **not require user authentication**.

Instead, security relies on the DNS server being configured to only allow trusted IPs to request the transfer.

If that restriction is missing or weak, anyone may be able to retrieve the full zone.

---

## Using `dig` for AXFR

To test for a zone transfer:

```bash
dig AXFR @ns1.inlanefreight.htb inlanefreight.htb
```

### Example Output

```text
; <<>> DiG 9.11.5-P1-1-Debian <<>> axfr inlanefrieght.htb @10.129.110.213
;; global options: +cmd
inlanefrieght.htb.         604800  IN      SOA     localhost. root.localhost. 2 604800 86400 2419200 604800
inlanefrieght.htb.         604800  IN      AAAA    ::1
inlanefrieght.htb.         604800  IN      NS      localhost.
inlanefrieght.htb.         604800  IN      A       10.129.110.22
admin.inlanefrieght.htb.   604800  IN      A       10.129.110.21
hr.inlanefrieght.htb.      604800  IN      A       10.129.110.25
support.inlanefrieght.htb. 604800  IN      A       10.129.110.28
inlanefrieght.htb.         604800  IN      SOA     localhost. root.localhost. 2 604800 86400 2419200 604800
;; Query time: 28 msec
;; SERVER: 10.129.110.213#53(10.129.110.213)
;; WHEN: Mon Oct 11 17:20:13 EDT 2020
;; XFR size: 8 records (messages 1, bytes 289)
```

### What AXFR means

`AXFR` is the query type used for a **full zone transfer**.

### What you gain from this

In the example above, the transfer exposes:

- root zone records
- internal subdomains such as:
  - `admin`
  - `hr`
  - `support`
- corresponding internal IP addresses

That is a major information disclosure issue.

---

## Using Fierce

Another tool that can test for zone transfers and enumerate DNS information is **Fierce**.

```bash
fierce --domain zonetransfer.me
```

### Why Fierce is useful

Fierce can help:

- enumerate name servers
- test for zone transfers
- gather DNS records
- map out the DNS namespace quickly

### What to look for in the output

Important records often include:

- `A`
- `AAAA`
- `MX`
- `NS`
- `TXT`
- `SRV`
- `PTR`
- `CNAME`

### High-value records

#### MX
Mail infrastructure.

#### TXT
Can reveal:
- verification tokens
- notes
- environment details
- operational hints

#### SRV
Can expose service locations such as:
- SIP
- Kerberos
- LDAP
- AD-integrated services

#### PTR
Can reveal host naming from reverse lookups.

---

# Domain Takeovers and Subdomain Enumeration

## Overview

A **domain takeover** happens when an attacker registers a domain that is still referenced by another system.

A **subdomain takeover** is the more common real-world case.

This occurs when:

- a subdomain has a `CNAME`
- that `CNAME` points to a third-party service
- the target resource no longer exists
- but the DNS record still points to it

If the attacker can claim that external resource, they may gain control of the victim subdomain.

---

## Example CNAME

```dns
sub.target.com.   60   IN   CNAME   anotherdomain.com
```

If `anotherdomain.com` expires or the linked third-party resource is unclaimed, an attacker may be able to take over `sub.target.com`.

---

## Why This Matters

A subdomain takeover may let an attacker:

- host malicious content under the victim’s domain
- serve phishing pages
- abuse trust in the organisation’s domain
- steal credentials or sessions
- bypass content trust assumptions

---

# Subdomain Enumeration

Before assessing takeover potential, you first need to enumerate subdomains.

---

## Using Subfinder

```bash
./subfinder -d inlanefreight.com -v
```

### Example Output

```text
[INF] Enumerating subdomains for inlanefreight.com
[alienvault] www.inlanefreight.com
[dnsdumpster] ns1.inlanefreight.com
[dnsdumpster] ns2.inlanefreight.com
...
ns2.inlanefreight.com
www.inlanefreight.com
ns1.inlanefreight.com
support.inlanefreight.com
[INF] Found 4 subdomains for inlanefreight.com in 20 seconds 11 milliseconds
```

### Why this is useful

Subfinder collects subdomains from public sources such as:

- DNSdumpster
- AlienVault
- BufferOver
- other OSINT sources

This is passive or low-noise compared to brute force.

---

## Using Subbrute

If internet-based passive enumeration is not suitable, you can brute-force subdomains directly with DNS queries.

```bash
git clone https://github.com/TheRook/subbrute.git >> /dev/null 2>&1
cd subbrute
echo "ns1.inlanefreight.com" > ./resolvers.txt
./subbrute.py inlanefreight.com -s ./names.txt -r ./resolvers.txt
```

### Why this matters

Subbrute is useful for:

- internal assessments
- environments without internet access
- brute-force DNS discovery against internal resolvers

### Example Findings

```text
inlanefreight.com
ns2.inlanefreight.com
www.inlanefreight.com
ms1.inlanefreight.com
support.inlanefreight.com
```

This can reveal hosts that do not appear in public sources.

---

## Checking CNAME Records

Once subdomains are found, inspect them for CNAMEs.

### Using `host`

```bash
host support.inlanefreight.com
```

### Example Output

```text
support.inlanefreight.com is an alias for inlanefreight.s3.amazonaws.com
```

### Why this matters

This tells you that the subdomain depends on an **AWS S3 bucket**.

If browsing to the subdomain shows an error such as:

- `NoSuchBucket`

then the target may be vulnerable to a **subdomain takeover** if the bucket can be claimed.

---

## Reference for Takeover Checks

A very useful resource is:

- **can-i-take-over-xyz**

This helps determine:

- whether a provider is historically vulnerable
- expected error messages
- takeover feasibility
- proof-of-concept methodology references

---

# DNS Spoofing

## Overview

DNS spoofing, also called **DNS cache poisoning**, is an attack in which false DNS information is provided so that a domain resolves to an attacker-controlled IP address.

The goal is usually to redirect users or systems to a fraudulent destination.

---

## Common DNS Spoofing Paths

Examples include:

- **MITM interception** between a victim and DNS infrastructure
- **compromise of a DNS server**
- **local network poisoning**
- **cache poisoning**
- **rogue DNS responses**

---

## Why DNS Spoofing Is Powerful

A successful DNS spoof can redirect users to:

- phishing pages
- fake login portals
- malware download sites
- attacker-controlled internal applications

Because users often trust the domain name they typed, DNS spoofing can be very effective.

---

# Local DNS Cache Poisoning

## Using Ettercap

From a local network position, tools such as **Ettercap** can be used to perform DNS spoofing.

The first step is to edit:

```bash
/etc/ettercap/etter.dns
```

### Example Configuration

```text
inlanefreight.com      A   192.168.225.110
*.inlanefreight.com    A   192.168.225.110
```

### What this does

It tells Ettercap to respond with the attacker IP `192.168.225.110` whenever the victim looks up:

- `inlanefreight.com`
- any subdomain under it

---

## Ettercap Attack Flow

General workflow:

1. Start Ettercap
2. Scan for hosts
3. Set victim as **Target1**
4. Set gateway as **Target2**
5. Enable the `dns_spoof` plugin

### Why the gateway is used

This places the attacker logically between the victim and the rest of the network, supporting MITM-style traffic manipulation.

---

## Result of a Successful Spoof

If successful, when the victim browses to:

- `inlanefreight.com`

they are redirected to the attacker-controlled IP:

- `192.168.225.110`

---

## Validation Example

A victim pinging the domain may now resolve it to the attacker host:

```cmd
ping inlanefreight.com
```

### Example Output

```text
Pinging inlanefreight.com [192.168.225.110] with 32 bytes of data:
Reply from 192.168.225.110: bytes=32 time<1ms TTL=64
Reply from 192.168.225.110: bytes=32 time<1ms TTL=64
Reply from 192.168.225.110: bytes=32 time<1ms TTL=64
Reply from 192.168.225.110: bytes=32 time<1ms TTL=64
```

### What this confirms

The victim is resolving the domain to the spoofed address rather than the legitimate one.

---

# What to Look For During DNS Assessment

When analysing DNS, focus on:

- DNS software and version
- exposed TCP/53 and UDP/53
- zone transfer misconfigurations
- public vs internal subdomain exposure
- CNAMEs to third-party providers
- dangling cloud resources
- TXT records with sensitive metadata
- SRV records revealing internal services
- opportunities for spoofing or poisoning
- split-horizon DNS misconfigurations

---

# High-Value DNS Findings

The following are especially valuable:

- successful **AXFR zone transfer**
- exposed internal hostnames
- admin/support/hr/dev subdomains
- `TXT` records with secrets or operational notes
- `CNAME` records pointing to unclaimed third-party resources
- evidence of **subdomain takeover**
- weak internal DNS trust
- ability to redirect users via local poisoning
- internal service discovery via `SRV` records

---

# Quick Commands

## Basic DNS Enumeration

```bash
nmap -p53 -Pn -sV -sC <target>
```

## Zone Transfer with `dig`

```bash
dig AXFR @<dns-server> <domain>
```

## Fierce

```bash
fierce --domain <domain>
```

## Subfinder

```bash
subfinder -d <domain> -v
```

## Subbrute

```bash
./subbrute.py <domain> -s ./names.txt -r ./resolvers.txt
```

## Check CNAME

```bash
host <subdomain>
nslookup <subdomain>
```

## Ettercap DNS Mapping File

```text
inlanefreight.com      A   192.168.225.110
*.inlanefreight.com    A   192.168.225.110
```

---

# Practical Takeaways

- DNS is often underestimated, but it can reveal a large part of an organisation’s structure.
- Zone transfers are one of the most valuable DNS misconfigurations.
- Subdomain enumeration is essential before testing for takeover opportunities.
- CNAME records pointing to cloud services deserve special attention.
- DNS spoofing on a local network can redirect users even when the domain name itself looks legitimate.
- TXT, SRV, NS, and MX records often provide much more intelligence than people expect.

---

# Tags

#attacking-common-services  
#dns  
#enumeration  
#zone-transfer  
#subdomain-takeover  
#dns-spoofing  
#ettercap  
#subfinder  
#subbrute  
#dig  
#fierce  
#networking  
#obsidian