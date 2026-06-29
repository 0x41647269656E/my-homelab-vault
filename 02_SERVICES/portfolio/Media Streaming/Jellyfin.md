---
type: service
category: Media Streaming
name: Jellyfin
slug: jellyfin
logo: /assets/logos/jellyfin.png
status: active
integration_status: integrated
opensource: true
docker: true
github_url: https://github.com/jellyfin/jellyfin
external_url: https://jellyfin.org
port: 8096
protocol: http
stack:
  - media
roles:
  - streaming
integrates_with:
  - Sonarr
  - Radarr
  - Bazarr
  - Jellystat
tags:
  - homelab
  - jellyfin
author: "0x41647269656E"
license: (via badge)
date: 09-01-2026
last_modified: 09-01-2026
created: (unknown)
---

# Jellyfin
![logo|120](/assets/logos/jellyfin.png)

![Stars](https://img.shields.io/github/stars/jellyfin/jellyfin?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/jellyfin/jellyfin?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/jellyfin/jellyfin?style=for-the-badge) ![License](https://img.shields.io/github/license/jellyfin/jellyfin?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/jellyfin/jellyfin?style=for-the-badge)

## 🧠 Description
**Jellyfin** Serveur multimédia open-source pour streamer films, séries, musique et photos.

## ⚙️ Fonctions principales
- 📺 Streaming multi-clients
- 👤 Comptes & accès
- 🎞️ Transcodage
- 📚 Métadonnées
- 🔌 Plugins

## 🔗 Intégrations
- [[Sonarr]]
- [[Radarr]]
- [[Bazarr]]
- [[Jellystat]]

## 🧬 Flux de données
```mermaid
graph LR
A[Service] --> B[À compléter]
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Accès réseau aux services intégrés (ex: [[Prowlarr]], [[Transmission]], [[Jellyfin]]…)
- (Optionnel) Stockage persistant (NAS, ZFS, etc.)

## Docker-compose
```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    user: "1000:1000"
    ports:
      - "8096:8096"
    volumes:
      - ./jellyfin/config:/config
      - ./jellyfin/cache:/cache
      - ./jellyfin/media:/media
    environment:
      - TZ=Europe/Paris
```

# Liens externes
- GitHub : https://github.com/jellyfin/jellyfin
- Site : https://jellyfin.org

# Notes
- À compléter (bonnes pratiques, profils TRaSH, hardlinks, permissions, etc.)

# Avis de l'auteur
- À compléter
