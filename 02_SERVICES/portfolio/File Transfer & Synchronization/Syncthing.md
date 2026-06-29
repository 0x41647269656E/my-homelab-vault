---
type: service
category: File Transfer & Synchronization
name: Syncthing
slug: syncthing
logo: /assets/logos/syncthing.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/syncthing/syncthing
external_url: https://syncthing.net
port: 8384
protocol: http
stack:
  - sync
roles:
  - sync
integrates_with:
  - NextCloud
  - Duplicati
tags:
  - homelab
  - syncthing
author: "0x41647269656E"
license: (via badge)
date: 09-01-2026
last_modified: 09-01-2026
created: (unknown)
---

# Syncthing
![logo|120](/assets/logos/syncthing.png)

![Stars](https://img.shields.io/github/stars/syncthing/syncthing?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/syncthing/syncthing?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/syncthing/syncthing?style=for-the-badge) ![License](https://img.shields.io/github/license/syncthing/syncthing?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/syncthing/syncthing?style=for-the-badge)

## 🧠 Description
**Syncthing** synchronisation P2P chiffrée entre machines (dossiers, versions, peer discovery).

## ⚙️ Fonctions principales
- 🔁 Sync bidirectionnelle
- 🔒 Chiffrement
- 🧾 Versions fichiers
- 🌐 Discovery
- 🖥️ UI

## 🔗 Intégrations
- [[NextCloud]]
- [[Duplicati]]

## 🧬 Flux de données
```mermaid
graph LR
DeviceA[Device A] <--> Syncthing <--> DeviceB[Device B]
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Volumes persistants (recommandé)
- (Optionnel) DNS / certificats / authentification selon usage

## Docker-compose
```yaml
services:
  syncthing:
    image: lscr.io/linuxserver/syncthing:latest
    container_name: syncthing
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
    ports:
      - "8384:8384"        # UI
      - "22000:22000/tcp"  # Sync
      - "22000:22000/udp"
      - "21027:21027/udp"  # Discovery
    volumes:
      - ./syncthing/config:/config
      - ./syncthing/data:/data
```

# Liens externes
- GitHub : https://github.com/syncthing/syncthing
- Site : https://syncthing.net

# Notes
- À compléter

# Avis de l'auteur
- À compléter
