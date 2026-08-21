---
title: "40 To perdus en une soirée : anatomie d'un échec RAID 5"
author: "0x41647269656E"
series: "Administration"
tags:
  - retour-experience
  - postmortem
  - mdadm
  - raid5
  - stockage
  - asm1166
  - jmb575
  - alignement
  - recuperation
reading-time: 30m
date: 12-08-2026
last_modified: 12-08-2026
status: draft
---

Cet article n'est pas un guide. C'est un post-mortem.

En 2025, j'ai perdu l'intégralité d'une grappe RAID 5 de six disques de 8 To, 40 To de données utiles en une soirée. Pas à cause d'un incendie, pas à cause d'un ransomware, pas même à cause d'une panne de disque. À cause d'une carte d'extension à 40 €, d'une carte mère morte au mauvais moment, et d'une commande tapée à 23 h qui a converti un problème réparable en perte définitive.

Je l'écris parce que la plupart des retours d'expérience sur le sujet se résument à « faites des sauvegardes ». C'est vrai, et ça ne justifie rien. Mais dans le cadre d'un Homelab, nous sommes souvent amenés à stocker des fichiers dont on se dit "Au pire, si ça tombe, je pourrais les téléchager à nouveau" et être tentés de ne pas conserver de copies. Ce qui m'intéresse ici, c'est la **chaîne causale** : chaque maillon était individuellement anodin, connu, documenté. C'est leur alignement qui a été fatal.

> [!warning] Le résumé, pour ceux qui n'iront pas plus loin
> Une grappe RAID 5 `mdadm` ne se « répare » pas avec un `mdadm --create`. Cette commande ne reconstruit rien : elle **écrit une nouvelle grappe** par-dessus l'ancienne. Si un seul des trois paramètres ordre des disques, taille de chunk, *data offset* diffère de l'original, les données sont perdues au moment où la resynchronisation démarre.

---

## 1. Le contexte matériel : six disques derrière un multiplicateur de ports

La grappe : 6xSeagate Ironwolf RED 8 To en RAID 5 logiciel (`mdadm`), soit 40 To utiles et la tolérance d'une seule panne disque. Rien d'exotique.

Le problème n'était pas les disques. Il était dans ce qui se trouvait entre eux et la carte mère : une carte d'extension SATA PCIe bon marché, articulée autour de deux puces : un contrôleur **ASMedia ASM1166** et un **multiplicateur de ports JMicron JMB575**.

[Le modèle en question](https://www.amazon.fr/gp/product/B098QPBCBJ/), pour référence.

### L'architecture d'une carte « 8 ports SATA à 40 € »

L'ASM1166 est un contrôleur SATA III honnête : 6 ports, interface PCIe 3.0 x2. Pour dépasser 6 ports sans ajouter un second contrôleur, ces cartes greffent un **multiplicateur de ports** JMB575 sur l'un des canaux de l'ASM1166. Un JMB575 prend un port SATA en entrée et en expose cinq en sortie.

```
                        ┌─────────────┐
   PCIe 3.0 x2 ─────────┤   ASM1166   │
                        └──┬──┬──┬──┬─┘
                           │  │  │  └──── port SATA natif
                           │  │  └─────── port SATA natif
                           │  └────────── port SATA natif
                           │
                    ┌──────┴──────┐
                    │   JMB575    │   ← 1 canal en entrée
                    └─┬─┬─┬─┬─┬───┘
                      │ │ │ │ └──── 5 disques se partagent
                      │ │ │ └────    la bande passante
                      │ │ └──────    d'UN SEUL port SATA
                      │ └────────
                      └──────────
```

Sur le papier, on obtient dix ports pour le prix d'un. En pratique, on obtient trois problèmes que je n'avais pas anticipés.

### Problème 1 : les disques ne s'arrêtent jamais

C'est le sujet traité en détail dans [[07_Spindown des disques sous Linux]], et c'est ici que la chaîne commence.

Ni l'ASM1166 ni, a fortiori, le JMB575 ne routent correctement les commandes ATA de gestion d'alimentation. `hdparm -y /dev/sdX` retourne un code de sortie 0. Aucune erreur, aucun avertissement dans `dmesg`. Mais le disque continue de tourner.

| Mécanisme | Comportement observé derrière ASM1166 + JMB575 |
|---|---|
| `hdparm -y` (veille immédiate) | Retour OK, **aucun effet** |
| `hdparm -S 120` (timeout auto) | Accepté, **jamais déclenché** |
| ALPM (`link_power_management_policy`) | Fichier sysfs **absent** |
| `hd-idle` | Détecte l'inactivité, envoie la commande, **sans effet** |

Conséquence directe : six disques mécaniques en rotation permanente, 8 760 heures par an, pendant plusieurs années. Têtes jamais parquées, paliers jamais refroidis.

> [!info] L'usure corrélée, ennemie du RAID
> Six disques du même modèle, du même lot, achetés le même jour, alimentés par la même alimentation, chauffés par le même flux d'air, sollicités **exactement** au même rythme par le RAID. Leurs courbes d'usure ne sont pas indépendantes — elles sont quasi identiques. Le calcul de fiabilité d'un RAID 5 suppose l'inverse. C'est une hypothèse fausse dans presque tous les homelabs.

### Problème 2 : SMART devient aveugle derrière la carte

Le passthrough des commandes SMART à travers un JMB575 est partiel et dépend du firmware. `smartctl` remonte des attributs pour certains disques, échoue avec `Unknown USB bridge` ou `Device does not support SMART` pour d'autres, et `smartd` finit par ne plus surveiller grand-chose.

Je n'avais donc **aucune visibilité** sur la dégradation réelle des disques. Pas de compteur `Reallocated_Sector_Ct` qui monte, pas de `Current_Pending_Sector` qui apparaît, pas d'alerte. Le premier signal a été un disque éjecté de la grappe.

### Problème 3 : les déconnexions intempestives

Le FIS-based switching des multiplicateurs de ports est notoirement capricieux selon les combinaisons pilote/firmware. Sous charge — typiquement pendant un `mdadm --check` mensuel — un disque disparaît une fraction de seconde. Pour `mdadm`, c'est une panne : le disque est éjecté, la grappe passe en mode dégradé.

Deux fois, j'ai remis le disque dans la grappe avec `mdadm --re-add` et laissé la reconstruction tourner. Deux fois, ça a fonctionné. J'ai classé l'incident comme « bizarrerie sans conséquence » au lieu de le classer comme « le contrôleur ment sur l'état de mes disques ».

> [!warning]
> Une resynchronisation complète d'une grappe de 40 To, c'est plusieurs jours de lecture intensive à 100 % sur cinq disques déjà usés, pour reconstruire le sixième. Chaque reconstruction inutile est une prise de risque majeure. J'en ai fait deux.

---

## 2. Le déclencheur : une carte mère qui meurt sur un socket mort

La carte mère a rendu l'âme après plusieurs années de service. Panne franche, pas de POST, aucune sortie vidéo. Rien de dramatique en soi : on remplace la carte mère, on rebranche les disques, on redémarre.

Sauf que le socket était sur la fin de sa vie : le **LGA1200**.

### Le piège d'un socket en fin de vie

LGA1200 (Intel 10e et 11e génération) n'est plus produit. Sur le marché du neuf, il ne reste que deux catégories :

| Segment | Ce qui reste disponible | Ce que ça implique |
|---|---|---|
| Entrée de gamme (H510) | Cartes à 70–90 €, souvent 4 ports SATA, 1 seul slot PCIe x16 utilisable | Impossible de tout rebrancher nativement |
| Très haut de gamme (Z590) | Cartes à 250–400 €, quand on en trouve | Prix injustifiable pour un CPU de 2021 |

Les milieu de gammes B560 et H570, les cartes entre 120 € et 150 € avec 6 ports SATA et deux slots PCIe corrects a purement et simplement disparu des stocks. C'est le comportement normal d'une plateforme en fin de vie : les distributeurs écoulent le bas de gamme en volume et gardent quelques références premium pour les remplacements de serveurs.

J'ai pris une entrée de gamme. Décision rationnelle sur le moment : remettre le serveur en route rapidement, sans réinvestir dans une plateforme condamnée. Mais cette carte avait moins de ports SATA, un chipset différent, un firmware différent et surtout, elle m'a obligé à **rebrancher les disques différemment**.

C'est le moment exact où le risque a changé de nature. Tant que le matériel reste identique, une grappe `mdadm` est stable. Dès qu'on change de contrôleur, on touche à trois choses que `mdadm` utilise pour se retrouver : la taille visible des périphériques, leur ordre d'énumération, et la géométrie rapportée.

---

## 3. Le mécanisme fatal : le désalignement

C'est ici que la perte devient définitive. Il faut d'abord comprendre où `mdadm` range ses informations.

### Où vivent les métadonnées d'une grappe

Chaque membre d'une grappe porte un **superbloc** décrivant la grappe entière : UUID, niveau de RAID, nombre de disques, position de ce disque dans l'ordre, taille de chunk, et le paramètre critique : le **`data offset`**.

```
Un membre de grappe (disque ou partition)
┌──────────┬─────────────────────┬───────────────────────────────┐
│ superbloc│    zone réservée    │      données du volume        │
│  md 1.2  │   (data offset)     │  D0 │ D1 │ D2 │ P │ D3 │ ...  │
└──────────┴─────────────────────┴───────────────────────────────┘
  4 KiB      ← 1 MiB à 128 MiB →   ↑
  du début       selon version      le premier octet réel du système de fichiers
```

Le `data offset` est une **zone réservée vide** entre le superbloc et les données. Elle existe pour permettre les redimensionnements et les changements de niveau RAID à chaud. Sa taille n'est pas une constante universelle : elle dépend de la version de `mdadm` qui a créé la grappe.

| Version de métadonnées | Emplacement du superbloc | Vulnérable à |
|---|---|---|
| `0.90` | Fin du périphérique | Tout changement de **taille visible** (HPA, redimensionnement de partition) |
| `1.0` | Fin du périphérique (−4 KiB) | Idem |
| `1.1` | Début du périphérique (offset 0) | Toute écriture sur les premiers secteurs |
| `1.2` (défaut moderne) | Début + 4 KiB | Toute écriture sur les premiers secteurs, **table de partition comprise** |

> [!info]
> La version `0.90` est plafonnée à 2 To par périphérique. Avec des disques de 8 To, on est nécessairement en métadonnées `1.x`, donc avec un superbloc situé **au début** du membre. C'est ce qui rend le scénario suivant possible.

### Le désalignement entre le membre et la table de partition parent

Deux façons de construire une grappe :

**Cas A — sur les disques entiers** : `mdadm --create /dev/md0 ... /dev/sdb /dev/sdc ...`
Les disques n'ont alors **aucune table de partition**. Le superbloc `1.2` occupe le secteur 8 (4 KiB).

**Cas B — sur des partitions** : `mdadm --create /dev/md0 ... /dev/sdb1 /dev/sdc1 ...`
Chaque disque porte une table GPT, et le superbloc est à 4 KiB **du début de la partition**, pas du début du disque.

Malheureusement, je n'avais pas conservé l'information de la manière dont j'avais à l'époque construit ma grappe RAID à l'aie de mdadm. Je n'avais pas conservé de **procédure d'installation**. Document que je m'efforce de produire et de conserver aujourd'hui pour chacune de mes installations perso.

Pour revenir à nos moutons, le cas A est un piège. Sur un disque sans table de partition, à peu près tout ce qui touche au stockage (l'installeur d'une distribution, un utilitaire de partitionnement, certains firmwares UEFI, un gestionnaire de disques graphique et dans mon cas ma nouvelle carte RAID proposent d'« initialiser » le disque et/ou écrivent sur les premiers blocs afin de déterminer. Et une initialisation GPT écrit :

```
LBA 0      : MBR de protection
LBA 1      : en-tête GPT
LBA 2–33   : table des entrées de partition (128 entrées × 128 octets)
LBA 34…    : première partition alignée, typiquement au secteur 2048
```

Le superbloc `md` de la version 1.2 se trouve au secteur 8. **Il est dans l'intervalle LBA 2–33.** Écrire une table GPT sur un membre de grappe l'écrase purement et simplement.

Dans le cas B, le désalignement prend une autre forme. Si la table de partition est recréée — parce que le nouveau contrôleur rapporte une géométrie différente, parce qu'un outil applique un alignement hérité sur 63 secteurs au lieu de 2048, ou simplement parce qu'on a refait les partitions « à l'identique » sans vérifier le secteur de départ exact — alors le début de `/dev/sdb1` se déplace.

```
Original :
  disque │══════════════│ partition sdb1 commence au secteur 2048
                         └─ superbloc md à 2048 + 8

Après recréation avec un alignement différent :
  disque │═══│ partition sdb1 commence au secteur 34
              └─ mdadm cherche le superbloc à 34 + 8 → il n'y a rien
```

Le décalage peut n'être que de quelques centaines de kilo-octets. Il suffit. `mdadm --examine /dev/sdb1` répond `No md superblock detected`, et le disque devient invisible pour la grappe.

### Ce qui s'est passé sur ma machine

Nouvelle carte mère, disques rebranchés dans un ordre différent, certains sur les ports natifs, d'autres encore derrière la carte d'extension. Au démarrage :

- l'énumération avait changé (`sdb` était devenu `sde`) — sans conséquence en soi, `mdadm` identifie ses membres par UUID, pas par nom de périphérique ;
- **deux membres sur six** ne présentaient plus de superbloc exploitable ;
- `/proc/mdstat` affichait une grappe inactive avec quatre disques sur six.

Sur un RAID 5, quatre disques sur six, c'est deux disques manquants pour une tolérance de un. La grappe ne peut pas démarrer. Mathématiquement, l'information est pourtant toujours là : les plateaux n'ont pas été effacés, seules les métadonnées de deux membres sont illisibles.

**C'était encore réparable.** Toutes les données étaient physiquement intactes.

---

## 4. La commande de trop

À ce stade, la recherche « mdadm no superblock detected » mène, avec une régularité déprimante, à la même réponse sur les forums : recréer la grappe avec `mdadm --create` en réutilisant exactement les mêmes paramètres. La grappe est alors « redécouverte » et les données réapparaissent.

C'est vrai. C'est aussi la façon la plus efficace de détruire définitivement 40 To.

### Pourquoi `--create` n'est pas une réparation

`mdadm --create` ne cherche rien, ne récupère rien, ne lit aucune métadonnée existante. Il **écrit une grappe neuve** : nouveaux superblocs sur tous les membres, nouvelle géométrie, puis démarrage immédiat d'une resynchronisation.

Pour que les données réapparaissent, il faut que la nouvelle grappe décrive **exactement** la même géométrie que l'ancienne. Trois paramètres doivent coïncider au bit près :

| Paramètre | Pourquoi ça casse si c'est faux | Piège |
|---|---|---|
| **Ordre des disques** | Les bandes sont lues dans le mauvais ordre | L'ordre d'énumération du noyau a changé avec la carte mère |
| **Taille de chunk** | Le découpage des bandes ne tombe pas au bon endroit | Défaut passé de 64 KiB à 512 KiB selon les versions |
| **`data offset`** | Les données sont lues à côté | Défaut passé de 2048 secteurs à un offset variable, jusqu'à 128 MiB, depuis `mdadm` 3.3 |

Le `data offset` est le plus vicieux, parce qu'il est invisible dans la commande. On retape scrupuleusement `--level=5 --raid-devices=6 --chunk=512`, on est convaincu d'avoir reproduit l'original — et la version de `mdadm` livrée avec la distribution installée sur la nouvelle carte mère réserve 128 MiB là où l'ancienne en réservait 1.

```
Grappe d'origine (mdadm 3.2) :
  │sb│ 1 MiB réservé │████████ données ████████████████████████
                      ↑ début réel du système de fichiers

Grappe recréée (mdadm 4.x) :
  │sb│ ──────── 128 MiB réservés ──────── │████ données ███████
                                           ↑ là où mdadm croit
                                             que commencent les données

  Décalage : 127 MiB. Le superbloc XFS n'est pas là. Rien n'est là.
```

### L'effet de la resynchronisation

Recréer la grappe aurait pu rester réversible : écrire de nouveaux superblocs n'écrase que quelques kilo-octets par disque. Le coup fatal, c'est la **resynchronisation** que `--create` lance automatiquement si on omet `--assume-clean`.

Sur un RAID 5, la parité n'est pas concentrée sur un disque : elle **tourne** d'une bande à l'autre, précisément pour répartir la charge d'écriture.

| Bande | sdb | sdc | sdd | sde | sdf | sdg |
|---|---|---|---|---|---|---|
| 0 | D0 | D1 | D2 | D3 | D4 | **P** |
| 1 | D5 | D6 | D7 | D8 | **P** | D9 |
| 2 | D10 | D11 | D12 | **P** | D13 | D14 |
| 3 | D15 | D16 | **P** | D17 | D18 | D19 |

La resynchronisation lit les cinq blocs de données de chaque bande, calcule `P = D0 ⊕ D1 ⊕ D2 ⊕ D3 ⊕ D4`, et **écrit** le résultat sur le disque de parité de cette bande. Avec un `data offset` faux, les cinq blocs lus ne sont pas les bons — mais l'écriture, elle, tombe bien quelque part de réel.

Résultat après une passe complète : environ **un sixième du volume total** a été écrasé par du bruit calculé à partir de données décalées. Et comme la parité tourne, ce sixième n'est pas regroupé : il est réparti uniformément sur les six disques, une bande sur six, partout. Aucun fichier de plus de quelques mégaoctets n'est intact. Aucun outil de récupération (`testdisk`, `photorec`, `xfs_repair`) ne peut reconstituer un système de fichiers troué de cette façon.

J'ai interrompu la resynchronisation à environ 4 %. C'était déjà terminé — 4 % d'une grappe de 40 To répartis uniformément, c'est chaque répertoire, chaque gros fichier, chaque en-tête de conteneur vidéo.

> [!warning] Le vrai enseignement de cette soirée
> L'incident matériel n'a détruit aucune donnée. Il a détruit deux superblocs, soit huit kilo-octets de métadonnées sur 48 To de plateaux. **C'est ma réponse à l'incident qui a détruit les données**, à 23 h, fatigué, sans image de sécurité, en appliquant une recette de forum sans en comprendre les préconditions.

---

## 5. Ce qu'il fallait faire

Pour référence — et parce que ces étapes se préparent **avant** l'incident, pas pendant.

### Avant : documenter la géométrie hors-ligne

Trois commandes, dont la sortie doit vivre ailleurs que sur la machine concernée (papier, gestionnaire de mots de passe, dépôt Git distant) :

```bash
# La géométrie complète, membre par membre : c'est CE fichier qui sauve la grappe
mdadm --examine /dev/sd[b-g]1 > mdadm-examine.txt

# La configuration de la grappe
mdadm --detail --scan >> mdadm-examine.txt

# Les tables de partition exactes, réinjectables avec sfdisk
for d in b c d e f g; do sfdisk -d /dev/sd$d; done >> mdadm-examine.txt
```

Les lignes qui comptent dans `mdadm --examine` :

```
        Version : 1.2
     Array UUID : 3a4f...
    Data Offset : 2048 sectors      ← celle-ci, avant tout
     Chunk Size : 512K              ← et celle-ci
   Device Role  : Active device 3   ← et l'ordre de chaque disque
```

### Pendant : ne jamais écrire sur les disques d'origine

La règle unique d'une récupération : **aucune écriture sur le support d'origine**. Le mécanisme standard sous Linux consiste à empiler un overlay copy-on-write : les lectures viennent du disque réel, les écritures partent dans un fichier creux.

```bash
# Passer les disques en lecture seule
blockdev --setro /dev/sdb

# Créer un fichier creux qui recevra toutes les écritures
truncate -s 20G /tmp/overlay-sdb.img
loop=$(losetup -f --show /tmp/overlay-sdb.img)

# Empiler l'overlay : lectures depuis sdb, écritures vers le fichier creux
size=$(blockdev --getsz /dev/sdb)
dmsetup create ovl-sdb --table "0 $size snapshot /dev/sdb $loop P 8"
```

Tous les essais d'assemblage se font ensuite sur `/dev/mapper/ovl-sdb`. Une tentative ratée se défait en supprimant le fichier creux. On peut se tromper autant de fois que nécessaire.

### La séquence correcte

1. `mdadm --assemble --scan` — la tentative normale, non destructive.
2. `mdadm --assemble --force /dev/md0 /dev/sd[b-g]1` — force l'assemblage malgré des compteurs d'événements divergents. Toujours non destructif pour les données.
3. `mdadm --examine` sur **chaque** membre encore lisible, pour reconstituer la géométrie réelle à partir des survivants.
4. Seulement en dernier recours, et **uniquement sur les overlays** : `mdadm --create --assume-clean` avec les paramètres relevés à l'étape 3. Le `--assume-clean` interdit la resynchronisation automatique — c'est lui qui sépare une tentative d'une destruction.
5. Vérifier avant de croire : `file -s /dev/md0` doit reconnaître le système de fichiers, `xfs_db -r -c "sb 0" -c print /dev/md0` doit afficher un superbloc cohérent. Tant que ce n'est pas le cas, la géométrie est fausse — on repart à l'étape 4 avec d'autres paramètres.

> [!tip]
> S'il faut retenir une seule chose de tout l'article : `--assume-clean`. C'est le drapeau qui transforme une commande irréversible en essai annulable.

---

## 6. Ce que j'en ai tiré pour l'infrastructure actuelle

Le serveur qui a remplacé celui-ci ne ressemble plus du tout au précédent, et chaque écart est une réponse directe à un maillon de cette chaîne.

| Maillon de l'échec | Choix actuel |
|---|---|
| Carte d'extension ASM1166 + multiplicateur JMB575 | **HBA LSI flashée en mode IT** — passthrough ATA intégral, SMART complet, pas de multiplicateur |
| Disques jamais parqués, usure corrélée | Spindown effectif, disques réellement au repos (cf. [[07_Spindown des disques sous Linux]]) |
| RAID 5 `mdadm` sur six disques | **Disques XFS indépendants, unifiés par mergerfs** — chaque disque est lisible seul, sur n'importe quelle machine |
| Géométrie de grappe non documentée | Aucune géométrie à documenter : il n'y a plus de grappe |
| Aucune sauvegarde hors ligne | Stratégie 3-2-1 avec `restic` (cf. [[Strategie-de-sauvegarde-restic-3-2-1]]) |
| Plateforme en fin de vie, migration subie | Stockage rendu portable : HBA + XFS, indépendants de la carte mère |

Le changement de fond est le passage du RAID à **mergerfs**. Ce n'est pas techniquement supérieur — il n'y a plus de redondance, plus de tolérance de panne, plus de reconstruction automatique. C'est un arbitrage différent :

- si un disque meurt, je perds **ce qu'il y avait dessus**, pas 40 To ;
- chaque disque porte un système de fichiers XFS complet et autonome, montable sur n'importe quelle machine Linux, sans métadonnées de grappe, sans ordre à respecter, sans `data offset` à retrouver ;
- il n'existe plus de commande capable de détruire l'ensemble du stockage en une frappe.

J'ai troqué de la disponibilité contre de la **survivabilité**. Pour un homelab où « je répare le week-end prochain » est une réponse acceptable, c'est le bon échange. La discussion complète est dans [[02_Partie Stockage]].

> [!quote]
> La redondance protège la disponibilité. Elle ne protège pas les données. Mes 40 To étaient redondés au moment où je les ai perdus.

---

## À retenir

1. **Une carte d'extension SATA bon marché n'est pas un détail d'implémentation.** Elle décide si vos disques peuvent dormir, si SMART fonctionne, et si le système voit la réalité de votre matériel. Un HBA en mode IT d'occasion coûte 50 € et supprime les trois problèmes.
2. **L'usure d'une grappe est corrélée.** Six disques identiques achetés ensemble et sollicités identiquement ne tombent pas en panne indépendamment. Le calcul de risque d'un RAID 5 suppose le contraire.
3. **Un socket en fin de vie est un risque d'infrastructure.** Quand la carte mère lâche, le remplacement disponible dicte l'architecture — au pire moment, sous contrainte de temps.
4. **La géométrie d'une grappe est une donnée critique.** `Data Offset`, `Chunk Size`, `Device Role` : trois lignes de texte, à conserver hors de la machine. Sans elles, une récupération devient une recherche exhaustive à l'aveugle.
5. **`mdadm --create` n'est pas une réparation.** C'est une création. Sur des données existantes, sans overlay et sans `--assume-clean`, c'est une destruction.
6. **La panne matérielle n'a détruit que huit kilo-octets.** Le reste, c'est la panique à 23 h. Face à un incident de stockage : arrêter la machine, dormir, lire, préparer des overlays. Les plateaux ne s'effacent pas tout seuls pendant la nuit.
