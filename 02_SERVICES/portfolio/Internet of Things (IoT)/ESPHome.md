---
type: service
category: Internet of Things (IoT)
name: ESPHome
slug: esphome
logo: /assets/logos/esphome.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/esphome/esphome
external_url: https://esphome.io/
port: 6052
protocol: http
stack:
  - iot
roles:
  - device-firmware
integrates_with:
  - Home Assistant
tags:
  - homelab
  - esphome
author: adrientanaka
license: (via badge)
created: (unknown)
---

# ESPHome
![logo|120](/assets/logos/esphome.png)

![Stars](https://img.shields.io/github/stars/esphome/esphome?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/esphome/esphome?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/esphome/esphome?style=for-the-badge) ![License](https://img.shields.io/github/license/esphome/esphome?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/esphome/esphome?style=for-the-badge)

## 🧠 Description
**ESPHome** outil de génération de firmware pour ESP8266/ESP32 avec config YAML, très intégré à Home Assistant.

## ⚙️ Fonctions principales
- 🧱 Firmware YAML
- 🔌 Intégration Home Assistant
- 📡 OTA updates
- 🧪 Logs & diagnostics
- ⚙️ Templates

## 🔗 Intégrations
- [[Home Assistant]]

## 🧬 Flux de données
```mermaid
graph LR
ESPHome --> Devices[ESP8266/ESP32]
Devices --> HomeAssistant[Home Assistant]
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Accès au matériel si nécessaire (ex: USB, GPIO, etc.)
- Volumes persistants (recommandé)

## Docker-compose
```yaml
services:
  esphome:
    image: ghcr.io/esphome/esphome:latest
    container_name: esphome
    restart: unless-stopped
    environment:
      - TZ=Europe/Paris
    ports:
      - "6052:6052"
    volumes:
      - ./esphome/config:/config
      # Accès USB série si flash direct (adapter):
      # - /dev/ttyUSB0:/dev/ttyUSB0
```

# Liens externes
- GitHub : https://github.com/esphome/esphome
- Site : https://esphome.io/

# Notes
- À compléter

# Avis de l'auteur
- À compléter
