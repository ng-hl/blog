---
date: '2025-06-04T17:14:57+02:00'
draft: false
title: 'Chapitre 3 - pfSense'
summary: "Homelab - Chapitre 3 - pfSense"
tags: ["homelab"]
categories: ["homelab"]
---

> Ce document contient les livrables issus de la phase d'installation et de configuration de pfSense. L'objectif est de pouvoir disposer d'une solution permettant de gérer simplement et efficacement les ouvertures réseaux entre les différents sous-réseaux du homelab mais également en provenance et à destination du WAN. L'occasion parfaite pour jouer un peu avec pfSense.

> La section `Règles` évolue au fur et à mesure de l'évolution du homelab

---

# 1. Installation de pfSense

Aprés avoir récupéré l'ISO de pfSense sur le site officiel, plus précisément sur le dépôt officiel car il faut s'inscrire pour obtenir l'ISO depuis le site. J'ai installé pfSense avec une configuration classique sans apporter de particularités majeures.

---

# 2. Configuration de pfSense

Ce que l'on souhaite, c'est l'association interfaces / réseau ci-dessous :

| Interface      | Réseau pfSense     | Sous-réseau | Adresse
|:-:    |:-:    |:-:    |:-:
| vmbr0     | WAN      | Réseau local (réseau local) | DHCP
| vmbr1     | CORE      | Réseau Core (homelab) | 192.168.100.254
| vmbr2     | VMS     | Réseau VMS (homelab) | 192.168.200.254

![Interfaces configuration](/images/interfaces-configuration.png)

Il est important de désactiver les options `Block private networks and loopback addresses` et `Block bogon networks` car l'interface WAN se trouve sur un réseau local en 192.168.1.0/24.

Afin de rendre l'interface webgui de pfSense accessible depuis le WAN, il est nécessaire d'ajouter une règle au niveau de cette interface.

![Imageswebgui rule](/images/webgui-rule.png)

---

# Application du HTTPS et renouvellement automatique du certificat

> A partir du chapitre 9 concernant le certificat wildcard *.ng-hl.com et le service acme, nous passons le serveur pfSense accessible seulement via HTTPS et nous configurons le renouvellement automatique du certificat via acme

Pour activé HTTPS sur pfSense, il suffit d'ajouter le contenu des éléments du certificat à savoir le fullchain et la clé privée associé au format .pem. dans la section "System" -> "Certificates".

Concernant le renouvellement automatique, il faut tout d'abord activer l'accés en SSH sur notre pfSense. Pour cela, se rendre dans la section "System" -> "Advanced" -> "Admin Access" pour activer "Secure Shell Server" et préciser le public key only. De plus, il est nécessaire d'ouvrir le flux en provenance de `acme-core` sur l'adresse de pfSense de l'interface core `192.168.100.254` sur le port tcp/22.



---

# 3. Règles (WIP)

| Date     | Description    | 
|:-:    |:-:    |
| 04/06/2025     | Initialisation du tableau. Etat du homelab au chapitre 10, "Le coffre fort" |

