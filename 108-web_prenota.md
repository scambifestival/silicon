## LEMP per prenota.scambi.org

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
>apt install nginx python3-certbot-nginx postgresql redis-server docker.io apparmor msmtp-mta

configurazione msmtp
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

configurazione postgres
>sudo -u postgres createuser pretix -P  (vedi keepass)  
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

configurazione redis
>nano /etc/redis/redis.conf

    unixsocket /var/run/redis/redis.sock
    unixsocketperm 777

>systemctl edit redis

    [Service]
    RuntimeDirectoryPreserve=yes

>systemctl restart redis

configurazione pretix
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
        pretix/standalone:stable all
    ExecStop=/usr/bin/docker stop %n

    [Install]
    WantedBy=multi-user.target

>systemctl daemon-reload  
>systemctl enable --now pretix

>crontab -e

  */15 * * * * /usr/bin/docker exec pretix.service pretix cron

configurazione firewall
>firewall-cmd --permanent --zone=public --add-service={http,https}  
>firewall-cmd --permanent --zone=docker --add-service={smtp,postgresql}  
>firewall-cmd --reload

configurazione nginx
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

modificare file configurazione per TLS  
>nano /etc/nginx/sites-available/prenota

    add_header Strict-Transport-Security "max-age=31536000";

>systemctl restart nginx

<br/>**backup locale**
