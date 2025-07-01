---
date: '2025-07-01T14:38:22+02:00'
draft: false
title: 'Hors Série 2 - Agents IA'
summary: "Homelab - Hors Série 2 - Agents IA"
tags: ["homelab"]
categories: ["homelab"]

---

> Ce document contient les livrables issus de la mise en place du service des agents IA via la solution `crewAI`. L'objectif est de réaliser un PoC concernant l'interraction d'IA via la création d'une équipe de développement dont la finalité et de créer une application web simple.

---

# 1. Création de la VM

Nous allons utiliser le template `debian12-template` créé lors du chapitre 4. Sur Proxmox on créé un clone complet à partir de ce template. Voici les caractéristiques de la VM :

| OS      | Hostname     | Adresse IP | Interface réseau | vCPU    | RAM   | Stockage
|:-:    |:-:    |:-:    |:-:    |:-:    |:-:    |:-:
| Debian 12.10     | crewai-vms      | 192.168.200.5    | vmbr2 (vms)    | 1     | 2024   | 20Gio

Il faut également penser à activer la sauvegarde automatique de la VM sur Proxmox en l'ajoutant au niveau de la politique de sauvegarde précédemment créée.

---

# 2. Configuration de l'OS via Ansible

> Les informations concernant Ansible sont disponibles au niveau des chapitres 7 et 8.

A présent, le playbook et les roles ayant pour objectif d'appliquer la configuration de base de l'OS sont disponible. Il faut se connecter en tant que l'utilisateur `ansible` sur le serveur `ansible-core.homelab` puis ajouter l'hôte `crewai-vms.homelab` au niveau du fichier d'inventaire `/opt/ansible/envs/200-core/00_inventory.yml` avec les éléments suivants

```yml
crewai-vms.homelab:
    ip: 192.168.200.5
    hostname: crewai-vms
```

Il est nécessaire d'ajouter les droits sudo sur l'utilisateur `ansible` au niveau du fichier `/etc/sudoers.d/ansible` avec les éléments ci-dessous. Il s'agit d'un oubli au niveau du template. (A corriger plus tard).

```bash
ansible ALL=(ALL) NOPASSWD: ALL
```

Pour exécuter le playbook, il faut lancer la commande suivante

```bash
ansible-playbook -i envs/200-core/00_inventory.yml -l 'crewai-vms.homelab,' playbooks/00_config_vm.yml
```

Voici le récapitulatif
