---
title: Installer Uptime Kuma sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - uptime-kuma
  - monitoring
  - podman
  - installation
  - securite
date: 07-06-2026
last_modified: 07-06-2026
reading-time: 10m
status: draft
---

# Installer Uptime Kuma sous Podman rootless

> [!abstract] TL;DR
> Uptime Kuma est un outil de **supervision de disponibilité** (le « est-ce que mes services répondent ? ») avec une interface soignée, des sondes variées (HTTP, TCP, ping, DNS, conteneur…) et des notifications (mail, Telegram, ntfy, webhooks). Conteneur unique, SQLite intégré. Installation Podman rootless triviale. Le point intéressant : **où le placer** pour qu'il détecte vraiment les pannes.

> [!info] Prérequis de lecture
> Socle de la série. Caddy, VPN, MAC, restic.

---

## Le Quadlet

```ini
# ~/.config/containers/systemd/uptime-kuma.container
[Unit]
Description=Uptime Kuma

[Container]
Image=docker.io/louislam/uptime-kuma:1
ContainerName=uptime-kuma
AutoUpdate=registry
Network=monitoring-internal.network

# Données (base SQLite, config, historique) : volume dédié
Volume=%h/uptime-kuma/data:/app/data:U,Z

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
podman network create monitoring-internal --internal
systemctl --user daemon-reload
systemctl --user start uptime-kuma.service
```

---

## Où le placer : le paradoxe du moniteur

> [!warning] Un moniteur sur la machine qu'il surveille a un angle mort
> Si Uptime Kuma tourne sur le même hôte que tes services et que **l'hôte entier tombe** (panne courant, crash kernel), le moniteur tombe avec — et ne t'alerte de rien. Il détecte très bien la panne d'**un service** parmi d'autres, mais pas la panne de **tout**.

> [!success] Les deux placements complémentaires
> 1. **Sur l'hôte (interne)** : surveille tes conteneurs entre eux, détecte qu'Immich ne répond plus alors que le reste tourne. Pratique et suffisant pour la plupart des pannes.
> 2. **Hors-site (idéal en complément)** : une instance Uptime Kuma chez un ami, sur un VPS minuscule, ou un service externe, qui surveille l'accessibilité **publique** de tes endpoints. C'est elle qui t'alerte si tout l'hôte tombe.
>
> Pour commencer, l'instance interne suffit. Si la disponibilité compte vraiment, ajoute une sonde externe.

---

## Exposition

```caddyfile
status.mondomaine.fr {
	import security_headers
	@allowed remote_ip 10.10.1.0/24
	handle @allowed {
		reverse_proxy uptime-kuma:3001
	}
	respond "Forbidden" 403
}
```

> [!tip] Les pages de statut publiques
> Uptime Kuma peut publier des **pages de statut** partageables (utile pour informer la famille « le serveur photo est en maintenance »). Si tu actives ça, cette page précise peut être exposée publiquement tout en gardant l'admin derrière VPN. Sépare les deux usages.

---

## Surveiller les conteneurs Podman

> [!info] Sonde « Docker/Podman container »
> Uptime Kuma propose un type de moniteur qui interroge le socket de l'API conteneur pour vérifier l'état des conteneurs. En rootless, exposer le socket Podman demande de la prudence (cf. la philosophie de [[Self-hosting sécurisé avec Podman]] sur le socket). Souvent, des sondes HTTP/TCP classiques sur les ports des services suffisent et évitent d'exposer le socket — je préfère cette approche plus sûre.

---

## Sauvegarde

> [!success] Quoi sauvegarder
> - **`~/uptime-kuma/data`** : base SQLite (moniteurs, historique, notifications, comptes). Petit, à sauvegarder. Prudence SQLite pour une copie à chaud cohérente.

---

## Conclusion

Uptime Kuma est simple et immédiatement utile. Le seul vrai choix d'architecture est le **placement** : une instance interne pour la granularité, idéalement complétée d'une sonde externe pour détecter la panne totale de l'hôte — l'angle mort classique de l'auto-supervision.

> [!note] Articles liés
> - [[Installer Dozzle sous Podman rootless]]
> - [[Installer Beszel ou Netdata sous Podman rootless]]

---

*Retour d'expérience personnel. Adapte domaines et plages VPN.*
