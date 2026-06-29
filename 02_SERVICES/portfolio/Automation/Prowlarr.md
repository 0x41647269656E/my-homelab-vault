---
type: service
category: Automation
name: Prowlarr
slug: prowlarr
logo: /assets/logos/prowlarr.png
status: active
integration_status: integrated
opensource: true
docker: true
github_url: https://github.com/Prowlarr/Prowlarr
external_url: https://prowlarr.com
port: 9696
protocol: http
stack:
  - automation
roles:
  - indexers
integrates_with:
  - Sonarr
  - Radarr
  - Lidarr
  - Readarr
  - Bazarr
tags:
  - homelab
  - prowlarr
author: "0x41647269656E"
license: (via badge)
date: 09-01-2026
last_modified: 09-01-2026
created: (unknown)
---

# Prowlarr
![logo|120](/assets/logos/prowlarr.png)

![Stars](https://img.shields.io/github/stars/Prowlarr/Prowlarr?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/Prowlarr/Prowlarr?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/Prowlarr/Prowlarr?style=for-the-badge) ![License](https://img.shields.io/github/license/Prowlarr/Prowlarr?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/Prowlarr/Prowlarr?style=for-the-badge)

## 🧠 Description
**Prowlarr** Gestion centralisée des indexers (torrent/usenet) pour l’ensemble des *Arr.

## ⚙️ Fonctions principales
- 🔎 Ajout & gestion d’indexers
- 🧠 Sync vers Sonarr/Radarr/etc.
- 🔐 Support API keys
- 📊 Santé des indexers
- 🧩 API & UI centralisées

## 🔗 Intégrations
- [[Sonarr]]
- [[Radarr]]
- [[Lidarr]]
- [[Readarr]]
- [[Bazarr]]

## 🧬 Flux de données
```mermaid
graph LR
Prowlarr --> Sonarr
Prowlarr --> Radarr
Prowlarr --> Lidarr
Prowlarr --> Readarr
Prowlarr --> Bazarr
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Accès réseau aux services intégrés (ex: [[Prowlarr]], [[Transmission]], [[Jellyfin]]…)
- (Optionnel) Stockage persistant (NAS, ZFS, etc.)

## Docker-compose
```yaml
services:
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
    ports:
      - "9696:9696"
    volumes:
      - ./prowlarr/config:/config
      - ./prowlarr/media:/data
```

# Liens externes
- GitHub : https://github.com/Prowlarr/Prowlarr
- Site : https://prowlarr.com

# Notes
- À compléter (bonnes pratiques, profils TRaSH, hardlinks, permissions, etc.)

# Avis de l'auteur
- À compléter
