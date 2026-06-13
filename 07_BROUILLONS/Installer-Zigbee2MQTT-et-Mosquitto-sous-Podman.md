---
title: Installer Zigbee2MQTT et Mosquitto sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - zigbee2mqtt
  - mosquitto
  - mqtt
  - domotique
  - podman
  - installation
  - securite
date: 07-06-2026
last_modified: 07-06-2026
reading-time: 15m
status: draft
---

# Installer Zigbee2MQTT et Mosquitto sous Podman rootless

> [!abstract] TL;DR
> Zigbee2MQTT fait le pont entre tes appareils Zigbee (ampoules, capteurs, prises) et le reste de ton infra, via le protocole **MQTT** servi par **Mosquitto** (le broker). C'est la brique annoncée dans [[Installer Home Assistant sous Podman rootless]] pour **isoler le besoin matériel** : c'est Zigbee2MQTT qui détient le dongle USB, pas Home Assistant. Le défi central est donc l'**accès au périphérique série** en rootless — le point le plus délicat de la domotique conteneurisée.

> [!info] Prérequis de lecture
> Socle de la série, et [[Installer Home Assistant sous Podman rootless]] pour le contexte domotique et le pattern d'isolation du matériel.

---

## Pourquoi découpler la radio de Home Assistant

> [!success] L'intérêt de l'architecture MQTT
> Plutôt que de donner le dongle USB à Home Assistant (qui devient alors couplé au matériel), on isole le besoin :
> - **Zigbee2MQTT** détient le dongle, parle au réseau Zigbee, et publie l'état des appareils sur MQTT.
> - **Mosquitto** est le broker MQTT central : il transporte les messages.
> - **Home Assistant** (et d'autres) s'abonnent à MQTT, sans jamais toucher au matériel.
>
> Résultat : un seul conteneur a besoin de l'accès device (Zigbee2MQTT), HA reste « propre », et tu peux redémarrer/migrer HA sans perturber le réseau Zigbee. Le découplage matériel est la bonne pratique.

---

## Le défi : le périphérique série en rootless

> [!warning] Accès au dongle USB en rootless
> Zigbee2MQTT a besoin de lire/écrire le dongle (`/dev/ttyACM0`, `/dev/ttyUSB0`, ou mieux un chemin stable `/dev/serial/by-id/...`). En rootless :
> - Ton utilisateur doit appartenir au groupe propriétaire du device (souvent `dialout`).
> - Le device doit être passé au conteneur via `AddDevice`.
> - Le mapping de groupe doit permettre l'accès depuis le conteneur.
>
> Identifie d'abord le chemin **stable** de ton dongle (qui ne change pas au reboot) :
> ```bash
> ls -l /dev/serial/by-id/
> # ex: usb-ITead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_xxxx -> ../../ttyACM0
> ```
> Pointe toujours le chemin `by-id`, jamais `ttyACM0` qui peut bouger.

---

## Mosquitto (le broker)

```bash
podman network create mqtt-internal --internal
```

```ini
# ~/.config/containers/systemd/mosquitto.container
[Unit]
Description=Mosquitto MQTT Broker
[Container]
Image=docker.io/library/eclipse-mosquitto:2
ContainerName=mosquitto
AutoUpdate=registry
Network=mqtt-internal.network
Volume=%h/mosquitto/config:/mosquitto/config:U,Z
Volume=%h/mosquitto/data:/mosquitto/data:U,Z
NoNewPrivileges=true
DropCapability=ALL
[Service]
Restart=on-failure
MemoryMax=256M
[Install]
WantedBy=default.target
```

> [!danger] Mosquitto sans authentification = accès ouvert à ta domotique
> Par défaut, certaines configs Mosquitto autorisent les connexions anonymes. **Active l'authentification** (`config/mosquitto.conf`) :
> ```
> listener 1883
> allow_anonymous false
> password_file /mosquitto/config/passwd
> ```
> Crée les identifiants :
> ```bash
> podman exec mosquitto mosquitto_passwd -c /mosquitto/config/passwd zigbee2mqtt
> ```
> Un broker MQTT ouvert, c'est laisser n'importe qui sur le réseau commander tes ampoules et lire tes capteurs. Le réseau interne `--internal` limite déjà la portée, mais l'auth est la bonne pratique.

---

## Zigbee2MQTT (avec le dongle)

```ini
# ~/.config/containers/systemd/zigbee2mqtt.container
[Unit]
Description=Zigbee2MQTT
Requires=mosquitto.service
After=mosquitto.service
[Container]
Image=docker.io/koenkk/zigbee2mqtt:latest
ContainerName=zigbee2mqtt
AutoUpdate=registry
Network=mqtt-internal.network

Volume=%h/zigbee2mqtt/data:/app/data:U,Z

# LE point clé : accès au dongle Zigbee (chemin stable by-id)
AddDevice=/dev/serial/by-id/usb-ITead_Sonoff_Zigbee_xxxx:/dev/ttyACM0

Environment=TZ=Europe/Paris

NoNewPrivileges=true
DropCapability=ALL

[Service]
Restart=on-failure
RestartSec=10
MemoryMax=512M

[Install]
WantedBy=default.target
```

La config (`~/zigbee2mqtt/data/configuration.yaml`) relie Zigbee2MQTT au broker :

```yaml
mqtt:
  server: mqtt://mosquitto:1883
  user: zigbee2mqtt
  password: !secret mqtt_password
serial:
  port: /dev/ttyACM0
frontend:
  port: 8080
```

```bash
systemctl --user daemon-reload
systemctl --user start zigbee2mqtt.service
journalctl --user -u zigbee2mqtt.service -f   # vérifier que le dongle est détecté
```

---

## Exposition : interface d'admin VPN-only

```caddyfile
zigbee.mondomaine.fr {
	import security_headers
	@allowed remote_ip 10.10.1.0/24   # admin uniquement
	handle @allowed {
		reverse_proxy zigbee2mqtt:8080
	}
	respond "Forbidden" 403
}
```

> [!warning] L'interface contrôle tes appareils physiques
> L'admin Zigbee2MQTT permet d'appairer/supprimer des appareils, de commander tes équipements. Strictement **admin-only**. La domotique touche au monde physique (serrures, chauffage) : la rigueur d'accès y est encore plus importante qu'ailleurs.

---

## Connexion à Home Assistant

Une fois Mosquitto et Zigbee2MQTT en place, HA s'y connecte via l'intégration MQTT (en pointant vers `mosquitto:1883` avec les identifiants). Zigbee2MQTT supporte la **découverte automatique** MQTT : tes appareils Zigbee apparaissent tout seuls dans HA. C'est tout l'intérêt du découplage — HA voit les appareils sans jamais toucher le dongle.

---

## Sauvegarde

> [!success] Quoi sauvegarder
> - **`~/zigbee2mqtt/data`** : `configuration.yaml` **et surtout** `coordinator_backup.json` + la base réseau. **Crucial** : ce backup contient les clés du réseau Zigbee. Sans lui, en cas de perte, tu dois **réappairer tous tes appareils un par un** — pénible avec 30 capteurs.
> - **`~/mosquitto/config`** : config et identifiants du broker.

---

## Conclusion

Zigbee2MQTT + Mosquitto est l'architecture domotique propre : le matériel isolé dans un seul conteneur, MQTT comme bus de communication, HA découplé. Les deux points de vigilance sont l'**accès au dongle en rootless** (chemin stable `by-id`, groupe `dialout`) et l'**authentification du broker** (jamais de MQTT anonyme). Et comme toute la domotique : admin VPN-only, car ça commande le monde physique.

> [!note] Articles liés
> - [[Installer Home Assistant sous Podman rootless]]
> - [[Installer ESPHome sous Podman rootless]]
> - [[Stratégie de sauvegarde restic 3-2-1]]

---

*Retour d'expérience personnel. Adapte le chemin du dongle, les domaines, les plages VPN. Le backup du coordinateur Zigbee est vital : sauvegarde-le.*
