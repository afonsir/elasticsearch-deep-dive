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
node.ingest: true
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

- Start the elasticsearch service:

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
node.ingest: false
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

- Start the elasticsearch service:

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
node.ingest: false
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

- Start the elasticsearch service:

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
