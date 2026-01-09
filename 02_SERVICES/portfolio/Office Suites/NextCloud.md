---
type: service
category: Office Suites
name: NextCloud
slug: nextcloud
logo: /assets/logos/nextcloud.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/nextcloud/server
external_url: https://nextcloud.com
port: 8087
protocol: http
stack:
  - cloud
roles:
  - collaboration
integrates_with:
  - Keycloak
  - Nginx Proxy Manager
  - Traefik
  - Syncthing
tags:
  - homelab
  - nextcloud
author: adrientanaka
license: (via badge)
created: (unknown)
---

# NextCloud
![logo|120](/assets/logos/nextcloud.png)

![Stars](https://img.shields.io/github/stars/nextcloud/server?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/nextcloud/server?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/nextcloud/server?style=for-the-badge) ![License](https://img.shields.io/github/license/nextcloud/server?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/nextcloud/server?style=for-the-badge)

## 🧠 Description
**NextCloud** plateforme de cloud personnel (fichiers, agenda, contacts, collaboration) extensible via apps.

## ⚙️ Fonctions principales
- ☁️ Fichiers & partage
- 📅 Agenda/Contacts (apps)
- 🗂️ Apps (OnlyOffice, Talk, etc.)
- 🔐 SSO/OIDC (selon config)
- 🧩 WebDAV

## 🔗 Intégrations
- [[Keycloak]]
- [[Nginx Proxy Manager]]
- [[Traefik]]
- [[Syncthing]]

## 🧬 Flux de données
```mermaid
graph LR
User[Utilisateurs] --> Nextcloud
Nextcloud --> Storage[Volumes/Stockage]
Nextcloud --> DB[(PostgreSQL)]
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Volumes persistants (recommandé)
- (Optionnel) DNS / certificats / authentification selon usage

## Docker-compose
```yaml
services:
  nextcloud-db:
    image: postgres:16
    container_name: nextcloud-db
    restart: unless-stopped
    environment:
      - POSTGRES_DB=nextcloud
      - POSTGRES_USER=adrientanaka
      - POSTGRES_PASSWORD=${NEXTCLOUD_DB_PASSWORD:-CHANGE_ME_DB_PASSWORD}
    volumes:
      - ./nextcloud/postgres:/var/lib/postgresql/data

  nextcloud-redis:
    image: redis:7-alpine
    container_name: nextcloud-redis
    restart: unless-stopped
    volumes:
      - ./nextcloud/redis:/data

  nextcloud:
    image: nextcloud:29-apache
    container_name: nextcloud
    restart: unless-stopped
    depends_on:
      - nextcloud-db
      - nextcloud-redis
    ports:
      - "8087:80"
    environment:
      - POSTGRES_HOST=nextcloud-db
      - POSTGRES_DB=nextcloud
      - POSTGRES_USER=adrientanaka
      - POSTGRES_PASSWORD=${NEXTCLOUD_DB_PASSWORD:-CHANGE_ME_DB_PASSWORD}
      - REDIS_HOST=nextcloud-redis
      - TZ=Europe/Paris
    volumes:
      - ./nextcloud/html:/var/www/html
      - ./nextcloud/data:/var/www/html/data
```

# Liens externes
- GitHub : https://github.com/nextcloud/server
- Site : https://nextcloud.com

# Notes
- À compléter

# Avis de l'auteur
- À compléter
