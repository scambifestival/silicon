# Baserow database  

## Procedure

Follow [Debian 11 template](001-Debian_11-template.md)

swap tuning  
>nano /etc/sysctl.d/88-tuning.conf

    vm.swappiness = 1
    vm.vfs_cache_pressure = 200

>sysctl --system

prerequirements installation  
>apt install docker.io apparmor

firewall configuration  
>firewall-cmd --permanent --zone=public --add-service={http,https}  
>firewall-cmd --reload

user silicon configuration  
>usermod -aG docker silicon

container configuration  
>mkdir -p /opt/baserow/data  
>nano /opt/baserow/custom_baserow_conf.sh

    export BASEROW_PUBLIC_URL=https://pino.scambi.org
    export BASEROW_CADDY_ADDRESSES=https://pino.scambi.org
    export EMAIL_SMTP=True
    export EMAIL_SMTP_HOST=mail.gandi.net
    export EMAIL_SMTP_PORT=587
    export EMAIL_SMTP_USER=staff@scambi.org
    export EMAIL_SMTP_PASSWORD=
    export EMAIL_SMTP_USE_TLS=True
    export FROM_EMAIL=staff@scambi.org

>chmod +x /opt/baserow/custom_baserow_conf.sh  

>docker run -d --name baserow-1_11_0 -v /opt/baserow/data:/baserow/data -v /opt/baserow/custom_baserow_conf.sh:/baserow/supervisor/env/custom_baserow_conf.sh -p 80:80 -p 443:443 --restart unless-stopped baserow/baserow:1.11.0

**local backup**

>mkdir -p /var/local/backup/raw  

borg configuration  
>apt install borgbackup  
>mkdir -p /var/local/backup/borg  

>borg init /var/local/backup/borg -e repokey (see Keepass database)  

>nano /var/local/backup/backup_script.sh

    #!/bin/bash

    export BASEROW_NAME=baserow-1_11_0

    /usr/bin/docker stop $BASEROW_NAME >/dev/null

    sleep 5

    /usr/bin/rsync -a --delete /opt/baserow/ /var/local/backup/raw/ 2>&1

    sleep 5

    /usr/bin/docker start $BASEROW_NAME >/dev/null

    sleep 5

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

    00 08 * * * /bin/bash /var/local/backup/backup_script.sh

**remote backup**

>borg init ssh://data1see@bckp1t4v.scambi:822/home/data1see/borg -e repokey (see Keepass database)  

>nano /var/local/backup/dr_script.sh

    #!/bin/bash

    # variables to configure
    BKP_STRING="/var/local/backup/raw/"
    export BORG_REPO="ssh://data1see@bckp1t4v.scambi:822/home/data1see/borg"
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

    30 08 * * * /bin/bash /var/local/backup/dr_script.sh
