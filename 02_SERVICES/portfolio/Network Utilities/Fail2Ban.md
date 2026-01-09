---
type: service
category: Network Utilities
name: Fail2Ban
slug: fail2ban
logo: /assets/logos/fail2ban.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/fail2ban/fail2ban
external_url: https://www.fail2ban.org
port: 0
protocol: 
stack:
  - security
roles:
  - intrusion-prevention
integrates_with:
  - NGINX
  - Apache HTTP Server
tags:
  - homelab
  - fail2ban
author: adrientanaka
license: (via badge)
created: (unknown)
---

# Fail2Ban
![logo|120](/assets/logos/fail2ban.png)

![Stars](https://img.shields.io/github/stars/fail2ban/fail2ban?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/fail2ban/fail2ban?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/fail2ban/fail2ban?style=for-the-badge) ![License](https://img.shields.io/github/license/fail2ban/fail2ban?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/fail2ban/fail2ban?style=for-the-badge)

## 🧠 Description
**Fail2Ban** surveille les logs et bannit automatiquement les IP malveillantes (brute force, scans…).

## ⚙️ Fonctions principales
- 🧯 Bans automatiques
- 📜 Jails & filtres
- 🧾 Analyse logs
- 🔌 Support iptables/nftables
- ⚙️ Extensible

## 🔗 Intégrations
- [[NGINX]]
- [[Apache HTTP Server]]

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
  fail2ban:
    image: crazymax/fail2ban:latest
    container_name: fail2ban
    restart: unless-stopped
    network_mode: host
    cap_add:
      - NET_ADMIN
      - NET_RAW
    volumes:
      - ./fail2ban/data:/data
      # Monte tes logs (adapter selon ta distro)
      - /var/log:/var/log:ro
```

# Liens externes
- GitHub : https://github.com/fail2ban/fail2ban
- Site : https://www.fail2ban.org

# Notes
- À compléter

# Avis de l'auteur
- À compléter
