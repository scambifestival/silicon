# Backup

[Time4VPS](https://time4vps.com) does not support Debian 11 on storage VPS, we installed Debian 9

## Procedure

Follow [Debian 11 template](001-Debian_11-template.md)

### Fix DNS and hostname

>nano /root/resolv.conf

```
nameserver 127.0.0.1
nameserver 80.209.227.143
nameserver 62.77.155.143
```

>crontab -e

```
@reboot cp /root/resolv.conf /etc/resolv.conf
@reboot hostnamectl set-hostname bckp1t4v
```

### Fix locale

>apt install locales  
>dpkg-reconfigure locales

select `it_IT.UTF-8`


### Install useful packages  

>nano /etc/apt/sources.list

```
deb http://deb.debian.org/debian stretch-backports main contrib non-free
```

>apt update  
>apt install rsync iftop screen  
>apt install -t stretch-backports borgbackup  

## User configuration

Each server has its own user configuration.

### Example

```
adduser --disabled-password --gecos "" --uid 2004 lemp1see
mkdir -p /home/lemp1see/{backup,borg,log}
echo "borg repo directly from host" > /home/lemp1see/backup/README
mkdir /home/lemp1see/.ssh
nano /home/lemp1see/.ssh/authorized_keys
command="borg serve --restrict-to-path /home/lemp1see/borg",restrict {insert here the ssh key}
chown -R lemp1see: /home/lemp1see
```
