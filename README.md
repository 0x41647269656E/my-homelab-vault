# My Homelab Vault

![Banner](https://raw.githubusercontent.com/0x41647269656E/my-homelab-vault/4823166e9ed9bf128046699773ad868c57346a13/banner.png)

Bienvenue dans mon dépot **Obsidian Vault pour Homelab** !

Dans ce dépôt, je conserve et documente toute mon infrastructure personnelle homelab : de la création, le choix des composants de la machine, l'installation des systèmes jusqu'à l'exploitation des services.

Je développe mon homelab comme une infrastructure personnelle de services auto-hébergée afin de conserver le contrôle de mes données, automatiser mon environnement numérique, tester des architectures complexes, expérimenter sans contrainte et, surtout, reprendre le contrôle total sur mes outils et mes informations en limitant l'accès à mes informations à des intelligences artificielles de recommandation de contenus apprenant des usages de chacun sur internet.

If you are not paying for the product, you _are_ the product
Celui qui paie l'orchestre, choisi la musique.

---
## 🧭 How to use

Ce dépôt est un **vault Obsidian** prêt à l’emploi.  
Pour l’utiliser :

1. **Installer [Obsidian](https://obsidian.md)** si ce n’est pas déjà fait.  
2. **Cloner ou télécharger** ce dépôt sur votre machine :  
```bash
   git clone git@github.com:0x41647269656E/my-homelab-vault.git
```
3. **Ouvrir Obsidian**.
4. Depuis l’écran d’accueil, cliquer sur **“Open folder as vault”**.
5. Sélectionner le dossier cloné `my-homelab-vault`.
6. Obsidian chargera automatiquement la configuration (`.obsidian/`) et affichera le contenu du vault.

## 🚀 Technologies principales utilisées

| Technologie   | Rôle                             |
| ------------- | -------------------------------- |
| Obsidian.md   | Documentation et prise de notes. |
| Podman        | Gestion de conteneurs.           |
| Syncthing     | Synchronisation P2P.             |
| Paperless-ngx | Gestion numérique des documents. |

## 📂 Structure du dépôt
``` 
│ my-homelab-vault/
│
├── .obsidian/                 # Configuration propre à Obsidian (thème, plugins, workspace…)
│
├── 00_INDEX/                  # Table des matières, liens internes, cartes mentales
│   ├── README.md
│   └── Map_of_Content.md
│
├── 01_INFRA/                  # Cœur de ton homelab : matériel et réseau
│   ├── hardware/              # Détails du matériel utilisé
│   │   ├── server_specs.md
│   │   ├── nas_setup.md
│   │   └── network_diagram.png
│   ├── network/               # Topologie réseau, VLAN, firewall
│   │   ├── homelab_network.md
│   │   └── router_config.md
│   └── virtualization/        # Hyperviseurs, VMs, LXC, etc.
│       ├── proxmox_setup.md
│       └── storage_pools.md
│
├── 02_SERVICES/               # Tous les services auto-hébergés
│   ├── reverse-proxy/         # Nginx Proxy Manager, Traefik, Caddy…
│   ├── monitoring/            # Grafana, Prometheus, etc.
│   ├── automation/            # Ansible, Cron, Watchtower
│   ├── media/                 # Jellyfin, Sonarr, Radarr, etc.
│   └── docs-management/       # Paperless-ngx, etc.
│
├── 03_DATAFLOW/               # Schémas de flux de données, sauvegardes
│   ├── backup_strategy.md
│   ├── syncthing_nodes.md
│   └── diagrams/
│
├── 04_SECURITY/               # Sécurité, chiffrement, authentification
│   ├── vpn.md
│   ├── firewall_rules.md
│   └── passwords_policy.md
│
├── 05_AUTOMATION/             # Scripts, templates, tâches automatisées
│   ├── ansible/
│   ├── bash/
│   └── powershell/
│
├── 06_RECHERCHE_ET_TESTS/     # Expérimentations, sandbox
│   ├── docker-experiments.md
│   └── k3s_cluster_notes.md
│
├── imgs/                      # Images utilisées dans les notes
│
├── README.md                  # Présentation du projet
└── LICENSE
```

## 🛠️ Objectifs

1.  Centraliser et documenter mon homelab.
2.  Gérer les services conteneurisés.
3.  Synchroniser mes données.
4.  Maintenir un flux de documents dématérialisé.

## 📚 Ressources externes

- https://obsidian.md
- https://podman.io
- https://syncthing.net
- https://paperless-ngx.readthedocs.io
- https://github.com/awesome-selfhosted/awesome-selfhosted

## 🧾 Licence

GPL‑3.0
