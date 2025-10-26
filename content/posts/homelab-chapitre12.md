---
date: '2025-07-21T20:39:51+02:00'
draft: false
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

Il faut également penser à activer la sauvegarde automatique de la VM sur Proxmox en l'ajoutant au niveau de la politique de sauvegarde précédemment créée.

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

```bash
TASKS RECAP ***************************************************************************************************************
mardi 19 août 2025  14:43:18 +0200 (0:00:00.255)       0:00:15.570 ************ 
=============================================================================== 
base_packages : Installation des paquets de base ------------------------------------------------------------------- 3.40s
dns_config : Autoremove et purge ----------------------------------------------------------------------------------- 2.38s
dns_config : Installation du paquet systemd-resolved --------------------------------------------------------------- 2.07s
base_packages : Mise à jour du cache apt --------------------------------------------------------------------------- 1.78s
Gathering Facts ---------------------------------------------------------------------------------------------------- 1.06s
motd : Déploiement du motd ----------------------------------------------------------------------------------------- 0.46s
hostname_config : Modification du hostname ------------------------------------------------------------------------- 0.46s
dns_config : Enable du daemon systemd-resolved --------------------------------------------------------------------- 0.39s
nftables : Activer et démarrer le service nftables ----------------------------------------------------------------- 0.39s
dns_config : Suppression du paquet resolvconf ---------------------------------------------------------------------- 0.38s
nftables : Déploiement de la configuration de nftables ------------------------------------------------------------- 0.31s
dns_config : Resart du daemon systemd-resolved --------------------------------------------------------------------- 0.30s
security_ssh : Restart du daemon sshd ------------------------------------------------------------------------------ 0.26s
nftables : Valider la configuration nftables ----------------------------------------------------------------------- 0.21s
hostname_config : Modification du fichier /etc/hosts --------------------------------------------------------------- 0.20s
ipv6_disable : Désactivation de la prise en charge de l'IPv6 globalement ------------------------------------------- 0.20s
dns_config : Suppression du fichier /etc/resolv.conf --------------------------------------------------------------- 0.19s
dns_config : Configuration du DNS dans /etc/resolved.conf ---------------------------------------------------------- 0.19s
ipv6_disable : Désactivation de la prise en charge de l'IPv6 par défaut -------------------------------------------- 0.14s
dns_config : Création du nouveau lien symbolique vers /etc/resolv.conf --------------------------------------------- 0.14s
```

---

# 3. Installation et configuration de OpenTofu

Mise à jour du cache apt

```bash
sudo apt update
```

Installation des package en prérequis

```bash
sudo apt install -y apt-transport-https ca-certificates curl gnupg
```

Ajout des dépôts de OpenTofu

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

```bash
curl -fsSL https://get.opentofu.org/opentofu.gpg | sudo tee /etc/apt/keyrings/opentofu.gpg >/dev/null
```

```bash
curl -fsSL https://packages.opentofu.org/opentofu/tofu/gpgkey | sudo gpg --no-tty --batch --dearmor -o /etc/apt/keyrings/opentofu-repo.gpg >/dev/null
```

```bash
sudo chmod a+r /etc/apt/keyrings/opentofu.gpg /etc/apt/keyrings/opentofu-repo.gpg
```

Ajout du dépôt dans la source list apt

```bash
echo \
  "deb [signed-by=/etc/apt/keyrings/opentofu.gpg,/etc/apt/keyrings/opentofu-repo.gpg] https://packages.opentofu.org/opentofu/tofu/any/ any main
deb-src [signed-by=/etc/apt/keyrings/opentofu.gpg,/etc/apt/keyrings/opentofu-repo.gpg] https://packages.opentofu.org/opentofu/tofu/any/ any main" | \
  sudo tee /etc/apt/sources.list.d/opentofu.list > /dev/null
```

```bash
sudo chmod a+r /etc/apt/sources.list.d/opentofu.list
```

Mise à jour du cache apt 

```bash
sudo apt update
```

Installation du paquet `tofu`

```bash
sudo apt install -y tofu
```

Vérification de l'installation et de la version

```bash
tofu --version
OpenTofu v1.10.5
on linux_amd64
```

---

# 4. Lien avec Proxmox VE

> Afin de pouvoir interagir avec notre Proxmox VE, un compte utilisateur est nécessaire. La bonne pratique est de créer un compte utilisateur dédié, basé sur le principe du moindre privilège avec un token.

> Les commandes ci-dessous sont à exécutées en tant que root sur le node `pve` Proxmox VE

Création de l'utilisateur `opentofu-deploy`

```bash
pveum useradd opentofu-deploy@pve --password "MyPassword"
```

Création du token `opentofu-token` associé à l'utilisateur `opentofu-deploy`. On veille à le conserver dans notre coffre fort.

```bash
pveum user token add opentofu-deploy@pve opentofu-token --comment "Token OpenTofu" --privsep
```

Création du rôle `Opentofu-role` dans lequel on applique les droits nécessaires

```bash
pveum roleadd Opentofu-role -privs "VM.Allocate,VM.Audit,VM.Config.CDROM,VM.Config.Disk,VM.Config.Memory,VM.Config.CPU,VM.Config.Network,VM.Config.Options,VM.Monitor,VM.clone,VM.PowerMgmt"
```

Rattachement de l'utilisateur `opentofu-deploy` au rôle `Opentofu-role`

```bash
pveum aclmod / -user opentofu-deploy@pve -role Opentofu-role
```

> Les commandes ci-dessous sont à exécutées sur le serveur `opentofu-core`

Création du répertoire dédiée pour `Opentofu`

```bash
mkdir -p /opt/opentofu
```

Initialiser le dépôt Git

```bash
git init
```

Création du fichier `.gitignore` avec les éléments ci-dessous

```bash
*.tfstate
*.tfstate.backup
*.tfvars
.terraform/
variables.tf
```

Création du fichier de configuration du provider `/opt/opentofu/provider.tf`

```hcl
terraform {
    required_providers {
        proxmox = {
        source = "bpg/proxmox"
        version = "0.85.1"
        }
    }
}

provider "proxmox" {
    endpoint = "https://pve.ng-hl.com:8006/"
    api_token = var.api_token
    insecure = false
}
```

Création du fichier contenant les variables `/opt/opentofu/variables.tf`. Il est également possible de saisir la valeur du token via une variable d'environnement, cela évite d'avoir un token en clair directement dans un fichier.

```hcl
variable "api_token" {
  description = "Token to connect Proxmox API"
  type        = string
value         = "XXXXXXXXXXXXX"
}
```

Positionnement au niveau du répertoire du projet et initialisation de Opentofu

```bash
tofu init
```

Tester la communication avec l'instance `Proxmox`

```bash
tofu plan

No changes. Your infrastructure matches the configuration.

OpenTofu has compared your real infrastructure against your configuration and found no differences, so no changes
are needed.
```

---

# 5. Déploiement et destruction d'une ressource

Création du fichier principal `/opt/opentofu/main.tf`. L'objectif est de déployer une VM de test à partir du template `debian12-template`

```hcl

```
