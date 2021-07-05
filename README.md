<!--
+----------+-----------+------------+--------------+----------+
| Server   | Node Name | Attributes | Roles        | JVM Heap |
+----------+-----------+------------+--------------+----------+
| master-1 | master-1  |            | master       | 768m     |
+----------+-----------+------------+--------------+----------+
| data-1   | data-1    | temp=hot   | data, ingest | 2g       |
+----------+-----------+------------+--------------+----------+
| data-2   | data-2    | temp=warm  | data, ingest | 2g       |
+----------+-----------+------------+--------------+----------+
-->

## Installing Elasticsearch in RPM systems

- Import the gpg key:

```bash
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

- Download the Elasticsearch package:

```bash
curl --remote-name \
  https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.6.0-x86_64.rpm
```

- Install the Elasticsearch package:

```bash
sudo rpm --install elasticsearch-7.6.0-x86_64.rpm
```

- Enable Elasticsearch service:

```bash
sudo systemctl enable elasticsearch
```

## Custom configurations:

### Master node:

- Elasticsearch config:

```yml
# /etc/elasticsearch/elasticsearch.yml

cluster.name: playground
node.name: master-1
network.host: [ _local_, _site_ ]
# Master IP addresses
discovery.seed_hosts: [ "<PRIVATE_MASTER_IP_ADDR>" ]
cluster.initial_master_nodes: [ "master-1" ]
node.master: true
node.data: false
node.ingest: false
node.ml: false
```

- JVM config:

```bash
# /etc/elasticsearch/jvm.options

# Xms Initial size of total heap space
# Xmx Maximum size of total heap space

-Xms768m
-Xmx768m
```

- Start the Elasticsearch service:

```bash
sudo systemctl start elasticsearch
```

### Data node 1:

- Elasticsearch config:

```yml
# /etc/elasticsearch/elasticsearch.yml

cluster.name: playground
node.name: data-1
node.attr.temp: hot
network.host: [ _local_, _site_ ]
# Master IP addresses
discovery.seed_hosts: [ "<PRIVATE_MASTER_IP_ADDR>" ]
cluster.initial_master_nodes: [ "master-1" ]
node.master: false
node.data: true
node.ingest: true
node.ml: false
```

- JVM config:

```bash
# /etc/elasticsearch/jvm.options

# Xms Initial size of total heap space
# Xmx Maximum size of total heap space

-Xms1g # default
-Xmx1g # default
```

- Start the Elasticsearch service:

```bash
sudo systemctl start elasticsearch
```

### Data node 2:

- Elasticsearch config:

```yml
# /etc/elasticsearch/elasticsearch.yml

cluster.name: playground
node.name: data-2
node.attr.temp: warm
network.host: [ _local_, _site_ ]
# Master IP addresses
discovery.seed_hosts: [ "<PRIVATE_MASTER_IP_ADDR>" ]
cluster.initial_master_nodes: [ "master-1" ]
node.master: false
node.data: true
node.ingest: true
node.ml: false
```

- JVM config:

```bash
# /etc/elasticsearch/jvm.options

# Xms Initial size of total heap space
# Xmx Maximum size of total heap space

-Xms1g # default
-Xmx1g # default
```

- Start the Elasticsearch service:

```bash
sudo systemctl start elasticsearch
```

## Testing Cluster

- Check the logs at:

```bash
less /var/log/elasticsearch/playground.log
```

- Check with local requests:

```bash
curl http://localhost:9200/_cat/nodes?v
curl http://localhost:9200/_cat/health?v
```

## Installing Kibana in RPM systems

- On master node, download the Kibana package:

```bash
curl --remote-name https://artifacts.elastic.co/downloads/kibana/kibana-7.6.0-x86_64.rpm
```

- Install the Kibana package:

```bash
sudo rpm --install kibana-7.6.0-x86_64.rpm
```

- Enable Kibana service:

```bash
sudo systemctl enable kibana
```

- Kibana config:

```yml
# /etc/kibana/kibana.yml

server.port: 8080
server.host: "<PRIVATE_MASTER_IP_ADDR>"
elasticsearch.hosts: [ "http://localhost:9200" ]
```

- Start the Kibana service:

```bash
sudo systemctl start kibana
```

## Creating a Public Key Infrastructure (PKI)

- Create a Certificate Authority file:

```bash
/usr/share/elasticsearch/bin/elasticsearch-certutil ca \
  --out /etc/elasticsearch/ca \
  --pass elastic_ca
```

- Create a certificate for master node:

```bash
/usr/share/elasticsearch/bin/elasticsearch-certutil cert \
  --ca /etc/elasticsearch/ca \
  --ca-pass elastic_ca \
  --name master_1 \
  --dns <MASTER_DNS_ADDRESS> \
  --ip <MASTER_PRIVATE_IP_ADDRESS> \
  --out /etc/elasticsearch/master-1 \
  --pass elastic_master_1
```

- Create certificates for data nodes:

```bash
# Data 1

/usr/share/elasticsearch/bin/elasticsearch-certutil cert \
  --ca /etc/elasticsearch/ca \
  --ca-pass elastic_ca \
  --name data_1 \
  --dns <DATA_1_DNS_ADDRESS> \
  --ip <DATA_1_PRIVATE_IP_ADDRESS> \
  --out /etc/elasticsearch/data-1 \
  --pass elastic_data_1

# Data 2

/usr/share/elasticsearch/bin/elasticsearch-certutil cert \
  --ca /etc/elasticsearch/ca \
  --ca-pass elastic_ca \
  --name data_2 \
  --dns <DATA_2_DNS_ADDRESS> \
  --ip <DATA_2_PRIVATE_IP_ADDRESS> \
  --out /etc/elasticsearch/data-2 \
  --pass elastic_data_2
```

- Move certificates to respective nodes:

```bash
# Data 1

chown <YOUR_USER>:<YOUR_USER> data-1

scp data-1 <DATA_1_PRIVATE_IP_ADDRESS>:/tmp

chown root:elasticsearch data-1 # at data 1 node and move to /etc/elasticsearch

# Data 2

chown <YOUR_USER>:<YOUR_USER> data-2

scp data-2 <DATA_2_PRIVATE_IP_ADDRESS>:/tmp

chown root:elasticsearch data-2 # at data 2 node and move to /etc/elasticsearch
```

## Encrypt the Transport Network

- Change certificate permissions file:

```bash
chmod 640 [master-1, data-1, data-2]
```

### Node Configurations:

- Elasticsearch config:

```yml
# /etc/elasticsearch/elasticsearch.yml

xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: full
xpack.security.transport.ssl.keystore.path: [master-1, data-1, data-2] # certificate name
xpack.security.transport.ssl.truststore.path: [master-1, data-1, data-2] # certificate name
```

- Add pass-phrase to Elasticsearch keystore:

```bash
# keystore

/usr/share/elasticsearch/bin/elasticsearch-keystore add \
  xpack.security.transport.ssl.keystore.secure_password

> type the pass-phrase


# truststore

/usr/share/elasticsearch/bin/elasticsearch-keystore add \
  xpack.security.transport.ssl.truststore.secure_password

> type the pass-phrase
```

- Restart Elasticsearch:

```bash
sudo systemctl restart elasticsearch
```

## Define Elasticsearch user passwords

- Redefine default passwords:

```bash
# elastic, apm_system, kibana, logstash_system, beats_system, remote_monitoring_user

/usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive
```

- Kibana config:

```yml
# /etc/kibana/kibana.yml

elasticsearch.username: "kibana"
elasticsearch.password: "<KIBANA_USER_PASSWORD>"
```

- Restart Kibana:

```bash
sudo systemctl restart kibana
```

## Encrypt the Client Network

**IMPORTANT**: In a production environment, is highly recommend to use a global signed certificate, instead of a self-signed certificate.

### Node Configurations:

- Elasticsearch config:

```yml
# /etc/elasticsearch/elasticsearch.yml

xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: [master-1, data-1, data-2]
xpack.security.http.ssl.truststore.path: [master-1, data-1, data-2]
```

- Add pass-phrase to Elasticsearch keystore:

```bash
# keystore

/usr/share/elasticsearch/bin/elasticsearch-keystore add \
  xpack.security.http.ssl.keystore.secure_password

> type the pass-phrase


# truststore

/usr/share/elasticsearch/bin/elasticsearch-keystore add \
  xpack.security.http.ssl.truststore.secure_password

> type the pass-phrase
```

- Restart Elasticsearch:

```bash
sudo systemctl restart elasticsearch
```

- Kibana config:

```yml
# /etc/kibana/kibana.yml

elasticsearch.hosts: [ "https://localhost:9200" ]
elasticsearch.ssl.verificationMode: none
```

- Restart Kibana:

```bash
sudo systemctl restart kibana
```

## Create Roles

- Create a role with read-only permissions to all indexes:

```json
POST _secutiry/role/read_only
{
  "indices": [
    {
      "names": [ "*" ],
      "privileges": [ "read" ]
    }
  ]
}

GET _security/role/read_only
```

## Create Users

- Create a user:

```json
POST _security/user/new_user
{
  "roles": [ "read_only", "kibana_user" ],
  "full_name": "User Full Name",
  "email": "user@company.com",
  "password": "mypwd123"
}

GET _security/user/new_user
```
