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

## La compartimentation des applications
### Silotage physique
### Reduction de la visibilité de l'OS

### Droits d'accès des applications

## Application des recommandations CNIL / NIST (CSF 2.0)
## Mises à jour et patching régulier

## Utilisation systématique d'authentification ou de flux VPN privés pour l'accès distant aux applications