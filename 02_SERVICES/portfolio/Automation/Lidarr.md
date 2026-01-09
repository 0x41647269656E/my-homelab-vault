---
type: service
category: Automation
name: Lidarr
slug: lidarr
logo: /assets/logos/lidarr.png
status: active
integration_status: integrated
opensource: true
docker: true
github_url: https://github.com/Lidarr/Lidarr
external_url: https://lidarr.audio
port: 8686
protocol: http
stack:
  - media
roles:
  - music
integrates_with:
  - Prowlarr
  - Transmission
tags:
  - homelab
  - lidarr
author: adrientanaka
license: (via badge)
created: (unknown)
---

# Lidarr
![logo|120](/assets/logos/lidarr.png)

![Stars](https://img.shields.io/github/stars/Lidarr/Lidarr?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/Lidarr/Lidarr?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/Lidarr/Lidarr?style=for-the-badge) ![License](https://img.shields.io/github/license/Lidarr/Lidarr?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/Lidarr/Lidarr?style=for-the-badge)

## 🧠 Description
**Lidarr** Gestionnaire de musique automatisé (monitoring, téléchargement, tagging/organisation).

## ⚙️ Fonctions principales
- 🎵 Suivi artistes/albums
- 🏷️ Tagging/organisation
- 🔄 Upgrade qualité
- 🔗 Indexers + downloaders
- 🧩 API complète

## 🔗 Intégrations
- [[Prowlarr]]
- [[Transmission]]

## 🧬 Flux de données
```mermaid
graph LR
Requests[Overseerr/Jellyseerr] --> ARR[Lidarr]
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
  lidarr:
    image: lscr.io/linuxserver/lidarr:latest
    container_name: lidarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
    ports:
      - "8686:8686"
    volumes:
      - ./lidarr/config:/config
      - ./lidarr/media:/data
```

# Liens externes
- GitHub : https://github.com/Lidarr/Lidarr
- Site : https://lidarr.audio

# Notes
- À compléter (bonnes pratiques, profils TRaSH, hardlinks, permissions, etc.)

# Avis de l'auteur
- À compléter
