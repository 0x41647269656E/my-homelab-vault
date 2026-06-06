---
title: Installer Syncthing sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - syncthing
  - podman
  - installation
  - securite
date: 06-06-2026
last_modified: 06-06-2026
reading-time: 15m
status: draft
---

# Installer Syncthing sous Podman rootless

> [!abstract] TL;DR
> Syncthing synchronise des dossiers en **pair-à-pair chiffré**, sans serveur central, entre tes appareils. C'est un cas différent des autres services : **un seul conteneur**, mais qui a besoin de **ports dédiés** (transfert, découverte) et qui synchronise directement des dossiers de `/media`. Je l'installe en Podman rootless, j'expose le strict minimum, et j'accède à son interface d'admin **via WireGuard uniquement**. Le point d'attention : Syncthing **écrit** dans les dossiers synchronisés, donc le montage `/media` se gère avec soin (sous-dossiers dédiés, `keep-id`, jamais `:U` sur la racine).

> [!info] Prérequis de lecture
> Socle : [[Self-hosting sécurisé avec Podman]], [[Reverse proxy Caddy avec TLS automatique]], [[Configuration WireGuard self-hosted]], confinement MAC, [[Stratégie de sauvegarde restic 3-2-1]]. La règle de montage `/media` est détaillée dans [[Installer Immich sous Podman rootless]].

---

## Ce qui rend Syncthing différent

Les autres services de la série sont des applications web derrière un reverse proxy. Syncthing est un **protocole de synchronisation P2P** : il établit des connexions directes (ou relayées) avec tes autres appareils. Ça change deux choses.

> [!info] Les ports de Syncthing
> - **`8384/tcp`** — l'interface d'administration web. Sensible : qui y accède contrôle la synchro. → derrière VPN.
> - **`22000/tcp` + `22000/udp`** — le transport des données synchronisées (protocole BEP + QUIC). → exposé pour permettre les connexions entrantes des pairs.
> - **`21027/udp`** — la découverte locale (broadcast LAN). → utile sur le réseau local, inutile à exposer sur Internet.

Le transport (22000) peut être exposé pour les connexions directes, ou tu peux t'appuyer sur les **relais** de Syncthing (chiffrés de bout en bout) et n'exposer aucun port entrant. Pour des pairs sur ton seul VPN, tout peut même rester dans le tunnel.

> [!warning] La découverte réseau en conteneur rootless
> La découverte locale par broadcast fonctionne mal à travers le NAT des conteneurs rootless. Deux options : accepter de ne pas avoir la découverte LAN auto (les pairs s'ajoutent par ID, ce qui est de toute façon plus sûr), ou utiliser un réseau Podman en mode plus permissif. Je préfère **ajouter les pairs manuellement par leur ID** — c'est explicite et ça évite d'exposer la découverte.

---

## Le montage des dossiers synchronisés

Syncthing **lit et écrit** les dossiers qu'il synchronise. Si je synchronise un dossier de `/media`, ce n'est donc pas du `:ro`.

> [!success] La bonne approche pour /media avec Syncthing
> - **Sous-dossiers dédiés à la synchro**, jamais la racine `/media` entière. Par exemple `/media/sync/documents`, `/media/sync/phone-camera`.
> - **`keep-id`** pour mapper ton UID hôte sans `chown` destructeur, afin que les fichiers synchronisés gardent une propriété cohérente côté hôte (et restent lisibles par d'autres services si besoin).
> - **Attention au croisement avec d'autres services** : si Syncthing écrit dans un dossier qu'Immich lit en `:ro`, c'est parfait (Syncthing alimente, Immich indexe). Mais évite que deux services **écrivent** le même dossier — conflits garantis.

> [!tip] Le pattern « Syncthing alimente, un autre service consomme »
> Mon usage favori : Syncthing pousse les photos de mon téléphone vers `/media/sync/phone-camera`, qu'Immich monte en lecture seule comme bibliothèque externe. Ou Syncthing dépose des scans dans `/media/paperless-inbox` que Paperless consomme. Syncthing devient le **transport**, les autres services les **consommateurs**. Chacun son rôle, pas de double écriture.

---

## Le Quadlet

```ini
# ~/.config/containers/systemd/syncthing.container
[Unit]
Description=Syncthing

[Container]
Image=docker.io/syncthing/syncthing:latest
ContainerName=syncthing
AutoUpdate=registry

# Réseau : Syncthing a besoin de joindre ses pairs, pas de --internal strict
# On publie les ports nécessaires
PublishPort=22000:22000/tcp
PublishPort=22000:22000/udp
# L'interface d'admin (8384) n'est PAS publiée ici : accès via Caddy/VPN

# --- Configuration et état de Syncthing : volume dédié ---
Volume=%h/syncthing/config:/var/syncthing/config:U,Z

# --- Dossiers synchronisés depuis /media : sous-dossiers dédiés, keep-id ---
Volume=/media/sync:/var/syncthing/sync:z

# Mapping d'UID pour cohérence des permissions
Environment=PUID=1000
Environment=PGID=1000

# Durcissement
NoNewPrivileges=true
DropCapability=ALL

[Service]
Restart=on-failure
RestartSec=10
MemoryMax=1G

[Install]
WantedBy=default.target
```

> [!info] Pourquoi 8384 n'est pas publié
> L'interface d'admin ne doit jamais être directement exposée. Je ne publie pas son port sur l'hôte ; à la place, je connecte Syncthing au réseau interne et je le proxifie via Caddy derrière VPN (bloc ci-dessous). Le transport (22000) lui est publié car les pairs doivent pouvoir s'y connecter.

```bash
mkdir -p /media/sync
systemctl --user daemon-reload
systemctl --user start syncthing.service
```

---

## Exposition : interface d'admin derrière VPN

Pour atteindre l'interface 8384 proprement, je la place sur un réseau joignable par Caddy et je filtre par VPN :

```caddyfile
sync.mondomaine.fr {
	import security_headers
	@allowed remote_ip 10.10.1.0/24   # admin uniquement : c'est MA synchro
	handle @allowed {
		reverse_proxy syncthing:8384
	}
	respond "Forbidden" 403
}
```

> [!warning] L'interface d'admin contrôle tes données
> Qui accède à l'admin Syncthing peut ajouter des dossiers, des pairs, exfiltrer. Elle reste **admin-only** (sous-réseau `10.10.1.0/24`). Syncthing protège aussi son interface par mot de passe : active-le en complément (défense en profondeur). Le transport P2P (22000) est chiffré de bout en bout et authentifié par ID d'appareil, donc l'exposer est acceptable ; l'admin, non.

---

## Sauvegarde

> [!success] Quoi sauvegarder pour Syncthing
> - **`~/syncthing/config`** : l'identité de l'appareil (clés), la liste des pairs, la config des dossiers. **Crucial** : sans ces clés, ton appareil change d'ID et tous les pairs doivent te ré-autoriser.
> - **Les dossiers synchronisés eux-mêmes** : ils sont déjà répliqués sur tes autres appareils (c'est le principe), mais Syncthing **n'est pas une sauvegarde** — une suppression se propage à tous les pairs.

> [!danger] Syncthing ≠ sauvegarde
> Erreur classique : croire que synchroniser = sauvegarder. Si tu supprimes ou chiffres un fichier (ransomware), la suppression/altération **se réplique** sur tous les pairs. Syncthing réplique l'état courant, il ne versionne pas par défaut. Pour de la vraie sauvegarde de ces dossiers, ils doivent entrer dans ta stratégie restic (cf. [[Stratégie de sauvegarde restic 3-2-1]]). Active aussi le **versioning de fichiers** dans Syncthing comme filet supplémentaire, mais ça ne remplace pas une sauvegarde hors-ligne.

---

## Conclusion

Syncthing est le cas du service **P2P avec ports dédiés**, différent des applis web de la série. Les points clés : ne publier que le transport (chiffré, authentifié), garder l'admin derrière VPN, monter des **sous-dossiers `/media` dédiés** en `keep-id` pour la synchro bidirectionnelle, et surtout ne jamais le confondre avec une sauvegarde. Son meilleur usage est en **transport** alimentant d'autres services (Immich, Paperless) qui consomment en lecture seule.

> [!note] Articles liés
> - [[Installer Immich sous Podman rootless]]
> - [[Installer Paperless-ngx sous Podman rootless]]
> - [[Stratégie de sauvegarde restic 3-2-1]]

---

*Retour d'expérience personnel. Adapte chemins, domaines, ports et images. La gestion réseau de Syncthing en rootless mérite des tests selon ta topologie.*
