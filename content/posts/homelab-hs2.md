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

---

# 2. Configuration de l'OS via Ansible

> Les informations concernant Ansible sont disponibles au niveau des chapitres 7 et 8.

A présent, le playbook et les roles ayant pour objectif d'appliquer la configuration de base de l'OS sont disponible. Il faut se connecter en tant que l'utilisateur `ansible` sur le serveur `ansible-core.homelab` puis ajouter l'hôte `crewai-vms.homelab` au niveau du fichier d'inventaire `/opt/ansible/envs/200-core/00_inventory.yml` avec les éléments suivants

```yml
crewai-vms.homelab:
    ip: 192.168.200.5
    hostname: crewai-vms
```

Pour exécuter le playbook, il faut lancer la commande suivante

```bash
ansible-playbook -i envs/200-core/00_inventory.yml -l 'crewai-vms.homelab,' playbooks/00_config_vm.yml
```

Voici le récapitulatif

```bash
mardi 01 juillet 2025  23:05:15 +0200 (0:00:00.260)       0:00:24.587 ********* 
=============================================================================== 
dns_config : Installation du paquet systemd-resolved --------------------------------------------------------------------- 15.34s
base_packages : Installation des paquets de base -------------------------------------------------------------------------- 3.28s
Gathering Facts ----------------------------------------------------------------------------------------------------------- 1.05s
dns_config : Autoremove et purge ------------------------------------------------------------------------------------------ 0.58s
motd : Déploiement du motd ------------------------------------------------------------------------------------------------ 0.46s
hostname_config : Modification du hostname -------------------------------------------------------------------------------- 0.45s
base_packages : Mise à jour du cache apt ---------------------------------------------------------------------------------- 0.40s
dns_config : Enable du daemon systemd-resolved ---------------------------------------------------------------------------- 0.39s
dns_config : Suppression du paquet resolvconf ----------------------------------------------------------------------------- 0.33s
dns_config : Resart du daemon systemd-resolved ---------------------------------------------------------------------------- 0.30s
security_ssh : Restart du daemon sshd ------------------------------------------------------------------------------------- 0.26s
dns_config : Suppression du fichier /etc/resolv.conf ---------------------------------------------------------------------- 0.20s
ipv6_disable : Désactivation de la prise en charge de l'IPv6 globalement -------------------------------------------------- 0.20s
hostname_config : Modification du fichier /etc/hosts ---------------------------------------------------------------------- 0.19s
dns_config : Configuration du DNS dans /etc/resolved.conf ----------------------------------------------------------------- 0.19s
ipv6_disable : Désactivation de la prise en charge de l'IPv6 par défaut --------------------------------------------------- 0.14s
security_ssh : Activation de l'authentification par clé ------------------------------------------------------------------- 0.14s
dns_config : Création du nouveau lien symbolique vers /etc/resolv.conf ---------------------------------------------------- 0.14s
security_ssh : Désactivaction de l'authentification par mot de passe ------------------------------------------------------ 0.14s
dns_config : Configuration du domaine dans /etc/resolved.conf ------------------------------------------------------------- 0.14s
```

---

# 3. Installation de crewAI

> Pour installer crewAI, il est nécessaire que le paquet `python3` soit installé sur le serveur. On peut vérifier avec cette commande `python3 --version`. Si aucune version de Python n'est détecté, il faut l'installer avec la commande suivante `sudo apt install -y python3`.

> Pour créer l'environnement Python isolé spécifique à crewAI, nous utilisons python UV (outils en Rust). Cette solution permet d'isoler proprement l'environnement virtuel et d'avoir des performances accrues concernant l'utilisation de pip (uv pip) et d'autres fonctionnalités de gestion trés intéressantes.

Nous installons `uv`

```bash
curl -Ls https://astral.sh/uv/install.sh | sh
```

Nous vérifions que `uv` est bien disponible sur le système

```bash
uv --version
```

Nous créons le répertoire qui va contenir la solution `crewai`

```bash
sudo mkdir /opt/crewai-project && sudo chown ngobert:ngobert /opt/crewai-project
```

Nous créons l'envrionnement virtuel géré par `uv` puis on se positionne à l'intérieur pour réaliser les opérations suivantes

```bash
# Création
uv venv

# Positionnement
source .venv/bin/activate
```

Nous lançons l'installation de `crewai` et des éléments annexes nécessaires.

```bash
uv pip install crewai langchain langchain-community openai python-dotenv
```

Enfin, nous pouvons vérifier que l'installation de `crewai`est fonctionnelle

```bash
crewai --version
```

# 4. Création du projet poc

Le projet que nous allons créer s'appel `poc`. L'outil `crewai` fournit une commande pour générer l'arborescence d'un projet.

```bash
crewai create crew poc
```

Pendant la création, nous sélectionnons le provider `openai`. Enfin, il est nécessaire de disposer d'un token pour interragir avec l'API d'openai. Pour cela, il faut se rendre sur ce [site](https://platform.openai.com) afin d'en générer un.

Voici le rendu que l'on doit avoir après avoir remplit toutes les informations demandées

```bash
API keys and model saved to .env file
Selected model: gpt-4
  - Created poc/.gitignore
  - Created poc/pyproject.toml
  - Created poc/README.md
  - Created poc/knowledge/user_preference.txt
  - Created poc/src/poc/__init__.py
  - Created poc/src/poc/main.py
  - Created poc/src/poc/crew.py
  - Created poc/src/poc/tools/custom_tool.py
  - Created poc/src/poc/tools/__init__.py
  - Created poc/src/poc/config/agents.yaml
  - Created poc/src/poc/config/tasks.yaml
Crew poc created successfully!
```

Le projet `poc` porte sur la création d'une équipe de développeur ayant pour objectif la réalisation d'une application web simple. Une page affiche un formulaire avec 4 cases à cocher (oui ou non) qui répondent à des questions :
- Heures de sommeil + 7h
- Prise des compléments alimentaires
- Entraînement
- + 10 000 pas
Ces informations sont ensuite stockées dans une base de données afin de conserver les éléments. Il est possible de répondre aux questions une seule fois par jour et un dashboard permet de voir les résultats quotidient. Tout les éléments seront packagés dans un docker-compose.yml et exécuté avec `docker compose`.
L'équipe de développement sera composée des agents suivants :
- Chef de projet, il définit les étapes du projet, supervise et coordonne l'équipe
- Développeur, il implémente le code (Python Flask)
- Rédacteur, il crée la documentation technique de la solution et la documentation utilisateur

Voici un schéma de la solution et de l'équipe

![Schéma poc crewAI](/images/schema_poc_crewai.png)

