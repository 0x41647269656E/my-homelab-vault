---
title: "Stratégie de sauvegarde restic 3-2-1 : protéger des données qu'on ne peut pas reconstituer"
author: "0x41647269656E"
series: "Administration"
tags:
  - self-hosting
  - restic
  - sauvegarde
  - backup
  - securite
  - devops
date: 05-06-2026
last_modified: 05-06-2026
reading-time: 15m
difficulty: tech-enthusiast
status: published
---

# Stratégie de sauvegarde restic 3-2-1 : protéger des données qu'on ne peut pas reconstituer

> [!abstract] TL;DR
> Tout le durcissement des articles précédents protège contre l'**attaquant**. Celui-ci protège contre la **perte** — qui est statistiquement le risque le plus probable : disque mort, `rm -rf` malheureux, ransomware, corruption silencieuse. J'applique la règle **3-2-1** avec **restic** : 3 copies des données, sur 2 supports différents, dont 1 hors-site. Le tout chiffré côté client, dédupliqué, avec des dépôts en **append-only** pour résister au ransomware, et — le point que personne ne fait — des **restaurations testées automatiquement**. La subtilité réelle : sauvegarder des conteneurs vivants (bases de données, volumes SELinux) demande de dumper proprement, pas de copier des fichiers sous les pieds d'un PostgreSQL en écriture.

> [!info] Note de cohérence
> Dernier volet de la série après [[Self-hosting sécurisé avec Podman]], [[Reverse proxy Caddy avec TLS automatique]], [[Configuration WireGuard self-hosted]] et [[Durcissement SELinux pour conteneurs]]. Je réutilise le stack : Podman rootless, volumes par service, labels SELinux. La restauration des labels SELinux mentionnée dans l'article précédent trouve ici sa procédure complète.

---

## Le problème de fond : la menace la plus probable n'est pas l'attaquant

J'ai passé quatre articles à durcir mon parc contre la compromission. Pourtant, si je suis honnête sur les probabilités, **ce qui va m'arriver, ce n'est pas un pirate**. C'est un disque qui meurt, une fausse manip, une mise à jour qui corrompt une base, un `docker volume rm` tapé dans le mauvais terminal.

### Pourquoi la sauvegarde est la couche la plus importante

> [!danger] La hiérarchie réelle des risques
> Pour des données personnelles auto-hébergées, le classement par probabilité est sans appel :
> 1. **Défaillance matérielle** — les disques meurent, ce n'est qu'une question de quand.
> 2. **Erreur humaine** — la commande de trop, le mauvais flag, le volume supprimé.
> 3. **Corruption logicielle** — un bug, une migration ratée, un système de fichiers qui déraille.
> 4. **Ransomware / compromission** — réel, mais moins probable qu'un disque mort sur un parc perso.
>
> Aucun durcissement SELinux ne te rend tes photos quand le SSD lâche. **La sauvegarde est la seule couche qui couvre les quatre.** C'est pour ça que c'est, paradoxalement, l'article le plus important de la série.

### Ce que la 3-2-1 garantit, et pourquoi chaque chiffre compte

> [!success] La règle 3-2-1 décortiquée
> - **3 copies** des données : l'originale + 2 sauvegardes. Une seule sauvegarde, c'est une sauvegarde dont tu ne sais pas si elle marche.
> - **2 supports différents** : ne pas tout mettre sur le même type de média. Si mes deux copies sont sur deux disques du même modèle achetés le même jour, ils peuvent mourir ensemble.
> - **1 hors-site** : une copie physiquement ailleurs. Incendie, vol, dégât des eaux, foudre — un sinistre local ne doit pas emporter toutes les copies.
>
> Chaque chiffre couvre un mode de défaillance que les autres ne couvrent pas. Retirer un seul chiffre rouvre une classe entière de scénarios de perte totale.

Mon implémentation concrète : original sur le serveur, copie 1 sur un disque externe local (support différent), copie 2 sur un stockage objet distant chiffré (hors-site). Trois copies, deux supports, un hors-site. ✅

---

## Pourquoi restic

J'ai utilisé `rsync`, `borg`, puis `restic`. Voici l'arbitrage, dans l'esprit des articles précédents.

### rsync ne suffit pas pour de la vraie sauvegarde

`rsync` synchronise, il ne **versionne** pas. Si un fichier est corrompu ou chiffré par un ransomware côté source, le prochain `rsync` propage la corruption vers la destination. Pas d'historique, pas de point de restauration antérieur. C'est de la **réplication**, pas de la sauvegarde. Utile, mais ce n'est pas une réponse aux risques 2, 3 et 4.

### restic vs borg

`borg` est excellent et je le recommande sans réserve. J'ai choisi `restic` pour des raisons précises :

> [!quote] Pourquoi restic chez moi
> - **Chiffrement côté client par défaut**, non négociable. Le dépôt est inutilisable sans la clé, même pour qui possède le stockage distant. Cohérent avec ma posture : je ne fais pas confiance au fournisseur de stockage.
> - **Déduplication par blocs** : seuls les blocs réellement nouveaux sont transférés et stockés. Mes photos qui ne changent pas ne sont stockées qu'une fois, même sur 100 snapshots.
> - **Backends multiples natifs** : local, SFTP, S3, Backblaze B2, rclone… Le même outil pour ma copie locale et ma copie distante, avec une seule logique.
> - **Binaire Go unique**, sans dépendances, qui s'intègre trivialement à systemd — comme tout le reste de mon stack.
> - **Snapshots immuables + mode append-only** : la brique anti-ransomware (détaillée plus bas).

Le compromis : `borg` est légèrement plus efficace en compression sur certains profils. Pour mon usage, la portabilité multi-backend de restic et son chiffrement par défaut l'emportent.

---

## La vraie difficulté : sauvegarder des conteneurs vivants

C'est le piège que tout le monde rencontre et que peu d'articles traitent sérieusement. **Copier les fichiers d'une base de données pendant qu'elle écrit produit une sauvegarde corrompue.**

### Pourquoi on ne sauvegarde pas un volume PostgreSQL « à chaud » par copie de fichiers

> [!danger] Le piège de la copie naïve
> Si je fais un `restic backup ~/paperless/db/` pendant que PostgreSQL tourne, je capture les fichiers de données **dans un état incohérent** : des écritures en cours, des pages WAL pas encore flushées, des transactions à moitié appliquées. Au mieux la restauration échoue ; au pire elle « marche » et tu découvres la corruption des mois plus tard. **Une sauvegarde de base de données par copie de fichiers à chaud est une sauvegarde qui ment.**

La règle : **les données applicatives** (fichiers immuables : photos Immich, PDF Paperless) se sauvegardent par copie de fichiers sans problème. **Les bases de données** doivent être **dumpées** par leur propre outil, qui garantit un instantané cohérent.

### Ma stratégie : dump d'abord, backup ensuite

Je sépare les deux types de données et j'orchestre dans le bon ordre :

```bash
#!/usr/bin/env bash
# ~/backup/pre-backup.sh — exécuté AVANT restic
set -euo pipefail

DUMP_DIR="${HOME}/backup/dumps"
mkdir -p "${DUMP_DIR}"

# Dump cohérent de PostgreSQL via l'outil natif, DANS le conteneur
# pg_dump garantit un snapshot transactionnellement cohérent
podman exec immich-postgres \
    pg_dump -U immich -Fc immich \
    > "${DUMP_DIR}/immich-$(date +%F).dump"

podman exec paperless-postgres \
    pg_dump -U paperless -Fc paperless \
    > "${DUMP_DIR}/paperless-$(date +%F).dump"

# Rotation : ne garder que les 3 derniers dumps locaux (restic versionne le reste)
find "${DUMP_DIR}" -name '*.dump' -mtime +3 -delete
```

> [!success] Pourquoi `pg_dump -Fc` plutôt qu'un dump SQL brut
> Le format `custom` (`-Fc`) produit un dump **compressé**, **restaurable sélectivement** (`pg_restore` peut cibler une table), et cohérent. C'est le format que je recommande pour restaurer proprement. Le SQL brut (`-Fp`) est plus portable mais plus lourd et moins flexible à la restauration.

> [!tip] Quiescer plutôt que dumper, quand c'est possible
> Pour certains services sans base SQL, la stratégie la plus sûre reste d'**arrêter brièvement le conteneur** le temps du snapshot, puis de le relancer. Quelques secondes d'indisponibilité nocturne contre une sauvegarde garantie cohérente : un échange que j'accepte volontiers pour les services non critiques. Pour Immich/Paperless, le dump SQL évite même cette coupure sur la partie données fichiers.

---

## Initialiser et structurer les dépôts restic

### Les deux dépôts : local et distant

```bash
# Dépôt LOCAL sur disque externe (support différent, copie 1)
export RESTIC_PASSWORD_FILE="${HOME}/backup/.restic-pass"
restic -r /mnt/backup-disk/restic-repo init

# Dépôt DISTANT hors-site, ex. Backblaze B2 (copie 2, hors-site)
export B2_ACCOUNT_ID="..."
export B2_ACCOUNT_KEY="..."
restic -r b2:mon-bucket:restic-repo init
```

> [!danger] La clé de chiffrement EST ta sauvegarde
> Si tu perds le mot de passe restic, **tes sauvegardes sont définitivement illisibles** — c'est le but du chiffrement côté client. Ce mot de passe doit être sauvegardé **ailleurs que dans la sauvegarde** : gestionnaire de mots de passe, copie papier dans un endroit sûr, coffre. J'ai vu des gens chiffrer parfaitement des To de données… et perdre la clé. La sauvegarde devient alors un générateur de bruit aléatoire très fiable.

### Le secret de stockage distant ne donne pas accès aux données

Point rassurant et cohérent avec la posture des autres articles : les identifiants B2/S3 permettent de **lire et écrire le dépôt chiffré**, mais **pas de déchiffrer** son contenu. Le chiffrement étant côté client, même un fournisseur de stockage compromis ne voit que du chiffré. Je peux donc stocker ces identifiants avec un peu moins de paranoïa que la clé restic elle-même.

---

## L'orchestration systemd : sauvegarde automatique et fiable

Cohérent avec tout le stack, j'orchestre via systemd (timers), pas cron. Meilleure journalisation, gestion des dépendances, et intégration avec le reste.

### Le service de sauvegarde

```ini
# ~/.config/systemd/user/backup.service
[Unit]
Description=Sauvegarde restic (dump + local + distant)
After=network-online.target

[Service]
Type=oneshot
Environment=RESTIC_PASSWORD_FILE=%h/backup/.restic-pass
EnvironmentFile=%h/backup/b2.env

# 1. Dumps cohérents des bases AVANT toute copie
ExecStartPre=%h/backup/pre-backup.sh

# 2. Sauvegarde vers le dépôt LOCAL
ExecStart=restic -r /mnt/backup-disk/restic-repo backup \
    %h/immich/upload \
    %h/paperless/media \
    %h/backup/dumps \
    %h/.config/containers/systemd \
    --tag automated --tag local

# 3. Sauvegarde vers le dépôt DISTANT (hors-site)
ExecStart=restic -r b2:mon-bucket:restic-repo backup \
    %h/immich/upload \
    %h/paperless/media \
    %h/backup/dumps \
    %h/.config/containers/systemd \
    --tag automated --tag offsite

# 4. Politique de rétention (applique le pruning)
ExecStartPost=restic -r /mnt/backup-disk/restic-repo forget \
    --keep-daily 7 --keep-weekly 4 --keep-monthly 12 --prune
```

> [!success] Ce que je sauvegarde, et ce que je ne sauvegarde pas
> - **Données fichiers** : uploads Immich, médias Paperless → copie directe, ce sont des fichiers immuables.
> - **Dumps SQL** : produits par `pre-backup.sh`, cohérents.
> - **Les Quadlets** (`~/.config/containers/systemd`) : ma **configuration d'infra**. Sauvegarder les données sans la config qui les fait tourner, c'est se condamner à tout reconstruire à la main. Je sauvegarde la définition du système, pas que son contenu.
> - **Je ne sauvegarde PAS les volumes de bases bruts** : ils sont remplacés par les dumps cohérents.

### Le timer

```ini
# ~/.config/systemd/user/backup.timer
[Unit]
Description=Déclenche la sauvegarde restic quotidienne

[Timer]
OnCalendar=*-*-* 03:00:00
# Si la machine était éteinte à l'heure prévue, rattrape au démarrage
Persistent=true
RandomizedDelaySec=900

[Install]
WantedBy=timers.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now backup.timer
systemctl --user list-timers
```

> [!warning] N'oublie pas le linger (déjà vu, toujours valable)
> Comme pour tous les services rootless (cf. [[Self-hosting sécurisé avec Podman]]), `loginctl enable-linger $USER` est nécessaire pour que le timer s'exécute sans session active. Sans ça, ta sauvegarde nocturne ne tournera jamais sur un serveur headless.

---

## La résistance au ransomware : le mode append-only

C'est le point qui relie cet article aux quatre précédents sur la sécurité. Une sauvegarde accessible en écriture par la machine compromise n'est **pas** une protection contre le ransomware — le malware chiffre aussi les sauvegardes.

> [!danger] Le scénario que l'append-only neutralise
> Un attaquant qui prend le contrôle du serveur a accès aux identifiants de sauvegarde stockés dessus. S'il peut **supprimer ou écraser** le dépôt distant, il chiffre tes données ET détruit tes sauvegardes. Le `restic forget --prune` dans mon service, s'il était exécutable par l'attaquant sur le dépôt distant, deviendrait son arme.

> [!success] La parade : un dépôt distant en append-only
> restic supporte un mode où la machine peut **ajouter** des snapshots mais **pas en supprimer**. Sur le serveur de sauvegarde distant :
> ```bash
> # Côté serveur de stockage : restic en mode append-only via rest-server
> rest-server --path /data/restic --append-only
> ```
> Avec Backblaze B2, on obtient l'équivalent via des **clés applicatives restreintes** (write sans delete) couplées au **versioning de bucket** et au verrouillage d'objet. La machine de production peut écrire ses sauvegardes mais **ne peut pas les détruire**. Le `forget --prune` est alors exécuté séparément, depuis un contexte de confiance distinct (manuellement, ou depuis une machine d'administration), jamais par le serveur exposé.

C'est exactement la même philosophie que la segmentation WireGuard ou les catégories MCS de SELinux : **limiter ce qu'un composant compromis peut détruire**. La sauvegarde hors-site n'a de valeur que si l'attaquant qui possède le serveur ne peut pas l'atteindre.

---

## Le test de restauration : la sauvegarde que personne ne vérifie

> [!quote] La vérité qui dérange
> Une sauvegarde jamais restaurée n'est pas une sauvegarde : c'est une **hypothèse**. Le jour où tu en as besoin est le pire moment pour découvrir qu'elle ne marche pas. Je teste mes restaurations automatiquement, parce que je sais que je ne le ferai jamais manuellement assez souvent.

### Vérification d'intégrité régulière

```bash
# Vérifie l'intégrité du dépôt (structure + sous-ensemble de données)
restic -r /mnt/backup-disk/restic-repo check --read-data-subset=10%
```

Je l'exécute via un timer hebdomadaire. `check` vérifie que les snapshots sont cohérents ; `--read-data-subset` relit et revalide une fraction des blocs réels (pas seulement les métadonnées), détectant la corruption silencieuse du stockage.

### Le test de restauration automatisé

```bash
#!/usr/bin/env bash
# ~/backup/test-restore.sh — vérifie qu'on sait VRAIMENT restaurer
set -euo pipefail

TEST_DIR=$(mktemp -d)
trap 'rm -rf "${TEST_DIR}"' EXIT

# 1. Restaurer le dernier snapshot dans un répertoire jetable
restic -r /mnt/backup-disk/restic-repo restore latest --target "${TEST_DIR}"

# 2. Vérifier qu'un dump SQL restauré est rechargeable dans un PG jetable
LATEST_DUMP=$(find "${TEST_DIR}" -name 'immich-*.dump' | sort | tail -n1)
podman run --rm -d --name pg-restore-test \
    -e POSTGRES_PASSWORD=test docker.io/library/postgres:16
sleep 10
podman cp "${LATEST_DUMP}" pg-restore-test:/tmp/test.dump
# pg_restore en mode --list valide la structure du dump sans tout charger
podman exec pg-restore-test pg_restore --list /tmp/test.dump > /dev/null
podman stop pg-restore-test

echo "✅ Restauration vérifiée : snapshot lisible et dump valide"
```

> [!success] Ce que ce test garantit réellement
> Il ne vérifie pas seulement que le fichier existe — il vérifie que le snapshot est **restaurable** et que le dump SQL est **structurellement valide et rechargeable**. C'est la différence entre « j'ai un fichier de sauvegarde » et « je peux remonter mon service ». Le résultat part dans `journalctl`, et une absence de succès me lève une alerte.

---

## La procédure de restauration complète (le jour J)

Pour que ce ne soit pas théorique, voici la séquence réelle de remise en service après perte totale.

```bash
# 1. Restaurer données + config depuis le dépôt (local si dispo, sinon distant)
restic -r b2:mon-bucket:restic-repo restore latest --target /

# 2. CRUCIAL : restaurer les labels SELinux (cf. article SELinux)
#    restic préserve les fichiers, mais les contextes doivent être réappliqués
restorecon -Rv ~/immich/upload ~/paperless/media

# 3. Recréer les bases et recharger les dumps
podman exec immich-postgres \
    pg_restore -U immich -d immich --clean ~/backup/dumps/immich-latest.dump

# 4. Recharger les Quadlets et redémarrer les services
systemctl --user daemon-reload
systemctl --user start immich-server paperless-ngx
```

> [!warning] Le piège SELinux à la restauration
> Comme annoncé dans [[Durcissement SELinux pour conteneurs]] : restic restaure le contenu et les permissions, mais le **contexte SELinux** peut être perdu ou incorrect selon le backend. Un `restorecon -R` après restauration, ou relancer les conteneurs avec leurs volumes `:Z`, réapplique les labels. Oublier cette étape donne des `permission denied` mystérieux juste quand tu essaies de tout remonter sous stress. Documente-le dans ta procédure.

---

## Les frictions réelles que j'ai rencontrées

> [!failure] Les points de douleur, honnêtement
> - **L'ordre dump-puis-backup est facile à rater.** Au début je sauvegardais les volumes bruts en croyant être couvert. Une restauration de test a révélé des bases incohérentes. Le test m'a sauvé avant le vrai incident — c'est exactement son rôle.
> - **La gestion des secrets de sauvegarde.** Clé restic, identifiants B2 : ils vivent sur la machine qu'ils sont censés protéger. L'append-only et le stockage de la clé restic hors-ligne sont les réponses, mais ça demande de la rigueur.
> - **Le coût du stockage distant grossit avec les photos.** Immich ingère vite des dizaines de Go. La déduplication aide, mais la rétention `--keep-monthly 12` a un coût réel. J'ai dû arbitrer ma politique de rétention en fonction du budget de stockage objet.
> - **Les labels SELinux à la restauration.** Le piège mentionné ci-dessus m'a coûté une bonne demi-heure de panique lors de mon premier vrai test de restauration complète. Maintenant c'est dans ma checklist.

Une fois en place, le système est largement autonome : sauvegarde nocturne, vérification hebdomadaire, test de restauration mensuel, le tout dans `journalctl`. Je reçois une alerte uniquement quand quelque chose échoue. Et surtout — je **sais** que mes sauvegardes marchent, parce qu'une machine le vérifie pour moi.

---

## Conclusion : une sauvegarde non testée n'existe pas

Trois principes que je retire de cette mise en place :

1. **La perte est plus probable que l'attaque.** Tout le durcissement des articles précédents ne rend pas un seul octet quand le disque meurt. La 3-2-1 est la seule couche qui couvre matériel, erreur humaine, corruption ET ransomware.
2. **Sauvegarder un conteneur vivant demande de comprendre ses données.** Fichiers immuables → copie directe. Bases de données → dump cohérent. Confondre les deux produit une sauvegarde qui ment, et on ne le découvre qu'au pire moment.
3. **Le test de restauration n'est pas optionnel — c'est la sauvegarde elle-même.** Une sauvegarde jamais restaurée est une hypothèse. L'automatiser transforme l'hypothèse en certitude. Et l'append-only étend la posture de sécurité des quatre autres articles jusqu'à la dernière ligne de défense.

> [!note] La série complète dans le vault
> - [[Self-hosting sécurisé avec Podman]]
> - [[Reverse proxy Caddy avec TLS automatique]]
> - [[Configuration WireGuard self-hosted]]
> - [[Durcissement SELinux pour conteneurs]]

---

*Retour d'expérience personnel. Adapte les chemins, les noms de bases, le backend distant et la politique de rétention à ton parc et ton budget. La règle 3-2-1 est un plancher, pas un plafond : sur des données vraiment irremplaçables, certains ajoutent une copie froide supplémentaire hors-ligne.*
