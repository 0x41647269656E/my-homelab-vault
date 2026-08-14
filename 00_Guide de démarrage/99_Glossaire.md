---
title: "Glossaire"
author: "0x41647269656E"
series: "Guide de démarrage"
tags:
  - homelab
  - glossaire
  - démarrage
reading-time: 10m
difficulty: newbie
date: 04-07-2026
last_modified: 04-07-2026
status: published
---
# Glossaire

Nous parlons dans ce guide de NAS et de Homelabs. Je distingue les deux pour des cas d'usages différents. A mon sens, un homelab est un environnement dans lequel on héberge des services que l'on souhaite mettre à l'épreuve (lab) et qui sert à héberger des services. Un NAS quand a lui sert à héberger des services simples et bas niveau de partage de fichiers, le stockage long-terme de sauvegardes et l'hébergement d'applications légères.

Les définitions ci-dessous couvrent les termes que vous croiserez tout au long de cette série. Elles sont volontairement simplifiées : chaque notion est approfondie dans la note correspondante du guide.

## Concepts généraux

- **Homelab** : environnement d'hébergement personnel dans lequel on expérimente et met à l'épreuve (*lab*) des services que l'on héberge soi-même. C'est autant un terrain d'apprentissage qu'une infrastructure de production domestique.

- **NAS** (*Network Attached Storage*) : machine dédiée au stockage accessible par le réseau. Je la distingue du homelab par son cas d'usage : partage de fichiers, stockage long-terme de sauvegardes et hébergement d'applications légères. Voir [[01_Partie Hardware]].

> [!note]
> Aujourd'hui, les entreprises comme Synology ou QNAP tendent à réduire le gap entre NAS et homelab. Mais les puissances de calcul de ces bestioles sont encore loin des processeurs grand public classiques dédiés à des ordinateurs de bureau ou des serveurs.

- **Auto-hébergement** (*self-hosting*) : le fait d'héberger soi-même ses services et ses données sur sa propre infrastructure, plutôt que de les confier à un prestataire cloud.

- **SaaS** (*Software as a Service*) : logiciel consommé en ligne via un abonnement (Office 365, Photoshop, Canva Pro...). Vous n'installez rien, mais vous ne possédez rien non plus.

- **Seedbox** : serveur distant loué, spécialisé dans le téléchargement et le partage de fichiers, généralement doté d'une très grosse bande passante. Voir [[01_Partie Hardware]].

## Matériel et stockage

- **RAID** (*Redundant Array of Independent Disks*) : technique combinant plusieurs disques en un seul volume logique, pour gagner en performance et/ou tolérer la panne d'un ou plusieurs disques. Attention : **le RAID n'est pas une sauvegarde**. Voir [[02_Partie Stockage]].

- **ZFS / OpenZFS** : système de fichiers "next-gen" intégrant la gestion de volumes, les snapshots et la vérification d'intégrité des données (checksums). Gourmand en RAM mais très robuste. Voir [[02_Partie Stockage]].

- **JBOD** (*Just a Bunch Of Disks*) : disques utilisés individuellement, sans redondance. Simple et flexible, adapté aux données non critiques. Voir [[02_Partie Stockage]].

- **mergerfs** : système de fichiers en espace utilisateur qui agrège plusieurs disques indépendants en un point de montage unique, sans les lier entre eux comme le ferait un RAID. La perte d'un disque ne fait perdre que son contenu. Voir [[02_Partie Stockage]].

- **XFS** : système de fichiers Linux classique, éprouvé et performant sur les gros fichiers. Souvent combiné à mergerfs dans les setups minimalistes. Voir [[02_Partie Stockage]].

- **CMR / SMR** (*Conventional / Shingled Magnetic Recording*) : deux techniques d'écriture des disques durs. Le SMR superpose les pistes pour gagner en densité au prix de performances d'écriture catastrophiques dans un NAS. Toujours privilégier le CMR. Voir [[08_Choisir ses disques durs]].

- **HBA** (*Host Bus Adapter*) : carte contrôleur PCIe exposant les disques directement au système (mode "IT"), sans couche RAID matérielle. Indispensable pour ZFS et préférable aux cartes RAID pour un homelab.

- **SMART** (*Self-Monitoring, Analysis and Reporting Technology*) : système de surveillance intégré aux disques durs et SSD, qui remonte des indicateurs d'usure et de santé permettant d'anticiper une panne.

- **Spindown** : mise en veille des plateaux d'un disque dur inactif pour réduire la consommation électrique et l'usure. Plus difficile à obtenir qu'il n'y paraît sous Linux. Voir [[07_Spindown des disques sous Linux]].

- **ECC** (*Error-Correcting Code*) : type de mémoire RAM capable de détecter et corriger les erreurs de bits. Recommandée (mais pas obligatoire) avec ZFS et pour les données critiques.

- **Onduleur / UPS** (*Uninterruptible Power Supply*) : batterie de secours qui maintient l'alimentation du serveur en cas de coupure de courant, laissant le temps d'un arrêt propre. Voir [[06_Point sur l'electricité]].

## Virtualisation et conteneurs

- **Hyperviseur** : logiciel qui exécute des machines virtuelles sur une machine physique (Proxmox, VMware ESXi...). Voir [[03_Les plateformes]].

- **Machine virtuelle (VM)** : ordinateur complet émulé par logiciel, avec son propre système d'exploitation. Isolation forte, mais consommation de ressources plus élevée qu'un conteneur.

- **Conteneur** : environnement d'exécution isolé qui partage le noyau du système hôte. Plus léger qu'une VM, il permet de compartimenter chaque application avec ses dépendances. Voir [[04_La Securité]].

- **Docker / Podman** : les deux moteurs de conteneurs les plus répandus. Podman, utilisé en mode *rootless* (sans privilèges root), offre une meilleure posture de sécurité pour un homelab.

## Réseau et sécurité

- **VPN** (*Virtual Private Network*) : tunnel chiffré entre deux machines à travers Internet. C'est la méthode privilégiée pour accéder à son homelab à distance sans exposer ses services. Voir [[05_Gestion des accès distants]].

- **WireGuard** : protocole VPN moderne, minimaliste et très performant, intégré au noyau Linux. Voir [[05_Gestion des accès distants]].

- **Reverse proxy** : serveur intermédiaire qui reçoit les requêtes entrantes et les redirige vers le bon service interne. Il centralise le TLS (HTTPS), l'authentification et le filtrage (Caddy, Nginx, Traefik...).

- **Zero Trust / ZTNA** (*Zero Trust Network Access*) : modèle de sécurité où aucune machine ni aucun utilisateur n'est considéré comme de confiance par défaut, même à l'intérieur du réseau. Chaque accès est authentifié et vérifié. Voir [[04_La Securité]] et [[05_Gestion des accès distants]].

- **Hardening** (durcissement) : ensemble des pratiques visant à réduire la surface d'attaque d'un système : installation minimale, désactivation des services inutiles, mises à jour régulières. Voir [[04_La Securité]].

- **Transcodage** : conversion à la volée d'un flux vidéo vers un format ou une résolution supportés par l'appareil de lecture. Très gourmand en CPU sans accélération matérielle (Intel Quick Sync, GPU). Voir [[01_Partie Hardware]].
