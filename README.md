# TP 2 : GRE over IPsec sous GNS3

Travail pratique de réseau (RSI3) : tunnel GRE entre deux sites, puis chiffrement de ce tunnel par IPsec, avec NAT sur les routeurs d'extrémité. Réalisé sous **GNS3** avec des routeurs Cisco **c7200**, des PC **VPCS** et **Wireshark**.

GRE transporte le multicast (utile pour un protocole de routage dynamique), ce qu'IPsec seul ne sait pas faire. On encapsule donc d'abord dans GRE, puis on chiffre le GRE avec IPsec.

Les commandes complètes sont dans [`commandes.md`](commandes.md).

## Contenu du dépôt

```
.
├── README.md
├── commandes.md
├── .gitignore
├── projet/       # fichier .gns3 ou .gns3project
├── captures/     # fichiers .pcap Wireshark
└── images/       # captures d'écran du rapport
```

## Prérequis

- GNS3 (client Windows) et la VM GNS3 sous VirtualBox
- Image IOS Cisco **c7200** avec support de la commande `crypto` (non fournie dans ce dépôt)
- Wireshark

## Table d'adressage

| Équipement | Interface | Adresse IP | Rôle |
|---|---|---|---|
| RouterA | FastEthernet0/0 | 10.0.10.1/24 | LAN site 1 (NAT inside) |
| RouterA | Serial1/0 | 201.20.20.1/30 | vers WAN (NAT outside) |
| RouterA | Tunnel0 | 172.16.1.1/24 | tunnel GRE |
| WAN | Serial1/0 | 201.20.20.2/30 | vers RouterA |
| WAN | Serial1/1 | 204.20.20.1/30 | vers RouterB |
| RouterB | FastEthernet0/0 | 10.0.20.1/24 | LAN site 2 (NAT inside) |
| RouterB | Serial1/0 | 204.20.20.2/30 | vers WAN (NAT outside) |
| RouterB | Tunnel0 | 172.16.1.2/24 | tunnel GRE |

Le routeur WAN n'a volontairement aucune route vers les réseaux `10.0.x.x` : il simule Internet, où les adresses privées ne sont pas routées. Les noms d'interfaces série sont à adapter à ta topologie.

## Déroulement

1. **Configuration de base** : adressage, NAT (`ACL 100` qui exclut le trafic inter-sites de la traduction), routes par défaut.
2. **Tunnel GRE** (MTU 1400). `ping 172.16.1.2` réussit. `ping 10.0.20.1` échoue : aucune route n'envoie ce trafic dans le tunnel, il part vers le WAN qui ne connaît pas ce réseau privé.
3. **Routes statiques** vers le LAN distant via l'adresse du tunnel : le ping réussit.
4. **Capture avant IPsec** : IP externe 201.20.20.1 → 204.20.20.2 (protocole 47, GRE), en-tête GRE, IP interne 172.16.1.1 → 10.0.20.1, ICMP. Tout est lisible en clair.
5. **IPsec** : ISAKMP pre-share, transform-set `strong` (`esp-3des esp-md5-hmac`), `ACL 101` (`permit gre` entre les deux adresses publiques), crypto map `S2SVPN` sur l'interface de sortie.
6. **Capture après IPsec** : `isakmp` puis `esp` (protocole 50). Le filtre `gre` ne montre plus rien car le GRE est encapsulé dans l'ESP.

## Encapsulation

Avant IPsec :

| Couche | Contenu |
|---|---|
| Liaison | trame du lien série (Cisco HDLC) |
| IP externe | 201.20.20.1 → 204.20.20.2, protocole 47 (GRE) |
| GRE | type de protocole IPv4 |
| IP interne | 172.16.1.1 → 10.0.20.1, protocole 1 (ICMP) |
| ICMP | Echo request |

Après IPsec (mode tunnel par défaut) :

| Couche | Contenu |
|---|---|
| Nouvel IP externe | 201.20.20.1 → 204.20.20.2, protocole 50 (ESP) |
| ESP | SPI et numéro de séquence |
| Chiffré (3DES) | IP externe d'origine + GRE + IP interne + ICMP |
| Trailer + ICV | padding et authentification HMAC-MD5 |

## Remarques de configuration

- L'énoncé utilise `crypto isakmp key 6 ...` (clé de type 6, qui exige une master key). La clé est saisie en clair ici, plus simple en TP.
- La stratégie `isakmp policy 1` ne définit que `pre-share` : les autres paramètres sont les valeurs par défaut de l'IOS (DES, SHA-1, DH groupe 1). C'est acceptable pour un TP, mais DES est faible.

## Ouvrir le projet

1. Démarre la VM GNS3 sous VirtualBox, puis GNS3 sous Windows.
2. *File → Open project* (ou *Import portable project* pour un `.gns3project`).
3. Associe tes propres images c7200 aux routeurs.
4. Démarre les nœuds et colle les commandes de `commandes.md`.

## Sécurité du dépôt

- La clé pré-partagée `P@ssword` est une clé de **laboratoire**. Ne la réutilise jamais en production.
- Les images IOS Cisco ne sont pas publiées (licence).
