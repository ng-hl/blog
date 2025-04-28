---
date: '2025-04-28T23:53:13+02:00'
draft: false
title: 'Kubernetes - Introduction'
summary: "Introduction à Kubernetes"
tags: ["kubernetes"]
categories: ["Kubernetes"]
---

> Cette série d'articles m'accompagne dans mon parcours d'apprentissage concernant Kubernetes et la certification CKA (Certified Kubernetes Administrator)

---

# Qu'est-ce que Kubernetes ?

Kubernetes ou K8s est un produit open source (ouvert par Google en 2014) qui permet le déploiement, le scaling et la gestion automatique des applications conteneurisées. En d'autres termes, il s'agit d'un orchestrateur de conteneurs.

# Todo list

- [ ] Kubernetes
    - [ ] Concepts
        - [ ] Architecture globale
            - [ ] Composants de la control plane
                - [ ] kube-apiserver
                - [ ] etcd
                - [ ] kube-scheduler
                - [ ] kube-controller-manager
                - [ ] cloud-controller-manager
            - [ ] Composants des nodes
                - [ ] kubelet
                - [ ] kube-proxy
                - [ ] Container runtime
            - [ ] Eléments supplémentaires (addons)
        - [ ] Les objets
        - [ ] Architecture d'un cluster
            - [ ] Les noeuds
            - [ ] Les controlleurs
            - [ ] ...