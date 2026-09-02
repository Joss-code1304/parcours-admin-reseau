# Semaine 4 — Contrôle d'accès avec les ACL

## Contexte

Mise en place d'une ACL étendue pour restreindre l'accès d'un poste précis 
du VLAN Management vers le VLAN Direction, tout en laissant tout le reste 
du trafic fonctionner normalement.

## Objectif

Comprendre la différence entre ACL standard et étendue, le rôle du sens 
d'application (`in`/`out`), l'importance de l'ordre des règles (premier 
match = action appliquée), et le mécanisme du refus implicite en fin de 
liste — puis mettre ça en pratique sur une vraie restriction métier.

## Scénario métier

Le poste `10.10.0.116` (VLAN Management) ne doit pas pouvoir accéder au 
réseau Direction (`10.10.0.96/28`), mais doit conserver un accès normal 
à tous les autres services de l'entreprise.

## Configuration appliquée

access-list 100 deny ip host 10.10.0.116 10.10.0.96 0.0.0.15
access-list 100 permit ip any any


Appliquée en entrée sur la sous-interface Management :

interface GigabitEthernet0/0.40
ip access-group 100 in


## Preuve — ACL active sur la bonne interface

Router#show ip interface gig0/0.40 | include access
Outgoing access list is not set
Inbound access list is 100
IP access violation accounting is disabled


## Preuve — comportement réel testé

**Trafic bloqué (Management → Direction) :**

C:>ping 10.10.0.100

Pinging 10.10.0.100 with 32 bytes of data:

Reply from 10.10.0.113: Destination host unreachable.
Reply from 10.10.0.113: Destination host unreachable.
Reply from 10.10.0.113: Destination host unreachable.
Reply from 10.10.0.113: Destination host unreachable.

Ping statistics for 10.10.0.100:
Packets: Sent = 4, Received = 0, Lost = 4 (100% loss)


**Trafic autorisé (Management → Ventes), non affecté :**

C:>ping 10.10.0.7

Pinging 10.10.0.7 with 32 bytes of data:
Reply from 10.10.0.7: bytes=32 time=14ms TTL=127
Reply from 10.10.0.7: bytes=32 time<1ms TTL=127
Reply from 10.10.0.7: bytes=32 time<1ms TTL=127
Reply from 10.10.0.7: bytes=32 time<1ms TTL=127

Ping statistics for 10.10.0.7:
Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)


**Compteurs de correspondance confirmant le fonctionnement réel :**

Router#show access-lists
Extended IP access list 100
10 deny ip host 10.10.0.116 10.10.0.96 0.0.0.15 (12 match(es))
20 permit ip any any (12 match(es))


## Incident rencontré et résolu

**Symptôme** : l'ACL était créée sans erreur de syntaxe, mais appliquée 
sur l'interface physique parente (`GigabitEthernet0/0`, sans adresse IP) 
plutôt que sur la sous-interface réellement utilisée par le VLAN Management 
(`GigabitEthernet0/0.40`).

**Cause** : dans une configuration router-on-a-stick, le trafic de chaque 
VLAN transite par sa propre sous-interface (avec encapsulation 802.1Q), pas 
par l'interface physique brute. Appliquer une ACL sur l'interface parente 
risque un filtrage trop large ou imprévisible, tous VLANs confondus, plutôt 
qu'un filtrage ciblé.

**Correction** : retrait de l'ACL de l'interface physique, puis application 
sur la sous-interface `.40` correspondant réellement au VLAN Management.

**Enseignement** : une ACL syntaxiquement correcte peut produire un 
comportement incorrect si son point d'application est mal choisi — la 
configuration de la règle elle-même ne suffit pas, son emplacement dans 
la topologie doit être vérifié séparément.

## Concepts clés retenus

- **ACL standard** : filtre uniquement sur l'adresse source — à placer 
près de la destination pour éviter un blocage trop large.
- **ACL étendue** : filtre sur source, destination, protocole et port — 
peut être placée près de la source grâce à sa précision.
- **Premier match gagne** : dès qu'une règle correspond, son action 
s'applique et les règles suivantes ne sont plus consultées.
- **Refus implicite** : une règle invisible `deny any any` existe toujours 
en fin de liste — sans un `permit` explicite final, tout le trafic non 
listé est bloqué.
