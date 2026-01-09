---
type: service
category: Web Servers
name: NGINX
slug: nginx
logo: /assets/logos/nginx.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/nginx/nginx
external_url: https://nginx.org
port: 8084
protocol: http
stack:
  - network
roles:
  - web-server
integrates_with:
  - Authelia
tags:
  - homelab
  - nginx
author: adrientanaka
license: (via badge)
created: (unknown)
---

# NGINX
![logo|120](/assets/logos/nginx.png)

![Stars](https://img.shields.io/github/stars/nginx/nginx?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/nginx/nginx?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/nginx/nginx?style=for-the-badge) ![License](https://img.shields.io/github/license/nginx/nginx?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/nginx/nginx?style=for-the-badge)

## 🧠 Description
**NGINX** serveur web et reverse-proxy très répandu, performant et flexible.

## ⚙️ Fonctions principales
- 🌐 Serveur web
- 🔁 Reverse-proxy
- 📦 Static files
- 🧩 Modules
- 🧾 Logs

## 🔗 Intégrations
- [[Authelia]]

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
  nginx:
    image: nginx:stable
    container_name: nginx
    restart: unless-stopped
    ports:
      - "8084:80"
    volumes:
      - ./nginx/html:/usr/share/nginx/html:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
```

# Liens externes
- GitHub : https://github.com/nginx/nginx
- Site : https://nginx.org

# Notes
- À compléter

# Avis de l'auteur
- À compléter
