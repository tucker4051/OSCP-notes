# Footprinting & OSINT Tools

## Overview

Footprinting involves collecting publicly available information about a target before active enumeration begins.

Typical objectives:

- identify domains and subdomains
- discover exposed services
- gather employee details
- map infrastructure
- identify technologies
- find leaked credentials
- discover attack surface

These techniques are usually **passive** and low-noise.

---

# 1. Search Engine Footprinting

| Tool | Use |
|------|-----|
| Google Dorking | search indexed sensitive content |
| Google Hacking Database (GHDB) | pre-built search queries |
| Shodan | discover internet-exposed services |
| Censys | identify exposed hosts & certificates |
| Exalead | search engine for organisational data |

Examples of sensitive data discovered:

- login portals
- exposed documents
- configuration files
- backup files
- admin panels

---

# 2. Web Services Footprinting

| Tool | Use |
|------|-----|
| Censys | discover certificates, hosts, services |
| Spyse | internet asset discovery |
| Shodan | IoT and exposed services search |
| Netcraft | hosting and infrastructure details |
| WebScan | detect exposed services and vulnerabilities |

Useful for:

- identifying infrastructure providers
- mapping hosting relationships
- identifying exposed services

---

# 3. Social Network Footprinting

| Tool | Use |
|------|-----|
| Maltego | relationship mapping |
| OSINT Framework | OSINT tool directory |
| Social-Engineer Toolkit (SET) | social engineering testing |
| Hunter.io | email discovery |
| LinkedIn Recon | employee enumeration |

Useful for:

- identifying employees
- building username lists
- identifying technologies used internally
- identifying organisational structure

---

# 4. Website Footprinting

| Tool | Use |
|------|-----|
| WhatWeb | detect web technologies |
| Wappalyzer | identify frameworks |
| Netcraft | website infrastructure |
| BuiltWith | technology stack analysis |
| FOCA | metadata extraction from documents |

Useful for:

- CMS identification
- framework discovery
- metadata leakage
- identifying vulnerable components

---

# 5. Email Footprinting

| Tool | Use |
|------|-----|
| theHarvester | email discovery |
| Hunter.io | domain email discovery |
| EmailVeritas | email validation |
| Maltego | email relationship mapping |
| Email Probe | email scraping |

Useful for:

- password spraying preparation
- phishing simulations
- credential enumeration

---

# 6. Whois Footprinting

| Tool | Use |
|------|-----|
| Whois | domain ownership details |
| WhoisXML API | advanced WHOIS queries |
| DomainTools | domain intelligence |
| ICANN WHOIS | official domain registry |
| Reverse Whois | identify related domains |

Useful for:

- discovering related domains
- identifying infrastructure ownership
- identifying organisation patterns

---

# 7. DNS Footprinting

| Tool | Use |
|------|-----|
| DNSdumpster | DNS recon |
| dig | DNS queries |
| nslookup | DNS lookup |
| Fierce | DNS scanning |
| Sublist3r | subdomain enumeration |
| Zone Transfer | full DNS database retrieval |

Useful for:

- identifying subdomains
- identifying mail servers
- identifying internal naming schemes

---

# 8. Network Footprinting

| Tool | Use |
|------|-----|
| Nmap | port scanning & service discovery |
| Netcat | service interaction |
| Wireshark | traffic capture |
| Zenmap | GUI for Nmap |
| Angry IP Scanner | host discovery |
| Traceroute | network path mapping |

Useful for:

- host discovery
- service identification
- network mapping
- protocol analysis

---

# 9. Social Engineering Footprinting

| Tool | Use |
|------|-----|
| Social-Engineer Toolkit (SET) | phishing simulation |
| Maltego | relationship mapping |
| BeEF | browser exploitation |
| Phishing Frenzy | phishing campaigns |
| King Phisher | phishing framework |
| EmailSpoof | spoofed email testing |

Useful for:

- phishing simulations
- awareness testing
- red team assessments

---

# Typical Footprinting Workflow

1. identify domains
2. discover subdomains
3. identify employees
4. identify technologies
5. identify infrastructure
6. map attack surface
7. prioritise targets

---

# Quick Tool Selection Guide

| Objective | Tool |
|----------|------|
| find subdomains | Sublist3r, dnsenum |
| find exposed services | Shodan, Censys |
| identify tech stack | WhatWeb, BuiltWith |
| find emails | theHarvester, Hunter.io |
| map relationships | Maltego |
| DNS recon | dig, Fierce |
| network discovery | Nmap |
| OSINT resources | OSINT Framework |

---

# Tags

#enumeration #footprinting #osint #recon #tools #methodology #oscp