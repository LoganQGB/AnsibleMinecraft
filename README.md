# 🏗️ Proxmox Minecraft Deployer

> Automatisation complète du provisionnement de conteneurs LXC sur Proxmox et du déploiement de serveurs Minecraft via Ansible.

---

## 📌 Présentation du projet

Ce projet s'inscrit dans une démarche d'**Infrastructure as Code (IaC)** : l'objectif est d'éliminer toute intervention manuelle lors du déploiement de serveurs Minecraft en environnement virtualisé.

À partir d'un simple appel de playbook Ansible, l'infrastructure est entièrement provisionnée et configurée :
- Création et démarrage des conteneurs LXC sur Proxmox via son **API REST**
- Mise à jour automatique de l'inventaire Ansible (`hosts`)
- Installation de Java 21 (JDK Oracle), des dépendances système et du serveur Minecraft
- Création d'un utilisateur dédié `minecraft` avec ses droits et sa clé SSH
- Lancement automatique du serveur dans une session `screen` détachée

---

## 🗂️ Structure du dépôt

```
.
├── creatct.yml       # Playbook 1 : Provisionnement des conteneurs LXC sur Proxmox
├── minecraft.yml     # Playbook 2 : Configuration et déploiement du serveur Minecraft
├── hosts             # Inventaire Ansible (généré automatiquement par creatct.yml)
├── secret.yml        # Variables chiffrées via Ansible Vault (token API Proxmox)
└── README.md
```

---

## 🏛️ Architecture

Le déploiement s'articule en deux phases séquentielles :

```
┌─────────────────────────────────────────────────────────────────┐
│  Machine de contrôle (conteneur Ansible)                        │
│                                                                 │
│   [1] creatct.yml ──► API Proxmox ──► Création conteneurs LXC  │
│              └──────────────────────► Mise à jour hosts         │
│                                                                 │
│   [2] minecraft.yml ─► SSH ──► Conteneurs LXC                  │
│              └──────────────────────► Installation Java 21      │
│                                       Installation Minecraft    │
│                                       Démarrage via screen      │
└─────────────────────────────────────────────────────────────────┘
```

| Phase | Playbook | Cible | Description |
|-------|----------|-------|-------------|
| 1 | `creatct.yml` | `localhost` → API Proxmox | Création des LXC, configuration réseau, mise à jour inventaire |
| 2 | `minecraft.yml` | Conteneurs LXC (`lxc_containers`) | Installation Java, déploiement Minecraft, lancement du service |

---

## 🛠️ Prérequis

### Environnement

| Composant | Version recommandée |
|-----------|---------------------|
| Ansible | ≥ 2.14 |
| Python | ≥ 3.10 |
| Proxmox VE | ≥ 7.x |
| Template LXC | `debian-12-standard_12.12-1_amd64.tar.zst` |

### Modules Ansible requis

```bash
ansible-galaxy collection install community.general
pip install proxmoxer requests --break-system-packages
```

### Accès SSH

La machine de contrôle Ansible doit disposer d'une paire de clés SSH. La clé publique (`~/.ssh/id_rsa.pub`) est automatiquement injectée dans les conteneurs lors du provisionnement.

```bash
ssh-keygen -t rsa -b 4096   # Si aucune clé n'existe
```

---

## 🔐 Configuration du Token API Proxmox

Ce projet utilise l'**authentification par token** (recommandée) plutôt que les identifiants root, conformément aux bonnes pratiques de sécurité.

### Création du token sur Proxmox

1. Se connecter en ssh au proxmox avec les droits root
2. créer un utilisateur dédié par exemple "ansible" :
```bash
pveum user add ansible@pve
pveum passwd ansible@pve
```
3. Créer un rôle personnalisé avec les permissions nécessaires pour provisionner des conteneurs :
```bash
pveum role add AnsibleProvisioner -privs "Datastore.Allocate,Datastore.AllocateSpace,Datastore.Audit,VM.Allocate,VM.Audit,VM.Config.CDROM,VM.Config.CPU,VM.Config.Disk,VM.Config.HWType,VM.Config.Memory,VM.Config.Network,VM.Config.Options,VM.PowerMgmt"
```
4. Associer le rôle AnsibleProvisioner à l'utilisateur ansible@pve :
```bash
pveum acl modify / -user ansible@pve -role AnsibleProvisioner
```
5. Créer un token pour l'utilisateur :
```bash
pveum user token add ansible@pve provisioning  --privsep 0
```
### Chiffrement du secret avec Ansible Vault

Le token secret ne doit **jamais** être stocké en clair dans le dépôt. Il est chiffré via Ansible Vault :

```bash
ansible-vault create secret.yml
```

Contenu de `secret.yml` :

```yaml
token_secret: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

---

## ⚙️ Variables de configuration

Les principales variables sont définies dans l'en-tête du playbook `creatct.yml` et peuvent être adaptées selon l'environnement.

| Variable | Valeur par défaut | Description |
|----------|-------------------|-------------|
| `container_count` | `2` | Nombre de conteneurs à créer |
| `base_vmid` | `305` | VMID de départ (incrémenté pour chaque conteneur) |
| `base_hostname` | `ServerMinecraft-` | Préfixe du nom des conteneurs (`ServerMinecraft-01`, `ServerMinecraft-02`…) |
| `base_ip` | `10.1.0.101` | IP de départ (incrémentée automatiquement) |
| `gateway` | `10.1.0.254` | Passerelle réseau |
| `netmask` | `24` | Masque de sous-réseau |
| `template` | `debian-12-standard_…` | Template LXC Debian 12 |
| `cores` | `2` | Nombre de cœurs CPU par conteneur |
| `memory` | `1024` | RAM en Mo par conteneur |
| `api_host` | `10.1.0.253` | Adresse IP du nœud Proxmox |
| `pool` | `EPREUVE-E6-LQUAGHEBEUR` | Pool Proxmox cible |

---

## 🚀 Utilisation

### Étape 1 — Provisionnement des conteneurs

Ce playbook crée les conteneurs LXC sur Proxmox, les démarre, attend leur disponibilité réseau, puis génère automatiquement le fichier `hosts`.

```bash
ansible-playbook creatct.yml --vault-password-file vault_pass.txt
```

**Ce que fait ce playbook, dans l'ordre :**
1. Nettoyage des entrées `known_hosts` pour éviter les conflits SSH
2. Création des conteneurs via l'API Proxmox (VMID, hostname, IP, CPU, RAM)
3. Démarrage des conteneurs
4. Attente de 10 secondes pour l'attribution IP
5. Récupération des IPs via `pct exec` sur le nœud Proxmox
6. Réinitialisation et mise à jour du fichier `hosts` (groupe `[lxc_containers]`)
7. Correction des permissions de `authorized_keys`

### Étape 2 — Déploiement du serveur Minecraft

Ce playbook se connecte aux conteneurs listés dans `hosts` et configure l'environnement Minecraft.

```bash
ansible-playbook minecraft.yml -i hosts
```

**Ce que fait ce playbook, dans l'ordre :**
1. Installation des paquets système (`screen`, `nano`, `wget`, `git`)
2. Téléchargement et installation de **Java 21 JDK** (Oracle)
3. Création de l'utilisateur `minecraft` avec sa clé SSH
4. Création du répertoire `/home/minecraft/server`
5. Téléchargement du fichier `server.jar` (Minecraft 1.21)
6. Création du fichier `eula.txt` (acceptation automatique de la licence)
7. Configuration des locales système (`en_US.UTF-8`)
8. Lancement du serveur dans une session `screen` détachée (`-dmS minecraft1`)

---

## 🔍 Vérification du déploiement

### Vérifier que les conteneurs sont actifs

```bash
# Depuis le nœud Proxmox
pct list
```

### Vérifier que le serveur Minecraft tourne

```bash
# Se connecter au conteneur
ssh minecraft@<IP_DU_CONTENEUR>

# Lister les sessions screen actives
screen -ls

# Se rattacher à la session Minecraft
screen -r minecraft1
```

> Pour quitter `screen` sans couper le serveur : `Ctrl+A` puis `D`

### Consulter les logs Minecraft

```bash
screen -r minecraft1
# Les logs s'affichent en direct. Quitter avec Ctrl+A D.
```

---

## 🔒 Sécurité

- Le token API Proxmox est chiffré via **Ansible Vault** et n'est jamais stocké en clair
- Les conteneurs sont créés avec une clé SSH injectée automatiquement (pas de mot de passe SSH)
- Le serveur Minecraft tourne sous l'utilisateur dédié `minecraft` (pas root)
- Les conteneurs sont de type **unprivileged: false** pour permettre la gestion réseau avancée

> ⚠️ Le mot de passe root des conteneurs (`root!`) défini dans le playbook doit être changé en production. Il est recommandé de le déplacer dans `secret.yml`.

---

## 📝 Notes techniques

- **Adressage IP automatique** : les IPs sont calculées dynamiquement par incrémentation à partir de `base_ip`. Pour `base_ip: 10.1.0.101` et `container_count: 2`, les conteneurs obtiennent `10.1.0.101` et `10.1.0.102`.
- **Idempotence** : le playbook `creatct.yml` réinitialise le fichier `hosts` à chaque exécution pour garantir un inventaire cohérent.
- **Gestion de la mémoire Minecraft** : le serveur est lancé avec `-Xmx500M -Xms500M`, adapté aux conteneurs de 1024 Mo de RAM.
- **Minecraft version** : le `server.jar` correspond à la version identifiée par le hash `95495a7f…` sur les serveurs Mojang (Minecraft 1.21).

---

## 👤 Auteur

**Logan Quaghebeur**  
BTS SIO option SISR  
Projet réalisé dans le cadre de l'épreuve E6
