---
title: "Pare-feu OPNsense : concepts et architecture pour segmenter un homelab"
author: "0x41647269656E"
series: "Hardening"
tags:
  - opnsense
  - firewall
  - reseau
  - segmentation
  - securite
  - zero-trust
  - hardening
reading-time: 20m
date: 05-07-2026
last_modified: 05-07-2026
status: draft
---

# Pare-feu OPNsense : concepts et architecture pour segmenter un homelab

> [!abstract] TL;DR
> Un homelab typique est un réseau **à plat** : toutes les VM, tous les conteneurs et toutes les machines de la maison se voient les uns les autres. La box FAI filtre ce qui entre depuis Internet, mais **rien ne filtre le trafic latéral** — une seule VM compromise voit tout. Cet article pose les concepts (pare-feu à états, inspection applicative, IDS/IPS) et conçoit une architecture cible : une VM **OPNsense** sur Proxmox qui segmente l'intérieur du homelab en zones (services internes, DMZ), pendant que la box FAI reste le routeur de la maison. La mise en œuvre pas-à-pas fait l'objet de l'article suivant : [[Installer-OPNsense-sur-Proxmox]].

> [!info] Note de cohérence
> Cet article est le premier d'un diptyque : ici le *pourquoi* et le *quoi*, dans [[Installer-OPNsense-sur-Proxmox]] le *comment*. Il s'appuie sur l'infrastructure décrite dans [[Proxmox-installation-securite]] (Proxmox VE sur Pandaria, VM Ubuntu + Podman rootless) et complète la démarche de défense en profondeur entamée avec [[Self-hosting-securise-avec-Podman]], [[Reverse-proxy-Caddy-avec-TLS-automatique]] et [[Configuration-WireGuard-self-hosted]].

---

## 1. Le problème : un homelab à plat

### L'état des lieux

Dans la configuration décrite jusqu'ici, Pandaria héberge Proxmox, et Proxmox héberge une VM Ubuntu qui fait tourner les services sous Podman. Côté réseau, tout ce petit monde est branché sur le même bridge :

```
Internet
   │
[Box FAI] 192.168.1.1 ──────────── LAN maison 192.168.1.0/24
   │
   ├── PC perso, téléphones, TV, objets connectés...
   │
   └── [Pandaria / Proxmox] ── vmbr0
          ├── interface d'admin Proxmox (8006)
          └── VM Ubuntu (Podman : Caddy, Immich, Jellyfin, ...)
```

Le bridge `vmbr0` est ponté sur la carte réseau physique : **toutes les VM sont des machines du LAN maison comme les autres**. La VM Ubuntu voit la TV connectée, le PC perso voit l'interface d'admin Proxmox, et une future VM de test verra tout ce beau monde aussi.

### Ce que la box FAI protège — et ce qu'elle ne protège pas

La box FAI fait du NAT et refuse par défaut les connexions entrantes depuis Internet. C'est une vraie protection, mais elle ne s'applique qu'à **un seul axe** : la frontière Internet ↔ maison.

> [!warning] Le trafic latéral n'est filtré par personne
> À l'intérieur du LAN, la box est un commutateur : elle transmet, elle ne filtre pas. Si un conteneur est compromis (image vérolée, CVE sur un service exposé, dépendance malveillante), l'attaquant peut depuis ce point :
> - scanner tout le `192.168.1.0/24` ;
> - attaquer l'interface d'admin Proxmox sur le port 8006 ;
> - attaquer les autres services (bases de données, partages) ;
> - viser les machines personnelles et les objets connectés — souvent les moins à jour du réseau.
>
> C'est le **mouvement latéral**, l'étape 2 de à peu près toutes les intrusions documentées. Un réseau à plat le rend gratuit.

Les couches déjà posées (Podman rootless, réseaux internes de conteneurs, Caddy en point d'entrée unique) réduisent la surface *dans* la VM Ubuntu. Mais entre les VM, entre les VM et l'hôte, entre le homelab et la maison : rien.

### Ce qu'on veut obtenir

L'objectif est de découper le réseau en **zones** dont les échanges sont explicitement autorisés ou refusés :

- une zone pour les services internes (accessibles depuis la maison ou le VPN, jamais depuis Internet) ;
- une zone d'exposition (DMZ) pour ce qui doit être joignable depuis Internet ;
- le LAN maison, qui reste sous la responsabilité de la box.

Et entre ces zones, un poste de contrôle : le pare-feu.

---

## 2. Ce qu'est un pare-feu, concrètement

Le mot « pare-feu » recouvre des choses très différentes selon la couche réseau où l'on travaille. Poser le vocabulaire évite de se raconter des histoires sur ce que l'outil protège réellement.

### Le pare-feu à états (stateful, couches 3-4)

C'est le cœur du métier, et c'est ce que fait OPNsense de base. Il examine chaque paquet et décide de le laisser passer ou non selon :

| Critère | Exemple |
|---|---|
| Adresse IP source / destination | `10.0.20.5` → `10.0.10.10` |
| Protocole | TCP, UDP, ICMP |
| Port source / destination | destination `443/tcp` |
| **État de connexion** | nouveau flux, ou réponse à un flux déjà autorisé |

Le mot important est **à états** (*stateful*) : le pare-feu tient une table des connexions en cours. Quand une règle autorise `A → B : 443`, les paquets de réponse `B → A` passent automatiquement parce qu'ils appartiennent à un état connu — pas besoin d'écrire la règle retour. C'est ce qui rend les politiques « tout est refusé sauf ce qui est explicitement autorisé » praticables.

> [!note] Ce qu'un pare-feu à états ne voit pas
> Il raisonne en IP et en ports. Il ne sait pas si le flux `443/tcp` autorisé transporte du HTTPS légitime, une exfiltration de données ou un canal de commande et contrôle. « Le port est ouvert » est le début de l'analyse, pas la fin.

### La couche applicative (couche 7) : IDS, IPS et inspection

Au-dessus du filtrage IP/port, une famille d'outils inspecte le **contenu** du trafic :

- **IDS** (*Intrusion Detection System*) : observe le trafic, le compare à une base de signatures d'attaques connues (exploits, malwares, scans) et **alerte**. Passif : il ne bloque rien.
- **IPS** (*Intrusion Prevention System*) : même moteur, mais placé en coupure — il peut **jeter** les paquets correspondant à une signature. Actif, donc aussi capable de bloquer du trafic légitime sur un faux positif.
- **Filtrage applicatif / DPI** : identification des applications indépendamment du port (reconnaître du BitTorrent sur le port 443), filtrage par catégories. C'est le créneau de produits comme Zenarmor.

Dans OPNsense, la brique IDS/IPS est **Suricata**, un moteur open source de référence, intégré nativement et alimenté par des jeux de règles communautaires (ET Open).

> [!tip] Démystifier le « pare-feu applicatif »
> Le terme est souvent employé comme argument marketing. En pratique, OPNsense est : un excellent **pare-feu à états** (via `pf`, le filtre de paquets de la famille BSD) **+** un **IDS/IPS** (Suricata) qu'on active si on le souhaite. La détection applicative a un vrai coût (CPU, RAM, faux positifs, entretien des règles) et un vrai bénéfice (visibilité sur ce qui circule) — les deux sont traités honnêtement dans [[Installer-OPNsense-sur-Proxmox]]. À ne pas confondre non plus avec un **WAF** (*Web Application Firewall*), qui protège une application web spécifique contre les injections SQL, XSS, etc. — c'est un autre outil, à un autre étage.

### Là où le pare-feu s'insère : entre les zones

Un pare-feu ne protège que le trafic qui le **traverse**. Toute l'architecture de la section 4 découle de cette évidence : il faut que les zones soient des réseaux distincts, reliés uniquement par le pare-feu, sinon il ne voit rien.

---

## 3. Pourquoi OPNsense, pourquoi en VM

### OPNsense en deux mots

[OPNsense](https://opnsense.org) est une distribution pare-feu/routeur open source (licence BSD 2 clauses), basée sur **FreeBSD** et sur le filtre de paquets **`pf`** — un lignage qui remonte à OpenBSD et qui équipe des pare-feu de production depuis plus de vingt ans. C'est un fork de pfSense créé en 2015, maintenu par la société néerlandaise Deciso, avec des sorties semestrielles et des correctifs réguliers.

Concrètement, OPNsense fournit dans une interface web unique : le pare-feu à états, le NAT, un serveur DHCP, un résolveur DNS (Unbound), Suricata, des VPN (WireGuard, OpenVPN, IPsec), et un système de plugins.

### Les alternatives, rapidement

| Option | Points forts | Points faibles | Verdict pour ce homelab |
|---|---|---|---|
| **OPNsense** | Interface web complète, Suricata intégré, releases fréquentes, documentation vivante, 100 % open source | Consomme une VM (CPU/RAM), FreeBSD moins familier que Linux | **Retenu** |
| **pfSense CE** | Même lignage, très répandu, énorme base de connaissances | Éditions communautaire/commerciale au périmètre mouvant, cadence de releases plus lente, gouvernance moins lisible | Solide aussi ; le choix OPNsense est autant une préférence de gouvernance que de technique |
| **nftables sur l'hôte Proxmox** | Zéro VM, zéro RAM consommée, contrôle total | Pas d'interface, pas d'IDS intégré, mélange les rôles hyperviseur/pare-feu, chaque règle se gère à la main | Viable pour un besoin minimal, mais on perd la lisibilité et l'outillage |
| **VLAN + switch manageable seul** | Segmentation propre au niveau 2 | Un switch ne **filtre** pas entre VLAN : il faut de toute façon un routeur-pare-feu pour l'inter-VLAN | Complémentaire, pas concurrent |

### VM sur Proxmox ou boîtier dédié ?

La question classique. Un pare-feu « sérieux » tourne sur sa propre machine ; en homelab, la virtualisation a des arguments solides. Le tableau honnête :

| Critère | VM sur Proxmox | Boîtier dédié |
|---|---|---|
| **Coût** | 0 € — Pandaria est déjà là | 100-300 € (mini-PC 2+ NIC) |
| **Consommation électrique** | 0 W de plus | 5-15 W de plus en continu |
| **Snapshots / retour arrière** | Snapshot Proxmox avant chaque mise à jour : retour arrière en une minute | Sauvegarde de config XML seulement |
| **Point de défaillance** | Pandaria tombe → le pare-feu tombe **avec les services qu'il protège** | Indépendant de l'hyperviseur |
| **Problème de la poule et de l'œuf** | Si la VM ne démarre pas, les zones internes n'ont plus de routeur ; il faut garder un accès à Proxmox qui ne dépende pas d'OPNsense | Aucun |
| **RAM** | 2-4 Go prélevés sur les 16 Go de Pandaria | RAM dédiée |
| **Sécurité de l'isolation** | Repose sur l'isolation de l'hyperviseur (très robuste, mais une couche de plus) | Isolation physique |
| **Réversibilité** | On teste, on jette, on recommence | Achat engageant |

> [!note] Le choix pour un homelab type Pandaria
> **VM sur Proxmox.** L'argument décisif : le rôle retenu ici est celui de **pare-feu interne** (section suivante). Si la VM OPNsense tombe, ce sont les zones internes du homelab qui perdent leur routeur — la maison, elle, continue de fonctionner normalement via la box. Le scénario catastrophe « plus personne n'a Internet parce qu'une VM a raté son boot » n'existe pas dans cette architecture, contrairement au cas où OPNsense serait le routeur de bordure. Les snapshots avant mise à jour et le coût nul font le reste. Le point de vigilance qui demeure : **l'accès d'administration à Proxmox ne doit jamais dépendre d'OPNsense** (Proxmox reste joignable directement sur le LAN maison).

### Le choix qu'on ne fait pas : routeur de bordure

Beaucoup de guides installent OPNsense en **routeur de bordure** : la box FAI passe en mode bridge et toute la maison traverse la VM. C'est l'architecture la plus puissante (contrôle total du trafic Internet, y compris de la maison), mais aussi la plus engageante : chaque maintenance de Proxmox coupe Internet pour tout le foyer, et le débogage se fait sous la pression du « ça marche plus ». Pour un homelab qui héberge aussi la vie numérique de la famille, commencer par un **pare-feu interne** est un choix assumé : périmètre plus petit, risque plus petit, et une migration vers la bordure reste possible plus tard si l'envie prend.

---

## 4. L'architecture cible

### Les zones

Trois zones, chacune avec un rôle et un niveau de confiance distincts :

| Zone | Réseau | Rôle | Niveau de confiance |
|---|---|---|---|
| **LAN maison** | `192.168.1.0/24` (box FAI) | Machines personnelles, IoT, et le côté « WAN » d'OPNsense | Moyen — c'est chez soi, mais l'IoT y traîne |
| **SERVICES** | `10.0.10.0/24` (vmbr1) | VM internes : Ubuntu/Podman, bases de données, outils d'admin | Élevé — rien n'y est exposé à Internet |
| **DMZ** | `10.0.20.0/24` (vmbr2) | Ce qui est joignable depuis Internet (reverse proxy public, à terme) | Faible — on suppose qu'un jour, quelque chose y sera compromis |

### Le schéma

```
Internet
   │
[Box FAI] 192.168.1.1
   │
   │  LAN maison 192.168.1.0/24
   ├── PC perso, téléphones, IoT
   ├── Proxmox (admin) 192.168.1.10 ── vmbr0
   │
   └──(vmbr0)── [VM OPNsense]
                   WAN  vtnet0 : 192.168.1.2      (vmbr0)
                   LAN  vtnet1 : 10.0.10.1 ────── vmbr1 ── zone SERVICES
                   │                                  └── VM Ubuntu/Podman 10.0.10.10
                   OPT1 vtnet2 : 10.0.20.1 ────── vmbr2 ── zone DMZ
                                                      └── Reverse proxy public 10.0.20.10
```

Les bridges `vmbr1` et `vmbr2` sont des **bridges internes** de Proxmox, sans carte physique attachée : leur trafic n'existe que dans l'hyperviseur, et le **seul** chemin pour en sortir passe par la VM OPNsense. C'est exactement la propriété recherchée : le pare-feu voit tout ce qui franchit une frontière de zone.

> [!info] Pourquoi la nomenclature WAN / LAN / OPT1 ?
> C'est celle d'OPNsense : l'interface « WAN » est le côté non fiable (ici, le LAN maison — vu du homelab, c'est l'extérieur), « LAN » est le premier réseau interne, « OPT1 » et suivants sont les zones additionnelles. Perturbant cinq minutes, standard ensuite.

> [!note] Double NAT, et pourquoi c'est acceptable ici
> Les zones internes sont masquées derrière l'IP WAN d'OPNsense (`192.168.1.2`) par du NAT de sortie, comme la maison est masquée derrière l'IP publique de la box. Le double NAT a mauvaise réputation, mais ses inconvénients (traversée NAT pour la VoIP ou le jeu en ligne) concernent des usages qui n'habitent pas dans ces zones. En échange, on évite de dépendre d'une route statique sur la box FAI — fonction que beaucoup de box gèrent mal ou pas du tout. Pour joindre un service interne depuis la maison, on passera par une redirection de port sur OPNsense ou, mieux, par [[Configuration-WireGuard-self-hosted|le VPN WireGuard]].

### La matrice de flux

Le cœur de la politique de sécurité tient dans un tableau : **qui a le droit de parler à qui, sur quoi**. Tout ce qui n'y figure pas est refusé.

| Source ↓ / Destination → | LAN maison | SERVICES | DMZ | Internet | OPNsense (admin) |
|---|---|---|---|---|---|
| **LAN maison** | (hors périmètre — box) | Ports publiés uniquement (via redirection) | HTTPS vers le reverse proxy | via box | HTTPS depuis le poste d'admin **uniquement** |
| **SERVICES** | ❌ refusé | — | ❌ refusé | ✅ HTTP/S, DNS, NTP (mises à jour) | DNS/NTP uniquement |
| **DMZ** | ❌ refusé | Flux déclarés uniquement (ex. reverse proxy → port du backend) | — | ✅ HTTP/S sortant limité | ❌ refusé |

Trois lignes de lecture :

1. **La DMZ est un cul-de-sac.** Ce qui y vit peut répondre aux clients et sortir en HTTP/S (certificats, mises à jour), mais ne peut initier **aucune** connexion vers les zones de confiance — sauf les flux nominativement déclarés, comme le reverse proxy qui joint le port précis de son backend.
2. **SERVICES ne redescend pas vers la maison.** Une VM interne compromise ne peut pas scanner le LAN familial ni toucher l'admin Proxmox.
3. **L'administration est un flux comme les autres** : l'interface web d'OPNsense n'est joignable que depuis une IP d'admin désignée, pas depuis toute la maison, et surtout pas depuis la DMZ.

La traduction de cette matrice en règles OPNsense (alias, règles par interface, journalisation) est le chapitre 6 de [[Installer-OPNsense-sur-Proxmox]].

---

## 5. Ce que disent les référentiels de sécurité

Cette architecture n'invente rien : elle applique à l'échelle d'un homelab des principes que les référentiels formalisent depuis longtemps. Les relier aux décisions concrètes permet de vérifier qu'on fait les choses pour de bonnes raisons — et de repérer ce qui manque encore.

### Défense en profondeur

Le principe (hérité du militaire, popularisé en cybersécurité par la NSA puis repris partout) : **aucune couche de protection n'est supposée infaillible**, donc on les empile pour qu'une défaillance soit rattrapée par la couche suivante. Dans notre empilement : conteneur rootless (une évasion de conteneur n'offre qu'un utilisateur non privilégié) → VM (une compromission de la VM reste dans la VM) → **zone réseau filtrée** (la VM compromise ne joint que ce que la matrice autorise) → alertes Suricata (le mouvement latéral tenté laisse des traces). Le pare-feu n'est pas *la* sécurité : c'est une couche, dont la valeur vient de ce qu'elle ne dépend pas des couches d'en dessous.

### Moindre privilège

Chaque flux de la matrice répond à la question « qui a **besoin** de parler à qui ? » et non « qui pourrait vouloir ? ». La politique par défaut est le refus ; chaque autorisation est explicite, nominative, et documentée. C'est le même principe que Podman rootless (le processus n'a que les droits nécessaires) appliqué au réseau.

### Zero Trust

Le modèle Zero Trust (formalisé notamment par le NIST dans la publication SP 800-207) tient en une phrase : **la position sur le réseau ne confère aucune confiance**. « Être sur le LAN » ne doit jamais suffire à accéder à quoi que ce soit. Notre architecture en applique la partie réseau : la maison n'accède pas librement aux services, la DMZ n'accède à rien, et l'admin est restreinte à un poste identifié. Elle n'en applique **pas** la partie identité (authentification forte à chaque accès applicatif) — c'est le rôle du SSO (Authelia/Authentik, au portefeuille des projets) et déjà partiellement celui de [[Configuration-WireGuard-self-hosted|la segmentation par profil WireGuard]]. Zero Trust est une direction, pas une case à cocher.

### Guide d'hygiène informatique de l'ANSSI

Le [guide d'hygiène de l'ANSSI](https://cyber.gouv.fr/publications/guide-dhygiene-informatique) — 42 mesures pensées pour les organisations, transposables presque telles quelles à un homelab — consacre plusieurs mesures au réseau, dont : **segmenter le réseau et cloisonner les systèmes** selon leur niveau de sensibilité (notre découpage en zones), maîtriser et filtrer les interconnexions (la matrice de flux), et cloisonner strictement ce qui est exposé à Internet (la DMZ en cul-de-sac). L'administration depuis un poste dédié via un flux dédié est aussi une recommandation ANSSI récurrente — c'est notre règle d'admin restreinte.

### CIS Controls

Les [CIS Controls v8](https://www.cisecurity.org/controls) recoupent les mêmes idées, avec un vocabulaire d'inventaire : le contrôle 12 (*Network Infrastructure Management*) demande une architecture réseau **documentée et tenue à jour** — c'est précisément le rôle de la matrice de flux et de cet article ; le contrôle 13 (*Network Monitoring and Defense*) demande de surveiller le trafic aux frontières — c'est Suricata et la journalisation des refus ; le contrôle 4 (configuration sécurisée) couvre le durcissement de l'appliance elle-même (HTTPS d'admin, utilisateur dédié, mises à jour).

> [!success] La grille de lecture à retenir
> Segmenter (zones) → filtrer (matrice, refus par défaut) → surveiller (logs, IDS) → maintenir (mises à jour, sauvegardes de config). Les quatre verbes reviennent dans tous les référentiels ; l'article d'installation suit exactement cet ordre.

---

## 6. Limites du dispositif, et bilan

### Ce que cette architecture ne protège pas

Autant le dire soi-même avant qu'un incident ne le fasse :

- **L'hôte Proxmox lui-même.** Il reste sur le LAN maison, hors zones. Sa protection relève d'autres mesures ([[Proxmox-installation-securite]] : pare-feu local, 2FA, mises à jour). Si l'hyperviseur est compromis, toutes les zones le sont — le pare-feu virtualisé ne peut pas protéger la machine qui le fait tourner.
- **Le trafic intra-zone.** Deux VM sur `vmbr1` se parlent sans traverser OPNsense. La segmentation a la granularité de ses zones ; à l'intérieur d'une zone, ce sont les couches inférieures qui travaillent (réseaux internes Podman, pare-feu local des VM).
- **Le LAN maison.** La box reste seule maîtresse à bord côté famille et IoT. Segmenter la maison elle-même (VLAN IoT, Wi-Fi invité) est un chantier séparé, qui demande du matériel réseau manageable.
- **Les attaques applicatives légitimes en apparence.** Une injection SQL sur un service exposé arrive en HTTPS parfaitement autorisé par la matrice. Suricata peut en détecter une partie ; un WAF et surtout des services à jour font le reste.
- **La disponibilité.** Une seule VM pare-feu, pas de haute disponibilité (CARP). Panne d'OPNsense = zones internes isolées jusqu'à intervention. En homelab, c'est un inconvénient acceptable ; les snapshots et la sauvegarde de config réduisent la durée de l'indisponibilité.

### Bilan avantages / inconvénients

| ✅ Pour | ❌ Contre |
|---|---|
| Le mouvement latéral cesse d'être gratuit : chaque franchissement de zone est filtré et journalisé | 2-4 Go de RAM et 2 vCPU prélevés sur Pandaria |
| La DMZ rend l'exposition Internet **structurellement** contenue | Une brique de plus à maintenir (mises à jour semestrielles, règles à entretenir) |
| Visibilité : logs de refus + alertes Suricata = on voit enfin ce qui circule | Complexité réseau accrue : double NAT, débogage à deux étages |
| Coût nul, réversible, snapshots avant chaque changement | SPOF pour les zones internes (pas de HA) |
| S'aligne sur les référentiels (ANSSI, CIS, Zero Trust) sans sur-ingénierie | Ne protège ni l'hôte Proxmox, ni l'intra-zone, ni la maison |

### La suite

L'architecture est posée ; reste à la construire. Le second article déroule tout, dans l'ordre : préparation des bridges Proxmox, création et installation de la VM, configuration initiale durcie, traduction de la matrice de flux en règles, activation de Suricata en IDS puis en IPS, sauvegardes et tests de validation.

→ **[[Installer-OPNsense-sur-Proxmox]]**
