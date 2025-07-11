---
date: '2025-05-07T21:32:53+02:00'
draft: false
title: 'Chapitre 1 - Le design'
summary: "Homelab - Chapitre 1 - Le design"
tags: ["homelab"]
categories: ["homelab"]
---

> Ce document contient les livrables issus de la phase de design du homelab. On doit se poser les bonnes questions pour répondre efficacement au besoin de départ, à savoir, disposer d'un environnement où l'on peut déployer rapidement des serveurs prêt à l'emploi pour divers cas d'usage. La conception est susceptible de changée au fur et à mesure des travaux, cette page est donc susceptible d'évoluer (mise à jour de l'inventaire, ajout de services/fonctionnalités, ...)

---

# 1. Le hardware

> Cette section est susceptible d'évoluer avec le temps. Des évolutions peuvent être appliquées avec l'acquisition de plus de compute pour améliorer les performances et la résilience ainsi que la mise en place d'un système de stockage plus adapté comme un NAS.

Pour mettre en place ce homelab, il nous faut un appareil qui dispose de suffisemment de compute soit au moins 32Go de RAM et 16vCPU ainsi qu'un minimum d'espace disque soit 1To. De plus, cette machine va être disponible tout le temps 24h/24 7j/7, il est donc important de prendre une solution qui ne comsome pas trop d'énergie.

| Date      | Compute      | Stockage      | Niveau de maturité      |
|:-:    |:-:    |:-:    |:-:    |
| 11/04/2025      | 1 node - E3B Mini PC (32Go RAM, 16 vCPU, 512Go SSD)     | 1 node - E3B Mini PC (512Go SSD)     | 1 🐟      |

---

# 1. Les environnements

Le homelab va être divisé en deux sous-réseaux principaux. Le premier ayant pour objectif d'héberger les divers services utiles au bon fonctionnement du homelab. Le second sera dédié au déploiement et à l'utilisation des VMs et containers pour les tests futures de technologie, OS, etc.

| Nom      | Description      | Adressage      |
|:-:    |---    |---    |
| Core      | Environnement de base du homelab (prod)      | 192.168.100.0/24      |
| VMS      | Environnement de déploiement des VMs (sandbox)     | 192.168.200.0/24      |

---

# 2. Les services

Pour disposer d'un environnement fonctionnel et confortable, nous avons besoin de différents services que l'on va détailler dans les sous-sections suivantes.

## 2.1. Firewall

Il s'agit de la seul VM qui aura une interface réseau directement sur mon réseau local (interface WAN d'un point de vue Firewall) et de ce fait, obtiendra une IP en 192.168.1.0/24. L'objectif est de gérer les autorisations concernant les communications entrantes et sortantes au niveau du homelab. Le choix technique se portera sur la solution `pfSense`.

## 2.2. Serveur DNS

Le DNS va nous permettre d'utiliser les noms associés à nos VM plutôt que les IP avec deux zones DNS `.homelab` (DNS interne du homelab) aisni que `ng-hl.com` (le domaine qui portera les services exposés sur mon réseau local). Le choix technique se portera sur la solution `bind9`. 

## 2.3. Machine d'administration centrale

Cette VM sera le point d'entrée vers les ressources du homelab. L'objectif est d'avoir une machine en frontal juste derrière le firewall avec un accés SSH ouvert depuis le WAN (mon réseau local) accessible à certaines IP. Cette machine pourra faire office de rebond et pourra héberger un certains nombre d'outils.

## 2.4. Serveur de gestion des configuration

Ce service va nous permettre de déployer les configurations des OS que nous déployons. Les actions serons initialisées manuellement dans un premier temps puis nous pourrons intégrer l'outil au sein d'une pipeline via Gitlab-CI plus tard. Le choix technique se portera sur la solution `Ansible`.

## 2.5. Coffre fort numérique

Le coffre fort numérique va nous permettre de stocker divers mots de passe et secrets. Le choix technique se portera sur `VaultWarden`, solution alternative et open source à BitWarden.

## 2.6. Serveur de versionning

Le serveur de versionning permettra la centralisation des différents éléments relatifs à notre infrastructure notammenent concernant l'infrastructure as code avec Terraform et Ansible. De plus, cette VM ouvre la possibilité d'automatiser nos déploiements futurs de VM via les runners et les fonctionnalités de la CI/CD. Le choix technique se portera sur la solution `Gitlab-ce`.

## 2.7. Stack d'observabilité

L'objectif est de disposer d'outils nous permettant de monitorer et de superviser les OS et les services grâce à la collecte des metriques ainsi qu'à l'alerting. Le choix technique se portera sur la "suite" `Prometheus/Grafana`.

## 2.8. Dashboard central

Afin de facilité l'administration du homelab et l'utilisation des différents service, nous allons mettre en place un dashboard moderne et confortable afin d'inventorier l'intégralité des services mis à disposition au sein du homelab. Le choix technique se portera sur `Homepage`

---

# 3. Schéma réseau physique

![Schéma physique](/images/schema_physique.png)

---

# 4. Schéma réseau logique

![Schéma logique](/images/schema_logique.png)

---

# 5. Priorisation

Afin de disposer rapidement d'un homelab fonctionnel avec le minimum de service requis, nous allons définir differents niveaux de maturité avec les mises en place des différents services qui y sont associées.

| Niveau     | Description      | Services     | Déploiement
|---    |:-:    |:-:    |:-:    |
| 🐟    | Le homelab est fonctionnel, il est possible de déployer des VMs préconfigurées à la main via des templates.      | Firewall, DNS, machine d'administration     | Template de VM sur Proxmox
| 🐬     | Le déploiement des VM est uniforme et automatisé. La machine de rebond centralisée peut communiquer avec l'entièreté des machines. Une PKI est en place ainsi que le nom de domaine ng-hl.com et une acme    | Gitlab-ce, Terraform, Ansible, PKI, certificat wildcard, acme     | Template de VM sur Proxmox avec Terraform et Ansible dans une pipeline Gitlab CI/CD 
| 🐳    | La stack d'observabilité est en place et le dashboard Homepage prêt à l'emploi avec une évolution dynamique.     | Prometheus, Grafana, Homepage, notifications (Discord ?)       | Image préconfigurée sur Proxmox avec Terraform et Ansible dans une pipeline Gitlab CI/CD

---

# 6. Todo lists

## 6.1. 🐟

- [x] Niveau 1
    - [x] Installation de Proxmox VE
    - [x] Configuration de Proxmox VE
        - [x] Création de l'utilisateur d'administration
        - [x] Mise en place des bons dépôts pour les mises à jour
        - [x] Mise en place de la sauvegarde déportée
        - [x] Configuration des interfaces vmbr1 et vmbr2
        - [x] Tester le bon fonctionnement
    - [x] Installation de pfSense
        - [x] Importer l'ISO de pfSense
        - [x] Configurer la VM avec trois interfaces (vmbr0, vmbr1 et vmbr2)
        - [x] Installer l'OS via l'ISO
        - [x] Rendre disponible l'interface d'administration depuis le WAN (réseau local)
    - [x] Créer un template de Debian 12
        - [x] Importer l'ISO de Debian 12
        - [x] Installer l'OS avec les éléments suivants
            - [x] Nom : debian12-template.homelab
            - [x] Disque : LVM partitionnement manuel
            - [x] Service : openssh-server
            - [x] Utilisateur : Création de l'utilisateur d'administration
            - [x] Authentification : Intégrer la clé SSH publique de l'utilisateur de la machine de gestion centralisée
            - [x] Utilisateur : Création de l'utilisateur ansible (groupe sudo)
            - [x] Authentification : Intégrer la clé SSH publique de l'utilisateur ansible
            - [x] Réseau : Configuration statique 192.168.100.11/24
        - [x] Tester le bon fonctionnement avec le déploiement d'une VM de test
        - [x] Convertir en tant que template
    - [x] Créer un template de RockyLinux 9
        - [x] Importer l'ISO de RockyLinux 9
        - [x] Installer l'OS avec les éléments suivants
            - [x] Nom : rocky9-template.homelab
            - [x] Disque : LVM partitionnement manuel
            - [x] Service : openssh-server
            - [x] Utilisateur : Création de l'utilisateur d'administration
            - [x] Authentification : Intégrer la clé SSH publique de l'utilisateur de la machine de gestion centralisée
            - [x] Utilisateur : Création de l'utilisateur ansible
            - [x] Authentification : Intégrer la clé SSH publique de l'utilisateur ansible
            - [x] Réseau : Configuration statique 192.168.100.12/24
        - [x] Tester le bon fonctionnement avec le déploiement d'une VM de test
        - [x] Convertir en tant que template
    - [x] Installation du DNS (Bind9)
        - [x] Mise en place de l'OS via les templates
        - [x] Activer la sauvegarde depuis Proxmox
        - [x] Modifications mineures de l'OS (changement du hostname, configuration réseau)
        - [x] Installation de bind9
        - [x] Configuration de la zone DNS et du forwarder
        - [x] Configuration de la zone DNS inverse
        - [x] Tests
    - [x] Création de la machine d'administration centrale `admin-core`
        - [x] Mise en place de l'OS via les templates
        - [x] Activer la sauvegarde depuis Proxmox
        - [x] Modifications mineures de l'OS (changement du hostname, configuration réseau)
        - [x] Modification de la configuration du résolveur DNS pour admin-core
        - [x] Test de la résolution interne depuis admin-core
        - [x] Test de la résolution externe depuis admin-core
        - [x] Importer les clés privées SSH utilisées au sein du homelab
        - [x] Modification du FW (accès SSH depuis le WAN uniquement sur cette VM)

---

## 6.2. 🐬 

- [ ] Niveau 2
    - [ ] Mise en place de Ansible
        - [x] Mise en place de l'OS via les templates
        - [x] Activer la sauvegarde depuis Proxmox
        - [x] Modifications mineures de l'OS (changement du hostname, configuration réseau)
        - [x] Intégration sur admin-core (alias ssh)
        - [x] Installation de Ansible (via pipx)
        - [x] Configuration de Ansible
        - [x] Intégration des hôtes déjà existants
            - [x] Installer le paquet python3
            - [x] Tester le bon fonctionnement des exécutions Ansible
        - [x] Gestion de l'adresse IP temporaire pour les nouvelles VM
        - [ ] Convertir les actions manuelles de configurations mineures avec Ansible (intégrer les actions de la section Sécurisation -> Templates / VM, voir plus bas)
        - [ ] Ajouter un linter
        - [ ] Tester le bon fonctionnement
    - [ ] Sécurisation
        - [ ] pfSense
            - [x] Activer le HTTPS
            - [ ] Activer le renouvellement automatique du certificat TLS
        - [ ] Templates / VM
            - [x] Durcissement de SSH
                - [x] Désactivation de l'accès root en direct via SSH
                - [x] Accès par clé uniquement
                - [x] Mise en place de fail2ban
            - [ ] Mise en place de nftables
    - [ ] Mise en place de Gitlab
        - [x] Mise en place de l'OS via les templates
        - [x] Configuration de l'OS via Ansible
        - [x] Installation de Gitlab CE
        - [x] Configuration de base de Gitlab CE
        - [x] Configuration du HTTPS avec le certificat *.ng-hl.com
        - [x] Configuration du renouvellement automatique du certificat
        - [ ] Tester le renouvellement du certificat (forcer) 
        - [x] Création d'un compte administrateur nominatif
        - [x] Création du groupe core
        - [ ] Création du projet core/ansible et versionner le code existant
        - [x] Création du projet core/deploy
    - [ ] Terraform
        - [ ] Créer le projet core/terraform
        - [ ] Intégration du provider Proxmox
        - [ ] Création d'une VM
        - [ ] Suppression d'une VM
        - [ ] Récupérer les informations pour avoir un inventaire dynamique
    - [ ] Nom de domaine
        - [x] Réserver un nom de domaine (CloudFlare, Duck DNS, ...)
        - [x] Générer le certificat wildcard *.ng-hl.com avec acme.sh
        - [ ] Gérer le renouvellement automatique avec acme.sh
    - [ ] Append : Coffre fort (Vaultwarden)
        - [x] Mise en place de l'OS via les templates
        - [x] Configuration de l'OS via Ansible ou manuellement suivant l'exécution de la tâche
        - [x] Installation de Vaultwarden
        - [x] Configuration de Vaultwarden
        - [x] Test d'utilisation
        - [ ] Stockage des éléments critiques
            - [ ] PKI
            - [x] Clés SSH
            - [ ] Attribuer des mots de passe uniques (utilisateur ngobert, root et pfSense)
            - [ ] Intégration avec Gitlab CI
        - [x] Tests
    - [x] Certificat wildcard *.ng-hl.com
        - [x] Réservation du nom de domaine
        - [x] Création du certificat
        - [x] Automatisation du renouvellement du certificat
            - [x] Configuration de acme + test de renouvellement forcé
            - [x] Script de déploiement du nouveau certificat (indexé sur la liste des services exposés)
            - [x] Test de bout en bout

---

## 6.3. 🐳

(WIP)

---

## 6.4. Projets annexes

- [ ] Projet - Agents IA
    - [ ] Équipe de développement d'agents IA
        - [ ] Serveur crewai-vms
            - [ ] Création du serveur
            - [ ] Configuration via Ansible
            - [ ] Installation de crewai via python uv
            - [ ] Configuration de crewai
        - [ ] Création du dépôt dev-ia sur Gitlab
        - [ ] Provisionnement sur ChatGPT
        - [ ] Configuration des prompts (1 chef d'équipe, 1 développeur, 1 testeur, 1 rédacteur de documentation)

---

# 7. Inventaire

| Hostname    | IP      | OS        | Hostname exposé
| :-:       | :-:       | :-:       | :-:       |
| pfsense-core.homelab    | 192.168.100.254    | Debian 12.10 |   
| dns-core.homelab    | 192.168.100.253    | Debian 12.10 |    
| admin-core.homelab    | 192.168.100.252    | Debian 12.10 |    
| pki-core.homelab | 192.168.100.251 | Debian 12.10 |
| ansible-core.homelab | 192.168.100.250 | Debian 12.10 |
| acme-core.homelab | 192.168.100.248 | Debian 12.10 |
| vaultwarden-core.homelab   | 192.168.100.249 | Debian 12.10 | vaultwarden-core.ng-hl.com vaultwarden.ng-hl.com (CNAME) |
| gitlab-core.homelab | 192.168.100.247 | Debian 12.10 | 
| ansibledev-core.homelab | 192.168.100.11 | Debian 12.10 |
| debian12-template-core.homelab | 192.168.100.10 | Debian 12.10 |
