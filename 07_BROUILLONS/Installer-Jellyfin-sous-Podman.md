---
title: Installer Jellyfin sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - jellyfin
  - podman
  - installation
  - securite
  - media
date: 07-06-2026
last_modified: 07-06-2026
reading-time: 15m
difficulty: tech-enthusiast
status: draft
---

# Installer Jellyfin sous Podman rootless

> [!abstract] TL;DR
> Jellyfin est un serveur multimédia libre (films, séries, musique) sans abonnement ni télémétrie. C'est le **cas d'usage `/media` en lecture seule par excellence** : il lit ta médiathèque sans jamais devoir y écrire. Conteneur unique, le seul vrai sujet technique étant le **transcodage matériel** (accès GPU). Je l'installe en Podman rootless, accès via Caddy, exposition selon ton usage (souvent VPN, parfois public pour le partage famille).

> [!info] Prérequis de lecture
> Socle de la série : [[Self-hosting sécurisé avec Podman]], [[Reverse proxy Caddy avec TLS automatique]], [[Configuration WireGuard self-hosted]], confinement MAC, [[Stratégie de sauvegarde restic 3-2-1]]. La règle de montage `/media` est posée dans [[Installer Immich sous Podman rootless]].

---

## Le bon élève du montage `/media`

Jellyfin ne **modifie jamais** tes fichiers média. Il les lit, les analyse, en extrait des métadonnées qu'il stocke dans **sa propre base** (volume dédié). C'est exactement le pattern idéal :

> [!success] Séparation nette
> - **`/media/movies`, `/media/series`, `/media/music`** → montés en **lecture seule** (`:ro,z`). Tes fichiers sont intouchables : un Jellyfin compromis ne peut ni chiffrer ni supprimer ta médiathèque.
> - **`~/jellyfin/config`** et **`~/jellyfin/cache`** → volumes dédiés inscriptibles pour la base, les métadonnées, les miniatures, le cache de transcodage.

---

## Le Quadlet

```ini
# ~/.config/containers/systemd/jellyfin.container
[Unit]
Description=Jellyfin Media Server

[Container]
Image=docker.io/jellyfin/jellyfin:latest
ContainerName=jellyfin
AutoUpdate=registry
Network=jellyfin-internal.network

# Données générées par Jellyfin : volumes dédiés
Volume=%h/jellyfin/config:/config:U,Z
Volume=%h/jellyfin/cache:/cache:U,Z

# Médiathèque en LECTURE SEULE : aucun risque sur tes fichiers
Volume=/media/movies:/media/movies:ro,z
Volume=/media/series:/media/series:ro,z
Volume=/media/music:/media/music:ro,z

# Durcissement
NoNewPrivileges=true
DropCapability=ALL

[Service]
Restart=on-failure
RestartSec=10
MemoryMax=3G

[Install]
WantedBy=default.target
```

```bash
podman network create jellyfin-internal --internal
systemctl --user daemon-reload
systemctl --user start jellyfin.service
```

---

## Le sujet technique : le transcodage matériel

Le seul vrai défi de Jellyfin. Quand un client ne peut pas lire un format nativement, Jellyfin **transcode** à la volée — coûteux en CPU. Le transcodage matériel (GPU) décharge le CPU mais demande d'exposer le GPU au conteneur.

> [!warning] GPU en rootless : un compromis
> Accéder au GPU (`/dev/dri` pour Intel/AMD VAAPI) depuis un conteneur rootless demande que ton utilisateur ait les droits sur le device (groupe `render`/`video`) :
> ```ini
> AddDevice=/dev/dri/renderD128
> ```
> Vérifie l'appartenance au bon groupe côté hôte. Le transcodage NVIDIA (NVENC) est plus complexe en rootless (nvidia-container-toolkit) — teste selon ton matériel. Si ton CPU encaisse le transcodage logiciel pour ton nombre de flux simultanés, tu peux t'en passer et garder l'isolation maximale.

> [!tip] Éviter le transcodage plutôt que l'optimiser
> Le meilleur transcodage est celui qu'on ne fait pas. Si tes clients (TV, app mobile) lisent nativement tes formats (H.264/H.265 en MP4/MKV), Jellyfin sert le fichier directement (*direct play*), zéro transcodage, zéro GPU nécessaire. Adapter ta médiathèque aux formats compatibles est souvent plus efficace que d'ajouter du GPU.

---

## Exposition

```caddyfile
media.mondomaine.fr {
	import security_headers
	reverse_proxy jellyfin:8096
}
```

> [!info] Public ou VPN ?
> Jellyfin est souvent exposé publiquement pour partager avec la famille hors domicile. C'est défendable (les flux ne sont pas des données sensibles), mais ça reste une surface : garde les mises à jour à jour, active des comptes utilisateurs distincts avec mots de passe forts, et envisage Authelia/Authentik en façade si tu exposes. Pour un usage strictement personnel, le VPN suffit. Évite `request_body` illimité ; le streaming gère ses propres flux.

---

## Sauvegarde

> [!success] Quoi sauvegarder
> - **`~/jellyfin/config`** : la base, les comptes, les réglages, l'historique de visionnage. C'est tout ce qui compte.
> - **PAS le cache** (`~/jellyfin/cache`) : reconstructible.
> - **PAS `/media`** : ta médiathèque a sa propre sauvegarde.

---

## Conclusion

Jellyfin est l'illustration parfaite du pattern `/media` en lecture seule : il enrichit l'expérience de ta médiathèque sans jamais pouvoir l'abîmer. Le seul arbitrage réel est le transcodage matériel, où l'accès GPU s'échange contre un peu d'isolation — à n'accorder que si le CPU ne suffit pas.

> [!note] Articles liés
> - [[Installer Immich sous Podman rootless]]
> - [[Installer Navidrome sous Podman rootless]]
> - [[Stratégie de sauvegarde restic 3-2-1]]

---

*Retour d'expérience personnel. Adapte chemins, domaines et accès GPU à ton matériel.*
