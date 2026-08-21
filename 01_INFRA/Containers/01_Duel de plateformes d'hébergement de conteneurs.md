---
title: "Docker, Podman ou Kubernetes : le duel des plateformes à conteneurs"
author: "0x41647269656E"
series: Conteneurs
tags:
  - docker
  - podman
  - kubernetes
  - quadlets
  - k3s
  - watchtower
  - auto-update
  - homelab
date: 11-08-2026
reading-time: 30min
difficulty: tech-enthusiast
---
# Docker, Podman ou Kubernetes : le duel des plateformes à conteneurs

> [!info] Objectif
> Choisir la plateforme qui fera tourner les conteneurs du homelab. Trois prétendants, dans leurs dernières versions à l'heure où j'écris ces lignes (août 2026) :
> - **Docker Engine 29**
> - **Podman 6** (et ses quadlets)
> - **Kubernetes 1.36** (incarné par k3s)
>
> Et trois critères de choix, volontairement terre-à-terre :
> 1. **L'effort technique** nécessaire pour appréhender la technologie ;
> 2. **Le temps de maintenance** récurrent, une fois la plateforme en production ;
> 3. **Les modalités de mise à jour des applications** : un simple pull ? Un mécanisme intégré ? Une appli tierce qui gère ça ?

---

## Introduction

Dans les articles précédents, nous avons vu [[00_Les avantages de la conteneurisation|pourquoi la conteneurisation s'impose pour l'auto-hébergement]] et [[03_Sécurite Auto Hébergement Podman|comment sécuriser ses conteneurs]]. Reste une question que tout le monde finit par se poser, généralement après avoir copié-collé son premier `docker run` : *sur quelle plateforme est-ce que je construis tout ça ?*

Ce choix est structurant. C'est lui qui déterminera :

- le format de vos fichiers de configuration (compose, quadlets, manifestes YAML) ;
- votre routine de maintenance du dimanche ;
- la façon dont vos 5 (puis 15, puis 30...) services se mettront à jour ;
- et la facilité avec laquelle vous pourrez tout reconstruire le jour où ça casse.

Contrairement au choix d'un dashboard ou d'un lecteur multimédia, revenir en arrière coûte cher : migrer 30 services d'un écosystème à un autre représente des jours de travail. Autant choisir en connaissance de cause.

> [!note] Un duel déséquilibré, et c'est assumé
> Comparer Docker, Podman et Kubernetes, c'est comparer un moteur de conteneurs, un moteur intégré à systemd, et un orchestrateur de clusters. Sur le papier, ce n'est pas la même catégorie. Mais dans la pratique du homelab, la question est bien celle-là : *"sur quoi je fais tourner mes services ?"* — et ces trois-là sont les réponses que vous croiserez partout. Le duel est donc pertinent, à condition de garder en tête que Kubernetes joue dans une catégorie de poids supérieure... avec les frais d'entrée qui vont avec (complexité, temps, maintenance...).

---

## Les prétendants : état des lieux (août 2026)

### Docker Engine 29 — le standard de fait

Docker n'est plus à présenter : c'est lui qui a démocratisé les conteneurs en 2013 et imposé le format d'image devenu standard OCI. Son architecture repose sur un daemon central (`dockerd`) qui s'appuie sur `containerd` et `runc` pour exécuter les conteneurs. Le client `docker` et le plugin Compose v2 (`docker compose`) complètent l'ensemble.

La branche majeure actuelle est la **v29**, sortie en novembre 2025 (29.7 à l'été 2026). C'est une version de fond qui modernise la tuyauterie :

- le **containerd image store** devient le backend par défaut pour les nouvelles installations (les installations existantes conservent leur graph driver historique, la migration reste opt-in) ;
- l'API minimale supportée passe à la version 1.44 : les clients plus anciens que Docker 25 sont désormais refusés ;
- un backend pare-feu **nftables** fait son apparition en expérimental, appelé à remplacer iptables à terme.

Côté cycle de vie, seules les **deux dernières branches** reçoivent des correctifs : il faut donc suivre environ une montée de version majeure par an.

### Podman 6 — le moteur sans daemon

Podman, porté par Red Hat, est le concurrent direct de Docker : même format d'images OCI, CLI quasi identique, mais une architecture radicalement différente. Pas de daemon central : chaque conteneur est un processus enfant lancé directement (fork/exec), ce qui s'intègre naturellement au **mode rootless** et à **systemd**. J'en ai déjà détaillé les bénéfices sécurité dans [[03_Sécurite Auto Hébergement Podman|l'article dédié]].

La branche actuelle est **Podman 6.0**, sortie fin juin 2026. Au menu :

- une réécriture de la gestion des fichiers de configuration ;
- des quadlets retravaillés : organisation en sous-répertoires, gestion facilitée via la commande `podman quadlet`, nouvelle clé `ImageVolume=` ;
- plusieurs adresses IP statiques par conteneur et une isolation réseau améliorée pour la compatibilité Docker.

> [!warning] Montée majeure = notes de version
> Podman 6.0 assume de nombreux *breaking changes* (c'est le prix de la réécriture de la configuration). Si vous êtes en 5.x, lisez les notes de version avant de monter. Et si vous êtes sur une distribution stable type Debian, vous ne verrez probablement pas Podman 6 avant la prochaine version de l'OS — les moteurs de conteneurs suivent le rythme des dépôts de votre distribution, c'est un point à intégrer dans votre choix d'OS serveur.

#### Le cas quadlets : pourquoi ils changent le duel

Les **quadlets** méritent qu'on s'y arrête, car c'est eux qui font passer Podman du statut de "Docker sans daemon" à celui de véritable plateforme d'hébergement.

Rappel rapide du fonctionnement : un quadlet est un **fichier texte déclaratif au format des unités systemd** (syntaxe INI, sections `[Unit]`, `[Container]`, `[Install]`...) décrivant un objet Podman, un conteneur, mais aussi un pod, un volume ou un réseau. Projet indépendant à l'origine (créé en 2021 par Alexander Larsson, de Red Hat), la fonctionnalité est intégrée nativement à Podman depuis la **version 4.4 (février 2023)**, où elle remplace l'ancien `podman generate systemd`, aujourd'hui déprécié : rien à installer en plus, c'est systemd lui-même qui lit ces fichiers et pilote Podman. C'est une exclusivité Podman, Docker n'a aucun équivalent natif, l'intégration systemd y reste artisanale (unités écrites à la main autour de `docker run`).

Le principe en pratique : au lieu d'un `docker-compose.yml` interprété par un outil tiers, vous décrivez chaque conteneur dans un petit fichier `.container` déposé dans `~/.config/containers/systemd/` (rootless) ou `/etc/containers/systemd/` (rootful). Au `daemon-reload`, un générateur systemd transforme ce fichier en **service systemd natif** :

```ini
# ~/.config/containers/systemd/jellyfin.container
[Unit]
Description=Jellyfin Media Server

[Container]
Image=docker.io/jellyfin/jellyfin:latest
AutoUpdate=registry
PublishPort=8096:8096
Volume=%h/containers/config/jellyfin:/config:Z
Volume=%h/medias:/media:ro,Z

[Service]
Restart=always

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user start jellyfin.service
journalctl --user -u jellyfin.service   # les logs sont dans journald
```

Votre conteneur devient un citoyen systemd de première classe : politique de redémarrage, ordre de démarrage, dépendances entre services (`After=`, `Requires=`), logs centralisés dans journald, démarrage au boot. Tout l'outillage que vous connaissez déjà sur un serveur Linux s'applique.

Il existe des unités `.pod` (regrouper des conteneurs), `.volume`, `.network`, `.image`, `.build`... et surtout `.kube`, qui permet d'exécuter un manifeste **au format Kubernetes** via `podman kube play`, sans cluster. Une passerelle intéressante : vous pouvez écrire du YAML Kubernetes aujourd'hui sous Podman, et le rejouer dans un vrai cluster demain.

### Kubernetes 1.36 — l'orchestrateur (incarné par k3s)

Kubernetes ne fait pas que lancer des conteneurs : il **orchestre**. Vous déclarez un état cible ("je veux n réplique d'un serveur apache, exposée en HTTPS, avec ce volume") et le cluster tente en permanence de converger vers cet état : redémarrage automatique, répartition sur plusieurs nœuds, rolling updates, gestion du réseau et des certificats via son écosystème.

Dans un homelab, personne n'installe Kubernetes "vanilla". La distribution de référence est **k3s** : un binaire unique qui embarque tout (runtime containerd, réseau, ingress Traefik, load-balancer), s'installe en une minute et se contente d'environ 500 Mo de RAM à vide en mono-nœud, avec une base SQLite à la place d'etcd. La version **k3s 1.36** (juillet 2026) suit Kubernetes 1.36 et passe sur etcd 3.6 pour les configurations en haute disponibilité.

Côté cycle de vie upstream : **trois versions mineures par an**, chacune supportée environ **14 mois**, et seules les trois dernières mineures reçoivent des correctifs. Kubernetes 1.37 est d'ailleurs attendu pour le 26 août 2026. Retenez ce rythme, il pèsera lourd dans le round 2.

---

## Round 1 — L'effort d'apprentissage

### Docker : la voie royale, pavée de docker-compose.yml

Le vocabulaire à acquérir pour démarrer tient sur un post-it : image, conteneur, volume, réseau, et le fichier compose qui assemble le tout. La documentation est pléthorique, chaque problème a déjà sa réponse sur un forum, et surtout : **l'intégralité de l'écosystème self-hosted est documentée pour Docker**. Le README de chaque projet fournit un `docker-compose.yml` prêt à copier. Jellyfin, Immich, Paperless-ngx, Home Assistant : installation en cinq minutes, littéralement.

Les vraies difficultés (comprendre les volumes et les permissions, les réseaux entre conteneurs, le reverse proxy) existent, mais la courbe est douce et progressive.

### Podman : un Docker-like sur Linux

La transition depuis Docker est volontairement indolore : le CLI est quasi identique (`alias docker=podman` fonctionne pour l'essentiel) et Podman expose un socket compatible avec l'API Docker — `docker compose` peut donc piloter Podman via `DOCKER_HOST`, et les compose files des README restent utilisables.

Mais s'arrêter là serait passer à côté de l'outil. Le "vrai" Podman se pratique avec les quadlets, et cela suppose d'être à l'aise avec **systemd** : unités, `systemctl`, `journalctl`, `daemon-reload`. Il faut aussi apprivoiser les subtilités du rootless : mapping des UID/GID (`podman unshare`), `loginctl enable-linger` pour que vos services survivent à la déconnexion, ports privilégiés (voir [[03_Sécurite Auto Hébergement Podman|l'article sécurité]]), étiquettes SELinux `:Z` selon la distribution.

L'effort est réel mais borné, et c'est mon argument préféré, **tout ce que vous apprenez est du Linux réutilisable**. Rien n'est propriétaire à Podman : vous montez en compétence sur systemd, les namespaces et journald, pas sur un outil.

Petit bémol : la communauté est plus petite que celle de Docker. Quand un conteneur se comporte bizarrement en rootless, la réponse se trouve plus souvent dans une issue GitHub que dans un tutoriel clé en main. Il faudra parfois "traduire" un compose file en quadlet : des outils comme `podlet` automatisent une partie du travail.

### Kubernetes : un changement de paradigme

Ici, on ne parle plus d'apprendre un outil mais un **modèle**. Avant de lancer votre premier service, il faudra assimiler : Pod, Deployment, ReplicaSet, Service, Ingress, ConfigMap, Secret, PersistentVolumeClaim, Namespace, Opérateurs et CRD... puis le cli `kubectl`, puis Helmchart pour installer des charts packagés, Argo pour les déployer et l'ensemble de l'écosystème à connaitre et garder en vue. Chaque concept est logique, mais ils sont nombreux et interdépendants : la marche d'entrée est un mur à gravir, comme le passage d'une architecture on-premise à des services serverless dans le cloud.

k3s réduit drastiquement l'effort d'**installation** une commande, un cluster fonctionnel mais ne réduit en rien l'effort d'assimilation technique des **concepts** qui entourent Kubernetes. L'écosystème self-hosted n'est pas écrit pour Kubernetes : le `docker-compose.yml` du README devra être traduit en manifestes ou remplacé par un chart Helm communautaire qu'il faudra évaluer et maintenir.

Comptez des heures pour Docker, des jours pour Podman/quadlets... et des semaines pour être simplement à l'aise sur Kubernetes.

> [!example] Verdict du round 1
> | Plateforme | Effort d'apprentissage | Concepts et charge |
> |---|---|---|
> | Docker 29 | 🟢 Faible | Volumes, réseaux, reverse proxy |
> | Podman 6 | 🟡 Modéré | systemd, subtilités rootless |
> | Kubernetes 1.36 | 🔴 Élevé | Paradigme complet + traduire tout l'écosystème compose |
>
> Docker remporte le round. Podman suit de près pour un utilisateur déjà à l'aise sous Linux ; Kubernetes est hors catégorie et nécessite des compétences très spécifiques.

---

## Round 2 — Le temps de maintenance

La plateforme installée, combien d'heures par mois pour la garder saine ? C'est le critère que l'on sous-estime systématiquement au moment du choix... et celui qui décide si votre homelab est encore vivant dans deux ans.

### Docker : un daemon à entretenir, sans plus

La maintenance se résume à suivre les mises à jour du paquet via votre gestionnaire habituel. Deux points d'attention :

- la mise à jour du daemon **redémarre les conteneurs**, sauf à activer `"live-restore": true` dans `/etc/docker/daemon.json`, réglage recommandé pour relancer les services en cas de besoin ;
- une montée de branche majeure par an environ (seules deux branches sont maintenues), avec parfois un chantier de fond, les installations historiques devront un jour planifier la migration vers le containerd image store.

Le daemon tourne en root et constitue un point de défaillance unique, mais soyons honnêtes : il est d'une fiabilité remarquable. La charge de maintenance réelle est faible. Seul la sécurité est un vrai 

### Podman : le moteur qui se fait oublier

Pas de daemon : Podman n'est qu'un binaire invoqué à la demande, et vos conteneurs sont des services systemd indépendants les uns des autres. Mettre à jour Podman = mettre à jour un paquet, sans interrompre quoi que ce soit qui tourne. La supervision (redémarrages, boot, logs) est déléguée à systemd, qui fait ce travail sur vos serveurs depuis des années.

Les points de vigilance sont ailleurs :

- les **montées majeures** (la 6.0 et sa réécriture de configuration en est l'exemple parfait) méritent une lecture attentive des notes de version, sur le même principe que Docker ;
- votre version de Podman dépend de votre distribution : sur une Debian stable, vous vivrez avec une version figée pendant deux ans. Pour bénéficier des évolutions rapides des quadlets, une Fedora ou un dépôt à jour est préférable.

Une fois en place, c'est la plateforme qui se fait le plus oublier : les quadlets tournent, systemd veille, et le dimanche après-midi m'appartient.

### Kubernetes : le cluster est un service à part entière

C'est ici que Kubernetes présente la facture. Le rythme upstream ( trois mineures par an, 14 mois de support chacune ) impose de monter de version **une à deux fois par an minimum**, sous peine de sortir du support sécurité. Pour situer : la 1.34, sortie il y a un an, ne recevra plus de correctifs après octobre 2026.

Une montée de version cluster, même facilitée par k3s (relancer le script d'installation, ou automatiser avec le system-upgrade-controller), reste un événement : lire les notes de version, vérifier les API dépréciées utilisées par vos manifestes et vos charts Helm, monter les nœuds dans l'ordre, contrôler que tout redémarre. k3s adoucit certains angles (les certificats internes, par exemple, sont automatiquement renouvelés au redémarrage du service lorsqu'ils approchent de l'expiration) mais il ne supprime pas l'exercice.

S'ajoute la maintenance de tout ce que le cluster héberge *en plus* de vos applications : l'ingress, les CRDs, les charts Helm qui évoluent, le monitoring (quasi indispensable pour savoir ce qui se passe dans la boîte noire). Le cluster n'est pas un socle inerte : c'est un service de plus à opérer, le plus exigeant de tous.

> [!example] Verdict du round 2
> | Plateforme | Charge récurrente | Nature de la charge |
> |---|---|---|
> | Docker 29 | 🟢 Faible | MàJ du paquet, un chantier de fond occasionnel |
> | Podman 6 | 🟢 Très faible | MàJ du paquet, vigilance sur les majeures |
> | Kubernetes 1.36 | 🔴 Élevée et **non négociable** | 1 à 2 upgrades cluster/an + charts + composants |
>
> Podman l'emporte d'une courte tête sur Docker (pas de daemon, pas d'interruption). Kubernetes transforme la maintenance en abonnement : c'est le prix de l'orchestration, à ajouter au prix déjà élevé de la maintenance du reste de l'infrastructure.

---

## Round 3 — La mise à jour des applications

Le critère décisif à mes yeux. Les 30 services publient chacun une nouvelle version toutes les quelques semaines : si la mise à jour n'est pas simple, idéalement automatique et réversible (\*), elle ne sera pas faite, et un service pas à jour exposé sur internet est une brèche en devenir.

### Docker : un simple pull... et une appli tierce pour automatiser

Le geste manuel est d'une simplicité biblique :

```bash
docker compose pull && docker compose up -d
```

Compose ne recrée que les conteneurs dont l'image a changé, les volumes de données ne sont pas touchés. C'est LE réflexe du self-hosting depuis dix ans.

Mais le moteur n'offre **aucun mécanisme natif** pour surveiller les registres et déclencher ces mises à jour : il faut une application tierce. Et sur ce terrain, l'actualité mérite un avertissement :

> [!warning] Watchtower est mort, vive Watchtower
> Le Watchtower historique (`containrrr/watchtower`), installé sur la moitié des homelabs de la planète, a été **archivé le 17 décembre 2025** : les mainteneurs du projet ont jeté l'éponge. Le projet a été repris par un fork actif, [nicholas-fedor/watchtower](https://github.com/nicholas-fedor/watchtower), compatible à l'identique : mêmes variables d'environnement, il suffit de remplacer l'image par `nickfedor/watchtower`. Si vos compose files pointent encore vers l'original, c'est le moment de corriger (voir la fiche [[Watchtower]]).


> [!info] Un changement de mainteneur, un phénomène courant
> Dans les projets portés par des passionnés, il est fréquent qu'un logiciel passe de main en main au fil des abandons et des reprises. Paperless en est l'illustration parfaite : délaissé par son auteur d'origine, repris sous le nom de paperless-ng... lui-même abandonné avant de renaître en [[Paperless-ngx]], aujourd'hui maintenu collectivement. Ce n'est pas un drame, un fork bien repris vaut mieux qu'un original à l'arrêt, mais c'est une réalité de l'auto-hébergement : il faut suivre la vie de ses outils, pas seulement leurs versions.

L'écosystème offre plusieurs philosophies :

- **Watchtower (fork)** : met à jour automatiquement, planification cron, notifications. Le pilote automatique.
- **What's up Docker (WUD)** ou **Diun** : surveillent les registres et **notifient**, à vous d'appuyer sur le bouton. Le pilote semi-automatique, plus prudent.
- **Dockge, Komodo, Portainer** : interfaces web de gestion de stacks avec bouton de mise à jour intégré (attention aux fonctionnalités premium payantes 😜).

Deux réserves. D'une part, ces outils exigent l'accès au socket Docker — un conteneur tiers avec les clés du royaume, j'en parle en détail dans la section [[03_Sécurite Auto Hébergement Podman#Le socket Docker : les clés du royaume|« Le socket Docker : les clés du royaume » de l'article sécurité]]. D'autre part, en cas de mise à jour qui casse, le retour arrière est manuel (re-tagger l'ancienne image et recréer le conteneur).

(\*) : _Attention, toutes les mises à jour applicatives ne se limitent pas à monter une nouvelle image sur le même volume de données, certaines versions majeures applicatives introduisent des braking-changes qui affectent les données sur disque (conversions de modèles de données de configuration, ...). Parfois, le retour arrière n'est pas possible. On conservera le principe général de faire un backup (ou un snapshot) avant de mettre à jour._
### Podman : l'auto-update est dans la boîte

C'est ici que les quadlets abattent leur carte maîtresse. Reprenez le fichier `jellyfin.container` plus haut : la ligne `AutoUpdate=registry` suffit à inscrire le conteneur au mécanisme de mise à jour **natif** de Podman. Il ne reste qu'à activer le timer systemd fourni :

```bash
systemctl --user enable --now podman-auto-update.timer   # chaque nuit par défaut
podman auto-update --dry-run                  # voir ce qui serait mis à jour
podman auto-update                            # déclencher manuellement
```

À chaque exécution, Podman compare le digest local à celui du registre, tire les nouvelles images et redémarre les unités systemd concernées. Cerise sur le gâteau : si le service échoue à redémarrer après la mise à jour, Podman effectue un **rollback automatique** vers l'image précédente. Couplez-le à un healthcheck (`HealthCmd=` + `Notify=healthy` dans le quadlet) et la détection d'échec devient réellement fiable : le conteneur n'est déclaré "à jour" que s'il est fonctionnellement sain.

Le tout **sans aucune application tierce, sans daemon, sans socket exposé** : le timer systemd et le moteur suffisent. Sobriété maximale pour un résultat que Docker n'atteint qu'en empilant des outils.

### Kubernetes : de kubectl à l'usine GitOps

Kubernetes ne connaît pas le "pull" : on déclare une nouvelle version, le cluster converge. Le geste basique :

```bash
kubectl set image deployment/jellyfin jellyfin=jellyfin/jellyfin:10.11.0
kubectl rollout status deployment/jellyfin
kubectl rollout undo deployment/jellyfin    # retour arrière en une commande
```

Et c'est objectivement le **moteur de mise à jour le plus puissant des trois** : rolling update sans coupure (l'ancien pod ne s'éteint que lorsque le nouveau est prêt), historique des révisions, rollback en une commande, le tout natif.

Pour l'automatisation, l'état de l'art est le **GitOps** : vos manifestes vivent dans un dépôt Git, un opérateur dans le cluster (**Flux** ou **ArgoCD**) applique en continu ce que Git décrit, et **Renovate** ouvre des pull requests dès qu'une image publie une nouvelle version. Vous mergez, le cluster se met à jour. Chaque montée de version est tracée, revue, réversible par un simple revert, c'est "techniquement somptueux".

Mais mesurons le chemin : là où Podman demande une ligne dans un fichier et un timer, il aura fallu monter un dépôt Git, un opérateur GitOps, Renovate et comprendre l'articulation de l'ensemble. Des alternatives plus légères existent (Keel joue les Watchtower du cluster, Renovate sait d'ailleurs aussi mettre à jour de simples compose files), mais on ne fera pas semblant : c'est une usine logicielle fournie en kit à construire soi-même. Avis aux amateurs de modélisme.

> [!tip] Quelle que soit la plateforme : épinglez vos tags
> Laisser `:latest` sur une base de données, c'est accepter une migration majeure surprise un mardi à 4h du matin. Épinglez au moins la version majeure (`postgres:16`, `jellyfin:10.11`) : les correctifs passent automatiquement, les sauts majeurs attendent votre décision. Les volumes préservent vos données, pas la compatibilité de leurs schémas.

> [!example] Verdict du round 3
> | Plateforme | Mécanisme | Automatisation | Rollback |
> |---|---|---|---|
> | Docker 29 | `compose pull` + `up -d` | Appli tierce (fork Watchtower, WUD...) | Manuel |
> | Podman 6 | `podman auto-update` **natif** | Timer systemd intégré | **Automatique** si échec |
> | Kubernetes 1.36 | Convergence déclarative | GitOps à construire (Flux/ArgoCD + Renovate) | `rollout undo`, natif |
>
> Podman gagne le rapport puissance/simplicité ; Kubernetes gagne la puissance absolue ; Docker se repose entièrement sur son écosystème : vivant, mais mouvant, l'épisode Watchtower l'a rappelé.

---

## La synthèse

| Critère                              | Docker 29                      | Podman 6 (quadlets)             | Kubernetes 1.36 (k3s)                |
| ------------------------------------ | ------------------------------ | ------------------------------- | ------------------------------------ |
| Effort d'apprentissage               | 🟢 Faible                      | 🟡 Modéré                       | 🔴 Élevé                             |
| Temps de maintenance                 | 🟢 Faible                      | 🟢 Très faible                  | 🔴 Élevé, récurrent                  |
| Mise à jour des applis               | 🟡 Simple mais outillage tiers | 🟢 Native + rollback            | 🟡 Puissante mais à construire       |
| Compatibilité écosystème self-hosted | 🟢 Totale                      | 🟢 Très bonne (compat Docker)   | 🔴 Traduction requise                |
| Atout maître                         | La documentation universelle   | La sobriété : systemd fait tout | Rolling updates, GitOps, multi-nœuds |

Trois profils, trois vainqueurs :

- **"Je débute, je veux mes services ce week-end"** → **Docker**. La marche d'entrée la plus basse, l'écosystème le plus documenté. Ajoutez le fork de Watchtower ou WUD pour les mises à jour, `live-restore` dans le daemon, et vous êtes bien équipé.
- **"Je veux un socle sobre et sécurisé, que je maintiendrai dix ans"** → **Podman + quadlets**. Un effort d'apprentissage initial modéré (surtout si systemd vous est familier), remboursé chaque semaine : rootless par défaut, pas de daemon, et des mises à jour automatiques avec rollback sans le moindre composant tiers.
- **"Je veux apprendre l'orchestration, pour le lab ou la carrière"** → **k3s**. Aucune des deux autres plateformes ne vous apprendra Kubernetes, et Kubernetes reste la compétence infra la plus demandée du marché. Mais choisissez-le pour apprendre, pas pour héberger trois services : la maintenance du cluster est un abonnement sans résiliation.

## Mon choix

Vous connaissez la conclusion si vous suivez ce vault : **Podman et ses quadlets** portent mon socle de services. La cohérence avec [[03_Sécurite Auto Hébergement Podman|ma démarche sécurité]] (rootless, pas de daemon, pas de socket à exposer à un outil de mise à jour), la sobriété d'un moteur qui se fait oublier, et `podman auto-update` qui fait le travail chaque nuit avec filet de sécurité : pour un homelab pensé comme une infrastructure durable, c'est l'équilibre qui me convient pour **mon** cas d'usage. Chacun doit se faire son avis.

A titre personnel, je ne renonce pas à Kubernetes pour autant : un [[k3s]] mono-nœud vit dans mon lab, précisément parce qu'un homelab sert \*aussi\* à ça : se confronter aux outils qu'on croise en entreprise, se batir des expériences, développer ses connaissances et faire de la veille technique. Helm et ArgoCD sont au programme des prochains articles de ma todo, et les unités `.kube` des quadlets ménagent une passerelle élégante entre les deux mondes : le YAML écrit pour Podman aujourd'hui pourra rejoindre le cluster demain.

> [!quote] Conseil final
> Ne choisissez pas la plateforme la plus impressionnante : choisissez celle que vous aurez encore plaisir à maintenir dans deux ans sur votre temps personnel, après votre journée de travail. Un `docker compose pull` exécuté régulièrement vaudra toujours mieux qu'un cluster Kubernetes à l'abandon.

---

## Sources et liens utiles

- [Docker Engine v29 — annonce officielle](https://www.docker.com/blog/docker-engine-version-29/)
- [Releases Podman](https://github.com/containers/podman/releases) et [documentation des quadlets](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)
- [podman-auto-update](https://docs.podman.io/en/latest/markdown/podman-auto-update.1.html)
- [Cycle de release Kubernetes](https://kubernetes.io/releases/release/) et [endoflife.date/kubernetes](https://endoflife.date/kubernetes)
- [k3s 1.36 — notes de version](https://docs.k3s.io/release-notes/v1.36.X)
- [Archivage de containrrr/watchtower](https://github.com/containrrr/watchtower) et [fork actif nicholas-fedor/watchtower](https://github.com/nicholas-fedor/watchtower)
- [Renovate](https://docs.renovatebot.com/), [Flux](https://fluxcd.io/), [ArgoCD](https://argo-cd.readthedocs.io/), [What's up Docker](https://getwud.github.io/wud/)
