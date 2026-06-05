---
title: "WireGuard self-hosted : segmenter l'accès par profil (perso, famille, amis)"
author: "0x41647269656E"
series: "Hardening"
tags:
  - self-hosting
  - wireguard
  - vpn
  - reseau
  - securite
  - devops
reading-time: 15m
date: 05-06-2026
last_modified: 05-06-2026
status: published
---

# WireGuard self-hosted : segmenter l'accès par profil (perso, famille, amis)

> [!abstract] TL;DR
> J'expose mes services sensibles **uniquement via WireGuard**, jamais sur Internet. Mais tous les pairs n'ont pas les mêmes droits : moi j'accède à tout, ma famille à la plupart des services, mes amis **seulement à Immich** (partage de photos) et rien d'autre. WireGuard seul ne sait pas faire ça — il route des paquets, il ne connaît pas la notion d'« appli ». La segmentation se construit en **trois couches** : un sous-réseau par profil, un filtrage `nftables` indexé sur l'IP du pair, et les restrictions d'IP de Caddy qu'on a déjà posées. Cet article est le retour d'expérience de cette architecture.

> [!info] Note de cohérence
> Suite de [[Self-hosting sécurisé avec Podman]] et [[Reverse proxy Caddy avec TLS automatique]]. Je réutilise le même stack : Podman rootless, Caddy en point d'entrée unique, services sur réseaux internes. WireGuard devient ici le **seul moyen d'atteindre** les services non publics, et le point où je discrimine qui voit quoi.

---

## Le problème de fond : un VPN route, il ne contrôle pas

Le réflexe naïf, c'est de penser « je mets tout le monde sur le VPN et c'est sécurisé ». C'est faux, et c'est un contresens sur ce qu'est WireGuard.

### Ce que WireGuard fait, et ce qu'il ne fait pas

WireGuard est un **tunnel chiffré point-à-point** d'une simplicité remarquable : chaque pair a une paire de clés, et le champ `AllowedIPs` décrit quelles IPs sont routées dans le tunnel. C'est tout. C'est minimaliste **par design**, et c'est sa force.

> [!warning] Le malentendu classique
> `AllowedIPs` n'est **pas une ACL de sécurité**. Côté serveur, il indique « les paquets venant de telle IP source appartiennent à tel pair ». Côté client, il indique « route tel trafic dans le tunnel ». Ce n'est **pas** un pare-feu. Une fois qu'un pair est dans le tunnel, sans filtrage supplémentaire, il peut parler à **tout ce qui est joignable** sur le réseau du serveur. Donner accès au VPN ≠ donner un accès cloisonné.

Autrement dit : si je mets un ami sur mon VPN sans rien d'autre, il peut atteindre Immich **et** Paperless **et** mon interface d'admin **et** tout ce qui écoute. C'est exactement ce qu'on ne veut pas.

### La segmentation se construit au-dessus

Pour que « les amis ne voient qu'Immich », il faut une couche de **contrôle d'accès** par-dessus le routage WireGuard. Mon architecture repose sur trois niveaux complémentaires, du plus grossier au plus fin :

1. **Adressage par profil** — chaque catégorie de pair vit dans un sous-réseau distinct (`/24` dédié). C'est le levier d'indexation pour tout le reste.
2. **Filtrage `nftables` sur l'hôte** — des règles qui autorisent ou bloquent le trafic en fonction de l'IP source (donc du profil), avant qu'il n'atteigne quoi que ce soit.
3. **Restriction applicative dans Caddy** — la couche déjà posée dans l'article précédent (`@notvpn ... respond 403`), qu'on affine par profil.

La défense en profondeur, ici, n'est pas un slogan : si une couche est mal configurée, la suivante rattrape. C'est précisément ce qu'on veut sur des données sensibles.

---

## L'architecture d'adressage : un sous-réseau par profil

Tout part du plan d'adressage. C'est la décision la plus structurante de l'article — bien la poser rend tout le filtrage trivial, la rater rend tout pénible.

### Mon découpage

> [!success] Plan d'adressage par profil
> Je réserve `10.10.0.0/16` au VPN, découpé en `/24` par profil :
> - **`10.10.1.0/24` — moi (admin).** Accès total. Mes propres appareils (laptop, téléphone, tablette).
> - **`10.10.2.0/24` — famille.** Accès à la plupart des services (Immich, Jellyfin, etc.), mais pas à l'admin ni aux outils d'infra.
> - **`10.10.3.0/24` — amis.** Accès à **Immich uniquement**. Rien d'autre n'est joignable, point.
>
> Le serveur WireGuard est en `10.10.0.1`.

L'intérêt de cette structure : **le profil d'un pair se lit dans son IP**. Un paquet venant de `10.10.3.x` est un ami, point. Mes règles de pare-feu n'ont plus qu'à raisonner sur des sous-réseaux, pas sur des IPs individuelles. Ajouter un ami, c'est lui attribuer une IP dans `10.10.3.0/24` et il hérite **automatiquement** des bonnes restrictions. Zéro règle à ajouter.

> [!tip] Pourquoi des /24 et pas des IPs individuelles
> J'ai d'abord essayé de gérer les droits IP par IP. Cauchemar : chaque nouveau pair imposait une nouvelle règle de pare-feu. En raisonnant par sous-réseau-profil, les règles sont **stables** : elles décrivent des catégories, pas des individus. L'attribution d'IP devient le seul acte de gestion. C'est le pattern qui passe à l'échelle d'une famille élargie + cercle d'amis.

---

## Le serveur WireGuard en conteneur rootless

Cohérent avec le reste du stack, WireGuard tourne en conteneur. Petite subtilité : il a besoin d'accès au module noyau et de capabilities réseau, ce qui demande un peu plus que les autres services.

### Le Quadlet WireGuard

> [!warning] WireGuard rootless : la nuance
> WireGuard utilise un module **noyau** (`wireguard`), partagé par tout l'hôte. Le conteneur n'a pas besoin d'être privilégié, mais il lui faut `CAP_NET_ADMIN` pour gérer l'interface, et le module doit être chargé côté hôte :
> ```bash
> sudo modprobe wireguard
> echo wireguard | sudo tee /etc/modules-load.d/wireguard.conf
> ```
> Selon ta version de noyau et ta config rootless, certains montages (`/dev/net/tun`) peuvent être nécessaires. C'est le service le moins « pur rootless » de mon parc, je l'assume.

```ini
# ~/.config/containers/systemd/wireguard.container
[Unit]
Description=WireGuard VPN server
After=network-online.target

[Container]
Image=docker.io/linuxserver/wireguard:latest
ContainerName=wireguard
AutoUpdate=registry

# Le seul port UDP exposé pour le VPN
PublishPort=51820:51820/udp

# --- Capabilities minimales nécessaires ---
DropCapability=ALL
AddCapability=CAP_NET_ADMIN
AddCapability=CAP_SYS_MODULE
# Nécessaire pour le forwarding et la gestion d'interface
AddCapability=CAP_NET_RAW

# Accès au device tun
AddDevice=/dev/net/tun

# Sysctl de forwarding dans le namespace réseau du conteneur
Sysctl=net.ipv4.ip_forward=1

# --- Volumes ---
Volume=%h/wireguard/config:/config:U,Z
Volume=/lib/modules:/lib/modules:ro

# Fuseau horaire pour des logs cohérents
Environment=TZ=Europe/Paris

[Service]
Restart=on-failure
RestartSec=10
MemoryMax=256M

[Install]
WantedBy=default.target
```

> [!info] Pourquoi pas le réseau interne Podman ici
> Contrairement à Immich ou Paperless, WireGuard doit pouvoir **router vers les autres réseaux**. Le trafic des pairs arrive dans le tunnel puis doit atteindre Caddy. Le routage et le filtrage de ce trafic se font au niveau de **l'hôte** (nftables), pas dans un réseau Podman isolé. C'est pour ça que WireGuard publie son port et que le contrôle d'accès vit sur l'hôte.

---

## La configuration WireGuard : serveur et pairs

### Le fichier serveur

Le `wg0.conf` côté serveur déclare l'interface et chaque pair avec son IP de profil. Voici la structure (clés tronquées) :

```ini
# /config/wg_confs/wg0.conf

[Interface]
Address = 10.10.0.1/16
ListenPort = 51820
PrivateKey = <CLE_PRIVEE_SERVEUR>
# Pas de PostUp iptables ici : le filtrage est géré par nftables sur l'hôte
# (voir section suivante) pour découpler la politique du conteneur

# ─── MOI (admin) — 10.10.1.0/24 ───
[Peer]
# Laptop perso
PublicKey = <CLE_PUBLIQUE_LAPTOP>
AllowedIPs = 10.10.1.10/32

[Peer]
# Téléphone perso
PublicKey = <CLE_PUBLIQUE_TEL>
AllowedIPs = 10.10.1.11/32

# ─── FAMILLE — 10.10.2.0/24 ───
[Peer]
PublicKey = <CLE_PUBLIQUE_CONJOINT>
AllowedIPs = 10.10.2.10/32

[Peer]
PublicKey = <CLE_PUBLIQUE_PARENT>
AllowedIPs = 10.10.2.11/32

# ─── AMIS — 10.10.3.0/24 ───
[Peer]
# Ami 1 — n'aura accès qu'à Immich (filtré par nftables + Caddy)
PublicKey = <CLE_PUBLIQUE_AMI1>
AllowedIPs = 10.10.3.10/32
```

> [!danger] `AllowedIPs` en /32 côté serveur, c'est important
> Chaque pair est déclaré avec son IP **exacte en /32**. Si tu mets `/24` ou `/16` côté serveur, tu autorises le pair à **usurper** n'importe quelle IP de la plage — et donc à se faire passer pour un autre profil. Le /32 par pair côté serveur est ce qui garantit qu'un ami est **bien** dans `10.10.3.x` et ne peut pas se prétendre dans `10.10.1.x` (admin). C'est une propriété de sécurité, pas un détail de routage.

### La config côté client : split tunnel

Pour les amis, je ne veux **pas** router tout leur trafic Internet via mon serveur (ni le coût, ni la responsabilité). Je fais du **split tunnel** : seul le trafic vers mes services passe dans le tunnel.

```ini
# Config client — profil AMI
[Interface]
PrivateKey = <CLE_PRIVEE_AMI1>
Address = 10.10.3.10/32
# DNS de mon serveur uniquement quand dans le tunnel (résolution des sous-domaines)
DNS = 10.10.0.1

[Peer]
PublicKey = <CLE_PUBLIQUE_SERVEUR>
Endpoint = vpn.mondomaine.fr:51820
# SPLIT TUNNEL : on ne route QUE le sous-réseau VPN, pas tout Internet
AllowedIPs = 10.10.0.0/16
# Maintient le tunnel actif derrière NAT (mobile)
PersistentKeepalive = 25
```

> [!tip] La différence admin vs ami côté client
> Pour **mon** profil admin, je mets parfois `AllowedIPs = 0.0.0.0/0` quand je veux router tout mon trafic via la maison (Wi-Fi public, etc.). Pour les **amis**, jamais : `AllowedIPs = 10.10.0.0/16` limite le tunnel à mes seuls services. Leur navigation normale ne me concerne pas et ne transite pas chez moi. C'est un choix de responsabilité autant que technique.

---

## La couche qui fait tout : le filtrage nftables par profil

C'est ici que « les amis ne voient qu'Immich » devient réel. WireGuard a routé le paquet jusqu'à l'hôte ; nftables décide maintenant s'il a le droit d'aller plus loin, **en fonction de son IP source** (donc de son profil).

### La logique

Tout le trafic des pairs traverse la chaîne `forward` de l'hôte avant d'atteindre Caddy (qui écoute sur 443). Mon principe : **deny par défaut**, puis autorisations explicites par profil.

```bash
#!/usr/sbin/nft -f
# /etc/nftables.d/wireguard-segmentation.nft

table inet wg_filter {

    # Ensemble des profils, pour des règles lisibles
    define ADMIN_NET   = 10.10.1.0/24
    define FAMILY_NET  = 10.10.2.0/24
    define FRIENDS_NET = 10.10.3.0/24

    # IP de l'hôte où écoute Caddy (443)
    define PROXY = 10.10.0.1

    chain forward {
        type filter hook forward priority 0; policy drop;

        # Les connexions déjà établies passent
        ct state established,related accept

        # ─── ADMIN : accès total ───
        ip saddr $ADMIN_NET accept

        # ─── FAMILLE : accès à Caddy (443) uniquement ───
        # (le routage applicatif fin se fait ensuite dans Caddy)
        ip saddr $FAMILY_NET ip daddr $PROXY tcp dport 443 accept

        # ─── AMIS : accès à Caddy (443) uniquement, comme la famille ───
        # La distinction Immich-only se fait dans Caddy par sous-réseau
        ip saddr $FRIENDS_NET ip daddr $PROXY tcp dport 443 accept

        # Tout le reste est droppé par la policy
    }
}
```

> [!success] Ce que cette politique garantit
> - **Admin (`10.10.1.0/24`)** : `accept` total. J'atteins tout, y compris en direct si besoin (SSH, interfaces d'admin).
> - **Famille et amis** : ne peuvent atteindre **que** le port 443 de Caddy. Ils ne peuvent **pas** parler directement à un conteneur, ni à SSH, ni à quoi que ce soit d'autre sur l'hôte. Toute leur interaction passe par le reverse proxy.
> - **Policy `drop`** : tout ce qui n'est pas explicitement autorisé est rejeté. C'est la posture par défaut correcte.

Remarque le choix de design : famille et amis ont la **même règle nftables** (accès à Caddy seulement). La distinction fine entre eux — qui voit quelle appli — **ne se fait pas ici**. nftables verrouille au niveau réseau (« tu ne parles qu'au proxy »), et Caddy fait le tri applicatif. Séparer les responsabilités ainsi garde nftables simple et stable.

---

## Le tri applicatif final : Caddy par sous-réseau

On boucle avec la couche de l'article précédent. Dans Caddy, je discrimine désormais par **sous-réseau de profil**, pas juste « VPN ou pas VPN ».

```caddyfile
# Immich — accessible à TOUS les profils VPN (et public pour l'app mobile)
photos.mondomaine.fr {
	import security_headers
	request_body {
		max_size 5GB
	}
	reverse_proxy immich-server:2283
}

# Paperless — famille + admin, JAMAIS les amis
docs.mondomaine.fr {
	import security_headers

	# Autorise admin (10.10.1) et famille (10.10.2), bloque le reste (dont amis)
	@allowed remote_ip 10.10.1.0/24 10.10.2.0/24
	handle @allowed {
		reverse_proxy paperless-ngx:8000
	}
	# Tout autre profil (amis 10.10.3, ou hors VPN) → 403
	respond "Forbidden" 403
}

# Interface d'admin infra — MOI uniquement
admin.mondomaine.fr {
	import security_headers

	@admin_only remote_ip 10.10.1.0/24
	handle @admin_only {
		reverse_proxy infra-dashboard:3000
	}
	respond "Forbidden" 403
}
```

> [!success] La matrice d'accès résultante
> | Service | Admin (`.1`) | Famille (`.2`) | Amis (`.3`) | Internet |
> |---|---|---|---|---|
> | **Immich** (photos) | ✅ | ✅ | ✅ | ✅ (app mobile) |
> | **Paperless** (docs) | ✅ | ✅ | ❌ | ❌ |
> | **Admin infra** | ✅ | ❌ | ❌ | ❌ |
>
> Un ami sur `10.10.3.x` qui tente `docs.mondomaine.fr` : nftables le laisse atteindre Caddy (443), mais Caddy renvoie `403` parce que son IP n'est pas dans les sous-réseaux autorisés. Double verrou. Et de toute façon, son `AllowedIPs` client ne route que `10.10.0.0/16`, donc il ne peut même pas atteindre Internet via moi.

> [!tip] Pourquoi cette redondance nftables + Caddy
> On pourrait penser que filtrer dans Caddy suffit. Mais nftables protège **tout ce qui n'est pas Caddy** (SSH, conteneurs en direct, services futurs), tandis que Caddy protège **au niveau applicatif** (par domaine). Les deux couches répondent à des menaces différentes. Si demain j'ajoute un service mal configuré qui écoute sur l'hôte, nftables empêche déjà les amis de l'atteindre, même si j'ai oublié de le déclarer dans Caddy.

---

## Onboarding d'un nouveau pair : le workflow concret

Pour que ce soit opérationnel et pas juste théorique, voici exactement ce que je fais pour ajouter un ami.

```bash
# 1. Générer la paire de clés du nouveau pair
wg genkey | tee ami2_private.key | wg pubkey > ami2_public.key

# 2. Choisir la prochaine IP libre dans le sous-réseau AMIS
#    -> 10.10.3.11 (la .10 était déjà prise par ami1)

# 3. Ajouter le bloc [Peer] dans wg0.conf côté serveur (IP en /32 !)
#    PublicKey = <contenu de ami2_public.key>
#    AllowedIPs = 10.10.3.11/32

# 4. Recharger WireGuard sans couper les autres tunnels
podman exec wireguard wg syncconf wg0 <(podman exec wireguard wg-quick strip wg0)
```

> [!success] Ce qui rend ça simple
> Je **n'ai aucune règle nftables à ajouter** : l'IP `10.10.3.11` est dans `10.10.3.0/24`, donc elle hérite automatiquement de la politique « amis ». Je **n'ai rien à toucher dans Caddy** non plus : la matrice est déjà définie par sous-réseau. L'unique acte de gestion est l'attribution d'une IP dans le bon `/24`. C'est tout le bénéfice de l'adressage par profil posé au début.

> [!warning] Transmettre la config client en sécurité
> La config client contient une clé privée. Je ne l'envoie **jamais** par un canal en clair (mail, SMS). QR code généré localement scanné en personne quand c'est possible, ou via un canal chiffré (Signal). Pour les amis peu techniques, le QR code de l'app WireGuard mobile est de loin le plus simple — ils scannent, c'est connecté.

---

## Les frictions réelles que j'ai rencontrées

> [!failure] Les points de douleur, honnêtement
> - **WireGuard en conteneur rootless n'est pas tout à fait pur.** Le module noyau, `CAP_NET_ADMIN`, `/dev/net/tun` — c'est le service qui demande le plus de capabilities de mon parc. Certains préfèrent le faire tourner directement sur l'hôte via `wg-quick` ; c'est défendable, mais je voulais l'uniformité du modèle conteneur.
> - **Le forwarding entre le namespace conteneur et l'hôte.** Faire transiter le trafic des pairs depuis l'interface WireGuard (dans le conteneur) vers Caddy a demandé de bien comprendre où vit le routage. C'est le point qui m'a pris le plus de temps à câbler correctement.
> - **nftables vs le pare-feu existant.** Si tu utilises `firewalld` ou des règles `iptables` héritées, attention aux conflits et à l'ordre des chaînes. J'ai dû consolider sur nftables pur pour garder une politique lisible.
> - **Le DNS dans le tunnel.** Pour que `photos.mondomaine.fr` résolve vers l'IP interne quand on est dans le VPN, il faut un DNS qui répond correctement (split-horizon ou résolution interne). Sans ça, les pairs résolvent l'IP publique et le split tunnel casse l'accès. C'est subtil et ça surprend.

Mais une fois l'architecture en place, la gestion quotidienne est triviale : ajouter quelqu'un = une IP dans le bon sous-réseau. Et surtout, je sais **exactement** qui peut voir quoi, sans avoir à y repenser à chaque ajout. La matrice d'accès est une propriété de l'architecture, pas une vérification manuelle.

---

## Conclusion : la segmentation est une décision d'adressage

Trois principes que je retire de cette mise en place :

1. **Un VPN route, il ne cloisonne pas.** Donner accès au tunnel n'est pas donner un accès maîtrisé. Le contrôle d'accès se construit **au-dessus** de WireGuard, jamais dans WireGuard seul.
2. **Le plan d'adressage est la fondation.** Un sous-réseau par profil transforme le contrôle d'accès en règles stables par catégorie, au lieu d'une gestion individuelle ingérable. La complexité se résout au moment de la conception de l'adressage, pas à chaque ajout de pair.
3. **Trois couches valent mieux qu'une.** WireGuard (qui entre), nftables (qui parle à quoi sur l'hôte), Caddy (qui voit quelle appli). Chaque couche couvre une menace que les autres ne couvrent pas. C'est la défense en profondeur appliquée, pas récitée.

> [!note] Pour aller plus loin dans le vault
> - [[Self-hosting sécurisé avec Podman]]
> - [[Reverse proxy Caddy avec TLS automatique]]
> - [[Durcissement SELinux pour conteneurs]]
> - [[Stratégie de sauvegarde restic 3-2-1]]

---

*Retour d'expérience personnel. Adapte les plages d'adressage, les noms de domaine et le provider à ton infra. La matrice d'accès ci-dessus est un exemple : la tienne dépend de ta propre cartographie « qui doit voir quoi ».*
