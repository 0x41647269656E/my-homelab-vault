---
title: Quel homelab pour quel usage ?
author: "0x41647269656E"
series: Guide de démarrage
reading-time: 10min
---
# Bienvenue !

Bienvenue dans ce guide de démarrage de ce projet de documentation consacré aux homelabs !

Ce guide se veut être une ressource pour celles et ceux qui, comme moi, ont un jour voulu héberger des services à la maison et reprendre le contrôle de leurs données.

Ce guide présente les différentes solutions techniques que j’ai étudiées, ainsi que les choix, parfois clivants, que j’ai faits. Ce guide n'a pas vocation à être une référence absolue en matière de homelabs mais simplement un retour d'expérience des solutions que j'ai évalué et mis en oeuvre.

Dans cette démarche, n’hésitez pas à consulter d’autres sources, à expérimenter différents outils et à explorer d’autres approches et pourquoi pas contribuer à ce guide ! Je partage ici ma démarche personnelle, les choix que j'ai réalisé, les outils que j'ai intégré. C'est là toute la richesse des homelabs, chacun se construit le sien avec ses besoins et ses contraintes.

Au sein de ce guide, j’essaie de mettre en avant les bonnes questions à se poser afin d’éviter certains écueils, qu’ils soient financiers, matériels, techniques ou liés à la sécurité, et de permettre à chacun d’aborder son projet personnel avec une vision claire, en anticipant les dépenses et en évitant les mauvaises surprises sur des aspects auxquels on ne pense pas toujours au départ.

# Le pourquoi du comment d'un homelab

Après plusieurs années à utiliser des services "cloud", je me lance en 2023 dans l'auto-hébergement d'applications. Vous trouverez ci-dessous un dossier complet de mes recherches. Les choix que j'ai évalué, les choses qui ont marché mais aussi les choses qui m'ont paru trop complexe à implémenter, les échecs et les ratés.

Il était important pour moi de partager les échecs rencontrés en cours de route. En 2025, j’ai notamment perdu près de 40 To de données à la suite d’un problème d’alignement de partitions et d'une carte mère défectueuse.

Cet incident m’a rappelé à quel point certains aspects de design d'infrastructure, parfois mal compris, peuvent avoir un impact critique sur l’intégrité du projet. C’est aussi pour cela que ce guide ne se limite pas aux réussites : il met en lumière les pièges concrets auxquels on peut être confronté, afin de vous aider vous à les anticiper et les éviter.

# Quel homelab pour quel usage ?

*A quoi ça sert d'héberger soi-même ses applications ?*

Plusieurs raisons à ça :

- La première, c'est l'aspect financier. Avec l'accès grandissants aux services en ligne, on multiplie les abonnements aux services. Netflix, Prime Vidéo, HBO, Disney+, Spotify, Deezer, Google One, Youtube Premium, Dropbox, ICloud et j'en passe... Au final, ces services finissent par coûter cher.

- Si j'arrête de payer, qu'est ce qui se passe ? Je perd l'accès à ma bibliothèque que j'avais soigneusement pris le temps de sélectionner, trier, classer, marquer, anoter, organisant dossiers, albums, playlists, bookmarks et reviews.

- Bien que tous ces services soient payants, vous restez le produit. Vos données sont utilisées par ces géants américains pour alimenter de grandes bases de données permettant à ces entreprises de proposer des produits et services toujours plus proches de vos attentes, normalisant ainsi l'offre. L'innovation n'est plus au centre des préoccupations des auteurs. Ceux-ci étant contrôlés par des productions toujours plus intrusives dans le process créatif cherchant à maximiser le rendement financier d'un film ou d'une série..

- Le rejet des algorithmes de recommandations : La raison pour laquelle les entreprises comme Netflix ou Youtube ont réussi a conquérir les foules, c'est la profondeur de leur catalogue. Des millions de contenus disponibles à porté de clic. Mais ce qui fait la force de ces entreprises, ce n'est pas uniquement la quantité de médias mais la réponse à la question : "Qu'est ce qu'on regarde ce soir ?". Vous avez regardé cette vidéo, vous allez forcément aimer ce contenu similaire. C'est agréable lorsqu'on souhaite découvrir un univers mais aussi limitant lorsqu'on sait que ceux-ci prennent en compte des données très personnelles : opinions politiques, humeur, localisation, heure de la journée, contenus passés, habitudes de consommations, publicités...Les vidéos, films et séries défilent et finissent par se ressembler. Un seul objectif pour eux : vous faire consommer toujours plus. Que ce soit qualitatif ou non.

- La reprise de contrôle technique : Utiliser un service sans en connaitre son fonctionnement interne me dérange. L'utilisation d'un logiciel propriétaire est donc à éviter. Vive le libre. Vive l'open source. ;)

- L'envie d'apprendre : faire des recherches, apprendre et découvrir une nouvelle communauté me fait envie. Héberger soi-même des services est un projet à part entière duquel je tirerai forcément de nouvelles connaissances.

# Glossaire

Nous parlons dans ce guide de NAS et de Homelabs. Je distingue les deux pour des cas d'usages différents. A mon sens, un homelab est un environnement dans lequel on héberge des services que l'on souhaite mettre à l'épreuve (lab) et qui sert à héberger des services. Un NAS quand a lui sert à héberger des services simples et bas niveau de partage de fichiers et d'applications légères.