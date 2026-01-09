---
type: service
category: Software Development
name: SonarQube
slug: sonarqube
logo: /assets/logos/sonarqube.png
status: active
integration_status: planned
opensource: true
docker: true
github_url: https://github.com/SonarSource/sonarqube
external_url: https://www.sonarsource.com/products/sonarqube/
port: 9001
protocol: http
stack:
  - dev
roles:
  - code-quality
integrates_with:
  - Gitea
  - GitHub Community
tags:
  - homelab
  - sonarqube
author: adrientanaka
license: (via badge)
created: (unknown)
---

# SonarQube
![logo|120](/assets/logos/sonarqube.png)

![Stars](https://img.shields.io/github/stars/SonarSource/sonarqube?style=for-the-badge) ![Release](https://img.shields.io/github/v/release/SonarSource/sonarqube?style=for-the-badge&include_prereleases=true) ![Lang](https://img.shields.io/github/languages/top/SonarSource/sonarqube?style=for-the-badge) ![License](https://img.shields.io/github/license/SonarSource/sonarqube?style=for-the-badge) ![Last Commit](https://img.shields.io/github/last-commit/SonarSource/sonarqube?style=for-the-badge)

## 🧠 Description
**SonarQube** plateforme d’analyse de qualité de code (bugs, vulnérabilités, code smells) intégrable aux pipelines CI/CD.

## ⚙️ Fonctions principales
- 🧪 Analyse statique
- 🔐 Détection vulnérabilités
- 📈 Qualité & couverture
- 🧩 Plugins/langages
- 🔁 Intégration CI

## 🔗 Intégrations
- [[Gitea]]
- [[GitHub Community]]

## 🧬 Flux de données
```mermaid
graph LR
Repo[Git] --> CI[CI Pipeline] --> SonarQube
SonarQube --> Reports[Rapports]
```

# Intégration

## Pré-requis
- Docker + Docker Compose
- Volumes persistants (recommandé)
- (Optionnel) DNS / certificats / authentification selon usage

## Docker-compose
```yaml
services:
  sonarqube-db:
    image: postgres:16
    container_name: sonarqube-db
    restart: unless-stopped
    environment:
      - POSTGRES_DB=sonarqube
      - POSTGRES_USER=adrientanaka
      - POSTGRES_PASSWORD=${SONARQUBE_DB_PASSWORD:-CHANGE_ME_DB_PASSWORD}
    volumes:
      - ./sonarqube/postgres:/var/lib/postgresql/data

  sonarqube:
    image: sonarqube:lts-community
    container_name: sonarqube
    restart: unless-stopped
    depends_on:
      - sonarqube-db
    environment:
      - SONAR_JDBC_URL=jdbc:postgresql://sonarqube-db:5432/sonarqube
      - SONAR_JDBC_USERNAME=adrientanaka
      - SONAR_JDBC_PASSWORD=${SONARQUBE_DB_PASSWORD:-CHANGE_ME_DB_PASSWORD}
      - TZ=Europe/Paris
    ports:
      - "9001:9000"
    volumes:
      - ./sonarqube/data:/opt/sonarqube/data
      - ./sonarqube/extensions:/opt/sonarqube/extensions
      - ./sonarqube/logs:/opt/sonarqube/logs
```

# Liens externes
- GitHub : https://github.com/SonarSource/sonarqube
- Site : https://www.sonarsource.com/products/sonarqube/

# Notes
- À compléter

# Avis de l'auteur
- À compléter
