---
date: '2025-10-18T09:13:18+02:00'
draft: true
title: 'Hors Série 4 - Ollama + Open-webui'
summary: "Homelab - Hors Série 4 - Ollama + Open-webui"
tags: ["homelab"]
categories: ["homelab"]
---

> Ce document contient les livrables issus de la mise en place du service `ollama` couplé avec `open-webui`. Cette solution permet d'héberger un modèle d'IA au sein du homelab avec une interaction via une interface web.

---

# 1. Création de la VM

Nous allons utiliser le template `debian12-template` créé lors du chapitre 4. Sur Proxmox on crée un clone complet à partir de ce template. Voici les caractéristiques de la VM :

| OS      | Hostname     | Adresse IP | Interface réseau | vCPU    | RAM   | Stockage
|:-:    |:-:    |:-:    |:-:    |:-:    |:-:    |:-:
| Debian 12.10     | ollama-vms      | 192.168.200.7    | vmbr2 (vms)    | 4     | 8192   | 20Gio + 25Gio

---

# 2. Configuration de l'OS via Ansible

> Les informations concernant Ansible sont disponibles au niveau des chapitres 7 et 8.

A présent, le playbook et les rôles ayant pour objectif d'appliquer la configuration de base de l'OS sont disponibles. Il faut se connecter en tant que l'utilisateur `ansible` sur le serveur `ansible-core.homelab` puis ajouter l'hôte `ollama-vms.homelab` au niveau du fichier d'inventaire `/opt/ansible/envs/200-vms/00_inventory.yml` avec les éléments suivants

```yml
ollma-vms.homelab:
    ip: 192.168.200.7
    hostname: ollama-vms
```

Pour exécuter le playbook, il faut lancer la commande suivante

```bash
ansible-playbook -i envs/200-vms/00_inventory.yml -l 'ollama-vms.homelab,' playbooks/00_config_vm.yml
```

Voici le récapitulatif

```bash
TASKS RECAP *******************************************************************************************************
samedi 18 octobre 2025  09:27:10 +0200 (0:00:00.260)       0:00:31.522 ******** 
=============================================================================== 
dns_config : Installation du paquet systemd-resolved ------------------------------------------------------ 16.93s
base_packages : Installation des paquets de base ----------------------------------------------------------- 3.94s
base_packages : Mise à jour du cache apt ------------------------------------------------------------------- 3.82s
Gathering Facts -------------------------------------------------------------------------------------------- 1.07s
nftables : Activer et démarrer le service nftables --------------------------------------------------------- 0.62s
dns_config : Autoremove et purge --------------------------------------------------------------------------- 0.58s
hostname_config : Modification du hostname ----------------------------------------------------------------- 0.48s
motd : Déploiement du motd --------------------------------------------------------------------------------- 0.47s
dns_config : Enable du daemon systemd-resolved ------------------------------------------------------------- 0.39s
dns_config : Suppression du paquet resolvconf -------------------------------------------------------------- 0.35s
nftables : Déploiement de la configuration de nftables ----------------------------------------------------- 0.31s
dns_config : Resart du daemon systemd-resolved ------------------------------------------------------------- 0.30s
security_ssh : Restart du daemon sshd ---------------------------------------------------------------------- 0.26s
nftables : Valider la configuration nftables --------------------------------------------------------------- 0.24s
dns_config : Suppression du fichier /etc/resolv.conf ------------------------------------------------------- 0.20s
ipv6_disable : Désactivation de la prise en charge de l'IPv6 globalement ----------------------------------- 0.20s
hostname_config : Modification du fichier /etc/hosts ------------------------------------------------------- 0.20s
dns_config : Configuration du DNS dans /etc/resolved.conf -------------------------------------------------- 0.19s
dns_config : Création du nouveau lien symbolique vers /etc/resolv.conf ------------------------------------- 0.15s
ipv6_disable : Désactivation de la prise en charge de l'IPv6 par défaut ------------------------------------ 0.14s
```


> Mise à jour de l'inventaire `00_inventory.yml` pour ajouter `ollama-vms` dans le groupe `prometheus_clients`.

Exécution du playbook pour la mise en place de `Node Exporter`. Il faut également mettre à jour la configuration de `prometheus-core` pour ajouter cet host.

```bash
ansible-playbook -i envs/200-vms/00_inventory.yml -l 'ollama-vms.homelab,' playbooks/01_prometheus_node_exporter.yml
```

Voici le récapitulatif 

```bash
TASKS RECAP *******************************************************************************************************
samedi 18 octobre 2025  09:40:02 +0200 (0:00:00.311)       0:00:04.596 ******** 
=============================================================================== 
Gathering Facts -------------------------------------------------------------------------------------------- 1.00s
node_exporter : Téléchargement de Node Exporter ------------------------------------------------------------ 0.97s
node_exporter : Décompression de Node Exporter ------------------------------------------------------------- 0.81s
node_exporter : Activation et démarrage le service --------------------------------------------------------- 0.61s
node_exporter : Copie du binaire dans /usr/local/bin ------------------------------------------------------- 0.34s
node_exporter : Restart node_exporter ---------------------------------------------------------------------- 0.31s
node_exporter : Création du service systemd ---------------------------------------------------------------- 0.30s
node_exporter : Création de l'utilisateur node_exporter ---------------------------------------------------- 0.25s
```

---

# 3. Déploiement de Ollama et de Open-webui

> Il est nécessaire d'installer Podman en amont puis on ajoute l'utilisateur `ngobert` au groupe `podman`

```bash
sudo apt update && sudo apt install -y podman podman-compose
```

Vérification des versions installées

```bash
podman --version
podman-compose --version
```

On crée la structure de répertoire pour héberger les fichiers nécessaires au déploiement du container

```bash
sudo mkdir -p /opt/ollama-stack
sudo chown ngobert:ngobert /opt/ollama-stack/
```

Création du fichier `/opt/ollama-stack/docker-compose.yml`

```yaml
version: "3.9"

services:
  ollama:
    image: docker.io/ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    volumes:
      - /srv/docker/ollama:/root/.ollama
    environment:
      - OLLAMA_DEFAULT_MODEL=phi3
      - OLLAMA_HOST=0.0.0.0
      - OLLAMA_NUM_THREADS=4
      - OLLAMA_KEEP_ALIVE=5m
    networks:
      - ollama-network
    security_opt:
      - no-new-privileges:true
    mem_limit: 6g
    cpus: 3.5

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    ports:
      - "3000:8080"
    volumes:
      - /srv/docker/open-webui:/app/backend/data
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - WEBUI_AUTH=true
    depends_on:
      - ollama
    networks:
      - ollama-network
    security_opt:
      - no-new-privileges:true
    mem_limit: 1g
    cpus: 0.5

volumes:
  ollama_data:
  open_webui_data:

networks:
  ollama-network:
    driver: bridge

```

Se positionner dans le bon répertoire `/opt/ollama-stack/` puis les deux containers via `podman`

```bash
podman-compose up -d
```

Téléchargement du modèle léger `llama3.2:3b`

```bash
podman exec -it ollama ollama pull llama3.2:3b
```


