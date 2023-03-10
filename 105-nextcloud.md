# Nextcloud

## Procedure

Follow [Debian 11 template](001-Debian_11-template.md)

swap tuning  
>nano /etc/sysctl.d/88-tuning.conf

    vm.swappiness = 1
    vm.vfs_cache_pressure = 150

>sysctl --system

install useful packages  
>apt install screen git gnupg rsync sudo

prerequirements installation  
>apt install apache2 python3-certbot-apache postgresql postgresql-client

postgresql configuration  
>sudo -u postgres psql  
>>CREATE USER nextcloud WITH CREATEDB PASSWORD 'SEE-KEEPASS!';  
>>\q

php installation  
>apt install php-fpm php-xml php-cli php-cgi php-pgsql php-mbstring php-gd php-curl php-zip php-json php-common php-intl php-bz2 php-gmp php-bcmath php-apcu php-pear php-ldap php-imagick libmagickcore-6.q16-6-extra  

php configuration  
>nano /etc/php/7.4/fpm/php.ini

    date.timezone = Europe/Rome

    memory_limit = 1024M

    post_max_size = 16G

    upload_max_filesize = 16GB

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

firewall configuration  
>firewall-cmd --permanent --zone=public --add-service={http,https}  
>firewall-cmd --reload

apache configuration  
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

**local backup**

>mkdir -p /var/local/backup/raw/{files,sql}  
>mkdir -p /var/local/backup/raw/files/var/www/nextcloud

>nano /etc/fstab  

    /var/www/nextcloud /var/local/backup/raw/files/var/www/nextcloud none bind 0 0

>mount -a

borg configuration  
>apt install borgbackup  
>mkdir -p /var/local/backup/borg  

>borg init /var/local/backup/borg -e repokey (see Keepass database)  

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
    export BORG_PASSPHRASE="see Keepass database"

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

**remote backup**

>borg init ssh://stor1see@bckp1t4v.scambi:822/home/stor1see/borg -e repokey (see Keepass database)  

>nano /var/local/backup/dr_script.sh

    #!/bin/bash

    # variables to configure
    BKP_STRING="/var/local/backup/raw/"
    export BORG_REPO="ssh://stor1see@bckp1t4v.scambi:822/home/stor1see/borg"
    export BORG_PASSPHRASE="see Keepass database"

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


### x.scambi.org

>nano /etc/apache2/sites-available/x.conf  

    <VirtualHost *:80>
      ServerName x.scambi.org
      DocumentRoot /var/www/x
      ErrorLog ${APACHE_LOG_DIR}/x-error.log
      CustomLog ${APACHE_LOG_DIR}/x-access.log combined

      <Directory /var/www/x/>
        Require all granted
      </Directory>
    </VirtualHost>

>a2ensite x  
>systemctl restart apache2  

>mkdir /var/www/x  
>chown www-data: /var/www/x

>certbot --apache -d x.scambi.org  

>nano /etc/apache2/sites-available/x-le-ssl.conf  

    <IfModule mod_headers.c>
      Header always set Strict-Transport-Security "max-age=15552000; includeSubDomains"
      Header set Access-Control-Allow-Origin "*"
      Header set Cache-Control "max-age=300, private"
    </IfModule>

>systemctl restart apache2

in Nextcloud you should enable the app "External storage support" and configure the folder "x.scambi.org"  

**local backup**

>mkdir -p /var/local/backup/raw/files/var/www/x  

>nano /etc/fstab  

    /var/www/x /var/local/backup/raw/files/var/www/x none bind 0 0

>mount -a
