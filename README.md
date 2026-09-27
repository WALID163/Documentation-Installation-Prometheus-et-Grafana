#  Installation de Prometheus & Grafana

Ce dépôt contient une procédure complète pour installer et configurer une stack de **supervision et de monitoring** basée sur **Prometheus** et **Grafana** sous Linux.

L'objectif est de mettre en place une solution permettant de **collecter, surveiller et visualiser des métriques**, avec notamment l'utilisation de **Blackbox Exporter** pour vérifier la disponibilité de services HTTP/HTTPS.

##  Composants

* **Prometheus** — collecte et stockage des métriques
* **Grafana** — visualisation des métriques sous forme de dashboards
* **Blackbox Exporter** — supervision de services HTTP/HTTPS
* **Systemd** — gestion des services
* **VirtualBox NAT** — accès aux services via redirection de ports

##  Prérequis

* Une machine virtuelle Linux
* VirtualBox ou un hyperviseur équivalent
* Une connexion Internet
* Les droits `root` ou `sudo`
* Un navigateur web

##  Installation

La procédure complète, détaillée étape par étape, est disponible dans le fichier :

 **[ Documentation d'installation](./install.md)**

Elle couvre notamment :

1. La redirection des ports
2. L'installation de Prometheus
3. L'installation de Grafana
4. La connexion entre Prometheus et Grafana
5. L'installation de Blackbox Exporter
6. La configuration des cibles à superviser
7. La vérification du fonctionnement de l'ensemble

##  Accès aux services

Une fois l'installation terminée :

| Service           | Adresse                 |
| ----------------- | ----------------------- |
| Prometheus        | `http://localhost:9090` |
| Grafana           | `http://localhost:3000` |
| Blackbox Exporter | `http://localhost:9115` |

##  Structure du dépôt

```text
.
├── README.md
└── install.md
```

##  Architecture

```text
             ┌────────────┐
             │   Grafana  │
             │    :3000   │
             └─────┬──────┘
                   │
                   ▼
             ┌────────────┐
             │ Prometheus │
             │    :9090   │
             └─────┬──────┘
                   │
                   ▼
          ┌──────────────────┐
          │ Blackbox Exporter│
          │      :9115       │
          └────────┬─────────┘
                   │
                   ▼
            Services surveillés
```

---

>  **Note :** Cette documentation a été réalisée à des fins d'apprentissage et peut être adaptée à différents environnements Linux et hyperviseurs.
