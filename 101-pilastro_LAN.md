## Pilastro LAN Virtuale

Host con Debian 11, openvpn e dnsmasq primari.

I record per LAN Virtuale utilizzano il dominio di primo livello *.scambi*  
file /etc/dnsmasq.d/dnsmasq.hosts, esempi:

    192.168.64.1    pila1see.scambi
    192.168.64.2    pila2sca.scambi

<br/> **procedura**

password di root : vedi Keepass

impostare tastiera italiana
>localectl set-keymap it

impostare hostname
>hostnamectl set-hostname pila1see

impostare fuso orario e installare chrony per NTP
>timedatectl set-timezone Europe/Rome  
>apt install chrony

generare chiave id_ed25519
>ssh-keygen -t ed25519

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

    Port 822
    PermitRootLogin without-password

creare utente di servizio
>adduser silicon (vedi Keepass)

configurazione certification authority per openvpn
>apt install openvpn easy-rsa  
>cd /etc/openvpn  
>make-cadir easyrsa  
>cd easyrsa  

>nano vars

    set_var EASYRSA                 "${0%/*}"
    set_var EASYRSA_PKI             "$PWD/pki"
    set_var EASYRSA_DN              "org"
    set_var EASYRSA_REQ_COUNTRY     "IT"
    set_var EASYRSA_REQ_PROVINCE    "IM"
    set_var EASYRSA_REQ_CITY        "Sanremo"
    set_var EASYRSA_REQ_ORG         "scambi"
    set_var EASYRSA_REQ_EMAIL       "info@scambi.org"
    set_var EASYRSA_REQ_OU          "Staff"
    set_var EASYRSA_KEY_SIZE        4096
    set_var EASYRSA_ALGO            rsa
    set_var EASYRSA_CA_EXPIRE       3650
    set_var EASYRSA_CERT_EXPIRE     365
    set_var EASYRSA_CRL_DAYS        3650
    set_var EASYRSA_NS_SUPPORT      "no"
    set_var EASYRSA_NS_COMMENT      "SCAMBI CERTIFICATE AUTHORITY"
    set_var EASYRSA_EXT_DIR         "$EASYRSA/x509-types"


>chmod +x vars  
>./easyrsa init-pki  
>./easyrsa build-ca (vedi Keepass - ca.scambi.org)  

generazione certificato server (esempio per vipien1.scambi.org)
>./easyrsa gen-req vipien1.scambi.org nopass  
>./easyrsa show-req vipien1.scambi.org  
>./easyrsa sign-req server vipien1.scambi.org  
>./easyrsa gen-dh  
>openvpn --genkey tls-crypt /etc/openvpn/server/ta.key  

>cp /etc/openvpn/easyrsa/pki/ca.crt /etc/openvpn/server/  
>cp /etc/openvpn/easyrsa/pki/issued/vipien1.scambi.org.crt /etc/openvpn/server/  
>cp /etc/openvpn/easyrsa/pki/private/vipien1.scambi.org.key /etc/openvpn/server/  
>cp /etc/openvpn/easyrsa/pki/dh.pem /etc/openvpn/server/  

generazione certificato client
>./easyrsa gen-req client-USERNAME nopass

oppure con password

>./easyrsa gen-req client-USERNAME

firmare il certificato
>./easyrsa sign-req client client-USERNAME

per gestire le revoche, occorre eseguire il comando
>./easyrsa revoke client-USERNAME (in caso di necessità)

generare il certificato CRL (sia la prima volta, sia dopo aver aggiunto revoche)
>./easyrsa gen-crl  
>cp /etc/openvpn/easyrsa/pki/crl.pem /etc/openvpn/server/

per creare file .ovpn, usare "ovpngen" (vedi https://github.com/nuciluc/ovpngen)

>/root/ovpngen vipien1.scambi.org /etc/openvpn/server/ca.crt /etc/openvpn/easyrsa/pki/issued/client-USERNAME.crt /etc/openvpn/easyrsa/pki/private/client-USERNAME.key /etc/openvpn/server/ta.key 1111 udp > /home/silicon/client-USERNAME.ovpn

configurazione openvpn

>nano /etc/openvpn/server.conf

    port 1111
    proto udp
    dev tun

    ca /etc/openvpn/server/ca.crt
    cert /etc/openvpn/server/vipien1.scambi.org.crt
    key /etc/openvpn/server/vipien1.scambi.org.key
    dh /etc/openvpn/server/dh.pem
    crl-verify /etc/openvpn/server/crl.pem

    server 192.168.66.0 255.255.255.0

    keepalive 10 120

    #comp-lzo
    persist-key
    persist-tun

    status openvpn-status.log
    verb 3

    #client-to-client

    tls-crypt /etc/openvpn/server/ta.key

    cipher AES-256-CBC
    auth SHA512
    tls-version-min 1.2
    tls-cipher TLS-DHE-RSA-WITH-AES-256-GCM-SHA384:TLS-DHE-RSA-WITH-AES-256-CBC-SHA:TLS-DHE-RSA-WITH-CAMELLIA-256-CBC-SHA
    ncp-ciphers AES-256-GCM:AES-256-CBC

    push "route 192.168.64.0 255.255.255.0"
    push "dhcp-option DNS 192.168.66.1"
    #push "redirect-gateway def1 bypass-dhcp"
    #push "redirect-gateway def1"

    user nobody
    group nogroup

    duplicate-cn


abilitare e far partire openvpn
>systemctl enable --now openvpn@server  

abilitare ip forward
>nano /etc/sysctl.d/88-openvpn.conf

    net.ipv4.ip_forward=1

>sysctl --system


installazione e configurazione tinc
>apt install tinc

>mkdir -p /etc/tinc/scambi/hosts  
>cd /etc/tinc

>nano nets.boot

    scambi

>nano scambi/tinc.conf

    Name=pila1see
    Device=/dev/net/tun
    AddressFamily=ipv4
    Mode=switch
    Port=777

    Cipher=aes-256-cbc
    Digest=SHA512

    #ConnectTo=pila2sca

>nano scambi/tinc-up

    #!/bin/sh
    ip link set $INTERFACE up
    ip addr add 192.168.64.1/24 dev $INTERFACE

>nano scambi/tinc-down

    #!/bin/sh
    ip addr del 192.168.64.1/24 dev $INTERFACE
    ip link set $INTERFACE down

>chmod +x scambi/tinc-*

>tincd -n scambi -K 4096

>nano scambi/hosts/pila1see

    Subnet = 192.168.64.1/32
    Address = vipien1.scambi.org
    Port = 777

abilitare e far partire tinc
>systemctl enable tinc  
>mkdir /etc/systemd/system/tinc\@scambi.service.d

>nano /etc/systemd/system/tinc\@scambi.service.d/after.conf

    [Unit]
    After=network.target


>systemctl enable --now tinc@scambi  

configurazione dnsmasq
>apt install dnsmasq

>nano /etc/default/dnsmasq

  	aggiungere ",.hosts,.resolv" alla variabile CONFIG_DIR

verificare che nel file /etc/dnsmasq.conf sia decommentato

    conf-dir=/etc/dnsmasq.d/,*.conf


>nano /etc/dnsmasq.d/dnsmasq.resolv

    nameserver DNS-FORNITORE
    nameserver 9.9.9.9

>nano /etc/dnsmasq.d/dnsmasq.conf

    #not read /etc/hosts
    no-hosts
    addn-hosts=/etc/dnsmasq.d/dnsmasq.hosts
    # don't read /etc/resolv.conf
    #no-resolv
    resolv-file=/etc/dnsmasq.d/dnsmasq.resolv
    # follow order of servers
    strict-order

    #server=/domain.com/8.8.8.8
    #rev-server=192.168.0.0/24,192.168.0.1
    #address=/banner.com/127.0.0.1

    #host-record=pippo,pippo.home,192.168.0.X,ipv6 (it creates PTR too)
    #txt-record=name,txt
    #ptr-record=name,target
    #cname=cname,target

    # number of names in cache
    cache-size=4096

    # add domain to simple names
    expand-hosts
    # domain of local dhcp ips
    domain=scambi,192.168.64.0/24,local

    # ttl from /etc/hosts or dhcp leases
    local-ttl=1800
    # ttl sent to clients
    #max-ttl=600
    # ttl cache
    max-cache-ttl=1800

    # interface
    #interface=lo
    except-interface=<interfaccia pubblica>
    # accept dns queries only from address of local subnet
    local-service
    # force binding only on selected interfaces - dynamic
    bind-dynamic

    # not forward local "not found" address upstream
    bogus-priv
    # not forward query without fqdn
    domain-needed

modificare /etc/resolv.conf mettendo in testa al file

    nameserver 127.0.0.1

modificare /etc/dhcp/dhclient.conf, se presente, decommentando

    prepend domain-name-servers 127.0.0.1;

abilitare e far partire il servizio
>systemctl enable --now dnsmasq  
>systemctl restart dnsmasq

sul primario occorre impostare un cron job per replicare il file */etc/dnsmasq.d/dnsmasq.hosts*
>nano /root/sync_dnsmasq.sh

    #!/bin/bash
    /usr/bin/scp -p -q -P822 /etc/dnsmasq.d/dnsmasq.hosts pila2sca.scambi:/etc/dnsmasq.d/dnsmasq.hosts
    sleep 1
    /usr/bin/ssh -p822 pila2sca.scambi systemctl restart dnsmasq

>crontab -e

    05 06 * * * /root/sync_dnsmasq.sh

>chmod +x /root/sync_dnsmasq.sh

applicare regole firewall con firewalld
>apt install firewalld

verificare che l'interfaccia pubblica sia *eth0*, altrimenti modificare la riga qui sotto

>firewall-cmd --state  
>firewall-cmd --permanent --zone=public --add-interface=eth0  
>firewall-cmd --permanent --zone=internal --add-interface=scambi  

>firewall-cmd --permanent --zone=internal --remove-service={mdns,samba-client,ssh}  

>firewall-cmd --permanent --new-service=aa_ssh  
>firewall-cmd --permanent --service=aa_ssh --set-description='custom ssh'  
>firewall-cmd --permanent --service=aa_ssh --add-port=822/tcp  

>firewall-cmd --permanent --new-service=aa_openvpn  
>firewall-cmd --permanent --service=aa_openvpn --set-description='custom openvpn'  
>firewall-cmd --permanent --service=aa_openvpn --add-port=1111/udp  

>firewall-cmd --permanent --new-service=aa_tinc  
>firewall-cmd --permanent --service=aa_tinc --set-description='custom tinc'  
>firewall-cmd --permanent --service=aa_tinc --add-port=777/tcp  
>firewall-cmd --permanent --service=aa_tinc --add-port=777/udp  

>firewall-cmd --permanent --zone=internal --add-service={aa_ssh,dns,snmp}  
>firewall-cmd --permanent --zone=home --add-service=aa_ssh  

>firewall-cmd --permanent --zone=public --remove-service=ssh  
>firewall-cmd --permanent --zone=public --add-service={aa_ssh,aa_openvpn,aa_tinc}  

>firewall-cmd --permanent --zone=internal --add-interface=tun0  
>firewall-cmd --permanent --zone=internal --add-forward

>firewall-cmd --permanent --zone=public --set-target=DROP  
>firewall-cmd --permanent --add-icmp-block-inversion  
>firewall-cmd --permanent --zone=public --add-icmp-block={echo-reply,echo-request,port-unreachable,time-exceeded}  

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

<br/> **prontuario comandi per certificati VPN**

###### creare utente
>./easyrsa gen-req client-USERNAME nopass  

oppure con password

>./easyrsa gen-req client-USERNAME

firmare il certificato (la password è quella della CA)
>./easyrsa sign-req client client-USERNAME

per creare file .ovpn, usare "ovpngen" (vedi https://github.com/nuciluc/ovpngen)

>/root/ovpngen vipien1.scambi.org /etc/openvpn/server/ca.crt /etc/openvpn/easyrsa/pki/issued/client-USERNAME.crt /etc/openvpn/easyrsa/pki/private/client-USERNAME.key /etc/openvpn/server/ta.key 1111 udp > /home/silicon/client-USERNAME.ovpn

###### rinnovare utente

firmare il certificato (la password è quella della CA)
>./easyrsa sign-req client client-USERNAME

>/root/ovpngen vipien1.scambi.org /etc/openvpn/server/ca.crt /etc/openvpn/easyrsa/pki/issued/client-USERNAME.crt /etc/openvpn/easyrsa/pki/private/client-USERNAME.key /etc/openvpn/server/ta.key 1111 udp > /home/silicon/client-USERNAME.ovpn
