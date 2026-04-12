# Nmap — Tools & Quick Commands

## Key Defaults

- scans **top 1000 TCP ports** by default
- performs **TCP scan** by default
- requires **sudo/root** for SYN scan (-sS) and many advanced features

---

# Target Specification

| Switch | Example | Description |
|--------|--------|------------|
|  | `nmap 192.168.1.1` | scan single IP |
|  | `nmap 192.168.1.1 192.168.2.1` | scan multiple IPs |
|  | `nmap 192.168.1.1-254` | scan range |
|  | `nmap scanme.nmap.org` | scan domain |
|  | `nmap 192.168.1.0/24` | scan CIDR range |
| -iL | `nmap -iL targets.txt` | scan from file |
| -iR | `nmap -iR 100` | scan random hosts |
| --exclude | `nmap --exclude 192.168.1.1` | exclude target |

---

# Scan Techniques

| Switch | Example | Description |
|--------|--------|------------|
| -sS | `nmap -sS 192.168.1.1` | SYN scan (default with root) |
| -sT | `nmap -sT 192.168.1.1` | TCP connect scan |
| -sU | `nmap -sU 192.168.1.1` | UDP scan |
| -sA | `nmap -sA 192.168.1.1` | ACK scan |
| -sW | `nmap -sW 192.168.1.1` | window scan |
| -sM | `nmap -sM 192.168.1.1` | maimon scan |

---

# Host Discovery

| Switch | Example | Description |
|--------|--------|------------|
| -sL | `nmap -sL 192.168.1.1-10` | list targets only |
| -sn | `nmap -sn 192.168.1.0/24` | host discovery only |
| -Pn | `nmap -Pn 192.168.1.1` | skip host discovery |
| -PS | `nmap -PS22,80 192.168.1.1` | SYN discovery |
| -PA | `nmap -PA22,80 192.168.1.1` | ACK discovery |
| -PU | `nmap -PU53 192.168.1.1` | UDP discovery |
| -PR | `nmap -PR 192.168.1.0/24` | ARP discovery |
| -n | `nmap -n 192.168.1.1` | no DNS resolution |

---

# Port Specification

| Switch | Example | Description |
|--------|--------|------------|
| -p | `nmap -p 21 192.168.1.1` | single port |
| -p | `nmap -p 21-100 192.168.1.1` | port range |
| -p | `nmap -p U:53,T:21-25,80 192.168.1.1` | tcp + udp |
| -p- | `nmap -p- 192.168.1.1` | all ports |
| -F | `nmap -F 192.168.1.1` | fast scan |
| --top-ports | `nmap --top-ports 2000 192.168.1.1` | scan top ports |
| -p-65535 | `nmap -p-65535 192.168.1.1` | ports 1–65535 |

---

# Service & Version Detection

| Switch | Example | Description |
|--------|--------|------------|
| -sV | `nmap -sV 192.168.1.1` | service version detection |
| --version-intensity | `nmap -sV --version-intensity 8` | intensity 0–9 |
| --version-light | `nmap -sV --version-light` | faster scan |
| --version-all | `nmap -sV --version-all` | full detection |
| -A | `nmap -A 192.168.1.1` | OS + scripts + traceroute |

---

# OS Detection

| Switch | Example | Description |
|--------|--------|------------|
| -O | `nmap -O 192.168.1.1` | detect OS |
| --osscan-limit | `nmap -O --osscan-limit` | skip unsuitable hosts |
| --osscan-guess | `nmap -O --osscan-guess` | aggressive guessing |
| --max-os-tries | `nmap -O --max-os-tries 1` | limit attempts |

---

# Timing Profiles

| Switch | Speed |
|--------|------|
| -T0 | paranoid |
| -T1 | sneaky |
| -T2 | polite |
| -T3 | normal (default) |
| -T4 | aggressive |
| -T5 | insane |

Example:

```bash
nmap -T4 192.168.1.1
```

---

# Performance Tuning

| Switch | Description |
|--------|------------|
| --host-timeout 5m | stop scanning host after time |
| --max-retries 3 | max retry attempts |
| --min-rate 100 | packets per second minimum |
| --max-rate 1000 | packets per second max |
| --min-parallelism 10 | increase concurrency |

---

# NSE Scripts

| Switch | Example | Description |
|--------|--------|------------|
| -sC | `nmap -sC 192.168.1.1` | default scripts |
| --script | `nmap --script=banner` | run script |
| --script http* | `nmap --script http*` | wildcard |
| --script vuln | `nmap --script vuln` | vuln scripts |
| --script-args | `nmap --script-args user=admin` | script arguments |

---

# Useful NSE Examples

```bash
# dns brute force
nmap --script dns-brute domain.com
```

```bash
# smb enumeration
nmap -p445 --script smb-enum* {IP}
```

```bash
# web title + banner
nmap -p80 --script http-title,banner {IP}
```

```bash
# sql injection test
nmap -p80 --script http-sql-injection {IP}
```

```bash
# vulnerability scan
nmap --script vuln {IP}
```

---

# Firewall / IDS Evasion

| Switch | Example | Description |
|--------|--------|------------|
| -f | fragment packets |
| --mtu 32 | custom mtu |
| -D | decoy scan |
| -S | spoof source IP |
| -g 53 | source port |
| --data-length 200 | random data padding |
| --proxies | proxy chain |

Example:

```bash
nmap -f 192.168.1.1
```

---

# Output Formats

| Switch | Description |
|--------|------------|
| -oN file | normal output |
| -oX file | xml output |
| -oG file | grepable output |
| -oA name | all formats |
| -v | verbose |
| -vv | very verbose |
| -d | debug |
| --reason | show port reason |
| --open | show open ports only |
| --resume | resume scan |

Example:

```bash
nmap -oA scan 192.168.1.1
```

---

# Useful Output Processing

```bash
# find web servers
nmap -p80 -sV -oG - {IP}/24 | grep open
```

```bash
# extract live hosts
grep "Up" scan.gnmap
```

```bash
# compare scans
ndiff scan1.xml scan2.xml
```

```bash
# convert xml to html
xsltproc scan.xml -o scan.html
```

```bash
# most common open ports
grep " open " results.nmap | sort | uniq -c | sort -rn
```

---

# Common Scan Patterns

## fast scan

```bash
nmap -T4 -F {IP}
```

---

## full tcp scan

```bash
sudo nmap -p- -T4 {IP}
```

---

## full tcp + version

```bash
sudo nmap -p- -sC -sV -T4 {IP}
```

---

## udp scan

```bash
sudo nmap -sU {IP}
```

---

## aggressive scan

```bash
sudo nmap -A {IP}
```

---

## stealth scan

```bash
sudo nmap -sS -T2 {IP}
```

---

## scan from file

```bash
nmap -iL targets.txt
```

---

# Misc

```bash
# ipv6 scan
nmap -6 {IP}
```

```bash
# show interfaces
nmap --iflist
```

```bash
# packet trace
nmap --packet-trace {IP}
```

```bash
# traceroute only
nmap --traceroute {IP}
```

---

# Tags

#tools #nmap #scanning #recon #oscp #quickref