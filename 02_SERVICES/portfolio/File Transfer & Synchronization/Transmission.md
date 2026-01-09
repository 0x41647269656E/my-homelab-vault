---
type: service
category: File Transfer & Synchronization
name: Transmission
slug: transmission
logo: /assets/logos/transmission.png
status: active
integration_status: integrated
opensource: true
docker: true
github_url: https://github.com/transmission/transmission
external_url: https://transmissionbt.com
port: 9091
protocol: http
stack:
  - download
roles:
  - downloader
integrates_with:
  - Sonarr
  - Radarr
  - Lidarr
  - Readarr
tags:
  - homelab
  - transmission
author: adrientanaka
license: (via badge)
created: (unknown)
---

# Transmission
![logo|120](/assets/logos/transmission.png)

![Stars](https://img.shields.io/github/stars/transmission/transmission?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/transmission/transmission?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/transmission/transmission?style=for-the-badge) ![License](https://img.shields.io/github/license/transmission/transmission?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/transmission/transmission?style=for-the-badge)

## 🧠 Description
**Transmission** Client BitTorrent léger et stable, contrôlable via interface web et RPC.

## ⚙️ Fonctions principales
- ⬇️ Téléchargement torrents
- 🌐 Web UI
- 🧲 Magnet links
- 📁 Watch folder
- 🔌 RPC

## 🔗 Intégrations
- [[Sonarr]]
- [[Radarr]]
- [[Lidarr]]
- [[Readarr]]

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
  transmission:
    image: lscr.io/linuxserver/transmission:latest
    container_name: transmission
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
      - USER=adrientanaka
      # Génère un mot de passe fort : openssl rand -base64 32
      - PASS=${TRANSMISSION_PASS:-CHANGE_ME_STRONG_PASSWORD}
    ports:
      - "9091:9091"
      - "51413:51413"
      - "51413:51413/udp"
    volumes:
      - ./transmission/config:/config
      - ./transmission/downloads:/downloads
      - ./transmission/data:/watch
```

# Liens externes
- GitHub : https://github.com/transmission/transmission
- Site : https://transmissionbt.com

# Notes
- À compléter (bonnes pratiques, profils TRaSH, hardlinks, permissions, etc.)

# Avis de l'auteur
- À compléter
