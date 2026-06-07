---
title: Installer Authelia ou Authentik sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - authelia
  - authentik
  - sso
  - mfa
  - podman
  - installation
  - securite
date: 07-06-2026
last_modified: 07-06-2026
reading-time: 15m
status: draft
---

# Installer Authelia ou Authentik sous Podman rootless

> [!abstract] TL;DR
> Authelia et Authentik ajoutent une **couche d'authentification centralisée (SSO + MFA)** devant tes services exposés. Au lieu que chaque appli gère ses comptes (inégalement), Caddy délègue l'authentification à un **portail unique** : login + 2FA une fois, accès à tout ce qui est autorisé. C'est la brique qui manquait pour durcir proprement les services exposés publiquement de la série. **Authelia** = léger, fichier de config, *forward auth*. **Authentik** = complet (vrai IdP OAuth/SAML), plus lourd. Je présente Authelia (intégration Caddy directe) et situe Authentik.

> [!info] Prérequis de lecture
> Socle de la série, et surtout [[Reverse proxy Caddy avec TLS automatique]] — Authelia se branche en *forward auth* sur Caddy. WireGuard, MAC, restic comme partout.

---

## Le problème : l'authentification éparpillée

> [!warning] Chaque service réinvente l'auth, mal
> Sans portail central, Immich a ses comptes, Jellyfin les siens, FreshRSS aussi… Niveaux de maturité hétérogènes, MFA absent ici, mots de passe faibles là. Pour les services **exposés sur Internet**, c'est une surface d'attaque dispersée et difficile à durcir uniformément.

> [!success] Ce que le SSO centralise
> - **Un seul point d'authentification** : login + MFA une fois, géré par un service conçu pour ça.
> - **MFA uniforme** (TOTP, WebAuthn/clés physiques) appliqué devant **tous** les services, même ceux qui n'ont pas de 2FA natif.
> - **Politique d'accès centralisée** : qui accède à quoi, par règle, depuis un seul endroit — exactement la philosophie de la matrice d'accès WireGuard, mais au niveau applicatif.
> - **Protection des services sans auth** : un service qui n'a aucune authentification native se retrouve protégé par le portail en amont.

---

## Choisir : Authelia ou Authentik ?

> [!info] Le bon choix
> - **Authelia** → léger (Go), configuration par fichier YAML, s'intègre en ***forward auth*** : Caddy demande à Authelia « cet utilisateur peut-il passer ? » avant chaque requête. Idéal pour protéger des services derrière un reverse proxy. Pas un IdP complet.
> - **Authentik** → un véritable fournisseur d'identité (OAuth2/OIDC, SAML, LDAP), interface d'admin riche, flux personnalisables. Plus lourd (Python + PostgreSQL + Redis + worker). À choisir si tu as besoin d'OIDC/SAML pour des intégrations natives (login « Se connecter avec mon serveur »).
>
> Pour simplement mettre un mur d'auth + MFA devant tes services exposés, **Authelia** suffit et reste simple. Pour un SSO d'entreprise complet, Authentik.

---

## Authelia : architecture

Authelia + une base de session (Redis recommandé pour la persistance des sessions) + un backend d'utilisateurs (fichier ou LDAP). Pour un usage perso/famille, le backend **fichier** suffit.

```bash
podman network create auth-internal --internal
```

```ini
# ~/.config/containers/systemd/authelia-redis.container
[Unit]
Description=Authelia Redis
[Container]
Image=docker.io/library/redis:7-alpine
ContainerName=authelia-redis
AutoUpdate=registry
Network=auth-internal.network
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
# ~/.config/containers/systemd/authelia.container
[Unit]
Description=Authelia
Requires=authelia-redis.service
After=authelia-redis.service
[Container]
Image=ghcr.io/authelia/authelia:latest
ContainerName=authelia
AutoUpdate=registry
Network=auth-internal.network
# Config (configuration.yml) + base utilisateurs : volume dédié
Volume=%h/authelia/config:/config:U,Z
Environment=TZ=Europe/Paris
NoNewPrivileges=true
DropCapability=ALL
[Service]
Restart=on-failure
MemoryMax=256M
[Install]
WantedBy=default.target
```

> [!info] Le réseau partagé avec Caddy
> Caddy doit pouvoir joindre Authelia pour le *forward auth*. Ajoute le réseau `auth-internal` au Quadlet de Caddy (comme tu ajoutes le réseau de chaque service qu'il dessert, cf. [[Reverse proxy Caddy avec TLS automatique]]).

---

## L'intégration Caddy : forward auth

C'est ici que tout se joue. Caddy interroge Authelia avant de laisser passer vers un service protégé.

```caddyfile
# Le portail Authelia lui-même
auth.mondomaine.fr {
	import security_headers
	reverse_proxy authelia:9091
}

# Un service protégé par Authelia (exemple : un service exposé publiquement)
app.mondomaine.fr {
	import security_headers

	# Forward auth : Caddy demande à Authelia si la requête est autorisée
	forward_auth authelia:9091 {
		uri /api/authz/forward-auth
		copy_headers Remote-User Remote-Groups Remote-Name Remote-Email
	}

	reverse_proxy mon-service:8080
}
```

> [!success] Le flux d'authentification
> 1. L'utilisateur demande `app.mondomaine.fr`.
> 2. Caddy interroge Authelia : session valide ?
> 3. Si non → redirection vers `auth.mondomaine.fr` (login + MFA).
> 4. Après authentification → retour à `app.mondomaine.fr`, Caddy laisse passer.
> 5. Les en-têtes `Remote-User`/`Remote-Groups` sont transmis au service, qui peut les exploiter pour de l'auto-login.

> [!tip] Combiner avec les restrictions IP existantes
> Tu peux empiler `forward_auth` **et** la restriction `remote_ip` par sous-réseau VPN. Exemple : services internes en VPN-only (pas besoin d'Authelia), services exposés publiquement protégés par Authelia + MFA. Chaque couche pour son contexte.

---

## Sauvegarde

> [!success] Quoi sauvegarder
> - **`~/authelia/config`** : `configuration.yml`, la base utilisateurs (fichier `users_database.yml` avec les hash), les politiques d'accès, les secrets/clés. C'est tout l'état d'Authelia.
> - **Les secrets** (clés de session, de chiffrement) sont critiques : sans eux, les sessions et certaines données 2FA sont invalidées. Traite-les comme les clés RSA de Vaultwarden.

---

## Conclusion

Authelia (ou Authentik pour les besoins IdP complets) est la brique qui transforme une collection de services aux auth hétérogènes en un parc avec **MFA uniforme et politique centralisée**. Son intégration en *forward auth* sur Caddy est élégante : le reverse proxy, déjà point d'entrée unique, devient aussi point d'authentification unique. C'est la suite logique de toute la série pour qui expose des services publiquement.

> [!note] Articles liés
> - [[Reverse proxy Caddy avec TLS automatique]]
> - [[Configuration WireGuard self-hosted]]
> - [[Installer Pihole ou AdGuard sous Podman rootless]]

---

*Retour d'expérience personnel. Adapte domaines, images, backend utilisateurs. Les secrets Authelia sont critiques : génère-les solidement et sauvegarde-les séparément.*
