---
title: Installer Stirling-PDF sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - stirling-pdf
  - podman
  - installation
  - securite
  - productivite
date: 07-06-2026
last_modified: 07-06-2026
reading-time: 10m
difficulty: tech-enthusiast
status: draft
---

# Installer Stirling-PDF sous Podman rootless

> [!abstract] TL;DR
> Stirling-PDF est une boîte à outils PDF web (fusion, découpe, compression, OCR, conversion, signature…) qui traite tout **localement** — l'argument de fond face aux outils PDF en ligne qui envoient tes documents sur des serveurs tiers. Conteneur unique, **sans stockage persistant de tes fichiers** (traitement à la volée). C'est le complément naturel de Paperless. Installation Podman rootless, accès VPN recommandé.

> [!info] Prérequis de lecture
> Socle de la série : [[Self-hosting sécurisé avec Podman]], [[Reverse proxy Caddy avec TLS automatique]], [[Configuration WireGuard self-hosted]], confinement MAC.

---

## L'argument : tes PDF ne quittent jamais ta machine

> [!success] Le sens de l'auto-hébergement ici
> « Compresser un PDF en ligne », c'est uploader un document — souvent administratif, donc sensible — sur un serveur inconnu. Stirling-PDF fait les mêmes opérations **chez toi**, sans que le fichier ne sorte. Pour qui héberge déjà Paperless avec des documents sensibles, c'est cohérent : la manipulation PDF rejoint la même logique de souveraineté sur les données.

Particularité technique agréable : Stirling-PDF **ne stocke pas** tes documents. Tu uploades, il traite, tu télécharges, c'est fini. Pas de volume de données sensibles à protéger, juste un peu de config.

---

## Le Quadlet

```ini
# ~/.config/containers/systemd/stirling-pdf.container
[Unit]
Description=Stirling-PDF

[Container]
Image=docker.io/stirlingtools/stirling-pdf:latest
ContainerName=stirling-pdf
AutoUpdate=registry
Network=stirling-internal.network

# Données de config légères (pas de documents persistants)
Volume=%h/stirling/configs:/configs:U,Z
# tmpfs pour le traitement à la volée : rien ne persiste
Tmpfs=/tmp:rw,size=1G

# Active OCR et fonctions avancées
Environment=DOCKER_ENABLE_SECURITY=true
Environment=INSTALL_BOOK_AND_ADVANCED_HTML_OPS=false

# Durcissement
NoNewPrivileges=true
DropCapability=ALL

[Service]
Restart=on-failure
RestartSec=10
MemoryMax=2G

[Install]
WantedBy=default.target
```

```bash
podman network create stirling-internal --internal
systemctl --user daemon-reload
systemctl --user start stirling-pdf.service
```

> [!info] Authentification optionnelle
> Avec `DOCKER_ENABLE_SECURITY=true`, Stirling-PDF peut activer une page de login (comptes locaux). Utile si tu l'exposes au-delà de toi-même. Combiné au VPN, ça suffit pour un usage perso/famille.

---

## Exposition

```caddyfile
pdf.mondomaine.fr {
	import security_headers
	request_body {
		max_size 500MB   # certains PDF sont lourds
	}
	@allowed remote_ip 10.10.1.0/24 10.10.2.0/24
	handle @allowed {
		reverse_proxy stirling-pdf:8080
	}
	respond "Forbidden" 403
}
```

> [!tip] Pourquoi VPN même sans données persistantes
> Même si Stirling-PDF ne stocke rien, les documents que tu y traites **transitent** par lui. Le garder derrière VPN évite qu'une faille de l'appli expose ce flux. Et comme c'est un outil personnel, l'exposition publique n'apporte rien.

---

## Sauvegarde

> [!success] Quoi sauvegarder
> - **`~/stirling/configs`** : réglages et éventuels comptes. Léger.
> - **Rien d'autre** : Stirling-PDF ne conserve pas tes documents, il n'y a pas de données métier à sauvegarder. C'est un des rares services quasi *stateless* de la série.

---

## Conclusion

Stirling-PDF est l'outil d'appoint idéal : *stateless*, durcissable au maximum, et porteur du même message que tout ton parc — tes documents restent chez toi. Il complète parfaitement Paperless pour qui veut manipuler ses PDF sans dépendre d'un service en ligne.

> [!note] Articles liés
> - [[Installer Paperless-ngx sous Podman rootless]]
> - [[Configuration WireGuard self-hosted]]

---

*Retour d'expérience personnel. Adapte domaines et plages VPN.*
