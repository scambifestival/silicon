## LEMP per scambi.org

<br/> **procedura**

<br/> seguire template Debian 10

installazione pacchetti utili
>apt install screen git gnupg rsync

installazione prerequisiti
>apt install nginx python3-certbot-nginx mariadb-server

configurazione mariadb
>mysql_secure_installation  (vedi Keepass)

>mysql -u root -p

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
    max_connections = 16
    innodb_buffer_pool_instances = 1

    innodb_log_file_size = 8M
    innodb_buffer_pool_size = 64M

>systemctl restart mariadb

installazione php 7.4
>wget -q https://packages.sury.org/php/apt.gpg -O- | apt-key add -  
>echo "deb https://packages.sury.org/php/ buster main" | tee /etc/apt/sources.list.d/php.list  

>apt update  
>apt install php7.4-fpm php7.4-xml php7.4-cli php7.4-cgi php7.4-mysql php7.4-mbstring php7.4-gd php7.4-curl php7.4-zip php7.4-json php7.4-common php7.4-intl php7.4-bz2 php7.4-gmp php7.4-bcmath php7.4-opcache php-pear php-imagick  

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

    pm.max_children = 10
    pm.start_servers = 4
    pm.min_spare_servers = 2
    pm.max_spare_servers = 6

>systemctl restart php7.4-fpm

configurare firewall
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
>nano /etc/letsencrypt/options-ssl-nginx.conf

    #ssl_protocols TLSv1 TLSv1.1 TLSv1.2;

    #ssl_ciphers "ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS";

>nano /etc/nginx/sites-available/scambiorg

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:ECDHE-RSA-AES128-GCM-SHA256:AES256+EECDH:DHE-RSA-AES128-GCM-SHA256:AES256+EDH:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES256-GCM-SHA384:AES128-GCM-SHA256:AES256-SHA256:AES128-SHA256:AES256-SHA:AES128-SHA:DES-CBC3-SHA:HIGH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4";
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

    /usr/bin/mysqldump --add-drop-database --all-databases --user=root > /var/local/backup/raw/sql/dump.sql

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

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:ECDHE-RSA-AES128-GCM-SHA256:AES256+EECDH:DHE-RSA-AES128-GCM-SHA256:AES256+EDH:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES256-GCM-SHA384:AES128-GCM-SHA256:AES256-SHA256:AES128-SHA256:AES256-SHA:AES128-SHA:DES-CBC3-SHA:HIGH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4";
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

>cd /root  
>wget http://ftp.it.debian.org/debian/pool/main/g/goaccess/goaccess_1.4-1_amd64.deb  
>dpkg -i goaccess_1.4-1_amd64.deb  
>apt -f install  

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

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:ECDHE-RSA-AES128-GCM-SHA256:AES256+EECDH:DHE-RSA-AES128-GCM-SHA256:AES256+EDH:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES256-GCM-SHA384:AES128-GCM-SHA256:AES256-SHA256:AES128-SHA256:AES256-SHA:AES128-SHA:DES-CBC3-SHA:HIGH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4";
    add_header Strict-Transport-Security "max-age=31536000";

>systemctl restart nginx
