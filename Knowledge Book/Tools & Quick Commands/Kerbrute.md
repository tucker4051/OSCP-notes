# Kerbrute

## Purpose
Kerberos user enumeration and password spraying.

## Common Commands

### Enumerate valid users
`kerbrute userenum -d domain.local users.txt

### Password spray
`kerbrute passwordspray -d domain.local users.txt 'Password123'

### Specify DC
`kerbrute userenum --dc 10.10.10.10 -d domain.local users.txt

## Notes
Does not trigger lockouts as quickly as SMB spraying.
Very useful early-stage AD enumeration.