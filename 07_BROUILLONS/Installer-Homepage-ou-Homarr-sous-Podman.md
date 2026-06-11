---
title: Installer Homepage ou Homarr sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - homepage
  - homarr
  - dashboard
  - podman
  - installation
  - securite
date: 07-06-2026
last_modified: 07-06-2026
reading-time: 15m
status: draft
---

# Installer Homepage ou Homarr sous Podman rootless

> [!abstract] TL;DR
> Un **tableau de bord d'accueil** unifie l'accès à tous tes services derrière une seule page : liens, état des services, widgets (météo, ressources, flux). **Homepage** est configuré par fichiers YAML (déclaratif, versionnable, léger). **Homarr** est configuré via une interface graphique (drag & drop, plus accessible). Les deux peuvent afficher l'**état** de tes services et des widgets dynamiques. Le point de sécurité : pour afficher des stats, ils veulent souvent des **API keys** de tes autres services — à gérer avec soin.

> [!info] Prérequis de lecture
> Socle de la série. Caddy, VPN, MAC, restic.

---

## Le point d'orgue du parc

Après avoir installé une douzaine de services, le dashboard est ce qui rend l'ensemble **utilisable au quotidien** : une page d'accueil unique avec tout à portée de clic, l'état de chaque service, et quelques widgets utiles. C'est le dernier service qu'on installe, celui qui chapeaute les autres.

---

## Choisir : Homepage ou Homarr ?

> [!info] Le bon choix
> - **Homepage** → configuration par **fichiers YAML**. Déclaratif, léger, versionnable (tu peux committer ta config dans GitLab), reproductible. Idéal pour un profil devops qui aime l'infra-as-code. Pas d'interface d'édition : tu édites des fichiers.
> - **Homarr** → configuration par **interface graphique** (glisser-déposer, édition visuelle). Plus accessible, plus joli out-of-the-box, mais l'état vit dans une base plutôt que dans des fichiers versionnables.
>
> Pour un public technophile/devops, **Homepage** colle à la philosophie déclarative de toute la série (Quadlets, Caddyfile, tout en fichiers). Je le présente ; Homarr suit la même logique de déploiement.

---

## Homepage : le Quadlet

```ini
# ~/.config/containers/systemd/homepage.container
[Unit]
Description=Homepage Dashboard

[Container]
Image=ghcr.io/gethomepage/homepage:latest
ContainerName=homepage
AutoUpdate=registry
Network=homepage-internal.network

# Config en fichiers YAML : éditables, versionnables
Volume=%h/homepage/config:/app/config:U,Z

# Domaines autorisés (sécurité Homepage récente)
Environment=HOMEPAGE_ALLOWED_HOSTS=home.mondomaine.fr

NoNewPrivileges=true
DropCapability=ALL
ReadOnly=true
Tmpfs=/tmp:rw,size=64M

[Service]
Restart=on-failure
RestartSec=10
MemoryMax=256M

[Install]
WantedBy=default.target
```

```bash
podman network create homepage-internal --internal
systemctl --user daemon-reload
systemctl --user start homepage.service
```

> [!info] La config en fichiers
> Homepage lit `services.yaml`, `widgets.yaml`, `settings.yaml`, `bookmarks.yaml` dans son dossier config. Tu déclares tes services, leurs liens, leurs widgets. Exemple minimal de `services.yaml` :
> ```yaml
> - Médias:
>     - Jellyfin:
>         href: https://media.mondomaine.fr
>         description: Films et séries
>         icon: jellyfin.png
> - Documents:
>     - Paperless:
>         href: https://docs.mondomaine.fr
>         icon: paperless.png
> ```

---

## Le point de sécurité : les widgets et leurs API keys

> [!warning] Les widgets veulent des accès
> Pour afficher « Immich : 12 000 photos » ou « disque : 60 % », Homepage/Homarr interroge l'API du service concerné, ce qui demande une **API key**. Tu te retrouves donc à stocker, dans la config du dashboard, des clés vers tes autres services. Conséquences :
> - Utilise des clés **en lecture seule** / à **permissions minimales** quand le service le permet.
> - Stocke-les via des **variables d'environnement secrètes** (Homepage supporte `{{HOMEPAGE_VAR_...}}`), pas en clair dans les YAML versionnés — sinon tu committes des secrets dans GitLab.
> - Le dashboard devient un mini-concentrateur d'accès : raison de plus pour le garder **derrière VPN**.

> [!danger] Ne committe jamais tes API keys
> Si tu versionnes ta config Homepage (excellente idée), assure-toi que les fichiers contenant des clés sont exclus ou que les clés passent par des variables d'environnement référencées. Un `git push` de tes API keys vers un dépôt, même privé, est une fuite.

---

## Exposition

```caddyfile
home.mondomaine.fr {
	import security_headers
	@allowed remote_ip 10.10.1.0/24 10.10.2.0/24
	handle @allowed {
		reverse_proxy homepage:3000
	}
	respond "Forbidden" 403
}
```

> [!info] Pourquoi VPN
> Le dashboard liste **tous** tes services et leur état : c'est une carte de ton infra. Plus le fait qu'il détient des API keys. Il reste derrière VPN (admin + famille selon ce que tu partages). C'est ta page d'accueil privée, pas une vitrine publique.

---

## Sauvegarde

> [!success] Quoi sauvegarder
> - **Homepage** : `~/homepage/config` (tous les YAML). Léger, idéalement déjà versionné dans GitLab (hors secrets). C'est tout.
> - **Homarr** : sa base (config visuelle, intégrations). Volume de données à inclure dans restic.

---

## Conclusion

Le dashboard est la touche finale qui transforme une collection de services en un parc cohérent et utilisable. Homepage pour l'approche déclarative versionnable (cohérente avec tout le reste de la série), Homarr pour l'édition visuelle. Le seul vrai enjeu de sécurité est la gestion des **API keys des widgets** : permissions minimales, secrets jamais committés, et dashboard derrière VPN car il cartographie tout ton parc.

> [!note] Articles liés
> - [[Installer Vikunja sous Podman rootless]]
> - [[Installer GitLab CE sous Podman rootless]]
> - [[Configuration WireGuard self-hosted]]

---

*Retour d'expérience personnel. Adapte domaines, images, et surtout la gestion des secrets de widgets.*
