---
type: service
category: Software Development
name: Gitea
slug: gitea
logo: /assets/logos/gitea.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/go-gitea/gitea
external_url: https://gitea.io/en-us/
port: 3004
protocol: http
stack:
  - dev
roles:
  - git
integrates_with:
  - SonarQube
  - Keycloak
tags:
  - homelab
  - gitea
author: "0x41647269656E"
license: (via badge)
date: 09-01-2026
last_modified: 09-01-2026
created: (unknown)
---

# Gitea
![logo|120](/assets/logos/gitea.png)

![Stars](https://img.shields.io/github/stars/go-gitea/gitea?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/go-gitea/gitea?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/go-gitea/gitea?style=for-the-badge) ![License](https://img.shields.io/github/license/go-gitea/gitea?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/go-gitea/gitea?style=for-the-badge)

## 🧠 Description
**Gitea** forge Git auto-hébergée (issues, PR, wiki, actions selon version) alternative légère à GitHub.

## ⚙️ Fonctions principales
- 🧑‍💻 Repos Git
- 🔀 PR/MR
- 🐞 Issues
- 📦 Packages (selon version)
- 🔐 SSO (selon config)

## 🔗 Intégrations
- [[SonarQube]]
- [[Keycloak]]

## 🧬 Flux de données
```mermaid
graph LR
Dev[Développeur] --> Gitea
Gitea --> CI[CI/CD]
Gitea --> Sonar[SonarQube]
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Accès au matériel si nécessaire (ex: USB, GPIO, etc.)
- Volumes persistants (recommandé)

## Docker-compose
```yaml
services:
  gitea-db:
    image: postgres:16
    container_name: gitea-db
    restart: unless-stopped
    environment:
      - POSTGRES_DB=gitea
      - POSTGRES_USER=adrientanaka
      - POSTGRES_PASSWORD=${GITEA_DB_PASSWORD:-CHANGE_ME_DB_PASSWORD}
    volumes:
      - ./gitea/postgres:/var/lib/postgresql/data

  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    restart: unless-stopped
    depends_on:
      - gitea-db
    environment:
      - USER_UID=1000
      - USER_GID=1000
      - GITEA__database__DB_TYPE=postgres
      - GITEA__database__HOST=gitea-db:5432
      - GITEA__database__NAME=gitea
      - GITEA__database__USER=adrientanaka
      - GITEA__database__PASSWD=${GITEA_DB_PASSWORD:-CHANGE_ME_DB_PASSWORD}
      - TZ=Europe/Paris
    ports:
      - "3004:3000"
      - "2222:22"
    volumes:
      - ./gitea/data:/data
```

# Liens externes
- GitHub : https://github.com/go-gitea/gitea
- Site : https://gitea.io/en-us/

# Notes
- À compléter

# Avis de l'auteur
- À compléter
