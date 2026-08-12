# Docker rootless vs Podman + Quadlets

## 1. Le rootless en général : socle commun

Docker rootless et Podman rootless reposent sur les **mêmes mécanismes noyau Linux** :

- **User namespaces** (`CLONE_NEWUSER`) : un utilisateur non privilégié apparaît comme `root` (UID 0) *dans* le conteneur, mais reste mappé vers un UID non privilégié arbitraire sur l'hôte.
- **subuid/subgid** : plage d'UID/GID allouée à l'utilisateur pour le remapping, définie dans `/etc/subuid` et `/etc/subgid`.
- **fuse-overlayfs** (ou overlayfs natif sur kernel ≥ 5.11-5.13) comme storage driver, en remplacement d'`overlay2` qui nécessite des privilèges.
- **slirp4netns** ou **pasta** pour le réseau, en émulation userspace (pas d'accès direct aux interfaces hôte).
- **cgroups v2** avec délégation systemd pour la gestion des limites de ressources (CPU/mémoire).

### Exemple de mapping subuid/subgid

```
# /etc/subuid
adrientanaka:100000:65536

# /etc/subgid
adrientanaka:100000:65536
```

### Schéma du flux rootless générique

```
Utilisateur (UID 1000)
  ├─ crée user namespace (UID 1000 → 0 dans le ns)
  ├─ crée mount namespace
  ├─ storage: fuse-overlayfs (ou overlayfs natif si kernel récent)
  └─ réseau: slirp4netns / pasta (tun/tap userspace)
```

---

## 2. Architecture fondamentale : daemon centralisé vs fork/exec direct

### Docker rootless

Modèle client-serveur classique : un daemon unique (`dockerd`) tourne en arrière-plan dans un user namespace, orchestré par `rootlesskit`. Le client `docker` lui parle via un socket Unix. Tous les conteneurs sont des enfants de ce daemon.

```
docker run nginx
  └─ docker CLI → socket → dockerd (process persistant)
                              └─ containerd
                                   └─ runc → conteneur (petit-fils du daemon)
```

**Conséquence** : si `dockerd` crashe ou si la session est tuée sans `loginctl enable-linger`, **tous** les conteneurs meurent avec lui — point de défaillance unique.

### Podman

Pas de daemon. Chaque `podman run` est un fork/exec direct : le process CLI devient (via `conmon`) le parent direct du conteneur.

```
podman run nginx
  └─ podman CLI → fork/exec direct
                    └─ conmon (monitor léger, 1 par conteneur)
                         └─ runc → conteneur (fils direct de conmon)
```

**Conséquence** : chaque conteneur est indépendant. Tuer un `conmon` n'affecte que son conteneur.

---

## 3. Les Quadlets : supervision native systemd

Un **Quadlet** est un fichier `.container` (syntaxe INI proche de `docker-compose.yml` mais native systemd), placé dans :
- `~/.config/containers/systemd/` (rootless)
- `/etc/containers/systemd/` (root)

Au démarrage, `systemd` (via `podman-system-generator`) transforme automatiquement ce fichier en unit systemd classique.

### Exemple : Jellyfin en Quadlet

```ini
# ~/.config/containers/systemd/jellyfin.container
[Unit]
Description=Jellyfin Media Server
After=network-online.target

[Container]
Image=docker.io/jellyfin/jellyfin:latest
AutoUpdate=registry
Volume=%h/jellyfin/config:/config:Z
Volume=%h/jellyfin/cache:/cache:Z
Volume=/mnt/media:/media:ro,Z
PublishPort=8096:8096
Environment=JELLYFIN_PublishedServerUrl=jellyfin.local

[Service]
Restart=on-failure
TimeoutStartSec=900

[Install]
WantedBy=default.target
```

### Commandes de gestion

```bash
systemctl --user daemon-reload
systemctl --user start jellyfin.service
systemctl --user enable jellyfin.service
systemctl --user status jellyfin.service
journalctl --user -u jellyfin.service -f
```

---

## 4. Tableau comparatif

| Aspect | Docker rootless | Podman + Quadlets |
|---|---|---|
| **Process model** | Daemon centralisé (`dockerd`) | Fork/exec direct, pas de daemon |
| **Supervision/restart** | Géré en interne par `dockerd` (`restart:` dans run/compose) | Délégué entièrement à `systemd` (`Restart=` natif) |
| **Persistance au boot** | `dockerd` doit tourner en continu (`loginctl enable-linger` + service du daemon) | Pas de daemon à maintenir — chaque conteneur est directement une unit systemd |
| **Logs** | `docker logs` (driver JSON par défaut, ou syslog) | Intégration native `journald` via `journalctl -u` |
| **Définition déclarative** | `docker-compose.yml` (Compose = outil séparé) | Fichier `.container` = syntaxe INI native systemd, pas d'outil tiers |
| **Dépendances entre services** | `depends_on` dans Compose (limité, pas de vraie gestion d'état) | `After=`, `Requires=`, `Wants=` — sémantique systemd complète |
| **Auto-update des images** | Nécessite Watchtower ou équivalent tiers | `AutoUpdate=registry` natif via `podman-auto-update.timer` |
| **Isolation par défaut** | Un seul user namespace pour tous les conteneurs du daemon | User namespace par conteneur possible (isolation plus fine) |
| **Point de défaillance unique** | Oui — `dockerd` down = tout down | Non — chaque conteneur indépendant |
| **Overhead mémoire** | Le daemon consomme des ressources en permanence | `conmon` minimal (~quelques Mo), rien ne tourne si pas de conteneur actif |
| **Réseau (rootless)** | slirp4netns / pasta, mêmes limitations | slirp4netns / pasta, mêmes limitations |
| **Communication inter-conteneurs** | Réseau bridge classique uniquement | **Pods** (namespace réseau partagé, concept hérité de Kubernetes) |

---

## 5. Point commun : les limitations réseau du rootless

Que ce soit Docker rootless ou Podman rootless, le réseau passe par les **mêmes mécanismes** (`slirp4netns` ou `pasta`) :
- Pas d'accès direct aux interfaces réseau de l'hôte
- Pas de vrai `network_mode: host` (reste dans un espace réseau simulé)
- Overhead sur le NAT logiciel (latence, débit réduit par rapport à un bridge kernel)

Les Quadlets ne changent rien à cette limitation — c'est une contrainte du kernel/user namespace, pas de l'orchestrateur.

---

## 6. Les Pods Podman — absents de Docker

Concept propre à l'écosystème Podman/CRI-O, hérité de Kubernetes. Un pod partage un même namespace réseau entre plusieurs conteneurs, permettant une communication directe en `localhost` sans bridge dédié.

### Exemple : pod média

```ini
# ~/.config/containers/systemd/media.pod
[Pod]
PodName=media-stack
PublishPort=8096:8096

[Install]
WantedBy=default.target
```

```ini
# ~/.config/containers/systemd/jellyfin.container
[Container]
Image=docker.io/jellyfin/jellyfin:latest
Pod=media-stack.pod
...
```

---

## 7. Résumé des gains d'une migration Docker Compose → Podman + Quadlets

1. **Suppression du point de défaillance unique** — plus de daemon central à surveiller
2. **Restart policy fiable et cohérente** avec le reste du système (tout passe par `systemctl --user`)
3. **Logs unifiés** dans `journald` (`journalctl --user -u jellyfin -u radarr -u sonarr`)
4. **Pods** pour simplifier la communication réseau entre plusieurs conteneurs d'un même stack, sans réseau bridge dédié

### Coût de la migration

Conversion du `docker-compose.yml` existant en fichiers `.container`. L'outil **`podlet`** peut automatiser une bonne partie de cette conversion à partir d'un fichier Compose existant.
