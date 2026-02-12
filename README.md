# 🏗️ Proxmox Minecraft Deployer

Ce projet permet d'automatiser entièrement le provisionnement de containers (LXC) sur **Proxmox** et l'installation de serveurs **Minecraft** optimisés grâce à **Ansible**.

Fini la configuration manuelle : une commande, et votre infrastructure est prête à accueillir vos joueurs.

---

## 📋 Architecture du projet

Le déploiement se divise en deux étapes distinctes :

1.  **Provisioning (`createct.yml`)** : Utilise l'API de Proxmox pour créer un container LXC à partir d'un template, configurer le réseau, le CPU, la RAM et le stockage.
2.  **Configuration (`minecraft.yml`)** : Installe Java (JRE/JDK), gère les dépendances système, crée l'utilisateur dédié et prépare l'environnement pour le serveur Minecraft.

---

## 🛠️ Prérequis

Avant de commencer, assurez-vous d'avoir :

* **Ansible** installé sur votre machine de contrôle.
* Un serveur **Proxmox VE** fonctionnel.
* Le module Python `proxmoxer` installé (`pip install proxmoxer`).
* Un **Token d'API Proxmox** (ou les identifiants root) pour permettre à Ansible de piloter l'hyperviseur.

---

## 🚀 Utilisation
