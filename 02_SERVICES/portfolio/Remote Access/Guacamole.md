---
type: service
category: Remote Access
name: Guacamole
slug: guacamole
logo: /assets/logos/guacamole.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/apache/guacamole-client
external_url: https://guacamole.apache.org
port: 8086
protocol: http
stack:
  - remote-access
roles:
  - remote-desktop
integrates_with:
  - Keycloak
tags:
  - homelab
  - guacamole
author: "0x41647269656E"
license: (via badge)
date: 09-01-2026
last_modified: 09-01-2026
created: (unknown)
---

# Guacamole
![logo|120](/assets/logos/guacamole.png)

![Stars](https://img.shields.io/github/stars/apache/guacamole-client?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/apache/guacamole-client?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/apache/guacamole-client?style=for-the-badge) ![License](https://img.shields.io/github/license/apache/guacamole-client?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/apache/guacamole-client?style=for-the-badge)

## 🧠 Description
**Guacamole** passerelle d’accès distant (RDP/VNC/SSH) via navigateur, pratique pour centraliser l’accès aux machines.

## ⚙️ Fonctions principales
- 🖥️ RDP/VNC/SSH via navigateur
- 👥 Gestion connexions
- 🔐 Auth intégrable (LDAP/OIDC selon config)
- 📜 Audit de base
- 🧩 Extensions

## 🔗 Intégrations
- [[Keycloak]]

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
  guacd:
    image: guacamole/guacd:latest
    container_name: guacd
    restart: unless-stopped

  guacamole-db:
    image: postgres:16
    container_name: guacamole-db
    restart: unless-stopped
    environment:
      - POSTGRES_DB=guacamole
      - POSTGRES_USER=adrientanaka
      - POSTGRES_PASSWORD=${GUACAMOLE_DB_PASSWORD:-CHANGE_ME_DB_PASSWORD}
    volumes:
      - ./guacamole/postgres:/var/lib/postgresql/data

  guacamole:
    image: guacamole/guacamole:latest
    container_name: guacamole
    restart: unless-stopped
    depends_on:
      - guacd
      - guacamole-db
    environment:
      - GUACD_HOSTNAME=guacd
      - POSTGRES_HOSTNAME=guacamole-db
      - POSTGRES_DATABASE=guacamole
      - POSTGRES_USER=adrientanaka
      - POSTGRES_PASSWORD=${GUACAMOLE_DB_PASSWORD:-CHANGE_ME_DB_PASSWORD}
    ports:
      - "8086:8080"
    volumes:
      - ./guacamole/extensions:/opt/guacamole/extensions
```

# Liens externes
- GitHub : https://github.com/apache/guacamole-client
- Site : https://guacamole.apache.org

# Notes
- À compléter

# Avis de l'auteur
- À compléter
