---
title: "Choisir ses disques durs pour un homelab NAS"
author: "0x41647269656E"
tags:
  - stockage
  - disque-dur
  - hdd
  - nas
  - smr
  - cmr
  - seagate
  - western-digital
  - toshiba
series: "Guide de démarrage"
reading-time: 20m
date: 29-06-2026
last_modified: 04-07-2026
status: published
---

Avant de choisir une architecture de stockage ([[02_Partie Stockage]]), il faut choisir les disques qui la supportent. Ce choix impacte directement les performances, la fiabilité et le coût total de votre homelab — et quelques erreurs classiques peuvent coûter cher. Tour d'horizon des marques, des technologies et des gammes disponibles en France en 2026.

# Choisir ses disques durs pour un homelab NAS

## Anatomie : ce qu'il y a dans un disque dur

Avant de parler marques et gammes, un rapide rappel de ce qu'on achète. Un disque dur mécanique est un empilement de **plateaux magnétiques** en rotation permanente (5 400 à 7 200 tours/minute), survolés par des **têtes de lecture/écriture** montées sur un bras mobile. Les données sont écrites sur des **pistes concentriques** — et c'est précisément l'agencement de ces pistes qui distingue les technologies CMR et SMR abordées plus bas.

![[hdd-anatomie.svg]]

> [!note] Pourquoi c'est important
> Ces pièces mobiles de précision expliquent la plupart des recommandations de cet article : sensibilité aux vibrations (d'où les capteurs anti-vibration des gammes NAS), usure mécanique cumulative (d'où l'intérêt des heures de fonctionnement dans les stats SMART), et consommation électrique liée à la rotation permanente (d'où le [[07_Spindown des disques sous Linux|spindown]]).

## Les acteurs du marché

Le marché des disques durs mécaniques (*HDD*) est aujourd'hui dominé par **trois fabricants** après une longue vague de consolidations. Comprendre la généalogie des marques évite beaucoup de confusions lors des achats.

### Western Digital et l'héritage HGST

**Western Digital (WD)** est l'un des deux géants historiques du secteur. En 2012, WD a racheté **HGST** (*HGST Global Storage Technologies*), anciennement la division stockage d'Hitachi elle-même héritée d'IBM. Ce rachat leur a permis d'absorber l'une des technologies les plus réputées pour la fiabilité des disques de datacenter.

```
Hitachi (disques)
      │
      └─→ HGST (2003)
                │
                └─→ Western Digital (rachat 2012)
                          │
                          ├─ WD Red / Red Plus / Red Pro  (NAS)
                          ├─ WD Gold / Ultrastar           (datacenter)
                          ├─ WD Blue                       (desktop)
                          └─ WD Purple                     (surveillance)
```

Depuis 2022-2023, WD a progressivement fusionné la marque HGST sous son propre étiquetage, mais les disques estampillés "Ultrastar" conservent le savoir-faire hérité de cette filière.

### Seagate

**Seagate** est l'autre géant, concurrent direct de WD depuis les années 80. En 2011, Seagate a racheté la **division disques durs de Samsung**, absorbant au passage leurs technologies et leur capacité de production. Seagate fabrique aujourd'hui le plus grand volume de disques HDD au monde.

```
Samsung (disques)
      │
      └─→ Seagate (rachat 2011)
                │
                ├─ IronWolf / IronWolf Pro   (NAS)
                ├─ Exos                      (datacenter)
                ├─ BarraCuda                 (desktop)
                └─ SkyHawk                   (surveillance)
```

### Toshiba

**Toshiba** est le troisième acteur, moins visible sur le marché grand public français mais présent dans les NAS et en datacenter. Leur division stockage est héritière des activités HDD de Fujitsu (racheté en 2009).

```
Fujitsu (disques)
      │
      └─→ Toshiba Storage (rachat 2009)
                │
                ├─ N300 / MN series   (NAS)
                ├─ MG series          (datacenter / enterprise)
                └─ P300 / X300        (desktop)
```

> [!note] Et les autres ?
> **Hitachi** en tant que marque de disques n'existe plus (intégré dans HGST puis WD). **Samsung** a cessé de produire des HDD en 2011 (rachat Seagate). **Maxtor** a été absorbé par Seagate en 2006. Le marché est aujourd'hui un oligopole à trois.

---

## SMR vs CMR : la différence fondamentale

C'est probablement **la décision la plus importante** lors d'un achat de disque pour NAS. Deux technologies d'écriture magnétique coexistent, avec des conséquences radicalement différentes sur les performances et la compatibilité.

![[hdd-cmr-vs-smr.svg]]

### CMR — Conventional Magnetic Recording

La technologie historique. Les pistes magnétiques sont gravées côte à côte sur le plateau, avec un espacement suffisant pour éviter toute interférence. Chaque piste est accessible et réinscriptible **indépendamment des autres**.

**Comportement :** lecture et écriture aléatoires performantes, reconstruction RAID rapide, compatible 24/7.

### SMR — Shingled Magnetic Recording

Une innovation des années 2010 pour augmenter la densité de stockage. Les pistes se **chevauchent comme des tuiles de toit** (*shingles*) — d'où le nom. La tête d'écriture est plus large que la tête de lecture, ce qui permet de graver des pistes plus serrées.

**Le problème :** écrire sur la piste B *écrase partiellement* la piste A. Pour modifier des données au milieu d'une bande, le firmware doit :
1. Lire toute la bande dans un buffer RAM interne
2. Modifier les données souhaitées
3. Réécrire toute la bande du début

Ce mécanisme de *read-modify-write* rend les **écritures aléatoires très lentes** et génère des pics de latence pouvant durer plusieurs secondes.

> [!warning] SMR et RAID : une combinaison dangereuse
> Lors d'une reconstruction RAID, le contrôleur écrit massivement et de façon aléatoire sur le disque en cours de resilvering. Un disque SMR peut être si lent que le contrôleur RAID le considère comme défaillant (*timeout*) et l'éjecte de la grappe — au pire moment possible. ZFS, mdadm et les NAS Synology/QNAP déconseillent ou refusent explicitement les disques SMR dans leurs configurations RAID.

> [!success] Cas d'usage valide pour le SMR
> Le SMR convient parfaitement aux usages **archivage / écriture séquentielle unique** : disque de sauvegarde froide, backup secondaire, stockage de photos écrites une seule fois. Le prix inférieur au gigaoctet peut être attractif dans ce contexte précis.

### L'affaire WD Red 2020

En 2020, la communauté homelab a découvert que Western Digital avait **silencieusement basculé une partie de sa gamme WD Red vers SMR** sans l'indiquer clairement — ni sur les emballages, ni dans les fiches produit. Des NAS entiers se sont retrouvés avec des performances catastrophiques et des reconstructions RAID interminables.

Sous la pression des utilisateurs et de la presse spécialisée, WD a dû clarifier sa gamme :

| Référence | Technologie | Usage recommandé |
|---|---|---|
| WD Red (sans suffixe) | **SMR** | Archive froide uniquement |
| **WD Red Plus** | **CMR** | NAS, RAID, usage 24/7 |
| **WD Red Pro** | **CMR** | NAS enterprise, 24/7 intensif |

> [!tip] Règle d'or
> Pour un NAS avec RAID ou mergerfs en usage régulier : **exiger explicitement la mention CMR**. Sur les fiches produit Amazon ou LDLC, vérifier toujours la fiche technique ou chercher « CMR » dans la description.

### Comment détecter un disque SMR

Les fabricants ne facilitent pas la tâche : la technologie d'enregistrement n'apparaît ni sur l'étiquette ni dans la sortie SMART standard. Plusieurs méthodes, de la plus simple à la plus fiable.

#### 1. Décoder la référence exacte du modèle

La méthode la plus rapide. Relever la référence complète (`smartctl -i /dev/sdX` ou l'étiquette) et la confronter aux listes officielles publiées par les fabricants après l'affaire de 2020 — WD et Seagate documentent désormais la technologie de chaque référence sur leurs sites. Des listes communautaires récapitulatives existent aussi (NASCompares maintient un tableau SMR/CMR par fabricant régulièrement mis à jour).

#### 2. Chercher le support TRIM (indice fort)

Un disque dur mécanique n'a normalement **aucune raison de supporter TRIM** — c'est une commande conçue pour les SSD. Or les disques SMR *drive-managed* (la totalité des SMR grand public) l'exposent, car leur firmware gère une couche de translation similaire à celle d'un SSD :

```bash
# Sur le disque à tester (remplacer sdX)
hdparm -I /dev/sdX | grep -i trim

# Sortie sur un disque SMR :
#    *    Data Set Management TRIM supported (limit 8 blocks)
# Sortie sur un disque CMR : rien
```

Un HDD qui annonce TRIM est **presque certainement SMR**. L'inverse n'est pas une garantie absolue (quelques vieux modèles SMR ne l'exposent pas), mais en pratique c'est l'indice le plus fiable en une commande.

#### 3. Vérifier le zonage (SMR host-managed uniquement)

```bash
lsblk -o NAME,SIZE,ZONED
```

Si la colonne `ZONED` affiche `host-aware` ou `host-managed`, le disque est SMR — mais ces modèles sont réservés aux datacenters. Les SMR grand public (*drive-managed*) affichent `none` comme un CMR : un résultat `none` **ne prouve donc rien**.

#### 4. Le test empirique : écriture aléatoire soutenue

La méthode infaillible mais destructive pour les données présentes (à réserver à un disque vide). Un disque SMR encaisse les premières écritures dans une zone tampon CMR de quelques dizaines de Go, puis s'effondre quand elle est pleine :

```bash
# Écritures aléatoires pendant 30 min — DÉTRUIT les données du disque !
fio --name=smr-test --filename=/dev/sdX --rw=randwrite \
    --bs=1M --direct=1 --runtime=1800 --time_based
```

Lecture du résultat : un **CMR** maintient un débit stable (~100-180 Mo/s) du début à la fin. Un **SMR** démarre à un débit comparable puis chute brutalement — parfois sous les 10 Mo/s avec des latences de plusieurs secondes — une fois le cache CMR saturé. C'est exactement le comportement qui fait échouer les reconstructions RAID.

> [!tip] En résumé
> Avant achat : méthode 1 (référence + listes fabricants). Disque en main : méthode 2 (`hdparm`), confirmée au besoin par la méthode 4 sur disque vierge.

---

## Les gammes NAS : ce que proposent les fabricants

Les disques "NAS" se distinguent des disques desktop par plusieurs caractéristiques techniques adaptées au fonctionnement 24/7 en environnement multi-disques. Chaque fabricant segmente son catalogue par usage — chez WD, la couleur fait office de code :

![[hdd-gammes-wd-seagate.svg]]

### Ce qui différencie un disque NAS d'un disque desktop

| Caractéristique     | Desktop (ex: WD Blue) | NAS (ex: WD Red+)     |
|---------------------|-----------------------|-----------------------|
| Cycle de vie prévu  | 8h/jour               | 24h/24, 7j/7          |
| Charge annuelle     | 55 To/an              | 180 To/an+            |
| TLER (RAID timeout) | ✗ (non configuré)     | ✓ (7-8 secondes)      |
| Anti-vibration      | ✗                     | ✓ (capteurs intégrés) |
| Garantie            | 2 ans                 | 3 à 5 ans             |

**TLER (Time-Limited Error Recovery)** est le paramètre clé pour le RAID : il limite le temps que le disque s'accorde pour corriger une erreur de lecture avant d'en informer le contrôleur. Sans TLER, un disque desktop peut mettre plusieurs minutes à tenter une correction, déclenchant un timeout RAID et une fausse alarme de défaillance.

### WD Red Plus — La référence entrée de gamme NAS

- **Technologie :** CMR
- **Capacités disponibles :** 1 à 8 To
- **Nombre de baies supportées :** jusqu'à 8 baies
- **Charge annuelle :** 180 To/an
- **Garantie :** 3 ans
- **Consommation :** ~5W inactif, ~6,4W en lecture/écriture (8 To)
- **MTBF (Mean Time Between Failures) :** 1 000 000 heures

C'est le choix naturel pour un NAS 2 à 6 baies à usage domestique (médias, sauvegardes, Nextcloud). Prix abordable, fiabilité correcte, compatible avec la quasi-totalité des NAS du marché.

### WD Red Pro — Pour les NAS plus exigeants

- **Technologie :** CMR
- **Capacités disponibles :** 2 à 24 To
- **Nombre de baies supportées :** jusqu'à 24 baies
- **Charge annuelle :** 300 To/an
- **Garantie :** 5 ans
- **MTBF :** 1 000 000 heures
- **Particularité :** taux de rotation 7200 RPM (vs 5400 RPM pour le Red Plus)

Les 7200 RPM apportent un gain en débit séquentiel et une latence réduite — pertinent si le NAS sert de stockage principal pour plusieurs utilisateurs simultanés ou du transcodage vidéo.

### Seagate IronWolf — L'alternative sérieuse

- **Technologie :** CMR
- **Capacités disponibles :** 1 à 20 To
- **Nombre de baies supportées :** jusqu'à 8 baies
- **Charge annuelle :** 180 To/an (jusqu'à 300 To/an sur certains modèles)
- **Garantie :** 3 ans
- **Particularité :** logiciel IronWolf Health Management intégré (diagnostic proactif, compatible Synology/QNAP)

Les IronWolf sont très bien intégrées dans l'écosystème Synology (DSM affiche les métriques de santé directement dans le panneau de contrôle). Fiabilité comparable aux WD Red Plus, prix souvent légèrement supérieur.

> [!info] Sur Pandaria
> Le serveur de ce homelab utilise 6 × **Seagate IronWolf 8 To**, connectées via une carte HBA LSI en IT mode. Ce choix offre 48 To bruts, organisés en JBOD via XFS + mergerfs — voir [[02_Partie Stockage]].

### Seagate IronWolf Pro — L'équivalent enterprise

- **Technologie :** CMR
- **Capacités disponibles :** 2 à 24 To
- **Nombre de baies supportées :** jusqu'à 24 baies
- **Charge annuelle :** 300 To/an
- **Garantie :** 5 ans + **plan de récupération de données Seagate offert**
- **MTBF :** 1 200 000 heures

Le plan de récupération de données inclus est une vraie différence : en cas de panne, Seagate prend en charge le rapatriement des données sur les disques récupérables. Pour un homelab avec des données importantes, c'est une assurance non négligeable.

### Toshiba N300 — Le troisième larron

- **Technologie :** CMR
- **Capacités disponibles :** 4 à 20 To
- **Nombre de baies supportées :** jusqu'à 8 baies
- **Charge annuelle :** 180 To/an
- **Garantie :** 3 ans
- **Particularité :** souvent moins cher que IronWolf à capacité égale, moins de visibilité/retours communautaires

Toshiba N300 est une option légitime, souvent ignorée car moins marketée. Les retours de la communauté homelab sont globalement positifs, avec une fiabilité comparable. À surveiller lors des soldes.

---

## Les gammes enterprise : overkill ou bonne affaire ?

Les disques "enterprise" (Seagate Exos, WD Ultrastar/Gold, Toshiba MG) sont conçus pour des datacenters avec des charges de 550 To/an et des garanties 5 ans. Ils se retrouvent régulièrement en occasion sur eBay ou Leboncoin lorsque les datacenters renouvellent leur parc.

| | NAS (Red+/IW) | NAS Pro (RedPro/IW Pro) | Enterprise (Exos/Gold) |
|---|---|---|---|
| Charge annuelle | 180 To/an | 300 To/an | 550 To/an |
| MTBF | 1 000 000 h | 1 200 000 h | 2 500 000 h |
| Garantie neuf | 3 ans | 5 ans | 5 ans |
| Vibration comp. | Basique | Avancée | Avancée |
| Prix neuf (8 To) | ~200-220€ | ~240-270€ | ~300-400€ |
| Prix occasion | Peu courant | Rare | ★ 60-120€ |

> [!warning] Acheter un disque enterprise d'occasion
> Un disque Exos ou Ultrastar récupéré d'un datacenter peut avoir **30 000 à 50 000 heures de fonctionnement** au compteur (3 à 6 ans en usage 24/7). À ce stade, le risque de panne augmente. Vérifier **systématiquement** les stats SMART avant utilisation, notamment :
> - `Reallocated_Sector_Ct` : doit être à 0 ou proche
> - `Current_Pending_Sector` : doit être à 0
> - `Offline_Uncorrectable` : doit être à 0
> - `Power_On_Hours` : le kilométrage du disque

Les disques enterprise reconditionnés sont intéressants pour du stockage secondaire ou de l'archivage, pas pour un pool principal contenant des données irremplaçables.

---

## Prix et rapport qualité/prix en France (2026)

Les prix HDD ont connu une lente baisse depuis les pics liés aux inondations en Thaïlande (2011) et à la pénurie COVID. En 2026, le marché NAS se situe approximativement comme suit :

> [!note] Remarque sur les prix
> Ces fourchettes sont indicatives, basées sur les prix observés sur Amazon.fr, LDLC, Rue du Commerce et Materiel.net. Les promotions (Black Friday, soldes) peuvent faire descendre les prix de 15 à 25 %.

### Tableau comparatif des prix NAS (entrée/milieu de gamme, CMR)

| Capacité | WD Red Plus | Seagate IronWolf | Toshiba N300 | Prix/To |
|----------|-------------|------------------|--------------|---------|
| 4 To | 80 – 95 € | 85 – 100 € | 75 – 90 € | ~20 €/To |
| 6 To | 120 – 140 € | 130 – 150 € | 115 – 135 € | ~22 €/To |
| 8 To | 190 – 220 € | 200 – 240 € | 185 – 215 € | ~25 €/To |
| 12 To | 270 – 310 € | 280 – 330 € | 260 – 300 € | ~24 €/To |
| 16 To | 340 – 390 € | 360 – 420 € | 330 – 380 € | ~23 €/To |
| 20 To | 450 – 550 € | 460 – 580 € | non dispo | ~25 €/To |

### Où acheter ?

- **LDLC** : bonne couverture NAS, SAV sérieux, prix légèrement au-dessus du marché
- **Amazon.fr** : prix souvent les plus bas, mais attention aux vendeurs marketplace (garantie réduite)
- **Rue du Commerce / Materiel.net** : comparables à LDLC, promos occasionnelles
- **Leboncoin / BackMarket** : pour les disques d'occasion/reconditionné, avec vérification SMART obligatoire

> [!tip] Le sweet spot en 2026
> Le **8 To** représente le meilleur rapport capacité/prix pour un NAS homelab : ~25 €/To contre ~20 €/To pour du 4 To. Pour un budget limité, partir sur 2 × 8 To plutôt que 4 × 4 To offre la même capacité brute pour un prix similaire, avec une empreinte électrique réduite.

### Impact de la consommation électrique sur le coût total

Un disque NAS consomme en moyenne **6 à 8W en activité** et **2 à 3W en veille**. Sur une configuration 6 disques active 24h/24 :

```
6 disques × 7W = 42W en continu
42W × 24h × 365j = ~368 kWh/an

Au tarif EDF 2026 (~0,25 €/kWh) : ~92 €/an en électricité
```

La mise en veille des disques (*spindown*) peut réduire cette facture de 40 à 60 % — voir [[02_Partie Stockage]] pour la discussion sur le spindown avec XFS + mergerfs vs ZFS.

---

## Fiabilité : les données de Backblaze

Backblaze, un service de cloud backup américain, publie **chaque trimestre des statistiques de fiabilité** sur ses dizaines de milliers de disques en production. C'est la référence open data la plus sérieuse du secteur.

Quelques constats récurrents des rapports Backblaze (à nuancer selon les lots et les années) :

- Les disques **Seagate** affichent historiquement des taux de défaillance plus variables selon les modèles — certains lots ont eu des problèmes significatifs, d'autres sont excellents.
- Les disques **HGST/WD Ultrastar** affichent régulièrement les taux de défaillance les plus bas toutes gammes confondues.
- **Toshiba** et **WD** (gammes NAS) se situent dans une moyenne correcte.

> [!warning] Ces statistiques ont leurs limites
> Backblaze utilise ses disques dans des conditions très spécifiques (serveurs de rack, densité élevée, refroidissement actif). Les chiffres ne se transposent pas directement à un usage homelab. Ils donnent cependant des tendances sur les modèles à surveiller. Voir les rapports sur [backblaze.com/cloud-storage/resources/hard-drive-test-data](https://www.backblaze.com/cloud-storage/resources/hard-drive-test-data).

---

## Recommandations par cas d'usage

### Homelab débutant, budget serré

**2 × Seagate IronWolf 4 To ou WD Red Plus 4 To**
- ~160-190 € pour 8 To bruts
- Suffisant pour démarrer avec Nextcloud, une bibliothèque multimédia modeste, des sauvegardes
- Évolutif : on peut ajouter des disques au fil du temps avec XFS + mergerfs

### NAS multimédia (Jellyfin, Plex, bibliothèque 4K)

**4 à 6 × WD Red Plus 8 To ou Seagate IronWolf 8 To**
- ~800-1300 € pour 32 à 48 To bruts
- La génération de miniatures Jellyfin et le transcodage n'écrivent pas de grosses quantités de données : le Red Plus suffit
- 8 To par disque = bon équilibre entre nombre de disques et capacité totale

### Stockage de données critiques (photos, documents, sauvegardes)

**2 × Seagate IronWolf Pro 12 To** (avec plan de récupération de données)
- ~560-660 € pour 24 To bruts (miroir = 12 To utilisables)
- La garantie 5 ans et le plan de récupération de données justifient le surcoût
- À combiner avec une stratégie 3-2-1 (voir futur article)

### Archive froide (données rarement accédées)

**WD Red (SMR, sans "Plus") ou disque desktop**
- ~60-90 € par 4 To
- Le SMR est acceptable en écriture séquentielle unique
- À ne pas mélanger avec des disques CMR dans un pool RAID

---

## Savoir lire une étiquette (et une fiche produit)

Au moment de l'achat — surtout en occasion ou en marketplace — l'étiquette du disque contient l'essentiel de ce qu'il faut vérifier :

![[hdd-etiquette-annotee.svg]]

Le point piège : la mention **CMR/SMR n'apparaît quasiment jamais** sur l'étiquette ni sur la boîte. C'est la **référence exacte du modèle** qui fait foi. Exemple chez WD en 8 To :

| Référence | Gamme réelle | Technologie |
|---|---|---|
| WD80EFAX | WD Red | **SMR** ⚠ |
| WD80EFZZ / WD80EFPX | WD Red Plus | CMR |
| WD8003FFBX | WD Red Pro | CMR |

> [!tip] Vérifier la garantie avant un achat d'occasion
> Le numéro de série permet d'interroger le statut de garantie directement chez le fabricant : [support.wdc.com](https://support.wdc.com) pour WD, [seagate.com/fr/fr/support/warranty-and-replacements](https://www.seagate.com/fr/fr/support/warranty-and-replacements/) pour Seagate. Un disque « neuf » vendu à prix cassé avec une garantie déjà entamée de 2 ans est un déstockage — ce n'est pas forcément rédhibitoire, mais ça se négocie.

---

## Checklist avant d'acheter

> [!abstract] À vérifier avant tout achat de disque pour NAS
> - [ ] **CMR confirmé** : chercher "CMR" dans la fiche technique, ou la mention "Plus" / "Pro" chez WD
> - [ ] **Gamme NAS** : pas de disque desktop dans un NAS (TLER absent, garantie réduite)
> - [ ] **Capacité × nombre de baies** : vérifier que votre contrôleur supporte la capacité unitaire (certains vieux contrôleurs sont limités à 4 To par disque)
> - [ ] **Vendor compatibility list** : si vous utilisez un NAS Synology ou QNAP, consulter la liste des disques compatibles officiellement supportés
> - [ ] **Achat par lots** : éviter d'acheter plusieurs disques du même lot de fabrication (même date, même numéro de série proche) — si un lot est défectueux, tous les disques peuvent tomber en même temps
> - [ ] **SMART post-réception** : vérifier les stats SMART dès réception avec `smartctl -a /dev/sdX`

---

## Pour aller plus loin

- [[02_Partie Stockage]] — Architecture : RAID, ZFS, JBOD, XFS + mergerfs
- [[01_INFRA/hardware/Pandaria]] — Spécifications matérielles du serveur de ce homelab
- Rapports Backblaze sur la fiabilité des disques (publiés chaque trimestre)
- Forums ServeTheHome et r/homelab pour les retours d'expérience communautaires
