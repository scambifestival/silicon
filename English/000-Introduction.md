## Introduction

All of the servers have a public IP, where they will provide the services.

Two of them (hosted by two different providers, in order to avoid a single point of failure) act as entrypoints to the whole infrastructure, connected within itself by TINC VPN.  
Both servers have an OpenVPN server (with DNS records `vpn1` and `vpn2`), too.

To avoid overlapping, the two VPN servers handle two different networks:
- `vpn1`: `192.168.66.0/24`
- `vpn2`: `192.168.68.0/24`

Thanks to TINC, routes for these two networks are added with the correct gateway.

The naming convention for the hostnames is the following:
- 4 letters: function
- 1 number: incremental
- 3 letters: hosting provider identifier

Below the list of the servers, ordered according to “internal” (TINC) IPs.

| TINC IP | hostname | ssh * | role | Public IP |
| --- | --- | --- | --- | --- |
| 192.168.64.1 | pila1het | 822 | vpn1 | 94.130.24.241 |
| 192.168.64.2 | pila2??? | 822 | vpn2 |  |
| 192.168.64.3 | bckp1t4v | 822 | backup |  |
| 192.168.64.4 | lemp1sca | 22 | LEMP scambi.org | ***REMOVED*** |
| 192.168.64.5 | stor1con | 22 | storage nextcloud | 207.180.218.154 |
| 192.168.64.6 | lemp2het | 22 | LEMP | 142.132.182.176 |

\* 822 - ssh internet | 22 - ssh LAN

Hosting providers in use:

- het - Hetzner
- sca - Scaleway
- t4v - Time4vps
- con - Contabo
