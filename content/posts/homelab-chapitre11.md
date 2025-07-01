---
date: '2025-06-30T22:05:38+02:00'
draft: false
title: 'Chapitre 11 - Gitlab'
summary: "Homelab - Chapitre 11 - Gitlab"
tags: ["homelab"]
categories: ["homelab"]
---

> Ce document contient les livrables issus de la mise en place du service `Gitlab`. L'objectif est de pouvoir disposer d'un serveur pour versionner le code de l'infrastructure mais également divers projets.

---

# 1. Création de la VM

Nous allons utiliser le template `debian12-template` créé lors du chapitre 4. Sur Proxmox on créé un clone complet à partir de ce template. Voici les caractéristiques de la VM :

| OS      | Hostname     | Adresse IP | Interface réseau | vCPU    | RAM   | Stockage
|:-:    |:-:    |:-:    |:-:    |:-:    |:-:    |:-:
| Debian 12.10     | gitlab-core      | 192.168.100.247    | vmbr1 (core)    | 2     | 4096   | 20Gio

Il faut également penser à activer la sauvegarde automatique de la VM sur Proxmox en l'ajoutant au niveau de la politique de sauvegarde précédemment créée.

---

# 2. Configuration de l'OS via Ansible

> Les informations concernant Ansible sont disponibles au niveau des chapitres 7 et 8.

A présent, le playbook et les roles ayant pour objectif d'appliquer la configuration de base de l'OS sont disponible. Il faut se connecter en tant que l'utilisateur `ansible` sur le serveur `ansible-core.homelab` puis ajouter l'hôte `gitlab-core.homelab` au niveau du fichier d'inventaire `/opt/ansible/envs/100-core/00_inventory.yml` avec les éléments suivants

```yml
gitlab-core.homelab:
    ip: 192.168.100.247
    hostname: gitlab-core
```

Il est nécessaire d'ajouter les droits sudo sur l'utilisateur `ansible` au niveau du fichier `/etc/sudoers.d/ansible` avec les éléments ci-dessous. Il s'agit d'un oubli au niveau du template. (A corriger plus tard).

```bash
ansible ALL=(ALL) NOPASSWD: ALL
```

Pour exécuter le playbook, il faut lancer la commande suivante

```bash
ansible-playbook -i envs/100-core/00_inventory.yml -l 'gitlab-core.homelab,' playbooks/00_config_vm.yml
```

Voici le récapitulatif

```bash
lundi 30 juin 2025  22:25:38 +0200 (0:00:00.255)       0:00:06.584 ************ 
=============================================================================== 
Gathering Facts ----------------------------------------------------------------------------------------------------------- 0.99s
dns_config : Autoremove et purge ------------------------------------------------------------------------------------------ 0.56s
dns_config : Installation du paquet systemd-resolved ---------------------------------------------------------------------- 0.45s
base_packages : Installation des paquets de base -------------------------------------------------------------------------- 0.45s
hostname_config : Modification du hostname -------------------------------------------------------------------------------- 0.43s
motd : Déploiement du motd ------------------------------------------------------------------------------------------------ 0.43s
base_packages : Mise à jour du cache apt ---------------------------------------------------------------------------------- 0.38s
dns_config : Enable du daemon systemd-resolved ---------------------------------------------------------------------------- 0.38s
dns_config : Suppression du paquet resolvconf ----------------------------------------------------------------------------- 0.33s
dns_config : Resart du daemon systemd-resolved ---------------------------------------------------------------------------- 0.29s
security_ssh : Restart du daemon sshd ------------------------------------------------------------------------------------- 0.26s
dns_config : Suppression du fichier /etc/resolv.conf ---------------------------------------------------------------------- 0.18s
dns_config : Configuration du DNS dans /etc/resolved.conf ----------------------------------------------------------------- 0.18s
hostname_config : Modification du fichier /etc/hosts ---------------------------------------------------------------------- 0.18s
ipv6_disable : Désactivation de la prise en charge de l'IPv6 globalement -------------------------------------------------- 0.18s
security_ssh : Activation de l'authentification par clé ------------------------------------------------------------------- 0.13s
ipv6_disable : Désactivation de la prise en charge de l'IPv6 par défaut --------------------------------------------------- 0.13s
dns_config : Création du nouveau lien symbolique vers /etc/resolv.conf ---------------------------------------------------- 0.13s
security_ssh : Désactivaction de l'authentification par mot de passe ------------------------------------------------------ 0.13s
dns_config : Configuration du FallbackDNS dans /etc/resolved.conf --------------------------------------------------------- 0.13s
```

---

# 3. Installation de Gitlab

> Avant de procéder à l'installation, il faut ajouter les entrée ̀`gitlab-core.ng-hl.com` et l'alias `gitlab.ng-hl.com`.

Pour installer Gitlab proprement, on suit la [documentation officielle](https://about.gitlab.com/install/#debian)

```bash
sudo apt-get install -y curl openssh-server ca-certificates perl postfix
```

```bash
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
```

```bash
sudo EXTERNAL_URL="https://gitlab.ng-hl.com" apt-get install gitlab-ce
```

On peut se connecter sur l'interface en HTTPS (avec un certificat auto-signé pour le moment) en tant que `root`. La première chose à faire est de créer un autre utilisateur que l'on va appeler `ngobert` avec les droits administrateur puis on désactive la fonctionnalité qui permet la création de nouveau compte depuis l'interface de connexion.

Enfin, pour que notre certificat wildcard *.ng-hl.com puisse porter le HTTPS, on récupère les fichiers de certificat et la clé privée pour les positionner au niveau du répertoire `/etc/gitlab/ssl` sur le serveur `gitlab-core.homelab` avec respectivement les noms `gitlab.ng-hl.com.crt` et `gitlab.ng-hl.com.key` puis on recharge gitlab avec les commandes ci-dessous.

```bash
sudo gitlab-ctl reconfigure
sudo gitlab-ctl restart
```