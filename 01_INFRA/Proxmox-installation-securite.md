---
title: "Installation et sécurisation de Proxmox VE 8"
author: "0x41647269656E"
tags:
  - proxmox
  - virtualisation
  - securite
  - hardening
  - ubuntu
  - podman
series: "Infrastructure"
reading-time: 30m
date: 29-06-2026
last_modified: 29-06-2026
status: published
---

# Installation et sécurisation de Proxmox VE 9.2

> [!abstract] TL;DR
> Ce guide couvre l'installation de Proxmox VE 9.2 depuis zéro : choix du schéma de partitionnement, configuration du stockage (LVM vs ZFS), durcissement post-installation, puis création d'une VM Ubuntu Server minimale sur laquelle Podman sera installé en mode rootless. Un playbook Ansible complémentaire automatise le hardening de la VM. Voir `05_AUTOMATION/ansible/ubuntu-server-hardening/`.

---

## Prérequis

- Machine physique avec au minimum : 4 cœurs CPU, 16 Go RAM, 1 SSD dédié à l'OS
- Une clé USB d'au moins 1 Go pour l'ISO
- Accès physique ou IPMI/iDRAC à la machine
- ISO Proxmox VE téléchargée depuis [proxmox.com/en/downloads](https://www.proxmox.com/en/downloads)

> [!tip] Vérifier l'intégrité de l'ISO
> Toujours vérifier le SHA256 de l'ISO avant de flasher.
> ```bash
> sha256sum proxmox-ve_*.iso
> ```
> Comparer avec la somme publiée sur le site officiel.

---

## 1. Stratégie de partitionnement

C'est le choix le plus structurant de toute l'installation. Proxmox propose plusieurs options via son installeur graphique.

### Options disponibles

| Système de fichiers | Avantages | Inconvénients | Recommandé si... |
|---|---|---|---|
| **ext4 + LVM** | Simple, éprouvé, faible overhead | Pas de checksums, snapshots limités | Peu de RAM, débutant |
| **ext4 sans LVM** | Encore plus simple | Pas de redimensionnement facile | Pas recommandé |
| **ZFS (RAID-Z / miroir)** | Intégrité des données, checksums, snapshots, compression | Gourmand en RAM (min 8 Go dédiés), pas de disques hétérogènes | Environnement avec RAM ECC et ≥ 32 Go RAM |
| **btrfs** | Snapshots, compression | Moins mature que ZFS pour usage production | Non recommandé en homelab |

### Recommandation pour un homelab type Pandaria

> [!note] Contexte
> Pandaria dispose d'un SSD NVMe de 256 Go pour l'OS et de 6 disques HDD gérés par mergerfs séparément. Proxmox sera installé sur le SSD, les HDD ne sont pas touchés par l'installeur.

**Choix retenu : `ext4` avec `LVM`** (option par défaut de Proxmox)

**Justification :**
- Le pool mergerfs sur les HDD est géré manuellement au niveau OS, pas par Proxmox Storage.
- LVM permet de redimensionner les volumes système et d'allouer de l'espace aux VMs via `local-lvm` (LVM-Thin).
- ZFS est intéressant mais nécessite idéalement de la RAM ECC et impacte les performances sans ECC.

### Schéma de partitionnement recommandé (ext4 + LVM)

L'installeur Proxmox crée automatiquement ce schéma sur le SSD cible :

```
/dev/nvme0n1
├── nvme0n1p1   512 MB   vfat   /boot/efi   (EFI)
├── nvme0n1p2   1 GB     ext4   /boot
└── nvme0n1p3   reste    LVM    pve (Volume Group)
    ├── pve/swap      8 GB    swap
    ├── pve/root      96 GB   ext4   /
    └── pve/data      (reste) LVM-Thin → stockage VMs et CTs
```

> [!info] LVM-Thin pour les VMs
> `local-lvm` (le datastore LVM-Thin créé par défaut) est idéal pour les VMs car il supporte le thin provisioning et les snapshots sans ZFS. Une VM de 32 Go ne consomme que l'espace réellement utilisé.

**Paramètres à ajuster dans l'installeur** :
- Sélectionner le bon disque cible (vérifier deux fois)
- Laisser le système de fichiers sur `ext4`
- Ajuster `hdsize` si le SSD est large (256 Go) : laisser 30–40 Go pour `pve/root`, le reste pour `pve/data`

---

## 2. Installation

### 2.1 Boot sur l'ISO

1. Flasher l'ISO sur clé USB : `dd if=proxmox-ve_*.iso of=/dev/sdX bs=4M status=progress`
2. Booter sur la clé, sélectionner **"Install Proxmox VE (Graphical)"**

### 2.2 Écrans importants

- **Sélection du disque** : choisir le SSD NVMe, vérifier que les HDD ne sont pas inclus
- **Pays / timezone** : Europe/Paris
- **Mot de passe root** : mot de passe fort, à stocker dans Vaultwarden — ce compte sera désactivé en SSH après installation
- **Hostname** : utiliser un FQDN (`pve.lan` ou `proxmox.homelab.local`)
- **IP** : configurer une IP statique dès l'installation (évite les problèmes DHCP après reboot)

> [!warning] IP statique obligatoire
> Proxmox ne fonctionne pas bien avec DHCP. Assigner une IP fixe dans l'installeur ou via DHCP-statique sur le routeur.

---

## 3. Hardening post-installation

Se connecter en SSH ou via la console Proxmox une fois l'installation terminée.

### 3.1 Sources APT : supprimer le dépôt enterprise

```bash
# Désactiver le dépôt entreprise (nécessite un abonnement payant)
echo "# disabled" > /etc/apt/sources.list.d/pve-enterprise.list
echo "# disabled" > /etc/apt/sources.list.d/ceph.list

# Ajouter le dépôt communautaire no-subscription
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list

apt update && apt full-upgrade -y
```

> [!info] No-subscription vs Enterprise
> Le dépôt `pve-no-subscription` reçoit les mises à jour stables légèrement après le canal enterprise. Parfaitement adapté à un homelab.

### 3.2 Créer un utilisateur admin dédié et désactiver root

Dans l'interface web Proxmox (`https://<IP>:8006`) :

1. **Datacenter → Users → Add** : créer un utilisateur `admin@pve` (realm `Proxmox VE authentication server`)
2. **Datacenter → Permissions → Add → User Permission** : assigner le rôle `Administrator` sur `/`
3. **Activer 2FA (TOTP)** :
   - User → Two Factor → TOTP → scanner avec une appli d'authentification (Aegis, FreeOTP)

> [!warning] Ne pas supprimer root@pam
> Désactiver root en SSH suffit. Le compte `root@pam` dans Proxmox reste nécessaire pour la récupération d'urgence.

### 3.3 Durcissement SSH

```bash
# Éditer la configuration SSH
cat > /etc/ssh/sshd_config.d/99-hardening.conf << 'EOF'
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AuthenticationMethods publickey
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no
AllowTcpForwarding no
EOF

systemctl restart sshd
```

> [!tip] Déposer sa clé publique avant
> ```bash
> ssh-copy-id -i ~/.ssh/id_ed25519.pub root@<IP_PROXMOX>
> ```
> Vérifier que la connexion par clé fonctionne **avant** de désactiver les mots de passe.

### 3.4 Firewall Proxmox

Proxmox intègre un firewall nftables configurable par l'interface web.

**Activer le firewall** :
1. **Datacenter → Firewall → Options** : `Firewall: Yes`
2. **Datacenter → Firewall → Add** (règles entrantes) :

| Direction | Action | Protocol | Port | Description |
|---|---|---|---|---|
| IN | ACCEPT | TCP | 8006 | Web UI Proxmox |
| IN | ACCEPT | TCP | 22 | SSH |
| IN | DROP | — | — | (règle par défaut finale) |

> [!note] Firewall par nœud vs datacenter
> Les règles au niveau **Datacenter** s'appliquent à tous les nœuds. Les règles au niveau du **nœud** s'appliquent uniquement à lui. Commencer par le niveau Datacenter.

### 3.5 Fail2ban

```bash
apt install fail2ban -y

cat > /etc/fail2ban/jail.d/proxmox.conf << 'EOF'
[proxmox]
enabled  = true
port     = https,http,8006
filter   = proxmox
backend  = systemd
maxretry = 5
findtime = 300
bantime  = 3600

[sshd]
enabled  = true
maxretry = 4
bantime  = 3600
EOF

cat > /etc/fail2ban/filter.d/proxmox.conf << 'EOF'
[Definition]
failregex = pvedaemon\[.*authentication failure; rhost=<HOST> user=.* msg=.*
ignoreregex =
EOF

systemctl enable --now fail2ban
```

### 3.6 Mises à jour automatiques de sécurité

```bash
apt install unattended-upgrades -y

cat > /etc/apt/apt.conf.d/50unattended-upgrades << 'EOF'
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
};
Unattended-Upgrade::AutoFixInterruptedDpkg "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "false";
EOF
```

### 3.7 Supprimer le bandeau "No valid subscription"

```bash
# Supprimer le popup de souscription dans l'UI (uniquement cosmétique)
sed -i.bak "s/data.status !== 'Active'/false/g" \
  /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
systemctl restart pveproxy
```

### 3.8 Audit avec Lynis (optionnel)

```bash
apt install lynis -y
lynis audit system
```

> [!tip] Score cible
> Un homelab bien configuré doit atteindre un score Lynis entre 65 et 80. Au-delà, le durcissement devient contre-productif (performances, complexité).

---

## 4. Création de la VM Ubuntu Server

### 4.1 Télécharger l'ISO Ubuntu Server

Dans **Proxmox → node → local → ISO Images → Download from URL** :

```
https://releases.ubuntu.com/noble/ubuntu-24.04.2-live-server-amd64.iso
```

### 4.2 Paramètres de la VM

Créer une VM via **Create VM** avec les paramètres suivants :

| Paramètre | Valeur | Justification |
|---|---|---|
| VM ID | 100 | Conventionnel |
| Name | `ubuntu-podman` | — |
| OS type | Linux 6.x | Kernel récent |
| Machine | `q35` | Support PCIe, IOMMU, Secure Boot |
| BIOS | `OVMF (UEFI)` | Plus sécurisé que SeaBIOS |
| CPU | 2 sockets, 1 core (= 2 vCPU) | Ajuster selon usage |
| RAM | 4096 Mo | Minimum confortable pour Podman |
| Disk | 32 Go, VirtIO SCSI, sur `local-lvm` | VirtIO = meilleures perfs |
| Network | `vmbr0`, VirtIO | — |

> [!info] q35 vs i440fx
> Le type de machine `q35` est recommandé pour les nouvelles VMs. Il supporte PCIe, le passthrough d'appareils et UEFI Secure Boot. `i440fx` reste disponible pour compatibilité avec de vieux OS.

### 4.3 Installation d'Ubuntu Server (minimale)

Durant l'installation :
- Sélectionner **"Ubuntu Server (minimized)"** — variante allégée sans services inutiles
- **Partitionnement** : LVM recommandé (permet snapshots Proxmox cohérents avec QEMU guest agent)
- **OpenSSH** : cocher l'installation à l'étape "Featured Server Snaps"
- Ne pas installer de snaps supplémentaires

> [!tip] QEMU Guest Agent
> Installer l'agent QEMU dans la VM pour que Proxmox puisse gérer les snapshots cohérents et récupérer l'IP de la VM :
> ```bash
> apt install qemu-guest-agent -y
> systemctl enable --now qemu-guest-agent
> ```
> Activer dans Proxmox : VM → Options → QEMU Guest Agent → Enabled

### 4.4 Automatiser le hardening avec Ansible

Une fois la VM accessible en SSH, appliquer le playbook de hardening :

```bash
# Depuis la machine de contrôle (pas depuis Proxmox)
cd 05_AUTOMATION/ansible/ubuntu-server-hardening/
ansible-playbook -i inventory.ini site.yml -K
```

Voir [[05_AUTOMATION/ansible/ubuntu-server-hardening/site.yml]] pour le détail des tâches.

---

## 5. Architecture cible

```
┌─────────────────────────────────────────────┐
│              Proxmox VE 8                   │
│   (SSD NVMe 256 Go — ext4 + LVM)           │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │  VM 100 — ubuntu-podman             │   │
│  │  Ubuntu Server 24.04 LTS (minimal)  │   │
│  │  32 Go VirtIO SCSI (LVM-Thin)       │   │
│  │                                     │   │
│  │  ┌────────────────────────────────┐ │   │
│  │  │  Podman rootless               │ │   │
│  │  │  Quadlets → systemd user       │ │   │
│  │  └────────────────────────────────┘ │   │
│  └──────────────────────────────────────┘   │
│                                             │
│  Storage séparé : /media (mergerfs, HDD)   │
└─────────────────────────────────────────────┘
```

---

## 6. Références

- [Documentation officielle Proxmox VE](https://pve.proxmox.com/wiki/Main_Page)
- [Proxmox Firewall](https://pve.proxmox.com/wiki/Firewall)
- [Ubuntu Server 24.04 LTS](https://ubuntu.com/server/docs)
- [Podman rootless](https://github.com/containers/podman/blob/main/docs/tutorials/rootless_tutorial.md)
- [[01_INFRA/Containers/Avantages_Conteneurisation_AutoHebergement]]
- [[04_SECURITY/Self-hosting-securise-avec-Podman]]
