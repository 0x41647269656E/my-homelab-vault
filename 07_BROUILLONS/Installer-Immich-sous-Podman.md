---
title: Installer Immich sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - immich
  - podman
  - installation
  - securite
date: 06-06-2026
last_modified: 06-06-2026
reading-time: 15m
status: draft
---

# Installer Immich sous Podman rootless

> [!abstract] TL;DR
> Immich est une galerie photo auto-hébergée avec ML (reconnaissance faciale, recherche sémantique). C'est un service **multi-conteneurs** : serveur, machine learning, PostgreSQL (avec extension vectorielle `pgvecto.rs`) et Redis. Je l'installe en **Podman rootless** via Quadlets, avec ma bibliothèque photo existante montée depuis `/media` en **lecture seule** pour l'import externe, et les données générées (miniatures, transcodages) sur un volume dédié inscriptible. Cet article assemble la stack en cohérence avec ma posture sécurité : réseau interne, accès via Caddy + WireGuard, confinement MAC, sauvegarde.

> [!info] Prérequis de lecture
> Cet article suppose le socle des articles précédents : [[Self-hosting sécurisé avec Podman]] (Quadlets, rootless, `:U`/`:Z`), [[Reverse proxy Caddy avec TLS automatique]], [[Configuration WireGuard self-hosted]], et un confinement MAC ([[Durcissement SELinux pour conteneurs]] ou [[Durcissement AppArmor pour conteneurs]]). Je ne réexplique pas ces fondations.

---

## La règle d'or pour monter `/media` : jamais `:U` sur tes données sources

C'est le point le plus important de tous ces articles d'installation, alors je le pose une fois, ici, et j'y renverrai ensuite.

> [!danger] `:U` sur `/media` détruirait tes permissions
> Le flag `:U` vu dans les articles précédents fait un `chown` **récursif** du volume vers l'UID mappé du conteneur. Sur un volume de données vide créé pour le service, c'est parfait. Sur **`/media`, qui contient des To de fichiers existants partagés potentiellement entre plusieurs services**, c'est une catastrophe : tu réécris la propriété de toute ta bibliothèque, et le prochain conteneur qui monte le même `/media` avec son propre `:U` re-`chown` tout, cassant l'accès du premier.

> [!success] La bonne pratique pour `/media`
> - **Monter `/media` en lecture seule** (`:ro`) chaque fois que le service n'a qu'à **lire** tes données sources (import de photos existantes, indexation de documents). Lecture seule = aucun risque sur tes fichiers, et un conteneur compromis ne peut pas les altérer.
> - **Utiliser `keep-id`** plutôt que `:U` quand le service doit écrire : `keep-id` mappe ton UID hôte vers le même UID dans le conteneur, sans rien `chown`. Les permissions Unix existantes de `/media` restent valides.
> - **Séparer données sources et données générées** : `/media` (tes originaux, en lecture seule) d'un côté ; un volume dédié inscriptible (`~/immich/...`) pour ce que le service produit, de l'autre.
>
> Sous SELinux, ajoute `:z` (minuscule, **partagé**) sur `/media` puisque plusieurs services le liront — jamais `:Z` majuscule qui le verrouillerait à un seul conteneur, ni de relabel destructeur. Sous AppArmor, rien à faire (cf. variante AppArmor).

Cette règle structure tous les montages `/media` des six articles. Retiens : **`/media` en `:ro` ou `keep-id`, données générées sur volume dédié.**

---

## L'architecture d'Immich : quatre conteneurs

Immich n'est pas un binaire unique. Il faut orchestrer :

- **`immich-server`** — l'API et l'interface web, le point d'entrée.
- **`immich-machine-learning`** — la reconnaissance faciale, le CLIP pour la recherche sémantique. Gros consommateur de RAM.
- **`immich-postgres`** — PostgreSQL avec l'extension vectorielle, indispensable pour la recherche par similarité.
- **`immich-redis`** — file de jobs et cache.

Les quatre vivent sur un **réseau Podman interne** dédié ; seul `immich-server` est exposé, et uniquement à travers Caddy.

```bash
# Réseau interne (sans accès Internet sortant)
podman network create immich-internal --internal
```

> [!warning] Le réseau ML peut avoir besoin de sortir
> Au premier démarrage, le conteneur ML télécharge ses modèles. Soit tu le laisses temporairement sur un réseau avec accès Internet le temps du téléchargement, soit tu pré-télécharges les modèles dans un volume. Une fois les modèles en cache, le `--internal` strict convient.

---

## Les Quadlets

Je place tout dans `~/.config/containers/systemd/`. Voici la stack, conteneur par conteneur.

### PostgreSQL avec extension vectorielle

```ini
# ~/.config/containers/systemd/immich-postgres.container
[Unit]
Description=Immich PostgreSQL (pgvecto.rs)

[Container]
Image=docker.io/tensorchord/pgvecto-rs:pg16-v0.2.0
ContainerName=immich-postgres
AutoUpdate=registry

Network=immich-internal.network

# Données de la base sur volume dédié (PAS dans /media) : :U OK ici, volume vierge
Volume=%h/immich/pgdata:/var/lib/postgresql/data:U,Z

Environment=POSTGRES_USER=immich
Environment=POSTGRES_DB=immich
Secret=immich_db_password,type=env,target=POSTGRES_PASSWORD

# Durcissement
NoNewPrivileges=true
DropCapability=ALL
AddCapability=CAP_CHOWN,CAP_SETUID,CAP_SETGID,CAP_DAC_OVERRIDE

[Service]
Restart=on-failure
RestartSec=10
MemoryMax=2G

[Install]
WantedBy=default.target
```

### Redis

```ini
# ~/.config/containers/systemd/immich-redis.container
[Unit]
Description=Immich Redis

[Container]
Image=docker.io/library/redis:7-alpine
ContainerName=immich-redis
AutoUpdate=registry
Network=immich-internal.network

NoNewPrivileges=true
DropCapability=ALL
ReadOnly=true
Tmpfs=/data:rw,size=256M

[Service]
Restart=on-failure
MemoryMax=512M

[Install]
WantedBy=default.target
```

### Machine Learning

```ini
# ~/.config/containers/systemd/immich-machine-learning.container
[Unit]
Description=Immich Machine Learning

[Container]
Image=ghcr.io/immich-app/immich-machine-learning:release
ContainerName=immich-machine-learning
AutoUpdate=registry
Network=immich-internal.network

# Cache des modèles ML sur volume dédié inscriptible
Volume=%h/immich/model-cache:/cache:U,Z

NoNewPrivileges=true
DropCapability=ALL

[Service]
Restart=on-failure
RestartSec=10
# Le ML est gourmand : on plafonne pour protéger l'hôte
MemoryMax=4G

[Install]
WantedBy=default.target
```

### Serveur (le cœur, avec les montages `/media`)

```ini
# ~/.config/containers/systemd/immich-server.container
[Unit]
Description=Immich Server
Requires=immich-postgres.service immich-redis.service
After=immich-postgres.service immich-redis.service immich-machine-learning.service

[Container]
Image=ghcr.io/immich-app/immich-server:release
ContainerName=immich-server
AutoUpdate=registry
Network=immich-internal.network

# --- Données GÉNÉRÉES par Immich (uploads app mobile, miniatures, transcodages) ---
# Volume dédié inscriptible : :U légitime, volume propre à Immich
Volume=%h/immich/library:/usr/src/app/upload:U,Z

# --- Bibliothèque EXISTANTE depuis /media, en LECTURE SEULE (import externe) ---
# Pas de :U ! :ro protège les originaux. :z (partagé) côté SELinux.
Volume=/media/pictures:/mnt/media/pictures:ro,z

Environment=DB_HOSTNAME=immich-postgres
Environment=DB_USERNAME=immich
Environment=DB_DATABASE_NAME=immich
Environment=REDIS_HOSTNAME=immich-redis
Environment=IMMICH_MACHINE_LEARNING_URL=http://immich-machine-learning:3003
Secret=immich_db_password,type=env,target=DB_PASSWORD

# Durcissement
NoNewPrivileges=true
DropCapability=ALL
ReadOnly=true
Tmpfs=/tmp:rw,size=2G

[Service]
Restart=on-failure
RestartSec=10
MemoryMax=4G

[Install]
WantedBy=default.target
```

> [!success] La distinction qui rend tout sain
> - `~/immich/library` (`:U,Z`) → ce qu'**Immich produit** (uploads mobiles, miniatures). Volume dédié, inscriptible, sauvegardé.
> - `/media/pictures` (`:ro,z`) → tes **originaux existants**, exposés en lecture seule comme **bibliothèque externe** Immich. Immich les indexe sans jamais y toucher. Un Immich compromis ne peut pas chiffrer ni supprimer tes photos sources.

### Importer la bibliothèque externe

Une fois la stack démarrée, on déclare le chemin externe dans Immich (interface admin → External Libraries) en pointant `/mnt/media/pictures` — le chemin **tel que vu dans le conteneur**. Immich scanne et indexe sans copier ni modifier les fichiers.

```bash
loginctl enable-linger $USER   # rappel : indispensable en rootless headless
systemctl --user daemon-reload
systemctl --user start immich-server.service
journalctl --user -u immich-server.service -f
```

---

## Exposition : Caddy + accès mobile

Immich est l'un des rares services que j'expose publiquement (auto-upload mobile en déplacement). Le bloc Caddy, repris de l'article reverse proxy :

```caddyfile
photos.mondomaine.fr {
	import security_headers
	request_body {
		max_size 5GB   # gros uploads vidéo
	}
	reverse_proxy immich-server:2283
}
```

> [!warning] Exposer Immich = surface d'attaque publique
> C'est un service joignable depuis Internet manipulant des données intimes. Active l'**authentification forte** (Immich supporte OAuth/OIDC), garde `AutoUpdate=registry` actif pour patcher vite les CVE, et surveille les logs d'accès. Si l'usage mobile nomade n'est pas indispensable, garde-le derrière WireGuard comme Paperless.

---

## Sauvegarde

Conforme à [[Stratégie de sauvegarde restic 3-2-1]] :

> [!success] Quoi sauvegarder pour Immich
> - **Dump SQL** de `immich-postgres` (cohérent, jamais le volume PG brut). Inclut tout l'index, les visages, les albums.
> - **`~/immich/library`** : les uploads mobiles et données générées. À sauvegarder.
> - **PAS `/media/pictures`** : tes originaux ont leur propre sauvegarde, ils ne sont pas « produits » par Immich. Les re-sauvegarder via Immich serait redondant.
> - **Le model-cache ML** : inutile de le sauvegarder, il se re-télécharge.

```bash
podman exec immich-postgres pg_dump -U immich -Fc immich > ~/backup/dumps/immich-$(date +%F).dump
```

---

## Conclusion

Immich illustre le pattern qui reviendra dans tous ces articles : **séparer les données sources (`/media`, lecture seule) des données générées (volume dédié, inscriptible)**, orchestrer un multi-conteneurs sur un réseau interne, n'exposer que le strict nécessaire via Caddy, et sauvegarder la base par dump cohérent. La bibliothèque externe en `:ro` est la clé : Immich enrichit tes photos sans jamais pouvoir les abîmer.

> [!note] Articles liés
> - [[Self-hosting sécurisé avec Podman]]
> - [[Reverse proxy Caddy avec TLS automatique]]
> - [[Stratégie de sauvegarde restic 3-2-1]]
> - [[Installer Paperless-ngx sous Podman rootless]]

---

*Retour d'expérience personnel. Adapte chemins, domaines et images à ton parc. Vérifie les noms d'images et variables d'environnement Immich à jour : le projet évolue vite.*
