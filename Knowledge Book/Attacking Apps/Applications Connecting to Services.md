# Applications Connecting to Services – Connection Strings and Credential Recovery

## Overview

Applications that connect to backend services often contain:

- connection strings
- hardcoded credentials
- API endpoints
- internal hostnames
- database drivers
- service paths

If these values are not protected properly, they can often be recovered through:

- static analysis
- debugging
- memory inspection
- metadata review
- decompilation

This is valuable during pentests because recovered credentials can often be used to:

- access databases
- move laterally
- test password reuse
- escalate privileges
- enumerate internal services

Think of these applications like delivery vans. Even if the doors are locked, the route sheet and keys are often sitting inside.

---

# 1. Why Applications Connected to Services Matter

A local application rarely works in isolation.

Many enterprise apps connect to:

- MSSQL
- MySQL
- Oracle
- HTTP APIs
- internal web services
- LDAP
- other middleware

To do this, they often store or construct values such as:

- usernames
- passwords
- DSNs
- ODBC strings
- API URLs
- bearer tokens
- internal ports
- keystore paths

If the application trusts the client environment too much, these values may be exposed.

---

# 2. Common Analysis Approaches

When examining applications that connect to services, the most useful approaches are:

## Static analysis
Read strings, metadata, code, and configs without running the binary.

## Dynamic analysis
Run the binary and inspect:

- memory
- registers
- arguments
- network traffic
- temporary files

## Decompilation / source recovery
Use appropriate tools to recover higher-level logic from:

- .NET assemblies
- Java JARs
- native binaries where possible

---

# 3. Scenario 1 – ELF Executable Examination

The notes begin with a binary found on a remote machine:

```text
octopus_checker
```

Running it locally shows it is trying to connect to a database.

## Example execution

```bash
./octopus_checker
```

## Output

```text
Program had started..
Attempting Connection 
Connecting ... 

The driver reported the following diagnostics whilst running SQLDriverConnect

01000:1:0:[unixODBC][Driver Manager]Can't open lib 'ODBC Driver 17 for SQL Server' : file not found
connected
```

## Why this matters

Even though the correct ODBC driver is missing locally, the error tells us a lot:

- it uses **ODBC**
- it expects:
  - `ODBC Driver 17 for SQL Server`
- it is attempting an MSSQL connection
- the connection string is likely built inside the binary

At this point, the target is not just the program’s output. The target is the **connection string in memory**.

---

# 4. Examining the ELF with GDB / PEDA

The notes use:

- `gdb`
- optionally with `PEDA`

to inspect the binary.

## Load binary

```bash
gdb ./octopus_checker
```

## Set Intel syntax

```gdb
set disassembly-flavor intel
```

## Disassemble `main`

```gdb
disas main
```

The disassembly reveals:

- string-related operations
- calls building pieces of a larger string
- a call to:

```text
SQLDriverConnect@plt
```

That is the key function.

---

# 5. Why `SQLDriverConnect` Matters

If a program is calling:

```text
SQLDriverConnect
```

then at some point it must provide:

- a DSN or connection string
- likely containing:
  - server
  - port
  - username
  - password
  - driver

Rather than trying to reconstruct the string manually from disassembly, it is usually faster to:

- set a breakpoint on the call
- inspect the relevant argument register at runtime

---

# 6. Breakpointing the SQL Connection

The notes set a breakpoint at the `SQLDriverConnect` call site.

## Example

```gdb
b *0x5555555551b0
run
```

When the program hits the breakpoint, inspect the registers.

## Result from the notes

The `RDX` register contains:

```text
DRIVER={ODBC Driver 17 for SQL Server};SERVER=localhost, 1401;UID=username;PWD=password;
```

## Why this is important

That immediately gives:

- DB driver
- server
- port
- username
- password

This is exactly the kind of secret often hidden in desktop apps and service utilities.

---

# 7. Practical Outcome of the ELF Analysis

Recovered connection string:

```text
DRIVER={ODBC Driver 17 for SQL Server};SERVER=localhost, 1401;UID=username;PWD=password;
```

## What to do with it

Now you can:

- attempt database access directly
- look for the service on the target/internal network
- test the creds against other services
- check for password reuse
- use it to understand app architecture

This is a great example of how dynamic analysis beats guesswork.

---

# 8. Key ELF Takeaway

When a native binary connects to a service, look for:

- imported network / DB functions
- connection APIs
- strings built in fragments
- runtime register values just before the connection call

If the app does not print the creds, memory often still will.

---

# 9. Scenario 2 – DLL File Examination

The second example is a DLL found during enumeration:

```text
MultimasterAPI.dll
```

DLLs are **Dynamically Linked Libraries**.

They often contain:

- shared code
- API logic
- controller logic
- connection strings
- business rules

The notes determine this file is a **.NET assembly**.

---

# 10. Metadata Review

The notes use a metadata inspection command:

```powershell
Get-FileMetaData .\MultimasterAPI.dll
```

The output contains useful strings including:

- `.NETFramework,Version=v4.6.1`
- `api/getColleagues`
- `http://localhost:8081`
- `POST`
- local development paths

## Why this matters

Even metadata can reveal:

- framework version
- API routes
- localhost services
- developer paths
- transport method

This is often enough to map how the application talks to its backend.

---

# 11. Using dnSpy

Because this is a .NET DLL, the notes move to:

- `dnSpy`

This is ideal for .NET because it allows:

- reading source-like code
- editing assemblies
- debugging managed code

The relevant area mentioned is:

```text
MultimasterAPI.Controllers -> ColleagueController
```

Inspection reveals a **database connection string containing the password**.

---

# 12. Why .NET Assemblies Are So Valuable

.NET binaries are often much easier to reverse than native binaries because tools like dnSpy can reconstruct code that is very close to the original source.

That makes it common to recover:

- credentials
- connection strings
- hidden endpoints
- role checks
- internal logic
- API secrets

In many cases, you do not even need a debugger. Simple decompilation is enough.

---

# 13. Practical Outcome of the DLL Analysis

From the DLL, a tester may recover:

- database connection string
- password
- API path such as:
  - `/api/getColleagues`
- service location:
  - `http://localhost:8081`

## What to do next

As with the ELF case, useful next steps are:

- test DB access
- check whether the password is reused elsewhere
- enumerate the local/internal API
- use spraying carefully against other services if in scope
- inspect adjacent controllers and models for more secrets

---

# 14. Typical Things to Hunt For

Whether you are examining an ELF, EXE, DLL, or JAR, look for:

## Database clues
- `Server=`
- `Data Source=`
- `Initial Catalog=`
- `UID=`
- `PWD=`
- `User ID=`
- `Password=`
- `jdbc:`
- `odbc`

## API clues
- `/api/`
- `localhost`
- internal hostnames
- bearer tokens
- auth headers

## Other secrets
- hardcoded usernames
- passwords
- keystore paths
- cert passwords
- LDAP bind strings
- SMTP credentials

---

# 15. Practical Workflow

## Step 1 – identify file type

Use:

- `file`
- `strings`
- metadata tools
- Detect It Easy / PE tooling / .NET tooling

Determine whether the target is:

- native ELF/EXE
- .NET assembly
- Java archive
- script wrapper

## Step 2 – run the program

Observe:

- console output
- connection attempts
- missing driver errors
- hostnames
- ports
- temporary files

## Step 3 – inspect statically

Look for:

- strings
- imported functions
- metadata
- embedded URLs
- developer paths

## Step 4 – inspect dynamically

For native binaries:

- use `gdb`
- break on network/DB functions
- inspect registers and memory

For .NET:

- use `dnSpy`
- review decompiled controllers/classes
- inspect config and constructors

## Step 5 – recover and validate credentials

Use recovered material against:

- databases
- APIs
- other internal services
- other login portals if password reuse is plausible and in scope

---

# 16. Useful Commands and Tools

## ELF / native binary analysis

### Run the binary

```bash
./octopus_checker
```

### Open with GDB

```bash
gdb ./octopus_checker
```

### Intel syntax

```gdb
set disassembly-flavor intel
```

### Disassemble main

```gdb
disas main
```

### Break on connection call

```gdb
b *0x5555555551b0
run
```

## .NET / DLL analysis

### Metadata review

```powershell
Get-FileMetaData .\MultimasterAPI.dll
```

### Decompilation

- `dnSpy`

---

# 17. Key Takeaways

- applications that connect to services often leak connection details
- native binaries may expose secrets at runtime even when strings are obfuscated or fragmented
- breaking on connection APIs is a highly effective way to recover credentials
- .NET assemblies are often easy to decompile and inspect directly
- metadata alone can reveal internal API paths and framework details
- recovered credentials should always be checked for:
  - direct service access
  - password reuse
  - lateral movement opportunities

---

# Tags

#thick-client
#service-connections
#connection-strings
#gdb
#peda
#odbc
#mssql
#dotnet
#dnspy
#dll
#elf
#credentials
#lateral-movement
#attacking-common-apps
#pentesting
#ctf
#obsidian