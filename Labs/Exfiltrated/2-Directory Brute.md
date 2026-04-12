```bash
gobuster dir -u http://exfiltrated.offsec -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -r 
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://exfiltrated.offsec
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Follow Redirect:         true
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
.gitignore           (Status: 200) [Size: 247]
.hta                 (Status: 403) [Size: 283]
.htaccess            (Status: 403) [Size: 283]
.htpasswd            (Status: 403) [Size: 283]
.well-known/assetlinks.json (Status: 200) [Size: 76]
.well-known/host-meta.json (Status: 200) [Size: 76]
.well-known/jwks.json (Status: 200) [Size: 76]
0                    (Status: 200) [Size: 21687]
_framework/blazor.boot.json (Status: 200) [Size: 76]
about                (Status: 200) [Size: 18975]
actions              (Status: 200) [Size: 0]
api                  (Status: 403) [Size: 17381]
api/experiments      (Status: 403) [Size: 17393]
api/experiments/configurations (Status: 403) [Size: 17408]
blog                 (Status: 200) [Size: 19417]
captcha              (Status: 200) [Size: 1868]
confirm              (Status: 403) [Size: 17407]
contribute.json      (Status: 200) [Size: 76]
cron                 (Status: 200) [Size: 43]
crossdomain.xml      (Status: 200) [Size: 104]
en                   (Status: 200) [Size: 21687]
favicon.ico          (Status: 200) [Size: 1150]
favorites            (Status: 200) [Size: 18959]
forgot               (Status: 200) [Size: 20406]
help                 (Status: 200) [Size: 18950]
icons                (Status: 403) [Size: 283]
index                (Status: 200) [Size: 21693]
index.php            (Status: 200) [Size: 21693]
install              (Status: 403) [Size: 10]
jwks.json            (Status: 200) [Size: 76]
login                (Status: 200) [Size: 18023]
logout               (Status: 200) [Size: 21687]
members              (Status: 200) [Size: 25299]
node_modules/.package-lock.json (Status: 200) [Size: 76]
npm-shrinkwrap.json  (Status: 200) [Size: 76]
package-lock.json    (Status: 200) [Size: 76]
package.json         (Status: 200) [Size: 76]
panel                (Status: 200) [Size: 6163]
policy               (Status: 200) [Size: 19008]
profile              (Status: 403) [Size: 17392]
redirect             (Status: 200) [Size: 1048]
registration         (Status: 200) [Size: 20800]
robots.txt           (Status: 200) [Size: 142]
search               (Status: 200) [Size: 19398]
server-status        (Status: 403) [Size: 283]
sitemap.xml          (Status: 200) [Size: 637]
tag                  (Status: 200) [Size: 18905]
terms                (Status: 200) [Size: 18995]
updates              (Status: 403) [Size: 283]
version.json         (Status: 200) [Size: 76]
web.xml              (Status: 200) [Size: 104]
webpack.manifest.json (Status: 200) [Size: 76]
Progress: 4750 / 4750 (100.00%)

```