---
author: "0x41647269656E"
title: Partie Hardware
series: Guide de démarrage
---
Dans cet article, on s'intéresse à lister les solutions qui s'offrent à un tech-enthusiat pour monter à la maison un homelab.
# Solutions matérielles
## Raspberry Pi

La première option : le Raspberry Pi. C'est un nano-ordinateur monocarte à processeur ARM conçu par des professeurs du département informatique de l'université de Cambridge à partir de 2006, et développé ensuite dans le cadre de la fondation Raspberry Pi à partir de 2009.

A l'origine du projet, un ordinateur personnel de 35$ accessible à tous. Aujourd'hui, les dernières éditions  plus puissantes frôlent les 100 euros, hors accessoires (carte sd, chargeur, case...). Bien qu'alléchant à première vu, ces petites bêtes tendent à manquer de puissance pour les applications les plus consommatrices. Le streaming de medias vidéo, notamment. C'est une solution parfaite pour celui qui souhaite héberger un service peu consommateur (Home Assistant ou un VPN ou un nœud TOR ou un nœud de transactions cryptos...) mais dès qu'on vient installer plusieurs applications, l'utilisation de l'une vient dégrever les performances de l'autre.

Pour palier à ce problème, plusieurs solutions. La première est de distribuer le calcul nécessaire entre plusieurs cartes RPi. Cependant, les mises en application pratiques de calcul distribué ne sont pas courantes et se cantonnent à des usages scientifiques ou de recherche fondamentale (voir projet [OctaPi](https://projects.raspberrypi.org/en/projects/octapi-calculating-pi)).

La deuxième solution est de distribuer les différentes strates de l'application au sein d'un groupe de machines (le serveur applicatif d'un coté, la base de données de l'autre, etc). Attention cependant, tous les conteneurs ne sont pas compatibles avec votre Pi. Les applications doivent avoir été compilées pour un processeur ARM. Certains applications proposent nativement des builds ARM mais pas toutes.

![[ClusterBuild0.jpg]] ![[cluster-overview.jpeg]]

Si l'achat d'un cluster de Raspberry Pi vous intéresse, voir [cet article](https://www.zenko.io/blog/how-i-made-a-kubernetes-cluster-with-a-couple-of-raspberry-pis/) et [cet article](https://anthonynsimon.com/blog/kubernetes-cluster-raspberry-pi/).  

En résumé :
- ✅ Parfait pour débuter
- ✅ Capable d'héberger des petites applications
- ✅ Permet de gagner en compétences dans le cas d'utilisation
- ✅ Possède des capacités d'IoT (bus de communications, form-factor, silence d'utilisation) intéressantes
- ✅ Eco-friendly : faible consommation énergétique
- 🚫 Finit par faire mal au porte monnaie quand on en achète plusieurs ou pour les configurations les plus musclées.
- 🚫 Prix par carte pouvant s'avérer rapidement un problème pour des grosses configurations.
## Intel NUC

Intel NUC (*Next Unit of Computing*) est un projet d'intel sorti en 2013 visant à proposer une nouvelle génération d'ordinateurs de bureau pour les entreprises. Le modèle le plus connu (si on exclue les éditions tournées vers le gaming) est basé sur une carte mère environ 4 × 4 pouces (10 × 10 cm). L'ordinateur était conçu pour être fixé sur les supports VESA. 12 générations de NUCs ont été produites à ce jour. Les kits vendus nécessitent d'inclure un disque dur ou SSD et la mémoire vive. Point à prendre en compte dans le calcul du prix.

  ![[Pasted image 20230219193812.png]]

Pour environ 400 euros, il est possible de monter un serveur de films avec une bibliothèque de plusieurs centaines de médias. Sa puissance moyenne, son form-factor et sa connectivités en font un serveur maison idéal pour un usage HTPC (Home Theater Personnal Computer).
  ![[Pasted image 20230219193849.png]]

NDLR : Certaines machines possèdent un récepteur infrarouge. Couplé à une télécommande type Logitech Harmony et une barre de son et nous avons un home cinéma qui roule.

![[Pasted image 20230219193826.png]]

En résumé :
- ✅ Design sobre et élégant, intégrable dans n'importe quel intérieur sans faire "geek".
- ✅ Puissance convenable permettant d'installer un serveur de films personnel.
- ✅ L'intégration d'un récepteur infrarouge
- 🚫 En cas de panne de la carte mère, il faut tout changer.
- 🚫 Attention à la taille du bloc d'alimentation (externe) des NUCs pour les installations les plus cossues (et la chauffe...).
## Barebone générique

  ![[Pasted image 20230221100030.png]]

Dans la lignée des NUCs d'Intel, plusieurs marques ont lancé des petits ordinateurs "Format VESA". On note Gigabyte, Zotac, ASUS qui sont par ailleurs des assembleurs électroniques produisants des composants comme des cartes mères. Leur savoir faire n'est plus à prouver.

Attention aux versions low-cost qu'on trouve sur les sites de vente en ligne. Ces petites bêtes étant sujets aux problèmes de chauffe, il est difficile de faire jouer la garantie.

![[Pasted image 20230221100204 1.png]]

En conclusion :
- ✅ Joli et petit
- ✅ Performances très raisonnables
- ✅ Peut héberger plusieurs applications de façon confortable
- ✅ Performances suffisantes pour un usage HTPC 1080p
- ✅ Silencieux au repos
- 🚫 En usage intensif, a tendance à chauffer et faire un bruit de fond
- 🚫 Réparabilité très limitée. En cas de panne, il faut renvoyer toute la carte mère.
- 🚫 Attention à la taille du bloc d'alimentation (externe) pour les installations cossues (et la chauffe...)
## Louer un serveur dans un datacenter

Disposer d'une machine sans avoir aucunes nuisances sonores dans l'appartement, quel pied ! Attention aux coûts en revanche... Cette option est parfaite pour celui qui souhaite héberger des services sans se soucier du matériel.

Voir : [https://hostingby.design/](https://hostingby.design/), [https://www.scaleway.com/fr/dedibox/start/](https://www.scaleway.com/fr/dedibox/start/)

En résumé :
- ✅ Aucun bruit à domicile
- ✅ Partage facilité (pour les copains 😀)
- 🚫 Service payant par abonnement
- 🚫 L'arrêt du paiement entraîne la perte des données
## Louer une seedbox

Usage cantonné au téléchargement et la diffusion de films, l'offre des seedboxs s'est peu a peu étoffé. A l'origine, un projet : augmenter votre ratio de téléchargement sur les trackers torrent (Protocole P2P BitTorrent). Aujourd'hui, grace à l'automatisation, les offres incluent également des serveurs Plex, lequel peut etre accédé depuis une box TV (type Xiaomi Mi TV Box S ou Apple TV) ou un Amazon Firestick.

Hébergement Plex + Seedbox : [seedbox.cc](https://seedboxes.cc/) ou [hostingby.design](https://hostingby.design/ "https://hostingby.design/")
Box TV : [Xiaomi Mi Box S 3rd gen](https://www.mi.com/fr/product/xiaomi-tv-box-s-3rd-gen/) ou [Amazon FireTV Stick 4K](https://www.amazon.fr/fire-tv-stick-4k/dp/B0CJKTWTVT/) ou [Apple TV 4K](https://www.apple.com/fr/apple-tv-4k/)

_NDLR : d'autres éditions et services existent, je vous laissent faire vos recherches_
_NDLR²: Certaines TV contiennent un store d'applications qui permet le téléchargement de logiciels comme Plex, Jellyfin ou Emby sans acheter de lecteur Android_

![[Pasted image 20230221101216.png]]

![[Pasted image 20230221101353.png]]

En résumé :
- ✅ Simple d'utilisation
- ✅ Aucun bruit à domicile
- ✅ Partage facilité (pour les copains 😀)
- ✅ Simplifie le process le téléchargement
- 🚫 Service payant par abonnement
- 🚫 Uniquement tourné vers les partages de contenus multimédia
## Acheter un NAS (Synology ou QNAP)

![[Pasted image 20230221105005.png]]

Un NAS (pour Network Attached Storage) est le plus souvent utilisé pour sauvegarder et partager des fichiers (photos de vacances, documents, fichiers multimédia). Avec le temps, des fonctionnalités annexes sont venues se greffer aux offres. Synology et QNAP sont les marques les plus avancées sur le sujet. On peut y trouver des VPNs, des logiciels de backups intégrés, des galeries photos, audio, vidéo, partage de contacts et de calendrier. Pour les éditions les plus musclées, la possibilité d'installer des applications tierces, des VMs, des containers.

Ici, ce que nous venons chercher c'est la facilité de configuration. Synology DSM fait référence dans ce domaine avec une interface web complète et ergonomique. Pas besoin de connaissances avancées en informatique pour se lancer.

![[Pasted image 20230220102006.png]]

Il y a cependant un revers de la médaille. Le prix. Un NAS est <u>très cher</u> au vu des composants qu'il contient. Le ratio prix/performance n'est pas intéressant. Ci-dessous, la carte mère d'un NAS Synology vendu 599€ sans disques (DS920+). Observez le refroidissement passif du processeur. Pas de miracles à l'horizon niveau performances.
  ![[Pasted image 20230220101736.png]]

A noter que pour les plus gourmands en stockage, il est possible d'ajouter des unités d'extension (non pourvues de capacités applicatives) qui viennent étendre les capacités de stockage d'un NAS existant.

![[Pasted image 20230221104919.png]]
  
Pour les adeptes de DSM qui ne souhaitent pas payer la facture d'un NAS Synology, voir la section [[#Xpenology (DiskStation Manager (_DSM_))]] plus bas dans cet article.


En résumé :

- ✅ DSM - Un OS complet et très ergonomique.
- ✅ Univers tout intégré, tout compatible, accessible en quelques clics à la Apple
- ✅ Puce matérielle supportant plusieurs niveaux de RAID
- ✅ Gamme de cartes d'extensions ([Voir le magasin](https://www.synology.com/fr-fr/products/accessories/add_in))
- ✅ Gamme d'unités d'extension de stockage ([Voir le magasin](https://www.synology.com/fr-fr/products/expansion))
- ✅ Design et form-factor sobre
- ✅ Silencieux et économe en énergie (DS920+ avec trois disques : 26 watts)
- 🚫 Rapport qualité / prix horrible
- 🚫 Support limité des encodages vidéos
- 🚫 Performances dans le bas du tableau
- 🚫 Produit orienté autours du stockage : très peu de ressources disponibles pour les usages multimédias et les VMs
## Acheter un ordinateur reconditionné

Pour celui qui ne souhaite pas se servir d'une machine comme d'un stockage durable pour réaliser ses sauvegardes et simplement héberger des services pour lui et ses amis, l'achat d'un ordinateur reconditionné est une option viable et peu coûteuse. On trouve sur les sites de vente aux enchères et/ou vente en ligne des entreprises spécialisées dans le reconditionnement de machines. Celles-ci sont achetées en lot à des sociétés ayant amorti leur matériel. En général, une entreprise amorti son parc informatique entre deux et trois ans. Les machines sont ensuite ouvertes par un professionnel, les disques sont nettoyés et les composants défectueux remplacés.

Viser les gammes Lenovo ThinkCentre M720q, M920q, M710 et M10q et Dell Optiplex Micro séries 3000/5000/7000 disposant d'une forte communauté de mods/makers. (Racks 10 pouces, adaptateurs spécifiques pour disques...)

![[Pasted image 20230221141835.png]]

Exemple de mise en racks (pour des applications type cluster kubernetes) :

![[il_1140xN.7351142329_r9v8.jpg]]

![[Pasted image 20251112114820.png]]

Pour se donner un ordre d'idée de prix :

Prix neuf avec un Intel Core i5-6500T, 8Go de ram et 256Go de SSD : [899€](https://www.ldlc.com/fiche/PB00235328.html)
Prix reconditionné dans la même configuration : [247€](https://www.ebay.fr/itm/285068609746)

En résumé :
- ✅ Rapport qualité prix le plus favorable de ce comparatif
- ✅ Ordinateur puissant pouvant héberger plusieurs applications
- ✅ Facilement démontable pour upgrade des composants
- ✅ Unités type "barebone" très discrètes
- 🚫 Durée de vie incertaine du matériel
- 🚫 Garantie du reconditionné : 6 mois à 1 an selon les vendeurs

## Monter son propre serveur

LA solution pour les power-users. C'est celle qui vous offre le plus de contrôle sur le choix du matériel et le dimensionnement, fonction de vos usages. C'est aussi la solution avec le rapport performances / prix la plus intéressante. Elle présente l'avantage de pouvoir remplacer et faire évoluer chaque composant au fil du temps.

Quatre critères à prendre en compte :

- Puissance et capacités de la machine
- Adhérence avec les solutions logicielles (cc TrueNAS 😘)
- Tolérance à la panne des composants (Ram ECC vs standard, RAID...)
- Si hébergé à domicile, attention au bruit ! N'oubliez pas que cette machine fonctionnera probablement 24h/24.
- La consommation d'une bécane allumé 24/24 est bien supérieure à celle d'un NAS Synology ultra optimisé. Pensez au coût de l'électricité.