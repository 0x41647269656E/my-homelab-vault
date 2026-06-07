---
title: Installer Headscale sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - headscale
  - tailscale
  - vpn
  - podman
  - installation
  - securite
  - reseau
date: 07-06-2026
last_modified: 07-06-2026
reading-time: 15m
status: draft
---

# Installer Headscale sous Podman rootless

> [!abstract] TL;DR
> Headscale est une réimplémentation open-source et auto-hébergée du **control plane de Tailscale**. Tu utilises les clients Tailscale officiels (excellents, multi-plateformes), mais le serveur de coordination est **chez toi** — pas de dépendance au cloud Tailscale. C'est une **alternative ou un complément à WireGuard nu** ([[Configuration WireGuard self-hosted]]) : Tailscale/Headscale gère automatiquement le NAT traversal, le maillage, le partage de clés — au prix d'une pièce d'infra supplémentaire. Cet article explique le compromis et l'installation.

> [!info] Prérequis de lecture
> Socle de la série, et surtout [[Configuration WireGuard self-hosted]] pour comprendre ce que Headscale automatise. Caddy, MAC, restic.

---

## WireGuard nu vs Headscale : le vrai arbitrage

Headscale et WireGuard reposent tous deux sur le protocole WireGuard. La différence est dans la **gestion**.

> [!info] Ce que Headscale automatise (et que WireGuard nu fait à la main)
> - **NAT traversal automatique** : Tailscale perce les NAT (techniques STUN/DERP) pour connecter des pairs derrière des box sans ouvrir de port. Avec WireGuard nu, il faut un endpoint joignable (port ouvert ou relais manuel).
> - **Maillage automatique** : tous les nœuds se voient entre eux sans config manuelle de chaque paire. Avec WireGuard nu, tu déclares chaque pair.
> - **Distribution des clés** : gérée par le control plane. Avec WireGuard nu, tu copies les clés publiques à la main.
> - **Onboarding simple** : un client s'enregistre avec une commande, pas de fichier de config à transmettre.

> [!warning] Le coût de cette automatisation
> - **Une pièce d'infra de plus** à héberger, sécuriser, sauvegarder (le control plane).
> - **Le control plane est critique** : qui le compromet peut potentiellement autoriser des nœuds. Il doit être exposé (les clients s'y connectent), donc durci.
> - **Plus de complexité conceptuelle** que la simplicité radicale de WireGuard nu.

> [!success] Mon arbitrage
> - **WireGuard nu** si ton topologie est simple, tes pairs peu nombreux, et que tu veux un minimum de surface (cf. la philosophie « complexité méritée » de toute la série). C'est mon choix par défaut.
> - **Headscale** si tu as **beaucoup d'appareils**, des pairs **derrière des NAT difficiles** (4G, CGNAT) où ouvrir un port est impossible, ou si tu veux l'onboarding ultra-simple pour des proches peu techniques. Le NAT traversal automatique est l'argument décisif.

---

## Architecture

Headscale est un binaire Go unique (le control plane). Les clients sont les **applications Tailscale officielles**. Headscale doit être **joignable publiquement** par les clients pour la coordination (mais le trafic des données reste en P2P chiffré entre pairs).

```bash
podman network create headscale-internal --internal
```

---

## Le Quadlet

```ini
# ~/.config/containers/systemd/headscale.container
[Unit]
Description=Headscale control plane

[Container]
Image=docker.io/headscale/headscale:latest
ContainerName=headscale
AutoUpdate=registry
Network=headscale-internal.network

# Config + base (SQLite par défaut, ou PostgreSQL pour gros déploiements)
Volume=%h/headscale/config:/etc/headscale:U,Z
Volume=%h/headscale/data:/var/lib/headscale:U,Z

Exec=serve

NoNewPrivileges=true
DropCapability=ALL

[Service]
Restart=on-failure
RestartSec=10
MemoryMax=512M

[Install]
WantedBy=default.target
```

La config principale vit dans `~/headscale/config/config.yaml` : tu y définis le `server_url` (l'URL publique via Caddy), les plages d'IP du réseau maillé, le backend de base, etc.

```bash
mkdir -p ~/headscale/{config,data}
# Récupère un config.yaml d'exemple depuis le dépôt Headscale, adapte-le
systemctl --user daemon-reload
systemctl --user start headscale.service
```

---

## Exposition : le control plane doit être public

> [!warning] Headscale est l'exception qui s'expose
> Contrairement à la plupart des services de la série (VPN-only), le control plane Headscale **doit** être joignable depuis Internet — c'est par lui que tes clients distants se coordonnent. C'est le rôle qu'aurait joué l'endpoint WireGuard public. On le durcit donc soigneusement.

```caddyfile
headscale.mondomaine.fr {
	import security_headers
	reverse_proxy headscale:8080
}
```

> [!info] WebSocket et coordination
> Les clients maintiennent une connexion longue (WebSocket/HTTP streaming) au control plane. Caddy gère ça nativement. Headscale fournit aussi un serveur DERP (relais) optionnel pour les cas où le P2P direct échoue — à activer selon tes besoins de NAT traversal.

---

## Gérer les nœuds et les ACL

> [!success] L'onboarding d'un client
> ```bash
> # Créer un utilisateur
> podman exec headscale headscale users create famille
> # Générer une clé de pré-authentification
> podman exec headscale headscale preauthkeys create --user famille --expiration 24h
> ```
> Le proche installe le client Tailscale officiel et exécute :
> ```bash
> tailscale up --login-server https://headscale.mondomaine.fr --authkey <clé>
> ```
> C'est tout. Pas de fichier de config à transmettre, pas de clé publique à copier manuellement — l'argument simplicité de Headscale.

> [!tip] Les ACL recréent ta segmentation par profil
> Headscale supporte des **ACL** (au format proche de Tailscale) qui rejouent la logique de segmentation de [[Configuration WireGuard self-hosted]] : définir que le groupe « amis » n'accède qu'à certains nœuds/ports, et « famille » à davantage. C'est l'équivalent de ta matrice d'accès, exprimée en politique Headscale plutôt qu'en nftables + sous-réseaux. Le principe — segmenter par profil — reste identique.

---

## Sauvegarde

> [!success] Quoi sauvegarder
> - **`~/headscale/config`** : `config.yaml`, les **clés privées du serveur** (essentielles — sans elles, tout le réseau maillé doit être reconstruit), les ACL.
> - **`~/headscale/data`** : la base (nœuds enregistrés, utilisateurs, clés de pré-auth). Prudence SQLite pour une copie cohérente.
> - Les clés serveur sont aussi critiques que les clés WireGuard ou les secrets Vaultwarden : sauvegarde séparée.

---

## Conclusion

Headscale est le bon choix quand la **gestion** de WireGuard nu devient pénible : beaucoup d'appareils, NAT difficiles, proches peu techniques. Tu gagnes le NAT traversal automatique et l'onboarding trivial, au prix d'un control plane à héberger, exposer et durcir. Pour un parc simple, WireGuard nu reste plus sobre — fidèle au principe de « complexité méritée » de la série. Headscale n'est pas « mieux », il répond à un problème d'échelle que WireGuard nu ne résout pas élégamment.

> [!note] Articles liés
> - [[Configuration WireGuard self-hosted]]
> - [[Reverse proxy Caddy avec TLS automatique]]
> - [[Stratégie de sauvegarde restic 3-2-1]]

---

*Retour d'expérience personnel. Adapte domaines, plages IP maillées, backend de base. Les clés serveur Headscale sont critiques : sauvegarde-les séparément.*
