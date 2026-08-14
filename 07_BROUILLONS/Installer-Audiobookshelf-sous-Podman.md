---
title: Installer Audiobookshelf sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - audiobookshelf
  - podman
  - installation
  - securite
  - media
date: 07-06-2026
last_modified: 07-06-2026
reading-time: 10m
difficulty: tech-enthusiast
status: draft
---

# Installer Audiobookshelf sous Podman rootless

> [!abstract] TL;DR
> Audiobookshelf est un serveur de **livres audio et podcasts** auto-hébergé, avec suivi de progression multi-appareils et apps mobiles dédiées. Conteneur unique. Particularité par rapport aux autres lecteurs média : il distingue les **livres audio** (lecture seule sur `/media`) des **podcasts** qu'il **télécharge** (donc un dossier inscriptible). Installation Podman rootless directe.

> [!info] Prérequis de lecture
> Socle de la série. Pattern média : [[Installer Jellyfin sous Podman rootless]]. Règle `/media` : [[Installer Immich sous Podman rootless]].

---

## Deux types de contenus, deux logiques de montage

> [!success] La distinction Audiobookshelf
> - **Livres audio existants** (`/media/audiobooks`) → **lecture seule** (`:ro,z`). Audiobookshelf les indexe, suit la progression dans sa base, sans toucher aux fichiers.
> - **Podcasts** → Audiobookshelf **télécharge** les épisodes. Il lui faut un dossier **inscriptible**. Je le place sur un volume dédié (`~/audiobookshelf/podcasts`) plutôt que dans `/media`, ou dans un sous-dossier `/media` dédié monté en `keep-id` si tu veux les ranger avec le reste.
> - **Métadonnées et config** → volumes dédiés.

---

## Le Quadlet

```ini
# ~/.config/containers/systemd/audiobookshelf.container
[Unit]
Description=Audiobookshelf

[Container]
Image=ghcr.io/advplyr/audiobookshelf:latest
ContainerName=audiobookshelf
AutoUpdate=registry
Network=audiobookshelf-internal.network

# Config + métadonnées : volumes dédiés inscriptibles
Volume=%h/audiobookshelf/config:/config:U,Z
Volume=%h/audiobookshelf/metadata:/metadata:U,Z

# Livres audio existants : LECTURE SEULE
Volume=/media/audiobooks:/audiobooks:ro,z

# Podcasts téléchargés : INSCRIPTIBLE (volume dédié)
Volume=%h/audiobookshelf/podcasts:/podcasts:U,Z

# Durcissement
NoNewPrivileges=true
DropCapability=ALL

[Service]
Restart=on-failure
RestartSec=10
MemoryMax=1G

[Install]
WantedBy=default.target
```

```bash
podman network create audiobookshelf-internal --internal
systemctl --user daemon-reload
systemctl --user start audiobookshelf.service
```

---

## Exposition

```caddyfile
books.mondomaine.fr {
	import security_headers
	reverse_proxy audiobookshelf:80
}
```

> [!info] Apps mobiles et exposition
> Les apps mobiles Audiobookshelf (iOS/Android) sont l'argument principal du projet : écouter en déplacement, télécharger pour l'hors-ligne. Comme pour Navidrome, ça pousse vers l'exposition publique. Mon arbitrage habituel : VPN si l'écoute nomade est occasionnelle ; exposition HTTPS avec comptes dédiés si c'est un usage quotidien partagé.

---

## Sauvegarde

> [!success] Quoi sauvegarder
> - **`~/audiobookshelf/config`** : base, comptes, **progression de lecture** (le plus précieux — perdre où tu en étais dans 40 h de livre audio est rageant).
> - **`~/audiobookshelf/metadata`** : couvertures, métadonnées. Reconstructible mais long.
> - **`~/audiobookshelf/podcasts`** : épisodes téléchargés, si tu veux les conserver.
> - **PAS `/media/audiobooks`** : sauvegardé par ailleurs.

---

## Conclusion

Audiobookshelf reprend le pattern média lecture seule, avec la nuance des **podcasts inscriptibles** : un service peut très bien lire ses sources en `:ro` tout en ayant, à côté, un dossier de travail inscriptible. La progression de lecture dans sa base est la donnée à ne pas perdre.

> [!note] Articles liés
> - [[Installer Navidrome sous Podman rootless]]
> - [[Installer Calibre-Web sous Podman rootless]]

---

*Retour d'expérience personnel. Adapte chemins et domaines.*
