---
date: '2025-10-09T21:50:18+02:00'
draft: false
title: 'Hors Série 3 - Onlyfacts'
summary: "Homelab - Hors Série 3 - Onlyfacts"
tags: ["homelab"]
categories: ["homelab"]
---

> Ce document contient les livrables issus de la mise en place du service `Onlyfacts`. Il s'agit d'un outil de récupération et de traitement des facts Ansible. L'objectif est de développer un outil de mise d'inventaire et de mise en conformité. Combien de Linux sont présents sur mon infrastructure ? Combien de Debian 11 ? Combien expose le service httpd ? Si oui, combien sur la version 2.4.2 ? Combien disposent-ils de mises à jour de sécurité en attente ?

> Onlyfacts est un outil toujours en cours de développement accessible via ce [dépôt](https://gitlab.com/nicolas.gobert/onlyfacts)

---

# 0. Todo list

- [ ] Gestion de l'exécution du playbook
    - [ ] Sécurisé via API SSH
        - [ ] Utilisateur spécifique
        - [ ] Clé SSH 
        - [ ] Droits restreints
- [ ] Gestion des archives des facts
- [ ] Intégration des facts dans la DB
- [ ] Gestion du roulement des configurations Ansible dans la DB
- [ ] Gestion des logs applicatifs
- [ ] Intégrer la clé SSH de l'utilisateur Ansible dans le container app
- [ ] Modifier la configuration de la commande Ansible (+ user friendly)
- [ ] Test déploiement from scratch

---

# 1. Création de la VM

Nous allons utiliser le template `debian12-template` créé lors du chapitre 4. Sur Proxmox on crée un clone complet à partir de ce template. Voici les caractéristiques de la VM :

| OS      | Hostname     | Adresse IP | Interface réseau | vCPU    | RAM   | Stockage
|:-:    |:-:    |:-:    |:-:    |:-:    |:-:    |:-:
| Debian 12.10     | onlyfacts-vms      | 192.168.200.6    | vmbr2 (vms)    | 1     | 2024   | 20Gio

---

# 2. Configuration de l'OS via Ansible

> Les informations concernant Ansible sont disponibles au niveau des chapitres 7 et 8.

A présent, le playbook et les rôles ayant pour objectif d'appliquer la configuration de base de l'OS sont disponibles. Il faut se connecter en tant que l'utilisateur `ansible` sur le serveur `ansible-core.homelab` puis ajouter l'hôte `onlyfacts-vms.homelab` au niveau du fichier d'inventaire `/opt/ansible/envs/200-vms/00_inventory.yml` avec les éléments suivants

```yml
onlyfacts-vms.homelab:
    ip: 192.168.200.6
    hostname: onlyfacts-vms
```

Pour exécuter le playbook, il faut lancer la commande suivante

```bash
ansible-playbook -i envs/200-vms/00_inventory.yml -l 'onlyfacts-vms.homelab,' playbooks/00_config_vm.yml
```

Voici le récapitulatif

```
TASKS RECAP *******************************************************************************************************************************************************************************************************************************
jeudi 09 octobre 2025  22:10:51 +0200 (0:00:00.248)       0:00:07.796 ********* 
=============================================================================== 
Gathering Facts -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 1.06s
dns_config : Autoremove et purge --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.64s
base_packages : Installation des paquets de base ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.49s
dns_config : Installation du paquet systemd-resolved ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.49s
motd : Déploiement du motd --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.46s
hostname_config : Modification du hostname ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.44s
base_packages : Mise à jour du cache apt ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.43s
nftables : Activer et démarrer le service nftables --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.40s
dns_config : Enable du daemon systemd-resolved ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.39s
dns_config : Suppression du paquet resolvconf -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.34s
nftables : Déploiement de la configuration de nftables ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.33s
dns_config : Resart du daemon systemd-resolved ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.30s
nftables : Valider la configuration nftables --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.25s
hostname_config : Modification du fichier /etc/hosts ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.20s
dns_config : Suppression du fichier /etc/resolv.conf ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.20s
ipv6_disable : Désactivation de la prise en charge de l'IPv6 globalement ----------------------------------------------------------------------------------------------------------------------------------------------------------- 0.19s
dns_config : Configuration du DNS dans /etc/resolved.conf -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.19s
ipv6_disable : Désactivation de la prise en charge de l'IPv6 par défaut ------------------------------------------------------------------------------------------------------------------------------------------------------------ 0.14s
security_ssh : Activation de l'authentification par clé ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.14s
dns_config : Création du nouveau lien symbolique vers /etc/resolv.conf ------------------------------------------------------------------------------------------------------------------------------------------------------------- 0.14s
```

Afin de déterminer la dimension des ressources nécessaires du serveur pour que le service soit fonctionnel, nous activons l'observabilité via Prometheus/Grafana pour cette machine.

Modification du fichier inventaire en ajoutant cette section

```yaml
prometheus_clients:
    hosts:
        onlyfacts-vms.homelab: 
```

Pour exécuter le playbook, il faut lancer la commande suivante

```bash
ansible-playbook -i envs/200-vms/00_inventory.yml -l 'onlyfacts-vms.homelab,' playbooks/01_prometheus_node_exporter.yml
```

Voici le récapitulatif

```
TASKS RECAP *******************************************************************************************************
jeudi 09 octobre 2025  22:24:05 +0200 (0:00:00.265)       0:00:04.483 ********* 
=============================================================================== 
Gathering Facts -------------------------------------------------------------------------------------------- 1.04s
node_exporter : Décompression de Node Exporter ------------------------------------------------------------- 0.83s
node_exporter : Téléchargement de Node Exporter ------------------------------------------------------------ 0.81s
node_exporter : Activation et démarrage le service --------------------------------------------------------- 0.64s
node_exporter : Copie du binaire dans /usr/local/bin ------------------------------------------------------- 0.32s
node_exporter : Création du service systemd ---------------------------------------------------------------- 0.30s
node_exporter : Restart node_exporter ---------------------------------------------------------------------- 0.27s
node_exporter : Création de l'utilisateur node_exporter ---------------------------------------------------- 0.26s
```

---

# 3. Installation de Docker Engine

> Pour exécuter `onlyfacts`, il est nécessaire que le paquet `docker-compose` soit installé sur le serveur. Pour cela, il suffit de suivre [la procédure d'installation officielle](https://docs.docker.com/engine/install/debian/#install-using-the-repository)

Vérification de l'installation de `Docker Compose`

```bash
$ docker compose version
Docker Compose version v2.40.0
```

---

# 4. Clonage du projet

Le projet va être clôner dans le répertoire `/opt`

```bash

```

