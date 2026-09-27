# Installation de Prometheus et Grafana

## Table des matières

* [Prérequis](#prérequis)
* [1. Redirection de ports (Port Forwarding)](#1-redirection-de-ports-port-forwarding)
* [2. Installation de Prometheus](#2-installation-de-prometheus)

  * [2.1 Téléchargement et Configuration](#21-téléchargement-et-configuration)
  * [2.2 Création du Service Systemd](#22-création-du-service-systemd)
* [3. Installation de Grafana](#3-installation-de-grafana)

  * [3.1 Téléchargement et Initialisation](#31-téléchargement-et-initialisation)
  * [3.2 Création du Service Systemd](#32-création-du-service-systemd)
* [4. Connexion Prometheus & Grafana](#4-connexion-prometheus--grafana)

  * [4.1 Ajout de la Data Source](#41-ajout-de-la-data-source)
  * [4.2 Visualisation (Dashboards)](#42-visualisation-dashboards)
* [5. Configuration des Exporters (Exemple : Blackbox)](#5-configuration-des-exporters-exemple--blackbox)

  * [5.1 Installation de Blackbox Exporter](#51-installation-de-blackbox-exporter)
  * [5.2 Intégration dans Prometheus](#52-intégration-dans-prometheus)
  * [5.3 Vérification des Cibles](#53-vérification-des-cibles)

---

## Prérequis

* 1 Machine Virtuelle Linux (distribution au choix, ex: Debian/Ubuntu/Rocky).
* 1 Navigateur web sur la machine hôte.
* Les droits `root` ou `sudo` sur la VM.
* Une connexion Internet depuis la VM.

> 💡 **Note Réseau :** Pour des raisons pratiques, la VM est configurée avec une carte réseau en mode **NAT**. L'accès aux interfaces se fait donc via une redirection de ports sur l'hôte. Si vous utilisez un hyperviseur de Type 1 (Bare Metal) ou un mode Pont (Bridge), l'accès se fera directement via l'IP de la VM.

---

## 1. Redirection de ports (Port Forwarding)

**Hyperviseur utilisé :** Oracle VirtualBox

Prometheus utilise le port **9090**, Grafana le port **3000** et Blackbox Exporter le port **9115**.

Dans les paramètres de votre VM sous VirtualBox :

**Réseau → Avancé → Redirection de ports**

Ajoutez les règles suivantes :

| **Nom**    | **Protocole** | **IP Hôte**         | **Port Hôte** | **IP Invité** | **Port Invité** |
| ---------- | ------------- | ------------------- | ------------: | ------------- | --------------: |
| grafana    | TCP           | `127.0.0.1` ou vide |        `3000` | vide          |          `3000` |
| prometheus | TCP           | `127.0.0.1` ou vide |        `9090` | vide          |          `9090` |
| blackbox   | TCP           | `127.0.0.1` ou vide |        `9115` | vide          |          `9115` |

> 💡 **Remarque :** La redirection du port `9115` n'est pas obligatoire si Blackbox Exporter est uniquement utilisé par Prometheus sur la VM. Elle est utile pour tester directement son interface depuis la machine hôte.

---

# 2. Installation de Prometheus

L'installation est réalisée à partir des binaires officiels (archive `tar.gz`).

## 2.1 Téléchargement et Configuration

* **Version de référence :** Linux amd64 v3.12.0
* **Checksum SHA256 :** `20da47f8e5303f74aecb78edd7f7e39041dac08ac4939dba75efd7a900ae8867`

Exécutez les commandes suivantes dans votre terminal :

```bash
# Déplacement dans le dossier temporaire
cd /tmp

# Téléchargement de l'archive
wget https://github.com/prometheus/prometheus/releases/download/v3.12.0/prometheus-3.12.0.linux-amd64.tar.gz

# Vérification du hash
sha256sum prometheus-3.12.0.linux-amd64.tar.gz

# Extraction
tar -xvzf prometheus-3.12.0.linux-amd64.tar.gz

# Déplacement dans le dossier d'installation
sudo mv prometheus-3.12.0.linux-amd64 /opt/prometheus
```

Le résultat de la commande `sha256sum` doit correspondre à :

```text
20da47f8e5303f74aecb78edd7f7e39041dac08ac4939dba75efd7a900ae8867
```

### Création de l'utilisateur Prometheus

Pour des raisons de sécurité, Prometheus ne doit pas être exécuté directement avec les privilèges `root`.

```bash
sudo useradd --no-create-home --shell /usr/sbin/nologin prometheus
```

Création des répertoires nécessaires :

```bash
sudo mkdir -p /etc/prometheus
sudo mkdir -p /var/lib/prometheus
```

Copie des fichiers nécessaires :

```bash
sudo cp /opt/prometheus/prometheus /usr/local/bin/
sudo cp /opt/prometheus/promtool /usr/local/bin/

sudo cp -r /opt/prometheus/consoles /etc/prometheus/
sudo cp -r /opt/prometheus/console_libraries /etc/prometheus/
sudo cp /opt/prometheus/prometheus.yml /etc/prometheus/
```

Attribution des permissions :

```bash
sudo chown -R prometheus:prometheus /etc/prometheus
sudo chown -R prometheus:prometheus /var/lib/prometheus

sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool
```

### Configuration de Prometheus

Éditez le fichier :

```bash
sudo nano /etc/prometheus/prometheus.yml
```

Configuration minimale :

```yaml
global:
  scrape_interval: 15s

scrape_configs:

  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
```

Cette configuration permet à Prometheus de récupérer ses propres métriques.

Avant de créer le service, vérifiez que la configuration YAML est valide :

```bash
sudo -u prometheus promtool check config /etc/prometheus/prometheus.yml
```

Le résultat attendu est similaire à :

```text
SUCCESS: /etc/prometheus/prometheus.yml is valid prometheus config file syntax
```

---

## 2.2 Création du Service Systemd

Créez le fichier suivant :

```bash
sudo nano /etc/systemd/system/prometheus.service
```

Ajoutez :

```ini
[Unit]
Description=Prometheus Monitoring
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus

Type=simple

ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Rechargez Systemd :

```bash
sudo systemctl daemon-reload
```

Activez Prometheus au démarrage :

```bash
sudo systemctl enable prometheus
```

Démarrez le service :

```bash
sudo systemctl start prometheus
```

Vérifiez son état :

```bash
sudo systemctl status prometheus
```

Vous devez obtenir un service avec l'état :

```text
Active: active (running)
```

Pour consulter les logs :

```bash
sudo journalctl -u prometheus -f
```

Prometheus est maintenant accessible depuis la machine hôte à l'adresse :

```text
http://localhost:9090
```

Prometheus permet également de recharger sa configuration à chaud lorsque l'option correspondante est activée.

---

# 3. Installation de Grafana

Grafana permet de représenter graphiquement les données collectées par Prometheus.

L'installation suivante utilise le binaire Linux officiel de **Grafana OSS 13.2.2**. La documentation officielle fournit également des paquets `.deb`, `.rpm` et des binaires autonomes.

## 3.1 Téléchargement et Initialisation

Déplacez-vous dans `/tmp` :

```bash
cd /tmp
```

Téléchargez Grafana :

```bash
wget https://dl.grafana.com/grafana/release/13.2.2/grafana_13.2.2_34846740809_linux_amd64.tar.gz
```

Vérifiez le fichier téléchargé :

```bash
sha256sum grafana_13.2.2_34846740809_linux_amd64.tar.gz
```

Le SHA256 attendu est :

```text
9662c838a09824fdb072e5f6fbdd45b62cf541b20f3d609ea5011e6e5f544c8f
```

Extrayez l'archive :

```bash
tar -zxvf grafana_13.2.2_34846740809_linux_amd64.tar.gz
```

Déplacez le dossier :

```bash
sudo mv grafana-v13.2.2 /opt/grafana
```

> ⚠️ Si le nom exact du dossier extrait diffère, utilisez `ls` pour vérifier son nom avant le `mv`.

Créez l'utilisateur Grafana :

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin grafana
```

Créez les répertoires nécessaires :

```bash
sudo mkdir -p /etc/grafana
sudo mkdir -p /var/lib/grafana
sudo mkdir -p /var/log/grafana
```

Copiez le fichier de configuration :

```bash
sudo cp /opt/grafana/conf/sample.ini /etc/grafana/grafana.ini
```

Attribuez les permissions :

```bash
sudo chown -R grafana:grafana /opt/grafana
sudo chown -R grafana:grafana /etc/grafana
sudo chown -R grafana:grafana /var/lib/grafana
sudo chown -R grafana:grafana /var/log/grafana
```

### Configuration du port

Éditez :

```bash
sudo nano /etc/grafana/grafana.ini
```

Recherchez la section :

```ini
[server]
```

Vérifiez que le port est configuré sur :

```ini
[server]
http_addr =
http_port = 3000
```

Le port `3000` correspond au port utilisé par défaut par Grafana.

---

## 3.2 Création du Service Systemd

Créez le service :

```bash
sudo nano /etc/systemd/system/grafana.service
```

Ajoutez :

```ini
[Unit]
Description=Grafana
Wants=network-online.target
After=network-online.target

[Service]
User=grafana
Group=grafana

Type=simple

WorkingDirectory=/opt/grafana

ExecStart=/opt/grafana/bin/grafana server \
  --config=/etc/grafana/grafana.ini \
  --homepath=/opt/grafana

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Rechargez Systemd :

```bash
sudo systemctl daemon-reload
```

Activez Grafana au démarrage :

```bash
sudo systemctl enable grafana
```

Démarrez Grafana :

```bash
sudo systemctl start grafana
```

Vérifiez le service :

```bash
sudo systemctl status grafana
```

Le résultat attendu est :

```text
Active: active (running)
```

En cas de problème, consultez les logs :

```bash
sudo journalctl -u grafana -f
```

Grafana est maintenant accessible depuis la machine hôte :

```text
http://localhost:3000
```

Lors de la première connexion, Grafana demande de définir les identifiants du compte administrateur. Le port HTTP par défaut est `3000`.

---

# 4. Connexion Prometheus & Grafana

Une fois Prometheus et Grafana installés, Grafana doit être configuré pour utiliser Prometheus comme source de données.

## 4.1 Ajout de la Data Source

Ouvrez Grafana depuis votre navigateur :

```text
http://localhost:3000
```

Connectez-vous avec le compte administrateur créé lors de la première connexion.

Dans Grafana :

**Connections → Data sources → Add new data source**

Sélectionnez :

```text
Prometheus
```

Dans le champ **Prometheus server URL**, indiquez :

```text
http://localhost:9090
```

> 💡 Si Grafana et Prometheus sont installés sur la même VM, `localhost:9090` est suffisant.

Cliquez ensuite sur :

```text
Save & test
```

Si la connexion fonctionne, Grafana affiche un message indiquant que la source de données est correctement connectée.

---

## 4.2 Visualisation (Dashboards)

Pour vérifier que les données remontent correctement, ouvrez :

**Dashboards → New → New dashboard**

Ajoutez une visualisation.

Dans la requête PromQL, essayez :

```promql
up
```

Cette métrique permet notamment de vérifier si les cibles surveillées par Prometheus sont accessibles.

Pour vérifier directement Prometheus :

```promql
up{job="prometheus"}
```

Le résultat attendu est :

```text
1
```

Une valeur de `1` signifie que Prometheus considère cette cible comme accessible.

Vous pouvez également importer un dashboard communautaire depuis :

**Dashboards → New → Import dashboard**

Puis renseigner l'identifiant du dashboard souhaité et sélectionner la source de données Prometheus.

---

# 5. Configuration des Exporters (Exemple : Blackbox)

Les exporters permettent d'exposer des métriques supplémentaires à Prometheus.

Dans cet exemple, **Blackbox Exporter** est utilisé pour effectuer des sondes HTTP/HTTPS.

Blackbox Exporter peut notamment effectuer des sondes HTTP, HTTPS, DNS, TCP, ICMP et gRPC.

Le port utilisé par défaut est :

```text
9115
```

---

## 5.1 Installation de Blackbox Exporter

### Téléchargement

La version utilisée dans cette procédure est **0.28.0**.

Déplacez-vous dans `/tmp` :

```bash
cd /tmp
```

Téléchargez l'archive :

```bash
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.28.0/blackbox_exporter-0.28.0.linux-amd64.tar.gz
```

Vérifiez le SHA256 :

```bash
sha256sum blackbox_exporter-0.28.0.linux-amd64.tar.gz
```

Le SHA256 attendu est :

```text
a38d3a43f9b408b8fd8683f432342ec809d2c755761f6b43f7264270cb260be
```

Extrayez l'archive :

```bash
tar -xvzf blackbox_exporter-0.28.0.linux-amd64.tar.gz
```

Déplacez le dossier :

```bash
sudo mv blackbox_exporter-0.28.0.linux-amd64 /opt/blackbox_exporter
```

Créez l'utilisateur :

```bash
sudo useradd --no-create-home --shell /usr/sbin/nologin blackbox
```

Créez le répertoire de configuration :

```bash
sudo mkdir -p /etc/blackbox_exporter
```

Copiez le fichier de configuration fourni :

```bash
sudo cp /opt/blackbox_exporter/blackbox.yml /etc/blackbox_exporter/blackbox.yml
```

Copiez le binaire :

```bash
sudo cp /opt/blackbox_exporter/blackbox_exporter /usr/local/bin/blackbox_exporter
```

Attribuez les permissions :

```bash
sudo chown -R blackbox:blackbox /etc/blackbox_exporter
sudo chown blackbox:blackbox /usr/local/bin/blackbox_exporter
```

### Configuration Blackbox

Éditez :

```bash
sudo nano /etc/blackbox_exporter/blackbox.yml
```

Pour un premier test HTTP, utilisez :

```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      method: GET
```

Vérifiez ensuite la configuration :

```bash
sudo -u blackbox blackbox_exporter \
  --config.file=/etc/blackbox_exporter/blackbox.yml \
  --config.check
```

### Création du service Systemd

Créez :

```bash
sudo nano /etc/systemd/system/blackbox_exporter.service
```

Ajoutez :

```ini
[Unit]
Description=Blackbox Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=blackbox
Group=blackbox

Type=simple

ExecStart=/usr/local/bin/blackbox_exporter \
  --config.file=/etc/blackbox_exporter/blackbox.yml

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Rechargez Systemd :

```bash
sudo systemctl daemon-reload
```

Activez le service :

```bash
sudo systemctl enable blackbox_exporter
```

Démarrez-le :

```bash
sudo systemctl start blackbox_exporter
```

Vérifiez son état :

```bash
sudo systemctl status blackbox_exporter
```

Le résultat attendu est :

```text
Active: active (running)
```

Consultez les logs avec :

```bash
sudo journalctl -u blackbox_exporter -f
```

---

## 5.2 Intégration dans Prometheus

Blackbox Exporter fonctionne selon le principe du **multi-target exporter pattern** : Prometheus demande à Blackbox Exporter de sonder une cible déterminée, puis récupère les métriques résultantes.

Éditez le fichier Prometheus :

```bash
sudo nano /etc/prometheus/prometheus.yml
```

Ajoutez le job suivant dans `scrape_configs` :

```yaml
  - job_name: "blackbox-http"

    metrics_path: /probe

    params:
      module: [http_2xx]

    static_configs:
      - targets:
          - https://example.com
          - https://www.google.com

    relabel_configs:

      - source_labels: [__address__]
        target_label: __param_target

      - source_labels: [__param_target]
        target_label: instance

      - target_label: __address__
        replacement: localhost:9115
```

La configuration complète peut donc ressembler à :

```yaml
global:
  scrape_interval: 15s

scrape_configs:

  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "blackbox-http"

    metrics_path: /probe

    params:
      module: [http_2xx]

    static_configs:
      - targets:
          - https://example.com
          - https://www.google.com

    relabel_configs:

      - source_labels: [__address__]
        target_label: __param_target

      - source_labels: [__param_target]
        target_label: instance

      - target_label: __address__
        replacement: localhost:9115
```

Vérifiez la configuration :

```bash
sudo -u prometheus promtool check config /etc/prometheus/prometheus.yml
```

Si la configuration est valide, redémarrez Prometheus :

```bash
sudo systemctl restart prometheus
```

Vérifiez :

```bash
sudo systemctl status prometheus
```

> 💡 Blackbox Exporter permet également de recharger sa configuration sans redémarrage lorsqu'une méthode de reload est utilisée. La documentation officielle décrit notamment le rechargement via `SIGHUP` ou l'endpoint `/-/reload`.

---

## 5.3 Vérification des Cibles

### Vérification directe de Blackbox Exporter

Depuis la VM, testez directement une cible :

```bash
curl "http://localhost:9115/probe?target=https://example.com&module=http_2xx"
```

Dans la réponse, recherchez :

```text
probe_success 1
```

Une valeur :

```text
probe_success 1
```

indique que la sonde a réussi.

Une valeur :

```text
probe_success 0
```

indique que la sonde a échoué.

Blackbox Exporter expose notamment la métrique `probe_success` pour indiquer le résultat de la sonde.

### Vérification depuis Prometheus

Ouvrez :

```text
http://localhost:9090
```

Allez dans :

**Status → Target health**

Vous devez retrouver le job :

```text
blackbox-http
```

Les cibles configurées doivent apparaître avec l'état :

```text
UP
```

Vous pouvez également effectuer la requête suivante dans Prometheus :

```promql
probe_success
```

Pour tester une cible particulière :

```promql
probe_success{instance="https://example.com"}
```

### Vérification dans Grafana

Dans Grafana, créez un nouveau panneau et utilisez :

```promql
probe_success
```

Vous pouvez également afficher le temps de réponse avec :

```promql
probe_duration_seconds
```

Ou surveiller spécifiquement les cibles disponibles :

```promql
up{job="blackbox-http"}
```

---

# Vérification finale

À ce stade, les trois services doivent être actifs :

```bash
sudo systemctl status prometheus
sudo systemctl status grafana
sudo systemctl status blackbox_exporter
```

Les trois services doivent afficher :

```text
Active: active (running)
```

Les interfaces sont accessibles aux adresses suivantes depuis la machine hôte :

| Service           | Adresse                 |   Port |
| ----------------- | ----------------------- | -----: |
| Prometheus        | `http://localhost:9090` | `9090` |
| Grafana           | `http://localhost:3000` | `3000` |
| Blackbox Exporter | `http://localhost:9115` | `9115` |

### Résumé de l'architecture

```text
                         Machine hôte
                              │
                    ┌─────────┴─────────┐
                    │                   │
              Port 3000             Port 9090
                    │                   │
                    ▼                   ▼
              ┌──────────┐       ┌────────────┐
              │  Grafana │──────▶│ Prometheus │
              └──────────┘       └──────┬─────┘
                                        │
                                        │ scrape
                                        ▼
                                ┌─────────────────┐
                                │ Blackbox Exporter│
                                │     :9115       │
                                └────────┬────────┘
                                         │
                                         │ HTTP/HTTPS
                                         ▼
                                  Cibles surveillées
```

La chaîne complète est donc :

```text
Blackbox Exporter
       ↓
    métriques
       ↓
   Prometheus
       ↓
  Data Source
       ↓
    Grafana
       ↓
   Dashboards
```

La configuration permet ainsi de surveiller la disponibilité de services HTTP/HTTPS avec Prometheus et de visualiser les résultats dans Grafana.
