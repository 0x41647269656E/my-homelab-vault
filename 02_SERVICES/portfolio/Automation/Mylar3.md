---
type: service
category: Automation
name: Mylar3
slug: mylar3
logo: /assets/logos/mylar3.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/mylar3/mylar3
external_url: https://github.com/mylar3/mylar3
port: 8091
protocol: http
stack:
  - media
roles:
  - comics
integrates_with:
  - Transmission
tags:
  - homelab
  - mylar3
author: adrientanaka
license: (via badge)
created: (unknown)
---

# Mylar3
![logo|120](/assets/logos/mylar3.png)

![Stars](https://img.shields.io/github/stars/mylar3/mylar3?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/mylar3/mylar3?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/mylar3/mylar3?style=for-the-badge) ![License](https://img.shields.io/github/license/mylar3/mylar3?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/mylar3/mylar3?style=for-the-badge)

## 🧠 Description
**Mylar3** gestionnaire automatisé de comics/BD (suivi séries, recherche, téléchargement, organisation).

## ⚙️ Fonctions principales
- 📚 Suivi séries comics
- 🔎 Recherche releases
- ⬇️ Download via client
- 📂 Organisation
- 🧩 API

## 🔗 Intégrations
- [[Transmission]]

## 🧬 Flux de données
```mermaid
graph LR
Mylar3 --> Downloader[Transmission]
Downloader --> Library[Comics Library]
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Accès au matériel si nécessaire (ex: USB, GPIO, etc.)
- Volumes persistants (recommandé)

## Docker-compose
```yaml
services:
  mylar3:
    image: lscr.io/linuxserver/mylar3:latest
    container_name: mylar3
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
    ports:
      - "8091:8090"
    volumes:
      - ./mylar3/config:/config
      - ./mylar3/comics:/comics
      - ./mylar3/downloads:/downloads
```

# Liens externes
- GitHub : https://github.com/mylar3/mylar3
- Site : https://github.com/mylar3/mylar3

# Notes
- À compléter

# Avis de l'auteur
- À compléter
