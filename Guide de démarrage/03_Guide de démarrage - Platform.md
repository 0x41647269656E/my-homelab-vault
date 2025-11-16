
### Le projet FreeNAS / TrueNAS

**TrueNas** est un système d'exploitation orienté "services" développé pour la création de NAS sous licence libre BSD. Basé sur l'OS FreeBSD, Il supporte de nombreux protocoles : CIFS, Samba, FTP, NFS, rsync, AFP, iSCSI, les rapport S.M.A.R.T et le RAID.

#### Historique du projet

Les premières versions de FreeNAS font leur apparition en 2005 et au cours de la décennie qui a suivi, il est devenu un nom connu avec plus de 10 millions de téléchargements et 1 million de déploiements dans le monde. Pendant de nombreuses années, FreeNAS et TrueNAS se sont développés côte à côte chez iXsystems - FreeNAS en tant qu'édition logicielle libre prise en charge par la communauté et TrueNAS en tant qu'édition d'entreprise pour les applications de stockage critiques. Les deux ont été maintenus séparément bien qu'ils aient beaucoup en commun, y compris une base de code presque identique.

#### TrueNAS Core (ex FreeNAS)

TrueNAS Core® (précédemment connu sous le nom de FreeNAS) est un système d'exploitation basé sur FreeBSD
Avec son RAID intégré, ses puissants outils de gestion des données et sa capacité à détecter et réparer automatiquement la corruption silencieuse des données (et la pourriture des bits), il se révèle être un très bon choix pour monter un NAS orienté autours de services.

![[Pasted image 20230219013306.png]]
> [!WARNING]
> Il est fortement conseillé de suivre les guidelines disponibles sur le site officiel et d'utiliser un ensemble de composants compatibles. Ce n'est pas parce que le composant est ancien et qu'il as été utilisé dans des milliers de références de cartes qu'il est automatiquement compatible. Voir : [TrueNAS Core Hardware Guide](https://www.truenas.com/docs/core/gettingstarted/corehardwareguide/)

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


https://discord.gg/bKBkDbEMbX


![[Pasted image 20230218210833.png]]

![[Pasted image 20230218210907.png]]

![[Pasted image 20230218211036.png]]

![[Pasted image 20230218211701.png]]


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