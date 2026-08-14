---
title: "Reverse proxy Caddy : le point d'entrée unique de mon self-hosting"
author: "0x41647269656E"
series: "Hardening"
tags:
  - self-hosting
  - caddy
  - reverse-proxy
  - tls
  - securite
  - devops
reading-time: 15m
difficulty: tech-enthusiast
date: 05-06-2026
last_modified: 05-06-2026
status: published
---

# Reverse proxy Caddy : le point d'entrée unique de mon homelab

> [!abstract] TL;DR
> Caddy est le **seul conteneur** de mon parc qui publie des ports sur l'hôte. Tout le reste vit dans des réseaux Podman internes, injoignable directement. Caddy gère le **TLS automatiquement** (ACME, Let's Encrypt, renouvellement transparent), termine le chiffrement, applique les en-têtes de sécurité, et route vers chaque service par son nom dans le réseau interne. Cet article est un retour d'expérience sur pourquoi j'ai choisi Caddy plutôt que Nginx/Traefik, comment je l'intègre en rootless avec un Quadlet, et la configuration complète et durcie de mon `Caddyfile`.

> [!info] Note de cohérence
> Cet article fait suite à [[Self-hosting sécurisé avec Podman]]. Je réutilise le même stack : Podman rootless, Quadlets systemd, réseaux internes par service. Si les notions de réseau `--internal` ou de remap d'UID ne te parlent pas, commence par là.

---

## Le rôle architectural : un seul point d'entrée, pas dix

La décision structurante de tout mon self-hosting tient en une phrase : **aucun service applicatif n'écoute sur une interface publique**. Jamais.

### Le problème que ça résout

Sans reverse proxy, chaque service que tu exposes publie ses propres ports, gère son propre TLS (ou pas), expose sa propre surface d'attaque. Multiplie par dix applis et tu obtiens :

- Dix surfaces réseau exposées, chacune avec son propre niveau de maturité sécurité (spoiler : très inégal).
- Dix configurations TLS à maintenir, ou pire, du HTTP en clair « parce que c'est derrière le réseau local ».
- Aucun endroit central pour appliquer des règles transverses (en-têtes de sécurité, rate limiting, authentification).
- Un cauchemar de certificats : soit tu jongles avec dix renouvellements, soit tu n'as pas de TLS du tout.

> [!success] Ce que le reverse proxy centralise
> Avec Caddy en façade, j'ai **un seul endroit** qui :
> - Publie les ports 80/443 — et eux seuls sont exposés sur l'hôte.
> - Termine **tout** le TLS, avec des certificats valides et auto-renouvelés.
> - Applique les en-têtes de sécurité (HSTS, CSP, etc.) de façon uniforme.
> - Route vers chaque backend par son **nom de conteneur**, sur un réseau interne chiffré du monde extérieur.
> - Centralise les logs d'accès — un seul journal à surveiller.

La topologie est simple : Internet → port 443 de l'hôte → conteneur Caddy → réseau Podman interne → conteneur applicatif. Le backend ne voit **jamais** une connexion qui ne soit pas passée par Caddy.

---

## Pourquoi Caddy plutôt que Nginx ou Traefik

J'ai utilisé les trois. Voici mon arbitrage, sans dogmatisme.

### Nginx : puissant mais le TLS est ta responsabilité

Nginx est increvable et ultra-performant. Mais pour mon usage, il a un défaut rédhibitoire : **le TLS n'est pas automatique**. Tu dois coller `certbot` à côté, gérer les hooks de renouvellement, recharger Nginx après chaque renouvellement. C'est de la plomberie que je ne veux plus maintenir. Et la syntaxe `nginx.conf`, avec ses pièges de `location` et de regex, est un nid à erreurs de sécurité subtiles (mauvaise normalisation de chemin, `proxy_pass` avec/sans slash final…).

### Traefik : excellent en dynamique, surdimensionné en mono-nœud

Traefik est génial dans un contexte orchestré : il **découvre dynamiquement** les services via les labels Docker/Kubernetes et se reconfigure tout seul. Mais cette feature est inutile à l'échelle qui est la mienne. Je n'ai pas de scaling dynamique, mes services sont stables, je suis très souvent seul sur l'environnement, je n'ai pas une infrastructure qui est changeante, pas de haute disponibilité... La configuration par labels et providers, la pile middleware, le dashboard — c'est de la complexité qui répond à un problème (l'infrastructure dynamique) que je n'ai pas, exactement comme Kubernetes dans l'article précédent.

### Caddy : le TLS automatique comme défaut, la config comme un texte lisible

> [!quote] Pourquoi Caddy a gagné chez moi
> - **HTTPS automatique par défaut.** Tu déclares un nom de domaine, Caddy obtient et renouvelle le certificat ACME tout seul. Zéro plomberie. C'est la fonctionnalité phare et elle marche.
> - **Le `Caddyfile` est lisible.** Une déclaration de site tient en quelques lignes compréhensibles, sans regex piégeuses.
> - **Des défauts sains.** TLS moderne, redirection HTTP→HTTPS automatique, en-têtes raisonnables out-of-the-box.
> - **Un seul binaire Go**, sans dépendances, qui tourne parfaitement en conteneur rootless.

Le compromis assumé : Caddy est légèrement moins performant que Nginx sous très forte charge. Pour un parc self-hosted personnel, c'est totalement hors sujet — je ne sature jamais quoi que ce soit.

---

## Intégration rootless : le Quadlet Caddy

Caddy est le seul conteneur autorisé à parler au monde extérieur. Son Quadlet reflète cette responsabilité particulière.

### Le piège des ports privilégiés en rootless

> [!warning] Ports < 1024 et rootless
> En rootless, un utilisateur non privilégié **ne peut pas** binder les ports 80 et 443 par défaut. Deux options propres :
>
> **Option A — abaisser le seuil des ports non privilégiés (mon choix) :**
> ```bash
> # /etc/sysctl.d/99-rootless-ports.conf
> net.ipv4.ip_unprivileged_port_start=80
> ```
> Puis :
> ```bash
> sudo sysctl --system
> ```
> Ça autorise le binding direct de 80/443 sans aucun privilège. C'est la solution la plus simple et je l'assume.
>
> **Option B — rediriger via le pare-feu :** garder Caddy sur 8080/8443 et faire une redirection `firewalld`/`nftables` depuis 80/443. Plus de couches, mais aucune modification du seuil système.

### Mon `caddy.container`

Je place le Quadlet dans `~/.config/containers/systemd/` comme les autres. Point important : j'utilise une **image Caddy custom** parce que j'ai besoin du module DNS pour le challenge ACME `DNS-01` (voir plus bas pourquoi).

```ini
# ~/.config/containers/systemd/caddy.container
[Unit]
Description=Caddy reverse proxy
# Caddy doit pouvoir joindre tous les réseaux internes des services
After=network-online.target

[Container]
# Image custom embarquant le plugin DNS (cf. section TLS)
Image=localhost/caddy-custom:latest
ContainerName=caddy
AutoUpdate=local

# --- Les SEULS ports exposés de tout le parc ---
PublishPort=80:80
PublishPort=443:443
PublishPort=443:443/udp

# --- Posture de sécurité ---
NoNewPrivileges=true
DropCapability=ALL
# Caddy a besoin de binder des ports : on rend juste cette capability
AddCapability=CAP_NET_BIND_SERVICE
ReadOnly=true
Tmpfs=/tmp:rw,size=64M

# --- Réseaux : un pied dans chaque réseau interne de service ---
Network=immich-internal.network
Network=paperless-internal.network
Network=infra-internal.network

# --- Volumes ---
# La config, en lecture seule pour le conteneur
Volume=%h/caddy/Caddyfile:/etc/caddy/Caddyfile:ro,Z
# Données ACME (certificats, clés) — DOIT être persistant
Volume=%h/caddy/data:/data:U,Z
# Cache de config Caddy
Volume=%h/caddy/config:/config:U,Z

# --- Secrets : token API DNS pour le challenge ACME ---
Secret=cloudflare_api_token,type=env,target=CF_API_TOKEN

[Service]
Restart=on-failure
RestartSec=10
MemoryMax=512M

[Install]
WantedBy=default.target
```

> [!danger] Le volume `/data` est critique
> C'est là que Caddy stocke **tes certificats et tes clés privées ACME**. S'il n'est pas persistant, Caddy redemande un certificat à chaque redémarrage et tu vas **exploser les rate limits de Let's Encrypt** (50 certificats/domaine/semaine). Sauvegarde ce volume, et ne le perds jamais.

### Construire l'image custom avec le module DNS

Caddy officiel n'embarque pas les plugins DNS. Pour le challenge `DNS-01` (indispensable si tu ne veux pas exposer le port 80, ou pour les certificats wildcard), il faut une image avec le bon module. En rootless, ça se build sans privilège :

```dockerfile
# ~/caddy/Containerfile
FROM docker.io/library/caddy:2-builder AS builder
RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare

FROM docker.io/library/caddy:2
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

```bash
podman build -t caddy-custom:latest ~/caddy/
```

Remplace `cloudflare` par le provider de ton registrar (`route53`, `gandi`, `ovh`, etc.).

---

## Le TLS automatique : comment ça marche vraiment

C'est l'argument de vente de Caddy, mais il faut comprendre ce qui se passe pour bien choisir son challenge ACME.

### HTTP-01 vs DNS-01 : mon arbitrage

> [!info] Les deux mécanismes de validation ACME
> **HTTP-01** : Let's Encrypt vérifie que tu contrôles le domaine en demandant un fichier sur `http://ton-domaine/.well-known/acme-challenge/...`. Simple, mais **exige que le port 80 soit accessible depuis Internet** et ne permet pas les certificats wildcard.
>
> **DNS-01** : Let's Encrypt te demande de créer un enregistrement TXT dans ta zone DNS. Caddy le fait automatiquement via l'API de ton provider (d'où le module DNS). **Aucun port entrant requis**, et permet les **certificats wildcard** (`*.mondomaine.fr`).

J'utilise **DNS-01** pour deux raisons de fond :

1. **Je n'ai pas besoin d'exposer le port 80.** Mon point d'entrée se réduit au strict 443. Moins de surface.
2. **Certificat wildcard.** Un seul `*.mondomaine.fr` couvre tous mes sous-domaines de services, y compris ceux qui ne sont **jamais exposés publiquement** (Paperless via VPN). HTTP-01 ne peut pas valider un domaine non joignable depuis Internet — DNS-01 si, puisque la validation passe par le DNS, pas par le service.

C'est le point clé : **DNS-01 me permet d'avoir du vrai TLS valide sur des services internes au VPN**, sans exposer quoi que ce soit. Pas de certificat auto-signé, pas d'avertissement navigateur.

---

## Le `Caddyfile` complet et durci

Voici ma configuration réelle, simplifiée mais fonctionnelle. Je l'explique bloc par bloc ensuite.

```caddyfile
# ~/caddy/Caddyfile

# ─────────────────────────────────────────────
# Options globales
# ─────────────────────────────────────────────
{
	# Email pour les notifications ACME (expiration, problèmes)
	email admin@mondomaine.fr

	# Challenge DNS-01 via Cloudflare (token injecté par secret systemd)
	acme_dns cloudflare {env.CF_API_TOKEN}

	# Logs d'accès structurés en JSON vers stdout (capté par journald)
	log {
		output stdout
		format json
		level INFO
	}

	# On ne divulgue pas la version du serveur
	servers {
		trusted_proxies static private_ranges
	}
}

# ─────────────────────────────────────────────
# Snippet réutilisable : en-têtes de sécurité
# ─────────────────────────────────────────────
(security_headers) {
	header {
		# Force HTTPS pendant 2 ans, sous-domaines inclus
		Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
		# Empêche le MIME-sniffing
		X-Content-Type-Options "nosniff"
		# Anti-clickjacking
		X-Frame-Options "SAMEORIGIN"
		# Politique de référent stricte
		Referrer-Policy "strict-origin-when-cross-origin"
		# Désactive des API navigateur non nécessaires
		Permissions-Policy "geolocation=(), microphone=(), camera=()"
		# On retire les en-têtes qui révèlent la stack
		-Server
		-X-Powered-By
	}
}

# ─────────────────────────────────────────────
# Immich — exposé publiquement (app mobile nomade)
# ─────────────────────────────────────────────
photos.mondomaine.fr {
	import security_headers

	# Immich gère de gros uploads : on relève la limite
	request_body {
		max_size 5GB
	}

	reverse_proxy immich-server:2283 {
		# Préserve l'IP cliente réelle pour les logs Immich
		header_up X-Real-IP {remote_host}
	}

	log {
		output file /data/logs/immich-access.log {
			roll_size 50MB
			roll_keep 10
		}
	}
}

# ─────────────────────────────────────────────
# Paperless-ngx — JAMAIS exposé, accès VPN only
# Le certificat wildcard permet du vrai TLS en interne
# ─────────────────────────────────────────────
docs.mondomaine.fr {
	import security_headers

	# Restriction supplémentaire : seul le sous-réseau VPN est autorisé
	@notvpn not remote_ip 10.8.0.0/24
	respond @notvpn "Forbidden" 403

	reverse_proxy paperless-ngx:8000
}

# ─────────────────────────────────────────────
# Redirection : tout le HTTP nu vers HTTPS (sécurité)
# ─────────────────────────────────────────────
http:// {
	redir https://{host}{uri} permanent
}
```

### Décryptage des choix de sécurité

> [!success] Ce qui durcit cette configuration
> - **`acme_dns cloudflare`** : challenge DNS-01, donc TLS valide partout sans exposer de port entrant, et wildcard possible.
> - **Snippet `security_headers`** importé sur chaque site : les en-têtes sont appliqués **uniformément**, impossible d'en oublier un sur un service. C'est le bénéfice du point d'entrée unique.
> - **`Strict-Transport-Security` avec `preload`** : le navigateur refusera tout downgrade HTTP. Attention, `preload` est un engagement quasi-irréversible — ne le mets que si tu es sûr de rester en HTTPS.
> - **`-Server` / `-X-Powered-By`** : on ne renseigne pas l'attaquant sur la stack sous-jacente.
> - **`@notvpn ... respond 403`** sur Paperless : double barrière. Même si quelqu'un résout `docs.mondomaine.fr`, Caddy refuse toute IP hors du sous-réseau WireGuard. Le service n'est de toute façon pas routable hors VPN, mais la défense en profondeur ne coûte rien ici.
> - **`trusted_proxies private_ranges`** : Caddy ne fait confiance aux en-têtes `X-Forwarded-*` que depuis les plages privées, pour éviter le spoofing d'IP cliente.

> [!tip] La résolution par nom de conteneur
> `reverse_proxy immich-server:2283` fonctionne parce que Caddy et Immich **partagent le réseau Podman `immich-internal`**. Podman fournit un DNS interne : le nom du conteneur résout vers son IP dans le réseau. Pas besoin d'IP en dur, pas de `links`. C'est propre et ça survit aux redémarrages.

---

## Recharger sans interruption et superviser

### Reload à chaud

Caddy recharge sa configuration **sans couper les connexions**. Après une modif du `Caddyfile` :

```bash
podman exec caddy caddy reload --config /etc/caddy/Caddyfile
```

Aucune coupure, aucun certificat redemandé. Si la nouvelle config est invalide, Caddy **garde l'ancienne** et te le signale — pas de risque de te couper l'accès en te trompant dans la syntaxe.

> [!tip] Valider avant d'appliquer
> Je valide toujours la syntaxe avant de recharger :
> ```bash
> podman exec caddy caddy validate --config /etc/caddy/Caddyfile
> ```

### Surveiller le bon fonctionnement

Les logs JSON partent dans `journald` via stdout, requêtables comme n'importe quel service systemd :

```bash
# Suivre l'activité de Caddy
journalctl --user -u caddy.service -f

# Vérifier l'obtention/renouvellement des certificats
journalctl --user -u caddy.service | grep -i "certificate obtained"
```

> [!warning] Surveille les erreurs ACME
> Le jour où le renouvellement échoue (token DNS expiré, rate limit, changement de provider), tu veux le savoir **avant** l'expiration du certificat, pas quand tes utilisateurs voient un avertissement navigateur. Je grep périodiquement les erreurs ACME et Caddy m'envoie aussi un mail via l'adresse `email` configurée. Caddy tente le renouvellement à ~30 jours de l'expiration, ce qui laisse une marge confortable pour réagir.

---

## Les frictions réelles que j'ai rencontrées

> [!failure] Les points de douleur, honnêtement
> - **Le module DNS oblige à builder une image custom.** L'image Caddy officielle ne suffit pas pour DNS-01. Pas insurmontable, mais c'est une étape de plus et une image à rebuilder à chaque montée de version majeure.
> - **Le token API DNS est un secret puissant.** Le token Cloudflare peut modifier ta zone DNS. Je le restreins au **strict minimum de permissions** (édition DNS sur la seule zone concernée) et il passe par les secrets systemd, jamais en clair dans le `Caddyfile`.
> - **`preload` sur HSTS est un engagement.** Une fois ton domaine dans la liste de preload des navigateurs, revenir en arrière prend des semaines. Ne l'active que si HTTPS est définitif.
> - **Le partage de réseaux multiples.** Caddy doit avoir un pied dans le réseau interne de **chaque** service qu'il dessert. Quand j'ajoute un service, je dois penser à ajouter sa `Network=` dans le Quadlet Caddy, sinon la résolution de nom échoue. Erreur facile à faire et un peu pénible à débugger.

Cela dit, une fois en place, c'est d'une stabilité remarquable. Je ne touche plus à mes certificats — ils se renouvellent seuls depuis des mois. Ajouter un service, c'est trois lignes dans le `Caddyfile` et un reload à chaud.

---

## Conclusion : la façade comme pilier de la posture de sécurité

Le reverse proxy n'est pas un détail de confort, c'est une **décision d'architecture de sécurité**. Trois principes que j'en retire :

1. **Réduire la surface à un point unique.** Un seul conteneur expose des ports, un seul endroit termine le TLS, un seul journal à surveiller. Tout le reste est dans le noir, sur des réseaux internes.
2. **Le TLS automatique élimine une classe entière d'erreurs.** Plus de certificats expirés, plus de plomberie certbot, plus de HTTP en clair « temporaire ». Et DNS-01 me donne du vrai TLS même sur des services qui ne touchent jamais Internet.
3. **Centraliser, c'est uniformiser la défense.** En-têtes de sécurité, restrictions d'IP, rate limiting : appliqués une fois dans un snippet, valables partout. Impossible d'oublier de durcir un service.

> [!note] Pour aller plus loin dans le vault
> - [[Self-hosting sécurisé avec Podman]]
> - [[Configuration WireGuard self-hosted]]
> - [[Durcissement SELinux pour conteneurs]]
> - [[Stratégie de sauvegarde restic 3-2-1]]

---

*Retour d'expérience personnel. Adapte les noms de domaine, le provider DNS et les plages IP VPN à ta propre infrastructure. Le `Caddyfile` ci-dessus est un point de départ durci, pas une vérité absolue.*
