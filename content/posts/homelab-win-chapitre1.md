---
date: '2025-10-28T13:43:14+01:00'
draft: false
title: 'Chapitre 1 - Le design Windows'
summary: "Homelab - Chapitre 1 - Le design Windows"
tags: ["homelab-windows"]
categories: ["Homelab Windows"]
---

> Ce document contient les livrables issus de la phase de design du homelab concernant la partie Windows. On doit se poser les bonnes questions pour répondre efficacement au besoin de départ, à savoir, disposer d'un environnement de sandbox `Windows` avec `Active Directory`. La conception est susceptible de changer au fur et à mesure des travaux, cette page est donc susceptible d'évoluer (mise à jour de l'inventaire, ajout de services/fonctionnalités, ...).

> Important ! Les informations relatif au hardware et au socle du Homelab comme pfSense par exemple, sont disponibles dans la catégorie `Homelab` et dans le `Chapitre 1 - Le design` concernant le homelab `VM-Factory`.

---

# 1. Les environnements

Le homelab va disposer de trois sous-réseaux (voir les chapitres concernant la conception du Homelab `VM-Factory`). Sur la série `Homelab Windows`, nous nous concentrons sur le sous-réseau `WIN`.

| Nom      | Description      | Adressage      |
|:-:    |---    |---    |
| Core      | Environnement de base du homelab (prod)      | 192.168.100.0/24      |
| VMS      | Environnement de déploiement des VMs (sandbox)     | 192.168.200.0/24      |
| WIN      | Environnement Windows     | 172.16.1.0/24      |

---

# 2. Les services

> De nombreux services comme Ansible, Grafana, Prometheus, etc. sont déjà détaillés dans la catégorie `Homelab VM-Factory`.

Pour disposer d'un environnement `Active Directory` fonctionnel et complet, nous avons besoin de différents services que l'on va détailler dans les sous-sections suivantes.

## 2.1. Annuaire Active Directory

L'annuaire Active Directory est un service qui va nous permettre de gérer un ensemble de ressources comme des comptes ordinateurs, des comptes utilisateurs, des GPO, etc. Ce service est intégré dans le catalogue de service proposé par Windows Server.

## 2.2. Service DNS

Le DNS va nous permettre d'utiliser les noms associés à nos VM plutôt que les IP avec deux zones DNS `win.ng-hl.local` (DNS interne du homelab) ainsi que `win.ng-hl.com` (le domaine qui portera les services exposés sur mon réseau local). Ce service est intégré dans le catalogue de service proposé par Windows Server. C'est également un élément central du fonctionnement d'un annuaire AD.

## 2.3. Service DHCP

Le DHCP va nous permettre de fournir une configuration réseau dynamique pour les ressources disponibles sur le sous-réseau `WIN`. Ce service est intégré dans le catalogue de service proposé par Windows Server.

## 2.4. PKI interne

Le PKI interne permet de disposer d'une autorité de certification afin de générer, signer et valider les certificats numériques. 

## 2.5. WSUS

WIP

## 2.6. WSUS

WIP

---

# 3.1. Schéma réseau physique

![Schéma physique](/images/schema_physique.png)

---

# 3.2. Schéma réseau logique

![Schéma logique](/images/schema_logique_win.png)

---

## 4. Priorisation

## 4.1. Infrastructure secondaire `Windows`

Comme dans la section précédente, nous allons définir différents niveaux de maturité avec les mises en place des différents services qui y sont associées.

| Niveau     | Description      | Services     
|---    |:-:    |:-:   
| 🥚    | L'infrastructure Windows est fonctionnel, il est possible de déployer des VMs préconfigurées à la main via des templates.      | Firewall, DNS, Windows Server Sysprep, AD, Windows Sysprep    
| 🐣     | Le domaine AD fournit un ensemble de services classiques : DHCP, DNS, PKI, serveur de fichier et WSUS. L'infrastructure est observée, supervisée et est accessible via un bastion : `bastion-win`. Amélioration de la sécurité.   | DHCP, DNS, PKI interne, WSUS, Prometheus, Grafana, AlertManager, Zabbix, Ping Castle     
| 🐤   |  Le dashboard Homepage prêt à l'emploi. Renforcement de la sécurité avec l'adoption du 0 trust au sein du homelab. Intégration de l'infrastructure Windows avec `VM-Factory`     | Homepage, Terraform, Ansible, notifications (Discord ?)

---

# 5. Todo lists

## .5.1. 🥚


- [ ] Niveau 1
    - [ ] Proxmox
        - [ ] Création d'une troisième interface réseau (vmbr2)
        - [ ] Création d'un folder spécifique `WIN`
        - [ ] Import des ISO 
            - [ ] Windows Server 2022
            - [ ] Windows 11
        - [ ] Création des templates
            - [ ] win2022-template
                - [ ] Activation RDP / WinRM
                - [ ] Installation des mises à jour
                - [ ] Configuration réseau statique avec une IP réservée `172.16.1.100`
                - [ ] Sysprep
                - [ ] Conversion en template
            - [ ] win11-template
                - [ ] Activation RDP / WinRM
                - [ ] Installation des mises à jour
                - [ ] Sysprep
                - [ ] Conversion en template
    - [ ] pfSense
        - [ ] Ajout de l'interface `WIN` avec l'IP `172.16.1.254`
        - [ ] Application de la règle de blocage par défaut
        - [ ] Autoriser l'accès à Internet sur le sous-réseau `WIN`
    - [ ] Active Directory
        - [ ] Déploiement du contrôleur de domaine primaire `dc01-win.homelab`
            - [ ] Création de la VM `dc01-win`
            - [ ] Installation du rôle AD DS + DNS
            - [ ] Création du domaine `win.ng-hl.com`
            - [ ] Vérification de la résolution DNS interne/externe
        - [ ] Déploiement du contrôleur de domaine secondaire `dc02-win.homelab`
            - [ ] Création de la VM `dc02-win`
            - [ ] Installation et promotion en DC secondaire
            - [ ] Vérification de la réplication AD/DNS
    - [ ] Ajout d'un poste dans l'AD
        - [ ] Création de la VM `w10-win`
        - [ ] Intégration dans l'AD
        - [ ] Tests
            - [ ] GPO
            - [ ] DNS
            - [ ] Kerberos
    - [ ] Hardening basique
        - [ ] GPO `securite-base`
            - [ ] Politique de mot de passe
            - [ ] Verrouillage de session
            - [ ] Restriction RDP
        - [ ] Désactivation du service `SMBv1`
        - [ ] Désactivation du service ̀`NTLMv1`

---

## 5.2. 🐣

- [ ] Niveau 2
    - [ ] Services Windows
        - [ ] DHCP
            - [ ] Installation du rôle
            - [ ] Configuration du scope `172.16.1.0/24` avec l'exclusion de la partie serveur
            - [ ] Activation du DNS dynamique
        - [ ] DNS
            - [ ] Configuration des forwarders vers les DNS interne `bind9` (voir section `Homelab VM-Factory`)
            - [ ] Tests
                - [ ] Résolution noms DNS internes `WIN`
                - [ ] Résolution noms DNS internes `CORE` et `VMS`
                - [ ] Résolution noms DNS externes
        - [ ] PKI interne
            - [ ] Installation du rôle ADCS
            - [ ] ...
        - [ ] Serveur de fichier
            - [ ] ...
        - [ ] WSUS
            - [ ] Installation du rôle
            - [ ] Configuration du rôle
            - [ ] Client via GPO
    - [ ] Supervision
        - [ ] Zabbix (sur le sous réseau `CORE`)
            - [ ] Mise en place de l'OS via les templates
            - [ ] Configuration de l'OS via Ansible
            - [ ] Installation de Zabbix
            - [ ] Configuration de base de Zabbix
            - [ ] Configuration du HTTPS avec le certificat *.ng-hl.com
            - [ ] Configuration du renouvellement automatique du certificat
            - [ ] Tester le renouvellement du certificat (forcer)
        - [ ] Installation de `Zabbix Agent 2` sur les serveurs Windows
        - [ ] Création de templates personnalisés
        - [ ] Configuration de l'alerting et des notifications par mail
    - [ ] Observabilité
        - [ ] Intégration d'un client Prometheus sur les serveurs Windows
        - [ ] Conception de dashboards Grafana
            - [ ] Performances Windows / AD
            - [ ] Disponiblité des services
            - [ ] Sécurité
        - [ ] Export des logs vers le puit de logs
    - [ ] Sécurité
        - [ ] Bastion
            - [ ] Création de la VM `bastion01-win`
            - [ ] Installation de RSAT
            - [ ] Accès RDP uniquement depuis l'IP d'administration (mon poste sur mon réseau local)
            - [ ] Installation des outils nécessaires (VSCode, GPO Editor, etc.)
        - [ ] Hardening
            - [ ] MFA ?
            - [ ] Application du CIS Benchmark Windows Server 2022
            - [ ] Application de Ping Castle
            - [ ] Journalisation des évènements

---

## 5.3. 🐤

- [ ] Niveau 3



---

# 6. Inventaire

| Hostname    | IP      | OS        | Hostname exposé
| :-:       | :-:       | :-:       | :-:       |
| dc01-win.homelab    | 172.16.1.253    | Windows Server 2022 |   
| dc02-win.homelab    | 172.16.1.252    | Windows Server 2022 |    