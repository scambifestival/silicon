## Nextcloud

<br/> **procedura**

<br/> seguire template Debian 10

installazione pacchetti utili
>apt install screen git gnupg rsync sudo

installazione prerequisiti
>apt install apache2 python3-certbot-apache postgresql postgresql-client

configurazione postgresql
>sudo -u postgres psql  
>>CREATE USER nextcloud WITH CREATEDB PASSWORD 'VEDI-KEEPASS!';  
>>\q

installazione php 7.4
>wget -q https://packages.sury.org/php/apt.gpg -O- | apt-key add -  
>echo "deb https://packages.sury.org/php/ buster main" | tee /etc/apt/sources.list.d/php.list  

>apt update  
>apt install php7.4-fpm php7.4-xml php7.4-cli php7.4-cgi php7.4-pgsql php7.4-mbstring php7.4-gd php7.4-curl php7.4-zip php7.4-json php7.4-common php7.4-intl php7.4-bz2 php7.4-gmp php7.4-bcmath php7.4-apcu php-pear php7.4-imagick libmagickcore-6.q16-6-extra  

configurazione php
>nano /etc/php/7.4/fpm/php.ini

    date.timezone = Europe/Rome

    memory_limit = 1024M

    upload_max_filesize = 513M

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
    SSLProtocol -all +TLSv1.2 +TLSv1.3
    SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:ECDHE-RSA-AES128-GCM-SHA256:AES256+EECDH:DHE-RSA-AES128-GCM-SHA256:AES256+EDH:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES256-GCM-SHA384:AES128-GCM-SHA256:AES256-SHA256:AES128-SHA256:AES256-SHA:AES128-SHA:DES-CBC3-SHA:HIGH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4

>/etc/letsencrypt/options-ssl-apache.conf

    #SSLProtocol             all -SSLv2 -SSLv3
    #SSLCipherSuite          ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS

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

    /usr/bin/rsync -a -R /./etc/apache2/sites-* /var/local/backup/raw/files/ 2>&1

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

    sleep 1

    sudo -u www-data php7.4 /var/www/nextcloud/occ maintenance:mode --off >>/dev/null 2>&1

    exit 0

>mkdir /var/local/backup/backup_log  

>crontab -e

    00 04 * * 1-6 /bin/bash /var/local/backup/backup_script.sh
