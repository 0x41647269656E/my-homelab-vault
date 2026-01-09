---
type: service
category: DNS
name: AdGuard Home
slug: adguard-home
logo: /assets/logos/adguard-home.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/AdguardTeam/AdGuardHome
external_url: https://adguard.com/en/adguard-home/overview.html
port: 3000
protocol: http
stack:
  - network
roles:
  - dns
integrates_with:
  - Uptime Kuma
tags:
  - homelab
  - adguard-home
author: adrientanaka
license: (via badge)
created: (unknown)
---

# AdGuard Home
![logo|120](/assets/logos/adguard-home.png)

![Stars](https://img.shields.io/github/stars/AdguardTeam/AdGuardHome?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/AdguardTeam/AdGuardHome?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/AdguardTeam/AdGuardHome?style=for-the-badge) ![License](https://img.shields.io/github/license/AdguardTeam/AdGuardHome?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/AdguardTeam/AdGuardHome?style=for-the-badge)

## 🧠 Description
**AdGuard Home** DNS filtrant (pubs/trackers) avec interface d’administration, idéal en homelab.

## ⚙️ Fonctions principales
- 🛡️ Blocage pubs/trackers
- 🧾 Logs DNS
- 📋 Listes & règles
- 🔐 DoH/DoT (selon config)
- 👨‍👩‍👧‍👦 Profils/clients (selon config)

## 🔗 Intégrations
- [[Uptime Kuma]]

## 🧬 Flux de données
```mermaid
graph LR
A[Service] --> B[À compléter]
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- DNS/ports ouverts selon l’usage
- (Optionnel) Base de données selon le service (voir ci-dessous)

## Docker-compose
```yaml
services:
  adguardhome:
    image: adguard/adguardhome:latest
    container_name: adguardhome
    restart: unless-stopped
    ports:
      - "3000:3000"        # UI d'admin
      - "53:53/tcp"
      - "53:53/udp"
    volumes:
      - ./adguardhome/work:/opt/adguardhome/work
      - ./adguardhome/conf:/opt/adguardhome/conf
```

# Liens externes
- GitHub : https://github.com/AdguardTeam/AdGuardHome
- Site : https://adguard.com/en/adguard-home/overview.html

# Notes
- À compléter

# Avis de l'auteur
- À compléter
