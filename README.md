# TP : Mise en place d'une solution SIEM avec ELK Stack

**UE S10-3 - Sécurité des informations et management des événements**  
**Travaux Pratiques - Travail individuel**

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
- Installer et configurer la stack ELK sur une machine Debian
- Configurer des agents de collecte (Beats) sur différents systèmes
- Créer des visualisations et des dashboards dans Kibana
- Analyser des événements de sécurité réels

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

#### Machine serveur ELK
- **OS** : Debian 13 (Bookworm)
- **CPU** : 1 cœur
- **RAM** : 2 Go
- **Réseau** : Accessible depuis le réseau ESAIP
- **Accès** : SSH depuis votre PC

#### Machine cliente (votre PC)
- **OS** : Windows
- **Réseau** : Même réseau que la VM (ESAIP)
- **Accès** : Navigateur web pour Kibana

#### Architecture réseau

```
┌─────────────────┐         ┌──────────────────┐
│   PC Windows    │         │  VM Debian 13    │
│                 │         │                  │
│  - Winlogbeat   │───────▶│  - Elasticsearch │
│  - Navigateur   │         │  - Logstash      │
│                 │         │  - Kibana        │
│                 │         │  - Filebeat      │
│                 │         │  - Metricbeat    │
└─────────────────┘         └──────────────────┘
         │                            │
         └─────────── Réseau ESAIP ───┘
```

### 1.4 Prérequis

Avant de commencer, assurez-vous d'avoir :

- Accès SSH à votre VM Debian
- Droits administrateur (sudo) sur la VM
- Accès réseau entre votre PC et la VM
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

#### Étape 1 : Connexion à la VM

Connectez-vous à votre VM Debian via SSH :

```bash
ssh esaip@IP_VM
```

Remplacez `IP_VM` par votre IP indiquée sur la page [proxmox](https://10.3.2.10:8006).

#### Étape 2 : Vérification du système

Vérifiez les informations de votre système :

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

**Vérification** : Notez la quantité de RAM disponible. Avec 2 Go de RAM, nous devrons optimiser Elasticsearch pour laisser de la mémoire aux autres services.

#### Étape 3 : Mise à jour du système

Mettez à jour les paquets système :

```bash
sudo apt update
sudo apt upgrade -y
```

#### Étape 4 : Installation des outils de base

Installez les outils nécessaires :

```bash
sudo apt install -y curl wget nano git gpg
```

---

### 2.2 Installation d'Elasticsearch
En cas de problème avec l'installation veuillez vérifier si la procédure n'a pas changée sur [elastic doc](https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-elasticsearch-with-debian-package)

#### Étape 1 : Ajout de la clé GPG Elastic

Ajoutez la clé GPG pour authentifier les paquets Elastic :

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch
sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
```

#### Étape 2 : Installation d'apt-transport-https

Installez le support HTTPS pour apt :

```bash
sudo apt install -y apt-transport-https
```

#### Étape 3 : Ajout du dépôt Elastic

Ajoutez le dépôt Elastic (nous utiliserons la version 9.x) :

```bash
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/9.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-9.x.list
```

#### Étape 4 : Mise à jour et installation

Mettez à jour les dépôts et installez Elasticsearch :

```bash
sudo apt-get update && sudo apt-get install elasticsearch
```

#### Étape 5 : Configuration d'Elasticsearch pour 2 Go de RAM

**IMPORTANT** : Avec 2 Go de RAM, nous devons limiter la mémoire heap d'Elasticsearch pour laisser de la mémoire aux autres services.

Créez le fichier de configuration de la mémoire heap pour la JVM :

```bash
sudo nano /etc/elasticsearch/jvm.options.d/heapsizemem.options
```

**Note** : L'extension du fichier doit être `.options` (pas `.conf`) pour être prise en compte par Elasticsearch.

Insérez les lignes suivantes :

```properties
# Optimisé pour 2 Go RAM (heap limité à 512 Mo)
-Xms512m
-Xmx512m
```

**Explication** : 
- Nous limitons le **heap** à 1 Go pour laisser suffisamment de mémoire au système et aux autres services (Kibana, Logstash, etc.).
- Avec 2 Go de RAM total, un heap de 512m est un bon compromis.
- **Important** : La mémoire totale utilisée par Elasticsearch sera supérieure au heap (environ 1Go) car elle inclut :
  - Le heap JVM ( 512m dans notre cas)
  - La mémoire native (off-heap)
  - Les bibliothèques partagées
  - Les processus auxiliaires (comme le controller ML)
  - Les buffers système

#### Étape 6 : Configuration réseau d'Elasticsearch

Éditez le fichier de configuration principal :

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

**Configuration pour Elasticsearch 9.2** :

La configuration par défaut d'Elasticsearch 9.2 inclut déjà certaines options. Vous devez vérifier et modifier les paramètres suivants :

```yaml
# Réseau et HTTP (déjà présent dans la config par défaut avec http.host)
# Vérifiez que ces lignes sont présentes :
http.host: 0.0.0.0
http.port: 9200

# Nom du cluster (optionnel, pour identification)
cluster.name: siem-cluster

node.name: siem-node-1

# Désactiver SSL/HTTPS pour simplifier (nécessaire car la sécurité est désactivée)(**à ne pas faire en production**)
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
- Dans Elasticsearch 9.2, utilisez `http.host` au lieu de `network.host`.
- Remplacez `"siem-node-1"` dans `cluster.initial_master_nodes` par la valeur de `node.name` que vous avez définie (ou gardez le nom de votre machine si vous n'avez pas changé `node.name`).

#### Étape 7 : Démarrage d'Elasticsearch

Démarrez le service Elasticsearch :

```bash
sudo systemctl start elasticsearch
```

Activez le démarrage automatique :

```bash
sudo systemctl enable elasticsearch
```

Vérifiez le statut :

```bash
sudo systemctl status elasticsearch
```

#### Étape 8 : Vérification du fonctionnement

Attendez 30-60 secondes que Elasticsearch démarre, puis testez :

```bash
curl http://localhost:9200
```

Vous devriez voir une réponse JSON avec les informations du cluster.

**Vérification** : Après redémarrage, vérifiez que le heap est bien limité avec :
```bash
curl http://localhost:9200/_nodes/jvm?pretty
```
Cherchez les valeurs `heap_used` et `heap_max` dans la réponse.

Testez également depuis votre PC Windows (remplacez `IP_VM` par l'IP de votre VM) :

```bash
# Depuis PowerShell ou CMD
curl http://IP_VM:9200
```

**Vérification** : Si vous obtenez une réponse JSON, Elasticsearch fonctionne correctement.

#### Aide : Résolution d'un problème de mémoire

**Situation** : Elasticsearch ne démarre pas ou plante avec une erreur de mémoire.

**Procédure** :
1. Vérifiez les logs : `sudo journalctl -u elasticsearch -n 50`
2. Identifiez l'erreur liée à la mémoire
3. Ajustez les paramètres `-Xms` et `-Xmx` dans `/etc/elasticsearch/jvm.options`
4. Redémarrez Elasticsearch : `sudo systemctl restart elasticsearch`

**Indice** : Avec 2 Go de RAM, un heap de 1 Go devrait fonctionner. Si vous avez des problèmes, vous pouvez réduire à 512m, mais cela peut impacter les performances.

---

### 2.3 Installation de Kibana

#### Étape 1 : Installation

Kibana devrait déjà être disponible dans le dépôt Elastic. Installez-le :

```bash
sudo apt install -y kibana
```

#### Étape 2 : Configuration de Kibana

Éditez le fichier de configuration :

```bash
sudo nano /etc/kibana/kibana.yml
```

Modifiez les paramètres suivants :

```yaml
# Adresse d'écoute
server.host: "0.0.0.0"

# Port
server.port: 5601

# URL d'Elasticsearch
elasticsearch.hosts: ["http://localhost:9200"]
node.options: "--max-old-space-size=128"
```

#### Étape 3 : Démarrage de Kibana

Démarrez le service :

```bash
sudo systemctl start kibana
sudo systemctl enable kibana
```

Vérifiez le statut :

```bash
sudo systemctl status kibana
```

**Note importante** : 
- Kibana peut prendre **1-2 minutes** pour démarrer complètement.
- **Lors du premier démarrage**, Kibana effectue des migrations d'objets sauvegardés qui peuvent prendre **3-5 minutes** supplémentaires.
- Pendant ce temps, la page web peut rester en chargement. C'est **normal**.
- Surveillez les logs pour voir l'état des migrations :
  ```bash
  sudo journalctl -u kibana -f
  ```
  Vous devriez voir "Starting saved objects migrations" puis "Migration completed" ou similaire.

#### Étape 4 : Accès à Kibana

Ouvrez votre navigateur web et accédez à :

```
http://IP_VM:5601
```

Remplacez `IP_VM` par l'adresse IP de votre VM.

**Vérification** : Vous devriez voir la page d'accueil de Kibana.

#### Aide : Configuration de l'accès depuis Windows

**Situation** : Vous ne pouvez pas accéder à Kibana depuis votre PC Windows.

**Procédure** :
1. Vérifiez que Kibana écoute sur toutes les interfaces : `sudo netstat -tlnp | grep 5601`
2. Vérifiez le pare-feu de la VM : `sudo ufw status`
3. Si le pare-feu bloque, autorisez le port : `sudo ufw allow 5601/tcp`
4. Vérifiez la connectivité depuis Windows : `ping IP_VM`
5. Testez depuis Windows : `curl http://IP_VM:5601` ou ouvrez dans le navigateur

**Indice** : Si vous utilisez un pare-feu, vous devrez peut-être aussi ouvrir le port 9200 pour Elasticsearch.

---

### 2.4 Installation de Logstash

#### Étape 1 : Installation de Java

Logstash nécessite Java. Installez OpenJDK :

```bash
sudo apt install -y default-jre
```

Vérifiez l'installation :

```bash
java -version
```
---

#### Étape 2 : Installation de Logstash

Installez Logstash :

```bash
sudo apt update
sudo apt install -y logstash
```

---

#### Étape 3 : Configuration de base de Logstash

Éditez le fichier de configuration principal :

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

Créez et corrigez les permissions nécessaires (évite 90 % des erreurs) :

```bash
sudo mkdir -p /var/log/logstash /var/lib/logstash
sudo chown -R logstash:logstash /var/log/logstash /var/lib/logstash
```
**IMPORTANT** : Avec 4 Go de RAM, nous devons limiter la mémoire heap de logstash pour laisser de la mémoire aux autres services.

Modifiez le fichier de configuration de la mémoire heap pour la JVM :

```bash
sudo nano /etc/logstash/jvm.options
```

Modifiez les lignes suivantes :

```properties
# Optimisé pour 4 Go RAM (heap limité à 256 Mo)
-Xms256m
-Xmx256m
```
---

#### Étape 4 : Création d’un pipeline de test (mode service compatible)

**Important** :
Le pipeline `stdin {}` **ne doit pas être utilisé avec systemd**, car il provoque l’arrêt immédiat de Logstash.
On utilise donc un **input file** pour un vrai test.

Créez le pipeline de test :

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

Créez le fichier de test :

```bash
sudo touch /tmp/logstash-test.log
sudo chmod 666 /tmp/logstash-test.log
```

---

#### Étape 5 : Démarrage de Logstash

Démarrez Logstash comme service :

```bash
sudo systemctl enable logstash
sudo systemctl restart logstash
```

Vérifiez le statut :

```bash
sudo systemctl status logstash
```

Vous devez voir :

```
Active: active (running)
```

---

#### Étape 6 : Test du pipeline

Dans un autre terminal, écrivez dans le fichier de test :

```bash
echo "Bonjour Logstash" >> /tmp/logstash-test.log
echo "Deuxième message" >> /tmp/logstash-test.log
```

Résultat attendu :

* Les logs apparaissent dans Elasticsearch (Allez dans **elasticsearch  →  index_management**
* L’index `test-logs-YYYY.MM.dd` est créé
* Logstash **reste actif**

---

#### Pipeline pour les logs Apache

**Objectif**

Créer un pipeline Logstash qui lit et parse les logs Apache.

---

##### Étape 1 : Installation d’Apache

```bash
sudo apt install -y apache2
```

Vérifiez qu’Apache fonctionne :

```bash
curl http://localhost
```

---

##### Étape 2 : Création du pipeline Apache

Créez le fichier :

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
Supprimez ou commentez `test.conf` pour éviter plusieurs pipelines non nécessaires :

```bash
sudo mv /etc/logstash/conf.d/test.conf /etc/logstash/conf.d/test.conf.disabled
```

Autoriser logstash à lire le fichier de log Apache :
```bash
sudo usermod -aG adm logstash
```

---

##### Étape 3 : Redémarrage de Logstash

```bash
sudo systemctl restart logstash
```

Vérification :

```bash
journalctl -u logstash -n 20 --no-pager
```

Vous devez voir :

* `Pipeline started`
* aucune erreur fatale

---

##### Étape 4 : Génération de trafic Apache

```bash
curl http://localhost
curl http://localhost
curl http://localhost
```

---

##### Étape 5 : Vérification dans Kibana

1. Aller dans **Stack Management → Data Views**
2. Créer un data view :

   ```
   apache-logs-*
   ```
3. Champ temporel : `@timestamp`
4. Aller dans **Discover**

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

## 3. Partie 2 : Configuration des agents Beats

### 3.1 Installation et configuration de Filebeat

#### Étape 1 : Installation de Filebeat et rsyslog

Depuis debian 8 journald est utilisé pour logger les évènements système ce qui rend la capture par filebeat ou logstash impossible.
Il faut donc réactiver rsyslog comme la configuration par défaut de ubuntu.

Installez et activez rsyslog

```bash
apt install rsyslog
systemctl enable --now rsyslog
```

Installez Filebeat sur la VM Debian :

```bash
sudo apt install -y filebeat
```

#### Étape 2 : Configuration de base

Éditez le fichier de configuration :

```bash
sudo nano /etc/filebeat/filebeat.yml
```

Configurez la section `output.elasticsearch` :

```yaml
output.elasticsearch:
  hosts: ["localhost:9200"]
  # Désactiver la sécurité si Elasticsearch n'a pas de sécurité
  # username: "elastic"
  # password: "changeme"
```

#### Étape 3 : Activation du module système

Activez le module système pour collecter les logs système :

```bash
sudo filebeat modules enable system
```

#### Étape 4 : Configuration du module système

Éditez la configuration du module :

```bash
sudo nano /etc/filebeat/modules.d/system.yml
```

La configuration par défaut devrait collecter :
- `/var/log/auth.log` (authentifications)
- `/var/log/syslog` (logs système)
- `/var/log/*.log` (autres logs)

#### Étape 5 : Démarrage de Filebeat

Démarrez Filebeat :

```bash
sudo systemctl start filebeat
sudo systemctl enable filebeat
```

Vérifiez le statut :

```bash
sudo systemctl status filebeat
```

#### Étape 6 : Vérification dans Kibana

1. Accédez à Kibana : `http://IP_VM:5601`
2. Allez dans **Stack Management → Data Views**
3. Créez une data view : `filebeat-*`
4. Sélectionnez le champ timestamp : `@timestamp`
5. Explorez les données dans **Discover**

**Vérification** : Vous devriez voir les logs système dans Kibana.

#### Configuration pour logs Apache

**Objectif** : Configurer Filebeat pour collecter spécifiquement les logs Apache.

**Tâche** :
1. Activez le module Apache : `sudo filebeat modules enable apache`
2. Éditez la configuration : `sudo nano /etc/filebeat/modules.d/apache.yml`
3. Configurez les chemins des logs Apache (généralement `/var/log/apache2/access.log` et `/var/log/apache2/error.log`)
4. Redémarrez Filebeat : `sudo systemctl restart filebeat`
5. Générez du trafic Apache : `curl http://localhost` (plusieurs fois)
6. Vérifiez dans Kibana que les logs Apache apparaissent avec l'index `filebeat-*`

**Indice** : Vous pouvez aussi créer une configuration personnalisée dans `filebeat.yml` avec une section `filebeat.inputs`.

---

### 3.2 Installation et configuration de Winlogbeat

#### Étape 1 : Téléchargement de Winlogbeat

Sur votre PC Windows, téléchargez Winlogbeat depuis le site Elastic :

1. Allez sur : https://www.elastic.co/downloads/beats/winlogbeat
2. Téléchargez la version Windows (ZIP)
3. Extrayez l'archive dans un dossier (ex: `C:\Program Files\Winlogbeat`)

**Alternative** : Utilisez PowerShell pour télécharger :

```powershell
# Dans PowerShell (en tant qu'administrateur)
Invoke-WebRequest -Uri "https://artifacts.elastic.co/downloads/beats/winlogbeat/winlogbeat-8.11.0-windows-x86_64.zip" -OutFile "winlogbeat.zip"
Expand-Archive winlogbeat.zip -DestinationPath "C:\Program Files\"
```

#### Étape 2 : Configuration de Winlogbeat

Éditez le fichier de configuration :

```powershell
notepad "C:\Program Files\Winlogbeat\winlogbeat.yml"
```

Configurez la section `output.elasticsearch` :

```yaml
output.elasticsearch:
  hosts: ["http://IP_VM:9200"]
  # Remplacez IP_VM par l'adresse IP de votre VM
  # Désactiver la sécurité si Elasticsearch n'a pas de sécurité
  # username: "elastic"
  # password: "changeme"
```

#### Étape 3 : Configuration de la collecte d'événements

Dans le même fichier, configurez les événements Windows à collecter :

```yaml
winlogbeat.event_logs:
  - name: Application
    ignore_older: 72h
  - name: System
    ignore_older: 72h
  - name: Security
    ignore_older: 72h
```

#### Étape 4 : Installation de Winlogbeat comme service

Installez Winlogbeat comme service Windows :

```powershell
# Dans PowerShell (en tant qu'administrateur)
cd "C:\Program Files\Winlogbeat"
.\install-service-winlogbeat.ps1
```

Si le script n'existe pas, utilisez cette commande :

```powershell
New-Service -Name "winlogbeat" -BinaryPathName "C:\Program Files\Winlogbeat\winlogbeat.exe -c C:\Program Files\Winlogbeat\winlogbeat.yml -path.home C:\Program Files\Winlogbeat" -StartupType Automatic
```

#### Étape 5 : Démarrage du service

Démarrez le service Winlogbeat :

```powershell
Start-Service winlogbeat
```

Vérifiez le statut :

```powershell
Get-Service winlogbeat
```

#### Étape 6 : Vérification dans Kibana

1. Attendez quelques minutes pour que les événements soient collectés
2. Dans Kibana, créez une data view : `winlogbeat-*`
3. Explorez les événements Windows dans **Discover**

**Vérification** : Vous devriez voir les événements Windows (Application, System, Security) dans Kibana.

#### Filtrage des événements critiques

**Objectif** : Configurer Winlogbeat pour collecter uniquement les événements de sécurité critiques.

**Tâche** :
1. Éditez `winlogbeat.yml`
2. Dans la section `winlogbeat.event_logs`, configurez des filtres pour le journal Security
3. Filtrez par EventID pour ne collecter que les événements critiques :
   - 4624 : Connexion réussie
   - 4625 : Échec de connexion
   - 4648 : Connexion avec identifiants explicites
   - 4672 : Privilèges spéciaux assignés
   - 4719 : Modification de la politique d'audit système
4. Redémarrez le service : `Restart-Service winlogbeat`
5. Vérifiez dans Kibana que seuls les événements filtrés apparaissent

**Indice** : Utilisez la section `processors` pour filtrer les événements par EventID.

**Exemple de configuration** :

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

---

### 3.3 Installation et configuration de Metricbeat

#### Étape 1 : Installation de Metricbeat

Installez Metricbeat sur la VM Debian :

```bash
sudo apt install -y metricbeat
```

#### Étape 2 : Configuration de base

Éditez le fichier de configuration :

```bash
sudo nano /etc/metricbeat/metricbeat.yml
```

Configurez la section `output.elasticsearch` :

```yaml
output.elasticsearch:
  hosts: ["localhost:9200"]
```

#### Étape 3 : Activation des modules système

Activez le module système pour collecter les métriques système :

```bash
sudo metricbeat modules enable system
```

#### Étape 4 : Configuration du module système

Le module système collecte par défaut :
- CPU
- Mémoire
- Disque
- Réseau
- Processus
- Système de fichiers

La configuration par défaut devrait suffire. Vérifiez si nécessaire :

```bash
sudo nano /etc/metricbeat/modules.d/system.yml
```

#### Étape 5 : Démarrage de Metricbeat

Démarrez Metricbeat :

```bash
sudo systemctl start metricbeat
sudo systemctl enable metricbeat
```

Vérifiez le statut :

```bash
sudo systemctl status metricbeat
```

#### Étape 6 : Vérification dans Kibana

1. Attendez quelques minutes
2. Dans Kibana, créez une data view : `metricbeat-*`
3. Explorez les métriques dans **Discover**
4. Allez dans **Stack Monitoring** pour voir les dashboards prédéfinis

**Vérification** : Vous devriez voir les métriques système (CPU, mémoire, disque, réseau) dans Kibana.

---

### 3.4 Installation et configuration de Packetbeat

**Note** : Packetbeat nécessite des privilèges élevés pour capturer le trafic réseau.

#### Étape 1 : Installation de Packetbeat

Installez Packetbeat :

```bash
sudo apt install -y packetbeat
```

#### Étape 2 : Configuration

Éditez le fichier de configuration :

```bash
sudo nano /etc/packetbeat/packetbeat.yml
```

Configurez les interfaces réseau à monitorer :

```yaml
packetbeat.interfaces.device: any

# Protocoles à capturer
packetbeat.protocols:
  - type: http
    ports: [80, 8080, 8000, 5000, 8002, 9200]
  - type: mysql
    ports: [3306]
  - type: redis
    ports: [6379]
```

Configurez la sortie :

```yaml
output.elasticsearch:
  hosts: ["localhost:9200"]
```

#### Étape 3 : Démarrage de Packetbeat

Démarrez Packetbeat (nécessite des privilèges root) :

```bash
sudo systemctl start packetbeat
sudo systemctl enable packetbeat
```

#### Étape 4 : Vérification

Vérifiez dans Kibana avec la data view `packetbeat-*`.

**Note** : Packetbeat peut être gourmand en ressources. Sur une VM avec 2 Go de RAM, utilisez-le avec précaution.

---

## 4. Partie 3 : Visualisation et analyse dans Kibana

### 4.1 Création de data views

#### Étape 1 : Accès aux data views

1. Dans Kibana, allez dans **Stack Management → Data Views**
2. Cliquez sur **Create data view*

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

**Tâche** :
1. Créez un nouveau dashboard : "Dashboard Sécurité"
2. Ajoutez les visualisations suivantes :
   - Graphique temporel des échecs d'authentification (Winlogbeat, EventID 4625)
   - Top 10 des IP sources avec le plus d'échecs de connexion
   - Carte géographique des connexions (si disponible)
   - Graphique des événements Windows par type (Security, Application, System)
   - Métriques système en temps réel (CPU, mémoire, disque)
3. Configurez des alertes (si disponible dans votre version) :
   - Alerte si plus de 10 échecs de connexion en 5 minutes
   - Alerte si utilisation CPU > 80%
4. Sauvegardez et partagez le dashboard

**Indice** : Utilisez les champs `winlog.event_id` pour filtrer les événements Windows spécifiques.

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
2. Sélectionnez l'index `winlogbeat-*`
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

**Scénario** : Analysez les logs pour identifier un pattern suspect d'attaque.

**Tâche** :
1. Générez ou simulez une activité suspecte :
   - Plusieurs tentatives de connexion échouées depuis différentes IP
   - Accès à des fichiers sensibles
   - Exécution de commandes suspectes
2. Dans Kibana, créez des recherches pour identifier :
   - Les IP sources des attaques
   - Les comptes ciblés
   - Les méthodes d'attaque utilisées
   - La timeline de l'attaque
3. Créez un rapport d'analyse avec :
   - Résumé de l'incident
   - Timeline des événements
   - Recommandations de réponse
4. Présentez vos findings dans un dashboard dédié

**Indice** : Utilisez les fonctionnalités de sauvegarde de recherche dans Kibana pour documenter vos analyses.

---

## 6. Annexes

### 6.1 Commandes utiles

#### Elasticsearch

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

#### Kibana

```bash
# Vérifier le statut
sudo systemctl status kibana

# Redémarrer
sudo systemctl restart kibana

# Voir les logs
sudo journalctl -u kibana -f
```

#### Logstash

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

```bash
# Vérifier le statut
sudo systemctl status metricbeat

# Tester la configuration
sudo metricbeat test config

# Lister les modules
sudo metricbeat modules list
```

#### Winlogbeat (Windows PowerShell)

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
2. Vérifiez la configuration dans `kibana.yml` :
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

1. **Vérifiez l'état des migrations dans les logs** :
   ```bash
   sudo journalctl -u kibana -f | grep -i "migration\|savedobjects"
   ```
   Vous devriez voir des messages comme "Starting saved objects migrations" puis "Migration completed" ou "Migration successful".

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

5. **Vérifiez les logs complets pour des erreurs** :
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
4. Vérifiez la connectivité vers Elasticsearch : `Test-NetConnection -ComputerName IP_VM -Port 9200`

#### Problème : Performance lente avec 2 Go de RAM

**Symptômes** : Système lent, services qui plantent

**Solutions** :
1. Réduisez le heap d'Elasticsearch (256m selon les autres services)
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

- [ ] Elasticsearch installé et fonctionnel
- [ ] Kibana accessible depuis votre PC Windows
- [ ] Logstash configuré avec au moins un pipeline
- [ ] Filebeat collectant les logs système
- [ ] Winlogbeat collectant les événements Windows
- [ ] Metricbeat collectant les métriques système
- [ ] Au moins 3 visualisations créées dans Kibana
- [ ] Un dashboard fonctionnel
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
