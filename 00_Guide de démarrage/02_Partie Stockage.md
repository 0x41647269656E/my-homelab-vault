---
author: "0x41647269656E"
title: Partie Stockage
series: Guide de démarrage
---
Dans cet article, on s'intéresse à lister les solutions qui s'offrent à un tech-enthusiat pour monter à la maison un homelab.
# Un point sur le stockage

Ces derniers temps, un grand nombre de technologies de stockage ont fait leur apparition. Commençons par démystifier les termes.
## Le RAID : la solution mid-range

Le RAID (*Redondant Arrays of Inexpensive Disks*) est une technologie historique de virtualisation du stockage permettant de réduire les risques de perte de données sur un ensemble de disques. On le retrouve sur la plupart des cartes mères modernes à l'aide d'un contrôleur de stockage compatible.
Il existe plusieurs niveaux de raids autorisant la perte d'un ou plusieurs supports physiques.

![[Pasted image 20230608133040.png]]

### Raid matériel

On parle de *raid matériel* pour désigner l'utilisation d'une "carte raid" physique au sein de la machine en charge des calculs nécessaire à l'aiguillage et les reconstructions de données au sein de la grappe de disques. L'adjonction d'une carte permet souvent d'augmenter le nombre de disques connectables sur le serveur et offre des débits de transfert supérieurs aux solutions logicielles pures grâce à l'utilisation de contrôleurs matériels dédiés (stockage, chiffrement, calcul de parité...etc).

A noter, un driver logiciel est nécessaire pour pouvoir piloter la carte. Celle-ci comporte un firmware que les communautés manipulent pour les rendre compatible avec les OS. Pour exemple, la carte LSI MegaRAID 9361-8i est très prisée de la communauté TrueNas car disponible sur internet à bas prix et, sous réserve de flasher le firmware, fonctionne nativement avec TrueNAS.

Voir l'excellente vidéo :  [HBA SAS vs RAID](https://youtu.be/xEbQohy6v8U) et [ce commentaire explicatif](https://www.reddit.com/r/HomeServer/comments/t9z1zj/comment/hzy9twe/)

### Raid logiciel

Sur le même modèle que le raid matériel, ici, les opérations de lecture/écritures, routage de données et calculs de parité sont réalisées par le CPU lequel discute avec le contrôleur de stockage AHCI. Dans ce cas de figure, les disques durs sont connectés directement sur la carte mère. Pour les environnements avec des besoins faibles en écriture ([HTPC](https://fr.wikipedia.org/wiki/Home_theater_personal_computer) par exemple...), cette solution peut être privilégiée pour réduire les coûts. Mais attention à prévoir un CPU suffisamment costaud. Mdadm est une solution d'émulation des technologies RAID installé nativement sur Ubuntu.

> [!failure] Attention !
> Le RAID comporte des avantages et des inconvénients. **C'est un moyen de prévenir une panne matérielle d'un disque, pas une panne logicielle !** Une corruption du système de fichiers sera indétectable au niveau du contrôleur RAID. Le RAID ne constitut pas une solution de backup viable.
>
> Parmi les inconvénients d'un raid logiciel, les disques sont sollicités régulièrement pour réaliser des contrôles pour détecter des corruptions d'écritures de blocs de données. On compare alors la donnée stockée à la donnée de parité pour s'assurer de ne pas avoir d'erreurs lors des opérations entrée/sortie. Ce mécanisme read-intensive empêche les têtes de lecture des disques de se parker occasionnant une usure prématurée des disques et des nuisances sonores.
>
> Un autre problème concerne la taille des volumes. Lorsqu'on utilise un logiciel de récupération de fichiers (ex: Testdisk/Photorec) , celui-ci propose de travailler sur des images virtuelles des disques ou d'extraire les données vers un support de stockage distant. Si vous supprimer un dossier avec 30To de données dedans sur une grappe RAID de 50 To, il vous faudra un autre support de stockage de même taille que les données à récupérer. Une récupération "in-place" (en lieu et place de l'existant) n'est à ma connaissance pas possible pour des fichiers supprimés.

> [!failure] Attention 2 !
>  Problème : les disques d’un RAID 5 sous Linux ne se mettent jamais en veille
>  
>  Sur un système Linux utilisant **`mdadm`** pour gérer un **RAID 5 logiciel**, les disques durs restent en activité permanente, même lorsqu’aucune opération de lecture ou d’écriture n’est effectuée par l’utilisateur. En pratique, les têtes de lecture ne se parquent jamais et les disques ne passent pas en mode veille (_spindown_).
>  
>  Ce comportement s’explique par le fait que le volume RAID est **monté en permanence sur le système**, ce qui entraîne de petits accès réguliers aux disques. Ces accès peuvent provenir du système de fichiers (mise à jour des horodatages avec `atime`), des services de supervision (`smartd`, `systemd`, `updatedb`, `collectd`, etc.) ou encore de processus de vérification de parité du RAID (`mdadm --check`).
> 
> Résultat :
> - Les disques ne s’arrêtent jamais complètement.
> - Le bruit mécanique et la consommation d’énergie augmentent.
> - L’usure à long terme des disques peut s’accentuer, notamment sur des serveurs personnels ou NAS domestiques censés rester silencieux et économes lorsqu’ils sont inactifs.
> 
>  Pour corriger cela, il est nécessaire d’identifier les processus responsables des accès, de configurer correctement la mise en veille des disques avec `hdparm`, de limiter les écritures inutiles (en désactivant `atime` et certaines tâches système), ou d’envisager une gestion dynamique du montage du RAID (via `autofs` ou un script `systemd`).

## ZFS - La solution next-gen

ZFS, qui signifie Zettabyte File System, est un système de fichiers performant et évolutif initialement développé par Sun Microsystems. Il offre des fonctionnalités **avancées** de gestion, d'intégrité et de fiabilité des données.

ZFS présente de nombreux avantages par rapport aux systèmes de fichiers traditionnels :

1. **Intégrité des données** : ZFS utilise un mécanisme de vérification de somme pour détecter et corriger les corruptions silencieuses des données. Il garantit ainsi que vos données restent intègres et protégées contre les erreurs matérielles ou logicielles.
    
2. **Regroupement du stockage** : ZFS vous permet de regrouper plusieurs dispositifs de stockage en une seule pool. Cette pool peut ensuite être divisée en ensembles de stockage virtuels qui peuvent être redimensionnés et gérés de manière dynamique selon les besoins.
     
3. **Snapshots basés sur la copie sur écriture** : ZFS propose une fonctionnalité puissante d'instantanés qui vous permet de créer des copies de votre système de fichiers à un instant précis. Ces instantanés sont légers, efficaces en termes d'espace et peuvent être créés quasi-instantanément.

4. **Compression des données** : ZFS prend en charge la compression des données en temps réel, ce qui permet de réduire l'espace de stockage utilisé. Attention à prévoir un CPU un peu plus costaud...

5. **Déduplication** : En utilisant des algorithmes de hachage comme SHA-256, le système de fichiers est capable de ne stocker qu'une version d'un même segment de données

6. Fonctionnalités similaires à RAID : ZFS intègre des fonctionnalités similaires à celles des systèmes RAID. Il propose différents niveaux de redondance des données, tels que la mise en miroir et les RAID basés sur la parité, pour protéger contre les défaillances de disque et assurer la disponibilité des données.

En résumé, ZFS est un système de fichiers avancé qui combine des fonctionnalités de gestion des données, d'intégrité et de fiabilité pour répondre aux besoins des environnements de stockage modernes dans la lignée des infrastructures hyperconvergés et "software-defined". Cette solution s'adresse à des installations haut de gamme avec plus de 10 disques installés (voir cartes SAS).

## JBOD -  le stockage de données non critiques

"Just a Bunch Of Disks", c'est un terme utilisé pour décrire une configuration de stockage dans laquelle plusieurs disques durs ou SSD sont regroupés sans utiliser de technologie de redondance ou de parité de données.

La répartition des données est séquentielle. Lorsque le premier disk est full, les données sont écrites sur les suivants.

Une configuration JBOD autorise l'ajout de disques de tailles inégales.

```
╔══════════════════════════════════════════╗
║                 JBOD                     ║
╚══════════════════════════════════════════╝

  Disque 1 (500 Go)  ███████████████████████████│100%
  Disque 2 (1 To)    ███████████████────────────│65%
  Disque 3 (2 To)    ───────────────────────────│0%
  Disque 4 (1 To)    ───────────────────────────│0%

```
## CEPH, le gorille en twingo

**Ceph** est une solution libre de **stockage distribué** (_software-defined storage_) qui propose trois interfaces principales :
- **Bloc** (_RBD – RADOS Block Device_) : pour un usage similaire à un disque dur virtuel, souvent utilisé par les machines virtuelles ou les conteneurs.
- **Fichiers** (_CephFS_) : un système de fichiers distribué compatible POSIX.
- **Objet** (_RADOS Gateway – S3/Swift-like_) : une interface compatible avec les API Amazon S3 et OpenStack Swift, adaptée aux applications modernes de stockage objet.

Les objectifs principaux de Ceph sont :
- Être **complètement distribué**, sans point unique de défaillance.
- Être **hautement extensible**, jusqu’à l’exaoctet de données
- Être **libre et open source**, soutenu par une communauté active.

Les données sont **répliquées** (ou distribuées via **erasure coding**), ce qui rend le système **tolérant aux pannes** et capable de s’auto-réparer en cas de défaillance matérielle.

Ceph convient particulièrement aux **grands volumes de données** et aux **installations décentralisées**, comme un ensemble de NAS familiaux répartis sur différents sites ou un cluster d’entreprise. Il est nativement intégré dans OpenStack et Proxmox (pour le stockage des images, volumes et objets) et Kubernetes (via le provisionneur Rook).

Grâce à son architecture basée sur RADOS (Reliable Autonomic Distributed Object Store), Ceph offre des performances élevées, une forte résilience et une grande souplesse de déploiement sur du matériel standard.
## GlusterFS



- Cartes SATA : https://www.amazon.fr/gp/product/B098QPBCBJ/







