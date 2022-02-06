## Backup

Since Time4VPS does not support Debian 10 on storage VPSs yet, Debian 9 has been installed.

<br/>

# Procedure

Follow Debian 10 template

fix locale
>apt install locales
>dpkg-reconfigure locales
>>selezionare it_IT.UTF-8

install useful packages
>apt install rsync borgbackup iftop screen

directories configuration
>mkdir -p /data/"hostname"/{backup,borg,log}

example:
>mkdir -p /data/{pila1het,lemp1sca,lamp1con}/{backup,borg,log}
