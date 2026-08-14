---
title: Installer Dozzle sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - dozzle
  - logs
  - monitoring
  - podman
  - installation
  - securite
date: 07-06-2026
last_modified: 07-06-2026
reading-time: 10m
difficulty: tech-enthusiast
status: draft
---

# Installer Dozzle sous Podman rootless

> [!abstract] TL;DR
> Dozzle est un visualiseur de **logs de conteneurs en temps réel** dans le navigateur : pas de base, pas de stockage, il lit les logs en direct et les affiche joliment. Léger et sans état. Le point sensible — et il est important — : Dozzle a besoin d'accéder au **socket Podman** pour lire les logs, ce qui touche directement au sujet de sécurité du tout premier article de la série. On va le faire **proprement**, en lecture seule.

> [!info] Prérequis de lecture
> Socle de la série, et impérativement [[Self-hosting sécurisé avec Podman]] sur la sensibilité du socket. Caddy, VPN, MAC.

---

## Le sujet central : l'accès au socket

> [!danger] Le socket conteneur est un actif sensible
> Comme rappelé dans [[Self-hosting sécurisé avec Podman]], le socket de l'API conteneur est puissant : qui y accède en écriture peut créer, détruire, monter des conteneurs — c'est une voie d'escalade. Dozzle a besoin de **lire** les logs via ce socket. La règle : exposer le socket **en lecture seule**, et profiter du fait qu'en **rootless**, le socket est celui de **ton utilisateur** (pas un socket root système), ce qui limite déjà la portée.

> [!success] Pourquoi le rootless aide ici
> En rootless, le socket Podman appartient à ton utilisateur non privilégié. Même monté dans Dozzle, il ne donne pas les clés de l'hôte comme le ferait le socket Docker root. C'est exactement l'argument de fond du premier article : le rootless réduit le blast radius, y compris pour les outils qui ont besoin du socket.

---

## Activer le socket Podman utilisateur

```bash
# Activer l'API Podman en socket utilisateur (rootless)
systemctl --user enable --now podman.socket
# Le socket vit dans $XDG_RUNTIME_DIR/podman/podman.sock
```

---

## Le Quadlet

```ini
# ~/.config/containers/systemd/dozzle.container
[Unit]
Description=Dozzle

[Container]
Image=docker.io/amir20/dozzle:latest
ContainerName=dozzle
AutoUpdate=registry
Network=monitoring-internal.network

# Socket Podman monté en LECTURE SEULE (:ro) — non négociable
Volume=%t/podman/podman.sock:/var/run/docker.sock:ro

Environment=DOZZLE_NO_ANALYTICS=true

# Durcissement
NoNewPrivileges=true
DropCapability=ALL
ReadOnly=true

[Service]
Restart=on-failure
RestartSec=10
MemoryMax=256M

[Install]
WantedBy=default.target
```

> [!info] `%t` et le chemin du socket
> Dans un Quadlet utilisateur, `%t` correspond à `$XDG_RUNTIME_DIR` — donc `%t/podman/podman.sock` pointe vers ton socket Podman rootless. Dozzle attend historiquement le socket à l'emplacement Docker (`/var/run/docker.sock`), d'où le mapping. Le `:ro` garantit que Dozzle ne peut que **lire**.

```bash
podman network create monitoring-internal --internal 2>/dev/null || true
systemctl --user daemon-reload
systemctl --user start dozzle.service
```

---

## Exposition

```caddyfile
logs.mondomaine.fr {
	import security_headers
	@allowed remote_ip 10.10.1.0/24   # admin uniquement : les logs révèlent beaucoup
	handle @allowed {
		reverse_proxy dozzle:8080
	}
	respond "Forbidden" 403
}
```

> [!warning] Les logs sont sensibles
> Les logs de conteneurs peuvent contenir des fragments de données, des tokens, des chemins, des erreurs révélatrices. Dozzle reste **admin-only**, jamais exposé. Active aussi l'authentification intégrée de Dozzle en complément du filtrage IP.

---

## Sauvegarde

> [!success] Rien à sauvegarder
> Dozzle est **sans état** : il lit les logs en direct, ne stocke rien. Aucun volume de données à sauvegarder. C'est un pur outil de visualisation.

---

## Conclusion

Dozzle est l'outil de confort par excellence pour suivre ses logs sans jongler avec `journalctl`. Son seul enjeu — réel — est l'accès au socket, qu'on traite en **lecture seule** et qu'on rend acceptable par le **rootless** (socket utilisateur, pas root). C'est une bonne illustration concrète de pourquoi le choix du rootless du premier article paie même pour les outils périphériques.

> [!note] Articles liés
> - [[Self-hosting sécurisé avec Podman]]
> - [[Installer Uptime Kuma sous Podman rootless]]

---

*Retour d'expérience personnel. Adapte domaines et plages VPN. Vérifie le chemin du socket selon ta config rootless.*
