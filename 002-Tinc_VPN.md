# Tinc VPN - Virtual LAN

Software for mesh connections, used to create a virtual LAN.  
Set the hostname and the internal IP address correctly (see [the introduction](README.md).)

These document is for "normal" servers.

## Procedure

>apt install tinc

>mkdir -p /etc/tinc/scambi/hosts  
>cd /etc/tinc

>nano nets.boot

```
scambi
```

>nano scambi/tinc.conf

(`HOST_NAME` must be substituted with the host name)

```
Name=HOST_NAME
Device=/dev/net/tun
AddressFamily=ipv4
Mode=switch
Port=777

Cipher=aes-256-cbc
Digest=SHA512

ConnectTo=pila1see
ConnectTo=pila2sca
```

>nano scambi/tinc-up

```
#!/bin/sh
ip link set $INTERFACE up
ip addr add 192.168.64.X/24 dev $INTERFACE
```

>nano scambi/tinc-down

```
#!/bin/sh
ip addr del 192.168.64.X/24 dev $INTERFACE
ip link set $INTERFACE down
```

>chmod +x scambi/tinc-*

>tincd -n scambi -K 4096

>nano scambi/hosts/”nome host”

```
Subnet = 192.168.64.X/32
Port = 777
```

copy `scambi/hosts/HOST_NAME` on pila1see  
copy `scambi/hosts/pila1see` on this host

>nano scambi/hosts/pila1see-up

```
#!/bin/sh
ip route add 192.168.64.1 dev $INTERFACE
ip route add 192.168.66.0/24 via 192.168.64.1 dev $INTERFACE
```

>nano scambi/hosts/pila1see-down

```
#!/bin/sh
ip route del 192.168.66.0/24 via 192.168.64.1 dev $INTERFACE
ip route del 192.68.64.1 dev $INTERFACE
```

>chmod +x scambi/hosts/pila1see-*

copy `scambi/hosts/HOST_NAME` on pila2csa  
copy `scambi/hosts/pila2sca` on this host

>nano scambi/hosts/pila2sca-up

```
#!/bin/sh
ip route add 192.168.64.2 dev $INTERFACE
ip route add 192.168.68.0/24 via 192.168.64.2 dev $INTERFACE
```

>nano scambi/hosts/pila2sca-down

```
#!/bin/sh
ip route del 192.168.68.0/24 via 192.168.64.2 dev $INTERFACE
ip route del 192.168.64.2 dev $INTERFACE
```

>chmod +x scambi/hosts/pila2sca-*

### Enable and start

Enable and start the service:

>systemctl enable tinc  
>mkdir /etc/systemd/system/tinc\@scambi.service.d  
>nano /etc/systemd/system/tinc\@scambi.service.d/after.conf

```
[Unit]
After=network.target
```

>systemctl enable --now tinc@scambi

add DNS records on pila1see

## Tricks

to have the list of nodes, so we can read it with the command `journalctl -u tinc@scambi`

>kill -s USR2 “tinc process PID”

purge deleted nodes
>tincd -n scambi -kWINCH
