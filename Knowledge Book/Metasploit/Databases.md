# Metasploit Databases

## Overview

Metasploit uses a **PostgreSQL-backed database** to store assessment data such as:

- hosts
- services
- notes
- vulnerabilities
- credentials
- loot
- scan imports

This is extremely useful when working on:

- larger boxes
- multiple hosts
- internal networks
- long assessments
- repeatable workflows

Benefits:

- keeps findings organised
- allows import/export of scan data
- supports workspaces
- lets you reuse findings inside modules
- makes host/service/credential tracking far easier

---

# 1. Why Use the Database

Without a database, it is easy to lose track of:

- discovered hosts
- open ports
- service versions
- credentials found
- loot collected
- which systems belong to which assessment

Metasploit’s database helps turn loose scan output into structured project data.

---

# 2. PostgreSQL Status

Check whether PostgreSQL is running:

```bash
sudo service postgresql status
```

Start PostgreSQL if needed:

```bash
sudo systemctl start postgresql
```

---

# 3. Initialise the MSF Database

Initial setup:

```bash
sudo msfdb init
```

Typical successful output includes:

- creating database user `msf`
- creating `msf` and `msf_test`
- creating `database.yml`
- creating initial schema

---

## Common Issue

If `msfdb init` errors due to version mismatch or package issues:

```bash
sudo apt update && sudo apt install metasploit-framework
```

Then retry:

```bash
sudo msfdb init
```

---

## Check Database Status

```bash
sudo msfdb status
```

This confirms:

- PostgreSQL state
- listening port
- config file path
- whether the database is configured

---

# 4. Start MSF with Database Connected

Launch Metasploit with database support:

```bash
sudo msfdb run
```

Or open msfconsole directly and verify DB status:

```bash
msfconsole -q
db_status
```

Expected:

```text
[*] Connected to msf. Connection type: postgresql.
```

---

# 5. Reinitialising the Database

If the DB is broken or config is inconsistent:

```bash
msfdb reinit
cp /usr/share/metasploit-framework/config/database.yml ~/.msf4/
sudo service postgresql restart
msfconsole -q
```

Then verify:

```bash
db_status
```

---

# 6. Database Commands

Show built-in database help:

```bash
help database
```

Common database commands:

| Command | Purpose |
|--------|---------|
| `db_status` | show DB connection status |
| `db_connect` | connect to existing DB |
| `db_disconnect` | disconnect current DB |
| `db_import` | import scan results |
| `db_export` | export DB contents |
| `db_nmap` | run Nmap and auto-store results |
| `db_rebuild_cache` | rebuild module cache |
| `hosts` | list hosts |
| `services` | list services |
| `notes` | list notes |
| `vulns` | list vulnerabilities |
| `loot` | list collected loot |
| `workspace` | manage workspaces |

---

# 7. Workspaces

## Overview

Workspaces are like project folders inside Metasploit.

Use them to separate:

- individual boxes
- clients
- networks
- subnets
- domains
- phases of an assessment

---

## List Workspaces

```bash
workspace
```

Example:

```text
* default
```

The `*` shows the active workspace.

---

## Create a Workspace

```bash
workspace -a Target_1
```

---

## Switch Workspace

```bash
workspace Target_1
```

---

## Show Workspace Help

```bash
workspace -h
```

Useful options:

| Command | Purpose |
|--------|---------|
| `workspace` | list workspaces |
| `workspace -a NAME` | add workspace |
| `workspace -d NAME` | delete workspace |
| `workspace -D` | delete all workspaces |
| `workspace NAME` | switch workspace |
| `workspace -r OLD NEW` | rename workspace |
| `workspace -v` | verbose listing |

---

# 8. Importing Scan Results

## Preferred Input

Use **Nmap XML** for imports whenever possible.

---

## Import File

```bash
db_import Target.xml
```

Expected output:

```text
[*] Importing 'Nmap XML' data
[*] Importing host 10.10.10.40
[*] Successfully imported ~/Target.xml
```

---

## Verify Imported Hosts

```bash
hosts
```

---

## Verify Imported Services

```bash
services
```

This is one of the most useful workflows in Metasploit.

---

# 9. Running Nmap from Inside MSF

You can run Nmap directly from Metasploit and store the results automatically.

## Example

```bash
db_nmap -sV -sS 10.10.10.8
```

Results are inserted directly into the database.

Then check:

```bash
hosts
services
```

---

## Why `db_nmap` Is Useful

- no need to leave msfconsole
- results are auto-stored
- avoids manual imports
- fast for simple workflows

---

# 10. Exporting / Backing Up Data

Export workspace contents:

```bash
db_export -f xml backup.xml
```

Formats supported:

- `xml`
- `pwdump`

Use this to:

- back up progress
- move data between systems
- archive assessment findings

---

# 11. Hosts Command

The `hosts` table stores discovered hosts and host metadata.

Common fields include:

- IP address
- hostname
- OS info
- comments
- service count
- vuln count
- tags

---

## Show Help

```bash
hosts -h
```

---

## Common Usage

```bash
hosts
hosts -u
hosts -S 10.10.
hosts -o hosts.csv
```

---

## Useful Options

| Option | Purpose |
|--------|---------|
| `-a` | add host manually |
| `-d` | delete host |
| `-u` | show only up hosts |
| `-o FILE` | export to CSV |
| `-S STRING` | search/filter |
| `-R` | set `RHOSTS` from results |
| `-n` | change host name |
| `-m` | add/change comment |
| `-t` | tag hosts |

---

## Set RHOSTS from Hosts Output

```bash
hosts -R
```

Very useful before running modules.

---

# 12. Services Command

The `services` table stores discovered services and their metadata.

Examples:

- port
- protocol
- service name
- state
- version info

---

## Show Help

```bash
services -h
```

---

## Common Usage

```bash
services
services -p 445
services -s smb
services -u
services -o services.csv
```

---

## Useful Options

| Option | Purpose |
|--------|---------|
| `-a` | add service manually |
| `-d` | delete service |
| `-u` | show only up services |
| `-p` | filter by port |
| `-s` | filter by service name |
| `-r` | protocol type |
| `-R` | set `RHOSTS` from results |
| `-S` | search/filter |
| `-U` | update existing service data |

---

## Example

```bash
services -p 445 -R
```

This can help target SMB-only hosts.

---

# 13. Credentials (`creds`)

The `creds` command stores and manages credentials discovered during an engagement.

Examples:

- usernames + passwords
- NTLM hashes
- SSH keys
- JtR hashes
- Postgres MD5
- non-replayable hashes

---

## Show Help

```bash
creds -h
```

---

## List Credentials

```bash
creds
```

---

## Filter Examples

```bash
creds -p 445
creds -s smb
creds -t NTLM
creds -u admin
creds -v
```

---

## Add Credentials Manually

### Username + password

```bash
creds add user:admin password:notpassword realm:workgroup
```

### Password only

```bash
creds add password:'password without username'
```

### NTLM hash

```bash
creds add user:admin ntlm:E2FC15074BF7751DD408E6B105741864:A1074A69B1BDE45403AB680504BBDD1A
```

### SSH key

```bash
creds add user:sshadmin ssh-key:/path/to/id_rsa
```

---

## Export Credentials

```bash
creds -o creds.csv
```

Or to JtR / hashcat-style output via filename extension.

---

# 14. Loot

The `loot` command stores files and artefacts collected from target systems.

Examples:

- password dumps
- registry exports
- hashes
- config files
- shadow/passwd dumps
- exfiltrated evidence files

---

## Show Help

```bash
loot -h
```

---

## List Loot

```bash
loot
```

---

## Filter Loot by Type

```bash
loot -t passwords
loot -t hashdump
```

---

## Add Loot Manually

```bash
loot -a -f lootfile.txt -i "Domain admin notes" -t notes 10.10.10.40
```

---

## Delete Loot

```bash
loot -d 10.10.10.40
```

---

# 15. Notes and Vulns

Also useful:

```bash
notes
vulns
```

These help track:

- observations
- module findings
- imported vulnerability data
- workflow annotations

---

# 16. Typical Workflow

## Start DB-backed Metasploit

```bash
msfconsole -q
db_status
```

## Create workspace

```bash
workspace -a Client_A
workspace Client_A
```

## Import or scan

```bash
db_import scan.xml
```

or

```bash
db_nmap -sV -sS 10.10.10.0/24
```

## Review data

```bash
hosts
services
creds
loot
```

## Set targets from results

```bash
services -p 445 -R
```

Then use matching modules.

---

# 17. Key Benefits in OSCP / Labs

- reduces note sprawl
- quickly filters targets by port/service
- lets you reuse results across modules
- keeps credentials and loot centralised
- helps when working multiple hosts quickly

---

# 18. Practical Notes

- prefer XML when importing Nmap
- use one workspace per box/client/network
- export important data regularly
- check `db_status` early if commands seem empty
- `hosts -R` and `services -R` are underrated time savers

---

# Cheatsheet

## DB setup

```bash
sudo systemctl start postgresql
sudo msfdb init
msfconsole -q
db_status
```

## Workspace

```bash
workspace
workspace -a Target_1
workspace Target_1
workspace -h
```

## Import / scan

```bash
db_import Target.xml
db_nmap -sV -sS 10.10.10.8
```

## Review

```bash
hosts
services
creds
loot
notes
vulns
```

## Export

```bash
db_export -f xml backup.xml
```

## Hosts / services targeting

```bash
hosts -R
services -p 445 -R
```

## Credentials

```bash
creds
creds add user:admin password:notpassword realm:workgroup
creds -t NTLM
creds -o creds.csv
```

## Loot

```bash
loot
loot -t hashdump
```

---

# Related Notes

- Metasploit overview
- modules
- targets
- payloads
- auxiliary scanning
- workspace management

---

# Tags

#metasploit #database #postgresql #workspaces #hosts #services #creds #loot #oscp