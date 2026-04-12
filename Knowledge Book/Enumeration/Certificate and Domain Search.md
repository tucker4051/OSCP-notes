# Certificate / Domain Search

## Overview

Certificate and domain search tools help identify:

- subdomains
- exposed infrastructure
- cloud storage buckets
- externally reachable hosts
- shadow IT assets
- forgotten environments

These techniques are passive and low-noise.

Often used early in recon phase.

---

## Quick Workflow

1. extract subdomains from CT logs
2. identify live hosts
3. resolve IP addresses
4. identify externally hosted services
5. investigate exposed storage buckets
6. pivot into active enumeration

---

# Online Tools

## Certificate Transparency Logs

https://crt.sh/

Search:

```
domain.com
```

Reveals:

- subdomains
- certificate history
- SAN entries
- related domains

---

## Domain Intelligence

https://domain.glass/

Shows:

- DNS records
- infrastructure relationships
- hosting providers
- ASN information

Useful for mapping attack surface.

---

## Public Storage Buckets

https://buckets.grayhatwarfare.com/

Search for:

```
domain.com
company name
brand name
```

May reveal exposed:

- AWS S3 buckets
- backups
- configuration files
- documents

---

# Command Line Enumeration

## Retrieve CT Logs via API

```bash
curl -s "https://crt.sh/?q={domain}&output=json" | jq .
```

---

## Extract Unique Subdomains

```bash
curl -s "https://crt.sh/?q={domain}&output=json" \
| jq . \
| grep name \
| cut -d ":" -f2 \
| awk '{gsub(/\\n/,"\n");}1;' \
| sort -u
```

---

## Identify Internet Accessible Hosts

Filters results to show hosts resolving to IP addresses belonging to target domain infrastructure.

```bash
for i in $(cat subdomainlist); do
host $i | grep "has address" | grep {domain} | cut -d " " -f1,4
done
```

---

## Extract IP Addresses

```bash
for i in $(cat subdomainlist); do
host $i | grep "has address" | grep {domain} | cut -d " " -f4 >> ip-addresses.txt
done
```

---

## Shodan Host Enumeration

Identify exposed services:

```bash
for i in $(cat ip-addresses.txt); do
shodan host $i
done
```

May reveal:

- open ports
- service banners
- historical exposures
- vulnerabilities

---

# DNS Record Enumeration

Retrieve all available DNS records:

```bash
dig any {domain}
```

Look for:

- MX records
- TXT records
- SPF entries
- mail infrastructure
- additional hosts

---

# Investigation Targets

Look for:

- dev.domain.com
- test.domain.com
- vpn.domain.com
- admin.domain.com
- mail.domain.com
- api.domain.com
- staging.domain.com

High value indicators:

- internal naming conventions
- cloud storage references
- exposed backups
- legacy services

---

# Example Workflow

```bash
# extract subdomains
curl -s "https://crt.sh/?q={domain}&output=json" \
| jq -r '.[].name_value' \
| sort -u > subdomainlist
```

```bash
# resolve hosts
for i in $(cat subdomainlist); do
host $i | grep "has address"
done
```

```bash
# extract IP addresses
for i in $(cat subdomainlist); do
host $i | grep "has address" | awk '{print $4}'
done > ip-addresses.txt
```

```bash
# query shodan
for i in $(cat ip-addresses.txt); do
shodan host $i
done
```

---

# Cheatsheet

```bash
# crt.sh extraction
curl -s "https://crt.sh/?q={domain}&output=json" \
| jq -r '.[].name_value' \
| sort -u
```

```bash
# resolve subdomains
for i in $(cat subdomainlist); do
host $i
done
```

```bash
# extract ips
for i in $(cat subdomainlist); do
host $i | awk '{print $4}'
done
```

```bash
# dns records
dig any {domain}
```

---

# Tags

#enumeration #osint #dns #ctlogs #recon #subdomains #oscp