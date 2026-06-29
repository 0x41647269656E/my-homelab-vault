---
type: service
category: Monitoring
name: Tautulli
slug: tautulli
logo: /assets/logos/tautulli.png
status: active
integration_status: integrated
opensource: true
docker: true
github_url: https://github.com/Tautulli/Tautulli
external_url: https://tautulli.com
port: 8181
protocol: http
stack:
  - monitoring
roles:
  - analytics
integrates_with:
  - Plex Media Server
tags:
  - homelab
  - tautulli
author: "0x41647269656E"
license: (via badge)
date: 09-01-2026
last_modified: 09-01-2026
created: (unknown)
---

# Tautulli
![logo|120](/assets/logos/tautulli.png)

![Stars](https://img.shields.io/github/stars/Tautulli/Tautulli?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/Tautulli/Tautulli?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/Tautulli/Tautulli?style=for-the-badge) ![License](https://img.shields.io/github/license/Tautulli/Tautulli?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/Tautulli/Tautulli?style=for-the-badge)

## 🧠 Description
**Tautulli** Statistiques, monitoring et historique de lecture pour Plex Media Server.

## ⚙️ Fonctions principales
- 📊 Stats Plex
- ⏱️ Historique de lecture
- 👥 Activité utilisateurs
- 📣 Notifications
- 🧩 API

## 🔗 Intégrations
- [[Plex Media Server]]

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
  tautulli:
    image: lscr.io/linuxserver/tautulli:latest
    container_name: tautulli
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
    ports:
      - "8181:8181"
    volumes:
      - ./tautulli/config:/config
      - ./tautulli/media:/data
```

# Liens externes
- GitHub : https://github.com/Tautulli/Tautulli
- Site : https://tautulli.com

# Notes
- À compléter (bonnes pratiques, profils TRaSH, hardlinks, permissions, etc.)

# Avis de l'auteur
- À compléter
