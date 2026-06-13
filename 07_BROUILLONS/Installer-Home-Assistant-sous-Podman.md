---
title: Installer Home Assistant sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - home-assistant
  - podman
  - installation
  - securite
  - domotique
date: 06-06-2026
last_modified: 06-06-2026
reading-time: 15m
status: draft
---

# Installer Home Assistant sous Podman rootless

> [!abstract] TL;DR
> Home Assistant (HA) Container est le hub domotique open-source. C'est le cas le plus **frottant avec le rootless** de toute la série : pour découvrir les appareils du réseau local (mDNS, SSDP) et piloter des dongles USB (Zigbee/Z-Wave), il veut souvent le **réseau de l'hôte** et l'**accès à des périphériques** — deux choses que le confinement rootless restreint par nature. Cet article explique les compromis à faire, comment les minimiser, et comment exposer l'interface en sécurité. Peu de lien avec `/media`, sauf pour des sauvegardes ou médias TTS.

> [!info] Prérequis de lecture
> Socle : [[Self-hosting sécurisé avec Podman]], [[Reverse proxy Caddy avec TLS automatique]], [[Configuration WireGuard self-hosted]], confinement MAC, [[Stratégie de sauvegarde restic 3-2-1]]. Note : on parle ici de **HA Container**, pas de HA OS (qui veut une VM dédiée) ni des add-ons (indisponibles hors HA OS/Supervised).

---

## Le conflit de fond : domotique vs confinement

Tous les services précédents s'isolaient bien : réseau interne, pas de périphérique, rootfs en lecture seule. Home Assistant casse ce moule, parce que la domotique a des besoins matériels et réseau que l'isolation gêne.

> [!warning] Les trois tensions avec le rootless
> 1. **Découverte réseau (mDNS/SSDP)** : HA détecte automatiquement Chromecast, imprimantes, ampoules… via des protocoles de broadcast qui traversent mal le NAT des conteneurs rootless. Pour que la découverte marche, HA veut le **réseau de l'hôte** (`network=host`).
> 2. **Périphériques USB** (dongle Zigbee/Z-Wave type ConBee, SkyConnect) : piloter ces radios demande l'accès à `/dev/ttyUSB*` ou `/dev/ttyACM*`. En rootless, accéder à un device hôte n'est pas trivial et peut nécessiter des ajustements de permissions.
> 3. **Intégrations locales** : beaucoup d'intégrations supposent une connectivité directe au LAN, pas un réseau conteneur isolé.

Chacune de ces tensions est un compromis de sécurité. L'enjeu de l'article : **n'accorder que ce qui est strictement nécessaire**, et compenser ailleurs.

---

## Les deux scénarios de déploiement

### Scénario A — réseau hôte (découverte locale nécessaire)

Si tu as des appareils à découvrir automatiquement sur ton LAN, HA a besoin du réseau hôte. C'est le scénario le plus courant.

```ini
# ~/.config/containers/systemd/homeassistant.container
[Unit]
Description=Home Assistant

[Container]
Image=ghcr.io/home-assistant/home-assistant:stable
ContainerName=homeassistant
AutoUpdate=registry

# Réseau de l'hôte : nécessaire pour mDNS/SSDP/découverte LAN
Network=host

# Configuration HA sur volume dédié (aucun lien avec /media)
Volume=%h/homeassistant/config:/config:U,Z
# Fuseau horaire pour les automatisations basées sur l'heure
Volume=/etc/localtime:/etc/localtime:ro

Environment=TZ=Europe/Paris

# Durcissement (limité par network=host, mais on garde le reste)
NoNewPrivileges=true

[Service]
Restart=on-failure
RestartSec=10
MemoryMax=2G

[Install]
WantedBy=default.target
```

> [!danger] `network=host` réduit l'isolation réseau
> Avec `network=host`, HA partage la pile réseau de l'hôte : il voit toutes les interfaces, tous les ports locaux. C'est un **affaiblissement réel** de l'isolation par rapport au réseau interne des autres services. Je l'accepte **uniquement** parce que la découverte domotique l'exige, et je compense : confinement MAC actif, mises à jour suivies, et surtout HA **n'est jamais exposé publiquement** (VPN only). Si tu n'as pas besoin de découverte auto (appareils ajoutés manuellement par IP), préfère le scénario B.

### Scénario B — réseau ponté (pas de découverte auto)

Si tu ajoutes tes appareils manuellement par IP, tu peux garder HA sur un réseau Podman normal et préserver l'isolation :

```ini
# Variante : réseau dédié au lieu de host
Network=homeassistant.network
PublishPort=8123:8123
```

C'est plus sûr mais tu perds la découverte automatique. Mon arbitrage dépend du parc domotique : beaucoup d'appareils « plug and discover » → scénario A ; installation maîtrisée par IP fixes → scénario B.

---

## Les périphériques USB (Zigbee/Z-Wave)

Si tu pilotes une radio Zigbee/Z-Wave par dongle USB :

```ini
# Ajouter au Quadlet l'accès au device série
AddDevice=/dev/ttyACM0
# ou /dev/ttyUSB0 selon le dongle
```

> [!warning] Permissions du device en rootless
> En rootless, ton utilisateur doit avoir le droit de lire/écrire le device hôte. Concrètement : ajoute ton utilisateur au groupe propriétaire du device (souvent `dialout`), et vérifie que le mapping de groupe du conteneur permet l'accès. C'est le point qui demande le plus de tâtonnement. Une règle udev stable (pour que le dongle garde le même chemin) aide énormément :
> ```bash
> # Identifier le dongle
> ls -l /dev/serial/by-id/
> ```
> Préfère pointer le chemin stable `/dev/serial/by-id/...` plutôt que `/dev/ttyACM0` qui peut changer au reboot.

> [!tip] Découpler la radio Zigbee de HA
> Pour limiter le couplage matériel, beaucoup (dont moi) font tourner la radio Zigbee via **Zigbee2MQTT** dans son propre conteneur, qui parle à HA par MQTT. HA n'a alors plus besoin du device USB — seul Zigbee2MQTT l'a. Ça isole le besoin matériel dans un conteneur dédié et garde HA « propre ». C'est un bon candidat pour un article complémentaire.

---

## Exposition : VPN par défaut, l'app mobile via le tunnel

```caddyfile
home.mondomaine.fr {
	import security_headers
	@allowed remote_ip 10.10.1.0/24 10.10.2.0/24   # admin + famille
	handle @allowed {
		reverse_proxy localhost:8123   # network=host : HA écoute sur l'hôte
	}
	respond "Forbidden" 403
}
```

> [!info] Le reverse_proxy avec network=host
> Comme HA est en `network=host`, il écoute directement sur le port 8123 de l'hôte. Caddy (lui-même en conteneur) doit pouvoir joindre `localhost:8123` de l'hôte — selon ta config réseau Caddy, ça peut demander `host.containers.internal:8123` au lieu de `localhost`. Adapte selon ce qui résout dans ton montage.

> [!warning] HA et la configuration de confiance des proxys
> Home Assistant **refuse** par défaut les connexions passant par un proxy inconnu. Il faut déclarer Caddy comme proxy de confiance dans `configuration.yaml` :
> ```yaml
> http:
>   use_x_forwarded_for: true
>   trusted_proxies:
>     - 10.10.0.0/16   # le réseau d'où vient Caddy/VPN
> ```
> Sans ça, tu auras une erreur « 400 Bad Request » à travers le proxy. C'est le piège classique d'installation.

L'app mobile Home Assistant fonctionne très bien à travers WireGuard : le tunnel monte, l'app synchronise. Pas besoin d'exposer HA sur Internet pour le pilotage nomade.

---

## Sauvegarde

> [!success] Quoi sauvegarder pour Home Assistant
> - **`~/homeassistant/config`** : tout est là. `configuration.yaml`, les automatisations, l'historique (base SQLite `home-assistant_v2.db`), les intégrations, les secrets. C'est le seul répertoire qui compte.
> - HA propose aussi des **snapshots/backups internes** (Paramètres → Sauvegardes) qui produisent une archive propre — utile en complément avant une mise à jour majeure.

> [!tip] La base d'historique peut être volumineuse
> `home-assistant_v2.db` (le recorder) grossit vite avec l'historique des capteurs. Si tu n'as pas besoin de tout l'historique, configure le `recorder` pour purger régulièrement, ou bascule l'historique long terme vers une base externe. Ça allège aussi la sauvegarde. Pour une sauvegarde cohérente du SQLite, applique la même prudence que pour Vaultwarden (commande `.backup` plutôt que copie à chaud).

---

## Conclusion

Home Assistant est le service qui **assume des compromis d'isolation** au nom de sa fonction : réseau hôte pour la découverte, accès device pour les radios. La discipline consiste à **n'accorder que le strict nécessaire** (scénario B si la découverte n'est pas requise, Zigbee2MQTT pour isoler le matériel), à **compenser par le MAC et le VPN strict**, et à ne jamais l'exposer publiquement. C'est l'illustration que le self-hosting sécurisé n'est pas « tout isoler à tout prix » mais « comprendre chaque exception et la borner ».

> [!note] Articles liés
> - [[Self-hosting sécurisé avec Podman]]
> - [[Configuration WireGuard self-hosted]]
> - [[Stratégie de sauvegarde restic 3-2-1]]

---

*Retour d'expérience personnel. La domotique est très dépendante du matériel : adapte devices, réseau et intégrations à ton installation. Vérifie les options HA Container à jour.*
