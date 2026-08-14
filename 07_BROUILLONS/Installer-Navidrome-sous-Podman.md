---
title: Installer Navidrome sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - navidrome
  - podman
  - installation
  - securite
  - media
date: 07-06-2026
last_modified: 07-06-2026
reading-time: 10m
difficulty: tech-enthusiast
status: draft
---

# Installer Navidrome sous Podman rootless

> [!abstract] TL;DR
> Navidrome est un serveur de streaming musical léger, compatible avec l'API Subsonic (donc une foule d'apps clientes : DSub, play:Sub, Symfonium…). Écrit en Go, **un seul conteneur minuscule**, très économe. Comme Jellyfin, c'est un consommateur en **lecture seule** de ta bibliothèque musicale dans `/media`. Installation Podman rootless triviale ; l'intérêt est la légèreté et l'écosystème Subsonic.

> [!info] Prérequis de lecture
> Socle de la série, voir [[Installer Jellyfin sous Podman rootless]] pour le pattern média identique. Règle `/media` : [[Installer Immich sous Podman rootless]].

---

## Un service modèle pour le rootless

Navidrome coche toutes les cases du conteneur idéal : binaire Go unique, pas de base externe (SQLite intégré), lecture seule sur les sources, faible empreinte mémoire. Le durcissement complet s'applique sans friction.

```ini
# ~/.config/containers/systemd/navidrome.container
[Unit]
Description=Navidrome Music Server

[Container]
Image=docker.io/deluan/navidrome:latest
ContainerName=navidrome
AutoUpdate=registry
Network=navidrome-internal.network

# Données Navidrome (base SQLite, cache) : volume dédié
Volume=%h/navidrome/data:/data:U,Z

# Bibliothèque musicale en LECTURE SEULE
Volume=/media/music:/music:ro,z

Environment=ND_SCANSCHEDULE=1h
Environment=ND_LOGLEVEL=info
# Transcodage à la volée pour économiser la bande passante mobile
Environment=ND_ENABLETRANSCODINGCONFIG=true

# Durcissement complet : Navidrome n'a besoin de rien de spécial
NoNewPrivileges=true
DropCapability=ALL
ReadOnly=true
Tmpfs=/tmp:rw,size=256M

[Service]
Restart=on-failure
RestartSec=10
MemoryMax=512M

[Install]
WantedBy=default.target
```

> [!success] `ReadOnly=true` possible ici
> Contrairement à GitLab ou HA, Navidrome accepte un rootfs en lecture seule : tout son état est dans `/data` (volume) et `/tmp` (tmpfs). C'est le profil de durcissement le plus propre de toute la série.

```bash
podman network create navidrome-internal --internal
systemctl --user daemon-reload
systemctl --user start navidrome.service
```

---

## Exposition

```caddyfile
music.mondomaine.fr {
	import security_headers
	reverse_proxy navidrome:4533
}
```

> [!info] L'écosystème Subsonic implique souvent une exposition
> L'intérêt de Navidrome est d'écouter sa musique en mobilité via une app Subsonic. Ça pousse vers une exposition publique. L'API Subsonic transmet historiquement les identifiants de façon faible (token/salt) : expose en HTTPS strict (Caddy s'en charge), utilise un mot de passe dédié non réutilisé, et si tu peux te contenter du VPN pour le mobile, c'est plus sûr.

---

## Sauvegarde

> [!success] Quoi sauvegarder
> - **`~/navidrome/data`** : base SQLite (playlists, favoris, historique d'écoute, comptes). Petit, à sauvegarder. Applique la prudence SQLite (`.backup`) si tu veux une copie à chaud cohérente.
> - **PAS `/media/music`** : sauvegardé par ailleurs.

---

## Conclusion

Navidrome est le service le plus simple de la série : léger, lecture seule, durcissable au maximum. Si tu veux du streaming musical sans la lourdeur de Jellyfin, c'est le choix net. Son seul point d'attention est l'authentification Subsonic en cas d'exposition publique.

> [!note] Articles liés
> - [[Installer Jellyfin sous Podman rootless]]
> - [[Installer Audiobookshelf sous Podman rootless]]

---

*Retour d'expérience personnel. Adapte chemins et domaines.*
