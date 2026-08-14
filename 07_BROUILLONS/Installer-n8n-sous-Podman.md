---
title: Installer n8n sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - n8n
  - automatisation
  - podman
  - installation
  - securite
date: 07-06-2026
last_modified: 07-06-2026
reading-time: 15m
difficulty: tech-enthusiast
status: draft
---

# Installer n8n sous Podman rootless

> [!abstract] TL;DR
> n8n est une plateforme d'**automatisation de workflows** (type Zapier/Make, mais auto-hébergée) : connecter des services, déclencher des actions, orchestrer des tâches via une interface visuelle. App + PostgreSQL. Le point sensible — et c'est le cœur de l'article — : n8n détient les **credentials de tes services connectés** (clés API, tokens, mots de passe), ce qui en fait une cible précieuse à durcir et un service à ne pas exposer publiquement à la légère.

> [!info] Prérequis de lecture
> Socle de la série. Réseau interne, Caddy, VPN, MAC, restic.

---

## La spécificité de sécurité : n8n est un coffre de credentials déguisé

> [!danger] Ce que contient une instance n8n
> Pour automatiser, n8n se connecte à tes services : ton mail, ton GitLab, des API tierces, peut-être ta base de données. Il **stocke les credentials** de tous ces connecteurs (chiffrés, mais présents). Une compromission de n8n, c'est l'accès potentiel à tout ce qu'il orchestre. C'est, dans une moindre mesure, le même profil de risque que Vaultwarden : un point qui concentre des accès.

Conséquence : **clé de chiffrement solide**, **pas d'exposition publique sans raison**, et durcissement soigné.

---

## Architecture

```bash
podman network create n8n-internal --internal
```

```ini
# ~/.config/containers/systemd/n8n-postgres.container
[Unit]
Description=n8n PostgreSQL
[Container]
Image=docker.io/library/postgres:16-alpine
ContainerName=n8n-postgres
AutoUpdate=registry
Network=n8n-internal.network
Volume=%h/n8n/pgdata:/var/lib/postgresql/data:U,Z
Environment=POSTGRES_DB=n8n
Environment=POSTGRES_USER=n8n
Secret=n8n_db_password,type=env,target=POSTGRES_PASSWORD
NoNewPrivileges=true
DropCapability=ALL
AddCapability=CAP_CHOWN,CAP_SETUID,CAP_SETGID,CAP_DAC_OVERRIDE
[Service]
Restart=on-failure
MemoryMax=512M
[Install]
WantedBy=default.target
```

```ini
# ~/.config/containers/systemd/n8n.container
[Unit]
Description=n8n
Requires=n8n-postgres.service
After=n8n-postgres.service
[Container]
Image=docker.io/n8nio/n8n:latest
ContainerName=n8n
AutoUpdate=registry
Network=n8n-internal.network
Volume=%h/n8n/data:/home/node/.n8n:U,Z

Environment=DB_TYPE=postgresdb
Environment=DB_POSTGRESDB_HOST=n8n-postgres
Environment=DB_POSTGRESDB_DATABASE=n8n
Environment=DB_POSTGRESDB_USER=n8n
Environment=N8N_HOST=automation.mondomaine.fr
Environment=N8N_PROTOCOL=https
Environment=WEBHOOK_URL=https://automation.mondomaine.fr/
# Clé de chiffrement des credentials : CRITIQUE, via secret
Secret=n8n_db_password,type=env,target=DB_POSTGRESDB_PASSWORD
Secret=n8n_encryption_key,type=env,target=N8N_ENCRYPTION_KEY

NoNewPrivileges=true
DropCapability=ALL

[Service]
Restart=on-failure
RestartSec=10
MemoryMax=1G

[Install]
WantedBy=default.target
```

> [!danger] N8N_ENCRYPTION_KEY est la clé de voûte
> Cette clé chiffre tous les credentials stockés. **Si tu la perds, tous tes connecteurs deviennent illisibles** ; si elle fuite, ils deviennent déchiffrables. Génère-la solidement, passe-la par les secrets systemd (jamais en clair), et sauvegarde-la **séparément** de la base — exactement comme la clé restic ou les secrets Vaultwarden. Si tu laisses n8n la générer automatiquement, récupère-la dans `~/n8n/data/config` et sauvegarde-la.

```bash
systemctl --user daemon-reload
systemctl --user start n8n.service
```

---

## Exposition : le cas des webhooks

> [!warning] Le dilemme webhook vs sécurité
> n8n peut être déclenché par des **webhooks** entrants (un service externe appelle n8n pour lancer un workflow). Ça **nécessite une exposition publique** du endpoint webhook. Mais l'interface d'admin, elle, n'a aucune raison d'être publique. Idéalement : exposer **uniquement** les chemins webhook, garder l'interface d'admin derrière VPN.

```caddyfile
automation.mondomaine.fr {
	import security_headers

	# Les webhooks entrants (publics, mais ce sont des endpoints précis)
	@webhook path /webhook/* /webhook-test/*
	handle @webhook {
		reverse_proxy n8n:5678
	}

	# Le reste (interface d'admin) : VPN uniquement
	@admin remote_ip 10.10.1.0/24
	handle @admin {
		reverse_proxy n8n:5678
	}

	respond "Forbidden" 403
}
```

> [!tip] Sécuriser les webhooks exposés
> Un webhook public est une porte ouverte : protège chaque workflow déclenché par webhook avec une authentification (header secret, HMAC) dans n8n lui-même. Ne te repose pas sur l'obscurité de l'URL. Si tu n'as pas besoin de webhooks externes, garde **tout** n8n derrière VPN, c'est le plus simple et le plus sûr.

---

## Sauvegarde

> [!success] Quoi sauvegarder
> - **Dump SQL** de `n8n-postgres` : workflows, exécutions, credentials chiffrés.
> - **`~/n8n/data`** : config, et la clé de chiffrement si auto-générée.
> - **La N8N_ENCRYPTION_KEY séparément** : sans elle, restaurer la base ne te rend pas tes credentials utilisables.
> - n8n permet aussi d'**exporter les workflows en JSON** — utile pour le versioning (tu peux même les committer dans ton GitLab).

```bash
podman exec n8n-postgres pg_dump -U n8n -Fc n8n > ~/backup/dumps/n8n-$(date +%F).dump
```

---

## Conclusion

n8n est puissant mais porte un risque spécifique : il **concentre des accès** à tous tes services connectés. Le traiter avec le sérieux d'un quasi-coffre — clé de chiffrement solide et sauvegardée à part, interface d'admin VPN-only, webhooks authentifiés et exposés au minimum — est la bonne posture. C'est l'automatisation au service du parc, pas une nouvelle faille.

> [!note] Articles liés
> - [[Installer Vaultwarden sous Podman rootless]]
> - [[Installer GitLab CE sous Podman rootless]]
> - [[Stratégie de sauvegarde restic 3-2-1]]

---

*Retour d'expérience personnel. Adapte domaines, images. La clé de chiffrement est l'élément le plus critique : génère-la et sauvegarde-la avec soin.*
