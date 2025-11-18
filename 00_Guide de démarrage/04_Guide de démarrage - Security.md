---
author: "0x41647269656E"
series: Guide de démarrage
title: Security
tags:
  - security
---
# La sécurité dans un homelab

La sécurité informatique est devenue un enjeu central dans tout système connecté, qu’il s’agisse d’une infrastructure d’entreprise, de quelques périphériques IoT ou d’un homelab complet exposé tout ou partiellement à Internet. Elle repose sur un ensemble de principes, de bonnes pratiques et de solutions dédiées qui permettent de protéger les données, les services et les utilisateurs contre les menaces. Le domaine de la cybersécurité est vaste, en évolution constante et peut s'avérer très couteux pour se doter de solutions à la pointe. Il faut pour un homelab trouver un équilibre.

Trouver le bon équilibre entre sécurité et innovation consiste à protéger suffisamment ses systèmes pour que l’effort nécessaire à les compromettre dépasse largement la valeur des données protégées, tout en conservant la souplesse nécessaire pour expérimenter, apprendre et faire évoluer son infrastructure.

Dans les articles que je publierai sur cette série, je prends la posture d'un informaticien qui, sans être expert cybersécurité, souhaite mettre en place le _bare minimum requirements_ pour exposer des services sur internet afin de pouvoir profiter de son homelab partout et pouvoir partager l'accès à ses services à sa famille et ses amis.

On distingue plusieurs grands principes qu'on tentera de mettre en oeuvre dans le cadre d'un homelab : 
- La compartimentation des applications : une application ne doit pas accéder aux données d'une autre.
- Le hardening : réduction de la surface d'attaque
- Principe de least privilege : ne donner que les droits nécessaires
- Application des recommandations CNIL / NIST (CSF 2.0)
- Mises à jour et patching régulier
- Utilisation systématique d'authentification ou de flux VPN privés pour l'accès distant aux applications

Les modèles récents, comme le **Zero Trust**, viennent renforcer cette approche en partant du principe qu'aucune machine, aucun utilisateur et aucun service ne doit être implicitement considéré comme fiable. Chaque accès doit être validé, authentifié, contextualisé et limité strictement à ce qui est nécessaire. Cette philosophie, même appliquée à petite échelle, contribue fortement à réduire les risques liés aux erreurs humaines, aux fuites de données ou aux accès non autorisés.

Enfin, il est indispensable d’intégrer dans sa démarche une dimension **opérationnelle** : sauvegardes régulières des configurations et des données sensibles (photos de famille, mots de passes, tout ce qui n'est pas sorti des studios d'hollywood et qu'on ne pourra jamais récupérer.), supervision des services, tests périodiques de restauration et surveillance du comportement global de la plateforme. Un système bien protégé mais mal surveillé reste vulnérable, tout comme un système bien surveillé mais jamais mis à jour. C’est la complémentarité de ces bonnes pratiques qui assure la résilience d’un homelab. Pensez-y si vous prévoyez d'exposer ces services sur internet.

Le but est donc de garder une vision pragmatique de la sécurité appliquée à un environnement personnel : suffisamment structurée pour être efficace, mais suffisamment légère pour rester accessible et ne pas y passer 20h par semaine. C'est pourquoi nous nous intéressons particulièrement à la conteneurisation et à l'utilisation d'outils d'automatisation qui vont nous permettre de casser et reconstruire rapidement.