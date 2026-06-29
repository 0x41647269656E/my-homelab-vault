---
type: service
category: Media Streaming
name: Jellyfin-webhook-plugin
slug: jellyfin-webhook-plugin
logo: /assets/logos/jellyfin-webhook-plugin.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/jellyfin/jellyfin-plugin-webhook
external_url: https://github.com/jellyfin/jellyfin-plugin-webhook
port: 0
protocol: 
stack:
  - media
roles:
  - plugin
integrates_with:
  - Jellyfin
  - Notifiarr
tags:
  - homelab
  - jellyfin-webhook-plugin
author: "0x41647269656E"
license: (via badge)
date: 09-01-2026
last_modified: 09-01-2026
created: (unknown)
---

# Jellyfin-webhook-plugin
![logo|120](/assets/logos/jellyfin-webhook-plugin.png)

![Stars](https://img.shields.io/github/stars/jellyfin/jellyfin-plugin-webhook?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/jellyfin/jellyfin-plugin-webhook?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/jellyfin/jellyfin-plugin-webhook?style=for-the-badge) ![License](https://img.shields.io/github/license/jellyfin/jellyfin-plugin-webhook?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/jellyfin/jellyfin-plugin-webhook?style=for-the-badge)

## 🧠 Description
**Jellyfin-webhook-plugin** plugin Jellyfin permettant d’émettre des webhooks (Discord, HTTP, etc.) sur des événements de lecture.

## ⚙️ Fonctions principales
- 🔔 Webhooks événements Jellyfin
- 🌐 HTTP/Discord
- ⚙️ Templates
- 🧩 Installation via catalogue plugins
- 🧾 Debug/logs

## 🔗 Intégrations
- [[Jellyfin]]
- [[Notifiarr]]

## 🧬 Flux de données
```mermaid
graph LR
Jellyfin --> WebhookPlugin[Webhook Plugin] --> Targets[Discord/Webhooks]
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Accès au matériel si nécessaire (ex: USB, GPIO, etc.)
- Volumes persistants (recommandé)

## Docker-compose
```yaml
# Jellyfin-webhook-plugin s'installe comme plugin DANS Jellyfin (pas un service à part).
# Ici un exemple de stack Jellyfin + un volume plugins persistant.

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
      - ./jellyfin/plugins:/config/plugins
    environment:
      - TZ=Europe/Paris
```

# Liens externes
- GitHub : https://github.com/jellyfin/jellyfin-plugin-webhook
- Site : https://github.com/jellyfin/jellyfin-plugin-webhook

# Notes
- À compléter

# Avis de l'auteur
- À compléter
