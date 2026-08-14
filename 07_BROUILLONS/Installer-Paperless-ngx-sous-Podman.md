---
title: Installer Paperless-ngx sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - paperless-ngx
  - podman
  - installation
  - securite
date: 06-06-2026
last_modified: 06-06-2026
reading-time: 15m
difficulty: tech-enthusiast
status: draft
---

# Installer Paperless-ngx sous Podman rootless

> [!abstract] TL;DR
> Paperless-ngx est une GED (gestion électronique de documents) qui ingère, OCR-ise, classe et indexe tous tes documents administratifs. Stack **multi-conteneurs** : application, PostgreSQL, Redis, et l'OCR (Tika/Gotenberg pour les formats bureautiques). Données extrêmement sensibles → **jamais exposé sur Internet**, accès WireGuard strict. Je monte un **dossier de consommation** depuis `/media` pour l'ingestion automatique, et je stocke l'archive indexée sur un volume dédié. Cet article assemble la stack en cohérence avec la posture sécurité de la série.

> [!info] Prérequis de lecture
> Socle : [[Self-hosting sécurisé avec Podman]], [[Reverse proxy Caddy avec TLS automatique]], [[Configuration WireGuard self-hosted]], confinement MAC, [[Stratégie de sauvegarde restic 3-2-1]]. La règle de montage `/media` (jamais `:U`, privilégier `:ro` ou `keep-id`) est détaillée dans [[Installer Immich sous Podman rootless]] — je m'appuie dessus.

---

## L'architecture : quatre rôles

- **`paperless-webserver`** — l'app Django, l'interface, l'API. Point d'entrée.
- **`paperless-postgres`** — la base : métadonnées, index, correspondants, types de documents.
- **`paperless-redis`** — file des tâches d'ingestion et d'OCR.
- **`paperless-gotenberg`** + **`paperless-tika`** — conversion et extraction de texte des formats bureautiques (Office, etc.). Optionnels mais quasi indispensables.

Tout sur un réseau interne ; seul le webserver est joignable, via Caddy, derrière VPN.

```bash
podman network create paperless-internal --internal
```

---

## Les trois types de données de Paperless

C'est le point structurant pour les volumes. Paperless distingue :

> [!success] data vs media vs consume
> - **`consume`** — le dossier **surveillé** : tout fichier déposé ici est automatiquement ingéré, OCR-isé, puis **supprimé** de ce dossier. C'est ma porte d'entrée depuis `/media`.
> - **`media`** (au sens Paperless) — l'**archive** des documents traités (originaux + versions PDF/A OCR-isées). Données générées/gérées par Paperless. Volume dédié inscriptible.
> - **`data`** — l'index de recherche, les modèles de classification. Volume dédié.
>
> Ne pas confondre le `media` **de Paperless** (son archive interne, sur volume dédié) avec **`/media` l'hôte** (tes données sources). Homonymie piégeuse.

> [!warning] Le dossier consume écrit dans /media
> Contrairement à Immich où `/media` était en lecture seule, ici le service **supprime** les fichiers consommés du dossier surveillé. Donc ce montage ne peut pas être `:ro`. J'utilise un sous-dossier **dédié à la consommation** dans `/media` (pas toute la racine), monté avec `keep-id` pour préserver les permissions sans `chown` destructeur.

---

## Les Quadlets

### PostgreSQL

```ini
# ~/.config/containers/systemd/paperless-postgres.container
[Unit]
Description=Paperless PostgreSQL

[Container]
Image=docker.io/library/postgres:16-alpine
ContainerName=paperless-postgres
AutoUpdate=registry
Network=paperless-internal.network

Volume=%h/paperless/pgdata:/var/lib/postgresql/data:U,Z

Environment=POSTGRES_DB=paperless
Environment=POSTGRES_USER=paperless
Secret=paperless_db_password,type=env,target=POSTGRES_PASSWORD

NoNewPrivileges=true
DropCapability=ALL
AddCapability=CAP_CHOWN,CAP_SETUID,CAP_SETGID,CAP_DAC_OVERRIDE

[Service]
Restart=on-failure
MemoryMax=1G

[Install]
WantedBy=default.target
```

### Redis, Gotenberg, Tika

```ini
# ~/.config/containers/systemd/paperless-redis.container
[Unit]
Description=Paperless Redis
[Container]
Image=docker.io/library/redis:7-alpine
ContainerName=paperless-redis
AutoUpdate=registry
Network=paperless-internal.network
NoNewPrivileges=true
DropCapability=ALL
ReadOnly=true
Tmpfs=/data:rw,size=128M
[Service]
Restart=on-failure
MemoryMax=256M
[Install]
WantedBy=default.target
```

```ini
# ~/.config/containers/systemd/paperless-gotenberg.container
[Unit]
Description=Paperless Gotenberg
[Container]
Image=docker.io/gotenberg/gotenberg:8
ContainerName=paperless-gotenberg
AutoUpdate=registry
Network=paperless-internal.network
# Gotenberg veut désactiver certaines routes pour la sécurité
Exec=gotenberg --chromium-disable-javascript=true --chromium-allow-list="file:///tmp/.*"
NoNewPrivileges=true
DropCapability=ALL
[Service]
Restart=on-failure
MemoryMax=1G
[Install]
WantedBy=default.target
```

```ini
# ~/.config/containers/systemd/paperless-tika.container
[Unit]
Description=Paperless Tika
[Container]
Image=docker.io/apache/tika:latest
ContainerName=paperless-tika
AutoUpdate=registry
Network=paperless-internal.network
NoNewPrivileges=true
DropCapability=ALL
[Service]
Restart=on-failure
MemoryMax=1G
[Install]
WantedBy=default.target
```

### Le webserver (avec les montages)

```ini
# ~/.config/containers/systemd/paperless-webserver.container
[Unit]
Description=Paperless-ngx webserver
Requires=paperless-postgres.service paperless-redis.service
After=paperless-postgres.service paperless-redis.service paperless-gotenberg.service paperless-tika.service

[Container]
Image=ghcr.io/paperless-ngx/paperless-ngx:latest
ContainerName=paperless-webserver
AutoUpdate=registry
Network=paperless-internal.network

# --- Données GÉNÉRÉES par Paperless : volumes dédiés inscriptibles ---
Volume=%h/paperless/data:/usr/src/paperless/data:U,Z
Volume=%h/paperless/media:/usr/src/paperless/media:U,Z

# --- Dossier de CONSOMMATION depuis /media : keep-id (écriture, pas :ro) ---
# Sous-dossier dédié, JAMAIS toute la racine /media
Volume=/media/paperless-inbox:/usr/src/paperless/consume:z

Environment=PAPERLESS_DBHOST=paperless-postgres
Environment=PAPERLESS_DBUSER=paperless
Environment=PAPERLESS_REDIS=redis://paperless-redis:6379
Environment=PAPERLESS_TIKA_ENABLED=1
Environment=PAPERLESS_TIKA_GOTENBERG_ENDPOINT=http://paperless-gotenberg:3000
Environment=PAPERLESS_TIKA_ENDPOINT=http://paperless-tika:9998
Environment=PAPERLESS_URL=https://docs.mondomaine.fr
# OCR en français + anglais
Environment=PAPERLESS_OCR_LANGUAGE=fra+eng
Secret=paperless_db_password,type=env,target=PAPERLESS_DBPASS
Secret=paperless_secret_key,type=env,target=PAPERLESS_SECRET_KEY

NoNewPrivileges=true
DropCapability=ALL

[Service]
Restart=on-failure
RestartSec=15
MemoryMax=3G

[Install]
WantedBy=default.target
```

> [!info] Le mapping UID du dossier consume
> Pour que Paperless lise et nettoie `/media/paperless-inbox`, l'UID du processus dans le conteneur doit avoir les droits sur ce dossier côté hôte. Avec `keep-id` (ou en réglant `PAPERLESS_USER_ID`/`PAPERLESS_GROUP_ID` selon le propriétaire réel du dossier), on évite tout `chown` destructeur. Crée le dossier avec ton UID, dépose tes scans dedans (scanner réseau, script, etc.), Paperless fait le reste.

```bash
mkdir -p /media/paperless-inbox
systemctl --user daemon-reload
systemctl --user start paperless-webserver.service
# Créer le superutilisateur :
podman exec -it paperless-webserver python3 manage.py createsuperuser
```

---

## Exposition : VPN strict, jamais public

Le bloc Caddy, repris tel quel de l'article reverse proxy :

```caddyfile
docs.mondomaine.fr {
	import security_headers
	@allowed remote_ip 10.10.1.0/24 10.10.2.0/24   # admin + famille, jamais amis ni Internet
	handle @allowed {
		reverse_proxy paperless-webserver:8000
	}
	respond "Forbidden" 403
}
```

> [!danger] Ce sont tes documents les plus sensibles
> Pièces d'identité, relevés bancaires, contrats. Paperless n'a **aucune raison** d'être joignable depuis Internet. Le double verrou (nftables limite l'accès au port 443 de Caddy + Caddy filtre par sous-réseau) garantit que seul le VPN admin/famille atteint le service. Cf. [[Configuration WireGuard self-hosted]] pour la matrice d'accès.

---

## Sauvegarde

> [!success] Quoi sauvegarder pour Paperless
> - **Dump SQL** de `paperless-postgres` (métadonnées, index, classification).
> - **`~/paperless/media`** : l'archive des documents (les fichiers eux-mêmes !). **Le plus important.**
> - **`~/paperless/data`** : index de recherche, modèles. Reconstructible mais long à régénérer.
> - **PAS `/media/paperless-inbox`** : c'est transitoire, vidé à chaque ingestion.

```bash
podman exec paperless-postgres pg_dump -U paperless -Fc paperless > ~/backup/dumps/paperless-$(date +%F).dump
```

> [!tip] Paperless a son propre exporteur
> Paperless fournit `document_exporter`, qui produit un export complet (documents + métadonnées en JSON) restaurable sur n'importe quelle instance. Je le lance périodiquement en complément du dump SQL — c'est ma garantie de pouvoir migrer ou restaurer même en cas de changement de version majeure :
> ```bash
> podman exec paperless-webserver document_exporter /usr/src/paperless/export
> ```

---

## Conclusion

Paperless illustre le cas du service qui **écrit** dans `/media` (le dossier consume) là où Immich ne faisait que lire. La parade reste la même philosophie : un **sous-dossier dédié** (jamais la racine `/media`), `keep-id` plutôt que `:U` destructeur, et une séparation nette entre la zone d'ingestion transitoire et l'archive durable sur volume dédié. Et comme pour Vaultwarden : ces données ne sortent jamais sur Internet.

> [!note] Articles liés
> - [[Configuration WireGuard self-hosted]]
> - [[Installer Immich sous Podman rootless]]
> - [[Stratégie de sauvegarde restic 3-2-1]]

---

*Retour d'expérience personnel. Adapte chemins, domaines, langues OCR et images. Vérifie les variables d'environnement Paperless à jour.*
