## Plan d'adressage — Contexte

Conception d'un plan d'adressage IPv4 en VLSM pour une entreprise de 91 postes 
répartis sur 4 services, à partir du bloc `10.10.0.0/22`, avec pour contrainte 
un gaspillage d'adresses minimal.

## Plan d'adressage

| Service | Hôtes requis | Réseau | Masque | Plage utile | Broadcast |
|---|---|---|---|---|---|
| Ventes | 50 | 10.10.0.0 | /26 | .1–.62 | .63 |
| Comptabilité | 25 | 10.10.0.64 | /27 | .65–.94 | .95 |
| Direction | 10 | 10.10.0.96 | /28 | .97–.110 | .111 |
| Management | 6 | 10.10.0.112 | /29 | .113–.118 | .119 |

## Architecture

- Routeur en configuration **router-on-a-stick** : une interface physique 
  segmentée en 4 sous-interfaces (une par VLAN), chacune encapsulée en 
  802.1Q avec son propre VLAN ID.
- 3 switches reliés en cascade, avec des liens **trunk** transportant 
  l'ensemble des VLANs entre eux et vers le routeur.
- Attribution d'adresses en **DHCP**, avec un pool distinct par VLAN 
  configuré directement sur le routeur.

## Preuve — capture Wireshark (mode Simulation)

Capture d'un paquet ICMP Echo Reply transitant entre le VLAN Management 
(10.10.0.115) et le VLAN Ventes (10.10.0.7). Les deux adresses appartenant 
à des sous-réseaux différents, ce trafic a nécessairement transité par le 
routeur pour être routé entre VLANs. L'adresse MAC de destination change 
à chaque saut du chemin, alors que les adresses IP source et destination 
restent identiques de bout en bout — preuve que le routage se fait au 
niveau 2 saut par saut, indépendamment de l'adressage IP global.

## Incident rencontré et résolu

**Symptôme :** aucune communication possible entre PC de VLANs différents, 
y compris les requêtes ARP vers la passerelle par défaut, sur certains 
switches uniquement.

**Diagnostic :** isolement méthodique du problème couche par couche 
(test ping direct au routeur, vérification de la table de routage, 
vérification des liens trunk). La configuration du routeur et des liens 
trunk s'est révélée correcte.

**Cause identifiée :** les VLANs 20, 30 et 40 n'avaient été créés que sur 
les switches directement rattachés à ces services. Sur un switch de 
transit intermédiaire, ces VLANs n'existaient pas localement — bien que 
le trunk les autorise en théorie, un switch ne peut activer un VLAN qu'il 
ne connaît pas.

**Correction :** création explicite de l'ensemble des VLANs (10, 20, 30, 40) 
sur les 3 switches de la topologie, garantissant que chaque switch de 
transit puisse relayer correctement tous les VLANs, indépendamment des 
PC qui lui sont directement rattachés.

**Enseignement :** un lien trunk correctement configuré ne suffit pas à 
faire transiter un VLAN — chaque switch impliqué dans le chemin doit 
également le connaître dans sa base VLAN locale.
