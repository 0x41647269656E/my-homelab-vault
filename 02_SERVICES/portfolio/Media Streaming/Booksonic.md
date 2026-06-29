---
type: service
category: Media Streaming
name: Booksonic
slug: booksonic
logo: /assets/logos/booksonic.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/linuxserver/docker-booksonic-air
external_url: https://github.com/linuxserver/docker-booksonic-air
port: 4040
protocol: http
stack:
  - media
roles:
  - audiobooks
integrates_with:
  - NGINX
tags:
  - homelab
  - booksonic
author: "0x41647269656E"
license: (via badge)
date: 09-01-2026
last_modified: 09-01-2026
created: (unknown)
---

# Booksonic
![logo|120](/assets/logos/booksonic.png)

![Stars](https://img.shields.io/github/stars/linuxserver/docker-booksonic-air?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/linuxserver/docker-booksonic-air?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/linuxserver/docker-booksonic-air?style=for-the-badge) ![License](https://img.shields.io/github/license/linuxserver/docker-booksonic-air?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/linuxserver/docker-booksonic-air?style=for-the-badge)

## 🧠 Description
**Booksonic** serveur de streaming (compatible Subsonic) pour livres audio/musique, avec clients variés.

## ⚙️ Fonctions principales
- 🎧 Streaming
- 📚 Bibliothèque
- 👥 Utilisateurs
- 📱 Clients Subsonic
- 🗄️ Volumes

## 🔗 Intégrations
- [[NGINX]]

## 🧬 Flux de données
```mermaid
graph LR
Clients[Apps] --> Booksonic --> Library[Books/Music]
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Volumes persistants (recommandé)
- (Optionnel) DNS / certificats / authentification selon usage

## Docker-compose
```yaml
services:
  booksonic-air:
    image: lscr.io/linuxserver/booksonic-air:latest
    container_name: booksonic-air
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
    ports:
      - "4040:4040"
    volumes:
      - ./booksonic/config:/config
      - ./booksonic/books:/books
      - ./booksonic/podcasts:/podcasts
      - ./booksonic/music:/music
```

# Liens externes
- GitHub : https://github.com/linuxserver/docker-booksonic-air
- Site : https://github.com/linuxserver/docker-booksonic-air

# Notes
- À compléter

# Avis de l'auteur
- À compléter
