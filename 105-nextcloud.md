## Nextcloud

<br/> **procedura**

<br/> seguire template Debian 11

tuning swap
>nano /etc/sysctl.d/88-tuning.conf

    vm.swappiness = 20
    vm.vfs_cache_pressure = 125

>sysctl --system

installazione pacchetti utili
>apt install screen git gnupg rsync sudo

installazione prerequisiti
>apt install apache2 python3-certbot-apache postgresql postgresql-client

configurazione postgresql
>sudo -u postgres psql  
>>CREATE USER nextcloud WITH CREATEDB PASSWORD 'VEDI-KEEPASS!';  
>>\q

installazione php

>apt install php-fpm php-xml php-cli php-cgi php-pgsql php-mbstring php-gd php-curl php-zip php-json php-common php-intl php-bz2 php-gmp php-bcmath php-apcu php-pear php-ldap php-imagick libmagickcore-6.q16-6-extra  

configurazione php
>nano /etc/php/7.4/fpm/php.ini

    date.timezone = Europe/Rome

    memory_limit = 1024M

    upload_max_filesize = 513M

    opcache.enable=1
    opcache.interned_strings_buffer=8
    opcache.max_accelerated_files=10000
    opcache.memory_consumption=128
    opcache.save_comments=1
    opcache.revalidate_freq=1

>nano /etc/php/7.4/fpm/pool.d/www.conf

    pm.max_children = 60
    pm.start_servers = 12
    pm.min_spare_servers = 6
    pm.max_spare_servers = 18
    pm.max_requests = 10000

>nano /etc/php/7.4/mods-available/apcu.ini

    apc.enable_cli=1

>systemctl restart php7.4-fpm

configurare firewall
>firewall-cmd --permanent --zone=public --add-service={http,https}  
>firewall-cmd --reload

configurazione apache

>nano /etc/apache2/conf-available/security.conf

    ServerTokens Prod
    ServerSignature Off

>nano /etc/apache2/sites-available/nuvola.conf

    <VirtualHost *:80>
      ServerName nuvola.scambi.org
      DocumentRoot /var/www/nextcloud
      ErrorLog ${APACHE_LOG_DIR}/nuvola-error.log
      CustomLog ${APACHE_LOG_DIR}/nuvola-access.log combined

      <Directory /var/www/nextcloud/>
        Require all granted
        AllowOverride All
        Options FollowSymLinks MultiViews

        <IfModule mod_dav.c>
          Dav off
        </IfModule>

      </Directory>
    </VirtualHost>

>a2enmod proxy_fcgi setenvif  
>a2enconf php7.4-fpm  
>a2enmod rewrite headers  
>a2dissite 000-default  
>a2ensite nuvola  
>systemctl restart apache2

>mkdir /var/www/nextcloud  

>certbot --apache -d nuvola.scambi.org

>nano /etc/apache2/sites-available/nuvola-le-ssl.conf

    <IfModule mod_headers.c>
      Header always set Strict-Transport-Security "max-age=15552000; includeSubDomains"
    </IfModule>

>systemctl restart apache2

>crontab -e -u www-data

    */5  *  *  *  * php7.4 -f /var/www/nextcloud/cron.php

<br/>**backup locale**

>mkdir -p /var/local/backup/raw/{files,sql}  
>mkdir -p /var/local/backup/raw/files/var/www/nextcloud

>nano /etc/fstab  

    /var/www/nextcloud /var/local/backup/raw/files/var/www/nextcloud none bind 0 0

>mount -a

configurazione borg
>apt install borgbackup  
>mkdir -p /var/local/backup/borg  

>borg init /var/local/backup/borg -e repokey (***REMOVED***)  

>nano /var/local/backup/backup_script.sh

    #!/bin/bash

    sudo -u www-data php7.4 /var/www/nextcloud/occ maintenance:mode --on >>/dev/null 2>&1

    sleep 60

    /usr/bin/rsync -a --delete -R /./etc/apache2/sites-* /var/local/backup/raw/files/ 2>&1

    sleep 1

    sudo -i -u postgres pg_dumpall -c > /var/local/backup/raw/sql/dump.sql

    sleep 1

    # variables to configure
    BKP_STRING="/var/local/backup/raw/"
    export BORG_REPO="/var/local/backup/borg"
    export BORG_PASSPHRASE="***REMOVED***"

    # script start
    LOG_FILE="$(dirname $0)/backup_log/$(date +%Y%m)_$(basename $0 .sh).log"

    echo -e "\n$(date +%Y%m%d-%H%M) - START EXECUTION" >>$LOG_FILE

    echo -e "\n$(date +%Y%m%d-%H%M) - START ARCHIVE CREATION\n" >>$LOG_FILE

    borg create -v --stats --compression lz4 $BORG_REPO::{now:%Y%m%d-%H%M} $BKP_STRING >>$LOG_FILE 2>&1

    if [ "$?" = "1" ] ; then
        echo -e "\n$(date +%Y%m%d-%H%M) - BACKUP ERROR\n" >>$LOG_FILE
        export BORG_REPO=""
        export BORG_PASSPHRASE=""
        exit 1
    fi

    echo -e "\n$(date +%Y%m%d-%H%M) - START PRUNE\n" >>$LOG_FILE
    borg prune -v --list $BORG_REPO --keep-daily=7 --keep-weekly=2 >>$LOG_FILE 2>&1

    if [ "$?" = "1" ] ; then
        echo -e "\n$(date +%Y%m%d-%H%M) - PRUNE ERROR\n" >>$LOG_FILE
        export BORG_REPO=""
        export BORG_PASSPHRASE=""
        exit 1
    fi

    echo -e "\n$(date +%Y%m%d-%H%M) - END EXECUTION\n" >>$LOG_FILE

    export BORG_REPO=""
    export BORG_PASSPHRASE=""

    sleep 1

    sudo -u www-data php7.4 /var/www/nextcloud/occ maintenance:mode --off >>/dev/null 2>&1

    exit 0

>mkdir /var/local/backup/backup_log  

>crontab -e

    00 04 * * * /bin/bash /var/local/backup/backup_script.sh

<br/>**backup remoto**
>borg init ssh://stor1see@bckp1t4v.scambi:822/home/stor1see/borg -e repokey (***REMOVED***)  

>nano /var/local/backup/dr_script.sh

    #!/bin/bash

    # variables to configure
    BKP_STRING="/var/local/backup/raw/"
    export BORG_REPO="ssh://stor1see@bckp1t4v.scambi:822/home/stor1see/borg"
    export BORG_PASSPHRASE="***REMOVED***"

    # script start
    LOG_FILE="$(dirname $0)/backup_log/$(date +%Y%m)_$(basename $0 .sh).log"

    echo -e "\n$(date +%Y%m%d-%H%M) - START EXECUTION" >>$LOG_FILE

    echo -e "\n$(date +%Y%m%d-%H%M) - START ARCHIVE CREATION\n" >>$LOG_FILE

    borg create -v --stats --compression lz4 $BORG_REPO::{now:%Y%m%d-%H%M} $BKP_STRING >>$LOG_FILE 2>&1

    if [ "$?" = "1" ] ; then
        echo -e "\n$(date +%Y%m%d-%H%M) - BACKUP ERROR\n" >>$LOG_FILE
        export BORG_REPO=""
        export BORG_PASSPHRASE=""
        exit 1
    fi

    echo -e "\n$(date +%Y%m%d-%H%M) - START PRUNE\n" >>$LOG_FILE
    borg prune -v --list $BORG_REPO --keep-daily=7 --keep-weekly=4 >>$LOG_FILE 2>&1

    if [ "$?" = "1" ] ; then
        echo -e "\n$(date +%Y%m%d-%H%M) - PRUNE ERROR\n" >>$LOG_FILE
        export BORG_REPO=""
        export BORG_PASSPHRASE=""
        exit 1
    fi

    echo -e "\n$(date +%Y%m%d-%H%M) - END EXECUTION\n" >>$LOG_FILE

    export BORG_REPO=""
    export BORG_PASSPHRASE=""
    exit 0

>crontab -e

    00 05 * * * /bin/bash /var/local/backup/dr_script.sh
