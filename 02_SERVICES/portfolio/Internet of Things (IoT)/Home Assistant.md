---
type: service
category: Internet of Things (IoT)
name: Home Assistant
slug: home-assistant
logo: /assets/logos/home-assistant.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/home-assistant/core
external_url: https://www.home-assistant.io/
port: 8123
protocol: http
stack:
  - iot
roles:
  - home-automation
integrates_with:
  - ESPHome
  - AdGuard Home
  - Prometheus
tags:
  - homelab
  - home-assistant
author: adrientanaka
license: (via badge)
created: (unknown)
---

# Home Assistant
![logo|120](/assets/logos/home-assistant.png)

![Stars](https://img.shields.io/github/stars/home-assistant/core?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/home-assistant/core?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/home-assistant/core?style=for-the-badge) ![License](https://img.shields.io/github/license/home-assistant/core?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/home-assistant/core?style=for-the-badge)

## 🧠 Description
**Home Assistant** plateforme domotique (automations, intégrations, tableaux de bord) pour centraliser capteurs et équipements.

## ⚙️ Fonctions principales
- 🏠 Automations
- 🔌 Intégrations (Zigbee/Z-Wave/cloud)
- 📊 Dashboards
- 🧩 Add-ons (selon install)
- 📜 Historique & logs

## 🔗 Intégrations
- [[ESPHome]]
- [[AdGuard Home]]
- [[Prometheus]]

## 🧬 Flux de données

```mermaid
graph LR
Devices[Capteurs/Equipements] --> HA[Home Assistant]
HA --> Automations[Automations]
HA --> Dashboards[UI]
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Accès au matériel si nécessaire (ex: USB, GPIO, etc.)
- Volumes persistants (recommandé)

## Docker-compose
```yaml
services:
  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    container_name: homeassistant
    restart: unless-stopped
    network_mode: host
    privileged: true
    environment:
      - TZ=Europe/Paris
    volumes:
      - ./home-assistant/config:/config
      # (Optionnel) DB externe recommandée à terme (MariaDB/Postgres)
```

# Liens externes
- GitHub : https://github.com/home-assistant/core
- Site : https://www.home-assistant.io/

# Notes
- À compléter

# Avis de l'auteur
- À compléter
