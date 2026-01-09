---
type: service
category: Backup
name: Duplicati
slug: duplicati
logo: /assets/logos/duplicati.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/duplicati/duplicati
external_url: https://www.duplicati.com
port: 8200
protocol: http
stack:
  - backup
roles:
  - backup
integrates_with:
  - Syncthing
tags:
  - homelab
  - duplicati
author: adrientanaka
license: (via badge)
created: (unknown)
---

# Duplicati
![logo|120](/assets/logos/duplicati.png)

![Stars](https://img.shields.io/github/stars/duplicati/duplicati?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/duplicati/duplicati?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/duplicati/duplicati?style=for-the-badge) ![License](https://img.shields.io/github/license/duplicati/duplicati?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/duplicati/duplicati?style=for-the-badge)

## 🧠 Description
**Duplicati** solution de sauvegarde chiffrée avec déduplication (destinations : S3, WebDAV, SSH, etc.).

## ⚙️ Fonctions principales
- 🔒 Chiffrement
- 🧠 Déduplication
- ⏱️ Planification
- ☁️ Destinations multiples
- ♻️ Rétention

## 🔗 Intégrations
- [[Syncthing]]

## 🧬 Flux de données
```mermaid
graph LR
Source[Répertoires] --> Duplicati --> Dest[Cloud/NAS/S3]
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Volumes persistants (recommandé)
- (Optionnel) DNS / certificats / authentification selon usage

## Docker-compose
```yaml
services:
  duplicati:
    image: lscr.io/linuxserver/duplicati:latest
    container_name: duplicati
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
    ports:
      - "8200:8200"
    volumes:
      - ./duplicati/config:/config
      - ./duplicati/backups:/backups
      - ./duplicati/source:/source:ro
```

# Liens externes
- GitHub : https://github.com/duplicati/duplicati
- Site : https://www.duplicati.com

# Notes
- À compléter

# Avis de l'auteur
- À compléter
