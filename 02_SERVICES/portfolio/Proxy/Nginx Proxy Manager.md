---
type: service
category: Proxy
name: Nginx Proxy Manager
slug: nginx-proxy-manager
logo: /assets/logos/nginx-proxy-manager.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/NginxProxyManager/nginx-proxy-manager
external_url: https://nginxproxymanager.com
port: 81
protocol: http
stack:
  - network
roles:
  - reverse-proxy
integrates_with:
  - Authelia
  - VaultWarden
  - NextCloud
tags:
  - homelab
  - nginx-proxy-manager
author: adrientanaka
license: (via badge)
created: (unknown)
---

# Nginx Proxy Manager
![logo|120](/assets/logos/nginx-proxy-manager.png)

![Stars](https://img.shields.io/github/stars/NginxProxyManager/nginx-proxy-manager?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/NginxProxyManager/nginx-proxy-manager?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/NginxProxyManager/nginx-proxy-manager?style=for-the-badge) ![License](https://img.shields.io/github/license/NginxProxyManager/nginx-proxy-manager?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/NginxProxyManager/nginx-proxy-manager?style=for-the-badge)

## 🧠 Description
**Nginx Proxy Manager** interface web pour gérer facilement NGINX (reverse-proxy, certificats, hôtes, redirections).

## ⚙️ Fonctions principales
- 🌐 Reverse proxy via UI
- 🔒 Certificats Let's Encrypt
- 📦 Hôtes/proxys/redirections
- 📜 Logs
- 👥 Gestion accès (selon usage)

## 🔗 Intégrations
- [[Authelia]]
- [[VaultWarden]]
- [[NextCloud]]

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
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - "80:80"
      - "81:81"      # UI
      - "443:443"
    volumes:
      - ./npm/data:/data
      - ./npm/letsencrypt:/etc/letsencrypt
```

# Liens externes
- GitHub : https://github.com/NginxProxyManager/nginx-proxy-manager
- Site : https://nginxproxymanager.com

# Notes
- À compléter

# Avis de l'auteur
- À compléter
