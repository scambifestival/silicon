## LAMP per Nextcloud

<br/> **procedura**

<br/> seguire template Debian 10

installazione pacchetti utili
>apt install screen git gnupg rsync

installazione prerequisiti
>apt install apache2 python3-certbot-apache mariadb-server

configurazione mariadb
>mysql_secure_installation  (vedi Keepass)

>mysql -u root -p

    CREATE DATABASE nuvola DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
    CREATE USER nuvola@localhost IDENTIFIED BY 'VEDI-KEEPASS!';
    GRANT ALL PRIVILEGES ON nuvola.* TO nuvola@localhost;
    FLUSH PRIVILEGES;
    exit

installazione php 7.4
>wget -q https://packages.sury.org/php/apt.gpg -O- | apt-key add -  
>echo "deb https://packages.sury.org/php/ buster main" | tee /etc/apt/sources.list.d/php.list  

>apt update  
>apt install php7.4-fpm php7.4-xml php7.4-cli php7.4-cgi php7.4-mysql php7.4-mbstring php7.4-gd php7.4-curl php7.4-zip php7.4-json php7.4-common php7.4-intl php7.4-bz2 php7.4-gmp php7.4-bcmath php-pear php-imagick  

configurazione php
>nano /etc/php/7.4/fpm/php.ini

    date.timezone = Europe/Rome

    memory_limit = 512M

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
>a2enmod rewrite  
>a2dissite 000-default  
>a2ensite nuvola
>systemctl restart apache2

>certbot --apache -d nuvola.scambi.org

>nano /etc/apache2/sites-available/nuvola-le-ssl.conf

    <IfModule mod_headers.c>
      Header always set Strict-Transport-Security "max-age=15552000; includeSubDomains"
    </IfModule>

    [ commentare virtualhost : 80 ]

systemctl restart apache2

<br/> **installazione nextcloud**

>cd /var/www/  
>wget https://download.nextcloud.com/server/releases/nextcloud-X.Y.Z.zip  
>unzip nextcloud-X.Y.Z.zip  
>chown -R www-data:www-data /var/www/nextcloud  

seguire wizard web
