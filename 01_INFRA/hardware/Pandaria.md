---
title: "Pandaria — fiche technique"
author: "0x41647269656E"
series: "Infrastructure"
tags:
  - hardware
  - serveur
  - nas
  - pandaria
reading-time: 10m
date: 31-05-2026
last_modified: 31-05-2026
status: published
---
# 🖥️ Présentation Machine Pandaria

ˋPandariaˋ est un serveur de stockage de données. Il contient la couche de stockage "disques durs" de mon homelab. Naturellement, sachant qu'il possède les disques, j'ai placé le serveur multimédia Jellyfin dessus.

## 📌 Vue d'ensemble
- **Nom de la machine :** Pandaria
- **Rôle principal :** Stockage de données / Serveur Multimédia
- **Statut :** Production
- **Localisation :** (physique ou réseau)
## ⚙️ Configuration matérielle

### 🧠 CPU

- Modèle : [Intel Core i5-11400F](https://www.intel.fr/content/www/fr/fr/products/sku/212271/intel-core-i511400f-processor-12m-cache-up-to-4-40-ghz/specifications.html)
- Nombre de cœurs : 6
- Nombre total de threads : 12
- Fréquence de base : 2.60 GHz

### 🧩 RAM

- **Modèle** : Kingston **HyperX Fury**
- **Total** : 16 Go (2 × 8 Go)
- **Type** : DDR4 3200 MHz (CL16)
- **Canaux** : Dual Channel
- **Référence** : Kingston KHX3200C16D4/8GX

### 💾 Stockage

#### Architecture physique

Le système repose sur un SSD NVMe Crucial P2 CT250P2SSD8 de **256 Go** dédié à l’OS, partitionné avec LVM et formaté en **ext4**. 

Le stockage de données est assuré par six disques durs de 8 To chacun dont voici les modèles : 

| Device | Nom commercial       | Modèle              |
| ------ | -------------------- | ------------------- |
| sda    | Seagate Ironwolf 8To | ST8000VN0022-2EL112 |
| sdb    | Seagate Ironwolf 8To | ST8000VN0022-2EL112 |
| sdc    | Seagate Ironwolf 8To | ST8000VN004-2M2101  |
| sdd    | Seagate Ironwolf 8To | ST8000VN004-2M2101  |
| sde    | Seagate Ironwolf 8To | ST8000VN004-2M2101  |
| sdf    | Seagate Ironwolf 8To | ST8000VN004-2M2101  |
Ils sont connectés à la carte-mère via un contrôleur HBA de type LSI SAS2008 (équivalent LSI 9211-8i) configuré en mode IT, exposant les disques directement au système sans couche RAID matériel. La [carte "RAID"](https://ebay.us/m/kw7qka) a été achetée car la carte-mère d'origine ne disposait pas suffisamment de ports SATA.

Chaque disque est partitionné avec une unique partition (`sda1` à `sdf1`) et formaté en **XFS** avec les options `noatime` et `inode64` afin d’optimiser les performances et la gestion des grands volumes.

Stockant principalement des données multimédias (films et séries), la perte d'une partie de la bibliothèque locale est acceptable et ne permet pas de justifier l'adjonction de technologies de redondances (comme RAID) plus complexes et plus coûteuses.
#### Description de l’architecture logique stockage

Les disques sont montés individuellement sous le point de montage `/storagepool/sdX`, puis agrégés logiquement à l’aide de **mergerfs**, qui permet de présenter un espace de stockage unifié monté sur `/media`. La configuration mergerfs utilise la stratégie `category.create=mfs` (Most Free Space) pour répartir les nouvelles données sur le disque disposant du plus d’espace libre, avec une réserve minimale de 50 Go par disque (`minfreespace=50G`) afin d’éviter les saturations. L’option `use_ino` garantit la cohérence des inodes, tandis que `allow_other` permet l’accès aux utilisateurs non root.

```
                ┌─────────────────────────────┐
                │       Pandaria (NAS)        │
                └────────────┬────────────────┘
                             │
               ┌─────────────┴─────────────┐
               │                           │
        ┌──────────────┐          ┌────────────────────┐
        │SSD NVMe 256Go│          │   HBA LSI SAS2008  │
        │     (OS)     │          │   (mode IT)        │
        └──────┬───────┘          └─────────┬──────────┘
               │                            │
     ┌─────────┴─────────┐      ┌───────────┴────────────┐
     │ / /boot /boot/efi │      │ 6 × HDD  (sda → sdf)   │   
	 └───────────────────┘      │ Seagate Ironwolf 8To   │
                                └───────────┬────────────┘
                                            │
                         ┌──────────────────┴──────────────────┐
                         │   /storagepool/sd[a-f] (XFS)        │
                         └──────────────────┬──────────────────┘
                                            │
                                 ┌──────────┴───────────┐
                                 │     mergerfs         │
                                 │     /media           │
                                 └──────────────────────┘
```

#### 
### 🌐 Réseau

Possédant une carte fille pour interconnecter les disques, ajouter 

- Interface :
- IP locale :
- MAC :
- VLAN :

---

## 🧱 Système

- **OS :**
- **Version :**
- **Hyperviseur :** (si applicable)
- **Filesystem :** (ext4, ZFS…)

---

## 📦 Services hébergés et packages utilisés

| Service               | Description                                                                                            | Port   |
| --------------------- | ------------------------------------------------------------------------------------------------------ | ------ |
| Jellyfin Media Server | Exposer un serveur de fichiers multimédias sur le réseau.                                              | 8096   |
| MergerFS              | Système de fichiers union permettant de regrouper plusieurs disques sous un seul répertoire logique    | /media |
| Fastfetch             | Outil d'affichage d'informations système, utilisé comme MOTD pour présenter le résumé de la machine    | N/A    |
| Docker                | Plateforme de conteneurisation utilisée pour déployer et exécuter les services de l'écosystème _*-arr_ |        |
| Smartd                | Service de surveillance automatique de l'état de santé S.M.A.R.T des disques durs.                     |        |
| Smartctl              | Utilitaire cli permettant de consulter les données des rapports et lancer des diagnostiques.           |        |


---

## 🐳 Conteneurs / VM

### Docker
- Liste des conteneurs :
  - 
  - 

### Machines virtuelles
| Nom | OS | RAM | CPU | Rôle |
|-----|----|-----|-----|------|

---

## 🔐 Sécurité

- Firewall :
- Accès SSH :
- Authentification :
- Backup :

---

## 📊 Monitoring

- Outils utilisés :
- Dashboard :
- Alertes :

---

## 🔄 Sauvegardes

- Méthode :
- Fréquence :
- Localisation :

---

## 🚀 Améliorations prévues

- Passage du boitier à un boitier rackable "spécial NAS" qui conserve des ventilateurs 120mm plein format : [Sliger CX4712](https://www.sliger.com/products/cx4712)
- 
- 

---

## 📝 Notes

- 
- 

---

## 🔗 Liens utiles

- Documentation :
- Accès web :
- Repo config :