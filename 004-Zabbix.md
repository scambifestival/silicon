# Zabbix

Open source software for infrastructural monitoring.  
For simplicity purposes, agents installed on servers are used, configured to be using the virtual LAN exclusively.

In this document, it is explained the procedure used to install the server, plus some configurations for monitoring.

## Procedure

>wget https://repo.zabbix.com/zabbix/6.0/debian/pool/main/z/zabbix-release/zabbix-release_6.0-1%2Bdebian11_all.deb  
>dpkg -i zabbix-release_6.0-1+debian11_all.deb  
>apt update

>apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-agent zabbix-sql-scripts mariadb-server  

mariadb configuration  
>mariadb-secure-installation  (see Keepass for the password)

>mysql -u root

    CREATE DATABASE zabbix DEFAULT CHARACTER SET utf8 COLLATE utf8_bin;
    CREATE USER zabbix@localhost IDENTIFIED BY 'SEE-KEEPASS!';
    GRANT ALL PRIVILEGES ON zabbix.* TO zabbix@localhost;
    FLUSH PRIVILEGES;
    exit

>zcat  /usr/share/doc/zabbix-sql-scripts/mysql/server.sql.gz | mysql -u zabbix -p zabbix  

>nano /etc/zabbix/zabbix_server.conf  

    DBPassword=SEE-KEEPASS!

>nano /etc/zabbix/apache.conf  

    php_value date.timezone Europe/Rome

>systemctl enable --now zabbix-server zabbix-agent apache2

>firewall-cmd --permanent --zone=internal --add-service={http,zabbix-server}  
>firewall-cmd --reload

>nano /etc/dnsmasq.d/dnsmasq.hosts

    # alias
    192.168.64.1    zabbix.scambi  

>systemctl restart dnsmasq apache2

connect to the web interface to complete the installation process  
>http://zabbix.scambi/zabbix

change user Admin password  
>Administration > Users > Admin > Change Password (see Keepass for the password)

### Template configuration

clone the "Linux by Zabbix agent" template naming it "AA_template_Linux_agent"  
disable 3 "component:os" items

enable notifications (after the Telegram bot configuration)  
>Configuration > Actions > Report problems to Zabbix administrators	> Enabled

to insert a new host under monitoring

>Configuration > Hosts > Create host

insert hostname
insert “Linux servers” in groups  
insert the IP in “IP address”  
add `AA_template_Linux_agent` in the document
