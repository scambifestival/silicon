## Introduzione

I server hanno IP pubblico su cui forniranno i servizi.

Due di questi server (di provider differenti per evitare single point of failure) fanno anche da entrypoint per tutta l’infrastruttura, collegata insieme tramite vpn TINC.<br/>
Entrambi i server hanno anche un server openvpn (con record DNS vipien1 e vipien2).

Per evitare sovrapposizione, i due server vpn gestiscono due reti differenti:<br/>
vpn1 - 192.168.66.0/24<br/>
vpn2 - 192.168.68.0/24

Con TINC vengono aggiunte le rotte per queste due reti con il gateway corretto.

La naming convention per gli hostname è la seguente:
- 4 lettere : funzione
- 1 numero : incrementale
- 3 lettere : sigla fornitore

Di seguito l’elenco dei server ordinati per IP "interni" (TINC)

| IP TINC | hostname | ssh * | ruolo | IP pubblico |
| --- | --- | --- | --- | --- |
| 192.168.64.1 | pila1see | 822 | vipien1 | ***REMOVED*** |
| 192.168.64.2 | pila2sca | 822 | vipien2 | ***REMOVED*** |
| 192.168.64.3 | bckp1t4v | 822 | backup |  |
| 192.168.64.4 | lemp1see | 22 | LEMP scambi.org | ***REMOVED*** |
| 192.168.64.5 | stor1see | 22 | storage nextcloud |  |
| 192.168.64.6 | lemp2see | 22 | LEMP collabora |  |

\* 822 - ssh internet | 22 - ssh LAN  

Elenco fornitori  
- see - seeweb
- t4v - time4vps
- sca - scaleway
- het - hetzner
- con - contabo
