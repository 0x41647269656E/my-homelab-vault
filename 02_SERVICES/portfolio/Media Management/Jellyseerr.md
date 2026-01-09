---
type: service
category: Media Management
name: Jellyseerr
slug: jellyseerr
logo: /assets/logos/jellyseerr.png
status: active
integration_status: integrated
opensource: true
docker: true
github_url: https://github.com/Fallenbagel/jellyseerr
external_url: https://github.com/Fallenbagel/jellyseerr
port: 5055
protocol: http
stack:
  - media
roles:
  - requests
integrates_with:
  - Sonarr
  - Radarr
  - Jellyfin
tags:
  - homelab
  - jellyseerr
author: adrientanaka
license: (via badge)
created: (unknown)
---

# Jellyseerr
![logo|120](/assets/logos/jellyseerr.png)

![Stars](https://img.shields.io/github/stars/Fallenbagel/jellyseerr?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/Fallenbagel/jellyseerr?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/Fallenbagel/jellyseerr?style=for-the-badge) ![License](https://img.shields.io/github/license/Fallenbagel/jellyseerr?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/Fallenbagel/jellyseerr?style=for-the-badge)

## 🧠 Description
**Jellyseerr** Portail de demandes pour Jellyfin/Emby, connecté à Sonarr/Radarr.

## ⚙️ Fonctions principales
- 🙋 Portail demandes
- 👥 Comptes utilisateurs (SSO possible)
- 🔗 Envoi vers Sonarr/Radarr
- 📣 Notifications
- 🧩 API

## 🔗 Intégrations
- [[Sonarr]]
- [[Radarr]]
- [[Jellyfin]]

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
  jellyseerr:
    image: fallenbagel/jellyseerr:latest
    container_name: jellyseerr
    restart: unless-stopped
    environment:
      - TZ=Europe/Paris
    ports:
      - "5055:5055"
    volumes:
      - ./jellyseerr/config:/app/config
```

# Liens externes
- GitHub : https://github.com/Fallenbagel/jellyseerr
- Site : https://github.com/Fallenbagel/jellyseerr

# Notes
- À compléter (bonnes pratiques, profils TRaSH, hardlinks, permissions, etc.)

# Avis de l'auteur
- À compléter
