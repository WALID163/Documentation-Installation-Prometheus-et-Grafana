

md_content = """# Guide d'Installation : Prometheus & Grafana

**Auteur :** Walid  
**Document :** Note Technique Personnelle Installation Prometheus+Grafana

---

## Table des matières
- [Prérequis](#prérequis)
- [1. Redirection de ports (Port Forwarding)](#1-redirection-de-ports-port-forwarding)
- [2. Installation de Prometheus](#2-installation-de-prometheus)
  - [2.1 Téléchargement et Configuration](#21-téléchargement-et-configuration)
  - [2.2 Création du Service Systemd](#22-création-du-service-systemd)
- [3. Installation de Grafana](#3-installation-de-grafana)
  - [3.1 Téléchargement et Initialisation](#31-téléchargement-et-initialisation)
  - [3.2 Création du Service Systemd](#32-création-du-service-systemd)
- [4. Connexion Prometheus & Grafana](#4-connexion-prometheus--grafana)
  - [4.1 Ajout de la Data Source](#41-ajout-de-la-data-source)
  - [4.2 Visualisation (Dashboards)](#42-visualisation-dashboards)
- [5. Configuration des Exporters (Exemple : Blackbox)](#5-configuration-des-exporters-exemple--blackbox)
  - [5.1 Installation de Blackbox Exporter](#51-installation-de-blackbox-exporter)
  - [5.2 Intégration dans Prometheus](#52-intégration-dans-prometheus)
  - [5.3 Vérification des Cibles](#53-vérification-des-cibles)

---

## Prérequis

* 1 Machine Virtuelle Linux (distribution au choix, ex: Debian/Ubuntu/Rocky).
* 1 Navigateur web sur la machine hôte.

> 💡 **Note Réseau :** Pour des raisons pratiques, la VM est configurée avec une carte réseau en mode **NAT**. L'accès aux interfaces se fait donc via une redirection de ports sur l'hôte. Si vous utilisez un hyperviseur de Type 1 (Bare Metal) ou un mode Pont (Bridge), l'accès se fera directement via l'IP de la VM.

---

## 1. Redirection de ports (Port Forwarding)

**Hyperviseur utilisé :** Oracle VirtualBox

Prometheus utilise le port **9090** et Grafana le port **3000**.  
Dans les paramètres de votre VM sous VirtualBox : **Réseau** -> **Avancé** -> **Redirection de ports**, ajoutez les règles suivantes :

| Nom | Protocole | IP Hôte | Port Hôte | IP Invité | Port Invité |
| :--- | :---: | :---: | :---: | :---: | :---: |
| grafana | TCP | `127.0.0.1` ou vide | `3000` | vide | `3000` |
| prometheus | TCP | `127.0.0.1` ou vide | `9090` | vide | `9090` |

---

## 2. Installation de Prometheus

L'installation est réalisée à partir des binaires officiels (archive tar.gz).

### 2.1 Téléchargement et Configuration

* **Version de référence :** Linux amd64 v3.12.0
* **Checksum SHA256 :** `20da47f8e5303f74aecb78edd7f7e39041dac08ac4939dba75efd7a900ae8867`

Exécutez les commandes suivantes dans votre terminal :

```bash
# Déplacement dans le dossier temporaire
cd /tmp

# Téléchargement de l'archive (ajoutez --no-check-certificate si problème SSL)
wget [https://github.com/prometheus/prometheus/releases/download/v3.12.0/prometheus-3.12.0.linux-amd64.tar.gz](https://github.com/prometheus/prometheus/releases/download/v3.12.0/prometheus-3.12.0.linux-amd64.tar.gz)

# Vérification du hash
sha256sum prometheus-3.12.0.linux-amd64.tar.gz

# Extraction
tar -xvzf prometheus-3.12.0.linux-amd64.tar.gz
