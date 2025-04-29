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

> Pour le moment, la todo list ci-dessous est élaborée grâce au points abordés dans la documentation officielle de Kubernetes.

- [ ] Kubernetes
    - [ ] Concepts
        - [ ] Architecture globale
            - [ ] Composants de la control plane
                - [ ] kube-apiserver
                - [ ] etcd
                - [ ] kube-scheduler
                - [ ] kube-controller-manager
                - [ ] cloud-controller-manager
            - [ ] Composants des nodes
                - [ ] kubelet
                - [ ] kube-proxy
                - [ ] Container runtime
            - [ ] Eléments supplémentaires (addons)
        - [ ] Les objets
        - [ ] Architecture d'un cluster
            - [ ] Les noeuds
            - [ ] Les controlleurs
            - [ ] Cloud controller manager
            - [ ] Self-Healing
            - [ ] Container runtime Interface (CRI)
            - [ ] Garbage collection
        - [ ] Les conteneurs
        - [ ] Charge de travail (Workloads)
            - [ ] Les pods
                - [ ] Cycle de vie
                - [ ] Initialisation
                - [ ] Sidecar
                - [ ] Conteneurs éphémères
                - [ ] Perturbations
                - [ ] QoS
                - [ ] Namespaces
                - [ ] API
            - [ ] Autoscaling des workloads
            - [ ] Gestion des workloads
        - [ ] Les services
            - [ ] Service
            - [ ] Ingress
            - [ ] Ingress controllers
            - [ ] Gateway API
            - [ ] EndpointSlices
        - [ ] Les réseaux
            - [ ] Politique des réseaux
            - [ ] DNS
            - [ ] IPv4/IPv6
            - [ ] Routage
            - [ ] Le réseaux sous Windows
        - [ ] Load balancing
            - [ ] ClusterIP allocation
            - [ ] Politique du trafic interne
        - [ ] Stockage
            - [ ] Volumes
            - [ ] Volumes peristents
            - [ ] Volumes projetés
            - [ ] Volumes éphémères
            - [ ] Approvisionnement dynamique de volume
            - [ ] Volumes snapshot
            - [ ] Capacité de stockage
            - [ ] Limites
            - [ ] Observabilité de l'état de santé des volumes
            - [ ] Stockage sous Windows
        - [ ] Configuration
            - [ ] Bonnes pratiques
            - [ ] ConfigMaps
            - [ ] Secrets
            - [ ] Sondes liveness, readiness et startup
            - [ ] Gestion des ressources pour les pods et les containers
            - [ ] Fichier kubeconfig
        - [ ] Sécurité
            - [ ] Cloud
            - [ ] Pods
            - [ ] Comptes de service
            - [ ] Kubernetes API
            - [ ] RBAC
            - [ ] Bonnes pratiques pour les secrets
            - [ ] Mutli-tenants
            - [ ] Hardening
            - [ ] Contraintes de sécurité au niveau du noyau Linux
            - [ ] Checklist
        - [ ] Politique
            - [ ] Limitation
            - [ ] Quotas
            - [ ] Limitation et réservation du PID
        - [ ] Orchestration
            - [ ] Les orchestrateurs Kubernetes
            - [ ] Assignation de pods à un noeud
            - [ ] ...
        - [ ] Administration d'un cluster

