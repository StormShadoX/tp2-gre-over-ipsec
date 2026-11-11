# TP 2 : commandes (GRE over IPsec)

Noms d'interfaces du c7200 utilisés : `FastEthernet0/0`, `Serial1/0`, `Serial1/1`. Vérifie les tiens avec `show ip interface brief` et adapte-les, surtout dans `tunnel source`, `ip nat inside source` et `crypto map`.

## 1. Postes (VPCS)

Les adresses des postes ne sont pas dans l'énoncé, `.2` a été choisi.
```
Poste1> ip 10.0.10.2/24 10.0.10.1
Poste1> save

Poste2> ip 10.0.20.2/24 10.0.20.1
Poste2> save
```

## 2. Configuration de base

### RouterA
```
enable
configure terminal
hostname RouterA
interface FastEthernet0/0
 ip address 10.0.10.1 255.255.255.0
 ip nat inside
 no shutdown
 exit
interface Serial1/0
 ip address 201.20.20.1 255.255.255.252
 ip nat outside
 no shutdown
 exit
ip route 0.0.0.0 0.0.0.0 201.20.20.2
access-list 100 deny ip 10.0.10.0 0.0.0.255 10.0.20.0 0.0.0.255
access-list 100 permit ip 10.0.10.0 0.0.0.255 any
ip nat inside source list 100 interface Serial1/0 overload
end
write memory
```

### WAN
```
enable
configure terminal
hostname WAN
interface Serial1/0
 ip address 201.20.20.2 255.255.255.252
 no shutdown
 exit
interface Serial1/1
 ip address 204.20.20.1 255.255.255.252
 no shutdown
 exit
end
write memory
```

### RouterB
```
enable
configure terminal
hostname RouterB
interface FastEthernet0/0
 ip address 10.0.20.1 255.255.255.0
 ip nat inside
 no shutdown
 exit
interface Serial1/0
 ip address 204.20.20.2 255.255.255.252
 ip nat outside
 no shutdown
 exit
ip route 0.0.0.0 0.0.0.0 204.20.20.1
access-list 100 deny ip 10.0.20.0 0.0.0.255 10.0.10.0 0.0.0.255
access-list 100 permit ip 10.0.20.0 0.0.0.255 any
ip nat inside source list 100 interface Serial1/0 overload
end
write memory
```

Le routeur WAN n'a aucune route : il est directement connecté aux deux réseaux `/30` et simule Internet.

## 3. Tunnel GRE

### RouterA
```
configure terminal
interface Tunnel0
 ip address 172.16.1.1 255.255.255.0
 ip mtu 1400
 tunnel source Serial1/0
 tunnel destination 204.20.20.2
 end
write memory
```

### RouterB
```
configure terminal
interface Tunnel0
 ip address 172.16.1.2 255.255.255.0
 ip mtu 1400
 tunnel source Serial1/0
 tunnel destination 201.20.20.1
 end
write memory
```

### Test
```
RouterA# ping 172.16.1.2
RouterA# ping 10.0.20.1
RouterA# show interface tunnel 0
```
- `ping 172.16.1.2` réussit.
- `ping 10.0.20.1` échoue : RouterA n'a que la route par défaut, le paquet part vers le WAN qui ne connaît pas `10.0.20.0/24`.

## 4. Routes via le tunnel

### RouterA
```
configure terminal
ip route 10.0.20.0 255.255.255.0 172.16.1.2
end
```

### RouterB
```
configure terminal
ip route 10.0.10.0 255.255.255.0 172.16.1.1
end
```

```
RouterA# ping 10.0.20.1
RouterA# show ip route
Poste1> ping 10.0.20.2
```

## 5. Capture avant IPsec

Capture Wireshark sur le lien RouterA–WAN, puis :
```
RouterA# ping 10.0.20.1 repeat 1
```
Filtre : `gre`. Dans le détail du paquet : IP externe 201.20.20.1 → 204.20.20.2 (protocole 47), GRE, IP interne 172.16.1.1 → 10.0.20.1, ICMP. Tout est en clair.

## 6. IPsec sur RouterA

```
configure terminal
crypto isakmp enable
crypto isakmp policy 1
 authentication pre-share
 exit
crypto isakmp key P@ssword address 204.20.20.2
crypto ipsec security-association lifetime seconds 4096
crypto ipsec transform-set strong esp-3des esp-md5-hmac
 exit
access-list 101 permit gre host 201.20.20.1 host 204.20.20.2
crypto map S2SVPN 10 ipsec-isakmp
 match address 101
 set peer 204.20.20.2
 set transform-set strong
 exit
interface Serial1/0
 crypto map S2SVPN
 end
write memory
```

## 7. IPsec sur RouterB (ACL et pair inversés)

```
configure terminal
crypto isakmp enable
crypto isakmp policy 1
 authentication pre-share
 exit
crypto isakmp key P@ssword address 201.20.20.1
crypto ipsec security-association lifetime seconds 4096
crypto ipsec transform-set strong esp-3des esp-md5-hmac
 exit
access-list 101 permit gre host 204.20.20.2 host 201.20.20.1
crypto map S2SVPN 10 ipsec-isakmp
 match address 101
 set peer 201.20.20.1
 set transform-set strong
 exit
interface Serial1/0
 crypto map S2SVPN
 end
write memory
```

## 8. Tests après IPsec

```
RouterA# clear crypto isakmp
RouterA# clear crypto sa
RouterB# clear crypto isakmp
RouterB# clear crypto sa
```
Démarre une capture sur le lien RouterA–WAN, puis :
```
RouterA# ping 10.0.20.1
RouterA# show crypto isakmp sa
RouterA# show crypto ipsec sa
RouterA# show crypto map
RouterA# show crypto isakmp policy
RouterA# show crypto engine connections active
```

Résultats attendus :

| Vérification | Attendu |
|---|---|
| `show crypto isakmp sa` | `QM_IDLE` entre 201.20.20.1 et 204.20.20.2 |
| `show crypto ipsec sa` | compteurs encaps/encrypt/decaps/decrypt > 0 |
| Wireshark `isakmp` | Main Mode puis Quick Mode (UDP 500) |
| Wireshark `esp` | paquets 201.20.20.1 ↔ 204.20.20.2, protocole 50 |
| Wireshark `gre` | vide (GRE caché dans l'ESP) |

## Dépannage

| Symptôme | Piste |
|---|---|
| Pas de `QM_IDLE` | clé, adresse du pair ou stratégie ISAKMP différente. `debug crypto isakmp` pendant un ping, puis `undebug all` |
| Compteurs à 0 | le trafic ne correspond pas à l'ACL 101, ou la crypto map n'est pas sur la bonne interface. Le ping doit viser 10.0.20.1 |
| ACL non symétriques | l'ACL 101 de RouterB doit être exactement inversée par rapport à celle de RouterA |
| `Tunnel0` down | `ping 204.20.20.2` depuis RouterA, vérifier la route par défaut |
