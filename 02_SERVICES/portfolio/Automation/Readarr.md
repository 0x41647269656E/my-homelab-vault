---
type: service
category: Automation
name: Readarr
slug: readarr
logo: /assets/logos/readarr.png
status: active
integration_status: integrated
opensource: true
docker: true
github_url: https://github.com/Readarr/Readarr
external_url: https://readarr.com
port: 8787
protocol: http
stack:
  - media
roles:
  - books
integrates_with:
  - Prowlarr
  - Transmission
tags:
  - homelab
  - readarr
author: adrientanaka
license: (via badge)
created: (unknown)
---

# Readarr
![logo|120](/assets/logos/readarr.png)

![Stars](https://img.shields.io/github/stars/Readarr/Readarr?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/Readarr/Readarr?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/Readarr/Readarr?style=for-the-badge) ![License](https://img.shields.io/github/license/Readarr/Readarr?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/Readarr/Readarr?style=for-the-badge)

## 🧠 Description
**Readarr** Gestionnaire de livres/ebooks/audiobooks automatisé (monitoring, téléchargement, organisation).

## ⚙️ Fonctions principales
- 📚 Suivi auteurs/séries
- 📂 Organisation ebooks/audiobooks
- 🔄 Upgrade qualité
- 🔗 Indexers + downloaders
- 🧩 API complète

## 🔗 Intégrations
- [[Prowlarr]]
- [[Transmission]]

## 🧬 Flux de données
```mermaid
graph LR
Requests[Overseerr/Jellyseerr] --> ARR[Readarr]
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
  readarr:
    image: lscr.io/linuxserver/readarr:latest
    container_name: readarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
    ports:
      - "8787:8787"
    volumes:
      - ./readarr/config:/config
      - ./readarr/media:/data
```

# Liens externes
- GitHub : https://github.com/Readarr/Readarr
- Site : https://readarr.com

# Notes
- À compléter (bonnes pratiques, profils TRaSH, hardlinks, permissions, etc.)

# Avis de l'auteur
- À compléter
