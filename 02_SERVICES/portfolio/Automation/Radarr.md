---
type: service
category: Automation
name: Radarr
slug: radarr
logo: /assets/logos/radarr.png
status: active
integration_status: integrated
opensource: true
docker: true
github_url: https://github.com/Radarr/Radarr
external_url: https://radarr.video
port: 7878
protocol: http
stack:
  - media
roles:
  - movies
integrates_with:
  - Prowlarr
  - Jellyfin
  - Transmission
tags:
  - homelab
  - radarr
author: adrientanaka
license: (via badge)
created: (unknown)
---

# Radarr
![logo|120](/assets/logos/radarr.png)

![Stars](https://img.shields.io/github/stars/Radarr/Radarr?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/Radarr/Radarr?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/Radarr/Radarr?style=for-the-badge) ![License](https://img.shields.io/github/license/Radarr/Radarr?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/Radarr/Radarr?style=for-the-badge)

## 🧠 Description
**Radarr** Gestionnaire de films automatisé (monitoring, téléchargement, renommage, organisation).

## ⚙️ Fonctions principales
- 🎬 Suivi de films
- 📂 Renommage/organisation
- 🔄 Upgrade qualité
- 🔗 Intégration indexers + downloaders
- 🧩 API complète

## 🔗 Intégrations
- [[Prowlarr]]
- [[Jellyfin]]
- [[Transmission]]

## 🧬 Flux de données
```mermaid
graph LR
Requests[Overseerr/Jellyseerr] --> ARR[Radarr]
ARR --> Prowlarr --> Indexers[Indexers]
ARR --> Downloader[Transmission]
Downloader --> Library[Media Library] --> Player[Jellyfin/Plex]
Bazarr --> Library
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Accès réseau aux services intégrés (ex: [[Prowlarr]], [[Transmission]], [[Jellyfin]]…)
- (Optionnel) Stockage persistant (NAS, ZFS, etc.)

## Docker-compose
```yaml
services:
  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
    ports:
      - "7878:7878"
    volumes:
      - ./radarr/config:/config
      - ./radarr/media:/data
```

# Liens externes
- GitHub : https://github.com/Radarr/Radarr
- Site : https://radarr.video

# Notes
- À compléter (bonnes pratiques, profils TRaSH, hardlinks, permissions, etc.)

# Avis de l'auteur
- À compléter
