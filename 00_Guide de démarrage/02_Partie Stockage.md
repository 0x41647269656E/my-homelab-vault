---
title: Partie Stockage
author: "0x41647269656E"
series: "Guide de démarrage"
tags:
  - stockage
  - raid
  - zfs
  - xfs
  - mergerfs
  - jbod
  - ceph
  - glusterfs
reading-time: 30m
date: 12-11-2025
last_modified: 29-06-2026
status: published
---
Dans cet article, on s'intéresse à lister les solutions qui s'offrent à un tech-enthusiat pour monter à la maison un homelab. Mais avant de comparer les technologies, commençons par ce qui devrait toujours venir en premier : le besoin.

# Avant la technique : le cahier des charges

L'erreur classique consiste à choisir la technologie d'abord (ZFS parce que tout Reddit en parle, Ceph parce que c'est ce que font les pros) puis à découvrir ses contraintes ensuite. Faisons l'inverse : les technologies de stockage ne sont que des réponses. Encore faut-il avoir posé les questions.

## De quoi doit-on se prémunir ?

Chaque menace appelle une parade différente — et aucune technologie ne les couvre toutes :

| Menace | Parade | Exemples de technologies |
|---|---|---|
| Panne d'un **disque** | Redondance locale | RAID 1/5/6, miroir ZFS, RAIDZ, SnapRAID |
| Panne d'une **machine** (alim, carte mère, RAM...) | Redondance multi-machines... ou une bonne restauration | Ceph — ou réplication + sauvegardes |
| **Erreur humaine** (le `rm -rf` du vendredi soir) | Snapshots + sauvegardes versionnées | Snapshots ZFS, backups |
| **Corruption silencieuse** (bit rot) | Checksums, self-healing, scrubs réguliers | ZFS, Ceph |
| **Sinistre** (foudre, incendie, vol) | Copie hors site | Règle 3-2-1 |
| **Ransomware** | Sauvegardes déconnectées ou immuables | Snapshots read-only, disque offline |

> [!failure] Répétons-le : la redondance n'est pas une sauvegarde
> La redondance protège la **disponibilité** (le service continue malgré un disque mort), pas les **données** : une suppression accidentelle, un chiffrement par ransomware ou une corruption logicielle seront fidèlement répliqués sur tous les disques de la grappe, instantanément. Mes 40 To perdus en 2025 ([[00_Starting point|l'histoire est ici]]) l'ont été malgré la redondance. La règle **3-2-1** (3 copies, 2 supports différents, 1 hors site) s'applique *en plus* de tout ce qui suit, jamais à la place.

## Les six questions à se poser

1. **Mes données sont-elles reconstructibles ?** Une vidéothèque se retélécharge ; les photos de famille et le dossier Paperless, non. Il est parfaitement sain d'avoir **deux politiques de stockage** dans le même homelab : le précieux (redondé, snapshoté, sauvegardé 3-2-1) et le reconstructible (parité légère, voire rien du tout).
2. **Combien de temps puis-je rester en panne ?** Si "je répare le week-end prochain" est acceptable — et pour un homelab, c'est presque toujours le cas — alors vous n'avez *pas besoin* de haute disponibilité, et vous venez d'économiser 80 % de la complexité de ce dossier. La HA sert à ne jamais s'arrêter ; la sauvegarde sert à toujours pouvoir repartir. On confond souvent les deux.
3. **Combien de données puis-je perdre ?** Autrement dit : quel âge aura ma dernière sauvegarde (ou réplication) au moment de l'incident ? Une heure ? Une journée ? C'est ce qui dimensionne la fréquence des snapshots et des backups.
4. **Comment le stockage va-t-il grandir ?** Pool dimensionné une bonne fois pour toutes (philosophie RAIDZ) ou ajout de disques au fil de l'eau et des promos (philosophie JBOD/Unraid) ? Les deux écoles s'opposent frontalement, on le verra.
5. **Quelles performances, pour quel réseau ?** Inutile de rêver en NVMe si tout transite par du gigabit : 1 GbE ≈ 110 Mo/s, c'est le plafond de verre de la plupart des homelabs. Quant au stockage distribué, ses performances sont littéralement *définies* par le réseau.
6. **Combien de machines ?** Une seule machine élimine d'office le stockage distribué (Ceph, GlusterFS) — ou le réduit à du théâtre, on y revient plus bas.

Gardez vos réponses sous le coude : la conclusion de cet article y renverra pour désigner la technologie adaptée à chaque cas.

# Un point sur le stockage

Ces derniers temps, un grand nombre de technologies de stockage ont fait leur apparition. Commençons par démystifier les termes.
## Le RAID : la solution mid-range

Le RAID (*Redondant Arrays of Inexpensive Disks*) est une technologie historique de virtualisation du stockage permettant de réduire les risques de perte de données sur un ensemble de disques. On le retrouve sur la plupart des cartes mères modernes à l'aide d'un contrôleur de stockage compatible.
Il existe plusieurs niveaux de raids autorisant la perte d'un ou plusieurs supports physiques.

![[Pasted image 20230608133040.png]]

### Les niveaux à connaître : RAID 0, 1, 5 (et les autres)

- **RAID 0 (striping)** : les données sont découpées en bandes réparties sur tous les disques. Capacités et débits s'additionnent, mais *aucune tolérance de panne* — pire, un seul disque mort emporte **la totalité** de la grappe. À réserver à des données jetables (cache, espace de travail), jamais à du stockage.
- **RAID 1 (miroir)** : chaque octet est écrit sur deux disques. On sacrifie 50 % de la capacité contre la tolérance à la perte d'un disque, des lectures rapides et surtout une récupération triviale : chaque membre du miroir reste lisible seul. Le choix par défaut d'un petit serveur à deux disques.
- **RAID 5 (parité répartie)** : n disques, capacité de n-1, tolère la perte d'un disque. Le compromis capacité/protection historique... qui vieillit mal à l'ère des gros disques : reconstruire une grappe de disques de 12-20 To prend des jours, pendant lesquels les survivants sont sollicités à 100 % — le pire moment pour rencontrer une erreur de lecture ou perdre un deuxième disque. Au-delà de quelques To par disque, préférez une double parité.
- **RAID 6 / RAIDZ2 (double parité)** : capacité de n-2, survit à deux pertes simultanées — donc à *une panne pendant la reconstruction*. Le standard raisonnable dès 4-5 gros disques.
- **RAID 10 (miroirs stripés)** : performances maximales, reconstructions rapides, 50 % de capacité. Le choix des bases de données et des VMs.

> [!tip] La vraie question n'est pas "quel RAID ?" mais "que se passe-t-il pendant la reconstruction ?"
> Une grappe dégradée n'est plus redondante. Le niveau de parité (simple ou double) se choisit en fonction du temps de reconstruction — donc de la taille des disques, pas seulement de leur nombre.

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

## OpenZFS - La solution next-gen

OpenZFS est un fork communautaire et open-source du système de fichier ZFS (_Zettabyte File System_) initialement développé par Sun Microsystems pour son système Solaris. Ce système de fichier performant et évolutif offre des fonctionnalités **avancées** de gestion, d'intégrité et de fiabilité des données.

( NDLR : Par abus de langage, je vais parler de ZFS, mais nous parlons de l'implémentation OpenZFS disponible sur les systèmes debian, freebsd...)

### Pros

ZFS présente de nombreux avantages par rapport aux systèmes de fichiers traditionnels :

1. **Intégrité des données** : ZFS utilise un mécanisme de vérification de somme de contrôle pour détecter et corriger les corruptions silencieuses des données. Il garantit ainsi que vos données restent intègres et protégées contre les erreurs matérielles ou logicielles.

2. **Regroupement du stockage** : ZFS vous permet de regrouper plusieurs dispositifs de stockage en une seule pool. Cette pool peut ensuite être divisée en ensembles de stockage virtuels qui peuvent être redimensionnés et gérés de manière dynamique selon les besoins.     
3. **Snapshots basés sur la copie sur écriture** : ZFS propose une fonctionnalité puissante d'instantanés qui vous permet de créer des copies de votre système de fichiers à un instant précis. Ces instantanés sont légers, efficaces en termes d'espace et peuvent être créés quasi-instantanément.

4. **Compression des données** : ZFS prend en charge la compression des données en temps réel, ce qui permet de réduire l'espace de stockage utilisé. Attention à prévoir un CPU un peu plus costaud...

5. **Déduplication** : En utilisant des algorithmes de hachage comme SHA-256, le système de fichiers est capable de ne stocker qu'une version d'un même segment de données

6. Fonctionnalités similaires à RAID : ZFS intègre des fonctionnalités similaires à celles des systèmes RAID. Il propose différents niveaux de redondance des données, tels que la mise en miroir et les RAID basés sur la parité, pour protéger contre les défaillances de disque et assurer la disponibilité des données.
### Cons

1.  On ne peut pas convertir “à chaud” un vdev existant d’un type RAIDZ vers un autre type (par ex. RAIDZ1 → RAIDZ2) : la géométrie (nombre de disques, parité…) est fixée lors de la création initiale.
2. Si on veut “augmenter” la capacité en remplaçant les disques existants par des disques de plus grande taille, ZFS le permet — mais il faut remplacer tous les disques un à un, attendre le resynchronisation (re-“resilvering”) pour chacun, **PUIS** la pool pourra utiliser la taille augmentée.
3. Concernant la suppression d’un disque d’un vdev RAIDZ (réduire le nombre de disques ou retirer un disque tout en gardant le pool) : c’est effectivement une opération difficile — ce n’est pas supporté en tant que fonction standard.
4. Enfin, l’idée qu’avec RAIDZ, les données sont distribuées / “stripées + parité” sur l’ensemble des disques implique qu'il n'est pas possible d'éteindre un disque individuellement (spin-down) sans impacter l’intégrité globale
5. Le passage d’un niveau de parité supérieur (par exemple RAIDZ1 → RAIDZ2) n’est toujours pas possible “in place". Il faut recréer un nouveau vdev/pool, migrer les données, ce qui requiert souvent un backup externe ou un espace temporaire suffisant. Voir [ce lien](https://mtlynch.io/raidz1-to-raidz2/)

### RAID-Z Expension de ZFS 

 L’expansion des grappes ZFS est en développement depuis plus de dix ans et financé depuis 2017 et ce n’est toujours pas parfait. La mise en oeuvre de cette fonctionnalité n'est officiel que depuis la sortie de OpenZFS 2.3 (début 2025) (voir [cet article](https://freebsdfoundation.org/blog/openzfs-raid-z-expansion-a-new-era-in-storage-flexibility/)). C'est donc une fonctionnalité récemment implémenté et à prendre avec des pincettes. Le système reste très limité dans ce qu’on peut ajouter et dans la manière dont on peut le faire. On ne peut toujours pas ajouter un disque et faire évoluer un pool de RAIDZ1 vers RAIDZ2. On ne peut pas non plus ajouter un disque de plus grande capacité et utiliser l’espace supplémentaire tant que _tous_ les autres disques n’ont pas été remplacés et égalisés. Retirer un disque d’un pool reste également très difficile. On ne peut pas éteindre individuellement des disques, parce que chaque fichier est réparti en bandes sur l’ensemble des disques. Et il existe encore d’autres inconvénients qui rendent ZFS plus coûteux par téraoctet, à moins d’acheter dès le départ un gros pool de disques.
### Conclusion

En résumé, ZFS est un système de fichiers avancé qui combine des fonctionnalités de gestion des données, d'intégrité et de fiabilité pour répondre aux besoins des environnements de stockage modernes dans la lignée des infrastructures hyperconvergés et "software-defined".
Cette solution s'adresse à des installations haut de gamme avec 4 à 20 disques installés. Il y a cependant des inconvénients reconnus des solutions ZFS / RAIDZ rendant l'expansion "progressive et souple" plus difficile que sur des systèmes RAID "add-as-you-go"
 
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

### Ce que le gorille mange réellement

Un cluster Ceph, même minimal, empile des démons : des moniteurs (MON, en quorum de 3), des managers, et un OSD par disque — chaque OSD réclamant de l'ordre de 4 à 5 Go de RAM. Ajoutez la réplication par défaut en 3 copies (votre capacité utile est divisée par trois) et un réseau rapide, idéalement dédié : **une écriture n'est acquittée qu'une fois répliquée sur les autres nœuds**, la latence du réseau devient donc la latence de votre stockage. En 1 GbE, l'expérience est douloureuse ; le 10 GbE n'est pas un luxe, c'est le ticket d'entrée.

### Mon avis : Ceph en mono-nœud

C'est techniquement possible (en abaissant le domaine de panne de la machine au disque), mais c'est faire monter le gorille dans la twingo : tout l'encombrement, aucun bénéfice. Vous payez la totalité de la complexité logicielle — démons, RAM, chemin d'I/O RADOS — pour un résultat qu'un miroir ZFS local obtient mieux, plus vite et plus simplement, avec des performances nettement supérieures sur le même matériel. Verdict : un bac à sable pour apprendre Ceph, jamais un endroit où poser des données.

### Mon avis : Ceph à trois nœuds

Trois nœuds, c'est le minimum vital (quorum des moniteurs oblige). Avec trois machines, du 10 GbE et des SSD, on obtient enfin le vrai spectacle : un nœud s'éteint, les services redémarrent ailleurs, le cluster se répare seul — l'hyperconvergence façon Proxmox+Ceph fonctionne réellement, et c'est l'installation homelab la plus proche de ce qui se fait en entreprise. Mais mesurez le coût : trois machines qui consomment en permanence, un tiers de capacité utile, des IOPS aléatoires très en retrait d'un NVMe local (la faute aux allers-retours réseau), et une couche d'exploitation à part entière (placement groups, scrubs, rééquilibrages). En résumé : **magnifique pour apprendre la haute disponibilité, légitime si la disponibilité est un vrai besoin ; disproportionné pour servir Jellyfin et trois sauvegardes** — relisez la question 2 du cahier des charges.
## GlusterFS

GlusterFS a longtemps été « le stockage distribué du pauvre » — et c'était un compliment. Le principe est élégant : chaque nœud expose de simples répertoires posés sur un système de fichiers classique (les *bricks*), que Gluster agrège en volumes distribués et/ou répliqués, montables via un client FUSE ou NFS. Pas de base d'objets, pas de démons de placement : les fichiers restent des fichiers, lisibles directement sur les bricks — un atout appréciable le jour où tout va mal. Face à Ceph, c'était l'option simple : un volume `replica 3` (ou `replica 2` + arbitre pour économiser un disque) se monte en une soirée.

Le tableau s'assombrit sur deux points :

- **Les performances.** Correctes en séquentiel sur de gros fichiers (le réseau reste le plafond), elles s'effondrent sur les métadonnées : parcourir des répertoires de milliers de petits fichiers (bibliothèque photos, Maildir...) interroge chaque réplique et devient vite pénible. En mono-nœud, l'exercice n'a tout simplement aucun sens — autant monter le disque en direct. À trois nœuds en `replica 3`, son terrain de jeu naturel, on obtient une tolérance de panne machine honnête au prix d'écritures multipliées par trois sur le réseau.
- **La santé du projet**, et c'est rédhibitoire : Red Hat, moteur historique du développement, a arrêté son produit Gluster Storage (fin de support fin 2024) et l'activité upstream est réduite depuis à une maintenance minimale.

> [!failure] Mon avis
> En 2026, on ne bâtit plus d'infrastructure neuve sur GlusterFS. Si le besoin distribué est réel : Ceph. S'il ne l'est pas (relire la question 2 du cahier des charges) : de la réplication simple entre machines. Je le documente parce que vous le croiserez dans d'innombrables tutoriels — datés.

# Quelle technologie pour quel besoin ?

Reprenons le cahier des charges du début et déroulons les scénarios types :

| Votre situation | La réponse raisonnable |
|---|---|
| Médias reconstructibles, disques dépareillés achetés au fil des promos, silence et facture électrique comptent | JBOD agrégé (mergerfs) + parité SnapRAID calculée périodiquement — ou [[03_Les plateformes\|Unraid]], qui industrialise exactement cette recette |
| Données uniques et précieuses, une seule machine | Miroir ZFS (2 disques) ou RAIDZ2 (5-8 disques) + snapshots + sauvegarde 3-2-1 |
| VMs et conteneurs exigeants en IOPS | Miroir ZFS sur SSD/NVMe ; le stockage froid à part, sur HDD |
| Survivre à la panne d'une *machine*, sans prétention à la continuité de service | Deux machines et de la réplication (zfs send/receive, [[Syncthing]], rsync) + bascule manuelle : 90 % du bénéfice de la HA pour 10 % de sa complexité |
| Trois machines, du 10 GbE, la HA comme objectif d'apprentissage ou vrai besoin | Ceph, idéalement via Proxmox — en connaissance du ticket d'entrée décrit plus haut |
| Grappe de disques identiques, pas envie de ZFS | RAID logiciel mdadm (1 ou 6) — en acceptant l'absence de checksums de bout en bout et les caveats listés plus haut |

Et deux principes pour arbitrer les cas restants :

- **La complexité est un coût récurrent, pas un investissement ponctuel.** Chaque étage (parité, ZFS, distribution) devra être compris, surveillé, puis réparé un jour de panne — par la même personne : vous. La technologie la plus avancée que vous maîtrisez *vraiment* vaudra toujours mieux que la plus avancée qui existe.
- **Aucune de ces technologies ne remplace la sauvegarde.** Le choix ci-dessus détermine combien de pannes vous encaissez sans interruption ; la règle 3-2-1 détermine si vos données existent encore l'année prochaine. Ce sont deux budgets séparés, et le second n'est pas négociable.

# Ressources diverses

- Cartes SATA : https://www.amazon.fr/gp/product/B098QPBCBJ/







