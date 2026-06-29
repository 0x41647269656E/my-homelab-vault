---
type: service
category: Monitoring
name: Jellystat
slug: jellystat
logo: /assets/logos/jellystat.png
status: active
integration_status: integrated
opensource: true
docker: true
github_url: https://github.com/CyferShepard/Jellystat
external_url: https://github.com/CyferShepard/Jellystat
port: 3000
protocol: http
stack:
  - monitoring
roles:
  - analytics
integrates_with:
  - Jellyfin
tags:
  - homelab
  - jellystat
author: "0x41647269656E"
license: (via badge)
date: 09-01-2026
last_modified: 09-01-2026
created: (unknown)
---

# Jellystat
![logo|120](/assets/logos/jellystat.png)

![Stars](https://img.shields.io/github/stars/CyferShepard/Jellystat?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/CyferShepard/Jellystat?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/CyferShepard/Jellystat?style=for-the-badge) ![License](https://img.shields.io/github/license/CyferShepard/Jellystat?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/CyferShepard/Jellystat?style=for-the-badge)

## 🧠 Description
**Jellystat** Statistiques et analytics pour Jellyfin (équivalent “Tautulli” côté Jellyfin).

## ⚙️ Fonctions principales
- 📊 Stats utilisateurs
- ⏱️ Historique de lecture
- 📈 Tableaux/graphes
- 🟢 Activité
- 🧩 API

## 🔗 Intégrations
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
  jellystat:
    image: cyfershepard/jellystat:latest
    container_name: jellystat
    restart: unless-stopped
    environment:
      - TZ=Europe/Paris
    ports:
      - "3000:3000"
    volumes:
      - ./jellystat/data:/app/backend/data
```

# Liens externes
- GitHub : https://github.com/CyferShepard/Jellystat
- Site : https://github.com/CyferShepard/Jellystat

# Notes
- À compléter (bonnes pratiques, profils TRaSH, hardlinks, permissions, etc.)

# Avis de l'auteur
- À compléter
