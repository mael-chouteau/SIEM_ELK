# Introduction au SIEM
## Sécurité des informations et management des événements

**UE S10-3 Systèmes & Réseaux**  
**Volume horaire encadré : 21 h | Charge horaire totale : 40 h**

---

## 1. Contexte et enjeux

Dans un environnement informatique moderne, les organisations génèrent des **millions d'événements** chaque jour :
- Connexions utilisateurs
- Accès aux ressources
- Tentatives d'intrusion
- Erreurs système
- Modifications de configuration
- Activités réseau

**Le défi** : Comment détecter une **menace réelle** parmi ce flux continu d'informations ?

Les attaques modernes sont :
- **Sophistiquées** : Multi-étapes, distribuées, furtives
- **Rapides** : Détection et réponse en temps réel nécessaires
- **Complexes** : Nécessitent la corrélation de multiples sources

**Le SIEM** (Security Information and Event Management) est la solution pour :
- **Centraliser** tous les événements de sécurité
- **Corréler** les événements pour détecter les menaces
- **Analyser** les patterns d'attaque
- **Répondre** rapidement aux incidents

---

## 2. Définition du SIEM

### 2.1 Qu'est-ce qu'un SIEM ?

**SIEM** = **Security Information and Event Management**

Un SIEM est une solution qui :
1. **Collecte** les logs et événements de sécurité depuis diverses sources
2. **Normalise** et **stocke** ces données dans un format unifié
3. **Corrèle** les événements pour identifier les menaces
4. **Analyse** les patterns et comportements suspects
5. **Alerte** les équipes de sécurité en temps réel
6. **Visualise** les données pour faciliter l'investigation

#### À quoi sert un SIEM ?

Un SIEM permet l'**ingestion de grands volumes de données** (logs) pour pouvoir les analyser et les présenter sous forme de **tableaux de bord** et d'**alertes** afin de traiter rapidement un incident (attaque/panne). 

Les principaux bénéfices sont :
- **Centralisation** : Tout regrouper en un seul point de recherche
- **Historique préservé** : Les logs ne peuvent pas être altérés par un attaquant
- **Analyse en temps réel** : Détection rapide des menaces
- **Investigation** : Facilite l'analyse forensique et la réponse aux incidents

### 2.2 Composants clés

Un SIEM moderne comprend généralement :

- **Collecteurs** : Agents qui récupèrent les logs (Beats, agents, syslog)
- **Moteur de corrélation** : Analyse les événements et détecte les patterns
- **Base de données** : Stockage des événements (Elasticsearch, base de données)
- **Interface de visualisation** : Dashboards et analyses (Kibana, interface web)
- **Moteur d'alertes** : Génération d'alertes et notifications

---

## 3. Objectifs pédagogiques

### 3.1 Objectifs en matière de compétences

À l'issue de ce cours, vous serez capable de :

-  **Mettre en place une solution SIEM** complète
-  **Centraliser et gérer les journaux** en provenance des IDS/IPS et équipements d'exploitation
-  **Détecter les menaces** parmi un grand volume d'information
-  **Créer des visualisations** avec les données chargées à l'aide de Kibana
-  **Analyser les données en temps réel** avec la pile ELK

### 3.2 Objectifs en matière de connaissances

Vous connaîtrez :

-  Les **caractéristiques clés des SIEM** et les solutions du marché
-  Les **technologies défensives** autour de la terminologie SIEM
-  Le **fonctionnement d'une solution SIEM** et ses avantages
-  Les **principes fondamentaux de la pile ELK** et les cas d'utilisation

---

## 4. Architecture SIEM

### 4.1 Architecture générale

```
┌─────────────────────────────────────────────────────────┐
│                    Sources de données                   │
├──────────┬──────────┬──────────┬──────────┬─────────────┤
│  IDS/IPS │  Firewall│  Serveurs│  Windows │  Appliances │
│          │          │  Linux   │  Events  │  Réseau     │
└────┬─────┴────┬─────┴────┬─────┴────┬─────┴─────┬───────┘
     │          │          │          │           │
     └──────────┴──────────┴──────────┴───────────┘
                    │
         ┌──────────▼──────────┐
         │  Agents/Collecteurs │
         │ (Beats, Syslog, etc)│
         └──────────┬──────────┘
                    │
         ┌──────────▼───────────┐
         │   Moteur SIEM        │
         │  ┌────────────────┐  │
         │  │  Normalisation │  │
         │  │  Corrélation   │  │
         │  │  Analyse       │  │
         │  └────────────────┘  │
         └──────────┬───────────┘
                    │
     ┌──────────────┴──────────────┐
     │                              │
┌────▼─────┐              ┌─────────▼────────┐
│  Stockage│              │   Visualisation  │
│ (Elastic)│              │     (Kibana)     │
└────┬─────┘              └──────────────────┘
     │
┌────▼─────┐
│  Alertes │
│ & Réponse│
└──────────┘
```

### 4.2 Chaîne de traitement des données

La chaîne de traitement des données dans un SIEM suit un flux structuré :

1. **Sources de données** : 
   - Logs systèmes (serveurs Linux/Windows)
   - IDS/IPS (systèmes de détection/prévention d'intrusion)
   - Équipements réseau (firewalls, routeurs, switches)
   - Applications et services

2. **Outils de collecte** :
   - **Beats** : Agents légers (Filebeat, Winlogbeat, Metricbeat)
   - **API** : Interfaces de programmation pour récupération de logs
   - **Agents** : Agents dédiés installés sur les sources
   - **Syslog** : Protocole standardisé de collecte de logs

3. **Traitement des logs** :
   - **Collecte** : Récupération des logs sur un serveur distinct de la source
   - **Normalisation** : Conversion en format standardisé
   - **Enrichissement** : Ajout de contexte (géolocalisation, réputation IP, etc.)
   - **Corrélation** : Analyse des relations entre événements
   - **Analyse** : Détection de patterns et comportements suspects

4. **Stockage** :
   - Archivage à court terme (données chaudes)
   - Archivage à long terme (données froides)
   - Gestion du cycle de vie des données

5. **Visualisation** :
   - **Dashboards** : Tableaux de bord personnalisables
   - **Index de recherche** : Recherche avancée dans les logs
   - **Alertes** : Notifications en temps réel en cas de détection de menace

### 4.3 Stratégies de déploiement

Le choix du mode de déploiement d'un SIEM est crucial et dépend des contraintes organisationnelles, techniques et réglementaires.

#### Déploiement on-premise

**Avantages** :
- **Contrôle total des données** : Souveraineté et maîtrise complète
- **Conformité stricte facilitée** : Respect des réglementations locales
- **Pas de latence réseau WAN** : Performance optimale

**Inconvénients** :
- **Coûts initiaux élevés (CapEx)** : Investissement matériel important
- **Maintenance matérielle complexe** : Nécessite une équipe dédiée
- **Scalabilité limitée au hardware** : Extension coûteuse et complexe

#### Déploiement cloud

**Avantages** :
- **Déploiement ultra-rapide** : Mise en service en quelques heures
- **Élasticité et scalabilité automatique** : Adaptation automatique à la charge
- **Maintenance gérée par le fournisseur** : Réduction de la charge opérationnelle

**Inconvénients** :
- **Coûts variables parfois imprévisibles** : Facturation selon l'usage
- **Dépendance internet/fournisseur** : Risque de coupure ou de changement de politique
- **Questions de confidentialité données** : Données hébergées chez un tiers

#### Déploiement hybride

**Avantages** :
- **Flexibilité optimale** : Meilleur des deux mondes
- **Données sensibles en local** : Sécurité renforcée pour les données critiques
- **Transition progressive possible** : Migration douce vers le cloud

**Inconvénients** :
- **Architecture plus complexe** : Gestion de deux environnements
- **Gestion unifiée difficile** : Nécessite des outils de coordination
- **Latence entre composants** : Performance potentiellement impactée

---

## 5. Solutions du marché

### 5.1 Solutions open-source

#### ELK Stack (Elastic Stack)
- **Elasticsearch** : Moteur de recherche et stockage
- **Logstash** : Traitement et transformation des logs
- **Kibana** : Interface de visualisation
- **Beats** : Agents légers de collecte
- **Avantages** : Gratuit, flexible, communauté active
- **Inconvénients** : Nécessite expertise technique

#### Wazuh
- SIEM open-source basé sur OSSEC
- Intégration avec ELK Stack
- Bon pour les petites/moyennes organisations

### 5.2 Solutions commerciales

#### Splunk
- Leader du marché
- Interface intuitive
- Coût élevé selon le volume de données

#### IBM QRadar
- Solution enterprise complète
- Bonne intégration avec l'écosystème IBM

#### ArcSight (Micro Focus)
- Solution mature et robuste
- Utilisée par de grandes organisations

#### Sentinel (Microsoft)
- Solution cloud-native
- Intégration avec l'écosystème Microsoft

### 5.3 Facteurs de choix d'une solution SIEM

Le choix d'une solution SIEM et de son mode d'hébergement doit prendre en compte plusieurs facteurs critiques :

#### Volume et scalabilité
- **Estimer le volume de logs** : Mesurer les EPS (Events Per Second) et GB par jour
- **Capacité d'évolution** : Vérifier que la solution peut évoluer avec la croissance de l'organisation
- **Performance** : S'assurer que la solution peut traiter le volume attendu en temps réel

#### Expertise et complexité
- **Évaluer l'expertise de l'équipe** : 
  - Solution clé en main (commerciale) pour les équipes moins techniques
  - Solution flexible (open-source) pour les équipes expérimentées
- **Courbe d'apprentissage** : Temps nécessaire pour maîtriser la solution
- **Support disponible** : Documentation, communauté, support commercial

#### Coûts
- **Calculer le coût total sur 3-5 ans** :
  - Licences et abonnements
  - Infrastructure (matériel, cloud)
  - Formation des équipes
  - Maintenance et support
- **Modèle de facturation** : Par volume, par utilisateur, forfaitaire
- **Coûts cachés** : Stockage, bande passante, intégrations

#### Intégration et compatibilité
- **Vérifier la compatibilité** avec les outils existants :
  - EDR (Endpoint Detection and Response)
  - Solutions cloud (AWS, Azure, GCP)
  - Firewalls et équipements réseau
  - Systèmes d'exploitation
- **Richesse des plugins et connecteurs** : Facilité d'intégration
- **Standards supportés** : Syslog, CEF, JSON, etc.

---

## 6. La pile ELK (Elastic Stack)

### 6.1 Présentation

**ELK** = **E**lasticsearch + **L**ogstash + **K**ibana

Aujourd'hui appelée **Elastic Stack**, elle inclut également **Beats**.

### 6.2 Composants

#### Elasticsearch

- **Rôle** : Moteur de recherche et d'analyse distribué, base de données NoSQL pour le stockage et l'indexation
- **Fonction** : Stockage et indexation des logs
- **Caractéristiques** : Distribué, scalable, recherche rapide
- **API** : RESTful pour l'interaction

##### Pourquoi Elasticsearch est parfait pour un SIEM ?

Elasticsearch présente plusieurs caractéristiques qui en font une solution idéale pour les besoins d'un SIEM :

- **Base de données orientée documents (JSON)** : Format flexible sans schéma rigide, adapté à la variété des formats de logs
- **Scalabilité horizontale** : Conçu pour scaler en ajoutant des nœuds, essentiel pour gérer de grands volumes
- **Moteur Lucene intégré** : Indexation ultra-rapide et recherche full-text quasi instantanée sur des millions de documents
- **Capacités analytiques puissantes** : Calcul de statistiques et métriques sur de grands volumes de données en temps réel
- **Distribution automatique** : Les données sont réparties en shards (fragments) et réplicas pour la performance et la tolérance aux pannes
- **Interface JSON standard via HTTP** : API REST simple et universelle
- **Gestion automatisée du cycle de vie** : Support des index Hot/Warm/Cold/Frozen pour optimiser les coûts de stockage

#### Logstash

- **Rôle** : Pipeline de traitement de données côté serveur (ETL) pour ingérer, transformer et enrichir les logs
- **Fonction** : Collecte, transformation, enrichissement des logs

##### Les 3 étapes du traitement Logstash

Logstash fonctionne selon un pipeline en trois étapes :

1. **Input (Récupération des sources)** :
   - **Files** : Lecture de fichiers de logs
   - **Beats** : Réception depuis les agents Beats
   - **Syslog** : Protocole syslog standard
   - **HTTP** : Réception via API REST
   - **TCP/UDP** : Réception de flux réseau
   - **Kafka** : Intégration avec Apache Kafka
   - **JDBC** : Connexion aux bases de données

2. **Filter (Transformation, normalisation et enrichissement)** :
   - **Grok** : Parsing de logs non structurés avec expressions régulières
   - **Date** : Parsing et normalisation des dates
   - **Mutate** : Modification des champs (renommage, conversion, suppression)
   - **GeoIP** : Enrichissement avec géolocalisation des adresses IP
   - **UserAgent** : Parsing des user agents HTTP
   - **Drop** : Suppression d'événements non pertinents
   - **JSON** : Parsing de données JSON

3. **Output (Destination/Sortie)** :
   - **Elasticsearch** : Envoi vers Elasticsearch pour indexation
   - **File** : Écriture dans des fichiers
   - **STDOUT** : Sortie console pour debug
   - **Graphite** : Envoi vers Graphite pour métriques
   - **MongoDB** : Stockage dans MongoDB

#### Kibana
- **Rôle** : Interface de visualisation et d'analyse
- **Fonction** : Dashboards, recherches, analyses
- **Fonctionnalités** :
  - Visualisations graphiques
  - Recherche avancée (KQL, Lucene)
  - Dashboards personnalisables
  - Alertes et monitoring

#### Beats

- **Rôle** : Famille d'agents légers de transfert de données (logs, métriques, audit) installés sur les sources
- **Types** :
  - **Filebeat** : Logs de fichiers
  - **Winlogbeat** : Événements Windows
  - **Metricbeat** : Métriques système
  - **Packetbeat** : Trafic réseau
  - **Auditbeat** : Audit système
  - **Heartbeat** : Monitoring de disponibilité

### 6.3 Architecture ELK

```
┌─────────┐     ┌──────────┐    ┌──────────┐
│ Filebeat│     │Winlogbeat│    │Metricbeat│
└────┬────┘     └────┬─────┘    └────┬─────┘
     │               │               │
     └───────────────┴───────────────┘
                     │
            ┌────────▼────────┐
            │    Logstash     │
            │  (optionnel)    │
            └────────┬────────┘
                     │
            ┌────────▼────────┐
            │  Elasticsearch  │
            └────────┬────────┘
                     │
            ┌────────▼────────┐
            │     Kibana      │
            └─────────────────┘
```

---

## 7. Investigation forensique et gestion des alertes

### 7.1 Bénéfices d'un SIEM pour l'investigation (forensics)

Un SIEM est un outil essentiel pour les investigations forensiques et la réponse aux incidents. Il apporte plusieurs bénéfices majeurs :

#### Timeline des événements
- **Reconstruction chronologique précise** de l'attaque pour comprendre l'enchaînement des actions malveillantes
- Visualisation temporelle des événements pour identifier le point d'entrée et la propagation

#### Préservation des preuves
- **Extraction et sécurisation des logs bruts** et artefacts numériques
- **Garantie de l'intégrité légale** : Les logs ne peuvent pas être altérés par un attaquant
- Traçabilité complète des actions pour les besoins juridiques

#### Pivot par entité
- **Analyse contextuelle** permettant de basculer rapidement entre différentes entités :
  - Adresses IP
  - Utilisateurs
  - Hôtes et systèmes
  - Hash de fichiers
- Corrélation automatique des événements liés à une même entité

#### Rapport d'incident
- **Documentation structurée** de l'incident incluant :
  - Vecteur initial d'attaque
  - Impact sur les systèmes et données
  - Données compromises
  - Timeline complète des événements

#### Recommandation de remédiation
- **Actions correctives concrètes** pour :
  - Bloquer la menace
  - Nettoyer les systèmes compromis
  - Prévenir la récurrence
- Génération automatique de playbooks de réponse

### 7.2 Ajustement des règles pour éviter les faux positifs

La gestion des faux positifs est cruciale pour maintenir l'efficacité d'un SIEM. Voici les éléments importants pour ajuster les règles :

#### Filtrer le bruit
- **Utiliser des listes autorisées** pour exclure les activités légitimes connues :
  - Scanners internes autorisés
  - Sauvegardes automatisées
  - Services système légitimes
  - Adresses IP whitelistées

#### Enrichir avec le contexte
- **Intégrer des informations contextuelles** pour différencier une anomalie réelle d'un comportement justifié :
  - **Heures** : Prendre en compte les horaires de travail
  - **Géolocalisation** : Identifier les connexions depuis des zones autorisées
  - **Rôles** : Considérer les permissions et rôles utilisateurs
  - **Contexte métier** : Activités normales selon le type d'organisation

#### Ajouter des seuils et fenêtres
- **Affiner les critères de déclenchement** :
  - Augmenter le nombre d'échecs requis avant alerte
  - Réduire ou élargir la fenêtre temporelle d'analyse
  - Combiner plusieurs conditions pour réduire les faux positifs
  - Utiliser des seuils adaptatifs basés sur l'historique

#### Test A/B
- **Tester les nouvelles règles en mode "silencieux"** avant production :
  - Comparer les résultats avec les règles existantes
  - Mesurer le taux de faux positifs
  - Ajuster avant activation
- **Réviser régulièrement** les règles existantes pour s'adapter aux changements

#### Boucle de rétroaction
- **Utiliser les retours des analystes** :
  - Marquer les faux positifs identifiés
  - Documenter les raisons des faux positifs
  - Améliorer la logique des règles basée sur ces retours
  - Créer un processus d'amélioration continue

---

*Document créé pour l'UE S10-3 - Sécurité des informations et management des événements*  
*École ESAIP - Ingénieur Informatique et Réseaux*
