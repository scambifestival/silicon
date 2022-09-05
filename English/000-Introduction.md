## Introduction

All of the servers have a public IP, where they will provide the services.

Two of them (hosted by two different providers, in order to avoid a single point of failure) act as entrypoints to the whole infrastructure, connected within itself by TINC VPN.  
Both servers have an OpenVPN server (with DNS records `vpien1` and `vpien2`), too.

To avoid overlapping, the two VPN servers handle two different networks:
- `vpn1`: `192.168.66.0/24`
- `vpn2`: `192.168.68.0/24`

Thanks to TINC, routes for these two networks are added with the correct gateway.

## Servers

Below the list of the servers, ordered according to “internal” (TINC) IPs.

| TINC IP      | hostname | ssh <sup>*</sup> | role              | Public IP       |
| ---          | ---      | ---              | ---               | ---             |
| 192.168.64.1 | pila1see | 822              | vipien1           |   |
| 192.168.64.2 | pila2sca | 822              | vipien2           |   |
| 192.168.64.3 | bckp1t4v | 822              | backup            |   |
| 192.168.64.4 | lemp1see | 22               | LEMP scambi.org   |   |
| 192.168.64.5 | stor1see | 22               | storage nextcloud |   |
| 192.168.64.6 | lemp2see | 22               | LEMP collabora    |   |

\* 822 - ssh internet | 22 - ssh LAN

The naming convention for the hostnames is the following:

- 4 letters: function
- 1 number: incremental
- 3 letters: hosting provider identifier

### Hosting providers

- `see` - [Seeweb](https://seeweb.it 'Seeweb official website')
- `t4v` - [Time4VPS](https://time4vps.com 'Time4VPS official website')
- `sca` - [Scaleway](https://scaleway.com 'Scaleway official website')
- `het` - [Hetzner](https://hetzner.com 'Hetzner official website')
- `con` - [Contabo](https://contabo.com 'Contabo official website')
