---
type: service
category: Automation
name: Bazarr
slug: bazarr
logo: /assets/logos/bazarr.png
status: active
integration_status: integrated
opensource: true
docker: true
github_url: https://github.com/morpheus65535/bazarr
external_url: https://www.bazarr.media
port: 6767
protocol: http
stack:
  - media
roles:
  - subtitles
integrates_with:
  - Sonarr
  - Radarr
tags:
  - homelab
  - bazarr
author: "0x41647269656E"
license: (via badge)
date: 09-01-2026
last_modified: 09-01-2026
created: (unknown)
---

# Bazarr
![logo|120](/assets/logos/bazarr.png)

![Stars](https://img.shields.io/github/stars/morpheus65535/bazarr?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/morpheus65535/bazarr?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/morpheus65535/bazarr?style=for-the-badge) ![License](https://img.shields.io/github/license/morpheus65535/bazarr?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/morpheus65535/bazarr?style=for-the-badge)

## 🧠 Description
**Bazarr** Téléchargement et gestion automatique des sous-titres, intégré à Sonarr/Radarr.

## ⚙️ Fonctions principales
- 💬 Recherche sous-titres
- 🔄 Synchronisation bibliothèques
- 🌍 Multi-fournisseurs
- 🧩 API
- ⚙️ Profilage langues

## 🔗 Intégrations
- [[Sonarr]]
- [[Radarr]]

## 🧬 Flux de données
```mermaid
graph LR
Sonarr --> Bazarr
Radarr --> Bazarr
Bazarr --> Subtitles[Providers de sous-titres]
Bazarr --> Library[Media Library]
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Accès réseau aux services intégrés (ex: [[Prowlarr]], [[Transmission]], [[Jellyfin]]…)
- (Optionnel) Stockage persistant (NAS, ZFS, etc.)

## Docker-compose
```yaml
services:
  bazarr:
    image: lscr.io/linuxserver/bazarr:latest
    container_name: bazarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
    ports:
      - "6767:6767"
    volumes:
      - ./bazarr/config:/config
      - ./bazarr/media:/data
```

# Liens externes
- GitHub : https://github.com/morpheus65535/bazarr
- Site : https://www.bazarr.media

# Notes
- À compléter (bonnes pratiques, profils TRaSH, hardlinks, permissions, etc.)

# Avis de l'auteur
- À compléter
