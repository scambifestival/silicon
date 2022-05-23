## Virtual LAN pillar

Host with primary Debian 10, openvpn e dnsmasq.

Records for the Virtual LAN use the first-level domain `.scambi`  
file `/etc/dnsmasq.d/dnsmasq.hosts`, examples:

    192.168.64.1    pila1het.scambi
    192.168.64.2    pila2sca.scambi

<br/>

### Procedure

root password: see KeePass

set Italian keyboard
>localectl set-keymap it

set the time zone and install chrony for NTP
>timedatectl set-timezone Europe/Rome  
>apt install chrony

generate id_rsa key
>ssh-keygen -t rsa -b 4096

add swap (suggested)
>dd if=/dev/zero of=/swapfile count=1024 bs=1M status=progress  
>chmod 600 /swapfile  
>mkswap /swapfile  
>swapon /swapfile  
>echo '/swapfile   none    swap    sw    0   0' | tee -a /etc/fstab  
>free -m

install useful packages
>apt install dnsutils unzip screen

edit ssh configuration
>nano /etc/ssh/sshd_config

    Port 822
    PermitRootLogin without-password

Create service user
>adduser silicon (vedi Keepass)

configure certification authority for openvpn
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

generate server certificate (example with `vpn1.scambi.org` values)
>./easyrsa gen-req vpn1.scambi.org nopass  
>./easyrsa show-req vpn1.scambi.org  
>./easyrsa sign-req server vpn1.scambi.org  
>./easyrsa gen-dh  
>openvpn --genkey --secret /etc/openvpn/server/ta.key  

>cp /etc/openvpn/easyrsa/pki/ca.crt /etc/openvpn/server/  
>cp /etc/openvpn/easyrsa/pki/issued/vpn1.scambi.org.crt /etc/openvpn/server/  
>cp /etc/openvpn/easyrsa/pki/private/vpn1.scambi.org.key /etc/openvpn/server/  
>cp /etc/openvpn/easyrsa/pki/dh.pem /etc/openvpn/server/  

generate client certificate
>./easyrsa gen-req client-USERNAME nopass

otherwise, with password

>./easyrsa gen-req client-USERNAME

sign the certificate
>./easyrsa sign-req client client-USERNAME

to handle revocations, execute the command
>./easyrsa revoke client-USERNAME (in caso di necessità)

generate CRL certificate (both the first time and after adding revocations)
>./easyrsa gen-crl  
>cp /etc/openvpn/easyrsa/pki/crl.pem /etc/openvpn/server/

to create the .ovpn file, use "ovpngen" (see https://github.com/graysky2/ovpngen)

>/root/ovpngen vpn1.scambi.org /etc/openvpn/server/ca.crt /etc/openvpn/easyrsa/pki/issued/client-USERNAME.crt /etc/openvpn/easyrsa/pki/private/client-USERNAME.key /etc/openvpn/server/ta.key 6990 udp > /home/silicon/client-USERNAME.ovpn

edit client-USERNAME.ovpn uncommenting "cipher" e "auth"

### openvpn configuration

>nano /etc/openvpn/server.conf

    port 6990
    proto udp
    dev tun

    ca /etc/openvpn/server/ca.crt
    cert /etc/openvpn/server/vpn1.scambi.org.crt
    key /etc/openvpn/server/vpn1.scambi.org.key
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

    tls-auth /etc/openvpn/server/ta.key 0

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


enable and start openvpn
>systemctl enable openvpn@server  
>systemctl start openvpn@server

enable IP forwarding
>nano /etc/sysctl.d/88-openvpn.conf

    net.ipv4.ip_forward=1

>sysctl --system


install and configure tinc
>apt install tinc

>cd /etc/tinc  
>mkdir -p scambi/hosts

>nano nets.boot

    scambi

>nano scambi/tinc.conf

    Name=pila1het
    Device=/dev/net/tun
    AddressFamily=ipv4
    Mode=switch
    Port=6996

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

>nano scambi/hosts/pila1het

    Subnet = 192.168.64.1/32
    Address = vpn1.scambi.org
    Port = 6996

enable and start tinc
>systemctl enable tinc  
>mkdir /etc/systemd/system/tinc\@scambi.service.d

>nano /etc/systemd/system/tinc\@scambi.service.d/after.conf

    [Unit]
    After=network.target


>systemctl enable tinc@scambi  
>systemctl start tinc@scambi

dnsmasq configuration
>apt install dnsmasq

>nano /etc/default/dnsmasq

    aggiungere ",.hosts,.resolv" alla variabile CONFIG_DIR

>nano /etc/dnsmasq.d/dnsmasq.resolv

    nameserver DNS-FORNITORE
    nameserver 1.0.0.1

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

edit `/etc/resolv.conf` by adding at the beginning of the file

    nameserver 127.0.0.1

in `/etc/dhcp/dhclient.conf`, uncomment:

    prepend domain-name-servers 127.0.0.1;

enable and start the service
>systemctl enable --now dnsmasq  
>systemctl restart dnsmasq

in the primary, it is necessary to set a CRON job to replicate the file `/etc/dnsmasq.d/dnsmasq.hosts`
>nano /root/sync_dnsmasq.sh

    #!/bin/bash
    /usr/bin/scp -p -q -P822 /etc/dnsmasq.d/dnsmasq.hosts pila2sca.scambi:/etc/dnsmasq.d/dnsmasq.hosts
    sleep 1
    /usr/bin/ssh -p822 pila2sca.scambi systemctl restart dnsmasq

>crontab -e

    05 06 * * * /root/sync_dnsmasq.sh

>chmod +x /root/sync_dnsmasq.sh

apply firewall rules with firewalld
>update-alternatives --config ip6tables (mettere legacy)  
>update-alternatives --config iptables (mettere legacy)

>apt install firewalld

verify that the public interface is *eth0*, otherwise, edit the row below:

>firewall-cmd --state  
>firewall-cmd --permanent --zone=public --add-interface=eth0  
>firewall-cmd --permanent --zone=internal --add-interface=scambi  

>firewall-cmd --permanent --zone=internal --remove-service={mdns,samba-client,ssh}  

>firewall-cmd --permanent --new-service=aa_ssh  
>firewall-cmd --permanent --service=aa_ssh --set-description='custom ssh'  
>firewall-cmd --permanent --service=aa_ssh --add-port=822/tcp  

>firewall-cmd --permanent --new-service=aa_openvpn  
>firewall-cmd --permanent --service=aa_openvpn --set-description='custom openvpn'  
>firewall-cmd --permanent --service=aa_openvpn --add-port=6990/udp  

>firewall-cmd --permanent --new-service=aa_tinc  
>firewall-cmd --permanent --service=aa_tinc --set-description='custom tinc'  
>firewall-cmd --permanent --service=aa_tinc --add-port=6996/tcp  
>firewall-cmd --permanent --service=aa_tinc --add-port=6996/udp  

>firewall-cmd --permanent --zone=internal --add-service={aa_ssh,dns,snmp}  
>firewall-cmd --permanent --zone=home --add-service=aa_ssh  

>firewall-cmd --permanent --zone=public --remove-service=ssh  
>firewall-cmd --permanent --zone=public --add-service={aa_ssh,aa_openvpn,aa_tinc}  

>firewall-cmd --permanent --zone=internal --add-interface=tun0  
>firewall-cmd --permanent --direct --add-rule ipv4 filter FORWARD 0 -i tun0 -j ACCEPT  
>firewall-cmd --permanent --direct --add-rule ipv4 filter FORWARD 0 -m state --state ESTABLISHED,RELATED -j ACCEPT  

>firewall-cmd --permanent --direct --add-rule ipv4 filter FORWARD 0 -i scambi -j ACCEPT  

>firewall-cmd --permanent --zone=public --set-target=DROP  
>firewall-cmd --permanent --direct --add-rule ipv4 filter INPUT 0 -p icmp -s 0.0.0.0/0 -d 0.0.0.0/0 -j ACCEPT  

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

    [sshd]
    enabled = true
    port = 822
    logpath = %(sshd_log)s
    backend = %(sshd_backend)s

>nano /etc/fail2ban/action.d/iptables-common.local

    [Init]
    blocktype = DROP

>systemctl restart fail2ban

<br/>

### VPN certificates cheat sheet

##### create user
>./easyrsa gen-req client-USERNAME nopass  

alternatively, with password

>./easyrsa gen-req client-USERNAME

sign the certificate (the password is the CA one)
>./easyrsa sign-req client client-USERNAME

to create the .ovpn file, use "ovpngen" (see <https://github.com/graysky2/ovpngen>)

edit ovpngen uncommenting "cipher" e "auth" (rows 69-70)

>/root/ovpngen vpn1.scambi.org /etc/openvpn/server/ca.crt /etc/openvpn/easyrsa/pki/issued/client-USERNAME.crt /etc/openvpn/easyrsa/pki/private/client-USERNAME.key /etc/openvpn/server/ta.key 6990 udp > /home/silicon/client-USERNAME.ovpn

##### renew user

sign the certificate (the password is the CA one)
>./easyrsa sign-req client client-USERNAME

>/root/ovpngen vpn1.scambi.org /etc/openvpn/server/ca.crt /etc/openvpn/easyrsa/pki/issued/client-USERNAME.crt /etc/openvpn/easyrsa/pki/private/client-USERNAME.key /etc/openvpn/server/ta.key 6990 udp > /home/silicon/client-USERNAME.ovpn