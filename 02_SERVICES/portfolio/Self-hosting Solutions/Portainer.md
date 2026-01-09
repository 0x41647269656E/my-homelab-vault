---
type: service
category: Self-hosting Solutions
name: Portainer
slug: portainer
logo: /assets/logos/portainer.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/portainer/portainer
external_url: https://www.portainer.io/
port: 9000
protocol: http
stack:
  - infra
roles:
  - docker-management
integrates_with:
  - Traefik
tags:
  - homelab
  - portainer
author: adrientanaka
license: (via badge)
created: (unknown)
---

# Portainer
![logo|120](/assets/logos/portainer.png)

![Stars](https://img.shields.io/github/stars/portainer/portainer?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/portainer/portainer?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/portainer/portainer?style=for-the-badge) ![License](https://img.shields.io/github/license/portainer/portainer?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/portainer/portainer?style=for-the-badge)

## 🧠 Description
**Portainer** interface d’administration Docker (containers, stacks, volumes, networks) utile pour piloter un homelab.

## ⚙️ Fonctions principales
- 🐳 Gestion Docker
- 📦 Stacks (Compose)
- 👥 RBAC (selon édition)
- 📊 Vue ressources
- 🔐 Secrets/registries (selon config)

## 🔗 Intégrations
- [[Traefik]]

## 🧬 Flux de données
```mermaid
graph LR
Docker[Docker Engine] --> Portainer
User[Utilisateur] --> Portainer
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Volumes persistants (recommandé)
- (Optionnel) DNS / certificats / authentification selon usage

## Docker-compose
```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    ports:
      - "9000:9000"
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./portainer/data:/data
```

# Liens externes
- GitHub : https://github.com/portainer/portainer
- Site : https://www.portainer.io/

# Notes
- À compléter

# Avis de l'auteur
- À compléter
