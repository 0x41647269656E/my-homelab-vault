---
title: Installer la stack *arr (Radarr, Sonarr, Prowlarr) sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - arr-stack
  - radarr
  - sonarr
  - prowlarr
  - podman
  - installation
  - securite
date: 07-06-2026
last_modified: 07-06-2026
reading-time: 15m
status: draft
---

# Installer la stack *arr (Radarr, Sonarr, Prowlarr) sous Podman rootless

> [!abstract] TL;DR
> La stack *arr automatise la gestion d'une médiathèque : **Prowlarr** (gestion des indexeurs), **Radarr** (films), **Sonarr** (séries), qui orchestrent un client de téléchargement et rangent les médias dans `/media`. C'est un cas multi-conteneurs où le point critique est le **partage cohérent des chemins** entre tous les composants — la fameuse contrainte des « hardlinks » qui fait ou défait la stack. Installation Podman rootless, accès strictement **VPN** (ces interfaces ne s'exposent jamais).

> [!info] Prérequis de lecture
> Socle de la série. Règle `/media` : [[Installer Immich sous Podman rootless]]. Pattern média : [[Installer Jellyfin sous Podman rootless]].

> [!warning] Cadre d'usage
> Cet article couvre l'installation technique de logiciels libres de gestion de médiathèque. Leur usage doit respecter le droit applicable en matière de droits d'auteur : ne télécharge et ne gère que des contenus que tu as le droit de détenir (créations personnelles, domaine public, contenus sous licence libre, sauvegardes autorisées). La responsabilité de l'usage t'incombe.

---

## La règle d'or : un seul arbre de chemins partagé

C'est LE point qui pose problème à tout le monde, alors je le mets en premier.

> [!danger] Le piège des chemins incohérents
> Si Radarr voit ses téléchargements dans `/downloads` mais que le client de téléchargement les voit dans `/data/torrents`, le déplacement vers la médiathèque se fait par **copie** (lent, double l'espace disque) au lieu d'un **hardlink** (instantané, zéro espace). Pire, ça casse le seeding. La règle absolue : **tous les conteneurs de la stack doivent voir le même arbre de fichiers au même chemin.**

> [!success] La structure recommandée sous /media
> Monter **un seul volume racine** identique dans tous les conteneurs :
> ```
> /media/library/
>   ├── downloads/      (téléchargements en cours)
>   ├── movies/         (films rangés par Radarr)
>   └── series/         (séries rangées par Sonarr)
> ```
> Chaque conteneur monte `/media/library:/data` (même chemin partout). Les hardlinks fonctionnent car `downloads` et `movies` sont sur le **même système de fichiers**, vus au **même chemin**. C'est inscriptible (la stack range les fichiers) → `keep-id`, jamais `:U` sur la racine `/media`.

---

## Les composants

- **Prowlarr** — gère les indexeurs (sources) et les distribue à Radarr/Sonarr. À configurer en premier.
- **Radarr** — films : surveille, récupère, renomme, range.
- **Sonarr** — séries : idem, avec gestion des épisodes/saisons.
- **Client de téléchargement** (ex. qBittorrent) — exécute les transferts. Souvent placé derrière un VPN dédié (kill-switch), sujet à part entière.

Tous sur un réseau interne commun pour communiquer.

```bash
podman network create arr-internal --internal
```

---

## Les Quadlets

### Prowlarr

```ini
# ~/.config/containers/systemd/prowlarr.container
[Unit]
Description=Prowlarr
[Container]
Image=ghcr.io/linuxserver/prowlarr:latest
ContainerName=prowlarr
AutoUpdate=registry
Network=arr-internal.network
Volume=%h/arr/prowlarr/config:/config:U,Z
Environment=PUID=1000
Environment=PGID=1000
Environment=TZ=Europe/Paris
NoNewPrivileges=true
DropCapability=ALL
[Service]
Restart=on-failure
MemoryMax=512M
[Install]
WantedBy=default.target
```

### Radarr

```ini
# ~/.config/containers/systemd/radarr.container
[Unit]
Description=Radarr
[Container]
Image=ghcr.io/linuxserver/radarr:latest
ContainerName=radarr
AutoUpdate=registry
Network=arr-internal.network
Volume=%h/arr/radarr/config:/config:U,Z
# CHEMIN PARTAGÉ : identique dans tous les conteneurs de la stack
Volume=/media/library:/data:z
Environment=PUID=1000
Environment=PGID=1000
Environment=TZ=Europe/Paris
NoNewPrivileges=true
DropCapability=ALL
[Service]
Restart=on-failure
MemoryMax=1G
[Install]
WantedBy=default.target
```

### Sonarr

```ini
# ~/.config/containers/systemd/sonarr.container
[Unit]
Description=Sonarr
[Container]
Image=ghcr.io/linuxserver/sonarr:latest
ContainerName=sonarr
AutoUpdate=registry
Network=arr-internal.network
Volume=%h/arr/sonarr/config:/config:U,Z
# MÊME chemin partagé
Volume=/media/library:/data:z
Environment=PUID=1000
Environment=PGID=1000
Environment=TZ=Europe/Paris
NoNewPrivileges=true
DropCapability=ALL
[Service]
Restart=on-failure
MemoryMax=1G
[Install]
WantedBy=default.target
```

```bash
mkdir -p /media/library/{downloads,movies,series}
systemctl --user daemon-reload
systemctl --user start prowlarr.service radarr.service sonarr.service
```

> [!info] Récupérer les API keys
> Chaque service expose une API key (Settings → General) qu'il faut renseigner dans les autres pour les interconnecter (Prowlarr pousse vers Radarr/Sonarr, ceux-ci parlent au client de téléchargement). C'est l'étape de câblage post-installation, à faire dans l'interface.

---

## Exposition : VPN strict, jamais public

> [!danger] Ces interfaces ne s'exposent jamais
> Les interfaces *arr donnent un contrôle total sur les téléchargements et le système de fichiers. Elles n'ont **aucune raison** d'être joignables depuis Internet. Accès **admin VPN uniquement**.

```caddyfile
radarr.mondomaine.fr {
	import security_headers
	@allowed remote_ip 10.10.1.0/24
	handle @allowed {
		reverse_proxy radarr:7878
	}
	respond "Forbidden" 403
}
# (blocs équivalents pour sonarr:8989 et prowlarr:9696)
```

> [!tip] Le client de téléchargement et son propre VPN
> Le client de téléchargement (qBittorrent/Transmission) est souvent placé dans un conteneur routé via un **VPN commercial avec kill-switch**, pour que son trafic ne passe jamais en clair par ton IP. C'est une configuration à part entière (gluetun + client), que je traiterais dans un article dédié si tu le souhaites — elle dépasse le cadre de la stack *arr elle-même.

---

## Sauvegarde

> [!success] Quoi sauvegarder
> - **`~/arr/*/config`** : les bases de chaque service (bibliothèque suivie, qualité profiles, historique, indexeurs). C'est ce qui prend du temps à reconfigurer.
> - **PAS `/media/library`** : les médias suivent la sauvegarde de `/media` (ou ne sont pas sauvegardés s'ils sont reconstituables).

---

## Conclusion

La stack *arr est un exercice de **cohérence de chemins** : un seul arbre `/media/library` monté identiquement partout, sous le même chemin `/data`, pour que les hardlinks fonctionnent. Le reste est du câblage d'API entre services sur un réseau interne. Et la règle non négociable : ces interfaces vivent derrière le VPN, jamais sur Internet.

> [!note] Articles liés
> - [[Installer Jellyfin sous Podman rootless]]
> - [[Configuration WireGuard self-hosted]]
> - [[Stratégie de sauvegarde restic 3-2-1]]

---

*Retour d'expérience personnel. Adapte chemins, domaines, plages VPN et images. Respecte le cadre légal applicable à tes contenus.*
