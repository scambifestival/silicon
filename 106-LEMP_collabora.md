## LEMP per collabora

<br/> **procedura**

<br/> seguire template Debian 11

tuning swap
>nano /etc/sysctl.d/88-tuning.conf

    vm.swappiness = 20

>sysctl --system

repository collabora
>cd /usr/share/keyrings  
>wget https://collaboraoffice.com/downloads/gpg/collaboraonline-release-keyring.gpg  

>nano /etc/apt/sources.list.d/collaboraonline.sources  

    Types: deb
    URIs: https://www.collaboraoffice.com/repos/CollaboraOnline/CODE-debian11
    Suites: ./
    Signed-By: /usr/share/keyrings/collaboraonline-release-keyring.gpg

>apt update

configurazione collabora
>apt install coolwsd code-brand  
>coolconfig set ssl.enable false  
>coolconfig set ssl.termination true  
>coolconfig set storage.wopi.host nuvola.scambi.org  
>coolconfig set per_document.max_concurrency 2  
>coolconfig set per_document.idle_timeout_secs 1800  
>coolconfig set-admin-password  (vedi Keepass)  
>systemctl restart coolwsd  

configurazione firewall
>firewall-cmd --permanent --zone=public --add-service={http,https}  
>firewall-cmd --reload

configurazione nginx
>apt install nginx python3-certbot-nginx  
>nano /etc/nginx/nginx.conf

    server_tokens off;

>nano /etc/nginx/sites-available/collabora  

    server {
      listen 80;
      server_name  collabora.scambi.org;

      access_log /var/log/nginx/collabora-access.log;
      error_log /var/log/nginx/collabora-error.log;

      # static files
      location ^~ /browser {
        proxy_pass http://127.0.0.1:9980;
        proxy_set_header Host $http_host;
      }


      # WOPI discovery URL
      location ^~ /hosting/discovery {
        proxy_pass http://127.0.0.1:9980;
        proxy_set_header Host $http_host;
      }


      # Capabilities
      location ^~ /hosting/capabilities {
        proxy_pass http://127.0.0.1:9980;
        proxy_set_header Host $http_host;
      }


      # main websocket
      location ~ ^/cool/(.*)/ws$ {
        proxy_pass http://127.0.0.1:9980;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $http_host;
        proxy_read_timeout 36000s;
      }


      # download, presentation and image upload
      location ~ ^/(c|l)ool {
        proxy_pass http://127.0.0.1:9980;
        proxy_set_header Host $http_host;
      }


      # Admin Console websocket
      location ^~ /cool/adminws {
        proxy_pass http://127.0.0.1:9980;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $http_host;
        proxy_read_timeout 36000s;
      }

    }


>rm /etc/nginx/sites-enabled/default  
>ln -s /etc/nginx/sites-available/collabora /etc/nginx/sites-enabled/  

>systemctl restart nginx  

>certbot --nginx -d collabora.scambi.org  

modificare file configurazione per TLS  
>nano /etc/nginx/sites-available/collabora  

    add_header Strict-Transport-Security "max-age=31536000";

>systemctl restart nginx

installazione font aggiuntivi
>apt install curl ttf-mscorefonts-installer  
>cd /root  
>wget http://ftp.it.debian.org/debian/pool/main/f/fonts-inter/fonts-inter_3.19+ds-2_all.deb  
>dpkg -i fonts-inter_3.19+ds-2_all.deb  
>mkdir -p /usr/share/fonts/googlefonts  
>curl https://fonts.google.com/download?family=Poppins -o Poppins.zip  
>curl https://fonts.google.com/download?family=Londrina%20Solid -o LondrinaSolid.zip  
>unzip Poppins.zip -d /usr/share/fonts/googlefonts/ -x OFL.txt  
>unzip LondrinaSolid.zip -d /usr/share/fonts/googlefonts/ -x OFL.txt  
>chmod -R --reference=/usr/share/fonts/opentype /usr/share/fonts/googlefonts  
>fc-cache -fv  

>coolwsd-systemplate-setup /opt/cool/systemplate /opt/collaboraoffice

test funzionamento  
https://collabora.scambi.org/browser/dist/admin/admin.html
