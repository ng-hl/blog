---
date: '2025-06-04T17:19:25+02:00'
draft: false
title: 'Chapitre 4 - Les templates (Debian 12 et Rocky Linux 9)'
summary: "Homelab - Chapitre 4 - Les templates (Debian 12 et Rocky Linux 9)"
tags: ["homelab"]
categories: ["homelab"]
---

> Ce document contient les livrables issus de la phase de création des templates pour les OS Debian 12 et Rocky Linux 9. L'objectif est de pouvoir disposer de templates de VM afin de créer facilement des clônes complets pour avoir une configuration de base pour l'ensemble de nos VM (utilisateur, clés SSH, installation des paquets de base, ...)

---

# 1. Import des images ISO

Nous avons besoin des ISO suivants pour réaliser la création de nos templates.

| Distribution      | Version     | Adresse de téléchargement de l'ISO
|:-:    |:-:    |:-:
| Debian     | 12.10      | [Download]()
| Rocky Linux     | 9.5      | [Download](https://download.rockylinux.org/pub/rocky/9/isos/x86_64/Rocky-9.5-x86_64-dvd.iso)

---

# 2. Création des templates

Les opérations sont les mêmes pour les deux distributions. Au niveau de Proxmox lors de la création des VM nous allons attribuer les ID 10001 et 10002 (la convention de nommage pour les templates 1000X pour notre homelab), activer le qemu guest agent, créer un disque SCSi de 20Go avec émulation SSD, 1 vCPU, 2048 Mo de RAM avec le balloning activé, une carte réseau positionnée sur le vmbr0 pour avoir un accés par pont au réseau local puis le tag `templates`. Les éléments suivants vont être configurés pour les deux VM créées à partir des ISO précédement récupérés.

| Item      | Debian 12.10     | Rocky Linux 9.5
|:-:    |:-:    |:-:
| Agent QEMU   | Installé (qemu-guest-agent package)     | Installé (qemu-guest-agent package) 
| Hostmane     | debian12-template      | rocky9-template
| Domaine      | homelab                | homelab
| Partitionnement     | LVM (/boot 512Mo, / 10Go, /home 3Go, /var 5Go, SWAP 1Go )      | LVM (/boot 512Mo, / 10Go, /home 3Go, /var 5Go, SWAP 1Go )

# 3. Configuration basique de l'OS

Voici la procédure utilisée pour configurer l'OS qui va servir de template

## 3.1 Mise à jour de l'OS
```bash
# Mise à jour du cache du gestionnaire de paquet et mise à jour des packages
# Debian12
apt update && apt upgrade -y

# Rocky9
sudo yum update -y
```

## 3.2 Installation de SSH et Sudo

```bash
# Installation du serveur openssh et sudo
apt install -y openssh-server sudo
```

## 3.3 Création de l'utilisateur ansible et gestion de la paire de clé associée

```bash
# Création de l'utilisateur ansible
sudo useradd --create-home --shell /bin/bash --groups sudo ansible
```

Nous allons créer les paires de clés pour l'utilisateur d'administration `ngobert` ainsi que pour l'utilisateur `ansible`.

```bash
sudo mkdir /root/identites
sudo ssh-keygen -t ed25519 -f /root/identites/id_admin -C "Utilisateur d'administration"
sudo ssh-keygen -t ed25519 -f /root/identites/id_ansible -C "Utilisateur Ansible"
```

> A partir du chapitre 10, "Coffre fort", nous pourrons nous servir de la solution `VaultWarden`pour stocker les clés SSH précédemment générées.

Ces clés doivent être stockées dans un environnement sécurisée. On peut autoriser notre clé privée à se connecter sur les machines en collant le contenu de la clé publique (*.pub) au niveau du fichier `/home/ngobert/.ssh/authorized_keys` et du fichier `/home/ansible/.ssh/authorized_keys`

## 3.4 Configuration temporaire du réseau

```bash
# Configuration du réseau (adresse IP standard non utilisable dans ce contexte)
# Debian12
sudo vim /etc/network/interfaces
[...]
auto eth0
iface eth0 inet static
    address 192.168.100.10/24 # Pour Debian12
    gateway 192.168.100.254

# Rocky9
nmtui
```

## 3.5 Configuration de sudo pour l'utilisateur ansible

Enfin, nous offrons la possibilité à l'utilisateur `ansible` d'exécuter toutes les commandes avec `sudo` sans demande de saisie du mot de passe. Pour cela, nous créons le fichier `/etc/sudoers.d/ansible` avec la ligne suivante

```bash
ansible ALL=(ALL) NOPASSWD: ALL
```

## 3.6 Durcissement de la configuration du daemon sshd

Nous modifions la configuration du daemon sshd pour améliorer la sécurité en autorisant uniquement la connexion ssh avec les utilisateurs `ngobert` et `root` avec une clé, on écoute uniquement sur l'interface choisie, etc.

Voici le contenu du fichier `/etc/ssh/sshd_config`

```bash
Include /etc/ssh/sshd_config.d/*.conf

Port 22
ListenAddress 192.168.100.10
LogLevel INFO
PermitRootLogin prohibit-password
PubkeyAuthentication yes
PasswordAuthentication no
PermitEmptyPasswords no
KbdInteractiveAuthentication no
UsePAM yes
AllowAgentForwarding yes
AllowTcpForwarding yes
X11Forwarding yes
PrintMotd no
PrintlastLog no
PermitTunnel no
AcceptEnv LANG LC_*
Subsystem       sftp    /usr/lib/openssh/sftp-server
AllowUsers ngobert root ansible
```

## 3.7 Mise en place de fail2ban

Toujours dans l'optique d'améliorer la sécurité, nous installons et configurons la solution `fail2ban` pour SSH

```bash
sudo apt install -y fail2ban
```

Nous créons le fichier `/etc/fail2ban/jail.d/ssh.conf` avec les éléments suivants

```bash
[sshd]
enabled = true
port    = ssh
filter  = sshd
logpath = /var/log/auth.log
bantime = 3600
findtime = 600
maxretry = 5
backend = systemd
```

Nous activons le daemon au démarrage du système et on l'exécute maintenant

```bash
sudo systemctl enable fail2ban
sudo systemctl restart fail2ban
```

## 3.8 Mise en place de nftables

Enfin, pour durcir davantage la sécurité du serveur, nous installons et configurons le firewall `nftables`.

```bash
sudo apt install -y nftables
```

Nous activons le daemon nftables au démarrage du système

```bash
sudo systemctl enable nftables
```

Nous modifions sommairement le fichier de configuration de nftables pour appliquer une autorisation vers le port 22 pour les connexions ssh

```bash
#!/usr/sbin/nft -f

table inet filter {
  chain input {
    type filter hook input priority 0;
    policy drop;
    iif lo accept
    ct state established,related accept

    ip protocol icmp accept

    ip saddr 192.168.100.252 iifname "ens19" tcp dport 22 accept
  }

  chain forward {
    type filter hook forward priority 0;
    policy drop;
  }

  chain output {
    type filter hook output priority 0;
    policy accept;
  }
}
```

Nous vérifions la syntaxe du fichier de configuration `/etc/nftables.conf`, puis nous appliquons les modifications en faisant un restart du daemon

```bash
sudo nft -f /etc/nftables.conf
sudo systemctl restart nftables
```