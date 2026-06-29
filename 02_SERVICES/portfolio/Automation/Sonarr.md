---
type: service
category: Automation
name: Sonarr
slug: sonarr
logo: /assets/logos/sonarr.png
status: active
integration_status: integrated
opensource: true
docker: true
github_url: https://github.com/Sonarr/Sonarr
external_url: https://sonarr.tv
port: 8989
protocol: http
stack:
  - media
roles:
  - series
integrates_with:
  - Prowlarr
  - Bazarr
  - Jellyfin
  - Transmission
tags:
  - homelab
  - sonarr
author: "0x41647269656E"
license: (via badge)
date: 09-01-2026
last_modified: 09-01-2026
created: (unknown)
---

# Sonarr
![logo|120](/assets/logos/sonarr.png)

![Stars](https://img.shields.io/github/stars/Sonarr/Sonarr?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/Sonarr/Sonarr?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/Sonarr/Sonarr?style=for-the-badge) ![License](https://img.shields.io/github/license/Sonarr/Sonarr?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/Sonarr/Sonarr?style=for-the-badge)

## 🧠 Description
**Sonarr** Gestionnaire de séries TV automatisé (monitoring, téléchargement, renommage, organisation).

## ⚙️ Fonctions principales
- 📡 Recherche & suivi d’épisodes
- 📂 Renommage/organisation
- 🔄 Upgrade qualité automatique
- 🔗 Intégration indexers + downloaders
- 🧩 API complète

## 🔗 Intégrations
- [[Prowlarr]]
- [[Bazarr]]
- [[Jellyfin]]
- [[Transmission]]

## 🧬 Flux de données
```mermaid
graph LR
Requests[Overseerr/Jellyseerr] --> ARR[Sonarr]
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
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
    ports:
      - "8989:8989"
    volumes:
      - ./sonarr/config:/config
      - ./sonarr/media:/data
```

# Liens externes
- GitHub : https://github.com/Sonarr/Sonarr
- Site : https://sonarr.tv

# Notes
- À compléter (bonnes pratiques, profils TRaSH, hardlinks, permissions, etc.)

# Avis de l'auteur
- À compléter
