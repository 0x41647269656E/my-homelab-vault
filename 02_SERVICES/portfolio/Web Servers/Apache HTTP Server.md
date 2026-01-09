---
type: service
category: Web Servers
name: Apache HTTP Server
slug: apache-http-server
logo: /assets/logos/apache-http-server.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/apache/httpd
external_url: https://httpd.apache.org
port: 8085
protocol: http
stack:
  - network
roles:
  - web-server
integrates_with:
  - (à compléter)
tags:
  - homelab
  - apache-http-server
author: adrientanaka
license: (via badge)
created: (unknown)
---

# Apache HTTP Server
![logo|120](/assets/logos/apache-http-server.png)

![Stars](https://img.shields.io/github/stars/apache/httpd?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/apache/httpd?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/apache/httpd?style=for-the-badge) ![License](https://img.shields.io/github/license/apache/httpd?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/apache/httpd?style=for-the-badge)

## 🧠 Description
**Apache HTTP Server** serveur web historique, très configurable, supportant de nombreux modules.

## ⚙️ Fonctions principales
- 🌐 Serveur web
- 🧩 Modules (PHP, auth, proxy...)
- 📦 VirtualHosts
- 🔒 TLS
- 🧾 Logs

## 🔗 Intégrations
- (à compléter)

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
  apache:
    image: httpd:2.4
    container_name: apache
    restart: unless-stopped
    ports:
      - "8085:80"
    volumes:
      - ./apache/htdocs:/usr/local/apache2/htdocs:ro
      - ./apache/conf:/usr/local/apache2/conf:ro
```

# Liens externes
- GitHub : https://github.com/apache/httpd
- Site : https://httpd.apache.org

# Notes
- À compléter

# Avis de l'auteur
- À compléter
