# Automated Fingerprinting

## Overview

Automated fingerprinting identifies:

- operating systems
- services
- versions
- technologies
- frameworks
- exposed attack surface

This allows rapid prioritisation of attack vectors.

Often performed early during enumeration to guide deeper testing.

---

## Quick Workflow

1. identify open ports
2. detect service versions
3. identify web technologies
4. map attack surface
5. identify potential vulnerabilities

---

# Nmap Fingerprinting

## Default Scripts + Version Detection

```bash
sudo nmap -sC -sV {IP}
```

Provides:

- service versions
- default NSE script results
- misconfigurations
- authentication info

---

## Full TCP Scan

```bash
sudo nmap -p- -sC -sV {IP}
```

Scans all ports.

---

## Aggressive Scan

```bash
sudo nmap -A {IP}
```

Includes:

- OS detection
- traceroute
- script scanning
- version detection

---

## OS Detection

```bash
sudo nmap -O {IP}
```

---

## UDP Scan

```bash
sudo nmap -sU {IP}
```

Important for:

- DNS
- SNMP
- TFTP
- IPMI

---

## Targeted Service Scanning

```bash
sudo nmap -p21,22,25,80,443,445,3306,3389 {IP}
```

---

# Banner Grabbing

## Netcat

```bash
nc -nv {IP} {port}
```

---

## Telnet

```bash
telnet {IP} {port}
```

Reveals:

- service name
- software version
- hostname
- OS hints

---

# Web Fingerprinting

## whatweb

```bash
whatweb http://{IP}
```

Detects:

- CMS
- frameworks
- web server
- plugins
- libraries

---

## Wappalyzer (Browser Extension)

Identifies:

- javascript frameworks
- analytics tools
- CMS platforms

---

## Nikto

Web vulnerability scanner:

```bash
nikto -h http://{IP}
```

Finds:

- outdated software
- dangerous files
- misconfigurations

---

## curl Headers

```bash
curl -I http://{IP}
```

Shows:

- server type
- frameworks
- cookies
- security headers

---

# SSL/TLS Fingerprinting

```bash
openssl s_client -connect {IP}:443
```

Reveals:

- certificate info
- domain names
- issuer details

---

# Service Detection Tools

## Netcat Scan Range

```bash
nc -zv {IP} 1-1000
```

---

## Enum4linux (SMB)

```bash
enum4linux {IP}
```

---

## smbclient

```bash
smbclient -L //{IP}
```

---

# High Value Information

Look for:

- software versions
- outdated services
- internal hostnames
- domain names
- authentication methods
- exposed admin panels
- API endpoints

Common targets:

- Apache
- nginx
- IIS
- PHP
- WordPress
- Tomcat
- Jenkins
- Drupal

---

# Example Workflow

```bash
sudo nmap -p- -sC -sV {IP}

whatweb http://{IP}

nikto -h http://{IP}

curl -I http://{IP}

openssl s_client -connect {IP}:443
```

---

# Cheatsheet

```bash
# full scan
sudo nmap -p- -sC -sV {IP}

# aggressive scan
sudo nmap -A {IP}

# udp scan
sudo nmap -sU {IP}

# banner grab
nc -nv {IP} {port}

# web fingerprint
whatweb http://{IP}

# nikto
nikto -h http://{IP}

# headers
curl -I http://{IP}

# ssl info
openssl s_client -connect {IP}:443
```

---

# Tags

#enumeration #fingerprinting #recon #nmap #web #oscp