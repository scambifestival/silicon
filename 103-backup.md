## Backup

time4vps non supporta ancora Debian 11 sui VPS storage, installato Debian 9

<br/> **procedura**

seguire template Debian 11

fix locale
>apt install locales
>dpkg-reconfigure locales
>>selezionare it_IT.UTF-8

installazione pacchetti utili
>apt install rsync borgbackup iftop screen

configurazione cartelle
>mkdir -p /data/"hostname"/{backup,borg,log}

esempio:
>mkdir -p /data/{pila1see,lemp1see,stor1see}/{backup,borg,log}
