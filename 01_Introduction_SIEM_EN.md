# Introduction to SIEM
## Information Security and Event Management

**Course Unit S10-3 Systems & Networks**  
**Supervised hours: 21 h | Total workload: 40 h**

---

## 1. Context and Challenges

In a modern IT environment, organizations generate **millions of events** every day:
- User connections
- Resource access
- Intrusion attempts
- System errors
- Configuration changes
- Network activities

**The challenge**: How to detect a **real threat** among this continuous flow of information?

Modern attacks are:
- **Sophisticated**: Multi-step, distributed, stealthy
- **Fast**: Real-time detection and response required
- **Complex**: Require correlation of multiple sources

**SIEM** (Security Information and Event Management) is the solution to:
- **Centralize** all security events
- **Correlate** events to detect threats
- **Analyze** attack patterns
- **Respond** quickly to incidents

---

## 2. SIEM Definition

### 2.1 What is a SIEM?

**SIEM** = **Security Information and Event Management**

A SIEM is a solution that:
1. **Collects** logs and security events from various sources
2. **Normalizes** and **stores** this data in a unified format
3. **Correlates** events to identify threats
4. **Analyzes** patterns and suspicious behaviors
5. **Alerts** security teams in real-time
6. **Visualizes** data to facilitate investigation

#### What is a SIEM used for?

A SIEM enables **ingestion of large volumes of data** (logs) to analyze and present them in the form of **dashboards** and **alerts** to quickly handle an incident (attack/failure).

The main benefits are:
- **Centralization**: Everything grouped in a single search point
- **Preserved history**: Logs cannot be altered by an attacker
- **Real-time analysis**: Rapid threat detection
- **Investigation**: Facilitates forensic analysis and incident response

### 2.2 Key Components

A modern SIEM typically includes:

- **Collectors**: Agents that retrieve logs (Beats, agents, syslog)
- **Correlation engine**: Analyzes events and detects patterns
- **Database**: Storage of events (Elasticsearch, database)
- **Visualization interface**: Dashboards and analysis (Kibana, web interface)
- **Alert engine**: Alert and notification generation

---

## 3. Learning Objectives

### 3.1 Skills Objectives

Upon completion of this course, you will be able to:

-  **Deploy a complete SIEM solution**
-  **Centralize and manage logs** from IDS/IPS and operational equipment
-  **Detect threats** among large volumes of information
-  **Create visualizations** with loaded data using Kibana
-  **Analyze data in real-time** with the ELK stack

### 3.2 Knowledge Objectives

You will know:

-  The **key characteristics of SIEMs** and market solutions
-  **Defensive technologies** around SIEM terminology
-  How a **SIEM solution works** and its advantages
-  The **fundamental principles of the ELK stack** and use cases

---

## 4. SIEM Architecture

### 4.1 General Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Data Sources                         │
├──────────┬──────────┬──────────┬──────────┬─────────────┤
│  IDS/IPS │  Firewall│  Servers │  Windows │  Network    │
│          │          │  Linux   │  Events  │  Appliances │
└────┬─────┴────┬─────┴────┬─────┴────┬─────┴─────┬───────┘
     │          │          │          │           │
     └──────────┴──────────┴──────────┴───────────┘
                    │
         ┌──────────▼──────────┐
         │  Agents/Collectors  │
         │ (Beats, Syslog, etc)│
         └──────────┬──────────┘
                    │
         ┌──────────▼───────────┐
         │   SIEM Engine         │
         │  ┌────────────────┐  │
         │  │ Normalization  │  │
         │  │ Correlation    │  │
         │  │ Analysis       │  │
         │  └────────────────┘  │
         └──────────┬───────────┘
                    │
     ┌──────────────┴──────────────┐
     │                              │
┌────▼─────┐              ┌─────────▼────────┐
│ Storage │              │  Visualization    │
│(Elastic)│              │     (Kibana)      │
└────┬─────┘              └──────────────────┘
     │
┌────▼─────┐
│ Alerts  │
│& Response│
└──────────┘
```

### 4.2 Data Processing Chain

The data processing chain in a SIEM follows a structured flow:

1. **Data Sources**: 
   - System logs (Linux/Windows servers)
   - IDS/IPS (Intrusion Detection/Prevention Systems)
   - Network equipment (firewalls, routers, switches)
   - Applications and services

2. **Collection Tools**:
   - **Beats**: Lightweight agents (Filebeat, Winlogbeat, Metricbeat)
   - **API**: Programming interfaces for log retrieval
   - **Agents**: Dedicated agents installed on sources
   - **Syslog**: Standardized protocol for log collection

3. **Log Processing**:
   - **Collection**: Retrieval of logs on a server separate from the source
   - **Normalization**: Conversion to standardized format
   - **Enrichment**: Addition of context (geolocation, IP reputation, etc.)
   - **Correlation**: Analysis of relationships between events
   - **Analysis**: Detection of patterns and suspicious behaviors

4. **Storage**:
   - Short-term archiving (hot data)
   - Long-term archiving (cold data)
   - Data lifecycle management

5. **Visualization**:
   - **Dashboards**: Customizable dashboards
   - **Search index**: Advanced search in logs
   - **Alerts**: Real-time notifications when threats are detected

### 4.3 Deployment Strategies

The choice of SIEM deployment mode is crucial and depends on organizational, technical, and regulatory constraints.

#### On-premise Deployment

**Advantages**:
- **Total data control**: Sovereignty and complete control
- **Strict compliance facilitated**: Respect for local regulations
- **No WAN network latency**: Optimal performance

**Disadvantages**:
- **High initial costs (CapEx)**: Significant hardware investment
- **Complex hardware maintenance**: Requires a dedicated team
- **Scalability limited to hardware**: Expensive and complex expansion

#### Cloud Deployment

**Advantages**:
- **Ultra-fast deployment**: Service activation in a few hours
- **Automatic elasticity and scalability**: Automatic adaptation to load
- **Maintenance managed by provider**: Reduced operational burden

**Disadvantages**:
- **Sometimes unpredictable variable costs**: Billing based on usage
- **Internet/provider dependency**: Risk of outage or policy change
- **Data confidentiality concerns**: Data hosted by a third party

#### Hybrid Deployment

**Advantages**:
- **Optimal flexibility**: Best of both worlds
- **Sensitive data on-premise**: Enhanced security for critical data
- **Progressive transition possible**: Smooth migration to cloud

**Disadvantages**:
- **More complex architecture**: Management of two environments
- **Unified management difficult**: Requires coordination tools
- **Latency between components**: Potentially impacted performance

---

## 5. Market Solutions

### 5.1 Open-source Solutions

#### ELK Stack (Elastic Stack)
- **Elasticsearch**: Search engine and storage
- **Logstash**: Log processing and transformation
- **Kibana**: Visualization interface
- **Beats**: Lightweight collection agents
- **Advantages**: Free, flexible, active community
- **Disadvantages**: Requires technical expertise

#### Wazuh
- Open-source SIEM based on OSSEC
- Integration with ELK Stack
- Good for small/medium organizations

### 5.2 Commercial Solutions

#### Splunk
- Market leader
- Intuitive interface
- High cost based on data volume

#### IBM QRadar
- Complete enterprise solution
- Good integration with IBM ecosystem

#### ArcSight (Micro Focus)
- Mature and robust solution
- Used by large organizations

#### Sentinel (Microsoft)
- Cloud-native solution
- Integration with Microsoft ecosystem

### 5.3 SIEM Solution Selection Factors

The choice of a SIEM solution and its hosting mode must consider several critical factors:

#### Volume and Scalability
- **Estimate log volume**: Measure EPS (Events Per Second) and GB per day
- **Growth capacity**: Verify that the solution can evolve with organizational growth
- **Performance**: Ensure the solution can process expected volume in real-time

#### Expertise and Complexity
- **Assess team expertise**: 
  - Turnkey solution (commercial) for less technical teams
  - Flexible solution (open-source) for experienced teams
- **Learning curve**: Time needed to master the solution
- **Available support**: Documentation, community, commercial support

#### Costs
- **Calculate total cost over 3-5 years**:
  - Licenses and subscriptions
  - Infrastructure (hardware, cloud)
  - Team training
  - Maintenance and support
- **Billing model**: By volume, by user, flat rate
- **Hidden costs**: Storage, bandwidth, integrations

#### Integration and Compatibility
- **Verify compatibility** with existing tools:
  - EDR (Endpoint Detection and Response)
  - Cloud solutions (AWS, Azure, GCP)
  - Firewalls and network equipment
  - Operating systems
- **Richness of plugins and connectors**: Ease of integration
- **Supported standards**: Syslog, CEF, JSON, etc.

---

## 6. The ELK Stack (Elastic Stack)

### 6.1 Overview

**ELK** = **E**lasticsearch + **L**ogstash + **K**ibana

Now called **Elastic Stack**, it also includes **Beats**.

### 6.2 Components

#### Elasticsearch

- **Role**: Distributed search and analytics engine, NoSQL database for storage and indexing
- **Function**: Log storage and indexing
- **Characteristics**: Distributed, scalable, fast search
- **API**: RESTful for interaction

##### Why is Elasticsearch Perfect for a SIEM?

Elasticsearch has several characteristics that make it an ideal solution for SIEM needs:

- **Document-oriented database (JSON)**: Flexible format without rigid schema, adapted to the variety of log formats
- **Horizontal scalability**: Designed to scale by adding nodes, essential for managing large volumes
- **Integrated Lucene engine**: Ultra-fast indexing and near-instant full-text search on millions of documents
- **Powerful analytical capabilities**: Calculate statistics and metrics on large data volumes in real-time
- **Automatic distribution**: Data is distributed in shards (fragments) and replicas for performance and fault tolerance
- **Standard JSON interface via HTTP**: Simple and universal REST API
- **Automated lifecycle management**: Support for Hot/Warm/Cold/Frozen indexes to optimize storage costs

#### Logstash

- **Role**: Server-side data processing pipeline (ETL) to ingest, transform, and enrich logs
- **Function**: Collection, transformation, log enrichment

##### The 3 Steps of Logstash Processing

Logstash operates according to a three-step pipeline:

1. **Input (Source Retrieval)**:
   - **Files**: Reading log files
   - **Beats**: Reception from Beats agents
   - **Syslog**: Standard syslog protocol
   - **HTTP**: Reception via REST API
   - **TCP/UDP**: Network flow reception
   - **Kafka**: Integration with Apache Kafka
   - **JDBC**: Database connections

2. **Filter (Transformation, Normalization, and Enrichment)**:
   - **Grok**: Parsing unstructured logs with regular expressions
   - **Date**: Date parsing and normalization
   - **Mutate**: Field modification (renaming, conversion, deletion)
   - **GeoIP**: Enrichment with IP address geolocation
   - **UserAgent**: HTTP user agent parsing
   - **Drop**: Removal of irrelevant events
   - **JSON**: JSON data parsing

3. **Output (Destination/Output)**:
   - **Elasticsearch**: Sending to Elasticsearch for indexing
   - **File**: Writing to files
   - **STDOUT**: Console output for debugging
   - **Graphite**: Sending to Graphite for metrics
   - **MongoDB**: Storage in MongoDB

#### Kibana
- **Role**: Visualization and analysis interface
- **Function**: Dashboards, searches, analysis
- **Features**:
  - Graphical visualizations
  - Advanced search (KQL, Lucene)
  - Customizable dashboards
  - Alerts and monitoring

#### Beats

- **Role**: Family of lightweight data transfer agents (logs, metrics, audit) installed on sources
- **Types**:
  - **Filebeat**: File logs
  - **Winlogbeat**: Windows events
  - **Metricbeat**: System metrics
  - **Packetbeat**: Network traffic
  - **Auditbeat**: System audit
  - **Heartbeat**: Availability monitoring

### 6.3 ELK Architecture

```
┌─────────┐     ┌──────────┐    ┌──────────┐
│ Filebeat│     │Winlogbeat│    │Metricbeat│
└────┬────┘     └────┬─────┘    └────┬─────┘
     │               │               │
     └───────────────┴───────────────┘
                     │
            ┌────────▼────────┐
            │    Logstash     │
            │  (optional)     │
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

## 7. Forensic Investigation and Alert Management

### 7.1 SIEM Benefits for Investigation (Forensics)

A SIEM is an essential tool for forensic investigations and incident response. It provides several major benefits:

#### Event Timeline
- **Precise chronological reconstruction** of the attack to understand the sequence of malicious actions
- Temporal visualization of events to identify the entry point and propagation

#### Evidence Preservation
- **Extraction and securing of raw logs** and digital artifacts
- **Legal integrity guarantee**: Logs cannot be altered by an attacker
- Complete traceability of actions for legal needs

#### Entity Pivot
- **Contextual analysis** allowing quick switching between different entities:
  - IP addresses
  - Users
  - Hosts and systems
  - File hashes
- Automatic correlation of events related to the same entity

#### Incident Report
- **Structured documentation** of the incident including:
  - Initial attack vector
  - Impact on systems and data
  - Compromised data
  - Complete event timeline

#### Remediation Recommendations
- **Concrete corrective actions** to:
  - Block the threat
  - Clean compromised systems
  - Prevent recurrence
- Automatic generation of response playbooks

### 7.2 Rule Tuning to Avoid False Positives

False positive management is crucial to maintain SIEM effectiveness. Here are the important elements for tuning rules:

#### Filter Noise
- **Use allowlists** to exclude known legitimate activities:
  - Authorized internal scanners
  - Automated backups
  - Legitimate system services
  - Whitelisted IP addresses

#### Enrich with Context
- **Integrate contextual information** to differentiate a real anomaly from justified behavior:
  - **Hours**: Consider working hours
  - **Geolocation**: Identify connections from authorized zones
  - **Roles**: Consider user permissions and roles
  - **Business context**: Normal activities according to organization type

#### Add Thresholds and Windows
- **Refine trigger criteria**:
  - Increase the number of failures required before alert
  - Reduce or widen the temporal analysis window
  - Combine multiple conditions to reduce false positives
  - Use adaptive thresholds based on history

#### A/B Testing
- **Test new rules in "silent" mode** before production:
  - Compare results with existing rules
  - Measure false positive rate
  - Adjust before activation
- **Regularly review** existing rules to adapt to changes

#### Feedback Loop
- **Use analyst feedback**:
  - Mark identified false positives
  - Document reasons for false positives
  - Improve rule logic based on this feedback
  - Create a continuous improvement process

---

*Document created for Course Unit S10-3 - Information Security and Event Management*  
*ESAIP School - Computer Science and Networks Engineering*
