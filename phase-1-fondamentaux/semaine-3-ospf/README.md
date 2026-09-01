# Semaine 3 — Routage dynamique avec OSPF

## Contexte

Ajout d'un deuxième routeur à la topologie existante, relié au premier par 
un lien point-à-point dédié (`10.10.10.0/30`). Mise en place d'OSPF pour que 
les deux routeurs s'annoncent automatiquement leurs réseaux respectifs, sans 
configuration manuelle de routes statiques.

## Objectif

Comprendre le fonctionnement d'un protocole de routage dynamique : 
établissement de voisinage, calcul de coût, annonce automatique des réseaux, 
et le rôle du découpage en aires — puis le mettre en pratique sur une 
topologie à deux routeurs.

## Plan d'adressage du nouveau lien inter-routeurs

| Lien | Réseau | Masque | Wildcard | IP Routeur 1 | IP Routeur 2 |
|---|---|---|---|---|---|
| Routeur1 ↔ Routeur2 | 10.10.10.0 | /30 | 0.0.0.3 | 10.10.10.1 | 10.10.10.2 |

## Configuration appliquée

**Sur le Routeur 1** (celui portant les 4 VLANs de la Semaine 1) :

router ospf 1
router-id 1.1.1.1
network 10.10.0.0 0.0.0.63 area 0
network 10.10.0.64 0.0.0.31 area 0
network 10.10.0.96 0.0.0.15 area 0
network 10.10.0.112 0.0.0.7 area 0
network 10.10.10.0 0.0.0.3 area 0

interface GigabitEthernet0/1
ip ospf network point-to-point


**Sur le Routeur 2 :**

router ospf 1
router-id 2.2.2.2
network 10.10.10.0 0.0.0.3 area 0

interface GigabitEthernet0/0
ip ospf network point-to-point


## Preuve — voisinage OSPF établi

Router#show ip ospf neighbor

Neighbor ID Pri State Dead Time Address Interface
2.2.2.2 0 FULL/- 00:00:31 10.10.10.1 GigabitEthernet0/1

Router#show ip ospf neighbor

Neighbor ID Pri State Dead Time Address Interface
1.1.1.1 0 FULL/- 00:00:35 10.10.10.2 GigabitEthernet0/0


État `FULL` symétrique confirmé des deux côtés — synchronisation complète 
des bases de données de routage.

## Preuve — routes apprises dynamiquement

Router#show ip route ospf
10.0.0.0/8 is variably subnetted, 6 subnets, 6 masks
O 10.10.0.0 [110/2] via 10.10.10.2, 00:00:54, GigabitEthernet0/0
O 10.10.0.64 [110/2] via 10.10.10.2, 00:00:54, GigabitEthernet0/0
O 10.10.0.96 [110/2] via 10.10.10.2, 00:00:54, GigabitEthernet0/0
O 10.10.0.112 [110/2] via 10.10.10.2, 00:00:54, GigabitEthernet0/0


Le Routeur 2 a appris automatiquement les 4 réseaux VLAN du Routeur 1, sans 
qu'aucune route statique n'ait été configurée pour ce lien.

## Incident rencontré et résolu

**Symptôme :** le voisinage restait bloqué à l'état `2WAY/DROTHER` sur les 
deux routeurs, au lieu de passer à `FULL`. Une seule des deux tables de 
routage se synchronisait (`show ip route ospf` vide sur un routeur, complet 
sur l'autre) — signe d'une synchronisation asymétrique.

**Diagnostic :** vérification méthodique dans l'ordre : configuration IP des 
interfaces (correcte), présence de la commande `network` sur les deux 
routeurs (correcte, confirmée par `show run | section router ospf`). La 
configuration en elle-même n'était pas en cause.

**Cause identifiée :** Packet Tracer traite par défaut une interface 
Ethernet comme un segment de type **broadcast**, ce qui déclenche un 
processus d'élection de DR (Designated Router) et BDR (Backup Designated 
Router) — même sur un lien direct à seulement 2 routeurs. Cette élection 
peut rester bloquée ou incomplète, empêchant la transition de `2WAY` 
vers `FULL`.

**Correction appliquée :**

interface <interface-du-lien>
ip ospf network point-to-point

Sur un lien de type point-à-point, aucune élection DR/BDR n'est nécessaire 
(elle n'a de sens que sur un segment partagé par plusieurs routeurs) — OSPF 
passe alors directement à l'état `FULL` dès que les deux extrémités 
échangent leurs bases de données complètes.

**Enseignement** : un état `2WAY` persistant sur un lien qui ne devrait 
relier que deux routeurs est un signal fort à vérifier le type de réseau 
OSPF configuré sur l'interface, avant de chercher une erreur dans les 
commandes `network` ou l'adressage IP.

## Autre erreur corrigée en cours de labo

Une première tentative de configuration des `network` sur le Routeur 1 
utilisait la même adresse réseau (`10.10.0.0`) répétée avec des wildcards 
différents pour les 4 VLANs, au lieu de l'adresse réseau propre à chaque 
VLAN. Erreur corrigée en reprenant le plan d'adressage réel de la Semaine 1 
(Ventes `10.10.0.0`, Comptabilité `10.10.0.64`, Direction `10.10.0.96`, 
Management `10.10.0.112`), chacun avec son wildcard mask calculé 
individuellement.

## Concepts clés retenus

- **Wildcard mask** : inverse binaire du masque de sous-réseau (`255 - 
chaque octet`), utilisé par la commande `network` en OSPF.
- **Coût OSPF** : `100 000 000 / bande passante en bps` — détermine le 
chemin préféré en cas de routes multiples.
- **Rôle de l'Area 0 (backbone)** : toute area non-backbone doit obligatoirement 
transiter par l'Area 0 pour communiquer avec une autre area — évite les 
boucles de résumé de routes et limite la propagation des recalculs SPF à 
l'ensemble du domaine.
