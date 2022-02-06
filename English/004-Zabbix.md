## Zabbix

Open source software for infrastructural monitoring.  
For simplicity purposes, agents installed on servers are used, configured to be using the virtual LAN exclusively.

In this document, it is explained the procedure used to install the server, plus some configurations for monitoring.

<br>

### Procedura

>wget https://repo.zabbix.com/zabbix/5.0/debian/pool/main/z/zabbix-release/zabbix-release_5.0-1+buster_all.deb  
>dpkg -i zabbix-release_5.0-1+buster_all.deb  
>apt update

>apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-agent mariadb-server  

mariadb configuration
>mysql_secure_installation  (vedi Keepass)

>mysql -u root -p

    CREATE DATABASE zabbix DEFAULT CHARACTER SET utf8 COLLATE utf8_bin;
    CREATE USER zabbix@localhost IDENTIFIED BY 'VEDI-KEEPASS!';
    GRANT ALL PRIVILEGES ON zabbix.* TO zabbix@localhost;
    FLUSH PRIVILEGES;
    exit

>zcat /usr/share/doc/zabbix-server-mysql*/create.sql.gz | mysql -uzabbix -p zabbix  

>nano /etc/zabbix/zabbix_server.conf  

    DBPassword=VEDI-KEEPASS!

>nano /etc/zabbix/apache.conf  

    php_value date.timezone Europe/Rome

>systemctl enable --now zabbix-server zabbix-agent apache2

>firewall-cmd --permanent --zone=internal --add-service=http  
>firewall-cmd --reload

>nano /etc/dnsmasq.d/dnsmasq.hosts

    # alias
    192.168.64.1    zabbix.scambi  

>systemctl restart dnsmasq apache2

connect to the web interface to complete the installation process
>http://zabbix.scambi/zabbix

change user Admin password  
>Administration > Users > Admin > Change Password (vedi Keepass)

<br>

### Template configuration

Clone “Template OS Linux by Zabbix agent” and renamed the cloned version “AA_template_base” (“AA_base_template” in Italian)  

after completing the procedure to enable the Telegram bot, enable notifications
>Configuration > Actions > Report problems to Zabbix administrators	> Enabled

to add a new host to be monitored:
>Configuration > Hosts > Create host  
<br/>add hostname  
add "Linux servers" in groups  
add IP in "IP address"  
<br/>in the “Templates” tab, add "AA_template_base"
