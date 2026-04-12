# Tools & Quick Commands — ffuf

## Overview

**ffuf (Fuzz Faster U Fool)** is a fast web fuzzer used for discovering:

- directories
- files
- pages
- parameters
- subdomains
- virtual hosts
- API endpoints
- values in requests

The keyword `FUZZ` marks the position where ffuf will insert values from a wordlist.

---

# Core Syntax

```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ
```

### Quick notes
- `-w` = wordlist
- `FUZZ` = injection point
- `-u` = target URL
- `-X` = HTTP method
- `-H` = header
- `-d` = POST data
- `-fc` = filter status code
- `-fs` = filter response size
- `-ac` = auto-calibrate false positives

---

# Help / Version

```bash
ffuf -h
ffuf -V
```

---

# Basic Fuzzing

## Directory fuzzing
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ
```

Use this when looking for hidden directories or paths.

## Extension fuzzing
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/indexFUZZ
```

Useful when testing file extensions like `.php`, `.bak`, `.txt`.

## Page fuzzing
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/blog/FUZZ.php
```

Useful when you already know the extension and want hidden page names.

## Recursive fuzzing
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -recursion -recursion-depth 1 -e .php -v
```

Good for going one layer deeper automatically after hits are found.

---

# Subdomains & VHosts

## Subdomain fuzzing
```bash
ffuf -w wordlist.txt:FUZZ -u https://FUZZ.example.com/
```

Use when wildcard DNS or valid subdomain routing exists.

## VHost fuzzing
```bash
ffuf -w wordlist.txt:FUZZ -u http://example.com/ -H "Host: FUZZ.example.com" -fs <size>
```

Useful when multiple virtual hosts live on one IP and are selected via the `Host` header.

### Tip
First send a normal request and note the common response size, then filter it with `-fs`.

---

# Parameter & Value Fuzzing

## GET parameter name fuzzing
```bash
ffuf -w wordlist.txt:FUZZ -u "http://target/page.php?FUZZ=test" -fs <size>
```

Use when you suspect hidden GET parameters.

## POST parameter name fuzzing
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/page.php -X POST -d "FUZZ=test" -H "Content-Type: application/x-www-form-urlencoded" -fs <size>
```

Use when testing hidden POST parameters.

## Value fuzzing
```bash
ffuf -w ids.txt:FUZZ -u http://target/page.php -X POST -d "id=FUZZ" -H "Content-Type: application/x-www-form-urlencoded" -fs <size>
```

Use when the parameter name is known, but the value is unknown.

---

# Multiple Wordlists

## Username + password positions
```bash
ffuf -w users.txt:USER -w pass.txt:PASS -u http://target/login -X POST -d "username=USER&password=PASS"
```

Useful for login testing where multiple positions need separate wordlists.

## Two fuzz positions in URL
```bash
ffuf -w wordlist1.txt:FUZZ1 -w wordlist2.txt:FUZZ2 -u http://target/FUZZ1/FUZZ2
```

## Host + path fuzzing
```bash
ffuf -w hosts.txt:HOST -w paths.txt:FUZZ -u http://HOST/FUZZ
```

---

# Methods, Headers, Cookies, Tokens

## POST request
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/endpoint -X POST
```

## PUT request
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/endpoint -X PUT
```

## Custom headers
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -H "X-Forwarded-For: 127.0.0.1" -H "User-Agent: Mozilla/5.0"
```

## Cookie fuzzing
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/ -H "Cookie: session=FUZZ"
```

## Bearer token fuzzing
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/ -H "Authorization: Bearer FUZZ"
```

---

# Filtering Results

Filtering is critical with ffuf. Without it, false positives will waste time.

## Filter 404 responses
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -fc 404
```

## Match only 200 and 301
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -mc 200,301
```

## Filter by response size
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -fs 12345
```

## Match size range
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -ms 0,100
```

## Filter by word count
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -fw 57
```

## Filter by line count
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -fl 22
```

## Filter by regex
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -fr "Not Found"
```

## Match by regex
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -mr "admin"
```

---

# Speed, Threads, Stability

## Delay between requests
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -p 0.1
```

Useful to reduce noise or avoid rate limiting.

## Set max rate
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -rate 10
```

## Limit threads
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -t 5
```

Lower threads can help with unstable apps or aggressive WAFs.

## Timeout
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -timeout 10
```

---

# Useful Advanced Options

## Multiple extensions
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -e .php,.html,.txt
```

## Max total runtime
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -maxtime 60
```

## Max job runtime
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -maxtime-job 30
```

## Auto-calibrate
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -ac
```

Very useful when the app returns lots of soft-404 style responses.

## Ignore body
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -ignore-body
```

Speeds things up when you only care about status/size.

## Replay hits through Burp
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -replay-proxy http://127.0.0.1:8080
```

Good for manually reviewing interesting requests only.

---

# Proxies & Auth

## Proxy all requests
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -x http://127.0.0.1:8080
```

## Proxy with auth
```bash
ffuf -w wordlist.txt:FUZZ -u https://target/FUZZ -x https://user:pass@proxy:port
```

## Set static cookie
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -b "session=1234567890"
```

---

# Output

## Verbose
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -v
```

## Silent results only
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -s
```

## JSON output
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -o results.json
```

## HTML output
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -of html -o results.html
```

## CSV output
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -of csv -o results.csv
```

## Markdown output
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -of md -o results.md
```

---

# JWT Examples

## Static JWT
```bash
ffuf -w wordlist.txt:FUZZ -u http://target/FUZZ -H "Authorization: Bearer <JWT>"
```

## Fuzz JWT values
```bash
ffuf -w jwt-payloads.txt:FUZZ -u http://target/ -H "Authorization: Bearer FUZZ"
```

---

# Useful Wordlists

## Directories / pages
```text
/opt/useful/SecLists/Discovery/Web-Content/directory-list-2.3-small.txt
```

## Extensions
```text
/opt/useful/SecLists/Discovery/Web-Content/web-extensions.txt
```

## Subdomains
```text
/opt/useful/SecLists/Discovery/DNS/subdomains-top1million-5000.txt
```

## Parameters
```text
/opt/useful/SecLists/Discovery/Web-Content/burp-parameter-names.txt
```

## API endpoints
```text
/opt/useful/SecLists/Discovery/Web-Content/api/api-endpoints.txt
```

## General fuzzing
```text
/opt/useful/SecLists/Fuzzing/fuzz-Bo0oM.txt
```

---

# Handy Helpers

## Add hosts entry
```bash
sudo sh -c 'echo "SERVER_IP example.com" >> /etc/hosts'
```

## Create numeric wordlist
```bash
for i in $(seq 1 1000); do echo $i >> ids.txt; done
```

## Quick POST test with curl
```bash
curl http://target/page.php -X POST -d 'id=test' -H 'Content-Type: application/x-www-form-urlencoded'
```

## Use bash-generated numbers directly
```bash
ffuf -w <(seq 1 100) -u http://target/FUZZ
```

---

# Practical Tips

- Start simple: status code + size filtering.
- For VHosts, always baseline the normal response first.
- `-ac` is often one of the most useful options.
- Use `-replay-proxy` with Burp when you only want interesting hits replayed.
- Reduce `-t` and add `-p` if the target is unstable or rate-limited.
- For parameter fuzzing, confirm the request manually in Burp or curl first.
- `-fs`, `-fw`, and `-fl` are often more useful than status codes alone.

---

# Common Quick Examples

## Directory brute force
```bash
ffuf -w /opt/useful/SecLists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://target/FUZZ -fc 404
```

## PHP page discovery
```bash
ffuf -w /opt/useful/SecLists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://target/FUZZ.php -fc 404
```

## VHost fuzzing
```bash
ffuf -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u http://target/ -H "Host: FUZZ.target.com" -fs 1234
```

## GET parameter fuzzing
```bash
ffuf -w /opt/useful/SecLists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u "http://target/page.php?FUZZ=test" -fs 0
```

## POST parameter fuzzing
```bash
ffuf -w /opt/useful/SecLists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u http://target/page.php -X POST -d "FUZZ=test" -H "Content-Type: application/x-www-form-urlencoded" -ac
```

---

# Tags

#tools #quick-commands #ffuf #fuzzing #web #enumeration #oscp