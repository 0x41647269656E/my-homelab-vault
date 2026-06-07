---
title: Installer Calibre-Web sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - calibre-web
  - podman
  - installation
  - securite
  - media
date: 07-06-2026
last_modified: 07-06-2026
reading-time: 10m
status: draft
---

# Installer Calibre-Web sous Podman rootless

> [!abstract] TL;DR
> Calibre-Web est une interface web élégante pour parcourir, lire et télécharger une **bibliothèque Calibre existante** (ebooks). Attention au piège : Calibre-Web **n'est pas Calibre** — il s'appuie sur une bibliothèque Calibre (avec son fichier `metadata.db`) que tu dois déjà avoir, ou créer. Conteneur unique. Le montage `/media` doit être **inscriptible** si tu veux ajouter des livres via l'interface, sinon lecture seule.

> [!info] Prérequis de lecture
> Socle de la série. Pattern média : [[Installer Jellyfin sous Podman rootless]]. Règle `/media` : [[Installer Immich sous Podman rootless]].

---

## Le piège : Calibre-Web a besoin d'une bibliothèque Calibre

> [!warning] metadata.db est obligatoire
> Calibre-Web ne sait pas fonctionner sur un simple dossier d'ePub en vrac. Il lui faut une **bibliothèque Calibre** structurée, identifiée par un fichier `metadata.db` à sa racine. Si tu pars de zéro, crée-la avec le Calibre desktop, ou laisse Calibre-Web en créer une vide au premier lancement. Pointe le volume vers le dossier **contenant `metadata.db`**.

> [!success] Lecture seule ou inscriptible, selon l'usage
> - Si tu **gères ta bibliothèque ailleurs** (Calibre desktop) et veux juste la consulter → monte `/media/books` en **lecture seule** (`:ro,z`).
> - Si tu veux **ajouter/éditer des livres via l'interface web** → montage **inscriptible** (`keep-id`), car Calibre-Web modifie alors `metadata.db` et écrit les fichiers.
>
> Choisis selon ton workflow. Je préfère la lecture seule + gestion desktop, pour garder `/media` intouchable.

---

## Le Quadlet

```ini
# ~/.config/containers/systemd/calibre-web.container
[Unit]
Description=Calibre-Web

[Container]
Image=ghcr.io/linuxserver/calibre-web:latest
ContainerName=calibre-web
AutoUpdate=registry
Network=calibre-web-internal.network

# Config Calibre-Web : volume dédié
Volume=%h/calibre-web/config:/config:U,Z

# Bibliothèque Calibre (doit contenir metadata.db)
# Lecture seule si gérée ailleurs :
Volume=/media/books:/books:ro,z

Environment=PUID=1000
Environment=PGID=1000
Environment=TZ=Europe/Paris

# Durcissement
NoNewPrivileges=true
DropCapability=ALL

[Service]
Restart=on-failure
RestartSec=10
MemoryMax=512M

[Install]
WantedBy=default.target
```

```bash
podman network create calibre-web-internal --internal
systemctl --user daemon-reload
systemctl --user start calibre-web.service
```

> [!info] Première connexion
> Identifiants par défaut : `admin` / `admin123`. **Change-les immédiatement.** Au premier lancement, Calibre-Web demande le chemin de la base Calibre : indique `/books` (le chemin **dans le conteneur**).

---

## Exposition

```caddyfile
library.mondomaine.fr {
	import security_headers
	@allowed remote_ip 10.10.1.0/24 10.10.2.0/24   # souvent VPN suffit pour des ebooks perso
	handle @allowed {
		reverse_proxy calibre-web:8083
	}
	respond "Forbidden" 403
}
```

> [!tip] Send-to-Kindle / OPDS
> Calibre-Web propose l'envoi par mail vers liseuse et un flux OPDS pour les apps de lecture (KOReader, Moon+ Reader…). Si tu veux le confort OPDS en mobilité, c'est un cas d'exposition publique ; sinon le VPN couvre l'usage domestique.

---

## Sauvegarde

> [!success] Quoi sauvegarder
> - **`~/calibre-web/config`** : comptes, étagères, réglages, base applicative.
> - **`metadata.db`** de la bibliothèque Calibre : le catalogue. S'il est dans `/media/books`, vérifie qu'il entre dans **une** sauvegarde (soit celle de `/media`, soit explicitement). C'est le fichier dont la perte fait le plus mal.
> - Les fichiers ebooks eux-mêmes suivent la sauvegarde de `/media`.

---

## Conclusion

Calibre-Web est simple une fois compris son prérequis : c'est une **interface sur une bibliothèque Calibre**, pas un gestionnaire autonome. Le `metadata.db` est le cœur du système. Mon conseil : gérer la bibliothèque au desktop, monter en lecture seule, garder `/media` intouchable.

> [!note] Articles liés
> - [[Installer Audiobookshelf sous Podman rootless]]
> - [[Installer Jellyfin sous Podman rootless]]

---

*Retour d'expérience personnel. Adapte chemins et domaines.*
