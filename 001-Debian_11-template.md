## Debian 11 template

This is the template to be applied to all the hosts, as a common starting framework.  

**procedure**

root password : see Keepass database  

set italian keyboard  
>localectl set-keymap it

set timezone and install chrony for NTP  
>timedatectl set-timezone Europe/Rome  
>apt install chrony

generate id_ed25519 key
>ssh-keygen -t ed25519

add swap (suggested)
>dd if=/dev/zero of=/swapfile count=1024 bs=1M status=progress  
>chmod 600 /swapfile  
>mkswap /swapfile  
>swapon /swapfile  
>echo '/swapfile   none    swap    sw    0   0' | tee -a /etc/fstab  
>free -m

install useful packages  
>apt install dnsutils unzip screen nano man-db

change ssh configuration  
>nano /etc/ssh/sshd_config

    Port 822 (for border servers only)
    PermitRootLogin without-password

create service user
>adduser silicon (see Keepass database for the password)


after [tinc](002-Tinc_VPN.md) installation and configuration  
- install and configure [dnsmasq](003-dnsmasq.md) so that it listens on localhost only  
- add 127.0.0.1 on the top of the file /etc/resolv.conf  

zabbix agent configuration  
>wget https://repo.zabbix.com/zabbix/6.0/debian/pool/main/z/zabbix-release/zabbix-release_6.0-1%2Bdebian11_all.deb  
>dpkg -i zabbix-release_6.0-1+debian11_all.deb  
>apt update  
>apt install zabbix-agent  

>nano /etc/zabbix/zabbix_agentd.conf

    server = zabbix.scambi  
    server active = zabbix.scambi  
    hostname =

>systemctl enable --now zabbix-agent  
>systemctl restart zabbix-agent  

apply firewall rules with firewalld  
>apt install firewalld

check that the public interface is *eth0*, and edit the following line if it is otherwise  

>firewall-cmd --state  
>firewall-cmd --permanent --zone=public --add-interface=eth0  
>firewall-cmd --permanent --zone=internal --add-interface=scambi  

>firewall-cmd --permanent --new-service=aa_tinc  
>firewall-cmd --permanent --service=aa_tinc --set-description='custom tinc servers'  
>firewall-cmd --permanent --service=aa_tinc --add-port=777/tcp  
>firewall-cmd --permanent --service=aa_tinc --add-port=777/udp  

>firewall-cmd --zone=public --permanent --remove-service=ssh  
>firewall-cmd --zone=internal --permanent --remove-service={mdns,samba-client}  

>firewall-cmd --permanent --zone=internal --add-service={ssh,zabbix-agent}  
>firewall-cmd --permanent --zone=public --add-service=aa_tinc  

>firewall-cmd --permanent --zone=public --set-target=DROP  
>firewall-cmd --permanent --zone=public --add-icmp-block-inversion  
>firewall-cmd --permanent --zone=public --add-icmp-block={echo-reply,echo-request,port-unreachable,time-exceeded,timestamp-request,timestamp-reply}  

>firewall-cmd --reload  


##### for border servers (custom ssh port)  
firewall configuration  
>firewall-cmd --permanent --new-service=aa_ssh  
>firewall-cmd --permanent --service=aa_ssh --set-description='custom ssh'  
>firewall-cmd --permanent --service=aa_ssh --add-port=822/tcp  
>firewall-cmd --permanent --zone=public --add-service=aa_ssh  
>firewall-cmd --permanent --zone=internal --add-service=aa_ssh  
>firewall-cmd --permanent --zone=internal --remove-service=ssh  
>firewall-cmd --reload  

fail2ban configuration  
>apt install fail2ban  
>nano /etc/fail2ban/jail.d/42-scambi.local  

    [INCLUDES]
    before = paths-debian.conf

    [DEFAULT]
    bantime = 300
    findtime = 600
    maxretry = 5
    bantime.increment = true

    [sshd]
    enabled = true
    port = 822
    logpath = %(sshd_log)s
    backend = %(sshd_backend)s
    banaction = nftables[blocktype=drop]

>systemctl restart fail2ban

**useful commands**  
>fail2ban-client status sshd
