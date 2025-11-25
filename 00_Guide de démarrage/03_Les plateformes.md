---
title: Les plateformes
author: "0x41647269656E"
series: Guide de démarrage
tags:
  - freenas
  - truenas
  - dsm
  - synology
  - xpenology
  - unraid
  - proxmox
date: 25-11-2025
---
# Les plateformes

Les systèmes comme TrueNAS, Openmediavault ou Unraid sont conçus pour transformer n’importe quel ordinateur en véritable serveur NAS, offrant une interface graphique intuitive où presque tout se fait en quelques clics : création de volumes, partage de fichiers, gestion des utilisateurs, sauvegardes automatiques, supervision du système, provisionnement d'applications, multimédia, etc.

Leur objectif est de rendre accessibles des technologies habituellement réservées aux professionnels, comme ZFS, la réplication ou la déduplication, tout en restant faciles à configurer.

Grâce à ces OS spécialisés, on peut construire un NAS personnalisé, performant et fiable, sans avoir besoin de maîtriser les lignes de commande. Tout est pensé pour offrir l'expérience d’un NAS prêt à l’emploi, mais avec la flexibilité du matériel.

 ---
## Le projet FreeNAS / TrueNAS

### Historique du projet

Les premières versions de FreeNAS sont apparues en 2005. Au cours des années, le logiciel est devenu très populaire, atteignant plus de 10 millions de téléchargements et plus d’un million de déploiements dans le monde. Pendant longtemps, FreeNAS et TrueNAS ont évolué en parallèle chez iXsystems : FreeNAS était la version libre soutenue par la communauté, tandis que TrueNAS était l’édition destinée aux entreprises pour les usages de stockage critiques. Bien qu’ils aient été gérés séparément, les deux partageaient une base de code commune.
#### TrueNAS Core

![[Pasted image 20230219013306.png]]

TrueNAS Core (Ex-FreeNAS) est un système d'exploitation orienté "services" développé sous licence libre BSD. Basé sur l'OS FreeBSD, Il supporte de nombreux protocoles : CIFS, Samba, FTP, NFS, rsync, AFP, iSCSI, les rapport S.M.A.R.T et son "RAID" intégré.

TrueNAS Core tire sa fiabilité de l’une de ses technologies centrales : **le système de fichiers ZFS**.

Pour rappel (voir l'article précédent [[02_Guide de démarrage - Storage]]), ZFS est un système de fichiers avancé conçu pour garantir la fiabilité et l’intégrité des données. ZFS intègre son propre système de RAID, le RAIDZ, qui remplace les RAID matériels et offre plusieurs niveaux de protection. Il propose aussi des outils puissants comme les snapshots, les clones, la compression et la déduplication pour gérer efficacement les données. L’un de ses atouts majeurs est la vérification systématique des blocs grâce aux checksums, permettant de détecter la corruption silencieuse et le bit rot. En cas d’erreur, ZFS peut automatiquement réparer les données via son mécanisme de self-healing. Le scrubbing, effectué régulièrement, analyse l’ensemble du stockage pour corriger les éventuelles anomalies. Grâce à ces fonctionnalités, TrueNAS Core offre une fiabilité exceptionnelle pour les services, les sauvegardes et les environnements critiques.

> [!WARNING]
> Il est fortement conseillé de suivre les guidelines disponibles sur le site officiel et d'utiliser un ensemble de composants compatibles. Ce n'est pas parce que le composant est ancien et qu'il as été utilisé dans des milliers de références de cartes mères qu'il est automatiquement compatible. Voir : [TrueNAS Core Hardware Guide](https://www.truenas.com/docs/core/gettingstarted/corehardwareguide/)

> [!info] NDLR : 
> De mon expérience, j'ai tenté d'intégrer TrueNAS Core en Juillet 2021 avec une carte mère [ASUS PRIME Z590M-PLUS](https://www.asus.com/fr/motherboards-components/motherboards/prime/prime-z590m-plus/techspec/), le NIC (_Network Interface Controller_) [Intel i219-V](https://www.intel.fr/content/www/fr/fr/products/sku/82186/intel-ethernet-connection-i219v/specifications.html) (Datant de 2015) présent sur ma carte mère n'était pas compatible avec TrueNAS Core nativement. Il nécessitait l'installation d'un driver. Petite particularité sur Free-BSD, ajouter un driver n’est pas aussi simple que sous Linux : cela nécessite généralement de recompiler le noyau. Concrètement, il faut récupérer le code source complet du noyau via Git, intégrer ou activer le pilote manquant, puis reconstruire et réinstaller l’ensemble du noyau pour que le matériel soit reconnu.
> 
> A date de cet article (25/11/2025), le contrôleur **Intel i219-V** _n’est toujours pas officiellement supporté_ **TrueNAS CORE**.

#### TrueNAS SCALE

![[Pasted image 20230219014534.png]]

TrueNAS SCALE® est une distribution HCI (Hyperconverged Infrastructure) Open Source basée sur Debian. Celle-ci propose un accès aux Linux Containers (LXC), le support de VMs (KVM), Docker et Kubernetes et d'un système de stockage de fichiers extensible basé sur ZFS et Gluster.

-   Hyperconverged Storage that Scales Up or Out
-   Integrated Linux Containers & VMs
-   Deploy as a Single Node or Cluster
-   Designed for Hybrid Clouds
-   Enterprise Support Options Coming Soon

Là où TrueNAS Core propose de s'interfacer avec des cartes matérielles assurant le stockage, TrueNAS SCALE assure la Software Define Storage

![[Pasted image 20230219014124.png]]


TrueNAS Enterprise propose des services supplémentaires incluant support technique, support avancé de matériel.

![[Pasted image 20230219014025.png]]

#### Problèmes rencontrés
##### File system gourmand en RAM à cause de la dédup
##### Les jails
##### Non prise en charge du driver
##### Compilation des sources du noyaux à chaque mise à jour
##### Les Flashs de firmwares de cartes raids

#### Conclusions

TrueNAS est certainement la meilleure option pour celui qui veux une solution modulable et pilotable à l'aide d'une interface graphique. La richesse de la distribution vient de son support d'OpenZFS (solution avancée de stockage (Snapshots, déduplication, clones, réplication...Etc)) et de la richesse de son magasin d'applications prêtes à l'emploi. Sa communauté est la plus grande dans le monde des distributions orientées NAS grace au soutien du modèle par une entreprise assurant un support des éditions entreprise de TrueNAS.

### Xpenology (DiskStation Manager)

Xpenology est une distribution du logiciel inclus dans les NAS Synology qu'il est possible d'installer sur de nombreuses configurations. Son installation requière, comme un Hackintosh, de faire passer la machine comme un matériel Synology reconnu. Une emulation de composants, la modification de l'adresse MAC de la carte réseau et d'autres sécurités doivent être contournées pour permettre l'installation de cette distribution. Synology utilise un code open-source pour diffuser l'offre de services de l'OS. Certains estiment que le logiciel étant bati par dessus devraient l'etre également (voir [Code Source Synology](http://sourceforge.net/projects/dsgpl/ "Code Source Synology")).

Retour d'expérience : Au delà de la curiosité technique que cela représente, je ne suis pas à l'aise à l'idée de faire tourner une distribution présentant par nature un retard dans la distribution des mises à jours de sécurité du système d'exploitation. La communauté devant faire sauter tous les verrous du constructeur pour proposer la mise à jour. L’installation de cette dernière est parfois très périlleuse. Enfin, on rappellera que les premières victimes du [ransonware Synolocker](https://korben.info/synolocker.html) étaient des utilisateurs XPEnology.

### Unraid
Unraid est une solution idéale pour transformer un ordinateur personnel ou un petit serveur en une véritable plateforme d’hébergement domestique, simple à administrer et très flexible. Basé sur un noyau Linux, il utilise un système de stockage hybride reposant sur XFS, Btrfs et un mécanisme de parité logiciel, permettant de combiner des disques de tailles et de types variés sans contrainte de RAID classique. Cette approche unique assure à la fois la protection des données et une extension de capacité aisée, tout en conservant un accès direct à chaque disque en cas de panne. Unraid gère nativement Docker et KVM, ce qui permet d’exécuter simultanément des conteneurs et des machines virtuelles isolées, le tout depuis une interface web fluide et claire. Il est compatible avec la majorité des architectures x86_64 et reconnaît sans difficulté la plupart des contrôleurs SATA, NVMe ou USB, ce qui en fait une solution particulièrement adaptée aux serveurs réutilisant du matériel grand public. Pour un homelab, c’est un excellent compromis entre puissance, flexibilité et simplicité : on peut y héberger Plex, Jellyfin, Nextcloud ou encore des services d’automatisation sans avoir besoin d’une expertise Linux approfondie.

### Proxmox

Proxmox adopte une approche plus orientée “infrastructure” et se prête particulièrement bien à un homelab évolutif et technique. Reposant sur Debian, il intègre nativement KVM pour la virtualisation complète et LXC pour les conteneurs légers, offrant un environnement capable d’exécuter aussi bien des systèmes d’exploitation complets que des services isolés à faible empreinte. Son moteur de stockage s’appuie sur ZFS, LVM, Ceph et NFS, permettant de créer des volumes redondants, de gérer le thin provisioning et de bénéficier de snapshots rapides et fiables. Proxmox exploite pleinement le matériel moderne, supportant les processeurs Intel et AMD 64 bits, la virtualisation VT-x/AMD-V, ainsi que le passthrough PCIe et GPU pour des usages plus avancés comme le transcodage ou la simulation réseau. Son interface web centralise la gestion des nœuds, du stockage et des réseaux virtuels, tout en offrant une ligne de commande puissante pour les utilisateurs expérimentés. Dans un homelab, il offre un terrain d’expérimentation complet, robuste et proche des environnements professionnels, idéal pour apprendre la gestion de clusters, la haute disponibilité et l’orchestration de services complexes.






https://forum.level1techs.com/t/project-poorly-planned-truenas-build/184990

https://www.reddit.com/r/HomeServer/


https://github.com/awesome-selfhosted/awesome-selfhosted




# Les tests qui n'ont pas fonctionné

https://www.openmediavault.org/


## Avoir tous les services sur la même machine
Soufflerie dans l'appart
## Le RAID logiciel
- En cas de perte, pas de récup possible
- Les raid checks réguliers sur une grosse grappe
## FreeNas / TrueNAS
- Basé sur FreeBSD : pas de prise en charge du contrôleur I-219V de ma carte mère : très courant sur les cartes mères. Un contrôleur gigabit ethernet intel à 2$ la puce. Sorti en 2015, FreeNas ne prends toujours pas en charge ce controleur. https://www.intel.fr/content/www/fr/fr/products/sku/82186/intel-ethernet-connection-i219v/specifications.html
- Obligé de recompiler entièrement le noyau pour ajouter un driver.
- https://www.truenas.com/community/threads/intel-ethernet-i219-v-no-driver-attached.93145/

# Conclusion

Le grand gagnant de la partie hardware
Partie software : soit truenas soit docker


https://www.truenas.com/docs/core/gettingstarted/corehardwareguide/