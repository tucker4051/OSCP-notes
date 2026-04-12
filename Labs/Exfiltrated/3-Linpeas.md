## System Info
Sudo version 1.8.31
Ubuntu 20.04.2 LTS

```bash
Kernel Exploit Registry

CVE: CVE-2021-3493 | Name: Ubuntu OverlayFS | Match data: pkg=linux-kernel,ver>=3.13,ver<5.14,x86_64 | Tags: ubuntu=(14.04|16.04|18.04|20.04|20.10) | Rank: 1 | Details: Only Ubuntu is affected.               

CVE: CVE-2021-22555 | Name: Netfilter heap out-of-bounds write | Match data: pkg=linux-kernel,ver>=2.6.19,ver<=5.12-rc6 | Tags: ubuntu=20.04{kernel:5.8.0-*} | Rank: 1 | Details: ip_tables kernel module must be loaded                              
CVE: CVE-2022-32250 | Name: nft_object UAF (NFT_MSG_NEWSET) | Match data: pkg=linux-kernel,ver<5.18.1,CONFIG_USER_NS=y,sysctl:kernel.unprivileged_userns_clone==1 | Tags: ubuntu=(22.04){kernel:5.15.0-27-generic} | Rank: 1 | Details: kernel.unprivileged_userns_clone=1 required (to obtain CAP_NET_ADMIN)
```

## Conf Files
/etc/mysql/mariadb.cnf
/etc/mysql/debian.cnf
/etc/alternatives/my.cnf -> /etc/mysql/mariadb.cnf
/etc/mysql/my.cnf -> /etc/alternatives/my.cnf

/var/www/html/subrion/includes/config.inc.php
/var/www/html/subrion/admin/templates/default/custom-config.tpl
/var/www/html/subrion/js/ckeditor/config.js
/var/www/html/subrion/js/ckeditor/build-config.js
/var/www/html/subrion/js/admin/configuration.js



## Interesting Files
/var/www/html/subrion/includes/api/storage.php
/var/www/html/subrion/includes/config.inc.php
