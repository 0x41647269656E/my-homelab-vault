---
type: service
category: Search Engines
name: Elasticsearch
slug: elasticsearch
logo: /assets/logos/elasticsearch.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/elastic/elasticsearch
external_url: https://www.elastic.co/elasticsearch
port: 9200
protocol: http
stack:
  - data
roles:
  - search
integrates_with:
  - Grafana
  - Paperless-ngx
tags:
  - homelab
  - elasticsearch
author: adrientanaka
license: (via badge)
created: (unknown)
---

# Elasticsearch

![logo|120](../assets/logos/elasticsearch.png)

![Stars](https://img.shields.io/github/stars/elastic/elasticsearch?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/elastic/elasticsearch?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/elastic/elasticsearch?style=for-the-badge) ![License](https://img.shields.io/github/license/elastic/elasticsearch?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/elastic/elasticsearch?style=for-the-badge)

## 🧠 Description
**Elasticsearch** moteur de recherche et d’analytics distribué (indexation, requêtes, agrégations), base de l’écosystème Elastic.

## ⚙️ Fonctions principales
- 🔎 Recherche plein texte
- 📦 Indexation
- 📊 Agrégations
- 🧩 API REST
- ⚖️ Scalabilité

## 🔗 Intégrations
- [[Grafana]]
- [[Paperless-ngx]]

## 🧬 Flux de données
```mermaid
graph LR
Ingest[Apps/Logs/Documents] --> Elasticsearch
Elasticsearch --> Grafana
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Volumes persistants (recommandé)
- (Optionnel) DNS / certificats / authentification selon usage

## Docker-compose
```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.13.4
    container_name: elasticsearch
    restart: unless-stopped
    environment:
      - node.name=es01
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
      - TZ=Europe/Paris
    ports:
      - "9200:9200"
    volumes:
      - ./elasticsearch/data:/usr/share/elasticsearch/data
```

# Liens externes
- GitHub : https://github.com/elastic/elasticsearch
- Site : https://www.elastic.co/elasticsearch

# Notes
- À compléter

# Avis de l'auteur
- À compléter
