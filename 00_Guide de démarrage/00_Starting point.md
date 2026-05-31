---
title: Quel homelab pour quel usage ?
author: "0x41647269656E"
series: Guide de démarrage
reading-time: 10min
---
# Bienvenue !

Bienvenue dans ce guide de démarrage de ce projet de documentation consacré aux homelabs !

Ce guide se veut être une ressource pour celles et ceux qui, comme moi, ont un jour voulu héberger des services à la maison et reprendre le contrôle de leurs données. Il présente différentes solutions techniques que j’ai étudiées, ainsi que les choix, parfois clivants, que j’ai réalisé. Ce guide n'a pas vocation à être une référence absolue en matière de homelabs mais simplement un retour d'expérience des solutions que j'ai évalué et mis en oeuvre.

Dans cette démarche, n’hésitez pas à consulter d’autres sources, à expérimenter différents outils et à explorer d’autres approches et pourquoi pas contribuer à ce guide ! Je partage ici ma démarche personnelle, les choix que j'ai réalisé, les outils que j'ai intégré. C'est là toute la richesse des homelabs, chacun se construit le sien avec ses besoins et ses contraintes.

Au sein de ce guide, j’essaie de mettre en avant les bonnes questions à se poser afin d’éviter certains écueils, qu’ils soient financiers, matériels, techniques ou liés à la sécurité et de permettre à chacun d’aborder son projet personnel avec une vision claire, en anticipant les dépenses et en évitant les mauvaises surprises sur des aspects auxquels on ne pense pas toujours au départ.

# Le pourquoi du comment d'un homelab

Après plusieurs années à utiliser des services "cloud", je me lance en 2023 dans l'auto-hébergement d'applications. Vous trouverez ci-dessous un dossier complet de mes recherches. Les choix que j'ai évalué, les choses qui ont marché mais aussi les choses qui m'ont paru trop complexe à implémenter, les échecs et les ratés.

Il était important pour moi de partager les échecs rencontrés en cours de route. En 2025, j’ai notamment perdu près de 40 To de données à la suite d’un problème d’alignement de partitions et d'une carte mère défectueuse.

Cet incident m’a rappelé à quel point certains aspects de design d'infrastructure, parfois mal compris, peuvent avoir un impact critique sur l’intégrité du projet. C’est aussi pour cela que ce guide ne se limite pas aux réussites : il met en lumière les pièges concrets auxquels on peut être confronté, afin de vous aider vous à les anticiper et les éviter.

# Quel homelab pour quel usage ?

*A quoi ça sert d'héberger soi-même ses applications ?*

Plusieurs raisons à ça :

- La première, c'est l'aspect financier. Avec l'accès grandissants aux services en ligne, on multiplie les abonnements aux services. Netflix, Prime Vidéo, HBO, Disney+, Spotify, Deezer, Google One, Youtube Premium, Dropbox, ICloud et j'en passe... Au final, ces services finissent par coûter cher. 100 euros de services par mois c'est 12.000 euros en dix ans. Ca calme.

- Si j'arrête de payer, qu'est ce qui se passe ? Je perd l'accès à ma bibliothèque que j'avais soigneusement pris le temps de sélectionner, trier, classer, marquer, annoter, organisant dossiers, albums, playlists, bookmarks et reviews pendant que ces plateformes conservent mon sentiment et mon profil d'utilisateur.

- Bien que tous ces services soient payants, vous restez le produit. Vos données sont utilisées par ces géants américains pour alimenter de grandes bases de données permettant à ces entreprises de proposer des produits et services toujours plus proches de vos attentes, normalisant ainsi l'offre. L'innovation n'est plus au centre des préoccupations des auteurs. Ceux-ci étant contrôlés par des productions toujours plus intrusives dans le process créatif cherchant à maximiser le rendement financier d'un film ou d'une série..

- Le rejet des algorithmes de recommandations : La raison pour laquelle les entreprises comme Netflix ou Youtube ont réussi a conquérir les foules, c'est la profondeur de leur catalogue. Des millions de contenus disponibles à porté de clic. Mais ce qui fait la force de ces entreprises, ce n'est pas uniquement la quantité de médias mais la réponse à la question : "Qu'est ce qu'on regarde ce soir ?". Vous avez regardé cette vidéo, vous allez forcément aimer ce contenu similaire. C'est agréable lorsqu'on souhaite découvrir un univers mais aussi limitant lorsqu'on sait que ceux-ci prennent en compte des données très personnelles : opinions politiques, humeur, localisation, heure de la journée, contenus passés, habitudes de consommations, publicités... Les vidéos, films et séries défilent et finissent par se ressembler. Un seul objectif pour eux : vous faire consommer toujours plus. Que ce soit qualitatif ou non.

- L'envie d'apprendre : faire des recherches, apprendre et découvrir une nouvelle communauté me fait envie. Comprendre les ressorts techniques et les ressources utilisées pour rendre un service. Héberger soi-même des services est un projet à part entière duquel je tirerai forcément de nouvelles connaissances.

- En 2016, à l'occasion du World Economic Forum, **Ida Auken** prononce une phrase : **_"Welcome to 2030, I own nothing, have no privacy and life has never been better."_**. C'est une caricature du monde vers lequel nous allons : on monde où l'on ne possède plus vraiment les objets, les oeuvres et les outils que l'on utilise.
	- L'appartement ou la maison est louée
	- La voiture est en leasing ou vous utilisez les services Uber
	- Les contenus multimédias sont derrière des abonnements (Spotify, Deezer, Netflix, Amazon Prime, Disney+, XBOX Game Pass, Playstation Plus, Youtube Premium)
	- Vos jeux sont exécutés dans le cloud sur des infrastructures qui ne vous appartiennent pas : GeForce Now, Stadia, Xbox Cloud Gaming).
	- Les logiciels sont en SaaS (Office 360, Photoshop, Canva Pro).
	- Vos fichiers sont stockés dans Dropbox ou Google Drive.
	- Vos souvenirs photos et vidéos sont hébergées dans iCloud.

Même lorsqu'on achète un média sans passer par un abonnement, comme des musiques sur iTunes, des jeux sur Steam, des films en VOD sur Orange ou Youtube (si si, ils vendent aussi des films), il vous est accordé une licence personnelle non transférable et révocable. **Acheter ne veut pas dire posséder.** L'état de californie a récemment fait voter une loi en ce sens pour forcer les entreprises à inscrire dans leurs conditions générales d'utilisation ces précisions.

Votre père vous a peut-être légué sa montre, sa collection de vinyls ou son appareil-photo. Ce que vous achetez en ligne, vous ne pourrez pas le transmettre à votre mort, le contrat de licence que vous avez acheté vous l'interdit.

Héberger soi-même, posséder ses fichiers, ses services et son infrastructure, ça vient en réponse, presque de manière militante à l'ensemble de ces problématiques.

# Glossaire

Nous parlons dans ce guide de NAS et de Homelabs. Je distingue les deux pour des cas d'usages différents. A mon sens, un homelab est un environnement dans lequel on héberge des services que l'on souhaite mettre à l'épreuve (lab) et qui sert à héberger des services. Un NAS quand a lui sert à héberger des services simples et bas niveau de partage de fichiers, le stockage long-terme de sauvegardes et l'hébergement d'applications légères.

Aujourd'hui, les entreprises comme Synology ou QNAP tendent à réduire ce gap. Mais les puissances de calcul de ces bestioles sont encore loin des processeurs grands publics classiques dédiés à des ordinateurs de bureau ou des serveurs.