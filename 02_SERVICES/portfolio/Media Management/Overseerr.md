---
type: service
category: Media Management
name: Overseerr
slug: overseerr
logo: /assets/logos/overseerr.png
status: active
integration_status: integrated
opensource: true
docker: true
github_url: https://github.com/sct/overseerr
external_url: https://overseerr.dev
port: 5055
protocol: http
stack:
  - media
roles:
  - requests
integrates_with:
  - Sonarr
  - Radarr
  - Plex Media Server
tags:
  - homelab
  - overseerr
author: "0x41647269656E"
license: (via badge)
date: 09-01-2026
last_modified: 09-01-2026
created: (unknown)
---

# Overseerr
![logo|120](/assets/logos/overseerr.png)

![Stars](https://img.shields.io/github/stars/sct/overseerr?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/sct/overseerr?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/sct/overseerr?style=for-the-badge) ![License](https://img.shields.io/github/license/sct/overseerr?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/sct/overseerr?style=for-the-badge)

## 🧠 Description
**Overseerr** Portail de demandes pour Plex (et parfois Jellyfin), connecté à Sonarr/Radarr.

## ⚙️ Fonctions principales
- 🙋 Portail demandes Plex
- 🔗 Envoi vers Sonarr/Radarr
- 👥 Gestion utilisateurs
- 📣 Notifications
- 🧩 API

## 🔗 Intégrations
- [[Sonarr]]
- [[Radarr]]
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
  overseerr:
    image: sctx/overseerr:latest
    container_name: overseerr
    restart: unless-stopped
    environment:
      - TZ=Europe/Paris
      # Génère une clé de chiffrement : openssl rand -hex 32
      - SECRET_KEY=${OVERSEERR_SECRET_KEY:-CHANGE_ME_SECRET_KEY}
    ports:
      - "5055:5055"
    volumes:
      - ./overseerr/config:/app/config
```

# Liens externes
- GitHub : https://github.com/sct/overseerr
- Site : https://overseerr.dev

# Notes
- À compléter (bonnes pratiques, profils TRaSH, hardlinks, permissions, etc.)

# Avis de l'auteur
- À compléter
