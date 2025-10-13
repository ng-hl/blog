---
date: '2025-10-13T20:54:34+02:00'
draft: false
title: 'FEX - Prometheus'
summary: "FEX - Prometheus"
tags: ["fex","prometheus"]
categories: ["fex"]
---

> Ce document est une fiche d'exploitation pour l'administration des disques via LVM (Logical Volum Manager).

---

Ci-dessous, les fichiers et répertoires utiles pour `prometheus`

```bash
/etc/prometheus/prometheus.yml # Configuraiton de Prometheus
/etc/prometheus/rules # Alerting rules
/var/lib/prometheus # Stockage de la TSBD par défaut
/etc/prometheus/certs/ # Répertoires des certificats
```

Interraction avec le daemon `prometheus`

```bash
sudo systemctl status prometheus
sudo systemctl start prometheus
sudo systemctl restart prometheus
sudo systemctl stop prometheus
```

Vérification de la configuration

```bash
promtool check config /etc/prometheus/prometheus.yml
```

---

Cas où l'on souhaite purger les données supprimer de la configuration de `prometheus` sans attendre la fin de la période de rétention en utilisant `l'API admin` (cas avec Blackbox Exporter)

```bash
# Activation de l'API d'administration
--web.enable-admin-api

# Exemple avec podman
sudo podman run -d \
    -p 9090:9090 \
    --name prometheus \
    -v /opt/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
    -v prometheus-data:/prometheus \
    docker.io/prom/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --web.enable-admin-api

# Test de la réponse de l'API
curl -X GET http://localhost:9090/api/v1/status/flags | jq
```

Lister les instances actives dans la `TSDB` de `Prometheus`

```bash
curl -s "http://localhost:9090/api/v1/label/instance/values" | jq
```

Lister les jobs actifs

```bash
curl -s "http://localhost:9090/api/v1/label/job/values" | jq
```

Effacer une instance

```bash
curl -X POST -g "http://localhost:9090/api/v1/admin/tsdb/delete_series?match[]={instance=\"https://ancien-site.com\"}"
```

Purge des métriques `tombstoned`

```bash
curl -X POST http://localhost:9090/api/v1/admin/tsdb/clean_tombstones
```
