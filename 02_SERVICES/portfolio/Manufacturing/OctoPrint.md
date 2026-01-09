---
type: service
category: Manufacturing
name: OctoPrint
slug: octoprint
logo: /assets/logos/octoprint.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/OctoPrint/OctoPrint
external_url: https://octoprint.org/
port: 5000
protocol: http
stack:
  - iot
roles:
  - 3d-print
integrates_with:
  - Home Assistant
tags:
  - homelab
  - octoprint
author: adrientanaka
license: (via badge)
created: (unknown)
---

# OctoPrint
![logo|120](/assets/logos/octoprint.png)

![Stars](https://img.shields.io/github/stars/OctoPrint/OctoPrint?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/OctoPrint/OctoPrint?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/OctoPrint/OctoPrint?style=for-the-badge) ![License](https://img.shields.io/github/license/OctoPrint/OctoPrint?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/OctoPrint/OctoPrint?style=for-the-badge)

## 🧠 Description
**OctoPrint** contrôle et supervision d’imprimantes 3D (upload gcode, suivi, webcam, plugins).

## ⚙️ Fonctions principales
- 🖨️ Contrôle imprimante 3D
- 📤 Upload G-code
- 📷 Webcam
- 🧩 Plugins
- 📈 Monitoring impression

## 🔗 Intégrations
- [[Home Assistant]]

## 🧬 Flux de données
```mermaid
graph LR
User[Utilisateur] --> OctoPrint
OctoPrint --> Printer[Imprimante 3D]
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Accès au matériel si nécessaire (ex: USB, GPIO, etc.)
- Volumes persistants (recommandé)

## Docker-compose
```yaml
services:
  octoprint:
    image: octoprint/octoprint:latest
    container_name: octoprint
    restart: unless-stopped
    ports:
      - "5000:80"
    environment:
      - TZ=Europe/Paris
    volumes:
      - ./octoprint/config:/octoprint
    devices:
      # Adaptation : port série de l'imprimante
      # - /dev/ttyUSB0:/dev/ttyUSB0
      # (Optionnel) Webcam:
      # - /dev/video0:/dev/video0
```

# Liens externes
- GitHub : https://github.com/OctoPrint/OctoPrint
- Site : https://octoprint.org/

# Notes
- À compléter

# Avis de l'auteur
- À compléter
