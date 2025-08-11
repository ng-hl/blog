---
date: '2025-07-31T21:56:48+02:00'
draft: false
title: 'Chapitre 13 - Prometheus'
summary: "Homelab - Chapitre 13 - Prometheus/Grafana"
tags: ["homelab","prometheus","grafana"]
categories: ["homelab"]
---

> Ce document contient les livrables issus de la mise en place du service `Prometheus`, `Grafana` et `AlertsManager`. L'objectif est de pouvoir disposer de services, concernant la collecte de métriques, l'exposition de dashboards et la génération d'alertes avec notifications par mail. Côté architecture, un reverse proyx `nginx` sera en frontal pour porter le certficat wildcard `*.ng-hl.com` et redirigera vers le serveur adéquat. Pour le niveau de maturité niveau 2, le chiffrement TLS sera appliqué uniquement au niveau de l'exposition vers l'extérieur à savoir le `nginx`. Concernant le niveau 3, il faudra mettre en place des communications chiffrés via TLS au travers de certificats fournit par une PKI interne pour adopter le principe d'une architecture 0 trust.

---

> Cette section couvre la partie reverse proxy avec `nginx`. Cette machine porte le nom `reverseproxysup-core.homelab` avec un CNAME `rps-core.homelab`.

# 1. Création de la VM

Nous allons utiliser le template `debian12-template` créé lors du chapitre 4. Sur Proxmox on crée un clone complet à partir de ce template. Voici les caractéristiques de la VM :

| OS      | Hostname     | Adresse IP | Interface réseau | vCPU    | RAM   | Stockage
|:-:    |:-:    |:-:    |:-:    |:-:    |:-:    |:-:
| Debian 12.10     | reverseproxysup-core rps-core (CNAME)    | 192.168.100.242    | vmbr1 (core)    | 1     | 2048   | 20Gio

Il faut également penser à activer la sauvegarde automatique de la VM sur Proxmox en l'ajoutant au niveau de la politique de sauvegarde précédemment créée.

---

# 2. Configuration de l'OS via Ansible

> Les informations concernant Ansible sont disponibles au niveau des chapitres 7 et 8.

A présent, le playbook et les rôles ayant pour objectif d'appliquer la configuration de base de l'OS sont disponibles. Il faut se connecter en tant que l'utilisateur `ansible` sur le serveur `ansible-core.homelab` puis ajouter l'hôte `rps-core.homelab` au niveau du fichier d'inventaire `/opt/ansible/envs/100-core/00_inventory.yml` avec les éléments suivants

```yml
rps-core.homelab:
    ip: 192.168.100.242
    hostname: reverseproxysup-core
```

Pour exécuter le playbook, il faut lancer la commande suivante

```bash
ansible-playbook -i envs/100-core/00_inventory.yml -l 'rps-core.homelab,' playbooks/00_config_vm.yml
```

Voici le récapitulatif

```bash
TASKS RECAP **********************************************************************************************************************
vendredi 08 août 2025  23:14:22 +0200 (0:00:00.260)       0:00:14.348 ********* 
=============================================================================== 
base_packages : Installation des paquets de base -------------------------------------------------------------------------- 4.30s
dns_config : Installation du paquet systemd-resolved ---------------------------------------------------------------------- 2.05s
base_packages : Mise à jour du cache apt ---------------------------------------------------------------------------------- 1.88s
Gathering Facts ----------------------------------------------------------------------------------------------------------- 1.06s
dns_config : Autoremove et purge ------------------------------------------------------------------------------------------ 0.56s
motd : Déploiement du motd ------------------------------------------------------------------------------------------------ 0.46s
hostname_config : Modification du hostname -------------------------------------------------------------------------------- 0.44s
dns_config : Enable du daemon systemd-resolved ---------------------------------------------------------------------------- 0.39s
dns_config : Suppression du paquet resolvconf ----------------------------------------------------------------------------- 0.33s
nftables : Déploiement de la configuration de nftables -------------------------------------------------------------------- 0.31s
dns_config : Resart du daemon systemd-resolved ---------------------------------------------------------------------------- 0.30s
nftables : Valider la configuration nftables ------------------------------------------------------------------------------ 0.27s
security_ssh : Restart du daemon sshd ------------------------------------------------------------------------------------- 0.26s
dns_config : Suppression du fichier /etc/resolv.conf ---------------------------------------------------------------------- 0.20s
ipv6_disable : Désactivation de la prise en charge de l'IPv6 globalement -------------------------------------------------- 0.19s
hostname_config : Modification du fichier /etc/hosts ---------------------------------------------------------------------- 0.19s
dns_config : Configuration du DNS dans /etc/resolved.conf ----------------------------------------------------------------- 0.19s
dns_config : Création du nouveau lien symbolique vers /etc/resolv.conf ---------------------------------------------------- 0.14s
ipv6_disable : Désactivation de la prise en charge de l'IPv6 par défaut --------------------------------------------------- 0.14s
security_ssh : Activation de l'authentification par clé ------------------------------------------------------------------- 0.14s
```

---

# 3. Déploiement et renouvellement automatique du certificat

> Afin que le mécanisme de déploiement et de renouvellement du certificat wildcard *.ng-hl.com puisse être fonctionnel, il faut modifier temporairement la configuration du service `ssh` pour autoriser la connexion root avec un mot de passe pour autoriser la connexion en root via la clé publique `id_acme_pub` en provenance de `acme-core`.

Se rendre sur le serveur `acme-core` en tant que `root` puis compléter le fichier `/root/acme-deploy/targets.yml`

```bash
  - host: rps-core.homelab
    user: root
    type: binary
    cert_path: /etc/ssl/certs/wildcard.ng-hl.com
    privkey_name: privkey.key
    cert_name: fullchain.cer
    reload_cmd: systemctl restart nginx
```

Exécuter le script `/root/acme-deploy/acme-deploy.sh` pour déployer le certificat et la privkey associée.

```bash
Syncing cert to rps-core.homelab...
Remote cert checksum: missing
Remote key checksum:  missing
Changes detected, copying certs...
fullchain.cer                                                                                   100% 2848     7.9MB/s   00:00    
privkey.key                                                                                     100%  227   829.2KB/s   00:00    
Reloading service on rps-core.homelab...
Deployment completed for rps-core.homelab.
```

---

# 4. Installation et configuration de nginx via Ansible

> Afin de commencer à intégrer le principe d'IaC déclarative et reproductible, je choisi d'installer et de configurer nginx pour l'instance `rps-core` via un playbook Ansible

Création du nouveau rôle `reverse_proxy_sup`

```bash
ansible-galaxy init roles/reverse_proxy_sup
```

Voici le contenu du fichier `tasks/main.yml`

```yaml
# SPDX-License-Identifier: MIT-0
---
# tasks file for roles/reverse_proxy_sup

- name: Installation de Nginx
  ansible.builtin.apt:
    name: nginx
    state: present
    update_cache: yes

- name: Création du répertoire SSL
  ansible.builtin.file:
    path: "{{ reverse_proxy_ssl_dir }}"
    state: directory
    owner: root
    group: root
    mode: '0755'

- name: Création du répertoire pour les configurations Nginx
  ansible.builtin.file:
    path: /etc/nginx/sites-available
    state: directory
    owner: root
    group: root
    mode: '0755'

- name: Création du répertoire sites-enabled
  ansible.builtin.file:
    path: /etc/nginx/sites-enabled
    state: directory
    owner: root
    group: root
    mode: '0755'

- name: Déploiement de la configuration Nginx pour le vHost supervision
  ansible.builtin.template:
    src: supervision.conf.j2
    dest: /etc/nginx/sites-available/supervision.conf
    owner: root
    group: root
    mode: '0644'

- name: Activation de la configuration Nginx pour le vHost supervision
  ansible.builtin.file:
    src: /etc/nginx/sites-available/supervision.conf
    dest: /etc/nginx/sites-enabled/supervision.conf
    state: link
    force: yes
  notify: Redémarrer Nginx

```

Voici le contenu du fichier `defaults/main.yml`

```yaml
# SPDX-License-Identifier: MIT-0
---
# defaults file for roles/reverse_proxy_sup

reverse_proxy_sup_domain: "supervision.ng-hl.com"
reverse_proxy_sup_ssl_dir: "/etc/ssl/certs/wildcard.ng-hl.com"
reverse_proxy_sup_cert_file: "fullchain.cer"
reverse_proxy_sup_key_file: "privkey.key"
reverse_proxy_sup_prometheus: "prometheus.ng-hl.com"
reverse_proxy_sup_grafana: "grafana.ng-hl.com"
reverse_proxy_sup_alertmanager: "alertmanager.ng-hl.com"

```

Voici le contenu du fichier `handlers/main.yml`

```yaml
# SPDX-License-Identifier: MIT-0
---
# handlers file for roles/reverse_proxy_sup

- name: Redémarrer Nginx
  ansible.builtin.systemd:
    name: nginx
    state: restarted
    enabled: yes

```

Voici le contenu du fichier `templates/supervision.conf.j2`

```j2
server {
    listen 80;
    server_name {{ reverse_proxy_sup_domain }};
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name {{ reverse_proxy_sup_domain }};

    ssl_certificate     {{ reverse_proxy_sup_ssl_dir }}/{{ reverse_proxy_sup_cert_file }};
    ssl_certificate_key {{ reverse_proxy_sup_ssl_dir }}/{{ reverse_proxy_sup_key_file }};
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    access_log /var/log/nginx/supervision_access.log;
    error_log  /var/log/nginx/supervision_error.log;

    location / {
        root /var/www/index.html;
    }
}

```

Création du playbook `10_reverse_proxy_sup.yml`

```yml
---
- name: Configuration du reverse proxy pour la supervision
  hosts: all
  become: true
  roles:
    - nftables
    - reverse_proxy_sup

```

Afin de vérifier que l'on applique bien une syntaxe cohérente et les bonnes pratiques Ansible, on exécute le linter `ansible-lint` sur notre nouveau playbook.

```bash
ansible-lint playbooks/10_reverse_proxy_sup.yml
Passed: 0 failure(s), 0 warning(s) on 11 files.
```

Application du playbook sur l'instance `rps-core`

```bash
ansible-playbook -i envs/100-core/00_inventory.yml -l 'rps-core.homelab,' playbooks/10_reverse_proxy_sup.yml
```

Voici le récapitulatif du passage du playbook

```bash
TASKS RECAP ***************************************************************************************************************
samedi 09 août 2025  21:10:15 +0200 (0:00:00.142)       0:00:03.828 *********** 
=============================================================================== 
Gathering Facts ---------------------------------------------------------------------------------------------------- 1.00s
reverse_proxy_sup : Installation de Nginx -------------------------------------------------------------------------- 0.91s
nftables : Déploiement de la configuration de nftables ------------------------------------------------------------- 0.44s
nftables : Activer et démarrer le service nftables ----------------------------------------------------------------- 0.38s
reverse_proxy_sup : Déploiement de la configuration Nginx pour Prometheus, Grafana et AlertManager ----------------- 0.29s
nftables : Valider la configuration nftables ----------------------------------------------------------------------- 0.20s
reverse_proxy_sup : Création du répertoire SSL --------------------------------------------------------------------- 0.19s
reverse_proxy_sup : Activation de la configuration Nginx ----------------------------------------------------------- 0.14s
reverse_proxy_sup : Création du répertoire sites-enabled ----------------------------------------------------------- 0.14s
reverse_proxy_sup : Création du répertoire pour les configurations Nginx ------------------------------------------- 0.13s
```

Test de la réponse concernant la requête HTTPS sur le port 443

```bash
curl https://supervision-core.ng-hl.com
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

---

> Cette section couvre la partie `Prometheus`

# 1. Création de la VM

Nous allons utiliser le template `debian12-template` créé lors du chapitre 4. Sur Proxmox on crée un clone complet à partir de ce template. Voici les caractéristiques de la VM :

| OS      | Hostname     | Adresse IP | Interface réseau | vCPU    | RAM   | Stockage
|:-:    |:-:    |:-:    |:-:    |:-:    |:-:    |:-:
| Debian 12.10     | prometheus-core      | 192.168.100.245    | vmbr1 (core)    | 2     | 4096   | 20Gio + 10Gio

Il faut également penser à activer la sauvegarde automatique de la VM sur Proxmox en l'ajoutant au niveau de la politique de sauvegarde précédemment créée.

> Ajout du second disque de 10Gio sur le lv `/dev/debian12-vg/debian12-var-lv`

---

# 2. Configuration de l'OS via Ansible

> Les informations concernant Ansible sont disponibles au niveau des chapitres 7 et 8.

A présent, le playbook et les rôles ayant pour objectif d'appliquer la configuration de base de l'OS sont disponibles. Il faut se connecter en tant que l'utilisateur `ansible` sur le serveur `ansible-core.homelab` puis ajouter l'hôte `prometheus-core.homelab` au niveau du fichier d'inventaire `/opt/ansible/envs/100-core/00_inventory.yml` avec les éléments suivants

```yml
prometheus-core.homelab:
    ip: 192.168.100.245
    hostname: prometheus-core
```

Pour exécuter le playbook, il faut lancer la commande suivante

```bash
ansible-playbook -i envs/100-core/00_inventory.yml -l 'prometheus-core.homelab,' playbooks/00_config_vm.yml
```

Voici le récapitulatif
 
```bash
TASKS RECAP **********************************************************************************************************************
vendredi 01 août 2025  21:32:25 +0200 (0:00:00.199)       0:00:06.948 ********* 
=============================================================================== 
Gathering Facts ----------------------------------------------------------------------------------------------------------- 1.00s
dns_config : Autoremove et purge ------------------------------------------------------------------------------------------ 0.58s
base_packages : Installation des paquets de base -------------------------------------------------------------------------- 0.46s
dns_config : Installation du paquet systemd-resolved ---------------------------------------------------------------------- 0.45s
motd : Déploiement du motd ------------------------------------------------------------------------------------------------ 0.44s
hostname_config : Modification du hostname -------------------------------------------------------------------------------- 0.42s
base_packages : Mise à jour du cache apt ---------------------------------------------------------------------------------- 0.39s
dns_config : Enable du daemon systemd-resolved ---------------------------------------------------------------------------- 0.38s
dns_config : Suppression du paquet resolvconf ----------------------------------------------------------------------------- 0.34s
dns_config : Resart du daemon systemd-resolved ---------------------------------------------------------------------------- 0.29s
nftables : Déploiement de la configuration de nftables -------------------------------------------------------------------- 0.29s
nftables : Valider la configuration nftables ------------------------------------------------------------------------------ 0.20s
ipv6_disable : Désactivation de la prise en charge de l'IPv6 globalement -------------------------------------------------- 0.20s
dns_config : Suppression du fichier /etc/resolv.conf ---------------------------------------------------------------------- 0.19s
hostname_config : Modification du fichier /etc/hosts ---------------------------------------------------------------------- 0.19s
dns_config : Configuration du DNS dans /etc/resolved.conf ----------------------------------------------------------------- 0.19s
ipv6_disable : Désactivation de la prise en charge de l'IPv6 par défaut --------------------------------------------------- 0.14s
dns_config : Création du nouveau lien symbolique vers /etc/resolv.conf ---------------------------------------------------- 0.14s
security_ssh : Désactivaction de l'authentification par mot de passe ------------------------------------------------------ 0.14s
security_ssh : Activation de l'authentification par clé ------------------------------------------------------------------- 0.14s
```

---

# 3. Installation et configuration de Prometheus

> Avant de procéder aux opérations ci-dessous, il est nécessaire d'installation `podman` sur le serveur. Nous n'utilisons pas `docker` par soucis de compatibilité avec `nftables`.

```bash
sudo apt install podman -y
```

Configuration de `podman` pour que l'espace disque utilisé soit situé au niveau de la partition `/var` et non pas `/home`

```bash
sudo mkdir -p /var/lib/containers
sudo chown -R ngobert:ngobert /var/lib/containers
mkdir -p ~/.config/containers
```

Création du fichier `~/.config/containers/storage.conf`

```bash
[storage]
driver = "overlay"
runroot = "/var/run/containers/storage"
graphroot = "/var/lib/containers/storage"
```

Restart du daemon `podman`

```bash
sudo systemctl restart podman
```

Une image prometheus est disponible sur DockerHub. Nous allons utiliser cette image pour héberger notre instance prometheus

Création du volume Docker pour la persistence de la données

```bash
sudo podman volume create prometheus-data
```

Création du répertoire /opt/prometheus

```bash
sudo mkdir -p /opt/prometheus
```

Modification du propriétaire et du groupe sur le répertoire précemment créé

```bash
sudo chown ngobert:ngobert /opt/prometheus
```

Création du fichier de configuration `/opt/prometheus/prometheus.yml`

```bash
global:
  scrape_interval:     15s
  evaluation_interval: 15s

rule_files:
  # - "first.rules"
  # - "second.rules"

alerting:
  alertmanagers:
  - static_configs:
    - targets:
       - localhost:9093

scrape_configs:
  - job_name: prometheus-core
    static_configs:
      - targets: ['prometheus-core.homelab:9090']
  - job_name: admin-core
    static_configs:
      - targets: ['admin-core.homelab:9090']
```


Exécution du container

```bash
sudo podman run -d \
    -p 9090:9090 \
    --name prometheus \
    -v /opt/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
    -v prometheus-data:/prometheus \
    docker.io/prom/prometheus \
    --config.file=/etc/prometheus/prometheus.yml
```

Modification du fichier de configuration nftables `/etc/nftables.conf` pour autoriser les requêtes sur le port de prometheus `tcp:9090` en provenance de `rps-core`. De plus, il faut autoriser le traffic forwardé via l'interface `podman0` qui est le bridge par défaut pour les containers podman

```bash
#!/usr/sbin/nft -f

table inet filter {
  chain input {
    type filter hook input priority 0;
    policy drop;
    iif lo accept
    ct state established,related accept

    ip protocol icmp accept

    
    ip saddr {{ 192.168.100.252, 192.168.100.250, 192.168.100.248 }} iifname "ens19" tcp dport 22 accept
    ip saddr 192.168.100.242 tcp dport 9090 accept
      }

  chain forward {
    type filter hook forward priority 0;
    policy drop;
    iifname "podman0" accept
    oifname "podman0" accept
  }

  chain output {
    type filter hook output priority 0;
    policy accept;
  }
}
```

Vérification de la syntaxe du fichier `/etc/nftables.conf` et restart du daemon `nftables`

```bash
sudo nft -c -f /etc/nftables.conf
sudo systemctl restart nftables
```

---

# 4. Ajout du vHost pormetheus.ng-hl.com sur le rps

> Ajout du vHost `prometheus.ng-hl.com` au niveau du reverse proxy `rps-core`. On passe encore une fois par Ansible pour réaliser les opérations.

Ajout du fichier `templates/prometheus.conf.j2`

```j2
server {
    listen 80;
    server_name {{ reverse_proxy_sup_prometheus }};
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name {{ reverse_proxy_sup_prometheus }};

    ssl_certificate     {{ reverse_proxy_sup_ssl_dir }}/{{ reverse_proxy_sup_cert_file }};
    ssl_certificate_key {{ reverse_proxy_sup_ssl_dir }}/{{ reverse_proxy_sup_key_file }};
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    access_log /var/log/nginx/supervision_access.log;
    error_log  /var/log/nginx/supervision_error.log;

    location / {
        proxy_pass http://prometheus-core.homelab:9090/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```

Rajout des actions au niveau du fichier `tasks/main.yml`

```yml
- name: Déploiement de la configuration Nginx pour le vHost Prometheus
  ansible.builtin.template:
    src: prometheus.conf.j2
    dest: /etc/nginx/sites-available/prometheus.conf
    owner: root
    group: root
    mode: '0644'

- name: Activation de la configuration Nginx pour le vHost Prometheus
  ansible.builtin.file:
    src: /etc/nginx/sites-available/prometheus.conf
    dest: /etc/nginx/sites-enabled/prometheus.conf
    state: link
    force: yes
  notify: Redémarrer Nginx
```

---

# 5. Ajout des métriques Prometheus pour tous les serveurs (actions manuelles)

Nous allons utiliser `NodeExporter` pour mettre à disposition des métriques systèmes classiques tels que le CPU, la mémoire, l'utilisation de l'espace disque, etc. depuis nos serveurs clients afin que le serveur `Prometheus` puissent venir scraper les données et les intégrer dans sa base de données.

> Cette section décrit les opérations manuelles pour l'installation du daemon `node_exporter` et la configuration de `Prometheus` pour intégrer le scraping. Nous intégrons ces actions au sein d'un rôle Ansible dans la section suivante.

Se connecter sur le serveur `admin-core`. C'est sur ce serveur que l'on va réaliser les opérations.

Téléchargement de la version 1.6.1 de node_exporter

```bash
cd /tmp/ && wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
```

Décompression de la tarball

```bash
tar xvf node_exporter-1.6.1.linux-amd64.tar.gz
```

Copie du binaire dans `/usr/local/bin`

```bash
sudo cp node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
```

Création du daemon `node_exporter`

```bash
sudo tee /etc/systemd/system/node_exporter.service <<EOF
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=default.target
EOF
```

Rechargement des daemons, activation au démarrage du système et immédiat

```bash
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
```

Vérification de l'ouverture du port avec les métriques

```bash
curl http://localhost:9100/metrics
```

> Il est important de vérifier que le flux est accepté sur le port 9100 pour les requêtes en provenance du serveur prometheus-core au niveau de nftables

Depuis le serveur `prometheus-core`

```bash
nc -vw 1 admin-core.homelab 9100
admin-core.homelab [192.168.100.252] 9100 (?) open
```

---

# 6. Ajout des métriques Prometheus pour tous les serveurs (Ansible)

> Nous allons créer un playbook Ansible `01_prometheus_node_exporter` pour installer et configurer `Node Exporter` grâce au rôle `node_exporter`. Ainsi, nous allons pouvoir déployer node exporter et la configuration nécessaires sur les serveurs existants. Ce playbook sera également utilisé pour les futurs serveurs également.

Création du playbook `01_prometheus_node_exporter`

```yml
---
- name: Déploiement de Node Exporter
  hosts: prometheus_clients
  become: true
  roles:
    - node_exporter
```

Création du rôle `node_exporter`

```bash
ansible-galaxy init roles/node_exporter
```

Contenu du fichier `/defaults/main.yml`

```yml
# SPDX-License-Identifier: MIT-0
---
# defaults file for roles/node_exporter

node_exporter_version: "1.6.1"
node_exporter_user: "node_exporter"
node_exporter_bin_path: "/usr/local/bin/node_exporter"
node_exporter_port: 9100
node_exporter_listen_address: "{{ ip }}"
```

> Nous utilisons la host_var `ip` définie au niveau du fichier d'inventaire qui représente l'IP utilisée en interne, au sein du homelab. C'est important d'écouter uniquement sur cette interface car certains serveurs peuvent avoir des interfaces en bridge directement accessible depuis mon réseau local. Nous voulons éviter que le scraping des données soit accessible à l'extérieur du homelab.

Contenu du fichier `/tasks/main.yml`

```yml
# SPDX-License-Identifier: MIT-0
---
# tasks file for roles/node_exporter

- name: Création de l'utilisateur node_exporter
  ansible.builtin.user:
    name: "{{ node_exporter_user }}"
    shell: /usr/sbin/nologin
    system: true
    create_home: false

- name: Téléchargement de Node Exporter
  ansible.builtin.get_url:
    url: >-
      https://github.com/prometheus/node_exporter/releases/download/v{{ node_exporter_version }}/node_exporter-{{ node_exporter_version }}.linux-amd64.tar.gzlinux-amd64.tar.gz"
    dest: "/tmp/node_exporter-{{ node_exporter_version }}.tar.gz"
    mode: '0644'

- name: Décompression de Node Exporter
  ansible.builtin.unarchive:
    src: "/tmp/node_exporter-{{ node_exporter_version }}.tar.gz"
    dest: /tmp
    remote_src: true

- name: Copie du binaire dans /usr/local/bin
  ansible.builtin.copy:
    src: "/tmp/node_exporter-{{ node_exporter_version }}.linux-amd64/node_exporter"
    dest: "{{ node_exporter_bin_path }}"
    mode: '0755'
    owner: root
    group: root
    remote_src: true

- name: Création du service systemd
  ansible.builtin.copy:
    dest: /etc/systemd/system/node_exporter.service
    content: |
      [Unit]
      Description=Node Exporter
      After=network.target

      [Service]
      User={{ node_exporter_user }}
      ExecStart={{ node_exporter_bin_path }} --web.listen-address={{ node_exporter_listen_address }}:{{ node_exporter_port }}

      [Install]
      WantedBy=default.target
    owner: root
    group: root
    mode: '0644'
  notify: Restart node_exporter

- name: Activation et démarrage le service
  ansible.builtin.systemd:
    name: node_exporter
    enabled: true
    state: started

```

Contenu du fichier `/handlers/main.yml`

```yml
# SPDX-License-Identifier: MIT-0
---
# handlers file for roles/node_exporter

- name: Restart node_exporter
  ansible.builtin.systemd:
    name: node_exporter
    state: restarted

```

Rajout du group `prometheus` au niveau du fichier d'inventaire `/envs/100-core/00_inventory.yml`

```yml
prometheus_clients:
  hosts:
    dns-core.homelab:
    admin-core.homelab:
    ansible-core.homelab:
    vaultwarden-core.homelab:
    gitlab-core.homelab:
    prometheus-core.homelab:
    grafana-core.homelab:
    rps-core.homelab:
```

Application du linter `ansible-lint` sur le playbook `01_prometheus_node_exporter.yml`

```bash
ansible-lint playbooks/01_prometheus_node_exporter.yml 
Passed: 0 failure(s), 0 warning(s) on 6 files
```

Exécution sur un serveur spécifique. Penser à faire un snapshot avant l'exécution du playbook

```bash
ansible-playbook playbooks/01_prometheus_node_exporter.yml -l "dns-core.homelab"
```

Mise à jour du fichier de configuration `/opt/prometheus/prometheus.yml` sur le serveur `prometheus-core`

```yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  # - "first.rules"
  # - "second.rules"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - localhost:9093

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['prometheus-core.homelab:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets:
          - admin-core.homelab:9100
          - dns-core.homelab:9100
          - ansible-core.homelab:9100
          - acme-core.homelab:9100
          - vaultwarden-core.homelab:9100
          - gitlab-core.homelab:9100
          - grafana-core.homelab:9100
          - rps-core.homelab:9100

```

---

> Cette section couvre la partie `Grafana`

# 1. Création de la VM

Nous allons utiliser le template `debian12-template` créé lors du chapitre 4. Sur Proxmox on crée un clone complet à partir de ce template. Voici les caractéristiques de la VM :

| OS      | Hostname     | Adresse IP | Interface réseau | vCPU    | RAM   | Stockage
|:-:    |:-:    |:-:    |:-:    |:-:    |:-:    |:-:
| Debian 12.10     | grafana-core      | 192.168.100.244    | vmbr1 (core)    | 2     | 4096   | 20Gio + 10Gio

Il faut également penser à activer la sauvegarde automatique de la VM sur Proxmox en l'ajoutant au niveau de la politique de sauvegarde précédemment créée.

> Ajout du second disque de 10Gio sur le lv `/dev/debian12-vg/debian12-var-lv`

---

# 2. Configuration de l'OS via Ansible

> Les informations concernant Ansible sont disponibles au niveau des chapitres 7 et 8.

A présent, le playbook et les rôles ayant pour objectif d'appliquer la configuration de base de l'OS sont disponibles. Il faut se connecter en tant que l'utilisateur `ansible` sur le serveur `ansible-core.homelab` puis ajouter l'hôte `grafana-core.homelab` au niveau du fichier d'inventaire `/opt/ansible/envs/100-core/00_inventory.yml` avec les éléments suivants

```yml
grafana-core.homelab:
    ip: 192.168.100.244
    hostname: grafana-core
```

Pour exécuter le playbook, il faut lancer la commande suivante

```bash
ansible-playbook -i envs/100-core/00_inventory.yml -l 'grafana-core.homelab,' playbooks/00_config_vm.yml
```

Voici le récapitulatif

```bash
TASKS RECAP **********************************************************************************************************************
samedi 09 août 2025  23:19:15 +0200 (0:00:00.269)       0:00:14.525 *********** 
=============================================================================== 
base_packages : Installation des paquets de base -------------------------------------------------------------------------- 3.81s
dns_config : Installation du paquet systemd-resolved ---------------------------------------------------------------------- 1.99s
base_packages : Mise à jour du cache apt ---------------------------------------------------------------------------------- 1.89s
Gathering Facts ----------------------------------------------------------------------------------------------------------- 1.07s
nftables : Activer et démarrer le service nftables ------------------------------------------------------------------------ 0.63s
dns_config : Autoremove et purge ------------------------------------------------------------------------------------------ 0.59s
motd : Déploiement du motd ------------------------------------------------------------------------------------------------ 0.46s
hostname_config : Modification du hostname -------------------------------------------------------------------------------- 0.44s
dns_config : Enable du daemon systemd-resolved ---------------------------------------------------------------------------- 0.39s
dns_config : Suppression du paquet resolvconf ----------------------------------------------------------------------------- 0.33s
nftables : Déploiement de la configuration de nftables -------------------------------------------------------------------- 0.32s
dns_config : Resart du daemon systemd-resolved ---------------------------------------------------------------------------- 0.31s
security_ssh : Restart du daemon sshd ------------------------------------------------------------------------------------- 0.27s
nftables : Valider la configuration nftables ------------------------------------------------------------------------------ 0.24s
ipv6_disable : Désactivation de la prise en charge de l'IPv6 globalement -------------------------------------------------- 0.20s
dns_config : Configuration du DNS dans /etc/resolved.conf ----------------------------------------------------------------- 0.20s
dns_config : Suppression du fichier /etc/resolv.conf ---------------------------------------------------------------------- 0.20s
hostname_config : Modification du fichier /etc/hosts ---------------------------------------------------------------------- 0.19s
dns_config : Création du nouveau lien symbolique vers /etc/resolv.conf ---------------------------------------------------- 0.14s
ipv6_disable : Désactivation de la prise en charge de l'IPv6 par défaut --------------------------------------------------- 0.14s
```

---

# 3. Installation et configuration de Grafana

> Tout comme Prometheus, la solution Grafana va être déployée en container. Avant de procéder aux opérations ci-dessous, il est nécessaire d'installation `podman` sur le serveur. Nous n'utilisons pas `docker` par soucis de compatibilité avec `nftables`.

```bash
sudo apt install podman -y
```

Configuration de `podman` pour que l'espace disque utilisé soit situé au niveau de la partition `/var` et non pas `/home`

```bash
sudo mkdir -p /var/lib/containers
sudo chown -R ngobert:ngobert /var/lib/containers
mkdir -p ~/.config/containers
```

Création du fichier `~/.config/containers/storage.conf`

```bash
[storage]
driver = "overlay"
runroot = "/var/run/containers/storage"
graphroot = "/var/lib/containers/storage"
```

Restart du daemon `podman`

```bash
sudo systemctl restart podman
```

Une image prometheus est disponible sur DockerHub. Nous allons utiliser cette image pour héberger notre instance prometheus

Création du volume Docker pour la persistence de la données

```bash
podman volume create grafana-data
```

Création du répertoire `/opt/grafana`

```bash
sudo mkdir -p /opt/grafana
```

Modification du propriétaire et du groupe sur le répertoire précemment créé

```bash
sudo chown ngobert:ngobert /opt/grafana
```

Création des répertoires nécessaires

```bash
mkdir -p /opt/grafana/provisioning/datasources
mkdir -p /opt/grafana/provisioning/dashboards
mkdir -p /opt/grafana/dashboards/node_exporter
```

Création du fichier de configuration `/opt/grafana/provisioning/datasources/prometheus_datasources.yml`

```yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus-core.homelab:9090
    isDefault: true

```

Création du fichier de configuration `/opt/grafana/provisioning/dashboards/dashboards.yml`

```yml 
apiVersion: 1

providers:
  - name: 'Node Exporter Dashboards'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    options:
      path: /var/lib/grafana/dashboards/node-exporter

```

Création du fichier `/opt/grafana/dashboards/node_exporter/node_exporter_full.json` avec le contenu du fichier de ce [lien](https://grafana.com/api/dashboards/1860/revisions/latest/download)

Exécution du container

```bash
sudo podman run -d \
  --name grafana \
  -p 3000:3000 \
  -v grafana-data:/var/lib/grafana \
  -v /opt/grafana/provisioning:/etc/grafana/provisioning \
  -v /opt/grafana/dashboards:/var/lib/grafana/dashboards \
  -e "GF_SERVER_ROOT_URL=https://grafana.ng-hl.com/" \
  -e "GF_SERVER_SERVE_FROM_SUB_PATH=true" \
  docker.io/grafana/grafana:latest
```

Modification du fichier de configuration nftables `/etc/nftables.conf` pour autoriser les requêtes sur le port de Grafana `tcp:3000` en provenance de `rps-core`. De plus, il faut autoriser le traffic forwardé via l'interface `podman0` qui est le bridge par défaut pour les containers podman

```bash
#!/usr/sbin/nft -f

table inet filter {
  chain input {
    type filter hook input priority 0;
    policy drop;
    iif lo accept
    ct state established,related accept

    ip protocol icmp accept

    
    ip saddr {{ 192.168.100.252, 192.168.100.250, 192.168.100.248 }} iifname "ens19" tcp dport 22 accept
    ip saddr 192.168.100.242 tcp dport 3000 accept
  }

  chain forward {
    type filter hook forward priority 0;
    policy drop;
    iifname "podman0" accept
    oifname "podman0" accept
  }

  chain output {
    type filter hook output priority 0;
    policy accept;
  }
}
```

Vérification de la syntaxe du fichier `/etc/nftables.conf` et restart du daemon `nftables`

```bash
sudo nft -c -f /etc/nftables.conf
sudo systemctl restart nftables
```

Pour se connecter sur l'interface de Grafana on tape cette url `https://grafana.ng-hl.com`. On saisie les credentials par défaut `admin/admin` pour la première connexion. On peut observer que le datasource `Prometheus` est bien fonctionnel et que le dashboard `Node Exporter Full` est accessible.
