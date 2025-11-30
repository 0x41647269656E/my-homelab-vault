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
reading-time: 45min
---
# Les OS spécialisés

Les systèmes comme TrueNAS, Unraid ou Openmediavault sont conçus pour transformer n’importe quel ordinateur en véritable serveur NAS / Homelab, offrant une interface graphique intuitive où presque tout se fait en quelques clics : création de volumes, partage de fichiers, gestion des utilisateurs, sauvegardes automatiques, supervision du système, provisionnement d'applications multimédia, etc.

Leur objectif est de rendre accessibles des technologies habituellement réservées aux professionnels, comme ZFS, la réplication ou la déduplication, tout en restant faciles à configurer au moyen d'interfaces graphiques.

Grâce à ces OS spécialisés, on peut construire un NAS ou un homelab, performant et fiable, sans avoir besoin de maîtriser les interfaces en lignes de commande, lire des docs et configurer manuellement des services. Tout est pensé pour offrir l'expérience d’une appliance prêt à l’emploi consumer-grade, mais avec la flexibilité du matériel sous-jacent.

Ces solutions viennent en réponse aux prix exorbitants des NAS _all-included_ vendus par des acteurs comme QNAP ou Synology qui captent une bonne partie de la valeur dans leur logiciel.

 ---
## Le projet FreeNAS / TrueNAS

### Historique du projet

Les premières versions de FreeNAS sont apparues en 2005. Au cours des années, le logiciel est devenu très populaire, atteignant plus de 10 millions de téléchargements et plus d’un million de déploiements dans le monde. Pendant longtemps, FreeNAS et TrueNAS ont évolué en parallèle chez iXsystems : FreeNAS était la version libre soutenue par la communauté, tandis que TrueNAS était l’édition destinée aux entreprises pour les usages de stockage critiques. Bien qu’ils aient été gérés séparément, les deux partageaient une base de code commune.

### Les versions

#### TrueNAS Core

![[Pasted image 20230219013306.png]]

TrueNAS Core (Ex-FreeNAS) est un système d'exploitation orienté "services" développé sous licence libre BSD. Basé sur l'OS FreeBSD, Il supporte de nombreux protocoles : CIFS, Samba, FTP, NFS, rsync, AFP, iSCSI, les rapport S.M.A.R.T et son "RAID" logiciel intégré.

TrueNAS Core tire sa fiabilité de l’une de ses technologies centrales : **le système de fichiers ZFS**.

Pour rappel (voir l'article précédent [[02_Guide de démarrage - Storage]]), ZFS est un système de fichiers avancé conçu pour garantir la fiabilité et l’intégrité des données. L’un de ses atouts majeurs est la vérification systématique des blocs grâce aux checksums, permettant de détecter la corruption silencieuse et le _bit rot_. En cas d’erreur, ZFS peut automatiquement réparer les données via son mécanisme de self-healing. Le scrubbing, effectué régulièrement, analyse l’ensemble du stockage pour corriger les éventuelles anomalies. Grâce à ces fonctionnalités, TrueNAS Core offre une fiabilité exceptionnelle pour les services, les sauvegardes et les environnements critiques.

> [!WARNING]
> Il est fortement conseillé de suivre les guidelines disponibles sur le site officiel et d'utiliser un ensemble de composants compatibles. Ce n'est pas parce que le composant est ancien et qu'il as été utilisé dans des milliers de références de cartes mères qu'il est automatiquement compatible. Voir : [TrueNAS Core Hardware Guide](https://www.truenas.com/docs/core/gettingstarted/corehardwareguide/)
##### Un mot sur le mécanisme de Jails de FreeBSD

Les _jails_ de FreeBSD sont l’une des plus anciennes et des plus élégantes technologies de virtualisation légère jamais conçues. Introduites au début des années 2000, elles incarnaient alors une vision pionnière : fournir des environnements isolés, sûrs et performants, directement intégrés dans le système d’exploitation, sans dépendre d’une couche logicielle complexe ou d’outils tiers. 
Pourtant, à l’heure où l’écosystème informatique semble s’être uniformisé autour des normes OCI (_Open Container Initiative_), de Docker et des registres publics de conteneurs, les jails apparaissent comme un héritage technique allant à contre-courant de l’évolution dominante.

Il ne s’agit pas de dire que les jails sont dépassées, loin de là. Mais leur philosophie tranche clairement avec la trajectoire prise par l’industrie. Les conteneurs Linux, standardisés sous OCI, misent sur la portabilité extrême : une image peut être construite sur une machine, poussée dans un registre public, tirée sur une autre, orchestrée via Kubernetes, intégrée dans une chaîne CI/CD, et migrée d’un cloud à un autre sans réécriture. C’est un modèle pensé pour un monde où le logiciel se déploie à grande échelle, dans des environnements hétérogènes, voire éphémères.

Les jails, en revanche, restent profondément ancrées dans l’univers FreeBSD. Elles tirent leur puissance de leur intégration directe dans le système et de la cohérence de la base, mais cette force devient une faiblesse lorsqu’on la compare à l’écosystème tentaculaire et standardisé des conteneurs modernes. Pas d’images universelles, pas de registres géants contenant des milliers d’applications prêtes à l’emploi, pas de métadonnées OCI, pas de « Docker pull » instantané pour tester un service en quelques secondes : le modèle FreeBSD reste beaucoup plus artisanal, plus orienté _sysadmin_, moins _devops_.

L’industrie, elle, avance massivement dans l’autre direction : standardisation, portabilité, automatisation, reproductibilité via des images versionnées et distribuables. Dans ce paysage, les jails ressemblent presque à un rappel du passé : un outil robuste et performant mais isolé, utilisé surtout dans des environnements spécialisés, où la cohérence du système prime sur la portabilité.

Peut-être que c’est leur force.  
Peut-être que c’est leur faiblesse.

Quoi qu’il en soit, elles rappellent que toutes les évolutions du secteur ne vont pas forcément dans le sens de la simplicité ou de la fiabilité. juste dans le sens du marché. Et de ce point de vue, FreeBSD Jails reste un choix volontairement à part, presque dissident, face à l’hégémonie des conteneurs OCI. Ce que vous créerez dans TrueNAS Core restera dans TrueNAS Core et ne sera pas reproductible dans un univers entreprise, où Linux est devenu hégémonique.
#### TrueNAS SCALE

![[Pasted image 20230219014534.png]]

TrueNAS SCALE est une plateforme HCI (Hyperconverged Infrastructure) Open Source basée sur Debian. Elle combine du stockage défini par logiciel (SDS) avec des services de virtualisation et d’orchestration. Elle inclut :
- un stockage hautement extensible basé sur ZFS et GlusterFS, permettant de monter en capacité (scale-up) ou d’ajouter des nœuds (scale-out)
- le support natif des machines virtuelles (KVM)
- l’intégration de conteneurs Linux, Docker et Kubernetes
- la possibilité d’être déployée en nœud unique ou en cluster
- une conception orientée Cloud hybride

![[Pasted image 20230219014124.png]]

Contrairement à TrueNAS CORE, qui repose davantage sur la compatibilité avec des cartes matérielles de stockage traditionnelles, TrueNAS SCALE fournit une solution de stockage entièrement définie par logiciel.

TODO : RAID matériel vs HCI SDS

#### TrueNAS Enterprise

TrueNAS Enterprise offre, en complément, une suite de services professionnels :
- support technique premium
- support matériel avancé
- stabilité renforcée
- Logiciels "entreprise" : Veam Backup, Citrix, VMWare...

et des options réservées aux appliances vendues par iXsystems (X-Series, M-Series..).

![[Pasted image 20230219014025.png]]

### Retour d'expérience et problèmes rencontrés

> [!info]- Compatibilité des composants : 
> De mon expérience, j'ai tenté d'intégrer TrueNAS Core en Juillet 2021 avec une carte mère [ASUS PRIME Z590M-PLUS](https://www.asus.com/fr/motherboards-components/motherboards/prime/prime-z590m-plus/techspec/), le NIC (_Network Interface Controller_) [Intel i219-V](https://www.intel.fr/content/www/fr/fr/products/sku/82186/intel-ethernet-connection-i219v/specifications.html) (Datant de 2015) présent sur ma carte mère n'était pas compatible avec TrueNAS Core nativement. Il nécessitait l'installation d'un driver. Petite particularité sur FreeBSD, ajouter un driver n’est pas aussi simple que sous Linux : cela nécessite généralement de recompiler le noyau. Concrètement, il faut récupérer le code source complet du noyau via Git, intégrer ou activer le pilote manquant, puis reconstruire et réinstaller l’ensemble du noyau pour que le matériel soit reconnu.
> 
> A date de cet article (25/11/2025), le contrôleur **Intel i219-V** _n’est toujours pas officiellement supporté_ **TrueNAS CORE**. Voir [lien](https://www.truenas.com/community/threads/intel-ethernet-i219-v-no-driver-attached.93145/).

> [!info]- Déduplication ZFS et consommation mémoire : 
> Déduplication ZFS : Le système de fichiers utilisé peut rapidement devenir gourmand en mémoire, notamment lorsque la déduplication est activée. Cette fonctionnalité, bien qu’efficace pour économiser de l’espace, exige une quantité importante de RAM pour maintenir ses tables en mémoire et garantir des performances acceptables. La dédup nécessite de maintenir une DDT (Deduplication Table) en mémoire. Plus elle est grande, plus elle doit être gardée en RAM pour éviter des I/O massifs et catastrophiques en performance. Sur ZFS, ça consomme environ *1 à 5 Go de RAM par To* de données selon la nature des fichiers, le niveau de fragmentation et la répétitivité, ce à quoi il faut ajouter la RAM dédiée au système.

> [!info]- Flash de cartes RAIDs, modes et HBA : 
> Pour conclure, une grande partie de la communauté TrueNAS Core accompagne les utilisateurs autours des mécanismes de flash de cartes RAID pour les rendre compatibles avec TrueNAS. Sur Reddit, on trouve des listes de cartes compatibles (possédant des clones chinois de modèles emblématiques sur Ebay) utilisable avec ou sans flash de firmware de la carte. La gestion des mises à jour de firmware pour les cartes RAID peut s’avérer périeuse et fastidieuse : elle impose souvent des manipulations manuelles, l'utilisation d'outil de flash de mémoire ou l’installation de logiciels anciens / hors d'age.
> 
> Voir l'excellente vidéo :  [HBA SAS vs RAID](https://youtu.be/xEbQohy6v8U)

### Conclusions

TrueNAS est certainement la meilleure option pour celui qui veux une solution simple pilotable à l'aide d'une interface graphique et qui soit entièrement basée sur du logiciel libre. La force de la distribution vient de son support d'OpenZFS et de l'intégration <prêt à l'emploi> d'applications. Sa communauté est une des plus grande dans le monde des distributions orientées NAS grace au soutien du modèle par une entreprise assurant un support des éditions entreprise de TrueNAS.

## Xpenology (Synology DiskStation Manager)

### Description du projet

Synology DiskStation Manager est un logiciel inclus dans les NAS Synology qui ne s'installe **QUE** sur des NAS de la marque Synology. Comme un Hackintosh, Xpenology fait passer la machine comme un matériel Synology reconnu. Une emulation de composants, la modification de l'adresse MAC de la carte réseau et d'autres sécurités doivent être contournées pour permettre l'installation de cette distribution. Synology utilise un code open-source pour diffuser l'offre de services de l'OS. Certains estiment que le logiciel étant bati par dessus devraient l'etre également (voir [Code Source Synology](http://sourceforge.net/projects/dsgpl/ "Code Source Synology")).

### Retour d'expérience

Au delà de la curiosité technique que cela représente, je ne suis pas à l'aise à l'idée de faire tourner une distribution présentant par nature un retard dans la distribution des mises à jours de sécurité du système d'exploitation. La communauté devant faire sauter tous les verrous du constructeur pour proposer la mise à jour. L’installation de cette dernière est parfois très périlleuse. Elle est également par nature toujours en retard concernant l'application des patchs de sécurité. Enfin, on rappellera que les premières victimes du [ransomware Synolocker](https://korben.info/synolocker.html) étaient des utilisateurs XPEnology. OK-tier pour une installation locale, pas pour une installation accessible via internet.

PS : ça s'installe sur un pc windows dans une VM pour les curieux :)

## Unraid

![[Pasted image 20251130140957.png]]

Unraid est une plateforme pour technophiles qui tire sa force de sa modularité, il se révèle simple à administrer et très flexible. Basé sur un noyau Linux, il utilise un système de stockage hybride reposant sur XFS, BTRFS et un mécanisme de parité logiciel, permettant de combiner des disques de tailles et de types variés sans contrainte de RAID classique. Cette approche unique assure à la fois la protection des données et une extension de capacité aisée, tout en conservant un accès direct à chaque disque en cas de panne. Unraid gère nativement Docker et KVM, ce qui permet d’exécuter simultanément des conteneurs et des machines virtuelles isolées, le tout depuis une interface web condensée et technique. Il est compatible avec la majorité des architectures x86_64 et reconnaît sans difficulté la plupart des contrôleurs SATA, NVMe ou USB, ce qui en fait une solution particulièrement adaptée aux serveurs réutilisant du matériel grand public. Pour un homelab, c’est un excellent compromis entre puissance, flexibilité et simplicité : on peut y héberger Plex, Jellyfin, Nextcloud ou encore des services d’automatisation sans avoir besoin d’une expertise Linux approfondie. Sa capacité à héberger des VMs en fait le "homelab" parfait d'un point de vue technique.

## Proxmox

Proxmox est une plateforme de virtualisation (ou hyperviseur) basée sur Debian, pensée pour gérer facilement des machines virtuelles et des conteneurs (LXC) dans un environnement unique. Elle est très appréciée dans les homelabs pour sa flexibilité et sa proximité avec les solutions professionnelles comme VMWare et Hyper-V. Proxmox VE (Virtual Environment) est basé sur un code open-source et gratuit dans son édition communautaire. Il se pilote au travers d'une interface web intuitive ou via ligne de commande (ou une API complémentaire qu'il est nécessaire d'activer par le biais d'un module python à installer) pour les utilisateurs avancées (ou automatisation 😜). 

A noter qu'il existe des playbooks Ansible pour déployer des VMs. Depuis le rachat de VMWare par Broadcom, la communauté autours de Proxmox suit une ascension fulgurante, ce qui en fait un produit intéressant dans le temps. Tout comme TrueNAS, supporté par iXSystemes, Proxmox est maintenue par une société qui vend des versions entreprises de la solution.

Dans son édition communautaire, il supporte plusieurs types de stockage : **ZFS**, **LVM**, **NFS**, **Ceph**, permettant la redondance, les snapshots, le _thin provisioning_ des VMs. Il profite pleinement du matériel moderne : CPU 64 bits Intel/AMD, virtualisation matérielle (VT-x, AMD-V), passthrough PCIe et GPU.

Proxmox est une solution viable dans le temps et basé sur des technologies récentes et maintenues. C'est une très bonne base pour un homelab possédant une base matériel solide (8 vCPUs, 16 go de ram mini) en stand-alone ou sur des configurations plus petites en cluster. En deça de ces ressources, l'overhead engendré par la multiplication des VMs (même deux..) sera trop important en comparaison d'une architecture où les applications s'exécutent sur un seul et même OS avec des conteneurs légers.

Proxmox ne propose pas encore de mécanisme permettant d'héberger des conteneurs Docker, Podman ou utilisant le standard OCI et le GPU passthrough ne fonctionne pas à l'aide des conteneurs LXC.
## OpenMediaVault

TODO : Tester la distrib :)
# Conclusion

Ces projets présentées sous la forme de plateformes NAS / Homelab, proposent des solutions "tout-en-un" et un écosystème à mettre en oeuvre pour déployer rapidement et de manière plus ou moins sécurisée des services type vidéothèque, partages dropbox, Google Drive et compagnie. Chaque solution s'avère adaptée dans un cas d'usage précis. On va essayer dans cette conclusion de lister les cas et les dimensionnements dans lesquelles une solution est adaptée par rapport aux autres.

On remarque que ZFS est au coeur de certaines solutions all-in-one (TrueNAS Core, TrueNAS Scale.). Ainsi, même si le système d’exploitation est gratuit, le coût global (incluant l’électricité et le fait de garder tous les disques constamment actifs (du fait du stripping des données) est généralement plus élevé avec une configuration entièrement basée sur ZFS sous TrueNAS que sous Unraid.

TrueNAS Core est une solution fiable idéale pour la sécurité physique des données (protection contre les défaillances matérielles, corruption des données). L’administration est réputée simple et accessible. En revanche, son cœur FreeBSD limite l’intégration de certains drivers récents (comprendre, après 2010). De plus, son système de jails limite la reproductibilité des configurations. TrueNAS Core sera votre allier pour avoir un NAS simple et DIY à la maison dont la configuration de stockage ne bouge pas dans le temps.

Unraid propose une approche très flexible reposant sur l’intégration directe de QEMU pour la virtualisation et Docker pour les services. Cette combinaison est un énorme avantage pour un usage "lab". La plus grande force d’Unraid reste son array à parité non-stripée et aux disques mixtes, qui permet d’ajouter du stockage petit à petit, avec n’importe quels disques disponibles, le tout avec très peu de limitations. On peut librement ajouter ou retirer des niveaux de parité, définir des préférences pour que certaines parts utilisent certains disques ou en évitent d’autres, etc. Et l’immense quantité de plugins et d’applications permet de transformer Unraid en une plateforme 'capable' pour héberger un homelab complet, melant tests techniques et auto-hébergement de services 'maison'.

Cependant, Unraid reste un produit propriétaire et payant. La licence “lifetime” complète atteint ~230 €, soit le prix d'un disque WD Red Plus 8 To à l'heure où j'écris ces lignes. Les autres licences n’incluent qu’un an de mises à jour, ce qui peut freiner les utilisateurs qui cherchent une solution gratuite dans le temps. En revanche, si vous pensez votre "homelab" comme une extension numérique de vous-même et un environnement que vous allez faire évoluer dans le temps (nombre de services hébergés, configurations, reprise de contrôle de vos outils numériques...) alors le prix de la licence s'effacera devant les autres gains (électricité, usure des disques, modularité...).

XPenology, de son côté, cherche à reproduire l’expérience Synology sur du matériel non officiel. Le principal inconvénient réside dans la nature même du projet, un projet "hacké", ce qui soulève des inquiétudes légitimes en matière de sécurité, certaines familles de malwares ciblant particulièrement les anciens NAS Synology outdated. C'est fun pour la bidouille mais on ne conservera pas cette solution pour un usage durable.

Enfin nous attaquons Proxmox. Proxmox est LA solution à mon sens concurrente d'Unraid. Proxmox étant devenu très populaire auprès des entreprises depuis le rachat de VMWare par Broadcom. Nous pouvons présager que  son orientation "hyperviseur" et sa proximité technique avec des solutions utilisées par les entreprises permettront au projet d'assurer son développement et financer sa maintenance au travers des ventes réalisées par l'édition entreprise. Ce sujet est LE talon d'achille de l'ensemble des logiciels libre. Comment financer la maintenance. On aurait aimé une intégration des conteneurs directement dans l'interface hôte autre que LXC, mais bon, on ne peut pas tout avoir 😀.
 
OpenMediaVault, enfin, se concentre sur le rôle de NAS simple et modulaire. Basé sur Debian et totalement open source, il offre une interface claire, des partages réseau faciles à configurer et un système de plugins qui étend ses capacités, notamment via Docker. Sa philosophie “NAS pur” en fait un choix naturel pour ceux qui veulent une solution gratuite, ouverte et simple à maintenir, même si elle reste moins polyvalente que Proxmox pour un homelab plus technique.

Dans l’ensemble, le choix dépend surtout du rôle principal que doit remplir le système : TrueNAS pour la robustesse de ZFS, Unraid pour la flexibilité VM + Docker dans un environnement très accessible, XPenology pour l’expérience Synology mais avec des risques réels de sécurité, Proxmox pour un homelab complet et évolutif, et OpenMediaVault pour un NAS léger, libre et facile à gérer.