# Common Ports

A quick-reference list of common TCP/UDP ports encountered during enumeration and penetration testing.

## Well-Known Ports

| Port | Protocol | Service | Description |
|---:|---|---|---|
| 20 | TCP | FTP Data | File transfer data channel. |
| 21 | TCP | FTP | File transfer control channel. |
| 22 | TCP | SSH | Secure remote shell access. |
| 23 | TCP | Telnet | Insecure remote shell access. |
| 25 | TCP | SMTP | Mail transfer between servers. |
| 53 | TCP/UDP | DNS | Domain name resolution. |
| 67 | UDP | DHCP Server | Assigns IP configuration to clients. |
| 68 | UDP | DHCP Client | Receives DHCP configuration. |
| 69 | UDP | TFTP | Simple unauthenticated file transfer. |
| 80 | TCP | HTTP | Unencrypted web traffic. |
| 88 | TCP/UDP | Kerberos | Authentication in Active Directory. |
| 110 | TCP | POP3 | Email retrieval from mailbox. |
| 111 | TCP/UDP | RPCbind | Maps RPC services to ports. |
| 119 | TCP | NNTP | Usenet/news transfer. |
| 123 | UDP | NTP | Network time synchronisation. |
| 135 | TCP | MSRPC | Microsoft RPC endpoint mapper. |
| 137 | UDP | NetBIOS Name Service | NetBIOS name resolution. |
| 138 | UDP | NetBIOS Datagram | NetBIOS datagram service. |
| 139 | TCP | NetBIOS Session | Legacy Windows file sharing. |
| 143 | TCP | IMAP | Email retrieval and synchronisation. |
| 161 | UDP | SNMP | Network device monitoring. |
| 162 | UDP | SNMP Trap | SNMP alert messages. |
| 179 | TCP | BGP | Internet routing protocol. |
| 389 | TCP/UDP | LDAP | Directory service queries. |
| 443 | TCP | HTTPS | Encrypted web traffic. |
| 445 | TCP | SMB | Windows file and printer sharing. |
| 465 | TCP | SMTPS | SMTP over SSL/TLS. |
| 514 | UDP | Syslog | Centralised log messages. |
| 515 | TCP | LPD | Network printing. |
| 587 | TCP | SMTP Submission | Authenticated mail submission. |
| 631 | TCP/UDP | IPP | Internet printing protocol. |
| 636 | TCP | LDAPS | LDAP over SSL/TLS. |
| 873 | TCP | rsync | File synchronisation. |
| 902 | TCP | VMware | VMware remote console/service access. |
| 989 | TCP | FTPS Data | FTP data over SSL/TLS. |
| 990 | TCP | FTPS Control | FTP control over SSL/TLS. |
| 993 | TCP | IMAPS | IMAP over SSL/TLS. |
| 995 | TCP | POP3S | POP3 over SSL/TLS. |

## Common Application and Admin Ports

| Port | Protocol | Service | Description |
|---:|---|---|---|
| 1025+ | TCP/UDP | Ephemeral/RPC | Dynamic client or RPC service ports. |
| 1080 | TCP | SOCKS | Proxy traffic forwarding. |
| 1194 | UDP | OpenVPN | VPN tunnelling. |
| 1433 | TCP | MSSQL | Microsoft SQL Server. |
| 1434 | UDP | MSSQL Browser | SQL Server instance discovery. |
| 1521 | TCP | Oracle DB | Oracle database listener. |
| 1723 | TCP | PPTP | Legacy VPN tunnelling. |
| 1812 | UDP | RADIUS Auth | Network authentication. |
| 1813 | UDP | RADIUS Accounting | Network accounting logs. |
| 2049 | TCP/UDP | NFS | Unix/Linux file sharing. |
| 2082 | TCP | cPanel | cPanel web interface. |
| 2083 | TCP | cPanel SSL | cPanel over SSL/TLS. |
| 2375 | TCP | Docker API | Unencrypted Docker daemon API. |
| 2376 | TCP | Docker API TLS | Docker daemon API over TLS. |
| 3000 | TCP | Dev Web App | Common development web server. |
| 3128 | TCP | Squid Proxy | Web proxy service. |
| 3268 | TCP | Global Catalog | Active Directory global catalog. |
| 3269 | TCP | Global Catalog SSL | Global catalog over SSL/TLS. |
| 3306 | TCP | MySQL | MySQL database service. |
| 3389 | TCP/UDP | RDP | Windows remote desktop. |
| 3690 | TCP | SVN | Subversion version control. |
| 4369 | TCP | Erlang Port Mapper | Erlang node discovery. |
| 4444 | TCP | Metasploit/Backdoor | Common listener or backdoor port. |
| 5000 | TCP | Flask/Dev Server | Common Python development server. |
| 5060 | TCP/UDP | SIP | VoIP signalling. |
| 5061 | TCP | SIP TLS | Encrypted VoIP signalling. |
| 5432 | TCP | PostgreSQL | PostgreSQL database service. |
| 5601 | TCP | Kibana | Elastic/Kibana web interface. |
| 5672 | TCP | AMQP | RabbitMQ message broker. |
| 5900 | TCP | VNC | Remote desktop access. |
| 5985 | TCP | WinRM HTTP | Windows remote management. |
| 5986 | TCP | WinRM HTTPS | Encrypted Windows remote management. |
| 6000 | TCP | X11 | X Window System display. |
| 6379 | TCP | Redis | Redis key-value database. |
| 6443 | TCP | Kubernetes API | Kubernetes control-plane API. |
| 6667 | TCP | IRC | Internet relay chat. |
| 8000 | TCP | HTTP Alt | Alternative web server port. |
| 8008 | TCP | HTTP Alt | Alternative web server port. |
| 8080 | TCP | HTTP Proxy/Web | Alternative web or proxy service. |
| 8081 | TCP | HTTP Alt | Alternative web management port. |
| 8443 | TCP | HTTPS Alt | Alternative HTTPS service. |
| 8888 | TCP | HTTP Alt | Alternative web interface. |
| 9000 | TCP | PHP-FPM/SonarQube | App backend or admin service. |
| 9090 | TCP | Prometheus | Prometheus monitoring interface. |
| 9200 | TCP | Elasticsearch | Elasticsearch REST API. |
| 9300 | TCP | Elasticsearch Node | Elasticsearch cluster transport. |
| 10000 | TCP | Webmin | Web-based Unix/Linux administration. |
| 11211 | TCP/UDP | Memcached | In-memory cache service. |
| 27017 | TCP | MongoDB | MongoDB database service. |
| 27018 | TCP | MongoDB | MongoDB shard/secondary service. |
| 50000 | TCP | SAP/DB2 | Common enterprise service port. |

## Active Directory Focus

| Port | Protocol | Service | Description |
|---:|---|---|---|
| 53 | TCP/UDP | DNS | AD domain name resolution. |
| 88 | TCP/UDP | Kerberos | Domain authentication tickets. |
| 135 | TCP | MSRPC | RPC endpoint mapping. |
| 139 | TCP | NetBIOS | Legacy SMB session service. |
| 389 | TCP/UDP | LDAP | Domain directory queries. |
| 445 | TCP | SMB | File shares and remote admin. |
| 464 | TCP/UDP | Kerberos Password | Kerberos password changes. |
| 593 | TCP | RPC over HTTP | Remote procedure calls over HTTP. |
| 636 | TCP | LDAPS | Encrypted LDAP queries. |
| 3268 | TCP | Global Catalog | Forest-wide AD searches. |
| 3269 | TCP | Global Catalog SSL | Encrypted global catalog searches. |
| 3389 | TCP/UDP | RDP | Remote desktop access. |
| 5985 | TCP | WinRM HTTP | PowerShell remoting over HTTP. |
| 5986 | TCP | WinRM HTTPS | PowerShell remoting over HTTPS. |

## Quick Enumeration Notes

| Finding | What to Check |
|---|---|
| `21/tcp FTP` | Anonymous login, writable directories, banner version. |
| `22/tcp SSH` | Weak credentials, old versions, key reuse. |
| `53/tcp/udp DNS` | Zone transfer, subdomain enumeration, AD records. |
| `80/443 HTTP(S)` | Virtual hosts, directories, technologies, auth bypass. |
| `88/tcp Kerberos` | AS-REP roasting, Kerberoasting, username validity. |
| `111/tcp RPCbind` | NFS exports and exposed RPC services. |
| `135/139/445 Windows` | SMB shares, null sessions, signing, relay potential. |
| `161/udp SNMP` | Public/private community strings, device info leaks. |
| `389/636 LDAP(S)` | Domain users, groups, computers, policies. |
| `2049 NFS` | Export permissions, `no_root_squash`, sensitive files. |
| `3306 MySQL` | Default creds, remote root access, database dumps. |
| `3389 RDP` | Weak creds, NLA status, exposed admin access. |
| `5985/5986 WinRM` | Valid domain creds for remote shell access. |
| `6379 Redis` | No auth, file write, key dumping. |
| `9200 Elasticsearch` | Unauthenticated data exposure. |
| `27017 MongoDB` | No auth, exposed databases. |

## Useful Nmap Commands

```bash
# Fast scan common ports
nmap -sV -sC -oA nmap/common <target>

# Full TCP port scan
nmap -p- --min-rate 5000 -oA nmap/full-tcp <target>

# Service scan discovered ports
nmap -sV -sC -p <ports> -oA nmap/services <target>

# UDP top ports
sudo nmap -sU --top-ports 100 -oA nmap/udp-top100 <target>

# Aggressive scan
nmap -A -oA nmap/aggressive <target>
```

## Tags

#pentesting #enumeration #ports #nmap #cheatsheet