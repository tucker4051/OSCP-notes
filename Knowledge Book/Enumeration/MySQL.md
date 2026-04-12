# MySQL Enumeration

## Overview
MySQL commonly runs on TCP **3306** and may allow:
- Anonymous access
- Weak credentials
- Database disclosure
- Credential harvesting from application tables
- Privilege escalation via writable DBs

Misconfigured MySQL instances frequently expose sensitive application data such as:
- usernames
- password hashes
- API keys
- configuration secrets

Reference:
https://dev.mysql.com/doc/refman/8.0/en/general-security-issues.html

---

## Default Port

| Port | Protocol | Service |
|------|----------|--------|
| 3306 | TCP | MySQL |

---

## Quick Workflow

1. Identify MySQL service
2. Attempt authentication (default creds / discovered creds)
3. Enumerate databases
4. Extract sensitive data

---

## Nmap Scanning

### Service Detection + NSE scripts
```bash
sudo nmap {target IP} -sV -sC -p3306 --script mysql*
```

### Example
```bash
sudo nmap 10.10.10.10 -sV -sC -p3306 --script mysql*
```

### Useful NSE scripts
| Script | Purpose |
|--------|--------|
| mysql-info | MySQL version info |
| mysql-users | Enumerate users |
| mysql-databases | List databases |
| mysql-variables | Server configuration |
| mysql-empty-password | Check for blank password |

---

## Connecting to MySQL

### Syntax
```bash
mysql -u <user> -p<password> -h <IP address>
```

⚠️ No space between `-p` and the password.

### Example
```bash
mysql -u root -ppassword -h 10.10.10.10
```

Prompted password version:
```bash
mysql -u root -p -h 10.10.10.10
```

---

## Database Enumeration Commands  
  
| Command | Description |  
|---------|-------------|  
| `show databases;` | List all databases |  
| `use database_name;` | Select a database |  
| `show tables;` | List tables in selected DB |  
| `show columns from table_name;` | Show table structure |  
| `select * from table_name;` | Dump all table contents |  
| `select * from table_name where column_name = "value";` | Filter results |

## Example Enumeration Workflow

```sql
show databases;

use employees;

show tables;

show columns from users;

select * from users;

select * from users where username = "admin";
```

---

## High Value Targets

### Common interesting database names
- users
- accounts
- members
- admin
- authentication
- wordpress
- customer
- login

### Sensitive column names
- username
- password
- passwd
- hash
- email
- token
- api_key

---

## Credential Harvesting Pattern

```sql
show databases;

use <database>;

show tables;

select * from users;
```

Look for:
- password hashes
- plaintext credentials
- password reset tokens
- session tokens

---

## Cheatsheet

```bash
# scan mysql
sudo nmap {IP} -sV -sC -p3306 --script mysql*

# connect
mysql -u root -p -h {IP}
mysql -u root -ppassword -h {IP}
```

```sql
show databases;

use <database>;

show tables;

show columns from <table>;

select * from <table>;

select * from <table> where <column> = "value";
```

---

## Notes

- MySQL may only allow local connections (127.0.0.1)
- Credentials often reused across services (SSH, SMB, web login)
- Check config tables for hardcoded credentials
- Look for password hashing format (useful for cracking)

---

## Tags
#enumeration #mysql #database #oscp #service-enum