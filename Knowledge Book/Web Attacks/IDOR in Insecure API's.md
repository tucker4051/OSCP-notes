# Web Attacks – IDOR in Insecure APIs

## Overview

IDOR vulnerabilities do not only expose files and static resources.  
They also commonly exist in **API endpoints and function calls**, where attackers may be able to:

- modify other users' data
- reset passwords
- change roles
- perform privileged actions
- create or delete accounts
- access sensitive profile information

This category is often referred to as:

### IDOR Information Disclosure
Reading data belonging to other users.

### IDOR Insecure Function Calls
Performing actions as another user.

Modern applications rely heavily on APIs, which significantly increases the attack surface for IDOR vulnerabilities.

Think of API IDOR like discovering not just another user's mailbox key, but also the ability to submit change-of-address forms on their behalf.

---

# 1. Scenario Overview

We continue using the **Employee Manager** application.

We navigate to:

```text
/profile/index.php
```

The page allows editing:

- full name
- email
- about section

These values persist, indicating they are stored in a backend database.

When submitting changes, the request is intercepted in Burp.

---

# 2. Identifying the API Endpoint

Captured request:

```http
PUT /profile/api.php/profile/1
```

PUT is commonly used in REST APIs to update resources.

Other REST verbs:

| Method | Purpose |
|--------|--------|
| GET | retrieve resource |
| POST | create resource |
| PUT | update resource |
| DELETE | remove resource |

The endpoint structure reveals something important:

```text
/profile/api.php/profile/1
```

The `1` likely corresponds to:

```text
uid = 1
```

Direct object reference detected.

---

# 3. JSON Request Body

Captured JSON payload:

```json
{
    "uid": 1,
    "uuid": "40f5888b67c748df7efba008e7c2f9d2",
    "role": "employee",
    "full_name": "Amy Lindon",
    "email": "a_lindon@employees.htb",
    "about": "A Release is like a boat. 80% of the holes plugged is not good enough."
}
```

Key observations:

### user-controlled parameters

- full_name
- email
- about

Expected editable fields.

---

### hidden parameters

- uid
- uuid
- role

These should normally be server-controlled.

Their presence in the client request is a red flag.

---

# 4. Client-Side Role Control

We also observe:

```http
Cookie: role=employee
```

The application appears to trust role information supplied by the client.

This is dangerous because:

users fully control HTTP requests.

Any parameter sent from the client can be modified.

Common attacker goal:

```json
"role": "admin"
```

If backend validation is weak, privilege escalation becomes possible.

---

# 5. Potential Attack Vectors

From the request structure, several possible attack paths exist:

### account takeover

modify:

```json
"uid": 2
```

---

### modify other users

send request to:

```http
/profile/api.php/profile/2
```

---

### privilege escalation

modify:

```json
"role": "admin"
```

---

### user creation

send:

```http
POST /profile/api.php/profile/3
```

---

### user deletion

send:

```http
DELETE /profile/api.php/profile/2
```

---

# 6. Attempt 1 – Change UID

Modify request:

```json
"uid": 2
```

Result:

```text
uid mismatch
```

Observation:

backend compares:

JSON uid value

against:

API endpoint id

Example:

```text
/profile/api.php/profile/1
```

must match:

```json
"uid": 1
```

Basic validation exists.

---

# 7. Attempt 2 – Modify Another User

Modify endpoint:

```http
/profile/api.php/profile/2
```

Modify payload:

```json
"uid": 2
```

Result:

```text
uuid mismatch
```

The application validates:

```json
uuid
```

against stored user UUID.

UUID acts as a secondary identifier.

This is an additional access control check.

---

# 8. Attempt 3 – Create New User

Modify method:

```http
POST
```

Modify uid:

```json
"uid": 3
```

Result:

```text
Creating new employees is for admins only
```

Application checks user role.

---

# 9. Attempt 4 – Delete User

Modify method:

```http
DELETE
```

Result:

```text
Deleting employees is for admins only
```

Again, role-based restriction exists.

---

# 10. Attempt 5 – Modify Role

Modify JSON:

```json
"role": "admin"
```

Result:

```text
Invalid role
```

Application validates role values.

Possible valid roles may include:

- employee
- manager
- admin
- administrator
- hr
- supervisor

But guessing blindly is inefficient.

---

# 11. Important Insight

So far, only write operations have been tested.

We have tested:

PUT  
POST  
DELETE  

These represent **insecure function calls**.

However, we have not yet tested:

GET

Information disclosure IDOR often reveals:

- user IDs
- UUID values
- role names
- internal identifiers

These can then be reused to perform function call attacks.

---

# 12. Why APIs Are High-Risk for IDOR

APIs often expose structured data:

```json
{
    "uid": 5,
    "role": "manager",
    "uuid": "c91f23..."
}
```

If accessible via IDOR, this data can enable:

privilege escalation
account takeover
horizontal movement between users

---

# 13. Typical API IDOR Pattern

### Step 1 – read another user's data

```http
GET /api/users/2
```

Response:

```json
{
  "uid": 2,
  "uuid": "81fe8bfe87576c3ecb22426f8e578473",
  "role": "admin"
}
```

---

### Step 2 – reuse identifiers

```http
PUT /api/users/2
```

```json
{
  "uid": 2,
  "uuid": "81fe8bfe87576c3ecb22426f8e578473",
  "role": "admin"
}
```

---

### Step 3 – escalate privileges

user now has elevated permissions.

---

# 14. Indicators of API IDOR Risk

Look for:

### identifiers exposed client-side

- uid
- uuid
- role
- account_id
- organisation_id

---

### role values in requests

```json
"role": "employee"
```

---

### hidden fields in JSON

values not editable via UI.

---

### predictable REST paths

```text
/api/user/1
/api/user/2
/api/user/3
```

---

# 15. Testing Workflow

## Step 1 – capture request

Use:

Burp Suite  
browser dev tools  

Look for:

PUT  
PATCH  
POST  

requests containing identifiers.

---

## Step 2 – identify object references

Common fields:

- uid
- id
- uuid
- role
- group_id
- account_id

---

## Step 3 – attempt information disclosure

Test:

```http
GET /profile/api.php/profile/2
GET /profile/api.php/profile/3
GET /profile/api.php/profile/4
```

Look for returned user data.

---

## Step 4 – reuse exposed identifiers

Combine:

uid  
uuid  
role  

to craft new requests.

---

## Step 5 – attempt privilege escalation

Modify:

```json
"role": "admin"
```

or call privileged endpoints.

---

# 16. Common API IDOR Targets

### profile updates

```http
PUT /api/profile/1
```

---

### password resets

```http
POST /api/reset-password
```

---

### role assignment

```http
PUT /api/users/role
```

---

### account deletion

```http
DELETE /api/users/5
```

---

### order modification

```http
PUT /api/orders/1001
```

---

# 17. Secure Design Principles

Never trust:

- uid from request
- role from cookie
- role from JSON
- uuid from client

Always validate server-side:

session identity must match resource owner.

---

# 18. Key Takeaways

- IDOR vulnerabilities often exist in API endpoints
- hidden parameters in JSON requests often reveal attack surface
- role-based access control must always be enforced server-side
- attackers often combine information disclosure IDOR with function call IDOR
- UUIDs alone do not guarantee security if they are exposed
- client-controlled role values are a major red flag
- REST APIs frequently expose predictable object references

---

# Tags

#idor
#api-security
#rest-api
#access-control
#privilege-escalation
#uuid
#rbac
#burp-suite
#json
#web-attacks
#attacking-web-apps
#pentesting
#ctf
#obsidian