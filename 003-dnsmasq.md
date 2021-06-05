## dnsmasq

Software leggero che può fare sia da DNS server/cache sia da DHCP.  
Nella LAN virtuale si usa solo la parte DNS, per risolvere i nomi interni con dominio di primo livello .scambi (non è un Top-Level Domain)

Questo documento è per gli host "normali".

<br/> **procedura**

> apt install dnsmasq

>nano /etc/default/dnsmasq

  	aggiungere ",.hosts,.resolv" alla variabile CONFIG_DIR

verificare che nel file /etc/dnsmasq.conf sia decommentato

    conf-dir=/etc/dnsmasq.d/,*.conf


>nano /etc/dnsmasq.d/dnsmasq.resolv

    nameserver 192.168.64.1
    nameserver 192.168.64.2
    nameserver 1.0.0.1

>nano /etc/dnsmasq.d/dnsmasq.conf

    #not read /etc/hosts
    no-hosts
    # don't read /etc/resolv.conf
    #no-resolv
    resolv-file=/etc/dnsmasq.d/dnsmasq.resolv
    # follow order of servers
    strict-order

    # number of names in cache
    cache-size=4096

    # add domain to simple names
    expand-hosts

    # ttl cache
    max-cache-ttl=1800

    # interface
    interface=lo
    # accept dns queries only from address of local subnet
    local-service
    # force binding only on selected interfaces
    bind-interfaces

    # not forward query without fqdn
    domain-needed

modificare /etc/resolv.conf mettendo in testa al file

    nameserver 127.0.0.1

modificare /etc/dhcp/dhclient.conf, decommentando

    prepend domain-name-servers 127.0.0.1;

abilitare e far partire il servizio
>systemctl enable --now dnsmasq  
>systemctl restart dnsmasq


<br/> **contabo - fix DNS**

nano /etc/network/interfaces

    ...
    dns-nameservers 127.0.0.1 213.136.95.10 213.136.95.11
    ...

>systemctl disable --now resolvconf  
>rm /run/dnsmasq/resolv.conf  
>systemctl restart dnsmasq

>ls -l /etc/resolv.conf

se */etc/resolv.conf* è un link simbolico:
>rm /etc/resolv.conf

>nano /etc/resolv.conf

    nameserver 127.0.0.1
