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

---

# 3. Installation de Prometheus