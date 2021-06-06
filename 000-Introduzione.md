## Introduzione

I server hanno IP pubblico su cui forniranno i servizi.

Due di questi server (di provider differenti per evitare single point of failure) fanno anche da entrypoint per tutta l’infrastruttura, collegata insieme tramite vpn TINC.<br/>
Entrambi i server hanno anche un server openvpn (con record DNS vpn1 e vpn2).

Per evitare sovrapposizione, i due server vpn gestiscono due reti differenti:<br/>
vpn1 - 192.168.66.0/24<br/>
vpn2 - 192.168.68.0/24

Con TINC vengono aggiunte le rotte per queste due reti con il gateway corretto.

La naming convention per gli hostname è la seguente:
- 4 lettere : funzione
- 1 numero : incrementale
- 3 lettere : sigla fornitore

Di seguito l’elenco dei server ordinati per IP "interni" (TINC)

| IP TINC | hostname | ssh | ruolo | IP pubblico |
| --- | --- | --- | --- | --- |
| 192.168.64.1 | pila1sca | 822 | vpn1 |  |
| 192.168.64.2 | pila2xyz | 822 | vpn2 |  |
| 192.168.64.3 | bckp1t4v | 822 | backup |  |
| 192.168.64.4 | lemp1sca | 822 | LEMP scambi.org |  |
| 192.168.64.5 | lamp1con | 822 | LAMP nextcloud |  |
