## TP: Implementing a SIEM Solution with the ELK Stack

**UE S10-3 - Information Security and Event Management** **Practical Work - Individual Assignment**

---

## Table of Contents

1. [Context and Objectives](https://www.google.com/search?q=%231-context-and-objectives)
2. [Part 1: ELK Stack Installation](https://www.google.com/search?q=%232-part-1--elk-stack-installation)
3. [Part 2: Configuring Beats Agents](https://www.google.com/search?q=%233-part-2--configuring-beats-agents)
4. [Part 3: Visualization and Analysis in Kibana](https://www.google.com/search?q=%234-part-3--visualization-and-analysis-in-kibana)
5. [Part 4: Practical Use Case](https://www.google.com/search?q=%235-part-4--practical-use-case)
6. [Appendices](https://www.google.com/search?q=%236-appendices)

---

## 1. Context and Objectives

### 1.1 TP Overview

This practical work (TP) allows you to apply the concepts learned in class by installing and configuring a complete SIEM solution based on the **ELK Stack** (Elasticsearch, Logstash, Kibana).

You will:

* Install and configure the ELK Stack on a Debian machine
* Configure collection agents (Beats) on different systems
* Create visualizations and dashboards in Kibana
* Analyze real security events

### 1.2 Learning Objectives

Upon completion of this TP, you will be able to:

* Install and configure Elasticsearch, Logstash, and Kibana
* Optimize Elasticsearch to run with limited resources
* Configure Filebeat to collect system logs
* Configure Winlogbeat to collect Windows events
* Configure Metricbeat to collect system metrics
* Create visualizations and dashboards in Kibana
* Analyze logs to detect suspicious activity

### 1.3 Technical Environment

#### ELK Server Machine

* **OS**: Debian 13 (Bookworm)
* **CPU**: 1 core
* **RAM**: 2 GB
* **Network**: Accessible from the ESAIP network
* **Access**: SSH from your PC

#### Client Machine (Your PC)

* **OS**: Windows
* **Network**: Same network as the VM (ESAIP)
* **Access**: Web browser for Kibana

#### Network Architecture

```
┌─────────────────┐         ┌──────────────────┐
│   Windows PC    │         │  Debian 13 VM    │
│                 │         │                  │
│  - Winlogbeat   │───────▶│  - Elasticsearch │
│  - Browser      │         │  - Logstash      │
│                 │         │  - Kibana        │
│                 │         │  - Filebeat      │
│                 │         │  - Metricbeat    │
└─────────────────┘         └──────────────────┘
         │                            │
         └─────────── ESAIP Network ───┘

```

### 1.4 Prerequisites

Before starting, ensure you have:

* SSH access to your Debian VM
* Administrator rights (sudo) on the VM
* Network access between your PC and the VM
* A modern web browser (Chrome, Firefox, Edge)
* Basic knowledge of Linux (commands, file editing)

### 1.5 Estimated Duration

* **Part 1**: 2-3 hours
* **Part 2**: 2-3 hours
* **Part 3**: 1-2 hours
* **Part 4**: 1-2 hours

**Total**: 6-10 hours of work

---

## 2. Part 1: ELK Stack Installation

### 2.1 Prerequisites and Environment Check

#### Step 1: Connect to the VM

Connect to your Debian VM via SSH:

```bash
ssh esaip@VM_IP

```

Replace `VM_IP` with your IP address indicated on the [proxmox](https://10.3.2.10:8006) page.

#### Step 2: System Verification

Check your system information:

```bash
# Debian version
cat /etc/debian_version

# System information
uname -a

# Available memory
free -h

# Disk space
df -h

```

**Verification**: Note the available RAM. With 2 GB of RAM, we will need to optimize Elasticsearch to leave memory for other services.

#### Step 3: System Update

Update system packages:

```bash
sudo apt update
sudo apt upgrade -y

```

#### Step 4: Installation of Basic Tools

Install necessary tools:

```bash
sudo apt install -y curl wget nano git gpg

```

---

### 2.2 Elasticsearch Installation

If you encounter any issues with the installation, please check if the procedure has changed on the [elastic doc](https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-elasticsearch-with-debian-package).

#### Step 1: Add the Elastic GPG Key

Add the GPG key to authenticate Elastic packages:

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

```

#### Step 2: Install apt-transport-https

Install HTTPS support for apt:

```bash
sudo apt install -y apt-transport-https

```

#### Step 3: Add the Elastic Repository

Add the Elastic repository (we will use version 9.x):

```bash
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/9.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-9.x.list

```

#### Step 4: Update and Installation

Update repositories and install Elasticsearch:

```bash
sudo apt-get update && sudo apt-get install elasticsearch

```

#### Step 5: Configure Elasticsearch for 2 GB of RAM

**IMPORTANT**: With 2 GB of RAM, we must limit the Elasticsearch heap memory to leave memory for other services.

Create the heap memory configuration file for the JVM:

```bash
sudo nano /etc/elasticsearch/jvm.options.d/heapsizemem.options

```

**Note**: The file extension must be `.options` (not `.conf`) to be recognized by Elasticsearch.

Insert the following lines:

```properties
# Optimized for 2 GB RAM (heap limited to 512 MB)
-Xms512m
-Xmx512m

```

**Explanation**:

* We limit the **heap** to 1 GB to leave enough memory for the system and other services (Kibana, Logstash, etc.).
* With a total of 2 GB of RAM, a 512m heap is a good compromise.
* **Important**: The total memory used by Elasticsearch will be greater than the heap (approximately 1 GB) because it includes:
* The JVM heap (512m in our case)
* Native memory (off-heap)
* Shared libraries
* Auxiliary processes (like the ML controller)
* System buffers



#### Step 6: Elasticsearch Network Configuration

Edit the main configuration file:

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml

```

**Configuration for Elasticsearch 9.2**:

The default configuration for Elasticsearch 9.2 already includes some options. You must check and modify the following settings:

```yaml
# Network and HTTP (already present in the default config with http.host)
# Verify that these lines are present:
http.host: 0.0.0.0
http.port: 9200

# Cluster name (optional, for identification)
cluster.name: siem-cluster

node.name: siem-node-1

# Disable SSL/HTTPS for simplicity (necessary because security is disabled) (**do not do this in production**)
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

**Important Notes**:

* In production, you should enable security and use SSL/TLS.
* In Elasticsearch 9.2, use `http.host` instead of `network.host`.
* Replace `"siem-node-1"` in `cluster.initial_master_nodes` with the value of `node.name` you defined (or keep your machine name if you haven't changed `node.name`).

#### Step 7: Start Elasticsearch

Start the Elasticsearch service:

```bash
sudo systemctl start elasticsearch

```

Enable automatic startup:

```bash
sudo systemctl enable elasticsearch

```

Check the status:

```bash
sudo systemctl status elasticsearch

```

#### Step 8: Verification of Operation

Wait 30-60 seconds for Elasticsearch to start, then test:

```bash
curl http://localhost:9200

```

You should see a JSON response with cluster information.

**Verification**: After restarting, verify that the heap is properly limited with:

```bash
curl http://localhost:9200/_nodes/jvm?pretty

```

Look for the `heap_used` and `heap_max` values in the response.

Also test from your Windows PC (replace `VM_IP` with your VM's IP):

```bash
# From PowerShell or CMD
curl http://VM_IP:9200

```

**Verification**: If you get a JSON response, Elasticsearch is working correctly.

#### Help: Troubleshooting a Memory Issue

**Situation**: Elasticsearch does not start or crashes with a memory error.

**Procedure**:

1. Check logs: `sudo journalctl -u elasticsearch -n 50`
2. Identify the memory-related error
3. Adjust the `-Xms` and `-Xmx` parameters in `/etc/elasticsearch/jvm.options`
4. Restart Elasticsearch: `sudo systemctl restart elasticsearch`

**Hint**: With 2 GB of RAM, a 1 GB heap should work. If you have issues, you can reduce it to 512m, but this may impact performance.

---

### 2.3 Kibana Installation

#### Step 1: Installation

Kibana should already be available in the Elastic repository. Install it:

```bash
sudo apt install -y kibana

```

#### Step 2: Kibana Configuration

Edit the configuration file:

```bash
sudo nano /etc/kibana/kibana.yml

```

Modify the following parameters:

```yaml
# Listening address
server.host: "0.0.0.0"

# Port
server.port: 5601

# Elasticsearch URL
elasticsearch.hosts: ["http://localhost:9200"]
node.options: "--max-old-space-size=128"

```

#### Step 3: Start Kibana

Start the service:

```bash
sudo systemctl start kibana
sudo systemctl enable kibana

```

Check the status:

```bash
sudo systemctl status kibana

```

**Important Note**:

* Kibana can take **1-2 minutes** to start completely.
* **During the first startup**, Kibana performs saved object migrations which can take an additional **3-5 minutes**.
* During this time, the web page may remain loading. This is **normal**.
* Monitor the logs to see the migration status:
```bash
sudo journalctl -u kibana -f

```


You should see "Starting saved objects migrations" then "Migration completed" or similar.

#### Step 4: Access Kibana

Open your web browser and access:

```
http://VM_IP:5601

```

Replace `VM_IP` with your VM's IP address.

**Verification**: You should see the Kibana home page.

#### Help: Configuring Access from Windows

**Situation**: You cannot access Kibana from your Windows PC.

**Procedure**:

1. Verify that Kibana is listening on all interfaces: `sudo netstat -tlnp | grep 5601`
2. Check the VM's firewall: `sudo ufw status`
3. If the firewall is blocking, allow the port: `sudo ufw allow 5601/tcp`
4. Check connectivity from Windows: `ping VM_IP`
5. Test from Windows: `curl http://VM_IP:5601` or open in the browser

**Hint**: If you are using a firewall, you may also need to open port 9200 for Elasticsearch.

---

### 2.4 Logstash Installation

#### Step 1: Install Java

Logstash requires Java. Install OpenJDK:

```bash
sudo apt install -y default-jre

```

Check the installation:

```bash
java -version

```

---

#### Step 2: Install Logstash

Install Logstash:

```bash
sudo apt update
sudo apt install -y logstash

```

---

#### Step 3: Logstash Basic Configuration

Edit the main configuration file:

```bash
sudo nano /etc/logstash/logstash.yml

```

Check/configure the following parameters:

```yaml
# Pipeline path
path.config: /etc/logstash/conf.d

# Logs path
path.logs: /var/log/logstash

# Internal data path (important to avoid errors)
path.data: /var/lib/logstash

```

Create and correct necessary permissions (prevents 90% of errors):

```bash
sudo mkdir -p /var/log/logstash /var/lib/logstash
sudo chown -R logstash:logstash /var/log/logstash /var/lib/logstash

```
**IMPORTANT**: With 4 GB of RAM, we must limit the Elasticsearch heap memory to leave memory for other services.

Edit the heap memory configuration file for the JVM:

```bash
sudo nano /etc/logstash/jvm.options
```
Modify the following lines:

```properties
# Optimized for 4 GB RAM (heap limited to 256 MB)
-Xms256m
-Xmx256m

```

---

#### Step 4: Create a Test Pipeline (Service Mode Compatible)

**Important**:
The `stdin {}` pipeline **should not be used with systemd**, as it causes Logstash to shut down immediately.
We will therefore use a **file input** for a real test.

Create the test pipeline:

```bash
sudo nano /etc/logstash/conf.d/test.conf

```

Configuration:

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
  # No filter for the test
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

Create the test file:

```bash
sudo touch /tmp/logstash-test.log
sudo chmod 666 /tmp/logstash-test.log

```

---

#### Step 5: Start Logstash

Start Logstash as a service:

```bash
sudo systemctl enable logstash
sudo systemctl restart logstash

```

Check the status:

```bash
sudo systemctl status logstash

```

You should see:

```
Active: active (running)

```

---

#### Step 6: Test the Pipeline

In another terminal, write to the test file:

```bash
echo "Hello Logstash" >> /tmp/logstash-test.log
echo "Second message" >> /tmp/logstash-test.log

```

Expected result:

* Logs appear in Elasticsearch (Go to **elasticsearch → index_management**)
* The `test-logs-YYYY.MM.dd` index is created
* Logstash **remains active**

---

#### Pipeline for Apache Logs

**Objective**

Create a Logstash pipeline that reads and parses Apache logs.

---

##### Step 1: Install Apache

```bash
sudo apt install -y apache2

```

Verify that Apache is working:

```bash
curl http://localhost

```

---

##### Step 2: Create the Apache Pipeline

Create the file:

```bash
sudo nano /etc/logstash/conf.d/apache.conf

```

Configuration:

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

**Important**:
Remove or comment out `test.conf` to avoid unnecessary multiple pipelines:

```bash
sudo mv /etc/logstash/conf.d/test.conf /etc/logstash/conf.d/test.conf.disabled

```

Allow logstash to read the Apache log file:

```bash
sudo usermod -aG adm logstash

```

---

##### Step 3: Restart Logstash

```bash
sudo systemctl restart logstash

```

Verification:

```bash
journalctl -u logstash -n 20 --no-pager

```

You should see:

* `Pipeline started`
* no fatal errors

---

##### Step 4: Generate Apache Traffic

```bash
curl http://localhost
curl http://localhost
curl http://localhost

```

---

##### Step 5: Verification in Kibana

1. Go to **Stack Management → Data Views**
2. Create a data view:
```
apache-logs-*

```


3. Time field: `@timestamp`
4. Go to **Discover**

Expected result:

* Apache requests visible
* Parsed fields (`clientip`, `verb`, `request`, `response`, etc.)

---

**Hint**

Grok Pattern Used:

```
%{COMBINEDAPACHELOG}

```

Official documentation: [grok](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html)

---

## 3. Part 2: Configuring Beats Agents

### 3.1 Filebeat Installation and Configuration

#### Step 1: Install Filebeat and rsyslog

Since Debian 8, journald is used to log system events, which makes capture by filebeat or logstash impossible.
Therefore, rsyslog needs to be reactivated, like the default configuration of Ubuntu.

Install and enable rsyslog

```bash
apt install rsyslog
systemctl enable --now rsyslog

```

Install Filebeat on the Debian VM:

```bash
sudo apt install -y filebeat

```

#### Step 2: Basic Configuration

Edit the configuration file:

```bash
sudo nano /etc/filebeat/filebeat.yml

```

Configure the `output.elasticsearch` section:

```yaml
output.elasticsearch:
  hosts: ["localhost:9200"]
  # Disable security if Elasticsearch has no security
  # username: "elastic"
  # password: "changeme"

```

#### Step 3: Enable System Module

Enable the system module to collect system logs:

```bash
sudo filebeat modules enable system

```

#### Step 4: System Module Configuration

Edit the module configuration:

```bash
sudo nano /etc/filebeat/modules.d/system.yml

```

The default configuration should collect:

* `/var/log/auth.log` (authentications)
* `/var/log/syslog` (system logs)
* `/var/log/*.log` (other logs)

#### Step 5: Start Filebeat

Start Filebeat:

```bash
sudo systemctl start filebeat
sudo systemctl enable filebeat

```

Check the status:

```bash
sudo systemctl status filebeat

```

#### Step 6: Verification in Kibana

1. Access Kibana: `http://VM_IP:5601`
2. Go to **Stack Management → Data Views**
3. Create a data view: `filebeat-*`
4. Select the timestamp field: `@timestamp`
5. Explore the data in **Discover**

**Verification**: You should see system logs in Kibana.

#### Configuration for Apache Logs

**Objective**: Configure Filebeat to specifically collect Apache logs.

**Task**:

1. Enable the Apache module: `sudo filebeat modules enable apache`
2. Edit the configuration: `sudo nano /etc/filebeat/modules.d/apache.yml`
3. Configure the Apache log paths (usually `/var/log/apache2/access.log` and `/var/log/apache2/error.log`)
4. Restart Filebeat: `sudo systemctl restart filebeat`
5. Generate Apache traffic: `curl http://localhost` (several times)
6. Verify in Kibana that Apache logs appear with the index `filebeat-*`

**Hint**: You can also create a custom configuration in `filebeat.yml` with a `filebeat.inputs` section.

---

### 3.2 Winlogbeat Installation and Configuration

#### Step 1: Download Winlogbeat

On your Windows PC, download Winlogbeat from the Elastic website:

1. Go to: [https://www.elastic.co/downloads/beats/winlogbeat](https://www.elastic.co/downloads/beats/winlogbeat)
2. Download the Windows version (ZIP)
3. Extract the archive to a folder (e.g., `C:\Program Files\Winlogbeat`)

**Alternative**: Use PowerShell to download:

```powershell
# In PowerShell (as administrator)
Invoke-WebRequest -Uri "https://artifacts.elastic.co/downloads/beats/winlogbeat/winlogbeat-8.11.0-windows-x86_64.zip" -OutFile "winlogbeat.zip"
Expand-Archive winlogbeat.zip -DestinationPath "C:\Program Files\"

```

#### Step 2: Winlogbeat Configuration

Edit the configuration file:

```powershell
notepad "C:\Program Files\Winlogbeat\winlogbeat.yml"

```

Configure the `output.elasticsearch` section:

```yaml
output.elasticsearch:
  hosts: ["http://VM_IP:9200"]
  # Replace VM_IP with your VM's IP address
  # Disable security if Elasticsearch has no security
  # username: "elastic"
  # password: "changeme"

```

#### Step 3: Event Collection Configuration

In the same file, configure the Windows events to collect:

```yaml
winlogbeat.event_logs:
  - name: Application
    ignore_older: 72h
  - name: System
    ignore_older: 72h
  - name: Security
    ignore_older: 72h

```

#### Step 4: Install Winlogbeat as a Service

Install Winlogbeat as a Windows service:

```powershell
# In PowerShell (as administrator)
cd "C:\Program Files\Winlogbeat"
.\install-service-winlogbeat.ps1

```

If the script does not exist, use this command:

```powershell
New-Service -Name "winlogbeat" -BinaryPathName "C:\Program Files\Winlogbeat\winlogbeat.exe -c C:\Program Files\Winlogbeat\winlogbeat.yml -path.home C:\Program Files\Winlogbeat" -StartupType Automatic

```

#### Step 5: Start the Service

Start the Winlogbeat service:

```powershell
Start-Service winlogbeat

```

Check the status:

```powershell
Get-Service winlogbeat

```

#### Step 6: Verification in Kibana

1. Wait a few minutes for events to be collected
2. In Kibana, create a data view: `winlogbeat-*`
3. Explore the Windows events in **Discover**

**Verification**: You should see Windows events (Application, System, Security) in Kibana.

#### Filtering Critical Events

**Objective**: Configure Winlogbeat to collect only critical security events.

**Task**:

1. Edit `winlogbeat.yml`
2. In the `winlogbeat.event_logs` section, configure filters for the Security log
3. Filter by EventID to collect only critical events:
* 4624: Successful login
* 4625: Failed login
* 4648: Login with explicit credentials
* 4672: Special privileges assigned
* 4719: System audit policy change


4. Restart the service: `Restart-Service winlogbeat`
5. Verify in Kibana that only the filtered events appear

**Hint**: Use the `processors` section to filter events by EventID.

**Example Configuration**:

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

### 3.3 Metricbeat Installation and Configuration

#### Step 1: Install Metricbeat

Install Metricbeat on the Debian VM:

```bash
sudo apt install -y metricbeat

```

#### Step 2: Basic Configuration

Edit the configuration file:

```bash
sudo nano /etc/metricbeat/metricbeat.yml

```

Configure the `output.elasticsearch` section:

```yaml
output.elasticsearch:
  hosts: ["localhost:9200"]

```

#### Step 3: Enable System Modules

Enable the system module to collect system metrics:

```bash
sudo metricbeat modules enable system

```

#### Step 4: System Module Configuration

The system module collects by default:

* CPU
* Memory
* Disk
* Network
* Processes
* Filesystem

The default configuration should suffice. Check if necessary:

```bash
sudo nano /etc/metricbeat/modules.d/system.yml

```

#### Step 5: Start Metricbeat

Start Metricbeat:

```bash
sudo systemctl start metricbeat
sudo systemctl enable metricbeat

```

Check the status:

```bash
sudo systemctl status metricbeat

```

#### Step 6: Verification in Kibana

1. Wait a few minutes
2. In Kibana, create a data view: `metricbeat-*`
3. Explore the metrics in **Discover**
4. Go to **Stack Monitoring** to see predefined dashboards

**Verification**: You should see system metrics (CPU, memory, disk, network) in Kibana.

---

### 3.4 Packetbeat Installation and Configuration

**Note**: Packetbeat requires elevated privileges to capture network traffic.

#### Step 1: Install Packetbeat

Install Packetbeat:

```bash
sudo apt install -y packetbeat

```

#### Step 2: Configuration

Edit the configuration file:

```bash
sudo nano /etc/packetbeat/packetbeat.yml

```

Configure the network interfaces to monitor:

```yaml
packetbeat.interfaces.device: any

# Protocols to capture
packetbeat.protocols:
  - type: http
    ports: [80, 8080, 8000, 5000, 8002, 9200]
  - type: mysql
    ports: [3306]
  - type: redis
    ports: [6379]

```

Configure the output:

```yaml
output.elasticsearch:
  hosts: ["localhost:9200"]

```

#### Step 3: Start Packetbeat

Start Packetbeat (requires root privileges):

```bash
sudo systemctl start packetbeat
sudo systemctl enable packetbeat

```

#### Step 4: Verification

Verify in Kibana with the `packetbeat-*` data view.

**Note**: Packetbeat can be resource-intensive. On a VM with 2 GB of RAM, use it cautiously.

---

## 4. Part 3: Visualization and Analysis in Kibana

### 4.1 Creating Data Views

#### Step 1: Access Data Views

1. In Kibana, go to **Stack Management → Data Views**
2. Click on **Create data view**

#### Step 2: Create a Data View for Filebeat

1. Enter the pattern: `filebeat-*`
2. Click on **Next step**
3. Select the timestamp field: `@timestamp`
4. Click on **Create data view**

#### Step 3: Creating Other Data Views

Repeat for:

* `winlogbeat-*`
* `metricbeat-*`
* `packetbeat-*` (if installed)

### 4.2 Creating Basic Visualizations

#### Visualization 1: Time Series Chart of Logs

1. Go to **Dashboard** > **Create visualization**
2. Choose **Line** (line chart)
3. Select the `filebeat-*` data view
4. Configure:
* **Y-axis**: Count
* **X-axis**: Date Histogram on `@timestamp`


5. Click on **Save** and give it a name: "Logs Over Time"

#### Visualization 2: Top 10 Log Sources

1. Create a new **Data Table** visualization
2. Select `filebeat-*`
3. Configure:
* **Metric**: Count
* **Buckets**: Terms on `host.name` (or `source`)


4. Limit to 10 results
5. Save: "Top 10 Log Sources"

#### Visualization 3: Distribution by Log Type

1. Create a **Pie Chart** visualization
2. Select `filebeat-*`
3. Configure:
* **Slice by**: Terms on `fileset.name` or `log.file.path`


4. Save: "Log Type Distribution"

### 4.3 Creating a Dashboard

#### Step 1: Create the Dashboard

1. Go to **Dashboard** > **Create dashboard**
2. Click on **Add** to add visualizations
3. Add the visualizations created previously

#### Step 2: Dashboard Organization

* Organize the visualizations logically
* Adjust the size of the panels
* Add a title: "SIEM Dashboard - Overview"

#### Step 3: Save

Save the dashboard: "Main SIEM Dashboard"

### 4.4 Log Search and Analysis

#### Simple Search

1. Go to **Discover**
2. Select the `filebeat-*` data view
3. Use the search bar to filter:
* `status:error` (error logs)
* `message:*failed*` (messages containing "failed")
* `@timestamp:[now-1h TO now]` (last hour)



#### Advanced Search with KQL

Use the KQL (Kibana Query Language) syntax:

```
host.name: "server-name" and message: "error"

```

```
@timestamp >= "now-24h" and log.level: "error"

```

#### Correlation Analysis

1. Search for suspicious patterns:
* Multiple login failures: `event.action: "authentication_failure"`
* Attempts to access sensitive files
* Suspicious network activity


2. Use filters to refine your search

### 4.5 Security Dashboard with Alerts

**Task**:

1. Create a new dashboard: "Security Dashboard"
2. Add the following visualizations:
* Time series chart of authentication failures (Winlogbeat, EventID 4625)
* Top 10 source IPs with the most failed logins
* Geographic map of connections (if available)
* Chart of Windows events by type (Security, Application, System)
* Real-time system metrics (CPU, memory, disk)


3. Configure alerts (if available in your version):
* Alert if more than 10 login failures in 5 minutes
* Alert if CPU usage > 80%


4. Save and share the dashboard

**Hint**: Use the `winlog.event_id` fields to filter specific Windows events.

---

## 5. Part 4: Practical Use Case

### 5.1 Scenario: Detecting an Intrusion Attempt

#### Context

You are a system administrator and must analyze logs to detect suspicious activity. A user reports strange connections to a server.

#### Available Data

You have access to the following logs:

* System logs (Filebeat)
* Windows events (Winlogbeat)
* System metrics (Metricbeat)

#### Mission

Identify the signs of an intrusion attempt by analyzing the logs.

### 5.2 Correlated Log Analysis

#### Step 1: Search for Authentication Failures

1. In Kibana, go to **Discover**
2. Select the `winlogbeat-*` index
3. Search for login failures:

```
winlog.event_id: 4625

```

4. Analyze:
* Number of failures
* Source IPs
* Targeted accounts
* Activity period



#### Step 2: Search for Suspicious Network Activity

1. If Packetbeat is installed, analyze network traffic
2. Search for:
* Connections from suspicious IPs
* Non-standard ports
* Abnormal traffic volumes



#### Step 3: System Log Analysis

1. In `filebeat-*`, search for:
* Attempts to access sensitive files
* Configuration changes
* Execution of suspicious commands



#### Step 4: Temporal Correlation

1. Use the time filter to identify periods of suspicious activity
2. Correlate events between different sources
3. Identify attack patterns

### 5.3 Creating Simple Correlation Rules

#### Rule 1: Brute Force Detection

**Logic**: More than 5 login failures from the same IP in 10 minutes

**Kibana Implementation**:

1. Create a **Data Table** visualization
2. Configure:
* **Metric**: Count
* **Buckets**: Terms on `source.ip` (or `winlog.event_data.IpAddress`)
* **Filter**: `winlog.event_id: 4625`


3. Add a time filter: `@timestamp:[now-10m TO now]`
4. Identify IPs with more than 5 occurrences

#### Rule 2: Off-Hours Activity Detection

**Logic**: Successful connections outside business hours (e.g., 10 PM - 6 AM)

**Implementation**:

1. Search for successful connections: `winlog.event_id: 4624`
2. Filter by time: `@timestamp:[now-24h TO now]`
3. Analyze connection times

### 5.4 Identifying a Suspicious Pattern

**Scenario**: Analyze logs to identify a suspicious attack pattern.

**Task**:

1. Generate or simulate suspicious activity:
* Multiple failed login attempts from different IPs
* Access to sensitive files
* Execution of suspicious commands


2. In Kibana, create searches to identify:
* The source IPs of the attacks
* The targeted accounts
* The attack methods used
* The attack timeline


3. Create an analysis report with:
* Incident summary
* Event timeline
* Response recommendations


4. Present your findings in a dedicated dashboard

**Hint**: Use the saved search features in Kibana to document your analyses.

---

## 6. Appendices

### 6.1 Useful Commands

#### Elasticsearch

```bash
# Check status
sudo systemctl status elasticsearch

# Restart
sudo systemctl restart elasticsearch

# View logs
sudo journalctl -u elasticsearch -f

# List indices
curl http://localhost:9200/_cat/indices?v

# Delete an index (be careful!)
curl -X DELETE http://localhost:9200/index-name

```

#### Kibana

```bash
# Check status
sudo systemctl status kibana

# Restart
sudo systemctl restart kibana

# View logs
sudo journalctl -u kibana -f

```

#### Logstash

```bash
# Check status
sudo systemctl status logstash

# Restart
sudo systemctl restart logstash

# Test a configuration
sudo /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/test.conf --config.test_and_exit

# View logs
sudo journalctl -u logstash -f

```

#### Filebeat

```bash
# Check status
sudo systemctl status filebeat

# Test configuration
sudo filebeat test config

# List modules
sudo filebeat modules list

# View logs
sudo journalctl -u filebeat -f

```

#### Metricbeat

```bash
# Check status
sudo systemctl status metricbeat

# Test configuration
sudo metricbeat test config

# List modules
sudo metricbeat modules list

```

#### Winlogbeat (Windows PowerShell)

```powershell
# Check status
Get-Service winlogbeat

# Restart
Restart-Service winlogbeat

# Test configuration
cd "C:\Program Files\Winlogbeat"
.\winlogbeat.exe test config -c .\winlogbeat.yml

```

### 6.2 Troubleshooting

#### Problem: Elasticsearch is not starting

**Symptoms**: Service in `failed` state or memory error

**Solutions**:

1. Check logs: `sudo journalctl -u elasticsearch -n 50`
2. Check available memory: `free -h`
3. Reduce the heap in `/etc/elasticsearch/jvm.options.d/heapsizemem.options`
4. Check permissions: `sudo chown -R elasticsearch:elasticsearch /var/lib/elasticsearch`

#### Problem: Kibana cannot connect to Elasticsearch

**Symptoms**: "Unable to connect to Elasticsearch" error

**Solutions**:

1. Verify that Elasticsearch is running: `curl http://localhost:9200`
2. Check the configuration in `kibana.yml`:
```bash
sudo grep "^elasticsearch.hosts" /etc/kibana/kibana.yml

```


The line must be uncommented (no `#` at the beginning)
3. Check logs: `sudo journalctl -u kibana -n 50`
4. Check the firewall

#### Problem: Kibana Page Stays Loading

**Symptoms**: The Kibana page appears but remains stuck on "Loading..." or "Kibana is starting," even after several minutes.

**Possible Causes**:

1. Saved object migrations are in progress (normal on first startup)
2. Kibana waits for migrations to complete before displaying the interface
3. Performance issue (lack of memory)

**Solutions**:

1. **Check migration status in logs**:
```bash
sudo journalctl -u kibana -f | grep -i "migration\|savedobjects"

```


You should see messages like "Starting saved objects migrations" then "Migration completed" or "Migration successful."
2. **Wait 3-5 minutes** during the first startup. Migrations can take time, especially with limited RAM.
3. **Verify Kibana is connected to Elasticsearch**:
```bash
sudo journalctl -u kibana | grep -i "connected to elasticsearch"

```


You should see "Successfully connected to Elasticsearch."
4. **Check Kibana service status**:
```bash
sudo systemctl status kibana

```


The service must be "active (running)."
5. **Check full logs for errors**:
```bash
sudo journalctl -u kibana -n 200 --no-pager

```


Look for errors (ERROR, FATAL) or important warnings.
6. **If migrations seem blocked** (more than 10 minutes), you can try to reset them (⚠️ **CAUTION**: this will delete all Kibana configurations):
```bash
# Stop Kibana
sudo systemctl stop kibana

# Delete Kibana indices (loss of all configurations, visualizations, dashboards)
curl -X DELETE "http://localhost:9200/.kibana*"

# Restart Kibana
sudo systemctl start kibana

# Wait 3-5 minutes for new migrations

```


7. **Check available memory**:
```bash
free -h

```


If memory is saturated, migrations can be very slow. Consider reducing the Elasticsearch heap.

**Note**: If you see "Starting saved objects migrations" in the logs but no end message, it's normal - migrations can take several minutes. Be patient and monitor the logs.

#### Problem: Logs do not appear in Kibana

**Symptoms**: No data in Discover

**Solutions**:

1. Verify that Beats are running: `sudo systemctl status filebeat`
2. Check indices in Elasticsearch: `curl http://localhost:9200/_cat/indices?v`
3. Check the Beats configuration
4. Check Beats logs: `sudo journalctl -u filebeat -f`

#### Problem: Winlogbeat is not collecting events

**Symptoms**: No Windows events in Kibana

**Solutions**:

1. Check the service: `Get-Service winlogbeat`
2. Check logs: `Get-EventLog -LogName Application -Source winlogbeat`
3. Check the configuration: `.\winlogbeat.exe test config`
4. Check connectivity to Elasticsearch: `Test-NetConnection -ComputerName VM_IP -Port 9200`

#### Problem: Slow Performance with 2 GB of RAM

**Symptoms**: Slow system, crashing services

**Solutions**:

1. Reduce the Elasticsearch heap (256m depending on other services)
2. Limit the number of indices
3. Disable non-essential features (ML, etc.)
4. Use swap if necessary (but it slows down)
5. Check memory consumption of other services (Kibana, Logstash)

### 6.3 Additional Resources

#### Official Documentation

* **Elastic Stack**: [https://www.elastic.co/guide/](https://www.elastic.co/guide/)
* **Elasticsearch**: [https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
* **Logstash**: [https://www.elastic.co/guide/en/logstash/current/index.html](https://www.elastic.co/guide/en/logstash/current/index.html)
* **Kibana**: [https://www.elastic.co/guide/en/kibana/current/index.html](https://www.elastic.co/guide/en/kibana/current/index.html)
* **Beats**: [https://www.elastic.co/guide/en/beats/index.html](https://www.elastic.co/guide/en/beats/index.html)

#### Grok Patterns

* **Grok Debugger**: [https://grokdebug.herokuapp.com/](https://grokdebug.herokuapp.com/)
* **Basic Patterns**: [https://github.com/elastic/logstash/blob/v1.4.2/patterns/grok-patterns](https://github.com/elastic/logstash/blob/v1.4.2/patterns/grok-patterns)

#### Communities

* **Elastic Forum**: [https://discuss.elastic.co/](https://discuss.elastic.co/)
* **Stack Overflow**: Tag `elasticsearch`, `logstash`, `kibana`

#### Useful Tools

* **Elasticsearch Head**: Plugin for visualizing Elasticsearch
* **Cerebro**: Web interface for managing Elasticsearch
* **Elasticsearch Curator**: Tool for managing indices

### 6.5 End of TP Checklist

Before finishing, verify that you have:

* [ ] Elasticsearch installed and functional
* [ ] Kibana accessible from your Windows PC
* [ ] Logstash configured with at least one pipeline
* [ ] Filebeat collecting system logs
* [ ] Winlogbeat collecting Windows events
* [ ] Metricbeat collecting system metrics
* [ ] At least 3 visualizations created in Kibana
* [ ] A functional dashboard
* [ ] Documented your configurations and discoveries

---

## Conclusion

Congratulations! You now have a complete, operational SIEM solution. You can:

* Collect logs from different sources
* Store and index data in Elasticsearch
* Visualize and analyze data in Kibana
* Detect suspicious activity

**Good work!**

---

*TP created for UE S10-3 - Information Security and Event Management* *ESAIP School - Computer and Networks Engineering*
