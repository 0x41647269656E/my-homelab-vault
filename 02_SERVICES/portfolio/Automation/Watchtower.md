---
type: service
category: Automation
name: Watchtower
slug: watchtower
logo: /assets/logos/watchtower.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/containrrr/watchtower
external_url: https://containrrr.dev/watchtower/
port: 0
protocol: 
stack:
  - automation
roles:
  - updates
integrates_with:
  - Notifiarr
tags:
  - homelab
  - watchtower
author: adrientanaka
license: (via badge)
created: (unknown)
---

# Watchtower
![logo|120](/assets/logos/watchtower.png)

![Stars](https://img.shields.io/github/stars/containrrr/watchtower?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/containrrr/watchtower?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/containrrr/watchtower?style=for-the-badge) ![License](https://img.shields.io/github/license/containrrr/watchtower?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/containrrr/watchtower?style=for-the-badge)

## 🧠 Description
**Watchtower** surveille les images Docker et met à jour automatiquement les conteneurs selon un planning.

## ⚙️ Fonctions principales
- 🔄 Auto-update conteneurs
- 🧹 Cleanup images
- ⏱️ Planification
- 📣 Notifications (Shoutrrr)
- ✅ Support tags/filters

## 🔗 Intégrations
- [[Notifiarr]]

## 🧬 Flux de données
```mermaid
graph LR
Registry[Registry] --> Watchtower
Watchtower --> Docker[Docker Engine]
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Volumes persistants (recommandé)
- (Optionnel) DNS / certificats / authentification selon usage

## Docker-compose
```yaml
services:
  watchtower:
    image: containrrr/watchtower:latest
    container_name: watchtower
    restart: unless-stopped
    environment:
      - TZ=Europe/Paris
      # Planifie les updates : tous les jours à 04:00
      - WATCHTOWER_SCHEDULE=0 0 4 * * *
      - WATCHTOWER_CLEANUP=true
      - WATCHTOWER_INCLUDE_RESTARTING=true
      # Notifications (optionnel)
      # - WATCHTOWER_NOTIFICATIONS=shoutrrr
      # - WATCHTOWER_NOTIFICATION_URL=${WATCHTOWER_NOTIFICATION_URL}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

# Liens externes
- GitHub : https://github.com/containrrr/watchtower
- Site : https://containrrr.dev/watchtower/

# Notes
- À compléter

# Avis de l'auteur
- À compléter
