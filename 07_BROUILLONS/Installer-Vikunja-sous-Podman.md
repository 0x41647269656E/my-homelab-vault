---
title: Installer Vikunja sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - vikunja
  - taches
  - productivite
  - podman
  - installation
  - securite
date: 07-06-2026
last_modified: 07-06-2026
reading-time: 10m
difficulty: tech-enthusiast
status: draft
---

# Installer Vikunja sous Podman rootless

> [!abstract] TL;DR
> Vikunja est un gestionnaire de **tâches et projets** auto-hébergé (listes, kanban, gantt, échéances, rappels), une alternative libre à Todoist/Asana. Backend Go + base (PostgreSQL recommandé). Les versions récentes regroupent frontend et API dans une **image unique**, ce qui simplifie le déploiement. Installation Podman rootless directe, sans piège majeur — un bon service « propre » pour souffler après n8n.

> [!info] Prérequis de lecture
> Socle de la série. Réseau interne, Caddy, VPN, MAC, restic.

---

## Architecture

```bash
podman network create vikunja-internal --internal
```

```ini
# ~/.config/containers/systemd/vikunja-postgres.container
[Unit]
Description=Vikunja PostgreSQL
[Container]
Image=docker.io/library/postgres:16-alpine
ContainerName=vikunja-postgres
AutoUpdate=registry
Network=vikunja-internal.network
Volume=%h/vikunja/pgdata:/var/lib/postgresql/data:U,Z
Environment=POSTGRES_DB=vikunja
Environment=POSTGRES_USER=vikunja
Secret=vikunja_db_password,type=env,target=POSTGRES_PASSWORD
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
# ~/.config/containers/systemd/vikunja.container
[Unit]
Description=Vikunja
Requires=vikunja-postgres.service
After=vikunja-postgres.service
[Container]
Image=docker.io/vikunja/vikunja:latest
ContainerName=vikunja
AutoUpdate=registry
Network=vikunja-internal.network
# Fichiers (pièces jointes des tâches) : volume dédié
Volume=%h/vikunja/files:/app/vikunja/files:U,Z

Environment=VIKUNJA_DATABASE_TYPE=postgres
Environment=VIKUNJA_DATABASE_HOST=vikunja-postgres
Environment=VIKUNJA_DATABASE_DATABASE=vikunja
Environment=VIKUNJA_DATABASE_USER=vikunja
Environment=VIKUNJA_SERVICE_PUBLICURL=https://tasks.mondomaine.fr
Secret=vikunja_db_password,type=env,target=VIKUNJA_DATABASE_PASSWORD
# Clé JWT pour les sessions
Secret=vikunja_jwt_secret,type=env,target=VIKUNJA_SERVICE_JWTSECRET

NoNewPrivileges=true
DropCapability=ALL

[Service]
Restart=on-failure
RestartSec=10
MemoryMax=512M

[Install]
WantedBy=default.target
```

> [!info] L'image unifiée
> Les versions récentes de Vikunja combinent l'API et le frontend dans une seule image qui sert tout sur un port unique. Plus besoin du double conteneur api+frontend des anciens guides. Vérifie la doc de la version que tu déploies, le projet a évolué sur ce point.

```bash
systemctl --user daemon-reload
systemctl --user start vikunja.service
```

---

## Exposition

```caddyfile
tasks.mondomaine.fr {
	import security_headers
	reverse_proxy vikunja:3456
}
```

> [!tip] Public ou VPN
> Vikunja a des apps mobiles et de bureau. Pour gérer tes tâches en mobilité, l'exposition publique est pratique (les tâches ne sont en général pas ultra-sensibles, mais ça dépend de ce que tu y mets). Avec inscription fermée (`VIKUNJA_SERVICE_ENABLEREGISTRATION=false` après création des comptes) et HTTPS strict, c'est raisonnable. Sinon, VPN.

> [!warning] Désactive l'inscription après setup
> Comme Vaultwarden, pense à couper l'inscription publique une fois tes comptes créés, pour éviter qu'un inconnu se crée un accès sur une instance exposée.

---

## Sauvegarde

> [!success] Quoi sauvegarder
> - **Dump SQL** de `vikunja-postgres` : projets, tâches, échéances, comptes, partages.
> - **`~/vikunja/files`** : pièces jointes des tâches.
> - **Le JWT secret** : sans lui, les sessions sont invalidées (moins critique qu'une clé de chiffrement, mais à conserver).

```bash
podman exec vikunja-postgres pg_dump -U vikunja -Fc vikunja > ~/backup/dumps/vikunja-$(date +%F).dump
```

---

## Conclusion

Vikunja est un service sans piège : app unifiée + PostgreSQL, durcissement standard, pattern de sauvegarde habituel. Le seul réflexe à ne pas oublier est de fermer l'inscription après création des comptes si tu l'exposes. Un bon outil pour organiser, dans la lignée propre de la série.

> [!note] Articles liés
> - [[Installer n8n sous Podman rootless]]
> - [[Installer Homepage ou Homarr sous Podman rootless]]

---

*Retour d'expérience personnel. Adapte domaines et images ; vérifie le mode d'image (unifiée) de ta version.*
