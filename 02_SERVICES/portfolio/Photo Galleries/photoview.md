---
type: service
category: Photo Galleries
name: photoview
slug: photoview
logo: /assets/logos/photoview.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/photoview/photoview
external_url: https://photoview.github.io/
port: 8088
protocol: http
stack:
  - photos
roles:
  - gallery
integrates_with:
  - NGINX
tags:
  - homelab
  - photoview
author: "0x41647269656E"
license: (via badge)
date: 09-01-2026
last_modified: 09-01-2026
created: (unknown)
---

# photoview
![logo|120](/assets/logos/photoview.png)

![Stars](https://img.shields.io/github/stars/photoview/photoview?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/photoview/photoview?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/photoview/photoview?style=for-the-badge) ![License](https://img.shields.io/github/license/photoview/photoview?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/photoview/photoview?style=for-the-badge)

## 🧠 Description
**photoview** galerie photo auto-hébergée avec indexation et interface web légère.

## ⚙️ Fonctions principales
- 🖼️ Galerie web
- 📁 Indexation dossiers
- 🔎 Recherche
- 👥 Partage (selon config)
- 🗄️ Cache

## 🔗 Intégrations
- [[NGINX]]

## 🧬 Flux de données
```mermaid
graph LR
Photos[Répertoire photos] --> PhotoView
PhotoView --> Cache
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Volumes persistants (recommandé)
- (Optionnel) DNS / certificats / authentification selon usage

## Docker-compose
```yaml
services:
  photoview:
    image: viktorstrate/photoview:latest
    container_name: photoview
    restart: unless-stopped
    ports:
      - "8088:80"
    environment:
      - PHOTOVIEW_DATABASE_DRIVER=sqlite
      - PHOTOVIEW_DATABASE_NAME=/database/photoview.db
      - TZ=Europe/Paris
      # (Optionnel) Auth OIDC possible selon configuration
    volumes:
      - ./photoview/database:/database
      - ./photoview/photos:/photos:ro
      - ./photoview/cache:/cache
```

# Liens externes
- GitHub : https://github.com/photoview/photoview
- Site : https://photoview.github.io/

# Notes
- À compléter

# Avis de l'auteur
- À compléter
