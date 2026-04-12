# Gobuster Cheat Sheet

Fast directory, vhost, DNS, and S3 enumeration.

---

# dir Mode (Web Content Discovery)

## Basic directory brute force

```bash
gobuster dir -u https://example.com -w ~/wordlists/shortlist.txt
```

---

## Show response length (useful for filtering)

```bash
gobuster dir -u https://example.com -w ~/wordlists/shortlist.txt -l
```

---

## Search for specific file extensions

```bash
gobuster dir -u https://example.com \
-w ~/wordlists/shortlist.txt \
-x php,html,txt
```

---

## Follow redirects

```bash
gobuster dir -u https://example.com \
-w ~/wordlists/shortlist.txt \
-r
```

---

## Add custom headers (useful for auth bypass testing)

```bash
gobuster dir -u https://example.com \
-w ~/wordlists/shortlist.txt \
-H "Authorization: Bearer TOKEN"
```

---

## Use cookies

```bash
gobuster dir -u https://example.com \
-w ~/wordlists/shortlist.txt \
-c "PHPSESSID=abcd1234"
```

---

## Use proxy (Burp)

```bash
gobuster dir -u https://example.com \
-w ~/wordlists/shortlist.txt \
-p http://127.0.0.1:8080
```

---

## Filter status codes

```bash
gobuster dir -u https://example.com \
-w ~/wordlists/shortlist.txt \
-s 200,204,301,302,403
```

---

## Exclude status codes

```bash
gobuster dir -u https://example.com \
-w ~/wordlists/shortlist.txt \
-b 404
```

---

## Expanded output (full URLs)

```bash
gobuster dir -u https://example.com \
-w ~/wordlists/shortlist.txt \
-e
```

---

# dns Mode (Subdomain Enumeration)

## Basic subdomain brute force

```bash
gobuster dns -d example.com -w ~/wordlists/subdomains.txt
```

---

## Show IP addresses

```bash
gobuster dns -d example.com \
-w ~/wordlists/subdomains.txt \
-i
```

---

## Use custom DNS resolver

```bash
gobuster dns -d example.com \
-w ~/wordlists/subdomains.txt \
-r 1.1.1.1
```

---

## Increase threads

```bash
gobuster dns -d example.com \
-w ~/wordlists/subdomains.txt \
-t 50
```

---

# vhost Mode (Virtual Host Discovery)

Useful when multiple sites hosted on same server.

```bash
gobuster vhost -u https://example.com \
-w ~/wordlists/vhosts.txt
```

---

## With proxy

```bash
gobuster vhost -u https://example.com \
-w ~/wordlists/vhosts.txt \
-p http://127.0.0.1:8080
```

---

# s3 Mode (AWS Bucket Enumeration)

```bash
gobuster s3 -w bucket-names.txt
```

---

# Useful Global Flags

## Output to file

```bash
-o results.txt
```

---

## Quiet mode (less noise)

```bash
-q
```

---

## Increase threads

```bash
-t 50
```

---

## Show verbose errors

```bash
-v
```

---

## Specify wordlist

```bash
-w wordlist.txt
```

---

## Hide progress bar

```bash
-z
```

---

# Common Wordlists

```text
/usr/share/seclists/Discovery/Web-Content/common.txt

/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt

/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

/usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
```

---

# Quick Usage Reference

## directory brute force

```bash
gobuster dir -u http://target \
-w /usr/share/seclists/Discovery/Web-Content/common.txt
```

---

## directory brute force with extensions

```bash
gobuster dir -u http://target \
-w directory-list-2.3-medium.txt \
-x php,txt,html
```

---

## subdomain brute force

```bash
gobuster dns --domain example.com --resolver 8.8.8.8 -w wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt
```

---

## vhost brute force

```bash
gobuster vhost -u http://target \
-w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

---