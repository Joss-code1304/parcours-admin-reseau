# Semaine 5 — Accès Internet avec NAT/PAT

## Contexte

Mise en place de PAT (NAT overload) pour permettre à l'ensemble des VLANs 
internes (Ventes, Comptabilité, Direction, Management) de partager une 
seule adresse IP publique pour accéder à un réseau externe simulé.

## Objectif

Comprendre les 3 types de NAT (statique, dynamique, PAT), le rôle des 
interfaces `inside`/`outside`, le mécanisme du port comme identifiant de 
traduction, puis configurer un accès Internet fonctionnel pour tout le 
réseau d'entreprise.

## Adressage du lien externe simulé

| Lien | Réseau | Routeur 1 (inside) | Routeur ISP simulé |
|---|---|---|---|
| Vers Internet | 203.0.113.0/30 | 203.0.113.1 | 203.0.113.2 |

*Plage `203.0.113.0/24` utilisée volontairement — réservée par convention 
pour la documentation et les exemples techniques (RFC 5737).*

## Configuration appliquée

**Déclaration des interfaces internes (une par VLAN) :**

interface GigabitEthernet0/0.10
ip nat inside
interface GigabitEthernet0/0.20
ip nat inside
interface GigabitEthernet0/0.30
ip nat inside
interface GigabitEthernet0/0.40
ip nat inside


**Déclaration de l'interface externe :**

interface GigabitEthernet0/2
ip address 203.0.113.1 255.255.255.252
ip nat outside


**Définition du trafic à traduire, et activation de PAT :**

access-list 1 permit 10.10.0.0 0.0.3.255
ip nat inside source list 1 interface GigabitEthernet0/2 overload


## Preuve — table de traduction en fonctionnement

Router#show ip nat translations
Pro Inside global Inside local Outside local Outside global
icmp 203.0.113.1:21 10.10.0.8:21 203.0.113.2:21 203.0.113.2:21
icmp 203.0.113.1:22 10.10.0.8:22 203.0.113.2:22 203.0.113.2:22
icmp 203.0.113.1:23 10.10.0.8:23 203.0.113.2:23 203.0.113.2:23
icmp 203.0.113.1:24 10.10.0.8:24 203.0.113.2:24 203.0.113.2:24


Un PC interne (`10.10.0.8`, adresse privée) a pu joindre le routeur externe 
simulé. Chaque requête ICMP a été traduite avec l'IP publique unique 
`203.0.113.1`, associée à un identifiant distinct par échange — c'est ce 
qui permet à plusieurs communications simultanées de partager la même 
adresse publique sans confusion sur le retour.

## Incident rencontré et résolu

**Symptôme** : la commande `show run | include ip nat` affichait 5 lignes 
`ip nat inside` au lieu des 4 attendues (une par VLAN).

**Diagnostic** : cette commande ne montre que le texte des lignes, sans 
leur contexte d'interface — impossible de savoir directement laquelle était 
en trop ou mal placée avec ce seul affichage.

**Vérification** : `show run | section interface` a permis de voir chaque 
ligne `ip nat inside` dans son contexte complet (à quelle interface elle 
appartient). Cette vue a confirmé que les 4 lignes étaient en réalité 
correctement réparties sur les 4 sous-interfaces VLAN, sans doublon ni 
erreur de placement.

**Enseignement** : une commande de filtrage rapide (`include`) est utile 
pour un premier coup d'œil, mais ne suffit pas à diagnostiquer un problème 
de placement — il faut revenir à une commande qui montre le contexte 
complet (`section`) avant de conclure à une erreur.

## Concepts clés retenus

- **PAT (NAT overload)** : plusieurs IP privées partagent une seule IP 
publique, distinguées par un port ou identifiant unique par connexion.
- **`ip nat inside` / `ip nat outside`** : indispensables pour indiquer au 
routeur le sens de traduction — une interface par côté du réseau, appliquée 
à chaque sous-interface concernée individuellement.
- **Table de traduction** : construite dynamiquement à chaque nouvelle 
connexion sortante, permet au routeur de savoir où renvoyer une réponse 
entrante.
- **Effet de sécurité indirect** : aucune connexion ne peut être initiée 
depuis l'extérieur vers un poste interne, faute de traduction préexistante 
dans la table — sans que ce soit un pare-feu à proprement parler.
