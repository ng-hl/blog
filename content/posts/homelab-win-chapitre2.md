---
date: '2025-10-28T14:44:36+01:00'
draft: false
title: 'Chapitre 2 - Les templates Windows'
summary: "Homelab - Chapitre 2 - Les templactes Windows"
tags: ["homelab-windows"]
categories: ["Homelab Windows"]
---

> Ce document contient les livrables issus de la phase de création des templates pour les OS `Windows Server 2022` et `Windows 11`. L'objectif est de pouvoir disposer de templates de VM afin de créer facilement des clones complets pour avoir une configuration de base pour l'ensemble de nos VM (mises à jour, préférence du système, installation de l'OS, ...).

---

# 1. Import des images ISO

Nous avons besoin des ISO suivants pour réaliser la création de nos templates.

| Distribution      | Version     | Adresse de téléchargement de l'ISO
|:-:    |:-:    |:-:
| Windows Server    | 2022      | [Download](https://www.microsoft.com/fr-fr/evalcenter/download-windows-server-2022)
| Windows Professionnel     | 11      | [Download](https://www.microsoft.com/fr-fr/software-download/windows11)

---

# 2. Création des templates

Au niveau de Proxmox, lors de la création des VM nous allons attribuer les ID 10011 et 10012 (la convention de nommage pour les templates 1001X pour notre homelab Windows), activer le qemu guest agent, créer un disque SCSI de 20Go avec émulation SSD, 2 vCPU, 4096 Mo de RAM avec le ballooning activé, une carte réseau positionnée sur le vmbr0 pour avoir un accès par pont au réseau local puis le tag `templates`. Les éléments suivants vont être configurés pour les deux VM créées à partir des ISO précédemment récupérés.

| Item      | Windows Server 2022     | Windows 11
|:-:    |:-:    |:-:
| Agent QEMU   | Installé (qemu-guest-agent package)     | Installé (qemu-guest-agent package) 
| Hostname     | win2022-template      | win11-template
| Partitionnement     | LVM (/boot 512Mo, / 10Go, /home 3Go, /var 5Go, SWAP 1Go )      | LVM (/boot 512Mo, / 10Go, /home 3Go, /var 5Go, SWAP 1Go )

# 3. Configuration basique de l'OS

Voici la procédure utilisée pour configurer l'OS qui va servir de template

## 3.1 Changer le hostname

```powershell

```

## 3.1 Mise à jour de l'OS

```powershell

```

## 3.2 Activation de RDP / WinRM

```powershell

```

## 3.3 Création de l'utilisateur ngobert

```powershell

```

> À partir du chapitre 10 du `Homelab VM-Factory`, "Coffre fort", nous pouvons nous servir de la solution `VaultWarden` pour stocker l'identité précédemment créée.

## 3.4 Configuration temporaire du réseau

```powershell

```

---

# 4. Sysprep et création du template sous Proxmox VE

