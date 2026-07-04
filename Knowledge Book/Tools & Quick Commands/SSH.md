## Overview

SSH provides several features that are useful during penetration testing, internal assessments, and lab environments.

Common uses include:

- establishing SOCKS proxies
    
- pivoting into internal networks
    
- forwarding local ports
    
- exposing local services to remote hosts
    
- simplifying repeated connections with `~/.ssh/config`
    
- authenticating with SSH keys
    
- forwarding graphical applications over X11
    
- using and inspecting SSH agents
    
- identifying opportunities to reuse forwarded or exposed agent sockets
    

> [!important]  
> These notes supplement the SSH manual pages and focus on practical assessment use.

---

# SSH Forwarding Concepts

SSH forwarding can be divided into three main types:

|Type|Purpose|
|---|---|
|Dynamic forwarding|Creates a SOCKS proxy through the SSH server|
|Local forwarding|Makes a remote service available on the SSH client|
|Remote forwarding|Makes a local service available on the SSH server|

General syntax:

```text
Dynamic:
ssh -D <local_bind_ip>:<local_port> <user>@<ssh_server>

Local:
ssh -L <local_bind_ip>:<local_port>:<destination_host>:<destination_port> <user>@<ssh_server>

Remote:
ssh -R <remote_bind_ip>:<remote_port>:<destination_host>:<destination_port> <user>@<ssh_server>
```

---

# 1. Dynamic Port Forwarding – SOCKS Proxy

## Purpose

Dynamic forwarding creates a SOCKS proxy on the SSH client.

Traffic sent through the SOCKS proxy is routed through the SSH server, allowing access to hosts and services reachable from the SSH server.

General flow:

```text
Assessment host
↓
Local SOCKS proxy
↓
SSH tunnel
↓
Compromised / accessible SSH server
↓
Internal network
```

---

## Command Line

Create a SOCKS proxy on:

```text
127.0.0.1:1080
```

through the remote SSH server:

```bash
ssh -D 127.0.0.1:1080 user@10.0.0.1
```

If the username is implicit:

```bash
ssh -D 127.0.0.1:1080 10.0.0.1
```

---

## SSH Config

```sshconfig
Host 10.0.0.1
    DynamicForward 127.0.0.1:1080
```

Then connect normally:

```bash
ssh 10.0.0.1
```

---

## Use with Proxy-Aware Tools

Example using ProxyChains:

```bash
proxychains nmap -sT -Pn 10.0.0.2
```

Example using `tsocks`:

```bash
tsocks rdesktop 10.0.0.2
```

---

## When to Use

Use dynamic forwarding when:

```text
You have SSH access to a pivot host
The pivot host can reach an internal network
You need to use several different tools or ports
You want a general SOCKS-based pivot
```

---

# 2. Local Port Forwarding

## Purpose

Local forwarding creates a listener on the SSH client.

Connections to the local listener are forwarded through the SSH server to a destination reachable from the SSH server.

General flow:

```text
Local assessment host
↓
Local listening port
↓
SSH tunnel
↓
SSH server
↓
Remote destination service
```

Syntax:

```bash
ssh -L <local_bind_ip>:<local_port>:<destination_host>:<destination_port> user@<ssh_server>
```

---

# Example 1 – Access a Service on the SSH Server

A service runs on the SSH server at:

```text
127.0.0.1:1521
```

Expose it locally on:

```text
127.0.0.1:10521
```

Command:

```bash
ssh -L 127.0.0.1:10521:127.0.0.1:1521 user@10.0.0.1
```

Connect locally:

```bash
nc -nv 127.0.0.1 10521
```

SSH config:

```sshconfig
Host 10.0.0.1
    LocalForward 127.0.0.1:10521 127.0.0.1:1521
```

---

# Example 2 – Expose the Forward to Other Local Hosts

Bind the local listener to all interfaces:

```bash
ssh -L 0.0.0.0:10521:127.0.0.1:1521 user@10.0.0.1
```

SSH config:

```sshconfig
Host 10.0.0.1
    LocalForward 0.0.0.0:10521 127.0.0.1:1521
```

Other hosts that can reach the SSH client may now connect to:

```text
<ssh_client_ip>:10521
```

> [!warning]  
> Binding to `0.0.0.0` exposes the forwarded service to other reachable hosts. Use only when necessary.

---

# Example 3 – Access Another Internal Host

The SSH server can reach:

```text
10.0.0.99:1521
```

Expose that service locally on:

```text
127.0.0.1:10521
```

Command:

```bash
ssh -L 127.0.0.1:10521:10.0.0.99:1521 user@10.0.0.1
```

SSH config:

```sshconfig
Host 10.0.0.1
    LocalForward 127.0.0.1:10521 10.0.0.99:1521
```

Connect through the tunnel:

```bash
nc -nv 127.0.0.1 10521
```

---

## Local Forwarding Decision Point

Use local forwarding when:

```text
You need access to one specific service
The destination is reachable from the SSH server
You want the service to appear on your local machine
The target service is bound only to localhost on the remote system
```

---

# 3. Remote Port Forwarding

## Purpose

Remote forwarding creates a listener on the SSH server.

Connections to that listener are forwarded through the SSH connection to a host and service reachable from the SSH client.

General flow:

```text
Remote SSH server
↓
Remote listening port
↓
SSH tunnel
↓
SSH client
↓
Local or client-side destination service
```

Syntax:

```bash
ssh -R <remote_bind_ip>:<remote_port>:<destination_host>:<destination_port> user@<ssh_server>
```

---

# Example 1 – Expose a Local Web Server to the SSH Server

A web server is running on the SSH client at:

```text
127.0.0.1:80
```

Make it accessible on the SSH server at:

```text
127.0.0.1:8000
```

Command:

```bash
ssh -R 127.0.0.1:8000:127.0.0.1:80 user@10.0.0.1
```

SSH config:

```sshconfig
Host 10.0.0.1
    RemoteForward 127.0.0.1:8000 127.0.0.1:80
```

From the SSH server:

```bash
curl http://127.0.0.1:8000/
```

---

# Example 2 – Expose Another Client-Side Host

The SSH client can reach:

```text
172.16.0.99:80
```

Make it available to the SSH server at:

```text
127.0.0.1:8000
```

Command:

```bash
ssh -R 127.0.0.1:8000:172.16.0.99:80 user@10.0.0.1
```

SSH config:

```sshconfig
Host 10.0.0.1
    RemoteForward 127.0.0.1:8000 172.16.0.99:80
```

From the SSH server:

```bash
curl http://127.0.0.1:8000/
```

---

# Example 3 – Expose the Remote Listener to Other Hosts

Bind the remote listener to all interfaces:

```bash
ssh -R 0.0.0.0:8000:172.16.0.99:80 user@10.0.0.1
```

SSH config:

```sshconfig
Host 10.0.0.1
    RemoteForward 0.0.0.0:8000 172.16.0.99:80
```

Other hosts that can reach the SSH server may now connect to:

```text
<ssh_server_ip>:8000
```

> [!warning]  
> This can expose the forwarded service to the wider network. SSH server configuration may also need `GatewayPorts` enabled.

---

## When to Use Remote Forwarding

Use remote forwarding when:

```text
The remote host needs access to a service on your machine
You want the remote host to download tools from your HTTP server
You need to expose a client-side internal service to the remote network
The target cannot directly reach your normal listener or web server
```

---

# Binding to Privileged Ports

On Linux, binding to TCP or UDP ports below `1024` normally requires root privileges.

Examples of privileged ports:

```text
22
53
80
443
445
```

Use higher ports when running as a standard user:

```text
8000
8080
10521
1080
9001
```

---

# 4. SSH Client Configuration

## File Location

SSH client settings can be stored in:

```text
~/.ssh/config
```

This avoids repeatedly typing long command-line options.

It also simplifies use with tools such as:

```text
scp
rsync
sftp
git
```

---

## Example Configuration

```sshconfig
Host pivot
    HostName 10.0.0.1
    Port 2222
    User ptm
    ForwardX11 yes
    DynamicForward 127.0.0.1:1080
    RemoteForward 8000 127.0.0.1:80
    LocalForward 10521 10.0.0.99:1521
```

Connect with:

```bash
ssh pivot
```

Copy a file:

```bash
scp tool.exe pivot:/tmp/
```

Use rsync:

```bash
rsync -av ./tools/ pivot:/tmp/tools/
```

---

# Useful SSH Config Options

|Option|Purpose|
|---|---|
|`Host`|Local alias for the connection|
|`HostName`|Real IP address or hostname|
|`User`|SSH username|
|`Port`|SSH service port|
|`IdentityFile`|Private key path|
|`DynamicForward`|SOCKS proxy|
|`LocalForward`|Local port forwarding|
|`RemoteForward`|Remote port forwarding|
|`ForwardX11`|Enable X11 forwarding|
|`ForwardAgent`|Enable SSH agent forwarding|

---

# 5. SSH Keys and authorized_keys

## Overview

SSH key authentication uses:

```text
private key
public key
```

The private key remains with the client.

The public key is added to the target account’s:

```text
~/.ssh/authorized_keys
```

---

## Generate a Key Pair

```bash
ssh-keygen -f mykey
```

Files created:

```text
mykey
mykey.pub
```

Display public key:

```bash
cat mykey.pub
```

---

## Add Public Key to Target

On the target:

```bash
mkdir -p ~/.ssh
```

```bash
echo '<public_key>' >> ~/.ssh/authorized_keys
```

Set permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

---

## Connect with Private Key

```bash
ssh -i mykey user@10.0.0.1
```

If SSH runs on another port:

```bash
ssh -i mykey -p 2222 user@10.0.0.1
```

---

## Shorter Public Keys

Where an arbitrary file-write primitive has a strict size limit, a shorter key may be useful.

Example:

```bash
ssh-keygen -f mykey -t rsa -b 768
```

Display:

```bash
cat mykey.pub
```

The trailing comment may be removed to shorten the key:

```text
user@host
```

> [!warning]  
> Small RSA key sizes are cryptographically weak and may be rejected by modern SSH implementations. Use only in controlled legacy lab situations where required.

---

## authorized_keys Permission Problems

SSH may ignore `authorized_keys` if permissions are too open.

Recommended permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Check ownership:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

Fix ownership:

```bash
chown -R <user>:<group> ~/.ssh
```

---

# 6. X11 Forwarding

## Purpose

X11 forwarding allows graphical programs running on the SSH server to display on the SSH client.

Examples:

```text
Firefox
text editors
file managers
GUI administration tools
```

The SSH client must have access to an X server.

---

## Standard X11 Forwarding

```bash
ssh -X user@10.0.0.1
```

Run a graphical application:

```bash
firefox
```

SSH config:

```sshconfig
Host 10.0.0.1
    ForwardX11 yes
```

---

## Trusted X11 Forwarding

```bash
ssh -Y user@10.0.0.1
```

SSH config:

```sshconfig
Host 10.0.0.1
    ForwardX11Trusted yes
```

> [!warning]  
> `-Y` grants the remote X11 client more trust than `-X`. Use only when required.

---

# 7. SSH Agents

## Overview

An SSH agent holds private keys in memory and performs authentication on behalf of the user.

Benefits:

```text
Private keys can remain encrypted on disk
Passphrases do not need to be entered for every connection
Multiple SSH sessions can reuse loaded keys
```

Risk:

```text
Anyone who can access the agent socket may be able to authenticate using the loaded keys.
```

---

## Start an SSH Agent

```bash
eval "$(ssh-agent)"
```

Alternative shell syntax:

```bash
eval `ssh-agent`
```

---

## Add a Private Key

```bash
ssh-add ~/dir/mykey
```

List loaded keys:

```bash
ssh-add -l
```

List public keys:

```bash
ssh-add -L
```

Remove all loaded keys:

```bash
ssh-add -D
```

---

# 8. Inspecting and Reusing SSH Agent Sockets

## Agent Socket

SSH agents listen on a Unix socket.

The socket path is stored in:

```text
SSH_AUTH_SOCK
```

Example:

```text
/tmp/ssh-tqiEl28473/agent.28473
```

Check current value:

```bash
echo $SSH_AUTH_SOCK
```

---

## Use an Accessible Agent Socket

Set the socket:

```bash
export SSH_AUTH_SOCK=/tmp/ssh-tqiEl28473/agent.28473
```

List loaded keys:

```bash
ssh-add -l
```

Attempt authentication:

```bash
ssh user@host
```

If the destination trusts a loaded key, SSH may authenticate without requiring the private key file.

---

## Find SSH Agent Sockets

Search running process environments:

```bash
ps auxeww | grep ssh-agent | grep SSH_AUTH_SOCK | sed 's/.*SSH_AUTH_SOCK=//' | cut -f 1 -d ' '
```

Search temporary directories:

```bash
find /tmp -type s -name 'agent.*' 2>/dev/null
```

Check processes:

```bash
ps aux | grep ssh-agent
```

---

## Identify Possible Destinations

Review known hosts:

```bash
cat ~/.ssh/known_hosts
```

Review SSH config:

```bash
cat ~/.ssh/config
```

Review shell history:

```bash
grep -i ssh ~/.bash_history
```

Look for:

```text
hostnames
usernames
jump hosts
custom SSH ports
IdentityFile paths
```

---

# 9. SSH Agent Forwarding

## Purpose

Agent forwarding allows the remote SSH session to use the SSH agent running on the client.

Command:

```bash
ssh -A user@10.0.0.1
```

SSH config:

```sshconfig
Host 10.0.0.1
    ForwardAgent yes
```

On the remote system:

```bash
echo $SSH_AUTH_SOCK
```

```bash
ssh-add -l
```

---

## Security Risk

Agent forwarding does not copy the private key to the remote server, but the remote system can use the forwarded agent while the session is active.

A privileged user on the remote host may be able to access the forwarded socket and authenticate to other hosts using the forwarded keys.

> [!warning]  
> Avoid agent forwarding to untrusted or compromised systems, especially when using sensitive keys.

---

# 10. SSH Connection Options

## Disable Interactive Shell

Use forwarding only:

```bash
ssh -N -L 127.0.0.1:8080:127.0.0.1:80 user@10.0.0.1
```

`-N` means:

```text
Do not execute a remote command.
```

---

## Run SSH in Background

```bash
ssh -fN -L 127.0.0.1:8080:127.0.0.1:80 user@10.0.0.1
```

Options:

|Option|Purpose|
|---|---|
|`-f`|Move SSH process to background|
|`-N`|Do not execute remote command|
|`-T`|Disable pseudo-terminal allocation|
|`-v`|Verbose output|
|`-vvv`|Maximum useful debugging output|
|`-p`|Specify SSH port|
|`-i`|Specify private key|
|`-D`|Dynamic forwarding|
|`-L`|Local forwarding|
|`-R`|Remote forwarding|
|`-A`|Agent forwarding|
|`-X`|X11 forwarding|
|`-Y`|Trusted X11 forwarding|

---

## Fail if Forwarding Cannot Be Created

```bash
ssh -o ExitOnForwardFailure=yes -N -L 127.0.0.1:8080:127.0.0.1:80 user@10.0.0.1
```

Useful for scripting and troubleshooting.

---

## Keep Tunnel Alive

```bash
ssh -o ServerAliveInterval=30 -o ServerAliveCountMax=3 user@10.0.0.1
```

SSH config:

```sshconfig
Host pivot
    ServerAliveInterval 30
    ServerAliveCountMax 3
```

---

# Practical Decision Tree

## Need Access to Many Internal Services?

Use dynamic forwarding:

```bash
ssh -D 127.0.0.1:1080 user@<pivot>
```

---

## Need Access to One Remote Service?

Use local forwarding:

```bash
ssh -L 127.0.0.1:<local_port>:<target_host>:<target_port> user@<pivot>
```

---

## Need the Remote Host to Reach Your Service?

Use remote forwarding:

```bash
ssh -R 127.0.0.1:<remote_port>:127.0.0.1:<local_port> user@<target>
```

---

## Need Persistent SSH Access with a Key?

Add the public key to:

```text
~/.ssh/authorized_keys
```

Then connect with:

```bash
ssh -i <private_key> user@<target>
```

---

## Found an ssh-agent Process?

Find the socket:

```bash
find /tmp -type s -name 'agent.*' 2>/dev/null
```

Set it:

```bash
export SSH_AUTH_SOCK=<socket>
```

Inspect keys:

```bash
ssh-add -l
```

Review likely targets:

```bash
cat ~/.ssh/known_hosts
cat ~/.ssh/config
```

---

# Troubleshooting

## Tunnel Opens but Service Does Not Respond

Check:

```text
destination host is reachable from the SSH server
destination port is correct
service is listening
bind address is correct
local port is not already in use
firewall permits traffic
```

Use verbose mode:

```bash
ssh -vvv -L 127.0.0.1:8080:10.0.0.99:80 user@10.0.0.1
```

---

## Remote Forward Is Bound Only to Localhost

The SSH server may restrict remote forwards to loopback.

Check server setting:

```text
GatewayPorts
```

Possible server configuration:

```text
GatewayPorts yes
```

or:

```text
GatewayPorts clientspecified
```

---

## Permission Denied Binding Port

Ports below `1024` generally require root.

Use a higher port:

```text
8080
8443
10521
```

---

## authorized_keys Is Ignored

Check:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Check ownership:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

Review SSH logs where accessible:

```bash
journalctl -u ssh
```

or:

```bash
tail -f /var/log/auth.log
```

---

## Agent Socket Does Not Work

Check:

```text
socket belongs to current or accessible user
agent process is still running
SSH_AUTH_SOCK path is correct
socket permissions allow access
keys are loaded
```

Commands:

```bash
ls -l $SSH_AUTH_SOCK
ssh-add -l
```

---

## SOCKS Proxy Does Not Work

Confirm SSH tunnel:

```bash
ss -lntp | grep 1080
```

Check ProxyChains configuration:

```text
socks5 127.0.0.1 1080
```

Use TCP-based scans:

```bash
proxychains nmap -sT -Pn <internal_host>
```

Avoid raw packet scans through SOCKS:

```text
-sS
UDP scanning
ICMP-based discovery
```

---

# Quick Command Reference

## SOCKS proxy

```bash
ssh -D 127.0.0.1:1080 user@<pivot>
```

## Local forward to service on SSH server

```bash
ssh -L 127.0.0.1:8080:127.0.0.1:80 user@<pivot>
```

## Local forward to another internal host

```bash
ssh -L 127.0.0.1:8080:10.0.0.99:80 user@<pivot>
```

## Remote forward local service

```bash
ssh -R 127.0.0.1:8000:127.0.0.1:80 user@<target>
```

## Remote forward another client-side host

```bash
ssh -R 127.0.0.1:8000:172.16.0.99:80 user@<target>
```

## Forwarding only

```bash
ssh -N -L 127.0.0.1:8080:127.0.0.1:80 user@<pivot>
```

## Background tunnel

```bash
ssh -fN -L 127.0.0.1:8080:127.0.0.1:80 user@<pivot>
```

## Generate key

```bash
ssh-keygen -f mykey
```

## Connect with key

```bash
ssh -i mykey user@<target>
```

## Start SSH agent

```bash
eval "$(ssh-agent)"
```

## Add key to agent

```bash
ssh-add ~/dir/mykey
```

## List agent keys

```bash
ssh-add -l
```

## Set agent socket

```bash
export SSH_AUTH_SOCK=<socket_path>
```

## Agent forwarding

```bash
ssh -A user@<target>
```

## X11 forwarding

```bash
ssh -X user@<target>
```

## Trusted X11 forwarding

```bash
ssh -Y user@<target>
```

## Verbose debugging

```bash
ssh -vvv user@<target>
```

---

# Security Notes

- Bind local forwards to `127.0.0.1` unless other systems need access.
    
- Binding to `0.0.0.0` can expose forwarded services.
    
- Remote forwarding may require `GatewayPorts`.
    
- Avoid forwarding sensitive SSH agents to untrusted hosts.
    
- Protect private keys with restrictive permissions:
    

```bash
chmod 600 <private_key>
```

- Protect SSH configuration:
    

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config
chmod 600 ~/.ssh/authorized_keys
```

- Remove temporary keys and tunnels after use.
    
- Avoid leaving assessment keys in `authorized_keys`.
    
- Document forwarded ports and pivot paths.
    

---

# Cleanup Checklist

Stop SSH tunnel processes:

```bash
ps aux | grep ssh
```

```bash
kill <pid>
```

Remove temporary keys from targets:

```bash
sed -i '\|<public_key_fragment>|d' ~/.ssh/authorized_keys
```

Remove local temporary keys:

```bash
rm -f mykey mykey.pub
```

Clear loaded agent keys:

```bash
ssh-add -D
```

Stop agent:

```bash
ssh-agent -k
```

Review:

```text
local forwarding listeners
remote forwarding listeners
SOCKS proxies
authorized_keys changes
temporary SSH config entries
agent forwarding sessions
```

---

# Key Takeaways

- Dynamic forwarding creates a SOCKS proxy for broad network access.
    
- Local forwarding exposes a remote service on the SSH client.
    
- Remote forwarding exposes a client-side service on the SSH server.
    
- Bind to `127.0.0.1` by default to reduce exposure.
    
- `~/.ssh/config` simplifies repeated connections and forwarding.
    
- `authorized_keys` enables key-based access when permissions are correct.
    
- SSH agents store usable private-key identities in memory.
    
- Anyone who can access an agent socket may be able to use loaded keys.
    
- Agent forwarding should not be used with sensitive keys on untrusted hosts.
    
- Use `-N`, `-f`, and `ExitOnForwardFailure` for reliable tunnels.
    

---

# Related Notes

- [[SSH]]
    
- [[Pivoting]]
    
- [[Port Forwarding]]
    
- [[SOCKS Proxy]]
    
- [[ProxyChains]]
    
- [[SSH Keys]]
    
- [[SSH Agent]]
    
- [[SSH Agent Forwarding]]
    
- [[X11 Forwarding]]
    
- [[Linux Post-Exploitation]]
    
- [[Lateral Movement]]
    

---

# Tags

#ssh  
#pivoting  
#port-forwarding  
#socks  
#proxychains  
#ssh-keys  
#authorized-keys  
#ssh-agent  
#agent-forwarding  
#x11  
#lateral-movement  
#tools  
#quick-commands  
#pentesting