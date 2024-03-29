# LEMP for scambi.org

## Procedure

Follow [Debian 11 template](001-Debian_11-template.md)

swap tuning
>nano /etc/sysctl.d/88-tuning.conf

    vm.swappiness = 1
    vm.vfs_cache_pressure = 150

>sysctl --system

install useful packages  
>apt install screen git gnupg rsync curl

prerequirements installation  
>apt install nginx python3-certbot-nginx mariadb-server msmtp-mta mutt

msmtp configuration  
>nano /etc/msmtprc

    defaults
    auth on
    tls on  

    account gandi
    host mail.gandi.net
    port 587
    tls_starttls on
    from staff@scambi.org
    user staff@scambi.org
    password abcdef   

    account default : gandi

>systemctl enable --now msmtpd

mutt configuration  
>su - silicon  
>nano .muttrc  

    set sendmail="/usr/bin/msmtp"
    set use_from=yes
    set realname="lemp1see.scambi"
    set from=staff@scambi.org
    set envelope_from=yes
    set copy = no
    set folder = ""

>exit

php installation  
>apt install php-fpm php-xml php-cli php-cgi php-mysql php-mbstring php-gd php-curl php-zip php-json php-common php-intl php-bz2 php-gmp php-bcmath php-opcache php-pear php-imagick  

php configuration  
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

firewall configuration
>firewall-cmd --permanent --zone=public --add-service={http,https}  
>firewall-cmd --reload

nginx configuration  
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
        
        location ~* .(?:webp|jpg|png|svg)$ {
            expires 1y;
            add_header Cache-Control "public";
        }

        error_page 404 403 /404.html;

    }


>rm /etc/nginx/sites-enabled/default  
>ln -s /etc/nginx/sites-available/scambiorg /etc/nginx/sites-enabled/

>mkdir /var/www/scambiorg  
>chown -R silicon:www-data /var/www/scambiorg  

>systemctl restart nginx

>certbot --nginx -d scambi.org,www.scambi.org

modify configuration file for TLS options  
>nano /etc/nginx/sites-available/scambiorg

    add_header Strict-Transport-Security "max-age=31536000";

>systemctl restart nginx

site compilation  
>curl -fsSL https://deb.nodesource.com/setup_lts.x | bash -  
>apt install gcc g++ make nodejs

>usermod -a -G www-data silicon  
>su - silicon  
>git clone https://github.com/scambifestival/scambi.org.git  

>nano build-scambiorg.sh

    #!/bin/bash

    LOG_FILE=/home/silicon/scambiorg-build.log

    echo -e "$(date +%Y%m%d-%H%M%S) - START EXECUTION" >$LOG_FILE

    cd /home/silicon/scambi.org

    updated=$(git fetch --dry-run 2>&1)
    if [[ -z "$updated" ]]
    then
      echo -e "\n$(date +%Y%m%d-%H%M%S) - NOTHING TO DO" >>$LOG_FILE
      echo -e "\n$(date +%Y%m%d-%H%M%S) - END EXECUTION" >>$LOG_FILE
      exit 0
    else
      rm -rf /home/silicon/scambi.org/web
      sleep 1
      echo -e "\n$(date +%Y%m%d-%H%M%S) - GIT PULL" >>$LOG_FILE
      git pull -q >>$LOG_FILE 2>&1
      sleep 1
      echo -e "\n$(date +%Y%m%d-%H%M%S) - NPM CLEAN-INSTALL" >>$LOG_FILE
      npm clean-install >>$LOG_FILE 2>&1
      sleep 1
      echo -e "\n$(date +%Y%m%d-%H%M%S) - NPM RUN BUILD" >>$LOG_FILE
      npm run build >>$LOG_FILE 2>&1
      if [[ $? -eq 0 ]]
      then
        sleep 1
        echo -e "\n$(date +%Y%m%d-%H%M%S) - RSYNC" >>$LOG_FILE
        rsync -a --delete /home/silicon/scambi.org/www/ /var/www/scambiorg/ >>$LOG_FILE 2>&1
        sleep 1
        echo -e "\n$(date +%Y%m%d-%H%M%S) - PERMISSIONS" >>$LOG_FILE
        chown -R silicon:www-data /var/www/scambiorg >>$LOG_FILE 2>&1
        find /var/www/scambiorg -type d -exec chmod 2775 {} \; >>$LOG_FILE 2>&1
        find /var/www/scambiorg -type f -exec chmod 664 {} \; >>$LOG_FILE 2>&1
        echo -e "\n$(date +%Y%m%d-%H%M%S) - END EXECUTION" >>$LOG_FILE
        sleep 1
        mutt -a $LOG_FILE -s "$(date +%Y%m%d-%H%M) - Build sito scambi.org" -- "staff@scambi.org" < "/dev/null"
        exit 0
      else
        echo -e "\n$(date +%Y%m%d-%H%M%S) - BUILD ERROR" >>$LOG_FILE
        echo -e "\n$(date +%Y%m%d-%H%M%S) - END EXECUTION" >>$LOG_FILE
        sleep 1
        mutt -a $LOG_FILE -s "$(date +%Y%m%d-%H%M) - Build sito scambi.org - ERROR" -- "staff@scambi.org" < "/dev/null"
        exit 1
      fi
    fi

>crontab -e  

    */15 * * * * bash /home/silicon/build-scambiorg.sh

**backup locale**

>mkdir -p /var/local/backup/raw/{files,sql}  

borg configuration  
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

**remote backup**

>borg init ssh://lemp1see@bckp1t4v.scambi:822/home/lemp1see/borg -e repokey (see Keepass database)  

>nano /var/local/backup/dr_script.sh

    #!/bin/bash

    # variables to configure
    BKP_STRING="/var/local/backup/raw/"
    export BORG_REPO="ssh://lemp1see@bckp1t4v.scambi:822/home/lemp1see/borg"
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

    30 04 * * * /bin/bash /var/local/backup/dr_script.sh


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

modify configuration file for TLS options  
>nano /etc/nginx/sites-available/visits

    add_header Strict-Transport-Security "max-age=31536000";

>systemctl restart nginx
