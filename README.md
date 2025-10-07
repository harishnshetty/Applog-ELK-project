# 🚀 App Log ELK Project

## For more projects, check out  
[https://harishnshetty.github.io/projects.html](https://harishnshetty.github.io/projects.html)

[![Video Tutorial](https://github.com/harishnshetty/image-data-project/blob/8391945b71df68d8741bf225ce4af892cd308b99/awsresourcefinder.jpg)](https://youtu.be/KNH_qe1vJAg)

## Required Setup

This document provides a step-by-step guide to setting up the ELK Stack (Elasticsearch, Logstash, Kibana) with Filebeat to monitor logs from a Java application running on AWS EC2 (Ubuntu).

### 1. Overview of ELK Stack

The ELK Stack consists of:
- **Elasticsearch** → Stores and indexes logs.
- **Logstash** → Processes and transforms logs before storing them in Elasticsearch.
- **Kibana** → Provides visualization and analysis of logs.
- **Filebeat** → Forwards logs from the application to Logstash.

### 2. Infrastructure Setup

We are using three EC2 Ubuntu machines:
1. **ELK Server** → Hosts Elasticsearch, Logstash, Kibana.
2. **Client Machine** → Hosts Java application and Filebeat.
3. **Web Server (Optional)** → Hosts an additional application for testing logs.

### 3. Step-by-Step Installation

#### Step 1: Install & Configure Elasticsearch (ELK Server)

**1.1 Install Java (Required for Elasticsearch & Logstash)**
```sh
sudo apt update && sudo apt install openjdk-17-jre-headless -y
```

```bash
apt-get update && apt dist-upgrade -y && apt-get install -y vim curl zip gnupg gpg 
```

```bash
nano /etc/environment
```
```bash
JAVA_HOME="/usr/lib/jvm/java-17-openjdk-amd64"
```
```bash
source /etc/environment
echo $JAVA_HOME
```

```bash
sudo ufw allow 80
sudo ufw allow 443
sudo ufw allow 22
sudo ufw allow 5601/tcp       # Kibana
sudo ufw allow 9200/tcp       # Elasticsearch
sudo ufw allow 5044/tcp       # Logstash
sudo ufw enable
```
**1.2 Install Elasticsearch**

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

sudo apt-get install apt-transport-https
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/9.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-9.x.list

sudo apt-get update && sudo apt-get install elasticsearch -y

```

**1.4 Start & Enable Elasticsearch**
```sh
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch
sudo systemctl status elasticsearch
```

## Auto Generated password [ Sample ]
- The generated password for the elastic built-in superuser is : In7bFzU8EyXJ+71ImQFR 



```bash
cp /etc/elasticsearch/elasticsearch.yml /etc/elasticsearch/elasticsearch_backup.yml
```




**1.3 Configure Elasticsearch**
```sh
sudo nano /etc/elasticsearch/elasticsearch.yml
```
Modify:
```bash
network.host: 0.0.0.0
discovery.seed_hosts: ["127.0.0.1", "[::1]","host1", "host2"]
```
```bash
sudo chown -R elasticsearch:elasticsearch /etc/elasticsearch /var/lib/elasticsearch /var/log/elasticsearch
```

```bash
systemctl restart elasticsearch
sudo systemctl status elasticsearch
```

```bash
curl -X GET "localhost:9200"
```
## for the Reset
```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic -i
```

**1.5 Verify Elasticsearch**
```sh
sudo curl -X GET -u elastic:password https://localhost:9200 --cacert /etc/elasticsearch/certs/http_ca.crt
```

#### Step 2: Install & Configure Logstash (ELK Server)

**2.1 Install Logstash**
```sh
sudo apt install logstash -y
```

**2.2 Configure Logstash to Accept Logs**
```sh
sudo nano /etc/logstash/conf.d/logstash.conf
```
Add:
```sh
input {
  beats {
    port => 5044
  }
}

filter {
  grok {
    match => { "message" => "%{TIMESTAMP_ISO8601:log_timestamp} %{LOGLEVEL:log_level} %{GREEDYDATA:log_message}" }
  }
}

output {
  elasticsearch {
    hosts => ["https://localhost:9200"]
    user => "elastic"                     # or "logstash_internal" if you created one
    password => "your_actual_password"    # replace with the real password
    ssl => true
    cacert => "/etc/elasticsearch/certs/http_ca.crt"
    index => "logs-%{+YYYY.MM.dd}"
  }

  stdout {
    codec => rubydebug
  }
}
```

**2.3 Start & Enable Logstash**
```sh
sudo systemctl start logstash
sudo systemctl enable logstash
sudo systemctl restart logstash
sudo systemctl status logstash
```


#### Step 3: Install & Configure Kibana (ELK Server)

**3.1 Install Kibana**
```sh
sudo apt install kibana -y
```

**3.2 Configure Kibana**
```sh
sudo nano /etc/kibana/kibana.yml
```
Modify:
```
server.port: 5601
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://localhost:9200"]
```

**3.3 Start & Enable Kibana**
```sh
sudo systemctl start kibana
sudo systemctl enable kibana
sudo systemctl restart kibana
sudo systemctl status kibana
```

**3.5 Access Kibana Dashboard**

Open a browser and go to:  
`http://<ELK_Server_Public_IP>:5601`

```bash
/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
```
```bash
/usr/share/kibana/bin/kibana-verification-code
```
#### Step 4: Install & Configure Filebeat (Client Machine)

**4.1 Install Filebeat**
```sh
sudo apt install filebeat -y
```

**4.2 Configure Filebeat to Send Logs to Logstash**
```sh
sudo nano /etc/filebeat/filebeat.yml
```
Modify:
```bash
# ============================== Filebeat inputs ===============================

filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/demoapp.log
    fields:
      app: demoapp
    fields_under_root: true

# ============================== Filebeat modules ==============================

filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false

# ============================== Filebeat outputs ==============================

# Comment Elasticsearch output
# output.elasticsearch:
#   hosts: ["localhost:9200"]

# Enable Logstash output
output.logstash:
  hosts: ["192.168.64.2:5044"]

# ============================== Logging =======================================

logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0644

```

**4.3 Start & Enable Filebeat**
```sh
sudo systemctl start filebeat
sudo systemctl enable filebeat
sudo systemctl restart filebeat
sudo systemctl status filebeat
```

**4.4 Verify Filebeat is Sending Logs**
```sh
sudo filebeat test config
```
## Config OK

```bash
curl -u elastic:password -X GET "http://localhost:9200/_cluster/health?pretty"
```
#### Step 5: Deploy node Application & Generate Logs

**5.1 Install not (If Not Installed)**
```sh
sudo apt update
sudo apt install -y nodejs npm
node -v
npm -v


mkdir ~/demo-node-app
cd ~/demo-node-app
nano app.js

```
```bash
const http = require('http');
const fs = require('fs');
const path = '/var/log/demoapp.log';

const server = http.createServer((req, res) => {
  const log = `${new Date().toISOString()} ${req.method} ${req.url}\n`;
  fs.appendFileSync(path, log);
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello from Demo Node.js App!\n');
});

server.listen(3000, () => {
  console.log('Demo Node.js app running on port 3000');
});

```
```bash
sudo node app.js
```
```bash
curl http://localhost:3000/test
curl http://localhost:3000/hello
curl http://localhost:3000/metrics
```
```bash
/var/log/demoapp.log
```


#### Step 6: View & Analyze Logs in Kibana

**6.1 Open Kibana Discover**
1. Go to Kibana → Discover.
2. Select `log*` index.
3. Search for:  
   `log.file.path: "/home/ubuntu/Boardgame/target/app.log"`
4. View structured fields (`log_timestamp`, `log_level`, `log_message`).

**6.2 Create Kibana Visualizations**
1. Pie Chart → Log level distribution.
2. Line Chart → Logs over time.
3. Data Table → Structured log table.

**6.3 Create a Kibana Dashboard**
1. Go to Kibana → Dashboard → Create Dashboard.
2. Add Pie Chart, Line Chart, Data Table.
3. Save as "Java Application Log Monitoring".

---

## Conclusion

You have successfully:
- Installed Elasticsearch, Logstash, Kibana, and Filebeat
- Set up a Java application to generate logs
- Parsed logs into structured fields using Grok
- Created a real-time Kibana dashboard for log monitoring