---
type: service
category: Media Streaming
name: Plex Media Server
slug: plex
logo: /assets/logos/plex.png
status: active
integration_status: planned
opensource: false
docker: true
github_url: https://github.com/plexinc/pms-docker
external_url: https://www.plex.tv/
port: 32400
protocol: http
stack:
  - media
roles:
  - streaming
integrates_with:
  - Overseerr
  - Tautulli
tags:
  - homelab
  - plex
author: "0x41647269656E"
license: (via badge)
date: 09-01-2026
last_modified: 09-01-2026
created: (unknown)
---

# Plex Media Server
![logo|120](/assets/logos/plex.png)

![Stars](https://img.shields.io/github/stars/plexinc/pms-docker?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/plexinc/pms-docker?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/plexinc/pms-docker?style=for-the-badge) ![License](https://img.shields.io/github/license/plexinc/pms-docker?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/plexinc/pms-docker?style=for-the-badge)

## 🧠 Description
**Plex Media Server** serveur multimédia propriétaire (freemium) pour streamer et gérer films/séries/musique, avec clients très répandus.

## ⚙️ Fonctions principales
- 📺 Streaming multi-clients
- 🎞️ Transcodage
- 📚 Métadonnées
- 👥 Gestion utilisateurs
- 🔌 Apps/clients

## 🔗 Intégrations
- [[Overseerr]]
- [[Tautulli]]

## 🧬 Flux de données
```mermaid
graph LR
Library[Media Library] --> Plex
Plex --> Clients[Apps]
Tautulli --> Plex
Overseerr --> Plex
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Accès au matériel si nécessaire (ex: USB, GPIO, etc.)
- Volumes persistants (recommandé)

## Docker-compose
```yaml
services:
  plex:
    image: lscr.io/linuxserver/plex:latest
    container_name: plex
    restart: unless-stopped
    network_mode: host
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
      # Claim (optionnel) : https://www.plex.tv/claim/
      - PLEX_CLAIM=${PLEX_CLAIM:-}
    volumes:
      - ./plex/config:/config
      - ./plex/media:/media
      - ./plex/transcode:/transcode
```

# Liens externes
- GitHub : https://github.com/plexinc/pms-docker
- Site : https://www.plex.tv/

# Notes
- À compléter

# Avis de l'auteur
- À compléter
