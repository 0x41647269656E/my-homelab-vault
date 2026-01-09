---
type: service
category: Communication (emails, notifs, push, rss, irc, xmpp)
name: Notifiarr
slug: notifiarr
logo: /assets/logos/notifiarr.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/Notifiarr/notifiarr
external_url: https://notifiarr.com/
port: 5454
protocol: http
stack:
  - communication
roles:
  - notifications
integrates_with:
  - Sonarr
  - Radarr
  - Uptime Kuma
  - Watchtower
tags:
  - homelab
  - notifiarr
author: adrientanaka
license: (via badge)
created: (unknown)
---

# Notifiarr
![logo|120](/assets/logos/notifiarr.png)

![Stars](https://img.shields.io/github/stars/Notifiarr/notifiarr?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/Notifiarr/notifiarr?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/Notifiarr/notifiarr?style=for-the-badge) ![License](https://img.shields.io/github/license/Notifiarr/notifiarr?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/Notifiarr/notifiarr?style=for-the-badge)

## 🧠 Description
**Notifiarr** hub de notifications (souvent Discord) et d’intégrations pour les *Arr et l’infra.

## ⚙️ Fonctions principales
- 📣 Notifications centralisées
- 🔗 Intégrations *Arr
- 📊 Monitoring léger
- ⚙️ Webhooks
- 🧩 Utilitaire homelab

## 🔗 Intégrations
- [[Sonarr]]
- [[Radarr]]
- [[Uptime Kuma]]
- [[Watchtower]]

## 🧬 Flux de données
```mermaid
graph LR
Arr[*Arr] --> Notifiarr --> Discord[Discord]
Uptime[Uptime Kuma] --> Notifiarr
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Volumes persistants (recommandé)
- (Optionnel) DNS / certificats / authentification selon usage

## Docker-compose
```yaml
services:
  notifiarr:
    image: golift/notifiarr:latest
    container_name: notifiarr
    restart: unless-stopped
    environment:
      - TZ=Europe/Paris
      # Token Notifiarr (Discord) : à générer côté Notifiarr.com
      - DN_API_KEY=${NOTIFIARR_API_KEY:-CHANGE_ME_API_KEY}
    ports:
      - "5454:5454"
    volumes:
      - ./notifiarr/config:/config
```

# Liens externes
- GitHub : https://github.com/Notifiarr/notifiarr
- Site : https://notifiarr.com/

# Notes
- À compléter

# Avis de l'auteur
- À compléter
