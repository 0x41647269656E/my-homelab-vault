---
type: service
category: Proxy
name: Traefik
slug: traefik
logo: /assets/logos/traefik.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/traefik/traefik
external_url: https://traefik.io/traefik/
port: 8083
protocol: http
stack:
  - network
roles:
  - reverse-proxy
integrates_with:
  - Authelia
  - Portainer
tags:
  - homelab
  - traefik
author: "0x41647269656E"
license: (via badge)
date: 09-01-2026
last_modified: 09-01-2026
created: (unknown)
---

# Traefik
![logo|120](/assets/logos/traefik.png)

![Stars](https://img.shields.io/github/stars/traefik/traefik?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/traefik/traefik?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/traefik/traefik?style=for-the-badge) ![License](https://img.shields.io/github/license/traefik/traefik?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/traefik/traefik?style=for-the-badge)

## 🧠 Description
**Traefik** reverse-proxy moderne orienté cloud-native, très utilisé avec Docker/Kubernetes.

## ⚙️ Fonctions principales
- 🧩 Découverte auto Docker/K8s
- 🔒 TLS/ACME
- 🧭 Routing flexible
- 📊 Dashboard
- ⚙️ Middlewares

## 🔗 Intégrations
- [[Authelia]]
- [[Portainer]]

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
  traefik:
    image: traefik:latest
    container_name: traefik
    restart: unless-stopped
    command:
      - --api.dashboard=true
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
    ports:
      - "80:80"
      - "443:443"
      - "8083:8080"     # dashboard (à sécuriser)
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik/data:/data
```

# Liens externes
- GitHub : https://github.com/traefik/traefik
- Site : https://traefik.io/traefik/

# Notes
- À compléter

# Avis de l'auteur
- À compléter
