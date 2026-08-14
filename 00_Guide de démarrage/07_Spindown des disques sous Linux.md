---
title: "Spindown des disques durs dans un NAS Linux : causes et solutions"
author: "0x41647269656E"
series: Administration
tags:
  - spindown
  - nas
  - disques
  - alimentation
  - hdparm
  - hd-idle
  - atime
  - smartd
  - linux
reading-time: 20m
difficulty: tech-enthusiast
date: 29-06-2026
last_modified: 29-06-2026
status: draft
---

Faire dormir les disques d'un NAS Linux est plus difficile qu'il n'y paraît. Vous pouvez vous retrouver dans un cas de figure où la commande `hdparm -y /dev/sda` s'exécute sans erreur mais pourtant le disque continue de tourner. Ce point a été un casse-tête logique pour moi pendant quelques jours. Ce guide recense toutes les causes possibles et les solutions associées.

## Pourquoi le spindown est crucial

Un NAS dédié aux films, séries et photos est par nature un stockage **froid** : les disques sont sollicités par rafales (lecture d'un film, import de photos), puis restent inactifs pendant des heures. Les laisser tourner en permanence a trois conséquences directes.

**Chaleur**

Un disque dur en rotation génère entre 5 et 8 W de chaleur par frottement mécanique (paliers, plateaux, tête de lecture). Plus les disques sont denses, plus ils ont de têtes de lectures, donc plus de frottements ! Dans un boîtier fermé avec 6 à 8 disques, cela représente 30 à 64 W de dissipation thermique continue. La chaleur est l'ennemi principal des disques mécaniques : chaque degré supplémentaire en régime de croisière accélère la dégradation des lubrifiants de palier et la déformation thermique des plateaux. Les études Backblaze montrent une corrélation nette entre température de fonctionnement et taux de panne au-delà de 40°C.

**Consommation électrique**

| État                     | Consommation par disque | 8 disques | Par an (8 760 h) |
| ------------------------ | ----------------------- | --------- | ---------------- |
| En rotation (idle)       | ~6 W                    | ~48 W     | ~420 kWh         |
| En veille (spindown)     | ~0,5 W                  | ~4 W      | ~35 kWh          |
| **Économie potentielle** |                         | **~44 W** | **~385 kWh/an**  |

À 0,25 €/kWh (tarif résidentiel français 2024), un NAS de 8 disques qui ne spindown jamais coûte environ **105 €/an** de plus en électricité qu'un NAS qui met ses disques en veille.

**Usure mécanique et projection des coûts**

Les disques durs s'usent principalement selon deux métriques : les **heures de rotation** (*power-on hours*) et le nombre de **cycles de démarrage** (*start/stop cycles*). Les Seagate IronWolf (ex. ST8000NTZ01 à 300–320 €) sont garantis pour environ 600 000 heures de rotation cumulées et 50 000 cycles de démarrage.

Scénario sans spindown (rotation permanente) :
- 8 760 h/an × 8 disques = 70 080 heures cumulées consommées par an
- Durée de vie estimée : **environ 5 ans**
- Coût de remplacement : 8 × 310 € = **2 480 €**, soit **496 €/an**

Scénario avec spindown effectif (8h actif, 16h en veille) :
- 2 920 h/an × 8 disques = 23 360 heures cumulées consommées par an
- Durée de vie estimée : **environ 12 à 14 ans**
- Coût de remplacement amorti sur 13 ans : **191 €/an**

> [!info] Projection financière
> Sur 10 ans, la différence représente environ **3 050 €** (remplacement anticipé × 2 + surcoût électrique cumulé) pour un parc de 8 × ST8000NTZ01. Le spindown n'est pas un confort — c'est une décision économique.

> [!warning] Cycles de démarrage
> Le spindown trop fréquent (toutes les 10 minutes) n'est pas neutre non plus. Chaque démarrage sollicite le moteur et les paliers. Viser un timeout de **20 à 40 minutes** d'inactivité avant mise en veille.

---

## Causes matérielles : ce qui empêche le spindown

### 1. Cartes d'extension PCIe vers SATA

Les cartes d'extension bon marché (puces **ASMedia ASM1062, ASM1166, JMicron JMB585, Marvell 88SE9235**) n'implémentent pas ou implémentent mal le routage des commandes ATA de gestion de l'alimentation. `hdparm -y /dev/sdX` s'exécute **sans erreur** mais la commande n'atteint jamais le disque.

Ces contrôleurs n'exposent pas non plus le fichier sysfs ALPM :

```bash
echo 'min_power' > /sys/class/scsi_host/hostX/link_power_management_policy
```

Le fichier est absent ou silencieusement ignoré. Le lien SATA reste actif en permanence.

| Mécanisme | Contrôleur natif (AHCI) | Carte d'extension PCIe |
|---|---|---|
| `hdparm -S` (timeout automatique) | ✓ | ✗ |
| `hdparm -y` (mise en veille manuelle) | ✓ | ✗ |
| ALPM (`link_power_management_policy`) | ✓ | ✗ |
| Spindown via `hd-idle` | ✓ | ✗ |

**Solution** : carte HBA en mode IT (ex. LSI 9207-8i flashée IT mode) qui transmet les commandes ATA sans interprétation (*passthrough* complet), ou rester sur les ports SATA natifs de la carte mère.

### 2. Polling interne du contrôleur

Certains contrôleurs émettent eux-mêmes des requêtes périodiques vers les disques pour détecter les déconnexions à chaud. Ces accès réguliers réveillent le disque même si celui-ci venait à s'endormir via un autre mécanisme.

### 3. Intel RST actif dans l'UEFI

Lorsque l'UEFI est configuré en mode **Intel RST** plutôt qu'en AHCI pur, le pilote RST désactive les transitions de lien SATA vers les états *Partial* et *Slumber*, et bloque les commandes ATA `STANDBY IMMEDIATE` et `SLEEP` pour optimiser les latences benchmark.

Résultat : même sur des ports SATA natifs, **les disques ne s'éteignent jamais** si RST est actif.

**Solution** : désactiver Intel RST dans l'UEFI, repasser en **mode AHCI**. Voir [[Intel RST — le fakeRAID]].

---

## Causes logicielles : les accès parasites

Même avec un matériel correct (AHCI, HBA IT-mode, pas de RST), les disques refusent souvent de s'endormir à cause d'accès logiciels permanents.

### 4. Mises à jour `atime` (horodatage d'accès)

Par défaut, chaque lecture d'un fichier déclenche une écriture sur son inode pour mettre à jour l'horodatage d'accès (`atime`). Sur un NAS avec Jellyfin ou Plex qui scanne sa bibliothèque toutes les heures, chaque fichier vidéo parcouru génère une écriture d'inode — le disque ne s'endort jamais.

```bash
# /etc/fstab — ajouter noatime à chaque volume de stockage
UUID=xxxx  /storagepool/sda  xfs  defaults,noatime  0  2
```

Pour EXT4, `lazytime` est une alternative plus douce : les mises à jour d'horodatage sont regroupées en mémoire et écrites seulement lors d'un `sync` ou d'une modification du contenu :

```bash
UUID=xxxx  /data  ext4  defaults,lazytime,noatime  0  2
```

### 5. Journal EXT4

EXT4 valide son journal toutes les 5 secondes par défaut via `kjournald`. Augmenter l'intervalle réduit les accès sans risque notable :

```bash
# /etc/fstab
UUID=xxxx  /data  ext4  defaults,noatime,commit=60  0  2
```

### 6. `smartd` — tests SMART horaires

`smartd` peut déclencher des tests automatiques toutes les heures. La directive `-n standby` empêche `smartd` de réveiller un disque en veille :

```bash
# /etc/smartd.conf
/dev/sda -a -n standby,10 -s S/../.././02
#              ↑                   ↑
#   ne pas réveiller          test court le dimanche à 2h seulement
```

`-n standby,10` : ne pas réveiller le disque ; après 10 tentatives consécutives en veille, considérer le disque absent. `S/../.././02` : test court une fois par semaine (dimanche à 2h) plutôt qu'à chaque heure.

### 7. `updatedb` / mlocate

`mlocate` parcourt tous les systèmes de fichiers montés chaque nuit pour mettre à jour son index. Ce scan réveille tous les disques.

```bash
# /etc/updatedb.conf
PRUNEPATHS="/media /storagepool"
```

Ou désactiver complètement si `locate` n'est pas utilisé :

```bash
systemctl disable --now mlocate.timer
```

### 8. Cache applicatif sur les volumes de données

Jellyfin, Plex et autres écrivent en continu des thumbnails, métadonnées et fichiers de transcodage. Si ces dossiers pointent vers les volumes de données, chaque scan de bibliothèque réveille les disques. Les rediriger vers le SSD système ou la RAM :

```bash
# /etc/fstab — cache Jellyfin en RAM (perdu au reboot, acceptable)
tmpfs  /var/cache/jellyfin  tmpfs  defaults,size=2G  0  0
```

### 9. `systemd-journald` en écriture continue

Par défaut, `journald` écrit les journaux sur disque. En mode volatile, les logs restent en RAM :

```bash
# /etc/systemd/journald.conf
[Journal]
Storage=volatile
```

### 10. Flush noyau trop fréquent

Le noyau Linux vide son cache d'écriture (*page cache*) vers les disques toutes les 5 secondes par défaut :

```bash
# /etc/sysctl.conf
vm.dirty_writeback_centisecs = 6000   # flush toutes les 60s
vm.dirty_expire_centisecs    = 6000   # données tolérées 60s en RAM
```

---

## Solutions complémentaires

### `hd-idle` — filet de sécurité

`hd-idle` surveille `/proc/diskstats` et envoie lui-même la commande `STANDBY` après un délai d'inactivité configurable par disque. Fonctionne indépendamment de `hdparm -S` et compense les contrôleurs qui ignorent le timer interne :

```bash
# /etc/hd-idle.conf
HD_IDLE_OPTS="-i 0 -a /dev/sda -i 1800 -a /dev/sdb -i 1800"
# spindown après 30 min d'inactivité
```

### `autofs` — montage à la demande (approche radicale)

`autofs` monte les volumes uniquement lors d'un accès et les démonte automatiquement après un timeout. Un disque démonté ne peut être atteint par aucun processus — c'est la solution la plus robuste, mais incompatible avec certains services qui supposent un montage permanent (Jellyfin, Samba).

---

> [!tip] Ordre de priorité
> Pour la majorité des NAS multimédia, cet empilement suffit à obtenir un spindown stable en 20 à 40 minutes :
> 1. `noatime` en fstab
> 2. `smartd -n standby` + planification hebdomadaire
> 3. Exclusion `updatedb`
> 4. `hd-idle` comme filet de sécurité
>
> Les points 8 à 10 (cache tmpfs, journald volatile, sysctl) sont du peaufinage pour les cas récalcitrants. `autofs` n'est utile que si les autres mesures échouent.
