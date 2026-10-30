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
- Mettre en place TLS (HTTP et transport) et l’authentification entre les composants
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
- **OS** : Debian 13 (Trixie)
- **Stack** : Elastic **9.5.x** (dépôt `9.x`)
- **Ressources** : **8 Go RAM**, **8 vCPU**, ~20 Go disque — si ~2 Go RAM, réduire le heap (§2.2) ; une VM SIEM < 4 Go est déconseillée dès que Kibana + Logstash tournent ensemble
- **Accès** : SSH ; Kibana sur le port `5601` (HTTP) ; Elasticsearch sur `9200` (HTTPS)
- **Heaps (8 Go)** : Elasticsearch `-Xms2g -Xmx2g` ; Logstash laissé au défaut du paquet (`jvm.options`, 1 Go) ; `xpack.ml.enabled: false`

#### VM à surveiller (Linux)
- **Rôle** : sources de logs / métriques envoyées vers le SIEM
- **OS** : Debian 13 — **~2 Go RAM**, **2 vCPU**
- **Services cibles** : Suricata (alertes), SSH (journald), Apache, Metricbeat
- **Agents** : Filebeat + Metricbeat → `https://IP_SIEM:9200` (TLS + utilisateur `beats_writer`)

#### Poste Windows
- Accès à Kibana : `http://IP_SIEM:5601` (login `elastic`)
- **Winlogbeat** (même branche que le SIEM) : journaux Application / System / Security → Elasticsearch `https://IP_SIEM:9200`
- Vérifiez quelle interface Windows atteint réellement le SIEM (`curl` + `Get-NetTCPConnection`) — un poste multi-homed peut joindre le labo via Ethernet alors que la Wi-Fi sert à autre chose

#### Architecture réseau

```
┌──────────────────┐         ┌──────────────────────┐
│  VM à surveiller │         │  VM SIEM (ELK)       │
│  (Debian 13)     │         │  (Debian 13)         │
│                  │         │                      │
│  - Suricata      │──logs──▶│  - Elasticsearch     │
│  - Apache        │  Beats  │  - Kibana :5601      │
│  - SSH/journald  │────────▶│  - Logstash (opt.)   │
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

Avec **~8 Go** de RAM et **8 vCPU** :

```properties
# Heap 2 Go — marge confortable pour Kibana + Logstash sur 8 Go
-Xms2g
-Xmx2g
```

Avec **~6 Go** de RAM (autre sizing courant) :

```properties
# Heap 1 Go
-Xms1g
-Xmx1g
```

**Explication** :
- `-Xms` / `-Xmx` : taille initiale et maximale du heap JVM (gardez-les égales).
- La mémoire totale utilisée par Elasticsearch dépasse le heap (off-heap, buffers, processus auxiliaires).
- Vérifiez après démarrage avec `curl -sk -u elastic:$ELASTIC_PASS --cacert /etc/elasticsearch/certs/http_ca.crt https://localhost:9200/_nodes/jvm?pretty` (`heap_max_in_bytes`).

#### Étape 6 : Certificats TLS

Installez `unzip` si besoin, puis générez une CA et un certificat nœud avec **SAN** (localhost + IP du SIEM). Remplacez `IP_SIEM` par l’IP réelle.

```bash
sudo apt install -y unzip
CERTDIR=/tmp/es-certs
sudo mkdir -p "$CERTDIR" && cd "$CERTDIR"
sudo /usr/share/elasticsearch/bin/elasticsearch-certutil ca --pem --out "$CERTDIR/ca.zip" --silent
sudo unzip -oq ca.zip
sudo /usr/share/elasticsearch/bin/elasticsearch-certutil cert --pem \
  --ca-cert "$CERTDIR/ca/ca.crt" --ca-key "$CERTDIR/ca/ca.key" \
  --dns localhost --dns "$(hostname -s)" \
  --ip 127.0.0.1 --ip IP_SIEM \
  --out "$CERTDIR/certs.zip" --silent
sudo unzip -oq certs.zip
INSTANCE=$(sudo find "$CERTDIR" -mindepth 1 -maxdepth 1 -type d ! -name ca | head -1)

sudo mkdir -p /etc/elasticsearch/certs
sudo cp "$CERTDIR/ca/ca.crt" /etc/elasticsearch/certs/ca.crt
sudo cp "$CERTDIR/ca/ca.crt" /etc/elasticsearch/certs/http_ca.crt
sudo cp "$INSTANCE"/*.crt /etc/elasticsearch/certs/node.crt
sudo cp "$INSTANCE"/*.key /etc/elasticsearch/certs/node.key
sudo chown -R elasticsearch:elasticsearch /etc/elasticsearch/certs
sudo chmod 640 /etc/elasticsearch/certs/*
# Copie utilisable pour scp vers les Beats / votre PC
sudo cp /etc/elasticsearch/certs/http_ca.crt /home/$USER/http_ca.crt
sudo chown $USER:$USER /home/$USER/http_ca.crt
```

#### Étape 7 : Configuration d'Elasticsearch

Éditez la configuration principale pour exposer l’API, nommer le nœud et activer TLS + authentification.

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

**Configuration pour Elasticsearch 9.x** :

```yaml
cluster.name: siem-cluster
node.name: siem-node-1
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 0.0.0.0
http.host: 0.0.0.0
http.port: 9200
discovery.type: single-node

xpack.security.enabled: true
xpack.security.enrollment.enabled: false
xpack.ml.enabled: false

xpack.security.http.ssl:
  enabled: true
  certificate: certs/node.crt
  key: certs/node.key
  certificate_authorities: ["certs/ca.crt"]

xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  certificate: certs/node.crt
  key: certs/node.key
  certificate_authorities: ["certs/ca.crt"]
```

**Notes** :
- Le `elasticsearch.yml` du paquet a déjà, **décommentés**, `path.data: /var/lib/elasticsearch` et `path.logs: /var/log/elasticsearch`. Gardez-les. Le reste du fichier est commenté : ajoutez le bloc ci-dessus, ne réécrivez pas le fichier à vide.
- Dans Elasticsearch 9.x, `http.host` complète / remplace l’ancien `network.host` pour l’API HTTP.
- Sur un seul nœud, `discovery.type: single-node` évite `cluster.initial_master_nodes`.
- Ne mettez **jamais** les mots de passe réels dans un dépôt Git.

#### Étape 8 : Démarrage et comptes

```bash
sudo systemctl enable --now elasticsearch
sudo systemctl status elasticsearch
```

Attendez 30–60 s. Sans mot de passe, l’API répond en **401** (TLS OK, auth requise) :

```bash
curl -sk -o /dev/null -w '%{http_code}\n' https://127.0.0.1:9200
```

Réinitialisez les comptes built-in (notez les « New value ») :

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic -b
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system -b
```

Créez un rôle et un utilisateur pour les Beats :

```bash
export ELASTIC_PASS='…'   # mot de passe elastic ci-dessus
export BEATS_PASS='…'     # mot de passe fort choisi pour beats_writer

curl -sk -u "elastic:${ELASTIC_PASS}" --cacert /etc/elasticsearch/certs/http_ca.crt \
  -X PUT "https://127.0.0.1:9200/_security/role/beats_writer_role" \
  -H 'Content-Type: application/json' -d '{
  "cluster": ["monitor","manage_index_templates","manage_ingest_pipelines","manage_pipeline","manage_ilm"],
  "indices": [{
    "names": ["filebeat-*","metricbeat-*","winlogbeat-*","logs-*","logstash-*"],
    "privileges": ["write","create","create_index","view_index_metadata","auto_configure","manage"]
  }]
}'

curl -sk -u "elastic:${ELASTIC_PASS}" --cacert /etc/elasticsearch/certs/http_ca.crt \
  -X POST "https://127.0.0.1:9200/_security/user/beats_writer" \
  -H 'Content-Type: application/json' -d "{
  \"password\": \"${BEATS_PASS}\",
  \"roles\": [\"beats_writer_role\"],
  \"full_name\": \"Beats lab writer\"
}"
```

#### Étape 9 : Vérification

```bash
curl -sk -u "elastic:${ELASTIC_PASS}" --cacert /etc/elasticsearch/certs/http_ca.crt https://127.0.0.1:9200
curl -sk -u "elastic:${ELASTIC_PASS}" --cacert /etc/elasticsearch/certs/http_ca.crt https://127.0.0.1:9200/_nodes/jvm?pretty
```

Vous devez obtenir du JSON (infos cluster / heap). Depuis un autre hôte du labo (avec la CA copiée) :

```bash
curl -sk -u "elastic:${ELASTIC_PASS}" --cacert ./http_ca.crt https://IP_SIEM:9200
```

Sur un cluster à **un seul nœud**, le statut `yellow` (réplicas non assignés) est **normal**.

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

Copiez la CA Elasticsearch, puis éditez `kibana.yml` pour pointer vers HTTPS et le compte `kibana_system`. Ne remplacez pas le fichier entier.

```bash
sudo mkdir -p /etc/kibana/certs
sudo cp /etc/elasticsearch/certs/http_ca.crt /etc/kibana/certs/http_ca.crt
sudo chown -R kibana:kibana /etc/kibana/certs
ENC_KEY=$(openssl rand -hex 32)   # ≥ 32 caractères — gardez-la
sudo nano /etc/kibana/kibana.yml
```

```yaml
server.host: "0.0.0.0"
server.port: 5601

elasticsearch.hosts: ["https://127.0.0.1:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "KIBANA_SYSTEM_PASS"
elasticsearch.ssl.certificateAuthorities: ["/etc/kibana/certs/http_ca.crt"]
elasticsearch.ssl.verificationMode: full

xpack.encryptedSavedObjects.encryptionKey: "ENC_KEY"
xpack.reporting.encryptionKey: "ENC_KEY"
xpack.security.encryptionKey: "ENC_KEY"
```

Le paquet Kibana 9.5 a déjà deux réglages **actifs** (non commentés). **Gardez-les** :

```yaml
logging:
  appenders:
    file:
      type: file
      fileName: /var/log/kibana/kibana.log
      layout:
        type: json
  root:
    appenders: [default, file]
pid.file: /run/kibana/kibana.pid
```

Dans le même fichier, `server.host`, `server.port` et `elasticsearch.hosts` sont **commentés** (`localhost`, `http://localhost:9200`). Décommentez-les et remplacez-les par le bloc HTTPS ci-dessus. Ajoutez les trois `encryptionKey` (absentes du paquet).

Remplacez `KIBANA_SYSTEM_PASS` et `ENC_KEY` par les valeurs réelles, puis :

```bash
sudo chmod 640 /etc/kibana/kibana.yml
sudo chown root:kibana /etc/kibana/kibana.yml
```

- `server.host: "0.0.0.0"` : accessible depuis votre PC Windows.
- Kibana écoute en **HTTP** sur `:5601` ; le lien **Kibana → Elasticsearch** est en **HTTPS**.

#### Étape 3 : Démarrage de Kibana

```bash
sudo systemctl enable --now kibana
sudo systemctl status kibana
```

**Note importante** : 
- Kibana peut prendre **1-2 minutes** pour démarrer complètement.
- **Lors du premier démarrage**, Kibana effectue des migrations d'objets sauvegardés qui peuvent prendre **3-5 minutes** supplémentaires.
- Pendant ce temps, la page web peut rester en chargement. C'est **normal**.
- Suivez les logs : `sudo journalctl -u kibana -f` — absence de `Unable to connect to Elasticsearch` / erreur SSL.

#### Étape 4 : Accès à Kibana

Depuis le navigateur de votre poste Windows (remplacez `IP_SIEM`) :

```
http://IP_SIEM:5601
```

Connectez-vous avec l’utilisateur **`elastic`** et le mot de passe obtenu via `elasticsearch-reset-password`.

**Vérification** : page de login puis interface Kibana.

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

#### Étape 1 : JDK

Le paquet Logstash 9 embarque son propre JDK (`/usr/share/logstash/jdk`). N’installez pas `default-jre` pour ce TP : le service n’utilise pas le `java` du système. Le heap reste celui de `/etc/logstash/jvm.options` (1 Go). Ce labo ne le modifie pas.

---

#### Étape 2 : Installation de Logstash

Rechargez apt si besoin, puis installez Logstash (toujours depuis le dépôt Elastic 9.x).

```bash
sudo apt update
sudo apt install -y logstash
```

---

#### Étape 3 : Configuration de base de Logstash

Le fichier `/etc/logstash/logstash.yml` du paquet est entièrement commenté. `/etc/logstash/pipelines.yml` pointe **déjà** vers les pipelines :

```yaml
- pipeline.id: main
  path.config: "/etc/logstash/conf.d/*.conf"
```

Ne dupliquez pas `path.config` dans `logstash.yml`. Décommentez seulement les deux lignes déjà présentes :

```yaml
path.data: /var/lib/logstash
path.logs: /var/log/logstash
```

```bash
sudo nano /etc/logstash/logstash.yml
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
    hosts => ["https://127.0.0.1:9200"]
    user => "elastic"
    password => "${ELASTIC_PASS}"
    ssl_enabled => true
    ssl_certificate_authorities => ["/etc/elasticsearch/certs/http_ca.crt"]
    index => "test-logs-%{+YYYY.MM.dd}"
  }
  stdout {
    codec => rubydebug
  }
}
```

Avant de démarrer Logstash, copiez la CA si besoin (`sudo cp /etc/elasticsearch/certs/http_ca.crt /etc/logstash/http_ca.crt` et adaptez `ssl_certificate_authorities`) et substituez le mot de passe `elastic` (évitez de laisser un secret en clair durablement — labo uniquement). Sur Logstash 9.x, l’ancien réglage `cacert` est **obsolète** : le pipeline refuse de démarrer s’il est encore présent.

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
    hosts => ["https://127.0.0.1:9200"]
    user => "elastic"
    password => "${ELASTIC_PASS}"
    ssl_enabled => true
    ssl_certificate_authorities => ["/etc/elasticsearch/certs/http_ca.crt"]
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

#### Étape 1 : Installation de Filebeat

Debian 13 enregistre l’authentification et le syslog dans **journald** (`journalctl`). Le module `system` de Filebeat lit ce journal directement (`var.use_journald`). rsyslog n’est pas nécessaire.

Installez Filebeat (agent de collecte vers Elasticsearch).

```bash
sudo apt install -y filebeat
```

#### Étape 2 : Configuration de base (sortie vers le SIEM)

Le paquet installe un `/etc/filebeat/filebeat.yml` **par défaut** qui pointe encore vers `localhost:9200` en HTTP, avec auth et TLS **commentés**. Vous partez de ce fichier : ne le remplacez pas en entier, **modifiez la section `output.elasticsearch`**.

Sur la VM à surveiller, récupérez d’abord la CA depuis le SIEM :

```bash
scp UTILISATEUR@IP_SIEM:~/http_ca.crt /tmp/http_ca.crt
sudo mkdir -p /etc/filebeat/certs /etc/metricbeat/certs
sudo cp /tmp/http_ca.crt /etc/filebeat/certs/http_ca.crt
sudo cp /tmp/http_ca.crt /etc/metricbeat/certs/http_ca.crt
```

Éditez ensuite la config :

```bash
sudo nano /etc/filebeat/filebeat.yml
```

**Ce que le fichier stock contient** (repères à chercher, Elastic 9.x) :

```yaml
output.elasticsearch:
  hosts: ["localhost:9200"]
  preset: balanced
  #protocol: "https"
  #username: "elastic"
  #password: "changeme"
```

Plus bas, une section **`#output.logstash:`** contient aussi des lignes `#ssl.certificate_authorities: …` — **ne les décommentez pas** : ce n’est pas la sortie Elasticsearch, et les Beats de ce TP parlent à Elasticsearch, pas à Logstash (`:5044`).

Conservez le reste du fichier paquet, en particulier :

- `filebeat.inputs` : l’exemple `filestream` reste `enabled: false`. Si vous le passez à `true`, Filebeat envoie aussi `/var/log/*.log` en plus des modules.
- `filebeat.config.modules` (`modules.d/*.yml`) : c’est ce qui charge system / apache / suricata.
- `setup.template.settings` (`index.number_of_shards: 1`).
- `processors` (au minimum `add_host_metadata`).

**Ce qu’il faut faire** dans `output.elasticsearch` seulement :

1. Remplacer `hosts` par l’URL **HTTPS** du SIEM (`https://` dans l’URL suffit ; vous pouvez laisser `#protocol: "https"` commenté).
2. Décommenter et adapter `username` / `password` (`beats_writer` / mot de passe du labo — **pas** `elastic`).
3. **Ajouter** un bloc `ssl:` (absent du YAML paquet sous `output.elasticsearch`) pour faire confiance à la CA du labo.

Résultat attendu :

```yaml
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["https://IP_SIEM:9200"]
  preset: balanced
  #protocol: "https"   # inutile si hosts commence déjà par https://
  username: "beats_writer"
  password: "BEATS_PASS"
  ssl:
    enabled: true
    certificate_authorities: ["/etc/filebeat/certs/http_ca.crt"]
    verification_mode: full
```

- Sans `https://` (ou sans `protocol: "https"`), Filebeat tente du **HTTP** clair → échec vers ES 9 en TLS.
- Sans le bloc `ssl` + CA, le handshake échoue souvent (*x509: certificate signed by unknown authority*).
- `verification_mode: full` exige que l’IP/hostname dans `hosts` soit dans le SAN du certificat nœud (le TP a généré un cert avec `IP_SIEM`).

Vérifiez : `sudo filebeat test output` — attendu : `TLS... handshake... OK` puis `talk to server... OK`.
#### Étape 3 : Activation des modules (système, Apache, Suricata)

Activez les modules Filebeat qui savent lire les logs SSH/auth, Apache et Suricata (EVE JSON) sans écrire les chemins à la main.

```bash
sudo filebeat modules enable system apache suricata
```

**Suricata** : n’activez le module que si `/var/log/suricata/eve.json` existe **et** que le fileset `eve` est configuré. Sinon : `sudo filebeat modules disable suricata` (sinon Filebeat refuse de démarrer).

#### Étape 4 : Configuration des modules

**Système (SSH / auth)** — `/etc/filebeat/modules.d/system.yml` :

```yaml
- module: system
  syslog:
    enabled: true
    var.use_journald: true
  auth:
    enabled: true
    var.use_journald: true
```

`var.use_journald: true` : Filebeat lit le journal systemd (facilities auth/authpriv pour `auth`, le reste pour `syslog`) au lieu de `/var/log/auth.log` et `/var/log/syslog`.

Le service du paquet tourne en root et peut lire le journal. S’il tourne sous l’utilisateur `filebeat`, ajoutez-le au groupe qui possède le journal, puis redémarrez :

```bash
sudo usermod -aG systemd-journal filebeat
sudo systemctl restart filebeat
```

Contrôle : `sudo journalctl -u filebeat -n 30` ne doit plus afficher un harvester sur `/var/log/auth.log`. Une erreur `permission denied` sur le journal signifie que le compte du service n’est pas dans `systemd-journal`.

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

Sans `var.paths`, le module Apache du paquet lit déjà `/var/log/apache2/` sur Debian. Les chemins ci-dessus ne servent que si les logs sont ailleurs. Suricata, lui, a besoin de `var.paths` vers `eve.json`.

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
sudo tee /etc/suricata/rules/lab-test.rules <<'EOF'
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

Les alertes sont dans `/var/log/suricata/eve.json`. Les règles gérées par `suricata-update` vivent sous `/etc/suricata/rules`.

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

Les index / data streams : `.ds-filebeat-*` (pattern Kibana : `filebeat-*`).

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
> Alignez la version Winlogbeat sur celle du SIEM.  
> Remplacez `IP_SIEM` par l’adresse IP de la VM ELK.

#### Étape 0 : Vérifier la connectivité vers Elasticsearch

Avant d’installer l’agent, confirmez que **ce** poste Windows joint le SIEM sur le port `9200` (sinon Winlogbeat installé mais muet).

```powershell
curl.exe -sk -u elastic:ELASTIC_PASS --cacert C:\path\to\http_ca.crt https://IP_SIEM:9200
# Doit renvoyer un JSON (cluster.name, version.number, …)

# Quelle IP source Windows est utilisée ?
Get-NetTCPConnection -RemoteAddress IP_SIEM -RemotePort 9200 |
  Select-Object LocalAddress, LocalPort, State
```

- `curl.exe` : le client HTTP Windows (évite l’alias PowerShell `curl` = `Invoke-WebRequest`).
- `Get-NetTCPConnection` : montre l’interface locale réellement utilisée vers le SIEM.

Un poste multi-homed (Wi-Fi + Ethernet) peut atteindre le SIEM via une interface différente de celle utilisée pour VNC ou le navigateur. Notez l’`LocalAddress` réellement utilisée.

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

L’archive ZIP n’est pas sur les VMs Linux : le `winlogbeat.yml` livré suit le même modèle que Filebeat (`hosts: ["localhost:9200"]`, `#protocol: "https"`, auth commentée, lignes `#ssl` sous `#output.logstash`). Même geste que §3.1 : URL `https://`, compte `beats_writer`, bloc `ssl` sous `output.elasticsearch`. Ne décommentez pas le SSL Logstash.

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
  hosts: ["https://IP_SIEM:9200"]
  username: "beats_writer"
  password: "BEATS_PASS"
  ssl:
    enabled: true
    certificate_authorities: ["C:\\path\\to\\http_ca.crt"]
    verification_mode: full
```

Copiez `http_ca.crt` depuis le SIEM sur le PC Windows et adaptez le chemin.  
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
2. Data view : `winlogbeat-*` (champ temporel `@timestamp`)
3. Dans **Discover**, filtres utiles :
   - Connexions réussies : `winlog.event_id: 4624`
   - Échecs de connexion : `winlog.event_id: 4625`
   - Canal Security : `winlog.channel: "Security"`
4. Data streams : `.ds-winlogbeat-*` (pattern Kibana : `winlogbeat-*`)

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

Même logique que Filebeat : le `metricbeat.yml` du paquet pointe vers `localhost:9200`, avec `#protocol: "https"` et auth commentés. Éditez **`output.elasticsearch`** seulement (ne décommentez pas le SSL sous `#output.logstash:`).

Conservez `metricbeat.config.modules`, `setup.template.settings` et `processors`. N’ajoutez pas de second bloc `metricbeat.modules:` dans ce fichier : le module `system` vient de `modules.d/system.yml` (cpu, load, memory, network, process, `socket_summary`, filesystem). Un bloc en plus collecte les mêmes métriques une seconde fois.

```bash
sudo nano /etc/metricbeat/metricbeat.yml
```

Même CA / utilisateur que Filebeat :

```yaml
output.elasticsearch:
  hosts: ["https://IP_SIEM:9200"]
  username: "beats_writer"
  password: "BEATS_PASS"
  ssl:
    enabled: true
    certificate_authorities: ["/etc/metricbeat/certs/http_ca.crt"]
    verification_mode: full
```

Vérifiez : `sudo metricbeat test output`.
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

Data streams : `.ds-metricbeat-*` (pattern Kibana : `metricbeat-*`).

#### Étape 6 : Vérification dans Kibana

1. Créez une data view : `metricbeat-*` (`@timestamp`)
2. Explorez dans **Discover** (`event.dataset: system.cpu`, `system.memory`, …)
3. Ouvrez (ou créez) votre dashboard métriques (voir §4.3)

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
  hosts: ["https://IP_SIEM:9200"]
  username: "beats_writer"
  password: "BEATS_PASS"
  ssl:
    enabled: true
    certificate_authorities: ["/etc/packetbeat/certs/http_ca.crt"]
    verification_mode: full
```

Copiez aussi la CA dans `/etc/packetbeat/certs/`.  
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

#### Dashboards à construire vous-même

Créez vos propres dashboards dans Kibana (**Analytics → Dashboard**). Ne comptez pas sur un import de fichiers fournis : sur Kibana **9.5**, `filebeat setup --dashboards` / `metricbeat setup --dashboards` échoue souvent (`fail to parse kibana version`), et ce n’est **pas** le livrable attendu.

Data views à créer (§4.1) : `filebeat-*`, `metricbeat-*`, `winlogbeat-*` (champ temporel `@timestamp`).

Cible minimale (noms libres, tant que le contenu est là) :

| Dashboard | Sources | Panneaux / filtres utiles |
| --- | --- | --- |
| Vue Linux / sécurité | Suricata, SSH/auth, Apache, Metricbeat | `event.kind: "alert"`, `source.ip`, auth SSH, `http.response.status_code`, CPU/mémoire |
| Métriques hôte | `metricbeat-*` | CPU, mémoire, load, disque `/`, réseau, processus (fenêtre ~24 h) |
| Windows | `winlogbeat-*` | `winlog.event_id: 4624` / `4625`, `winlog.channel` |
| GeoIP (si Suricata + trafic public) | `filebeat-*` | cartes `destination.geo.location` / `source.geo.location`, pays / IP |

Astuce : en **Discover**, filtrez, **Save** la recherche, puis **Add from library** sur le dashboard.

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

**Tâche (partie Linux)** — créez un dashboard « Sécurité Linux » avec au minimum :
- Alertes Suricata dans le temps (`event.kind: "alert"`, breakdown `rule.name`)
- Top IP sources (`source.ip`)
- Événements `system.auth` (SSH)
- Accès Apache (`http.response.status_code`)
- Métriques Metricbeat (CPU / mémoire)

**Tâche (partie Windows — Winlogbeat §3.2)** — créez un dashboard Windows avec :
- Connexions réussies (`winlog.event_id: 4624`)
- Échecs d’authentification (`winlog.event_id: 4625`)
- Journal Security / répartition des canaux (`winlog.channel`)
- Hôte source (`host.name`)

Vérifiez que les panneaux affichent des documents. Alertes Kibana (seuils) : optionnel selon licence / version.

**Indice** : champs Windows `winlog.event_id`, `winlog.channel` ; champs Suricata ECS `rule.name`, `event.kind`.

### 4.6 Carte GeoIP : où part le trafic, qui arrive

Le pipeline d’ingest Filebeat du module Suricata (`filebeat-*-suricata-eve-pipeline`) applique déjà un processeur **GeoIP** sur `source.ip` et `destination.ip`. Elasticsearch tient à jour les bases GeoLite2 (City, Country, ASN). Il n’y a pas de compte MaxMind à créer.

Les adresses privées (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) ne reçoivent pas de coordonnées. Sur la carte, seuls les points publics apparaissent :

- trafic **sortant** : `destination.geo.location` (la source est la VM, donc privée) ;
- connexions **vers la VM** : `source.geo.location` (la destination est la VM, donc privée).

Créez un dashboard GeoIP (data view `filebeat-*`, fenêtre ~24 h) avec au minimum : une carte destinations (`destination.geo.location`), une carte sources (`source.geo.location`), des tableaux pays/IP destination et source, et (si présent) les alertes `rule.name: "ESAIP-LAB-SSH-PROBE"`.

Un site web hébergé derrière un CDN se géolocalise souvent aux États-Unis ou en France, pas dans le pays du nom de domaine. Le pays affiché est celui de l’adresse IP réellement contactée.

#### Trafic sortant

Lancez les requêtes **depuis la VM à surveiller**. Un `curl` vers `localhost` ne passe pas par `eth0`, donc Suricata ne le voit pas.

```bash
for h in \
  www.service-public.fr www.bundestag.de www.cam.ac.uk www.boe.es \
  www.uniroma1.it www.uva.nl www.kth.se www.uio.no www.uw.edu.pl \
  www.ulisboa.pt www.tcd.ie www.univie.ac.at www.uoa.gr www.mit.edu \
  www.utoronto.ca www.uba.ar www.unam.mx www.uchile.cl www.snu.ac.kr \
  www.sydney.edu.au www.auckland.ac.nz www.iitb.ac.in www.uct.ac.za \
  www.um5.ac.ma www.metu.edu.tr www.tau.ac.il www.uaeu.ac.ae \
  www.tsinghua.edu.cn www.ntu.edu.tw www.chula.ac.th www.ui.ac.id \
  www.hi.is www.unibuc.ro www.elte.hu www.gov.za www.admin.ch \
  www.canada.ca www.japan.go.jp www.australia.gov.au
do
  curl -sS -o /dev/null -w "%{http_code} ${h}\n" \
    --connect-timeout 5 --max-time 12 -L --max-redirs 2 "https://${h}/" || true
done
```

- `-L` : suit les redirections ; `--max-redirs 2` : s’arrête après deux sauts.
- Un code HTTP quelconque (même `403` ou un échec TLS après le SYN) suffit : Suricata enregistre le flux, et le pipeline remplit `destination.geo.*`.

Contrôle dans Discover : `event.dataset: "suricata.eve" and source.ip: "IP_SURVEILLEE" and destination.geo.country_name: *`.

#### Connexions entrantes

La VM n’est pas joignable depuis Internet. Pour avoir des points « qui se connecte », on envoie **quelques SYN TCP vers le port 22 de la VM surveillée seulement**, avec une adresse source publique dans l’en-tête IP. Le handshake ne se termine pas : aucun mot de passe n’est essayé, et `sshd` ne voit pas de session.

Vérifiez d’abord qu’aucune règle ne va jeter ces paquets :

- Suricata est en IDS : dans `/etc/suricata/suricata.yaml`, `copy-mode: ips` reste commenté, `af-packet` écoute `eth0`.
- `rule-files` ne charge que des règles `alert` (pas de `drop` / `reject` actif).
- UFW est inactif et la politique iptables est `ACCEPT`.

Ajoutez une alerte de labo, pas un blocage. Fichier `/etc/suricata/rules/lab-geo.rules` :

```
alert tcp !$HOME_NET any -> $HOME_NET 22 (msg:"ESAIP-LAB-SSH-PROBE"; flags:S; sid:9000002; rev:1;)
```

Dans `rule-files`, à côté de `lab-test.rules` :

```yaml
rule-files:
  - lab-test.rules
  - lab-geo.rules
```

Puis `sudo suricatasc -c reload-rules`. `$HOME_NET` contient déjà `10.0.0.0/8` et `172.16.0.0/12` : un SSH réel depuis le réseau du labo ne matche pas. Suricata peut prévenir que le sens « vers le client » est ignoré pour un simple SYN ; les SYN vers le serveur restent détectés.

Depuis le poste de travail, forger l’adresse source exige les droits root (`nping` : `Mode TCP requires root privileges`), et la passerelle du campus peut jeter un paquet dont la source n’est pas celle du poste. Émettez les SYN **depuis la VM SIEM**, sur le même LAN que la sonde, uniquement vers `IP_SURVEILLEE` port 22. Trois SYN par adresse suffisent.

```bash
# Sur la VM SIEM, en root. Remplacez IP_SURVEILLEE.
sudo python3 - << 'PY'
import socket, struct, random, time
dst = "IP_SURVEILLEE"
sources = ["8.8.8.8", "80.67.169.12", "194.150.168.168", "129.67.1.190",
           "202.12.27.33", "223.5.5.5", "200.160.0.8", "157.92.1.1",
           "129.78.5.8", "137.132.1.1", "196.216.2.6", "132.66.1.1"]

def checksum(data):
    if len(data) % 2:
        data += b"\x00"
    s = sum(struct.unpack("!%dH" % (len(data) // 2), data))
    while s > 0xFFFF:
        s = (s & 0xFFFF) + (s >> 16)
    return ~s & 0xFFFF

sock = socket.socket(socket.AF_INET, socket.SOCK_RAW, socket.IPPROTO_RAW)
for src in sources:
    for n in range(3):
        ip_s, ip_d = socket.inet_aton(src), socket.inet_aton(dst)
        ip_id = random.randint(1, 65535)
        ip = struct.pack("!BBHHHBBH4s4s", 0x45, 0, 40, ip_id, 0, 64, 6, 0, ip_s, ip_d)
        ip = struct.pack("!BBHHHBBH4s4s", 0x45, 0, 40, ip_id, 0, 64, 6, checksum(ip), ip_s, ip_d)
        seq = random.randint(0, 2**32 - 1)
        tcp = struct.pack("!HHLLBBHHH", 41000 + n, 22, seq, 0, 5 << 4, 2, 65535, 0, 0)
        pseudo = struct.pack("!4s4sBBH", ip_s, ip_d, 0, 6, len(tcp))
        tcp = struct.pack("!HHLLBBHHH", 41000 + n, 22, seq, 0, 5 << 4, 2, 65535, checksum(pseudo + tcp), 0)
        sock.sendto(ip + tcp, (dst, 0))
        time.sleep(0.02)
print("envoyé", 3 * len(sources))
PY
```

Contrôle sur la sonde : `tcpdump -ni eth0 'tcp dst port 22'` voit les SYN, et `eve.json` contient `ESAIP-LAB-SSH-PROBE`. Dans Kibana : `rule.name: "ESAIP-LAB-SSH-PROBE" and source.geo.country_name: *`.

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
curl -sk -u elastic:$ELASTIC_PASS --cacert /etc/elasticsearch/certs/http_ca.crt https://localhost:9200/_cat/indices?v

# Supprimer un index (attention !)
curl -sk -u elastic:$ELASTIC_PASS --cacert /etc/elasticsearch/certs/http_ca.crt -X DELETE https://localhost:9200/nom-index
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

### 6.2 Logstash 9 — options SSL obsolètes (output Elasticsearch)

Sur Logstash 9.x, ces réglages **refusent le démarrage** du plugin. Le TP utilise déjà les nouveaux noms ; ne les remettez pas « comme dans un vieux tuto ».

| Obsolète (ne plus utiliser) | Remplacer par |
| --- | --- |
| `cacert` | `ssl_certificate_authorities` (liste de chemins PEM) |
| `ssl => true` / `ssl => false` | `ssl_enabled => true` / `false` |
| `ssl_certificate_verification` | `ssl_verification_mode` (`full` ou `none`) |
| `keystore` / `keystore_password` | `ssl_keystore_path` / `ssl_keystore_password` |
| `truststore` / `truststore_password` | `ssl_truststore_path` / `ssl_truststore_password` |

Toujours valides dans ce TP : `hosts`, `user`, `password`, `index`, `ssl_enabled`, `ssl_certificate_authorities`.

**Ne pas confondre** avec d’autres outils (syntaxe différente, toujours correcte) :
- `curl --cacert …`
- `elasticsearch-certutil --ca-cert …`
- Beats / Winlogbeat : bloc YAML `ssl:` avec `enabled`, `certificate_authorities`, `verification_mode`
- Kibana : `elasticsearch.ssl.certificateAuthorities`, `elasticsearch.ssl.verificationMode`

### 6.3 Dépannage

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
1. Vérifiez qu'Elasticsearch fonctionne : `curl -sk -u elastic:$ELASTIC_PASS --cacert /etc/elasticsearch/certs/http_ca.crt https://localhost:9200`
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
   curl -sk -u elastic:$ELASTIC_PASS --cacert /etc/elasticsearch/certs/http_ca.crt -X DELETE "https://localhost:9200/.kibana*"
   
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

#### Problème : `filebeat test output` échoue (TLS / HTTPS)

**Symptômes** : `connection... ERROR`, `x509: …`, ou tentative en HTTP alors qu’ES écoute en HTTPS.

**Causes fréquentes** (YAML **paquet par défaut**) :
1. `hosts` encore à `localhost:9200` sans `https://`, et `#protocol: "https"` toujours commenté
2. `#username` / `#password` encore commentés, ou compte `elastic` au lieu de `beats_writer`
3. Bloc `ssl:` / CA manquant sous `output.elasticsearch` (les lignes `#ssl.certificate_authorities` du fichier stock sont sous **Logstash**, pas sous Elasticsearch)
4. Fichier CA absent : `/etc/filebeat/certs/http_ca.crt`

**Solutions** : reprendre §3.1 étape 2 ; recopier la CA ; `sudo filebeat test output` jusqu’à `handshake... OK` et `talk to server... OK`.

#### Problème : Les logs n'apparaissent pas dans Kibana

**Symptômes** : Aucune donnée dans Discover

**Solutions** :
1. Vérifiez que les Beats fonctionnent : `sudo systemctl status filebeat`
2. Vérifiez les index dans Elasticsearch : `curl -sk -u elastic:$ELASTIC_PASS --cacert /etc/elasticsearch/certs/http_ca.crt https://localhost:9200/_cat/indices?v`
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
- [ ] Kibana accessible (`http://IP_SIEM:5601`) avec login `elastic`
- [ ] Elasticsearch joignable en HTTPS avec CA + identifiants
- [ ] Logstash configuré avec au moins un pipeline (test ou beats)
- [ ] Filebeat sur la VM à surveiller : modules `system`, `apache`, `suricata`
- [ ] Alertes Suricata visibles (`event.kind: "alert"`)
- [ ] Traces SSH / auth visibles (`event.dataset: "system.auth"`)
- [ ] Logs Apache visibles (`event.dataset: "apache.access"`)
- [ ] Metricbeat collectant les métriques hôte (`metricbeat-*`)
- [ ] Un dashboard Linux / sécurité que **vous** avez créé (au moins 3 visualisations)
- [ ] Un dashboard GeoIP que **vous** avez créé : destinations (sortant) et sources (vers la VM)
- [ ] Winlogbeat installé (même branche que le SIEM), service **Running**, data view `winlogbeat-*`
- [ ] Un dashboard Windows que **vous** avez créé (4624 / 4625 / canaux)
- [ ] Configurations et découvertes documentées (y compris l’IP source Windows → SIEM:9200)
