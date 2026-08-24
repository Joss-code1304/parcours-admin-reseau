# Semaine 2 — Redondance et résilience réseau (STP & EtherChannel)

## Contexte

Mise en place d'une redondance physique entre switches (deux câbles reliant 
la même paire), puis stabilisation de cette redondance pour éviter les 
boucles de niveau 2 tout en exploitant pleinement la bande passante disponible.

## Objectif

Comprendre pourquoi une redondance non maîtrisée casse un réseau (boucle L2, 
broadcast storm), comment STP la prévient par défaut, puis comment 
EtherChannel dépasse la limite de STP (un lien bloqué = bande passante perdue) 
en agrégeant plusieurs liens physiques en un seul lien logique actif.

## Partie A — STP : élection du root bridge

**Topologie** : 3 switches en triangle, avec un lien redondant (2 câbles) 
entre deux d'entre eux.

**Constat initial** : par défaut, sans configuration, STP a élu comme root 
bridge le switch ayant l'adresse MAC la plus basse — un critère purement 
arbitraire, sans rapport avec la position stratégique du switch dans le 
réseau.

**Correction appliquée** :

spanning-tree vlan 1 root primary

Cette commande abaisse automatiquement la priorité du switch choisi 
(de 32769 à 24577 dans ce cas) pour garantir son élection, permettant de 
choisir le root bridge selon un critère métier (position centrale, 
puissance matérielle, stabilité) plutôt que de laisser le hasard décider.

**Vérification** : `show spanning-tree` confirme le nouveau root bridge, 
avec ses 3 ports en état `Desg FWD`, et les ports bloqués (`Altn BLK`) 
réorganisés sur les switches voisins en conséquence.

## Partie B — EtherChannel : exploiter la redondance au lieu de la bloquer

**Configuration appliquée**, sur les deux ports du lien redondant, des 
deux côtés du lien :

interface range fa0/1 - 2
channel-group 1 mode active


**Vérification** : `show etherchannel summary` confirme les deux ports 
en état `(P)` — regroupés dans le port-channel `Po1`, traités par STP 
comme un lien unique (donc plus aucun port en blocking sur ce lien).

## Preuve — test de résilience

Ping continu lancé entre deux PC situés de part et d'autre du lien 
EtherChannel. Un des deux câbles physiques a été débranché en pleine 
simulation : le trafic a continué à passer sans interruption, la charge 
basculant automatiquement sur le câble restant.

## Incidents rencontrés et résolus

**Incident 1 — Plage d'interfaces trop large**

*Symptôme* : `show etherchannel summary` affichait 3 ports au lieu de 2, 
dont un en état `(I)` stand-alone au lieu de `(P)`.

*Cause* : la commande `interface range fa0/1 - 3` incluait un troisième 
port connecté à un switch différent de celui du lien redondant. 
EtherChannel ne peut regrouper que des liens allant vers la **même** 
paire d'équipements — LACP a donc refusé d'agréger ce port incohérent, 
le laissant isolé plutôt que de créer un lien logique invalide.

*Correction* : identification précise des deux ports allant réellement 
vers le même switch voisin, application de `channel-group` uniquement 
sur cette plage restreinte (`fa0/1 - 2`).

**Incident 2 — Câblage physique modifié après un rebranchement**

*Symptôme* : même signature qu'à l'Incident 1 (un port en `(I)`), 
alors que la configuration côté commandes était correcte.

*Cause* : après un débranchement/rebranchement des câbles pour nettoyer 
la topologie, un des deux ports ne pointait plus vers le même switch 
voisin que l'autre.

*Correction* : vérification visuelle de la topologie logique pour 
confirmer la destination réelle de chaque câble avant de réappliquer la 
configuration.

**Enseignement commun aux deux incidents** : une sortie `show etherchannel 
summary` affichant un port en `(I)` n'indique jamais une erreur de syntaxe 
de commande — elle indique systématiquement une incohérence entre les 
deux extrémités du lien (mauvais port, mauvaise destination, ou 
configuration manquante côté voisin). Le réflexe de diagnostic correct 
est de vérifier la cohérence physique et la configuration des deux côtés 
du lien, pas de re-taper la même commande côté source.

## Défi complémentaire — changement de root bridge stratégique

Exercice réalisé : forcer manuellement le root bridge sur le switch 
jugé le plus stratégique de la topologie (plutôt que de laisser 
l'élection par défaut sur l'adresse MAC la plus basse), avec 
vérification de la réorganisation des ports forwarding/blocking en 
conséquence.
