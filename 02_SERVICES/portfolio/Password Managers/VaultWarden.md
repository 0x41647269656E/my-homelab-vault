---
type: service
category: Password Managers
name: VaultWarden
slug: vaultwarden
logo: /assets/logos/vaultwarden.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/dani-garcia/vaultwarden
external_url: https://github.com/dani-garcia/vaultwarden
port: 8082
protocol: http
stack:
  - security
roles:
  - passwords
integrates_with:
  - NGINX
  - Traefik
  - Nginx Proxy Manager
tags:
  - homelab
  - vaultwarden
author: "0x41647269656E"
license: (via badge)
date: 09-01-2026
last_modified: 09-01-2026
created: (unknown)
---

# VaultWarden
![logo|120](/assets/logos/vaultwarden.png)

![Stars](https://img.shields.io/github/stars/dani-garcia/vaultwarden?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/dani-garcia/vaultwarden?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/dani-garcia/vaultwarden?style=for-the-badge) ![License](https://img.shields.io/github/license/dani-garcia/vaultwarden?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/dani-garcia/vaultwarden?style=for-the-badge)

## 🧠 Description
**VaultWarden** implémentation légère compatible Bitwarden pour héberger ton gestionnaire de mots de passe.

## ⚙️ Fonctions principales
- 🔑 Coffre Bitwarden compatible
- 👥 Organisations & partages
- 🔒 WebVault + API
- 🧾 Journaux & admin token
- 🔌 WebSocket (temps réel)

## 🔗 Intégrations
- [[NGINX]]
- [[Traefik]]
- [[Nginx Proxy Manager]]

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
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      - TZ=Europe/Paris
      - WEBSOCKET_ENABLED=true
      - SIGNUPS_ALLOWED=false
      # Génère une admin token : openssl rand -base64 48
      - ADMIN_TOKEN=${VAULTWARDEN_ADMIN_TOKEN:-CHANGE_ME_ADMIN_TOKEN}
    ports:
      - "8082:80"
    volumes:
      - ./vaultwarden/data:/data
```

# Liens externes
- GitHub : https://github.com/dani-garcia/vaultwarden
- Site : https://github.com/dani-garcia/vaultwarden

# Notes
- À compléter

# Avis de l'auteur
- À compléter
