---
type: service
category: Monitoring
name: Prometheus
slug: prometheus
logo: /assets/logos/prometheus.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/prometheus/prometheus
external_url: https://prometheus.io/
port: 9090
protocol: http
stack:
  - monitoring
roles:
  - metrics
integrates_with:
  - Grafana
  - Uptime Kuma
tags:
  - homelab
  - prometheus
author: "0x41647269656E"
license: (via badge)
date: 09-01-2026
last_modified: 09-01-2026
created: (unknown)
---

# Prometheus
![logo|120](/assets/logos/prometheus.png)

![Stars](https://img.shields.io/github/stars/prometheus/prometheus?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/prometheus/prometheus?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/prometheus/prometheus?style=for-the-badge) ![License](https://img.shields.io/github/license/prometheus/prometheus?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/prometheus/prometheus?style=for-the-badge)

## 🧠 Description
**Prometheus** collecte des métriques (time-series) depuis des cibles (exporters) et permet d’alerter via des règles.

## ⚙️ Fonctions principales
- 📈 Collecte métriques
- 🧠 PromQL
- 🚨 Alerting (via Alertmanager)
- 🧩 Exporters
- 🗄️ Stockage TSDB

## 🔗 Intégrations
- [[Grafana]]
- [[Uptime Kuma]]

## 🧬 Flux de données
```mermaid
graph LR
Targets[Services/Exporters] --> Prometheus
Prometheus --> Grafana
Prometheus --> Alerts[Alertmanager]
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Volumes persistants (recommandé)
- (Optionnel) DNS / certificats / authentification selon usage

## Docker-compose
```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    user: "1000:1000"
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/config:/etc/prometheus:ro
      - ./prometheus/data:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --web.enable-lifecycle=true
```

# Liens externes
- GitHub : https://github.com/prometheus/prometheus
- Site : https://prometheus.io/

# Notes
- À compléter

# Avis de l'auteur
- À compléter
