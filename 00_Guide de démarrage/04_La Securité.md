---
author: "0x41647269656E"
series: Guide de démarrage
title: Security
tags:
  - security
reading-time: 30min
---
# La sécurité dans un homelab

La sécurité informatique est devenue un enjeu central dans tout système connecté, qu’il s’agisse d’une infrastructure d’entreprise, de quelques périphériques IoT ou d’un homelab complet exposé tout ou partiellement à Internet. Elle repose sur un ensemble de principes, de bonnes pratiques et de solutions dédiées qui permettent de protéger les données, les services et les utilisateurs contre les menaces. Le domaine de la cybersécurité est vaste, en évolution constante et peut s'avérer très couteux pour se doter de solutions à la pointe. Il faut pour un homelab trouver un équilibre.

Trouver le bon équilibre entre sécurité et innovation consiste à protéger suffisamment ses systèmes pour que l’effort nécessaire à les compromettre dépasse largement la valeur des données protégées, tout en conservant la souplesse nécessaire pour expérimenter, apprendre et faire évoluer son infrastructure.

Dans les articles que je publierai sur cette série, je prends la posture d'un informaticien qui, sans être expert cybersécurité, souhaite mettre en place le _bare minimum requirements_ pour exposer des services sur internet afin de pouvoir profiter de son homelab partout et pouvoir partager l'accès à ses services à sa famille et ses amis.

On distingue plusieurs grands principes qu'on tentera de mettre en oeuvre dans le cadre d'un homelab : 
- La compartimentation des applications : une application ne doit pas accéder aux données d'une autre.
- Le hardening : réduction de la surface d'attaque
- Application des recommandations CNIL / NIST (CSF 2.0)
- Mises à jour et patching régulier
- Utilisation systématique d'authentification ou de flux VPN privés pour l'accès distant aux applications

Les modèles récents, comme le **Zero Trust**, viennent renforcer cette approche en partant du principe qu'aucune machine, aucun utilisateur et aucun service ne doit être implicitement considéré comme fiable. Chaque accès doit être validé, authentifié, contextualisé et limité strictement à ce qui est nécessaire. Cette philosophie, même appliquée à petite échelle, contribue fortement à réduire les risques liés aux erreurs humaines, aux fuites de données ou aux accès non autorisés.

Enfin, il est indispensable d’intégrer dans sa démarche une dimension **opérationnelle** : sauvegardes régulières des configurations et des données sensibles (photos de famille, mots de passes, tout ce qui n'est pas sorti des studios d'hollywood et qu'on ne pourra jamais récupérer.), supervision des services, tests périodiques de restauration et surveillance du comportement global de la plateforme. Un système bien protégé mais mal surveillé reste vulnérable, tout comme un système bien surveillé mais jamais mis à jour. C’est la complémentarité de ces bonnes pratiques qui assure la résilience d’un homelab. Pensez-y si vous prévoyez d'exposer ces services sur internet.

Le but est donc de garder une vision pragmatique de la sécurité appliquée à un environnement personnel : suffisamment structurée pour être efficace, mais suffisamment légère pour rester accessible et ne pas y passer 20h par semaine. C'est pourquoi nous nous intéressons particulièrement à la conteneurisation et à l'utilisation d'outils d'automatisation qui vont nous permettre de casser et reconstruire rapidement.

---
## Hardening

Le hardening consiste à réduire la surface d’attaque d’un système en éliminant tout ce qui n’est pas strictement nécessaire à son fonctionnement : services inutiles, ports ouverts par défaut, configurations faibles, permissions trop larges ou comportements non contrôlés. L’objectif n’est pas seulement de se protéger contre une intrusion externe, mais aussi de limiter les risques internes, les erreurs de configuration et les effets indésirables d’une mauvaise isolation entre services.

Ces principes s’inscrivent dans une logique plus large inspirée des cadres de référence modernes tels que le **NIST Cybersecurity Framework** ou les **CIS Controls**, qui rappellent l’importance d’identifier ses actifs, de comprendre les risques associés à chaque service et d’adopter une posture proactive face aux vulnérabilités. Même dans un environnement personnel, l’application de ces standards permet de structurer la démarche et de gagner en maturité sans complexité excessive.

### Pourquoi minimiser une installation serveur ?

Dans le contexte d’un homelab, il est tentant d’installer un système complet pour disposer immédiatement de toutes les fonctionnalités. Cependant, chaque composant présent sur un système, qu’il soit utilisé ou non, représente une potentielle source de vulnérabilités. Réduire l’installation d’un serveur à son strict nécessaire permet donc de diminuer considérablement la surface d’attaque.

Un système minimal limite le nombre de packages installés, de services actifs et de comptes utilisateurs pré-configurés. En cas de faille dans l’un de ces éléments, l’impact est mécaniquement réduit. De plus, un système épuré améliore la lisibilité opérationnelle : tout ce qui est présent et actif sert réellement à quelque chose. Cela facilite la maintenance, le diagnostic et le suivi de sécurité au quotidien.

Enfin, un système léger réduit les ressources consommées (RAM, stockage, temps de boot) et se montre plus prévisible. Dans un homelab où l’on souhaite tester, apprendre et parfois automatiser, cette stabilité est un avantage majeur.

### Réduction de la surface d’attaque : principes généraux

La réduction de la surface d’attaque repose sur quelques principes simples :

#### 1. Supprimer ce qui est inutile
Tout package ou service qui ne contribue pas directement à la fonction du serveur doit être retiré. Moins de logiciels signifie moins de vecteurs d’attaque potentiels et moins de correctifs à appliquer.

#### 2. Désactiver les services non essentiels
Un service actif, même s'il n’est jamais utilisé, peut exposer des ports réseau, créer des entrées dans les logs ou introduire des dépendances non souhaitées. Une désactivation systématique permet de clarifier le rôle de la machine.

#### 3. Nettoyer les utilisateurs et groupes par défaut
Certains comptes systèmes sont nécessaires, mais d’autres sont parfois installés par défaut pour faciliter certaines fonctionnalités non requises. Un nettoyage soigneux limite les possibilités d’escalade latérale.

#### 4. Durcir les accès distants
Un accès root à distance est un risque significatif : il supprime toute étape supplémentaire d’authentification ou de contrôle d’intégrité. L’idée est de réserver ce compte à une utilisation locale. Dans le cas d'un homelab, on peut considérer que les opérations de maintenance (mises à jour, changements de configuration, patching de sécurité...) sont réalisées depuis le réseau domestique local.

#### 5. Adopter un principe du _least privilege_
Chaque utilisateur ou service ne doit disposer que des droits strictement nécessaires. Cela limite les actions possibles en cas de compromission.

---
## La compartimentation des applications

La compartimentation des applications consiste à isoler chaque service, application ou composant afin de limiter les interactions non nécessaires entre eux. Ce principe, central en cybersécurité, vise à empêcher qu'une compromission d'un élément puisse se propager à l’ensemble du système. En d’autres termes, même si un service est attaqué, l’impact reste contenu dans un périmètre strictement défini.

Dans le contexte d’un homelab, où de nombreuses applications cohabitent (serveurs médias, services domotiques, archivage de documents, outils de partage, bibliothèques de photos, vidéos, etc.), une isolation efficace réduit considérablement les risques de fuite de données ou d’escalade latérale.

Le silotage (ou compartimentation) consiste à séparer les applications sur des machines ou environnements distincts : serveurs dédiés, machines virtuelles, voire clusters de virtualisation. Cette approche offre une isolation forte : un service compromis n’a pas de visibilité directe sur ceux hébergés ailleurs.

Dans un homelab, pour isoler des services sensibles comme ceux contenant des données personnelles, on trouve :

- L’utilisation de machines virtuelles
- l’usage de différents hôtes physiques
- La mise en oeuvre de conteneurisation
- La segmentation de l'infrastructure réseau
- La mise en oeuvre de firewalls (applicatifs ou physiques)

Cette approche améliore la résilience globale, mais peut demander davantage de ressources matérielles (notamment la RAM) et de maintenance.

### Conteneurisation vs machines virtuelles : avantages et limites

La conteneurisation et la virtualisation répondent toutes deux à un besoin d’isolation, mais avec des approches et niveaux de sécurité différents. Dans un homelab, il est important de comprendre leurs forces, leurs faiblesses et leurs impacts sur la sécurité, la performance et la consommation de ressources.

#### Conteneurisation

Les pour et les contres : 
- ✅ Légèreté et rapidité : les conteneurs partagent le kernel de l’hôte, consomment peu de ressources et se déploient très rapidement.
- ✅ Reproductibilité et portabilité : les images permettent des déploiements identiques sur divers environnements.
- ✅ Isolation logique avancée : grâce aux namespaces, cgroups et capabilities, chaque application peut être fortement cloisonnée.
- ✅ Écosystème mature : Docker, Podman, Kubernetes, Nomad… facilitent l’orchestration et l’automatisation.
- ✅ Mode rootless : réduit les risques en exécutant les conteneurs sans privilèges administrateur sur l’hôte.
- 🚫 Partage du kernel : une faille dans le kernel affecte tous les conteneurs.
- 🚫 Élévation de privilèges : des attaques existent permettant à un conteneur vulnérable d’obtenir un accès root sur l’hôte.
- 🚫 Isolation limitée comparée à une VM : les protections restent principalement logicielles.
- 🚫 Mode rootless pas toujours compatible avec toutes les applications (notamment celles nécessitant des ports <1024 ou des capacités particulières).
- 🚫 Risque de mauvaise configuration : montages de volumes trop permissifs, network mode « host », conteneurs en privileged mode, etc.

#### Virtualisation

Les pour et les contres : 
- ✅ Isolation forte : chaque VM dispose de son propre kernel, ce qui crée une véritable barrière entre les environnements
- ✅ Confinement efficace : une compromission dans une VM n’impacte pas facilement les autres
- ✅ Compatibilité élevée : possibilité d’exécuter différents OS (Linux, BSD, Windows…)
- ✅ Support matériel (VT-x/AMD-V, IOMMU, passthrough) permettant d’aller jusqu’à isoler des périphériques entiers (GPU, Carte PCI Express, péripéhriques USB)
- ✅ Surface d’attaque plus prévisible : les hyperviseurs (ESXi, Proxmox, Xen…) sont conçus pour la sécurité
- 🚫 Consommation de ressources importante : chaque VM requiert CPU, RAM et stockage dédiés (et une quantité supplémentaire est consommée)
- 🚫 Temps de déploiement plus long : installation d’un OS complet, maintenance plus lourde
- 🚫 Complexité accrue de patching / MaJ si le nombre de VMs augmente
- 🚫 Overhead matériel non négligeable, surtout sur des homelabs modestes
- 🚫 Snapshots / backups volumineux et gestion parfois lourde

En pratique, un homelab mature peut combiner les deux approches :
- Un hyperviseur pour isoler les rôles critiques (Reverse Proxy, Web Application Firewall, Certs & Secrets Management, présentation du stockage)
- Des conteneurs pour la souplesse de gestion, le footprint léger et l’automatisation (services web, applications légères, monitoring, outils)

En combinant silotage physique, isolation logicielle et gestion stricte des droits, la compartimentation devient un pilier essentiel du hardening et de la sécurité globale d’un homelab moderne.

---
## Mises à jour et patching régulier

Les mises à jour sont l’un des piliers les plus sous-estimés, mais aussi l’un des plus efficaces pour maintenir un homelab sécurisé. La plupart des attaques automatisées ne ciblent pas des failles inconnues ( _aussi appelées zero-day_ ), mais des vulnérabilités publiques, souvent corrigées depuis longtemps.
### Pourquoi c’est critique ?

- Un service non mis à jour peut être scanné et compromis automatiquement en quelques minutes après son exposition.
- Un hyperviseur non patché représente un risque majeur si une vulnérabilité permet une évasion de VM.
- Des conteneurs basés sur des images anciennes peuvent embarquer des failles critiques.
### Bonnes pratiques

- Automatiser les mises à jour de sécurité lorsque c’est raisonnable (unattended-upgrades, Watchtower, Ansible).    
- Mettre à jour en priorité les services exposés à Internet : reverse proxy, portail SSO, VPN.
- Utiliser des images minimales et actualisées pour les conteneurs.
- Planifier un cycle régulier (hebdomadaire ou mensuel) pour passer en revue l’ensemble des services.
- Tester les mises à jour lorsque cela peut avoir un impact : bases de données, hyperviseur, services critiques.

L’objectif est de rester à jour sans y passer trop de temps, mais sans laisser des machines critiques accumuler des vulnérabilités.

---

## Utilisation systématique d’authentification ou de flux privés pour l’accès distant aux applications

Lorsque l’on souhaite accéder à son homelab depuis l’extérieur, il est indispensable de protéger chaque service avec une authentification solide ou, mieux encore, de ne pas les exposer du tout sans passer par une solution type VPN, overlay, zero trust, proxy...

### Pourquoi éviter l’exposition directe ?

- Internet est constamment scanné par des bots cherchant des ports ouverts.
- Dans un environnement professionnel, les applications sont constamment surveillées et mises à jour (quoi que...), mais sur un homelab, le rythme de mise à jour et les temps alloué à l'administration peuvent être plus faible qu'une équipe à temps complet. Certaines de vos applications seront inévitablement exposées à des failles connues.
- Un simple service non protégé (Jellyfin, une vieille interface admin, un panneau proxmox…) peut offrir un accès total à l'infrastructure.
- Une faille dans un service exposé peut immédiatement compromettre l’ensemble du réseau domestique de la personne chez qui le homelab est installé (Caméra wifi ? TV connectée ? Domotique ?).
### Approches recommandées

#### 1. Passer par un VPN privé (voir [[05_Gestion des accès distants]])

Solutions recommandées : WireGuard, Tailscale, OpenVPN.  
Avantages :
- Pas d’exposition directe des services internes.
- Tunnel chiffré.
- Authentification forte (clé, MFA, SSO).
- Simplicité de configuration et excellente sécurité.
#### 2. Imposer une authentification forte

Pour les services nécessitant d’être exposés (reverse proxy, ressources publiques) :
- Utiliser un SSO (Authentik, Authelia, Keycloak ou Gitlab (oui oui...)).
- Activer le MFA.
- Limiter les accès via un WAF ou un filtrage IP.
- Éviter de laisser les services gérer eux-mêmes le login/password (souvent faible).
- Éviter de laisser LES GENS gérer eux-mêmes leurs mots de passes...
#### 3. Segmenter et limiter les flux
- Ne pas mettre tous les services dans le même réseau.
- Isoler les applications critiques (NAS, backups, gestion des secrets).
- Ne pas exposer les dashboards, panels admin ou monitoring au travers d'internet
### Architecture typique conseillée
- Reverse proxy exposé avec MFA obligatoire.
- Services utilisateurs accessibles **après authentification SSO**.
- Services internes (base de données, stockage, monitoring) accessibles uniquement via VPN
- Segmentation réseau stricte entre zones publiques, privées et administratives.