---
type: service
category: Self-hosting Solutions
name: k3s
slug: k3s
logo: /assets/logos/k3s.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/k3s-io/k3s
external_url: https://k3s.io/
port: 6443
protocol: https
stack:
  - infra
roles:
  - kubernetes
integrates_with:
  - Traefik
  - Prometheus
tags:
  - homelab
  - k3s
author: adrientanaka
license: (via badge)
created: (unknown)
---

# k3s
![logo|120](/assets/logos/k3s.png)

![Stars](https://img.shields.io/github/stars/k3s-io/k3s?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/k3s-io/k3s?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/k3s-io/k3s?style=for-the-badge) ![License](https://img.shields.io/github/license/k3s-io/k3s?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/k3s-io/k3s?style=for-the-badge)

## 🧠 Description
**k3s** distribution Kubernetes légère, idéale pour un cluster homelab (ARM/x86) avec peu de ressources.

## ⚙️ Fonctions principales
- ☸️ Kubernetes léger
- 📦 Helm/Manifests
- 🧩 Add-ons
- 🔐 RBAC
- 📈 Observabilité (à intégrer)

## 🔗 Intégrations
- [[Traefik]]
- [[Prometheus]]

## 🧬 Flux de données
```mermaid
graph LR
Nodes[Nodes] --> k3s
k3s --> Workloads[Workloads]
k3s --> Ingress[Ingress/Traefik]
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Accès au matériel si nécessaire (ex: USB, GPIO, etc.)
- Volumes persistants (recommandé)

## Docker-compose
```yaml
services:
  k3s:
    image: rancher/k3s:latest
    container_name: k3s
    restart: unless-stopped
    privileged: true
    command: server
    environment:
      - TZ=Europe/Paris
      # Token cluster : openssl rand -hex 32
      - K3S_TOKEN=${K3S_TOKEN:-CHANGE_ME_CLUSTER_TOKEN}
    ports:
      - "6443:6443"   # Kubernetes API
    volumes:
      - ./k3s/data:/var/lib/rancher/k3s
      - /var/run:/var/run
```

# Liens externes
- GitHub : https://github.com/k3s-io/k3s
- Site : https://k3s.io/

# Notes
- À compléter

# Avis de l'auteur
- À compléter
