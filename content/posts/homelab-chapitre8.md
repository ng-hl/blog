---
date: '2025-06-07T23:15:37+02:00'
draft: false
title: 'Chapitre 8 - Configuration de base avec Ansible'
summary: "Homelab - Chapitre 8 - Configuration de base avec Ansible"
tags: ["homelab"]
categories: ["homelab"]
---

> Ce document contient les livrables issus de la mise ne place des roles et playbooks Ansible nécessaires à la configuration des nouvelles VM. L'objectif est de pouvoir disposer d'un ensemble de vm avec une configuration cohérente post-deploiement.

---

# 1. Conversion des tâches de configuration manuelle

> Pour le développement de ces roles et les essais, nous allons déployer une VM à partir du template `debian12-template`. Pour le moment, nous gérerons les rollback des applications du playbook via les snapshot intégrés sur Proxmox VE. Notre VM de test aura les caractéristiques suivantes

| Hostname | IPv4 | Sous réseau |
| :-: | :-: | :-: |
| ansibledev-core | 192.168.100.11 | core |

> Autre élément important, nous réservons l'IP `192.168.100.11` pour la VM temporaire `ansibledev-core`

Avant la mise en place d'Ansible, nous réalisions un ensemble d'opération pour configurer les VM fraîchement créée sur notre Proxmox VE. Nous allons traduire cela en playbook Ansibe.

Modification de la configuration réseau avec le fichier `/etc/network/interfaces`

- [x] Configuration IPv4 de l'interface réseau via `networking` (manuellement)
- [ ] Gestion de la mise à jour du DNS (manuellement)
- [ ] Désactivation de l'IPv6
- [ ] Configuration du hostname
- [ ] Modification de /etc/hosts avec le bon hostname
- [ ] Configuration du résolveur DNS
- [x] Installation des paquets de "base" (ajout)
- [x] Modification du motd (ajout)

Tout d'abord, nous créons l'architecture pour l'ensemble des roles que nous allons créer

```bash
ansible-galaxy init roles/network_config
ansible-galaxy init roles/ipv6_disable
ansible-galaxy init roles/hostname_config
ansible-galaxy init roles/local_dns_config
ansible-galaxy init roles/dns_config
```



