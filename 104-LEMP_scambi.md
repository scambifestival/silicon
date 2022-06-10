## LEMP per scambi.org

<br/> **procedura**

<br/> seguire template Debian 11

tuning swap
>nano /etc/sysctl.d/88-tuning.conf

    vm.swappiness = 1
    vm.vfs_cache_pressure = 150

>sysctl --system

installazione pacchetti utili
>apt install screen git gnupg rsync

installazione prerequisiti
>apt install nginx python3-certbot-nginx mariadb-server

configurazione mariadb
>mariadb-secure-installation  (vedi Keepass)

>mysql -u root

    CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
    CREATE USER wpuser@localhost IDENTIFIED BY 'VEDI-KEEPASS!';
    GRANT ALL PRIVILEGES ON wordpress.* TO wpuser@localhost;
    FLUSH PRIVILEGES;
    exit

tuning mariadb
>nano /etc/mysql/conf.d/zz_performance.cnf  

    [mysqld]
    performance_schema = on

    [server]
    max_connections = 24
    innodb_buffer_pool_instances = 1

    innodb_log_file_size = 32M
    innodb_buffer_pool_size = 128M

>systemctl restart mariadb

installazione php
>apt install php-fpm php-xml php-cli php-cgi php-mysql php-mbstring php-gd php-curl php-zip php-json php-common php-intl php-bz2 php-gmp php-bcmath php-opcache php-pear php-imagick  

configurazione php
>nano /etc/php/7.4/fpm/php.ini

    date.timezone = Europe/Rome

    post_max_size = 10M

    upload_max_filesize = 8M

    opcache.enable=1
    opcache.fast_shutdown=1
    opcache.interned_strings_buffer=8
    opcache.max_accelerated_files=4000
    opcache.memory_consumption=64
    opcache.revalidate_freq=60
    opcache.validate_timestamps=1

>nano /etc/php/7.4/fpm/pool.d/www.conf

    pm.max_children = 20
    pm.start_servers = 8
    pm.min_spare_servers = 4
    pm.max_spare_servers = 8
    pm.max_requests = 10000

>systemctl restart php7.4-fpm

configurazione firewall
>firewall-cmd --permanent --zone=public --add-service={http,https}  
>firewall-cmd --reload

configurazione nginx
>nano /etc/nginx/nginx.conf

    server_tokens off;

>nano /etc/nginx/sites-available/scambiorg

    server {
        listen 80;
        listen [::]:80;
        server_name www.scambi.org;
        rewrite ^ http://scambi.org$request_uri? permanent;
    }

    server {
        listen 80;
        listen [::]:80;
        server_name scambi.org;
        root /var/www/scambiorg;

        client_max_body_size 8M;

        access_log /var/log/nginx/scambiorg-access.log;
        error_log /var/log/nginx/scambiorg-error.log;

        location / {
            try_files $uri $uri/ /index.php?args;
            index index.php index.html;
        }

        location ~ \.php$ {
            try_files $uri =404;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include fastcgi_params;
            fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        }

    }


>rm /etc/nginx/sites-enabled/default  
>ln -s /etc/nginx/sites-available/scambiorg /etc/nginx/sites-enabled/

>mkdir /var/www/scambiorg  
>chown -R silicon:www-data /var/www/scambiorg  
>find /var/www/scambiorg -type d -exec chmod 2775 {} \\;  <br>
>find /var/www/scambiorg -type f -exec chmod 664 {} \\;   

>systemctl restart nginx

>certbot --nginx -d scambi.org,www.scambi.org

modificare file configurazione per TLS  
>nano /etc/nginx/sites-available/scambiorg

    add_header Strict-Transport-Security "max-age=31536000";

>systemctl restart nginx

<br/>**backup locale**

>mkdir -p /var/local/backup/raw/{files,sql}  

configurazione borg
>apt install borgbackup  
>mkdir -p /var/local/backup/borg  

>borg init /var/local/backup/borg -e repokey (***REMOVED***)  

>nano /var/local/backup/backup_script.sh

    #!/bin/bash

    /usr/bin/rsync -a --delete -R /./etc/nginx/sites-* /var/local/backup/raw/files/ 2>&1

    sleep 1

    /usr/bin/rsync -a --delete -R /./var/www/ /var/local/backup/raw/files/ 2>&1

    sleep 1

    /usr/bin/mysqldump --user=root wordpress > /var/local/backup/raw/sql/wordpress.sql
    /usr/bin/mysqldump --user=root prenota > /var/local/backup/raw/sql/prenota.sql

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
    exit 0

>mkdir /var/local/backup/backup_log  

>crontab -e

    00 04 * * * /bin/bash /var/local/backup/backup_script.sh


<br/>**backup remoto**
>borg init ssh://lemp1see@bckp1t4v.scambi:822/home/lemp1see/borg -e repokey (***REMOVED***)  

>nano /var/local/backup/dr_script.sh

    #!/bin/bash

    # variables to configure
    BKP_STRING="/var/local/backup/raw/"
    export BORG_REPO="ssh://lemp1see@bckp1t4v.scambi:822/home/lemp1see/borg"
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

    30 04 * * * /bin/bash /var/local/backup/dr_script.sh


### prenota.scambi.org

>mysql -u root -p

    CREATE DATABASE prenota DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
    CREATE USER prenota@localhost IDENTIFIED BY 'VEDI-KEEPASS!';
    GRANT ALL PRIVILEGES ON prenota.* TO prenota@localhost;
    FLUSH PRIVILEGES;
    exit


>nano /etc/nginx/sites-available/prenota

    server {
        listen 80;
        listen [::]:80;
        server_name prenota.scambi.org;
        root /var/www/prenota;

        client_max_body_size 8M;

        access_log /var/log/nginx/prenota-access.log;
        error_log /var/log/nginx/prenota-error.log;

        location / {
            try_files $uri $uri/ /index.php?args;
            index index.php index.html;
        }

        location ~ \.php$ {
            try_files $uri =404;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include fastcgi_params;
            fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        }

    }


>ln -s /etc/nginx/sites-available/prenota /etc/nginx/sites-enabled/

>mkdir /var/www/prenota  
>chown -R silicon:www-data /var/www/prenota  
>chmod 2755 /var/www/prenota  

>systemctl restart nginx

>certbot --nginx -d prenota.scambi.org

modificare file configurazione per TLS  
>nano /etc/nginx/sites-available/prenota

    add_header Strict-Transport-Security "max-age=31536000";

>systemctl restart nginx

modifiche wordpress
>find /var/www/prenota -type d -exec chmod 2775 {} \\;  <br>
>find /var/www/prenota -type f -exec chmod 664 {} \\;

>nano /var/www/prenota/wp-config.php  

    define('FS_METHOD', 'direct');
    define('FS_CHMOD_DIR', 0755);
    define('FS_CHMOD_FILE', 0644);

### visits.scambi.org

>mkdir /var/www/visits  
>chown -R silicon:www-data /var/www/visits  

>apt install goaccess  

>nano /root/visits-script.sh

    #!/bin/bash
    /usr/bin/zcat -f /var/log/nginx/scambiorg-access.log* | /usr/bin/goaccess - --log-format=COMBINED --anonymize-ip -o /var/www/visits/index.html

>crontab -e

        12,42 * * * * /bin/bash /root/visits-script.sh

>nano /etc/nginx/sites-available/visits

    server {
        listen 80;
        listen [::]:80;
        server_name visits.scambi.org;
        root /var/www/visits;

        access_log /var/log/nginx/visits-access.log;
        error_log /var/log/nginx/visits-error.log;

        location / {
            try_files $uri $uri/ /index.php?args;
            index index.php index.html;
        }

        location ~ \.php$ {
            try_files $uri =404;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include fastcgi_params;
            fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        }

    }

>ln -s /etc/nginx/sites-available/visits /etc/nginx/sites-enabled/

>systemctl restart nginx

>certbot --nginx -d visits.scambi.org

modificare file configurazione per TLS  
>nano /etc/nginx/sites-available/visits

    add_header Strict-Transport-Security "max-age=31536000";

>systemctl restart nginx
