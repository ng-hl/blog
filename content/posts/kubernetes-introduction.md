---
date: '2025-04-28T23:53:13+02:00'
draft: false
title: 'Kubernetes - Introduction'
summary: "Introduction à Kubernetes"
tags: ["kubernetes"]
categories: ["Kubernetes"]
---

> Cette série d'articles m'accompagne dans mon parcours d'apprentissage concernant Kubernetes et la certification CKA (Certified Kubernetes Administrator)

# Qu'est-ce que Kubernetes ?

Kubernetes ou K8s est un produit open source (ouvert par Google en 2014) qui permet le déploiement, le scaling et la gestion automatique des applications conteneurisées. En d'autres termes, il s'agit d'un orchestrateur de conteneurs.

# Todo list

> Pour le moment, la todo list ci-dessous est élaborée grâce au points abordés dans la documentation officielle de Kubernetes et par rapport à la playlist Kubernetes V2 proposée par Xavki.

- [ ] Kubernetes
    - [x] Architecture
        - [x] Composants de la control plane
            - [x] kube-apiserver
            - [x] etcd
            - [x] kube-scheduler
            - [x] kube-controller-manager
            - [x] cloud-controller-manager
        - [x] Composants des nodes
            - [x] kubelet
            - [x] kube-proxy
            - [x] Containers
            - [x] Container runtime
    - [x] Outils de CLI
        - [x] kube-ps1
        - [x] kubextl / kubens
        - [x] aliases
        - [x] Autocomplétion    
    - [x] Namespaces
    - [x] Pods
    - [x] ReplicaSet
    - [x] Deployment
        - [X] Stratégie
    - [x] Rollout et history
    - [x] Les services
        - [x] ClusterIP
        - [x] NodePort
        - [x] LB
        - [x] ExternalName
        