# Introduction

All of the servers have a public IP where they provide the services.

Two of them (hosted by two different providers, in order to avoid a single point of failure) act as entrypoints to the entire infrastructure, connected together by [tinc VPN](https://tinc-vpn.org 'tinc official website').  
Both servers have also an [OpenVPN server](https://openvpn.net/access-server 'Access Server | OpenVPN') (with DNS records `vipien1` and `vipien2`).

To avoid overlapping, the two VPN servers handle two different networks:

- `vipien1` - 192.168.66.0/24
- `vipien2` - 192.168.68.0/24

With tinc, routes are added for these two networks with the correct gateway.

## Servers

Below is the list of servers sorted by "internal" IP (tinc)

The naming convention for hostnames is as follows:

- 4 letters: function
- 1 number: incremental
- 3 letters: supplier abbreviation

| IP TINC      | Hostname | SSH[^1] | Role                  | Spec
| ------------ | -------- | ------- | --------------------- |------------------
| 192.168.64.1 | pila1see | 822     | vipien1               |                 |
| 192.168.64.2 | pila2sca | 822     | vipien2               |                 |
| 192.168.64.3 | bckp1t4v | 822     | backup                |                 |
| 192.168.64.4 | lemp1see | 22      | LEMP scambi.org       |                 |
| 192.168.64.5 | stor1see | 22      | storage Nextcloud     |                 |
| 192.168.64.6 | lemp2see | 22      | Nginx Collabora       |                 |
| 192.168.64.7 | data1see | 22      | database Baserow      |2 Core, 2 Gb RAM | 
| 192.168.64.8 |          | 22      |                       |                 |

## Suppliers list

- `see` - [Seeweb](https://seeweb.it)
- `t4v` - [Time4VPS](https://time4vps.com)
- `sca` - [Scaleway](https://scaleway.com)
- `het` - [Hetzner](https://hetzner.com)
- `con` - [Contabo](https://contabo.com)

[^1]: `822` - SSH internet | `22` - SSH LAN
