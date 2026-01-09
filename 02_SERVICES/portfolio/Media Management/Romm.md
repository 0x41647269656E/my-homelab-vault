---
type: service
category: Media Management
name: Romm
slug: romm
logo: /assets/logos/romm.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/rommapp/romm
external_url: https://romm.app
port: 3003
protocol: http
stack:
  - media
roles:
  - rom-library
integrates_with:
  - Dashy
tags:
  - homelab
  - romm
author: adrientanaka
license: (via badge)
created: (unknown)
---

# Romm
![logo|120](/assets/logos/romm.png)

![Stars](https://img.shields.io/github/stars/rommapp/romm?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/rommapp/romm?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/rommapp/romm?style=for-the-badge) ![License](https://img.shields.io/github/license/rommapp/romm?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/rommapp/romm?style=for-the-badge)

## 🧠 Description
**Romm** gestionnaire de bibliothèque de ROMs (métadonnées, scan, organisation) pour retrogaming.

## ⚙️ Fonctions principales
- 🎮 Gestion ROMs
- 🏷️ Métadonnées
- 🔎 Recherche
- 📁 Organisation
- 🧩 UI web

## 🔗 Intégrations
- [[Dashy]]

## 🧬 Flux de données
```mermaid
graph LR
ROMs[ROMs] --> Romm --> Metadata[Metadata]
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Volumes persistants (recommandé)
- (Optionnel) DNS / certificats / authentification selon usage

## Docker-compose
```yaml
services:
  romm:
    image: rommapp/romm:latest
    container_name: romm
    restart: unless-stopped
    ports:
      - "3003:3000"
    environment:
      - TZ=Europe/Paris
      # Génère un secret : openssl rand -hex 32
      - ROMM_SECRET_KEY=${ROMM_SECRET_KEY:-CHANGE_ME_SECRET_KEY}
    volumes:
      - ./romm/config:/config
      - ./romm/library:/library
      - ./romm/assets:/assets
```

# Liens externes
- GitHub : https://github.com/rommapp/romm
- Site : https://romm.app

# Notes
- À compléter

# Avis de l'auteur
- À compléter
