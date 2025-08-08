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

Après avoir récupéré l'ISO de pfSense sur le site officiel, plus précisément sur le dépôt officiel car il faut s'inscrire pour obtenir l'ISO depuis le site. J'ai installé pfSense avec une configuration classique sans apporter de particularités majeures.

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

# 3. Application du HTTPS et renouvellement automatique du certificat

> A partir du chapitre 9 concernant le certificat wildcard *.ng-hl.com et le service acme, nous passons le serveur pfSense accessible seulement via HTTPS et nous configurons le renouvellement automatique du certificat via acme

Pour activer HTTPS sur pfSense, il suffit d'ajouter le contenu des éléments du certificat à savoir le fullchain et la clé privée associée au format .pem dans la section "System" -> "Certificates".

---

# 4. Renouvellement automatique du certificat (à partir du chapitre 9 - ACME)

> Initialement, j'ai choisi d'utiliser le serveur `acme-core` pour gérer le renouvellement automatique du certificat dans le but de fédérer la partie acme. Après quelques tentatives et dans un soucis de mise en place de solution simple, correctement intégré tout en préservant la sécurité, j'ai choisi d'utiliser la fonctionnalité `acme` nativement disponible au sein de pfSense.

Nous allons utiliser le paquet `acme` présent nativement au sein du gestionnaire de paquet de pfSense. L'objectif est de configurer le client acme pour pouvoir interagir avec le service Cloudflare qui héberge le domaine `ng-hl.com`. 

Tout d'abord, nous installons le paquet `acme`. Se rendre dans "System" -> "Package Manager" -> "Available Packages" -> "acme". À présent, on peut retrouver dans l'onglet "Services" la section "Acme Certificates".

Ensuite, nous créons un `Account Key`. Il faut renseigner un nom et sélectionner le type de `ACME Server`. Pour faire des tests il convient de choisir l'environnement de Staging de Let's Encrypt et pour appliquer en production l'environnement de Production. L'account key est générée automatiquement par pfSense.

Maintenant, nous créons un `Certificate`, il est nécessaire de renseigner les champs utiles et de sélectionner l'Account Key précédemment créée. Dans la section, `Domain SAN list`, nous créons un élément en mode `Enable`, avec le nom de domaine `*.ng-hl.com` et avec la méthode `DNS-Cloudflare`. Il ne nous reste plus qu'à renseigner le `token` qui nous permet d'écrire sur la zone DNS ng-hl.com sur `Cloudflare`. Enfin, nous pouvons également ajouter une action pour recharger le webgui de pfSense lors du passage de acme avec la méthode `Shell Command` et la commande `/etc/rc.restart_webgui`.

> A ce stade, il est utile de faire un snapshot de la machine `pfsense-core.homelab` au niveau de Proxmox VE pour pouvoir revenir en arrière en cas de problème.

Pour exécuter le mécanisme `acme` et générer le certificat (si la date d'expiration le permet), il faut cliquer sur le bouton `Issues/Renew` dans l'onglet `Certificates` du service `acme`.

Pour conclure, il ne nous reste plus qu'à appliquer le bon certificat en se dirigeant vers "System" -> "Advanced". Puis choisir le certificat nouvellement généré dans la liste déroulante.

---

# X. Règles (WIP)

| Date     | Description    | 
|:-:    |:-:    |
| 04/06/2025     | Initialisation du tableau. Etat du homelab au chapitre 10, "Le coffre fort" |

