---
date: '2025-08-19T21:29:01+02:00'
draft: false
title: 'Kubernetes - Architecture'
summary: "L'architecture de Kubernetes"
tags: ["kubernetes"]
categories: ["Kubernetes"]
---

> Cette série d'articles m'accompagne dans mon parcours d'apprentissage concernant Kubernetes et la certification CKA (Certified Kubernetes Administrator)

> Cet article propose un tour d'horizon concernant l'architecture et les éléments clés de Kubernetes.

---

# 1. La control plane

- [x] Control plane
    - [x] kubeapi-server
    - [x] etcd
    - [x] kube-scheduler
    - [x] kube-controller-manager
    - [x] cloud-controller-manager

![Architecture Control Plane](/images/architecture_control_plane.png)

La `control plane` représente l'ensemble des node `master`. C'est à dire, les machines qui portent les éléments décisionnels et de pilotage pour le cluster Kubernetes. Une control plane est composée de 1 à plusieurs master node, idéalement 3, 5, 7, etc.

Le `kubeapi-server` est le point d'entrée pour interagir avec les master nodes. Il est possible de communiquer avec cet élément via des outils comme `kubectl` ou via des requêtes HTTP via `curl`.

La base de données `etcd` contient toutes les informations du cluster Kubernetes, y compris le `desired state` et le `current state`.

Le `controller manager` récupère des informations présentes au sein de `etcd` à travers `api-server` pour déclencher des actions (création, suppression, etc.).

Le `scheduler` va gérer les ressources et appliquer du filtering et du scoring.

Le ̀`cloud-controller-manager` est l'élément qui permet l'interraction avec l'API d'un cloud provider.

---

# 2. Les workers (les nodes)

- [x] Composants des workers (nodes)
    - [x] kubelet
    - [x] kube-proxy
    - [x] Containers
    - [x] Container runtime