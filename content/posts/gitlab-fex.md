---
date: '2025-07-21T21:10:21+02:00'
draft: false
title: 'FEX - Gitlab'
summary: "FEX - Gitlab"
tags: ["fex","gitlab"]
categories: ["fex"]
---

> Ce document est une fiche d'exploitation pour l'administration d'un serveur Gitlab via la CLI.

---

Ci-dessous, les répertoires et fichiers utiles

```bash
/etc/gitlab/ # Configuration
/var/log/gitlab/ # Logs
/var/opt/gitlab # Données persistentes
/var/opt/gitlab/backups # Backups
```

Ci-dessous, les commandes utiles pour interagir avec Gitlab en CLI

```bash
# Voir le status d'un ou des services
gitlab-ctl status
gitlab-ctl status <service>

# Restart d'un ou des services
gitlab-ctl restart
gitlab-ctl restart <service>

# Regénération de la configuration (appui sur /etc/gitlab/gitlab.rb) et restart des services manquants
gitlab-ctl reconfigure

# Voir les logs d'un ou des services
gitlab-ctl tail
gitlab-ctl tail <service>

# Voir les informations système
gitlab-rake gitlab:env:info

# Créer un fichier de backup
gitlab-backup create
# Restaurer un fichier de backup
gitlab-backup restore BACKUP=<file>

# Vérification globale 
gitlab-rake gitlab:check
```