# XSS Discovery  
  
## Overview  
**Goal:** identify user-controlled input that is rendered in the browser and executes JavaScript.  
  
XSS occurs when **unsanitised input** is inserted into the DOM and executed by the browser.  
  
---  
  
## 3 Types of XSS  
  
| Type | Stored? | Where processed | Example |  
|------|--------|----------------|--------|  
| **Reflected** | No | Server → response | Search field |  
| **Stored** | Yes | DB → page load | Comments |  
| **DOM-based** | No | Client-side JS only | URL fragments (#) |  
  
> [!info]  
> If input is reflected into page source and interpreted as code → potential XSS  
  
---  
  
# Discovery Methodology  
  
## 1. Automated Discovery  
  
### Scanner behaviour  
  
**Passive scan**  
- analyses JS/DOM for unsafe sinks  
- detects potential DOM XSS  
  
**Active scan**  
- sends payloads to parameters  
- checks reflection/execution  
  
---  
  
### Common tools  
  
| Tool | Type |  
|------|------|  
| Burp Suite Pro | Commercial |  
| Nessus | Commercial |  
| OWASP ZAP | Open source |  
| XSStrike | Open source |  
| BruteXSS | Open source |  
| XSSer | Open source |  
  
---  
  
## XSStrike Example  
  
```bash  
git clone https://github.com/s0md3v/XSStrike.git  
cd XSStrike  
pip install -r requirements.txt  
python xsstrike.py
```

Scan URL parameter:

`python xsstrike.py -u "http://SERVER_IP:PORT/index.php?task=test"

Example output:

```[+] Payload: <HtMl%09onPoIntERENTER+=+confirm()>  
[!] Efficiency: 100  
[!] Confidence: 10
```

> [!tip]  
> automated tools help identify reflection points quickly but **manual validation is always required**

---

## Tool Strategy (Exam mindset)

Automated tools help:

- identify input fields quickly
- find reflection points
- prioritise manual testing

BUT:

> automated tools ≠ proof of exploitability

Always confirm manually.

---

# 2. Manual Discovery

## Step 1 — Identify input vectors

Check anywhere user input appears:

- form fields
- search boxes
- URL parameters
- headers
    - User-Agent
    - Referer
    - Cookie
- file names
- JSON parameters
- POST bodies
- API parameters

---

## Step 2 — Test simple payloads

Start minimal:

`<script>alert(1)</script>


If filtered:

`<img src=x onerror=alert(1)>


`svg/onload=alert(1)


Break out of attributes:

`"><script>alert(1)</script>


`'><script>alert(1)</script>


---

## Step 3 — Check reflection context

Where does input appear?

|Context|Example|
|---|---|
|HTML|`<div>INPUT</div>`|
|attribute|`<input value="INPUT">`|
|JS|`var x="INPUT"`|
|URL|`?q=INPUT`|
|DOM|`document.write(INPUT)`|

Context determines payload selection.

---

## Step 4 — Observe behaviour

Check:

- page source (CTRL+U)
- dev tools → Elements
- dev tools → Network
- JS console errors

Look for:

- reflected payload
- encoded characters
- filtered tags
- escaped quotes
- DOM modifications

---

# DOM XSS Detection

DOM XSS does NOT reach server.

Indicators:

- payload processed via JS
- URL fragment (#) used
- no HTTP request triggered

Example:

http://site/page#test

Example vulnerable JS:

`document.getElementById("output").innerHTML = location.hash;

---

## DOM XSS Sources vs Sinks

### Sources (user-controlled input)

location  
document.URL  
document.referrer  
document.cookie  
window.name  
history.pushState

### Dangerous sinks

innerHTML  
outerHTML  
document.write  
eval  
setTimeout  
setInterval  
Function()

> [!warning]  
> source → sink flow without sanitisation = XSS opportunity

---

# Payload Sources

Large payload lists:

- PayloadAllTheThings
- PayloadBox
- SecLists

Useful for:

- filter bypass
- context matching
- WAF evasion

---

# Why many payloads fail

Payload success depends on:

- injection location
- filtering rules
- encoding
- JS parsing context

Examples:

|Payload type|Use case|
|---|---|
|`<script>`|HTML injection|
|`onerror=`|attribute injection|
|`</script>` breakouts|JS injection|
|encoded payloads|filter bypass|

---

# Advanced Technique (Optional)

Custom automation script concept:

1. send payload list automatically
2. compare response
3. detect reflection patterns

Useful Python libraries:

requests  
BeautifulSoup  
re

Useful when:

- many parameters
- dynamic apps
- complex reflection patterns

---

# Code Review Method

Most reliable detection method.

Trace input flow:

INPUT → backend → template → browser → DOM

Look for unsanitised variables inserted into templates:

`{{ user_input }}

What is this?

or:

`var input = "<?=$_GET['q']?>";

---

## Identify

### Sources

where input enters application

### Sinks

where JavaScript executes

Common sinks:

innerHTML  
document.write  
eval  
Function()  
setTimeout

---

# Quick Discovery Workflow (Exam)

## Fast checklist

1. identify inputs
2. inject basic payload
3. observe reflection
4. determine context
5. adjust payload
6. confirm execution

---

# Ultra-Minimal Payload Set

`<script>alert(1)</script>  
  
`<img src=x onerror=alert(1)>  `
  
`svg/onload=alert(1)  
  
`"><script>alert(1)</script>  
  
`'><script>alert(1)</script>

---

# Key Insight

> XSS discovery is primarily about understanding context rather than memorising payloads.

Automation accelerates discovery  
manual testing confirms exploitability

---

# Related Notes

[[XSS Payload Cheatsheet]]  
[[DOM XSS]]  
[[Burp Testing Workflow]]  
[[Web App Testing Methodology]]  
[[Injection Attacks Overview]]

---

# Tags

#xss  
#web_security  
#web_app_testing  
#oscp  
#injection  
#owasp  
#javascript  
#burp  
#appsec  
#dom_xss  
#stored_xss  
#reflected_xss  