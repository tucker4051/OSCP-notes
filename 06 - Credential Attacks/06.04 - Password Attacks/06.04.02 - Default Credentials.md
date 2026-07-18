# Scenarios
## Common
```
admin:admin
admin:password
admin:1234
admin:12345
admin:123456
admin:12345678
admin:admin123
admin:password123
admin:qwerty
admin:qwerty123
admin:letmein

administrator:password
administrator:admin

root:root
root:toor
root:password
root:1234
root:123456

user:user
user:password

guest:guest
guest:password

test:test
test:password

demo:demo

info:info

support:support

operator:operator

service:service

backup:backup

webadmin:webadmin
webadmin:password

sysadmin:sysadmin
sysadmin:password

it:it

it:password
```

## Linux / Unix Systems
```
root:root
root:toor

ubuntu:ubuntu
ubuntu:password

kali:kali

vagrant:vagrant

pi:raspberry

admin:admin

user:user

test:test

oracle:oracle

postgres:postgres

mysql:mysql

ftp:ftp
anonymous:anonymous

www-data:www-data

git:git

nagios:nagios
tomcat:tomcat
```

## Windows / Active Directory
```
Administrator:Password123
Administrator:Password1
Administrator:Passw0rd
Administrator:P@ssw0rd
Administrator:Admin123
Administrator:Welcome1
Administrator:Winter2023
Administrator:Summer2023

admin:Password123
admin:Admin123

user:Password123
user:Welcome1

guest:guest
```

## Web Apps / CMS
### Wordpress
```
admin:admin
admin:password
admin:wordpress
admin:wpadmin
administrator:administrator
```

### Joomla
```
admin:admin
admin:password
```

### Drupal
```
admin:admin
admin:password
```

### Tomcat Manager
```
tomcat:tomcat
admin:admin
admin:password
tomcat:s3cret
admin:s3cret
```

### Jenkins
```
admin:admin
admin:password
jenkins:jenkins
```

### phpMyAdmin
```
root:
root:root
root:password
mysql:mysql
```

### WebLogic
```
weblogic:weblogic
system:manager
```

## Databases
### MySQL
```
root:
root:root
root:password
mysql:mysql
admin:admin
```

### PostgreSQL

```
postgres:postgres  
postgres:password
```

### MSSQL

```
sa:password  
sa:Password123  
sa:P@ssw0rd
```

### MongoDB

```
admin:admin  
mongo:mongo
```

### Oracle

```
system:oracle  
sys:oracle  
scott:tiger
```

## Network Devices / Infrastructure

### Cisco

```
cisco:cisco  
admin:cisco  
admin:admin
```

### pfSense

```
admin:pfsense
```

### Ubiquiti

```
ubnt:ubnt
```

### Mikrotik

```
admin:  
admin:admin
```

### HP iLO

```
Administrator:admin  
Administrator:password
```

### IPMI

```
ADMIN:ADMIN  
root:calvin
```

## FTP / File Transfer

```
anonymous:anonymous  
ftp:ftp  
ftpuser:ftpuser  
test:test
```

## Email Servers

### Exchange / SMTP

```
admin:admin  
administrator:password
```

### Zimbra
```
admin:admin
```

## SNMP Default Strings

Useful during enumeration.

```
public  
private  
community  
manager  
admin
```

Example:

```
snmpwalk -v2c -c public 10.10.10.10
```

## Default Credential Wordlists (Recommended)

Useful sources for automation.
```
SecLists/Passwords/Default-Credentials/  
SecLists/Usernames/top-usernames-shortlist.txt  
  
SecLists/Passwords/Common-Credentials/  
SecLists/Usernames/cirt-default-usernames.txt 
``` 
  
```
Metasploit:  
auxiliary/scanner/http/http_default_creds
```

# Quick Testing Strategy

Efficient workflow during OSCP labs:

### Step 1 — try highest probability combos first

```
admin:admin  
admin:password  
root:root  
root:toor  
user:user  
test:test  
guest:guest
```

### Step 2 — try service specific defaults

Example:

```
tomcat:tomcat  
postgres:postgres  
mysql:mysql  
jenkins:jenkins
```

### Step 3 — try seasonal/company patterns

```
CompanyName2024!  
CompanyName123!  
Winter2024!  
Spring2024!
```

---

# 13. Hydra Example

```
hydra -L users.txt -P passwords.txt 10.10.10.10 http-post-form "/login.php:user=^USER^&pass=^PASS^:Invalid"
```

---

# 14. Burp Intruder Tip

Cluster bomb common username/password combos first:

```
admin  
administrator  
root  
test  
user  
guest  
info  
support  
sysadmin
```

with:

```
admin  
password  
Password123  
P@ssw0rd  
123456  
qwerty  
letmein
```
