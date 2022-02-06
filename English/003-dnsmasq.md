## dnsmasq

Light software that can act both as DNS server/cache and as DHCP.  
In the virtual LAN, only the DNS part is used, in order to resolve internal names with first-level domain `.scambi` (not a TLD).

This document is for “normal” hosts.

<br>

### Procedure 

`apt install dnsmasq`

`nano /etc/default/dnsmasq`

add ",.hosts,.resolv" to the `CONFIG_DIR` variable

verify that in the file `/etc/dnsmasq.conf` is uncommented

    conf-dir=/etc/dnsmasq.d/,*.conf


>nano /etc/dnsmasq.d/dnsmasq.resolv

    nameserver 192.168.64.1
    nameserver 192.168.64.2
    nameserver 9.9.9.9

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

at the beginning of file `/etc/resolv.conf` add:

    nameserver 127.0.0.1

in `/etc/dhcp/dhclient.conf`, uncomment the following:

    prepend domain-name-servers 127.0.0.1;

enable and start the service:  
`systemctl enable --now dnsmasq`  
`systemctl restart dnsmasq`

<br />

### Contabo - fix DNS

`nano /etc/network/interfaces`

    ...
    dns-nameservers 127.0.0.1 213.136.95.10 213.136.95.11
    ...

>systemctl disable --now resolvconf  
>rm /run/dnsmasq/resolv.conf  
>systemctl restart dnsmasq

>ls -l /etc/resolv.conf

if */etc/resolv.conf* is a symbolic link:
>rm /etc/resolv.conf

>nano /etc/resolv.conf

    nameserver 127.0.0.1
