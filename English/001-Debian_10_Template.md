## Debian 10 Template

This is the template to be applied to all the hosts, as a common starting framework.

### Procedure

root password: see `secrets.kbdx`, KeePass database located at the root of this repository.

Set the Italian keymapping
```
localectl set-keymap it
```

set the time zone and install chrony for NTP
```
timedatectl set-timezone Europe/Rome  
apt install chrony
```

generate `id_rsa` key
```
ssh-keygen -t rsa -b 4096
```

add swap (suggested)
```
dd if=/dev/zero of=/swapfile count=1024 bs=1M status=progress  
chmod 600 /swapfile  
mkswap /swapfile  
swapon /swapfile  
echo '/swapfile   none    swap    sw    0   0' | tee -a /etc/fstab  
free -m
```

install useful packages
```
apt install dnsutils unzip screen
```

edit ssh configuration: `nano /etc/ssh/sshd_config`:

```
Port 822 # only for machines at the border
PermitRootLogin without-password
```

create service user
```
adduser silicon # see secrets.kbdx
```

after [tinc](002-Tinc_VPN.md) installation and configuration…
- …install e configure [dnsmasq](003-dnsmasq.md), so that it listens in localhost only
- prepend `127.0.0.1` to  `/etc/resolv.conf`

configuration of zabbix agent

```
wget https://repo.zabbix.com/zabbix/5.0/debian/pool/main/z/zabbix-release/zabbix-release_5.0-1+buster_all.deb  
dpkg -i zabbix-release_5.0-1+buster_all.deb  
apt update  
apt install zabbix-agent
```

then, in `/etc/zabbix/zabbix_agentd.conf`, set:

    server = zabbix.scambi  
    server active = zabbix.scambi  
    hostname =

…and run `systemctl enable --now zabbix-agent`

set firewall rules with firewalld
```
update-alternatives --config ip6tables # set “legacy”
update-alternatives --config iptables # set “legacy”
```

…then run `apt install firewalld`

check that the public interface is *`eth0`*, and edit the following line if it is otherwise

```
firewall-cmd --state
firewall-cmd --permanent --zone=public --add-interface=eth0
firewall-cmd --permanent --zone=internal --add-interface=scambi
```

```
firewall-cmd --permanent --new-service=aa_tinc
firewall-cmd --permanent --service=aa_tinc --set-description='custom tinc servers'
firewall-cmd --permanent --service=aa_tinc --add-port=6996/tcp
firewall-cmd --permanent --service=aa_tinc --add-port=6996/udp
```

```
firewall-cmd --zone=public --permanent --remove-service=ssh
firewall-cmd --zone=internal --permanent --remove-service={mdns,samba-client}
```

```
firewall-cmd --permanent --zone=internal --add-service={ssh,zabbix-agent}
firewall-cmd --permanent --zone=public --add-service=aa_tinc
```

```
firewall-cmd --permanent --zone=public --set-target=DROP
firewall-cmd --permanent --direct --add-rule ipv4 filter INPUT 0 -p icmp -s 0.0.0.0/0 -d 0.0.0.0/0 -j ACCEPT
```

finally, run `firewall-cmd --reload`.

##### For border machines (custom SSH port)

Firewall configuration

```
firewall-cmd --permanent --new-service=aa_ssh
firewall-cmd --permanent --service=aa_ssh --set-description='custom ssh'
firewall-cmd --permanent --service=aa_ssh --add-port=822/tcp
firewall-cmd --permanent --zone=public --add-service=aa_ssh
firewall-cmd --permanent --zone=internal --add-service=aa_ssh
firewall-cmd --permanent --zone=internal --remove-service=ssh
firewall-cmd --reload
```

**fail2ban configuration**

install it `apt install fail2ban`, then, in `/etc/fail2ban/jail.d/42-scambi.local`:

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

in `/etc/fail2ban/action.d/iptables-common.local`

    [Init]
    blocktype = DROP

`systemctl restart fail2ban`

##### Useful commands

`fail2ban-client status sshd`
