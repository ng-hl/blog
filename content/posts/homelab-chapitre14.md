---
date: '2025-08-13T20:35:47+02:00'
draft: false
title: 'Chapitre 14 - SMTP'
summary: "Homelab - Chapitre 14 - SMTP"
tags: ["homelab","smtp"]
categories: ["homelab"]
---

> Ce document contient les livrables issus de la mise en place du service `Prometheus`, `Grafana` et `AlertsManager`. L'objectif est de pouvoir disposer de services, concernant la collecte de métriques, l'exposition de dashboards et la génération d'alertes avec notifications par mail. Côté architecture, un reverse proyx `nginx` sera en frontal pour porter le certficat wildcard `*.ng-hl.com` et redirigera vers le serveur adéquat. Pour le niveau de maturité niveau 2, le chiffrement TLS sera appliqué uniquement au niveau de l'exposition vers l'extérieur à savoir le `nginx`. Concernant le niveau 3, il faudra mettre en place des communications chiffrés via TLS au travers de certificats fournit par une PKI interne pour adopter le principe d'une architecture 0 trust.

---

# 1. Création de la VM

Nous allons utiliser le template `debian12-template` créé lors du chapitre 4. Sur Proxmox on crée un clone complet à partir de ce template. Voici les caractéristiques de la VM :

| OS      | Hostname     | Adresse IP | Interface réseau | vCPU    | RAM   | Stockage
|:-:    |:-:    |:-:    |:-:    |:-:    |:-:    |:-:
| Debian 12.10     | mail-core    | 192.168.100.241   | vmbr1 (core)    | 1     | 1024   | 20Gio

Il faut également penser à activer la sauvegarde automatique de la VM sur Proxmox en l'ajoutant au niveau de la politique de sauvegarde précédemment créée.

---

# 2. Configuration de l'OS via Ansible

> Les informations concernant Ansible sont disponibles au niveau des chapitres 7 et 8.

A présent, le playbook et les rôles ayant pour objectif d'appliquer la configuration de base de l'OS sont disponibles. Il faut se connecter en tant que l'utilisateur `ansible` sur le serveur `ansible-core.homelab` puis ajouter l'hôte `mail-core.homelab` au niveau du fichier d'inventaire `/opt/ansible/envs/100-core/00_inventory.yml` avec les éléments suivants

```yml
mail-core.homelab:
    ip: 192.168.100.241
    hostname: mail-core
```

Pour exécuter le playbook, il faut lancer la commande suivante

```bash
ansible-playbook -i envs/100-core/00_inventory.yml -l 'mail-core.homelab,' playbooks/00_config_vm.yml
```

Voici le récapitulatif

> Mise à jour de l'inventaire `00_inventory.yml` pour ajouter `mail-core` dans le groupe `prometheus_clients`.

Exécution du playbook pour la mise en place de `Node Exporter`. Il faut également mettre à jour la configuration de `prometheus-core` pour ajouter cet host.

```bash
ansible-playbook playbooks/01_prometheus_node_exporter.yml -l "mail-core.homelab"
```

---

# 3. Installation et configuration de postfix via Ansible

> Afin de commencer à intégrer le principe d'IaC déclarative et reproductible, je choisi d'installer et de configurer `postfix` pour l'instance `mail-core` via un playbook Ansible.

Création du nouveau rôle `serveur_mail`

```bash
ansible-galaxy init roles/serveur_mail
```

Voici le contenu du fichier `tasks/main.yml`

```yaml
# SPDX-License-Identifier: MIT-0
---
# tasks file for roles/serveur_mail

- name: Installation de Postfix et rsyslog
  ansible.builtin.apt:
    name:
      - postfix
      - rsyslog
    state: present
    update_cache: true

- name: Déploiement de la configuration main.cf
  ansible.builtin.template:
    src: main.cf.j2
    dest: /etc/postfix/main.cf
    owner: root
    group: root
    mode: '0644'
  notify: Redémarrer Postfix

- name: Déploiement de la configuration master.cf
  ansible.builtin.template:
    src: master.cf.j2
    dest: /etc/postfix/master.cf
    owner: root
    group: root
    mode: '0644'
  notify: Redémarrer Postfix

- name: Déploiement de la configuration rsyslog
  ansible.builtin.template:
    src: rsyslog.d_postfix.conf.j2
    dest: /etc/rsyslog.d/postfix.conf
    owner: root
    group: root
    mode: '0644'
  notify: Redémarrer rsyslog

- name: Exécution et activation au démarrage du service rsyslog
  ansible.builtin.systemd:
    name: rsyslog
    state: started
    enabled: true

- name: Exécution et activation au démarrage du daemon postfix
  ansible.builtin.systemd:
    name: postfix
    state: started
    enabled: true

```

Voici le contenu du fichier `defaults/main.yml`

```yaml
# SPDX-License-Identifier: MIT-0
---
# defaults file for roles/serveur_mail

serveur_mail_postfix_myhostname: "mail-core.homelab"
serveur_mail_postfix_mydomain: "homelab"
serveur_mail_postfix_mynetworks: "192.168.100.0/24"
serveur_mail_postfix_relayhost: "[smtp.gmail.com]:587"
serveur_mail_postfix_relay_user: "<compte_gmail>"
serveur_mail_postfix_relay_password: "<password>"


```

Voici le contenu du fichier `handlers/main.yml`

```yaml
# SPDX-License-Identifier: MIT-0
---
# handlers file for roles/serveur_mail

- name: Redémarrer rsyslog
  ansible.builtin.systemd:
    name: rsyslog
    state: restarted

- name: Redémarrer Postfix
  ansible.builtin.systemd:
    name: postfix
    state: restarted

```

Voici le contenu du fichier `templates/main.cf.j2`

```j2
myhostname = {{ serveur_mail_postfix_myhostname }}
mydomain = {{ serveur_mail_postfix_mydomain }}
myorigin = $mydomain
inet_interfaces = all
inet_protocols = all
mydestination = $myhostname, localhost.$mydomain, localhost
mynetworks = {{ serveur_mail_postfix_mynetworks }}
relayhost = {{ serveur_mail_postfix_relayhost }}
smtp_use_tls = yes
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_sasl_tls_security_options = noanonymous
smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt

```

Voici le contenu du fichier `templates/master.cf.j2`

```j2
smtp      inet  n       -       y       -       -       smtpd
  -o smtpd_relay_restrictions=permit_mynetworks,reject_unauth_destination
  -o mynetworks={{ serveur_mail_postfix_mynetworks }}
  -o smtpd_tls_security_level=may
  -o smtpd_sasl_auth_enable=no
```

Voici le contenu du fichier `rsyslog.d_postfix.conf.j2`

```j2
mail.*    -/var/log/mail.log
```

Création du playbook `11_serveur_mail.yml`

```yml
---
- name: Installation et configuration du serveur de mail
  hosts: serveur_mail
  become: true
  roles:
    - serveur_mail

```

Mise à jour de l'inventaire avec le groupe `serveur_mail`

```yml
serveur_mail:
    hosts:
        mail-core.homelab:
```

Afin de vérifier que l'on applique bien une syntaxe cohérente et les bonnes pratiques Ansible, on exécute le linter `ansible-lint` sur notre nouveau playbook.

```bash
ansible-lint playbooks/11_serveur_mail.yml 
Passed: 0 failure(s), 0 warning(s) on 6 files.
```

Application du playbook sur l'instance `mail-core`

```bash
ansible-playbook -i envs/100-core/00_inventory.yml -l 'mail-core.homelab,' playbooks/11_serveur_mail.yml
```

Voici le récapitulatif du passage du playbook

```bash
TASKS RECAP **********************************************************************************************************************
mercredi 13 août 2025  22:16:27 +0200 (0:00:00.251)       0:00:03.769 ********* 
=============================================================================== 
Gathering Facts ----------------------------------------------------------------------------------------------------------- 0.93s
serveur_mail : Installation de Postfix et rsyslog ------------------------------------------------------------------------- 0.88s
serveur_mail : Déploiement de la configuration main.cf -------------------------------------------------------------------- 0.42s
serveur_mail : Exécution et activation au démarrage du service rsyslog ---------------------------------------------------- 0.39s
serveur_mail : Déploiement de la configuration rsyslog -------------------------------------------------------------------- 0.36s
serveur_mail : Déploiement de la configuration master.cf ------------------------------------------------------------------ 0.29s
serveur_mail : Redémarrer rsyslog ----------------------------------------------------------------------------------------- 0.25s
serveur_mail : Exécution et activation au démarrage du daemon postfix ----------------------------------------------------- 0.24s
```

---

# 4. Test de l'envoi de mail vers l'extérieur

Installation du paquet `mailutils` qui est un client de messagerie en CLI. 

```bash
sudo apt install mailutils -y
```

Envoi d'un mail de test vers l'extérieur

```bash
echo "Test depuis mail-core" | mail -s "Test mail" <adresse_mail>
```