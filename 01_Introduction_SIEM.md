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

### 4.2 Flux de données

1. **Collection** : Les agents collectent les logs depuis les sources
2. **Normalisation** : Les logs sont convertis en format standardisé
3. **Enrichissement** : Ajout de contexte (géolocalisation, réputation IP, etc.)
4. **Corrélation** : Analyse des relations entre événements
5. **Stockage** : Archivage pour analyse historique
6. **Visualisation** : Présentation dans des dashboards
7. **Alerte** : Notification en cas de détection de menace

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

---

## 6. La pile ELK (Elastic Stack)

### 6.1 Présentation

**ELK** = **E**lasticsearch + **L**ogstash + **K**ibana

Aujourd'hui appelée **Elastic Stack**, elle inclut également **Beats**.

### 6.2 Composants

#### Elasticsearch
- **Rôle** : Moteur de recherche et base de données NoSQL
- **Fonction** : Stockage et indexation des logs
- **Caractéristiques** : Distribué, scalable, recherche rapide
- **API** : RESTful pour l'interaction

#### Logstash
- **Rôle** : Pipeline de traitement de données
- **Fonction** : Collecte, transformation, enrichissement des logs
- **Composants** :
  - **Input** : Sources de données (fichiers, syslog, Beats, etc.)
  - **Filter** : Transformation et enrichissement (parsing, géolocalisation)
  - **Output** : Destination (Elasticsearch, fichiers, etc.)

#### Kibana
- **Rôle** : Interface de visualisation et d'analyse
- **Fonction** : Dashboards, recherches, analyses
- **Fonctionnalités** :
  - Visualisations graphiques
  - Recherche avancée (KQL, Lucene)
  - Dashboards personnalisables
  - Alertes et monitoring

#### Beats
- **Rôle** : Agents légers de collecte
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

## 7. Plan du cours

### Séance 1 : Introduction au SIEM
- Concepts fondamentaux
- Architecture et composants
- Solutions du marché

### Séance 2 : Architecture et déploiement SIEM
- Architecture SIEM
- Stratégies de déploiement
- Journaux et événements

### Séance 3 : Collection et corrélation d'événements
- Collection d'événements
- Corrélation d'événements
- Règles de corrélation

### Séance 4 : Données et investigation numérique
- Forensic Data
- Analyse post-incident
- Conservation des données

### Séance 5 : Détection et prévention
- Détection d'intrusions
- Prévention d'intrusions
- Tolérance aux intrusions

### Séance 6-7 : TP - Installation ELK
- Installation de la stack ELK
- Configuration de base
- Premiers tests

### Séance 8-9 : TP - Agents Beats
- Configuration de Filebeat
- Configuration de Winlogbeat
- Configuration de Metricbeat

### Séance 10-11 : TP - Visualisation Kibana
- Création de visualisations
- Création de dashboards
- Analyse de données

### Séance 12-13 : Configuration avancée Logstash
- Pipelines complexes
- Filtres avancés
- Enrichissement de données

### Séance 14 : Introduction à Splunk
- Présentation de Splunk
- Principes de fonctionnement
- Création de rapports et visualisations

### Séance 15 : Cas pratiques et synthèse
- Scénarios d'attaque
- Analyse de logs réels
- Synthèse du cours

---

## 8. Évaluation

### Modalités

L'évaluation se fait en **contrôle continu** :

- **Tests écrits courts** (QCM/QCU)
- **Devoirs maison**
- **Exercices en temps limité**
- **Projet pratique** sur ELK

### Coefficients

- Aucune évaluation ne peut dépasser **50%** du poids final
- Répartition équilibrée entre les différents types d'évaluation

### Exercices d'entraînement

- Exercices répétitifs sur Moodle
- Auto-évaluation et correction par les pairs
- Correction en classe avec explications

---

## 9. Prérequis

Pour suivre ce cours, vous devez maîtriser :

- **Réseaux TCP/IP** : Protocoles, adressage, routage
- **Systèmes** : Linux, Windows, administration système
- **Sécurité Informatique - CEH** : Concepts de base en cybersécurité

---

## 10. Bibliographie

### Ouvrages de référence

1. **Applied Network Security Monitoring: Collection, Detection, and Analysis**
   - Auteurs : Chris Sanders et Jason Smith
   - Focus : NSM et analyse de sécurité

2. **Blue Team Handbook: Incident Response Edition**
   - Auteur : Don Murdoch GSE
   - Focus : Guide pratique pour les équipes de sécurité

3. **Digital Forensics and Incident Response**
   - Auteur : Gerard Johansen
   - Focus : Techniques de forensic et réponse aux incidents

### Ressources en ligne

- Documentation officielle Elastic : https://www.elastic.co/guide/
- Documentation Splunk : https://docs.splunk.com/
- Communautés et forums spécialisés

---

## 11. Conclusion

Le SIEM est un **outil essentiel** pour la sécurité moderne des systèmes d'information. Il permet de :

- **Voir** ce qui se passe dans votre infrastructure
- **Détecter** les menaces en temps réel
- **Analyser** les incidents de sécurité
- **Répondre** rapidement aux attaques

Au cours de ce module, vous apprendrez à :
- Installer et configurer une solution SIEM complète
- Collecter et analyser les événements de sécurité
- Créer des visualisations et des dashboards
- Détecter et analyser les menaces


---

*Document créé pour l'UE S10-3 - Sécurité des informations et management des événements*  
*École ESAIP - Ingénieur Informatique et Réseaux*
