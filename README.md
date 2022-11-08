---
description: This repository contains instructions for the installation and maintenance of Scambi Festival self-hosted infrastructure.
---
# Introduction

All of the servers have a public IP, where they will provide the services.

Two of them (hosted by two different providers, in order to avoid a single point of failure) act as entrypoints to the entire infrastructure, connected together by TINC vpn.  
Both servers have also an OpenVPN server (with DNS records vipien1 and vipien2).

To avoid overlapping, the two VPN servers handle two different networks:
- vipien1 - 192.168.66.0/24
- vipien2 - 192.168.68.0/24

With TINC, routes are added for these two networks with the correct gateway.

The naming convention for hostnames is as follows:
- 4 letters: function
- 1 number: incremental
- 3 letters: supplier abbreviation

Below is the list of servers sorted by "internal" IP (TINC)

| IP TINC | hostname | ssh \* | role | public IP |
| --- | --- | --- | --- | --- |
| 192.168.64.1 | pila1see | 822 | vipien1 |  |
| 192.168.64.2 | pila2sca | 822 | vipien2 |  |
| 192.168.64.3 | bckp1t4v | 822 | backup |  |
| 192.168.64.4 | lemp1see | 22 | LEMP scambi.org |  |
| 192.168.64.5 | stor1see | 22 | storage nextcloud |  |
| 192.168.64.6 | lemp2see | 22 | nginx collabora |  |
| 192.168.64.7 | data1see | 22 | database baserow |  |
| 192.168.64.8 |  | 22 |  |  |
| 192.168.64.9 | lemp4het | 22 | LEMP pan (mastodon) |  |

\* 822 - ssh internet | 22 - ssh LAN  

Supplier list  
- see - seeweb
- t4v - time4vps
- sca - scaleway
- het - hetzner
- con - contabo
