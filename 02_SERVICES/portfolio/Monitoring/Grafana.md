---
type: service
category: Monitoring
name: Grafana
slug: grafana
logo: /assets/logos/grafana.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/grafana/grafana
external_url: https://grafana.com/
port: 3001
protocol: http
stack:
  - monitoring
roles:
  - dashboards
integrates_with:
  - Prometheus
  - Elasticsearch
  - Keycloak
tags:
  - homelab
  - grafana
author: adrientanaka
license: (via badge)
created: (unknown)
---

# Grafana
![logo|120](/assets/logos/grafana.png)

![Stars](https://img.shields.io/github/stars/grafana/grafana?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/grafana/grafana?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/grafana/grafana?style=for-the-badge) ![License](https://img.shields.io/github/license/grafana/grafana?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/grafana/grafana?style=for-the-badge)

## 🧠 Description
**Grafana** plateforme de visualisation/observabilité (dashboards, alerting) compatible avec de nombreuses sources (Prometheus, Elasticsearch, etc.).

## ⚙️ Fonctions principales
- 📊 Dashboards
- 🔔 Alerting
- 🔌 Datasources multiples
- 👥 Gestion utilisateurs
- ⚙️ Provisioning as code

## 🔗 Intégrations
- [[Prometheus]]
- [[Elasticsearch]]
- [[Keycloak]]

## 🧬 Flux de données
```mermaid
graph LR
Prometheus --> Grafana
Elasticsearch --> Grafana
Users[Utilisateurs] --> Grafana
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Volumes persistants (recommandé)
- (Optionnel) DNS / certificats / authentification selon usage

## Docker-compose
```yaml
services:
  grafana:
    image: grafana/grafana-oss:latest
    container_name: grafana
    restart: unless-stopped
    user: "1000:1000"
    environment:
      - GF_SECURITY_ADMIN_USER=adrientanaka
      # Génère un mot de passe : openssl rand -base64 32
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD:-CHANGE_ME_ADMIN_PASSWORD}
      - GF_USERS_ALLOW_SIGN_UP=false
      - TZ=Europe/Paris
    ports:
      - "3001:3000"
    volumes:
      - ./grafana/data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
```

# Liens externes
- GitHub : https://github.com/grafana/grafana
- Site : https://grafana.com/

# Notes
- À compléter

# Avis de l'auteur
- À compléter
