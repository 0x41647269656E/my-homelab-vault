---
title: Installer Vaultwarden sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - vaultwarden
  - podman
  - installation
  - securite
date: 06-06-2026
last_modified: 06-06-2026
reading-time: 15m
difficulty: tech-enthusiast
status: draft
---

# Installer Vaultwarden sous Podman rootless

> [!abstract] TL;DR
> Vaultwarden est une réimplémentation légère du serveur Bitwarden en Rust : un **seul conteneur**, compatible avec tous les clients Bitwarden officiels. C'est le service qui exige ma posture de sécurité la **plus stricte** — il contient littéralement les clés de tout le reste. Je l'installe en Podman rootless, **jamais exposé directement sur Internet** (accès WireGuard uniquement par défaut), avec chiffrement TLS, confinement MAC maximal, et une sauvegarde traitée comme critique. Pas d'accès à `/media` : Vaultwarden ne touche aucune donnée média.

> [!info] Prérequis de lecture
> Socle des articles : [[Self-hosting sécurisé avec Podman]], [[Reverse proxy Caddy avec TLS automatique]], [[Configuration WireGuard self-hosted]], confinement MAC ([[Durcissement SELinux pour conteneurs]] / [[Durcissement AppArmor pour conteneurs]]), [[Stratégie de sauvegarde restic 3-2-1]].

---

## Pourquoi Vaultwarden mérite un traitement à part

> [!danger] Le coffre-fort est la cible la plus précieuse
> Vaultwarden détient les mots de passe de **tous** mes autres services et comptes. Une compromission ici n'est pas un incident parmi d'autres : c'est la compromission de tout. Toute la prudence des articles précédents s'applique ici **au maximum**. En particulier : ce service n'a aucune raison d'être exposé publiquement « pour la commodité ». La commodité ne justifie jamais d'exposer le coffre.

Conséquence directe : **Vaultwarden vit derrière WireGuard**, comme Paperless. Le navigateur et l'app Bitwarden synchronisent via le tunnel. Pour le partage de mots de passe avec la famille, ils rejoignent le profil VPN « famille » (cf. [[Configuration WireGuard self-hosted]]).

---

## Architecture : un conteneur, des données minuscules mais critiques

Vaultwarden est mono-conteneur. Sa base est un SQLite par défaut (suffisant pour un usage perso/famille) ou PostgreSQL pour de plus gros déploiements. Je reste sur SQLite : moins de surface, un seul fichier à sauvegarder.

Les données tiennent dans un volume `~/vaultwarden/data` : la base, les pièces jointes, les clés. Petites en volume, maximales en sensibilité.

```bash
podman network create vaultwarden-internal --internal
```

---

## Le Quadlet

```ini
# ~/.config/containers/systemd/vaultwarden.container
[Unit]
Description=Vaultwarden (Bitwarden server)

[Container]
Image=docker.io/vaultwarden/server:latest
ContainerName=vaultwarden
AutoUpdate=registry
Network=vaultwarden-internal.network

# Données critiques sur volume dédié (aucun lien avec /media)
Volume=%h/vaultwarden/data:/data:U,Z

# --- Configuration ---
# Inscription désactivée : on ne veut PAS que n'importe qui crée un compte
Environment=SIGNUPS_ALLOWED=false
# Token admin pour le panneau /admin, en secret (jamais en clair)
Secret=vaultwarden_admin_token,type=env,target=ADMIN_TOKEN
# Domaine pour les liens et WebAuthn
Environment=DOMAIN=https://vault.mondomaine.fr

# --- Durcissement maximal ---
NoNewPrivileges=true
DropCapability=ALL
ReadOnly=true
Tmpfs=/tmp:rw,size=64M

[Service]
Restart=on-failure
RestartSec=10
MemoryMax=512M

[Install]
WantedBy=default.target
```

> [!success] Les choix de durcissement spécifiques
> - **`SIGNUPS_ALLOWED=false`** : après avoir créé mon compte (et ceux de la famille), je coupe toute nouvelle inscription. Si quelqu'un atteint l'interface, il ne peut pas se créer un accès.
> - **`ADMIN_TOKEN` en secret** : le panneau `/admin` est puissant. Le token passe par les secrets systemd, jamais dans le Quadlet en clair. Idéalement je désactive carrément l'admin une fois la config faite.
> - **`ReadOnly=true`** : le rootfs est immuable, seul `/data` est inscriptible.
> - **`DropCapability=ALL`** : Vaultwarden en Rust n'a besoin d'aucune capability particulière.

```bash
systemctl --user daemon-reload
systemctl --user start vaultwarden.service
```

---

## Exposition : WireGuard d'abord, jamais Internet par défaut

Le bloc Caddy restreint l'accès au sous-réseau VPN, exactement comme Paperless dans l'article reverse proxy :

```caddyfile
vault.mondomaine.fr {
	import security_headers

	# Admin (10.10.1) et famille (10.10.2) seulement. Amis exclus, Internet exclu.
	@allowed remote_ip 10.10.1.0/24 10.10.2.0/24
	handle @allowed {
		reverse_proxy vaultwarden:80
	}
	respond "Forbidden" 403
}
```

> [!warning] Le piège du "juste exposer un peu"
> La tentation d'exposer Vaultwarden pour y accéder en déplacement est réelle. Ma réponse : le client Bitwarden fonctionne **hors-ligne** avec le coffre mis en cache localement et chiffré. J'ai besoin de la synchro, pas d'un accès permanent. WireGuard se connecte en quelques secondes quand je dois synchroniser. La surface d'attaque publique d'un coffre de mots de passe ne vaut pas ce gain de confort.

> [!info] WebSocket pour la synchro temps réel
> Vaultwarden gère les notifications de synchro via WebSocket sur le même port avec les images récentes. Caddy proxifie le WebSocket automatiquement, rien de spécial à configurer dans la plupart des cas. Si tu as une version plus ancienne, vérifie la doc pour un éventuel port de notifications séparé.

---

## Sauvegarde : traiter le coffre comme irremplaçable

> [!danger] La sauvegarde SQLite doit être cohérente
> Copier le fichier SQLite pendant une écriture peut le corrompre, comme pour PostgreSQL. Vaultwarden recommande l'usage de la commande SQLite `.backup` qui produit une copie cohérente même base ouverte.

```bash
# Sauvegarde cohérente de la base SQLite + données
podman exec vaultwarden sqlite3 /data/db.sqlite3 ".backup '/data/db-backup.sqlite3'"
# Puis restic embarque /data (db-backup + attachments + clés)
```

> [!success] Ce qui est vital dans la sauvegarde Vaultwarden
> - **`db.sqlite3`** : le coffre lui-même.
> - **`rsa_key*`** : les clés de signature des tokens. **Sans elles, restaurer la base ne suffit pas** à remonter le service proprement.
> - **`attachments/`** et **`sends/`** : pièces jointes et partages.
>
> Tout est dans `/data`. La sauvegarde chiffrée de restic + l'append-only (cf. article sauvegarde) sont ici non négociables : c'est le service où une sauvegarde compromise ou perdue fait le plus mal.

---

## Conclusion

Vaultwarden est le service qui condense toute la philosophie de la série : un seul conteneur, mais durci au maximum, **non exposé** par principe, accessible uniquement via le tunnel segmenté, et sauvegardé comme le bien le plus précieux du parc — parce qu'il l'est. Pas d'accès `/media`, pas de compromis sur l'exposition. Le coffre se mérite.

> [!note] Articles liés
> - [[Configuration WireGuard self-hosted]]
> - [[Reverse proxy Caddy avec TLS automatique]]
> - [[Stratégie de sauvegarde restic 3-2-1]]

---

*Retour d'expérience personnel. Adapte domaines, plages VPN et images. Le token admin et les clés RSA sont les éléments les plus sensibles : traite-les en conséquence.*
