Installation de Prometheus et GrafanaAuteur : JOUAL WalidEntreprise : GUERLAINTable des matièresPrérequisPort Forwarding ou Redirection de portsInstallation de Prometheus3.1 Installation3.2 Création d'un service au démarrage (Prometheus)Installation de Grafana4.1 Installation4.2 Création d'un service au démarrage (Grafana)Collecter les scrapes de Prometheus pour les afficher avec Grafana5.1 DashboardLes exporters6.1 Installation des exporters6.2 Intégration des exporters6.3 Vérification de la disponibilité des modulesPrérequis1 machine virtuelle sous Linux (distribution au choix).1 navigateur externe.Note : Pour des raisons techniques, la VM utilisée lors des installations a été gardée en NAT. Pour accéder à l'interface de nos serveurs, un simple forwarding de port a été effectué.En cas de réalisation directe sur un hyperviseur de type 1, les modalités d'accès aux interfaces changent légèrement à cause de la configuration réseau.1. Port Forwarding ou Redirection de portsHyperviseur utilisé : Oracle VirtualBoxDans VirtualBox, avant toutes choses, il faut savoir que Prometheus et Grafana utilisent respectivement les ports 9090 et 3000.Dans les paramètres de la VM → Réseau → Redirection de ports (Indiquer les ports du localhost à rediriger) :NomProtocoleIP hôtePort hôteIP invitéPort invitégrafanaTCP30003000prometheusTCP909090902. Installation de PrometheusL'installation de Prometheus dans cette documentation a été effectuée à partir de l'archive officielle et non du dépôt de distribution des paquets.Depuis le site officiel de Prometheus, cherchez la version la plus récente. Pour plus d'informations, se rendre sur : Getting started | Prometheus.2.1 InstallationVersion actuelle pour Linux : prometheus-3.12.0.linux-amd64.tar.gzSHA256 : 20da47f8e5303f74aecb78edd7f7e39041dac08ac4939dba75efd7a900ae8867Dans le terminal :Bashcd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v3.12.0/prometheus-3.12.0.linux-amd64.tar.gz
(Si problème de certificat, ajouter --no-check-certificate pour ignorer l'inspection SSL)Bash# Toujours vérifier l'empreinte d'un fichier
sha256sum prometheus-3.12.0.linux-amd64.tar.gz

# Extraction de l'archive
tar -xvzf prometheus-3.12.0.linux-amd64.tar.gz
Une fois le fichier dézippé, on crée un utilisateur avec des droits restreints ainsi que les dossiers où vont être stockées les données de l'application :Bashsudo useradd --no-create-home --shell /usr/sbin/nologin prometheus
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
sudo mkdir /usr/local/bin/prometheus    # À laisser en root (binaire de l'application)

sudo chown -R prometheus:prometheus /etc/prometheus
sudo chown -R prometheus:prometheus /var/lib/prometheus
Ensuite, il faut copier-coller les fichiers de Prometheus au bon endroit :Bashcd prometheus-3.12.0.linux-amd64
sudo cp prometheus /usr/local/bin/prometheus/
sudo cp promtool /usr/local/bin/prometheus/
sudo cp prometheus.yml /etc/prometheus/
2.2 Création d'un service au démarrage (Prometheus)Pour pouvoir démarrer et utiliser les fichiers fournis à la VM, il est nécessaire de créer un fichier de configuration .service pour systemd :Bashsudo nano /etc/systemd/system/prometheus.service
Configuration de base (prometheus.service) :Ini, TOML[Unit]
Description=prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus

Type=simple

ExecStart=/usr/local/bin/prometheus/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.listen-address=0.0.0.0:9090

Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
Prendre en compte le nouveau service et le démarrer :Bashsudo systemctl daemon-reload
sudo systemctl restart prometheus
sudo systemctl status prometheus
À l'aide d'un navigateur externe à la VM, vérifiez si Prometheus est bien installé en vous rendant sur http://localhost:9090.Prometheus est maintenant installé.Vous pouvez à présent supprimer le zip et le dossier extrait de Prometheus (ou attendre la fin complète des installations en cas de problèmes).3. Installation de GrafanaDe la même manière que pour Prometheus, l'installation de Grafana a été effectuée à partir de l'archive officielle et non du dépôt.Pour plus d'informations, se rendre sur la page : Set up Grafana | Grafana documentation.Version récente actuelle : 13.0.2Empreinte SHA256 : 6720d8b0b48d92e2b33b7bf30b38480c12964ccd87285e5e754aa554165edf2d3.1 InstallationDans /opt :Bashcd /opt
sudo mkdir grafana
cd grafana
sudo wget https://dl.grafana.com/grafana/release/13.0.2/grafana_13.0.2_2681684_9631_linux_amd64.tar.gz

# Vérification de l'empreinte
sha256sum grafana_13.0.2_2681684_9631_linux_amd64.tar.gz

# Extraction
sudo tar -xvzf grafana_13.0.2_2681684_9631_linux_amd64.tar.gz
(Important : il faut dézipper dans le dossier en question. En cas de changement de dossier par la suite, Grafana cessera de fonctionner).À partir de là, l'application est sur le serveur, il ne reste plus qu'à initialiser le tout :Bashcd grafana-13.0.2
./bin/grafana server web    # ou juste : ./bin/grafana server
Attendre l'initialisation du serveur, puis vérifier l'installation avec un navigateur externe sur http://localhost:3000.On crée ensuite un utilisateur dédié pour éviter de donner les droits root :Bashsudo useradd --no-create-home --shell /usr/sbin/nologin grafana
sudo chown -R grafana:grafana /opt/grafana
sudo mkdir -p /var/lib/grafana
sudo mkdir -p /var/log/grafana
sudo mkdir -p /etc/grafana
sudo chown -R grafana:grafana /var/lib/grafana /var/log/grafana /etc/grafana
3.2 Création d'un service au démarrage (Grafana)Dans la même démarche que pour Prometheus, il nous faut un fichier de configuration pour le daemon systemd :Bashsudoedit /etc/systemd/system/grafana.service
Configuration de base (grafana.service) :Ini, TOML[Unit]
Description=grafana
After=network.target

[Service]
Type=simple

User=grafana
Group=grafana

WorkingDirectory=/opt/grafana/grafana-13.0.2

ExecStart=/opt/grafana/grafana-13.0.2/bin/grafana server

Restart=always
RestartSec=5

Environment="GF_PATH_DATA=/var/lib/grafana"
Environment="GF_PATH_LOGS=/var/log/grafana"
Environment="GF_PATH_CONFIG=/etc/grafana"

[Install]
WantedBy=multi-user.target
Recharger le gestionnaire systemd et lancer le service :Bashsudo systemctl daemon-reload
sudo systemctl restart grafana
sudo systemctl status grafana
Fix probables en cas d'échec (fail) :Mauvais chemin (path) pour le WorkingDirectory.Mauvais chemin pour l'exécuteur (ExecStart).Problème de droits pour le fichier de données ou les dossiers dans /var ou /etc.Pour accorder les droits au dossier data en cas de besoin :Bashcd /opt/grafana/grafana-13.0.2
sudo chmod -R 777 data
4. Collecter les scrapes de Prometheus pour les afficher avec GrafanaDepuis l'interface de Grafana (http://localhost:3000) :Aller dans Connections → Data sources.Cliquer sur Add new data source.Sélectionner Prometheus.Ajouter l'adresse de Prometheus : http://localhost:9090 (ou http://ip_vm:9090).4.1 DashboardPour le test présent, l'installation de Node Exporter a été effectuée pour collecter des métriques sur l'état de santé de la VM. (Les outils de scrape sont faciles à installer, nous le verrons plus tard).En fonction des outils présents sur Prometheus, il est possible de créer des Dashboards ou de récupérer des templates pré-configurés sur le site officiel : Grafana dashboards | Grafana Labs.5. Les exportersLes exporters sont des outils qui permettent à Prometheus de récupérer des données de scan (scrapes) à partir d'une « cible ». L'installation se fait de manière assez simple et en deux temps.5.1 Installation des exportersDans notre exemple, nous allons utiliser un outil développé par la communauté Prometheus permettant de tester la disponibilité et les performances d'un service à l'aide de requêtes externes : Blackbox Exporter.Dans le terminal :Bash# Téléchargement de Blackbox v0.27.0
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.27.0/blackbox_exporter-0.27.0.linux-amd64.tar.gz
tar -xvf blackbox_exporter-0.27.0.linux-amd64.tar.gz

# Déplacement des fichiers dans le dossier de blackbox
cd blackbox_exporter-0.27.0.linux-amd64
sudo mv blackbox_exporter /usr/local/bin
sudo mkdir /etc/blackbox/
sudo mv blackbox.yml /etc/blackbox/
Configuration des droits d'accès :Bashsudo useradd --no-create-home --shell /usr/sbin/nologin blackbox
sudo chown blackbox:blackbox /usr/local/bin/blackbox_exporter
sudo chown -R blackbox:blackbox /etc/blackbox
Création du service systemd :Bashsudoedit /etc/systemd/system/blackbox.service
Configuration de base (blackbox.service) :Ini, TOML[Unit]
Description=blackbox
After=network.target

[Service]
Type=simple
User=blackbox
Group=blackbox

ExecStart=/usr/local/bin/blackbox_exporter \
  --config.file=/etc/blackbox/blackbox.yml

Restart=always

[Install]
WantedBy=multi-user.target
Activation et démarrage du service :Bashsudo systemctl daemon-reload
sudo systemctl start blackbox
sudo systemctl status blackbox
sudo systemctl enable blackbox
Pour des besoins spécifiques, il faudra parfois modifier le fichier /etc/blackbox/blackbox.yml. Dans notre situation actuelle, ce ne sera pas nécessaire.5.2 Intégration des exportersPour que Prometheus collecte les données de Blackbox, il faut lui indiquer qu'il existe un nouveau « job » à prendre en compte. La démarche est globalement identique pour tous les exporters.Dans le terminal :Bashsudoedit /etc/prometheus/prometheus.yml
Ajouter la configuration suivante dans le bloc scrape_configs :YAML  - job_name: "blackbox"
    metrics_path: /probe
    params:
      module: [icmp]
    static_configs:
      - targets:
          - 192.168.210.9
        labels:
          rooms: "Camera salle Kiss Kiss"
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115
ATTENTION : Le fichier .yml est très sensible à l'indentation.Pour vérifier si une erreur de syntaxe est présente, il suffit d'exécuter la commande suivante :Bashpromtool check config /etc/prometheus/prometheus.yml
Si une erreur apparaît à une ligne spécifique, vous pouvez inspecter les lignes en question avec la commande :Bashnl -ba /etc/prometheus/prometheus.yml | sed -n 'xx,xxp' # Remplacer xx par les numéros de lignes
Après vérification, il faut relancer Prometheus pour appliquer les changements :Bashsudo systemctl restart prometheus
5.3 Vérification de la disponibilité des modulesDepuis l'interface Web de Prometheus, rendez-vous dans l'onglet Status → Targets (ou Target health).Vérifiez si le nouveau module blackbox s'est installé et s'affiche correctement à l'état UP.
