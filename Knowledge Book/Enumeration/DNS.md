# DNS Enumeration

## Overview

DNS (Domain Name System) translates domain names into IP addresses.

DNS enumeration can reveal:

- internal hosts
- subdomains
- mail servers
- infrastructure details
- technology stack clues
- potential attack surfaces

---

## Default Port

| Port | Protocol | Service |
|------|----------|--------|
| 53 | TCP/UDP | DNS |

---

## Quick Workflow

1. identify DNS server
2. query records (A, MX, NS, TXT)
3. attempt zone transfer
4. brute force subdomains
5. enumerate virtual hosts

---

# DIG Enumeration

## SOA Record

Provides authoritative DNS information.

```bash
dig soa domain.com
```

Example:

```bash
dig soa inlanefreight.com
```

---

## Name Servers

```bash
dig ns domain.com @{IP}
```

Example:

```bash
dig ns inlanefreight.htb @{IP}
```

---

## Version Disclosure

May reveal DNS software version.

```bash
dig CH TXT version.bind @{IP}
```

---

## ANY Query

Retrieve available records (may be restricted).

```bash
dig any domain.com @{IP}
```

---

## Zone Transfer

Attempts to retrieve full DNS database.

```bash
dig axfr domain.com @{IP}
```

Look for:

- internal hostnames
- dev systems
- admin portals
- mail servers

---

## Reverse Lookup

```bash
dig -x {IP}
```

---

## Trace DNS Resolution Path

```bash
dig +trace domain.com
```

---

## Short Output

```bash
dig +short domain.com
```

---

## Display Only Answer Section

```bash
dig +noall +answer domain.com
```

---

# DNS Record Types

| Record | Purpose |
|--------|--------|
| A | IPv4 address |
| AAAA | IPv6 address |
| MX | mail server |
| NS | name servers |
| TXT | text records (often contain verification info) |
| CNAME | alias |
| SOA | domain authority info |

---

# dnsenum Enumeration

## Subdomain Bruteforce

Using SecLists:

```bash
dnsenum --dnsserver {IP} --enum -p 0 -s 0 \
-o subdomains.txt \
-f /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
domain.com
```

Alternative:

```bash
dnsenum --enum domain.com \
-f /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
-r
```

Finds:

- subdomains
- DNS records
- zone transfer opportunities

---

# Virtual Host Enumeration

## Gobuster VHOST

```bash
gobuster vhost \
-u http://{IP} \
-w wordlist.txt \
--append-domain
```

Useful flags:

| Flag | Purpose |
|------|--------|
| -t | increase threads |
| -k | ignore TLS errors |
| -o | save output |

Example:

```bash
gobuster vhost -u http://10.10.10.10 \
-w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
--append-domain -t 50
```

May need to update hosts file:

```bash
sudo nano /etc/hosts
```

Add:

```
10.10.10.10 dev.domain.com
```

---

# Additional DNS Tools

| Tool | Use Case |
|------|----------|
| nslookup | basic DNS queries |
| host | quick lookups |
| dnsrecon | comprehensive DNS recon |
| fierce | recursive subdomain discovery |
| theHarvester | OSINT email/domain data |

---

# Useful DIG Commands

```bash
dig domain.com
dig domain.com A
dig domain.com AAAA
dig domain.com MX
dig domain.com NS
dig domain.com TXT
dig domain.com CNAME
dig domain.com SOA
```

Query specific DNS server:

```bash
dig @{IP} domain.com
```

---

# High Value Findings

Look for:

- internal domains
- dev environments
- staging servers
- mail servers
- VPN portals
- admin panels
- API endpoints
- TXT records containing tokens

Common naming patterns:

- dev.domain.com
- test.domain.com
- vpn.domain.com
- mail.domain.com
- admin.domain.com
- staging.domain.com

---

# Cheatsheet

```bash
# soa record
dig soa domain.com

# name servers
dig ns domain.com @{IP}

# zone transfer
dig axfr domain.com @{IP}

# dns version
dig CH TXT version.bind @{IP}

# reverse lookup
dig -x {IP}

# brute force subdomains
dnsenum --enum domain.com \
-f /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt

# vhost brute force
gobuster vhost -u http://{IP} \
-w wordlist.txt --append-domain
```

---

# Tags

#enumeration #dns #subdomain #oscp #recon #service-enum