# Enumeration – IMAP / POP3

## Overview

IMAP and POP3 are common mail retrieval protocols used to access email stored on a mail server.

Default ports:

| Port | Service |
|------|---------|
| 110 | POP3 |
| 995 | POP3S (POP3 over SSL/TLS) |
| 143 | IMAP |
| 993 | IMAPS (IMAP over SSL/TLS) |

The encrypted variants (`993` and `995`) protect traffic between client and server using SSL/TLS.

## Protocol differences

### POP3
- designed mainly for downloading mail
- commonly pulls messages from server to client
- simpler protocol
- often used for basic mailbox access

### IMAP
- designed for remote mailbox management
- supports folders, flags, status checks, and selective retrieval
- better for server-side mail handling and multi-device access

Think of POP3 as downloading letters from a post office box, while IMAP lets you organise the entire mailbox remotely.

---

# 1. Initial Enumeration

## Nmap scan

Start by checking the common POP3 and IMAP ports:

```bash
sudo nmap <TARGET_IP> -sV -p110,143,993,995 -sC
```

## What this gives you

- open mail ports
- service versions
- SSL/TLS support
- possible banners
- default NSE script output

---

# 2. Connecting to the Service

## cURL against IMAPS

```bash
curl -k 'imaps://<TARGET_IP>' --user user:p4ssw0rd -v
```

## Why use this

Useful for:

- quickly testing credentials
- checking whether IMAPS is enabled
- viewing server responses

`-k` ignores certificate validation errors.

---

# 3. SSL/TLS Interaction

To interact with encrypted POP3/IMAP manually, use `openssl` or `ncat`.

## POP3S

```bash
openssl s_client -connect <TARGET_IP>:995
```

## IMAPS

```bash
openssl s_client -connect <TARGET_IP>:993
```

## Why this matters

Useful for:

- banner grabbing
- manual authentication
- command testing
- checking TLS configuration
- verifying certificates

---

# 4. IMAP Basics

IMAP commands usually begin with a client-defined tag like:

```text
A1
```

The server uses the same tag in its response so the client can match replies to requests.

Example:

```text
A1 LOGIN username password
```

---

# 5. IMAP Authentication

## Basic login

```text
A1 LOGIN username password
```

## Quoted values

Use quotes if values contain spaces or special characters:

```text
A1 LOGIN "username" "pass word"
```

## Important note

A double quote inside quoted values must be escaped with `\`.

---

# 6. IMAP Mailbox Enumeration

## List all folders / mailboxes

```text
A1 LIST "" *
```

## List INBOX hierarchy

```text
A1 LIST INBOX *
```

## List a specific folder hierarchy

```text
A1 LIST "Archive" *
```

## Why this matters

This helps identify:

- mailbox structure
- archived mail
- custom folders
- potentially sensitive user-created folders

---

# 7. IMAP Mailbox Management

## Create mailbox

```text
A1 CREATE INBOX.Archive.2012
A1 CREATE "To Read"
```

## Delete mailbox

```text
A1 DELETE INBOX.Archive.2012
A1 DELETE "To Read"
```

## Rename mailbox

```text
A1 RENAME "INBOX.One" "INBOX.Two"
```

## Why this matters

Useful during authenticated testing to understand:

- access rights
- mailbox permissions
- whether user can create, rename, or remove folders

---

# 8. IMAP Subscription Enumeration

## List subscribed mailboxes

```text
A1 LSUB "" *
```

## Why this matters

Shows which folders the user is actively subscribed to, which may differ from all existing folders.

---

# 9. IMAP Mailbox Status

## Get mailbox status

```text
A1 STATUS INBOX (MESSAGES UNSEEN RECENT)
```

## Common useful fields

- `MESSAGES`
- `UNSEEN`
- `RECENT`

## Why this matters

Gives quick mailbox statistics without fully opening the mailbox.

---

# 10. Selecting a Mailbox

## Select mailbox

```text
A1 SELECT INBOX
```

## Why this matters

Required before many message retrieval operations.

It tells the server which mailbox you want to work in.

---

# 11. Listing Messages in IMAP

## Fetch message flags

```text
A1 FETCH 1:* (FLAGS)
```

## Fetch message flags using UIDs

```text
A1 UID FETCH 1:* (FLAGS)
```

## Why this matters

Useful for:

- enumerating message count
- identifying read/unread state
- mapping mailbox contents before retrieval

---

# 12. Retrieving IMAP Message Content

## Retrieve body text of message 2

```text
A1 FETCH 2 BODY[TEXT]
```

## Retrieve all data for message 2

```text
A1 FETCH 2 ALL
```

## Retrieve specific UID and message body without setting seen flag

```text
A1 UID FETCH 102 (UID RFC822.SIZE BODY.PEEK[])
```

## Why this matters

Useful for:

- reading message bodies
- reviewing metadata
- grabbing full emails
- avoiding changing message state with `BODY.PEEK[]`

---

# 13. Closing and Logging Out of IMAP

## Close current mailbox

```text
A1 CLOSE
```

## Logout

```text
A1 LOGOUT
```

---

# 14. POP3 Basics

POP3 is much simpler than IMAP.

Typical workflow:

1. authenticate
2. list messages
3. retrieve messages
4. optionally delete messages
5. quit

---

# 15. POP3 Authentication

## Username

```text
USER Stan
```

Expected response:

```text
+OK Please enter a password
```

## Password

```text
PASS SeCrEt
```

Expected response:

```text
+OK valid logon
```

## Logout

```text
QUIT
```

Expected response:

```text
+OK Bye-bye.
```

---

# 16. POP3 Message Enumeration

## Mailbox statistics

```text
STAT
```

Example response:

```text
+OK 2 320
```

Meaning:

- 2 messages
- 320 total octets

## List all messages

```text
LIST
```

Example response:

```text
+OK 2 messages (320 octets)
1 120
2 200
```

## List a single message size

```text
LIST 2
```

Example response:

```text
+OK 2 200
```

## Why this matters

Useful for quickly mapping mailbox contents before retrieval.

---

# 17. Retrieving POP3 Messages

## Retrieve full message

```text
RETR 1
```

Expected response:

```text
+OK 120 octets follow.
...
```

## Retrieve headers and first lines only

```text
TOP 2 1
```

Expected response includes:

- full headers
- first requested body lines

## Why this matters

`TOP` is useful for:

- previewing messages
- identifying interesting emails
- reducing noise during enumeration

---

# 18. POP3 Message Deletion and Reset

## Mark message for deletion

```text
DELE 2
```

Expected response:

```text
+OK message deleted
```

## Undo deletions

```text
RSET
```

Expected response:

```text
+OK maildrop has 2 messages (320 octets)
```

## Important note

Deletion is typically finalised when the session ends cleanly with `QUIT`.

---

# 19. POP3 Unique Message IDs

## Get UID for a message

```text
UIDL 1
```

Example response:

```text
+OK 1 6866N
```

## Why this matters

POP3 clients use `UIDL` to track which messages have already been downloaded.

Useful for:

- message tracking
- scripting
- identifying new mail across sessions

---

# 20. POP3 Utility Commands

## NOOP

```text
NOOP
```

Expected response:

```text
+OK
```

Server does nothing except confirm connection is alive.

## RSET

```text
RSET
```

Undeletes any messages marked for deletion in current session.

---

# 21. Practical Enumeration Workflow

## Step 1 – scan the ports

```bash
sudo nmap <TARGET_IP> -sV -p110,143,993,995 -sC
```

## Step 2 – identify encrypted vs unencrypted services

Look for:

- POP3 on `110` / `995`
- IMAP on `143` / `993`

## Step 3 – connect manually

```bash
openssl s_client -connect <TARGET_IP>:993
openssl s_client -connect <TARGET_IP>:995
```

## Step 4 – test authentication

For IMAP:

```text
A1 LOGIN username password
```

For POP3:

```text
USER username
PASS password
```

## Step 5 – enumerate mailboxes or messages

IMAP:

```text
A1 LIST "" *
A1 SELECT INBOX
A1 FETCH 1:* (FLAGS)
```

POP3:

```text
STAT
LIST
RETR 1
```

---

# 22. Useful Commands

## Nmap

```bash
sudo nmap <TARGET_IP> -sV -p110,143,993,995 -sC
```

## cURL IMAPS

```bash
curl -k 'imaps://<TARGET_IP>' --user user:p4ssw0rd -v
```

## OpenSSL POP3S

```bash
openssl s_client -connect <TARGET_IP>:995
```

## OpenSSL IMAPS

```bash
openssl s_client -connect <TARGET_IP>:993
```

## IMAP login

```text
A1 LOGIN username password
```

## IMAP list mailboxes

```text
A1 LIST "" *
```

## IMAP select mailbox

```text
A1 SELECT INBOX
```

## IMAP fetch messages

```text
A1 FETCH 1:* (FLAGS)
A1 FETCH 2 BODY[TEXT]
```

## POP3 login

```text
USER username
PASS password
```

## POP3 list messages

```text
STAT
LIST
```

## POP3 retrieve message

```text
RETR 1
TOP 2 1
```

---

# 23. Key Takeaways

- POP3 commonly uses ports `110` and `995`
- IMAP commonly uses ports `143` and `993`
- `993` and `995` are the encrypted variants
- IMAP is richer and better for mailbox management
- POP3 is simpler and focused on retrieval
- `openssl s_client` is useful for manual SSL/TLS interaction
- IMAP enumeration often starts with `LOGIN`, `LIST`, and `SELECT`
- POP3 enumeration often starts with `USER`, `PASS`, `STAT`, and `LIST`

---

# Tags

#enumeration
#imap
#pop3
#imaps
#pop3s
#email
#mail
#openssl
#nmap
#curl
#attacking-common-services
#pentesting
#ctf
#obsidian