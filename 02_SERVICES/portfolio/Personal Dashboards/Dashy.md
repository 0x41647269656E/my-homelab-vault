---
type: service
category: Personal Dashboards
name: Dashy
slug: dashy
logo: /assets/logos/dashy.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/Lissy93/dashy
external_url: https://dashy.to/
port: 8080
protocol: http
stack:
  - ux
roles:
  - dashboard
integrates_with:
  - Portainer
  - Grafana
  - Uptime Kuma
tags:
  - homelab
  - dashy
author: adrientanaka
license: (via badge)
created: (unknown)
---

# Dashy
![logo|120](/assets/logos/dashy.png)

![Stars](https://img.shields.io/github/stars/Lissy93/dashy?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/Lissy93/dashy?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/Lissy93/dashy?style=for-the-badge) ![License](https://img.shields.io/github/license/Lissy93/dashy?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/Lissy93/dashy?style=for-the-badge)

## 🧠 Description
**Dashy** dashboard personnel auto-hébergé pour centraliser les liens et widgets de ton homelab.

## ⚙️ Fonctions principales
- 🧩 Widgets
- 🔎 Recherche
- 🗂️ Sections/tiles
- 🔐 Auth (selon config)
- 🎨 Thèmes

## 🔗 Intégrations
- [[Portainer]]
- [[Grafana]]
- [[Uptime Kuma]]

## 🧬 Flux de données
```mermaid
graph LR
User[Utilisateur] --> Dashy
Dashy --> Links[Services du homelab]
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Volumes persistants (recommandé)
- (Optionnel) DNS / certificats / authentification selon usage

## Docker-compose
```yaml
services:
  dashy:
    image: lissy93/dashy:latest
    container_name: dashy
    restart: unless-stopped
    environment:
      - TZ=Europe/Paris
      # (Optionnel) Auth basique via env si souhaité
    ports:
      - "8080:8080"
    volumes:
      - ./dashy/conf.yml:/app/public/conf.yml:ro
```

# Liens externes
- GitHub : https://github.com/Lissy93/dashy
- Site : https://dashy.to/

# Notes
- À compléter

# Avis de l'auteur
- À compléter
