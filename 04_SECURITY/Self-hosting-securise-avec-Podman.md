---
title: "Self-hosting sécurisé : pourquoi je suis passé à Podman (et pas à Kubernetes)"
author: "0x41647269656E"
series: "Hardening"
tags:
  - self-hosting
  - podman
  - securite
  - devops
  - conteneurs
reading-time: 15m
date: 05-06-2026
last_modified: 05-06-2026
status: published
---

# Self-hosting sécurisé : pourquoi je suis passé à Podman (et pas à Kubernetes)

> [!abstract] TL;DR
> J'héberge une dizaine de services (Immich, Paperless-ngx, Syncthing, VaultWarden, etc.) sur un serveur unique exposé en partie sur Internet. Après plusieurs années avec Docker et Docker-Compose, j'ai migré l'ensemble sur **Podman rootless** avec des **Quadlets** gérés par systemd. Cet article est un retour d'expérience sur les raisons de fond de ce choix : le modèle de sécurité du daemon Docker, le coût opérationnel réel de Kubernetes pour un parc mono-nœud, et ce que Podman change concrètement dans ma posture de défense.

---

## Le contexte : ce que j'héberge et pourquoi ça change tout

Avant de parler d'outillage, il faut poser le décor, parce que le bon choix dépend entièrement de la surface d'attaque qu'on accepte d'avoir.

Mon parc tient sur **un seul serveur** (un mini-PC x86 avec du stockage attaché, rien d'exotique). Dessus tournent grosso modo :

- **Immich** — galerie photo avec ML pour la reconnaissance faciale et le tagging. Exposé sur Internet derrière un reverse proxy, parce que je veux l'app mobile en déplacement.
- **Paperless-ngx** — GED qui ingère absolument tous mes documents administratifs. Données extrêmement sensibles. Accessible uniquement via VPN.
- **Syncthing** — synchronisation P2P, ouvre ses propres ports.
- **VaultWarden** — Gestionnaire de mots de passes, utilisant le port 80
- Quelques briques d'infra : un reverse proxy (Caddy), une base PostgreSQL mutualisée, du monitoring.

Le point crucial : **ces services manipulent mes données personnelles** (photos de famille, documents d'identité, relevés bancaires numérisés) et **une partie est joignable depuis Internet**. La question n'est donc pas « est-ce que ça marche », mais « qu'est-ce qui se passe le jour où Immich a une CVE d'exécution de code à distance ».

C'est cette question, et seulement elle, qui m'a fait reconsidérer Docker comme plateforme d'hébergement de mes conteneurs. La latéralisation est possible, 

---

## Le problème de fond #1 : le daemon Docker est un single point of pwnage

J'ai pendant des années hébergé mes conteneurs sous docker. D'un point de vue fonctionnel, ça fonctionnait très bien, impact sur les ressources maitrisé, mises à jours facilité, pas de conflit entre les installations... Le déclic m'est venu en lisant des articles post-mortems d'évasion de conteneurs.
En regardant les logs applicatifs, je me rends compte que mon réseau reçoit un grand nombre d'attaques automatisées. Il est donc nécessaire de s'en prémunir, à notre échelle.

### L'architecture daemon, expliquée par ses conséquences

Docker repose sur un **daemon central (`dockerd`) qui tourne en root**. Quand tu lances un conteneur, ton client parle à ce daemon, et c'est lui qui crée et possède le processus. Toute la chaîne de descendance des conteneurs remonte à un processus root persistant.

Trois conséquences concrètes, et elles sont toutes désagréables :

> [!danger] Le groupe `docker` ≈ root
> Quiconque peut parler au socket Docker peut faire ceci :
> ```bash
> docker run -v /:/host -it alpine chroot /host sh
> ```
> Et voilà un shell root sur l'hôte. **Appartenir au groupe `docker`, c'est être root sur la machine**, sans `sudo`, sans trace dans `auth.log`. Ce n'est pas un bug, c'est le design. La doc Docker le dit elle-même.

> [!danger] Les applications lancées en root à l'intérieur du conteneur
La deuxième conséquence est plus insidieuse. Beaucoup d'images, et beaucoup de tutos de self-hosting, font tourner le **processus applicatif en root à l'intérieur du conteneur**. Comme par défaut le conteneur partage le namespace user de l'hôte, `root` dans le conteneur **est** `root (UID 0)` sur l'hôte. Une évasion de conteneur — via une CVE du runtime, un montage mal fichu, un `--privileged` traîné dans un compose — te donne directement root sur la machine.

> [!danger] Le daemon : un anneau pour les contrôler tous
La troisième : le daemon est un **point de défaillance unique**. Il redémarre, tous tes conteneurs sautent. Il a une fuite mémoire, tout le monde trinque. Et surtout, sa surface d'attaque (l'API REST sur le socket) est un actif qu'il faut protéger en tant que tel.

### Ce que je voulais à la place

Une règle simple : **si Immich se fait compromettre, le blast radius doit s'arrêter à Immich**. Pas l'hôte, pas mes autres conteneurs, pas Paperless. Idéalement, l'attaquant se retrouve avec les droits d'un utilisateur non privilégié sans shell, dans un namespace isolé, sans rien d'intéressant à voler latéralement.

Docker rootless existe et a beaucoup progressé, je ne le nie pas. Mais c'est un mode greffé sur une architecture qui n'a pas été pensée pour ça au départ, avec son lot de limitations (cgroups v2 requis, contraintes réseau, `--privileged` problématique). Podman, lui, **est rootless par conception**.

---

## Podman : le modèle qui correspond au niveau de protection que je souhaite avoir

### Pas de daemon, fork-exec direct

Podman n'a **pas de daemon**. Quand je lance un conteneur, Podman fork-exec directement le runtime (`crun` chez moi) et le processus conteneur devient un **enfant de ma session utilisateur**. Pas d'intermédiaire root persistant, pas de socket privilégié à protéger.

> [!info] Conséquence directe
> La supervision des conteneurs est déléguée à **systemd**, qui sait déjà parfaitement faire ça (redémarrage, dépendances, journalisation, watchdog). On ne réinvente pas un superviseur, on réutilise le meilleur de l'écosystème Linux.

### Rootless + user namespaces : le vrai sujet

C'est ici que tout se joue. En rootless, Podman utilise les **user namespaces** et le mécanisme `subuid`/`subgid`. Concrètement :

- Mon utilisateur (disons UID `1000`) se voit attribuer une **plage d'UIDs subordonnés**, par exemple `100000–165535`, dans `/etc/subuid`.
- Le `root` **à l'intérieur** du conteneur (UID 0) est **mappé sur mon UID 1000** sur l'hôte.
- Les autres UIDs du conteneur sont mappés dans la plage `100000+`.

Vérifie ta config avec :

```bash
cat /etc/subuid /etc/subgid
# user:100000:65536
podman unshare cat /proc/self/uid_map
```

Le résultat est radical : **un attaquant qui devient root dans le conteneur et s'échappe se retrouve avec UID 1000 sur l'hôte** — un utilisateur ordinaire sans privilèges, qui ne possède que ses propres fichiers. Le `root` du conteneur n'est plus le `root` de la machine. C'est exactement la propriété de sécurité que je cherchais.

> [!warning] Le piège des permissions de volumes
> Cette élégance a un coût opérationnel qu'il faut comprendre. Comme les UIDs sont remappés, un fichier écrit par le conteneur apparaît sur l'hôte avec un UID dans la plage `100000+`, pas avec ton UID. Pour les volumes bind-mountés, tu **dois** utiliser l'option `:U` ou `keep-id` :
> ```bash
> podman run -v ./data:/data:U immich-server
> ```
> Le `:U` fait que Podman `chown` récursivement le volume vers l'UID mappé approprié. Sans ça, tu passeras une soirée à débugger des `permission denied` incompréhensibles. C'est **le** point de friction n°1 quand on migre depuis Docker. Je l'ai payé, tu n'es pas obligé.

### Compatibilité CLI : la migration coûte moins cher qu'on croit

Podman implémente la même CLI que Docker. La quasi-totalité de mes commandes ont fonctionné en remplaçant `docker` par `podman`, voire avec un simple alias :

```bash
alias docker=podman
```

`podman-compose` existe aussi et avale les fichiers `docker-compose.yml`. Mais — et c'est important pour la suite — **je ne l'utilise pas**, et c'est un choix délibéré que j'explique plus bas.

---

## Pourquoi PAS Kubernetes : le coût caché de la complexité

C'est la question qu'on me pose systématiquement. « Si tu veux du sérieux, pourquoi pas k8s, ou au moins k3s ? » Réponse honnête : **parce que Kubernetes résout des problèmes que je n'ai pas, en m'en créant d'autres bien réels.**

### Kubernetes répond à un problème de scale et de résilience multi-nœuds

Kubernetes brille quand tu as :

- **Plusieurs nœuds** entre lesquels orchestrer et rescheduler des charges.
- Un besoin de **haute disponibilité** avec bascule automatique.
- Du **scaling horizontal** dynamique en fonction de la charge.
- Une équipe et une chaîne CI/CD qui justifient l'abstraction déclarative.

J'ai **un seul nœud**. Si ce nœud tombe, aucun orchestrateur au monde ne va rescheduler Immich sur un nœud qui n'existe pas. La principale promesse de k8s — la résilience par orchestration multi-nœuds — **est structurellement inutile dans mon cas**.

### Le coût opérationnel est, lui, bien réel

> [!quote] Ce que k8s m'aurait coûté
> - **Des manifests YAML à maintenir** : Deployment, Service, Ingress, PersistentVolumeClaim, ConfigMap, Secret… pour chaque service. Multiplie par dix applis. C'est une codebase d'infrastructure à part entière, avec ses montées de version d'API (`apps/v1`, déprédations, etc.).
> - **Une couche réseau (CNI)** à comprendre et débugger : overlay, NetworkPolicies, CoreDNS.
> - **La gestion de l'etcd, des certificats du cluster, des upgrades** de version k8s — un sport à plein temps.
> - **Une surcharge mémoire/CPU permanente** du control plane, sur une machine où chaque Go compte pour les services qui m'intéressent vraiment.

Même k3s, qui allège énormément (binaire unique, SQLite au lieu d'etcd, batteries incluses), reste un orchestrateur. Il m'imposerait le modèle mental Kubernetes — pods, services, ingress, RBAC — pour faire tourner des conteneurs qui n'ont besoin que de démarrer au boot et de redémarrer s'ils plantent.

> [!tip] Le bon dimensionnement de l'outil
> La question n'est pas « quel est l'outil le plus puissant » mais « quel est l'outil dont la complexité est justifiée par mon problème ». Pour un parc mono-nœud, **systemd est déjà mon orchestrateur** : il gère les dépendances, les redémarrages, l'ordre de boot, la journalisation. Ajouter Kubernetes par-dessus, c'est mettre un grutier sur un chantier où je n'ai qu'à poser une étagère.

### Le pont élégant : `podman generate kube`

Et ironie de l'histoire : Podman parle quand même le Kubernetes **sans en avoir la lourdeur**. Il sait générer et consommer des manifests Pod/Deployment au format Kubernetes :

```bash
podman generate kube mon-pod > mon-app.yaml
podman kube play mon-app.yaml
```

Ça me donne une **porte de sortie** : si un jour mon parc grossit au point de justifier un vrai cluster, mes définitions sont déjà à moitié dans le bon format. Je n'ai pas peint dans un coin. C'est ça, choisir la simplicité sans fermer les portes.

---

## Mon architecture concrète : Quadlets + systemd

Voici le cœur de mon retour d'expérience, le truc que j'aurais aimé qu'on m'explique plus tôt.

### Abandonner `podman-compose` pour les Quadlets

Au début j'ai utilisé `podman-compose` parce que c'était le chemin de moindre résistance depuis Docker. Erreur. C'est un wrapper Python qui réimplémente la sémantique de Compose au-dessus de Podman, avec des écarts subtils et un cycle de vie déconnecté de l'init système. Les conteneurs ne survivaient pas proprement au reboot, et le démarrage n'était pas ordonnancé par systemd.

La bonne réponse, depuis Podman 4.4+, ce sont les **Quadlets** : des fichiers de description déclaratifs (`.container`, `.network`, `.volume`, `.pod`) que systemd transforme **automatiquement en services** au démarrage. On obtient le déclaratif qu'on aimait dans Compose, mais **nativement intégré à l'init du système**, en rootless, sans daemon, sans wrapper.

### Exemple réel : mon unit Immich

Je place mes Quadlets utilisateur dans `~/.config/containers/systemd/`. Voici une version simplifiée mais fonctionnelle de mon `immich-server.container` :

```ini
# ~/.config/containers/systemd/immich-server.container
[Unit]
Description=Immich Server
# On attend que la base et redis soient prêts
Requires=immich-postgres.service immich-redis.service
After=immich-postgres.service immich-redis.service

[Container]
Image=ghcr.io/immich-app/immich-server:release
ContainerName=immich-server
AutoUpdate=registry

# --- Posture de sécurité ---
# Pas de nouveaux privilèges, jamais
SecurityLabelType=container_t
NoNewPrivileges=true
# On droppe toutes les capabilities et on ne rajoute que le strict nécessaire
DropCapability=ALL
# Système de fichiers racine en lecture seule
ReadOnly=true
# Tmpfs pour les répertoires qui ont besoin d'écrire
Tmpfs=/tmp:rw,size=512M

# --- Réseau : pas d'exposition directe, on passe par le proxy ---
Network=immich-internal.network

# --- Volumes avec remap d'UID rootless ---
Volume=%h/immich/upload:/usr/src/app/upload:U,Z
Volume=/etc/localtime:/etc/localtime:ro

# --- Secrets via systemd, jamais en clair dans le fichier ---
Environment=DB_HOSTNAME=immich-postgres
Environment=REDIS_HOSTNAME=immich-redis
Secret=immich_db_password,type=env,target=DB_PASSWORD

[Service]
Restart=on-failure
RestartSec=10
# Limites de ressources : un conteneur compromis ne doit pas tuer l'hôte
MemoryMax=4G
CPUQuota=200%

[Install]
WantedBy=default.target
```

Quelques points que je veux souligner, parce que c'est là que se joue la vraie sécurité :

> [!success] Le durcissement, ligne par ligne
> - `NoNewPrivileges=true` : interdit l'escalade via setuid. Devrait être un défaut universel.
> - `DropCapability=ALL` : on enlève **toutes** les capabilities Linux et on ne réajoute que ce dont le service a strictement besoin. La plupart des apps web n'ont besoin de rien.
> - `ReadOnly=true` + `Tmpfs` : le rootfs est immuable. Un attaquant ne peut pas déposer de binaire persistant dans le conteneur.
> - `:Z` sur les volumes : applique le **label SELinux** privé au volume. Combiné à SELinux en mode `enforcing`, c'est une seconde barrière indépendante des namespaces.
> - `MemoryMax` / `CPUQuota` : un conteneur qui part en vrille (crypto-miner injecté, fork bomb) ne fait pas tomber tout le serveur.

Après avoir déposé le fichier, on recharge :

```bash
systemctl --user daemon-reload
systemctl --user start immich-server.service
journalctl --user -u immich-server.service -f
```

### Le détail qui fait toute la différence : `loginctl enable-linger`

> [!bug] L'erreur que tout le monde fait une fois
> En rootless, **les services utilisateur s'arrêtent quand tu te déconnectes** par défaut. Tu reboot le serveur, personne ne se logge en SSH, et… aucun service ne démarre. J'ai cherché une heure la première fois.
> La solution :
> ```bash
> loginctl enable-linger $USER
> ```
> Ça autorise les services de l'utilisateur à tourner **sans session active**, dès le boot. À faire absolument sur un serveur headless.

---

## La défense en profondeur autour des conteneurs

Le conteneur durci n'est qu'une couche. Voici le reste de ma posture, parce que la sécurité n'est jamais un seul outil.

### Segmentation réseau : ne jamais exposer le conteneur applicatif

Aucun de mes services applicatifs n'écoute directement sur une interface publique. Tout passe par **Caddy** (reverse proxy), qui est le seul à publier des ports. Chaque service vit dans un **réseau Podman interne** dédié, et seul le proxy a un pied dans le réseau public et dans le réseau interne.

```bash
podman network create immich-internal --internal
```

Le flag `--internal` interdit tout trafic sortant vers l'extérieur depuis ce réseau. Immich ne peut littéralement pas initier de connexion vers Internet — utile contre l'exfiltration en cas de compromission.

### Exposition minimale : VPN par défaut, Internet par exception

Mon arbitrage est simple et tient en une phrase :

> [!quote] Ma règle d'exposition
> **Un service n'est joignable depuis Internet que si l'usage mobile nomade le justifie réellement.** Tout le reste est derrière WireGuard.

Concrètement, Paperless-ngx (mes documents les plus sensibles) n'est **jamais** exposé : j'y accède uniquement via mon tunnel WireGuard. Immich l'est, parce que je veux l'auto-upload depuis mon téléphone, mais avec authentification forte et derrière le proxy durci.

### Surface logicielle : mises à jour et provenance des images

- **`AutoUpdate=registry`** dans les Quadlets, couplé au timer `podman-auto-update.timer`, applique automatiquement les nouvelles images. Sur des CVE critiques d'apps exposées, le délai de patch est un facteur de risque majeur.
- Je privilégie les **images officielles** (`ghcr.io/immich-app/...`) et je me méfie des images communautaires « tout-en-un » qui empaquettent dix services dans un conteneur root. Une appli = un conteneur = un user namespace.
- **`podman` n'a pas besoin de root pour pull/build/run**, donc même ma chaîne de build d'images custom tourne sans privilège.

### Sauvegardes : la dernière ligne, et la plus importante

Aucune posture de sécurité ne remplace une **sauvegarde testée**. Mes volumes de données et mes dumps PostgreSQL partent chiffrés (restic) vers un stockage hors-site. La règle 3-2-1 n'est pas négociable quand on héberge ses propres données — parce que le risque le plus probable n'est pas l'attaquant, c'est le disque qui meurt ou le `rm -rf` malheureux.

---

## Ce que la migration m'a coûté, honnêtement

Pour que ce retour d'expérience soit utile, voici les **vraies frictions** que j'ai rencontrées :

> [!failure] Les points de douleur réels
> - **Les permissions de volumes (`:U`, `keep-id`)** : de loin le sujet le plus pénible au démarrage. Comprendre le remapping d'UID est un prérequis, pas une option.
> - **Le réseau rootless** : Podman utilise `pasta`/`slirp4netns` en rootless, et certaines subtilités (préservation de l'IP source, ports < 1024) demandent de la config (`net.ipv4.ip_unprivileged_port_start`). C'est documenté, mais ça surprend.
> - **L'écosystème de tutos** : 90 % des guides de self-hosting supposent Docker + Compose en root. Il faut traduire mentalement, surtout sur les permissions.
> - **`podman-compose` est un faux ami** : il marche « presque », ce qui est pire que pas du tout. Le passage aux Quadlets demande un investissement initial réel.

Mais une fois la courbe d'apprentissage passée, le système est **plus simple à raisonner** que mon ancien stack Docker. systemd gère tout, les logs sont dans `journalctl`, le modèle de sécurité est clair et vérifiable. Je dors mieux.

---

## Conclusion : choisir l'outil dont la complexité est méritée

Mon arbitrage tient en trois principes que j'applique désormais systématiquement :

1. **Le blast radius prime sur le confort.** Le rootless de Podman fait que `root` dans le conteneur n'est plus `root` sur l'hôte. C'est la propriété de sécurité fondamentale, et elle est native, pas greffée.
2. **La complexité doit être justifiée par le problème.** Kubernetes résout le scale multi-nœuds. Sur un nœud unique, il ajoute une codebase de manifests et une surface opérationnelle pour une résilience que la topologie rend impossible. systemd + Quadlets fait le travail réel.
3. **Ne jamais fermer les portes.** `podman generate kube` me garde un chemin vers Kubernetes si mon parc grossit. Choisir simple aujourd'hui n'interdit pas de grossir demain.

> [!note] Pour aller plus loin dans le vault
> - [[Durcissement SELinux pour conteneurs]]
> - [[Configuration WireGuard self-hosted]]
> - [[Stratégie de sauvegarde restic 3-2-1]]
> - [[Reverse proxy Caddy avec TLS automatique]]

---

*Retour d'expérience personnel, à adapter à ta propre modélisation de menace. Le bon stack est celui dont tu comprends chaque couche.*
