---
title: Installer Linkwarden ou Wallabag sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - linkwarden
  - wallabag
  - podman
  - installation
  - securite
  - productivite
date: 07-06-2026
last_modified: 07-06-2026
reading-time: 15m
difficulty: tech-enthusiast
status: draft
---

# Installer Linkwarden ou Wallabag sous Podman rootless

> [!abstract] TL;DR
> Deux services pour le même besoin — **sauvegarder et archiver des liens / lire plus tard** — avec deux philosophies. **Wallabag** est le « read-it-later » classique : il extrait le texte propre d'un article pour une lecture épurée. **Linkwarden** est un gestionnaire de favoris moderne qui **archive la page entière** (capture complète, PDF, capture d'écran) contre la disparition des contenus. Je présente les deux installations Podman rootless et t'aide à choisir. Accès VPN ou public selon ton usage mobile.

> [!info] Prérequis de lecture
> Socle de la série. Réseau interne, Caddy, VPN, MAC, restic.

---

## Choisir : Wallabag ou Linkwarden ?

> [!success] Le bon choix selon l'usage
> - **Wallabag** → tu veux **lire** des articles confortablement plus tard (mode lecture épuré, synchro liseuse, apps mobiles matures). Centré sur le **texte**.
> - **Linkwarden** → tu veux **archiver et organiser** des liens durablement, avec capture complète de la page (le contenu reste accessible même si le site disparaît), collections, tags, partage. Centré sur la **préservation et l'organisation**.
>
> Les deux peuvent coexister, mais pour la plupart des gens l'un des deux suffit. Si tu hésites : Wallabag pour le lecteur, Linkwarden pour l'archiviste.

---

## Option A — Wallabag

Architecture : app + base (PostgreSQL recommandé) + Redis optionnel.

```bash
podman network create wallabag-internal --internal
```

```ini
# ~/.config/containers/systemd/wallabag-postgres.container
[Unit]
Description=Wallabag PostgreSQL
[Container]
Image=docker.io/library/postgres:16-alpine
ContainerName=wallabag-postgres
AutoUpdate=registry
Network=wallabag-internal.network
Volume=%h/wallabag/pgdata:/var/lib/postgresql/data:U,Z
Environment=POSTGRES_DB=wallabag
Environment=POSTGRES_USER=wallabag
Secret=wallabag_db_password,type=env,target=POSTGRES_PASSWORD
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
# ~/.config/containers/systemd/wallabag.container
[Unit]
Description=Wallabag
Requires=wallabag-postgres.service
After=wallabag-postgres.service
[Container]
Image=docker.io/wallabag/wallabag:latest
ContainerName=wallabag
AutoUpdate=registry
Network=wallabag-internal.network
Volume=%h/wallabag/data:/var/www/wallabag/data:U,Z
Environment=SYMFONY__ENV__DATABASE_DRIVER=pdo_pgsql
Environment=SYMFONY__ENV__DATABASE_HOST=wallabag-postgres
Environment=SYMFONY__ENV__DATABASE_NAME=wallabag
Environment=SYMFONY__ENV__DATABASE_USER=wallabag
Environment=SYMFONY__ENV__DOMAIN_NAME=https://read.mondomaine.fr
Secret=wallabag_db_password,type=env,target=SYMFONY__ENV__DATABASE_PASSWORD
NoNewPrivileges=true
DropCapability=ALL
[Service]
Restart=on-failure
MemoryMax=768M
[Install]
WantedBy=default.target
```

```caddyfile
read.mondomaine.fr {
	import security_headers
	reverse_proxy wallabag:80
}
```

> [!info] Sauvegarde Wallabag
> Dump SQL de `wallabag-postgres` (tes articles, tags, annotations) + `~/wallabag/data`. Wallabag exporte aussi tes articles en plusieurs formats (JSON, ePub…) — utile pour la portabilité.

---

## Option B — Linkwarden

Architecture : app (Next.js) + PostgreSQL. Linkwarden lance des navigateurs headless pour archiver les pages, donc plus gourmand que Wallabag.

```bash
podman network create linkwarden-internal --internal
```

```ini
# ~/.config/containers/systemd/linkwarden-postgres.container
[Unit]
Description=Linkwarden PostgreSQL
[Container]
Image=docker.io/library/postgres:16-alpine
ContainerName=linkwarden-postgres
AutoUpdate=registry
Network=linkwarden-internal.network
Volume=%h/linkwarden/pgdata:/var/lib/postgresql/data:U,Z
Environment=POSTGRES_DB=linkwarden
Environment=POSTGRES_USER=linkwarden
Secret=linkwarden_db_password,type=env,target=POSTGRES_PASSWORD
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
# ~/.config/containers/systemd/linkwarden.container
[Unit]
Description=Linkwarden
Requires=linkwarden-postgres.service
After=linkwarden-postgres.service
[Container]
Image=ghcr.io/linkwarden/linkwarden:latest
ContainerName=linkwarden
AutoUpdate=registry
Network=linkwarden-internal.network
# Les archives de pages (PDF, captures) sont stockées ici : à sauvegarder
Volume=%h/linkwarden/data:/data/data:U,Z
Environment=DATABASE_URL=postgresql://linkwarden:PASS@linkwarden-postgres:5432/linkwarden
Environment=NEXTAUTH_URL=https://links.mondomaine.fr
Secret=linkwarden_nextauth_secret,type=env,target=NEXTAUTH_SECRET
NoNewPrivileges=true
DropCapability=ALL
[Service]
Restart=on-failure
# Le navigateur headless d'archivage est gourmand
MemoryMax=2G
[Install]
WantedBy=default.target
```

> [!warning] Le DATABASE_URL contient le mot de passe
> Linkwarden attend l'URL de connexion complète. Évite de mettre le mot de passe en clair dans le Quadlet : construis l'URL via un fichier d'environnement secret ou un script de démarrage qui assemble la variable depuis le secret. Ne laisse pas `PASS` en dur comme dans l'exemple simplifié ci-dessus.

```caddyfile
links.mondomaine.fr {
	import security_headers
	reverse_proxy linkwarden:3000
}
```

> [!success] Sauvegarde Linkwarden
> Dump SQL de `linkwarden-postgres` (liens, collections, tags) **et** surtout **`~/linkwarden/data`** : c'est là que vivent les **archives de pages** (PDF, captures, contenu complet). Sans ce volume, tu perds tout l'intérêt de Linkwarden — l'archivage. À sauvegarder absolument.

---

## Exposition : selon l'usage mobile

Les deux services ont des apps/extensions mobiles pour sauvegarder un lien en un tap. Comme pour FreshRSS, ça pousse vers l'exposition publique. Arbitrage habituel : public en HTTPS strict si tu sauvegardes des liens depuis ton téléphone toute la journée ; VPN si l'usage est surtout sédentaire. Ni l'un ni l'autre ne stocke de données ultra-sensibles, donc l'exposition est moins critique que Vaultwarden/Paperless.

---

## Conclusion

Wallabag et Linkwarden répondent au même besoin par deux angles : **lire** (Wallabag, texte épuré) ou **archiver** (Linkwarden, page complète préservée). Côté installation, même pattern : app + PostgreSQL sur réseau interne. La différence opérationnelle clé est que Linkwarden a un **volume d'archives** lourd et précieux à sauvegarder, là où Wallabag tient surtout dans sa base.

> [!note] Articles liés
> - [[Installer FreshRSS sous Podman rootless]]
> - [[Stratégie de sauvegarde restic 3-2-1]]

---

*Retour d'expérience personnel. Adapte chemins, domaines, images. Ne laisse jamais de mot de passe en clair dans un Quadlet : passe par les secrets.*
