# TP : Mise en place d'une solution SIEM avec ELK Stack

**UE S10-3 - Sécurité des informations et management des événements**  
**Travaux Pratiques - Travail individuel**

Version anglaise : [README_EN.md](README_EN.md).

---

## Table des matières

1. [Contexte et objectifs](#1-contexte-et-objectifs)
2. [Partie 1 : Installation de la stack ELK](#2-partie-1--installation-de-la-stack-elk)
3. [Partie 2 : Configuration des agents Beats](#3-partie-2--configuration-des-agents-beats)
4. [Partie 3 : Visualisation et analyse dans Kibana](#4-partie-3--visualisation-et-analyse-dans-kibana)
5. [Partie 4 : Cas d'usage pratique](#5-partie-4--cas-dusage-pratique)
6. [Annexes](#6-annexes)

---

## 1. Contexte et objectifs

### 1.1 Présentation du TP

Ce TP vous permettra de mettre en pratique les concepts vus en cours en installant et configurant une solution SIEM complète basée sur la **stack ELK** (Elasticsearch, Logstash, Kibana).

Vous allez :
- Installer et configurer la stack ELK sur une VM SIEM Debian
- Configurer Suricata, Apache, Filebeat et Metricbeat sur une VM Linux à surveiller
- Configurer Winlogbeat sur un poste Windows pour envoyer les journaux vers le SIEM
- Créer des visualisations et des dashboards dans Kibana
- Analyser des événements de sécurité réels (alertes IDS, auth SSH, accès web, métriques, événements Windows)

### 1.2 Objectifs d'apprentissage

À l'issue de ce TP, vous serez capable de :

- Installer et configurer Elasticsearch, Logstash et Kibana
- Optimiser Elasticsearch pour fonctionner avec des ressources limitées
- Configurer Filebeat pour collecter les logs système
- Configurer Winlogbeat pour collecter les événements Windows
- Configurer Metricbeat pour collecter les métriques système
- Créer des visualisations et des dashboards dans Kibana
- Analyser des logs pour détecter des activités suspectes

### 1.3 Environnement technique

Le labo ESAIP utilise **deux VMs Linux** plus un **poste Windows** (navigateur Kibana et Winlogbeat).

#### VM SIEM (collecteur ELK)
- **Rôle** : Elasticsearch, Kibana, Logstash
- **OS** : Debian 13 (Trixie) — vérifié en labo : `13.5`
- **Stack observée** : Elastic **9.5.x** (dépôt `9.x`) — labo enseignant : **9.5.4**
- **RAM** : selon votre affectation Proxmox (souvent ~2 Go en salle ; le labo enseignant peut avoir davantage)
- **Accès** : SSH ; Kibana sur le port `5601`

#### VM à surveiller (Linux)
- **Rôle** : sources de logs / métriques envoyées vers le SIEM
- **OS** : Debian 13
- **Services cibles** : Suricata (alertes), SSH (`auth.log`), Apache, Metricbeat
- **Agents** : Filebeat + Metricbeat pointant vers l’IP de la VM SIEM (`IP_SIEM:9200`)

#### Poste Windows
- Accès à Kibana : `http://IP_SIEM:5601`
- **Winlogbeat** (même branche que le SIEM, ex. **9.5.4**) : journaux Application / System / Security → Elasticsearch `http://IP_SIEM:9200`
- Vérifiez quelle interface Windows atteint réellement le SIEM (`curl` + `Get-NetTCPConnection`) — un poste multi-homed peut joindre le labo via Ethernet alors que la Wi-Fi sert à autre chose

#### Architecture réseau

```
┌──────────────────┐         ┌──────────────────────┐
│  VM à surveiller │         │  VM SIEM (ELK)       │
│  (Debian 13)     │         │  (Debian 13)         │
│                  │         │                      │
│  - Suricata      │──logs──▶│  - Elasticsearch     │
│  - Apache        │  Beats  │  - Kibana :5601      │
│  - SSH/rsyslog   │────────▶│  - Logstash (opt.)   │
│  - Filebeat      │         │                      │
│  - Metricbeat    │         │                      │
└──────────────────┘         └──────────▲───────────┘
                                        │
┌──────────────────┐                    │
│  Poste Windows   │──── Winlogbeat ────┘
│  - navigateur    │     :9200
│  - Winlogbeat    │
└──────────────────┘
         (réseau labo)
```

### 1.4 Prérequis

Avant de commencer, assurez-vous d'avoir :

- Accès SSH à la VM SIEM **et** à la VM à surveiller
- Droits administrateur (sudo) sur les deux VMs
- Accès réseau entre les deux VMs (port `9200` vers le SIEM) et depuis votre PC vers Kibana (`5601`)
- Navigateur web moderne (Chrome, Firefox, Edge)
- Connaissances de base en Linux (commandes, édition de fichiers)

### 1.5 Durée estimée

- **Partie 1** : 2-3 heures
- **Partie 2** : 2-3 heures
- **Partie 3** : 1-2 heures
- **Partie 4** : 1-2 heures

**Total** : 6-10 heures de travail

---

## 2. Partie 1 : Installation de la stack ELK

### 2.1 Prérequis et vérification de l'environnement

#### Étape 1 : Connexion à la VM SIEM

Ouvrez une session SSH sur la **VM SIEM** (collecteur ELK) : c’est sur cette machine que vous installerez Elasticsearch, Kibana et Logstash.

```bash
ssh UTILISATEUR@IP_SIEM
```

`IP_SIEM` et `UTILISATEUR` : valeurs indiquées sur [Proxmox](https://10.3.2.10:8006). Les étapes d’installation ELK ci-dessous se font **sur cette VM**.

#### Étape 2 : Vérification du système

Avant d’installer ELK, contrôlez l’OS, le noyau, la RAM et l’espace disque — le heap JVM dépend de la mémoire disponible.

```bash
# Version de Debian
cat /etc/debian_version

# Informations système
uname -a

# Mémoire disponible
free -h

# Espace disque
df -h
```

- `free -h` : mémoire libre/utilisée (lisible pour un humain).
- `df -h` : espace disque par partition.

**Vérification** : notez la RAM (`free -h`). Avec ~2 Go, limitez le heap JVM (voir §2.2). Avec davantage de RAM, vous pouvez monter le heap (ex. `1g`) sans saturer le système.

#### Étape 3 : Mise à jour du système

Actualisez la liste des paquets puis appliquez les mises à jour disponibles, pour éviter des dépendances obsolètes au moment d’installer Elastic.

```bash
sudo apt update
sudo apt upgrade -y
```

`-y` : confirme automatiquement les questions d’apt (pas de prompt interactif).

#### Étape 4 : Installation des outils de base

Installez les utilitaires dont le TP a besoin (téléchargements, édition, clé GPG).

```bash
sudo apt install -y curl wget nano git gpg
```

---

### 2.2 Installation d'Elasticsearch

Si l'installation échoue, vérifiez que la procédure n'a pas changé dans la [documentation Elastic](https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-elasticsearch-with-debian-package).

#### Étape 1 : Ajout de la clé GPG Elastic

Téléchargez la clé publique Elastic et placez-la dans le keyring apt : apt ne fera confiance qu’aux paquets signés avec cette clé.

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
```

- `-qO -` : télécharge en silence et écrit sur la sortie standard (pour le pipe).
- `gpg --dearmor` : convertit la clé ASCII en format binaire attendu par apt.
- Le `|` (pipe) est **obligatoire** — sans lui, la clé n’est pas écrite dans le keyring.

#### Étape 2 : Installation d'apt-transport-https

Autorisez apt à récupérer des paquets via HTTPS (requis pour le dépôt Elastic).

```bash
sudo apt install -y apt-transport-https
```

#### Étape 3 : Ajout du dépôt Elastic

Enregistrez le dépôt Elastic 9.x dans les sources apt, en liant la clé GPG ajoutée à l’étape 1.

```bash
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/9.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-9.x.list
```

- `signed-by=...` : apt vérifie les paquets avec le keyring créé plus haut.
- `tee` : écrit la ligne dans le fichier sources tout en l’affichant à l’écran (avec `sudo`).

#### Étape 4 : Mise à jour et installation

Rechargez les listes de paquets (le nouveau dépôt apparaît), puis installez Elasticsearch.

```bash
sudo apt-get update && sudo apt-get install elasticsearch
```

`&&` : n’installe que si la mise à jour des listes a réussi.

#### Étape 5 : Configuration d'Elasticsearch (mémoire heap)

**IMPORTANT** : adaptez le heap à la RAM réelle de la VM SIEM.

Ouvrez (ou créez) le fichier d’options JVM dédié au heap — c’est ici que vous limiterez la mémoire d’Elasticsearch.

```bash
sudo nano /etc/elasticsearch/jvm.options.d/heapsizemem.options
```

**Note** : l’extension doit être `.options` (pas `.conf`) pour être prise en compte par Elasticsearch.

Avec **~2 Go** de RAM (salle typique), placez ceci dans le fichier :

```properties
# Heap limité à 512 Mo
-Xms512m
-Xmx512m
```

Avec **~8 Go** de RAM (labo enseignant / VM plus large) :

```properties
# Heap 1 Go — laisse de la marge pour Kibana et Logstash
-Xms1g
-Xmx1g
```

**Explication** :
- `-Xms` / `-Xmx` : taille initiale et maximale du heap JVM (gardez-les égales).
- La mémoire totale utilisée par Elasticsearch dépasse le heap (off-heap, buffers, processus auxiliaires).
- Vérifiez après démarrage avec `curl http://localhost:9200/_nodes/jvm?pretty` (`heap_max_in_bytes`).

#### Étape 6 : Configuration réseau d'Elasticsearch

Éditez la configuration principale pour exposer l’API HTTP, nommer le nœud et désactiver la sécurité (labo uniquement).

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

**Configuration pour Elasticsearch 9.x** (observé en labo : **9.5.4**) :

La configuration par défaut d'Elasticsearch 9.x inclut déjà certaines options. Vous devez vérifier et modifier les paramètres suivants :

```yaml
# Réseau et HTTP (déjà présent dans la config par défaut avec http.host)
# Vérifiez que ces lignes sont présentes :
http.host: 0.0.0.0
http.port: 9200

# Nom du cluster (optionnel, pour identification)
cluster.name: siem-cluster

node.name: siem-node-1

# Désactiver SSL/HTTPS pour le labo (à ne pas faire en production)
# Enable security features
xpack.security.enabled: false

xpack.security.enrollment.enabled: false

# Enable encryption for HTTP API client connections, such as Kibana, Logstash, and Agents
xpack.security.http.ssl:
  enabled: false
  keystore.path: certs/http.p12

# Enable encryption and mutual authentication between cluster nodes
xpack.security.transport.ssl:
  enabled: false
  verification_mode: certificate
  keystore.path: certs/transport.p12
  truststore.path: certs/transport.p12
  
cluster.initial_master_nodes: ["siem-node-1"]

```

**Notes importantes** :
- En production, vous devriez activer la sécurité et utiliser SSL/TLS.
- Dans Elasticsearch 9.x, utilisez `http.host` au lieu de `network.host`.
- Remplacez `"siem-node-1"` dans `cluster.initial_master_nodes` par la valeur de `node.name` que vous avez définie (ou gardez le nom de votre machine si vous n'avez pas changé `node.name`).
- Par défaut, `xpack.security.enabled` vaut `true` : pour le labo, forcez `false` comme ci-dessus, sinon Beats et `curl` sans identifiants échoueront.

#### Étape 7 : Démarrage d'Elasticsearch

Lancez le service maintenant, puis activez le démarrage automatique au boot pour qu’Elasticsearch survive aux redémarrages de la VM.

```bash
sudo systemctl start elasticsearch
```

```bash
sudo systemctl enable elasticsearch
```

Contrôlez que le service est bien `active (running)` :

```bash
sudo systemctl status elasticsearch
```

#### Étape 8 : Vérification du fonctionnement

Attendez 30-60 secondes que Elasticsearch initialise le cluster, puis interrogez l’API locale : une réponse JSON confirme que le nœud répond.

```bash
curl http://localhost:9200
```

Vous devriez voir une réponse JSON avec les informations du cluster.

**Vérification** : après redémarrage, confirmez que le heap appliqué correspond à votre fichier `.options` :

```bash
curl http://localhost:9200/_nodes/jvm?pretty
```

`?pretty` : formate le JSON pour le lire plus facilement. Cherchez `heap_used` et `heap_max` dans la réponse.

Testez aussi depuis un autre hôte du labo (remplacez `IP_SIEM`) : cela prouve que `http.host: 0.0.0.0` expose bien le port `9200` sur le réseau.

```bash
# Depuis un autre hôte du labo
curl http://IP_SIEM:9200
```

**Vérification** : si vous obtenez une réponse JSON, Elasticsearch fonctionne correctement.

#### Aide : Résolution d'un problème de mémoire

**Situation** : Elasticsearch ne démarre pas ou plante avec une erreur de mémoire.

**Procédure** :
1. Vérifiez les logs : `sudo journalctl -u elasticsearch -n 50`
2. Identifiez l'erreur liée à la mémoire
3. Ajustez les paramètres `-Xms` et `-Xmx` dans `/etc/elasticsearch/jvm.options`
4. Redémarrez Elasticsearch : `sudo systemctl restart elasticsearch`

**Indice** : Avec ~2 Go de RAM, partez sur un heap de **512m** (§2.2). Un heap de 1 Go convient plutôt à une VM SIEM plus large (~8 Go).

---

### 2.3 Installation de Kibana

#### Étape 1 : Installation

Kibana est dans le même dépôt Elastic déjà configuré. Installez le paquet pour disposer de l’interface web du SIEM.

```bash
sudo apt install -y kibana
```

#### Étape 2 : Configuration de Kibana

Éditez la config pour faire écouter Kibana sur toutes les interfaces et le pointer vers Elasticsearch local.

```bash
sudo nano /etc/kibana/kibana.yml
```

Modifiez (ou vérifiez) les paramètres suivants :

```yaml
# Adresse d'écoute
server.host: "0.0.0.0"

# Port
server.port: 5601

# URL d'Elasticsearch
elasticsearch.hosts: ["http://localhost:9200"]
```

- `server.host: "0.0.0.0"` : accessible depuis votre PC Windows, pas seulement en local.
- `elasticsearch.hosts` : Kibana lit/écrit les données via l’API Elasticsearch.

#### Étape 3 : Démarrage de Kibana

Démarrez Kibana et activez le démarrage au boot.

```bash
sudo systemctl start kibana
sudo systemctl enable kibana
```

Contrôlez l’état du service :

```bash
sudo systemctl status kibana
```

**Note importante** : 
- Kibana peut prendre **1-2 minutes** pour démarrer complètement.
- **Lors du premier démarrage**, Kibana effectue des migrations d'objets sauvegardés qui peuvent prendre **3-5 minutes** supplémentaires.
- Pendant ce temps, la page web peut rester en chargement. C'est **normal**.
- Suivez les logs en direct (`-f` = follow) pour voir l’état des migrations :

  ```bash
  sudo journalctl -u kibana -f
  ```

  Vous devriez voir "Starting saved objects migrations" puis "Migration completed" ou similaire.

#### Étape 4 : Accès à Kibana

Depuis le navigateur de votre poste Windows, ouvrez l’URL ci-dessous (remplacez `IP_SIEM`) :

```
http://IP_SIEM:5601
```

**Vérification** : vous devriez voir la page d'accueil de Kibana.

#### Aide : Configuration de l'accès depuis Windows

**Situation** : Vous ne pouvez pas accéder à Kibana depuis votre PC Windows.

**Procédure** :
1. Vérifiez que Kibana écoute sur toutes les interfaces : `sudo netstat -tlnp | grep 5601`
2. Vérifiez le pare-feu de la VM : `sudo ufw status`
3. Si le pare-feu bloque, autorisez le port : `sudo ufw allow 5601/tcp`
4. Vérifiez la connectivité depuis votre PC : `ping IP_SIEM`
5. Testez depuis votre PC : `curl http://IP_SIEM:5601` ou ouvrez dans le navigateur

**Indice** : si un pare-feu est actif, ouvrez aussi le port 9200 pour qu'Elasticsearch reste joignable depuis les agents.

---

### 2.4 Installation de Logstash

#### Étape 1 : Installation de Java

Logstash tourne sur la JVM. Installez un JRE OpenJDK avant le paquet Logstash.

```bash
sudo apt install -y default-jre
```

Confirmez que `java` est bien dans le PATH :

```bash
java -version
```

---

#### Étape 2 : Installation de Logstash

Rechargez apt si besoin, puis installez Logstash (toujours depuis le dépôt Elastic 9.x).

```bash
sudo apt update
sudo apt install -y logstash
```

---

#### Étape 3 : Configuration de base de Logstash

Éditez la config principale pour pointer vers les répertoires de pipelines, logs et données internes.

```bash
sudo nano /etc/logstash/logstash.yml
```

Vérifiez / configurez les paramètres suivants :

```yaml
# Chemin des pipelines
path.config: /etc/logstash/conf.d

# Chemin des logs
path.logs: /var/log/logstash

# Chemin des données internes (important pour éviter les erreurs)
path.data: /var/lib/logstash
```

Créez les répertoires manquants et donnez-les à l’utilisateur `logstash` — sans cela, le service échoue souvent au démarrage.

```bash
sudo mkdir -p /var/log/logstash /var/lib/logstash
sudo chown -R logstash:logstash /var/log/logstash /var/lib/logstash
```

- `mkdir -p` : crée les dossiers (et parents) s’ils n’existent pas.
- `chown -R logstash:logstash` : propriétaire et groupe du service Logstash sur toute l’arborescence.

---

#### Étape 4 : Création d’un pipeline de test (mode service compatible)

**Important** :
Le pipeline `stdin {}` **ne doit pas être utilisé avec systemd**, car il provoque l’arrêt immédiat de Logstash.
On utilise donc un **input file** pour un vrai test.

Créez le fichier de pipeline de test : Logstash lira ce conf au prochain redémarrage.

```bash
sudo nano /etc/logstash/conf.d/test.conf
```

Configuration :

```ruby
input {
  file {
    path => "/tmp/logstash-test.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    mode => "tail"
  }
}
filter {
  # Aucun filtre pour le test
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "test-logs-%{+YYYY.MM.dd}"
  }
  stdout {
    codec => rubydebug
  }
}
```

- `sincedb_path => "/dev/null"` : ignore la position de lecture précédente (utile pour un test reproductible).
- `index => "test-logs-%{+YYYY.MM.dd}"` : crée un index Elasticsearch daté du jour.

Créez le fichier source lu par le pipeline et rendez-le accessible en écriture pour le test :

```bash
sudo touch /tmp/logstash-test.log
sudo chmod 666 /tmp/logstash-test.log
```

`chmod 666` : lecture/écriture pour tous (pratique en labo pour `echo >>` sans sudo) — **pas** pour la production.

---

#### Étape 5 : Démarrage de Logstash

Activez le service au boot et redémarrez-le pour charger le nouveau pipeline.

```bash
sudo systemctl enable logstash
sudo systemctl restart logstash
```

Contrôlez que le service tourne :

```bash
sudo systemctl status logstash
```

Vous devez voir :

```
Active: active (running)
```

---

#### Étape 6 : Test du pipeline

Dans un autre terminal, ajoutez des lignes au fichier surveillé : Logstash les lit, les envoie vers Elasticsearch et les affiche en debug.

```bash
echo "Bonjour Logstash" >> /tmp/logstash-test.log
echo "Deuxième message" >> /tmp/logstash-test.log
```

`>>` : ajoute en fin de fichier (sans écraser le contenu existant).

Résultat attendu :

* Les lignes apparaissent dans Elasticsearch. Dans Kibana : **Stack Management → Index Management**, index `test-logs-YYYY.MM.dd`.
* Logstash **reste actif** (il ne doit pas s'arrêter après le test).

---

#### Pipeline pour les logs Apache

**Objectif**

Créer un pipeline Logstash qui lit et parse les logs Apache.

---

##### Étape 1 : Installation d’Apache

Installez Apache sur la VM SIEM pour produire des `access.log` locaux que Logstash pourra parser (exercice pipeline ; la collecte recommandée en deux VMs reste Filebeat, §3.1).

```bash
sudo apt install -y apache2
```

Vérifiez qu’Apache répond en local (page par défaut) :

```bash
curl http://localhost
```

---

##### Étape 2 : Création du pipeline Apache

Créez le pipeline qui lit `/var/log/apache2/access.log`, parse avec Grok, et indexe dans Elasticsearch.

```bash
sudo nano /etc/logstash/conf.d/apache.conf
```

Configuration :

```ruby
input {
  file {
    path => "/var/log/apache2/access.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    mode => "tail"
  }
}

filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "apache-logs-%{+YYYY.MM.dd}"
  }
  stdout {
    codec => rubydebug
  }
}
```

**Important** :
Désactivez le pipeline de test pour ne garder qu’un pipeline actif (évite des index inutiles et du bruit) :

```bash
sudo mv /etc/logstash/conf.d/test.conf /etc/logstash/conf.d/test.conf.disabled
```

Ajoutez l’utilisateur `logstash` au groupe `adm` pour qu’il puisse lire les logs Apache (souvent en `640 root:adm`) :

```bash
sudo usermod -aG adm logstash
```

`-aG` : ajoute au groupe sans retirer les autres groupes existants.

---

##### Étape 3 : Redémarrage de Logstash

Rechargez Logstash pour prendre en compte `apache.conf` et le nouveau groupe.

```bash
sudo systemctl restart logstash
```

Consultez les 20 dernières lignes du journal du service (`--no-pager` : affichage direct, sans pager interactif) :

```bash
journalctl -u logstash -n 20 --no-pager
```

Vous devez voir :

* `Pipeline started`
* aucune erreur fatale

---

##### Étape 4 : Génération de trafic Apache

Générez quelques requêtes HTTP : chaque `curl` ajoute une ligne dans `access.log`, que Logstash enverra vers l’index `apache-logs-*`.

```bash
curl http://localhost
curl http://localhost
curl http://localhost
```

---

##### Étape 5 : Vérification dans Kibana

1. Allez dans **Stack Management → Data Views**
2. Créez une data view :

   ```
   apache-logs-*
   ```
3. Champ temporel : `@timestamp`
4. Allez dans **Discover**

Résultat attendu :

* Requêtes Apache visibles
* Champs parsés (`clientip`, `verb`, `request`, `response`, etc.)

---

**Indice**

Pattern Grok utilisé :

```
%{COMBINEDAPACHELOG}
```

Documentation officielle : [grok](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html)

---

## 3. Partie 2 : Configuration des agents Beats (VM à surveiller)

> **Où travailler** : les étapes Filebeat / Metricbeat / Suricata / Apache de cette partie se font sur la **VM à surveiller**, pas sur le SIEM.  
> Remplacez `IP_SIEM` par l’adresse IP de la VM ELK.  
> Ajoutez d’abord le dépôt Elastic `9.x` sur cette VM (même procédure GPG + `elastic-9.x.list` qu’en §2.2), puis installez les paquets Beats.

### 3.1 Installation et configuration de Filebeat

#### Étape 1 : Installation de Filebeat et rsyslog

Depuis Debian 8, journald est utilisé pour logger les événements système ; sans rsyslog, Filebeat ne trouve pas `/var/log/auth.log` / `/var/log/syslog` aux chemins classiques.

Installez rsyslog et démarrez-le immédiatement (`--now` = enable + start) pour écrire les logs classiques sur disque.

```bash
sudo apt install -y rsyslog
sudo systemctl enable --now rsyslog
```

Installez ensuite Filebeat (agent de collecte de fichiers/logs vers Elasticsearch).

```bash
sudo apt install -y filebeat
```

#### Étape 2 : Configuration de base (sortie vers le SIEM)

Éditez la config Filebeat pour envoyer les événements vers la **VM SIEM**, pas vers `localhost`.

```bash
sudo nano /etc/filebeat/filebeat.yml
```

Configurez la section `output.elasticsearch` pour cibler **le SIEM** (pas `localhost` si Filebeat tourne sur la VM à surveiller) :

```yaml
output.elasticsearch:
  hosts: ["IP_SIEM:9200"]
  # Sécurité désactivée côté labo (xpack.security.enabled: false)
```

`IP_SIEM:9200` : adresse et port HTTP d’Elasticsearch sur le collecteur. Vérifiez la connectivité : `curl http://IP_SIEM:9200` puis `sudo filebeat test output`.

#### Étape 3 : Activation des modules (système, Apache, Suricata)

Activez les modules Filebeat qui savent lire les logs SSH/auth, Apache et Suricata (EVE JSON) sans écrire les chemins à la main.

```bash
sudo filebeat modules enable system apache suricata
```

#### Étape 4 : Configuration des modules

**Système (SSH / auth)** — `/etc/filebeat/modules.d/system.yml` :

```yaml
- module: system
  syslog:
    enabled: true
  auth:
    enabled: true
```

**Apache** — `/etc/filebeat/modules.d/apache.yml` :

```yaml
- module: apache
  access:
    enabled: true
    var.paths: ["/var/log/apache2/access.log*"]
  error:
    enabled: true
    var.paths: ["/var/log/apache2/error.log*"]
```

**Suricata** — `/etc/filebeat/modules.d/suricata.yml` :

```yaml
- module: suricata
  eve:
    enabled: true
    var.paths: ["/var/log/suricata/eve.json"]
```

**Installation Suricata + règle de labo** (sur la VM à surveiller) :

Installez Suricata (IDS) et Apache (cible HTTP du test), puis démarrez Apache pour que le stimulus HTTP ait un service derrière.

```bash
sudo apt install -y suricata
# Apache doit être actif pour le test HTTP ci-dessous
sudo apt install -y apache2
sudo systemctl enable --now apache2
```

Écrivez une règle de labo dans le répertoire des règles Debian : `tee` crée le fichier avec le contenu entre `EOF` (heredoc).

```bash
sudo tee /var/lib/suricata/rules/lab-test.rules <<'EOF'
# Règle de labo défensive — alerte sur requête HTTP vers /esaip-lab-test
alert http any any -> $HOME_NET any (msg:"ESAIP-LAB-HTTP-TEST"; http.uri; content:"/esaip-lab-test"; sid:9000001; rev:1;)
EOF
```

- `<<'EOF'` : lit le texte jusqu’à la ligne `EOF` (guillemets simples = pas d’interpolation shell).
- `sid:9000001` : identifiant unique de la règle (évite les collisions avec les règles officielles).

Dans `/etc/suricata/suricata.yaml`, vérifiez que `af-packet` écoute **`eth0`** (pas seulement `lo`) et que `rule-files` charge au moins `lab-test.rules` :

```yaml
rule-files:
  - lab-test.rules
```

Démarrez Suricata et contrôlez qu’il est actif : sans service démarré, aucune alerte ne sera écrite dans `eve.json`.

```bash
sudo systemctl enable --now suricata
sudo systemctl status suricata
```

Les alertes sont dans `/var/log/suricata/eve.json`. Les règles gérées par `suricata-update` vivent sous `/var/lib/suricata/rules`.

**Générer une alerte qui traverse vraiment eth0** : un `curl` vers `localhost` sur la VM surveillée ne produit **pas** de trafic sur `eth0`. Lancez le stimulus **depuis une autre machine du labo** (ex. le SIEM) :

```bash
# Depuis la VM SIEM (remplacez IP_SURVEILLEE)
curl -s -o /dev/null -w "%{http_code}\n" http://IP_SURVEILLEE/esaip-lab-test
```

- `-s` : mode silencieux ; `-o /dev/null` : jette le corps de la réponse ; `-w "%{http_code}\n"` : affiche seulement le code HTTP.
- `IP_SURVEILLEE` : IP de la VM à surveiller (pas `localhost`, pas l’IP du SIEM).

Un code HTTP `404` est normal (l’URL n’existe pas sur Apache) : Suricata doit tout de même lever `ESAIP-LAB-HTTP-TEST` (`in_iface: eth0`). Vérifiez ensuite dans Kibana : `event.module: "suricata" and event.kind: "alert"` (signature : `rule.name`).

#### Étape 5 : Démarrage de Filebeat

Activez et démarrez Filebeat pour qu’il envoie en continu auth, Apache et Suricata vers Elasticsearch.

```bash
sudo systemctl enable --now filebeat
sudo systemctl status filebeat
```

Les index / data streams observés en labo : `.ds-filebeat-9.5.4-*` (pattern Kibana : `filebeat-*`).

#### Étape 6 : Vérification dans Kibana

1. Accédez à Kibana : `http://IP_SIEM:5601`
2. **Stack Management → Data Views** → pattern `filebeat-*`, champ `@timestamp`
3. Dans **Discover**, filtrez :
   - Alertes Suricata : `event.module: "suricata" and event.kind: "alert"` (champ signature : `rule.name`)
   - SSH / auth : `event.dataset: "system.auth"`
   - Apache : `event.dataset: "apache.access"`

#### Apache (service)

Si Apache n’est pas encore installé sur la VM à surveiller, installez-le, démarrez-le et générez une requête locale pour produire des lignes `access.log` que Filebeat enverra au SIEM.

```bash
sudo apt install -y apache2
sudo systemctl enable --now apache2
curl http://localhost
```

Générez du trafic, puis vérifiez `apache.access` dans Discover.

> **Note Logstash / Apache** : le pipeline Logstash `apache.conf` du §2.4 (lecture locale de `/var/log/apache2` sur le SIEM) reste un exercice utile sur la VM SIEM. En architecture deux VMs, la collecte Apache **recommandée** est Filebeat module `apache` sur la VM à surveiller.

---

### 3.2 Installation et configuration de Winlogbeat

> **Où travailler** : sur votre **poste Windows** (PowerShell **en tant qu’administrateur**).  
> Alignez la version Winlogbeat sur celle du SIEM (labo enseignant : **9.5.4**).  
> Remplacez `IP_SIEM` par l’adresse IP de la VM ELK.

#### Étape 0 : Vérifier la connectivité vers Elasticsearch

Avant d’installer l’agent, confirmez que **ce** poste Windows joint le SIEM sur le port `9200` (sinon Winlogbeat installé mais muet).

```powershell
curl.exe -s http://IP_SIEM:9200
# Doit renvoyer un JSON (cluster.name, version.number, …)

# Quelle IP source Windows est utilisée ?
Get-NetTCPConnection -RemoteAddress IP_SIEM -RemotePort 9200 |
  Select-Object LocalAddress, LocalPort, State
```

- `curl.exe` : le client HTTP Windows (évite l’alias PowerShell `curl` = `Invoke-WebRequest`).
- `Get-NetTCPConnection` : montre l’interface locale réellement utilisée vers le SIEM.

**Note labo** : un poste multi-homed (Wi-Fi + Ethernet) peut atteindre le SIEM via une interface différente de celle utilisée pour VNC ou le navigateur. Notez l’`LocalAddress` réellement utilisée (ex. observé en labo enseignant : Ethernet `172.16.192.4` → `10.2.1.3:9200`).

Si `curl` échoue : route (`Find-NetRoute -RemoteIPAddress IP_SIEM`), pare-feu Windows, et ouverture du port `9200` côté SIEM.

#### Étape 1 : Téléchargement et extraction

Téléchargez le ZIP Winlogbeat aligné sur la version du SIEM, extrayez-le, puis copiez les fichiers sous `Program Files` (emplacement standard du service).

```powershell
# PowerShell administrateur
$ver = "9.5.4"
Invoke-WebRequest -Uri "https://artifacts.elastic.co/downloads/beats/winlogbeat/winlogbeat-$ver-windows-x86_64.zip" -OutFile "$env:TEMP\winlogbeat.zip"
Expand-Archive "$env:TEMP\winlogbeat.zip" -DestinationPath "$env:TEMP\winlogbeat-extract" -Force
# Le ZIP crée un sous-dossier winlogbeat-9.5.4-windows-x86_64 — on le copie sous Program Files
New-Item -ItemType Directory -Path "C:\Program Files\Winlogbeat" -Force | Out-Null
Copy-Item -Path "$env:TEMP\winlogbeat-extract\winlogbeat-$ver-windows-x86_64\*" -Destination "C:\Program Files\Winlogbeat" -Recurse -Force
```

- `$ver` : version Winlogbeat (doit matcher le SIEM, ex. 9.5.4).
- `Invoke-WebRequest` / `Expand-Archive` : téléchargement puis extraction du ZIP.
- `| Out-Null` : masque la sortie de création du dossier.

Alternative manuelle : https://www.elastic.co/downloads/beats/winlogbeat — extraire puis placer le contenu dans `C:\Program Files\Winlogbeat`.

#### Étape 2 : Configuration (`winlogbeat.yml`)

Éditez `C:\Program Files\Winlogbeat\winlogbeat.yml` (Bloc-notes ou VS Code) et configurez au minimum les journaux Application / System / Security et la sortie vers le SIEM :

```yaml
winlogbeat.event_logs:
  - name: Application
    ignore_older: 72h
  - name: System
    ignore_older: 72h
  - name: Security
    ignore_older: 72h

output.elasticsearch:
  hosts: ["http://IP_SIEM:9200"]
  # Sécurité désactivée côté labo (xpack.security.enabled: false) — pas de mot de passe ES
```

`ignore_older: 72h` : ne renvoie pas les événements plus vieux que 72 h (évite un flood au premier démarrage).

Puis validez la syntaxe YAML, la connexion Elasticsearch et la version de l’agent :

```powershell
cd "C:\Program Files\Winlogbeat"
.\winlogbeat.exe test config -c .\winlogbeat.yml
.\winlogbeat.exe test output -c .\winlogbeat.yml
.\winlogbeat.exe version
```

`-c .\winlogbeat.yml` : chemin explicite du fichier de configuration.

#### Étape 3 : Installation et démarrage du service

Enregistrez Winlogbeat comme service Windows (`ExecutionPolicy Bypass` autorise l’exécution du script fourni), puis démarrez-le et vérifiez le statut.

```powershell
cd "C:\Program Files\Winlogbeat"
PowerShell.exe -ExecutionPolicy Bypass -File .\install-service-winlogbeat.ps1
Start-Service winlogbeat
Get-Service winlogbeat
# Attendu : Status = Running, StartType = Automatic
```

Si le script d’installation est absent, créez le service manuellement (chemin binaire + config + `path.home`) :

```powershell
New-Service -Name "winlogbeat" `
  -BinaryPathName "`"C:\Program Files\Winlogbeat\winlogbeat.exe`" -c `"C:\Program Files\Winlogbeat\winlogbeat.yml`" -path.home `"C:\Program Files\Winlogbeat`"" `
  -StartupType Automatic -DisplayName "winlogbeat"
Start-Service winlogbeat
```

#### Étape 4 : Générer un événement Security de test (bénin)

Pour avoir au moins un échec de connexion visible (`4625`) en plus des `4624` naturels, provoquez un échec d’auth local bénin, puis (optionnel) un événement Application de contrôle.

```powershell
# Échec d’auth local bénin (génère souvent un EventID 4625)
net use \\127.0.0.1\IPC$ /user:baduser WrongPass123
# Événement Application de contrôle (optionnel)
eventcreate /T INFORMATION /ID 1000 /L APPLICATION /D "Test SIEM Winlogbeat"
```

- `net use ... /user:baduser` : tentative volontairement invalide → souvent EventID **4625**.
- `eventcreate` : écrit un événement Application facilement filtrable dans Kibana.

#### Étape 5 : Vérification dans Kibana

1. Attendez une minute que les événements partent
2. Data view : `winlogbeat-*` (champ temporel `@timestamp`) — id instructeur : `dv-winlogbeat`
3. Dans **Discover**, filtres utiles :
   - Connexions réussies : `winlog.event_id: 4624`
   - Échecs de connexion : `winlog.event_id: 4625`
   - Canal Security : `winlog.channel: "Security"`
4. Data stream observé en labo : `.ds-winlogbeat-9.5.4-*` (pattern Kibana : `winlogbeat-*`)

**Vérification** : documents Application / System / Security présents ; agent.version aligné sur le SIEM.

#### Filtrage des événements critiques (optionnel)

**Objectif** : ne conserver que certains EventID Security.

**Exemple** :

```yaml
winlogbeat.event_logs:
  - name: Security
    processors:
      - drop_event:
          when:
            not:
              or:
                - equals:
                    winlog.event_id: 4624
                - equals:
                    winlog.event_id: 4625
                - equals:
                    winlog.event_id: 4648
                - equals:
                    winlog.event_id: 4672
                - equals:
                    winlog.event_id: 4719
```

Redémarrez le service pour appliquer le filtre : `Restart-Service winlogbeat`.

---

### 3.3 Installation et configuration de Metricbeat

#### Étape 1 : Installation de Metricbeat

Sur la **VM à surveiller**, installez Metricbeat pour collecter CPU, mémoire, disque, réseau, etc. et les envoyer au SIEM.

```bash
sudo apt install -y metricbeat
```

#### Étape 2 : Configuration de base

Éditez la config pour pointer la sortie vers Elasticsearch sur la VM SIEM.

```bash
sudo nano /etc/metricbeat/metricbeat.yml
```

Configurez la sortie vers le SIEM :

```yaml
output.elasticsearch:
  hosts: ["IP_SIEM:9200"]
```

#### Étape 3 : Activation des modules système

Activez le module `system` (métriques hôte) — sans cela, Metricbeat tourne mais n’envoie presque rien d’utile pour le dashboard labo.

```bash
sudo metricbeat modules enable system
```

#### Étape 4 : Configuration du module système

Le module système collecte par défaut CPU, mémoire, disque, réseau, processus, filesystem. La configuration par défaut suffit en général ; ouvrez le fichier seulement si vous voulez ajuster les périodes ou désactiver un jeu de métriques :

```bash
sudo nano /etc/metricbeat/modules.d/system.yml
```

#### Étape 5 : Démarrage de Metricbeat

Activez et démarrez le service, contrôlez son statut, puis testez la connexion vers Elasticsearch.

```bash
sudo systemctl enable --now metricbeat
sudo systemctl status metricbeat
sudo metricbeat test output
```

Data stream observé : `.ds-metricbeat-9.5.4-*` (pattern Kibana : `metricbeat-*`).

#### Étape 6 : Vérification dans Kibana

1. Créez une data view : `metricbeat-*` (`@timestamp`)
2. Explorez dans **Discover** (`event.dataset: system.cpu`, `system.memory`, …)
3. Ouvrez le dashboard labo (voir §4.3)

---

### 3.4 Packetbeat (optionnel — hors checklist Linux)

> **Optionnel** : Packetbeat **n’est pas** une étape obligatoire de la séance Linux.  
> Filebeat (Suricata / auth / Apache) + Metricbeat suffisent pour le livrable.  
> Ne l’activez que si vous avez de la RAM et du temps ; il nécessite des privilèges élevés et peut être gourmand.

Si vous le tentez malgré tout, installez Packetbeat et éditez sa config (interface + protocoles + sortie SIEM) :

```bash
sudo apt install -y packetbeat
sudo nano /etc/packetbeat/packetbeat.yml
```

```yaml
packetbeat.interfaces.device: any
packetbeat.protocols:
  - type: http
    ports: [80, 8080, 8000, 5000, 8002, 9200]
output.elasticsearch:
  hosts: ["IP_SIEM:9200"]
```

`device: any` : écoute sur toutes les interfaces (gourmand en CPU/RAM).

Démarrez le service pour commencer à envoyer le trafic capturé vers Elasticsearch :

```bash
sudo systemctl enable --now packetbeat
```

Vérification : data view `packetbeat-*` dans Kibana.

---

## 4. Partie 3 : Visualisation et analyse dans Kibana

### 4.1 Création de data views

#### Étape 1 : Accès aux data views

1. Dans Kibana, allez dans **Stack Management → Data Views**
2. Cliquez sur **Create data view**

#### Étape 2 : Création d'une data view pour Filebeat

1. Entrez le pattern : `filebeat-*`
2. Cliquez sur **Next step**
3. Sélectionnez le champ timestamp : `@timestamp`
4. Cliquez sur **Create data view**

#### Étape 3 : Création d'autres data views

Répétez pour :
- `winlogbeat-*`
- `metricbeat-*`
- `packetbeat-*` (si installé)

### 4.2 Création de visualisations de base

#### Visualisation 1 : Graphique temporel des logs

1. Allez dans **Dashboard** > **Create visualization**
2. Choisissez **Line** (graphique linéaire)
3. Sélectionnez la data view `filebeat-*`
4. Configurez :
   - **Y-axis** : Count
   - **X-axis** : Date Histogram sur `@timestamp`
5. Cliquez sur **Save** et donnez un nom : "Logs dans le temps"

#### Visualisation 2 : Top 10 des sources de logs

1. Créez une nouvelle visualisation **Data Table**
2. Sélectionnez `filebeat-*`
3. Configurez :
   - **Metric** : Count
   - **Buckets** : Terms sur `host.name` (ou `source`)
4. Limitez à 10 résultats
5. Sauvegardez : "Top 10 sources de logs"

#### Visualisation 3 : Répartition par type de log

1. Créez une visualisation **Pie Chart**
2. Sélectionnez `filebeat-*`
3. Configurez :
   - **Slice by** : Terms sur `fileset.name` ou `log.file.path`
4. Sauvegardez : "Répartition des types de logs"

### 4.3 Création d'un dashboard

#### Étape 1 : Création du dashboard

1. Allez dans **Dashboard** > **Create dashboard**
2. Cliquez sur **Add** pour ajouter des visualisations
3. Ajoutez les visualisations créées précédemment

#### Étape 2 : Organisation du dashboard

- Organisez les visualisations de manière logique
- Ajustez la taille des panneaux
- Ajoutez un titre : "Dashboard SIEM - Vue d'ensemble"

#### Étape 3 : Sauvegarde

Sauvegardez le dashboard : "Dashboard SIEM Principal"

#### Dashboards labo déjà préparés (référence)

En labo enseignant, les objets suivants sont créés dans Kibana (menu **Analytics → Dashboard**, ou URL relative) :

| Nom | ID / chemin | Sources | Filtres Discover utiles |
| --- | --- | --- | --- |
| **Dashboard SIEM Lab Linux** | `/app/dashboards#/view/dash-siem-lab-linux` | Suricata, SSH/auth, Apache, Metricbeat | panneaux = recherches sauvegardées |
| **Dashboard SIEM — Suricata** | `/app/dashboards#/view/dash-siem-suricata` | Alertes Suricata | `event.kind: "alert"` |
| **Dashboard SIEM — Métriques hôte** | `/app/dashboards#/view/dash-siem-metrics` | Metricbeat (CPU, mémoire, load, réseau, disque, processus, sockets) | `event.dataset: system.cpu` / `system.network`… |
| **Dashboard SIEM Lab Windows** | `/app/dashboards#/view/dash-siem-lab-windows` | Winlogbeat (Security / System / Application) | `winlog.event_id: 4624` / `4625` |

Recherches sauvegardées Linux : `Lab — Alertes Suricata`, `Lab — Connexions SSH / auth`, `Lab — Apache access`, `Lab — Métriques hôte`.  
Recherches sauvegardées Windows : `Lab — Connexions Windows réussies (4624)`, `Lab — Échecs de connexion Windows (4625)`, `Lab — Journal Security Windows`, `Lab — Canaux Windows (Security/System/Application)`.

Le dashboard **Métriques hôte** (`dash-siem-metrics`) affiche des graphes temporels Metricbeat : CPU (`system.cpu.*.norm.pct`), mémoire (`system.memory.actual.used.pct`), load 1/5/15, débit réseau in/out en bytes/s par interface (`system.network.in/out.bytes`, dérivée), utilisation disque par montage (`system.filesystem.used.pct`), top processus et sockets. Import NDJSON : `instructor/kibana/dashboards-metrics.ndjson` (en plus des fichiers Linux/Windows).

Data views : `filebeat-*` (`dv-filebeat`), `metricbeat-*` (`dv-metricbeat`), `winlogbeat-*` (`dv-winlogbeat`).

#### Import des dashboards (méthode qui marche sur Kibana 9.5)

**Ne comptez pas sur** `filebeat setup --dashboards` / `metricbeat setup --dashboards` : sur Kibana **9.5**, `/api/status` ne renvoie plus de champ `version` semver, et Beats échoue avec :

```text
fail to get the Kibana version: fail to parse kibana version (): passed version is not semver
```

**Méthode validée en labo** — import NDJSON (UI ou API) :

1. **UI** : **Stack Management → Saved Objects → Import** → fichiers `instructor/kibana/dashboards-lab-linux.ndjson`, `dashboards-metrics.ndjson` et `dashboards-lab-windows.ndjson` (fournis par l’enseignant, sans secrets) → cocher *Overwrite* si demandé.
2. **API** (depuis une machine qui joint le SIEM) — chaque `curl` envoie un fichier NDJSON à l’API d’import des objets Kibana (`overwrite=true` remplace un objet déjà présent du même id) :

```bash
curl -s -H 'kbn-xsrf: true' \
  -F file=@dashboards-lab-linux.ndjson \
  "http://IP_SIEM:5601/api/saved_objects/_import?overwrite=true"

curl -s -H 'kbn-xsrf: true' \
  -F file=@dashboards-metrics.ndjson \
  "http://IP_SIEM:5601/api/saved_objects/_import?overwrite=true"

curl -s -H 'kbn-xsrf: true' \
  -F file=@dashboards-lab-windows.ndjson \
  "http://IP_SIEM:5601/api/saved_objects/_import?overwrite=true"
```

- `kbn-xsrf: true` : en-tête anti-CSRF exigé par l’API Kibana.
- `-F file=@...` : envoie le fichier NDJSON en multipart (le `@` = chemin local du fichier).
- Placez-vous dans le dossier où se trouvent les `.ndjson` fournis par l’enseignant (ou indiquez le chemin complet après `@`).

Réponse attendue : `"success": true` et import des data views, recherches et dashboards listés ci-dessus.

Alternative : créer à la main les data views + visualisations (§4.1–4.3 / §4.5) sans importer le NDJSON.

### 4.4 Recherche et analyse de logs

#### Recherche simple

1. Allez dans **Discover**
2. Sélectionnez la data view `filebeat-*`
3. Utilisez la barre de recherche pour filtrer :
   - `status:error` (logs d'erreur)
   - `message:*failed*` (messages contenant "failed")
   - `@timestamp:[now-1h TO now]` (dernière heure)

#### Recherche avancée avec KQL

Utilisez la syntaxe KQL (Kibana Query Language) :

```
host.name: "nom-serveur" and message: "error"
```

```
@timestamp >= "now-24h" and log.level: "error"
```

#### Analyse de corrélation

1. Recherchez des patterns suspects :
   - Plusieurs échecs de connexion : `event.action: "authentication_failure"`
   - Tentatives d'accès à des fichiers sensibles
   - Activité réseau suspecte

2. Utilisez les filtres pour affiner votre recherche

### 4.5 Dashboard de sécurité avec alertes

**Tâche (partie Linux — faite en labo)** :
1. Utilisez **Dashboard SIEM Lab Linux** / **Dashboard SIEM — Suricata**, ou recréez un dashboard « Dashboard Sécurité Linux » avec :
   - Alertes Suricata dans le temps (`event.kind: "alert"`, breakdown `rule.name`)
   - Top IP sources (`source.ip`)
   - Événements `system.auth` (SSH)
   - Accès Apache (`http.response.status_code`)
   - Métriques Metricbeat (CPU / mémoire)
2. Sauvegardez le dashboard.

**Tâche (partie Windows — Winlogbeat §3.2)** :
1. Ouvrez **Dashboard SIEM Lab Windows** (`/app/dashboards#/view/dash-siem-lab-windows`), ou créez un dashboard équivalent avec :
   - Connexions réussies (`winlog.event_id: 4624`)
   - Échecs d’authentification (`winlog.event_id: 4625`)
   - Journal Security / répartition des canaux (`winlog.channel`)
   - Hôte source (`host.name`)
2. Vérifiez que les panneaux affichent des documents (data view `winlogbeat-*`).
3. Alertes Kibana optionnelles (selon licence / version) : seuil d’échecs de connexion, CPU élevé.

**Indice** : champs Windows `winlog.event_id`, `winlog.channel` ; champs Suricata ECS `rule.name`, `event.kind`.

---

## 5. Partie 4 : Cas d'usage pratique

### 5.1 Scénario : Détection d'une tentative d'intrusion

#### Contexte

Vous êtes administrateur système et vous devez analyser les logs pour détecter une activité suspecte. Un utilisateur signale des connexions étranges à un serveur.

#### Données disponibles

Vous avez accès aux logs suivants :
- Logs système (Filebeat)
- Événements Windows (Winlogbeat)
- Métriques système (Metricbeat)

#### Mission

Identifiez les signes d'une tentative d'intrusion en analysant les logs.

### 5.2 Analyse de logs corrélés

#### Étape 1 : Recherche d'échecs d'authentification

1. Dans Kibana, allez dans **Discover**
2. Sélectionnez la data view `winlogbeat-*`
3. Recherchez les échecs de connexion :

```
winlog.event_id: 4625
```

4. Analysez :
   - Nombre d'échecs
   - IP sources
   - Comptes ciblés
   - Période d'activité

#### Étape 2 : Recherche d'activité réseau suspecte

1. Si Packetbeat est installé, analysez le trafic réseau
2. Recherchez :
   - Connexions depuis des IP suspectes
   - Ports non standard
   - Volumes de trafic anormaux

#### Étape 3 : Analyse des logs système

1. Dans `filebeat-*`, recherchez :
   - Tentatives d'accès à des fichiers sensibles
   - Modifications de configuration
   - Exécution de commandes suspectes

#### Étape 4 : Corrélation temporelle

1. Utilisez le filtre temporel pour identifier les périodes d'activité suspecte
2. Corrélez les événements entre différentes sources
3. Identifiez les patterns d'attaque

### 5.3 Création de règles de corrélation simples

#### Règle 1 : Détection de brute force

**Logique** : Plus de 5 échecs de connexion depuis la même IP en 10 minutes

**Implémentation dans Kibana** :
1. Créez une visualisation **Data Table**
2. Configurez :
   - **Metric** : Count
   - **Buckets** : Terms sur `source.ip` (ou `winlog.event_data.IpAddress`)
   - **Filter** : `winlog.event_id: 4625`
3. Ajoutez un filtre temporel : `@timestamp:[now-10m TO now]`
4. Identifiez les IP avec plus de 5 occurrences

#### Règle 2 : Détection d'activité hors heures

**Logique** : Connexions réussies en dehors des heures de bureau (ex: 22h-6h)

**Implémentation** :
1. Recherchez les connexions réussies : `winlog.event_id: 4624`
2. Filtrez par heure : `@timestamp:[now-24h TO now]`
3. Analysez les heures de connexion

### 5.4 Identification d'un pattern suspect

**Scénario** : les événements du TP sont votre matière. Reliez-les pour raconter ce qui s'est passé, au lieu d'enchaîner des écrans sans lien.

**Tâche** : partez des événements déjà produits (alerte Suricata `ESAIP-LAB-HTTP-TEST`, lignes `system.auth`, échec Windows `4625`).

1. Dans Kibana, créez des recherches pour identifier :
   - Les IP sources
   - Les comptes ciblés
   - Ce qui a déclenché l'événement (règle Suricata, échec d'authentification, etc.)
   - La chronologie
2. Rédigez un court rapport :
   - Résumé
   - Chronologie
   - Ce que vous feriez ensuite (bloquer, investiguer, ignorer un test de labo)
3. Présentez ces recherches dans un dashboard dédié

**Indice** : Utilisez les fonctionnalités de sauvegarde de recherche dans Kibana pour documenter vos analyses.

---

## 6. Annexes

### 6.1 Commandes utiles

Mémo rapide pour le dépannage en séance. Chaque bloc regroupe les gestes les plus fréquents (statut, redémarrage, logs, test).

#### Elasticsearch

Statut / redémarrage du service, suivi des logs (`-f` = follow), liste des index, suppression d’un index (irréversible) :

```bash
# Vérifier le statut
sudo systemctl status elasticsearch

# Redémarrer
sudo systemctl restart elasticsearch

# Voir les logs
sudo journalctl -u elasticsearch -f

# Lister les index
curl http://localhost:9200/_cat/indices?v

# Supprimer un index (attention !)
curl -X DELETE http://localhost:9200/nom-index
```

- `_cat/indices?v` : vue tabulaire des index (`v` = en-têtes de colonnes).
- `-X DELETE` : méthode HTTP DELETE — remplacez `nom-index` ; ne supprimez pas les index système (`.kibana*`, etc.) sans raison.

#### Kibana

Statut, redémarrage et logs du service Kibana :

```bash
# Vérifier le statut
sudo systemctl status kibana

# Redémarrer
sudo systemctl restart kibana

# Voir les logs
sudo journalctl -u kibana -f
```

#### Logstash

Statut, redémarrage, test de config hors service (`-f` = fichier de pipeline ; `--config.test_and_exit` = valide puis quitte sans démarrer le pipeline), logs :

```bash
# Vérifier le statut
sudo systemctl status logstash

# Redémarrer
sudo systemctl restart logstash

# Tester une configuration
sudo /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/test.conf --config.test_and_exit

# Voir les logs
sudo journalctl -u logstash -f
```

#### Filebeat

Statut, validation de la config YAML, liste des modules, logs :

```bash
# Vérifier le statut
sudo systemctl status filebeat

# Tester la configuration
sudo filebeat test config

# Lister les modules
sudo filebeat modules list

# Voir les logs
sudo journalctl -u filebeat -f
```

#### Metricbeat

Statut, validation de config, liste des modules :

```bash
# Vérifier le statut
sudo systemctl status metricbeat

# Tester la configuration
sudo metricbeat test config

# Lister les modules
sudo metricbeat modules list
```

#### Winlogbeat (Windows PowerShell)

Statut du service, redémarrage, test de la config depuis le répertoire d’installation :

```powershell
# Vérifier le statut
Get-Service winlogbeat

# Redémarrer
Restart-Service winlogbeat

# Tester la configuration
cd "C:\Program Files\Winlogbeat"
.\winlogbeat.exe test config -c .\winlogbeat.yml
```

### 6.2 Dépannage

#### Problème : Elasticsearch ne démarre pas

**Symptômes** : Service en état `failed` ou erreur de mémoire

**Solutions** :
1. Vérifiez les logs : `sudo journalctl -u elasticsearch -n 50`
2. Vérifiez la mémoire disponible : `free -h`
3. Réduisez le heap dans `/etc/elasticsearch/jvm.options`
4. Vérifiez les permissions : `sudo chown -R elasticsearch:elasticsearch /var/lib/elasticsearch`

#### Problème : Kibana ne peut pas se connecter à Elasticsearch

**Symptômes** : Erreur "Unable to connect to Elasticsearch"

**Solutions** :
1. Vérifiez qu'Elasticsearch fonctionne : `curl http://localhost:9200`
2. Vérifiez la configuration dans `kibana.yml` — la ligne `elasticsearch.hosts` doit être décommentée :

   ```bash
   sudo grep "^elasticsearch.hosts" /etc/kibana/kibana.yml
   ```

   La ligne doit être décommentée (sans `#` au début)
3. Vérifiez les logs : `sudo journalctl -u kibana -n 50`
4. Vérifiez le pare-feu

#### Problème : Page Kibana reste en chargement (loading)

**Symptômes** : La page Kibana s'affiche mais reste bloquée sur "Loading..." ou "Kibana is starting", même après plusieurs minutes.

**Causes possibles** :
1. Les migrations d'objets sauvegardés sont en cours (normal au premier démarrage)
2. Kibana attend que les migrations se terminent avant d'afficher l'interface
3. Problème de performance (manque de mémoire)

**Solutions** :

1. **Vérifiez l'état des migrations dans les logs** — filtrez le journal Kibana sur les messages de migration :

   ```bash
   sudo journalctl -u kibana -f | grep -i "migration\|savedobjects"
   ```

   `| grep -i "..."` : ne garde que les lignes contenant ces mots (insensible à la casse). Vous devriez voir des messages comme "Starting saved objects migrations" puis "Migration completed" ou "Migration successful".

2. **Attendez 3-5 minutes** lors du premier démarrage. Les migrations peuvent prendre du temps, surtout avec peu de RAM.

3. **Vérifiez que Kibana est bien connecté à Elasticsearch** :

   ```bash
   sudo journalctl -u kibana | grep -i "connected to elasticsearch"
   ```

   Vous devriez voir "Successfully connected to Elasticsearch".

4. **Vérifiez l'état du service Kibana** :

   ```bash
   sudo systemctl status kibana
   ```

   Le service doit être "active (running)".

5. **Vérifiez les logs complets pour des erreurs** — les 200 dernières lignes, sans pager :

   ```bash
   sudo journalctl -u kibana -n 200 --no-pager
   ```

   Cherchez les erreurs (ERROR, FATAL) ou les warnings importants.

6. **Si les migrations semblent bloquées** (plus de 10 minutes), vous pouvez essayer de les réinitialiser (⚠️ **ATTENTION** : cela supprimera toutes les configurations Kibana) :

   ```bash
   # Arrêtez Kibana
   sudo systemctl stop kibana
   
   # Supprimez les index Kibana (perte de toutes les configurations, visualisations, dashboards)
   curl -X DELETE "http://localhost:9200/.kibana*"
   
   # Redémarrez Kibana
   sudo systemctl start kibana
   
   # Attendez 3-5 minutes pour les nouvelles migrations
   ```

   `.kibana*` : index internes de Kibana (objets sauvegardés). Ne faites ceci qu’en dernier recours en labo.

7. **Vérifiez la mémoire disponible** :

   ```bash
   free -h
   ```

   Si la mémoire est saturée, les migrations peuvent être très lentes. Considérez réduire le heap d'Elasticsearch.

**Note** : Si vous voyez dans les logs "Starting saved objects migrations" mais pas de message de fin, c'est normal - les migrations peuvent prendre plusieurs minutes. Patientez et surveillez les logs.

#### Problème : Les logs n'apparaissent pas dans Kibana

**Symptômes** : Aucune donnée dans Discover

**Solutions** :
1. Vérifiez que les Beats fonctionnent : `sudo systemctl status filebeat`
2. Vérifiez les index dans Elasticsearch : `curl http://localhost:9200/_cat/indices?v`
3. Vérifiez la configuration des Beats
4. Vérifiez les logs des Beats : `sudo journalctl -u filebeat -f`

#### Problème : Winlogbeat ne collecte pas d'événements

**Symptômes** : Aucun événement Windows dans Kibana

**Solutions** :
1. Vérifiez le service : `Get-Service winlogbeat`
2. Vérifiez les logs : `Get-EventLog -LogName Application -Source winlogbeat`
3. Vérifiez la configuration : `.\winlogbeat.exe test config`
4. Vérifiez la connectivité vers Elasticsearch : `Test-NetConnection -ComputerName IP_SIEM -Port 9200`

#### Problème : Performance lente avec 2 Go de RAM

**Symptômes** : Système lent, services qui plantent

**Solutions** :
1. Réduisez le heap d'Elasticsearch (par exemple 256 Mo si Kibana et Logstash tournent en même temps)
2. Limitez le nombre d'indices
3. Désactivez les fonctionnalités non essentielles (ML, etc.)
4. Utilisez un swap si nécessaire (mais cela ralentit)
5. Vérifiez la consommation mémoire des autres services (Kibana, Logstash)

### 6.3 Ressources supplémentaires

#### Documentation officielle

- **Elastic Stack** : https://www.elastic.co/guide/
- **Elasticsearch** : https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html
- **Logstash** : https://www.elastic.co/guide/en/logstash/current/index.html
- **Kibana** : https://www.elastic.co/guide/en/kibana/current/index.html
- **Beats** : https://www.elastic.co/guide/en/beats/index.html

#### Patterns Grok

- **Grok Debugger** : https://grokdebug.herokuapp.com/
- **Patterns de base** : https://github.com/elastic/logstash/blob/v1.4.2/patterns/grok-patterns

#### Communautés

- **Forum Elastic** : https://discuss.elastic.co/
- **Stack Overflow** : Tag `elasticsearch`, `logstash`, `kibana`

#### Outils utiles

- **Elasticsearch Head** : Plugin pour visualiser Elasticsearch
- **Cerebro** : Interface web pour gérer Elasticsearch
- **Elasticsearch Curator** : Outil pour gérer les index

### 6.5 Checklist de fin de TP

Avant de terminer, vérifiez que vous avez :

- [ ] Elasticsearch installé et fonctionnel sur la VM SIEM
- [ ] Kibana accessible depuis votre navigateur (`http://IP_SIEM:5601`)
- [ ] Logstash configuré avec au moins un pipeline (test ou beats)
- [ ] Filebeat sur la VM à surveiller : modules `system`, `apache`, `suricata`
- [ ] Alertes Suricata visibles (`event.kind: "alert"`)
- [ ] Traces SSH / auth visibles (`event.dataset: "system.auth"`)
- [ ] Logs Apache visibles (`event.dataset: "apache.access"`)
- [ ] Metricbeat collectant les métriques hôte (`metricbeat-*`)
- [ ] Au moins le **Dashboard SIEM Lab Linux** (ou 3 visualisations + un dashboard équivalent)
- [ ] Winlogbeat installé (même branche que le SIEM), service **Running**, data view `winlogbeat-*`
- [ ] Au moins le **Dashboard SIEM Lab Windows** (ou équivalent 4624 / 4625 / canaux)
- [ ] Configurations et découvertes documentées (y compris l’IP source Windows → SIEM:9200)

---

## Conclusion

Félicitations ! Vous avez maintenant une solution SIEM complète opérationnelle. Vous pouvez :

- Collecter des logs depuis différentes sources
- Stocker et indexer les données dans Elasticsearch
- Visualiser et analyser les données dans Kibana
- Détecter des activités suspectes

**Bon travail !**

---

*TP créé pour l'UE S10-3 - Sécurité des informations et management des événements*  
*École ESAIP - Ingénieur Informatique et Réseaux*
