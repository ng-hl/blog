---
date: '2025-08-15T23:12:11+02:00'
draft: false
title: 'Chapitre 15 - Matrice et supervision des services'
summary: "Homelab - Chapitre 15 - Matrice et supervision des services"
tags: ["homelab","daemons"]
categories: ["homelab"]
---

# 1. Matrice des services et des ports associés

> Cette matrice est une référence pour identifier les services critiques au sein du homelab. De plus, cette matrice est utile pour mettre en place et maintenir les métriques d'observabilité et les alertes de supervision.

| Hostname    | Services      | Port exposé        | Daemon |   Types (systemd -> node exposter - http/https -> blackbox) |
| :-:       | :-:       | :-:       | :-:       | :-: |
| pfsense-core.homelab | https | 443 | / | https |
| dns-core.homelab | ssh | 22 | sshd.service | systemd | 
| dns-core.homelab | bind9 | 53 | bind9.service | systemd |
| admin-core.homelab| ssh | 22 | sshd.service | systemd |
| ansible-core.homelab| ssh | 22 | sshd.service | systemd |
| acme-core.homelab| ssh | 22 | sshd.service | systemd |
| vaultwarden-core.homelab| ssh | 22 | sshd.service | systemd |
| vaultwarden-core.homelab| https | Vaultwarden | docker.service | https |
| gitlab-core.homelab| ssh | 22 | sshd.service | systemd |
| gitlab-core.homelab| https | Gitlab | nginx (via Gitlab-ce) | https |
| prometheus-core.homelab| ssh | 22 | sshd.service | systemd |
| prometheus-core.homelab| Prometheus | 9090 | podman.service | http |
| grafana-core.homelab| ssh | 22 | sshd.service | systemd |
| grafana-core.homelab| Grafana | 3000 | podman.service | http |
| alertmanager-core.homelab| ssh | 22 | sshd.service | systemd |
| alertmanager-core.homelab| Alertmanager | 9093 | podman.service | http |
| reverseproxysup-core.homelab| ssh | 22 | sshd.service | systemd |
| reverseproxysup-core.homelab| Reverse proxy | 443 | nginx.service | https |
| mail-core.homelab| ssh | 22 | sshd.service | systemd |
| mail-core.homelab| smtp | 25 | postfix.service | systemd |

---

# 2. Observabilité et supervision sur les services de type http et https

> Afin d'observer et de superviser les services web exposé nous allons utiliser `blackbox`. Ce service va être ajouté au niveau du serveur `prometheus-core` via un container géré par podman, tout comme le container `prometheus`. Nous n'avons pas d'actions à réaliser pour `node exporter`, celui étant directement géré par le container `prometheus` (voir le chapitre 13).

> Il est important d'ouvrir les flux via `nftables` pour toutes les url que l'on veut checker avec `blackbox`.

> Par défaut sous Podman, deux containers exécutés sur un même host ne partage pas un bridge commun et donc ne peuvent pas communiquer entre eux. C'est pour cela qu'il est nécessaire de créer un réseau podman et d'y intégrer les deux containers `prometheus` et `blackbox-exporter`.

```bash
sudo podman network create prometheus-network
```

Exécution du container `blackbox` sur le serveur `prometheus-core`

```bash
sudo podman run -d \
    --name blackbox-exporter \
    --network prometheus-network \
    -p 9115:9115 \
    quay.io/prometheus/blackbox-exporter:latest
```

Ajout au niveau de la configuration du container `prometheus` via le fichier `/opt/prometheus/prometheus.yml`

```yml
- job_name: 'blackbox_http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - https://vaultwarden.ng-hl.com
        - https://gitlab.ng-hl.com
        - https://prometheus.ng-hl.com
        - https://grafana.ng-hl.com
        - https://alertmanager.ng-hl.com
        - https://pfsense.ng-hl.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

Restart du container `prometheus` pour charger la nouvelle configuration

```bash
sudo podman restart prometheus
```

---

# 3. Observabilité et supervision sur les services de type systemd