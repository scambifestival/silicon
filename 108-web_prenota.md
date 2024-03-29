# Web server for prenota.scambi.org

## Procedure

Follow [Debian 11 template](001-Debian_11-template.md)

swap tuning  
>nano /etc/sysctl.d/88-tuning.conf

    vm.swappiness = 1
    vm.vfs_cache_pressure = 200

>sysctl --system

install useful packages  
>apt install screen git gnupg rsync

prerequirements installation  
>apt install nginx python3-certbot-nginx postgresql redis-server docker.io apparmor msmtp-mta

msmtp configuration  
>nano /etc/default/msmtpd

    INTERFACE=172.17.0.1
    PORT=25

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

>systemctl edit msmtpd

    [Unit]
    After=docker.service
    Requires=docker.service

    [Service]
    ExecStartPre=/bin/sleep 10

>systemctl daemon-reload  
>systemctl enable --now msmtpd

postgres configuration  
>sudo -u postgres createuser pretix -P  (see keepass for password)  
>sudo -u postgres createdb -O pretix pretix

>nano /etc/postgresql/13/main/postgresql.conf

    listen_addresses = 'localhost,172.17.0.1'

>nano /etc/postgresql/13/main/pg_hba.conf

    host    pretix          pretix          172.17.0.1/16           md5

>systemctl edit postgresql@

    [Unit]
    After=docker.service
    Requires=docker.service

    [Service]
    ExecStartPre=/bin/sleep 10

>systemctl daemon-reload  
>systemctl restart postgresql

redis configuration  
>nano /etc/redis/redis.conf

    unixsocket /var/run/redis/redis.sock
    unixsocketperm 777

>systemctl edit redis

    [Service]
    RuntimeDirectoryPreserve=yes

>systemctl restart redis

pretix configuration  
>mkdir /opt/{pretix,pretix-data}  
>nano /opt/pretix/pretix.cfg

    [pretix]
    instance_name=Scambi Festival
    url=https://prenota.scambi.org
    currency=EUR
    datadir=/data
    trust_x_forwarded_for=on
    trust_x_forwarded_proto=on

    [database]
    backend=postgresql
    name=pretix
    user=pretix
    password=(vedi keepass)
    host=172.17.0.1

    [mail]
    from=staffi@scambi.org
    host=172.17.0.1

    [redis]
    location=unix:///var/run/redis/redis.sock?db=0
    sessions=true

    [celery]
    backend=redis+socket:///var/run/redis/redis.sock?virtual_host=1
    broker=redis+socket:///var/run/redis/redis.sock?virtual_host=2

>chown -R 15371:15371 /opt/pretix*  
>chmod 0700 /opt/pretix/pretix.cfg  

>docker pull pretix/standalone:stable  

>mkdir /root/scambi-pretix  
>nano /root/scambi-pretix/Dockerfile  

    FROM pretix/standalone:stable
    USER root
    RUN pip3 install pretix-fontpack-free
    USER pretixuser
    RUN cd /pretix/src && make production

>cd /root/scambi-pretix  
>docker build . -t scambi-pretix  

>nano /etc/systemd/system/pretix.service

    [Unit]
    Description=pretix
    After=docker.service
    Requires=docker.service

    [Service]
    TimeoutStartSec=0
    ExecStartPre=-/usr/bin/docker kill %n
    ExecStartPre=-/usr/bin/docker rm %n
    ExecStart=/usr/bin/docker run --name %n -p 127.0.0.1:8345:80 \
        -v /opt/pretix-data:/data \
        -v /opt/pretix:/etc/pretix \
        -v /var/run/redis:/var/run/redis \
        --sysctl net.core.somaxconn=4096 \
        scambi-pretix all
    ExecStop=/usr/bin/docker stop %n

    [Install]
    WantedBy=multi-user.target

>systemctl daemon-reload  
>systemctl enable --now pretix

>crontab -e

  */15 * * * * /usr/bin/docker exec pretix.service pretix cron

firewall configuration  
>firewall-cmd --permanent --zone=public --add-service={http,https}  
>firewall-cmd --permanent --zone=docker --add-service={smtp,postgresql}  
>firewall-cmd --reload

nginx configuration  
>nano /etc/nginx/nginx.conf

    server_tokens off;

>nano /etc/nginx/sites-available/prenota

    server {
        listen 80;
        listen [::]:80;
        server_name prenota.scambi.org;

        access_log /var/log/nginx/prenota-access.log;
        error_log /var/log/nginx/prenota-error.log;

        location / {
            proxy_pass http://localhost:8345;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
            proxy_set_header Host $http_host;
        }
    }

>rm /etc/nginx/sites-enabled/default  
>ln -s /etc/nginx/sites-available/prenota /etc/nginx/sites-enabled/

>systemctl restart nginx

>certbot --nginx -d prenota.scambi.org

modify configuration file for TLS options  
>nano /etc/nginx/sites-available/prenota

    add_header Strict-Transport-Security "max-age=31536000";

>systemctl restart nginx

**local backup**

>mkdir -p /var/local/backup/raw/{files,sql}  

borg configuration  
>apt install borgbackup  
>mkdir -p /var/local/backup/raw/{files,sql}

>borg init /var/local/backup/borg -e repokey (see Keepass database)  

>nano /var/local/backup/backup_script.sh

    #!/bin/bash

    /usr/bin/rsync -a --delete -R /./etc/nginx/sites-* /var/local/backup/raw/files/ 2>&1

    sleep 1

    /usr/bin/rsync -a --delete -R /./opt/ /var/local/backup/raw/files/ 2>&1

    sleep 1

    sudo -Hiu postgres pg_dump pretix > /var/local/backup/raw/sql/pretix.sql 2>&1

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

    30 03 * * * /bin/bash /var/local/backup/backup_script.sh

**remote backup**

>borg init ssh://lemp3het@bckp1t4v.scambi:822/home/lemp3het/borg -e repokey (see Keepass database)  

>nano /var/local/backup/dr_script.sh

    #!/bin/bash

    # variables to configure
    BKP_STRING="/var/local/backup/raw/"
    export BORG_REPO="ssh://lemp3het@bckp1t4v.scambi:822/home/lemp3het/borg"
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

    45 03 * * * /bin/bash /var/local/backup/dr_script.sh
