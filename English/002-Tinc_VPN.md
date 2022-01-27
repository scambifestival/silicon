## Tinc VPN - Virtual LAN

Software for mesh connections, used to create a virtual LAN.
Set the correct hostname and the internal IP address consistently with the criteria established in [000-Introduction](000-Introduction.md).

This document is for “normal” hosts.

<br>

### Procedure

`apt install tinc`

```
mkdir -p /etc/tinc/scambi/hosts
cd /etc/tinc
```

`nano nets.boot`

    scambi  

`nano scambi/tinc.conf` 

    Name="nome host" (senza virgolette)
    Device=/dev/net/tun
    AddressFamily=ipv4
    Mode=switch
    Port=6996

    Cipher=aes-256-cbc
    Digest=SHA512

    ConnectTo=pila1het
    #ConnectTo=pila2sca

`nano scambi/tinc-up`

    #!/bin/sh
    ip link set $INTERFACE up
    ip addr add 192.168.64.X/24 dev $INTERFACE

`nano scambi/tinc-down`

    #!/bin/sh
    ip addr del 192.168.64.X/24 dev $INTERFACE
    ip link set $INTERFACE down

`chmod +x scambi/tinc-*`

`tincd -n scambi -K 4096`

`nano scambi/hosts/“hostname” # replace “hostname” with the actual one`

    Subnet = 192.168.64.X/32
    Port = 6996

copy `scambi/hosts/hostname` on `pila1sca`
copy `scambi/hosts/pila1het` on this host

`nano scambi/hosts/pila1het-up`

    #!/bin/sh
    ip route add 192.168.64.1 dev $INTERFACE
    ip route add 192.168.66.0/24 via 192.168.64.1 dev $INTERFACE

`nano scambi/hosts/pila1het-down`

    #!/bin/sh
    ip route del 192.168.66.0/24 via 192.168.64.1 dev $INTERFACE
    ip route del 192.68.64.1 dev $INTERFACE

`chmod +x scambi/hosts/pila1het-`

copy `scambi/hosts/“hostname”` on `pila2sca`
copy `scambi/hosts/pila2sca` on this host

`nano scambi/hosts/pila2sca-up`

    #!/bin/sh
    ip route add 192.168.64.2 dev $INTERFACE
    ip route add 192.168.68.0/24 via 192.168.64.2 dev $INTERFACE

`nano scambi/hosts/pila2sca-down`

    #!/bin/sh
    ip route del 192.168.68.0/24 via 192.168.64.2 dev $INTERFACE
    ip route del 192.168.64.2 dev $INTERFACE

`chmod +x scambi/hosts/pila2sca-*`


<br>

enable and start the service

```
systemctl enable tinc
mkdir /etc/systemd/system/tinc\@scambi.service.d
```

`nano /etc/systemd/system/tinc\@scambi.service.d/after.conf`

    [Unit]
    After=network.target

`systemctl enable --now tinc@scambi`

add record to DNS on `pila1sca`

<br>

##### tricks

Having a list of nodes, to be read with the command `journalctl -u tinc@scambi`

`kill -s USR2 “tinc process PID”`

purge deleted nodes
`tincd -n scambi -kWINCH`
