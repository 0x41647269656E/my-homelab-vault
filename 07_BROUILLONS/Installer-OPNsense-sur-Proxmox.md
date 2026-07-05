---
title: "Installer OPNsense sur Proxmox : pare-feu segmenté, de l'ISO à Suricata"
author: "0x41647269656E"
series: "Hardening"
tags:
  - opnsense
  - proxmox
  - firewall
  - suricata
  - ids
  - virtualisation
  - reseau
  - hardening
reading-time: 30m
date: 05-07-2026
last_modified: 05-07-2026
status: draft
---

# Installer OPNsense sur Proxmox : pare-feu segmenté, de l'ISO à Suricata

> [!abstract] TL;DR
> Mise en œuvre complète de l'architecture définie dans [[Pare-feu-OPNsense-concepts-et-architecture]] : création des bridges internes Proxmox (`vmbr1` zone SERVICES, `vmbr2` zone DMZ), VM OPNsense (2 vCPU, 4 Go, VirtIO), installation, configuration initiale durcie (HTTPS d'admin restreint, utilisateur dédié, mises à jour), traduction de la matrice de flux en alias et règles *deny-by-default*, activation de Suricata en IDS puis en IPS avec les règles ET Open, sauvegardes, et tests de validation zone par zone.

> [!info] Note de cohérence
> Second volet du diptyque OPNsense. Le *pourquoi* (zones, matrice de flux, référentiels, limites) est dans [[Pare-feu-OPNsense-concepts-et-architecture]] — cet article suppose l'architecture acquise et déroule le *comment*. L'hyperviseur est le Proxmox VE installé dans [[Proxmox-installation-securite]]. Versions de référence : Proxmox VE 9.x, OPNsense 25.x — les chemins d'interface peuvent bouger légèrement d'une version à l'autre, la logique non.

---

## Rappel de la cible

```
[Box FAI] 192.168.1.1 ── LAN maison 192.168.1.0/24
   │
   └──(vmbr0)── [VM OPNsense]
                   WAN  vtnet0 : 192.168.1.2      (vmbr0)
                   LAN  vtnet1 : 10.0.10.1 ────── vmbr1 ── zone SERVICES
                   OPT1 vtnet2 : 10.0.20.1 ────── vmbr2 ── zone DMZ
```

| Zone | Bridge | Réseau | Interface OPNsense |
|---|---|---|---|
| LAN maison (côté « extérieur ») | `vmbr0` | `192.168.1.0/24` | WAN (`vtnet0`) |
| SERVICES | `vmbr1` | `10.0.10.0/24` | LAN (`vtnet1`) |
| DMZ | `vmbr2` | `10.0.20.0/24` | OPT1 (`vtnet2`) |

---

## 1. Prérequis et préparation

### Ressources

Dimensionnement pour un pare-feu de homelab avec Suricata :

| Ressource | Valeur | Justification |
|---|---|---|
| vCPU | 2 (type `host`) | `pf` est peu gourmand ; Suricata profite d'un second cœur |
| RAM | 4 Go, **ballooning désactivé** | 1 Go suffit au pare-feu seul ; les règles ET Open chargées en mémoire par Suricata font grimper la note |
| Disque | 20 Go | Système + logs + jeux de règles, avec de la marge |
| NIC | 3 × VirtIO | Une par zone |

### Télécharger et vérifier l'ISO

Sur un poste de travail, depuis [opnsense.org/download](https://opnsense.org/download/) : image **dvd**, architecture **amd64**. L'image est compressée en `.bz2`.

```bash
# Vérifier l'empreinte AVANT décompression (les sommes publiées portent sur le .bz2)
sha256sum OPNsense-25.1-dvd-amd64.iso.bz2
# → comparer avec le fichier de checksums publié sur le miroir de téléchargement

bunzip2 OPNsense-25.1-dvd-amd64.iso.bz2
```

Puis téléverser l'ISO dans Proxmox : **Datacenter → pve → local → ISO Images → Upload**.

> [!tip] Vérifier l'intégrité, toujours
> Même réflexe que pour l'ISO Proxmox dans [[Proxmox-installation-securite]] : un pare-feu construit sur une image corrompue ou altérée protège surtout l'attaquant.

---

## 2. Le réseau Proxmox : bridges internes

### Créer vmbr1 et vmbr2

**Interface Proxmox → nœud `pve` → System → Network → Create → Linux Bridge** :

| Champ | vmbr1 | vmbr2 |
|---|---|---|
| Name | `vmbr1` | `vmbr2` |
| IPv4/CIDR | *(vide)* | *(vide)* |
| Bridge ports | *(vide)* | *(vide)* |
| Comment | `SERVICES 10.0.10.0/24` | `DMZ 10.0.20.0/24` |

Puis **Apply Configuration**.

Deux champs vides qui font toute la sécurité du montage :

- **Bridge ports vide** : aucune carte physique attachée. Le trafic de ces bridges n'existe que dans l'hyperviseur et ne peut sortir que par une VM connectée aux deux côtés — c'est-à-dire OPNsense, et uniquement elle.
- **IPv4 vide** : l'hôte Proxmox **n'a pas d'adresse** dans les zones. Il ne participe pas à ces réseaux, ne peut pas y être attaqué, et ne peut pas servir à les contourner. L'admin Proxmox continue de passer par le LAN maison, comme avant — c'est le point de vigilance « poule et œuf » de l'article précédent : l'accès à l'hyperviseur ne dépend jamais d'OPNsense.

### Pourquoi VirtIO pour les cartes réseau

VirtIO est l'interface réseau paravirtualisée de KVM : la VM sait qu'elle est virtualisée et parle directement à l'hyperviseur au lieu de dialoguer avec une fausse carte Intel émulée (E1000). Résultat : débit nettement supérieur et moins de CPU consommé. FreeBSD a un excellent pilote `vtnet`. C'est le choix par défaut évident — avec un piège, traité au chapitre 5.

---

## 3. Création de la VM

**Create VM**, puis onglet par onglet :

| Onglet | Champ | Valeur | Pourquoi |
|---|---|---|---|
| General | Name | `opnsense` | |
| OS | ISO image | `OPNsense-25.1-dvd-amd64.iso` | |
| OS | Guest OS Type | **Other** | FreeBSD n'est pas dans la liste ; « Other » convient |
| System | Machine / BIOS | défauts (`i440fx`, SeaBIOS) | Pas de besoin UEFI ici |
| System | Qemu Agent | ✅ coché | L'agent sera installé plus tard (chapitre 5) |
| Disks | Bus | **VirtIO SCSI single**, 20 Go | Performances ; `Discard` ✅ si le stockage est sur SSD |
| CPU | Sockets / Cores | 1 / 2, Type **host** | `host` expose les instructions AES-NI, utiles à Suricata et aux VPN |
| Memory | Memory | 4096, **Ballooning ❌** | Un pare-feu doit avoir sa RAM garantie ; FreeBSD gère mal le ballooning |
| Network | Bridge | `vmbr0`, Model **VirtIO**, Firewall ❌ | Le pare-feu Proxmox est redondant ici : OPNsense fait le travail |

Après création, **Hardware → Add → Network Device** deux fois :

- `net1` : bridge `vmbr1`, VirtIO, Firewall décoché ;
- `net2` : bridge `vmbr2`, VirtIO, Firewall décoché.

> [!warning] L'ordre des cartes compte
> OPNsense verra `net0` comme `vtnet0`, `net1` comme `vtnet1`, `net2` comme `vtnet2`. Créer les cartes dans l'ordre WAN → SERVICES → DMZ évite de jongler à l'assignation. En cas de doute, comparer les adresses MAC entre l'onglet Hardware de Proxmox et la console OPNsense.

Enfin, **Options** :

- **Start at boot : Yes** — un pare-feu qui ne démarre pas tout seul ne sert à rien après une coupure de courant ;
- **Start/Shutdown order : 1** — OPNsense doit être debout **avant** les VM des zones, sinon elles démarrent sans passerelle. Donner aux autres VM un ordre supérieur et, au besoin, un `up delay` de 30 s sur OPNsense.

---

## 4. Installation d'OPNsense

Démarrer la VM, ouvrir la console. L'ISO boote en mode *live* ; se connecter avec l'utilisateur **`installer`** (mot de passe `opnsense`) pour lancer l'installeur.

1. **Keymap** : `fr.kbd` si clavier AZERTY.
2. **Type d'installation** : **Install (ZFS)**, pool `stripe` sur l'unique disque (`da0`/`vtbd0`). ZFS sur un seul disque n'apporte pas de redondance, mais ses écritures transactionnelles (*copy-on-write*) rendent le système de fichiers robuste aux arrêts brutaux — appréciable pour une appliance qui doit survivre à une coupure secteur.
3. **Mot de passe root** : long, généré, stocké dans Vaultwarden.
4. Redémarrer ; pendant le reboot, **retirer l'ISO** (Hardware → CD/DVD → Do not use any media).

### Assignation des interfaces (console)

Au premier boot, OPNsense assigne parfois WAN/LAN tout seul, mais on vérifie. Connexion console en `root`, menu :

- **Option 1) Assign interfaces** : pas de VLAN, puis `WAN=vtnet0`, `LAN=vtnet1`, `OPT1=vtnet2`.
- **Option 2) Set interface IP address** → LAN : IP statique `10.0.10.1/24`, pas de DHCP (les serveurs des zones auront des IP statiques — un serveur qui change d'adresse casse des règles de pare-feu), pas d'IPv6 pour l'instant.

À ce stade, l'interface web tourne sur `https://10.0.10.1`… joignable uniquement depuis la zone SERVICES, qui est vide. Classique problème d'amorçage.

### Amorçage : atteindre l'interface web depuis le LAN maison

Par défaut, OPNsense bloque tout en entrée sur WAN — y compris nous. Le contournement propre et documenté : désactiver **temporairement** le filtre de paquets depuis la console.

1. Console → **Option 8) Shell** → `pfctl -d` (« packet filter disabled »).
2. Depuis le poste d'admin : `https://192.168.1.2` (accepter le certificat auto-signé pour l'instant).
3. Dérouler la configuration des chapitres suivants — **en commençant par la règle WAN d'administration** (chapitre 6), sans quoi le prochain `pfctl -e` ou reboot nous éjecte.
4. Réactiver le filtre : `pfctl -e` (un reboot le réactive aussi).

> [!warning] Fenêtre de vulnérabilité assumée
> Pendant `pfctl -d`, le pare-feu ne filtre plus rien. Acceptable quelques minutes sur un LAN maison, à ne jamais reproduire si l'interface WAN donnait sur Internet. Refermer dès que la règle d'admin est posée.

### Le wizard de configuration

`System → Wizard`, étape par étape :

- **General** : hostname `opnsense`, domain `home.arpa` (domaine réservé à cet usage par la RFC 8375), DNS `9.9.9.9` / `1.1.1.1` (ou le résolveur local plus tard), **Override DNS** décoché.
- **Time** : fuseau `Europe/Paris`.
- **WAN** : type **Static**, IP `192.168.1.2/24`, passerelle `192.168.1.1`. Et le piège du guide :

> [!warning] Décocher « Block private networks » sur le WAN
> Cette option, pensée pour un WAN raccordé à Internet, jette tout le trafic venant d'adresses RFC1918. Notre WAN **est** un réseau RFC1918 (`192.168.1.0/24`) : la laisser cochée bloque tout le LAN maison, y compris le poste d'admin. La décocher. « Block bogon networks » peut rester décochée aussi, même raison.

- **LAN** : déjà configuré (`10.0.10.1/24`).
- **Root password** : déjà posé à l'installation, ne pas le retaper en plus faible.

Sur la box FAI, **réserver l'IP `192.168.1.2`** pour la MAC de `net0` (ou l'exclure de la plage DHCP) : la passerelle des zones ne doit jamais changer d'adresse.

---

## 5. Configuration initiale durcie

Avant toute règle de flux, on solidifie l'appliance elle-même — contrôle 4 des CIS Controls, pour reprendre la grille de l'article précédent.

### Mises à jour d'abord

`System → Firmware → Status → Check for updates`, appliquer, redémarrer si demandé. L'ISO a toujours quelques semaines de retard sur les correctifs.

### Utilisateur d'administration dédié

`System → Access → Users → Add` : utilisateur nominatif (ex. `adrien-admin`), mot de passe fort, membre du groupe **admins**. Se reconnecter avec ce compte, puis réserver `root` à la console. Bonus : `System → Access → Users → [utilisateur] → OTP seed` pour activer le TOTP (2FA), avec `System → Settings → Administration → Authentication` sur « Local + Timebased One Time Password ».

### Interface web : HTTPS, port, écoute

`System → Settings → Administration` :

| Réglage | Valeur | Pourquoi |
|---|---|---|
| Protocol | HTTPS | Jamais d'admin en clair, même en interne |
| SSL Certificate | auto-signé par défaut (améliorable plus tard via une CA interne) | |
| HTTP Redirect | **Disable** ✅ | Ne rien écouter du tout sur le port 80 |
| Listen Interfaces | LAN, WAN | WAN est nécessaire pour l'admin depuis le poste dédié ; l'accès y sera verrouillé par la règle du chapitre 6 |
| Secure Shell | activé, **clés uniquement**, sur LAN | Récupération en cas de GUI cassée |

### Comprendre la règle anti-lockout

OPNsense installe d'office sur l'interface **LAN** une règle invisible qui autorise toujours l'accès à l'interface web et au SSH — l'*anti-lockout rule*. Elle empêche de scier la branche : impossible de s'enfermer dehors en éditant les règles LAN. Elle n'existe que sur LAN — c'est pour ça que l'accès admin par le WAN exige une règle explicite, et c'est très bien ainsi : cet accès-là doit être une décision, pas un défaut.

### Le piège VirtIO : l'offloading matériel

`Interfaces → Settings` : vérifier que **Hardware CRC**, **Hardware TSO** et **Hardware LRO** sont bien **désactivés** (c'est le défaut d'OPNsense — vérifier quand même). Ces mécanismes délèguent des calculs réseau à la « carte » ; sur une carte VirtIO virtuelle, ils produisent des sommes de contrôle invalides et des paquets géants que `pf` et surtout Suricata digèrent mal. Symptômes classiques quand c'est activé : débits erratiques, connexions qui pendent, IDS aveugle. À contrôler après chaque restauration de config.

### L'agent invité QEMU

`System → Firmware → Plugins` → installer **`os-qemu-guest-agent`**. Proxmox peut alors arrêter la VM proprement (ACPI ne suffit pas toujours) et afficher ses IP. L'option « Qemu Agent » de la VM a été cochée au chapitre 3 ; redémarrer la VM une fois le plugin en place.

---

## 6. Les règles de pare-feu : traduire la matrice de flux

Rappel de la politique ([[Pare-feu-OPNsense-concepts-et-architecture#4. L'architecture cible|article 1, section 4]]) : **tout est refusé par défaut, chaque autorisation est explicite et journalisée**. OPNsense refuse d'office ce qu'aucune règle n'autorise ; notre travail consiste donc uniquement à écrire les bonnes autorisations — et à retirer celles, trop larges, posées par défaut.

### Les alias d'abord

`Firewall → Aliases` — les alias donnent des noms aux IP et aux ports ; les règles restent lisibles et les changements se font en un seul endroit :

| Nom | Type | Contenu | Rôle |
|---|---|---|---|
| `HOST_ADMIN` | Host | `192.168.1.50` | Le poste d'administration, et lui seul |
| `NET_SERVICES` | Network | `10.0.10.0/24` | Zone SERVICES |
| `NET_DMZ` | Network | `10.0.20.0/24` | Zone DMZ |
| `NETS_PRIVES` | Network | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | Tout l'espace RFC1918, pour écrire « vers Internet et rien d'autre » |
| `HOST_PODMAN` | Host | `10.0.10.10` | La VM Ubuntu/Podman |
| `HOST_RPROXY` | Host | `10.0.20.10` | Le reverse proxy public en DMZ |
| `PORTS_WEB` | Port | `80`, `443` | HTTP/S |

### Règles interface par interface

Dans OPNsense, une règle s'applique sur l'interface où le trafic **entre** dans le pare-feu, et les règles s'évaluent de haut en bas, première correspondance gagnante. `Firewall → Rules → [interface]`.

**WAN** (ce qui vient du LAN maison) :

| # | Action | Proto | Source | Destination | Port | Log | Description |
|---|---|---|---|---|---|---|---|
| 1 | Pass | TCP | `HOST_ADMIN` | This Firewall | 443 | ✅ | Admin GUI depuis le poste dédié uniquement |
| — | *(défaut)* Block | * | * | * | * | | Tout le reste du LAN maison est refusé |

C'est la règle à créer **avant** de réactiver `pfctl -e` au chapitre 4.

**LAN — zone SERVICES** (ce qui sort des VM internes) — supprimer d'abord les règles par défaut « LAN → any », beaucoup trop permissives, puis :

| # | Action | Proto | Source | Destination | Port | Log | Description |
|---|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | `NET_SERVICES` | This Firewall | 53, 123 | | DNS et NTP vers OPNsense |
| 2 | Pass | TCP | `NET_SERVICES` | **! `NETS_PRIVES`** | `PORTS_WEB` | | Mises à jour et dépôts : Internet seulement |
| 3 | Block | * | `NET_SERVICES` | * | * | ✅ | Refus explicite, journalisé |

Le `!` (case *Invert* sur la destination) est l'idiome clé : « vers tout **sauf** les réseaux privés », c'est-à-dire vers Internet sans jamais pouvoir redescendre vers la maison ni traverser vers la DMZ. La règle 3 est redondante avec le refus implicite, mais elle **journalise** — on veut voir les tentatives.

**OPT1 — zone DMZ** (ce qui sort des machines exposées) :

| # | Action | Proto | Source | Destination | Port | Log | Description |
|---|---|---|---|---|---|---|---|
| 1 | Pass | TCP | `HOST_RPROXY` | `HOST_PODMAN` | 8096 | ✅ | Ex. : reverse proxy → Jellyfin, ce flux précis et rien d'autre |
| 2 | Pass | TCP | `NET_DMZ` | ! `NETS_PRIVES` | `PORTS_WEB` | | Certificats ACME, mises à jour |
| 3 | Block | * | `NET_DMZ` | * | * | ✅ | La DMZ est un cul-de-sac |

La règle 1 illustre le principe : chaque flux DMZ → SERVICES est **nominatif** (une source, une destination, un port, une raison). On en ajoutera d'autres au fil des besoins, jamais de « DMZ → SERVICES : any ».

> [!tip] Documenter dans l'outil
> Le champ Description de chaque règle est la matrice de flux vivante. Une règle sans description est une règle qu'on n'osera plus supprimer dans un an.

### NAT

- **Sortie** : `Firewall → NAT → Outbound` en mode **Automatic** — les zones sont masquées derrière `192.168.1.2`, rien à faire.
- **Entrée** (plus tard, quand un service sera réellement exposé) : sur la box FAI, rediriger `80/443` vers `192.168.1.2` ; puis `Firewall → NAT → Port Forward` : WAN `443/tcp` → `HOST_RPROXY:443`, avec la règle de pare-feu associée créée automatiquement. Double redirection à cause du double NAT — c'est le prix assumé de l'architecture, documenté dans l'article 1.

---

## 7. Suricata : IDS d'abord, IPS ensuite

La segmentation limite les déplacements ; Suricata regarde le **contenu** de ce qui circule quand même. `Services → Intrusion Detection`.

### Activer en mode IDS (détection seule)

Onglet **Administration** :

| Réglage | Valeur | Pourquoi |
|---|---|---|
| Enabled | ✅ | |
| IPS mode | ❌ pour l'instant | On observe avant de bloquer |
| Promiscuous mode | ✅ | Voir tout le trafic des interfaces surveillées |
| Interfaces | **LAN, OPT1** | Les frontières intéressantes : ce qui entre et sort des zones. Le WAN ne voit que du trafic déjà NATé, moins lisible |
| Pattern matcher | Hyperscan | Le plus rapide sur CPU x86 récent |

Onglet **Download** : cocher les jeux de règles **ET Open** pertinents — un socle raisonnable : `emerging-exploit`, `emerging-malware`, `emerging-scan`, `emerging-trojan`, `emerging-worm`, `emerging-attack_response`. Puis **Download & Update Rules**, et activer la mise à jour planifiée quotidienne (le bouton crée l'entrée cron dans `System → Settings → Cron`).

> [!note] ET Open, c'est quoi ?
> *Emerging Threats Open* : un jeu de règles communautaire maintenu par Proofpoint, gratuit, couvrant exploits connus, malwares, scanners et canaux C2. La version « Pro » ajoute fraîcheur et couverture, mais l'Open est le standard de fait du homelab. Chaque catégorie cochée = de la RAM consommée et du CPU par paquet : commencer sobre, élargir si la machine suit.

### La phase d'observation

Laisser tourner **une à deux semaines** en IDS pur et prendre l'habitude de l'onglet **Alerts** :

- identifier les **faux positifs** récurrents (le scanner du NAS, les sondes de l'app mobile Immich ont vite fait de matcher une règle de scan) ;
- les neutraliser proprement : onglet **Policy**, créer une règle de politique qui passe les SID concernés en `disable` — plutôt que de désactiver des catégories entières.

Bloquer sans cette phase, c'est découvrir Suricata le jour où il coupe la sauvegarde.

### Passer en IPS (blocage)

Une fois les alertes assainies :

1. `Interfaces → Settings` : re-vérifier que CRC/TSO/LRO matériels sont désactivés (chapitre 5) — **prérequis absolu** du mode IPS.
2. **Administration → IPS mode** ✅, sauvegarder, appliquer.
3. Onglet **Policy** : créer une politique qui passe en **drop** les catégories à haute confiance (`emerging-exploit`, `emerging-malware`, `emerging-trojan`), en laissant le reste en alerte simple.

> [!warning] IPS + VirtIO : le point technique à connaître
> En mode IPS, Suricata s'insère en coupure via **netmap**, une API réseau haute performance de FreeBSD. Historiquement, netmap et le pilote `vtnet` (VirtIO) ont eu des frictions ; sur les versions récentes d'OPNsense c'est fonctionnel, **à condition** que l'offloading matériel soit désactivé. Si après activation le trafic devient erratique sur une interface surveillée : c'est le premier suspect. Le repli sans douleur : rester en IDS + refus journalisés — la détection sans le blocage vaut déjà très cher.

Côté ressources, ordre de grandeur constaté en homelab : ~1 Go de RAM supplémentaire avec ce socle de règles, CPU négligeable au repos, quelques pourcents sous trafic soutenu. C'est la raison des 4 Go alloués au chapitre 3.

---

## 8. Sauvegarde et exploitation

### La config XML : le vrai trésor

Toute la configuration d'OPNsense tient dans **un fichier XML**. `System → Configuration → Backups` :

- **Download configuration** : à faire après chaque changement significatif, fichier stocké avec les sauvegardes chiffrées (voir la stratégie [[Strategie-de-sauvegarde-restic-3-2-1|restic 3-2-1]]) ;
- l'option de chiffrement du XML est recommandée : le fichier contient les secrets (hash des mots de passe, clés VPN…).

Restaurer = installer un OPNsense vierge + importer le XML : la reconstruction complète prend un quart d'heure.

### Snapshots Proxmox : le filet avant chaque saut

Avant chaque mise à jour de firmware ou changement de règles ambitieux :

```
VM opnsense → Snapshots → Take Snapshot   (ex. "avant-maj-25.7")
```

Mise à jour ratée, règle qui enferme dehors : *Rollback*, une minute, réglé. Purger les snapshots obsolètes après validation (un snapshot LVM-Thin qui vieillit dégrade les I/O). Inclure aussi la VM dans le job de sauvegarde `vzdump` régulier de Proxmox.

### Routine d'exploitation

| Fréquence | Geste |
|---|---|
| Hebdomadaire | Coup d'œil aux alertes Suricata et aux refus journalisés inhabituels |
| Mensuelle | `System → Firmware` : appliquer les correctifs (snapshot avant) |
| Semestrielle | Version majeure OPNsense : lire les notes de version, snapshot, mise à jour |
| Après tout changement | Export du XML de config |

---

## 9. Vérifications : prouver que ça filtre

Une politique de sécurité non testée est une hypothèse. Chaque ligne de la matrice de flux se vérifie — les refus **autant que** les autorisations. Ouvrir `Firewall → Log Files → Live View` dans un onglet pendant les tests : chaque refus doit s'y afficher en rouge avec la bonne règle en cause.

**Depuis une VM de la zone DMZ** (`10.0.20.x`) :

```bash
# Doit ÉCHOUER : la DMZ ne redescend pas vers SERVICES
ping -c 2 10.0.10.10
nc -vz -w 3 10.0.10.10 22        # → timeout, et une ligne rouge dans Live View

# Doit ÉCHOUER : ni vers le LAN maison, ni vers l'admin du pare-feu
nc -vz -w 3 192.168.1.10 8006
curl -m 5 -k https://10.0.20.1

# Doit RÉUSSIR : le flux nominatif déclaré, et la sortie web
nc -vz -w 3 10.0.10.10 8096      # la règle reverse proxy → Jellyfin
curl -m 5 https://deb.debian.org
```

**Depuis la VM Ubuntu/Podman** (`10.0.10.10`) :

```bash
# Doit RÉUSSIR : DNS via OPNsense, dépôts sur Internet
dig @10.0.10.1 opnsense.org +short
curl -m 5 https://archive.ubuntu.com

# Doit ÉCHOUER : pas de redescente vers la maison, pas de traversée vers la DMZ
ping -c 2 192.168.1.10
nc -vz -w 3 10.0.20.10 443
```

**Depuis le LAN maison** :

```bash
# Doit RÉUSSIR depuis HOST_ADMIN (192.168.1.50), ÉCHOUER depuis toute autre machine
curl -m 5 -k https://192.168.1.2
```

**Tester Suricata** — depuis une VM d'une zone surveillée :

```bash
curl -m 5 http://testmyids.com
```

Ce site inoffensif renvoie une réponse qui matche la signature `GPL ATTACK_RESPONSE id check returned root` (SID 2100498) : une alerte doit apparaître dans `Services → Intrusion Detection → Alerts`. En mode IPS avec la catégorie en *drop*, la requête doit en plus échouer. Pas d'alerte = l'IDS ne voit pas le trafic — retourner au chapitre 7 (interfaces surveillées, promiscuous, offloading).

> [!success] Critères de sortie du brouillon
> L'article passera de `07_BROUILLONS` à `04_SECURITY` (et en `status: published`) quand tous les tests ci-dessus auront été déroulés sur Pandaria avec le résultat attendu, et que la config XML validée aura rejoint les sauvegardes.

---

## Conclusion

Le homelab n'est plus à plat : trois zones, une matrice de flux appliquée règle par règle, un IDS qui regarde les frontières, des sauvegardes qui rendent l'ensemble reconstructible en un quart d'heure. Chaque franchissement de zone est désormais une décision explicite — journalisée quand elle est refusée, documentée quand elle est accordée.

Les chantiers suivants s'inscrivent naturellement dans ce cadre : brancher [[Configuration-WireGuard-self-hosted|WireGuard]] sur cette segmentation (le VPN atterrit dans une zone, pas « sur le réseau »), déplacer le point d'entrée public [[Reverse-proxy-Caddy-avec-TLS-automatique|Caddy]] dans la DMZ, et à plus long terme faire de même pour la maison elle-même (VLAN IoT) — le jour où un switch manageable rejoindra le rack.
