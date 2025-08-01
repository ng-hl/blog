---
date: '2025-07-31T21:56:48+02:00'
draft: false
title: 'Chapitre 13 - Prometheus'
summary: "Homelab - Chapitre 13 - Prometheus"
tags: ["homelab","prometheus"]
categories: ["homelab"]
---

> Ce document contient les livrables issus de la mise en place du service `Prometheus`. L'objectif est de pouvoir disposer d'un serveur de collecte de métriques pour la supervision. L'outil sera couplé avec `Grafana` pour la partie dahsboard.

---

# 1. Création de la VM

Nous allons utiliser le template `debian12-template` créé lors du chapitre 4. Sur Proxmox on créé un clone complet à partir de ce template. Voici les caractéristiques de la VM :

| OS      | Hostname     | Adresse IP | Interface réseau | vCPU    | RAM   | Stockage
|:-:    |:-:    |:-:    |:-:    |:-:    |:-:    |:-:
| Debian 12.10     | prometheus-core      | 192.168.100.245    | vmbr1 (core)    | 2     | 4096   | 20Gio

Il faut également penser à activer la sauvegarde automatique de la VM sur Proxmox en l'ajoutant au niveau de la politique de sauvegarde précédemment créée.

---

# 2. Configuration de l'OS via Ansible

> Les informations concernant Ansible sont disponibles au niveau des chapitres 7 et 8.

A présent, le playbook et les roles ayant pour objectif d'appliquer la configuration de base de l'OS sont disponibles. Il faut se connecter en tant que l'utilisateur `ansible` sur le serveur `ansible-core.homelab` puis ajouter l'hôte `prometheus-core.homelab` au niveau du fichier d'inventaire `/opt/ansible/envs/100-core/00_inventory.yml` avec les éléments suivants

```yml
prometheus-core.homelab:
    ip: 192.168.100.245
    hostname: prometheus-core
```

Il est nécessaire d'ajouter les droits sudo sur l'utilisateur `ansible` au niveau du fichier `/etc/sudoers.d/ansible` avec les éléments ci-dessous. Il s'agit d'un oubli au niveau du template. (A corriger plus tard).

```bash
ansible ALL=(ALL) NOPASSWD: ALL
```

Pour exécuter le playbook, il faut lancer la commande suivante

```bash
ansible-playbook -i envs/100-core/00_inventory.yml -l 'prometheus-core.homelab,' playbooks/00_config_vm.yml
```

Voici le récapitulatif
...
TASKS RECAP **********************************************************************************************************************
vendredi 01 août 2025  21:32:25 +0200 (0:00:00.199)       0:00:06.948 ********* 
=============================================================================== 
Gathering Facts ----------------------------------------------------------------------------------------------------------- 1.00s
dns_config : Autoremove et purge ------------------------------------------------------------------------------------------ 0.58s
base_packages : Installation des paquets de base -------------------------------------------------------------------------- 0.46s
dns_config : Installation du paquet systemd-resolved ---------------------------------------------------------------------- 0.45s
motd : Déploiement du motd ------------------------------------------------------------------------------------------------ 0.44s
hostname_config : Modification du hostname -------------------------------------------------------------------------------- 0.42s
base_packages : Mise à jour du cache apt ---------------------------------------------------------------------------------- 0.39s
dns_config : Enable du daemon systemd-resolved ---------------------------------------------------------------------------- 0.38s
dns_config : Suppression du paquet resolvconf ----------------------------------------------------------------------------- 0.34s
dns_config : Resart du daemon systemd-resolved ---------------------------------------------------------------------------- 0.29s
nftables : Déploiement de la configuration de nftables -------------------------------------------------------------------- 0.29s
nftables : Valider la configuration nftables ------------------------------------------------------------------------------ 0.20s
ipv6_disable : Désactivation de la prise en charge de l'IPv6 globalement -------------------------------------------------- 0.20s
dns_config : Suppression du fichier /etc/resolv.conf ---------------------------------------------------------------------- 0.19s
hostname_config : Modification du fichier /etc/hosts ---------------------------------------------------------------------- 0.19s
dns_config : Configuration du DNS dans /etc/resolved.conf ----------------------------------------------------------------- 0.19s
ipv6_disable : Désactivation de la prise en charge de l'IPv6 par défaut --------------------------------------------------- 0.14s
dns_config : Création du nouveau lien symbolique vers /etc/resolv.conf ---------------------------------------------------- 0.14s
security_ssh : Désactivaction de l'authentification par mot de passe ------------------------------------------------------ 0.14s
security_ssh : Activation de l'authentification par clé ------------------------------------------------------------------- 0.14s

---

# 3. Installation de Prometheus

> Avant de procéder aux opérations ci-dessous, il est nécessaire d'installation `docker engine` sur le serveur.

Une image Docker prometheus est disponible sur DockerHub. Nous allons utiliser cette image pour héberger notre instance prometheus

Création du volume Docker pour la persistence de la données

```bash
docker volume create prometheus-data
```

Création du répertoire /opt/prometheus

```bash
sudo mkdir -p /opt/prometheus
```

Modification du propriétaire et du groupe sur le répertoire précemment créé

```bash
sudo chown ngobert:ngobert /opt/prometheus
```

Création du fichier de configuration /opt/prometheus/prometheus.yml

```bash
global:
  scrape_interval:     15s
  evaluation_interval: 15s

rule_files:
  # - "first.rules"
  # - "second.rules"

alerting:
  alertmanagers:
  - static_configs:
    - targets:
       - localhost:9093

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']
  - job_name: admin-core
    static_configs:
      - targets: ['admin-core.homelab']
```

Création du répertoire pour les certificats TLS

```bash
mkdir -p /opt/prometheus/certs
```

Il est nécessaire de récupérer le certificat wildcard `*.ng-hl.com` ainsi que la clé privée associée. Après avoir autoriser la clé `id_acme` sur le serveur `prometheus-core` on peut procéder au scp

```bash
scp -i ~/.ssh/id_acme /etc/ssl/private/wildcard.ng-hl.com/privkey.key root@prometheus-core.homelab:/tmp/
scp -i ~/.ssh/id_acme /etc/ssl/certs/wildcard.ng-hl.com/fullchain.pem root@prometheus-core.homelab:/tmp/
```

Ensuite, on déplace les éléments dans `/opt/prometheus/certs/` et on modifier le propriétaire et le groupe `ngobert:ngobert`.

Création du fichier `tls.yml`

```bash
tls_server_config:
  cert_file: /etc/prometheus/certs/fullchain.pem
  key_file: /etc/prometheus/certs/privkey.pem
```

Exécution du container

```bash
docker run -d \
    -p 9090:9090 \
    -v /opt/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
    -v prometheus-data:/prometheus \
    prom/prometheus
```

Modification du fichier de configuration nftables /etc/nftables.conf pour autoriser les requêtes HTTPS sur le port de prometheus `tcp:9090`.

> Il faut également penser à ouvrir le flux depuis l'extérieur du homelab pour pouvoir y accéder depuis mon réseau local.