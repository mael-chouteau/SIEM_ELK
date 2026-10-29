# Lab: Deploy a SIEM with the ELK Stack

**Course unit S10-3 — Information security and event management**  
**Practical work — individual**

French version: [README.md](README.md).

You name your own Kibana dashboards and visualisations; titles in this handout are suggestions only.

---

## Contents

1. [Context and objectives](#1-context-and-objectives)
2. [Part 1: Install the ELK stack](#2-part-1-install-the-elk-stack)
3. [Part 2: Configure Beats agents](#3-part-2-configure-beats-agents)
4. [Part 3: Visualise and analyse in Kibana](#4-part-3-visualise-and-analyse-in-kibana)
5. [Part 4: Practical case](#5-part-4-practical-case)
6. [Appendices](#6-appendices)

---

## 1. Context and objectives

### 1.1 What this lab is

This lab puts the lecture into practice: you install and configure a full SIEM based on the **ELK stack** (Elasticsearch, Logstash, Kibana).

You will:
- Install and configure the ELK stack on a Debian SIEM VM
- Configure Suricata, Apache, Filebeat and Metricbeat on a Linux VM you are monitoring
- Configure Winlogbeat on a Windows machine so its logs reach the SIEM
- Build visualisations and dashboards in Kibana
- Analyse real security events (IDS alerts, SSH auth, web access, metrics, Windows events)

### 1.2 Learning outcomes

By the end of this lab you will be able to:

- Install and configure Elasticsearch, Logstash and Kibana
- Set up TLS (HTTP and transport) and authentication between components
- Tune Elasticsearch for a machine with limited RAM
- Configure Filebeat to collect system logs
- Configure Winlogbeat to collect Windows events
- Configure Metricbeat to collect system metrics
- Build visualisations and dashboards in Kibana
- Read logs closely enough to spot suspicious activity

### 1.3 Lab environment

The ESAIP lab uses **two Linux VMs** plus a **Windows machine** (Kibana in the browser, and Winlogbeat).

#### SIEM VM (ELK collector)
- **Role**: Elasticsearch, Kibana, Logstash
- **OS**: Debian 13 (Trixie)
- **Stack**: Elastic **9.5.x** (repository `9.x`)
- **Resources**: **8 GB RAM**, **8 vCPU**, ~20 GB disk — if ~2 GB RAM, lower the heap (§2.2); under 4 GB with Kibana + Logstash is not recommended
- **Access**: SSH; Kibana on port `5601` (HTTP); Elasticsearch on `9200` (HTTPS)
- **Heaps (8 GB)**: Elasticsearch `-Xms2g -Xmx2g`; Logstash left at the package default (`jvm.options`, 1 GB); `xpack.ml.enabled: false`

#### Monitored VM (Linux)
- **Role**: log and metric sources sent to the SIEM
- **OS**: Debian 13 — **~2 GB RAM**, **2 vCPU**
- **Target services**: Suricata (alerts), SSH (`auth.log`), Apache, Metricbeat
- **Agents**: Filebeat + Metricbeat → `https://IP_SIEM:9200` (TLS + `beats_writer` user)

#### Windows machine
- Kibana: `http://IP_SIEM:5601` (login `elastic`)
- **Winlogbeat** (same branch as the SIEM): Application / System / Security logs → Elasticsearch `https://IP_SIEM:9200`
- Check which Windows interface actually reaches the SIEM (`curl` + `Get-NetTCPConnection`). A multi-homed PC can reach the lab over Ethernet while Wi-Fi is used for something else.

#### Network layout

```
┌──────────────────┐         ┌──────────────────────┐
│  Monitored VM    │         │  SIEM VM (ELK)       │
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
│  Windows machine │──── Winlogbeat ────┘
│  - browser       │     :9200
│  - Winlogbeat    │
└──────────────────┘
         (lab network)
```

### 1.4 Prerequisites

Before you start, make sure you have:

- SSH access to the SIEM VM **and** the monitored VM
- Administrator rights (`sudo`) on both VMs
- Network access between the two VMs (port `9200` towards the SIEM) and from your PC to Kibana (`5601`)
- A current browser (Chrome, Firefox, Edge)
- Basic Linux skills (commands, editing files)

### 1.5 Estimated time

- **Part 1**: 2–3 hours
- **Part 2**: 2–3 hours
- **Part 3**: 1–2 hours
- **Part 4**: 1–2 hours

**Total**: 6–10 hours of work

---

## 2. Part 1: Install the ELK stack

### 2.1 Check the environment

#### Step 1: Connect to the SIEM VM

Open an SSH session on the **SIEM VM** (the ELK collector). Elasticsearch, Kibana and Logstash are installed on this machine.

```bash
ssh USER@IP_SIEM
```

`IP_SIEM` and `USER` are the values shown in [Proxmox](https://10.3.2.10:8006). Every ELK install step below runs **on this VM**.

#### Step 2: Check the system

Before installing ELK, check the OS, kernel, RAM and disk. The JVM heap depends on how much memory you actually have.

```bash
# Debian version
cat /etc/debian_version

# Kernel and hostname
uname -a

# Memory
free -h

# Disk space
df -h
```

- `free -h`: free and used memory, in human-readable units.
- `df -h`: disk space per filesystem.

**Check**: write down the RAM (`free -h`). With about 2 GB, keep the JVM heap small (see §2.2). With more RAM you can raise the heap (for example `1g`) without filling the machine.

#### Step 3: Update the system

Refresh the package lists, then apply available updates, so Elastic does not install against stale dependencies.

```bash
sudo apt update
sudo apt upgrade -y
```

`-y`: answer yes to apt prompts, so the upgrade does not stop and wait.

#### Step 4: Install basic tools

Install the tools this lab needs (downloads, editing, the GPG key).

```bash
sudo apt install -y curl wget nano git gpg
```

---

### 2.2 Install Elasticsearch

If the install fails, check that the procedure has not changed in the [Elastic documentation](https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-elasticsearch-with-debian-package).

#### Step 1: Add the Elastic GPG key

Download Elastic's public key and store it in the apt keyring. apt will then trust only packages signed with that key.

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
```

- `-qO -`: download quietly and write to standard output, so the pipe can read it.
- `gpg --dearmor`: turn the ASCII key into the binary form apt expects.
- The `|` (pipe) is **required**. Without it, the key is never written into the keyring.

#### Step 2: Install apt-transport-https

Allow apt to fetch packages over HTTPS. The Elastic repository needs that.

```bash
sudo apt install -y apt-transport-https
```

#### Step 3: Add the Elastic repository

Register the Elastic 9.x repository in apt's sources, and bind it to the GPG key from step 1.

```bash
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/9.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-9.x.list
```

- `signed-by=...`: apt checks packages against the keyring you just created.
- `tee`: writes the line into the sources file and prints it (needed so `sudo` can write the file).

#### Step 4: Update and install

Reload package lists so the new repository shows up, then install Elasticsearch.

```bash
sudo apt-get update && sudo apt-get install elasticsearch
```

`&&`: install only if the list update succeeded.

#### Step 5: Configure the Elasticsearch heap

**Important**: set the heap from the SIEM VM's real RAM.

Open (or create) the JVM options file used for the heap. This is where you cap Elasticsearch's memory.

```bash
sudo nano /etc/elasticsearch/jvm.options.d/heapsizemem.options
```

**Note**: the extension must be `.options` (not `.conf`), or Elasticsearch ignores the file.

With **about 2 GB** of RAM (typical classroom VM), put this in the file:

```properties
# Heap capped at 512 MB
-Xms512m
-Xmx512m
```

With **about 8 GB** of RAM and **8 vCPU**:

```properties
# 2 GB heap — comfortable headroom for Kibana + Logstash on 8 GB
-Xms2g
-Xmx2g
```

With **about 6 GB** of RAM:

```properties
# 1 GB heap
-Xms1g
-Xmx1g
```

**What this means**:
- `-Xms` / `-Xmx`: initial and maximum JVM heap. Keep them equal.
- Elasticsearch uses more RAM than the heap (off-heap, buffers, helper processes).
- After startup, check with `curl -sk -u elastic:$ELASTIC_PASS --cacert /etc/elasticsearch/certs/http_ca.crt https://localhost:9200/_nodes/jvm?pretty` (`heap_max_in_bytes`).

#### Step 6: TLS certificates

Install `unzip` if needed, then generate a CA and a node certificate with a **SAN** (localhost + SIEM IP). Replace `IP_SIEM` with the real address.

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
sudo cp /etc/elasticsearch/certs/http_ca.crt /home/$USER/http_ca.crt
sudo chown $USER:$USER /home/$USER/http_ca.crt
```

#### Step 7: Elasticsearch configuration

Edit the main config to expose the API, name the node, and enable TLS plus authentication.

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

**Elasticsearch 9.x**:

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

**Notes**:
- The package `elasticsearch.yml` already has **uncommented** `path.data: /var/lib/elasticsearch` and `path.logs: /var/log/elasticsearch`. Keep them. The rest of the file is commented: add the block above, do not blank the file.
- On Elasticsearch 9.x, `http.host` is the HTTP API bind setting.
- On a single node, `discovery.type: single-node` avoids `cluster.initial_master_nodes`.
- Never commit real passwords to Git.

#### Step 8: Start the service and create accounts

```bash
sudo systemctl enable --now elasticsearch
sudo systemctl status elasticsearch
```

Wait 30–60 s. Without a password, the API returns **401** (TLS OK, auth required):

```bash
curl -sk -o /dev/null -w '%{http_code}\n' https://127.0.0.1:9200
```

Reset the built-in accounts (write down the “New value” lines):

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic -b
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system -b
```

Create a role and user for Beats:

```bash
export ELASTIC_PASS='…'
export BEATS_PASS='…'

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

#### Step 9: Check that it answers

```bash
curl -sk -u "elastic:${ELASTIC_PASS}" --cacert /etc/elasticsearch/certs/http_ca.crt https://127.0.0.1:9200
curl -sk -u "elastic:${ELASTIC_PASS}" --cacert /etc/elasticsearch/certs/http_ca.crt https://127.0.0.1:9200/_nodes/jvm?pretty
```

From another lab host (with the CA copied):

```bash
curl -sk -u "elastic:${ELASTIC_PASS}" --cacert ./http_ca.crt https://IP_SIEM:9200
```

On a **single-node** cluster, status `yellow` (unassigned replicas) is **normal**.

#### Help: out-of-memory at startup

**Situation**: Elasticsearch does not start, or crashes with a memory error.

**What to do**:
1. Read the logs: `sudo journalctl -u elasticsearch -n 50`
2. Find the memory error
3. Lower `-Xms` and `-Xmx` in `/etc/elasticsearch/jvm.options`
4. Restart: `sudo systemctl restart elasticsearch`

**Hint**: with about 2 GB of RAM, start from a **512m** heap (§2.2). A 1 GB heap fits a larger SIEM VM (about 8 GB).

---

### 2.3 Install Kibana

#### Step 1: Install

Kibana is in the Elastic repository you already added. Install the package to get the SIEM web UI.

```bash
sudo apt install -y kibana
```

#### Step 2: Configure Kibana

Copy the Elasticsearch CA, then point Kibana at HTTPS and the `kibana_system` account. Do not replace the whole file.

```bash
sudo mkdir -p /etc/kibana/certs
sudo cp /etc/elasticsearch/certs/http_ca.crt /etc/kibana/certs/http_ca.crt
sudo chown -R kibana:kibana /etc/kibana/certs
ENC_KEY=$(openssl rand -hex 32)
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

The Kibana 9.5 package already has two **active** settings. **Keep them**:

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

In the same file, `server.host`, `server.port` and `elasticsearch.hosts` are **commented** (`localhost`, `http://localhost:9200`). Uncomment them and replace them with the HTTPS block above. Add the three `encryptionKey` lines (they are not in the package).

Replace `KIBANA_SYSTEM_PASS` and `ENC_KEY` with real values, then:

```bash
sudo chmod 640 /etc/kibana/kibana.yml
sudo chown root:kibana /etc/kibana/kibana.yml
```

- `server.host: "0.0.0.0"`: reachable from your Windows PC.
- Kibana listens in **HTTP** on `:5601`; the **Kibana → Elasticsearch** link is **HTTPS**.

#### Step 3: Start Kibana

Start Kibana and enable it at boot.

```bash
sudo systemctl start kibana
sudo systemctl enable kibana
```

Check the service:

```bash
sudo systemctl status kibana
```

**Important**:
- Kibana can take **1–2 minutes** to finish starting.
- **On the first start**, Kibana migrates saved objects. That can take another **3–5 minutes**.
- The web page may keep loading during that time. That is **normal**.
- Follow the logs live (`-f` means follow) to see the migration:

  ```bash
  sudo journalctl -u kibana -f
  ```

  You should see "Starting saved objects migrations", then "Migration completed" or something similar.

#### Step 4: Open Kibana

From the browser on your Windows machine, open this URL (replace `IP_SIEM`):

```
http://IP_SIEM:5601
```

There is no login on this lab: security is disabled. You should land on the Kibana home page.

#### Help: Kibana is not reachable from Windows

**Situation**: you cannot open Kibana from your Windows PC.

**What to do**:
1. Check that Kibana listens on every interface: `sudo netstat -tlnp | grep 5601`
2. Check the VM firewall: `sudo ufw status`
3. If the firewall blocks the port: `sudo ufw allow 5601/tcp`
4. From your PC, check connectivity: `ping IP_SIEM`
5. From your PC, test: `curl http://IP_SIEM:5601`, or open the URL in the browser

**Hint**: if a firewall is on, also open port 9200 so the agents can reach Elasticsearch.

---

### 2.4 Install Logstash

#### Step 1: JDK

The Logstash 9 package ships its own JDK (`/usr/share/logstash/jdk`). Do not install `default-jre` for this lab: the service does not use the system `java`. The heap stays at `/etc/logstash/jvm.options` (1 GB). This lab does not change it.

---

#### Step 2: Install Logstash

Refresh apt if needed, then install Logstash from the same Elastic 9.x repository.

```bash
sudo apt update
sudo apt install -y logstash
```

---

#### Step 3: Basic Logstash configuration

The package `/etc/logstash/logstash.yml` is fully commented. `/etc/logstash/pipelines.yml` **already** points at the pipelines:

```yaml
- pipeline.id: main
  path.config: "/etc/logstash/conf.d/*.conf"
```

Do not copy `path.config` into `logstash.yml`. Uncomment only the two lines that are already there:

```yaml
path.data: /var/lib/logstash
path.logs: /var/log/logstash
```

```bash
sudo nano /etc/logstash/logstash.yml
```

Create any missing directories and give them to the `logstash` user. Without that, the service often fails at startup.

```bash
sudo mkdir -p /var/log/logstash /var/lib/logstash
sudo chown -R logstash:logstash /var/log/logstash /var/lib/logstash
```

- `mkdir -p`: create the directories (and parents) if they are missing.
- `chown -R logstash:logstash`: owner and group of the Logstash service, on the whole tree.

---

#### Step 4: A test pipeline that works under systemd

**Important**:
Do **not** use a `stdin {}` pipeline with systemd. Logstash exits immediately.
Use a **file input** so the test is a real, long-running pipeline.

Create the test pipeline file. Logstash reads it on the next restart.

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
  # No filter for this test
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

- `sincedb_path => "/dev/null"`: forget the previous read position, so the test can be repeated.
- `index => "test-logs-%{+YYYY.MM.dd}"`: create an Elasticsearch index named with today's date.
- On Logstash 9.x, the old `cacert` setting is **obsolete** and prevents the pipeline from starting. Use `ssl_certificate_authorities` (and `ssl_enabled => true`) instead.

Create the file the pipeline reads, and make it writable for the test:

```bash
sudo touch /tmp/logstash-test.log
sudo chmod 666 /tmp/logstash-test.log
```

`chmod 666`: read and write for everyone. Handy in the lab so `echo >>` works without sudo. **Not** for production.

---

#### Step 5: Start Logstash

Enable the service at boot and restart it so it loads the new pipeline.

```bash
sudo systemctl enable logstash
sudo systemctl restart logstash
```

Confirm it is running:

```bash
sudo systemctl status logstash
```

You must see:

```
Active: active (running)
```

---

#### Step 6: Test the pipeline

In another terminal, append lines to the watched file. Logstash reads them, sends them to Elasticsearch, and prints them in debug.

```bash
echo "Bonjour Logstash" >> /tmp/logstash-test.log
echo "Deuxième message" >> /tmp/logstash-test.log
```

`>>`: append to the file. It does not wipe what is already there.

Expected result:

* The lines show up in Elasticsearch. In Kibana: **Stack Management → Index Management**, index `test-logs-YYYY.MM.dd`.
* Logstash **stays running**. It must not exit after the test.

---

#### Apache log pipeline

**Goal**

Build a Logstash pipeline that reads and parses Apache logs.

---

##### Step 1: Install Apache

Install Apache on the SIEM VM so you get local `access.log` lines for Logstash to parse. This is a pipeline exercise. In the two-VM layout, the recommended Apache collection is still Filebeat (§3.1).

```bash
sudo apt install -y apache2
```

Check that Apache answers locally (the default page):

```bash
curl http://localhost
```

---

##### Step 2: Create the Apache pipeline

Create the pipeline that reads `/var/log/apache2/access.log`, parses it with Grok, and indexes it in Elasticsearch.

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

**Important**:
Disable the test pipeline so only one pipeline stays active. That avoids extra indexes and noise.

```bash
sudo mv /etc/logstash/conf.d/test.conf /etc/logstash/conf.d/test.conf.disabled
```

Add the `logstash` user to the `adm` group so it can read Apache logs (often mode `640`, owner `root:adm`):

```bash
sudo usermod -aG adm logstash
```

`-aG`: add the group without removing the groups the user already has.

---

##### Step 3: Restart Logstash

Reload Logstash so it picks up `apache.conf` and the new group membership.

```bash
sudo systemctl restart logstash
```

Show the last 20 lines of the service journal (`--no-pager`: print and exit, no interactive pager):

```bash
journalctl -u logstash -n 20 --no-pager
```

You must see:

* `Pipeline started`
* no fatal error

---

##### Step 4: Generate Apache traffic

Send a few HTTP requests. Each `curl` adds a line to `access.log`, which Logstash forwards to the `apache-logs-*` index.

```bash
curl http://localhost
curl http://localhost
curl http://localhost
```

---

##### Step 5: Check in Kibana

1. Go to **Stack Management → Data Views**
2. Create a data view:

   ```
   apache-logs-*
   ```
3. Time field: `@timestamp`
4. Open **Discover**

Expected result:

* Apache requests are visible
* Parsed fields (`clientip`, `verb`, `request`, `response`, and so on)

---

**Hint**

Grok pattern used:

```
%{COMBINEDAPACHELOG}
```

Official documentation: [grok](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html)

---

## 3. Part 2: Configure Beats agents (monitored VM)

> **Where to work**: Filebeat, Metricbeat, Suricata and Apache in this part run on the **monitored VM**, not on the SIEM.  
> Replace `IP_SIEM` with the ELK VM's address.  
> First add the Elastic `9.x` repository on this VM (same GPG + `elastic-9.x.list` procedure as §2.2), then install the Beats packages.

### 3.1 Install and configure Filebeat

#### Step 1: Install Filebeat and rsyslog

Since Debian 8, journald stores system events. Without rsyslog, Filebeat does not find `/var/log/auth.log` and `/var/log/syslog` at the usual paths.

Install rsyslog and start it immediately (`--now` means enable and start) so classic log files are written to disk.

```bash
sudo apt install -y rsyslog
sudo systemctl enable --now rsyslog
```

Then install Filebeat, the agent that ships files and logs to Elasticsearch.

```bash
sudo apt install -y filebeat
```

#### Step 2: Basic configuration (output to the SIEM)

The package ships a **default** `/etc/filebeat/filebeat.yml` that still points at `localhost:9200` over HTTP, with auth and TLS **commented out**. Start from that file: do not replace it wholesale — **edit the `output.elasticsearch` section**.

On the monitored VM, fetch the CA from the SIEM first:

```bash
scp USER@IP_SIEM:~/http_ca.crt /tmp/http_ca.crt
sudo mkdir -p /etc/filebeat/certs /etc/metricbeat/certs
sudo cp /tmp/http_ca.crt /etc/filebeat/certs/http_ca.crt
sudo cp /tmp/http_ca.crt /etc/metricbeat/certs/http_ca.crt
```

Then edit the config:

```bash
sudo nano /etc/filebeat/filebeat.yml
```

**What the stock file looks like** (landmarks to find, Elastic 9.x):

```yaml
output.elasticsearch:
  hosts: ["localhost:9200"]
  preset: balanced
  #protocol: "https"
  #username: "elastic"
  #password: "changeme"
```

Further down, a **`#output.logstash:`** section also has `#ssl.certificate_authorities: …` lines — **do not uncomment those**. That is not the Elasticsearch output, and Beats in this lab talk to Elasticsearch, not to Logstash (`:5044`).

Keep the rest of the package file, in particular:

- `filebeat.inputs`: the sample `filestream` stays `enabled: false`. If you set it to `true`, Filebeat also ships `/var/log/*.log` on top of the modules.
- `filebeat.config.modules` (`modules.d/*.yml`): this is what loads system / apache / suricata.
- `setup.template.settings` (`index.number_of_shards: 1`).
- `processors` (at least `add_host_metadata`).

**What to change** under `output.elasticsearch` only:

1. Set `hosts` to the SIEM **HTTPS** URL (`https://` in the URL is enough; you can leave `#protocol: "https"` commented).
2. Uncomment and set `username` / `password` (`beats_writer` / lab password — **not** `elastic`).
3. **Add** an `ssl:` block (missing under `output.elasticsearch` in the package YAML) so Filebeat trusts the lab CA.

Expected result:

```yaml
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["https://IP_SIEM:9200"]
  preset: balanced
  #protocol: "https"   # not needed if hosts already starts with https://
  username: "beats_writer"
  password: "BEATS_PASS"
  ssl:
    enabled: true
    certificate_authorities: ["/etc/filebeat/certs/http_ca.crt"]
    verification_mode: full
```

- Without `https://` (or without `protocol: "https"`), Filebeat uses plain **HTTP** → fails against ES 9 with TLS.
- Without the `ssl` + CA block, the handshake often fails (*x509: certificate signed by unknown authority*).
- `verification_mode: full` requires the IP/hostname in `hosts` to match a SAN on the node certificate (this lab generates a cert with `IP_SIEM`).

Check with `sudo filebeat test output` — expect `TLS... handshake... OK` then `talk to server... OK`.
#### Step 3: Enable modules (system, Apache, Suricata)

Enable the Filebeat modules that already know how to read SSH/auth logs, Apache and Suricata (EVE JSON). You do not have to type the paths by hand.

```bash
sudo filebeat modules enable system apache suricata
```

**Suricata**: enable the module only if `/var/log/suricata/eve.json` exists **and** the `eve` fileset is configured. Otherwise run `sudo filebeat modules disable suricata` (Filebeat will refuse to start).

#### Step 4: Configure the modules

**System (SSH / auth)** — `/etc/filebeat/modules.d/system.yml`:

```yaml
- module: system
  syslog:
    enabled: true
  auth:
    enabled: true
```

**Apache** — `/etc/filebeat/modules.d/apache.yml`:

```yaml
- module: apache
  access:
    enabled: true
    var.paths: ["/var/log/apache2/access.log*"]
  error:
    enabled: true
    var.paths: ["/var/log/apache2/error.log*"]
```

Without `var.paths`, the package Apache module already reads `/var/log/apache2/` on Debian. The paths above matter only if the logs live elsewhere. Suricata does need `var.paths` pointing at `eve.json`.

**Suricata** — `/etc/filebeat/modules.d/suricata.yml`:

```yaml
- module: suricata
  eve:
    enabled: true
    var.paths: ["/var/log/suricata/eve.json"]
```

**Install Suricata and a lab rule** (on the monitored VM):

Install Suricata (the IDS) and Apache (the HTTP target of the test), then start Apache so the HTTP request has a service behind it.

```bash
sudo apt install -y suricata
# Apache must be running for the HTTP test below
sudo apt install -y apache2
sudo systemctl enable --now apache2
```

Write a lab rule into Debian's rules directory. `tee` creates the file from the text between `EOF` (a heredoc).

```bash
sudo tee /var/lib/suricata/rules/lab-test.rules <<'EOF'
# Defensive lab rule — alert on HTTP requests to /esaip-lab-test
alert http any any -> $HOME_NET any (msg:"ESAIP-LAB-HTTP-TEST"; http.uri; content:"/esaip-lab-test"; sid:9000001; rev:1;)
EOF
```

- `<<'EOF'`: read until a line that is exactly `EOF`. Single quotes mean the shell does not expand variables.
- `sid:9000001`: a unique rule id, so it does not collide with official rules.

In `/etc/suricata/suricata.yaml`, check that `af-packet` listens on **`eth0`** (not only `lo`) and that `rule-files` loads at least `lab-test.rules`:

```yaml
rule-files:
  - lab-test.rules
```

Start Suricata and confirm it is active. If the service is down, nothing is written to `eve.json`.

```bash
sudo systemctl enable --now suricata
sudo systemctl status suricata
```

Alerts are in `/var/log/suricata/eve.json`. Rules managed by `suricata-update` live under `/var/lib/suricata/rules`.

**Raise an alert that really crosses eth0**. A `curl` to `localhost` on the monitored VM does **not** produce traffic on `eth0`. Send the request **from another lab machine** (for example the SIEM):

```bash
# From the SIEM VM (replace IP_MONITORED)
curl -s -o /dev/null -w "%{http_code}\n" http://IP_MONITORED/esaip-lab-test
```

- `-s`: quiet; `-o /dev/null`: discard the response body; `-w "%{http_code}\n"`: print only the HTTP status code.
- `IP_MONITORED`: the monitored VM's address. Not `localhost`, and not the SIEM's own address.

HTTP `404` is normal (that URL does not exist in Apache). Suricata must still raise `ESAIP-LAB-HTTP-TEST` (`in_iface: eth0`). Then check Kibana: `event.module: "suricata" and event.kind: "alert"` (signature field: `rule.name`).

#### Step 5: Start Filebeat

Enable and start Filebeat so it keeps shipping auth, Apache and Suricata events to Elasticsearch.

```bash
sudo systemctl enable --now filebeat
sudo systemctl status filebeat
```

Indexes / data streams: `.ds-filebeat-*` (Kibana pattern: `filebeat-*`).

#### Step 6: Check in Kibana

1. Open Kibana: `http://IP_SIEM:5601` (login `elastic`)
2. **Stack Management → Data Views** → pattern `filebeat-*`, time field `@timestamp`
3. In **Discover**, filter:
   - Suricata alerts: `event.module: "suricata" and event.kind: "alert"` (signature field: `rule.name`)
   - SSH / auth: `event.dataset: "system.auth"`
   - Apache: `event.dataset: "apache.access"`

#### Apache (the service)

If Apache is not installed yet on the monitored VM, install it, start it, and send a local request. That produces `access.log` lines for Filebeat to ship to the SIEM.

```bash
sudo apt install -y apache2
sudo systemctl enable --now apache2
curl http://localhost
```

Generate some traffic, then check `apache.access` in Discover.

> **Logstash / Apache**: the Logstash `apache.conf` pipeline in §2.4 (local read of `/var/log/apache2` on the SIEM) is still a useful exercise on the SIEM VM. In the two-VM layout, the **recommended** Apache collection is the Filebeat `apache` module on the monitored VM.

---

### 3.2 Install and configure Winlogbeat

> **Where to work**: on your **Windows machine** (PowerShell **as administrator**).  
> Match the Winlogbeat version to the SIEM.  
> Replace `IP_SIEM` with the ELK VM's address.

#### Step 0: Check connectivity to Elasticsearch

Before installing the agent, confirm that **this** Windows machine can reach the SIEM on port `9200`. Otherwise Winlogbeat installs and stays silent.

```powershell
curl.exe -sk -u elastic:ELASTIC_PASS --cacert C:\\path\\to\\http_ca.crt https://IP_SIEM:9200
# Must return JSON (cluster.name, version.number, …)

# Which Windows source IP is used?
Get-NetTCPConnection -RemoteAddress IP_SIEM -RemotePort 9200 |
  Select-Object LocalAddress, LocalPort, State
```

- `curl.exe`: the Windows HTTP client. It avoids the PowerShell alias `curl`, which is `Invoke-WebRequest`.
- `Get-NetTCPConnection`: shows the local interface actually used towards the SIEM.

A multi-homed PC (Wi-Fi + Ethernet) may reach the SIEM on a different interface from the one used for VNC or the browser. Write down the `LocalAddress` that is really used.

If `curl` fails: check the route (`Find-NetRoute -RemoteIPAddress IP_SIEM`), the Windows firewall, and that port `9200` is open on the SIEM.

#### Step 1: Download and extract

Download the Winlogbeat ZIP that matches the SIEM version, extract it, and copy the files under `Program Files` (the usual place for the service).

```powershell
# Administrator PowerShell
$ver = "9.5.4"
Invoke-WebRequest -Uri "https://artifacts.elastic.co/downloads/beats/winlogbeat/winlogbeat-$ver-windows-x86_64.zip" -OutFile "$env:TEMP\winlogbeat.zip"
Expand-Archive "$env:TEMP\winlogbeat.zip" -DestinationPath "$env:TEMP\winlogbeat-extract" -Force
# The ZIP creates a winlogbeat-9.5.4-windows-x86_64 folder — copy that under Program Files
New-Item -ItemType Directory -Path "C:\Program Files\Winlogbeat" -Force | Out-Null
Copy-Item -Path "$env:TEMP\winlogbeat-extract\winlogbeat-$ver-windows-x86_64\*" -Destination "C:\Program Files\Winlogbeat" -Recurse -Force
```

- `$ver`: Winlogbeat version. It must match the SIEM, for example 9.5.4.
- `Invoke-WebRequest` / `Expand-Archive`: download, then extract the ZIP.
- `| Out-Null`: hide the output of creating the directory.

Manual alternative: https://www.elastic.co/downloads/beats/winlogbeat — extract, then put the contents in `C:\Program Files\Winlogbeat`.

#### Step 2: Configuration (`winlogbeat.yml`)

The ZIP is not on the Linux VMs. The shipped `winlogbeat.yml` follows the same pattern as Filebeat (`hosts: ["localhost:9200"]`, `#protocol: "https"`, commented auth, `#ssl` lines under `#output.logstash`). Same edit as §3.1: `https://` URL, `beats_writer`, an `ssl` block under `output.elasticsearch`. Do not uncomment the Logstash SSL lines.

Edit `C:\Program Files\Winlogbeat\winlogbeat.yml` (Notepad or VS Code). At minimum, collect Application, System and Security, and send them to the SIEM:

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

`ignore_older: 72h`: do not ship events older than 72 hours. That avoids a flood on the first start.

Then check the YAML, the Elasticsearch connection, and the agent version:

```powershell
cd "C:\Program Files\Winlogbeat"
.\winlogbeat.exe test config -c .\winlogbeat.yml
.\winlogbeat.exe test output -c .\winlogbeat.yml
.\winlogbeat.exe version
```

`-c .\winlogbeat.yml`: explicit path to the configuration file.

#### Step 3: Install and start the service

Register Winlogbeat as a Windows service (`ExecutionPolicy Bypass` lets the supplied script run), then start it and check the status.

```powershell
cd "C:\Program Files\Winlogbeat"
PowerShell.exe -ExecutionPolicy Bypass -File .\install-service-winlogbeat.ps1
Start-Service winlogbeat
Get-Service winlogbeat
# Expected: Status = Running, StartType = Automatic
```

If the install script is missing, create the service by hand (binary path, config, and `path.home`):

```powershell
New-Service -Name "winlogbeat" `
  -BinaryPathName "`"C:\Program Files\Winlogbeat\winlogbeat.exe`" -c `"C:\Program Files\Winlogbeat\winlogbeat.yml`" -path.home `"C:\Program Files\Winlogbeat`"" `
  -StartupType Automatic -DisplayName "winlogbeat"
Start-Service winlogbeat
```

#### Step 4: Generate a benign Security test event

To see at least one failed logon (`4625`) in addition to the natural `4624` events, cause a harmless local authentication failure, then optionally write a control event in the Application log.

```powershell
# Harmless local auth failure (often produces EventID 4625)
net use \\127.0.0.1\IPC$ /user:baduser WrongPass123
# Control event in the Application log (optional)
eventcreate /T INFORMATION /ID 1000 /L APPLICATION /D "Test SIEM Winlogbeat"
```

- `net use ... /user:baduser`: a deliberately invalid attempt, which often becomes EventID **4625**.
- `eventcreate`: writes an Application event that is easy to filter in Kibana.

#### Step 5: Check in Kibana

1. Wait about a minute for events to ship
2. Data view: `winlogbeat-*` (time field `@timestamp`)
3. In **Discover**, useful filters:
   - Successful logons: `winlog.event_id: 4624`
   - Failed logons: `winlog.event_id: 4625`
   - Security channel: `winlog.channel: "Security"`
4. Data streams: `.ds-winlogbeat-*` (Kibana pattern: `winlogbeat-*`)

**Check**: Application, System and Security documents are present, and `agent.version` matches the SIEM.

#### Filter to a few Security event IDs (optional)

**Goal**: keep only some Security event IDs.

**Example**:

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

Restart the service so the filter applies: `Restart-Service winlogbeat`.

---

### 3.3 Install and configure Metricbeat

#### Step 1: Install Metricbeat

On the **monitored VM**, install Metricbeat to collect CPU, memory, disk, network and the rest, and send them to the SIEM.

```bash
sudo apt install -y metricbeat
```

#### Step 2: Basic configuration

Same idea as Filebeat: the package `metricbeat.yml` points at `localhost:9200`, with `#protocol: "https"` and auth commented. Edit **`output.elasticsearch` only** (do not uncomment SSL under `#output.logstash:`).

Keep `metricbeat.config.modules`, `setup.template.settings` and `processors`. Do not add a second `metricbeat.modules:` block in this file: the `system` module comes from `modules.d/system.yml` (cpu, load, memory, network, process, `socket_summary`, filesystem). An extra block collects the same metrics twice.

```bash
sudo nano /etc/metricbeat/metricbeat.yml
```

Same CA / user as Filebeat:

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

Check with `sudo metricbeat test output`.
#### Step 3: Enable the system module

Enable the `system` module (host metrics). Without it, Metricbeat runs but sends almost nothing useful for the lab dashboard.

```bash
sudo metricbeat modules enable system
```

#### Step 4: System module configuration

By default the system module collects CPU, memory, disk, network, processes and filesystem. The default file is enough. Open it only if you want to change periods or turn a metric set off:

```bash
sudo nano /etc/metricbeat/modules.d/system.yml
```

#### Step 5: Start Metricbeat

Enable and start the service, check its status, then test the connection to Elasticsearch.

```bash
sudo systemctl enable --now metricbeat
sudo systemctl status metricbeat
sudo metricbeat test output
```

Data streams: `.ds-metricbeat-*` (Kibana pattern: `metricbeat-*`).

#### Step 6: Check in Kibana

1. Create a data view: `metricbeat-*` (`@timestamp`)
2. Explore in **Discover** (`event.dataset: system.cpu`, `system.memory`, …)
3. Open (or create) your metrics dashboard (see §4.3)

---

### 3.4 Packetbeat (optional — not on the Linux checklist)

> **Optional**: Packetbeat is **not** a required step of the Linux session.  
> Filebeat (Suricata / auth / Apache) + Metricbeat are enough for the deliverable.  
> Turn it on only if you have RAM and time left. It needs extra privileges and can be heavy.

If you still want to try it, install Packetbeat and edit its config (interface, protocols, SIEM output):

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

Also copy the CA into `/etc/packetbeat/certs/`.  
`device: any`: listen on every interface. That costs CPU and RAM.

Start the service so captured traffic is sent to Elasticsearch:

```bash
sudo systemctl enable --now packetbeat
```

Check: data view `packetbeat-*` in Kibana.

---

## 4. Part 3: Visualise and analyse in Kibana

### 4.1 Create data views

#### Step 1: Open data views

1. In Kibana, go to **Stack Management → Data Views**
2. Click **Create data view**

#### Step 2: Filebeat data view

1. Name pattern: `filebeat-*`
2. Click **Next step**
3. Timestamp field: `@timestamp`
4. Click **Create data view**

#### Step 3: The other data views

Repeat for:
- `winlogbeat-*`
- `metricbeat-*`
- `packetbeat-*` (if installed)

### 4.2 Basic visualisations

#### Visualisation 1: Logs over time

1. Go to **Dashboard** > **Create visualization**
2. Choose **Line**
3. Select the `filebeat-*` data view
4. Configure:
   - **Y-axis**: Count
   - **X-axis**: Date Histogram on `@timestamp`
5. Click **Save** and name it "Logs over time"

#### Visualisation 2: Top 10 log sources

1. Create a **Data Table** visualisation
2. Select `filebeat-*`
3. Configure:
   - **Metric**: Count
   - **Buckets**: Terms on `host.name` (or `source`)
4. Limit to 10 results
5. Save as "Top 10 log sources"

#### Visualisation 3: Breakdown by log type

1. Create a **Pie** visualisation
2. Select `filebeat-*`
3. Configure:
   - **Slice by**: Terms on `fileset.name` or `log.file.path`
4. Save as "Log types"

### 4.3 Build a dashboard

#### Step 1: Create the dashboard

1. Go to **Dashboard** > **Create dashboard**
2. Click **Add** to add visualisations
3. Add the visualisations you just saved

#### Step 2: Lay it out

- Group the panels in an order you can explain
- Resize the panels
- Title: "SIEM dashboard — overview"

#### Step 3: Save

Save the dashboard as "SIEM dashboard — main"

#### Dashboards you build yourself

Create your own dashboards in Kibana (**Analytics → Dashboard**). Do **not** count on imported files from someone else: on Kibana **9.5**, `filebeat setup --dashboards` / `metricbeat setup --dashboards` often fails (`fail to parse kibana version`), and that is **not** the expected deliverable.

Data views to create (§4.1): `filebeat-*`, `metricbeat-*`, `winlogbeat-*` (time field `@timestamp`).

Minimum target (any titles, as long as the content is there):

| Dashboard | Sources | Useful panels / filters |
| --- | --- | --- |
| Linux / security overview | Suricata, SSH/auth, Apache, Metricbeat | `event.kind: "alert"`, `source.ip`, SSH auth, `http.response.status_code`, CPU/memory |
| Host metrics | `metricbeat-*` | CPU, memory, load, disk `/`, network, processes (~24 h window) |
| Windows | `winlogbeat-*` | `winlog.event_id: 4624` / `4625`, `winlog.channel` |
| GeoIP (if Suricata + public traffic) | `filebeat-*` | maps `destination.geo.location` / `source.geo.location`, country / IP |

Tip: in **Discover**, filter, **Save** the search, then **Add from library** on the dashboard.

### 4.4 Search and analyse logs

#### Simple search

1. Go to **Discover**
2. Select the `filebeat-*` data view
3. Use the search bar:
   - `status:error` (error logs)
   - `message:*failed*` (messages containing "failed")
   - `@timestamp:[now-1h TO now]` (last hour)

#### Advanced search with KQL

KQL is the Kibana Query Language:

```
host.name: "server-name" and message: "error"
```

```
@timestamp >= "now-24h" and log.level: "error"
```

#### Correlation

1. Look for suspicious patterns:
   - Repeated authentication failures: `event.action: "authentication_failure"`
   - Attempts to open sensitive files
   - Unusual network activity

2. Use filters to narrow the search

### 4.5 Security dashboard

**Task (Linux)** — build a "Linux security" dashboard with at least:
- Suricata alerts over time (`event.kind: "alert"`, breakdown on `rule.name`)
- Top source IPs (`source.ip`)
- `system.auth` events (SSH)
- Apache access (`http.response.status_code`)
- Metricbeat metrics (CPU / memory)

**Task (Windows — Winlogbeat §3.2)** — build a Windows dashboard with:
- Successful logons (`winlog.event_id: 4624`)
- Authentication failures (`winlog.event_id: 4625`)
- Security log / channel breakdown (`winlog.channel`)
- Source host (`host.name`)

Check that the panels show documents. Optional Kibana alerts (thresholds) depend on licence / version.

**Hint**: Windows fields `winlog.event_id`, `winlog.channel`; Suricata ECS fields `rule.name`, `event.kind`.

### 4.6 GeoIP map: where traffic goes, who arrives

The Filebeat Suricata ingest pipeline (`filebeat-*-suricata-eve-pipeline`) already runs a **GeoIP** processor on `source.ip` and `destination.ip`. Elasticsearch keeps the GeoLite2 databases up to date (City, Country, ASN). You do not create a MaxMind account.

Private addresses (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) get no coordinates. Only public addresses show up on the map:

- **outbound** traffic: `destination.geo.location` (the source is the VM, so it is private);
- connections **to the VM**: `source.geo.location` (the destination is the VM, so it is private).

Build a GeoIP dashboard (data view `filebeat-*`, ~24 h window) with at least: a destination map (`destination.geo.location`), a source map (`source.geo.location`), destination/source country and IP tables, and (if present) alerts `rule.name: "ESAIP-LAB-SSH-PROBE"`.

A website behind a CDN often geolocates to the United States or France, not to the country in the domain name. The country on the map is the country of the IP that was actually contacted.

#### Outbound traffic

Run the requests **from the monitored VM**. A `curl` to `localhost` does not cross `eth0`, so Suricata does not see it.

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

- `-L`: follow redirects; `--max-redirs 2`: stop after two hops.
- Any HTTP status (even `403`, or a TLS failure after the SYN) is enough: Suricata records the flow, and the pipeline fills `destination.geo.*`.

Check in Discover: `event.dataset: "suricata.eve" and source.ip: "IP_SURVEILLEE" and destination.geo.country_name: *`.

#### Inbound connections

The VM is not reachable from the Internet. To plot “who connects”, send **a few TCP SYNs to port 22 of the monitored VM only**, with a public address in the IP source field. The handshake does not complete: no password is tried, and `sshd` never sees a session.

Check first that nothing will drop these packets:

- Suricata stays in IDS mode: in `/etc/suricata/suricata.yaml`, `copy-mode: ips` stays commented, and `af-packet` listens on `eth0`.
- `rule-files` loads only `alert` rules (no active `drop` / `reject`).
- UFW is inactive and the iptables policy is `ACCEPT`.

Add a lab alert, not a block. File `/var/lib/suricata/rules/lab-geo.rules`:

```
alert tcp !$HOME_NET any -> $HOME_NET 22 (msg:"ESAIP-LAB-SSH-PROBE"; flags:S; sid:9000002; rev:1;)
```

In `rule-files`, next to `lab-test.rules`:

```yaml
rule-files:
  - lab-test.rules
  - lab-geo.rules
```

Then `sudo suricatasc -c reload-rules`. `$HOME_NET` already includes `10.0.0.0/8` and `172.16.0.0/12`, so a real SSH session from the lab network does not match. Suricata may warn that the to-client direction is ignored for a bare SYN; SYNs toward the server are still detected.

From the workstation, forging the source address needs root (`nping` reports `Mode TCP requires root privileges`), and the campus gateway may drop a packet whose source is not the workstation’s own address. Send the SYNs **from the SIEM VM**, on the same LAN as the sensor, only toward `IP_SURVEILLEE` port 22. Three SYNs per address are enough.

```bash
# On the SIEM VM, as root. Replace IP_SURVEILLEE.
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
print("sent", 3 * len(sources))
PY
```

On the sensor, `tcpdump -ni eth0 'tcp dst port 22'` shows the SYNs, and `eve.json` contains `ESAIP-LAB-SSH-PROBE`. In Kibana: `rule.name: "ESAIP-LAB-SSH-PROBE" and source.geo.country_name: *`.

---

## 5. Part 4: Practical case

### 5.1 Scenario: spot an intrusion attempt

#### Context

You are the system administrator. You have to read the logs and decide whether something suspicious happened. A user reports odd connections to a server.

#### Data you already have

- System logs (Filebeat)
- Windows events (Winlogbeat)
- Host metrics (Metricbeat)
- Suricata alerts, if you finished §3.1

#### Mission

Find the signs of an intrusion attempt by reading those logs. Use the events this lab already produced. Do not invent a second attack.

### 5.2 Correlate the logs

#### Step 1: Authentication failures

1. In Kibana, open **Discover**
2. Select the `winlogbeat-*` data view
3. Search for failed logons:

```
winlog.event_id: 4625
```

4. Look at:
   - How many failures
   - Source IPs
   - Targeted accounts
   - When they happened

#### Step 2: Network activity

1. If Packetbeat is installed, look at the captured traffic
2. Otherwise use the Suricata alert and Metricbeat network graphs
3. Look for:
   - Connections from unexpected addresses
   - Unusual ports
   - Traffic volume that does not match the rest of the lab

#### Step 3: System logs

1. In `filebeat-*`, look for:
   - Attempts to open sensitive files
   - Configuration changes
   - Commands you did not expect

#### Step 4: Time correlation

1. Use the time filter to isolate the busy window
2. Line up events from Filebeat, Winlogbeat and Metricbeat
3. Say what the sequence is, in order

### 5.3 Simple correlation rules

#### Rule 1: Brute force

**Logic**: more than 5 failed logons from the same IP in 10 minutes.

**In Kibana**:
1. Create a **Data Table** visualisation
2. Configure:
   - **Metric**: Count
   - **Buckets**: Terms on `source.ip` (or `winlog.event_data.IpAddress`)
   - **Filter**: `winlog.event_id: 4625`
3. Time filter: `@timestamp:[now-10m TO now]`
4. Find IPs with more than 5 hits

#### Rule 2: Activity outside office hours

**Logic**: successful logons outside office hours (for example 22:00–06:00).

**How**:
1. Search successful logons: `winlog.event_id: 4624`
2. Time filter: `@timestamp:[now-24h TO now]`
3. Read the hours of those logons

### 5.4 Tell the story of the events

**Scenario**: the lab events are your evidence. Connect them so you can say what happened, instead of clicking through screens that do not relate.

**Task**: start from events you already produced (Suricata alert `ESAIP-LAB-HTTP-TEST`, `system.auth` lines, Windows failure `4625`).

1. In Kibana, save searches that show:
   - Source IPs
   - Targeted accounts
   - What raised the event (Suricata rule, authentication failure, and so on)
   - The timeline
2. Write a short report:
   - Summary
   - Timeline
   - What you would do next (block, investigate, or ignore a lab test)
3. Put those searches on a dedicated dashboard

**Hint**: save the Discover searches so the report points at something you can reopen.

---

## 6. Appendices

### 6.1 Useful commands

A short troubleshooting sheet for the session. Each block groups the usual actions (status, restart, logs, test).

#### Elasticsearch

Service status and restart, follow the logs (`-f` means follow), list indexes, delete an index (this cannot be undone):

```bash
# Status
sudo systemctl status elasticsearch

# Restart
sudo systemctl restart elasticsearch

# Logs
sudo journalctl -u elasticsearch -f

# List indexes
curl -sk -u elastic:$ELASTIC_PASS --cacert /etc/elasticsearch/certs/http_ca.crt https://localhost:9200/_cat/indices?v

# Delete an index (careful)
curl -sk -u elastic:$ELASTIC_PASS --cacert /etc/elasticsearch/certs/http_ca.crt -X DELETE https://localhost:9200/index-name
```

- `_cat/indices?v`: table of indexes (`v` adds column headers).
- `-X DELETE`: HTTP DELETE. Replace `index-name`. Do not delete system indexes (`.kibana*`, and so on) unless you mean to.

#### Kibana

Status, restart and logs of the Kibana service:

```bash
# Status
sudo systemctl status kibana

# Restart
sudo systemctl restart kibana

# Logs
sudo journalctl -u kibana -f
```

#### Logstash

Status, restart, config test outside the service (`-f` is the pipeline file; `--config.test_and_exit` checks the config and exits without starting the pipeline), logs:

```bash
# Status
sudo systemctl status logstash

# Restart
sudo systemctl restart logstash

# Test a configuration
sudo /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/test.conf --config.test_and_exit

# Logs
sudo journalctl -u logstash -f
```

#### Filebeat

Status, YAML check, module list, logs:

```bash
# Status
sudo systemctl status filebeat

# Test the configuration
sudo filebeat test config

# List modules
sudo filebeat modules list

# Logs
sudo journalctl -u filebeat -f
```

#### Metricbeat

Status, config check, module list:

```bash
# Status
sudo systemctl status metricbeat

# Test the configuration
sudo metricbeat test config

# List modules
sudo metricbeat modules list
```

#### Winlogbeat (Windows PowerShell)

Service status, restart, config test from the install directory:

```powershell
# Status
Get-Service winlogbeat

# Restart
Restart-Service winlogbeat

# Test the configuration
cd "C:\Program Files\Winlogbeat"
.\winlogbeat.exe test config -c .\winlogbeat.yml
```

### 6.2 Logstash 9 — obsolete SSL options (Elasticsearch output)

On Logstash 9.x these settings **prevent the plugin from starting**. This lab already uses the new names; do not put the old ones back from outdated tutorials.

| Obsolete (do not use) | Replace with |
| --- | --- |
| `cacert` | `ssl_certificate_authorities` (list of PEM paths) |
| `ssl => true` / `ssl => false` | `ssl_enabled => true` / `false` |
| `ssl_certificate_verification` | `ssl_verification_mode` (`full` or `none`) |
| `keystore` / `keystore_password` | `ssl_keystore_path` / `ssl_keystore_password` |
| `truststore` / `truststore_password` | `ssl_truststore_path` / `ssl_truststore_password` |

Still valid in this lab: `hosts`, `user`, `password`, `index`, `ssl_enabled`, `ssl_certificate_authorities`.

**Do not confuse** with other tools (different syntax, still correct):
- `curl --cacert …`
- `elasticsearch-certutil --ca-cert …`
- Beats / Winlogbeat: YAML `ssl:` block with `enabled`, `certificate_authorities`, `verification_mode`
- Kibana: `elasticsearch.ssl.certificateAuthorities`, `elasticsearch.ssl.verificationMode`

### 6.3 Troubleshooting

#### Problem: Elasticsearch does not start

**Symptoms**: service `failed`, or a memory error.

**What to try**:
1. Logs: `sudo journalctl -u elasticsearch -n 50`
2. Free memory: `free -h`
3. Lower the heap in `/etc/elasticsearch/jvm.options`
4. Permissions: `sudo chown -R elasticsearch:elasticsearch /var/lib/elasticsearch`

#### Problem: Kibana cannot connect to Elasticsearch

**Symptoms**: "Unable to connect to Elasticsearch".

**What to try**:
1. Elasticsearch is up: `curl -sk -u elastic:$ELASTIC_PASS --cacert /etc/elasticsearch/certs/http_ca.crt https://localhost:9200`
2. In `kibana.yml`, `elasticsearch.hosts` must be uncommented:

   ```bash
   sudo grep "^elasticsearch.hosts" /etc/kibana/kibana.yml
   ```

   The line must not start with `#`.
3. Logs: `sudo journalctl -u kibana -n 50`
4. Firewall

#### Problem: Kibana stays on the loading page

**Symptoms**: the page opens but stays on "Loading..." or "Kibana is starting", even after several minutes.

**Possible causes**:
1. Saved-object migrations are still running (normal on the first start)
2. Kibana waits for those migrations before showing the UI
3. The VM is short on memory

**What to try**:

1. **Watch migrations in the logs** — keep only migration lines from the Kibana journal:

   ```bash
   sudo journalctl -u kibana -f | grep -i "migration\|savedobjects"
   ```

   `| grep -i "..."` keeps lines that contain those words, ignoring case. You should see "Starting saved objects migrations", then "Migration completed" or "Migration successful".

2. **Wait 3–5 minutes** on the first start. Migrations are slow when RAM is tight.

3. **Confirm Kibana reached Elasticsearch**:

   ```bash
   sudo journalctl -u kibana | grep -i "connected to elasticsearch"
   ```

   You should see "Successfully connected to Elasticsearch".

4. **Check the Kibana service**:

   ```bash
   sudo systemctl status kibana
   ```

   It must be "active (running)".

5. **Read the full recent log** — last 200 lines, no pager:

   ```bash
   sudo journalctl -u kibana -n 200 --no-pager
   ```

   Look for ERROR, FATAL, or serious warnings.

6. **If migrations look stuck** (more than 10 minutes), you can reset them. **Warning**: this deletes every Kibana configuration.

   ```bash
   # Stop Kibana
   sudo systemctl stop kibana

   # Delete Kibana indexes (all visualisations, dashboards and saved objects are lost)
   curl -sk -u elastic:$ELASTIC_PASS --cacert /etc/elasticsearch/certs/http_ca.crt -X DELETE "https://localhost:9200/.kibana*"

   # Start Kibana
   sudo systemctl start kibana

   # Wait 3–5 minutes for the new migrations
   ```

   `.kibana*`: Kibana's internal indexes (saved objects). Do this only as a last resort in the lab.

7. **Check free memory**:

   ```bash
   free -h
   ```

   If memory is full, migrations are very slow. Consider lowering the Elasticsearch heap.

**Note**: "Starting saved objects migrations" with no completion line yet is normal. Migrations can take several minutes. Wait, and keep an eye on the logs.

#### Problem: `filebeat test output` fails (TLS / HTTPS)

**Symptoms**: `connection... ERROR`, `x509: …`, or plain HTTP while ES expects HTTPS.

**Common causes** (**default package** YAML):
1. `hosts` still `localhost:9200` without `https://`, and `#protocol: "https"` still commented
2. `#username` / `#password` still commented, or using `elastic` instead of `beats_writer`
3. Missing `ssl:` / CA under `output.elasticsearch` (stock `#ssl.certificate_authorities` lines sit under **Logstash**, not Elasticsearch)
4. Missing CA file: `/etc/filebeat/certs/http_ca.crt`

**What to try**: redo §3.1 step 2; recopy the CA; run `sudo filebeat test output` until `handshake... OK` and `talk to server... OK`.

#### Problem: logs do not show up in Kibana

**Symptoms**: Discover is empty.

**What to try**:
1. Beats are running: `sudo systemctl status filebeat`
2. Indexes exist: `curl -sk -u elastic:$ELASTIC_PASS --cacert /etc/elasticsearch/certs/http_ca.crt https://localhost:9200/_cat/indices?v`
3. Recheck the Beats configuration
4. Beats logs: `sudo journalctl -u filebeat -f`

#### Problem: Winlogbeat collects nothing

**Symptoms**: no Windows events in Kibana.

**What to try**:
1. Service: `Get-Service winlogbeat`
2. Logs: `Get-EventLog -LogName Application -Source winlogbeat`
3. Config: `.\winlogbeat.exe test config`
4. Path to Elasticsearch: `Test-NetConnection -ComputerName IP_SIEM -Port 9200`

#### Problem: the VM is slow on 2 GB of RAM

**Symptoms**: the system is slow, services crash.

**What to try**:
1. Lower the Elasticsearch heap (for example 256 MB if Kibana and Logstash run at the same time)
2. Limit how many indexes you keep
3. Turn off features you are not using (machine learning, and so on)
4. Add swap if you must (it will be slower)
5. See how much RAM Kibana and Logstash are using

### 6.3 Further reading

#### Official documentation

- **Elastic Stack**: https://www.elastic.co/guide/
- **Elasticsearch**: https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html
- **Logstash**: https://www.elastic.co/guide/en/logstash/current/index.html
- **Kibana**: https://www.elastic.co/guide/en/kibana/current/index.html
- **Beats**: https://www.elastic.co/guide/en/beats/index.html

#### Grok patterns

- **Grok Debugger**: https://grokdebug.herokuapp.com/
- **Base patterns**: https://github.com/elastic/logstash/blob/v1.4.2/patterns/grok-patterns

#### Communities

- **Elastic forum**: https://discuss.elastic.co/
- **Stack Overflow**: tags `elasticsearch`, `logstash`, `kibana`

#### Extra tools

- **Elasticsearch Head**: a plugin to browse Elasticsearch
- **Cerebro**: a web UI to manage Elasticsearch
- **Elasticsearch Curator**: a tool to manage indexes

### 6.5 End-of-lab checklist

Before you finish, confirm that:

- [ ] Elasticsearch is installed and answering on the SIEM VM
- [ ] Kibana opens in your browser (`http://IP_SIEM:5601`)
- [ ] Logstash has at least one pipeline (test or beats)
- [ ] Filebeat on the monitored VM: modules `system`, `apache`, `suricata`
- [ ] Suricata alerts are visible (`event.kind: "alert"`)
- [ ] SSH / auth traces are visible (`event.dataset: "system.auth"`)
- [ ] Apache logs are visible (`event.dataset: "apache.access"`)
- [ ] Metricbeat is collecting host metrics (`metricbeat-*`)
- [ ] A Linux / security dashboard **you** built (at least 3 visualisations)
- [ ] A GeoIP dashboard **you** built: destination points (outbound) and source points (inbound to the VM)
- [ ] Winlogbeat is installed (same branch as the SIEM), the service is **Running**, data view `winlogbeat-*` exists
- [ ] A Windows dashboard **you** built (4624 / 4625 / channels)
- [ ] Configurations and findings are written down, including the Windows source IP towards SIEM:9200
*Lab written for course unit S10-3 — Information security and event management*  
*ESAIP — Computer Science and Networks*
