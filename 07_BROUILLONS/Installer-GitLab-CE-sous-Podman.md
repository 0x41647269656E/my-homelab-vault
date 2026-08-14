---
title: Installer GitLab CE sous Podman rootless
author: "0x41647269656E"
series: Installation des solutions open source
tags:
  - self-hosting
  - gitlab
  - podman
  - installation
  - securite
date: 06-06-2026
last_modified: 06-06-2026
reading-time: 15m
difficulty: tech-enthusiast
status: draft
---

# Installer GitLab CE sous Podman rootless

> [!abstract] TL;DR
> GitLab CE est un **monolithe lourd** : un seul conteneur (image omnibus) qui embarque déjà PostgreSQL, Redis, Sidekiq, Nginx interne, Gitaly… C'est l'opposé d'Immich côté philosophie. Le défi n'est pas le multi-conteneurs mais la **gestion des ressources** (GitLab est gourmand) et des **ports** (HTTP + SSH Git). Je l'installe en Podman rootless, accès via Caddy derrière VPN, avec un SSH Git sur port dédié. Peu d'interaction avec `/media` : GitLab gère ses propres données. Cet article se concentre sur l'apprivoisement du monolithe.

> [!info] Prérequis de lecture
> Socle : [[Self-hosting sécurisé avec Podman]], [[Reverse proxy Caddy avec TLS automatique]], [[Configuration WireGuard self-hosted]], confinement MAC, [[Stratégie de sauvegarde restic 3-2-1]].

---

## La philosophie inverse : un monolithe omnibus

Là où Immich éclate ses fonctions en quatre conteneurs, GitLab fait le choix opposé : **tout dans une image**. L'image omnibus contient sa propre base, son cache, ses workers, son serveur web. Avantage : un seul objet à déployer. Inconvénient : c'est **gros, lent à démarrer, et gourmand**.

> [!warning] GitLab a besoin de ressources
> Compte **4 Go de RAM minimum**, 8 Go confortables. Le démarrage initial (reconfigure interne) prend plusieurs minutes — c'est normal, GitLab initialise sa base et ses services. Ne panique pas si le conteneur semble inactif au premier boot : surveille les logs, il travaille. Sur un mini-PC self-hosted partagé avec d'autres services, GitLab est souvent le plus gros consommateur. Dimensionne en conséquence.

Ce monolithe a une conséquence sur le durcissement : on ne peut pas appliquer `ReadOnly=true` (GitLab écrit partout dans son arborescence) ni `DropCapability=ALL` aussi agressivement que sur un service mono-fonction. Le confinement repose davantage sur le **rootless**, le **réseau interne** et les **limites de ressources**.

---

## Les ports de GitLab

> [!info] Deux ports fonctionnels
> - **HTTP (interface web + API + registry)** → proxifié par Caddy.
> - **SSH (clone/push Git via SSH)** → port dédié. GitLab écoute par défaut sur 22 en interne ; en rootless on le mappe sur un port hôte non privilégié, et on configure GitLab pour annoncer ce port aux utilisateurs.

---

## Le Quadlet

```ini
# ~/.config/containers/systemd/gitlab.container
[Unit]
Description=GitLab CE

[Container]
Image=docker.io/gitlab/gitlab-ce:latest
ContainerName=gitlab
AutoUpdate=registry
HostName=gitlab.mondomaine.fr

Network=gitlab-internal.network

# SSH Git sur port hôte dédié (2222 -> 22 interne)
PublishPort=2222:22

# --- Données GitLab : trois volumes dédiés (config, logs, data) ---
# Volumes propres à GitLab, pas de lien avec /media : :U légitime
Volume=%h/gitlab/config:/etc/gitlab:U,Z
Volume=%h/gitlab/logs:/var/log/gitlab:U,Z
Volume=%h/gitlab/data:/var/opt/gitlab:U,Z

# --- Configuration via la variable omnibus ---
# external_url + port SSH annoncé aux utilisateurs
Environment=GITLAB_OMNIBUS_CONFIG=external_url 'https://git.mondomaine.fr'; gitlab_rails['gitlab_shell_ssh_port'] = 2222; nginx['listen_port'] = 80; nginx['listen_https'] = false;

# Durcissement adapté à un monolithe (moins agressif, à dessein)
NoNewPrivileges=true
# SHM nécessaire à PostgreSQL embarqué
ShmSize=256m

[Service]
Restart=on-failure
RestartSec=30
# Démarrage lent : on laisse le temps
TimeoutStartSec=600
# GitLab est gourmand : on plafonne pour protéger les autres services
MemoryMax=6G

[Install]
WantedBy=default.target
```

> [!success] Les choix spécifiques au monolithe
> - **`nginx['listen_https'] = false`** : le TLS est terminé par Caddy en amont. GitLab parle en HTTP clair sur le réseau interne, Caddy chiffre vers l'extérieur. On ne fait pas du TLS deux fois.
> - **`gitlab_shell_ssh_port = 2222`** : GitLab doit **annoncer** le bon port SSH dans les URL de clone, sinon les utilisateurs copient `git@...:22` qui ne marche pas. On aligne l'annonce sur le port publié.
> - **`MemoryMax=6G`** + **`TimeoutStartSec=600`** : on borne la gourmandise et on tolère le démarrage lent.
> - **Pas de `ReadOnly=true`** : impossible sur l'omnibus qui écrit partout. C'est un compromis assumé, compensé par le rootless et l'isolation réseau.

```bash
podman network create gitlab-internal --internal
mkdir -p ~/gitlab/{config,logs,data}
systemctl --user daemon-reload
systemctl --user start gitlab.service
# Suivre le long premier démarrage :
journalctl --user -u gitlab.service -f
```

> [!info] Récupérer le mot de passe root initial
> Au premier démarrage, GitLab génère un mot de passe root temporaire :
> ```bash
> podman exec gitlab grep 'Password:' /etc/gitlab/initial_root_password
> ```
> Connecte-toi, change-le immédiatement, puis ce fichier est supprimé automatiquement après 24 h.

---

## Exposition : Caddy + SSH Git

### Interface web via Caddy

```caddyfile
git.mondomaine.fr {
	import security_headers
	# GitLab peut servir de gros artefacts / images de registry
	request_body {
		max_size 2GB
	}
	@allowed remote_ip 10.10.1.0/24 10.10.2.0/24   # admin + famille/collaborateurs VPN
	handle @allowed {
		reverse_proxy gitlab:80
	}
	respond "Forbidden" 403
}
```

> [!tip] Exposer ou non GitLab publiquement
> Si tu collabores avec des gens hors VPN, tu peux exposer GitLab publiquement — mais c'est une grosse surface d'attaque (registry, runners, API). Mon choix : **VPN par défaut**, je n'expose publiquement que si une vraie collaboration externe l'exige, et alors avec MFA obligatoire et inscription fermée. Pour un usage perso/famille, le VPN suffit largement.

### SSH Git

Le port 2222 est publié directement (pas via Caddy, le SSH n'est pas du HTTP). Tes URL de clone deviennent :

```bash
git clone ssh://git@git.mondomaine.fr:2222/groupe/projet.git
```

> [!warning] SSH Git et exposition
> Si GitLab reste derrière VPN, le port 2222 n'a besoin d'être joignable que dans le tunnel. Adapte tes règles nftables (cf. [[Configuration WireGuard self-hosted]]) pour autoriser 2222 depuis les sous-réseaux VPN concernés, et rien d'autre.

---

## Sauvegarde

GitLab a son **propre système de sauvegarde**, et c'est lui qu'il faut utiliser — pas une copie de fichiers du monolithe à chaud.

> [!danger] Ne copie pas les volumes GitLab à chaud
> Comme pour toute base de données, copier `/var/opt/gitlab` pendant que GitLab tourne donne un état incohérent. GitLab fournit `gitlab-backup`, qui produit une archive cohérente (dépôts, base, uploads, CI artifacts).

```bash
# Sauvegarde applicative cohérente (données : dépôts, base, etc.)
podman exec gitlab gitlab-backup create

# L'archive atterrit dans /var/opt/gitlab/backups -> ~/gitlab/data/backups côté hôte
```

> [!success] Les deux moitiés de la sauvegarde GitLab
> 1. **L'archive `gitlab-backup`** → contient les données (dépôts, base, artefacts).
> 2. **Les fichiers de configuration et les secrets** (`/etc/gitlab/gitlab-secrets.json`, `/etc/gitlab/gitlab.rb`) → **NE sont PAS** dans l'archive backup, à dessein. Sans `gitlab-secrets.json`, tu ne peux pas déchiffrer les données restaurées (2FA, tokens, variables CI chiffrées).
>
> **Il faut sauvegarder les deux**, idéalement séparément (l'archive d'un côté, les secrets de l'autre, comme pour une clé de chiffrement). restic embarque ensuite l'archive + le contenu de `~/gitlab/config`. Cf. [[Stratégie de sauvegarde restic 3-2-1]].

---

## Conclusion

GitLab CE est le contre-pied de la série : un **monolithe** qu'on n'éclate pas, qu'on ne peut pas durcir aussi agressivement qu'un micro-service, et dont le vrai enjeu est la **gestion des ressources** et la **double sauvegarde** (archive + secrets). On compense le durcissement limité par le rootless, l'isolation réseau et des limites mémoire strictes. Et comme souvent : VPN par défaut, exposition publique seulement si la collaboration externe l'impose vraiment.

> [!note] Articles liés
> - [[Self-hosting sécurisé avec Podman]]
> - [[Reverse proxy Caddy avec TLS automatique]]
> - [[Stratégie de sauvegarde restic 3-2-1]]

---

*Retour d'expérience personnel. GitLab évolue vite : vérifie l'image, les options omnibus et les ressources recommandées à jour. Adapte domaines, ports et plages VPN.*
