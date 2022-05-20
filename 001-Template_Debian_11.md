## Template Debian 11

Template da applicare a tutti gli host, come base comune di partenza.

<br/> **procedura**

password di root : vedi Keepass

impostare tastiera italiana
>localectl set-keymap it

impostare fuso orario e installare chrony per NTP
>timedatectl set-timezone Europe/Rome  
>apt install chrony

generare chiave id_rsa
>ssh-keygen -t rsa -b 4096

aggiungere swap (consigliato)
>dd if=/dev/zero of=/swapfile count=1024 bs=1M status=progress  
>chmod 600 /swapfile  
>mkswap /swapfile  
>swapon /swapfile  
>echo '/swapfile   none    swap    sw    0   0' | tee -a /etc/fstab  
>free -m

installare pacchetti utili
>apt install dnsutils unzip screen nano man-db

modifica configurazione ssh
>nano /etc/ssh/sshd_config

    Port 822 (solo per macchine di frontiera)
    PermitRootLogin without-password

creare utente di servizio
>adduser silicon (vedi Keepass)

dopo aver installato e configurato [tinc](002-Tinc_VPN.md)
- installare e configurare [dnsmasq](003-dnsmasq.md) che ascolti solo in localhost <br/>
- aggiungere 127.0.0.1 in testa al file /etc/resolv.conf

configurazione agente zabbix
>wget https://repo.zabbix.com/zabbix/6.0/debian/pool/main/z/zabbix-release/zabbix-release_6.0-1%2Bdebian11_all.deb  
>dpkg -i zabbix-release_6.0-1+debian11_all.deb  
>apt update  
>apt install zabbix-agent  

>nano /etc/zabbix/zabbix_agentd.conf

    server = zabbix.scambi  
    server active = zabbix.scambi  
    hostname =

>systemctl enable --now zabbix-agent

applicare regole firewall con firewalld
>apt install firewalld

verificare che l'interfaccia pubblica sia *eth0*, altrimenti modificare la riga qui sotto

>firewall-cmd --state  
>firewall-cmd --permanent --zone=public --add-interface=eth0  
>firewall-cmd --permanent --zone=internal --add-interface=tun0  

>firewall-cmd --permanent --new-service=aa_tinc  
>firewall-cmd --permanent --service=aa_tinc --set-description='custom tinc servers'  
>firewall-cmd --permanent --service=aa_tinc --add-port=777/tcp  
>firewall-cmd --permanent --service=aa_tinc --add-port=777/udp  

>firewall-cmd --zone=public --permanent --remove-service=ssh  
>firewall-cmd --zone=internal --permanent --remove-service={mdns,samba-client}  

>firewall-cmd --permanent --zone=internal --add-service={ssh,zabbix-agent}  
>firewall-cmd --permanent --zone=public --add-service=aa_tinc  

>firewall-cmd --permanent --zone=public --set-target=DROP  
>firewall-cmd --permanent --direct --add-rule ipv4 filter INPUT 0 -p icmp -s 0.0.0.0/0 -d 0.0.0.0/0 -j ACCEPT  

>firewall-cmd --reload  


##### per macchine di frontiera (porta ssh modificata) <br/>
<br/> configurazione firewall
>firewall-cmd --permanent --new-service=aa_ssh  
>firewall-cmd --permanent --service=aa_ssh --set-description='custom ssh'  
>firewall-cmd --permanent --service=aa_ssh --add-port=822/tcp  
>firewall-cmd --permanent --zone=public --add-service=aa_ssh  
>firewall-cmd --permanent --zone=internal --add-service=aa_ssh  
>firewall-cmd --permanent --zone=internal --remove-service=ssh  
>firewall-cmd --reload  

configurazione fail2ban
>apt install fail2ban  
>nano /etc/fail2ban/jail.d/42-scambi.local  

    [INCLUDES]
    before = paths-debian.conf

    [DEFAULT]
    bantime = 300
    findtime = 600
    maxretry = 5

    [sshd]
    enabled = true
    port = 822
    logpath = %(sshd_log)s
    backend = %(sshd_backend)s

>nano /etc/fail2ban/action.d/nftables-common.local

    [Init]
    blocktype = DROP

>systemctl restart fail2ban

<br/> **comandi utili**
>fail2ban-client status sshd
