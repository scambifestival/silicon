# Debian 11 template

This is the template to be applied to all the hosts, as a common starting framework.

## Procedure

The credentials for root access are all stored in [`secrets.kdbx` KeePass database](secrets.kdbx).

Set Italian keyboard key map
>localectl set-keymap it

Set the time zone and install [chrony](https://chrony.tuxfamily.org) for <abbr title='Network Time Protocol'>NTP</abbr>
>timedatectl set-timezone Europe/Rome  
>apt install chrony

Generate id_ed25519 key
>ssh-keygen -t ed25519

Add swap (suggested)
>dd if=/dev/zero of=/swapfile count=1024 bs=1M status=progress  
>chmod 600 /swapfile  
>mkswap /swapfile  
>swapon /swapfile  
>echo '/swapfile   none    swap    sw    0   0' | tee -a /etc/fstab  
>free -m

Install useful packages  
>apt install dnsutils unzip screen nano nvim man-db

Change SSH configuration  
>nano /etc/ssh/sshd_config

```
Port 822 # (for border servers only)
PermitRootLogin without-password
```

Create service user
>adduser silicon (see Keepass database for the password)

After [tinc installation and configuration](002-Tinc_VPN.md):

- [install and configure dnsmasq](003-dnsmasq.md) so that it listens on localhost only
- add 127.0.0.1 on the top of the file /etc/resolv.conf

## Zabbix agent configuration

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

## Firewall

Apply firewall rules with [firewalld](https://firewalld.org)
>apt install firewalld

make sure that <mark>the public interface is `eth0`</mark> by editing the following line if necessary

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


### Custom SSH port for border servers

#### Firewall configuration

>firewall-cmd --permanent --new-service=aa_ssh  
>firewall-cmd --permanent --service=aa_ssh --set-description='custom ssh'  
>firewall-cmd --permanent --service=aa_ssh --add-port=822/tcp  
>firewall-cmd --permanent --zone=public --add-service=aa_ssh  
>firewall-cmd --permanent --zone=internal --add-service=aa_ssh  
>firewall-cmd --permanent --zone=internal --remove-service=ssh  
>firewall-cmd --reload  

#### fail2ban configuration  

>apt install fail2ban  
>nano /etc/fail2ban/jail.d/42-scambi.local

```
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
```

>systemctl restart fail2ban

## Useful commands

>fail2ban-client status sshd
