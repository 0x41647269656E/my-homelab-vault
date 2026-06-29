---
type: service
category: Federated Identity & Authentication
name: Authelia
slug: authelia
logo: /assets/logos/authelia.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/authelia/authelia
external_url: https://www.authelia.com
port: 9091
protocol: http
stack:
  - security
roles:
  - authentication
integrates_with:
  - Traefik
  - Nginx Proxy Manager
  - NGINX
tags:
  - homelab
  - authelia
author: "0x41647269656E"
license: (via badge)
date: 09-01-2026
last_modified: 09-01-2026
created: (unknown)
---

# Authelia
![logo|120](/assets/logos/authelia.png)

![Stars](https://img.shields.io/github/stars/authelia/authelia?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/authelia/authelia?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/authelia/authelia?style=for-the-badge) ![License](https://img.shields.io/github/license/authelia/authelia?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/authelia/authelia?style=for-the-badge)

## 🧠 Description
**Authelia** fournit une authentification forte (2FA/MFA) et un SSO léger pour protéger des applications derrière un reverse-proxy.

## ⚙️ Fonctions principales
- 🔐 2FA/MFA (TOTP/WebAuthn selon config)
- 🧾 Politiques d’accès
- 🔗 SSO (OIDC selon versions/config)
- 🧩 Intégration reverse-proxy
- 📋 Logs & audit de base

## 🔗 Intégrations
- [[Traefik]]
- [[Nginx Proxy Manager]]
- [[NGINX]]

## 🧬 Flux de données
```mermaid
graph LR
User[Utilisateur] --> Proxy[Traefik/Nginx Proxy Manager/NGINX]
Proxy --> Authelia
Authelia --> App[Applications]
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- DNS/ports ouverts selon l’usage
- (Optionnel) Base de données selon le service (voir ci-dessous)

## Docker-compose
```yaml
services:
  authelia:
    image: authelia/authelia:latest
    container_name: authelia
    restart: unless-stopped
    environment:
      - TZ=Europe/Paris
      # Génère une JWT secret : openssl rand -hex 32
      - AUTHELIA_JWT_SECRET=${AUTHELIA_JWT_SECRET:-CHANGE_ME_JWT_SECRET}
    ports:
      - "9091:9091"
    volumes:
      - ./authelia/config:/config
      - ./authelia/data:/var/lib/authelia
```

# Liens externes
- GitHub : https://github.com/authelia/authelia
- Site : https://www.authelia.com

# Notes
- À compléter

# Avis de l'auteur
- À compléter
