---
title: Installer ESPHome sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - esphome
  - domotique
  - diy
  - podman
  - installation
  - securite
date: 07-06-2026
last_modified: 07-06-2026
reading-time: 15m
difficulty: tech-enthusiast
status: draft
---

# Installer ESPHome sous Podman rootless

> [!abstract] TL;DR
> ESPHome transforme des microcontrôleurs bon marché (ESP32, ESP8266) en **capteurs et actionneurs domotiques** configurés par YAML, intégrés nativement à Home Assistant. Le conteneur ESPHome est un **atelier de compilation et de flash** : tu y écris la config de tes appareils DIY, il compile le firmware et le téléverse. Deux particularités : il a besoin d'accéder aux **ports série USB** pour le premier flash (ensuite c'est OTA par Wi-Fi), et c'est un service d'**atelier** plus qu'un service tournant en permanence.

> [!info] Prérequis de lecture
> Socle de la série, et [[Installer Home Assistant sous Podman rootless]] pour le contexte domotique. ESPHome complète HA côté capteurs DIY, là où [[Installer Zigbee2MQTT et Mosquitto sous Podman rootless]] couvre l'écosystème Zigbee.

---

## ESPHome dans l'écosystème domotique

> [!success] Où il se situe
> - **Zigbee2MQTT** → appareils Zigbee du commerce (ampoules, capteurs Aqara…).
> - **ESPHome** → appareils **que tu fabriques ou flashes toi-même** sur base ESP32/ESP8266 (capteur de température maison, contrôleur d'arrosage, détecteur custom).
> - Les deux s'intègrent à Home Assistant, mais ESPHome est l'atelier du bricoleur électronique.
>
> ESPHome communique avec HA via son protocole natif (API chiffrée) sur le Wi-Fi, une fois l'appareil flashé. Pas besoin de MQTT, contrairement à Zigbee2MQTT.

---

## La particularité : un service d'atelier

> [!info] ESPHome ne « tourne » pas comme les autres
> Contrairement à Immich ou Jellyfin qui servent en continu, ESPHome est surtout un **environnement de configuration et de compilation**. Tu l'utilises pour créer/modifier la config d'un appareil, compiler son firmware, le flasher. Une fois tes appareils déployés et stables, le conteneur ESPHome peut même rester éteint la plupart du temps — tu le rallumes pour une mise à jour de config. C'est une nuance d'usage : ne le traite pas comme un service critique 24/7.

---

## Le défi : l'accès série pour le premier flash

> [!warning] USB nécessaire uniquement au premier flash
> Un ESP32 neuf doit être flashé **par câble USB** la première fois. Ensuite, ESPHome pousse les mises à jour **OTA** (over-the-air) par Wi-Fi — plus besoin d'USB. Donc l'accès device n'est requis qu'au démarrage de chaque nouvel appareil :
> ```ini
> AddDevice=/dev/serial/by-id/usb-xxxx:/dev/ttyUSB0
> ```
> Même logique que Zigbee2MQTT : chemin stable `by-id`, groupe `dialout`. Si tu préfères, beaucoup flashent le tout premier firmware depuis un autre outil (navigateur Web Serial sur un poste) puis font **tout** en OTA via ESPHome — ce qui évite de donner l'accès USB au conteneur.

---

## Le Quadlet

```ini
# ~/.config/containers/systemd/esphome.container
[Unit]
Description=ESPHome

[Container]
Image=ghcr.io/esphome/esphome:latest
ContainerName=esphome
AutoUpdate=registry

# ESPHome a besoin du réseau local pour l'OTA et la découverte mDNS des appareils
Network=host

# Configurations YAML de tes appareils : volume dédié
Volume=%h/esphome/config:/config:U,Z

# Accès USB pour le premier flash (optionnel si tu flashes en OTA uniquement)
AddDevice=/dev/serial/by-id/usb-xxxx:/dev/ttyUSB0

NoNewPrivileges=true
DropCapability=ALL

[Service]
Restart=on-failure
RestartSec=10
MemoryMax=2G

[Install]
WantedBy=default.target
```

> [!warning] network=host pour l'OTA et mDNS
> Comme Home Assistant, ESPHome bénéficie du réseau hôte pour découvrir les appareils sur le LAN (mDNS) et pousser les mises à jour OTA. C'est le même compromis d'isolation que HA : acceptable pour un service d'atelier non exposé, à compenser par MAC + pas d'exposition publique. Si tes appareils ont des IP fixes et que tu n'as pas besoin de découverte, un réseau ponté avec le port publié peut suffire.

> [!warning] La compilation est gourmande
> Compiler un firmware ESP est intensif (CPU + RAM, plusieurs minutes). D'où `MemoryMax=2G`. Sur un petit serveur, lance les compilations quand les autres services sont peu sollicités.

```bash
mkdir -p ~/esphome/config
systemctl --user daemon-reload
systemctl --user start esphome.service
```

---

## Exposition : atelier strictement privé

```caddyfile
esphome.mondomaine.fr {
	import security_headers
	@allowed remote_ip 10.10.1.0/24   # admin uniquement (network=host : localhost:6052)
	handle @allowed {
		reverse_proxy localhost:6052
	}
	respond "Forbidden" 403
}
```

> [!danger] ESPHome compile et flashe du code sur des appareils physiques
> L'interface ESPHome permet de modifier le firmware d'appareils qui peuvent contrôler des éléments physiques. Accès **admin-only** absolu, jamais exposé. Avec `network=host`, ESPHome écoute sur le port 6052 de l'hôte (ajuste `localhost`/`host.containers.internal` selon ta config Caddy, comme pour HA).

> [!info] Le secret d'API et le chiffrement
> Chaque appareil ESPHome communique avec HA via une clé de chiffrement d'API. ESPHome génère et gère ces clés dans les configs YAML. Elles sont sensibles : qui les a peut contrôler l'appareil. Elles font partie de ce qu'on sauvegarde.

---

## Sauvegarde

> [!success] Quoi sauvegarder
> - **`~/esphome/config`** : **toutes les configs YAML** de tes appareils, leurs clés d'API/OTA, les secrets. C'est l'essentiel — recréer la config de 15 capteurs custom à la main serait long.
> - Le cache de compilation (souvent dans le même volume) est jetable mais volumineux ; tu peux l'exclure pour alléger la sauvegarde.

---

## Conclusion

ESPHome est l'atelier domotique du bricoleur : un service de **configuration et compilation** plus qu'un service permanent. Ses particularités — accès USB au premier flash (contournable par OTA), `network=host` pour la découverte, compilation gourmande — en font un cas à part. Comme toute la domotique : admin-only strict, car il programme des appareils qui agissent sur le monde physique. Tes configs YAML sont le trésor à sauvegarder.

> [!note] Articles liés
> - [[Installer Home Assistant sous Podman rootless]]
> - [[Installer Zigbee2MQTT et Mosquitto sous Podman rootless]]
> - [[Stratégie de sauvegarde restic 3-2-1]]

---

*Retour d'expérience personnel. Adapte le chemin du device, les domaines, les plages VPN. Tes configs YAML d'appareils sont irremplaçables : sauvegarde-les.*
