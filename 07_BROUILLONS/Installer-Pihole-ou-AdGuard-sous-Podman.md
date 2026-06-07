---
title: Installer Pi-hole ou AdGuard Home sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - pihole
  - adguard
  - dns
  - podman
  - installation
  - securite
  - reseau
date: 07-06-2026
last_modified: 07-06-2026
reading-time: 15m
status: draft
---

# Installer Pi-hole ou AdGuard Home sous Podman rootless

> [!abstract] TL;DR
> Pi-hole et AdGuard Home sont des **serveurs DNS filtrants** : ils bloquent pub et traçage pour tout le réseau, au niveau DNS. Au-delà du blocage, ils résolvent une vraie pièce manquante de la série : le **DNS split-horizon** évoqué dans l'article WireGuard, qui fait que `photos.mondomaine.fr` pointe vers l'IP interne quand on est dans le tunnel. Le défi technique central : un DNS doit écouter sur le **port 53**, ce qui demande une attention particulière en rootless.

> [!info] Prérequis de lecture
> Socle de la série, et surtout [[Configuration WireGuard self-hosted]] — cet article complète le DNS du VPN. Caddy, MAC, restic comme partout.

---

## La double valeur : filtrage ET DNS interne

> [!success] Pourquoi c'est plus qu'un bloqueur de pub
> 1. **Filtrage réseau** : blocage des domaines publicitaires/traçage pour tous les appareils, sans extension par navigateur. Effet immédiat sur tout le foyer.
> 2. **DNS split-horizon** : c'est le chaînon manquant de [[Configuration WireGuard self-hosted]]. En définissant des **entrées DNS locales** (`photos.mondomaine.fr → 10.10.0.1`), tes services résolvent vers leur **IP interne** quand tu es dans le VPN, au lieu de l'IP publique. Résultat : TLS valide (certificat wildcard de Caddy) **et** trafic qui ne sort jamais sur Internet pour des services internes.
>
> C'est cette deuxième fonction qui rend Pi-hole/AdGuard structurant dans ton architecture, pas juste confortable.

---

## Choisir : Pi-hole ou AdGuard Home ?

> [!info] Comparaison rapide
> - **Pi-hole** → l'historique, immense communauté, écosystème de listes de blocage très riche, interface orientée stats. Architecture historiquement un peu plus éclatée.
> - **AdGuard Home** → plus moderne et intégré (un seul binaire Go), **DNS chiffré natif** (DoH/DoT/DoQ) sans add-on, configuration plus simple du split-horizon.
>
> Pour une installation neuve avec besoin de DNS chiffré et de réécritures DNS simples, je penche **AdGuard Home**. Pour la richesse communautaire et l'habitude, Pi-hole. Je présente AdGuard Home, plus simple à intégrer ; le principe Pi-hole est analogue.

---

## Le défi : le port 53 en rootless

> [!warning] DNS et port privilégié
> Le DNS écoute sur le **port 53** (UDP et TCP), un port privilégié (< 1024). En rootless, c'est le même sujet que Caddy sur 80/443 (cf. [[Reverse proxy Caddy avec TLS automatique]]). Solution cohérente avec le reste du parc : abaisser le seuil des ports non privilégiés.
> ```bash
> # /etc/sysctl.d/99-rootless-ports.conf  (déjà posé pour Caddy)
> net.ipv4.ip_unprivileged_port_start=53
> ```
> Si tu as déjà mis `=80` pour Caddy, descends à `53` pour couvrir le DNS aussi. Alternative : redirection nftables depuis 53 vers un port haut.

> [!danger] Conflit avec le resolver système
> Sur beaucoup de distros, `systemd-resolved` occupe déjà le port 53 en local. Il faut le libérer (désactiver son écoute sur 53, ou le reconfigurer) avant qu'AdGuard/Pi-hole puisse binder. Vérifie avec `sudo ss -ulnp | grep :53`. C'est le piège d'installation n°1.

---

## Le Quadlet (AdGuard Home)

```ini
# ~/.config/containers/systemd/adguard.container
[Unit]
Description=AdGuard Home

[Container]
Image=docker.io/adguard/adguardhome:latest
ContainerName=adguard
AutoUpdate=registry

# DNS sur 53 (TCP+UDP). Interface d'admin sur 3000 (proxifiée par Caddy).
PublishPort=53:53/tcp
PublishPort=53:53/udp

# Config et données (statistiques, listes) : volumes dédiés
Volume=%h/adguard/work:/opt/adguardhome/work:U,Z
Volume=%h/adguard/conf:/opt/adguardhome/conf:U,Z

# Durcissement (le DNS a besoin de binder, on garde NET_BIND_SERVICE)
NoNewPrivileges=true
DropCapability=ALL
AddCapability=CAP_NET_BIND_SERVICE

[Service]
Restart=on-failure
RestartSec=10
MemoryMax=512M

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user start adguard.service
# Premier setup via http://<hôte>:3000 (assistant initial)
```

---

## La configuration split-horizon (le cœur)

Une fois AdGuard installé, c'est ici que la magie opère pour ton VPN.

> [!success] Les réécritures DNS
> Dans AdGuard (Filtres → Réécritures DNS) ou Pi-hole (Local DNS → DNS Records), ajoute :
> ```
> photos.mondomaine.fr   →  10.10.0.1
> docs.mondomaine.fr     →  10.10.0.1
> vault.mondomaine.fr    →  10.10.0.1
> *.mondomaine.fr        →  10.10.0.1   (si supporté : wildcard vers Caddy)
> ```
> `10.10.0.1` étant l'IP de l'hôte côté VPN où écoute Caddy. Désormais, un appareil sur le VPN qui demande `photos.mondomaine.fr` reçoit l'IP **interne** : le trafic va directement à Caddy via le tunnel, avec le certificat wildcard valide, sans détour par Internet.

> [!info] Brancher le VPN sur ce DNS
> Dans la config WireGuard des pairs (cf. [[Configuration WireGuard self-hosted]]), le champ `DNS = 10.10.0.1` pointait déjà vers l'hôte. AdGuard/Pi-hole devient ce résolveur : il applique le filtrage **et** les réécritures locales pour tous les pairs du tunnel. La boucle est bouclée.

---

## Exposition : interface d'admin derrière VPN

```caddyfile
dns.mondomaine.fr {
	import security_headers
	@allowed remote_ip 10.10.1.0/24   # admin uniquement : ça contrôle tout le DNS
	handle @allowed {
		reverse_proxy adguard:3000
	}
	respond "Forbidden" 403
}
```

> [!warning] Le service DNS contrôle toute ta résolution
> Qui contrôle l'admin du DNS peut rediriger n'importe quel domaine (phishing interne). L'interface reste **admin-only**. Le service DNS (port 53) lui-même n'a besoin d'être joignable que par ton réseau local et tes pairs VPN — pas par Internet. N'expose **jamais** un résolveur DNS ouvert sur Internet (risque d'abus en amplification DDoS).

---

## Sauvegarde

> [!success] Quoi sauvegarder
> - **`~/adguard/conf`** : toute ta config (réécritures DNS, listes de blocage, clients, réglages). C'est l'essentiel.
> - **`~/adguard/work`** : statistiques et données de requêtes. Optionnel (historique).

---

## Conclusion

Pi-hole/AdGuard Home apporte deux choses : le filtrage réseau, confortable, et surtout le **DNS split-horizon** qui complète enfin l'architecture WireGuard de la série — TLS valide sur des services internes sans les exposer. Les deux pièges à connaître : libérer le port 53 du resolver système, et ne jamais exposer le DNS ouvert sur Internet.

> [!note] Articles liés
> - [[Configuration WireGuard self-hosted]]
> - [[Reverse proxy Caddy avec TLS automatique]]
> - [[Installer Authelia ou Authentik sous Podman rootless]]

---

*Retour d'expérience personnel. Adapte domaines, IP VPN et images. Vérifie le conflit port 53 selon ta distro.*
