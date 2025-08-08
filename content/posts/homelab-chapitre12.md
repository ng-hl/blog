---
date: '2025-07-21T20:39:51+02:00'
draft: true
title: 'Chapitre 12 - OpenTofu'
summary: "Homelab - Chapitre 12 - OpenTofu"
tags: ["homelab","opentofu"]
categories: ["homelab"]
---

> Ce document contient les livrables issus de la mise en place du service `OpenTofu`. L'objectif est de pouvoir exécuter du `OpenTofu` sur un serveur dédié. On pourrait tout à fait se passer d'un serveur spécifique pour cet usage et utiliser directement la machine d'administraiton `admin-core`.

---

# 1. Création de la VM

Nous allons utiliser le template `debian12-template` créé lors du chapitre 4. Sur Proxmox on crée un clone complet à partir de ce template. Voici les caractéristiques de la VM :

| OS      | Hostname     | Adresse IP | Interface réseau | vCPU    | RAM   | Stockage
|:-:    |:-:    |:-:    |:-:    |:-:    |:-:    |:-:
| Debian 12.10     | opentofu-core      | 192.168.100.246    | vmbr1 (core)    | 1     | 2048   | 20Gio

Il s'agit d'un serveur qui ne contiendra pas de données persistantes critiques. Je choisis donc de ne pas appliquer de stratégie de sauvegarde sur cette machine.

---

# 2. Configuration de l'OS via Ansible

> Les informations concernant Ansible sont disponibles au niveau des chapitres 7 et 8.

A présent, le playbook et les rôles ayant pour objectif d'appliquer la configuration de base de l'OS sont disponibles. Il faut se connecter en tant que l'utilisateur `ansible` sur le serveur `ansible-core.homelab` puis ajouter l'hôte `opentofu-core.homelab` au niveau du fichier d'inventaire `/opt/ansible/envs/100-core/00_inventory.yml` avec les éléments suivants

```yml
opentofu-core.homelab:
    ip: 192.168.100.246
    hostname: opentofu-core
```

Il est nécessaire d'ajouter les droits sudo sur l'utilisateur `ansible` au niveau du fichier `/etc/sudoers.d/ansible` avec les éléments ci-dessous. Il s'agit d'un oubli au niveau du template. (À corriger plus tard).

```bash
ansible ALL=(ALL) NOPASSWD: ALL
```

Pour exécuter le playbook, il faut lancer la commande suivante

```bash
ansible-playbook -i envs/100-core/00_inventory.yml -l 'opentofu-core.homelab,' playbooks/00_config_vm.yml
```

Voici le récapitulatif

