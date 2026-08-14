---
title: Installer Beszel ou Netdata sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - beszel
  - netdata
  - monitoring
  - podman
  - installation
  - securite
date: 07-06-2026
last_modified: 07-06-2026
reading-time: 15m
difficulty: tech-enthusiast
status: draft
---

# Installer Beszel ou Netdata sous Podman rootless

> [!abstract] TL;DR
> Deux approches du **monitoring de ressources** (CPU, RAM, disque, réseau, températures). **Beszel** est léger, moderne, multi-serveurs avec une architecture hub + agent, idéal pour un parc self-hosted modeste. **Netdata** est extrêmement détaillé (des milliers de métriques par seconde), puissant mais plus lourd et bavard. Je présente les deux et t'oriente. Complément naturel d'Uptime Kuma (disponibilité) et Dozzle (logs) : ici on regarde la **santé matérielle**.

> [!info] Prérequis de lecture
> Socle de la série. Caddy, VPN, MAC, restic. Complémentaire de [[Installer Uptime Kuma sous Podman rootless]] et [[Installer Dozzle sous Podman rootless]].

---

## La trilogie de l'observabilité

> [!success] Trois outils, trois questions
> - **Uptime Kuma** : « mes services répondent-ils ? » (disponibilité)
> - **Dozzle** : « que disent-ils ? » (logs)
> - **Beszel/Netdata** : « comment se porte la machine ? » (ressources)
>
> Les trois se complètent. Beszel/Netdata est celui qui te prévient **avant** la panne : disque qui se remplit, RAM saturée, température qui grimpe. C'est la maintenance préventive.

---

## Choisir : Beszel ou Netdata ?

> [!info] Le bon choix
> - **Beszel** → léger, joli, **multi-serveurs natif** (un hub central + des agents sur chaque machine), alertes simples, faible empreinte. Parfait pour surveiller un ou quelques hôtes sans se noyer dans les métriques.
> - **Netdata** → granularité extrême (par seconde, par processus, par conteneur), détection d'anomalies, immense couverture. Mais plus gourmand, et l'envoi de données vers Netdata Cloud est à désactiver si tu veux rester 100 % local.
>
> Pour un parc self-hosted perso, **Beszel** est souvent le meilleur rapport simplicité/valeur. Netdata si tu veux du diagnostic fin et que la charge ne te dérange pas.

---

## Option A — Beszel (hub + agent)

Beszel sépare le **hub** (interface + base) de l'**agent** (collecte sur chaque machine). Sur un hôte unique, les deux cohabitent.

```bash
podman network create monitoring-internal --internal 2>/dev/null || true
```

```ini
# ~/.config/containers/systemd/beszel-hub.container
[Unit]
Description=Beszel Hub
[Container]
Image=ghcr.io/henrygd/beszel/beszel:latest
ContainerName=beszel-hub
AutoUpdate=registry
Network=monitoring-internal.network
Volume=%h/beszel/data:/beszel_data:U,Z
NoNewPrivileges=true
DropCapability=ALL
[Service]
Restart=on-failure
MemoryMax=256M
[Install]
WantedBy=default.target
```

```ini
# ~/.config/containers/systemd/beszel-agent.container
[Unit]
Description=Beszel Agent
[Container]
Image=ghcr.io/henrygd/beszel/beszel-agent:latest
ContainerName=beszel-agent
AutoUpdate=registry
Network=monitoring-internal.network
# L'agent lit les métriques système ; accès en lecture aux infos hôte
Volume=/proc:/host/proc:ro
# Clé publique fournie par le hub pour authentifier l'agent
Environment=KEY=ssh-ed25519_AAAA...
Environment=PORT=45876
NoNewPrivileges=true
DropCapability=ALL
[Service]
Restart=on-failure
MemoryMax=128M
[Install]
WantedBy=default.target
```

> [!info] Le couplage hub-agent
> Le hub génère une clé que tu colles dans la config de l'agent (`KEY=`). L'agent expose ses métriques, le hub les collecte de façon authentifiée. Pour ajouter une autre machine plus tard, tu déploies juste un agent supplémentaire pointant vers le même hub — c'est l'intérêt de l'architecture.

---

## Option B — Netdata

```ini
# ~/.config/containers/systemd/netdata.container
[Unit]
Description=Netdata
[Container]
Image=docker.io/netdata/netdata:latest
ContainerName=netdata
AutoUpdate=registry
Network=monitoring-internal.network
HostName=mon-serveur
Volume=%h/netdata/config:/etc/netdata:U,Z
Volume=%h/netdata/lib:/var/lib/netdata:U,Z
Volume=%h/netdata/cache:/var/cache/netdata:U,Z
# Accès lecture aux métriques système
Volume=/proc:/host/proc:ro
Volume=/sys:/host/sys:ro
Environment=DO_NOT_TRACK=1
NoNewPrivileges=true
[Service]
Restart=on-failure
MemoryMax=1G
[Install]
WantedBy=default.target
```

> [!warning] Netdata et la confidentialité
> Netdata pousse vers son offre Cloud. `DO_NOT_TRACK=1` et l'absence de *claim* vers Netdata Cloud gardent tout local. Vérifie qu'aucune connexion sortante n'est établie si tu veux un monitoring strictement chez toi. Netdata accède aussi à beaucoup de `/proc` et `/sys` — c'est nécessaire à sa granularité, mais c'est un accès étendu à assumer.

---

## Exposition

```caddyfile
metrics.mondomaine.fr {
	import security_headers
	@allowed remote_ip 10.10.1.0/24   # admin uniquement
	handle @allowed {
		reverse_proxy beszel-hub:8090   # ou netdata:19999
	}
	respond "Forbidden" 403
}
```

> [!warning] Les métriques révèlent ton infra
> Le détail des ressources, processus et conteneurs renseigne un attaquant sur ta machine. Monitoring **admin-only**, jamais public.

---

## Sauvegarde

> [!success] Quoi sauvegarder
> - **Beszel** : `~/beszel/data` (config, historique, alertes, comptes).
> - **Netdata** : `~/netdata/config` (config) et éventuellement `~/netdata/lib` (historique). Le cache est jetable.
> - Faible priorité globalement : le monitoring est reconstructible, on perd surtout l'historique.

---

## Conclusion

Beszel ou Netdata complète la trilogie d'observabilité avec la **santé matérielle** — la maintenance préventive qui évite les pannes plutôt que les constater. Beszel pour la simplicité et le multi-serveurs léger ; Netdata pour la granularité maximale, au prix d'un accès système étendu et d'une vigilance sur le « tout local ». Comme tout le monitoring : admin-only.

> [!note] Articles liés
> - [[Installer Uptime Kuma sous Podman rootless]]
> - [[Installer Dozzle sous Podman rootless]]

---

*Retour d'expérience personnel. Adapte domaines, plages VPN, images. Désactive toute télémétrie sortante si tu veux rester 100 % local.*
