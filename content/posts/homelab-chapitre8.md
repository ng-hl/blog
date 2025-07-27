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
- [x] Gestion de la mise à jour du DNS (manuellement)
- [x] Désactivation de l'IPv6
- [x] Configuration du hostname
- [x] Modification de /etc/hosts avec le bon hostname
- [x] Configuration du résolveur DNS
- [x] Installation des paquets de "base" (ajout)
- [x] Modification du motd (ajout)

Tout d'abord, nous créons l'architecture pour l'ensemble des roles que nous allons créer

```bash
ansible-galaxy init roles/ipv6_disable
ansible-galaxy init roles/hostname_config
ansible-galaxy init roles/dns_config
```

Voici le détail des roles ci-dessus. Pour commencer le rôle `ipv6_disable`

```bash
#SPDX-License-Identifier: MIT-0
---
# tasks file for roles/ipv6_disable

- name: Désactivation de la prise en charge de l\'IPv6 globalement
  ansible.posix.sysctl:
    name: net.ipv6.conf.all.disable_ipv6
    value: 1
    state: present
    sysctl_set: yes
    reload: true

- name: Désactivation de la prise en charge de l\'IPv6 par défaut
  ansible.posix.sysctl:
    name: net.ipv6.conf.default.disable_ipv6
    value: 1
    state: present
    sysctl_set: yes
    reload: true
```

Ensuite, le role `hostname_config`. La variable `hostname` est initialisée au niveau de l'inventaire.

```bash
#SPDX-License-Identifier: MIT-0
---
# tasks file for roles/hostname_config

- name: Modification du hostname
  ansible.builtin.hostname:
    name: '{{ hostname }}' 
    use: systemd

- name: Modification du fichier /etc/hosts
  ansible.builtin.lineinfile:
    path: /etc/hosts
    regexp: '^127\.0\.1\.1\s+'
    line: "127.0.1.1 {{ hostname }}.homelab {{ hostname }}"
    state: present
    backrefs: true
```

Poursuivons avec le role `dns_config`. Pour ce role ci, l'indenpotence n'est pas respectée. En effet, quelque soit l'état des éléments, le restart du service `systemd-resolved`, la suppression du fichier `/etc/resolv.conf` et le création du lien symbolique sont des actions exécutées à chaque déclenchement du role.

```bash
#SPDX-License-Identifier: MIT-0
---
# tasks file for roles/dns_config

- name: Installation du paquet systemd-resolved
  ansible.builtin.apt:
    name: systemd-resolved
    state: present

- name: Suppression du paquet resolvconf
  ansible.builtin.apt:
    name: resolvconf
    state: absent

- name: Autoremove et purge
  ansible.builtin.apt:
    autoremove: yes
    purge: true

- name: Enable du daemon systemd-resolved
  ansible.builtin.systemd_service:
    name: systemd-resolved
    enabled: true

- name: Configuration du DNS dans /etc/resolved.conf
  ansible.builtin.ini_file:
    path: /etc/systemd/resolved.conf
    section: "Resolve"
    option: "DNS"
    value: "192.168.100.253"
    state: present

- name: Configuration du FallbackDNS dans /etc/resolved.conf
  ansible.builtin.ini_file:
    path: /etc/systemd/resolved.conf
    section: "Resolve"
    option: "FallbackDNS"
    value: "1.1.1.1"
    state: present

- name: Configuration du domaine dans /etc/resolved.conf
  ansible.builtin.ini_file:
    path: /etc/systemd/resolved.conf
    section: "Resolve"
    option: "Domains"
    value: "~."
    state: present

- name: Resart du daemon systemd-resolved
  ansible.builtin.systemd_service:
    name: systemd-resolved
    state: restarted

- name: Suppression du fichier /etc/resolv.conf
  ansible.builtin.file:
    path: /etc/resolv.conf
    state: absent

- name: Création du nouveau lien symbolique vers /etc/resolv.conf
  ansible.builtin.file:
    src: /run/systemd/resolve/stub-resolv.conf
    dest: /etc/resolv.conf
    state: link
```

Enfin, les deux derniers rôles ajoutés pour l'occasion, `base_package` et `motd`

```bash
#SPDX-License-Identifier: MIT-0
---
# tasks file for roles/base_packages
# Rôle permettant l'installation des paquets de base

- name: Mise à jour du cache apt
  ansible.builtin.apt:
    update_cache: yes
    cache_valid_time: 3600

- name: Installation des paquets de base
  ansible.builtin.apt:
    name:
      - curl
      - wget
      - tmux
      - htop
      - jq
      - tree
      - git
      - figlet
    state: present

#SPDX-License-Identifier: MIT-0
---
# tasks file for roles/motd
# Rôle permettant la modification du motd

- name: Déploiement du motd
  ansible.builtin.template:
    src: motd.j2
    dest: /etc/motd
    mode: '0644'
```

Avec le fichier template `motd.j2`

```bash
########################################################
#   Hostname     : {{ ansible_hostname }}
#   IP Address   : {{ ansible_default_ipv4.address }}
#   Network Zone : {{ 'core' if 'core' in inventory_hostname else 'vms' if 'vms' in inventory_hostname else 'unknown' }}
#   Uptime       : {{ (ansible_uptime_seconds / 60) | int }} minutes
#   Disk Usage   : {{ ansible_mounts[0].size_available | human_readable }}/{{ ansible_mounts[0].size_total | human_readable }}
########################################################
```

---

# 2. Ajout du linter

Nous choisissions d'utiliser `ansible-lint` pour vérifier et corriger le code Ansible que je produit. Il s'agit de l'outil officiellement utilisé par `Ansible Galaxy`, l'outil est simple d'utilisation, permet de surveiller un certains nombre d'éléments comme la syntaxe, les bonnes pratiques syntaxiques, le respect de l'indempotence, etc. De plus, l'outil permet d'appliquer des corrections avec l'option `--fix` (anciennement `--write`). Nous installons également `yamllint`qui permet d'appliquer des vérification mais sur la syntaxe et les bonnes pratiques concernant le `yaml`.

Nous avons déjà installé `ansible-lint` pendant l'installation et la configuration de `Ansible` au niveau du chapitre précédent (chapitre 7). Pour rappel la commande utilisée est celle-ci (à appliquer dans le contexte pipx de Ansible).

```bash
pipx install "ansible-lint[yamllint]"
```

Nous exécutons `ansible-lint` couplé à `yamllint` sur le playbook précédemment créé `00_config_vm.yml`

```bash
ansible-lint playbooks/00_config_vm.yml
```

Voici le récapitulatif des résultats obtenus :

```bash
# Rule Violation Summary

  1 syntax-check profile:min tags:core,unskippable
  6 schema profile:min tags:core
 29 yaml profile:min tags:formatting,yaml
  1 yaml profile:min tags:formatting,yaml
  2 yaml profile:min tags:formatting,yaml
  2 yaml profile:min tags:formatting,yaml
  3 yaml profile:min tags:formatting,yaml
 15 meta-incorrect profile:min tags:metadata
  3 fqcn profile:min tags:formatting

Failed: 62 failure(s), 0 warning(s) on 31 files.
```

Après application du `fix`, il reste 51 remontées.

```bash
# Rule Violation Summary

  1 syntax-check profile:min tags:core,unskippable
  6 schema profile:min tags:core
 29 yaml profile:min tags:formatting,yaml
 15 meta-incorrect profile:min tags:metadata

Failed: 51 failure(s), 0 warning(s) on 31 files.
```

Ci-dessous, un exemple de modification concernant les meta informations du rôle `security_ssh`. Voici le fichier avant les modifications :

```bash
---
#SPDX-License-Identifier: MIT-0
galaxy_info:
  author: your name
  description: your role description
  company: your company (optional)

  # If the issue tracker for your role is not on github, uncomment the
  # next line and provide a value
  # issue_tracker_url: http://example.com/issue/tracker

  # Choose a valid license ID from https://spdx.org - some suggested licenses:
  # - BSD-3-Clause (default)
  # - MIT
  # - GPL-2.0-or-later
  # - GPL-3.0-only
  # - Apache-2.0
  # - CC-BY-4.0
  license: license (GPL-2.0-or-later, MIT, etc)
```

Voici le fichier après les modifications :

```bash
---
#SPDX-License-Identifier: MIT-0
galaxy_info:
  author: Nicolas Gobert
  description: Sécurisation du daemon sshd
  company: Homelab

  # If the issue tracker for your role is not on github, uncomment the
  # next line and provide a value
  # issue_tracker_url: http://example.com/issue/tracker

  # Choose a valid license ID from https://spdx.org - some suggested licenses:
  # - BSD-3-Clause (default)
  # - MIT
  # - GPL-2.0-or-later
  # - GPL-3.0-only
  # - Apache-2.0
  # - CC-BY-4.0
  license: MIT
```

