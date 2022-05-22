## Zabbix

Software opensource di monitoraggio infrastrutturale.
Per semplicità si usano gli agenti installati sui server, configurati per usare solo la LAN virtuale.

In questo documento è presente la procedura adottata per installare il server e alcune configurazioni del monitoraggio.

<br/> **procedura**

>wget https://repo.zabbix.com/zabbix/6.0/debian/pool/main/z/zabbix-release/zabbix-release_6.0-1%2Bdebian11_all.deb  
>dpkg -i zabbix-release_6.0-1+debian11_all.deb  
>apt update

>apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-agent zabbix-sql-scripts mariadb-server  

configurazione mariadb
>mariadb-secure-installation  (vedi Keepass)

>mysql -u root

    CREATE DATABASE zabbix DEFAULT CHARACTER SET utf8 COLLATE utf8_bin;
    CREATE USER zabbix@localhost IDENTIFIED BY 'VEDI-KEEPASS!';
    GRANT ALL PRIVILEGES ON zabbix.* TO zabbix@localhost;
    FLUSH PRIVILEGES;
    exit

>zcat  /usr/share/doc/zabbix-sql-scripts/mysql/server.sql.gz | mysql -u zabbix -p zabbix  

>nano /etc/zabbix/zabbix_server.conf  

    DBPassword=VEDI-KEEPASS!

>nano /etc/zabbix/apache.conf  

    php_value date.timezone Europe/Rome

>systemctl enable --now zabbix-server zabbix-agent apache2

>firewall-cmd --permanent --zone=internal --add-service={http,zabbix-server}  
>firewall-cmd --reload

>nano /etc/dnsmasq.d/dnsmasq.hosts

    # alias
    192.168.64.1    zabbix.scambi  

>systemctl restart dnsmasq apache2

collegarsi al web per continuare l'installazione
>http://zabbix.scambi/zabbix

cambiare password user Admin  
>Administration > Users > Admin > Change Password (vedi Keepass)

<br/> **configurazione template**  

clonare il template "Linux by Zabbix agent" chiamandolo "AA_template_Linux_agent"  
disattivare i tre Items di tipo "component:os"  

abilitare notifiche (dopo aver fatto procedura per bot Telegram)
>Configuration > Actions > Report problems to Zabbix administrators	> Enabled

per inserire un nuovo host a monitoraggio
>Configuration > Hosts > Create host  
<br/>inserire hostname  
inserire "Linux servers" in groups  
inserire l'IP in "IP address"  
<br/>nella scheda Templates aggiungere "AA_template_Linux_agent"
