# Sécurité et Auto-Hébergement avec Podman

## Introduction

L’auto-hébergement séduit de plus en plus : reprendre le contrôle de ses données, réduire sa dépendance aux services centralisés, apprendre en construisant sa propre infrastructure, ou simplement héberger ses outils préférés chez soi ou sur un VPS. Gestionnaire de mots de passe, cloud personnel, monitoring, blog, VPN, domotique… les usages sont nombreux.

Mais auto-héberger ne consiste pas seulement à « faire tourner un service ». Dès qu’une application est exposée au réseau, la question de la sécurité devient centrale : mises à jour, isolation, permissions, sauvegardes, ports ouverts, accès administrateur, secrets, journalisation…

C’est précisément là que les conteneurs apportent une réponse moderne, simple et pragmatique. Et parmi eux, **Podman** se distingue particulièrement pour celles et ceux qui veulent allier simplicité, sécurité et sobrièveté technique.

Dans cet article, nous allons voir pourquoi Podman est un excellent choix pour l’auto-hébergement sécurisé, comment gérer les contraintes réseau comme les ports privilégiés, comment exécuter des images demandant des privilèges élevés, et quelles bonnes pratiques adopter pour héberger sereinement vos services.

---

## Présentation de l’auto-hébergement et de la sécurité

L’auto-hébergement consiste à installer et administrer soi-même des services numériques sur une machine que l’on contrôle : mini-PC à la maison, NAS, serveur dédié, VPS ou machine virtuelle.

Les avantages sont réels : on garde la maîtrise de ses données et la liberté de configurer ses outils comme on l’entend, souvent pour moins cher qu’un empilement d’abonnements, tout en s’affranchissant des plateformes et de leurs conditions changeantes. Et chaque service installé est une occasion de plus de monter en compétence.

Mais tout ce que ces plateformes faisaient à votre place vous revient : c’est vous qui maintenez le système à jour, qui sécurisez les accès, qui gardez un œil sur les journaux, qui décidez de ce qui est exposé au réseau et qui sauvegardez les données. Et le jour où quelque chose casse, c’est encore vous qui gérez l’incident, généralement un dimanche soir.

Un service mal configuré peut exposer des fichiers sensibles, permettre des intrusions ou servir de point d’entrée sur tout le serveur.

L’objectif n’est donc pas d’atteindre une sécurité parfaite, illusoire, mais de **réduire les risques** par couches successives : isolation, moindre privilège, segmentation réseau, mises à jour et supervision.

---

## Pourquoi choisir des conteneurs

Les conteneurs sont devenus un standard car ils répondent à plusieurs problèmes historiques de l’auto-hébergement.

### Isolation applicative

Installer plusieurs services directement sur un système hôte finit souvent en chaos :

- versions incompatibles ;
- dépendances conflictuelles ;
- fichiers dispersés ;
- configuration difficile à reproduire.

Les conteneurs encapsulent l’application et ses dépendances dans une image exécutable de manière isolée.

### Déploiement reproductible

Un simple fichier de configuration permet de reconstruire un service à l’identique sur une autre machine.

### Maintenance simplifiée

Mettre à jour un service revient souvent à :

1. tirer une nouvelle image ;
2. redémarrer le conteneur ;
3. conserver les volumes de données.

### Sécurité améliorée

Un conteneur n’est pas une machine virtuelle, mais il limite l’impact d’un incident lorsqu’il est bien configuré :

- espaces de noms isolés ;
- permissions restreintes ;
- systèmes de fichiers en lecture seule ;
- capabilities Linux réduites ;
- utilisateur non privilégié.

### Sobriété

Les conteneurs consomment généralement moins de ressources que plusieurs VM complètes.

---

## Pourquoi Podman ?

Podman est un moteur de conteneurs compatible avec l’écosystème OCI (Open Container Initiative). Il permet de gérer images, conteneurs, volumes et pods, tout en restant proche de l’expérience Docker côté commandes.

Exemple :

```bash
podman run -d --name nginx -p 8080:80 nginx
```

### Ce qui distingue Podman

- démon central non nécessaire ;
- fonctionnement rootless natif ;
- meilleure intégration Linux ;
- pods natifs ;
- génération d’unités systemd ;
- approche sécurité très solide.

### Un moteur sans daemon

Contrairement à certaines solutions reposant sur un service central tournant en permanence avec privilèges élevés, Podman lance les processus directement.

Cela réduit :

- la surface d’attaque ;
- la complexité ;
- les dépendances système.

### Le socket Docker : les clés du royaume

Un aparté s’impose ici, car il éclaire tout l’intérêt du modèle sans daemon.

Dans le monde Docker, chaque commande passe par l’API du daemon, exposée via le socket Unix `/var/run/docker.sock`. Or ce daemon tourne en root : quiconque peut écrire sur ce socket peut lui demander de lancer n’importe quel conteneur, avec n’importe quelles options.

```bash
docker run -v /:/host --privileged -it alpine chroot /host
```

Une commande de ce type, envoyée via le socket, monte l’intégralité du système de fichiers de l’hôte dans un conteneur contrôlé par l’attaquant. L’accès au socket Docker **est** un accès root — d’où son surnom : « les clés du royaume ». Corollaire souvent ignoré : ajouter un utilisateur au groupe `docker`, c’est lui donner un équivalent de sudo sans mot de passe. Docker le documente d’ailleurs explicitement.

Le piège, c’est que tout un écosystème d’outils très populaires réclame précisément ce socket pour fonctionner :

- Watchtower, pour les mises à jour automatiques ;
- Portainer, Dockge et les autres interfaces de gestion ;
- Traefik, pour l’auto-découverte des conteneurs ;
- divers agents de supervision.

On se retrouve donc à confier les clés du royaume à des conteneurs tiers — parfois exposés au réseau — en pariant sur la qualité de leur code et l’intégrité de leur chaîne de distribution.

Si vous restez sous Docker, des parades existent :

1. intercaler un **socket-proxy** (par exemple `docker-socket-proxy`), qui ne laisse passer que les appels d’API strictement nécessaires à l’outil ;
2. réserver ces outils à un réseau interne, jamais exposé ;
3. monter le socket en lecture seule (`:ro`)... protection largement illusoire : cela empêche de remplacer le fichier, pas d’écrire *dans* le socket — l’API reste pleinement utilisable.

Et sous Podman ? Le problème disparaît en grande partie par construction : pas de daemon, donc pas de socket root central obligatoire. Podman sait exposer un socket compatible avec l’API Docker (`podman.socket`) pour les outils qui en dépendent, mais il est optionnel et, en rootless, il porte les droits d’un utilisateur ordinaire — pas ceux de root. Quant au besoin le plus courant, les mises à jour automatiques, Podman le couvre nativement via systemd, sans socket ni conteneur tiers : voir [[Duel_Docker_Podman_Kubernetes|le comparatif des plateformes]].

### Le mode rootless

C’est probablement l’argument majeur.

Avec Podman, un utilisateur standard peut lancer ses propres conteneurs sans être root. Cela signifie qu’une compromission du service conteneurisé n’accorde pas automatiquement des privilèges administrateur sur la machine.

En pratique :

```bash
podman ps
podman run -d -p 8080:80 nginx
```

peuvent être exécutés depuis votre compte utilisateur.

### Comparaison avec des solutions plus lourdes

Pour un homelab ou un petit serveur, déployer Kubernetes, OpenShift ou une orchestration complexe est souvent disproportionné.

Vous ajoutez :

- complexité opérationnelle ;
- courbe d’apprentissage élevée ;
- consommation mémoire ;
- nombreuses briques à maintenir.

Podman convient parfaitement lorsque vous avez :

- quelques services ;
- un ou plusieurs hôtes simples ;
- besoin de stabilité ;
- préférence pour la lisibilité.

---

## Gestion des Ports

Lorsqu’on auto-héberge, il faut exposer des services :

- HTTP : 80
- HTTPS : 443
- SSH : 22
- DNS : 53

### Le problème des ports < 1024

Sous Linux, les ports inférieurs à 1024 sont dits **privilégiés**. Historiquement, seuls les processus disposant de privilèges élevés peuvent s’y attacher directement.

En mode rootless, cela pose un problème :

```bash
podman run -p 80:80 nginx
```

peut échouer selon la configuration du système.

### Méthodes de redirection des ports

Heureusement, plusieurs approches existent.

### 1. Reverse proxy sur ports élevés

Exposez vos services internes sur 8080, 8443, etc., puis utilisez un reverse proxy frontal.

Exemple :

- Podman nginx interne : 8080
- Reverse proxy système : 80/443

### 2. Redirection via pare-feu

Avec nftables ou iptables, redirigez 80 vers 8080 et 443 vers 8443.

Principe :

```text
80 -> 8080
443 -> 8443
```

Le conteneur reste non privilégié.

### 3. Modifier la limite système

Le noyau Linux permet d’abaisser la limite des ports non privilégiés :

```bash
sysctl net.ipv4.ip_unprivileged_port_start=80
```

À évaluer selon votre politique de sécurité.

### 4. Socket activation avec systemd

Systemd peut écouter sur 80/443 puis transmettre au service lancé ensuite.

Très élégant pour certains usages.

### Quelle méthode choisir ?

Pour la majorité des auto-hébergeurs :

- reverse proxy + ports élevés ;
- ou redirection nftables ;

sont les solutions les plus propres.

---

## Conteneurs et Droits

Certaines images supposent un fonctionnement root. C’est fréquent avec :

- anciennes images ;
- logiciels écrivant partout ;
- serveurs modifiant des fichiers système ;
- processus nécessitant ports privilégiés ;
- permissions mal pensées.

### Comment les gérer

### 1. Chercher une image rootless-friendly

De nombreux projets fournissent désormais :

- utilisateur applicatif dédié ;
- UID/GID configurables ;
- volumes propres ;
- documentation Podman.

C’est toujours la meilleure option.

### 2. Mapper les UID/GID

Podman rootless utilise des user namespaces. On peut adapter les permissions des volumes.

Exemple :

```bash
podman unshare chown -R 1000:1000 data/
```

### 3. Utiliser `--user`

Forcer l’utilisateur du processus :

```bash
podman run --user 1000:1000 ...
```

### 4. Ajouter uniquement les capabilities nécessaires

Évitez `--privileged`.

Préférez :

```bash
--cap-add NET_BIND_SERVICE
```

si seul l’accès aux ports bas est requis.

### 5. Dernier recours : conteneur rootful dédié

Si une image impose réellement root :

- exécutez-la séparément ;
- limitez ses volumes ;
- segmentez le réseau ;
- surveillez-la davantage.

---

## Bonnes pratiques de sécurité

Voici les réflexes essentiels.

### Utiliser le moindre privilège

- utilisateur non root ;
- capabilities minimales ;
- pas de `--privileged` sauf nécessité absolue.

### Limiter l’exposition réseau

N’exposez publiquement que ce qui est indispensable.

Préférez :

- VPN ;
- accès interne ;
- reverse proxy filtrant ;
- listes IP autorisées.

### Maintenir à jour

Mettez à jour :

- système hôte ;
- images conteneurs ;
- dépendances applicatives.

### Gérer les secrets proprement

Évitez mots de passe en clair dans les commandes ou dépôts Git.

Utilisez :

- variables d’environnement sécurisées ;
- fichiers montés ;
- gestionnaires de secrets.

### Sauvegarder les volumes

Les données vivent souvent dans les volumes :

- bases SQL ;
- uploads ;
- configurations ;
- certificats.

Sauvegardez-les régulièrement et testez la restauration.

### Journaliser et surveiller

Consultez :

```bash
podman logs NOM
podman ps
podman inspect NOM
```

Ajoutez idéalement :

- alertes disque ;
- supervision CPU/RAM ;
- expiration certificats ;
- disponibilité HTTP.

### Lire seule quand possible

Montez les conteneurs en lecture seule si l’application le permet :

```bash
--read-only
```

### Séparer les services

Un service critique ne devrait pas partager inutilement ses volumes ou son réseau avec un autre.

---

## Exemple concret d’architecture simple et sûre

Un serveur personnel peut ressembler à ceci :

- Podman rootless par utilisateur dédié ;
- Caddy ou Nginx frontal ;
- applications sur ports internes ;
- nftables en entrée ;
- sauvegardes nocturnes ;
- fail2ban sur SSH ;
- authentification par clés ;
- mises à jour régulières.

Services possibles :

- Nextcloud ;
- Vaultwarden ;
- Gitea ;
- Grafana ;
- Home Assistant ;
- Jellyfin.

---

## Conclusion

L’auto-hébergement n’a jamais été aussi accessible, mais il doit s’accompagner d’une culture minimale de sécurité. Les conteneurs permettent d’isoler les services, de simplifier les mises à jour et de rendre les déploiements reproductibles.

Parmi les solutions disponibles, **Podman** brille particulièrement grâce à :

- son mode rootless natif ;
- l’absence de daemon central ;
- sa sobriété ;
- sa compatibilité avec les standards OCI ;
- sa simplicité d’usage.

Même les contraintes classiques — comme les ports < 1024 ou certaines images prévues pour root — disposent de solutions propres et maîtrisables.

Le meilleur conseil reste simple : commencez petit.

Déployez un premier service non critique, documentez votre configuration, sauvegardez vos données, apprenez à surveiller vos journaux… puis étendez progressivement votre infrastructure.

L’auto-hébergement n’est pas réservé aux experts. Avec une approche progressive et des outils modernes comme Podman, il devient possible de reprendre le contrôle **en toute sécurité**.
