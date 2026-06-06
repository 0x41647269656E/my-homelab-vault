---
title: "Durcissement AppArmor pour conteneurs : le MAC sur Debian/Ubuntu"
author: "0x41647269656E"
series: Hardening
tags:
  - self-hosting
  - apparmor
  - podman
  - securite
  - durcissement
  - devops
date: 05-06-2026
last_modified: 05-06-2026
reading-time: 15m
status: draft
---
# Durcissement AppArmor pour conteneurs : le MAC sur Debian/Ubuntu

> [!abstract] TL;DR
> Variante Debian/Ubuntu de l'article SELinux. Même objectif — une couche de **MAC** qui rattrape ce que les namespaces laisseraient passer — mais un modèle différent : AppArmor confine par **chemin de fichier** (*path-based*), pas par étiquette (*label-based*). Concrètement, mes conteneurs tournent sous le profil `containers-default` chargé par Podman, et pour les services exposés je charge un **profil custom** qui restreint les chemins accessibles. Le point d'honnêteté important : **AppArmor n'a pas d'équivalent direct au MCS** de SELinux — l'isolation fine *entre* conteneurs repose donc davantage sur les namespaces et le réseau que sur le MAC. Cet article explique le modèle, la mise en place, et le diagnostic des blocages.

> [!info] Note de cohérence
> Variante de [[Durcissement SELinux pour conteneurs]], à lire si ton serveur tourne sous Debian/Ubuntu plutôt que Fedora/RHEL. Le reste de la série reste valable tel quel : [[Self-hosting sécurisé avec Podman]], [[Reverse proxy Caddy avec TLS automatique]], [[Configuration WireGuard self-hosted]], [[Stratégie de sauvegarde restic 3-2-1]]. Différence pratique notable : **sous AppArmor, les flags `:z`/`:Z` sur les volumes ne servent à rien** — c'est un mécanisme purement SELinux. J'y reviens.

---

## Le problème de fond : pareil que SELinux, modèle différent

La motivation est identique à l'article SELinux et je ne la réexpose pas en détail : les namespaces relèvent du **DAC** (contrôle par identité/UID) ; il manque une couche **MAC** (contrôle par politique, indépendant de l'identité) pour rattraper une évasion de conteneur. Si ce raisonnement DAC vs MAC ne te parle pas, relis l'intro de l'article SELinux — il est strictement transposable.

Ce qui change, c'est **comment** le MAC est implémenté.

### Path-based vs label-based : la différence qui structure tout

> [!info] Les deux philosophies du MAC sous Linux
> - **SELinux (label-based)** : chaque fichier et processus porte une **étiquette** persistante stockée dans les attributs étendus (xattr). La politique raisonne sur ces labels. Un fichier « est » un certain type, où qu'il soit.
> - **AppArmor (path-based)** : la politique raisonne sur les **chemins de fichiers**. Un profil dit « ce programme peut lire `/etc/immich/**` et écrire `/var/lib/immich/**` ». Ce sont les chemins qui définissent les permissions, pas une étiquette attachée au fichier.

Cette différence n'est pas cosmétique, elle a des conséquences pratiques directes :

> [!success] Ce qu'AppArmor rend plus simple
> - **Les profils sont lisibles directement** : un profil AppArmor ressemble à une liste de chemins avec des permissions (`r`, `w`, `ix`…). On comprend ce qu'un profil autorise en le lisant, sans outils de traduction.
> - **Pas de relabel de volumes.** Le piège n°1 de SELinux (`:Z` oublié → `permission denied`) **n'existe pas** sous AppArmor. Tu bind-montes un volume, il est accessible si le profil autorise son chemin. Pas de catégorie à propager, pas de `restorecon` après restauration de sauvegarde.

> [!warning] Ce qu'AppArmor rend plus faible
> - **Pas d'isolation MCS entre conteneurs.** C'est le point critique. Sous SELinux, chaque conteneur recevait une catégorie unique (`c142,c537`) qui l'empêchait de toucher les fichiers d'un autre conteneur, même sous le même UID. **AppArmor n'a pas cet équivalent natif** : par défaut, tous les conteneurs partagent le même profil (`containers-default`), donc le MAC ne les distingue pas entre eux.
> - **Le confinement par chemin est contournable par les liens et montages.** Un modèle basé sur les chemins est, par nature, plus sensible aux astuces de résolution de chemin qu'un modèle basé sur des étiquettes inamovibles.

C'est l'arbitrage de fond à accepter : AppArmor est **plus simple à opérer** mais offre une **isolation inter-conteneurs plus faible** que SELinux. Sous Debian/Ubuntu, on compense ce manque en s'appuyant davantage sur les **autres couches** (namespaces rootless, réseaux internes par service, UID distincts).

---

## Comment AppArmor confine un conteneur Podman

### Le profil par défaut : `containers-default`

Quand AppArmor est actif (cas par défaut sur Ubuntu, et sur Debian depuis un moment), Podman charge automatiquement un profil nommé quelque chose comme `containers-default-X.Y.Z` pour chaque conteneur. Ce profil restreint déjà pas mal : pas d'écriture dans les chemins système sensibles, pas d'accès à `/proc` et `/sys` arbitraire, etc.

Vérifie l'état d'AppArmor et le profil appliqué :

```bash
# AppArmor est-il actif ?
sudo aa-status

# Quel profil s'applique à un conteneur en cours ?
podman inspect immich-server --format '{{ .AppArmorProfile }}'
# containers-default-0.57.4
```

Tu peux aussi voir le profil effectif depuis l'intérieur via le processus :

```bash
# Le mode "enforce" signifie que les violations sont bloquées (pas juste loggées)
sudo aa-status | grep -A2 "enforce mode"
```

### Les modes : `enforce` vs `complain`

> [!info] Les deux modes d'un profil AppArmor
> - **`enforce`** → les actions non autorisées par le profil sont **bloquées** (et journalisées). C'est l'objectif en production.
> - **`complain`** → les actions non autorisées sont **autorisées mais journalisées**. C'est le mode pour **apprendre** ce dont un service a besoin avant de durcir (équivalent fonctionnel du `permissive` SELinux, mais par profil et non global).

L'intérêt du mode `complain` par profil : contrairement au `setenforce 0` de SELinux qui désactive **tout** le MAC du système, sous AppArmor tu peux mettre **un seul profil** en `complain` pour le déboguer, pendant que tous les autres restent en `enforce`. C'est plus granulaire et nettement moins risqué.

---

## Les volumes : pourquoi `:z`/`:Z` ne servent à rien ici

C'est la différence pratique la plus immédiate quand on vient d'un stack SELinux ou qu'on suit un guide écrit pour Fedora.

> [!danger] Ne mets pas `:Z` sous AppArmor en pensant être protégé
> Les flags `:z` et `:Z` sont un mécanisme **purement SELinux** : ils déclenchent un relabel des fichiers. Sous AppArmor, ils sont **silencieusement ignorés** (Podman ne fait rien d'utile avec). Si tu copies un Quadlet écrit pour SELinux (comme ceux des articles précédents), le `:Z` ne casse rien mais ne **protège rien** non plus. Ne te crois pas confiné entre conteneurs juste parce qu'il y a un `:Z` — sous AppArmor, ça n'a aucun effet.

Conséquence sur les Quadlets de la série : sous Debian/Ubuntu, le `:U` reste **indispensable** (c'est le remap d'UID rootless, indépendant du MAC), mais le `:Z` est inutile. Le volume Immich devient :

```ini

# Sous SELinux (articles précédents) :
# Volume=%h/immich/upload:/usr/src/app/upload:U,Z

# Sous AppArmor (Debian/Ubuntu) : le :Z ne sert à rien, on garde juste :U
Volume=%h/immich/upload:/usr/src/app/upload:U
```

> [!success] La bonne nouvelle qui va avec
> Tout le chapitre « débogage des `permission denied` à cause d'un `:Z` oublié » de l'article SELinux **disparaît**. Sous AppArmor, un `permission denied` sur un volume vient quasi toujours du **DAC** (UID mal mappé → vérifier `:U`) ou du **profil AppArmor** qui n'autorise pas le chemin (cas rare avec le profil par défaut sur des volumes de données). Moins de couches à confondre.

---

## Garder AppArmor actif : le bon réflexe

Même logique que pour SELinux : la tentation de tout désactiver au premier blocage existe, et c'est une erreur.

> [!danger] Ne désactive pas AppArmor globalement
> L'équivalent paresseux du `setenforce 0` est de désactiver le service AppArmor ou de lancer les conteneurs avec `--security-opt apparmor=unconfined`. C'est jeter la couche MAC. La bonne approche granulaire :
> ```bash
> # Mettre UN profil en complain pour le déboguer (pas tout le système)
> sudo aa-complain /etc/apparmor.d/usr.bin.mon-service
> # ... observer les logs, ajuster le profil ...
> # Puis remettre en enforce
> sudo aa-enforce /etc/apparmor.d/usr.bin.mon-service
> ```
> Tu ne désactives jamais le MAC pour tous les conteneurs ; tu relâches temporairement un seul profil, le temps de comprendre.

Pour un conteneur précis et **uniquement en dépannage**, on peut le passer unconfined :

```bash
# UNIQUEMENT pour isoler si AppArmor est bien la cause d'un blocage
podman run --security-opt apparmor=unconfined ...
```

Si le problème disparaît en `unconfined`, tu sais que c'est AppArmor ; tu corriges alors le profil, et tu **remets le confinement**. `unconfined` n'est jamais un état final acceptable pour un service exposé.

---

## Écrire un profil custom pour un service exposé

Le profil par défaut suffit pour la plupart des services. Pour mes services exposés sur Internet (Immich), je charge un profil plus serré. Sous AppArmor, on peut **apprendre** le profil au lieu de l'écrire à la main.

### La méthode d'apprentissage : `complain` puis `aa-logprof`

> [!success] Le workflow d'apprentissage AppArmor
> 1. Démarrer avec un profil minimal en mode `complain`.
> 2. Faire **fonctionner le service normalement** quelque temps (usage réel : upload de photos, ML, etc.) — toutes les actions sont autorisées mais loggées.
> 3. Lancer `aa-logprof` : l'outil lit les logs, te montre **chaque accès observé**, et te demande interactivement de l'autoriser ou le refuser, construisant le profil au plus juste.
> 4. Passer le profil en `enforce`.

```bash
# Outils d'édition de profils
sudo apt install -y apparmor-utils

# Analyser les logs et construire/affiner le profil interactivement
sudo aa-logprof
```

`aa-logprof` est l'équivalent fonctionnel d'`udica` côté SELinux, mais en plus interactif : il observe le comportement réel et te propose les règles. C'est, à mon sens, le point où AppArmor est plus agréable que SELinux — la génération de profil sur mesure est plus naturelle.

### À quoi ressemble un profil

Un profil AppArmor est lisible, c'est sa force. Structure typique (simplifiée) :

```
# /etc/apparmor.d/podman.immich
#include <tunables/global>

profile podman-immich flags=(attach_disconnected) {
  #include <abstractions/base>

  # Réseau
  network inet tcp,
  network inet udp,

  # Lecture seule sur les binaires de l'app
  /usr/src/app/** r,
  /usr/local/bin/** rix,

  # Lecture-écriture sur les seules données métier
  /usr/src/app/upload/** rw,
  /tmp/** rw,

  # Refus explicite des chemins sensibles (deny est prioritaire)
  deny /etc/shadow r,
  deny /proc/sysrq-trigger w,
  deny /sys/firmware/** r,
}
```

> [!info] La lecture d'un profil
> - `r` = lecture, `w` = écriture, `ix` = exécuter en restant sous ce profil, `rix` = lecture + exécution confinée.
> - `**` = récursif, `*` = un seul niveau.
> - **`deny` est prioritaire** sur toute autorisation : une règle `deny` ne peut pas être contournée par un `allow` ailleurs. C'est utile pour blinder explicitement les chemins sensibles, même si le reste du profil était trop large.

Charger et appliquer le profil, puis le référencer dans le Quadlet :

```bash
sudo apparmor_parser -r /etc/apparmor.d/podman.immich
sudo aa-enforce /etc/apparmor.d/podman.immich
```

```ini
# Dans le Quadlet immich-server.container, remplacer SecurityLabelType (SELinux)
# par l'option AppArmor :
# SecurityLabelType=container_t        <- SELinux, à RETIRER sous AppArmor
SecurityOpt=apparmor=podman-immich
```

---

## Diagnostiquer un blocage AppArmor

Comme SELinux, AppArmor bloque silencieusement côté appli (`permission denied`) mais journalise tout. La routine de diagnostic.

### Lire les refus dans les logs

Les violations apparaissent dans le journal du noyau avec le mot-clé `apparmor="DENIED"` :

```bash
# Refus AppArmor récents
sudo journalctl -k --since "10 min ago" | grep apparmor

# Ou via dmesg
sudo dmesg | grep -i apparmor

# Si auditd est installé, plus structuré :
sudo ausearch -m avc -ts recent | grep apparmor
```

Un refus typique te donne l'**opération** (`open`, `exec`…), le **profil** concerné, et le **chemin** refusé. Contrairement aux AVC SELinux, c'est directement lisible : tu vois quel chemin manque, tu l'ajoutes au profil (ou tu décides que c'est une intrusion légitime à bloquer).

> [!success] Le diagnostic en deux temps (plus court que SELinux)
> 1. Le **chemin refusé** est-il un accès légitime du service ? → ajouter la règle au profil via `aa-logprof` ou à la main, recharger.
> 2. Sinon → le profil fait son travail, c'est peut-être un comportement anormal à investiguer.
>
> Pas de troisième cas « catégorie MCS incohérente » comme sous SELinux, puisqu'il n'y a pas de MCS. Le diagnostic est structurellement plus simple.

### Recharger un profil après modification

```bash
# Recharger un profil modifié sans redémarrer
sudo apparmor_parser -r /etc/apparmor.d/podman.immich

# Vérifier qu'il est bien en enforce
sudo aa-status | grep immich
```

---

## Compenser l'absence de MCS : l'isolation inter-conteneurs

C'est le point que je veux traiter franchement, parce que c'est la vraie faiblesse relative d'AppArmor pour notre usage.

> [!warning] Le manque à combler
> Sous SELinux, Immich compromis ne pouvait pas toucher les données de Paperless grâce aux catégories MCS, **automatiquement**. Sous AppArmor avec le profil par défaut, les deux conteneurs partagent le même profil : le MAC ne les sépare pas. Il faut donc obtenir cette isolation **autrement**.

> [!success] Mes trois leviers de compensation
> 1. **Un profil custom par service exposé**, restreint à ses seuls chemins de données. Si chaque profil n'autorise que `/usr/src/app/upload/**` pour Immich, le confinement par chemin recrée une partie de l'isolation — Immich confiné ne liste même pas les chemins de Paperless.
> 2. **Des UID distincts par service en rootless.** Faire tourner chaque service sous un **utilisateur système différent** (donc une plage subuid différente) rétablit une isolation DAC forte entre conteneurs : les données de Paperless appartiennent à un UID auquel le processus Immich n'a aucun droit.
> 3. **Réseaux internes par service** (déjà en place, cf. [[Self-hosting sécurisé avec Podman]] et [[Reverse proxy Caddy avec TLS automatique]]) : un conteneur compromis ne peut de toute façon pas atteindre les autres sur le réseau.
>
> La combinaison profil-par-chemin + UID-distinct + réseau-isolé reconstitue, à la main, ce que MCS offrait gratuitement. C'est plus de configuration, mais le résultat est solide.

C'est cohérent avec la philosophie de toute la série : quand une couche est plus faible, on s'appuie davantage sur les autres. Ici, l'absence de MCS se compense par un usage plus rigoureux des namespaces et du réseau.

---

## Les frictions réelles

> [!failure] Les points de douleur, honnêtement
> - **L'absence de MCS demande de la rigueur manuelle.** Sous SELinux, l'isolation inter-conteneurs était gratuite. Sous AppArmor, je dois activement séparer les UID et écrire des profils par service pour atteindre un niveau comparable. C'est du travail en plus, facile à négliger.
> - **Les guides Podman supposent souvent SELinux.** Beaucoup de doc parle de `:Z`, de `container_t`, de `setsebool` — inapplicable tel quel sous AppArmor. Il faut traduire mentalement, et reconnaître ce qui est silencieusement ignoré (`:Z`) de ce qui plante.
> - **Le confinement par chemin et les montages dynamiques.** Si un service écrit dans des chemins variables, le profil par chemin devient pénible à maintenir (`aa-logprof` aide, mais il faut repasser dessus). C'est moins robuste que des labels qui suivent le fichier.
> - **`complain` oublié en `complain`.** Facile de mettre un profil en mode apprentissage et d'oublier de le repasser en `enforce` — il ne protège alors plus rien tout en semblant chargé. Vérifier régulièrement `aa-status`.

Cela dit, au quotidien, AppArmor est plus reposant que SELinux sur un point précis : **plus de saga des labels de volumes**. La restauration de sauvegarde ne casse pas les contextes (pas de `restorecon` à prévoir, cf. [[Stratégie de sauvegarde restic 3-2-1]]), et le débogage est plus direct. L'arbitrage global dépend de ce que tu valorises : isolation inter-conteneurs forte par défaut (SELinux) ou simplicité opérationnelle (AppArmor).

---

## Conclusion : même but, compromis différent

Trois principes que je retire de cette variante :

1. **AppArmor offre le MAC, avec un modèle plus simple mais plus faible sur l'isolation inter-conteneurs.** Path-based vs label-based n'est pas qu'un détail : l'absence de MCS est la vraie conséquence à assumer sous Debian/Ubuntu.
2. **La simplicité opérationnelle est réelle.** Pas de relabel de volumes, pas de `:Z`, pas de contexte perdu à la restauration, un débogage plus direct. Pour qui ne veut pas vivre avec les subtilités SELinux, c'est un confort tangible.
3. **Ce qu'une couche ne donne pas, les autres le compensent.** UID distincts, profils par chemin, réseaux isolés : la philosophie de défense en profondeur de toute la série reste la même. AppArmor s'y insère, simplement avec un point d'appui différent.

> [!note] La série complète dans le vault
> - [[Self-hosting sécurisé avec Podman]]
> - [[Reverse proxy Caddy avec TLS automatique]]
> - [[Configuration WireGuard self-hosted]]
> - [[Durcissement SELinux pour conteneurs]] (variante Fedora/RHEL de cet article)
> - [[Stratégie de sauvegarde restic 3-2-1]]

---

*Retour d'expérience personnel sur système AppArmor (Debian/Ubuntu). Si ton serveur tourne sous Fedora/RHEL, lis plutôt la variante SELinux — le modèle de confinement y est plus fort sur l'isolation inter-conteneurs, au prix d'une opération plus exigeante. Les profils et chemins ci-dessus sont des exemples à transposer à ton parc.*
