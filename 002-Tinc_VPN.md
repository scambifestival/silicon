## Tinc VPN - Virtual LAN

Software per connessioni mesh, utilizzato per creare una LAN virtuale.  
Impostare correttamente il nome host e l'indirizzo IP interno 'X' (vedi [000-Introduzione](000-Introduzione.md)).

Questo documento è per gli host "normali".

<br/> **procedura**

>apt install tinc

>mkdir -p /etc/tinc/scambi/hosts  
>cd /etc/tinc

>nano nets.boot

    scambi  

>nano scambi/tinc.conf  

    Name="nome host" (senza virgolette)
    Device=/dev/net/tun
    AddressFamily=ipv4
    Mode=switch
    Port=6996

    Cipher=aes-256-cbc
    Digest=SHA512

    ConnectTo=pila1het
    #ConnectTo=pila2xyz

>nano scambi/tinc-up

    #!/bin/sh
    ip link set $INTERFACE up
    ip addr add 192.168.64.X/24 dev $INTERFACE

>nano scambi/tinc-down

    #!/bin/sh
    ip addr del 192.168.64.X/24 dev $INTERFACE
    ip link set $INTERFACE down

>chmod +x scambi/tinc-*

>tincd -n scambi -K 4096

>nano scambi/hosts/”nome host”

    Subnet = 192.168.64.X/32
    Port = 6996

copiare scambi/hosts/”nome host” su pila1sca  
copiare scambi/hosts/pila1het su questo host

>nano scambi/hosts/pila1het-up

    #!/bin/sh
    ip route add 192.168.64.1 dev $INTERFACE
    ip route add 192.168.66.0/24 via 192.68.64.1 dev $INTERFACE

>nano scambi/hosts/pila1het-down

    #!/bin/sh
    ip route del 192.168.66.0/24 via 192.68.64.1 dev $INTERFACE
    ip route del 192.68.64.1 dev $INTERFACE

>chmod +x scambi/hosts/pila1het-*

copiare scambi/hosts/”nome host” su pila2xyz  
copiare scambi/hosts/pila2xyz su questo host

>nano scambi/hosts/pila2xyz-up

    #!/bin/sh
    ip route add 192.168.64.2 dev $INTERFACE
    ip route add 192.168.68.0/24 via 192.168.64.2 dev $INTERFACE

>nano scambi/hosts/pila2xyz-down

    #!/bin/sh
    ip route del 192.168.68.0/24 via 192.168.64.2 dev $INTERFACE
    ip route del 192.168.64.2 dev $INTERFACE

>chmod +x scambi/hosts/pila2xyz-*


<br/> abilitare e far partire il servizio

>systemctl enable tinc  
>mkdir /etc/systemd/system/tinc\@scambi.service.d  

>nano /etc/systemd/system/tinc\@scambi.service.d/after.conf

    [Unit]
    After=network.target

>systemctl enable --now tinc@scambi

<br/> aggiungere record al dns su pila1sca

<br/> **tricks**

avere lista nodi, da leggere poi con il comando *journalctl -u tinc@scambi*

>kill -s USR2 “tinc process PID”

purge nodi cancellati
>tincd -n scambi -kWINCH
