# Homarr, Heimdall, Flame et Homelable : quel outil choisir pour son homelab ?

Organiser l'accès à ses services autohebergés est un probleme classique pour toute personne qui gere un homelab ou un serveur personnel. Plusieurs projets open source repondent a ce besoin, mais avec des philosophies tres differentes. Cet article compare quatre d'entre eux : Homarr, Heimdall, Flame et Homelable.

Il faut noter des le depart que les trois premiers appartiennent a la meme categorie (dashboards/launchers d'applications), alors que Homelable repond a un besoin different : la visualisation de la topologie du reseau plutot que le simple lancement d'applications.

## Tableau comparatif rapide

| Critere | Homarr | Heimdall | Flame | Homelable |
|---|---|---|---|---|
| Categorie | Dashboard moderne | Dashboard/launcher classique | Startpage legere | Visualiseur d'infrastructure |
| Stack technique | Application moderne avec authentification integree (credentials, OIDC, LDAP) | PHP/Laravel | Node.js + Express, React, TypeScript, SQLite | Backend Python, frontend TypeScript |
| Etoiles GitHub | (projet distinct, voir homarr-labs/homarr) | environ 9,2k | environ 6,4k | environ 2k |
| Licence | Open source | MIT | MIT | MIT |
| Interface | Glisser deposer, sans YAML/JSON | Editeur web classique | Editeurs integres (apps et bookmarks) | Canvas interactif type diagramme reseau |
| Integrations | Tres nombreuses (Plex, Jellyfin, Home Assistant, les Arrs, Docker, etc.) | Applications reconnues automatiquement par nom (foundation apps et enhanced apps) | Integration Docker et Kubernetes par labels/annotations | Scanner reseau (nmap), Zigbee2MQTT, widget pour gethomepage, serveur MCP |
| Authentification | Credentials, OIDC, LDAP, gestion fine des permissions | Non native (protection via reverse proxy) | Systeme d'authentification simple avec mot de passe | Authentification par mot de passe pour l'edition, mode lecture seule public optionnel |
| Langues | 26 langues | Une vingtaine de langues (qualite variable) | Non precise dans la documentation | Non precise dans la documentation |
| Public cible | Utilisateurs cherchant un dashboard riche et scalable, y compris en multi utilisateurs | Utilisateurs cherchant un launcher simple et eprouve | Utilisateurs cherchant une startpage legere avec editeurs integres | Utilisateurs voulant cartographier et surveiller physiquement leur infrastructure |

## Homarr

Homarr se positionne comme un dashboard moderne, pense pour etre a la fois simple d'usage et capable de monter en charge. L'accroche du projet est claire : offrir un dashboard puissant sans configuration YAML ou JSON, grace a un systeme de glisser deposer.

Points forts observes sur le site officiel :

- Systeme de glisser deposer pour organiser le tableau de bord, sans fichier de configuration a editer.
- Plus de 10 000 icones disponibles via plusieurs depots communautaires.
- Authentification complete (identifiants, OIDC, LDAP) avec gestion des permissions, ce qui le distingue nettement de Heimdall et Flame.
- Systeme de taches en arriere plan concu pour bien fonctionner meme avec de nombreux utilisateurs.
- 26 langues disponibles grace a un programme de traduction communautaire.
- Tres large catalogue d'integrations : Plex, Jellyfin, Home Assistant, la suite des Arrs (Sonarr, Radarr, Readarr, Prowlarr), qBittorrent, AdGuard Home, et bien d'autres.

Homarr convient particulierement aux personnes qui veulent un dashboard complet, multi utilisateurs, avec une gestion des droits serieuse.

## Heimdall

Heimdall est l'un des projets les plus anciens et les plus connus dans cette categorie, maintenu par l'equipe LinuxServer.io. Il s'agit d'une application PHP/Laravel dont l'objectif est simple : regrouper tous les liens vers vos applications web sur une seule page, avec la possibilite de l'utiliser comme page de demarrage de navigateur.

Caracteristiques principales :

- Deux types d'applications supportees : les "foundation apps" (icone et couleur remplies automatiquement) et les "enhanced apps" (affichage de statistiques en direct via l'API de l'application, par exemple la vitesse de telechargement pour NZBGet ou SABnzbd).
- Barre de recherche integree avec choix entre Google, Bing ou DuckDuckGo.
- Installation via Docker multi architecture (x86-64, armhf, arm64) ou installation manuelle via PHP/Laravel.
- Pas de systeme d'authentification natif : la protection se fait generalement via un reverse proxy avec authentification basique (documentation fournie pour Nginx et le proxy SWAG de LinuxServer.io).
- Traduction disponible dans une vingtaine de langues, avec une qualite qui varie selon les contributions communautaires.
- Projet mature avec plus de 1 200 commits et une base d'utilisateurs etablie de longue date.

Heimdall reste un choix solide pour qui cherche un launcher simple, stable et bien documente, sans avoir besoin d'un systeme d'authentification integre.

## Flame

Flame se presente comme une startpage autohebergee, avec un accent particulier sur la simplicite d'edition directement depuis l'interface, sans avoir a modifier de fichiers.

Fonctionnalites cles :

- Creation, modification et suppression des applications et des favoris directement via des editeurs integres dans l'interface graphique.
- Epinglage des elements favoris sur l'ecran d'accueil pour un acces rapide.
- Barre de recherche avec filtrage local, onze fournisseurs de recherche web integres, et possibilite d'en ajouter d'autres.
- Systeme d'authentification simple pour proteger les parametres, applications et favoris.
- Widget meteo avec temperature actuelle, couverture nuageuse et animation selon la meteo.
- Integration Docker native : les conteneurs portant certains labels sont automatiquement detectes et ajoutes au dashboard.
- Integration Kubernetes via annotations sur les ingress.
- Import de favoris HTML depuis un navigateur (fonctionnalite experimentale).
- Stack technique moderne : backend Node.js/Express avec Sequelize et SQLite, frontend React/Redux/TypeScript.

Flame se distingue par sa legerete et par l'integration Docker/Kubernetes native, ce qui en fait un bon choix pour les utilisateurs qui gerent deja leurs services via des labels de conteneurs.

## Homelable

Homelable sort du cadre strict du "dashboard d'applications" pour proposer autre chose : un visualiseur d'infrastructure homelab sous forme de diagramme reseau interactif, avec surveillance de l'etat en direct des services.

Ce que propose le projet :

- Un scanner reseau base sur nmap qui detecte automatiquement les machines et services presents sur les plages CIDR configurees, avant de les proposer dans une file d'attente d'appareils en attente de validation.
- Un systeme de verification de sante des noeuds avec plusieurs methodes possibles : ping, HTTP, HTTPS, connexion TCP, verification SSH, endpoint Prometheus ou endpoint /health personnalise.
- Une integration avec Zigbee2MQTT pour importer automatiquement la topologie d'un reseau Zigbee (coordinateur, routeurs, appareils terminaux) directement sur le canvas.
- Un mode "Live View" permettant de partager une vue en lecture seule du canvas, sans authentification, activable via une cle secrete.
- Un widget de statistiques compatible avec gethomepage via une API JSON dediee.
- Un serveur MCP (Model Context Protocol) optionnel, qui permet a des clients IA compatibles (Claude Code, Claude Desktop) de lire la topologie du homelab et d'agir dessus : lister les noeuds, ajouter ou modifier des elements, declencher un scan reseau.
- Une version dediee pour Home Assistant, disponible separement via HACS.

Homelable ne remplace pas un dashboard d'applications comme Homarr, Heimdall ou Flame. Il s'agit plutot d'un complement, utile pour cartographier physiquement et visuellement son infrastructure, avec un vrai interet pour les homelabs comportant du materiel reseau et des appareils IoT/Zigbee.

## Comment choisir ?

- **Vous voulez un dashboard moderne, multi utilisateurs, avec authentification avancee et de tres nombreuses integrations** : Homarr est le choix le plus complet.
- **Vous voulez un launcher simple, eprouve depuis des annees, avec peu de configuration** : Heimdall reste une valeur sure, surtout si vous avez deja un reverse proxy pour gerer l'authentification.
- **Vous voulez une startpage legere avec editeurs integres et une bonne integration Docker/Kubernetes** : Flame est adapte, particulierement pour les infrastructures containerisees.
- **Vous voulez cartographier visuellement votre reseau, surveiller l'etat de vos services et importer votre topologie Zigbee** : Homelable repond a un besoin different et complementaire des trois autres outils.

Dans la pratique, rien n'empeche de combiner un dashboard classique (Homarr, Heimdall ou Flame) avec Homelable pour la partie visualisation et surveillance d'infrastructure, les deux usages n'etant pas mutuellement exclusifs.
